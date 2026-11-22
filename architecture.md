# Huddle Ticket Automation: Architecture

**Status:** Draft v0.2 · **Last updated:** 2026-11-22 · **Scope:** high-level architecture. Detailed designs (schemas, IAM policies, Terraform modules) live in separate documents.

---

## 1. Purpose

When Slack posts an AI-generated huddle summary to a channel (or a user runs `/meetingsummary`), this system:

1. extracts action items from the summary with an LLM,
2. drops any that duplicate tickets already in the channel's GitHub Project, and
3. adds the rest to that Project as **draft items** for manual review.

Today this runs as three n8n workflows (`transcription`, `setup`, `transcription-error-handler`), where the LLM both extracts tickets and drops duplicates in a single prompt that is given the existing tickets' titles and summaries. This document describes the target: **Python Lambdas on AWS**, deployed with **Terraform**, with selected components later rewritten in **Rust**.

## 2. Goals, constraints and non-goals

### Goals

- **Low cost.** Idle cost is approximately zero; per-run infrastructure cost is a fraction of a cent. LLM tokens are the dominant cost.
- **Self-hosted per organisation.** Each organisation deploys its own stack with Terraform and its own Slack app.
- **Language-neutral components.** Any Lambda can be rewritten in Rust without changing the others.
- **Production-grade.** Verified requests, idempotent processing, no stored transcripts, observable failures.

### Constraints

- Slack requires an acknowledgement within 3 seconds; OpenAI webhooks require a 2xx within a few seconds.
- GitHub integration uses a dedicated bot account with a classic personal access token (PAT), because user-owned Projects v2 are not reachable via fine-grained tokens or GitHub Apps. **User-owned projects only.**
- Users of a deployment are trusted members of one organisation.
- One environment for now.

### Non-goals (for now)

Multi-tenant hosting, organisation-owned projects, several repos per project, a UI, multiple environments, Zero Data Retention (ZDR) support.

## 3. System overview

```mermaid
flowchart LR
  subgraph External
    Slack[Slack]
    OpenAI[OpenAI API]
    Judge[Duplicate judge API]
    GitHub[GitHub GraphQL]
  end

  subgraph AWS["AWS (us-east-1)"]
    APIGW[API Gateway HTTP API]
    SummaryIn[Lambda: slack-summary]
    ProjectIn[Lambda: slack-newproject]
    Hook[Lambda: webhook]
    FailQ[("SQS: failed-events")]
    Starter[Lambda: starter]
    Finisher[Lambda: finisher]
    ProjectW[Lambda: newproject-worker]
    Jobs[("DynamoDB: jobs")]
    Links[("DynamoDB: channel_project_link")]
    SSM[("SSM Parameter Store")]
  end

  Slack -->|summary events and command| APIGW
  Slack -->|setup command and modal submissions| APIGW
  OpenAI -->|webhook| APIGW
  APIGW --> SummaryIn
  APIGW --> ProjectIn
  APIGW --> Hook
  SummaryIn -->|async invoke| Starter
  ProjectIn -->|async invoke| ProjectW
  ProjectIn -->|open modal| Slack
  Hook -->|async invoke| Finisher
  Starter -.->|on failure| FailQ
  Finisher -.->|on failure| FailQ
  ProjectW -.->|on failure| FailQ
  Starter --> Jobs
  Starter --> Links
  Finisher --> Jobs
  ProjectW --> Links
  Starter -->|fetch summary| Slack
  Starter -->|start background response| OpenAI
  Finisher -->|retrieve response| OpenAI
  Finisher -->|score duplicates| Judge
  Finisher -->|existing items, create drafts| GitHub
  Finisher -->|thread reply| Slack
  ProjectW -->|validate project and repo| GitHub
  ProjectW -->|confirmation| Slack
```

### Public routes (API Gateway HTTP API)

| Route | Caller | Handled by |
|---|---|---|
| `POST /slack/summary` | Slack: `file_shared` events (the app's Events request URL) and the `/meetingsummary` slash command | `slack-summary` |
| `POST /slack/newproject` | Slack: the `/setup` slash command and the setup modal submission (the app's Interactivity request URL) | `slack-newproject` |
| `POST /webhook` | OpenAI (`response.*` events) | `webhook` |

Routes are grouped by what they do, not by Slack payload type. Slack allows one Events request URL, one Interactivity request URL and a separate URL per slash command, so this mapping is configured in the Slack app manifest. The slash commands keep their user-facing names (`/meetingsummary`, `/setup`); only the URL paths differ.

`/slack/summary` receives two payload formats: JSON events and form-encoded slash-command payloads. A thin parser per format normalises both into one `StartWork` message, and verification and the hand-off are shared. Only the events format needs URL-verification handling and the huddle-summary filter, and only the slash command returns an in-channel acknowledgement.

A Slack app accepts only one URL for all event subscriptions and one for all interactive components, and both are pointed at the routes above. Every subscribed event therefore arrives at `/slack/summary`, and every interactive payload (such as a modal submission) arrives at `/slack/newproject`. If more event types or interactive components are added later, the handlers dispatch on event type or `callback_id` (or the routes are renamed).

The default API Gateway URL is used; no custom domain is required. There is **no VPC and no load balancer**.

## 4. Components

| Component | Trigger | Responsibility |
|---|---|---|
| **slack-summary** | API Gateway (`/slack/summary`) | Verify Slack signature and timestamp (raw body). Parse either a `file_shared` event or a `/meetingsummary` payload and normalise it into one `StartWork` message. For events: answer URL-verification challenges and ignore everything except Slack-generated huddle summary files. Asynchronously invoke `starter` (IDs only on the huddle path, summary text on the manual path) and return immediately; the slash command also returns an in-channel acknowledgement. Needs the Slack signing secret only, not the bot token. |
| **slack-newproject** | API Gateway (`/slack/newproject`) | Verify Slack signature and timestamp. On `/setup`, open the modal within Slack's `trigger_id` window. On modal submission, asynchronously invoke `newproject-worker` and acknowledge. Needs the signing secret and the bot token. |
| **starter** | Async invoke (from `slack-summary`) | Claim the job (conditional write). Look up the channel's project; if none, tell the user to run `/setup`; otherwise record `last_summary_at` on the link. Huddle path: resolve the file via `files.info` (channel and the message's `thread_ts`/`ts` from `shares`, public or private) and fetch the summary text with `canvases.getContent`. Start a **background** OpenAI response with the ticket-extraction prompt and structured-output schema. Record the response ID and move the job to `generating`. |
| **webhook** | API Gateway | Verify the webhook signature (Standard Webhooks headers). Asynchronously invoke `finisher` with `{response_id, event_type}`. Return 2xx only after Lambda has accepted the invocation. |
| **finisher** | Async invoke (from `webhook`) | Move the job to `finishing` (conditional transition, so duplicate deliveries are harmless). Retrieve the response. Fetch existing project items from GitHub (paginated). Score new tickets against existing ones and against each other using the duplicate judge. Create draft items for tickets below the threshold. Reply in the Slack thread. Mark the job `completed` or `failed`. |
| **newproject-worker** | Async invoke (from `slack-newproject`) | After the modal is submitted: validate the project URL and repo URL, confirm the bot can write to the project, accept a pending repo invitation if present, upsert the channel→project link, confirm in the channel. |

Hand-offs between Lambdas use **asynchronous Lambda invocation**. A Lambda stops running as soon as it returns, so work that must continue after a response runs in a separately invoked function. The caller invokes the next function with `InvocationType=Event`, receives an immediate acknowledgement that the event was accepted, and returns. Lambda holds the event, retries failed runs (up to two retries), and after the final retry or the maximum event age sends the event to a shared on-failure destination: the SQS queue `failed-events`. Retry settings, maximum event age and the failure queue's retention are configured per consumer function (values deferred, see §16). Delivery is at-least-once, which the conditional writes in §5.4 already tolerate.

## 5. Key flows

### 5.1 Huddle summary to draft tickets

```mermaid
sequenceDiagram
  autonumber
  participant S as Slack
  participant I as slack-summary
  participant ST as starter
  participant L as channel_project_link
  participant J as jobs
  participant O as OpenAI
  participant W as webhook
  participant F as finisher
  participant G as GitHub
  participant D as duplicate judge

  S->>I: file_shared event
  I->>I: verify signature, filter
  I->>ST: async invoke (IDs only)
  I-->>S: 200 (inside 3 seconds)
  ST->>J: claim job (conditional write)
  ST->>L: look up project, set last_summary_at
  ST->>S: fetch summary text
  ST->>O: create response (background mode)
  ST->>J: status = generating, store response ID
  O-->>W: response.completed webhook
  W->>W: verify signature
  W->>F: async invoke (response ID)
  W-->>O: 2xx
  F->>J: transition to finishing
  F->>O: retrieve response
  F->>G: fetch existing project items
  F->>D: score duplicates
  F->>G: create draft items
  F->>S: reply in thread
  F->>J: status = completed
```

### 5.2 Manual summary (`/meetingsummary`)

Enters through the same route as huddle summaries (`/slack/summary`) and is normalised into the same `StartWork` message. The flow is identical from step 3 onward, except the summary text arrives in the command payload (carried in the invocation payload) instead of being fetched from Slack. Both entry paths produce the same job shape.

### 5.3 Project setup (`/setup`)

1. `slack-newproject` receives `/setup` at `/slack/newproject` and opens a modal explaining what is needed and why (project URL, repository URL).
2. The user submits; Slack posts the submission to the same route; `slack-newproject` asynchronously invokes `newproject-worker` and acknowledges.
3. `newproject-worker` validates access, accepts a pending repo invitation if present, stores the link, and confirms in the channel.

The modal (rather than command arguments) is deliberate: it documents the requirements for the user and leaves room for a richer in-Slack form later.

### 5.4 Job lifecycle

```mermaid
stateDiagram-v2
  [*] --> received: claimed by starter
  received --> generating: OpenAI response created
  generating --> finishing: webhook delivered
  finishing --> completed: drafts created, Slack notified
  received --> failed
  generating --> failed: failed, cancelled or incomplete event
  finishing --> failed
  completed --> [*]
  failed --> [*]
```

Every transition is a **conditional write**. A duplicate invocation or webhook delivery fails the condition and is dropped, so every step is safe to retry.

## 6. Data model

DynamoDB, on-demand billing. No transcripts are ever written to DynamoDB.

### `channel_project_link`

| Attribute | Type | Notes |
|---|---|---|
| `channel_id` (PK) | string | Slack channel |
| `project_id` | string | GitHub Project node ID |
| `last_summary_at` | string (ISO 8601, UTC) | When a summary was last received for this project's channel. Set by the `starter` with an update (not an overwrite) so it never clobbers `project_id` |
| (metadata) | | Optional fields such as `updated_at` |

### `jobs`

| Attribute | Type | Notes |
|---|---|---|
| `job_id` (PK) | string | Doubles as the dedupe key (derivation deferred, see §16) |
| `status` | string | `received`, `generating`, `finishing`, `completed`, `failed` |
| `source` | string | `huddle` or `manual` |
| `channel_id`, `thread_ts` | string | Where to reply |
| `project_id` | string | Copied from the link at claim time |
| `response_id` | string | Indexed by a GSI so the finisher can look the job up |
| `created_item_ids` | list | Recorded as drafts are created, so a retry does not duplicate them |
| `error` | string | Failure reason, if any |
| `created_at`, `ttl` | number | Rows expire automatically |

### Configuration (SSM Parameter Store)

| Type | Parameters |
|---|---|
| SecureString | Slack signing secret, Slack bot token, GitHub PAT, OpenAI API key, OpenAI webhook secret, duplicate-judge API key |
| String | Duplicate threshold, generation model name, judge model/vendor name |

Secrets are fetched at cold start and cached in the execution environment.

## 7. Integrations and seams

Each external dependency sits behind a small interface so it can be faked in tests, swapped for another vendor, or reimplemented in Rust one adapter at a time.

| Interface | Purpose | Initial adapter | Possible swaps |
|---|---|---|---|
| `TicketGenerator` | Start and retrieve a background generation | OpenAI Responses API (GPT Luna) | Another vendor; synchronous mode |
| `DuplicateJudge` | Score each new ticket against existing tickets and each other, returning probabilities | Jev or OpenAI's decisions API (**evaluation pending**) | Embeddings plus threshold; plain LLM |
| `GitHubClient` | Read project items (paginated; Issue, DraftIssue and PullRequest titles and bodies), create draft items (`addProjectV2DraftIssue`), validate project access (`viewerCanUpdate`), list and accept repo invitations | GitHub GraphQL and REST | |
| `SlackClient` | Fetch file and canvas content, post thread replies, open modals | Slack Web API | |
| `JobStore`, `LinkStore` | Persistence | DynamoDB | |
| `Config` | Secrets and settings | SSM Parameter Store | |

### Duplicate detection

Duplicate checking is deliberately **separate from ticket generation**. The generation call no longer carries existing tickets, and a cheap calibrated classifier produces a probability per (new ticket, existing ticket) pair. The threshold lives in SSM and can be tuned without redeploying. The vendor is chosen by evaluating 30–50 labelled real ticket pairs, not by vendor benchmarks.

## 8. Language neutrality and repository layout

"Language-neutral" means the contracts between components do not depend on Python:

- **Contracts live outside code.** Invocation payloads (such as `StartWork`), the job record shape, the ticket schema and the judge request/response are JSON Schema files with example fixtures. Python (pydantic) and Rust (serde) implementations must both pass the same fixtures.
- **Duplicated logic is pinned by test vectors, not shared libraries.** Slack signature verification, Standard Webhooks verification and dedupe-key derivation exist in each language, so they are small and covered by shared vectors.
- **Deployment is runtime-agnostic.** One Terraform `lambda_function` module takes runtime, handler and artifact as variables (Python on a Python runtime; Rust as a `bootstrap` binary on `provided.al2023`). Use arm64 for both.
- **State is stored in language-neutral shapes** (plain strings, numbers, lists).

Proposed monorepo layout (indicative):

```
.
├── architecture.md
├── contracts/              # JSON Schemas, fixtures, signature test vectors
├── prompts/                # versioned LLM prompts
├── lambdas/
│   ├── slack_summary/
│   ├── slack_newproject/
│   ├── webhook/
│   ├── starter/
│   ├── finisher/
│   └── newproject_worker/
├── python_common/          # Python-only helpers shared by Lambdas (signature checks, config, Lambda invoke client)
├── slack/                  # app manifest template (derive from the current slack-bot-manifest.json)
├── infra/
│   ├── bootstrap/          # one-time: Terraform state bucket
│   └── terraform/
│       └── modules/        # lambda_function, failure_queue, http_api, tables, ssm_parameters
├── scripts/                # build and packaging
└── README.md
```

## 9. Security

- **Request verification.** Every public route verifies its signature over the raw request body before doing any work: Slack's signing secret plus timestamp window on `/slack/*`, the Standard Webhooks signature on `/webhook`. Unverified requests never reach an LLM.
- **Least-privilege IAM per Lambda.** For example, `webhook` can read only the webhook secret and invoke `finisher`; `slack-summary` can read only the Slack signing secret and invoke `starter`; only `starter` and `finisher` can read the OpenAI key.
- **Secrets** live in SSM Parameter Store (SecureString). Terraform creates the parameters with placeholder values and ignores later changes, so real values are set out of band and never enter Terraform state.
- **No stored transcripts.** DynamoDB holds IDs and status only. Transcript text is never logged. On the manual path the summary text travels in the invocation payload, which Lambda holds only until the event is processed or expires (maximum event age) and which is copied to `failed-events` if processing fails, so the maximum event age and that queue's retention are kept short (values deferred, see §16).
- **OpenAI retention.** Background mode makes OpenAI retain response data for roughly ten minutes and is not ZDR-compatible. Acceptable today (no retention requirement); it is one reason generation sits behind the `TicketGenerator` interface.
- **Draft-only output.** Tickets are created as draft items for human review. The LLM has no tools and its output is schema-constrained, which limits the impact of prompt injection from meeting content or existing tickets.
- **GitHub token scope.** The classic PAT needs `repo` and `project` scopes, which is broad. A dedicated bot account limits the blast radius. Because users are trusted, any workspace member can link a channel to any project the bot can access. This is an accepted limitation of the trusted-organisation model.

## 10. Reliability and failure modes

| Failure | Handling |
|---|---|
| Slack retries or posts the summary twice | Both trigger the starter; its conditional claim drops the second. |
| Slack 3-second deadline | The Slack routes only verify and hand off (or open the modal). |
| Async invoke is rejected at a Slack route or the webhook (for example throttled) | Return 5xx; the sender retries. 2xx is returned only after Lambda accepts the invocation. |
| Duplicate OpenAI webhook delivery (at-least-once) | Finisher's conditional transition drops duplicates. |
| OpenAI cannot reach the webhook | OpenAI retries for up to 72 hours with exponential backoff. |
| Generation fails, is cancelled or incomplete | The webhook subscribes to these events too; the finisher posts a failure message in the thread. |
| Finisher crashes mid-run | Lambda retries the invocation (up to two retries); steps are idempotent and created item IDs are recorded on the job. After the final retry the event goes to `failed-events`, which has an alarm. |
| Channel has no project configured | Reply asking the user to run `/setup` (before any LLM call). |
| Duplicate judge unavailable | Proposed default: **fail open** (create all tickets, note it in the thread), since drafts are reviewed anyway. |
| GitHub token revoked or lacking access | Job fails with a thread message and a `failed-events` alarm. |
| **OpenAI never delivers an event** | **Accepted risk.** No watchdog or polling fallback. Recovery is a manual re-run with `/meetingsummary`. |

Detailed error handling and alert routing are deferred until production behaviour is known.

## 11. Cost

Estimates at list prices (verify before relying on them); excludes LLM tokens.

| Item | Estimate |
|---|---|
| Infrastructure per run (a few Lambda-seconds, DynamoDB, API Gateway) | ≈ $0.025 per 1,000 runs |
| Fixed monthly infrastructure | ≈ $0 (free tiers, standard SSM parameters, on-demand DynamoDB, small log volume, a Terraform state bucket holding a few kilobytes) |
| HTTP API | ≈ $1 per million requests |

Deliberately avoided: an Application Load Balancer (roughly $16+/month idle), a VPC with a NAT gateway (roughly $32+/month), and Secrets Manager (about $0.40 per secret per month, with rotation we cannot use because none of our providers support it). Set explicit CloudWatch log retention to avoid unbounded log growth.

## 12. Infrastructure and deployment

- **Terraform** with remote state in S3. Recent Terraform versions support S3-native state locking, removing the need for a DynamoDB lock table (confirm the minimum version when implementing). A one-time `bootstrap` stack creates the state bucket.
- **Modules:** `lambda_function` (role, log group with retention, function, optional async-invoke settings with retries and failure destination), `failure_queue`, `http_api`, DynamoDB tables, SSM parameters.
- **Build:** a script builds Python zips (and later Rust binaries) before `terraform apply`; CI takes over at the CI/CD stage.

### First deployment (order matters)

1. Apply `bootstrap`, then the main stack. Outputs include the API base URL.
2. Create the Slack app from the manifest template **without** request URLs; install it; copy the signing secret and bot token.
3. Set all secret SSM values out of band (CLI).
4. Set the Slack request URLs: the Events URL and the `/meetingsummary` command URL point to `/slack/summary`; the `/setup` command URL and the Interactivity URL point to `/slack/newproject`. Slack verifies the Events URL, so `slack-summary` must answer the URL-verification challenge and its secrets must already exist.
5. Register the webhook URL in the OpenAI dashboard, subscribe to the `response.*` terminal events, and store the webhook secret in SSM.
6. Create the GitHub bot account and PAT; invite the bot to the channels, projects and repos to connect.

Steps 2, 5 and 6 are manual and belong in the README.

The current `slack-bot-manifest.json` already carries the bot scopes (`canvases:read`, `channels:history`, `channels:read`, `chat:write`, `commands`, `files:read`, `groups:history`, `groups:read`), the `file_shared` event and both slash commands, all with empty URLs. The target manifest additionally needs interactivity enabled (for the setup modal submission) and its request URL, and its `long_description` should stop referring to n8n.

## 13. Observability and testing

- **Logging:** structured JSON with `job_id` as the correlation ID. Python uses AWS Lambda Powertools (Logger, Parameters, Metrics); Rust will use `tracing`. Idempotency is hand-rolled via conditional writes (portable) rather than Powertools' idempotency utility.
- **Alarms:** `failed-events` queue depth and the Lambda `AsyncEventsDropped` metric, routed to an email or SNS topic (destination deferred).
- **Testing:** unit tests with fakes behind the interfaces in §7; contract tests against the shared fixtures; signature test vectors; real-AWS integration tests are defined at the CI/CD stage.

## 14. Delivery roadmap

1. **Python on AWS.** Implement the six Lambdas against the contracts. Recommended: deploy a walking skeleton early (`slack-summary` answering Slack's challenge) to validate the Slack and OpenAI handshakes before the full logic exists. Cut over from n8n using a test channel and project.
2. **Terraform.** Modules and first-deployment procedure as in §12.
3. **Rust.** Suggested order: `webhook` (smallest, a few-seconds deadline), then `slack-summary` (3-second deadline, where cold starts matter) and `slack-newproject`, then the workers.
4. **CI/CD.** Build, test, deploy pipeline; separate environments considered here.
5. **UI.** HTTP API supports JWT authorizers for when it is needed.
6. **Maintain, optimise, add features.**

## 15. Decision log

| # | Decision | Alternatives considered | Rationale |
|---|---|---|---|
| 1 | Lambdas + DynamoDB | Always-on EC2 | Pay per use; near-zero idle cost |
| 2 | API Gateway HTTP API; no ALB, no VPC | ALB, REST API, VPC | Cheapest; security comes from request verification and IAM, not networking |
| 3 | Background mode + webhook for the slow LLM call | Synchronous worker; polling; Step Functions; Batch API | Lowest per-run cost (no idle billing); accepted trade-offs: not ZDR-compatible, lost-event risk, manual webhook registration |
| 4 | SSM Parameter Store | Secrets Manager | Free; our secrets cannot use automatic rotation |
| 5 | Language-neutral contracts; Powertools in the Python phase | Shared Python library | Enables incremental Rust port |
| 6 | Separate, calibrated duplicate decision | Duplicates judged inside generation | Cheaper; tunable threshold; vendor-pluggable |
| 7 | Bot account with classic PAT | GitHub App; OAuth app | Only option that reaches user-owned Projects v2 |
| 8 | Self-hosted per organisation, one environment, `us-east-1` | Multi-tenant SaaS | Simplest model; cheapest region |
| 9 | Hand-rolled idempotency via conditional writes | Powertools idempotency utility | Portable to Rust |
| 10 | Modal-based `/setup` | Command arguments | Self-documenting; extensible to a richer Slack form |
| 11 | One Lambda per public endpoint (`slack-summary`, `slack-newproject`, `webhook`) | One Lambda routing all endpoints | Different secrets and permissions per route; independent deployment, failure isolation and Rust porting |
| 12 | Routes grouped by function: `/slack/summary` (file events and `/meetingsummary`), `/slack/newproject` (`/setup` and its modal), `/webhook` | Routes per Slack payload type (events, commands, interactivity) | One purpose and one handler per route; entry points that share logic share an endpoint |
| 13 | Asynchronous Lambda invocation for hand-offs, with one shared SQS failure queue | An SQS queue plus DLQ per consumer; HTTP calls between Lambdas; self-invocation | A Lambda stops when it returns, so post-response work needs its own invocation. Async invoke gives automatic retries and failure capture with far fewer resources. A visible backlog and per-hop concurrency caps are not needed at this scale (revisit if they become necessary, see §16) |

## 16. Deferred, open items and known issues

### Deferred (out of scope for this document)

- Dedupe-key choice for Slack's double post (file ID versus event ID)
- Detailed error handling and alert routing
- AWS tuning: reserved concurrency, budgets and alarms, throttling
- Async invocation settings (retry count, maximum event age) and `failed-events` retention; the async payload size limit versus the longest manual summary (verify the current limit)
- Reintroducing SQS between hops if a visible backlog or per-hop concurrency caps are needed
- Separate environments, CI/CD, UI
- Showing suppressed duplicates to users
- Zero Data Retention support (would require synchronous generation)
- Organisation-owned projects and several repos per project
- Per-channel duplicate thresholds

### Open items to confirm

- **Duplicate-judge vendor** (Jev, OpenAI's decisions API, or other): decide after evaluating labelled ticket pairs.
- **Judge-outage policy:** proposed default is fail open.
- **Manual-path retention:** maximum event age and `failed-events` retention should be short, since the payload carries transcript text.

### Known issues in the current n8n workflow (fix, do not port)

Re-verified against `n8n-workflows/*.json` on 2026-11-22; all are still present.

**`transcription.json`**

- **No pagination.** *Get Existing Project Tickets* queries `items(first: 100)` and never pages, so projects with more than 100 items are only partly compared. It reads Issue, DraftIssue and PullRequest titles and bodies.
- **Summary regex never matches.** *Extract Title and Summary* uses `## Summary\s*([^#]*)`, but the prompt's body template emits `## 1. Summary`. (The structured-output schema's `description` field says `## Summary`, so the prompt and schema disagree with each other too.) Result: the existing-ticket "summary" is always an empty string and the LLM compares titles only.
- **Wrong `thread_ts` on the slash-command path.** *Code in JavaScript1* sets `thread_ts` from the `x-slack-request-timestamp` header, which is the request time, not a message timestamp. Replies on this path therefore do not attach to the user's message. (On the huddle path it is correctly the file's message `thread_ts`/`ts`, but it is stored as a number, which risks precision loss on Slack timestamps.)
- **Racy duplicate window.** The per-channel lock (*Get Previous File Upload to Channel* → *Check if File Sent Last 10 Seconds* → *Upsert Record of File Upload*) is a read-then-write, not atomic, so two near-simultaneous events can both pass. It also silently drops any legitimate upload within 10 seconds of the previous one. It applies to the huddle path only.
- **Prompt defects.** The prompt contains paste artifacts (a stray `[11:23 AM]`, non-breaking spaces and run-together markdown) and tells the model to output "an empty JSON array" while the schema requires `{ "issues": [] }`.
- **Weak huddle filter.** *If1* only checks `user_id == USLACKBOT`; any Slackbot-shared file passes, not just huddle summaries. The target filters on Slack-generated huddle summary files.
- **No request verification.** The Slack trigger node and the two raw webhooks (`/meetingsummary`, `/setup`) do not verify Slack signatures (the README even says not to add a signing secret to the credential). The target verifies every request (§9).
- **Partial failures.** Drafts are created one at a time; a failure partway is reported only as a count ("N tickets encountered an error"), with no record of which were created, so a retry can duplicate them. The target records `created_item_ids` (§6).
- **Error handler coverage.** *transcription-error-handler* can reply in the thread only if *Get Upload Message Timestamp* ran, i.e. the huddle path after file lookup; other failures go to the channel root. It also emails via Gmail.

**`setup.json`**

- **Narrow URL parsing.** Only `https://github.com/users/<login>/projects/<n>` URLs match. The repository is assumed to be owned by the same login as the project (`GET /repos/{login}/{repo}` using the last URL segment).
- **Repo is validated but never stored or used.** The link table holds only `channel_id` and `project_id`. Duplicate detection reads project items, not the repo, so the README's statement that "only the one you enter here will be scanned for duplicate tickets" is inaccurate for the current workflow.
- **Invitation lookup not paginated.** `user/repository_invitations?per_page=100` reads one page.
- **Interaction model.** Setup uses an n8n *Send and Wait* form (60-minute limit) rather than a Slack modal; the modal is a target-design change (decision 10).

**Data tables.** The workflows use two n8n data tables (`slack_channel_lock`: `channel_id,last_upload_ts`; `slack_channel_project_link`: `channel_id,project_id`), matching the two schema CSVs. The README describes a single three-column table and should be corrected separately. In the target, the lock table is replaced by the `jobs` dedupe key and `last_summary_at` on the link (§6).

## 17. Glossary

- **Lambda:** a short-lived function billed per millisecond of execution.
- **HTTP API:** the cheaper API Gateway type that routes public HTTPS requests to Lambdas.
- **Async invocation:** invoking a Lambda without waiting for it to finish. Lambda accepts the event, returns immediately, runs it later and retries failures.
- **SQS:** a message queue. Here it is only the `failed-events` queue that collects events which exhausted their retries.
- **Idempotent:** safe to run more than once with the same effect as running once.
- **Conditional write:** a DynamoDB write that succeeds only if a condition holds (for example "this row does not exist yet"); the basis of our dedupe and state transitions.
- **GSI:** global secondary index, which lets DynamoDB look up rows by an attribute other than the primary key.
- **TTL:** a DynamoDB feature that automatically deletes rows after a timestamp.
- **Webhook:** an HTTP call a service makes to you when something happens.
- **Standard Webhooks:** the signature scheme OpenAI uses (`webhook-id`, `webhook-timestamp`, `webhook-signature` headers).
- **SSM Parameter Store:** AWS's configuration and secrets store; SecureString parameters are KMS-encrypted.

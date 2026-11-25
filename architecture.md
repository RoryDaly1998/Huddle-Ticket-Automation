# Huddle Ticket Automation: Architecture

**Status:** Draft v0.5 · **Last updated:** 2026-11-25 · **Scope:** high-level architecture. Detailed designs (schemas, IAM policies, Terraform modules) live in separate documents.

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
    Embed[Embeddings API]
    GitHub[GitHub GraphQL]
  end

  subgraph AWS["AWS (us-east-1)"]
    APIGW[API Gateway HTTP API]
    SummaryIn[Lambda: summary-posted]
    ProjectIn[Lambda: project-added]
    Hook[Lambda: issues-generated]
    Starter[Lambda: process-summary]
    Finisher[Lambda: process-issues]
    ProjectW[Lambda: setup-project]
    FailQ[("SQS: failed-events")]
    FailW[Lambda: job-failed]
    Sched[EventBridge Scheduler]
    Stale[Lambda: check-stale-jobs]
    Jobs[("DynamoDB: jobs")]
    FailedJobs[("DynamoDB: failed_jobs")]
    Links[("DynamoDB: channel_project_link")]
    Embeddings[("DynamoDB: ticket_embeddings")]
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
  Starter -.->|failure| FailQ
  Finisher -.->|failure| FailQ
  ProjectW -.->|failure| FailQ
  FailQ --> FailW
  FailW -->|mark failed| Jobs
  FailW -->|log| FailedJobs
  FailW -->|failure message| Slack
  Sched -->|every 15 min| Stale
  Stale -->|find stale jobs| Jobs
  Stale -->|check response| OpenAI
  Stale -->|recover| Finisher
  Stale -.->|failure| FailQ
  Starter --> Jobs
  Starter --> Links
  Finisher --> Jobs
  Finisher -->|cached vectors| Embeddings
  ProjectW --> Links
  Starter -->|fetch summary| Slack
  Starter -->|start background response| OpenAI
  Finisher -->|retrieve response| OpenAI
  Finisher -->|embed tickets| Embed
  Finisher -->|score duplicates| Judge
  Finisher -->|existing items, create drafts| GitHub
  Finisher -->|thread reply| Slack
  ProjectW -->|validate project and repo| GitHub
  ProjectW -->|confirmation| Slack
```

### Public routes (API Gateway HTTP API)

| Route | Caller | Handled by |
|---|---|---|
| `POST /slack/summary` | Slack: `file_shared` events (the app's Events request URL) and the `/meetingsummary` slash command | `summary-posted` |
| `POST /slack/newproject` | Slack: the `/setup` slash command and the setup modal submission (the app's Interactivity request URL) | `project-added` |
| `POST /webhook` | OpenAI (`response.*` events) | `issues-generated` |

Routes are grouped by what they do, not by Slack payload type. Slack allows one Events request URL, one Interactivity request URL and a separate URL per slash command, so this mapping is configured in the Slack app manifest. The slash commands keep their user-facing names (`/meetingsummary`, `/setup`); only the URL paths differ.

`/slack/summary` receives two payload formats: JSON events and form-encoded slash-command payloads. A thin parser per format normalises both into one `summary_posted_to_process_summary` message, and verification and the hand-off are shared. Only the events format needs URL-verification handling and the huddle-summary filter, and only the slash command returns an in-channel acknowledgement.

A Slack app accepts only one URL for all event subscriptions and one for all interactive components, and both are pointed at the routes above. Every subscribed event therefore arrives at `/slack/summary`, and every interactive payload (such as a modal submission) arrives at `/slack/newproject`. If more event types or interactive components are added later, the handlers dispatch on event type or `callback_id` (or the routes are renamed).

The default API Gateway URL is used; no custom domain is required. There is **no VPC and no load balancer**.

## 4. Components

| Component | Trigger | Responsibility |
|---|---|---|
| **summary-posted** | API Gateway (`/slack/summary`) | Verify Slack signature and timestamp (raw body). Parse either a `file_shared` event or a `/meetingsummary` payload and normalise it into one `summary_posted_to_process_summary` message. For events: answer URL-verification challenges and ignore every file not shared by Slackbot (`user_id == USLACKBOT`), a cheap pre-filter, since the event carries no huddle fields and this Lambda holds only the signing secret. The authoritative huddle check runs in `process-summary`. Asynchronously invoke `process-summary` (IDs only on the huddle path, summary text on the manual path) and return immediately; the slash command also returns an in-channel acknowledgement. Needs the Slack signing secret only, not the bot token. |
| **project-added** | API Gateway (`/slack/newproject`) | Verify Slack signature and timestamp. On `/setup`, open the modal within Slack's `trigger_id` window. On modal submission, asynchronously invoke `setup-project` and acknowledge. Needs the signing secret and the bot token. |
| **process-summary** | Async invoke (from `summary-posted`) | Claim the job (conditional write). Look up the channel's project; if none, tell the user to run `/setup`; otherwise record `last_summary_at` on the link. Huddle path: resolve the file via `files.info` (channel and the message's `thread_ts`/`ts` from `shares`, public or private); if it is not a Slack-generated huddle summary, end the job quietly before any LLM call; otherwise fetch the summary text with `canvases.getContent`. Manual path: slash commands carry no message timestamp, so find `summary-posted`'s in-channel ack (it includes a short job reference) with `conversations.history` and thread replies under it, falling back to channel-level replies if it never appears. Start a **background** OpenAI response with the ticket-extraction prompt and structured-output schema. Record the response ID and move the job to `generating`. A permanent error is sent to `failed-events` (§5.5). |
| **issues-generated** | API Gateway | Verify the webhook signature (Standard Webhooks headers). Asynchronously invoke `process-issues` with `{response_id, event_type}`. Return 2xx only after Lambda has accepted the invocation. |
| **process-issues** | Async invoke (from `issues-generated` or `check-stale-jobs`) | Move the job to `finishing` (conditional transition, so duplicate deliveries are harmless). Retrieve the response. Fetch the project's existing Issues and DraftIssues from GitHub (paginated; pull requests and archived items are excluded). Sync the embedding cache, embed the new tickets, shortlist the most similar candidates for each new ticket, and judge each new ticket against its shortlist in parallel (§7, *Duplicate detection*). Create draft items, each with a `## Summary` section, for tickets that are not duplicates, in one GraphQL request, and cache their embeddings. Reply in the Slack thread and mark the job `completed`. A failed, cancelled or incomplete generation, or a permanent error, is sent to `failed-events`. |
| **setup-project** | Async invoke (from `project-added`) | After the modal is submitted: validate the project URL and repo URL, confirm the bot can write to the project, accept a pending repo invitation if present, upsert the channel→project link, confirm in the channel. Problems the user can fix are explained in the channel; a permanent GitHub error is sent to `failed-events`. |
| **job-failed** | SQS (`failed-events`) | The only component that fails jobs and posts failure messages (§5.5). Mark the job `failed` unless it has completed, write a permanent row to `failed_jobs`, and, only when that row is new, post one failure message. |
| **check-stale-jobs** | EventBridge Scheduler (every 15 minutes) | Find jobs neither `completed` nor `failed` 30 minutes after they were created. Recover a `generating` job whose OpenAI response has completed by invoking `process-issues`; send every other one to `failed-events` (§5.6). |

Hand-offs between Lambdas use **asynchronous Lambda invocation**. A Lambda stops running as soon as it returns, so work that must continue after a response runs in a separately invoked function. The caller invokes the next function with `InvocationType=Event`, receives an immediate acknowledgement that the event was accepted, and returns. Lambda holds the event and retries failed runs (two retries). After the final retry, or once the event is older than the **maximum event age of 10 minutes**, Lambda sends the invocation record to the workers' on-failure destination, the SQS queue `failed-events`. Delivery is at-least-once, which the conditional writes in §5.4 already tolerate.

## 5. Key flows

### 5.1 Huddle summary to draft tickets

```mermaid
sequenceDiagram
  autonumber
  participant S as Slack
  participant I as summary-posted
  participant ST as process-summary
  participant L as channel_project_link
  participant J as jobs
  participant O as OpenAI
  participant W as issues-generated
  participant F as process-issues
  participant G as GitHub
  participant E as ticket_embeddings
  participant M as embeddings API
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
  F->>E: read cached vectors
  F->>M: embed cache misses and new tickets
  F->>E: write misses, refresh TTLs, delete stale rows
  F->>D: judge each new ticket against its shortlist (parallel)
  F->>G: create draft items (one request)
  F->>E: cache the created drafts' vectors
  F->>S: reply in thread
  F->>J: status = completed
```

### 5.2 Manual summary (`/meetingsummary`)

Enters through the same route as huddle summaries (`/slack/summary`) and is normalised into the same `summary_posted_to_process_summary` message. The flow is identical from step 3 onward, except the summary text arrives in the command payload (carried in the invocation payload) instead of being fetched from Slack. Both entry paths produce the same job shape.

### 5.3 Project setup (`/setup`)

1. `project-added` receives `/setup` at `/slack/newproject` and opens a modal explaining what is needed and why (project URL, repository URL).
2. The user submits; Slack posts the submission to the same route; `project-added` asynchronously invokes `setup-project` and acknowledges.
3. `setup-project` validates access, accepts a pending repo invitation if present, stores the link, and confirms in the channel.

The modal (rather than command arguments) is deliberate: it documents the requirements for the user and leaves room for a richer in-Slack form later.

### 5.4 Job lifecycle

```mermaid
stateDiagram-v2
  [*] --> received: claimed by process-summary
  received --> generating: OpenAI response created
  generating --> finishing: webhook delivered
  finishing --> completed: drafts created, Slack notified
  received --> completed: not a huddle summary, or no project linked
  received --> failed: by job-failed
  generating --> failed: by job-failed
  finishing --> failed: by job-failed
  completed --> [*]
  failed --> [*]
```

Every transition is a **conditional write**. A duplicate invocation or webhook delivery that finds the job has moved on fails the condition and is dropped, so every step is safe to retry. While `process-summary` or `process-issues` works on a job it holds a **lease** (`claimed_at` while `received`, `finishing_started_at` while `finishing`). A delivery that meets a live lease raises a retryable `LeaseHeld` and is retried by Lambda; once the lease has expired or been released, the retry takes the job over and resumes it.

Only `job-failed` moves a job to `failed`, and nothing moves a job out of `failed` or `completed`. So a Lambda retry still pending when a job is failed finds it `failed` and drops itself.

### 5.5 Failures

Every failure takes one path, so users hear about each failed job exactly once and every failure is logged permanently.

```mermaid
flowchart LR
  W["process-summary, process-issues, setup-project"] -->|permanent error, then return| Q[("SQS: failed-events")]
  L["Lambda async retries exhausted, max event age passed, timeout or crash"] -->|on-failure destination| Q
  C[check-stale-jobs] -->|stale job| Q
  Q --> F[job-failed]
  F -->|failed, unless completed| J[("jobs")]
  F -->|new row, message_sent| FJ[("failed_jobs")]
  F -->|once, if the row is new| S[Slack]
```

1. **Workers** classify errors (`python_common.failures`). A permanent one (for example GitHub refusing a request, or a generation that failed) is sent to `failed-events` with the job's ID, channel, thread if known and a short reason code, and the worker returns normally, so Lambda does not retry. A retryable one releases the lease and is re-raised for Lambda's retries; if those run out, Lambda's on-failure destination sends the invocation record to the same queue. Workers never mark jobs failed or post failure messages. If the send to the queue itself fails, the worker treats it as retryable.
2. **`job-failed`** handles each message once:
   - Skip a final error of `LeaseHeld`: another run owns the job.
   - Mark the job `failed` with the reason, in one conditional write that is refused if the job has `completed` (then stop). A failure with no job row, such as one before the claim or from `setup-project`, carries on.
   - Create the `failed_jobs` row, only if its `id` is new. If it exists, stop: the failure was already handled, so a redelivered message never posts twice.
   - For a huddle job that never reached `generating`, confirm with `files.info` that the file is a huddle summary; otherwise post nothing, as for any non-summary file.
   - Post one message, in the thread if known, otherwise in the channel. Summary failures ask the user to check the project and run `/meetingsummary`; resharing a huddle file does not work, because its job ID is already taken. Setup failures ask the user to run `/setup` again.
   - Record whether the message was sent (`message_sent`). A Slack error is logged and never retried: failed runs are not critical, since a user can always rerun.
3. **Not failures:** a shared file that is not a huddle summary, and a channel with no project, end the job as `completed` without a failure.

### 5.6 Stale jobs

EventBridge Scheduler runs `check-stale-jobs` every 15 minutes. It reads every job that is neither `completed` nor `failed` and was created more than **30 minutes** ago:

| Job state | Action |
|---|---|
| `generating`, and OpenAI reports the response completed | Invoke `process-issues` to finish it, recovering a lost or late webhook |
| Any other stale job (`received`, `finishing`, or `generating` whose response failed, is still running, or could not be checked) | Send it to `failed-events` |

The 30-minute cutoff relies on the workers' **maximum event age of 10 minutes**. Each stage then ends its retries within about 13 minutes, including a final run at its Lambda timeout, which leaves a few minutes for generation. So no original Lambda retry should still be pending or running when a job is failed for being stale. If either value changes, revisit both.

Accepted edge cases: a webhook that arrives after its job was failed is dropped; and a recovery or run still in progress when its job is failed may still create drafts, so the user can see both drafts and a failure message.

## 6. Data model

DynamoDB, on-demand billing. No transcripts are ever written to DynamoDB.

### `channel_project_link`

| Attribute | Type | Notes |
|---|---|---|
| `channel_id` (PK) | string | Slack channel |
| `project_id` | string | GitHub Project node ID |
| `last_summary_at` | string (ISO 8601, UTC) | When a summary was last received for this project's channel. Set by `process-summary` with an update (not an overwrite) so it never clobbers `project_id` |
| (metadata) | | Optional fields such as `updated_at` |

### `jobs`

| Attribute | Type | Notes |
|---|---|---|
| `job_id` (PK) | string | Doubles as the dedupe key. Huddle path: the Slack `file_id`. Manual path: lowercase hex `sha256(channel_id + ':' + trigger_id)`, pinned by `contracts/vectors/job_id.json`. Revisit once real Slack payloads confirm `file_id` is stable across Slack's double post |
| `status` | string | `received`, `generating`, `finishing`, `completed`, `failed` |
| `source` | string | `huddle` or `manual` |
| `channel_id`, `thread_ts` | string | Where to reply |
| `project_id` | string | Copied from the link at claim time |
| `response_id` | string | Indexed by a GSI so `process-issues` and `job-failed` can look the job up |
| `error` | string | Failure reason, set by `job-failed` |
| `created_at`, `ttl` | number | Unix seconds. `check-stale-jobs` measures staleness from `created_at`; rows expire after 30 days |
| `claimed_at` | number | Unix seconds when `process-summary`'s claim lease (90 s) started; removed on release |
| `finishing_started_at` | number | Unix seconds when `process-issues`'s lease (180 s) started; removed on release |

### `failed_jobs`

The permanent failure log, written only by `job-failed`, one row per failure. **No TTL.** It never holds summary text, invocation payloads or raw error messages.

| Attribute | Type | Notes |
|---|---|---|
| `id` (PK) | string | The job ID; for `setup-project`, which has no job, the async invocation's request ID. Created with a conditional put, so each failure is handled once |
| `sender` | string | The Lambda that failed: `process-summary`, `process-issues`, `setup-project` or `check-stale-jobs` |
| `reason` | string | A short code: the worker's error code (e.g. `github_forbidden`, `generation_failed`), `<condition>:<errorType>` for exhausted retries (e.g. `RetriesExhausted:GitHubUnavailable`), or `stale_<status>` |
| `channel_id`, `thread_ts` | string | Where the message went; `thread_ts` only if known |
| `failed_at` | number | Unix seconds when `job-failed` logged it |
| `message_sent` | boolean | `false` when the row is created, `true` once Slack accepts the failure message; a failed post leaves it `false` and is not retried |

### `ticket_embeddings`

A disposable cache of embedding vectors for existing project items, so each run embeds only new or changed tickets. Any row can be lost without harm: a missing row is a cache miss and is re-embedded.

| Attribute | Type | Notes |
|---|---|---|
| `project_id` (PK) | string | GitHub Project node ID; one Query returns a project's whole cache |
| `item_id` (SK) | string | Project item node ID (`PVTI_…`), the same for drafts we create and items added by hand |
| `text_hash` | string | Lowercase hex SHA-256 of the item's embed text (§7). A different hash means the title or summary was edited, so the row is a miss |
| `embedding` | binary | The vector as little-endian float32 values |
| `embedding_model`, `dimensions` | string, number | The model and dimension count the vector was made with. A mismatch with the current SSM values is a miss, so changing either re-embeds lazily with no migration |
| `ttl` | number | Unix seconds; DynamoDB deletes the row after it. Set to 180 days ahead on write, and pushed back to 180 days ahead whenever a run finds less than 90 days left. Both values are code constants |

Rows are written for cache misses, refreshed as above, and deleted when their `item_id` is no longer among the project's items. Deletion happens only after the paginated GitHub fetch has completed, so a partial fetch never deletes rows. The table holds vectors and hashes derived from ticket text, never the text itself.

### Configuration (SSM Parameter Store)

| Type | Parameters |
|---|---|
| SecureString | Slack signing secret, Slack bot token, GitHub PAT, OpenAI API key, OpenAI webhook secret, duplicate-judge API key, embeddings API key |
| String | Duplicate threshold, generation model name, judge model name, embedding model name (initially `text-embedding-3-small`), embedding dimensions (initially 256), shortlist size K (initially 10), shortlist minimum similarity (initially 0.5) |

Secrets are fetched at cold start and cached in the execution environment.

## 7. Integrations and seams

Each external dependency sits behind a small interface so it can be faked in tests, swapped for another vendor, or reimplemented in Rust one adapter at a time.

| Interface | Purpose | Initial adapter | Possible swaps |
|---|---|---|---|
| `TicketGenerator` | Start and retrieve a background generation | OpenAI Responses API (GPT Luna) | Another vendor; synchronous mode |
| `Embedder` | Turn ticket texts into vectors, in batches | OpenAI embeddings API (`text-embedding-3-small`, 256 dimensions) | Another embeddings vendor; AWS Bedrock |
| `DuplicateJudge` | Score a new ticket against its shortlisted candidates, returning a probability per pair | OpenAI's Decisions API | Another classifier; plain LLM |
| `GitHubClient` | Read project items (paginated; titles and bodies of Issues and DraftIssues, excluding pull requests and archived items), create a run's draft items in one request (one aliased `addProjectV2DraftIssue` per draft), validate project access (`viewerCanUpdate`), list and accept repo invitations | GitHub GraphQL and REST | |
| `SlackClient` | Fetch file and canvas content, post thread replies, open modals | Slack Web API | |
| `JobStore`, `LinkStore`, `EmbeddingStore`, `FailedJobStore` | Persistence | DynamoDB | |
| `FailureQueue` | Send a failure to `job-failed` | SQS (`failed-events`) | |
| `Config` | Secrets and settings | SSM Parameter Store | |

### Duplicate detection

Duplicate checking is deliberately **separate from ticket generation**. The generation call no longer carries existing tickets. Instead, a two-stage funnel runs in `process-issues`:

1. **Shortlist by embeddings (recall).** Every ticket is embedded, and cosine similarity picks the few plausible matches for each new ticket. This stage is cheap and only rules out clearly unrelated tickets.
2. **Judge the shortlist (precision).** OpenAI's Decisions API reads the new ticket and each shortlisted candidate together and returns the probability that they describe the same work. A probability at or above the duplicate threshold (SSM) is a duplicate.

The two stages fail differently: embeddings measure topic, so "Add dark mode" and "Fix dark mode contrast bug" score as similar, while the judge can tell related tickets from duplicates. Thresholds are tuned against 30–50 labelled real ticket pairs with `scripts/eval_judge.py` (see `evals/README.md`).

**Draft body format.** Every draft `process-issues` creates has the body `## Summary\n\n{summary}\n\n{body}`, where `summary` and `body` come from the extracted ticket (`ticket_generator_to_process_issues.schema.json`). The `## Summary` section is a core part of the design: it is what duplicate detection compares. The extraction schema (and the matching pydantic model) makes the section parse back exactly: `summary` is one line with no CR or LF that starts with neither whitespace nor `#`, and `body` must start with `#`, so the section always ends at the body's first heading.

**Embed text.** Every ticket, new or existing, is compared through one text, built the same way everywhere:

- The title, trimmed, then a blank line, then the summary, trimmed: `{title}\n\n{summary}`. If there is no summary, the text is the title alone.
- Both parts are capped before joining: trim, keep the first N Unicode code points (256 for the title, GitHub's own limit; 1,500 for the summary), then trim the end again. The longest embed text is therefore 1,758 code points, at most 7,032 UTF-8 bytes, which keeps it under the embeddings API's 8,192-token input limit (its tokenizer never produces more tokens than bytes).
- A new ticket's summary is its `summary` field.
- An existing item's summary is the section under the first line that is exactly `## Summary` (trailing whitespace ignored), up to the next line starting with `#` or the end of the body. Line endings are normalised to `\n` first. An item with no such heading, or an empty section, has no summary, so only its title is compared. This includes `## 1. Summary` headings from n8n-era tickets and issues written by hand.
- The summary in a draft we created must parse back **exactly** to the ticket's `summary`, so the text (and hash) `process-issues` caches when it creates a draft equals the text the next run builds from GitHub.
- The same text (capped) is embedded, hashed for the cache (§6) and sent to the judge. The builder, including the caps, is pinned by shared test vectors (§8).

**Algorithm** (per `process-issues` run):

1. **Embed.** Build the embed text for each existing item and look it up in the cache. Embed the misses and all new tickets in batched calls, then update the cache (§6).
2. **Shortlist.** For each new ticket, the candidate pool is the existing items plus every other new ticket. Keep the K most similar candidates (K = 10 initially) whose cosine similarity is at least the minimum (0.5 initially). Both values live in SSM. An empty shortlist means no judge call and no duplicate.
3. **Judge in parallel.** One Decisions request per new ticket with a non-empty shortlist: the new ticket is the `input`, and each candidate is one predicate question named by its ID. Requests run concurrently, with a small fixed concurrency cap. There is no overall time budget: each request has its own timeout.
4. **Resolve in extraction order.** Walk the new tickets in the order the model extracted them. A ticket is a **duplicate** if its normalised title equals that of an existing item or an earlier kept ticket, or if it scores at or above the threshold against an existing item, or against an earlier kept ticket in either direction (its own request, or the earlier ticket's request). Otherwise it is kept. So when two new tickets duplicate each other, the later one is dropped. A ticket that matches only an earlier *dropped* ticket is kept.
5. **Create and cache.** Create drafts for all kept tickets in one GraphQL request, then write their vectors to the cache under their new item IDs, so the next run does not re-embed them.

**No partial-success tracking.** The request is not a transaction: GitHub can create some drafts and then return an error or time out. Any error fails the run as a whole, retryable or permanent as usual. A retry fetches the project again, so the title check in step 4 skips the drafts an earlier attempt created (they appear in the reply as duplicates). Nothing records which drafts a run created.

**Failure handling (fail open).** Drafts are reviewed by people, so a failed check creates tickets rather than losing them:

- Embeddings API failure: skip steps 2–3; only the title checks in step 4 apply.
- A Decisions request fails, or one answer is a refusal or missing: those pairs are unscored and count as not duplicates. Other tickets' requests are unaffected, and a pair can still be caught by the other ticket's request.
- Cache read or write failure: proceed as if every item were a miss, and skip the cache writes. The cache never fails a run.

## 8. Language neutrality and repository layout

"Language-neutral" means the contracts between components do not depend on Python:

- **Contracts live outside code.** Invocation payloads (such as `summary_posted_to_process_summary`), the job record shape, the ticket schema and the judge request/response are JSON Schema files with example fixtures. Python (pydantic) and Rust (serde) implementations must both pass the same fixtures.
- **Schema naming rule (project-wide).** Every schema is named `<sender>_to_<receiver>` in snake_case, so the direction of every exchange reads from the file name (`contracts/schemas/<name>.schema.json`). The sides are named as follows:
  - our services use their Lambda names, or `lambdas` when several Lambdas make the same call;
  - storage is `db_<what is stored>` (`db_jobs`, `db_embeddings`);
  - swappable vendor seams use their role (`ticket_generator`, `duplicate_judge`, `embedder`), so a vendor swap does not rename the contract;
  - Slack and GitHub use the vendor name.

  | Kind | Schemas |
  |---|---|
  | Hand-offs between our Lambdas | `summary_posted_to_process_summary`, `lambdas_to_process_issues` (from `issues-generated` and `check-stale-jobs`), `project_added_to_setup_project` |
  | Failures | `lambdas_to_job_failed` (failures our Lambdas send to `failed-events`), `lambda_to_job_failed` (the Lambda service's on-failure record, as it arrives through the queue) |
  | Storage | `lambdas_to_db_jobs`, `process_issues_to_db_embeddings`, `job_failed_to_db_failed_jobs` |
  | OpenAI | `ticket_generator_to_process_issues` (the extracted tickets, also sent as the structured-output format), `ticket_generator_to_issues_generated` (the webhook event), `process_summary_to_ticket_generator` (the Responses request), `process_issues_to_embedder`, `embedder_to_process_issues` |
  | Duplicate judge | `process_issues_to_duplicate_judge`, `duplicate_judge_to_process_issues` |
  | Slack | `slack_to_summary_posted`, `slack_to_project_added` (what Slack posts to our routes; form bodies are described decoded), `slack_to_lambdas` (Web API responses we read), `lambdas_to_slack` (Web API requests we send), `summary_posted_to_slack`, `project_added_to_slack` (our replies to Slack) |
  | GitHub | `lambdas_to_github` (GraphQL operations, variables and REST invitation calls), `github_to_lambdas` (the response fields we read) |

  Schemas for payloads we receive describe only the fields we rely on and allow any others; schemas for payloads we send are exact. A schema that covers several calls holds one `$def` per call, and a document is checked against its call's `$def`; its fixtures are named `<def>__<case>.json`. The structured-output format inside `process_summary_to_ticket_generator` is the `ticket_generator_to_process_issues` schema: the request schema describes it as an object, and fixtures write it as `{"$ref": "ticket_generator_to_process_issues.schema.json"}`, which tests replace with the file's content (minus `$schema`, as sent) before checking the request. The OpenAI Response envelope (status, output text, errors) has no schema: `ticket_generator_to_process_issues` is sent to OpenAI verbatim and must stay a plain strict-mode schema, so the envelope is validated by pydantic only (`ResponseEnvelope`; a malformed one fails the job with a generation error, decision 26). The Slack reply schemas (`summary_posted_to_slack`, `project_added_to_slack`) describe the JSON bodies only; an empty 200 has no body and no schema. Pydantic class names for stored rows and vendor payloads stay domain names (`Job`, `TicketEmbedding`, `JudgeRequest`, …); hand-off models use the schema name in PascalCase.
- **Duplicated logic is pinned by test vectors, not shared libraries.** Slack signature verification, Standard Webhooks verification, dedupe-key derivation, and the embed text and its hash (§7) exist in each language, so they are small and covered by shared vectors.
- **Deployment is runtime-agnostic.** One Terraform `lambda_function` module takes runtime, handler and artifact as variables (Python on a Python runtime; Rust as a `bootstrap` binary on `provided.al2023`). Use arm64 for both.
- **State is stored in language-neutral shapes** (plain strings, numbers, lists, and little-endian float32 bytes for vectors).

Proposed monorepo layout (indicative):

```
.
├── architecture.md
├── contracts/              # JSON Schemas, fixtures, signature test vectors
├── prompts/                # versioned LLM prompts
├── src/                    # all deployable Python code
│   ├── lambdas/
│   │   ├── summary_posted/
│   │   ├── project_added/
│   │   ├── issues_generated/
│   │   ├── process_summary/
│   │   ├── process_issues/
│   │   ├── setup_project/
│   │   ├── job_failed/
│   │   └── check_stale_jobs/
│   └── python_common/      # Python-only helpers shared by Lambdas (signature checks, config, Lambda invoke client)
├── tests/                  # mirrors the contents of src/ (and scripts/)
├── slack/                  # app manifest template (derive from the current slack-bot-manifest.json)
├── infra/
│   ├── bootstrap/          # one-time: Terraform state bucket
│   └── terraform/
│       └── modules/        # lambda_function, failure_queue, schedule, http_api, tables, ssm_parameters
├── scripts/                # build and packaging
└── README.md
```

## 9. Security

- **Request verification.** Every public route verifies its signature over the raw request body before doing any work: Slack's signing secret plus timestamp window on `/slack/*`, the Standard Webhooks signature on `/webhook`. Unverified requests never reach an LLM.
- **Least-privilege IAM per Lambda.** For example, `issues-generated` can read only the webhook secret and invoke `process-issues`; `summary-posted` can read only the Slack signing secret and invoke `process-summary`; only `process-summary`, `process-issues` and `check-stale-jobs` can read the OpenAI key; only `process-issues` can read the judge and embeddings keys and the `ticket_embeddings` table; only `job-failed` can write `failed_jobs`.
- **Secrets** live in SSM Parameter Store (SecureString). Terraform creates the parameters with placeholder values and ignores later changes, so real values are set out of band and never enter Terraform state.
- **No stored transcripts.** DynamoDB holds IDs, status, failure reason codes, and (in `ticket_embeddings`) vectors and hashes derived from ticket titles and summaries, which already live in GitHub. It never holds transcript or summary text. Embeddings can be partly inverted back into text; that is an accepted risk, since the source text is ticket content rather than transcripts. Transcript text is never logged. On the manual path the summary text travels in the invocation payload, which Lambda holds for at most the 10-minute maximum event age. If the run fails for good, Lambda copies that payload into `failed-events`. `job-failed` deletes the message once handled and never copies the payload into `failed_jobs`, but a message it cannot handle stays in the queue for the default 4-day retention. That short-lived copy is an accepted exception.
- **OpenAI retention.** The background response is created with `store=true`, so OpenAI keeps the response, including the summary input, for at least 30 days. With `store=false` it would be deleted after about 10 minutes, but `process-issues` must be able to retrieve it whenever the webhook arrives, and `check-stale-jobs` checks it 30–45 minutes after the job started (§5.6). Robust retrieval was chosen over shorter retention. "No stored transcripts" still holds for our own storage (DynamoDB and logs). Background mode is also not ZDR-compatible. Both are acceptable today (no retention requirement), and are one reason generation sits behind the `TicketGenerator` interface.
- **Draft-only output.** Tickets are created as draft items for human review. The LLM has no tools and its output is schema-constrained, which limits the impact of prompt injection from meeting content or existing tickets.
- **GitHub token scope.** The classic PAT needs `repo` and `project` scopes, which is broad. A dedicated bot account limits the blast radius. Because users are trusted, any workspace member can link a channel to any project the bot can access. This is an accepted limitation of the trusted-organisation model.

## 10. Reliability and failure modes

| Failure | Handling |
|---|---|
| Slack retries or posts the summary twice | Both trigger `process-summary`. Its conditional claim drops the second once the first has moved the job on; if the second arrives while the first still holds its claim lease, it raises `LeaseHeld` and Lambda retries it, and the retry is then dropped (or resumes the job if the first run died). |
| Slack 3-second deadline | The Slack routes only verify and hand off (or open the modal). |
| Async invoke is rejected at a Slack route or `/webhook` (for example throttled) | Return 5xx; the sender retries. 2xx is returned only after Lambda accepts the invocation. |
| Duplicate OpenAI webhook delivery (at-least-once) | `process-issues`'s conditional transition drops duplicates. |
| OpenAI delivers the webhook late, or never | `check-stale-jobs` checks the response 30–45 minutes after the job started: it recovers a completed one through `process-issues`, and fails the job otherwise (§5.6). A webhook arriving after the job failed is dropped. |
| Generation fails, is cancelled or incomplete | The webhook subscribes to these events too; `process-issues` sends the failure to `failed-events`, and `job-failed` tells the thread. |
| A worker meets a permanent error (for example a revoked GitHub token) | The worker sends it to `failed-events` and returns; `job-failed` fails the job, logs it and tells the user once (§5.5). |
| A worker crashes, times out or keeps hitting retryable errors | Lambda retries (two retries, within the 10-minute maximum event age). Steps are idempotent: a retry's title check skips drafts an earlier attempt created. After that, the invocation record goes to `failed-events` and on to `job-failed`. |
| `job-failed` itself fails | SQS redelivers the message until it succeeds or the 4-day retention expires. No dead-letter queue or alarm: failures are not critical, as users can rerun. A failed Slack post is logged in `failed_jobs` (`message_sent = false`) and not retried. |
| Channel has no project configured | Reply asking the user to run `/setup` (before any LLM call). The job completes; it is not a failure. |
| Duplicate judge unavailable | **Fail open** per request: the affected pairs count as not duplicates, other tickets keep their verdicts, and the thread notes it. Drafts are reviewed anyway. |
| Embeddings API unavailable | **Fail open:** no shortlist or judge calls; only exact-title duplicates are dropped, and the thread notes it. |
| `ticket_embeddings` unavailable | Treated as an empty cache: everything is embedded and nothing is written. The run is not failed. |

Alert routing is deferred; `failed_jobs` is the failure log.

## 11. Cost

Estimates at list prices (verify before relying on them); excludes LLM tokens.

| Item | Estimate |
|---|---|
| Infrastructure per run (a few Lambda-seconds, DynamoDB, API Gateway) | ≈ $0.025 per 1,000 runs |
| Fixed monthly infrastructure | ≈ $0 (free tiers, standard SSM parameters, on-demand DynamoDB, small log volume, a Terraform state bucket holding a few kilobytes) |
| HTTP API | ≈ $1 per million requests |
| Stale-job schedule | ≈ $0: EventBridge Scheduler's free tier is 14 million invocations a month (then $1 per million); every 15 minutes is about 2,900. The scans and occasional OpenAI checks are a few Lambda-seconds each |
| Failure queue | ≈ $0: a few SQS requests per failure |
| Embeddings (counted here, though they are LLM tokens) | Negligible: a ticket's embed text is about 100 tokens, and the cache means only new or edited tickets are embedded. The cache's storage is about 1 KB per item at 256 dimensions |

Deliberately avoided: an Application Load Balancer (roughly $16+/month idle), a VPC with a NAT gateway (roughly $32+/month), and Secrets Manager (about $0.40 per secret per month, with rotation we cannot use because none of our providers support it). Set explicit CloudWatch log retention to avoid unbounded log growth.

## 12. Infrastructure and deployment

- **Terraform** with remote state in S3. Recent Terraform versions support S3-native state locking, removing the need for a DynamoDB lock table (confirm the minimum version when implementing). A one-time `bootstrap` stack creates the state bucket.
- **Modules:** `lambda_function` (role, log group with retention, function, optional async-invoke settings: two retries, a 10-minute maximum event age and the `failed-events` destination), `failure_queue` (the queue with default retention, no dead-letter queue, and the event source mapping to `job-failed`), `schedule` (EventBridge Scheduler, every 15 minutes, to `check-stale-jobs`), `http_api`, DynamoDB tables, SSM parameters.
- **Build:** a script builds Python zips (and later Rust binaries) before `terraform apply`; CI takes over at the CI/CD stage.

### First deployment (order matters)

1. Apply `bootstrap`, then the main stack. Outputs include the API base URL.
2. Create the Slack app from the manifest template **without** request URLs; install it; copy the signing secret and bot token.
3. Set all secret SSM values out of band (CLI).
4. Set the Slack request URLs: the Events URL and the `/meetingsummary` command URL point to `/slack/summary`; the `/setup` command URL and the Interactivity URL point to `/slack/newproject`. Slack verifies the Events URL, so `summary-posted` must answer the URL-verification challenge and its secrets must already exist.
5. Register the webhook URL in the OpenAI dashboard, subscribe to the `response.*` terminal events, and store the webhook secret in SSM.
6. Create the GitHub bot account and PAT; invite the bot to the channels, projects and repos to connect.

Steps 2, 5 and 6 are manual and belong in the README.

The current `slack-bot-manifest.json` already carries the bot scopes (`canvases:read`, `channels:history`, `channels:read`, `chat:write`, `commands`, `files:read`, `groups:history`, `groups:read`), the `file_shared` event and both slash commands, all with empty URLs. The target manifest additionally needs interactivity enabled (for the setup modal submission) and its request URL, and its `long_description` should stop referring to n8n.

## 13. Observability and testing

- **Logging:** structured JSON with `job_id` as the correlation ID. Python uses AWS Lambda Powertools (Logger, Parameters, Metrics); Rust will use `tracing`. Idempotency is hand-rolled via conditional writes (portable) rather than Powertools' idempotency utility.
- **Failure log:** every failure is a permanent row in `failed_jobs` (§6), with its reason and whether the user was told. There are no alarms for now; alert routing is deferred.
- **Testing:** unit tests with fakes behind the interfaces in §7; contract tests against the shared fixtures; signature test vectors; real-AWS integration tests are defined at the CI/CD stage.

## 14. Delivery roadmap

1. **Python on AWS.** Implement the eight Lambdas against the contracts. Recommended: deploy a walking skeleton early (`summary-posted` answering Slack's challenge) to validate the Slack and OpenAI handshakes before the full logic exists. Cut over from n8n using a test channel and project.
2. **Terraform.** Modules and first-deployment procedure as in §12.
3. **Rust.** Suggested order: `issues-generated` (smallest, a few-seconds deadline), then `summary-posted` (3-second deadline, where cold starts matter) and `project-added`, then the workers.
4. **CI/CD.** Build, test, deploy pipeline; separate environments considered here.
5. **UI.** HTTP API supports JWT authorizers for when it is needed.
6. **Maintain, optimise, add features.**

## 15. Decision log

| # | Decision | Alternatives considered | Rationale |
|---|---|---|---|
| 1 | Lambdas + DynamoDB | Always-on EC2 | Pay per use; near-zero idle cost |
| 2 | API Gateway HTTP API; no ALB, no VPC | ALB, REST API, VPC | Cheapest; security comes from request verification and IAM, not networking |
| 3 | Background mode + webhook for the slow LLM call | Synchronous worker; polling; Step Functions; Batch API | Lowest per-run cost (no idle billing); accepted trade-offs: not ZDR-compatible, manual webhook registration. A lost or late event is covered by the stale-job check (decision 30) |
| 4 | SSM Parameter Store | Secrets Manager | Free; our secrets cannot use automatic rotation |
| 5 | Language-neutral contracts; Powertools in the Python phase | Shared Python library | Enables incremental Rust port |
| 6 | Separate, calibrated duplicate decision | Duplicates judged inside generation | Cheaper; tunable threshold; vendor-pluggable |
| 7 | Bot account with classic PAT | GitHub App; OAuth app | Only option that reaches user-owned Projects v2 |
| 8 | Self-hosted per organisation, one environment, `us-east-1` | Multi-tenant SaaS | Simplest model; cheapest region |
| 9 | Hand-rolled idempotency via conditional writes | Powertools idempotency utility | Portable to Rust |
| 10 | Modal-based `/setup` | Command arguments | Self-documenting; extensible to a richer Slack form |
| 11 | One Lambda per public endpoint (`summary-posted`, `project-added`, `issues-generated`) | One Lambda routing all endpoints | Different secrets and permissions per route; independent deployment, failure isolation and Rust porting |
| 12 | Routes grouped by function: `/slack/summary` (file events and `/meetingsummary`), `/slack/newproject` (`/setup` and its modal), `/webhook` | Routes per Slack payload type (events, commands, interactivity) | One purpose and one handler per route; entry points that share logic share an endpoint |
| 13 | Asynchronous Lambda invocation for hand-offs, with one shared on-failure destination (`failed-events`) | An SQS queue plus DLQ per consumer; HTTP calls between Lambdas; self-invocation | A Lambda stops when it returns, so post-response work needs its own invocation. Async invoke gives automatic retries and failure capture with far fewer resources. A visible backlog and per-hop concurrency caps are not needed at this scale (revisit if they become necessary, see §16) |
| 14 | OpenAI's Decisions API as the duplicate judge | Jev; a plain LLM prompt | Per-pair probabilities with a tunable threshold; same vendor and account as generation |
| 15 | Embedding shortlist before the judge: top K (10) above a minimum cosine similarity (0.5), one predicate per shortlisted candidate in one request per new ticket | Judge every candidate; one combined "duplicate of any of these?" predicate; lexical similarity (BM25, token overlap) | Bounds the judge's input regardless of project size; keeps per-candidate probabilities and the matched ticket's ID; embeddings catch reworded duplicates that lexical methods miss |
| 16 | Embedding cache in DynamoDB (`ticket_embeddings`), keyed by project and item, validated by a hash of the embed text plus model and dimensions; 180-day TTL refreshed once under 90 days remain; rows for items no longer in the project are deleted after a complete fetch | No cache; S3 object per project; GitHub `updatedAt` for staleness | DynamoDB as the single source of truth for state. The hash re-embeds only real title or summary edits, while `updatedAt` also changes on status, priority and assignee edits. The cache is disposable, so expiry is aggressive |
| 17 | Every draft body starts with a `## Summary` section, and duplicate detection matches exactly that heading; items without it are compared by title alone | Parse several heading styles; compare the whole body | One schema going forward. Exact round-tripping keeps the cache's hashes stable for drafts we create |
| 18 | Judge new tickets in parallel, then resolve in extraction order (the later of two duplicate new tickets is dropped); no overall time budget | Sequential judging against earlier kept tickets, under a time budget | Shortlists keep each request small, so per-request timeouts bound the run without a budget, and parallel requests cut latency |
| 19 | Only Issues and DraftIssues are read from a project; every other item type (pull requests, any future type, read as a plain string so it cannot fail the listing) and archived items are skipped | All project items | A PR is usually the work on a ticket, not a ticket. Archived items are treated as gone |
| 20 | A separate SSM secret for the embeddings API key | Reuse the OpenAI or judge key | Can be rotated, scoped and billed separately |
| 21 | Lambdas named for the event they handle or the work they do: `summary-posted`, `project-added`, `issues-generated` (entry points), `process-summary`, `process-issues`, `setup-project` (workers), and `job-failed`, `check-stale-jobs` (failure handling) | `slack-summary`, `slack-newproject`, `webhook`, `starter`, `finisher`, `newproject-worker` | Names say what happens, not which vendor calls in or which step it is |
| 22 | Every schema is named `<sender>_to_<receiver>` (§8), including storage and third-party payloads | Descriptive names (`job`, `judge_request`, `ticket_extraction`); names per payload (`start_work`, `finish_work`) | One rule for every contract; the file name always says who writes and who reads |
| 23 | Guarantee the `## Summary` round trip in the extraction schema: `summary` is one line not starting with whitespace or `#`, and `body` starts with `#` | Cache a new draft under the embed text parsed back from its own body; no constraint | The model's structured output enforces the rule, and the cached hash equals what the next run computes from GitHub |
| 24 | Cap the embed text's title (256 code points) and summary (1,500 code points) inside the shared embed-text rules | Truncate only the embeddings API input; accept over-long inputs and fail open | One over-long item can never disable the shortlist for its project, and the hash, the judge input and the cache all use the same capped text |
| 25 | No length limit on the extracted ticket's `title` in the schema | `maxLength: 256`; a length-limiting `pattern` | Strict structured outputs do not support `maxLength`; embed texts already cap titles, and GitHub rejects longer titles anyway |
| 26 | The OpenAI Response envelope has no JSON Schema; only the tickets inside it do | One schema holding the envelope with the ticket format as a `$def` | The ticket schema is sent to OpenAI verbatim as the structured-output format and strict mode rejects unsupported keywords, so it must stay a plain root schema. The envelope is OpenAI's own shape, validated by pydantic (`ResponseEnvelope`). The request that carries the format does have a schema, `process_summary_to_ticket_generator`, whose fixtures refer to the ticket schema instead of copying it |
| 27 | One failure path: workers send permanent errors to `failed-events` and Lambda's on-failure destination sends exhausted ones there; only `job-failed` fails jobs and messages users | Each worker fails its own jobs and posts; an on-failure destination that invokes a Lambda directly; a queue that holds failures for manual replay | One place to fail jobs and word messages; the queue redelivers if `job-failed` errors; known and unexpected failures look the same to users and in the log |
| 28 | `failed_jobs` is a permanent failure log (no TTL), keyed by job ID (or request ID for `setup-project`); the message is sent only when the row is first created, and `message_sent` records the outcome; no dead-letter queue, no alarms, default 4-day queue retention | A short queue retention; a DLQ and alarm on the queue; retrying failed Slack posts | Failed runs are not critical: users can rerun. Creating the row first makes a redelivered message post nothing, and a lost post is logged rather than retried |
| 29 | Drafts are created in one GraphQL request, and partial success is not tracked anywhere: any error fails the run, and a retry's title check skips drafts already created | One request per draft, recording each created item ID on the job (`created_item_ids`) | Less code and state. The cost of a partial success is one retry. Rate limits are deliberately left as ordinary failures for now |
| 30 | `check-stale-jobs` on EventBridge Scheduler every 15 minutes: any job neither `completed` nor `failed` 30 minutes after `created_at` is recovered (a completed `generating` response is finished) or failed | Accepting the lost-webhook risk; a daily check at 24 hours; a 15-minute cutoff | Covers lost webhooks and lost failure events promptly at no cost. 30 minutes clears each stage's retries under the 10-minute maximum event age (decision 31), so a pending Lambda retry is not failed under it; Lambda offers no way to ask whether an event is still queued for retry |
| 31 | Worker async settings: two retries, maximum event age 10 minutes | Defaults (two retries, 6 hours) | Bounds how long a job can be retrying, which the 30-minute stale cutoff relies on, while still riding out brief 429s and 5xxs |

## 16. Deferred, open items and known issues

### Deferred (out of scope for this document)

- Alert routing (for example on `failed_jobs` growth)
- AWS tuning: reserved concurrency, budgets and alarms, throttling
- GitHub rate-limit handling (today a rate limit is an ordinary failure)
- The async payload size limit versus the longest manual summary (verify the current limit)
- Reintroducing SQS between hops if a visible backlog or per-hop concurrency caps are needed
- Separate environments, CI/CD, UI
- Showing suppressed duplicates to users
- Zero Data Retention support (would require synchronous generation)
- Organisation-owned projects and several repos per project
- Per-channel duplicate thresholds

### Open items to confirm

- **Shortlist minimum similarity.** 0.5 is a starting guess. A reworded duplicate can score below it, especially at 256 dimensions, and the judge never sees a candidate the shortlist drops. Check on the labelled pairs that true duplicates clear it and land in the top K, and lower it if not.
- **On-failure records:** confirm the record's shape and `condition` values, and that the request ID it carries matches `context.aws_request_id` across retries (`setup-project` failures are keyed by it).

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
- **Partial failures.** Drafts are created one at a time; a failure partway is reported only as a count ("N tickets encountered an error"), with no record of which were created, so a retry can duplicate them. In the target, a retry's title check skips drafts already created (§7).
- **Error handler coverage.** *transcription-error-handler* can reply in the thread only if *Get Upload Message Timestamp* ran, i.e. the huddle path after file lookup; other failures go to the channel root. It also emails via Gmail. The target handles every failure in `job-failed` (§5.5).

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
- **On-failure destination:** where Lambda sends a record of an asynchronous invocation that failed for good (the original payload, the error type and why it gave up).
- **SQS:** a message queue. Here it is only the `failed-events` queue, which carries every failure to `job-failed`.
- **EventBridge Scheduler:** AWS's managed cron service; it invokes `check-stale-jobs` every 15 minutes.
- **Idempotent:** safe to run more than once with the same effect as running once.
- **Conditional write:** a DynamoDB write that succeeds only if a condition holds (for example "this row does not exist yet"); the basis of our dedupe and state transitions.
- **GSI:** global secondary index, which lets DynamoDB look up rows by an attribute other than the primary key.
- **TTL:** a DynamoDB feature that automatically deletes rows after a timestamp.
- **Embedding:** a fixed-length list of numbers representing a text, such that texts with similar meaning have nearby vectors.
- **Cosine similarity:** a score of how closely two vectors point the same way; for embeddings, a measure of how similar two texts are in meaning.
- **Webhook:** an HTTP call a service makes to you when something happens.
- **Standard Webhooks:** the signature scheme OpenAI uses (`webhook-id`, `webhook-timestamp`, `webhook-signature` headers).
- **SSM Parameter Store:** AWS's configuration and secrets store; SecureString parameters are KMS-encrypted.

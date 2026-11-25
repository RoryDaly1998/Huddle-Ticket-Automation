# Python Refactor Plan

The Python rewrite is built: contracts, the common library, all six original Lambdas, duplicate detection, the judge eval script and zip packaging. The next piece of work is the **failure-handling rework** (section 1), which adds two Lambdas and removes a fair amount of code. This file holds only what the design docs don't:

- the **failure-handling rework**, with the **dead code** it creates,
- the **open items**, and
- the **infra contract** that the Terraform phase must meet.

Design decisions live in [architecture.md](architecture.md) (failures: §5.5 and §5.6, decisions 27–31). Every change keeps the checks in CLAUDE.md green and adds unit, contract and test-vector tests as it goes.

## 1. Failure-handling rework

### 1.1 What changes

One path reports every failure. Workers stop failing jobs and stop posting failure messages; a new `job-failed` Lambda does both, once, and logs every failure permanently in a new `failed_jobs` table. A new `check-stale-jobs` Lambda, run every 15 minutes by EventBridge Scheduler, recovers or fails jobs that never finished. Drafts are created in one GraphQL request, and partial success is no longer tracked anywhere.

| Failure | How it reaches `job-failed` |
|---|---|
| Permanent error a worker recognises (`python_common.failures.failure_code`) | The worker sends it to the `failed-events` queue and returns normally |
| Retryable error that outlasts Lambda's retries, a timeout or a crash | Lambda's on-failure destination sends the invocation record to the same queue |
| Job not `completed` or `failed` 30 minutes after `created_at` | `check-stale-jobs` sends it to the same queue (after trying to recover a completed generation) |

### 1.2 Work by component

**Contracts** (first, so code and Rust ports share them):

- New `lambdas_to_job_failed`: what our Lambdas send to the queue. Fields: `id` (job ID, or the async request ID for `setup-project`), `sender` (`process-summary`, `process-issues`, `setup-project`, `check-stale-jobs`), `reason` (a short code), `channel_id`, optional `thread_ts`. Never summary text.
- New `lambda_to_job_failed`: the on-failure destination record the Lambda service sends. Received data, so describe only the fields read (`requestContext.requestId`, `requestContext.condition`, `requestContext.functionArn`, `requestPayload`, `responsePayload.errorType`).
- New `job_failed_to_db_failed_jobs`: a `failed_jobs` row (section 3).
- `lambdas_to_db_jobs`: remove `created_item_ids`.
- Rename `issues_generated_to_process_issues` to `lambdas_to_process_issues` (and the model `IssuesGeneratedToProcessIssues` to `LambdasToProcessIssues`), since `check-stale-jobs` now sends it too.
- `lambdas_to_github` and `github_to_lambdas`: replace the single-draft `add_draft_issue_*` defs with a batch create, one aliased `addProjectV2DraftIssue` per ticket, titles and bodies passed only as variables.
- Update `contracts/schemas/architecture.drawio` to match the Mermaid diagram in `contracts/schemas/README.md`: the queue feeding `job-failed`, the `failed_jobs` table, and `check-stale-jobs` with its schedule.
**Common library** (`src/python_common/`):

- `FailureQueue` interface, an SQS adapter and a fake. `send()` raises on any failure, so the caller can release its lease and re-raise for Lambda's retry.
- `FailedJobStore` interface, a DynamoDB adapter and a fake: `create(row) -> bool` (a conditional put on `attribute_not_exists(id)`; `False` if the row exists) and `mark_message_sent(id)`.
- `JobStore.mark_failed(job_id, error)` replaces the status-specific version: one `UpdateItem` conditioned on `attribute_exists(job_id) AND status <> completed`, with `ReturnValuesOnConditionCheckFailure=ALL_OLD` to tell a completed job (skip) from a missing one (carry on). Returns the updated job.
- `JobStore.find_stale(cutoff)`: a `Scan` for jobs in `received`, `generating` or `finishing` with `created_at` before `cutoff`. Fine at this scale.
- `GitHubClient.add_draft_issues(project_id, drafts) -> list[str]`: one request, item IDs in ticket order. Any GraphQL error raises the usual `GitHubApiError` or `GitHubUnavailable`, whichever applies, even if some drafts were created.
- `Config`: `failure_queue_url()` and `failed_jobs_table()`.
- Move `is_huddle_summary` (now in `lambdas/process_summary/huddle_file.py`) where `job-failed` can import it too.
- `failures.py`: rewrite the module docstring. Permanent errors are now sent to the queue, not used to fail the job.

**`process-summary`:**

- A permanent error sends `{id: job_id, sender, reason, channel_id, thread_ts}` to the queue and returns. If the send fails, release the claim and re-raise.
- Retryable errors are unchanged: release the claim, re-raise.
- No more `mark_failed`, Slack failure posts or `is_summary` tracking. The quiet exit for non-summary files and the `/setup` hint stay.

**`process-issues`:**

- `response.failed`, `.cancelled` and `.incomplete` events, and a retrieved response that did not complete, send `generation_<status>` to the queue.
- A permanent error sends its code to the queue. Retryable errors and failed sends release the lease and re-raise.
- Drafts come from one `add_draft_issues` call. A retry's title check skips drafts an earlier attempt created, which then show in the reply as duplicates (accepted).
- The completion reply lists created tickets and duplicates only.

**`setup-project`:**

- A `GitHubApiError` sends `{id: context.aws_request_id, sender: "setup-project", reason: "github_<code>", channel_id}` to the queue instead of posting.
- Fixable problems (invalid URL, project not found or read-only, repository not accessible) are still answered by the worker. A permanent Slack error posting the result is still only logged, as the link is already stored.

**`job-failed` (new, triggered by the `failed-events` queue):**

1. Parse the message as either contract. For an invocation record, take `id` from the job ID in `requestPayload` (`setup-project`: `requestContext.requestId`) and make the reason `<condition>:<errorType>`. If `errorType` is `LeaseHeld`, log and stop, because another run owns the job.
2. For a job, find it (by `job_id`, or by `response_id` through the GSI for a `process-issues` payload) and `mark_failed` it. Stop if it is `completed`. Carry on if there is no job row, except for a `process-issues` record whose `response_id` matches no job: with no `id` or channel, log it and stop.
3. `create` the `failed_jobs` row with `message_sent = false`. Stop if it already exists, so a redelivered message never posts twice.
4. For a huddle job with no `response_id` (it never reached `generating`), or with no row, check the file with `files.info` and `is_huddle_summary`. Post nothing unless it is confirmed.
5. Post once: in the thread if `thread_ts` is known, otherwise in the channel. Summary failures say to check the project, then rerun `/meetingsummary` (reposting a huddle file is dropped, because its `job_id` is taken). Setup failures say to run `/setup` again.
6. On success set `message_sent = true`. A Slack error is logged and the message is not retried.

**`check-stale-jobs` (new, EventBridge Scheduler, `rate(15 minutes)`):**

- For each job from `find_stale(now - 30 minutes)`:
  - `generating` whose response has completed: invoke `process-issues` with `{response_id, event_type: "response.completed"}`;
  - otherwise queue a failure: `generation_<status>` if OpenAI ended the response, or `stale_<status>` if it is still running, the job is `received` or `finishing`, or the status check itself fails.
- A pass that times out is finished by the next one. Re-queued failures are harmless, because `job-failed` stops at an existing row.
- The 30-minute cutoff depends on the 10-minute maximum event age (section 3). Change them together.

**Build:** each Lambda is a uv workspace member, so give `job_failed` and `check_stale_jobs` their own `pyproject.toml` under `src/lambdas/`, add them to `[tool.uv.sources]` in the root `pyproject.toml`, and refresh `uv.lock` (`build.py`'s `check_lock` fails otherwise). Add both to `LAMBDA_NAMES` in `scripts/build.py`. Neither needs resource files.

### 1.3 Dead code this creates

Delete these with their tests and fixtures once the replacements above exist.

| Where | What | Why |
|---|---|---|
| `src/lambdas/process_summary/handler.py` | `fail_job`; the `notify`/`is_summary` logic in `run_claimed_job`; the `FAILURE_MESSAGE` import; the failure-posting part of the module docstring | `job-failed` fails jobs and posts |
| `src/lambdas/process_summary/messages.py` | `FAILURE_MESSAGE` | Moves to `job-failed` |
| `src/lambdas/process_issues/handler.py` | `fail_generation`, `fail_job`, `mark_failed`, `recorded_item_ids`; the `earlier_drafts`, `failed` and `failure_codes` fields of `log_completion`; the failure part of the module docstring | Failures go to the queue; no recorded drafts or per-ticket failures |
| `src/lambdas/process_issues/messages.py` | `FAILURE_MESSAGE`, `FAILURES_HEADING`, `FAILURE_LINE`, the refused-tickets section of `reply_sections` | No per-ticket failures; failure message moves |
| `src/lambdas/process_issues/drafts.py` | `DraftFailure`; `DraftResults.failures`; the created-versus-new split (`DraftResults.created` against `new_drafts`); the per-ticket loop and its `append_created_item_id` call; the module docstring | One batch request, nothing recorded |
| `src/lambdas/process_issues/dedupe.py` | `recorded_drafts`; the `created_item_ids` parameter of `plan_tickets`; `KeptTicket` (left wrapping only a `Ticket`); step 1 "Already created" and the recorded-items note in the docstring | No `created_item_ids` |
| `src/lambdas/setup_project/handler.py` | `post_failure`; the `FAILURE_MESSAGE` import; the failure-message sentence of the module docstring | Failure goes to the queue |
| `src/lambdas/setup_project/messages.py` | `FAILURE_MESSAGE` | Moves to `job-failed` |
| `src/python_common/interfaces.py` | `JobStore.append_created_item_id`; `FailableStatus`; the `current`-status form of `JobStore.mark_failed`; `GitHubClient.add_draft_issue`; the `JobStore` docstring's "any non-final status to failed" | Replaced as in 1.2 |
| `src/python_common/stores/job_store.py` | `append_created_item_id`; `JobChanges.created_item_id`; the current `mark_failed` | As above |
| `src/python_common/fakes/job_store.py` | `append_created_item_id`; the current `mark_failed` | As above |
| `src/python_common/models.py` | `Job.created_item_ids` | No recorded drafts |
| `src/python_common/clients/github.py` | `ADD_DRAFT_ISSUE_MUTATION`, `AddDraftIssueData`, `add_draft_issue` | Replaced by the batch create |
| `src/python_common/fakes/github.py` | `add_draft_issue`, `FORBIDDEN_CODE` (if nothing else uses it) | As above |
| `contracts/` | `created_item_ids` in `lambdas_to_db_jobs` with its fixtures (`finishing_with_created_item_ids`, `empty_created_item_ids`, `created_item_ids_not_array`, `empty_created_item_id`, and the field in `completed` and `finishing_with_lease`); the `add_draft_issue_*` defs and fixtures in `lambdas_to_github` and `github_to_lambdas` | As above |
| Tests | Failure-posting and `mark_failed` cases in the three worker `test_handler.py` files; recorded-draft cases in `test_dedupe.py` and `test_drafts.py`; refused-ticket cases in `test_messages.py`; `append_created_item_id` and old `mark_failed` cases in both `test_job_store.py` files; `add_draft_issue` cases in both `test_github.py` files; `created_item_ids` cases in `test_models.py` and the field in `tests/python_common/helpers.py` | Code removed |

These remain in use: `failure_code` and `GENERATION_FAILURE_PREFIX` (they become queued reasons), `release_claim` and `release_finishing` (retryable errors), `InvalidTransition` and `AlreadyClaimed` handling (dropping duplicates and retries of failed jobs), `log_permanent_error` in `setup-project` (the Slack result post), and `invoke_async` (used by `check-stale-jobs`).

### 1.4 Tests to add

- Each job worker: a retry or duplicate delivery against a `failed` job does nothing (no Slack post, no generation, no drafts). No worker moves a job out of `failed`.
- Each worker: a permanent error sends one well-formed `lambdas_to_job_failed` message and returns; a failed send releases the lease and re-raises.
- `job-failed`: both message shapes; `LeaseHeld` skipped; completed job skipped; missing job row still logged; existing `failed_jobs` row posts nothing; non-summary or unconfirmable huddle file posts nothing; thread versus channel; Slack error leaves `message_sent = false` without raising; the row never holds `requestPayload` or summary text.
- `check-stale-jobs`: each stale status; completed response recovered; ended, running and unreadable responses failed; fresh jobs ignored.
- `add_draft_issues`: titles and bodies only in variables; IDs in order; a response with partial data and errors raises.

## 2. Open items

- **Real Slack payloads.**
  - Capture real `file_shared` events and the `files.info` and `canvases.getContent` responses, plus a slash command and a modal submission, as redacted fixtures. The fixtures in `contracts/fixtures/slack_to_summary_posted/`, `slack_to_project_added/` and `slack_to_lambdas/` are synthetic and doc-derived only; see their READMEs.
  - Confirm the huddle filter's `files.info` signals, and that `file_id` is stable across Slack's double post (the `job_id` choice depends on it).
- **Reproducible zips (user decision).**
  - Today the zips are byte-identical only when built on the same machine. Windows Python's zlib-ng compresses differently from Linux zlib. Also, with `core.autocrlf=true` and no `.gitattributes`, a Windows checkout has CRLF line endings, so a Windows-built `process-summary` sends a CRLF prompt to OpenAI.
  - Options: add `.gitattributes` (`* text=auto eol=lf`) and renormalise, and/or build deploy zips in one place only (later: CI).
- **Live API checks.** None of these have been run against the real services yet:
  - the real keys can reach the Decisions API (a 403 "not enabled" was reported during its beta) and the embeddings API;
  - OpenAI strict structured outputs accept the `ticket_generator_to_process_issues` patterns (`summary` and `body`). If the model returns text that breaks them, pydantic rejects it and the job fails with a generation error;
  - GitHub accepts `archivedStates: [NOT_ARCHIVED]` and leaves archived items out.
  - GitHub accepts a batch of aliased `addProjectV2DraftIssue` mutations for a typical meeting within its 10-second request timeout. Rate limits (5 points per mutating request, about 80 content-creating requests a minute) are deliberately not handled yet: hitting one is an ordinary failure.
- **AWS behaviour to confirm when deploying.**
  - The on-failure record's shape and `condition` values (`RetriesExhausted` is documented; the age-expiry value is not shown in the samples).
  - `context.aws_request_id` matches the record's `requestContext.requestId` across async retries, since `setup-project` failures are keyed by it.
  - The async payload size limit versus the longest manual summary.

## 3. Infra contract for Terraform

What the Python code expects from the infrastructure. Items marked *(rework)* apply once section 1 lands.

- **Artifacts.**
  - `uv run python scripts/build.py` writes `dist/<snake_case_name>.zip`.
  - Runtime **`python3.14`, arm64**; handler `lambdas.<snake_case_name>.handler.handler`.
  - boto3 is not in the zips; the runtime provides it.
  - `process-summary`'s zip carries `prompts/ticket_extraction_v1.md` and `contracts/schemas/ticket_generator_to_process_issues.schema.json` at its root. The other zips carry no resource files.
  - Zips are deterministic only when built on one setup (see section 2), so `source_code_hash` is stable only if the zips always come from the same machine.
- **Environment for all eight.**
  - `SSM_PREFIX`: parameters live at `<SSM_PREFIX>/<name>`.
  - `POWERTOOLS_SERVICE_NAME`: the Lambda's name.
  - Env vars that name functions accept anything `lambda:InvokeFunction`'s `FunctionName` accepts.
- **SSM parameters.** Each is read on first use with `ssm:GetParameter` (with decryption for SecureStrings; `kms:Decrypt` only for a customer-managed key). A role gets only the parameters in its row below.
  - SecureString: `slack-signing-secret`, `slack-bot-token`, `github-token`, `openai-api-key`, `openai-webhook-secret`, `judge-api-key`, `embeddings-api-key`.
  - String: `generation-model`, `judge-model`, `duplicate-threshold`, `embedding-model`, `embedding-dimensions`, `shortlist-top-k`, `shortlist-min-similarity`.
- **Per Lambda.** The SSM, DynamoDB and invoke/send columns are exactly what the code calls:

  | Lambda | Trigger | Env vars | SSM parameters | DynamoDB | Invokes or sends | Timeout |
  |---|---|---|---|---|---|---|
  | `summary-posted` | `POST /slack/summary` | `PROCESS_SUMMARY_FN` | `slack-signing-secret` | none | invoke `process-summary` | A few seconds; must answer Slack within **3 s**, cold start included |
  | `project-added` | `POST /slack/newproject` | `SETUP_PROJECT_FN` | `slack-signing-secret`, `slack-bot-token` | none | invoke `setup-project` | A few seconds; must answer within **3 s** (`views.open` is capped at 2 s) |
  | `issues-generated` | `POST /webhook` | `PROCESS_ISSUES_FN` | `openai-webhook-secret` | none | invoke `process-issues` | A few seconds; quick 2xx for OpenAI |
  | `process-summary` | async | `JOBS_TABLE`, `LINKS_TABLE`, `FAILURE_QUEUE_URL` | `slack-bot-token`, `openai-api-key`, `generation-model` | jobs: `PutItem`, `UpdateItem`; links: `GetItem`, `UpdateItem` | send to `failed-events` | **≤ 85 s** |
  | `process-issues` | async | `JOBS_TABLE`, `EMBEDDINGS_TABLE`, `FAILURE_QUEUE_URL` | `slack-bot-token`, `openai-api-key`, `github-token`, `judge-api-key`, `judge-model`, `duplicate-threshold`, `embeddings-api-key`, `embedding-model`, `embedding-dimensions`, `shortlist-top-k`, `shortlist-min-similarity` | jobs: `UpdateItem`, `Query` on `<jobs-table-arn>/index/response_id-index`; ticket_embeddings: `Query`, `BatchWriteItem` | send to `failed-events` | **≤ 175 s** |
  | `setup-project` | async | `LINKS_TABLE`, `FAILURE_QUEUE_URL` | `github-token`, `slack-bot-token` | links: `UpdateItem` | send to `failed-events` | A few GitHub and Slack calls at 10 s each |
  | `job-failed` *(rework)* | SQS `failed-events`, batch size 1 | `JOBS_TABLE`, `FAILED_JOBS_TABLE` | `slack-bot-token` | jobs: `UpdateItem`, `Query` on `response_id-index`; failed_jobs: `PutItem`, `UpdateItem` | none | **30 s** |
  | `check-stale-jobs` *(rework)* | EventBridge Scheduler, `rate(15 minutes)` | `JOBS_TABLE`, `PROCESS_ISSUES_FN`, `FAILURE_QUEUE_URL` | `openai-api-key` | jobs: `Scan` | invoke `process-issues`; send to `failed-events` | **120 s** |

  Before the rework, `process-issues` also needs jobs `GetItem`, and the three workers have no `FAILURE_QUEUE_URL`.

- **Async invoke settings** for `process-summary`, `process-issues` and `setup-project`: `maximum_retry_attempts = 2`, **`maximum_event_age_in_seconds = 600`**, on-failure destination `failed-events` (each role needs `sqs:SendMessage` on it, which the workers also use directly). Keep the event age at 10 minutes: the stale-job cutoff (30 minutes) assumes each stage is finished retrying within about 13 minutes. Revisit both together, and whenever a Lambda timeout grows.
- **Failure queue** `failed-events` *(rework)*: standard SQS, default retention (4 days), **visibility timeout 180 s** (at least six times `job-failed`'s timeout), no dead-letter queue. An event source mapping triggers `job-failed`, whose role needs `sqs:ReceiveMessage`, `sqs:DeleteMessage` and `sqs:GetQueueAttributes`. A message `job-failed` keeps failing on is redelivered until it expires.
- **Schedule** *(rework)*: an `aws_scheduler_schedule` with `rate(15 minutes)`, flexible time window `OFF`, targeting `check-stale-jobs` through a role allowed `lambda:InvokeFunction` on it.
- **Tables.**
  - `jobs`: key `job_id`, TTL on `ttl`, and GSI `response_id-index` (partition key `response_id`, projection `ALL`).
  - `failed_jobs` *(rework)*: key `id` (string), **no TTL**: it is the permanent failure log.
  - `channel_project_link`: key `channel_id`.
  - `ticket_embeddings`: partition key `project_id`, sort key `item_id`, TTL on `ttl`.
- **Timeouts and leases.**
  - Every client makes one attempt per call and never sleeps; Lambda's async retry does the retrying.
  - Per-call timeouts: OpenAI 30 s, GitHub 10 s per request (one request per 100 project items, and one request for all new drafts), Slack 10 s, judge 20 s per request (up to 8 concurrently), embeddings 20 s per batch.
  - Lambda timeouts must stay below the job leases (`CLAIM_LEASE` 90 s, `FINISHING_LEASE` 180 s), with a margin, so a lease never expires mid-run. Hence the 85 s and 175 s limits above.

# Python Refactor Plan

The code under `src/` implements the previous design (v0.5): eight Lambdas, hand-written pydantic models, shared fixtures and test vectors. This plan migrates it to [architecture.md](architecture.md) v0.6. It holds only what the architecture does not:

- the **work list**, in order (section 1),
- the **open items** (section 2), and
- the **infra contract** that the Terraform phase must meet (section 3).

Every step keeps `uv run ruff check . && uv run ruff format --check . && uv run mypy && uv run pytest` green.

## 1. Work list

### 1.0 Before starting

- **The user commits the current state.** `contracts/`, `src/`, `tests/` and `scripts/` are untracked, and step 1.2 deletes files there that git cannot restore. Claude does not commit.

### 1.1 Repository tidy

- Create `docs/` and move `architecture.md`, `python-refactor-plan.md` and the `.drawio` diagrams (from `contracts/schemas/`) into it. Update every link to them, including in `CLAUDE.md` and `README.md`.
- Delete the judge evaluation: `evals/`, `scripts/eval_judge.py`, `tests/scripts/test_eval_judge.py` and the eval parts of `tests/scripts/helpers.py`.
- `README.md` still describes the n8n setup; bring its routes and names in line once the Python stack replaces n8n.

### 1.2 Contracts

- Rename `contracts/` to `contract-schemas/` and move the schemas up one level into `internal/{lambda_to_lambda,lambda_to_db}` and `external/{slack,github,ticket_generator,embedder,duplicate_judge}`.
- Split every multi-call schema into one schema per call, named and placed as in architecture §8. Remove the `$def`-per-call convention and the `<def>__<case>.json` fixture naming.
- Add schemas that do not exist yet: the OpenAI create and retrieve envelopes (`receipt_id_to_process_summary`, `tickets_to_process_tickets`, `finished_responses_to_process_stuck_jobs`), the Slack replies (`url_verification_to_slack`, `summary_response_to_slack`, the shared empty-200 `confirm_receipt_to_slack`), and every third-party payload the code parses with a hand-written model today (about 48 classes in the handlers and clients).
- Keep the extracted-ticket format as its own plain root schema (it is sent to OpenAI verbatim); its name is open (section 2).
- Delete `contracts/fixtures/` and `contracts/vectors/`, and the contract-test helpers in `tests/helpers.py` that read them (`contract_validator`, `ecma_pattern`, `fixture_bytes`, `def_fixture_documents` and the like). Rewrite the tests that used them with inline data.
- Move `contracts/schemas/README.md` (already rewritten for the target) to `contract-schemas/` and drop its "Current state" section.
- Update the runtime schema path: `SCHEMA_PATH` in `src/python_common/clients/generator.py`, the resource list in `scripts/build.py` and the root lookup in `src/python_common/resources.py`.

### 1.3 Generated models

- Add `datamodel-code-generator` as a dev dependency and a `scripts/generate_models.py` that regenerates every model from `contract-schemas/` into one committed package (for example `src/python_common/contracts/`). Generated files are never edited by hand.
- Class names are schema names in PascalCase (`add_job_to_db_jobs` → `AddJobToDbJobs`); this replaces domain names such as `Job`, `Ticket` and `TicketEmbedding` throughout `src/` and `tests/`.
- A hand-written base class carries `strict=True`, `frozen=True` and `hide_input_in_errors=True`; `extra` follows each schema's `additionalProperties`.
- Move behaviour a schema cannot express into hand-written code beside the models:
  - the cross-field checks `check_response_fields` and `check_embedding_length` (`src/python_common/models.py`);
  - `embed_text`, `for_job`, `with_ttl`, `pack_vector`, `unpack_vector` and the base64 vector encoding (`Base64Bytes`);
  - keeping the service-error record's `requestPayload` out of `repr` (it can hold summary text).
- Check that pydantic's Rust regex engine reads the schemas' ECMA-style `pattern`s the same way (`$` means the end of the string); the old test shim existed for this.
- Running the generator is manual until CI/CD, which adds a pre-commit hook and a CI check that fail when regenerating changes the committed models.

### 1.4 Lambdas

- **Merge** `summary_posted` and `project_added` into `slack_webhook`, with the routes `/events`, `/meeting_summary`, `/setup` and `/modals` (architecture §3). Each route loads only what it needs; the bot token is loaded only for `file_shared` and `/setup` (the modal submission makes no Slack call).
- **Move the huddle check** from `process_summary` to `slack_webhook`: on `file_shared`, call `files.info`, hand off only confirmed huddle summaries (moving `is_huddle_summary` with it), and put the message's `ts`/`thread_ts` in `handoff_to_process_summary`. Drop the `USLACKBOT` pre-filter. `process_summary` then only calls `canvases.getContent` on the huddle path, and the "not a huddle summary" quiet exit goes.
- **Rename:** `issues_generated` → `ticket_generator_webhook`, `process_issues` → `process_tickets`, `job_failed` → `process_failed_events`, `check_stale_jobs` → `process_stuck_jobs`. Rename the matching env vars (`PROCESS_ISSUES_FN` → `PROCESS_TICKETS_FN`, `FAILED_JOBS_TABLE` → `FAILED_EVENTS_TABLE`), `Config` methods, logger service names, failure `sender` values, the `stale_<status>` reason codes (→ `stuck_<status>`), each Lambda's `pyproject.toml` and `[tool.uv.sources]` entry, `uv.lock`, and `LAMBDA_NAMES` in `scripts/build.py`.
- **Storage names:** the `failed_jobs` table becomes `failed_events_store` (`FailedJobStore` → `FailedEventStore`), `ticket_embeddings` becomes `embeddings`, and the `failed-events` queue becomes `failed_events_queue`.
- **Failure rules** (architecture §5.5):
  - Entry points send a handled error after verification as `runtime_error_to_process_failed_events` and answer 2xx; only unexpected or transient errors (including a rejected invoke or a failed `files.info`) answer 5xx. Both entry points need `FAILURE_QUEUE_URL` and a `FailureQueue`.
  - `process_failed_events` drops its `files.info` step (`needs_huddle_check`, `is_confirmed_huddle_summary`).
  - Parse the two message shapes with the generated `RuntimeErrorToProcessFailedEvents` and `ServiceErrorToProcessFailedEvents` models and normalise them into one failure.
- **`handoff_to_process_tickets`** is one schema for both senders (`ticket_generator_webhook` and `process_stuck_jobs`).

### 1.5 Last

- Update `contracts/schemas/schema_architecture.drawio` (by then in `docs/`) if anything above changed a contract, and delete `schema_architecture.backup.drawio` once the user is happy with the tidied diagram.
- Update the Slack app manifest: interactivity enabled, the four request URLs, no reference to n8n.

## 2. Open items

- **Real Slack payloads.** Confirm which `files.info` fields mark a huddle summary (the summary is populated before the file is posted, but the field values in `SlackFile` are still unconfirmed), and that `file_id` is stable across Slack's double post (the `job_id` choice depends on it).
- **Extracted-ticket format.** Its schema file needs a name under the `<operation>_to_<destination>` rule, and `tickets_to_process_tickets` should refer to it rather than copy it.
- **Terraform variables for settings (assumption to confirm).** Non-secret settings are Terraform variables that Terraform writes into the existing SSM String parameters, so the code keeps reading SSM. The alternative is passing them as env vars, which would change `Config`.
- **Reproducible zips (user decision).**
  - Today the zips are byte-identical only when built on the same machine. Windows Python's zlib-ng compresses differently from Linux zlib. Also, with `core.autocrlf=true` and no `.gitattributes`, a Windows checkout has CRLF line endings, so a Windows-built `process_summary` sends a CRLF prompt to OpenAI.
  - Options: add `.gitattributes` (`* text=auto eol=lf`) and renormalise, and/or build deploy zips in one place only (later: CI).
- **Live API checks.** None of these have been run against the real services yet:
  - the real keys can reach the Decisions API (a 403 "not enabled" was reported during its beta) and the embeddings API;
  - OpenAI strict structured outputs accept the extracted-ticket format's patterns (`summary` and `body`). If the model returns text that breaks them, the model rejects it and the job fails with a generation error;
  - GitHub accepts `archivedStates: [NOT_ARCHIVED]` and leaves archived items out;
  - GitHub accepts a batch of aliased `addProjectV2DraftIssue` mutations for a typical meeting within its 10-second request timeout. Rate limits (5 points per mutating request, about 80 content-creating requests a minute) are deliberately not handled yet: hitting one is an ordinary failure;
  - `files.info` plus the hand-off fits inside Slack's 3 seconds on a cold start.
- **AWS behaviour to confirm when deploying.**
  - The service-error record's shape and `condition` values (`RetriesExhausted` is documented; the age-expiry value is not shown in the samples).
  - `context.aws_request_id` matches the record's `requestContext.requestId` across async retries, since `setup_project` failures are keyed by it.
  - The async payload size limit versus the longest manual summary.
  - An OpenAI client can be created inside `process_stuck_jobs.zip`, the only zip with `openai` but without `httpx` and `certifi`. Only the imports have been checked, statically.
- **Messages `process_failed_events` can never handle (user decision).** An unrecognised function name, a service-error record whose `requestPayload` fails validation, or an `errorType` containing whitespace makes it raise, so SQS redelivers the message every 180 s until it expires. A job is still failed and reported later through `process_stuck_jobs`; a `setup_project` failure is never reported. Options: keep this, or log and drop such messages.
- **SQS send timeout (user decision).** The failure queue client keeps botocore's 60 s connect and read timeouts, so a hung send could outlast `process_summary`'s 85 s limit before the lease is released, and would blow an entry point's deadline. Entry points need a short send timeout.

## 3. Infra contract for Terraform

What the Python code will expect from the infrastructure once section 1 is done.

- **Artifacts.**
  - `uv run python scripts/build.py` writes `dist/<lambda_name>.zip`.
  - Runtime **`python3.14`, arm64**; handler `lambdas.<lambda_name>.handler.handler`.
  - boto3 is not in the zips; the runtime provides it.
  - `process_summary`'s zip carries `prompts/ticket_extraction_v1.md` and the extracted-ticket format schema from `contract-schemas/external/ticket_generator/` at its root. The other zips carry no resource files.
  - Zips are deterministic only when built on one setup (see section 2), so `source_code_hash` is stable only if the zips always come from the same machine.
- **Function names.** Deploy every Lambda under its name (`slack_webhook`, `ticket_generator_webhook`, `process_summary`, `process_tickets`, `setup_project`, `process_failed_events`, `process_stuck_jobs`): `process_failed_events` identifies the sender of a service error from the function ARN.
- **Environment for all seven.**
  - `SSM_PREFIX`: parameters live at `<SSM_PREFIX>/<name>`.
  - `POWERTOOLS_SERVICE_NAME`: the Lambda's name.
  - Env vars that name functions accept anything `lambda:InvokeFunction`'s `FunctionName` accepts.
- **SSM parameters.** Each is read on first use with `ssm:GetParameter` (with decryption for SecureStrings; `kms:Decrypt` only for a customer-managed key). A role gets only the parameters in its row below.
  - SecureString, written by the deploy workflow from GitHub Actions secrets (Terraform creates placeholders and ignores changes): `slack-signing-secret`, `slack-bot-token`, `github-token`, `openai-api-key`, `openai-webhook-secret`, `judge-api-key`, `embeddings-api-key`.
  - String, from Terraform variables: `generation-model`, `judge-model`, `duplicate-threshold`, `embedding-model`, `embedding-dimensions`, `shortlist-top-k`, `shortlist-min-similarity`.
- **Per Lambda.** The SSM, DynamoDB and invoke/send columns are exactly what the code calls:

  | Lambda | Trigger | Env vars | SSM parameters | DynamoDB | Invokes or sends | Timeout |
  |---|---|---|---|---|---|---|
  | `slack_webhook` | `POST /events`, `/meeting_summary`, `/setup`, `/modals` | `PROCESS_SUMMARY_FN`, `SETUP_PROJECT_FN`, `FAILURE_QUEUE_URL` | `slack-signing-secret`, `slack-bot-token` | none | invoke `process_summary`, `setup_project`; send to `failed_events_queue` | A few seconds; must answer Slack within **3 s**, cold start, `files.info` and `views.open` (capped at 2 s) included |
  | `ticket_generator_webhook` | `POST /tickets_generated` | `PROCESS_TICKETS_FN`, `FAILURE_QUEUE_URL` | `openai-webhook-secret` | none | invoke `process_tickets`; send to `failed_events_queue` | A few seconds; quick 2xx for OpenAI |
  | `process_summary` | async | `JOBS_TABLE`, `LINKS_TABLE`, `FAILURE_QUEUE_URL` | `slack-bot-token`, `openai-api-key`, `generation-model` | jobs: `PutItem`, `UpdateItem`; links: `GetItem`, `UpdateItem` | send to `failed_events_queue` | **≤ 85 s** |
  | `process_tickets` | async | `JOBS_TABLE`, `EMBEDDINGS_TABLE`, `FAILURE_QUEUE_URL` | `slack-bot-token`, `openai-api-key`, `github-token`, `judge-api-key`, `judge-model`, `duplicate-threshold`, `embeddings-api-key`, `embedding-model`, `embedding-dimensions`, `shortlist-top-k`, `shortlist-min-similarity` | jobs: `UpdateItem`, `Query` on `<jobs-table-arn>/index/response_id-index`; embeddings: `Query`, `BatchWriteItem` | send to `failed_events_queue` | **≤ 175 s** |
  | `setup_project` | async | `LINKS_TABLE`, `FAILURE_QUEUE_URL` | `github-token`, `slack-bot-token` | links: `UpdateItem` | send to `failed_events_queue` | A few GitHub and Slack calls at 10 s each |
  | `process_failed_events` | SQS `failed_events_queue`, batch size 1 | `JOBS_TABLE`, `FAILED_EVENTS_TABLE` | `slack-bot-token` | jobs: `UpdateItem`, `Query` on `response_id-index`; failed_events_store: `PutItem`, `UpdateItem` | none | **30 s** |
  | `process_stuck_jobs` | EventBridge Scheduler `check_for_stuck_jobs`, `rate(15 minutes)` | `JOBS_TABLE`, `PROCESS_TICKETS_FN`, `FAILURE_QUEUE_URL` | `openai-api-key` | jobs: `Scan` | invoke `process_tickets`; send to `failed_events_queue` | **120 s** |

- **Async invoke settings** for `process_summary`, `process_tickets` and `setup_project`: `maximum_retry_attempts = 2`, **`maximum_event_age_in_seconds = 600`**, on-failure destination `failed_events_queue` (each role needs `sqs:SendMessage` on it, which the workers also use directly). Keep the event age at 10 minutes: the stuck-job cutoff (30 minutes) assumes each stage is finished retrying within about 13 minutes. Revisit both together, and whenever a Lambda timeout grows.
- **Failure queue** `failed_events_queue`: standard SQS, default retention (4 days), **visibility timeout 180 s** (at least six times `process_failed_events`'s timeout), no dead-letter queue. An event source mapping triggers `process_failed_events`, whose role needs `sqs:ReceiveMessage`, `sqs:DeleteMessage` and `sqs:GetQueueAttributes`. A message it keeps failing on is redelivered until it expires.
- **Schedule**: an `aws_scheduler_schedule` named `check_for_stuck_jobs` with `rate(15 minutes)`, flexible time window `OFF`, targeting `process_stuck_jobs` through a role allowed `lambda:InvokeFunction` on it.
- **Tables.**
  - `jobs`: key `job_id`, TTL on `ttl`, and GSI `response_id-index` (partition key `response_id`, projection `ALL`).
  - `failed_events_store`: key `id` (string), **no TTL**: it is the permanent failure log.
  - `channel_project_link`: key `channel_id`.
  - `embeddings`: partition key `project_id`, sort key `item_id`, TTL on `ttl`.
- **Timeouts and leases.**
  - Every client makes one attempt per call and never sleeps; Lambda's async retry (or, at an entry point, the sender's retry) does the retrying.
  - Per-call timeouts: OpenAI 30 s, GitHub 10 s per request (one request per 100 project items, and one request for all new drafts), Slack 10 s (an entry point needs a shorter budget to stay inside 3 s), judge 20 s per request (up to 8 concurrently), embeddings 20 s per batch.
  - Lambda timeouts must stay below the job leases (`CLAIM_LEASE` 90 s, `FINISHING_LEASE` 180 s), with a margin, so a lease never expires mid-run. Hence the 85 s and 175 s limits above.

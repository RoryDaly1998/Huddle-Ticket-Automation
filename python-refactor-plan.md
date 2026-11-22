# Python Refactor Plan

## Context

The only working implementation is the three n8n workflows. [architecture.md](architecture.md) defines the target: six Python Lambdas on AWS, with Terraform and Rust ports to follow later. This plan covers **only the Python code, contracts, prompts, tests and zip packaging**. Terraform, the Slack manifest and the README rewrite are out of scope. Where Python depends on infra, the plan lists the assumed interface (env vars, IAM, handler paths) in the "Python-side assumptions for infra" section.

## Decisions made

| Topic | Decision |
|---|---|
| Scope | Python + build packaging only |
| Runtime | **Python 3.14** (GA on Lambda), arm64. 3.15 is only a Lambda public preview (no SLA). Bump later by changing one variable; CI also runs the tests on 3.15. |
| Tooling | uv (workspace + lockfile), ruff (lint + format), mypy (strict on `python_common` and contracts), pytest |
| Libraries | pydantic v2, `aws-lambda-powertools` (Logger, Parameters, Metrics), `httpx`, `openai`, `boto3` (runtime-provided, dev-only locally), `moto` for DynamoDB tests, `respx` for HTTP fakes |
| Duplicate judge | OpenAI's decisions model behind `DuplicateJudge` (final choice, no fallback). Phase 0 reads the API docs to pin the exact endpoint and request/response format in `contracts/judge_*.schema.json`. |
| Testing for now | **Unit tests only** (plus contract/test-vector tests, which are also unit-level). No integration, end-to-end or smoke tests, and no GitHub Actions workflow until the CI/CD phase. Lint, mypy and pytest are run manually (optionally via a local pre-commit config). |
| Job ID | Slack `file_id` (huddle path); manual path: `sha256(channel_id + trigger_id)`. Revisit once real Slack payloads are captured. |
| Judge outage | Fail open (create all tickets, note in thread) |

## Open items to resolve during implementation

1. Exact decisions-model API format (Phase 0, from docs).
2. Async-invoke payload limit (currently 256 KB; verify). Decided: `slack-summary` has a size guard in code that measures the serialised `StartWork` payload against a configured limit (with headroom). If too large it does not invoke `starter` and returns an in-channel message asking for a shorter summary. The message text is a named constant with a unit test. No stashing in DynamoDB (transcripts must not be stored).
3. What Slack actually sends on huddle summary upload: capture real `file_shared` payloads and the `files.info` response, and record them as fixtures. Confirm that `file_id` is stable across the double post.

## Repository layout (Python side)

```
contracts/            # JSON Schemas + fixtures + signature test vectors (language-neutral)
  schemas/ start_work, job, ticket_extraction, judge_request, judge_response, newproject_work
  fixtures/ valid & invalid examples per schema
  vectors/  slack_signature.json, standard_webhooks.json, job_id.json
prompts/              # versioned: ticket_extraction_v1.md (fixed regex/format defects)
src/                  # all deployable Python code (not itself a package; import as python_common.x, lambdas.x)
  python_common/        # pyproject, mypy strict
    config.py           # SSM via Powertools Parameters, cached at cold start
    slack_verify.py     # raw-body HMAC + 5 min timestamp window
    webhook_verify.py   # Standard Webhooks
    lambda_invoke.py    # InvocationType=Event wrapper
    models.py           # pydantic models generated/validated against contracts
    stores/ job_store.py (conditional transitions), link_store.py
    clients/ slack.py, github.py, generator.py, judge.py
    fakes/  in-memory implementations of every interface
  lambdas/<snake_case_name>/handler.py + pyproject.toml   # six Lambdas, thin handlers: slack_summary, slack_newproject, webhook, starter, finisher, newproject_worker
scripts/ build.py (per-Lambda arm64 zips), eval_judge.py   # dev tooling, not deployed; stays at root
tests/                # at root; mirrors the contents of src/ (no src segment) plus scripts/; all unit-level, contract and vector tests included; stores tested with moto/fakes
pyproject.toml, uv.lock, ruff/mypy/pytest config
```

## Phases

### Phase 0: Foundations and spikes
- uv workspace, ruff/mypy/pytest config, optional local pre-commit config run manually. No GitHub Actions (deferred to the CI/CD phase).
- Capture real Slack payloads (events, `files.info`, `canvases.getContent`, slash command, modal submission) into fixtures. Redact IDs.
- Spike the OpenAI decisions API and Responses background mode and webhooks (event types, signature headers). Record findings in the plan.
- Verify the async payload limit.

### Phase 1: Contracts and common library
- Write JSON Schemas and fixtures first. pydantic models must pass every valid fixture and reject every invalid one (contract tests).
- Test vectors, with positive and negative cases: Slack signature, Standard Webhooks, `job_id`. Implement `slack_verify` and `webhook_verify` in `python_common`, and the small `job_id` derivation inside `slack-summary` (the only place that derives it; `starter` just reads it from `StartWork`), all tested against the vectors.
- `Config` wrapper, structured logging with `job_id` correlation, a rule that transcript text is never logged (a test asserts the log filter).
- `JobStore` conditional-write state machine: `received → generating → finishing → completed | failed`. A failed condition raises `AlreadyClaimed` or `InvalidTransition` and the handler drops the event. Also a GSI lookup by `response_id`, `created_item_ids` append, and TTL. `LinkStore` upserts with `update_item` so `last_summary_at` never clobbers `project_id`.
- Interfaces (Protocols) with fakes: `TicketGenerator`, `DuplicateJudge`, `GitHubClient`, `SlackClient`, `JobStore`, `LinkStore`, `Config`.

### Phase 2: Walking skeleton (`slack-summary`)
- Verify the signature on the raw body (use the API Gateway `body` plus `isBase64Encoded`), answer `url_verification`, filter to Slack-generated huddle summary files (not just Slackbot), and normalise events and `/meetingsummary` into `StartWork`. Async-invoke `starter` and return 200. The slash command returns an in-channel ack.
- Size guard for manual summaries (see open item 2) with a user-facing error message. Return 5xx if the async invoke is rejected.
- Deploying early to validate the Slack handshake belongs to the Terraform phase.

### Phase 3: `starter`
- Claim the job (conditional put). A duplicate is dropped.
- Look up the link. If none, post the `/setup` hint (no LLM call). Otherwise set `last_summary_at`.
- Huddle path: `files.info` (channel and `thread_ts`/`ts` from `shares`, public or private), `canvases.getContent`. Keep `thread_ts` as a **string**. Manual path: use the thread of the user's message, not the request timestamp.
- `BackgroundTicketGenerator.start()`: background Responses call with the versioned prompt and strict structured-output schema (`{"issues": []}`), no existing tickets in the prompt. Store `response_id`, status `generating`.

### Phase 4: `webhook` and `finisher`
(Renamed from `openai-webhook`; architecture.md and CLAUDE.md now use the new names.)
- Webhook: verify the signature, handle `response.completed`, `failed`, `cancelled` and `incomplete`, async-invoke `finisher` with `{response_id, event_type}`, return 2xx only after Lambda accepts.
- Finisher: transition to `finishing`. On a failure event, post a failure message and mark `failed`.
- Retrieve the response and validate against the ticket schema.
- GitHub: paginated item fetch (Issue, DraftIssue and PullRequest; cursor loop, fixes the n8n 100-item bug). `addProjectV2DraftIssue` per ticket, recording each ID in `created_item_ids` so a retry skips already-created ones.
- Dedupe: judge new tickets against existing ones and against each other, using the threshold from SSM. Fail open on judge errors.
- Slack thread reply (created count, skipped duplicates, partial failures with detail). Mark `completed` or `failed`.

### Phase 5: `slack-newproject` and `newproject-worker`
- `slack-newproject`: verify, open the setup modal on `/setup` within the `trigger_id` window, and on `view_submission` validate input and async-invoke the worker (Python handlers dispatch on `callback_id`).
- Worker: parse `https://github.com/users/<login>/projects/<n>` and repo URLs, with clear errors for other forms. Check the project via `viewerCanUpdate`, validate the repo (allowing an owner different from the project login if possible), list invitations **with pagination** and accept a pending one, upsert the link, and confirm in the channel.

### Phase 6: Judge evaluation and tuning
- `scripts/eval_judge.py`: run the 30–50 labelled pairs through the OpenAI decisions adapter, output a precision/recall curve, and suggest a threshold. Store the threshold in SSM, not in code.

### Phase 7: Packaging and cutover readiness
- `scripts/build.py`: `uv export` per-Lambda requirements, install for `manylinux2014_aarch64`/py3.14, zip with `python_common`, exclude boto3, report sizes.
- Handler path, env var and IAM assumptions documented in "Python-side assumptions for infra" for the Terraform stage.
- No integration/end-to-end tests in this phase; those arrive with the CI/CD phase.

## Known n8n bugs: how the Python fixes them

| n8n bug | Fix |
|---|---|
| No pagination past 100 items | cursor loop in `GitHubClient.list_items` |
| Summary regex mismatch | no regex: structured output has explicit `title`/`summary` fields; prompt and schema agree |
| Wrong `thread_ts` on slash path | use the message/command context ts, kept as string |
| Racy 10s dedupe window | conditional put on `job_id`; no time window |
| Prompt defects | clean `prompts/ticket_extraction_v1.md`; contract test checks prompt/schema agreement |
| Weak huddle filter | filter on Slack-generated huddle summary files |
| No signature verification | verified in every route handler |
| Partial failure | `created_item_ids` makes retries idempotent |

## Python-side assumptions for infra
- Env vars: `JOBS_TABLE`, `LINKS_TABLE`, `SSM_PREFIX`, `STARTER_FN`, `FINISHER_FN`, `NEWPROJECT_WORKER_FN`, `POWERTOOLS_SERVICE_NAME`.
- Handlers: `handler.handler` in each `src/lambdas/<snake_case_name>/`; runtime `python3.14`, arm64.
- IAM per §9 of architecture.md (only `starter` and `finisher` read the OpenAI key; and so on).

## Verification
- Manually run `uv run ruff check . && uv run mypy src/python_common && uv run pytest`: unit, contract and test-vector tests must be green.
- Unit tests assert per handler: duplicate deliveries are dropped, retries create no duplicate drafts (fakes), the oversize-summary guard replies with the error, and no transcript text reaches logs.
- Test vectors (fixed input → expected output pairs, e.g. body + secret + timestamp → expected Slack signature) are stored as JSON in `contracts/vectors/`, so the later Rust ports can run the same cases.

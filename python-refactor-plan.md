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
| Testing for now | **Unit tests only** (plus contract/test-vector tests, which are also unit-level). No integration, end-to-end or smoke tests, and no GitHub Actions workflow until the CI/CD phase. Lint, mypy and pytest are run manually. |
| Job ID | Slack `file_id` (huddle path); manual path: lowercase hex `sha256(channel_id + ':' + trigger_id)` (pinned by `contracts/vectors/job_id.json`). Revisit once real Slack payloads are captured. |
| Judge outage | Fail open (create all tickets, note in thread). A `refusal` or missing answer for a pair counts as a judge error for that pair (treated as not a duplicate) |
| Huddle filter | Cheap `USLACKBOT` pre-filter in `slack-summary`; the authoritative huddle-summary check runs in `starter` after `files.info` (see Phase 0 findings, Q1) |
| Manual-path thread | Thread under `slack-summary`'s in-channel ack, located by `starter` with `conversations.history`; channel-level fallback (Q2) |
| OpenAI `store` | `store=true` on the background response (Q3) |
| Finisher resume (lease) | `mark_finishing` succeeds from `generating`, or from `finishing` when the lease has expired. It writes `finishing_started_at` (the lease) conditionally. Lease duration is a named constant at least the finisher's Lambda timeout (an infra assumption). **Live lease** → raise a retryable `LeaseHeld` error, so Lambda's async retry backoff tries again later rather than dropping. **`completed`/`failed`** → `InvalidTransition`, dropped quietly. On a caught exception the finisher releases the lease before re-raising, so a Lambda retry can resume at once. Resumed runs skip drafts already in `created_item_ids`. Implemented in Phase 4 |
| Starter resume (claim lease) | Same pattern as the finisher. `claim` writes `claimed_at`. It succeeds if the job is absent, or if the job is `received` and its lease has expired or was released. A live lease raises the retryable `LeaseHeld`. Any other status raises `AlreadyClaimed` (drop). On a caught exception, `starter` releases the lease before re-raising. If a retry starts a second OpenAI response, only the `response_id` stored by `mark_generating` is finished; webhooks for any other response find no job and are dropped. Implemented in Phase 3 part B |
| Ticket shape | Each extracted ticket is `{title, summary, body}`: `summary` is 1–3 plain sentences and is what the duplicate judge compares; `body` is the markdown draft body (n8n-style sections: problem statement, scope, acceptance criteria, etc.). The schema change lands with the prompt in Phase 3 |
| Lambda-local modules | Allowed (e.g. `lambdas.slack_summary.job_id`). Zips keep the `lambdas/<name>/` package path, so the handler string is `lambdas.<name>.handler.handler` |
| Async payload limit | 1 MB (1,048,576 bytes) per F4; the size guard uses a configured limit below that, with headroom |

## Open items to resolve during implementation

1. Exact decisions-model API format (Phase 0, from docs).
2. Async-invoke payload limit (currently 256 KB; verify). Decided: `slack-summary` has a size guard in code that measures the serialised `StartWork` payload against a configured limit (with headroom). If too large it does not invoke `starter` and returns an in-channel message asking for a shorter summary. The message text is a named constant with a unit test. No stashing in DynamoDB (transcripts must not be stored).
3. What Slack actually sends on huddle summary upload: capture real `file_shared` payloads and the `files.info` response, and record them as fixtures. Confirm that `file_id` is stable across the double post.

## Phase 0 findings

Documentation research done on 2026-11-22, plus inspection of the installed `openai` 3.57.0 SDK. Status of the open items above:

- **Item 1 (judge API format):** recorded below (§F1). The Decisions API is in OpenAI's official docs.
- **Item 2 (async payload limit):** verified as **1 MB**, not 256 KB (§F4).
- **Item 3 (real Slack payloads):** **still open.** `contracts/fixtures/slack/` holds synthetic, doc-derived fixtures only; see its `README.md`.

### F0. Workspace as built

- uv workspace. The root `pyproject.toml` is a non-installable project (`[tool.uv] package = false`) whose members are `src/python_common` and `src/lambdas/*`. Each member is also non-installable. Code is imported as `python_common.x` / `lambdas.x` through pytest's `pythonpath = ["src"]` (and `mypy_path = "src"`).
- Member names: `python-common`, `slack-summary`, `slack-newproject`, `webhook`, `starter`, `finisher`, `newproject-worker`. uv normalises names, so the snake_case directory name also works (checked with `--package slack_summary`).
- `python-common` depends on `pydantic` and `aws-lambda-powertools`. Client libraries are extras: `http` (httpx) and `openai`. Each Lambda depends on `python-common` with only the extras it needs:

  | Lambda | Dependency |
  |---|---|
  | `slack-summary`, `webhook` | `python-common` |
  | `slack-newproject`, `newproject-worker` | `python-common[http]` |
  | `starter`, `finisher` | `python-common[http,openai]` |

  The root depends on every member, so a plain `uv sync` installs everything the tests and mypy need.
- Per-Lambda requirements for `scripts/build.py` (verified for all six Lambdas). The output has pinned versions only, with no workspace or local-path lines, and contains no boto3/botocore. httpx and openai appear only for the Lambdas that request those extras:

  ```
  uv export --package <member-name> --no-dev --no-hashes --no-emit-workspace --frozen
  ```

  `--frozen` assumes `uv.lock` is current, so `build.py` should run `uv lock --check` first.

- `openai` 3.57 depends on `httpx2` (not `httpx`), so `starter` and `finisher` will ship both `httpx` and `httpx2`.

### F1. Duplicate judge: OpenAI Decisions API

Sources: [guide](https://developers.openai.com/api/docs/guides/decisions.md), [API reference](https://developers.openai.com/api/reference/resources/decisions/methods/create/index.md), [Python reference](https://developers.openai.com/api/reference/python/resources/decisions/index.md), and the installed SDK (`openai.types.decision`).

- **Endpoint:** `POST https://api.openai.com/v1/decisions`, with `Authorization: Bearer <OpenAI API key>` (the same key as generation). The page mentions no beta header.
- **SDK:** `client.decisions.create(*, input, model, questions, prompt_cache_key=..., prompt_cache_options=..., safety_identifier=...) -> Decision` (present in `openai` 3.57.0; `python-common[openai]` pins `openai>=3.57`).
- **Model:** `gpt-6-luna`, "the only currently supported model".
- **Status:** the guide says "The Decisions API is now generally available". It was previously a public beta, when it allowed 200 questions per request. Third-party articles disagree on rollout dates and on whether every key has access; one reported a 403 "not enabled" during the beta. Not blocking, but check access with the real key early.
- **Request:**
  - `input`: a string (≤ 10,485,760 chars) or an ordered array of messages (`role`, `type: "message"`, `content` parts `input_text` / `input_image` / `input_json`; JSON parts ≤ 256 KiB, 32 levels, 10,000 values).
  - `questions`: 1–400 per request, answered in order. Question types:
    - `{"type": "predicate", "instructions": str, "name"?: str}` estimates the probability that a statement about the input is true;
    - `{"type": "choice", "instructions", "name"?, "choices": [{"value": str|bool, "description"?}]}` with 2–255 unique choices;
    - `{"type": "score", "instructions", "name"?, "levels": [{"label", "description"?}]}` with 2–10 levels.
  - Optional: `prompt_cache_key` (≤ 64 chars), `prompt_cache_options` (`mode` `implicit`|`explicit`, `ttl` `"30m"`), `safety_identifier` (≤ 128 chars).
- **Response (`Decision`):**
  - `model`.
  - `usage`: `input_tokens`, `input_tokens_details.{cached_tokens, cache_write_tokens}`, `output_tokens`, `output_tokens_details.reasoning_tokens`, `total_tokens`.
  - `answers[]`, in question order, discriminated by `type`:
    - `predicate`: `{name, probability}`;
    - `choice`: `{name, choice, confidence, probabilities: [{value, probability}]}`;
    - `score`: `{name, score, confidence, probabilities: [{label, value, probability}]}`, where `score` is the probability-weighted mean of 0-based level indices;
    - `refusal`: `{name}`. A refusal affects only that question.

  Example from the reference: one predicate named `damaged` returns `{"type": "predicate", "name": "damaged", "probability": 0.95}`.
- **Calibration:** the docs do not claim the probabilities are calibrated. They say to set thresholds from labelled examples according to the cost of false positives and negatives, which matches Phase 6.
- **Pairwise use:** the docs give no duplicate-detection recipe. Their closest pattern puts a shared reference in `input` and one predicate per candidate, which maps to: `input` = the new ticket; one `predicate` per candidate ticket (existing and earlier-new), with the candidate in `instructions` and its ID in `name` to correlate answers. More than 400 candidates means several requests.
- **Pricing and data:**
  - Uncached input costs $0.10 per 1M tokens, and output tokens are not charged.
  - Caching needs a reusable prefix of at least 1,024 tokens, and cache breakpoints are not allowed inside question instructions.
  - ZDR is supported for eligible customers.
- **For Phase 1's judge schemas:**
  - `judge_request` mirrors `{model, input, questions[]}` (predicate only).
  - `judge_response` mirrors `answers[]` with the `predicate` and `refusal` variants (plus `model`/`usage` if wanted).
  - The judge must handle `refusal` and missing answers. Treating them as judge errors (fail open) is the obvious reading of the existing decision, to confirm in Phase 1.

### F2. OpenAI Responses background mode

Sources: [background guide](https://developers.openai.com/api/docs/guides/background), [data controls](https://developers.openai.com/api/docs/guides/your-data), [create reference](https://developers.openai.com/api/reference/resources/responses/methods/create/index.md), and SDK types.

- **Create:** `POST /v1/responses` with `"background": true`; SDK `client.responses.create(..., background=True)`. Save the returned `id`.
  - Optional `background_options.completion_window`: `"soonest"` (default), `"2h"` or `"8h"`. The `2h`/`8h` windows are discounted and require `background: true`, Standard processing and no streaming.
- **Structured output:** `text.format = {"type": "json_schema", "name": <a-zA-Z0-9_- , ≤64>, "schema": {...}, "strict": true, "description"?}`.
- **Strict-mode schema keywords** ([structured outputs, "Supported schemas"](https://developers.openai.com/api/docs/guides/structured-outputs), checked in Phase 3): types string, number, boolean, integer, object, array, enum and `anyOf`; strings allow `pattern` and `format`; numbers `multipleOf`, `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`; arrays `minItems`, `maxItems`. Every object needs `additionalProperties: false`, and the root must be an object (not `anyOf`). `allOf`, `not`, `dependentRequired`, `dependentSchemas`, `if`/`then`/`else` are unsupported, and an unsupported schema is an API error. The root `$schema` is stripped before sending.
- **Retrieve:** `GET /v1/responses/{id}`; SDK `client.responses.retrieve(id)`.
- **Cancel:** `POST /v1/responses/{id}/cancel`; SDK `client.responses.cancel(id)`. It is idempotent.
- **Statuses:** the SDK's `Response.status` is `Literal["queued", "in_progress", "completed", "failed", "cancelled", "incomplete"]`. `queued` and `in_progress` are non-terminal. The guide's prose spells it "canceled", but the SDK value and the webhook event both use **`cancelled`**. Failure detail is in `error`, and truncation detail in `incomplete_details`.
- **Retention:** this conflicts with the plan's assumption.
  - The data-controls page says: "Background mode with the default completion window stores response data to disk for roughly 10 minutes … If you omit `store` or set it to `false`, the response is deleted after the temporary retention period."
  - The SDK's `store` docstring says it "Defaults to true when omitted. If set to true, response data will be stored for at least 30 days".
  - So Phase 3 must set `store` explicitly:
    - `store=false` keeps OpenAI-side retention to about 10 minutes, but the finisher must retrieve within that window, while webhook delivery retries for up to 72 hours.
    - `store=true` (or omitting it, per the SDK) keeps the summary input for at least 30 days.
  - See the open questions.
- **ZDR:** ZDR projects can use the `2h`/`8h` windows (store is treated as false, and state is stored temporarily). This partly contradicts architecture §9's "not ZDR-compatible". Not acted on.

### F3. OpenAI webhooks

Sources: [webhooks guide](https://developers.openai.com/api/docs/guides/webhooks), [webhooks reference](https://developers.openai.com/api/reference/resources/webhooks/index.md), the [Standard Webhooks spec](https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md), and the SDK source (`openai.resources.webhooks`).

- **Event types:** `response.completed`, `response.failed`, `response.cancelled`, `response.incomplete` (double "l" in `cancelled`).
- **Payload:** `{"object": "event", "id": "evt_…", "type": "response.completed", "created_at": 1750287018, "data": {"id": "resp_…"}}`. **`data.id` is the response ID**, which is all the finisher needs. `object` is optional on response events.
- **Headers:** `webhook-id`, `webhook-timestamp` (Unix seconds) and `webhook-signature`, plus `user-agent: OpenAI/1.0 (+https://platform.openai.com/docs/webhooks)` and `content-type: application/json`.
- **Signature scheme** (Standard Webhooks symmetric, confirmed in SDK source):
  - The secret is `whsec_` + base64; the spec allows 24–64 bytes. The HMAC key is the base64-decoded part after `whsec_`.
  - Signed content: `f"{webhook_id}.{webhook_timestamp}.{raw_body}"`, using the raw body and never re-serialised JSON.
  - Signature: base64(HMAC-SHA256(key, signed content)).
  - The `webhook-signature` header holds one or more space-separated `v1,<base64>` entries (more than one during secret rotation). Accept if any matches, using a constant-time compare.
- **Tolerance:** the spec only says "within some allowable tolerance". The SDK default is `tolerance=300` seconds, applied in both directions (it rejects timestamps that are "too old" or "too new").
- **SDK helpers:**
  - `client.webhooks.unwrap(payload, headers, *, secret=None) -> UnwrapWebhookEvent` verifies, then parses.
  - `client.webhooks.verify_signature(payload, headers, *, secret=None, tolerance=300) -> None` raises `openai.InvalidWebhookSignatureError`. The secret can also come from `OpenAI(webhook_secret=...)` or `OPENAI_WEBHOOK_SECRET`.
  - SDK quirk: header entries without a `v1,` prefix are compared verbatim. Our `webhook_verify` should accept only `v1` entries, and the vectors should include that negative case. The SDK is a handy cross-check when generating `standard_webhooks.json` vectors.
- **Delivery:**
  - A non-2xx response, or no response "within a few seconds", is retried for up to 72 hours with exponential backoff.
  - 3xx redirects are treated as failures.
  - Duplicates can happen, so `webhook-id` is the idempotency key.
  - Endpoints are registered per project in the dashboard, and the secret is shown once.

### F4. AWS Lambda async invoke payload limit

Source: [Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html).

- The "Invocation payload (request and response)" quota is "6 MB each for request and response (synchronous) … **1 MB (asynchronous)**". There is also "1 MB for the total combined size of request line and header values".
- The page notes that it uses "MB (rather than MiB) to refer to 1,024 KB", so the async limit is 1,048,576 bytes.
- The 256 KB figure in item 2 is out of date. The size guard's configured limit should be derived from 1 MB minus headroom.

### F5. Slack Web API and payloads

Sources are linked per item. Fixtures are in `contracts/fixtures/slack/`.

- **Request signing** ([verifying requests](https://docs.slack.dev/authentication/verifying-requests-from-slack)):
  - Headers `X-Slack-Signature` and `X-Slack-Request-Timestamp`.
  - Basestring `"v0:" + timestamp + ":" + raw_body`; signature `"v0=" + hex(HMAC-SHA256(signing_secret, basestring))`.
  - Reject timestamps more than 5 minutes from now, and use a constant-time compare.
  - The page has a worked example (secret `8f742231b10e8888abcd99yyyzzz85a5`, timestamp `1531420618`, expected `v0=a2114d57b48eac39b9ad189dd8316235a7b4a8d21a10bd27519666489c69b503`), which is a good first positive vector for Phase 1. Copy the exact body from the page.
- **`url_verification`** ([doc](https://docs.slack.dev/reference/events/url_verification)): `{"token", "challenge", "type": "url_verification"}`. Respond with 200 and the challenge as `text/plain`, form or JSON `{"challenge": …}`.
- **`file_shared`** ([doc](https://docs.slack.dev/reference/events/file_shared)), scope `files:read`:
  - The event carries only `type`, `channel_id`, `file_id`, `user_id`, `file: {id}` and `event_ts`, inside the `event_callback` envelope (`token`, `team_id`, `api_app_id`, `event`, `type`, `authorizations`, `event_id`, `event_time`).
  - **No huddle fields are on the event.**
- **`files.info`** ([doc](https://docs.slack.dev/reference/methods/files.info)): `GET https://slack.com/api/files.info?file=<id>`, scope `files:read`, Tier 4.
  - `file.shares.public|private` maps channel ID → list of `{ts, thread_ts, channel_name, team_id, reply_count, …}`. Only `public` appears in the docs; `private` is known from the n8n workflow.
  - File-object fields documented for huddles ([file object](https://docs.slack.dev/reference/objects/file-object)): `is_huddle_canvas`, `canvas_metadata.originating_huddle_id`, `huddle_summary_id`, `huddle_transcript_file_id`, `huddle_participant_ids`, `huddle_date_start`, `huddle_date_end`, `ai_generated`. There is no example of their values.
  - Canvas `filetype` is not documented; third parties report `quip`.
  - The fixtures README grades each signal by evidence level and proposes a filter to confirm.
- **Canvas content** ([canvases.getContent](https://docs.slack.dev/reference/methods/canvases.getContent)): this is the documented method, the one n8n already uses.
  - `POST https://slack.com/api/canvases.getContent`, scope `canvases:read` (bot or user), form or JSON.
  - Arguments: `canvas_id` (the file ID) and `content_type` = `markdown` (default) | `html`.
  - Response: `{"ok": true, "content": "<markdown>"}` with no other fields.
  - Errors include `canvas_not_found`, `access_denied`, `no_permission`, `missing_scope`, `canvas_disabled_user_team` and `ratelimited`.
- **`chat.postMessage`** ([doc](https://docs.slack.dev/reference/methods/chat.postMessage)): `POST`, scope `chat:write`, JSON accepted.
  - `thread_ts`: "Provide another message's `ts` value to make this message a reply. Avoid using a reply's `ts` value; use its parent instead." (string). `reply_broadcast` is optional.
  - Response: `{ok, channel, ts, message}`.
  - `text`: keep it to about 4,000 chars; it is truncated above 40,000.
  - Errors include `not_in_channel`, `channel_not_found`, `restricted_action_thread_locked` and `ratelimited`.
- **`views.open`** ([doc](https://docs.slack.dev/reference/methods/views.open)): `POST`, no scope needed, JSON accepted.
  - Arguments: `trigger_id` (expires 3 seconds after it is sent, single use; [modals](https://docs.slack.dev/surfaces/modals)) and `view`.
  - Errors: `expired_trigger_id`, `exchanged_trigger_id`, `invalid_trigger_id`, `view_too_large` (over 250 kB).
- **Slash commands** ([doc](https://docs.slack.dev/interactivity/implementing-slash-commands)):
  - The body is `application/x-www-form-urlencoded` with `token` (deprecated), `team_id`, `team_domain`, `channel_id`, `channel_name`, `user_id`, `user_name`, `command`, `text`, `response_url`, `trigger_id`, `api_app_id` (plus `enterprise_*` on Enterprise Grid).
  - Ack within 3,000 ms. `response_type` is `ephemeral` (default) or `in_channel`.
  - **No `ts`/`thread_ts` is sent, and "Slash commands created by developers cannot … be invoked in message threads."**
- **Interactivity** ([handling](https://docs.slack.dev/interactivity/handling-user-interaction), [view payloads](https://docs.slack.dev/reference/interaction-payloads/view-interactions-payload), [modals](https://docs.slack.dev/surfaces/modals)):
  - The body is form-encoded `payload=<JSON>`, and must be acked within 3 seconds.
  - `view_submission` carries `team`, `user`, `view.{id, callback_id, private_metadata, state.values[block_id][action_id].value, hash}`, **but no channel**. The channel from `/setup` must therefore travel in `private_metadata`, set at `views.open`.
  - An empty 200 closes the modal. `{"response_action": "errors", "errors": {block_id: message}}` shows validation errors; `update`, `push` and `clear` are also available.

### Questions raised by Phase 0 (resolved by the user, 2026-11-22)

1. **Where the huddle filter runs: in `starter`.**
   - `slack-summary` keeps signing-secret-only IAM. It only pre-filters `file_shared` events cheaply, on `user_id == USLACKBOT`.
   - `starter` claims the job, then calls `files.info`. If the file is not a Slack-generated huddle summary, it ends the job quietly before any LLM call and posts nothing.
2. **Manual-path `thread_ts`: thread under the ack, found with `conversations.history`.**
   - `slack-summary`'s in-channel ack for `/meetingsummary` includes a short, unique job reference derived from `job_id`.
   - `starter` reads recent channel history (`conversations.history`, starting at the command time, with the bot's existing `channels:history`/`groups:history` scopes) and finds the ack by that reference. Its `ts` (a string) becomes `thread_ts`.
   - If the ack isn't visible yet, `starter` retries briefly (bounded). If it never appears, it falls back to channel-level replies, so `thread_ts` is optional on the manual path.
   - `search.messages` is not used: it needs a user token.
3. **Background `store`: `store=true`.**
   - The user chose robust retrieval over late webhook delivery.
   - This means OpenAI keeps the summary input for at least 30 days, which departs from architecture §9 ("no stored transcripts" still holds for *our* storage: DynamoDB and logs).

## Repository layout (Python side)

```
contracts/            # JSON Schemas + fixtures + signature test vectors (language-neutral)
  schemas/ start_work, job, ticket_extraction, judge_request, judge_response, newproject_work, finish_work
  fixtures/ valid & invalid examples per schema
  vectors/  slack_signature.json, standard_webhooks.json, job_id.json
prompts/              # versioned: ticket_extraction_v1.md (fixed regex/format defects)
src/                  # all deployable Python code (not itself a package; import as python_common.x, lambdas.x)
  python_common/        # pyproject, mypy strict
    config.py           # SSM via Powertools Parameters, cached at cold start
    slack_verify.py     # raw-body HMAC + 5 min timestamp window
    webhook_verify.py   # Standard Webhooks
    lambda_invoke.py    # InvocationType=Event wrapper
    api_gateway.py      # HTTP API v2 events and responses, shared by the entry-point Lambdas
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
- uv workspace, ruff/mypy/pytest config, run manually. No GitHub Actions (deferred to the CI/CD phase).
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
  - `addProjectV2DraftIssue` is not idempotent. A `GitHubUnavailable` from it (timeout, transport error or 5xx) may mean the draft was created without its ID coming back, so `created_item_ids` cannot cover it and a retry could duplicate it. Part B must either accept that risk explicitly, or on resume re-list the project's items and skip a ticket whose exact title already exists as a draft. (Found in Phase 4 part A review; open for part B.)
- Dedupe: judge new tickets against existing ones and against each other, using the threshold from SSM. Fail open on judge errors.
- Slack thread reply (created count, skipped duplicates, partial failures with detail). Mark `completed` or `failed`.

### Phase 5: `slack-newproject` and `newproject-worker`
- `slack-newproject`: verify, open the setup modal on `/setup` within the `trigger_id` window, and on `view_submission` validate input and async-invoke the worker (Python handlers dispatch on `callback_id`).
- Worker: parse `https://github.com/users/<login>/projects/<n>` and repo URLs, with clear errors for other forms. Check the project via `viewerCanUpdate`, validate the repo (allowing an owner different from the project login if possible), list invitations **with pagination** and accept a pending one, upsert the link, and confirm in the channel.

### Phase 6: Judge evaluation and tuning
- `scripts/eval_judge.py`: run the 30–50 labelled pairs through the OpenAI decisions adapter, output a precision/recall curve, and suggest a threshold. Store the threshold in SSM, not in code.
- **As built (2026-11-23):**
  - Run from the repo root with AWS credentials and `SSM_PREFIX` set: `uv run python scripts/eval_judge.py PAIRS.jsonl [--min-precision 0.95] [--csv CURVE.csv] [--write-ssm]`. A small `sys.path` shim in the script puts `src/` on the path, so no `PYTHONPATH` is needed (ruff `E402` is ignored for `scripts/**`).
  - Input: JSONL, one labelled pair per line (`ticket` `{title, summary, body}`, `existing_item` `{title, body?}`, `duplicate` bool, optional `id`/`note`), validated by a strict pydantic model in the script (not a `contracts/` schema, because no Lambda reads this format). A bad line is an error naming the line, and the whole file is checked before any API call. Format, labelling advice and a **synthetic** 10-pair example are in `evals/` (`README.md`, `judge_pairs_synthetic.jsonl`). Real pairs may hold private project content and must not be committed.
  - Scoring: `DecisionsDuplicateJudge(SsmConfig())` scores each new ticket alone against its one item (`score_ticket(0, [ticket], [item])`). A `JudgeError` or refusal leaves the pair unscored: it is counted and listed, and left out of the curve.
  - Curve: confusion counts, precision and recall at every distinct score taken as the threshold, using the production rule (duplicate when score `>=` threshold). Printed as a table, and optionally written as CSV.
  - Suggestion: the **lowest threshold whose precision is at least `--min-precision`** (default 0.95), favouring precision because a false positive drops real work. If none qualifies, nothing is suggested. The current SSM value is printed alongside when it can be read. The value is the exact score (`repr`), which `parse_duplicate_threshold` reads back unchanged.
  - SSM: by default the script only prints the `aws ssm put-parameter` command. Only with `--write-ssm` (spelt in full; abbreviations are rejected) does it write `<SSM_PREFIX>/duplicate-threshold` (String, overwrite). The path comes from `python_common.config.parameter_path`, which `SsmConfig` also reads through.
  - Tooling: `scripts` is in mypy `files`, and pytest `pythonpath` includes `.` so tests import `scripts.eval_judge`. Tests are in `tests/scripts/test_eval_judge.py`.

### Phase 7: Packaging and cutover readiness
- `scripts/build.py`: `uv export` per-Lambda requirements, install for `manylinux2014_aarch64`/py3.14, zip with `python_common`, exclude boto3, report sizes.
- Handler path, env var and IAM assumptions documented in "Python-side assumptions for infra" for the Terraform stage.
- No integration/end-to-end tests in this phase; those arrive with the CI/CD phase.
- **As built (2026-11-23):**
  - Run from the repo root: `uv run python scripts/build.py [LAMBDA ...]` (snake_case names; none builds all six). It needs network access to fetch wheels (uv's cache is reused). No `sys.path` shim: the script does not import `python_common`.
  - Steps: `uv lock --check` (a stale lock stops the build), then per Lambda the F0 `uv export` command (with the snake_case names in `LAMBDA_NAMES`, which uv normalises to the member names), a requirement filter, `uv pip install --target <tmp> --python-platform aarch64-manylinux2014 --python-version 3.14 --only-binary :all: --no-deps --requirements <file>`, and a zip written to `dist/<name>.zip` (via `<name>.zip.tmp` and an atomic replace, so a failed build never leaves a partial zip). All uv calls go through one helper, `run_uv`.
  - boto3 exclusion: the filter reads the `# via` annotations of the export and drops `RUNTIME_PROVIDED_PACKAGES` (`boto3`) plus every package pulled in only by dropped packages (a fixed point). Anything another runtime dependency also needs is kept: `jmespath` stays for Powertools, and `botocore` would stay if a non-boto3 package needed it. **Today no Lambda's export contains boto3** (it is dev-only), so the filter removes nothing. It is kept, and unit-tested, as a guard. `uv export --prune boto3` gives the same result on the same graph.
  - Zip contents: the installed wheels, `python_common/` (without `fakes/`, which no runtime code imports), `lambdas/__init__.py`, `lambdas/<name>/` and the Lambda's files in `LAMBDA_RESOURCES` (only `starter`: the prompt and the ticket schema). Left out: `__pycache__`, `*.pyc`, `pyproject.toml`, and uv's `.lock` and host console-script launchers (`bin/`) from the install directory. The finisher imports `python_common.clients.generator`, but it only reads files in `start()`, so the finisher's zip needs no resources. A test pins the starter's map to `generator.PROMPT_PATH`/`SCHEMA_PATH`, and another checks that every mapped file exists.
  - Deterministic zips: entries sorted by name, fixed timestamp (1980-01-01), Unix `rw-r--r--` regular-file mode, DEFLATE level 9. Two full builds on one Windows host, with a source file's mtime changed between them, gave byte-identical zips. **Bytes are only stable on one build setup:** this Windows Python uses zlib-ng, which compresses differently from Linux zlib, and with `core.autocrlf=true` and no `.gitattributes` a Windows checkout has CRLF line endings in some source, prompt and schema files where Linux/CI has LF. So zips built on different machines differ, and Terraform would redeploy. It also means a Windows-built `starter` sends a CRLF prompt to OpenAI. **Open (user decision):** add a `.gitattributes` (`* text=auto eol=lf`) and renormalise, and/or build deploy zips only in one place (later: CI).
  - Size check: each zip's zipped and unzipped sizes are printed, and the build exits 1 if one is over `MAX_ZIPPED_BYTES` (50 MiB, direct upload) or `MAX_UNZIPPED_BYTES` (250 MiB). A size exactly on a limit passes. First real build: `starter` and `finisher` about 6.9 MiB zipped / 23.1 MiB unzipped, `slack-newproject` and `newproject-worker` 4.0 / 12.2 MiB, `slack-summary` and `webhook` 3.5 / 10.5 MiB.
  - Checked by hand once: each handler imports with only the extracted zip on `sys.path` (`python -I -S`), plus host builds of the two native wheels (`pydantic-core`, `jiter`) and a separate boto3 standing in for the runtime. The `starter` zip loads its prompt and schema from inside the zip.
  - Tests: `tests/scripts/test_build.py`. `scripts/` is already in mypy `files`.

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
- **Artifacts:** `uv run python scripts/build.py` writes one zip per Lambda to `dist/<snake_case_name>.zip` (gitignored). The builds are deterministic on one build setup, so Terraform's `source_code_hash` (e.g. `filebase64sha256`) changes only when the contents do, provided the zips always come from the same machine/OS and Python build (zlib and checkout line endings differ between Windows and Linux; see Phase 7 "As built"). Runtime **`python3.14`, arm64**. Handler: `lambdas.<snake_case_name>.handler.handler`. The zip root holds `python_common/`, `lambdas/__init__.py`, `lambdas/<snake_case_name>/`, the installed dependencies and any resource files. boto3 is not in the zips; the runtime provides it.
- **Environment for all six:** `SSM_PREFIX` (SSM parameter paths are `<SSM_PREFIX>/<name>`; a trailing slash is ignored), and `POWERTOOLS_SERVICE_NAME` (recommended: the Lambda's name; logs show `service_undefined` without it). Per-Lambda variables are in the table. They hold table names and function names (anything `lambda:Invoke`'s `FunctionName` accepts).
- **SSM parameters** (architecture §6): SecureString `slack-signing-secret`, `slack-bot-token`, `github-token`, `openai-api-key`, `openai-webhook-secret`, `judge-api-key`; String `generation-model`, `judge-model`, `duplicate-threshold`. Each is read on first use with `ssm:GetParameter` (with decryption for SecureStrings; add `kms:Decrypt` only if they use a customer-managed key). A Lambda's role needs only the parameters in its row (architecture §9).
- **Per Lambda** (the SSM, DynamoDB and invoke columns list exactly what the code calls, i.e. the IAM the code itself needs, besides the usual CloudWatch Logs permissions and the failure destination below):

  | Lambda | Trigger | Env vars (besides the two above) | SSM parameters it reads | DynamoDB | May invoke (`lambda:InvokeFunction`, async) | Timeout | Extra files in its zip |
  |---|---|---|---|---|---|---|---|
  | `slack_summary` | `POST /slack/summary` | `STARTER_FN` | `slack-signing-secret` | none | `starter` (`StartWork`) | Must answer Slack within **3 s**, cold start included; a timeout of a few seconds | none |
  | `slack_newproject` | `POST /slack/newproject` | `NEWPROJECT_WORKER_FN` | `slack-signing-secret`, `slack-bot-token` | none | `newproject_worker` (`NewprojectWork`) | Must answer Slack within **3 s**. `views.open` is capped at 2 s (`VIEWS_OPEN_TIMEOUT_SECONDS`): the `trigger_id` and Slack's wait both end 3 s after the command, so a slow call gives up in time to send the failure reply | none |
  | `webhook` | `POST /webhook` | `FINISHER_FN` | `openai-webhook-secret` | none | `finisher` (`FinishWork`) | Quick 2xx for OpenAI; a few seconds | none |
  | `starter` | async from `slack_summary` | `JOBS_TABLE`, `LINKS_TABLE` | `slack-bot-token`, `openai-api-key`, `generation-model` | jobs: `PutItem` (claim), `UpdateItem`; links: `GetItem`, `UpdateItem` (`last_summary_at`) | none | **≤ 85 s** (claim lease, below) | `prompts/ticket_extraction_v1.md`, `contracts/schemas/ticket_extraction.schema.json` |
  | `finisher` | async from `webhook` | `JOBS_TABLE` | `slack-bot-token`, `openai-api-key`, `github-token`, `judge-api-key`, `judge-model`, `duplicate-threshold` | jobs: `GetItem`, `UpdateItem`, `Query` on the **`response_id-index`** GSI (grant it on the index ARN, `<jobs-table-arn>/index/response_id-index`) | none | **≤ 175 s** (finishing lease, below) | none |
  | `newproject_worker` | async from `slack_newproject` | `LINKS_TABLE` | `github-token`, `slack-bot-token` | links: `UpdateItem` (upsert) | none | No lease. A run is a few GitHub and Slack calls at 10 s each (plus one GitHub request per page of invitations) | none |

  - `GetItem` reads use `ConsistentRead=True`. The `response_id-index` query is eventually consistent (GSIs cannot read consistently), so the finisher looks again once after `INDEX_LAG_WAIT_SECONDS`. Most writes carry a `ConditionExpression`; `claim` and `mark_finishing` also ask for `ReturnValuesOnConditionCheckFailure="ALL_OLD"` to tell a live lease from another status, which needs no `GetItem`. None of this needs extra IAM.
  - The jobs table is keyed by `job_id`, has TTL on `ttl`, and has a GSI `response_id-index` with partition key `response_id` and projection `ALL` (`find_by_response_id` reads the whole job from the index). The links table is keyed by `channel_id`.
  - `starter`, `finisher` and `newproject_worker` are invoked asynchronously. Their async-invoke config sends failures to the shared `failed-events` queue (architecture §4, §10). Lambda sends to that destination with the function's execution role, so these three roles also need `sqs:SendMessage` on it.
- Resource files: `python_common.resources.resource_path()` walks up from `python_common/` to find files by repo-relative path. The `starter` zip root must contain `prompts/ticket_extraction_v1.md` and `contracts/schemas/ticket_extraction.schema.json` next to `python_common/`; `scripts/build.py` puts them there (`LAMBDA_RESOURCES`). The other Lambdas read no files. `finisher` imports the generator module, but only `start()` reads the files.
- `webhook`: handler `lambdas.webhook.handler.handler`, env `SSM_PREFIX` and `FINISHER_FN`; IAM to read only `<SSM_PREFIX>/openai-webhook-secret` and to `lambda:InvokeFunction` on `finisher`. Its payload to `finisher` is `FinishWork` (`contracts/schemas/finish_work.schema.json`). It answers 401 (bad signature), 400 (malformed event), 200 (accepted or ignored event type) or 503 (invoke rejected, so OpenAI redelivers).
- Outbound call timeouts (Phase 4 part A). Every client makes **one attempt per call** and never sleeps; retrying is Lambda's async retry.
  - OpenAI generation (`starter`, and `retrieve` in `finisher`): `OPENAI_TIMEOUT_SECONDS` = 30 s, `max_retries=0`.
  - GitHub (`python_common.clients.github`): `GITHUB_TIMEOUT_SECONDS` = 10 s per request. `list_items` makes one request per 100 project items.
  - Slack (`python_common.clients.slack`): `SLACK_TIMEOUT_SECONDS` = 10 s per request (2 s for `views.open` in `slack_newproject`).
  - Duplicate judge (`python_common.clients.judge`): `JUDGE_TIMEOUT_SECONDS` (in `python_common.models`, shared with `finisher`'s time budget) = 20 s per request, `max_retries=0`. It makes one request per new ticket per 400 candidates (existing items plus earlier new tickets), run in sequence.
  - These bound each call, not the finisher's total. The worst case grows with tickets × chunks: for example, 10 new tickets against 1,000 items is 30 judge requests, up to 600 s. That would exceed the finisher's 175 s timeout, so part B should give the finisher an overall time budget (for example, fail open on the judge once it is used up) or otherwise cap the number of judge calls. (Done in part B: `lambdas.finisher.budget.TimeBudget`, taken from the Lambda context's remaining time, fails open on the judge once the budget runs out.)
- Lease timing: `starter` and `finisher` Lambda timeouts must be strictly shorter than their lease constants (`CLAIM_LEASE` = **90 s**, `FINISHING_LEASE` = **180 s**, in `python_common.stores.job_store`), with a margin, so a run's lease never expires while it is still running. Lease starts and the current time are compared in whole seconds (a lease can expire up to 1 s early) and host clocks differ, so: `starter` timeout ≤ **85 s**, `finisher` timeout ≤ **175 s**. Keeping them that short also means a retry after a timeout falls inside Lambda's async retry window (about 1 minute, then about 2).

## Verification
- Manually run `uv run ruff check . && uv run ruff format --check . && uv run mypy && uv run pytest`: unit, contract and test-vector tests must be green. `uv run mypy` checks `src/python_common`, `src/lambdas` and the dev scripts in `scripts/` in strict mode (the paths are set by `files` in `[tool.mypy]`).
- Before a deploy, build the zips with `uv run python scripts/build.py` (needs network for wheels). It must exit 0: the lock is current and every zip is within Lambda's size limits.
- Unit tests assert per handler: duplicate deliveries are dropped, retries create no duplicate drafts (fakes), the oversize-summary guard replies with the error, and no transcript text reaches logs.
- Test vectors (fixed input → expected output pairs, e.g. body + secret + timestamp → expected Slack signature) are stored as JSON in `contracts/vectors/`, so the later Rust ports can run the same cases.

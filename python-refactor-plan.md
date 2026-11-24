# Python Refactor Plan

Phases 0–7 of the Python rewrite are built: contracts, the common library, all six Lambdas, the judge eval script and zip packaging. The earlier versions of this plan, with the API research and as-built notes, are in git history. This file holds only:

- the **remaining Python work**,
- the **open items**, and
- the **infra contract** that the Terraform phase must meet.

Design decisions live in [architecture.md](architecture.md); the work below implements them. Every change keeps the checks in CLAUDE.md green and adds unit, contract and test-vector tests as it goes.

## 1. Naming

The docs use the Lambda and hand-off schema names decided on 2026-11-24. **The code still uses the old names.** Rename it to match:

| Old name (code today) | New name (hyphenated / snake_case) |
|---|---|
| `slack-summary` / `slack_summary` | `summary-posted` / `summary_posted` |
| `slack-newproject` / `slack_newproject` | `project-added` / `project_added` |
| `webhook` | `issues-generated` / `issues_generated` |
| `starter` | `process-summary` / `process_summary` |
| `finisher` | `process-issues` / `process_issues` |
| `newproject-worker` / `newproject_worker` | `setup-project` / `setup_project` |
| `start_work.schema.json`, model `StartWork` | `summary_posted_to_process_summary.schema.json` |
| `finish_work.schema.json`, model `FinishWork` | `issues_generated_to_process_issues.schema.json` |
| `newproject_work.schema.json`, model `NewprojectWork` | `project_added_to_setup_project.schema.json` |
| env `STARTER_FN`, `FINISHER_FN`, `NEWPROJECT_WORKER_FN` | `PROCESS_SUMMARY_FN`, `PROCESS_ISSUES_FN`, `SETUP_PROJECT_FN` |

The rename touches:
- the directories under `src/lambdas/` and `tests/lambdas/`
- the uv workspace members
- `scripts/build.py`'s `LAMBDA_NAMES` and `LAMBDA_RESOURCES`
- the contract schemas and their fixture directories
- the pydantic model names
- the env var names

Hand-off schemas follow the naming rule in architecture §8 (`<sending service>_to_<receiving service>`).

## 2. Embedding shortlist, embedding cache and parallel judging

Implements architecture §6 (`ticket_embeddings`) and §7 (*Duplicate detection*). Replaces today's sequential dedupe in `process-issues`.

- **Draft body.** `drafts.DRAFT_BODY_TEMPLATE` becomes `"## Summary\n\n{summary}\n\n{body}"`. No drafts have been created yet, so nothing needs migrating.
- **Embed text** (new module in `python_common`, pinned by `contracts/vectors/embed_text.json`):
  - `summary_section(body) -> str | None`: normalise line endings to `\n`, find the first line equal to `## Summary` (trailing whitespace ignored), take the lines up to the next line starting with `#` or the end, and trim. An empty result counts as no summary. `## 1. Summary` and other headings never match.
  - `embed_text(title, summary)`: `f"{title.strip()}\n\n{summary.strip()}"`, or `title.strip()` alone when there is no summary.
  - `text_hash(text)`: lowercase hex SHA-256 of the UTF-8 text.
  - Vectors: bodies with and without the heading, `## 1. Summary`, CRLF bodies, trailing whitespace on the heading, an empty section, a section followed by another heading, and a round trip (a draft body built from `title`/`summary`/`body` gives back the same embed text and hash as the ticket).
- **GitHub fetch.** `list_items` returns only Issues and DraftIssues and leaves out archived items (first check how the items query exposes archived items). `ProjectItem` gains the parsed summary (or the embed text).
- **`Embedder`:**
  - Interface, fake and OpenAI adapter (`client.embeddings.create`), with the model and `dimensions` taken from SSM.
  - Batched to the API's per-request input limit (check it in the docs).
  - One attempt per call with a timeout (`EMBEDDINGS_TIMEOUT_SECONDS`) and `max_retries=0`. Failures raise a new `EmbeddingError`.
  - Reads `embeddings-api-key`.
- **`EmbeddingStore`:**
  - Interface, fake and DynamoDB store for `ticket_embeddings`.
  - Query by `project_id` (paginated), batch-write rows, set `ttl` on refresh, and batch-delete stale rows.
  - Vectors are stored as little-endian float32 bytes.
  - Add `contracts/schemas/ticket_embedding.schema.json` with fixtures, so Rust reads the same shape.
  - Callers catch every store error and treat it as an empty cache, with writes skipped.
- **Cache sync** (`process-issues`):
  - A row is a hit when `text_hash`, `embedding_model` and `dimensions` all match.
  - Embed the misses together with all new tickets, then write the misses.
  - Push rows with less than 90 days to `ttl` back to 180 days ahead. The TTL values are code constants.
  - After a complete fetch, delete rows whose `item_id` was not fetched.
  - After drafts are created, write their vectors under the new item IDs.
- **Shortlist:**
  - The pool is the existing items (minus this job's own recorded drafts) plus the other new tickets.
  - Rank by cosine similarity and keep the top K at or above the minimum. Plain Python is enough at 256 dimensions and hundreds of items; add numpy only if profiling says so.
  - An empty shortlist means no judge call.
- **Judge.**
  - `DuplicateJudge` takes one ticket and its shortlist, and returns a probability per candidate ID (`None` for refusals and missing answers).
  - Questions are named `existing_<i>` and `new_<j>`.
  - The `input` and every predicate use the embed text, replacing `body[:2000]` for existing items.
  - Drop the 400-question chunking. `judge_request.schema.json` keeps `maxItems: 400`, the API's limit.
- **`plan_tickets`:**
  - Run the judge requests concurrently in a thread pool with a fixed cap (`JUDGE_CONCURRENCY`, for example 8).
  - Then walk the tickets in extraction order:
    - recorded drafts are kept;
    - an identical normalised title to an existing item or an earlier kept ticket means duplicate;
    - a score at or above the threshold against an existing item, or against an earlier kept ticket in either direction, means duplicate.
  - A failed request leaves only its own ticket's pairs unscored.
  - An embedding failure skips the shortlist and the judge; the title checks still run.
  - Remove `budget.py`, `judge_cost`, `JUDGE_RESERVE` and the `out_of_time` reporting.
- **Slack reply.** Say when the embeddings step failed, as for judge failures, and drop the out-of-time message.

## 3. Update the judge eval

`scripts/eval_judge.py` (usage in `evals/README.md`) needs updating after section 2:

- **Scoring call.** Call the new judge interface (a shortlist per ticket) instead of `score_ticket(index, tickets, items)`.
- **Text compared.** Pairs are compared on embed texts, so labelled `existing_item` bodies need a `## Summary` section to be compared on more than the title.
- **Shortlist check.** Report each pair's cosine similarity and whether true duplicates clear the shortlist minimum (0.5), so that SSM value is tuned from the same data. A true duplicate below the minimum never reaches the judge.

## 4. Open items

- **Real Slack payloads.**
  - Capture real `file_shared` events and the `files.info` and `canvases.getContent` responses, plus a slash command and a modal submission, as redacted fixtures. `contracts/fixtures/slack/` holds synthetic, doc-derived fixtures only; see its README.
  - Confirm the huddle filter's `files.info` signals, and that `file_id` is stable across Slack's double post (the `job_id` choice depends on it).
- **Reproducible zips (user decision).**
  - Today the zips are byte-identical only when built on the same machine. Windows Python's zlib-ng compresses differently from Linux zlib. Also, with `core.autocrlf=true` and no `.gitattributes`, a Windows checkout has CRLF line endings, so a Windows-built `process-summary` sends a CRLF prompt to OpenAI.
  - Options: add `.gitattributes` (`* text=auto eol=lf`) and renormalise, and/or build deploy zips in one place only (later: CI).
- **API access.** Check that the real keys can reach the Decisions API (a 403 "not enabled" was reported during its beta) and the embeddings API.

## 5. Infra contract for Terraform

What the Python code expects from the infrastructure. The "(§2)" marks mean the item arrives with section 2.

- **Artifacts.**
  - `uv run python scripts/build.py` writes `dist/<snake_case_name>.zip`.
  - Runtime **`python3.14`, arm64**; handler `lambdas.<snake_case_name>.handler.handler`.
  - boto3 is not in the zips; the runtime provides it.
  - `process-summary`'s zip carries `prompts/ticket_extraction_v1.md` and `contracts/schemas/ticket_extraction.schema.json` at its root. The other zips carry no resource files.
  - Zips are deterministic only when built on one setup (see open items), so `source_code_hash` is stable only if the zips always come from the same machine.
- **Environment for all six.**
  - `SSM_PREFIX`: parameters live at `<SSM_PREFIX>/<name>`.
  - `POWERTOOLS_SERVICE_NAME`: the Lambda's name.
  - Env vars that name functions accept anything `lambda:InvokeFunction`'s `FunctionName` accepts.
- **SSM parameters.** Each is read on first use with `ssm:GetParameter` (with decryption for SecureStrings; `kms:Decrypt` only for a customer-managed key). A role gets only the parameters in its row below.
  - SecureString: `slack-signing-secret`, `slack-bot-token`, `github-token`, `openai-api-key`, `openai-webhook-secret`, `judge-api-key`, `embeddings-api-key` (§2).
  - String: `generation-model`, `judge-model`, `duplicate-threshold`, plus `embedding-model`, `embedding-dimensions`, `shortlist-top-k` and `shortlist-min-similarity` (§2).
- **Per Lambda.** The SSM, DynamoDB and invoke columns are exactly what the code calls:

  | Lambda | Trigger | Env vars | SSM parameters | DynamoDB | May invoke (async) | Timeout |
  |---|---|---|---|---|---|---|
  | `summary-posted` | `POST /slack/summary` | `PROCESS_SUMMARY_FN` | `slack-signing-secret` | none | `process-summary` | A few seconds; must answer Slack within **3 s**, cold start included |
  | `project-added` | `POST /slack/newproject` | `SETUP_PROJECT_FN` | `slack-signing-secret`, `slack-bot-token` | none | `setup-project` | A few seconds; must answer within **3 s** (`views.open` is capped at 2 s) |
  | `issues-generated` | `POST /webhook` | `PROCESS_ISSUES_FN` | `openai-webhook-secret` | none | `process-issues` | A few seconds; quick 2xx for OpenAI |
  | `process-summary` | async | `JOBS_TABLE`, `LINKS_TABLE` | `slack-bot-token`, `openai-api-key`, `generation-model` | jobs: `PutItem`, `UpdateItem`; links: `GetItem`, `UpdateItem` | none | **≤ 85 s** |
  | `process-issues` | async | `JOBS_TABLE`, `EMBEDDINGS_TABLE` (§2) | `slack-bot-token`, `openai-api-key`, `github-token`, `judge-api-key`, `judge-model`, `duplicate-threshold`; §2 adds `embeddings-api-key`, `embedding-model`, `embedding-dimensions`, `shortlist-top-k`, `shortlist-min-similarity` | jobs: `GetItem`, `UpdateItem`, `Query` on `<jobs-table-arn>/index/response_id-index`; ticket_embeddings (§2): `Query`, `BatchWriteItem`, `UpdateItem` | none | **≤ 175 s** |
  | `setup-project` | async | `LINKS_TABLE` | `github-token`, `slack-bot-token` | links: `UpdateItem` | none | A few GitHub and Slack calls at 10 s each |

- **Tables.**
  - `jobs`: key `job_id`, TTL on `ttl`, and GSI `response_id-index` (partition key `response_id`, projection `ALL`).
  - `channel_project_link`: key `channel_id`.
  - `ticket_embeddings` (§2): partition key `project_id`, sort key `item_id`, TTL on `ttl`.
- **Failure destination.** `process-summary`, `process-issues` and `setup-project` send async-invoke failures to the shared `failed-events` queue, so their roles need `sqs:SendMessage` on it.
- **Timeouts and leases.**
  - Every client makes one attempt per call and never sleeps; Lambda's async retry does the retrying.
  - Per-call timeouts: OpenAI 30 s, GitHub 10 s per request (one request per 100 project items), Slack 10 s, judge 20 s per request (run concurrently from §2), embeddings one timeout per batch (§2).
  - Lambda timeouts must stay below the job leases (`CLAIM_LEASE` 90 s, `FINISHING_LEASE` 180 s), with a margin, so a lease never expires mid-run. Hence the 85 s and 175 s limits above.

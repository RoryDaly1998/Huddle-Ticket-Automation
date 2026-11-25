# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

Slack huddle summaries → LLM-extracted action items → de-duplicated against the channel's GitHub Project → added as **draft items**.

The branch `20-refactor-into-python` is mid-migration. The only deployed implementation is the **n8n workflows** in `n8n-workflows/` (set up per [README.md](README.md)). A Python/AWS rewrite is being built under `src/`, `tests/`, `contracts/`, `prompts/` and `scripts/`; Terraform does not exist yet.

The eight Lambdas are `summary-posted`, `project-added` and `issues-generated` (API Gateway entry points), `process-summary`, `process-issues` and `setup-project` (asynchronously invoked workers), and `job-failed` (fed by the `failed-events` SQS queue) and `check-stale-jobs` (EventBridge Scheduler, every 15 minutes); the code uses the snake_case forms under `src/lambdas/`. The last two are planned, not built: see the failure-handling rework in the plan.

## Where to look

- [architecture.md](architecture.md): the target design and the source of truth for the rewrite. It covers components, flows, the data model, duplicate detection (§7), the decision log (§15) and the n8n bugs to fix rather than port (§16).
- [python-refactor-plan.md](python-refactor-plan.md): the remaining Python work (next: the failure-handling rework and the dead code it removes), open items, and the infra contract (env vars, SSM parameters, IAM, timeouts, async settings) that Terraform must meet.

## Commands

Run manually (there is no CI yet): `uv run ruff check . && uv run ruff format --check . && uv run mypy && uv run pytest`. Build the Lambda zips with `uv run python scripts/build.py`.

## CodeGraph

- Use the CodeGraph tool (`codegraph_explore`) for multi-file edits and changes that span several modules, to find the symbols and call paths involved.
- Do not use it for small bug fixes or for editing a single file; read and edit the file directly.

## Rules that apply to every change

- **Schema naming:** every JSON Schema is named `<sender>_to_<receiver>` in snake_case, e.g. `summary_posted_to_process_summary`, `lambdas_to_db_jobs`, `process_issues_to_duplicate_judge`. Storage is `db_<what is stored>` (`db_jobs`, `db_embeddings`); swappable vendors use their role (`ticket_generator`, `duplicate_judge`, `embedder`); Slack and GitHub use the vendor name; several Lambdas making the same call are `lambdas`. Details in architecture §8.
- **Contracts and test vectors:** contracts live in `contracts/` as JSON Schemas with fixtures, so the later Rust ports can pass the same cases. Logic that must exist in both languages is pinned by shared test vectors, not shared libraries.
- **Never store huddle transcripts or summary text** in our storage (DynamoDB, including `failed_jobs`, and logs). The only exception is Lambda's own copy of a failed manual-path payload in the `failed-events` queue.
- **Failures take one path (architecture §5.5):** workers never mark a job `failed` or post failure messages. A permanent error is sent to `failed-events` and the worker returns; a retryable one releases the lease and is re-raised for Lambda's retries. Only `job-failed` fails jobs and messages users, once. Nothing moves a job out of `failed` or `completed`.
- **No partial-success handling:** steps are idempotent and a failure fails the whole run; a retry redoes it.
- The two rules above are the target. The current workers still fail jobs, post failure messages and record `created_item_ids` until the failure-handling rework in the plan lands. Don't extend that old behaviour in the meantime.
- **Retry timing is coupled:** the workers' 10-minute maximum event age and the 30-minute stale-job cutoff depend on each other. Change them together.
- **Entry-point Lambdas must answer fast:** Slack within 3 s, OpenAI webhooks with a quick 2xx. Do slow work in an asynchronously invoked Lambda. Verify Slack signatures over the raw request body.

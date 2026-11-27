# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

Slack huddle summaries → LLM-extracted action items → de-duplicated against the channel's GitHub Project → added as **draft items**.

The only deployed implementation is the **n8n workflows** in `n8n-workflows/` (set up per [README.md](README.md)). A Python/AWS rewrite lives under `src/`, `tests/`, `contracts/`, `prompts/` and `scripts/`; Terraform does not exist yet.

The code still implements the previous design (eight Lambdas, hand-written models). The target is seven Lambdas: `slack_webhook` and `ticket_generator_webhook` (API Gateway entry points), `process_summary`, `process_tickets` and `setup_project` (asynchronously invoked workers), `process_failed_events` (fed by `failed_events_queue`) and `process_stuck_jobs` (EventBridge Scheduler, every 15 minutes). [python-refactor-plan.md](docs/python-refactor-plan.md) lists the migration steps in order.

## Where to look

- [architecture.md](docs/architecture.md): the target design and the source of truth. Contracts and naming are §8, failures §5.5, the decision log §15.
- [python-refactor-plan.md](docs/python-refactor-plan.md): the migration work list, open items, and the infra contract Terraform must meet.

## Commands

Run manually (no CI yet): `uv run ruff check . && uv run ruff format --check . && uv run mypy && uv run pytest`. Build the Lambda zips with `uv run python scripts/build.py`. The user commits; never commit.

## CodeGraph

Use `codegraph_explore` for changes that span several modules. For a small fix or a single-file edit, read and edit the file directly.

## Rules

These describe the target. Until the plan step that introduces a rule has landed, keep the existing code, models, fixtures and tests working; never delete them ahead of the plan.

- **Schema naming:** `<operation>_to_<destination>` in snake_case: what the payload is, then who receives it (`add_tickets_to_github`, `ticket_ids_to_process_tickets`). Destinations are Lambda names, `db_<table>`, `slack`/`github`, or a vendor role (`ticket_generator`, `embedder`, `duplicate_judge`). `confirm_receipt_to_<destination>` only when nothing in the body is used. One schema per call; a queue is never a destination. Details in architecture §8.
- **Generated models:** pydantic models are generated from the JSON Schemas and committed; never edit them by hand. Change the schema and regenerate. A class's name is its schema's name in PascalCase.
- **Unit tests only:** pytest with fakes and inline test data; no shared fixture or vector files.
- **Never store huddle transcripts or summary text** in our storage (DynamoDB, including `failed_events_store`, and logs). The only exception is Lambda's own copy of a failed manual-path payload in `failed_events_queue`.
- **Failures take one path (architecture §5.5):** only server-side failures enter it.
  - Workers never mark a job `failed` or post failure messages. A permanent error is sent to `failed_events_queue` and the worker returns; a retryable one releases the lease and is re-raised for Lambda's retries.
  - Entry points send a handled error to the queue and answer 2xx; only unexpected or transient errors answer 5xx, so the sender retries.
  - Only `process_failed_events` fails jobs and messages users, once. Nothing moves a job out of `failed` or `completed`.
- **No partial-success handling:** steps are idempotent and a failure fails the whole run; a retry redoes it.
- **Retry timing is coupled:** the workers' 10-minute maximum event age and the 30-minute stuck-job cutoff depend on each other. Change them together.
- **Entry points must answer fast:** Slack within 3 s, OpenAI with a quick 2xx. Do slow work in an asynchronously invoked Lambda. Verify signatures over the raw request body. Each route loads only what it needs.

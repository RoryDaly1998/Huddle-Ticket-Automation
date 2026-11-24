# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

Slack huddle summaries → LLM-extracted action items → de-duplicated against the channel's GitHub Project → added as **draft items**.

The branch `20-refactor-into-python` is mid-migration. The only deployed implementation is the **n8n workflows** in `n8n-workflows/` (set up per [README.md](README.md)). A Python/AWS rewrite is being built under `src/`, `tests/`, `contracts/`, `prompts/` and `scripts/`; Terraform does not exist yet.

**Names are mid-rename.** The docs use the new Lambda names (`summary-posted`, `project-added`, `issues-generated`, `process-summary`, `process-issues`, `setup-project`) and hand-off schema names, but the code still uses the old ones (`slack-summary`, `starter`, `finisher`, …). The mapping is in the plan's "Naming" section.

## Where to look

- [architecture.md](architecture.md): the target design and the source of truth for the rewrite. It covers components, flows, the data model, duplicate detection (§7), the decision log (§15) and the n8n bugs to fix rather than port (§16).
- [python-refactor-plan.md](python-refactor-plan.md): the remaining Python work, open items, and the infra contract (env vars, SSM parameters, IAM, timeouts) that Terraform must meet.

## Commands

Run manually (there is no CI yet): `uv run ruff check . && uv run ruff format --check . && uv run mypy && uv run pytest`. Build the Lambda zips with `uv run python scripts/build.py`.

## Rules that apply to every change

- **Schema naming:** a JSON Schema for information passed from one of our services to another is named `<sending service>_to_<receiving service>` in snake_case (e.g. `summary_posted_to_process_summary.schema.json`). Other schemas keep descriptive names.
- **Contracts and test vectors:** contracts live in `contracts/` as JSON Schemas with fixtures, so the later Rust ports can pass the same cases. Logic that must exist in both languages is pinned by shared test vectors, not shared libraries.
- **Never store huddle transcripts or summary text** in our storage (DynamoDB, logs).
- **Entry-point Lambdas must answer fast:** Slack within 3 s, OpenAI webhooks with a quick 2xx. Do slow work in an asynchronously invoked Lambda. Verify Slack signatures over the raw request body.

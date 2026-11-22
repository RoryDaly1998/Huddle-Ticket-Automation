# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

Slack huddle summaries → LLM-extracted action items → de-duplicated against the channel's GitHub Project → added as **draft items**. The current branch (`20-refactor-into-python`) is mid-migration: the only working implementation is the **n8n workflows** in `n8n-workflows/`; the Python/AWS rewrite described in [architecture.md](architecture.md) has not been written yet (no Python source, tests, Terraform, or build/lint commands exist). `.venv/` is a local, gitignored virtualenv.

## Where things are

- [architecture.md](architecture.md) — target design (Python Lambdas on AWS via Terraform, selected pieces later in Rust). Source of truth for the rewrite: components (§4), flows (§5), data model (§6), proposed monorepo layout (§8), decision log (§15), and known n8n bugs to *fix, not port* (§16).
- [README.md](README.md) — setup guide for the **current n8n deployment** (Slack app, n8n credentials, data tables, GitHub classic PAT).
- `n8n-workflows/` — exported workflow JSON: `transcription.json` (main flow), `setup.json` (`/setup` project linking), `transcription-error-handler.json`. Edit by importing/pasting into n8n; there is no build step.
- [slack-bot-manifest.json](slack-bot-manifest.json) — Slack app manifest (paste into app settings).
- `channel-lock-table-schema.csv`, `project-link-table-schema.csv` — column headers for the two n8n data tables (`channel_id,last_upload_ts` and `channel_id,project_id`).

## Architecture essentials (target)

- Lambdas: `slack-summary`, `slack-newproject`, `webhook` (API Gateway entry points that verify signatures and return fast), and `starter`, `finisher`, `newproject-worker` (invoked asynchronously with `InvocationType=Event`; failures go to a shared `failed-events` SQS queue). Delivery is at-least-once, so state transitions use conditional writes on a `jobs` record (`generating` → `finishing` → `completed`/`failed`).
- LLM work uses a **background** OpenAI response; completion arrives via a signed webhook (Standard Webhooks) rather than polling.
- Contracts (invocation payloads, job record, ticket schema, judge I/O) are JSON Schema files with shared fixtures; logic that must exist in both Python and Rust (Slack/webhook signature verification, dedupe keys) is pinned by shared test vectors, not shared libraries.
- GitHub access is a bot account with a **classic** PAT (fine-grained tokens/Apps can't reach user-owned Projects v2); user-owned projects only.
- Transcripts/summary text must not be stored.

## Gotchas

- Slack must be acked within 3s; OpenAI webhooks need a quick 2xx — never do slow work inline in the entry-point Lambdas.
- Slack signature verification must use the raw request body.
- Known n8n bugs listed in architecture.md §16 (no pagination past 100 project items, `## Summary` vs `## 1. Summary` regex mismatch, wrong `thread_ts` on the slash-command path, racy 10s dedupe window) should be fixed in the rewrite rather than ported.

# Schemas

Each `.schema.json` file is a JSON Schema for one exchange between two parts of the system: Lambda to Lambda, Lambda to database, or Lambda to an outside API. The schemas are the contracts: a Lambda can be rewritten (Rust is planned) without changing the others.

The design these follow is in [architecture.md](../../architecture.md) §8: naming, layout and the full list of schemas. In short:

- **One schema per call.** A request and its response are separate schemas.
- **Named `<operation>_to_<destination>`**: what the payload is, then who receives it (`add_tickets_to_github`, then `ticket_ids_to_process_tickets`). A response is `confirm_receipt_to_<destination>` only when nothing in its body is used.
- **Models are generated from these files.** The pydantic models are generated and committed, never edited by hand; a class's name is its schema's name in PascalCase.
- **Layout:** `contract-schemas/internal/{lambda_to_lambda,lambda_to_db}` and `contract-schemas/external/{slack,github,ticket_generator,embedder,duplicate_judge}`.

## Current state

The files in this directory are still the previous set (`<sender>_to_<receiver>`, several calls per schema, with fixtures in `../fixtures/` and vectors in `../vectors/`). They move to the layout above in step 1.2 of [python-refactor-plan.md](../../python-refactor-plan.md).

## Diagram

[`schema_architecture.drawio`](schema_architecture.drawio) draws every contract as a labelled arrow: solid for a request or message, dashed for a response. Open it with the Draw.io Integration extension for VS Code, or with File > Open at [app.diagrams.net](https://app.diagrams.net/). [`architecture.drawio`](architecture.drawio) is the older diagram of the previous design.

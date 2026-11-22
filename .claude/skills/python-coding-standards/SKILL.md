---
name: python-coding-standards
description: Coding standards for all Python source in this project - simplicity, separation of concerns, DRY, snake_case file and directory names, type hints, docstrings, comment policy and ordering rules. Use when writing, changing or reviewing non-test Python code.
---

# Python coding standards

Python source is written to these standards and reviewed against them. Test code follows the `python-test-writing-and-refactoring` skill instead wherever the two differ.

## Design

- **Simplicity**: write the simplest code that does the job. No speculative features, configuration or layers. No clever tricks where plain code works.
- **Separation of concerns**: each function, method and class does one thing. Keep I/O, parsing, business logic and presentation apart.
- **DRY and abstraction**: extract repeated logic into a well-named function or class, and reuse existing helpers instead of re-implementing them. Do not abstract something used only once unless it clearly improves readability.
- **File organisation**: each module has one clear responsibility. Group related modules into packages. Keep files small enough to read in one pass.

## Naming

- **Directories and files are snake_case**: lowercase letters, digits and underscores only, e.g. `src/lambdas/slack_summary/handler.py`, `prompts/ticket_extraction_v1.md`. Put versions and variants in the name with underscores, not extra dots or hyphens. The only exceptions are names fixed by tools or conventions (`pyproject.toml`, `uv.lock`, `conftest.py`, `__init__.py`, `README.md`) and multi-part extensions such as `.schema.json`.
- **Variables, functions, methods and modules** are descriptive snake_case. Classes are PascalCase. Constants are UPPER_SNAKE_CASE. No single-letter names except trivial loop indices or comprehension variables.

## Typing and documentation

- **Type hints** on every non-test function and method signature (all parameters and return types). Be precise and avoid `Any` unless it is genuinely unavoidable.
- **Docstrings** on every non-test module-level function, class and method. Make them detailed but concise and in plain language. Say what it does, its arguments, what it returns and what it raises. Do not restate the type hints.
- **Inline comments** are kept to a minimum. Use them only to explain a particularly complex step, never to narrate obvious code.

## Ordering

- Module-level functions in a file are in alphabetical order.
- Within a class, dunder methods come first (`__init__` first), then the remaining methods in alphabetical order.

## Tooling

- Code passes the project's ruff lint, ruff format and mypy configuration.

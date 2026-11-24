---
name: python-coding-standards
description: Coding standards for all Python source in this project - simplicity, separation of concerns, DRY, snake_case file and directory names, type hints, docstrings, comment policy and ordering rules. Use when writing, changing or reviewing non-test Python code.
---

# Python coding standards

Python source is written to these standards and reviewed against them. Test code follows the `python-test-writing-and-refactoring` skill instead wherever the two differ.

## Design

- **Simplicity**: write the simplest code that does the job. No speculative features, configuration or layers. No clever tricks where plain code works.
- **Separation of concerns**: each function, method and class does one thing. Keep I/O, parsing, business logic and presentation apart.
- **DRY and abstraction**: extract repeated logic into a well-named function or class, and reuse existing helpers instead of re-implementing 
- **File organisation**: each module has one clear responsibility. Group related modules into packages.

## Naming

- **Directories and files are snake_case**: lowercase letters, digits and underscores only, e.g. `src/lambdas/summary_posted/handler.py`, `prompts/ticket_extraction_v1.md`. Put versions and variants in the name with underscores, not extra dots or hyphens. The only exceptions are names fixed by tools or conventions (`pyproject.toml`, `uv.lock`, `conftest.py`, `__init__.py`, `README.md`) and multi-part extensions such as `.schema.json`.
- **Variables, functions, methods and modules** are descriptive snake_case. Classes are PascalCase. Constants are UPPER_SNAKE_CASE. No single-letter names except trivial loop indices or comprehension variables.

## Typing and documentation

- **Type hints** on every non-test function and method signature (all parameters and return types). Be precise and avoid `Any` unless it is genuinely unavoidable.
- **Docstrings** on every non-test module-level function, class and method. Use PEP 257 style (the ruff `pydocstyle` convention is `pep257`): a one-line summary, then a blank line and free-form plain-language prose where more detail is needed. Do not use `Args:`, `Returns:` or `Raises:` sections. Mention a parameter, return value or raised exception in the prose only when it is not obvious from the name and type hints. Make them detailed but concise. Do not restate the type hints. Write in simple language and use technical terms only where necessary.
- **Inline comments** are kept to a minimum. Use them only to explain a particularly complex step, never to narrate obvious code.

## Ordering

- Module-level functions in a file are in alphabetical order.
- Within a class, dunder methods come first (`__init__` first), then the remaining methods in alphabetical order.

## Tooling

- Code passes the project's ruff lint, ruff format and mypy configuration.

---
name: python-test-writing-and-refactoring
description: Rules for writing or refactoring Python unit tests with pytest - layout, one test class per function/class/method, ordering, patching at point of use, mock/fixture/helper placement, and full base/edge/boundary/failure coverage. Use only when writing, extending or refactoring Python unit tests; not for writing production code.
---

# Python test writing and refactoring

Follow every rule below when writing new tests or refactoring existing ones. These rules are not optional.

## 1. Intended behaviour comes first

- Design tests against the **intended** behaviour, taken from the plan, the architecture doc, contracts and the user. Never derive expected values by reading what the current code returns.
- Never write a test to fit existing code, and never weaken a test (loosen an assertion, delete a case, widen a tolerance, mark it xfail/skip) so that weak or incorrect code passes. If a test fails because the code is wrong, fix the code or report the defect.
- If the intended behaviour of any case is unknown or ambiguous, **ask the user** before writing that test. If you cannot ask directly, stop and return the questions.

## 2. Layout and configuration

- Use **pytest** with **pytest-mock** (`mocker`) for all unit tests.
- Configure pytest in `pyproject.toml` so tests live in a separate `tests/` directory:

  ```toml
  [tool.pytest.ini_options]
  testpaths = ["tests"]
  pythonpath = ["src"]
  addopts = ["--strict-markers", "--import-mode=importlib"]
  ```

  `importlib` mode lets test files with the same name sit in different directories without `__init__.py` files. `pythonpath = ["src"]` makes the source packages importable (drop it if the project has no `src/` directory).

- `tests/` sits at the repository root and mirrors the project's Python layout one-to-one, whatever the directories are called: one test file per source file, at the same relative path, with a `test_` prefix. If the Python code lives under `src/`, mirror the contents of `src/` and leave out the `src` segment. Python outside `src/` mirrors at its own path. Examples:
  - `src/python_common/stores/job_store.py` → `tests/python_common/stores/test_job_store.py`
  - `src/lambdas/slack_summary/handler.py` → `tests/lambdas/slack_summary/test_handler.py`
  - `scripts/build.py` → `tests/scripts/test_build.py`
- Contract and test-vector tests are unit tests of the module they exercise, so they go in that module's mirrored test file.
- Do not add `__init__.py` files under `tests/`. Import helpers by their full path from the repository root, e.g. `from tests.lambdas.slack_summary.helpers import build_event`; a bare `from helpers import ...` does not work in `importlib` mode. (Tested with pytest 9.2: in the default import mode, two `test_handler.py` files in different directories fail with "import file mismatch" unless the directories are packages. `importlib` mode avoids that without `__init__.py` files. Tests importing from a `helpers.py` need the full path because a bare import fails in `importlib` mode.)
- Tests have no docstrings or type hints, so if ruff's `D` (pydocstyle) or `ANN` (annotations) rules are enabled, add them to `[tool.ruff.lint.per-file-ignores]` for `"tests/**"`. Do the same for `S101` (assert) if bandit rules are enabled.

## 3. Structure of a test file

In this order:

1. Imports.
2. Module-level fixtures and mocks shared by more than one test class.
3. Test classes, in the order their targets appear in the source file.

Rules:

- **One test class per unit**: each class maps to exactly one function, class or method. Name it `Test<Name>` in PascalCase: `parse_summary` → `TestParseSummary`. For a method, prefix the owning class so names never collide: `JobStore.transition` → `TestJobStoreTransition`. The class-level test class (`TestJobStore`) covers construction and class-level behaviour (`__init__`, validation, class attributes); every other method gets its own test class.
- **Order tests within a class by the line of code they exercise**, top of the function first. Parametrised cases follow the same order.
- **Test method names** are concise and descriptive of the behaviour: `test_returns_none_for_unknown_channel`, `test_raises_on_expired_timestamp`.
- **No docstrings, type hints or comments** anywhere in test files.
- **One assertion per test.** Use `@pytest.mark.parametrize` (with `ids=`) to cover several inputs of the same shape instead of adding assertions. A `pytest.raises` block or a single `assert_called_once_with` counts as the one assertion.

## 4. Isolation and mocking

- A test covers **only the logic of the unit under test**. Never test secondary functions, dependencies or integrations. Mock every collaborator the unit calls: other functions in the same module, other modules, third-party libraries, network, filesystem, environment, clock and randomness.
- **Patch at the point of use, never where the thing is defined.** Patch the name in the namespace of the module under test:
  - `python_common/clients/slack.py` does `from python_common.config import get_config` → patch `"python_common.clients.slack.get_config"`, not `"python_common.config.get_config"`.
  - `python_common/slack_verify.py` does `import time` → patch `"python_common.slack_verify.time"`, not `"time.time"`.
- Place each mock or fixture at the **narrowest scope that covers every user of it**:

  | Used by | Put it |
  |---|---|
  | One test | Inside that test |
  | Several tests in one class | A fixture method on that class |
  | Several classes in one file | A module-level fixture at the top of the file |
  | Several files in one directory | `conftest.py` in that directory |
  | Files in several directories | `conftest.py` in the lowest directory that contains all of them |

- Use `autouse=True` only when every test in the fixture's scope needs it; otherwise request the fixture by name.
- **Helper functions** (builders, factories, payload makers that are not fixtures) go in a separate `helpers.py` next to the test files that use them. If several directories use a helper, move it to a `helpers.py` in the lowest directory that contains all of them, following the same rule as for conftest. Never define helpers inside a test file.

## 5. Coverage

Every function, class and method is fully unit-tested. For each unit, cover:

- **Happy path for every type of input** it accepts (each accepted type, each enum member, each optional argument given and omitted).
- **Every edge case**: empty, `None`, zero, negative, single item vs many, duplicates, unusual but valid values.
- **Every error path**: each `raise`, each `except` branch, each early return, each validation failure.
- **Every branch** of every `if`/`match`/loop (zero, one and many iterations).

Specific rules:

- **Boundaries**: test just below, exactly on, and just above every boundary (e.g. a 300-second limit → 299, 300, 301).
- **Elapsed time**: when the behaviour depends on real elapsed time that is not mocked, test either side of the boundary with a buffer large enough that the test's own runtime cannot flip the result (e.g. a 5-second timeout → 4 s and 6 s, not 4.999 s and 5.001 s). Prefer mocking the clock at point of use where the code allows it, then test exactly on the boundary too.
- **Strings**: test the base value, empty string, different case, leading whitespace and trailing whitespace. Drop a variant only if it makes no sense for that function.

## 6. Refactoring existing tests

- Apply every rule above. Move mocks, fixtures and helpers to their correct scope, split multi-assertion tests (parametrise where possible), reorder classes and tests, and remove docstrings, comments and type hints.
- Coverage must never go down. Keep every behaviour previously tested unless it contradicts the intended behaviour; if it does, ask the user.
- Never change an expected value to match the code without confirming the intended behaviour.

## 7. Workflow

1. Read the unit under test and its spec. List its inputs, branches, boundaries and error paths before writing any test.
2. Resolve any unknown intended behaviour with the user.
3. Write the tests following sections 2-5.
4. Run `pytest` on the new test file, then the full suite.
5. A failing test means the code or the test design is wrong against intended behaviour - fix the code or report the defect. Never weaken the test.
6. Walk the checklist below.

## 8. Checklist

- [ ] Test file mirrors the source path; no `__init__.py` under `tests/`; helpers imported by full path.
- [ ] One `Test<Name>` class per function, class and method, ordered as in the source file.
- [ ] Tests within each class ordered by the line they exercise.
- [ ] One assertion per test; parametrised where inputs share a shape.
- [ ] No docstrings, type hints or comments.
- [ ] Every collaborator mocked, patched at point of use.
- [ ] Mocks, fixtures and helpers at the narrowest shared scope; helpers in `helpers.py`.
- [ ] Happy path per input type, edge cases, error paths and every branch covered.
- [ ] Boundaries tested below, on and above; elapsed-time boundaries tested either side with a buffer.
- [ ] Strings tested for base, empty, case, leading and trailing whitespace.
- [ ] Expected values come from intended behaviour, not the current code.

## Example

Source `python_common/slack_verify.py` defines, in order, `compute_signature`, `is_fresh` and `verify_signature`. `verify_signature` calls the other two.

Intended behaviour: the signature is `"v0=" + HMAC-SHA256(secret, "v0:<timestamp>:<body>")`.

```python
from unittest.mock import ANY

import pytest

from python_common.slack_verify import compute_signature, is_fresh, verify_signature


@pytest.fixture
def secret():
    return "signing-secret"


class TestComputeSignature:
    @pytest.fixture
    def hmac_mock(self, mocker):
        hmac_mock = mocker.patch("python_common.slack_verify.hmac")
        hmac_mock.new.return_value.hexdigest.return_value = "abc"
        return hmac_mock

    @pytest.mark.parametrize(
        "body",
        [b"body", b"", b"BODY", b" body", b"body "],
        ids=["base", "empty", "different_case", "leading_space", "trailing_space"],
    )
    def test_signs_versioned_base_string(self, secret, hmac_mock, body):
        compute_signature(secret, 1_000, body)
        hmac_mock.new.assert_called_once_with(b"signing-secret", b"v0:1000:" + body, ANY)

    def test_returns_v0_prefixed_digest(self, secret, hmac_mock):
        assert compute_signature(secret, 1_000, b"body") == "v0=abc"


class TestIsFresh:
    @pytest.fixture
    def clock(self, mocker):
        return mocker.patch("python_common.slack_verify.time")

    @pytest.mark.parametrize(
        ("age", "expected"),
        [(299, True), (300, True), (301, False)],
        ids=["below_limit", "on_limit", "above_limit"],
    )
    def test_age_against_limit(self, clock, age, expected):
        clock.time.return_value = 1_000 + age
        assert is_fresh(1_000) is expected


class TestVerifySignature:
    @pytest.fixture
    def fresh(self, mocker):
        return mocker.patch("python_common.slack_verify.is_fresh", return_value=True)

    @pytest.fixture
    def signature(self, mocker):
        return mocker.patch("python_common.slack_verify.compute_signature", return_value="v0=abc")

    def test_returns_false_when_stale(self, secret, fresh, signature):
        fresh.return_value = False
        assert verify_signature(secret, 1_000, b"body", "v0=abc") is False

    def test_returns_true_when_signature_matches(self, secret, fresh, signature):
        assert verify_signature(secret, 1_000, b"body", "v0=abc") is True

    def test_returns_false_when_signature_differs(self, secret, fresh, signature):
        assert verify_signature(secret, 1_000, b"body", "v0=xyz") is False
```

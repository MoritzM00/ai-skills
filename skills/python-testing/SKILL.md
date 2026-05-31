---
name: python-testing
description: Python testing strategies using pytest, TDD methodology, fixtures, mocking, parametrization, async tests, property-based tests, and coverage policy.
---

# Python Testing Patterns

Use this skill when writing, reviewing, or improving Python tests. Prefer pytest
for new test suites unless the project already standardizes on `unittest`.

## Testing Approach

- Prefer TDD for new behavior and bug fixes: write a failing test, make it pass
  with the smallest useful change, then refactor with tests green.
- For legacy code, start with characterization tests around current behavior
  before changing internals.
- Test observable behavior and public contracts. Avoid assertions that only lock
  down incidental implementation details.
- Keep tests independent. No test should require another test to have run first.
- Use integration tests for module boundaries, persistence, I/O, permissions, and
  serialization. Use unit tests for pure logic and branching behavior.

## What To Test

Prioritize tests around commitments the code makes to callers and users:

- Happy paths for public functions, commands, endpoints, and workflows.
- Boundary cases: empty inputs, `None`, min/max values, duplicates, ordering,
  time zones, encodings, and malformed input.
- Failure behavior: validation errors, exceptions, retries, timeouts, permission
  denial, partial writes, rollback, and cleanup.
- State changes and side effects: database writes, files, cache updates,
  outbound calls, emitted events, and metrics that are part of the contract.
- Integration contracts: serialization, API schemas, database migrations,
  authentication, authorization, configuration, and environment handling.
- Regressions: reproduce the bug with the smallest failing test before fixing it.

Avoid implementation-detail tests:

- Do not assert private helper calls, exact loop structure, temporary variables,
  or intermediate data shapes unless they are public contracts.
- Do not assert mock call counts just to prove code took a particular path.
  Assert the final behavior instead; assert collaborator calls only at real
  boundaries such as external APIs, repositories, queues, or payment clients.
- Do not snapshot broad output. Snapshot only stable user-facing output, and keep
  snapshots small enough to review.
- If a behavior-preserving refactor breaks a test, the test is probably
  over-specified.

## Project Setup

Prefer a `src/` layout for installable packages and pytest's importlib import
mode for new projects.

Typical layout: `src/mypackage/` for package code, `tests/conftest.py` for
shared fixtures, and `tests/unit/`, `tests/integration/`, or `tests/e2e/` when
those categories are useful. Avoid `tests/__init__.py` unless tests must be
importable as packages or you need duplicate test module names.

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "--strict-markers",
    "--import-mode=importlib",
]
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]
filterwarnings = [
    "error::DeprecationWarning:mypackage",
]
```

Keep warning output visible by default. Do not add `--disable-warnings` as a
standard option; filter known third-party noise deliberately.

## Assertions

Use plain `assert` statements. pytest rewrites assertions to show useful failure
details.

```python
import re

import pytest


def test_normalize_email():
    assert normalize_email(" Alice@Example.COM ") == "alice@example.com"


def test_rejects_bad_email():
    with pytest.raises(ValueError, match="invalid email"):
        normalize_email("not-an-email")


def test_literal_exception_text():
    with pytest.raises(ValueError, match=re.escape("invalid value [x]")):
        parse_value("[x]")


def test_exception_details():
    with pytest.raises(CustomError) as exc_info:
        raise CustomError("bad request", code=400)

    assert exc_info.value.code == 400
```

`pytest.raises(..., match=...)` uses a regular expression via `re.search()`.
Use `re.escape()` for literal messages containing regex metacharacters. On
pytest 8.4+, `pytest.raises(..., check=...)` can validate exception attributes:

```python
def test_exception_check():
    with pytest.raises(CustomError, check=lambda exc: exc.code == 400):
        raise CustomError("bad request", code=400)
```

## Fixtures

Use fixtures for setup, teardown, and reusable object construction. Prefer
returning simple values; use `yield` only when teardown is required.

```python
import pytest


@pytest.fixture
def user():
    return User(id=1, name="Alice")


@pytest.fixture
def database():
    db = Database(":memory:")
    db.create_tables()
    yield db
    db.close()


def test_user_lookup(database, user):
    database.insert(user)
    assert database.get_user(user.id) == user
```

Fixture scopes are `function` by default. Use broader scopes only for resources
that are safe to share:

```python
@pytest.fixture(scope="session")
def expensive_resource():
    resource = ExpensiveResource()
    yield resource
    resource.cleanup()
```

Use built-in temporary path fixtures for filesystem work:

```python
def test_write_report(tmp_path):
    report_path = tmp_path / "report.txt"
    write_report(report_path, ["a", "b"])

    assert report_path.read_text() == "a\nb\n"
```

Prefer `tmp_path` for new tests. Use `tmpdir` only when maintaining code that
expects `py.path.local`.

Use `autouse=True` sparingly. Hidden setup makes tests harder to understand:

```python
@pytest.fixture(autouse=True)
def reset_config():
    Config.reset()
    yield
    Config.cleanup()
```

## Parametrization and Markers

Use parametrization for examples that exercise the same behavior:

```python
import pytest


@pytest.mark.parametrize(
    ("text", "expected"),
    [
        ("hello", "HELLO"),
        ("world", "WORLD"),
        ("PyThOn", "PYTHON"),
    ],
    ids=["lowercase", "another-lowercase", "mixed-case"],
)
def test_uppercase(text, expected):
    assert text.upper() == expected
```

Parameterized fixtures are useful for backend matrices:

```python
@pytest.fixture(params=["sqlite", "postgresql"], ids=["sqlite", "postgresql"])
def db(request):
    database = Database.for_backend(request.param)
    yield database
    database.close()
```

Register custom markers in pytest config and select them with `-m`:

```python
@pytest.mark.integration
def test_api_integration(client):
    assert client.get("/health").status_code == 200
```

```bash
pytest -m "not slow"
pytest -m "integration and not slow"
pytest -k "user and login"
```

## Mocking and Patching

Patch names where the system under test looks them up, not necessarily where the
object was originally defined. If `myapp.service` does
`from mypackage import external_api_call`, patch
`myapp.service.external_api_call`.

Prefer dependency injection, fakes, or small in-memory implementations when they
are clearer than patching. When patching, use `autospec=True` or `spec_set=True`
to catch misspelled methods and signature drift.

```python
from unittest.mock import patch


@patch("myapp.service.external_api_call", autospec=True)
def test_fetch_status(api_call_mock):
    api_call_mock.return_value = {"status": "ok"}

    assert fetch_status() == "ok"

    api_call_mock.assert_called_once_with("/status")
```

When autospeccing methods on a class, remember that the instance is passed as
`self`:

```python
from unittest.mock import patch


@patch("myapp.repository.Database.connect", autospec=True)
def test_database_connection(connect_mock):
    db = Database()
    db.connect("localhost")

    connect_mock.assert_called_once_with(db, "localhost")
```

Autospec class patches should exercise code that instantiates the class:

```python
from unittest.mock import patch


@patch("myapp.service.DBConnection", autospec=True)
def test_load_users(db_connection_cls):
    db = db_connection_cls.return_value
    db.query.return_value = [{"id": 1}]

    assert load_users() == [{"id": 1}]

    db_connection_cls.assert_called_once_with("read-only")
    db.query.assert_called_once_with("SELECT * FROM users")
```

For explicit dependencies, an injected mock is often simpler:

```python
from unittest.mock import Mock


def test_create_user():
    repo = Mock(spec_set=UserRepository)
    repo.save.return_value = User(id=1, name="Alice")

    service = UserService(repo)
    user = service.create_user(name="Alice")

    assert user.name == "Alice"
    repo.save.assert_called_once()
```

Use `mock_open` only when the test should avoid real filesystem I/O. Prefer
`tmp_path` when real file behavior matters.

```python
from unittest.mock import mock_open, patch


@patch("builtins.open", new_callable=mock_open, read_data="file content")
def test_file_reading(mock_file):
    assert read_file("test.txt") == "file content"
    mock_file.assert_called_once_with("test.txt", "r")
```

## Async Tests

Use pytest-asyncio for asyncio tests. In strict mode, async fixtures should use
`@pytest_asyncio.fixture`. If the project only uses asyncio, set
`asyncio_mode = auto` for the simplest configuration. Keep strict mode when
multiple async test plugins, such as trio and asyncio, need to coexist.

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

```python
import pytest
import pytest_asyncio


@pytest_asyncio.fixture
async def async_client():
    app = create_app()
    async with app.test_client() as client:
        yield client


@pytest.mark.asyncio
async def test_api_endpoint(async_client):
    response = await async_client.get("/api/data")
    assert response.status_code == 200
```

`patch()` creates an `AsyncMock` automatically when the patched target is an
async function:

```python
import pytest
from unittest.mock import patch


@pytest.mark.asyncio
@patch("myapp.service.async_api_call")
async def test_async_mock(api_call_mock):
    api_call_mock.return_value = {"status": "ok"}

    assert await my_async_function() == {"status": "ok"}

    api_call_mock.assert_awaited_once()
```

## Property-Based Testing

Use Hypothesis for pure logic with many valid inputs: parsers, serializers,
normalizers, numeric code, validators, and round-trip conversions. State
invariants instead of listing only hand-picked cases.

```python
from hypothesis import given, strategies as st


@given(st.lists(st.integers()))
def test_sort_is_idempotent(values):
    ordered = sort_values(values)

    assert sort_values(ordered) == ordered
    assert sorted(ordered) == sorted(values)
```

Keep generated examples small unless broad exploration is necessary. When a
failure is found, Hypothesis shrinks it; add a normal regression test when the
minimal case documents an important bug.

## Coverage

Set coverage thresholds as project policy, not as a universal rule. A common
baseline is 80%+ line coverage; critical paths deserve focused tests and often
near-complete line and branch coverage. Do not chase a percentage instead of
useful assertions.

Use coverage.py directly, or pytest-cov when the project wants coverage
integrated into pytest. `pytest --cov` requires the pytest-cov plugin.

```toml
[tool.coverage.run]
branch = true
source = ["mypackage"]

[tool.coverage.report]
show_missing = true
fail_under = 80
```

```bash
coverage run -m pytest
coverage report --show-missing
coverage html

pytest --cov=mypackage --cov-branch --cov-report=term-missing --cov-report=html
pytest --cov=mypackage --cov-fail-under=80
```

## Database Tests

For SQLAlchemy 2.x ORM tests where application code may call `Session.commit()`,
bind the session to an external transaction and use
`join_transaction_mode="create_savepoint"`. The outer transaction is rolled back
after the test.

```python
import pytest
from sqlalchemy.orm import Session


@pytest.fixture
def db_session(engine):
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection, join_transaction_mode="create_savepoint")

    try:
        yield session
    finally:
        session.close()
        transaction.rollback()
        connection.close()


def test_create_user(db_session):
    user = User(name="Alice", email="alice@example.com")
    db_session.add(user)
    db_session.commit()

    retrieved = db_session.query(User).filter_by(name="Alice").first()
    assert retrieved.email == "alice@example.com"
```

This recipe relies on reliable SAVEPOINT support from the database and driver.
If SAVEPOINTs are unreliable, avoid commits in tests or recreate isolated test
databases instead.

## Web Endpoint Tests

Use the framework's test client rather than live network calls for app-local
endpoints. Example for Flask-style clients:

```python
import pytest


@pytest.fixture
def client():
    app = create_app(testing=True)
    with app.test_client() as client:
        yield client


def test_create_user(client):
    response = client.post(
        "/api/users",
        json={"name": "Alice", "email": "alice@example.com"},
    )

    assert response.status_code == 201
    assert response.json["name"] == "Alice"
```

## Review Checklist

- Does each test assert behavior a user or caller depends on?
- Would a behavior-preserving refactor keep this test passing?
- Are fixtures explicit enough to make setup visible?
- Are mocks limited to boundaries and patched where the code under test looks up
  the object?
- Are slow, integration, or external-resource tests marked?
- Are warnings visible unless deliberately filtered?
- Do coverage gates reflect risk rather than vanity percentages?
- Would `tmp_path`, parametrization, or Hypothesis make the test simpler or more
  exhaustive?

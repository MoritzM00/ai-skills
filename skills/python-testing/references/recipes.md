# Python Testing Recipes

Use these recipes only when the task needs the specific integration pattern.
Keep the main skill focused on testing judgment that applies across projects.

## File mocks

Use `mock_open` only when the test should avoid real filesystem I/O. Prefer
`tmp_path` when real file behavior matters.

```python
from unittest.mock import mock_open, patch


@patch("builtins.open", new_callable=mock_open, read_data="file content")
def test_file_reading(mock_file):
    assert read_file("test.txt") == "file content"
    mock_file.assert_called_once_with("test.txt", "r")
```

## SQLAlchemy savepoint sessions

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

## Web endpoint clients

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

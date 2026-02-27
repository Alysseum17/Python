# Day 14: Testing & Code Quality — pytest

## 🎯 Goal

Learn professional testing in Python with `pytest`. Testing is non-negotiable in Data Engineering — pipelines run on schedules, process millions of records, and break silently if untested. You'll learn pytest from basics to advanced patterns: fixtures, parametrize, mocking, testing exceptions, test organization, and integration testing.

Coming from Java (JUnit) and JS (Jest), the concepts are familiar. pytest is more Pythonic — less boilerplate, more magic.

---

## 1. Why pytest (not unittest)

```
Python has two testing frameworks:
  unittest — built-in, Java-style (like JUnit: classes, setUp, tearDown)
  pytest   — third-party, Pythonic (like Jest: functions, fixtures, magic)

pytest wins because:
  ✅ Plain assert (no assertEqual, assertIn, etc.)
  ✅ Fixtures (cleaner than setUp/tearDown)
  ✅ Parametrize (test many inputs with one function)
  ✅ Better error messages (shows actual values)
  ✅ Huge plugin ecosystem
  ✅ Runs unittest tests too (backward compatible)
  ✅ Industry standard — used by almost every Python project

unittest is like JUnit 4. pytest is like JUnit 5 + Mockito + AssertJ combined.
```

---

## 2. First Tests — The Basics

### Write your first test

```python
# test_math.py

def add(a: int, b: int) -> int:
    return a + b

def test_add_positive():
    assert add(2, 3) == 5

def test_add_negative():
    assert add(-1, -1) == -2

def test_add_zero():
    assert add(0, 0) == 0
    assert add(5, 0) == 5
```

```bash
# Run tests
pytest test_math.py -v

# Output:
# test_math.py::test_add_positive PASSED
# test_math.py::test_add_negative PASSED
# test_math.py::test_add_zero PASSED
# =================== 3 passed in 0.01s ====================
```

### pytest conventions

```python
# File naming:   test_*.py  or  *_test.py
# Function naming: test_*
# Class naming:    Test*  (no __init__)

# pytest auto-discovers tests by these naming conventions
# Just run 'pytest' in project root — it finds everything

# Project structure:
# myproject/
# ├── src/
# │   └── myproject/
# │       ├── calculator.py
# │       └── utils.py
# └── tests/
#     ├── test_calculator.py
#     └── test_utils.py
```

### Rich assertions — just use `assert`

```python
# pytest gives AMAZING error messages with plain assert

def test_list_equality():
    expected = [1, 2, 3, 4, 5]
    actual = [1, 2, 3, 4, 6]
    assert actual == expected
    
    # Error output:
    # AssertionError: assert [1, 2, 3, 4, 6] == [1, 2, 3, 4, 5]
    #   At index 4 diff: 6 != 5

def test_dict_equality():
    expected = {"name": "Alice", "age": 25, "city": "Kyiv"}
    actual = {"name": "Alice", "age": 26, "city": "London"}
    assert actual == expected
    
    # Error output shows EXACTLY which keys differ:
    # E   AssertionError: assert {...} == {...}
    # E     Differing items:
    # E     {'age': 26} != {'age': 25}
    # E     {'city': 'London'} != {'city': 'Kyiv'}

def test_string_containment():
    message = "Hello, World!"
    assert "World" in message
    assert message.startswith("Hello")
    assert len(message) == 13

# Compare with JUnit:
#   assertEquals(expected, actual)      →  assert actual == expected
#   assertTrue(condition)               →  assert condition
#   assertIn(item, collection)          →  assert item in collection
#   assertIsNone(value)                 →  assert value is None
#   assertIsInstance(obj, cls)          →  assert isinstance(obj, cls)

# Compare with Jest:
#   expect(actual).toBe(expected)       →  assert actual == expected
#   expect(actual).toContain(item)      →  assert item in actual
#   expect(actual).toBeNull()           →  assert actual is None
```

---

## 3. Testing Exceptions

```python
import pytest

def divide(a: int, b: int) -> float:
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

# Test that exception IS raised
def test_divide_by_zero():
    with pytest.raises(ValueError):
        divide(10, 0)

# Test exception message
def test_divide_by_zero_message():
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(10, 0)

# Capture exception for further inspection
def test_divide_by_zero_details():
    with pytest.raises(ValueError) as exc_info:
        divide(10, 0)
    
    assert "zero" in str(exc_info.value)
    assert exc_info.type is ValueError

# Test that exception is NOT raised
def test_divide_normal():
    result = divide(10, 2)  # should not raise
    assert result == 5.0

# Compare:
# JUnit:  assertThrows(ValueError.class, () -> divide(10, 0))
# Jest:   expect(() => divide(10, 0)).toThrow("Cannot divide by zero")
```

---

## 4. Fixtures — Setup & Teardown

Fixtures replace setUp/tearDown. They're more flexible: dependency injection, scoping, composability.

### Basic fixtures

```python
import pytest

@pytest.fixture
def sample_users():
    """Provide sample user data for tests."""
    return [
        {"id": 1, "name": "Alice", "age": 25, "active": True},
        {"id": 2, "name": "Bob", "age": 30, "active": False},
        {"id": 3, "name": "Charlie", "age": 35, "active": True},
    ]

@pytest.fixture
def empty_list():
    return []

# Use fixtures by putting them as test function parameters
# pytest injects them automatically (dependency injection!)

def test_user_count(sample_users):
    assert len(sample_users) == 3

def test_active_users(sample_users):
    active = [u for u in sample_users if u["active"]]
    assert len(active) == 2

def test_empty(empty_list):
    assert len(empty_list) == 0
```

### Fixtures with setup AND teardown

```python
import pytest
import sqlite3
from pathlib import Path

@pytest.fixture
def database():
    """Create a test database, clean up after test."""
    db_path = Path("test.db")
    conn = sqlite3.connect(str(db_path))
    cursor = conn.cursor()
    cursor.execute("CREATE TABLE users (id INTEGER, name TEXT, age INTEGER)")
    cursor.execute("INSERT INTO users VALUES (1, 'Alice', 25)")
    cursor.execute("INSERT INTO users VALUES (2, 'Bob', 30)")
    conn.commit()
    
    yield conn  # ← this is where the test runs
    
    # Everything after yield = teardown (like finally)
    conn.close()
    db_path.unlink(missing_ok=True)

def test_query_users(database):
    cursor = database.cursor()
    cursor.execute("SELECT COUNT(*) FROM users")
    count = cursor.fetchone()[0]
    assert count == 2

def test_query_by_name(database):
    cursor = database.cursor()
    cursor.execute("SELECT age FROM users WHERE name = ?", ("Alice",))
    age = cursor.fetchone()[0]
    assert age == 25

# Compare with JUnit:
# @BeforeEach void setUp() { ... }
# @AfterEach void tearDown() { ... }
# pytest: yield separates setup from teardown in ONE function
```

### Fixture scopes

```python
@pytest.fixture(scope="function")  # default — new for each test
def per_test_fixture():
    return create_resource()

@pytest.fixture(scope="class")     # shared across tests in a class
def per_class_fixture():
    return create_expensive_resource()

@pytest.fixture(scope="module")    # shared across entire test file
def per_module_fixture():
    return create_very_expensive_resource()

@pytest.fixture(scope="session")   # shared across ALL test files
def per_session_fixture():
    return create_global_resource()

# Use wider scopes for expensive resources (DB connections, API clients)
# Use function scope (default) when tests need isolation
```

### Composing fixtures

```python
@pytest.fixture
def db_connection():
    conn = sqlite3.connect(":memory:")
    yield conn
    conn.close()

@pytest.fixture
def db_with_schema(db_connection):
    """Depends on db_connection fixture."""
    cursor = db_connection.cursor()
    cursor.execute("CREATE TABLE users (id INTEGER, name TEXT)")
    db_connection.commit()
    return db_connection

@pytest.fixture
def db_with_data(db_with_schema):
    """Depends on db_with_schema, which depends on db_connection."""
    cursor = db_with_schema.cursor()
    cursor.executemany(
        "INSERT INTO users VALUES (?, ?)",
        [(1, "Alice"), (2, "Bob"), (3, "Charlie")],
    )
    db_with_schema.commit()
    return db_with_schema

# Fixture chain: db_connection → db_with_schema → db_with_data
# pytest resolves dependencies automatically

def test_user_count(db_with_data):
    cursor = db_with_data.cursor()
    cursor.execute("SELECT COUNT(*) FROM users")
    assert cursor.fetchone()[0] == 3
```

### `conftest.py` — shared fixtures

```python
# tests/conftest.py — automatically loaded by pytest
# Fixtures here are available to ALL test files in the directory

import pytest

@pytest.fixture
def api_client():
    """Shared API client fixture."""
    client = create_test_client()
    yield client
    client.close()

@pytest.fixture
def sample_config():
    """Shared config fixture."""
    return {
        "database_url": "sqlite:///:memory:",
        "debug": True,
        "batch_size": 100,
    }

# No need to import! pytest finds conftest.py automatically.
# tests/test_api.py can just use:
# def test_something(api_client):
#     ...
```

---

## 5. Parametrize — Test Many Inputs

### Basic parametrize

```python
import pytest

def is_palindrome(s: str) -> bool:
    s = s.lower().replace(" ", "")
    return s == s[::-1]

# Instead of writing 5 separate test functions:
@pytest.mark.parametrize("input_str, expected", [
    ("racecar", True),
    ("hello", False),
    ("A man a plan a canal Panama", True),
    ("", True),
    ("a", True),
    ("ab", False),
    ("Was it a car or a cat I saw", True),
])
def test_is_palindrome(input_str, expected):
    assert is_palindrome(input_str) == expected

# Output:
# test_palindrome.py::test_is_palindrome[racecar-True] PASSED
# test_palindrome.py::test_is_palindrome[hello-False] PASSED
# test_palindrome.py::test_is_palindrome[A man a plan...-True] PASSED
# ...

# Compare with JUnit @ParameterizedTest + @ValueSource
# Compare with Jest test.each([...])
```

### Parametrize with IDs

```python
@pytest.mark.parametrize("value, expected", [
    pytest.param(0, "zero", id="zero"),
    pytest.param(1, "one", id="positive"),
    pytest.param(-1, "negative one", id="negative"),
    pytest.param(999, "nine nine nine", id="large"),
])
def test_number_to_words(value, expected):
    assert number_to_words(value) == expected

# Output:
# test::test_number_to_words[zero] PASSED
# test::test_number_to_words[positive] PASSED
# test::test_number_to_words[negative] PASSED
# test::test_number_to_words[large] PASSED
```

### Multiple parametrize (cartesian product)

```python
@pytest.mark.parametrize("x", [1, 2, 3])
@pytest.mark.parametrize("y", [10, 20])
def test_multiply(x, y):
    result = x * y
    assert result == x * y

# Runs 6 tests: (1,10), (1,20), (2,10), (2,20), (3,10), (3,20)
```

### Parametrize with fixtures

```python
@pytest.fixture(params=["sqlite", "postgres", "mysql"])
def db_engine(request):
    """Test against multiple database engines."""
    engine_type = request.param
    engine = create_engine(engine_type)
    yield engine
    engine.dispose()

def test_insert_and_query(db_engine):
    # This test runs 3 times — once per database engine
    db_engine.execute("INSERT INTO test VALUES (1, 'Alice')")
    result = db_engine.execute("SELECT * FROM test")
    assert len(result) == 1
```

---

## 6. Mocking — Isolate Your Tests

### `unittest.mock` — Python's built-in mocking (used with pytest)

```python
from unittest.mock import Mock, patch, MagicMock
import pytest

# ── Basic Mock ──
def test_mock_basics():
    # Create a mock object
    mock_db = Mock()
    
    # Mock returns Mock for any attribute/method call
    mock_db.query("SELECT 1")
    mock_db.connect()
    
    # Verify it was called
    mock_db.query.assert_called_once_with("SELECT 1")
    mock_db.connect.assert_called_once()
    
    # Configure return value
    mock_db.query.return_value = [{"id": 1, "name": "Alice"}]
    result = mock_db.query("SELECT * FROM users")
    assert result == [{"id": 1, "name": "Alice"}]
```

### `patch` — replace real objects with mocks

```python
# src/myproject/service.py
import requests

def fetch_user(user_id: int) -> dict:
    response = requests.get(f"https://api.example.com/users/{user_id}")
    response.raise_for_status()
    return response.json()

# tests/test_service.py
from unittest.mock import patch, Mock

def test_fetch_user_success():
    # patch replaces requests.get with a mock during the test
    with patch("myproject.service.requests.get") as mock_get:
        # Configure the mock
        mock_response = Mock()
        mock_response.json.return_value = {"id": 1, "name": "Alice"}
        mock_response.raise_for_status.return_value = None
        mock_get.return_value = mock_response
        
        # Call the real function — it uses our mock instead of requests
        result = fetch_user(1)
        
        # Verify
        assert result == {"id": 1, "name": "Alice"}
        mock_get.assert_called_once_with("https://api.example.com/users/1")

# ⚠️ IMPORTANT: patch the object WHERE IT'S USED, not where it's defined
# ✅ patch("myproject.service.requests.get")    — where requests is used
# ❌ patch("requests.get")                      — where requests is defined
```

### `patch` as decorator

```python
@patch("myproject.service.requests.get")
def test_fetch_user(mock_get):
    """mock_get is automatically injected as parameter."""
    mock_get.return_value.json.return_value = {"id": 1, "name": "Alice"}
    mock_get.return_value.raise_for_status.return_value = None
    
    result = fetch_user(1)
    assert result["name"] == "Alice"

# Multiple patches — order is reversed (bottom patch = first parameter)
@patch("myproject.service.requests.post")
@patch("myproject.service.requests.get")
def test_something(mock_get, mock_post):
    pass
```

### Mocking with `side_effect`

```python
from unittest.mock import patch, Mock

# side_effect: make mock raise an exception
def test_fetch_user_network_error():
    with patch("myproject.service.requests.get") as mock_get:
        mock_get.side_effect = ConnectionError("Network down")
        
        with pytest.raises(ConnectionError):
            fetch_user(1)

# side_effect: different return values for sequential calls
def test_retry_logic():
    with patch("myproject.service.requests.get") as mock_get:
        mock_get.side_effect = [
            ConnectionError("Timeout"),   # 1st call fails
            ConnectionError("Timeout"),   # 2nd call fails
            Mock(json=lambda: {"ok": True}, raise_for_status=lambda: None),  # 3rd succeeds
        ]
        
        result = fetch_with_retry(url="http://...", max_retries=3)
        assert result == {"ok": True}
        assert mock_get.call_count == 3

# side_effect: custom function
def test_with_custom_side_effect():
    def fake_get(url):
        if "users" in url:
            return Mock(json=lambda: [{"name": "Alice"}])
        return Mock(json=lambda: [])
    
    with patch("myproject.service.requests.get", side_effect=fake_get):
        users = fetch_user(1)
```

### pytest-mock — cleaner mocking

```python
# pip install pytest-mock

def test_fetch_user(mocker):
    """mocker fixture from pytest-mock — cleaner than unittest.mock."""
    mock_get = mocker.patch("myproject.service.requests.get")
    mock_get.return_value.json.return_value = {"id": 1, "name": "Alice"}
    mock_get.return_value.raise_for_status.return_value = None
    
    result = fetch_user(1)
    assert result["name"] == "Alice"

# mocker.patch is the same as unittest.mock.patch,
# but auto-cleanup is handled by the fixture (no context manager needed)
```

---

## 7. Testing Patterns for Data Engineering

### Test data validation

```python
import pytest

# Assume we have a Pydantic-like validator (or manual validation)
def validate_event(event: dict) -> dict:
    """Validate an analytics event."""
    required = {"event_id", "user_id", "event_type", "timestamp"}
    missing = required - set(event.keys())
    if missing:
        raise ValueError(f"Missing fields: {missing}")
    if not isinstance(event["user_id"], int) or event["user_id"] <= 0:
        raise ValueError("user_id must be a positive integer")
    if event["event_type"] not in ("click", "view", "purchase"):
        raise ValueError(f"Invalid event_type: {event['event_type']}")
    return event


class TestEventValidation:
    """Group related tests in a class."""

    def test_valid_event(self):
        event = {
            "event_id": "e1",
            "user_id": 42,
            "event_type": "click",
            "timestamp": "2025-02-14T12:00:00",
        }
        result = validate_event(event)
        assert result == event

    @pytest.mark.parametrize("missing_field", [
        "event_id", "user_id", "event_type", "timestamp",
    ])
    def test_missing_required_field(self, missing_field):
        event = {
            "event_id": "e1",
            "user_id": 42,
            "event_type": "click",
            "timestamp": "2025-02-14T12:00:00",
        }
        del event[missing_field]
        with pytest.raises(ValueError, match="Missing fields"):
            validate_event(event)

    @pytest.mark.parametrize("bad_user_id", [0, -1, "abc", None])
    def test_invalid_user_id(self, bad_user_id):
        event = {
            "event_id": "e1",
            "user_id": bad_user_id,
            "event_type": "click",
            "timestamp": "2025-02-14T12:00:00",
        }
        with pytest.raises(ValueError):
            validate_event(event)

    @pytest.mark.parametrize("bad_type", ["delete", "", "CLICK", "unknown"])
    def test_invalid_event_type(self, bad_type):
        event = {
            "event_id": "e1",
            "user_id": 42,
            "event_type": bad_type,
            "timestamp": "2025-02-14T12:00:00",
        }
        with pytest.raises(ValueError, match="Invalid event_type"):
            validate_event(event)
```

### Test data transformations

```python
import pytest

def transform_user(raw: dict) -> dict:
    """Transform raw user data to standardized format."""
    return {
        "id": raw["id"],
        "full_name": f"{raw['first_name']} {raw['last_name']}".strip(),
        "email": raw["email"].lower().strip(),
        "age": int(raw["age"]),
        "is_active": raw.get("status", "inactive") == "active",
    }

class TestTransformUser:
    
    def test_basic_transform(self):
        raw = {
            "id": 1,
            "first_name": "Alice",
            "last_name": "Smith",
            "email": "  Alice@Example.COM  ",
            "age": "25",
            "status": "active",
        }
        result = transform_user(raw)
        
        assert result["id"] == 1
        assert result["full_name"] == "Alice Smith"
        assert result["email"] == "alice@example.com"
        assert result["age"] == 25
        assert result["is_active"] is True
    
    def test_missing_status_defaults_inactive(self):
        raw = {
            "id": 1,
            "first_name": "Bob",
            "last_name": "Jones",
            "email": "bob@test.com",
            "age": "30",
            # no "status" key
        }
        result = transform_user(raw)
        assert result["is_active"] is False
    
    def test_whitespace_handling(self):
        raw = {
            "id": 1,
            "first_name": "  Charlie  ",
            "last_name": "  Brown  ",
            "email": "  charlie@test.com  ",
            "age": "35",
        }
        result = transform_user(raw)
        assert result["email"] == "charlie@test.com"
```

### Test file processors

```python
import pytest
import csv
import json
from pathlib import Path

@pytest.fixture
def sample_csv(tmp_path):
    """Create a temporary CSV file for testing."""
    csv_file = tmp_path / "test_data.csv"
    csv_file.write_text(
        "name,age,city\n"
        "Alice,25,Kyiv\n"
        "Bob,30,London\n"
        "Charlie,35,NYC\n"
    )
    return csv_file

@pytest.fixture
def sample_jsonl(tmp_path):
    """Create a temporary JSONL file."""
    jsonl_file = tmp_path / "test_events.jsonl"
    events = [
        {"id": 1, "event": "click"},
        {"id": 2, "event": "view"},
        {"id": 3, "event": "click"},
    ]
    jsonl_file.write_text(
        "\n".join(json.dumps(e) for e in events) + "\n"
    )
    return jsonl_file

def test_csv_reader(sample_csv):
    """Test reading CSV file."""
    with open(sample_csv) as f:
        reader = csv.DictReader(f)
        rows = list(reader)
    
    assert len(rows) == 3
    assert rows[0]["name"] == "Alice"
    assert rows[0]["age"] == "25"

def test_jsonl_filter(sample_jsonl):
    """Test filtering JSONL events."""
    clicks = []
    with open(sample_jsonl) as f:
        for line in f:
            event = json.loads(line.strip())
            if event["event"] == "click":
                clicks.append(event)
    
    assert len(clicks) == 2
    assert all(e["event"] == "click" for e in clicks)

# tmp_path is a built-in pytest fixture — gives you a unique temp directory
# Automatically cleaned up after test session
```

### Test with database (SQLite in-memory)

```python
import pytest
import sqlite3

@pytest.fixture
def db():
    """In-memory SQLite for fast tests."""
    conn = sqlite3.connect(":memory:")
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            name TEXT NOT NULL,
            email TEXT UNIQUE,
            age INTEGER
        )
    """)
    conn.commit()
    yield conn
    conn.close()

@pytest.fixture
def db_with_users(db):
    """Database pre-populated with test data."""
    cursor = db.cursor()
    test_users = [
        (1, "Alice", "alice@test.com", 25),
        (2, "Bob", "bob@test.com", 30),
        (3, "Charlie", "charlie@test.com", 35),
    ]
    cursor.executemany(
        "INSERT INTO users VALUES (?, ?, ?, ?)", test_users
    )
    db.commit()
    return db

class TestUserRepository:
    
    def test_insert_user(self, db):
        cursor = db.cursor()
        cursor.execute(
            "INSERT INTO users (name, email, age) VALUES (?, ?, ?)",
            ("Alice", "alice@test.com", 25)
        )
        db.commit()
        
        cursor.execute("SELECT COUNT(*) FROM users")
        assert cursor.fetchone()[0] == 1
    
    def test_find_by_email(self, db_with_users):
        cursor = db_with_users.cursor()
        cursor.execute("SELECT * FROM users WHERE email = ?", ("alice@test.com",))
        user = cursor.fetchone()
        
        assert user is not None
        assert user["name"] == "Alice"
        assert user["age"] == 25
    
    def test_duplicate_email_fails(self, db_with_users):
        cursor = db_with_users.cursor()
        with pytest.raises(sqlite3.IntegrityError):
            cursor.execute(
                "INSERT INTO users (name, email, age) VALUES (?, ?, ?)",
                ("Duplicate", "alice@test.com", 28)
            )
```

---

## 8. Markers — Categorize and Control Tests

```python
import pytest

# Skip a test
@pytest.mark.skip(reason="Not implemented yet")
def test_future_feature():
    pass

# Skip conditionally
@pytest.mark.skipif(
    sys.platform == "win32",
    reason="Unix only"
)
def test_unix_feature():
    pass

# Expected failure (test is known to fail)
@pytest.mark.xfail(reason="Bug #123 not fixed yet")
def test_known_bug():
    assert broken_function() == "expected"

# Custom markers — categorize tests
@pytest.mark.slow
def test_large_dataset():
    """This test takes 30 seconds."""
    pass

@pytest.mark.integration
def test_database_connection():
    """Needs real database."""
    pass

@pytest.mark.unit
def test_pure_function():
    """Fast, no dependencies."""
    pass
```

```bash
# Run only specific markers
pytest -m "unit"                  # only unit tests
pytest -m "not slow"              # skip slow tests
pytest -m "integration"           # only integration tests
pytest -m "unit or integration"   # both
```

```ini
# pyproject.toml — register custom markers
[tool.pytest.ini_options]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks integration tests",
    "unit: marks unit tests",
]
```

---

## 9. Test Organization

### Recommended structure

```
tests/
├── conftest.py              # shared fixtures
├── unit/                    # fast, isolated tests
│   ├── conftest.py
│   ├── test_validators.py
│   ├── test_transformers.py
│   └── test_utils.py
├── integration/             # tests with real dependencies
│   ├── conftest.py
│   ├── test_database.py
│   └── test_api.py
└── e2e/                     # end-to-end pipeline tests
    ├── conftest.py
    └── test_pipeline.py
```

### Naming conventions

```python
# Good test names describe WHAT is tested and EXPECTED behavior
def test_validate_event_rejects_missing_user_id():
    ...

def test_transform_lowercases_email():
    ...

def test_csv_reader_handles_empty_file():
    ...

def test_retry_succeeds_on_third_attempt():
    ...

# Bad test names
def test_1():        # meaningless
def test_thing():    # too vague
def test_it_works(): # not specific
```

### Test class grouping

```python
class TestUserValidation:
    """Group related validation tests."""
    
    def test_valid_user_passes(self): ...
    def test_empty_name_fails(self): ...
    def test_invalid_email_fails(self): ...
    def test_negative_age_fails(self): ...

class TestUserTransformation:
    """Group related transformation tests."""
    
    def test_name_normalization(self): ...
    def test_email_lowercase(self): ...
    def test_age_conversion(self): ...
```

---

## 10. pytest Configuration

### `pyproject.toml`

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = [
    "-v",                    # verbose output
    "--tb=short",            # short tracebacks
    "--strict-markers",      # error on unknown markers
    "-ra",                   # show reasons for all non-passed tests
]
markers = [
    "slow: slow tests",
    "integration: needs external services",
    "unit: fast unit tests",
]
filterwarnings = [
    "ignore::DeprecationWarning",
]
```

### Running tests — useful commands

```bash
# Run all tests
pytest

# Verbose
pytest -v

# Run specific file
pytest tests/test_validators.py

# Run specific test
pytest tests/test_validators.py::test_email_valid

# Run specific class
pytest tests/test_validators.py::TestEmailValidation

# Run tests matching pattern
pytest -k "email"               # tests with "email" in name
pytest -k "email and not slow"  # combine patterns

# Stop on first failure
pytest -x

# Run last failed tests only
pytest --lf

# Run failed first, then rest
pytest --ff

# Show print() output (normally captured)
pytest -s

# Show local variables in traceback
pytest -l

# Parallel execution (needs pytest-xdist)
pytest -n 4    # run on 4 cores

# Coverage report
pytest --cov=src --cov-report=html
```

---

## 11. Coverage

```bash
# Install
pip install pytest-cov

# Run with coverage
pytest --cov=src --cov-report=term-missing

# Output:
# Name                    Stmts   Miss  Cover   Missing
# -----------------------------------------------------
# src/calculator.py          20      2    90%   15, 23
# src/validator.py           35      5    86%   12-14, 28, 35
# src/transformer.py         45      0   100%
# -----------------------------------------------------
# TOTAL                     100      7    93%

# HTML report (browseable)
pytest --cov=src --cov-report=html
# Open htmlcov/index.html in browser

# Fail if coverage below threshold
pytest --cov=src --cov-fail-under=90
```

```toml
# pyproject.toml — coverage configuration
[tool.coverage.run]
source = ["src"]
omit = ["*/tests/*", "*/migrations/*"]

[tool.coverage.report]
show_missing = true
fail_under = 85
exclude_lines = [
    "if __name__ == .__main__.",
    "raise NotImplementedError",
    "pass",
]
```

---

## 12. Doctests — Tests in Docstrings

```python
def celsius_to_fahrenheit(celsius: float) -> float:
    """Convert Celsius to Fahrenheit.
    
    >>> celsius_to_fahrenheit(0)
    32.0
    >>> celsius_to_fahrenheit(100)
    212.0
    >>> celsius_to_fahrenheit(-40)
    -40.0
    """
    return celsius * 9 / 5 + 32


def flatten_dict(d: dict, parent_key: str = "", sep: str = ".") -> dict:
    """Flatten nested dict with dot-separated keys.
    
    >>> flatten_dict({"a": 1, "b": {"c": 2, "d": 3}})
    {'a': 1, 'b.c': 2, 'b.d': 3}
    
    >>> flatten_dict({"x": {"y": {"z": 1}}})
    {'x.y.z': 1}
    
    >>> flatten_dict({})
    {}
    """
    items = {}
    for k, v in d.items():
        new_key = f"{parent_key}{sep}{k}" if parent_key else k
        if isinstance(v, dict):
            items.update(flatten_dict(v, new_key, sep))
        else:
            items[new_key] = v
    return items
```

```bash
# Run doctests
pytest --doctest-modules

# Or standalone
python -m doctest mymodule.py -v
```

```
Doctests are good for:
  ✅ Simple examples in documentation
  ✅ Verifying docs stay accurate
  ✅ Quick sanity checks

Doctests are NOT good for:
  ❌ Complex test scenarios
  ❌ Tests needing setup/teardown
  ❌ Edge cases and error handling
  → Use pytest for those
```

---

## 13. Async Testing

```python
import pytest
import asyncio

# pip install pytest-asyncio

@pytest.mark.asyncio
async def test_async_function():
    result = await some_async_operation()
    assert result == "expected"

@pytest.mark.asyncio
async def test_async_with_gather():
    results = await asyncio.gather(
        fetch("url1"),
        fetch("url2"),
    )
    assert len(results) == 2

# Async fixtures
@pytest.fixture
async def async_db():
    conn = await create_async_connection()
    yield conn
    await conn.close()

@pytest.mark.asyncio
async def test_async_query(async_db):
    result = await async_db.execute("SELECT 1")
    assert result is not None
```

---

## 📝 Practice Tasks

### Task 1: Test Suite for Calculator
Write a calculator module with `add`, `subtract`, `multiply`, `divide`, `power`, `sqrt`. Write comprehensive tests: normal cases, edge cases (zero, negative, floats), exceptions (division by zero, negative sqrt). Use parametrize for multiple inputs.

### Task 2: Test Data Validator
Take the event validator from section 7 and write a full test suite with: valid inputs, every possible invalid input, edge cases (empty strings, wrong types, boundary values). Aim for 100% coverage.

### Task 3: Test with Mocking
Write a `WeatherService` class that fetches weather from an API. Test it by mocking the HTTP calls. Test scenarios: success, network error, timeout, invalid response JSON, 404 response. Never make real HTTP calls in tests.

### Task 4: Test a File Pipeline
Write a function that reads a CSV, validates each row, transforms valid rows, and writes output. Test the full pipeline using `tmp_path` fixture for temp files. Test: normal flow, empty file, malformed rows, missing columns.

### Task 5: Integration Tests with SQLite
Build a `UserRepository` class with `create`, `get_by_id`, `get_by_email`, `update`, `delete`, `list_all`. Write integration tests using in-memory SQLite. Test: CRUD operations, duplicate email constraint, not-found cases, update non-existent user.

### Task 6: Test Coverage Challenge
Take any module from previous days (file stats, CSV cleaner, JSON converter, etc.) and write tests to achieve 95%+ coverage. Run `pytest --cov` and iteratively fill gaps.

---

## 📚 Resources

- [pytest Documentation](https://docs.pytest.org/)
- [Real Python — Testing with pytest](https://realpython.com/pytest-python-testing/)
- [Real Python — Mocking](https://realpython.com/python-mock-library/)
- [pytest Fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html)
- [pytest Parametrize](https://docs.pytest.org/en/stable/how-to/parametrize.html)
- [pytest-cov](https://pytest-cov.readthedocs.io/)
- [pytest-asyncio](https://pytest-asyncio.readthedocs.io/)
- [pytest-mock](https://pytest-mock.readthedocs.io/)
- [Effective Python Testing with pytest (article)](https://realpython.com/pytest-python-testing/)

---

## 🔑 Key Takeaways

1. **Use pytest, not unittest** — less boilerplate, better errors, more features.
2. **Plain `assert`** — no `assertEqual`, just `assert x == y`. pytest shows the diff.
3. **Fixtures > setUp/tearDown** — dependency injection, composable, scoped.
4. **`yield` in fixtures** — everything before yield = setup, after = teardown.
5. **`conftest.py`** — shared fixtures auto-discovered by pytest. No imports needed.
6. **`parametrize`** — test many inputs with one function. Avoid copy-paste tests.
7. **Mock WHERE it's used** — `patch("mymodule.requests.get")`, not `patch("requests.get")`.
8. **`tmp_path`** — built-in fixture for temp files. Auto-cleaned.
9. **`-k` for filtering** — `pytest -k "email and not slow"` runs matching tests.
10. **Coverage as a guide, not a goal** — 100% coverage doesn't mean bug-free. Aim for 85-95% on critical code.

---

> 🎉 **Phase 2 Complete!** You've finished Python Core + Intermediate.
> 
> **Next up — Phase 4: Databases & SQL (Days 15-18)**
> - Day 15: SQL Fundamentals with Python (sqlite3)
> - Day 16: SQLAlchemy Core — SQL Toolkit
> - Day 17: SQLAlchemy ORM — Object-Relational Mapping
> - Day 18: Database Design & Migrations (Alembic)

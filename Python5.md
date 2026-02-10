# Day 5: Exceptions, Modules, Type Hints & Pydantic

## 🎯 Goal

Three essential topics today that tie together into professional-grade Python:
1. **Exceptions** — Python's error handling (similar to Java try/catch but with unique patterns)
2. **Modules & Packages** — Python's import system (like Node's require/import or Java's packages)
3. **Type Hints & Pydantic** — bringing TypeScript-level type safety to Python (critical for Data Engineering)

---

## 1. Exceptions — Error Handling

### Basic try/except

```python
# Python uses try/except (not try/catch like Java/JS)
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Can't divide by zero!")

# Multiple exception types
try:
    value = int("not a number")
except ValueError:
    print("Invalid number format")
except TypeError:
    print("Wrong type")

# Catch multiple in one line
try:
    # ...
    pass
except (ValueError, TypeError) as e:
    print(f"Error: {e}")

# Catch ALL exceptions (use sparingly!)
try:
    risky_operation()
except Exception as e:
    print(f"Something went wrong: {e}")
    # Exception catches most errors
    # BaseException catches EVERYTHING (including KeyboardInterrupt, SystemExit)
    # Almost never use BaseException
```

### The full try/except/else/finally

```python
try:
    file = open("data.txt")
    data = file.read()
except FileNotFoundError:
    print("File not found!")
    data = None
except PermissionError:
    print("No permission!")
    data = None
else:
    # Runs ONLY if NO exception occurred
    # This is Python-unique — Java/JS don't have this
    print(f"Read {len(data)} characters")
finally:
    # ALWAYS runs — same as Java/JS
    print("Cleanup done")

# Why use else?
# Put code that should only run on success in else, not in try.
# This way you don't accidentally catch exceptions from that code.

# ❌ Bad — might catch unrelated ValueError from process_data()
try:
    data = parse_input(raw)
    result = process_data(data)  # if this raises ValueError, it's caught too!
except ValueError:
    print("Bad input")

# ✅ Good — only catches ValueError from parse_input()
try:
    data = parse_input(raw)
except ValueError:
    print("Bad input")
else:
    result = process_data(data)  # exceptions here propagate normally
```

### Exception hierarchy (key ones)

```
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception                  ← catch this, not BaseException
    ├── ValueError             ← wrong value ("int('abc')")
    ├── TypeError              ← wrong type (1 + "2")
    ├── KeyError               ← dict key not found
    ├── IndexError             ← list index out of range
    ├── AttributeError         ← attribute not found
    ├── FileNotFoundError      ← file doesn't exist
    ├── IOError                ← I/O operation failed
    ├── OSError                ← OS-level error
    ├── RuntimeError           ← generic runtime error
    ├── StopIteration          ← iterator exhausted
    ├── ImportError             
    │   └── ModuleNotFoundError
    └── ArithmeticError
        ├── ZeroDivisionError
        └── OverflowError
```

### Raising exceptions

```python
# Raise built-in exception
def divide(a, b):
    if b == 0:
        raise ValueError("Divisor cannot be zero")
    return a / b

# Re-raise current exception
try:
    risky_operation()
except ValueError:
    print("Logging the error...")
    raise  # re-raises the SAME exception with original traceback

# Raise with cause (exception chaining)
try:
    data = json.loads(raw_input)
except json.JSONDecodeError as e:
    raise ValueError(f"Invalid config format") from e
    # The traceback shows BOTH exceptions:
    # "The above exception was the direct cause of the following exception"

# Suppress original context
try:
    data = json.loads(raw_input)
except json.JSONDecodeError:
    raise ValueError("Invalid config") from None
    # Hides the original exception
```

### Custom exceptions

```python
# Simple custom exception
class AppError(Exception):
    """Base exception for our application."""
    pass

class ValidationError(AppError):
    """Raised when data validation fails."""
    pass

class NotFoundError(AppError):
    """Raised when a resource is not found."""
    pass

# Custom exception with extra data
class APIError(AppError):
    """Raised when an API call fails."""
    
    def __init__(self, message: str, status_code: int, response_body: dict = None):
        super().__init__(message)
        self.status_code = status_code
        self.response_body = response_body or {}
    
    def __str__(self):
        return f"APIError({self.status_code}): {super().__str__()}"

# Usage
try:
    raise APIError("Rate limited", 429, {"retry_after": 60})
except APIError as e:
    print(e)                # APIError(429): Rate limited
    print(e.status_code)    # 429
    print(e.response_body)  # {"retry_after": 60}

# Hierarchy pattern for larger projects:
# project/
#   exceptions.py:
#     class AppError(Exception): ...
#     class ValidationError(AppError): ...
#     class DatabaseError(AppError): ...
#     class AuthError(AppError): ...
#   
#   service.py:
#     from .exceptions import ValidationError
#     raise ValidationError("email is required")

# Java comparison:
# class APIException extends RuntimeException {
#     private int statusCode;
#     ...
# }
```

### Exception patterns for real code

```python
# LBYL vs EAFP — two competing philosophies

# LBYL: "Look Before You Leap" (Java style)
if key in dictionary:
    value = dictionary[key]
else:
    value = default

# EAFP: "Easier to Ask Forgiveness than Permission" (Python style ✅)
try:
    value = dictionary[key]
except KeyError:
    value = default

# Python community strongly prefers EAFP because:
# 1. It's often faster (no double lookup)
# 2. Avoids race conditions (file might disappear between check and open)
# 3. It's more Pythonic

# But for dicts, just use .get():
value = dictionary.get(key, default)  # best of both worlds

# EAFP with files
try:
    with open("config.json") as f:
        config = json.load(f)
except FileNotFoundError:
    config = default_config
except json.JSONDecodeError:
    print("Warning: corrupted config, using defaults")
    config = default_config
```

### Context managers for exception safety (preview of Day 7)

```python
# ❌ Unsafe — if process() raises, file is never closed
file = open("data.txt")
data = file.read()
process(data)
file.close()

# ✅ Safe — with statement guarantees cleanup
with open("data.txt") as file:
    data = file.read()
    process(data)
# file.close() is called automatically, even if process() raises

# Same concept as Java try-with-resources:
# try (var file = new FileReader("data.txt")) { ... }
```

---

## 2. Modules & Packages

### Import system

```python
# Import entire module
import math
print(math.sqrt(16))  # 4.0

# Import specific items
from math import sqrt, pi
print(sqrt(16))  # 4.0

# Import with alias
import numpy as np
from collections import defaultdict as dd

# Import everything (avoid in production code!)
from math import *  # pollutes namespace, hard to track origins

# Relative imports (within a package)
from . import utils           # from current package
from ..models import User     # from parent package
from .helpers import validate # from sibling module
```

### Module vs Script — `__name__` guard

```python
# mymodule.py
def hello():
    return "Hello from mymodule!"

# This block runs ONLY when file is executed directly (not imported)
if __name__ == "__main__":
    # This is the entry point when running: python mymodule.py
    print(hello())
    
    # Common uses:
    # - Quick testing
    # - CLI entry point
    # - Demo code

# JS equivalent: there's no direct equivalent
# Java equivalent: public static void main(String[] args) { }
```

### Creating packages

```
# Package = directory with __init__.py

myproject/
├── __init__.py           # makes it a package (can be empty)
├── models/
│   ├── __init__.py
│   ├── user.py
│   └── product.py
├── services/
│   ├── __init__.py
│   ├── auth.py
│   └── payment.py
├── utils/
│   ├── __init__.py
│   ├── validators.py
│   └── helpers.py
└── main.py
```

```python
# myproject/models/user.py
class User:
    def __init__(self, name: str):
        self.name = name

# myproject/models/__init__.py — control what's exported
from .user import User
from .product import Product

# Now consumers can do:
from myproject.models import User  # clean import

# Without __init__.py exports:
from myproject.models.user import User  # less clean
```

### `__init__.py` patterns

```python
# myproject/__init__.py

# 1. Empty — just marks directory as a package
# (most common for small projects)

# 2. Re-export key items — create a clean public API
from .models import User, Product
from .services import AuthService
__all__ = ["User", "Product", "AuthService"]  # controls "from myproject import *"

# 3. Package-level initialization
import logging
logging.getLogger(__name__).addHandler(logging.NullHandler())

# 4. Version
__version__ = "1.0.0"
```

### How Python finds modules — `sys.path`

```python
import sys
print(sys.path)
# ['',                          # current directory
#  '/usr/lib/python3.12',       # standard library
#  '/usr/lib/python3.12/site-packages',  # installed packages
#  ...]

# Python searches these directories IN ORDER when you import
# You can add paths:
sys.path.append('/my/custom/path')
# But prefer proper package installation instead
```

### Standard library highlights (modules you should know exist)

```python
# Data & text
import json          # JSON encode/decode
import csv           # CSV read/write
import re            # regex
import datetime      # dates and times
import decimal       # precise decimal math (money!)

# System & files
import os            # OS interface (paths, env vars)
import sys           # system-specific (argv, exit, path)
import pathlib       # modern path handling (prefer over os.path)
import shutil        # file operations (copy, move, delete dirs)
import tempfile      # temporary files and directories

# Collections & algorithms
import collections   # Counter, defaultdict, deque, namedtuple
import itertools     # chain, product, permutations, groupby
import functools     # reduce, lru_cache, partial, wraps
import operator      # itemgetter, attrgetter (for sorting)

# Concurrency (Day 11-12)
import threading
import multiprocessing
import asyncio

# Data formats
import pickle        # Python object serialization (binary)
import sqlite3       # built-in SQL database
import xml.etree.ElementTree  # XML parsing

# Utilities
import logging       # structured logging
import typing        # type hints
import dataclasses   # dataclass decorator
import abc           # abstract base classes
import copy          # shallow/deep copy
import hashlib       # hashing (md5, sha256)
import uuid          # unique IDs
```

---

## 3. Type Hints — Bringing TypeScript Vibes to Python

### Why type hints matter

Python is dynamically typed, but type hints (added in 3.5+) give you:
- **Documentation** — what does this function expect/return?
- **IDE support** — autocomplete, error detection
- **Static analysis** — mypy/pyright catch bugs before runtime
- **Required by frameworks** — FastAPI, Pydantic, SQLAlchemy 2.0 all use them

```python
# Without hints — what does this even take/return?
def process(data, config):
    ...

# With hints — crystal clear
def process(data: list[dict[str, Any]], config: Config) -> ProcessResult:
    ...
```

### Basic type hints

```python
# Variables
name: str = "Alice"
age: int = 25
height: float = 1.82
is_active: bool = True
nothing: None = None

# Functions
def greet(name: str, excited: bool = False) -> str:
    if excited:
        return f"HELLO {name.upper()}!!!"
    return f"Hello, {name}"

# None return
def log(message: str) -> None:
    print(message)
```

### Complex types

```python
from typing import Any, Optional, Union

# Collections (Python 3.9+ — use built-in types)
names: list[str] = ["Alice", "Bob"]
ages: dict[str, int] = {"Alice": 25, "Bob": 30}
unique: set[int] = {1, 2, 3}
point: tuple[float, float] = (3.0, 4.0)        # fixed length tuple
values: tuple[int, ...] = (1, 2, 3, 4, 5)      # variable length tuple

# Optional — value or None
def find_user(user_id: int) -> Optional[str]:
    """Returns user name or None."""
    pass

# Python 3.10+ — use | instead of Optional/Union
def find_user(user_id: int) -> str | None:
    pass

def process(value: int | str | float) -> str:
    return str(value)

# Any — opt out of type checking
def debug_print(data: Any) -> None:
    print(data)

# Nested
users: list[dict[str, str | int]] = [
    {"name": "Alice", "age": 25},
    {"name": "Bob", "age": 30},
]

# Callable (for function parameters)
from typing import Callable

def apply(func: Callable[[int, int], int], a: int, b: int) -> int:
    return func(a, b)
# Callable[[ArgType1, ArgType2], ReturnType]

# TypeVar — generics
from typing import TypeVar

T = TypeVar("T")

def first(items: list[T]) -> T | None:
    """Return first item or None — works with any type."""
    return items[0] if items else None

first([1, 2, 3])        # inferred as int
first(["a", "b", "c"])  # inferred as str
```

### Type aliases

```python
# Simple alias
UserId = int
UserName = str
UserMap = dict[UserId, UserName]

def get_users() -> UserMap:
    return {1: "Alice", 2: "Bob"}

# Complex alias (Python 3.12+)
type JsonValue = str | int | float | bool | None | list["JsonValue"] | dict[str, "JsonValue"]

# Pre-3.12
from typing import TypeAlias
JsonValue: TypeAlias = str | int | float | bool | None | list["JsonValue"] | dict[str, "JsonValue"]

# For Data Engineering — you'll define types like:
type Row = dict[str, Any]
type DataFrame = list[Row]
type Schema = dict[str, type]
```

### TypedDict — type-safe dictionaries

```python
from typing import TypedDict, NotRequired

class UserDict(TypedDict):
    name: str
    age: int
    email: NotRequired[str]  # optional key

# Type checker ensures correct keys and value types
user: UserDict = {"name": "Alice", "age": 25}  # ✅
# user: UserDict = {"name": "Alice"}            # ❌ missing 'age'
# user: UserDict = {"name": "Alice", "age": "25"}  # ❌ age should be int

# At runtime it's just a regular dict — no validation!
# For runtime validation, use Pydantic (next section)

# Useful for typing JSON responses, config files, etc.
class APIResponse(TypedDict):
    status: int
    data: list[UserDict]
    meta: dict[str, Any]
```

### Literal types

```python
from typing import Literal

def set_status(status: Literal["active", "inactive", "pending"]) -> None:
    """Only these exact string values are allowed."""
    pass

set_status("active")    # ✅
# set_status("deleted")  # ❌ type checker flags this

# Like TypeScript's: type Status = "active" | "inactive" | "pending"

# Useful for config values, API parameters, etc.
def sort_users(
    users: list[dict],
    by: Literal["name", "age", "created_at"] = "name",
    order: Literal["asc", "desc"] = "asc"
) -> list[dict]:
    ...
```

### Running type checkers

```bash
# Install mypy
pip install mypy

# Run on a file
mypy my_script.py

# Run on entire project
mypy src/

# Strict mode (more checks)
mypy --strict src/

# pyright (faster alternative, used by VS Code Pylance)
pip install pyright
pyright src/

# Configure in pyproject.toml:
# [tool.mypy]
# strict = true
# warn_return_any = true
# disallow_untyped_defs = true
```

---

## 4. Pydantic — Runtime Data Validation

Pydantic is **the** data validation library in Python. It uses type hints to validate data at **runtime** (unlike mypy which only checks statically). Essential for Data Engineering, FastAPI, and any data pipeline.

### Why Pydantic?

```python
# Problem: raw dicts from JSON/API/database are unvalidated
user_data = {"name": "Alice", "age": "25", "email": "not-an-email"}
# Is age a string or int? Is email valid? What if fields are missing?

# dataclasses don't validate:
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int

user = User(name="Alice", age="not a number")  # ✅ No error! age is a string.
# Runtime breaks later when you do math on age

# Pydantic validates AND converts:
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int

user = User(name="Alice", age="25")    # ✅ age auto-converted to int: 25
# user = User(name="Alice", age="abc") # ❌ ValidationError!
```

### Basic Pydantic models

```python
from pydantic import BaseModel, Field, field_validator
from datetime import datetime

class User(BaseModel):
    name: str
    age: int
    email: str
    is_active: bool = True                      # default value
    tags: list[str] = []                        # mutable default is safe in Pydantic!
    created_at: datetime = Field(default_factory=datetime.now)

# From dict (most common — parsing JSON, API responses, DB rows)
user = User(**{"name": "Alice", "age": 25, "email": "alice@example.com"})
# or equivalently:
user = User(name="Alice", age=25, email="alice@example.com")

print(user.name)          # Alice
print(user.model_dump())  # {"name": "Alice", "age": 25, ...} — to dict
print(user.model_dump_json())  # JSON string

# Auto-conversion (coercion)
user = User(name="Alice", age="25", email="alice@example.com")
print(type(user.age))     # <class 'int'> — converted from "25"!
```

### Validation with Field()

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0, description="Price must be positive")
    quantity: int = Field(ge=0, default=0)
    sku: str = Field(pattern=r"^[A-Z]{2}-\d{4}$")  # regex validation

# ✅ Valid
p = Product(name="Widget", price=9.99, sku="AB-1234")

# ❌ All of these raise ValidationError:
# Product(name="", price=9.99, sku="AB-1234")       # name too short
# Product(name="Widget", price=-5, sku="AB-1234")   # price not > 0
# Product(name="Widget", price=9.99, sku="invalid")  # sku doesn't match pattern
```

### Custom validators

```python
from pydantic import BaseModel, field_validator, model_validator

class User(BaseModel):
    name: str
    age: int
    email: str
    password: str
    password_confirm: str
    
    @field_validator("name")
    @classmethod
    def name_must_not_be_empty(cls, v: str) -> str:
        """Validate a single field."""
        v = v.strip()
        if not v:
            raise ValueError("Name cannot be empty")
        return v.title()  # transform: return value is stored
    
    @field_validator("age")
    @classmethod
    def age_must_be_reasonable(cls, v: int) -> int:
        if not 0 <= v <= 150:
            raise ValueError("Age must be between 0 and 150")
        return v
    
    @field_validator("email")
    @classmethod
    def email_must_contain_at(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("Invalid email format")
        return v.lower()
    
    @model_validator(mode="after")
    def passwords_match(self) -> "User":
        """Validate across multiple fields."""
        if self.password != self.password_confirm:
            raise ValueError("Passwords don't match")
        return self

# Usage
user = User(
    name="  alice  ",
    age=25,
    email="Alice@Example.COM",
    password="secret",
    password_confirm="secret"
)
print(user.name)   # "Alice" — transformed by validator
print(user.email)  # "alice@example.com" — lowercased
```

### Nested models

```python
from pydantic import BaseModel

class Address(BaseModel):
    street: str
    city: str
    country: str
    zip_code: str | None = None

class Company(BaseModel):
    name: str
    address: Address          # nested model
    employees: list["Employee"] = []

class Employee(BaseModel):
    name: str
    role: str
    company: Company | None = None

# Pydantic validates the entire nested structure!
company = Company(
    name="Acme",
    address={                          # auto-converted to Address
        "street": "123 Main St",
        "city": "Kyiv",
        "country": "Ukraine"
    },
    employees=[
        {"name": "Alice", "role": "Developer"},   # auto-converted to Employee
        {"name": "Bob", "role": "Designer"},
    ]
)

print(company.address.city)             # Kyiv
print(company.employees[0].name)        # Alice
print(type(company.address))            # <class 'Address'>
```

### Parsing external data (JSON, dicts)

```python
from pydantic import BaseModel
import json

class Config(BaseModel):
    host: str
    port: int = 8080
    debug: bool = False
    database_url: str
    allowed_origins: list[str] = []

# From JSON string
json_str = '{"host": "localhost", "port": "3000", "database_url": "postgres://...", "debug": "true"}'
config = Config.model_validate_json(json_str)
print(config.port)    # 3000 (int, not "3000")
print(config.debug)   # True (bool, not "true")

# From dict
data = {"host": "localhost", "database_url": "postgres://..."}
config = Config.model_validate(data)

# From JSON file
with open("config.json") as f:
    raw = json.load(f)
config = Config.model_validate(raw)

# Export
config.model_dump()       # → dict
config.model_dump_json()  # → JSON string
config.model_dump(exclude={"database_url"})  # exclude sensitive fields
config.model_dump(include={"host", "port"})  # only these fields
```

### Pydantic Settings — environment variables

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    """Reads from environment variables automatically!"""
    database_url: str
    redis_url: str = "redis://localhost:6379"
    debug: bool = False
    api_key: str
    
    model_config = {
        "env_file": ".env",           # also reads from .env file
        "env_prefix": "APP_",         # looks for APP_DATABASE_URL, APP_API_KEY, etc.
    }

# Set env vars: APP_DATABASE_URL=postgres://... APP_API_KEY=secret
settings = Settings()
print(settings.database_url)

# This is THE way to handle configuration in Python apps
# Similar to: Spring's @ConfigurationProperties or dotenv in Node
```

### Pydantic for Data Engineering — real examples

```python
from pydantic import BaseModel, Field, field_validator
from datetime import datetime
from enum import Enum

class EventType(str, Enum):
    CLICK = "click"
    VIEW = "view"
    PURCHASE = "purchase"

class Event(BaseModel):
    """Schema for raw analytics events — validates data from Kafka/API."""
    event_id: str = Field(min_length=1)
    event_type: EventType
    user_id: int = Field(gt=0)
    timestamp: datetime
    properties: dict[str, str | int | float | bool] = {}
    
    @field_validator("timestamp")
    @classmethod
    def timestamp_not_future(cls, v: datetime) -> datetime:
        if v > datetime.now():
            raise ValueError("Timestamp cannot be in the future")
        return v

# Validate a batch of events from a data pipeline
raw_events = [
    {"event_id": "e1", "event_type": "click", "user_id": 42, "timestamp": "2025-02-10T12:00:00"},
    {"event_id": "e2", "event_type": "purchase", "user_id": 7, "timestamp": "2025-02-10T12:05:00",
     "properties": {"amount": 99.99}},
]

valid_events = []
errors = []

for raw in raw_events:
    try:
        event = Event.model_validate(raw)
        valid_events.append(event)
    except Exception as e:
        errors.append({"raw": raw, "error": str(e)})

print(f"Valid: {len(valid_events)}, Errors: {len(errors)}")

# This pattern — validate → split good/bad → process good, log bad
# is the foundation of every data pipeline
```

### Pydantic vs dataclasses vs TypedDict

```
TypedDict:
  ✅ Just type hints for dicts
  ❌ NO runtime validation
  Use: typing hints for function params, API response shapes

dataclass:
  ✅ Auto __init__, __repr__, __eq__
  ❌ NO runtime validation or coercion
  Use: internal data containers, simple DTOs

Pydantic BaseModel:
  ✅ Runtime validation + coercion
  ✅ JSON serialization/deserialization
  ✅ Custom validators
  ✅ Settings management
  ❌ Slightly slower than dataclass (validation overhead)
  Use: external data (API, JSON, DB, files), config, schemas

Rule of thumb for Data Engineering:
  External data → Pydantic (validate at boundaries)
  Internal data → dataclass (trust within your code)
```

---

## 5. Type Casting (Conversions)

```python
# Explicit casting — Python doesn't do implicit conversion (unlike JS)
int("42")         # 42
int(3.14)         # 3 (truncates, NOT rounds)
int("0xff", 16)   # 255 (from hex)
float("3.14")     # 3.14
str(42)           # "42"
bool(0)           # False
bool("")          # False
bool([])          # False
bool(None)        # False
bool(1)           # True
bool("hello")     # True

# Between collections
list((1, 2, 3))           # [1, 2, 3]
tuple([1, 2, 3])          # (1, 2, 3)
set([1, 2, 2, 3])         # {1, 2, 3}
dict([("a", 1), ("b", 2)]) # {"a": 1, "b": 2}

# Safe casting pattern
def safe_int(value, default=0):
    try:
        return int(value)
    except (ValueError, TypeError):
        return default

safe_int("42")       # 42
safe_int("abc")      # 0
safe_int(None)       # 0
safe_int("42", -1)   # 42
```

---

## 6. Putting It All Together — Professional Module Structure

```
myproject/
├── pyproject.toml          # project config (replaces setup.py)
├── src/
│   └── myproject/
│       ├── __init__.py
│       ├── config.py       # Pydantic Settings
│       ├── exceptions.py   # custom exceptions
│       ├── models/
│       │   ├── __init__.py
│       │   ├── user.py     # Pydantic models
│       │   └── event.py
│       ├── services/
│       │   ├── __init__.py
│       │   └── user_service.py
│       └── utils/
│           ├── __init__.py
│           └── validators.py
├── tests/
│   ├── __init__.py
│   ├── test_models.py
│   └── test_services.py
└── README.md
```

```python
# src/myproject/exceptions.py
class AppError(Exception):
    """Base application error."""
    pass

class ValidationError(AppError):
    def __init__(self, field: str, message: str):
        self.field = field
        super().__init__(f"Validation error on '{field}': {message}")

class NotFoundError(AppError):
    def __init__(self, resource: str, id: str | int):
        self.resource = resource
        self.id = id
        super().__init__(f"{resource} with id={id} not found")

class ExternalServiceError(AppError):
    def __init__(self, service: str, status_code: int, detail: str = ""):
        self.service = service
        self.status_code = status_code
        super().__init__(f"{service} returned {status_code}: {detail}")
```

```python
# src/myproject/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_name: str = "MyProject"
    debug: bool = False
    database_url: str
    redis_url: str = "redis://localhost:6379"
    log_level: str = "INFO"
    
    model_config = {"env_file": ".env"}

settings = Settings()  # singleton — import this everywhere
```

```python
# src/myproject/models/user.py
from pydantic import BaseModel, Field, field_validator
from datetime import datetime

class UserCreate(BaseModel):
    """Input model — for creating a user."""
    name: str = Field(min_length=1, max_length=100)
    email: str
    age: int = Field(ge=0, le=150)
    
    @field_validator("email")
    @classmethod
    def validate_email(cls, v):
        if "@" not in v:
            raise ValueError("Invalid email")
        return v.lower()

class UserResponse(BaseModel):
    """Output model — for API responses."""
    id: int
    name: str
    email: str
    age: int
    created_at: datetime
    
    model_config = {"from_attributes": True}  # allows creating from ORM objects
```

```python
# src/myproject/services/user_service.py
from ..models.user import UserCreate, UserResponse
from ..exceptions import NotFoundError, ValidationError

class UserService:
    def __init__(self, db):
        self.db = db
    
    def create_user(self, data: UserCreate) -> UserResponse:
        # data is already validated by Pydantic!
        if self.db.user_exists(data.email):
            raise ValidationError("email", "Email already registered")
        
        user = self.db.insert_user(data.model_dump())
        return UserResponse.model_validate(user)
    
    def get_user(self, user_id: int) -> UserResponse:
        user = self.db.find_user(user_id)
        if not user:
            raise NotFoundError("User", user_id)
        return UserResponse.model_validate(user)
```

---

## 📝 Practice Tasks

### Task 1: Exception Hierarchy
Create a custom exception hierarchy for a data pipeline:
- `PipelineError` (base)
  - `ExtractionError` (source system issues)
  - `TransformationError` (data processing issues)
  - `LoadError` (destination issues)
  - `ValidationError` (schema violations, with field name and expected type)

Write a function that simulates a pipeline and raises appropriate exceptions.

### Task 2: Safe Data Parser
Write a `safe_parse` function that takes a JSON string and a Pydantic model class, returns either the parsed model or a structured error report with all validation failures.

### Task 3: Configuration System
Create a Pydantic Settings class for a data pipeline application with: database connection, S3 bucket, batch size, retry count, log level. Load from a `.env` file. Include validation (batch size > 0, log level must be one of DEBUG/INFO/WARNING/ERROR).

### Task 4: Module Structure
Create a proper Python package for a "task manager" with:
- `models/` — Pydantic models for Task (title, description, status, priority, due_date)
- `exceptions.py` — custom exceptions
- `services/task_service.py` — CRUD operations (use a list as fake DB)
- `__init__.py` — clean public API
- Write a `main.py` that demonstrates the full package

### Task 5: Data Validation Pipeline
Create Pydantic models for CSV data validation:
- Read a CSV (make a sample one)
- Validate each row against a Pydantic model
- Collect valid rows and error rows separately
- Print a summary report

### Task 6: Type Hints Challenge
Add comprehensive type hints to this untyped code (make mypy happy in strict mode):
```python
def process_records(records, filter_fn=None, transform_fn=None):
    results = []
    for record in records:
        if filter_fn and not filter_fn(record):
            continue
        if transform_fn:
            record = transform_fn(record)
        results.append(record)
    return results
```

---

## 📚 Resources

- [Python Official — Errors and Exceptions](https://docs.python.org/3/tutorial/errors.html)
- [Python Official — Modules](https://docs.python.org/3/tutorial/modules.html)
- [Real Python — Exception Handling](https://realpython.com/python-exceptions/)
- [Real Python — Python Modules and Packages](https://realpython.com/python-modules-packages/)
- [Real Python — Type Hints](https://realpython.com/python-type-checking/)
- [Pydantic v2 Documentation](https://docs.pydantic.dev/latest/)
- [mypy Documentation](https://mypy.readthedocs.io/en/stable/)
- [typing Module — Official Docs](https://docs.python.org/3/library/typing.html)

---

## 🔑 Key Takeaways

1. **EAFP over LBYL** — Python prefers try/except over checking conditions upfront.
2. **`else` in try** — runs only on success; keeps your error handling precise.
3. **Custom exceptions** — create a hierarchy for your app; always inherit from `Exception`.
4. **`__name__ == "__main__"`** — the entry point guard; use it in every script.
5. **`__init__.py`** — controls your package's public API; re-export what matters.
6. **Type hints don't enforce at runtime** — they're for tooling (mypy, IDE) only.
7. **Pydantic validates at runtime** — use for all external data boundaries.
8. **Pydantic Settings** — the standard way to handle config & environment variables.
9. **Pattern: validate at boundaries, trust internally** — Pydantic at edges, dataclasses inside.

---

> **Tomorrow (Day 6):** Iterators, Generators & Generator Pipelines — the foundation of memory-efficient data processing. This is where Python starts to really shine for Data Engineering.

# Day 7: Decorators & Context Managers

## 🎯 Goal

Two of Python's most elegant features that you'll use daily:
1. **Decorators** — modify/enhance functions and classes without changing their code (like Java annotations but way more powerful)
2. **Context Managers** — guarantee cleanup with `with` statement (like Java try-with-resources but more flexible)

Both rely heavily on closures and first-class functions from Day 2.

---

## 1. Decorators — The Concept

A decorator is a function that takes a function and returns a modified version of it. That's it.

```python
# Without decorator syntax — what's actually happening
def my_decorator(func):
    def wrapper():
        print("Before")
        func()
        print("After")
    return wrapper

def say_hello():
    print("Hello!")

# Manually decorating
say_hello = my_decorator(say_hello)  # replace function with wrapped version
say_hello()
# Before
# Hello!
# After

# With @ syntax — exact same thing, just cleaner
@my_decorator
def say_hello():
    print("Hello!")

# @my_decorator is syntactic sugar for: say_hello = my_decorator(say_hello)
```

### Comparison with other languages

```
Java @Override, @Deprecated, @Transactional:
  → Annotations — metadata markers, processed by framework/compiler
  → Can't modify the function's behavior directly

JS/TS decorators (@injectable, @Component):
  → Closer to Python's, but still mostly metadata + framework magic
  → TC39 Stage 3, not widely used outside frameworks

Python decorators:
  → Actual functions that wrap other functions
  → YOU write the behavior — no framework needed
  → Much more powerful and flexible
```

---

## 2. Building Decorators Step by Step

### Step 1: Basic decorator (no arguments to wrapper)

```python
def simple_decorator(func):
    def wrapper():
        print(f"Calling {func.__name__}")
        result = func()
        print(f"Done with {func.__name__}")
        return result
    return wrapper

@simple_decorator
def greet():
    print("Hello!")
    return "greeting sent"

greet()
# Calling greet
# Hello!
# Done with greet
```

### Step 2: Handle any function signature with *args/**kwargs

```python
def log_calls(func):
    def wrapper(*args, **kwargs):
        """Wrapper that passes through ALL arguments."""
        print(f"→ {func.__name__}({args}, {kwargs})")
        result = func(*args, **kwargs)
        print(f"← {func.__name__} returned {result}")
        return result
    return wrapper

@log_calls
def add(a, b):
    return a + b

@log_calls
def greet(name, excited=False):
    return f"{'HI' if excited else 'Hello'}, {name}!"

add(3, 5)
# → add((3, 5), {})
# ← add returned 8

greet("Alice", excited=True)
# → greet(('Alice',), {'excited': True})
# ← greet returned HI, Alice!
```

### Step 3: Preserve function metadata with `functools.wraps`

```python
from functools import wraps

# ❌ Without @wraps — metadata is lost
def bad_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@bad_decorator
def hello():
    """Say hello."""
    pass

print(hello.__name__)  # "wrapper" — WRONG! Lost the original name
print(hello.__doc__)   # None — WRONG! Lost the docstring

# ✅ With @wraps — metadata preserved
def good_decorator(func):
    @wraps(func)  # copies __name__, __doc__, __module__, etc.
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@good_decorator
def hello():
    """Say hello."""
    pass

print(hello.__name__)  # "hello" ✅
print(hello.__doc__)   # "Say hello." ✅

# RULE: ALWAYS use @wraps in your decorators. No exceptions.
```

### The complete decorator template

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        # --- before ---
        result = func(*args, **kwargs)
        # --- after ---
        return result
    return wrapper
```

This is your go-to template. Memorize it.

---

## 3. Practical Decorators

### Timer

```python
import time
from functools import wraps

def timer(func):
    """Measure execution time of a function."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "done"

slow_function()  # slow_function took 1.0012s
```

### Retry

```python
import time
from functools import wraps

def retry(max_attempts=3, delay=1.0, exceptions=(Exception,)):
    """Retry a function on failure."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    print(f"Attempt {attempt}/{max_attempts} failed: {e}")
                    if attempt < max_attempts:
                        time.sleep(delay)
            raise last_exception
        return wrapper
    return decorator

@retry(max_attempts=3, delay=0.5, exceptions=(ConnectionError, TimeoutError))
def fetch_data(url):
    # might fail due to network issues
    import random
    if random.random() < 0.7:
        raise ConnectionError("Server unavailable")
    return {"data": "success"}
```

### Cache / Memoize

```python
from functools import wraps

def memoize(func):
    """Cache function results based on arguments."""
    cache = {}
    
    @wraps(func)
    def wrapper(*args, **kwargs):
        # Create a hashable key from args and kwargs
        key = (args, tuple(sorted(kwargs.items())))
        if key not in cache:
            cache[key] = func(*args, **kwargs)
        return cache[key]
    
    wrapper.cache = cache          # expose cache for inspection
    wrapper.cache_clear = cache.clear  # allow clearing
    return wrapper

@memoize
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print(fibonacci(100))  # instant!
print(fibonacci.cache)  # see what's cached

# In production, use functools.lru_cache instead:
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

### Validate arguments

```python
from functools import wraps

def validate_types(**expected_types):
    """Validate argument types at runtime."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Check kwargs
            for param, expected in expected_types.items():
                if param in kwargs:
                    value = kwargs[param]
                    if not isinstance(value, expected):
                        raise TypeError(
                            f"Argument '{param}' expected {expected.__name__}, "
                            f"got {type(value).__name__}"
                        )
            return func(*args, **kwargs)
        return wrapper
    return decorator

@validate_types(name=str, age=int)
def create_user(name, age):
    return {"name": name, "age": age}

create_user(name="Alice", age=25)   # ✅
# create_user(name="Alice", age="25")  # ❌ TypeError
```

### Authentication / Authorization

```python
from functools import wraps

def require_auth(func):
    """Check if user is authenticated before calling function."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        # In real code, check session/token/header
        user = get_current_user()
        if not user:
            raise PermissionError("Authentication required")
        return func(*args, **kwargs)
    return wrapper

def require_role(*roles):
    """Check if user has required role."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            user = get_current_user()
            if user.role not in roles:
                raise PermissionError(f"Requires one of: {roles}")
            return func(*args, **kwargs)
        return wrapper
    return decorator

@require_auth
@require_role("admin", "moderator")
def delete_user(user_id: int):
    """Only admins and moderators can delete users."""
    pass

# Stacking decorators — execution order is BOTTOM UP:
# delete_user = require_auth(require_role("admin", "moderator")(delete_user))
# 1. require_role checks role
# 2. require_auth checks authentication
# Think of it as: outermost decorator runs first
```

### Logging decorator

```python
import logging
from functools import wraps

def log(level=logging.INFO):
    """Log function calls with configurable level."""
    def decorator(func):
        logger = logging.getLogger(func.__module__)
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            logger.log(level, f"Calling {func.__name__}")
            try:
                result = func(*args, **kwargs)
                logger.log(level, f"{func.__name__} returned successfully")
                return result
            except Exception as e:
                logger.exception(f"{func.__name__} raised {type(e).__name__}: {e}")
                raise
        return wrapper
    return decorator

@log(level=logging.DEBUG)
def process_data(data):
    return [x * 2 for x in data]
```

---

## 4. Decorators with Arguments — The Three-Level Pattern

This is the trickiest part. When a decorator takes arguments, you need **three nested functions**.

```python
# Decorator WITHOUT arguments — two levels
def simple(func):           # Level 1: receives the function
    @wraps(func)
    def wrapper(*a, **kw):  # Level 2: receives the function's args
        return func(*a, **kw)
    return wrapper

@simple         # No parentheses!
def my_func(): ...


# Decorator WITH arguments — three levels
def with_args(arg1, arg2):    # Level 1: receives DECORATOR's args
    def decorator(func):      # Level 2: receives the function
        @wraps(func)
        def wrapper(*a, **kw): # Level 3: receives the function's args
            print(f"Args: {arg1}, {arg2}")
            return func(*a, **kw)
        return wrapper
    return decorator

@with_args("hello", 42)  # WITH parentheses — calls level 1 first!
def my_func(): ...

# What happens:
# 1. with_args("hello", 42) is called → returns decorator
# 2. decorator(my_func) is called → returns wrapper
# 3. my_func is now wrapper
```

### Optional arguments pattern — both `@deco` and `@deco(...)` work

```python
from functools import wraps

def repeat(_func=None, *, times=2):
    """Can be used as @repeat or @repeat(times=5)."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    
    if _func is not None:
        # Called as @repeat (without parentheses)
        return decorator(_func)
    # Called as @repeat(times=5) (with parentheses)
    return decorator

@repeat
def say_hi():
    print("Hi!")

@repeat(times=5)
def say_hello():
    print("Hello!")

say_hi()     # prints Hi! twice
say_hello()  # prints Hello! five times
```

---

## 5. Class-Based Decorators

Instead of nested functions, use a class with `__call__`. Cleaner for complex decorators with state.

```python
from functools import wraps

class CountCalls:
    """Track how many times a function is called."""
    
    def __init__(self, func):
        wraps(func)(self)  # preserve metadata
        self.func = func
        self.count = 0
    
    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"{self.func.__name__} called {self.count} time(s)")
        return self.func(*args, **kwargs)

@CountCalls
def process():
    return "done"

process()  # process called 1 time(s)
process()  # process called 2 time(s)
print(process.count)  # 2

# Class decorator WITH arguments
class RateLimit:
    """Limit function calls per time window."""
    
    def __init__(self, max_calls: int, period: float = 60.0):
        self.max_calls = max_calls
        self.period = period
        self.calls = []
    
    def __call__(self, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            import time
            now = time.time()
            # Remove expired calls
            self.calls = [t for t in self.calls if now - t < self.period]
            if len(self.calls) >= self.max_calls:
                raise RuntimeError(
                    f"Rate limit exceeded: {self.max_calls} calls per {self.period}s"
                )
            self.calls.append(now)
            return func(*args, **kwargs)
        return wrapper

@RateLimit(max_calls=5, period=60)
def api_call():
    return "response"
```

---

## 6. Decorating Classes

Decorators can also wrap entire classes.

```python
from functools import wraps
from datetime import datetime

def add_timestamp(cls):
    """Add created_at timestamp to any class."""
    original_init = cls.__init__
    
    @wraps(original_init)
    def new_init(self, *args, **kwargs):
        original_init(self, *args, **kwargs)
        self.created_at = datetime.now()
    
    cls.__init__ = new_init
    return cls

@add_timestamp
class User:
    def __init__(self, name: str):
        self.name = name

user = User("Alice")
print(user.created_at)  # 2025-02-14 12:34:56.789

# You've already seen the most famous class decorator:
from dataclasses import dataclass

@dataclass  # this is a class decorator!
class Point:
    x: float
    y: float

# Other built-in class decorators you know:
# @dataclass(frozen=True)
# @functools.total_ordering
```

### Singleton decorator

```python
def singleton(cls):
    """Ensure only one instance of a class exists."""
    instances = {}
    
    @wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    
    return get_instance

@singleton
class Database:
    def __init__(self, url: str):
        self.url = url
        print(f"Connecting to {url}...")

db1 = Database("postgres://localhost/mydb")  # Connecting to postgres://...
db2 = Database("postgres://localhost/other")  # NOT called — returns existing
print(db1 is db2)  # True
```

---

## 7. Stacking Decorators

```python
@decorator_a
@decorator_b
@decorator_c
def my_function():
    pass

# Equivalent to:
# my_function = decorator_a(decorator_b(decorator_c(my_function)))

# Execution order:
# decorator_c wraps first (innermost)
# decorator_b wraps second
# decorator_a wraps third (outermost)
#
# When my_function() is called:
# decorator_a's wrapper runs first
# then decorator_b's wrapper
# then decorator_c's wrapper
# then the original function

# Example:
@timer
@retry(max_attempts=3)
@log()
def fetch_data(url):
    pass

# Call flow:
# 1. timer starts timing
# 2. retry attempts the call
# 3. log logs the call
# 4. fetch_data runs
# 5. log logs the result
# 6. retry handles any exception
# 7. timer reports elapsed time
```

---

## 8. Real-World Decorator Patterns for Data Engineering

```python
from functools import wraps
import time
import logging

logger = logging.getLogger(__name__)

def pipeline_step(step_name: str):
    """Decorator for data pipeline steps — logs, times, handles errors."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            logger.info(f"[{step_name}] Starting...")
            start = time.perf_counter()
            try:
                result = func(*args, **kwargs)
                elapsed = time.perf_counter() - start
                
                # Log row count if result is a list/iterable with len
                count = len(result) if hasattr(result, '__len__') else '?'
                logger.info(f"[{step_name}] Done in {elapsed:.2f}s — {count} records")
                return result
            except Exception as e:
                elapsed = time.perf_counter() - start
                logger.error(f"[{step_name}] Failed after {elapsed:.2f}s: {e}")
                raise
        return wrapper
    return decorator

@pipeline_step("extract")
def extract_data(source: str) -> list[dict]:
    """Extract data from source."""
    return [{"id": i, "value": i * 10} for i in range(1000)]

@pipeline_step("transform")
def transform_data(raw: list[dict]) -> list[dict]:
    """Transform raw data."""
    return [{"id": r["id"], "value_doubled": r["value"] * 2} for r in raw]

@pipeline_step("load")
def load_data(data: list[dict], target: str) -> list[dict]:
    """Load data to target."""
    return data

# Run pipeline
raw = extract_data("api://data")
transformed = transform_data(raw)
load_data(transformed, "postgres://db/table")
# [extract] Starting...
# [extract] Done in 0.01s — 1000 records
# [transform] Starting...
# [transform] Done in 0.00s — 1000 records
# [load] Starting...
# [load] Done in 0.00s — 1000 records
```

---

## 9. Context Managers — The `with` Statement

Context managers guarantee cleanup, even if exceptions occur.

### Basic usage (you've seen this)

```python
# File handling — most common use
with open("data.txt", "r") as f:
    content = f.read()
# f.close() is called automatically — even if an exception occurs

# Without context manager — risky:
f = open("data.txt")
try:
    content = f.read()
finally:
    f.close()

# Java equivalent:
# try (var reader = new BufferedReader(new FileReader("data.txt"))) { ... }
```

### How it works — the protocol

```python
# A context manager is any object with __enter__ and __exit__

class ManagedFile:
    def __init__(self, filename: str, mode: str = "r"):
        self.filename = filename
        self.mode = mode
        self.file = None
    
    def __enter__(self):
        """Called when entering 'with' block. Returns the managed resource."""
        print(f"Opening {self.filename}")
        self.file = open(self.filename, self.mode)
        return self.file  # this is what 'as f' receives
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """Called when exiting 'with' block — ALWAYS, even on exception.
        
        Args:
            exc_type: Exception class (or None if no exception)
            exc_val:  Exception instance (or None)
            exc_tb:   Traceback (or None)
        
        Returns:
            True to suppress the exception, False/None to propagate it
        """
        print(f"Closing {self.filename}")
        if self.file:
            self.file.close()
        
        if exc_type is not None:
            print(f"Exception occurred: {exc_type.__name__}: {exc_val}")
        
        return False  # Don't suppress exceptions (almost always want False)

with ManagedFile("test.txt", "w") as f:
    f.write("Hello!")
# Opening test.txt
# Closing test.txt
```

### Exception handling in `__exit__`

```python
class SafeDB:
    def __enter__(self):
        self.connection = create_connection()
        self.transaction = self.connection.begin()
        return self.connection
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            # Exception occurred — rollback
            self.transaction.rollback()
            print(f"Rolled back due to: {exc_val}")
        else:
            # Success — commit
            self.transaction.commit()
        self.connection.close()
        return False  # propagate exceptions

# Usage — auto commit/rollback
with SafeDB() as db:
    db.execute("INSERT INTO users ...")
    db.execute("UPDATE accounts ...")
    # If any line raises → rollback
    # If all succeed → commit
```

---

## 10. `contextlib` — Easy Context Managers

### `@contextmanager` — write context managers as generators

Instead of writing a full class with `__enter__`/`__exit__`, use a generator:

```python
from contextlib import contextmanager

@contextmanager
def managed_file(filename, mode="r"):
    """Everything before yield = __enter__
       The yielded value = what 'as' receives
       Everything after yield = __exit__"""
    print(f"Opening {filename}")
    f = open(filename, mode)
    try:
        yield f  # <-- this is where the 'with' block runs
    finally:
        print(f"Closing {filename}")
        f.close()

with managed_file("test.txt", "w") as f:
    f.write("Hello!")

# The try/finally ensures cleanup even on exceptions
# This is MUCH more concise than the class approach
```

### Practical context managers

```python
from contextlib import contextmanager
import time
import os
import tempfile

# Timer
@contextmanager
def timer(label="Operation"):
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        print(f"{label} took {elapsed:.4f}s")

with timer("Data processing"):
    data = [x ** 2 for x in range(1_000_000)]

# Temporary directory
@contextmanager
def temp_workspace():
    """Create a temporary directory, clean up after."""
    original_dir = os.getcwd()
    with tempfile.TemporaryDirectory() as tmpdir:
        os.chdir(tmpdir)
        try:
            yield tmpdir
        finally:
            os.chdir(original_dir)

with temp_workspace() as workspace:
    print(f"Working in: {workspace}")
    # create files, do work...
# directory is automatically deleted

# Change environment variable temporarily
@contextmanager
def env_var(key: str, value: str):
    """Temporarily set an environment variable."""
    old_value = os.environ.get(key)
    os.environ[key] = value
    try:
        yield
    finally:
        if old_value is None:
            del os.environ[key]
        else:
            os.environ[key] = old_value

with env_var("DEBUG", "true"):
    print(os.environ["DEBUG"])  # "true"
print(os.environ.get("DEBUG"))  # None (restored)

# Database transaction
@contextmanager
def transaction(connection):
    """Auto commit/rollback pattern."""
    tx = connection.begin()
    try:
        yield connection
        tx.commit()
    except Exception:
        tx.rollback()
        raise
```

### `contextlib` utilities

```python
from contextlib import suppress, redirect_stdout, closing, ExitStack
import io

# suppress — ignore specific exceptions
with suppress(FileNotFoundError):
    os.remove("might_not_exist.txt")
# No error even if file doesn't exist
# Cleaner than try/except: pass

# redirect_stdout — capture print output
f = io.StringIO()
with redirect_stdout(f):
    print("This goes to the string buffer")
output = f.getvalue()  # "This goes to the string buffer\n"

# closing — add close() to objects that don't support 'with'
from contextlib import closing
from urllib.request import urlopen

with closing(urlopen("https://example.com")) as page:
    content = page.read()

# ExitStack — manage multiple context managers dynamically
with ExitStack() as stack:
    files = [
        stack.enter_context(open(f"file_{i}.txt", "w"))
        for i in range(5)
    ]
    # All 5 files will be closed when exiting the block
    for i, f in enumerate(files):
        f.write(f"Content for file {i}")
```

---

## 11. Combining Decorators and Context Managers

```python
from contextlib import contextmanager
from functools import wraps
import time
import logging

logger = logging.getLogger(__name__)

# A decorator that also works as a context manager!
class track_time:
    """Use as @track_time or with track_time('label'):"""
    
    def __init__(self, label_or_func=None):
        if callable(label_or_func):
            # Used as @track_time (without parentheses)
            self.func = label_or_func
            self.label = label_or_func.__name__
            wraps(label_or_func)(self)
        else:
            # Used as @track_time("label") or with track_time("label"):
            self.func = None
            self.label = label_or_func or "operation"
    
    def __call__(self, *args, **kwargs):
        if self.func is not None:
            # Acting as decorator
            start = time.perf_counter()
            result = self.func(*args, **kwargs)
            elapsed = time.perf_counter() - start
            logger.info(f"{self.label} took {elapsed:.4f}s")
            return result
        else:
            # Acting as decorator factory — args[0] is the function
            func = args[0]
            self.func = func
            self.label = self.label or func.__name__
            wraps(func)(self)
            return self
    
    def __enter__(self):
        self.start = time.perf_counter()
        return self
    
    def __exit__(self, *exc):
        elapsed = time.perf_counter() - self.start
        logger.info(f"{self.label} took {elapsed:.4f}s")
        return False

# All three usages work:
@track_time
def process_a():
    time.sleep(0.1)

@track_time("custom label")
def process_b():
    time.sleep(0.1)

with track_time("inline block"):
    time.sleep(0.1)
```

---

## 12. Async Context Managers (Preview for Day 12)

```python
import asyncio
from contextlib import asynccontextmanager

@asynccontextmanager
async def async_db_connection(url: str):
    """Async context manager for database connections."""
    print(f"Connecting to {url}...")
    connection = await create_async_connection(url)
    try:
        yield connection
    finally:
        await connection.close()
        print("Connection closed")

# Usage:
# async with async_db_connection("postgres://...") as db:
#     result = await db.execute("SELECT * FROM users")

# Also works as class:
class AsyncTimer:
    async def __aenter__(self):
        self.start = time.perf_counter()
        return self
    
    async def __aexit__(self, *exc):
        elapsed = time.perf_counter() - self.start
        print(f"Took {elapsed:.4f}s")
```

---

## 13. Built-in Decorators You Should Know

```python
# @property — getter/setter (Day 4)
class User:
    @property
    def name(self): ...
    
    @name.setter
    def name(self, value): ...

# @staticmethod, @classmethod (Day 4)
class MyClass:
    @staticmethod
    def utility(): ...
    
    @classmethod
    def factory(cls): ...

# @dataclass (Day 4)
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

# @abstractmethod (Day 4)
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self): ...

# @functools.wraps — preserve metadata (this day)
# @functools.lru_cache — memoization (Day 2)
# @functools.total_ordering — auto comparison methods (Day 4)
# @functools.singledispatch — function overloading by type

from functools import singledispatch

@singledispatch
def process(data):
    raise TypeError(f"Unsupported type: {type(data)}")

@process.register(str)
def _(data):
    return f"Processing string: {data}"

@process.register(list)
def _(data):
    return f"Processing list of {len(data)} items"

@process.register(dict)
def _(data):
    return f"Processing dict with keys: {list(data.keys())}"

print(process("hello"))         # Processing string: hello
print(process([1, 2, 3]))      # Processing list of 3 items
print(process({"a": 1}))       # Processing dict with keys: ['a']

# This is Python's version of method overloading!
# Java: void process(String s), void process(List l)
# Python: @singledispatch
```

---

## 📝 Practice Tasks

### Task 1: Debug Decorator
Write a `@debug` decorator that prints: function name, all arguments (positional and keyword) with their values, return value, and execution time. Use `@wraps`.

### Task 2: Retry with Exponential Backoff
Enhance the retry decorator to support exponential backoff: first retry after 1s, second after 2s, third after 4s, etc. Add a `max_delay` parameter to cap the wait time.

### Task 3: Cache with Expiration
Write a `@cache(ttl=60)` decorator that caches results but expires them after `ttl` seconds. Include a `.cache_info()` method that shows hits, misses, and current cache size.

### Task 4: Database Context Manager
Write a context manager (using `@contextmanager`) that:
- Opens a SQLite connection on enter
- Provides a cursor
- Auto-commits on success
- Auto-rollbacks on exception
- Always closes the connection

```python
with database("my.db") as cursor:
    cursor.execute("CREATE TABLE IF NOT EXISTS users (name TEXT, age INT)")
    cursor.execute("INSERT INTO users VALUES (?, ?)", ("Alice", 25))
    # auto-commit if no exception
```

### Task 5: Pipeline Framework
Build a mini data pipeline framework using decorators:
```python
pipeline = Pipeline()

@pipeline.step("extract")
def extract():
    return [{"name": "Alice"}, {"name": "Bob"}]

@pipeline.step("transform")
def transform(data):
    return [{"name": d["name"].upper()} for d in data]

@pipeline.step("load")
def load(data):
    print(f"Loaded {len(data)} records")

pipeline.run()  # executes steps in order, passing data between them
```

### Task 6: Access Control Decorator
Write decorators for a permission system:
```python
@require_auth
@require_permission("admin:write")
@rate_limit(max_calls=10, period=60)
def delete_all_users():
    pass
```
Implement all three decorators. Use a simple dict as a "session store" for the current user.

---

## 📚 Resources

- [Real Python — Primer on Decorators](https://realpython.com/primer-on-python-decorators/)
- [Real Python — Context Managers](https://realpython.com/python-with-statement/)
- [Python Official — Decorators](https://docs.python.org/3/glossary.html#term-decorator)
- [Python Official — contextlib](https://docs.python.org/3/library/contextlib.html)
- [Real Python — functools](https://realpython.com/python-functools/)
- [PEP 318 — Decorators for Functions](https://peps.python.org/pep-0318/)
- [PEP 343 — The "with" Statement](https://peps.python.org/pep-0343/)

---

## 🔑 Key Takeaways

1. **Decorators are just closures** — a function that wraps another function. The `@` syntax is sugar.
2. **Always use `@wraps`** — without it, you lose `__name__`, `__doc__`, and debugging breaks.
3. **Three-level nesting** — decorators with arguments need: args → function → wrapper.
4. **Class-based decorators** — use `__call__` for stateful decorators.
5. **Context managers guarantee cleanup** — use `with` for files, connections, locks, transactions.
6. **`@contextmanager`** — write context managers as generators (much simpler than class approach).
7. **`suppress`** — cleaner than `try/except: pass`.
8. **`ExitStack`** — manage dynamic number of context managers.
9. **Stacking order** — bottom decorator wraps first, top decorator's wrapper runs first.
10. **Decorators + context managers** — combine them for powerful pipeline instrumentation.

---

> **Tomorrow (Day 8):** File Handling & Serialization — reading/writing text, CSV, JSON, YAML, binary files, `pathlib`, and processing large files efficiently. Essential for Data Engineering data ingestion.

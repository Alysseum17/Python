# Day 7: Decorators & Context Managers

## 🎯 Goal

Master Python's "magic" syntax for wrapping functionality. Decorators and context managers are everywhere in Python frameworks (Flask, FastAPI, pytest) and are essential for clean, maintainable code.

---

## 1. Functions as First-Class Citizens (Quick Review)

Before decorators, remember that in Python functions are objects:

```python
def greet(name):
    return f"Hello, {name}!"

# Functions can be assigned to variables
say_hello = greet
print(say_hello("Alice"))  # Hello, Alice!

# Functions can be passed as arguments
def execute(func, arg):
    return func(arg)

print(execute(greet, "Bob"))  # Hello, Bob!

# Functions can return other functions
def make_multiplier(n):
    def multiply(x):
        return x * n
    return multiply

times_three = make_multiplier(3)
print(times_three(10))  # 30
```

This is the foundation for decorators.

---

## 2. What Are Decorators?

**Decorators wrap a function to modify its behavior without changing its code.**

Think of it like this:
- You have a function that does something
- You want to add behavior (logging, timing, validation, etc.)
- You don't want to modify the original function
- You "decorate" it with extra functionality

### The Manual Way (Before Understanding @syntax)

```python
def my_function():
    print("Hello!")

# Wrap it manually
def add_logging(func):
    def wrapper():
        print("Function is about to run")
        func()  # Call the original function
        print("Function finished")
    return wrapper

# Decorate manually
my_function = add_logging(my_function)
my_function()
# Output:
# Function is about to run
# Hello!
# Function finished
```

### The @ Syntax (Syntactic Sugar)

Python's `@decorator` is just shorthand for the above:

```python
def add_logging(func):
    def wrapper():
        print("Function is about to run")
        func()
        print("Function finished")
    return wrapper

@add_logging  # Same as: my_function = add_logging(my_function)
def my_function():
    print("Hello!")

my_function()
# Output:
# Function is about to run
# Hello!
# Function finished
```

**The `@` syntax is applied at function definition time, not call time!**

---

## 3. Building Your First Decorator (Step by Step)

### Step 1: Basic decorator (no arguments to decorated function)

```python
def timer(func):
    """Decorator that times function execution"""
    def wrapper():
        import time
        start = time.time()
        func()  # Call original function
        end = time.time()
        print(f"{func.__name__} took {end - start:.4f} seconds")
    return wrapper

@timer
def slow_function():
    import time
    time.sleep(1)
    print("Done!")

slow_function()
# Output:
# Done!
# slow_function took 1.0001 seconds
```

### Step 2: Handle functions with arguments

**Problem:** What if the decorated function takes arguments?

```python
# This breaks!
@timer
def greet(name):  # This function needs an argument
    print(f"Hello, {name}!")

greet("Alice")  # ❌ TypeError: wrapper() takes 0 positional arguments but 1 was given
```

**Solution:** Use `*args` and `**kwargs` in wrapper

```python
def timer(func):
    def wrapper(*args, **kwargs):  # Accept any arguments
        import time
        start = time.time()
        result = func(*args, **kwargs)  # Pass them through
        end = time.time()
        print(f"{func.__name__} took {end - start:.4f} seconds")
        return result  # Don't forget to return!
    return wrapper

@timer
def greet(name):
    print(f"Hello, {name}!")
    return "Done"

result = greet("Alice")
# Output:
# Hello, Alice!
# greet took 0.0001 seconds
print(result)  # Done
```

### Step 3: Preserve function metadata with `functools.wraps`

**Problem:** Decorated functions lose their original name and docstring

```python
def timer(func):
    def wrapper(*args, **kwargs):
        """Wrapper function"""
        import time
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} took {end - start:.4f} seconds")
        return result
    return wrapper

@timer
def greet(name):
    """Greets a person by name"""
    print(f"Hello, {name}!")

print(greet.__name__)  # wrapper (not greet!)
print(greet.__doc__)   # Wrapper function (not the original docstring!)
```

**Solution:** Use `@functools.wraps` inside your decorator

```python
from functools import wraps

def timer(func):
    @wraps(func)  # Preserves func's metadata
    def wrapper(*args, **kwargs):
        import time
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} took {end - start:.4f} seconds")
        return result
    return wrapper

@timer
def greet(name):
    """Greets a person by name"""
    print(f"Hello, {name}!")

print(greet.__name__)  # greet ✅
print(greet.__doc__)   # Greets a person by name ✅
```

**Always use `@wraps(func)` in your decorators!** This is the standard pattern.

---

## 4. Common Decorator Patterns

### Pattern 1: Timing/Profiling

```python
from functools import wraps
import time

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} took {end - start:.4f}s")
        return result
    return wrapper

@timer
def fetch_data():
    time.sleep(0.5)
    return [1, 2, 3]

data = fetch_data()  # fetch_data took 0.5001s
```

### Pattern 2: Logging

```python
import logging
from functools import wraps

logging.basicConfig(level=logging.INFO)

def log_call(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        logging.info(f"Calling {func.__name__} with args={args}, kwargs={kwargs}")
        result = func(*args, **kwargs)
        logging.info(f"{func.__name__} returned {result}")
        return result
    return wrapper

@log_call
def add(a, b):
    return a + b

add(3, 5)
# INFO:root:Calling add with args=(3, 5), kwargs={}
# INFO:root:add returned 8
```

### Pattern 3: Validation

```python
from functools import wraps

def validate_positive(func):
    @wraps(func)
    def wrapper(n):
        if n <= 0:
            raise ValueError(f"{func.__name__} requires positive number, got {n}")
        return func(n)
    return wrapper

@validate_positive
def square_root(n):
    return n ** 0.5

print(square_root(16))  # 4.0
# square_root(-4)       # ValueError: square_root requires positive number, got -4
```

### Pattern 4: Retry Logic

```python
from functools import wraps
import time

def retry(max_attempts=3, delay=1):
    """Retry decorator - we'll see how this works in next section!"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    print(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay}s...")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry(max_attempts=3, delay=0.5)
def flaky_api_call():
    import random
    if random.random() < 0.7:  # 70% chance of failure
        raise ConnectionError("API timeout")
    return "Success!"

# Will retry up to 3 times
result = flaky_api_call()
```

### Pattern 5: Caching/Memoization

```python
from functools import wraps

def memoize(func):
    cache = {}  # Closure variable - persists between calls
    
    @wraps(func)
    def wrapper(*args):
        if args not in cache:
            print(f"Computing {func.__name__}{args}")
            cache[args] = func(*args)
        else:
            print(f"Using cached result for {func.__name__}{args}")
        return cache[args]
    return wrapper

@memoize
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print(fibonacci(5))
# Computing fibonacci(5)
# Computing fibonacci(4)
# Computing fibonacci(3)
# Computing fibonacci(2)
# Computing fibonacci(1)
# Computing fibonacci(0)
# Using cached result for fibonacci(1)
# Using cached result for fibonacci(2)
# Using cached result for fibonacci(3)
# 5

# Standard library has this: functools.lru_cache
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci_fast(n):
    if n < 2:
        return n
    return fibonacci_fast(n - 1) + fibonacci_fast(n - 2)
```

---

## 5. Decorators with Arguments (Decorator Factories)

**This is where it gets tricky!** When you want to pass arguments to the decorator itself:

```python
@retry(max_attempts=3, delay=1)  # How does this work?
def my_function():
    pass
```

### Understanding the Three Layers

When a decorator has arguments, you need **three nested functions**:

```python
def decorator_with_args(arg1, arg2):  # Layer 1: Takes decorator arguments
    def decorator(func):               # Layer 2: Takes the function to decorate
        @wraps(func)
        def wrapper(*args, **kwargs):  # Layer 3: Replaces the function
            # Use arg1, arg2, func, args, kwargs here
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

Let's build this step by step:

### Step 1: Simple decorator (no arguments)

```python
def repeat(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        func(*args, **kwargs)
        func(*args, **kwargs)  # Always repeats twice
    return wrapper

@repeat
def greet(name):
    print(f"Hello, {name}!")

greet("Alice")
# Hello, Alice!
# Hello, Alice!
```

### Step 2: Make it configurable (add decorator arguments)

```python
def repeat(times):  # Now this takes the number of times
    def decorator(func):  # This takes the actual function
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):  # Use 'times' from outer scope
                func(*args, **kwargs)
        return wrapper
    return decorator  # Return the actual decorator

@repeat(times=3)  # repeat(3) returns the decorator, which decorates greet
def greet(name):
    print(f"Hello, {name}!")

greet("Alice")
# Hello, Alice!
# Hello, Alice!
# Hello, Alice!
```

### How it actually works (mental model)

```python
# When Python sees:
@repeat(times=3)
def greet(name):
    print(f"Hello, {name}!")

# It does this:
# Step 1: Call repeat(times=3), which returns 'decorator' function
decorator_func = repeat(times=3)

# Step 2: Pass greet to that decorator function
greet = decorator_func(greet)

# Now greet is actually 'wrapper' that repeats 3 times
```

### Real-world example: Authentication decorator

```python
from functools import wraps

def require_auth(role="user"):
    """Decorator that checks if user has required role"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # In real app, you'd get this from session/token
            current_user_role = kwargs.get('user_role', 'guest')
            
            if current_user_role == 'admin' or current_user_role == role:
                return func(*args, **kwargs)
            else:
                raise PermissionError(f"Requires {role} role, you are {current_user_role}")
        return wrapper
    return decorator

@require_auth(role="admin")
def delete_user(user_id, user_role=None):
    print(f"Deleting user {user_id}")

@require_auth(role="user")
def view_profile(user_id, user_role=None):
    print(f"Viewing profile {user_id}")

# Usage
view_profile(123, user_role="user")      # ✅ Works
delete_user(123, user_role="admin")      # ✅ Works
# delete_user(123, user_role="user")     # ❌ PermissionError
```

---

## 6. Stacking Decorators

You can apply multiple decorators to the same function:

```python
from functools import wraps
import time

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"Time: {time.time() - start:.4f}s")
        return result
    return wrapper

def log_call(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Finished {func.__name__}")
        return result
    return wrapper

@timer       # Applied second (outer)
@log_call    # Applied first (inner)
def greet(name):
    time.sleep(0.1)
    print(f"Hello, {name}!")

greet("Alice")
# Output:
# Calling greet
# Hello, Alice!
# Finished greet
# Time: 0.1001s
```

**Order matters!** Decorators are applied bottom-to-top:

```python
@timer
@log_call
def greet(name):
    pass

# Is equivalent to:
greet = timer(log_call(greet))
```

---

## 7. Class-Based Decorators

You can also use classes as decorators (less common but useful for complex decorators):

```python
from functools import wraps

class CountCalls:
    def __init__(self, func):
        self.func = func
        self.count = 0
        wraps(func)(self)  # Preserve metadata
    
    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"{self.func.__name__} has been called {self.count} times")
        return self.func(*args, **kwargs)

@CountCalls
def greet(name):
    print(f"Hello, {name}!")

greet("Alice")  # greet has been called 1 times
greet("Bob")    # greet has been called 2 times
greet("Charlie")# greet has been called 3 times
print(greet.count)  # 3
```

With arguments (even more complex):

```python
class Repeat:
    def __init__(self, times):
        self.times = times
    
    def __call__(self, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(self.times):
                func(*args, **kwargs)
        return wrapper

@Repeat(times=3)
def greet(name):
    print(f"Hello, {name}!")

greet("Alice")
# Hello, Alice!
# Hello, Alice!
# Hello, Alice!
```

---

## 8. Context Managers: The `with` Statement

**Context managers handle setup and teardown automatically.** Most common use: file handling.

### The Problem

```python
# Without context manager (manual cleanup)
file = open('data.txt', 'w')
try:
    file.write('Hello')
finally:
    file.close()  # Must remember to close, even if error occurs!
```

### The Solution: `with` statement

```python
# With context manager (automatic cleanup)
with open('data.txt', 'w') as file:
    file.write('Hello')
# file.close() is called automatically, even if error occurs!
```

**The `with` statement guarantees cleanup**, similar to Java's try-with-resources or C#'s `using`.

---

## 9. Creating Context Managers (Two Ways)

### Method 1: Class-Based (The Protocol)

A context manager is any class that implements two methods:
- `__enter__()` - called when entering the `with` block
- `__exit__()` - called when exiting the `with` block (even if error)

```python
class DatabaseConnection:
    def __init__(self, db_name):
        self.db_name = db_name
        self.connection = None
    
    def __enter__(self):
        print(f"Opening connection to {self.db_name}")
        self.connection = f"Connection to {self.db_name}"
        return self.connection  # This is assigned to 'as' variable
    
    def __exit__(self, exc_type, exc_value, traceback):
        # exc_type, exc_value, traceback are None if no error occurred
        # If error occurred, they contain exception info
        print(f"Closing connection to {self.db_name}")
        self.connection = None
        
        # Return False to propagate exception, True to suppress it
        return False

# Usage
with DatabaseConnection("mydb") as conn:
    print(f"Using {conn}")
    # If we raise an exception here, __exit__ still runs!

# Output:
# Opening connection to mydb
# Using Connection to mydb
# Closing connection to mydb
```

### Understanding `__exit__` parameters

```python
class ErrorHandler:
    def __enter__(self):
        print("Entering")
        return self
    
    def __exit__(self, exc_type, exc_value, traceback):
        if exc_type is None:
            print("No error occurred")
        else:
            print(f"Error occurred: {exc_type.__name__}: {exc_value}")
        
        # Return True to suppress the exception
        # Return False (or None) to let it propagate
        return False  # Let errors propagate

with ErrorHandler():
    print("Inside block")
    # raise ValueError("Something went wrong")  # Uncomment to see error handling
```

### Method 2: Function-Based (Using `contextlib`)

Much simpler! Use the `@contextmanager` decorator:

```python
from contextlib import contextmanager

@contextmanager
def database_connection(db_name):
    # Setup (before yield)
    print(f"Opening connection to {db_name}")
    connection = f"Connection to {db_name}"
    
    try:
        yield connection  # Value given to 'as' variable
    finally:
        # Cleanup (after yield)
        print(f"Closing connection to {db_name}")

# Usage - exactly the same!
with database_connection("mydb") as conn:
    print(f"Using {conn}")

# Output:
# Opening connection to mydb
# Using Connection to mydb
# Closing connection to mydb
```

**How it works:**
1. Code before `yield` = `__enter__` (setup)
2. `yield value` = value returned by `__enter__`
3. Code after `yield` = `__exit__` (cleanup)
4. `finally` ensures cleanup even if exception occurs

---

## 10. Real-World Context Manager Examples

### Example 1: Timer context manager

```python
import time
from contextlib import contextmanager

@contextmanager
def timer(label):
    start = time.time()
    try:
        yield  # No value needed
    finally:
        end = time.time()
        print(f"{label}: {end - start:.4f}s")

with timer("Database query"):
    time.sleep(0.5)
    # Query database here

# Output: Database query: 0.5001s
```

### Example 2: Temporary directory

```python
from contextlib import contextmanager
import os
import tempfile
import shutil

@contextmanager
def temporary_directory():
    """Create a temporary directory, yield it, then delete it"""
    temp_dir = tempfile.mkdtemp()
    try:
        yield temp_dir
    finally:
        shutil.rmtree(temp_dir)

with temporary_directory() as tmp_dir:
    print(f"Working in {tmp_dir}")
    # Create files in tmp_dir
    with open(os.path.join(tmp_dir, "test.txt"), "w") as f:
        f.write("Hello!")
    
    # Check file exists
    print(os.path.exists(os.path.join(tmp_dir, "test.txt")))  # True

# After exiting, directory is deleted
```

### Example 3: Database transaction (very common pattern)

```python
from contextlib import contextmanager

@contextmanager
def transaction(db_connection):
    """Automatically commit or rollback database transaction"""
    try:
        yield db_connection
        db_connection.commit()  # Success - commit changes
        print("Transaction committed")
    except Exception as e:
        db_connection.rollback()  # Error - rollback changes
        print(f"Transaction rolled back due to: {e}")
        raise  # Re-raise the exception

# Usage
with transaction(db) as conn:
    conn.execute("INSERT INTO users VALUES (1, 'Alice')")
    conn.execute("INSERT INTO users VALUES (2, 'Bob')")
    # If any error occurs, both inserts are rolled back
```

### Example 4: Changing directory temporarily

```python
import os
from contextlib import contextmanager

@contextmanager
def change_dir(path):
    """Temporarily change working directory"""
    old_dir = os.getcwd()
    try:
        os.chdir(path)
        yield
    finally:
        os.chdir(old_dir)

print(f"Currently in: {os.getcwd()}")

with change_dir("/tmp"):
    print(f"Now in: {os.getcwd()}")
    # Do work in /tmp

print(f"Back in: {os.getcwd()}")
```

### Example 5: Suppressing specific exceptions

```python
from contextlib import suppress

# Instead of:
try:
    os.remove("file.txt")
except FileNotFoundError:
    pass  # It's okay if file doesn't exist

# Use suppress:
with suppress(FileNotFoundError):
    os.remove("file.txt")  # No error if file doesn't exist
```

---

## 11. Combining Decorators and Context Managers

You can use both together for powerful patterns:

```python
from contextlib import contextmanager
from functools import wraps
import time

@contextmanager
def timing_context(label):
    start = time.time()
    try:
        yield
    finally:
        print(f"{label}: {time.time() - start:.4f}s")

def timed_function(func):
    """Decorator that times function execution"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        with timing_context(func.__name__):
            return func(*args, **kwargs)
    return wrapper

@timed_function
def slow_operation():
    time.sleep(0.5)
    return "Done"

result = slow_operation()
# Output: slow_operation: 0.5001s
```

---

## 12. Common Patterns in Popular Frameworks

### FastAPI (web framework)

```python
from fastapi import FastAPI, Depends

app = FastAPI()

# Dependency injection using decorators
def get_db():
    db = connect_to_database()
    try:
        yield db  # This is a generator! Works like context manager
    finally:
        db.close()

@app.get("/users/{user_id}")  # Route decorator
async def get_user(user_id: int, db=Depends(get_db)):  # Depends is a decorator
    return db.query(f"SELECT * FROM users WHERE id={user_id}")
```

### Pytest (testing framework)

```python
import pytest

@pytest.fixture  # Decorator that creates reusable test fixtures
def sample_data():
    return [1, 2, 3, 4, 5]

@pytest.mark.parametrize("input,expected", [  # Decorator for parameterized tests
    (1, 2),
    (2, 4),
    (3, 6),
])
def test_double(input, expected):
    assert input * 2 == expected
```

### Flask (web framework)

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")  # Route decorator
def home():
    return "Hello, World!"

@app.route("/user/<name>")
def user(name):
    return f"Hello, {name}!"

# Custom decorator for login required
def login_required(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        if not user_is_logged_in():
            return "Please log in", 401
        return func(*args, **kwargs)
    return wrapper

@app.route("/dashboard")
@login_required
def dashboard():
    return "Welcome to your dashboard"
```

---

## 13. Advanced: contextlib Utilities

```python
from contextlib import (
    contextmanager,
    closing,
    suppress,
    redirect_stdout,
    ExitStack
)
import io

# closing: ensures .close() is called
from urllib.request import urlopen
with closing(urlopen("http://example.com")) as page:
    content = page.read()
# page.close() called automatically

# suppress: ignore specific exceptions
with suppress(FileNotFoundError, PermissionError):
    os.remove("might_not_exist.txt")

# redirect_stdout: capture print output
output = io.StringIO()
with redirect_stdout(output):
    print("This goes to StringIO")
    print("Not to console")

print(output.getvalue())  # "This goes to StringIO\nNot to console\n"

# ExitStack: manage multiple context managers dynamically
with ExitStack() as stack:
    files = [stack.enter_context(open(f"file{i}.txt", "w")) for i in range(5)]
    # All 5 files opened
    for i, f in enumerate(files):
        f.write(f"File {i}")
# All 5 files closed automatically
```

---

## 14. Common Pitfalls and Best Practices

### Pitfall 1: Forgetting `@wraps`

```python
# BAD - loses function metadata
def my_decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

# GOOD - preserves metadata
from functools import wraps

def my_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

### Pitfall 2: Mutable default arguments in decorators

```python
# BAD - cache dict is shared across all decorated functions!
def memoize_bad(func, cache={}):  # ❌ Mutable default
    @wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper

# GOOD - cache is created per decorated function
def memoize_good(func):
    cache = {}  # ✅ Created inside decorator
    @wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper
```

### Pitfall 3: Not handling return values

```python
# BAD - loses return value
def bad_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        func(*args, **kwargs)  # ❌ Return value is lost!
    return wrapper

# GOOD - preserves return value
def good_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)  # ✅ Return the result
    return wrapper
```

### Best Practice: Make decorators optional

```python
# Support both @decorator and @decorator()
def my_decorator(func=None, *, option=True):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            if option:
                print("Option is on")
            return f(*args, **kwargs)
        return wrapper
    
    if func is None:
        # Called with arguments: @my_decorator(option=False)
        return decorator
    else:
        # Called without arguments: @my_decorator
        return decorator(func)

@my_decorator  # Works
def func1():
    pass

@my_decorator()  # Works
def func2():
    pass

@my_decorator(option=False)  # Works
def func3():
    pass
```

---

## 📝 Practice Tasks

### Task 1: Build a Retry Decorator
Create a `@retry` decorator that:
- Retries a function up to N times if it raises an exception
- Has optional delay between retries
- Logs each attempt
- Re-raises the exception if all attempts fail

Test it with a function that randomly fails.

### Task 2: Build a Rate Limiter Decorator
Create a `@rate_limit(calls=5, period=60)` decorator that:
- Limits function calls to N calls per time period (in seconds)
- Raises an exception if limit is exceeded
- Tracks calls per function (not globally)

### Task 3: Build a Cache Decorator with TTL
Create a `@cache(ttl=60)` decorator that:
- Caches function results
- Expires cached values after TTL seconds
- Works with any function signature
- Has a `.clear_cache()` method

### Task 4: Build a Database Transaction Context Manager
Create a context manager `transaction(connection)` that:
- Starts a transaction on enter
- Commits on successful exit
- Rolls back on exception
- Works with a mock database connection class you create

### Task 5: Build a File Batch Processor
Create a context manager `batch_writer(filename, batch_size=100)` that:
- Buffers writes to a file
- Flushes when batch_size is reached
- Flushes remaining items on exit
- Handles errors gracefully

```python
with batch_writer("output.txt", batch_size=3) as writer:
    for i in range(10):
        writer.write(f"Line {i}\n")
# Should write in batches of 3, final batch of 1
```

### Task 6: Combine Both
Create a function that:
- Uses a timing decorator
- Uses a logging decorator
- Uses a database transaction context manager
- Uses a temporary file context manager

Stack them properly and verify all cleanup happens.

---

## 🔗 Resources

- [PEP 318 - Decorators for Functions and Methods](https://www.python.org/dev/peps/pep-0318/)
- [PEP 343 - The "with" Statement](https://www.python.org/dev/peps/pep-0343/)
- [Python Decorators Official Docs](https://docs.python.org/3/glossary.html#term-decorator)
- [contextlib Official Docs](https://docs.python.org/3/library/contextlib.html)
- [Real Python - Primer on Python Decorators](https://realpython.com/primer-on-python-decorators/)
- [Real Python - Context Managers](https://realpython.com/python-with-statement/)
- [functools.wraps Documentation](https://docs.python.org/3/library/functools.html#functools.wraps)

---

> **Tomorrow (Day 8):** File Handling & Serialization - working with CSV, JSON, YAML, binary files, and the `pathlib` module.

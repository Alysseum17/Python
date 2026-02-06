# Day 2: Functions, Lambdas, Scope & Closures

## 🎯 Goal

Master Python functions in depth. Most basics will transfer from JS/Java, but Python has unique features like `*args/**kwargs`, keyword-only arguments, and a different scoping model that deserve close attention.

---

## 1. Function Basics

```python
# Basic function
def greet(name):
    return f"Hello, {name}!"

# No return → returns None (like JS returning undefined)
def say_hi():
    print("Hi!")

result = say_hi()  # prints "Hi!"
print(result)      # None
```

### Type hints (optional but recommended — you'll love this from TS)

```python
def greet(name: str) -> str:
    return f"Hello, {name}!"

# Python does NOT enforce these at runtime! They're just hints.
# (Unlike TypeScript which enforces at compile time)
greet(42)  # Works at runtime, but mypy/pyright will flag it

# Complex type hints
from typing import Optional, Union

def find_user(user_id: int) -> Optional[str]:  # str | None
    """Find user by ID. Returns None if not found."""
    if user_id == 1:
        return "Alice"
    return None

# Python 3.10+ — you can use | instead of Union
def process(value: int | str) -> str:
    return str(value)
```

### Docstrings (Python's JSDoc / Javadoc)

```python
def calculate_bmi(weight_kg: float, height_m: float) -> float:
    """
    Calculate Body Mass Index.
    
    Args:
        weight_kg: Weight in kilograms.
        height_m: Height in meters.
    
    Returns:
        BMI as a float value.
    
    Raises:
        ValueError: If height is zero or negative.
    """
    if height_m <= 0:
        raise ValueError("Height must be positive")
    return weight_kg / (height_m ** 2)

# Access docstring
print(calculate_bmi.__doc__)
help(calculate_bmi)
```

---

## 2. Arguments — This is Where Python Gets Interesting

### Positional and keyword arguments

```python
def create_user(name, age, city):
    return f"{name}, {age}, from {city}"

# All of these work:
create_user("Alice", 30, "Kyiv")              # positional
create_user(name="Alice", age=30, city="Kyiv") # keyword
create_user("Alice", city="Kyiv", age=30)      # mixed (positional first!)

# This FAILS:
# create_user(name="Alice", 30, "Kyiv")  # SyntaxError: positional after keyword
```

### Default values

```python
def create_user(name: str, age: int = 0, role: str = "user") -> dict:
    return {"name": name, "age": age, "role": role}

create_user("Alice")                    # {"name": "Alice", "age": 0, "role": "user"}
create_user("Bob", 25)                  # {"name": "Bob", "age": 25, "role": "user"}
create_user("Charlie", role="admin")    # {"name": "Charlie", "age": 0, "role": "admin"}
```

### ⚠️ CRITICAL: Mutable Default Argument Trap

This is one of the **most famous Python gotchas**. Read this carefully.

```python
# ❌ DANGEROUS — default list is shared across ALL calls!
def add_item(item, items=[]):
    items.append(item)
    return items

print(add_item("a"))  # ["a"]        — looks fine
print(add_item("b"))  # ["a", "b"]   — WAIT WHAT?!
print(add_item("c"))  # ["a", "b", "c"]  — it keeps accumulating!
```

**Why does this happen?**

In Python, default argument values are evaluated **once** when the function is **defined**, not each time it's called. So `items=[]` creates ONE list object at definition time, and every call that uses the default shares that same list.

This is completely different from JS where `function f(items = [])` creates a new array on each call.

```python
# ✅ CORRECT — use None as sentinel
def add_item(item, items=None):
    if items is None:
        items = []        # new list created on each call
    items.append(item)
    return items

print(add_item("a"))  # ["a"]
print(add_item("b"))  # ["b"]  — fresh list each time!

# This pattern applies to ALL mutable defaults: list, dict, set
# Rule: NEVER use mutable objects as default arguments
```

---

## 3. `*args` and `**kwargs` — Python's Rest/Spread

This is similar to JS rest parameters (`...args`) but split into two: one for positional, one for keyword.

### `*args` — collects extra positional arguments into a tuple

```python
def sum_all(*args):
    """args is a tuple of all positional arguments."""
    print(type(args))  # <class 'tuple'>
    print(args)        # (1, 2, 3, 4, 5)
    return sum(args)

sum_all(1, 2, 3, 4, 5)  # 15

# JS equivalent:
# function sumAll(...args) { return args.reduce((a, b) => a + b, 0); }
```

### `**kwargs` — collects extra keyword arguments into a dict

```python
def create_tag(tag_name, **kwargs):
    """kwargs is a dict of all keyword arguments."""
    print(type(kwargs))  # <class 'dict'>
    print(kwargs)        # {"class_name": "header", "id": "main"}
    
    attrs = " ".join(f'{k}="{v}"' for k, v in kwargs.items())
    return f"<{tag_name} {attrs}>"

create_tag("div", class_name="header", id="main")
# <div class_name="header" id="main">

# JS has no direct equivalent — closest is destructuring:
# function createTag(tagName, {...rest}) { }
```

### Combining everything — the full signature order

```python
def full_example(a, b, *args, key1="default", **kwargs):
    """
    a, b     — required positional
    *args    — extra positional (tuple)
    key1     — keyword with default (MUST come after *args)
    **kwargs — extra keyword (dict)
    """
    print(f"a={a}, b={b}")
    print(f"args={args}")
    print(f"key1={key1}")
    print(f"kwargs={kwargs}")

full_example(1, 2, 3, 4, 5, key1="custom", extra="hello")
# a=1, b=2
# args=(3, 4, 5)
# key1=custom
# kwargs={"extra": "hello"}
```

### Unpacking (Python's spread operator)

```python
# Spread a list into positional args
numbers = [1, 2, 3]
print(*numbers)  # same as print(1, 2, 3)

def add(a, b, c):
    return a + b + c

add(*numbers)  # 6  — like JS: add(...numbers)

# Spread a dict into keyword args
config = {"host": "localhost", "port": 8080}

def connect(host, port):
    print(f"Connecting to {host}:{port}")

connect(**config)  # like JS: connect({...config}) but maps to named params

# Combine both
def func(a, b, c, host, port):
    pass

func(*numbers, **config)  # a=1, b=2, c=3, host="localhost", port=8080
```

---

## 4. Keyword-Only and Positional-Only Arguments

This is **Python-unique** — neither JS nor Java has this.

### Keyword-only arguments (after `*`)

```python
# Everything after * MUST be passed as keyword
def fetch_data(url, *, timeout=30, retries=3):
    print(f"GET {url} (timeout={timeout}, retries={retries})")

fetch_data("https://api.com")                     # ✅ 
fetch_data("https://api.com", timeout=60)          # ✅
fetch_data("https://api.com", 60)                  # ❌ TypeError!
# This forces callers to be explicit — great for readability
```

**Why is this useful?** Imagine reading `fetch_data("https://api.com", 60, 3)` — you can't tell what 60 and 3 mean. With keyword-only args, it's always clear: `fetch_data("https://api.com", timeout=60, retries=3)`.

### Positional-only arguments (before `/`) — Python 3.8+

```python
# Everything before / MUST be passed positionally
def pow(base, exp, /):
    return base ** exp

pow(2, 10)           # ✅  → 1024
pow(base=2, exp=10)  # ❌ TypeError!
```

### Full control

```python
def example(pos_only, /, normal, *, kw_only):
    pass

example(1, 2, kw_only=3)         # ✅
example(1, normal=2, kw_only=3)  # ✅
example(pos_only=1, normal=2, kw_only=3)  # ❌ pos_only is positional-only
```

---

## 5. First-Class Functions & Higher-Order Functions

Functions are objects in Python, just like in JS.

```python
# Assign to variable
def shout(text):
    return text.upper()

yell = shout  # no parentheses — assigning the function itself
print(yell("hello"))  # HELLO

# Pass as argument (like JS callbacks)
def apply(func, value):
    return func(value)

apply(shout, "hello")  # HELLO
apply(len, "hello")    # 5

# Return from function
def multiplier(factor):
    def multiply(x):
        return x * factor
    return multiply  # returning the inner function

double = multiplier(2)
triple = multiplier(3)
print(double(5))   # 10
print(triple(5))   # 15
# This is also a closure — more on that below
```

### Built-in higher-order functions: `map`, `filter`, `sorted`

```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# map — like JS .map()
squared = list(map(lambda x: x ** 2, numbers))
# [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

# filter — like JS .filter()
evens = list(filter(lambda x: x % 2 == 0, numbers))
# [2, 4, 6, 8, 10]

# ⚡ BUT! Pythonic way is list comprehensions (Day 3):
squared = [x ** 2 for x in numbers]
evens = [x for x in numbers if x % 2 == 0]
# Much cleaner — Python devs prefer comprehensions over map/filter

# sorted — with key function
words = ["banana", "apple", "cherry", "date"]
sorted(words)                        # alphabetical
sorted(words, key=len)               # by length: ["date", "apple", "banana", "cherry"]
sorted(words, key=len, reverse=True) # longest first

# Sorting complex objects
users = [
    {"name": "Charlie", "age": 30},
    {"name": "Alice", "age": 25},
    {"name": "Bob", "age": 28},
]
sorted(users, key=lambda u: u["age"])  # sort by age
```

---

## 6. Lambda Functions

Lambdas are anonymous one-expression functions. Like JS arrow functions but **much more limited**.

```python
# Lambda syntax
add = lambda x, y: x + y
add(3, 5)  # 8

# JS equivalent: const add = (x, y) => x + y

# Key differences from JS arrow functions:
# - Only ONE expression (no multi-line, no statements)
# - No curly braces, no explicit return
# - Cannot contain assignments or complex logic

# ✅ Good lambda use — short, clear, one-liner
sorted(users, key=lambda u: u["age"])
button.on_click(lambda event: print(event))

# ❌ Bad lambda use — too complex, use def instead
# process = lambda x: x.strip().lower().replace(" ", "_") if x else ""
# Better as:
def process(x):
    if not x:
        return ""
    return x.strip().lower().replace(" ", "_")
```

### Lambda with `map`, `filter`, `reduce`

```python
from functools import reduce

numbers = [1, 2, 3, 4, 5]

# map
list(map(lambda x: x * 2, numbers))    # [2, 4, 6, 8, 10]

# filter
list(filter(lambda x: x > 3, numbers)) # [4, 5]

# reduce — like JS .reduce()
reduce(lambda acc, x: acc + x, numbers)        # 15
reduce(lambda acc, x: acc + x, numbers, 100)   # 115 (with initial value)

# Note: reduce was moved to functools because Guido van Rossum
# considers it less readable than a simple loop
```

---

## 7. Scope — The LEGB Rule

This is where Python differs significantly from JS. Python uses the **LEGB** rule:

```
L — Local       (inside the current function)
E — Enclosing   (inside any enclosing function — for nested functions)
G — Global      (module level)
B — Built-in    (Python's built-in names: print, len, etc.)
```

### Basic scope

```python
x = "global"  # Global scope

def outer():
    x = "enclosing"  # Enclosing scope
    
    def inner():
        x = "local"  # Local scope
        print(x)     # "local" — L wins
    
    inner()
    print(x)  # "enclosing" — inner's x didn't affect this

outer()
print(x)  # "global" — nothing changed here either
```

### ⚠️ The `global` and `nonlocal` keywords — IMPORTANT DIFFERENCE FROM JS

In JS, you can freely read and modify outer variables. In Python, you can **read** outer variables but **cannot reassign** them without special keywords.

```python
counter = 0

def increment():
    # counter += 1  # ❌ UnboundLocalError!
    # Python sees the assignment and assumes counter is local,
    # but it hasn't been defined locally yet.
    pass

# Why does this happen?
# Python decides at COMPILE TIME whether a variable is local or not.
# If there's any assignment to 'counter' in the function, Python treats
# it as local for the ENTIRE function — even before the assignment line.
```

**Fix with `global`:**

```python
counter = 0

def increment():
    global counter  # "I want to modify the global one"
    counter += 1

increment()
increment()
print(counter)  # 2
```

**Fix with `nonlocal` (for enclosing scope):**

```python
def make_counter():
    count = 0
    
    def increment():
        nonlocal count  # "I want to modify the enclosing one"
        count += 1
        return count
    
    return increment

counter = make_counter()
print(counter())  # 1
print(counter())  # 2
print(counter())  # 3
```

### Reading vs. reassigning — the nuance

```python
data = [1, 2, 3]

def modify():
    # Reading and MUTATING is fine — no global needed!
    data.append(4)  # ✅ Works! We're mutating, not reassigning.
    
    # data = [5, 6, 7]  # ❌ This would be reassignment — needs global

modify()
print(data)  # [1, 2, 3, 4]
```

**The rule:** You only need `global`/`nonlocal` when you're **reassigning** (changing what the variable points to). Mutating the object it already points to (like `.append()`) is fine.

### Comparison with JS

```javascript
// JS — this just works, no special keyword needed
let counter = 0;
function increment() {
    counter += 1; // Fine! JS freely accesses outer scope
}
```

```python
# Python — need explicit permission to reassign outer variables
counter = 0
def increment():
    global counter  # Must declare intent
    counter += 1
```

---

## 8. Closures — Deep Dive

A closure is a function that remembers values from its enclosing scope even after the outer function has returned.

### Basic closure

```python
def make_greeting(greeting):
    # 'greeting' is captured by the inner function
    def greet(name):
        return f"{greeting}, {name}!"
    return greet

hello = make_greeting("Hello")
hey = make_greeting("Hey")

print(hello("Alice"))  # Hello, Alice!
print(hey("Bob"))      # Hey, Bob!

# 'greeting' is gone from the stack, but the closure remembers it
# You can inspect it:
print(hello.__closure__[0].cell_contents)  # "Hello"
```

### Practical closure examples

```python
# 1. Function factory (like the multiplier example above)
def make_power(exp):
    def power(base):
        return base ** exp
    return power

square = make_power(2)
cube = make_power(3)
print(square(5))  # 25
print(cube(5))    # 125

# 2. Data hiding / encapsulation (like JS module pattern)
def create_bank_account(initial_balance):
    balance = initial_balance
    
    def deposit(amount):
        nonlocal balance
        balance += amount
        return balance
    
    def withdraw(amount):
        nonlocal balance
        if amount > balance:
            raise ValueError("Insufficient funds")
        balance -= amount
        return balance
    
    def get_balance():
        return balance
    
    return {
        "deposit": deposit,
        "withdraw": withdraw,
        "get_balance": get_balance
    }

account = create_bank_account(100)
account["deposit"](50)          # 150
account["withdraw"](30)         # 120
print(account["get_balance"]()) # 120
# balance is completely hidden — no way to access it directly

# 3. Configurable logger
def make_logger(prefix):
    def log(message):
        print(f"[{prefix}] {message}")
    return log

error = make_logger("ERROR")
info = make_logger("INFO")
error("Something broke")  # [ERROR] Something broke
info("Server started")    # [INFO] Server started
```

### ⚠️ Classic Closure Trap — Late Binding

This catches everyone — JS developers included (it's the same bug as the classic `var` in a `for` loop).

```python
# ❌ BUG — all functions return 4!
functions = []
for i in range(5):
    functions.append(lambda: i)

print([f() for f in functions])  # [4, 4, 4, 4, 4]  — not [0, 1, 2, 3, 4]!
```

**Why?** The lambda captures the **variable** `i`, not its **value**. By the time you call the lambdas, the loop has finished and `i` is 4.

```python
# ✅ FIX 1 — default argument captures current value
functions = []
for i in range(5):
    functions.append(lambda i=i: i)  # i=i captures current value

print([f() for f in functions])  # [0, 1, 2, 3, 4] ✅

# ✅ FIX 2 — use a factory function
def make_func(i):
    return lambda: i

functions = [make_func(i) for i in range(5)]
print([f() for f in functions])  # [0, 1, 2, 3, 4] ✅
```

**JS comparison:** Same issue with `var`. Fixed with `let` (block-scoped) or IIFE:
```javascript
// JS bug (var)
for (var i = 0; i < 5; i++) { ... }  // same problem
// JS fix (let)
for (let i = 0; i < 5; i++) { ... }  // let creates new scope each iteration
```

Python has no `let` equivalent — you must use the default argument trick or a factory.

---

## 9. Advanced Function Features

### `functools.partial` — pre-fill some arguments

```python
from functools import partial

def power(base, exp):
    return base ** exp

square = partial(power, exp=2)  # pre-fill exp=2
cube = partial(power, exp=3)

print(square(5))  # 25
print(cube(5))    # 125

# Useful for callbacks where you need a specific signature
import json

# Create a custom json dumper with specific settings
pretty_json = partial(json.dumps, indent=2, ensure_ascii=False)
print(pretty_json({"name": "Олексій", "age": 25}))
```

### `functools.lru_cache` — memoization

```python
from functools import lru_cache

@lru_cache(maxsize=128)  # cache up to 128 results
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# Without cache: fibonacci(100) would take forever
# With cache: instant!
print(fibonacci(100))  # 354224848179261915075

# View cache stats
print(fibonacci.cache_info())
# CacheInfo(hits=98, misses=101, maxsize=128, currsize=101)

# Clear cache
fibonacci.cache_clear()
```

### Functions can have attributes

```python
def my_func():
    """A function with custom attributes."""
    my_func.call_count += 1
    return "called"

my_func.call_count = 0

my_func()
my_func()
my_func()
print(my_func.call_count)  # 3

# Every function has these built-in attributes:
print(my_func.__name__)    # "my_func"
print(my_func.__doc__)     # "A function with custom attributes."
```

---

## 10. Putting It All Together — Real-World Patterns

### Retry function using closures

```python
import time
import random

def with_retry(max_retries=3, delay=1.0):
    """Creates a retry wrapper with configurable retries and delay."""
    def decorator(func):
        def wrapper(*args, **kwargs):
            last_error = None
            for attempt in range(1, max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_error = e
                    print(f"Attempt {attempt}/{max_retries} failed: {e}")
                    if attempt < max_retries:
                        time.sleep(delay)
            raise last_error
        return wrapper
    return decorator

# Usage (this is actually a decorator — Day 7 topic, but good to see now)
@with_retry(max_retries=3, delay=0.5)
def unstable_api_call():
    if random.random() < 0.7:
        raise ConnectionError("Server unavailable")
    return {"status": "ok"}

# This pattern combines: closures + *args/**kwargs + higher-order functions
```

### Pipeline using higher-order functions

```python
def pipeline(*functions):
    """Chain functions together — each output feeds into the next input."""
    def execute(data):
        result = data
        for func in functions:
            result = func(result)
        return result
    return execute

# Create a text processing pipeline
process_text = pipeline(
    str.strip,
    str.lower,
    lambda s: s.replace(" ", "_"),
    lambda s: s.replace(".", ""),
)

print(process_text("  Hello World. "))  # "hello_world"

# Similar to JS: pipe = (...fns) => (x) => fns.reduce((v, f) => f(v), x)
```

---

## 📝 Practice Tasks

### Task 1: Flexible Calculator
Write a `calc` function that accepts two required numbers and an operation keyword argument with default `"add"`. Support add, subtract, multiply, divide. Use `match-case`.

### Task 2: Argument Explorer
Write a function `describe_call(*args, **kwargs)` that prints a formatted description of all arguments it received, their types, and count.

### Task 3: Make Counter with Reset
Using closures, create a `make_counter(start=0)` that returns a dict with `increment`, `decrement`, `reset`, and `get_value` functions. The counter should support an optional `step` parameter.

### Task 4: Memoize Decorator
Write your own `memoize` function (not using `lru_cache`) that caches results based on arguments. Test with an expensive recursive function like fibonacci.

```python
def memoize(func):
    cache = {}
    def wrapper(*args):
        # Your implementation here
        pass
    return wrapper
```

### Task 5: Function Composition
Write a `compose` function that takes any number of functions and returns a new function that applies them right-to-left (like mathematical composition: `f(g(h(x)))`). Test with at least 3 functions.

### Task 6: Late Binding Bug Fix
Given this buggy code, explain why it's broken and fix it in two different ways:
```python
handlers = {}
for event in ["click", "hover", "scroll"]:
    handlers[event] = lambda: print(f"Handling {event}")

handlers["click"]()   # prints "Handling scroll" — wrong!
handlers["hover"]()   # prints "Handling scroll" — wrong!
```

---

## 📚 Resources

- [Python Official — Defining Functions](https://docs.python.org/3/tutorial/controlflow.html#defining-functions)
- [Python Official — More on Functions](https://docs.python.org/3/tutorial/controlflow.html#more-on-defining-functions)
- [Real Python — Python Scope & the LEGB Rule](https://realpython.com/python-scope-legb-rule/)
- [Real Python — Python Closures](https://realpython.com/python-closure/)
- [Real Python — Python Lambda Functions](https://realpython.com/python-lambda/)
- [Real Python — Primer on Python Decorators](https://realpython.com/primer-on-python-decorators/) (preview for Day 7)
- [functools documentation](https://docs.python.org/3/library/functools.html)

---

## 🔑 Key Takeaways

1. **Mutable default args are evil** — always use `None` as default for lists/dicts/sets.
2. **LEGB rule** — Python decides scope at compile time; use `global`/`nonlocal` to reassign outer variables.
3. **`*args` and `**kwargs`** — Python's version of rest params, split by positional vs keyword.
4. **Keyword-only args** (after `*`) make APIs self-documenting.
5. **Late binding in closures** — lambdas capture variables, not values. Use `i=i` trick to fix.
6. **Prefer comprehensions** over `map`/`filter` in idiomatic Python.

---

> **Tomorrow (Day 3):** Data Structures Deep Dive — lists, tuples, sets, dicts, comprehensions, named tuples. This is where Python really shines.

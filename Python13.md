# Day 13: Performance & Profiling

## 🎯 Goal

Learn to find and fix performance bottlenecks in Python. Python is ~50-100x slower than Java/C for raw computation, but in practice this rarely matters because bottlenecks are usually I/O, algorithms, or data structures — not the language. Today you'll learn to measure first, optimize second, and use Python's built-in tools to make informed decisions.

**The golden rule:** Don't guess where the bottleneck is. Profile it. Developers are notoriously bad at predicting what's slow.

---

## 1. `timeit` — Micro-Benchmarks

### Command line usage

```bash
# Time a simple expression
python -m timeit "sum(range(1000))"
# 100000 loops, best of 5: 12.3 usec per loop

# Compare two approaches
python -m timeit "'-'.join(str(i) for i in range(100))"
python -m timeit "'-'.join([str(i) for i in range(100)])"
# List comprehension is usually faster than generator for join()
```

### In-code usage

```python
import timeit

# Time a statement (runs it many times, returns total seconds)
time = timeit.timeit("sum(range(1000))", number=100_000)
print(f"{time:.3f}s for 100K iterations")

# Time with setup code
time = timeit.timeit(
    stmt="sorted(data)",
    setup="import random; data = [random.random() for _ in range(1000)]",
    number=1000,
)
print(f"Sorting 1000 items: {time/1000*1000:.2f}ms per call")

# Using as a decorator/function timer (most practical)
def benchmark(func, *args, runs=1000, **kwargs):
    """Simple benchmark helper."""
    times = timeit.repeat(
        lambda: func(*args, **kwargs),
        number=runs,
        repeat=5,
    )
    best = min(times) / runs
    print(f"{func.__name__}: {best*1000:.4f}ms per call (best of 5)")
    return best
```

### Comparing approaches

```python
import timeit

# Example: finding an element — list vs set
setup = """
import random
data_list = list(range(100_000))
data_set = set(range(100_000))
target = 99_999  # worst case for list
"""

list_time = timeit.timeit("target in data_list", setup=setup, number=1000)
set_time = timeit.timeit("target in data_set", setup=setup, number=1000)

print(f"List lookup: {list_time:.4f}s")   # ~1.5s
print(f"Set lookup:  {set_time:.4f}s")    # ~0.0001s
print(f"Set is {list_time/set_time:.0f}x faster")  # ~15,000x faster!
```

---

## 2. `cProfile` — Function-Level Profiling

### Basic profiling

```python
import cProfile

def slow_function():
    total = 0
    for i in range(1_000_000):
        total += i ** 2
    return total

def process_data():
    data = list(range(100_000))
    sorted_data = sorted(data, reverse=True)
    result = slow_function()
    filtered = [x for x in sorted_data if x % 2 == 0]
    return len(filtered)

# Profile the function
cProfile.run("process_data()")

# Output looks like:
#          8 function calls in 0.312 seconds
#
#    ncalls  tottime  percall  cumtime  percall filename:lineno(function)
#         1    0.000    0.000    0.312    0.312 <string>:1(<module>)
#         1    0.285    0.285    0.285    0.285 script.py:3(slow_function)
#         1    0.015    0.015    0.015    0.015 {built-in method builtins.sorted}
#         1    0.012    0.012    0.012    0.012 script.py:10(<listcomp>)
#
# NOW you know: slow_function takes 91% of time. Optimize THERE.
```

### Column meanings

```
ncalls   — number of calls to this function
tottime  — total time IN this function (excluding sub-calls)
percall  — tottime / ncalls
cumtime  — total time in this function INCLUDING sub-calls
percall  — cumtime / ncalls
filename:lineno(function) — where it is

Key insight:
  tottime = where CPU time is actually spent
  cumtime = the "total cost" of calling this function
  
  If cumtime >> tottime, the function itself is fast
  but it calls slow sub-functions.
```

### Save and analyze profiles

```python
import cProfile
import pstats

# Save profile to file
cProfile.run("process_data()", "output.prof")

# Analyze saved profile
stats = pstats.Stats("output.prof")
stats.strip_dirs()                    # remove path prefixes
stats.sort_stats("cumulative")        # sort by cumulative time
stats.print_stats(20)                 # top 20 functions

# Sort options:
# "cumulative" — total time including sub-calls (find costly call trees)
# "tottime"    — time in function only (find actual bottlenecks)
# "calls"      — number of calls (find excessive calling)
```

### Profile from command line

```bash
# Profile a script
python -m cProfile -s cumulative my_script.py

# Save to file for visualization
python -m cProfile -o profile.prof my_script.py

# Visualize with snakeviz (browser-based)
pip install snakeviz
snakeviz profile.prof
# Opens interactive flame graph in browser — very useful!
```

### Profile a specific function with decorator

```python
import cProfile
import functools

def profile(func):
    """Decorator to profile a function."""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        profiler.enable()
        result = func(*args, **kwargs)
        profiler.disable()
        
        stats = pstats.Stats(profiler)
        stats.strip_dirs()
        stats.sort_stats("cumulative")
        stats.print_stats(15)
        return result
    return wrapper

@profile
def my_pipeline():
    # your code here
    pass
```

---

## 3. `line_profiler` — Line-by-Line Profiling

When `cProfile` tells you WHICH function is slow, `line_profiler` tells you WHICH LINE.

```bash
pip install line_profiler
```

```python
# Add @profile decorator (provided by line_profiler, not a custom one)
@profile
def slow_function(data):
    result = []                          # Line 1
    for item in data:                    # Line 2
        if item % 2 == 0:               # Line 3
            result.append(item ** 2)     # Line 4
    total = sum(result)                  # Line 5
    sorted_result = sorted(result)       # Line 6
    return sorted_result                 # Line 7
```

```bash
# Run with kernprof
kernprof -l -v my_script.py

# Output:
# Line #   Hits    Time   Per Hit   % Time  Line Contents
# ======   ====    ====   =======   ======  ============
#      2          1      1.0      1.0    0.0  result = []
#      3     100000  25000.0      0.3   20.0  for item in data:
#      4     100000  30000.0      0.3   24.0  if item % 2 == 0:
#      5      50000  55000.0      1.1   44.0  result.append(item ** 2)
#      6          1  10000.0  10000.0    8.0  total = sum(result)
#      7          1   5000.0   5000.0    4.0  sorted_result = sorted(result)

# Now you know: line 5 (append + power) takes 44% of time
```

---

## 4. Memory Profiling

### `sys.getsizeof` — object size

```python
import sys

# Size of basic objects
sys.getsizeof(0)            # 28 bytes (int)
sys.getsizeof(3.14)         # 24 bytes (float)
sys.getsizeof("")           # 49 bytes (empty string)
sys.getsizeof("hello")      # 54 bytes
sys.getsizeof([])            # 56 bytes (empty list)
sys.getsizeof([1,2,3])       # 88 bytes (3-item list)
sys.getsizeof({})            # 64 bytes (empty dict)
sys.getsizeof(set())         # 216 bytes (empty set!)
sys.getsizeof((1,2,3))       # 64 bytes (tuple — lighter than list)

# ⚠️ getsizeof doesn't count nested objects!
data = [[1,2,3], [4,5,6]]
sys.getsizeof(data)          # 72 bytes — only the outer list!
# Doesn't include the inner lists!

# Deep size calculation
def deep_getsizeof(obj, seen=None):
    """Recursively calculate total memory of an object."""
    if seen is None:
        seen = set()
    obj_id = id(obj)
    if obj_id in seen:
        return 0
    seen.add(obj_id)
    
    size = sys.getsizeof(obj)
    if isinstance(obj, dict):
        size += sum(deep_getsizeof(k, seen) + deep_getsizeof(v, seen) 
                    for k, v in obj.items())
    elif isinstance(obj, (list, tuple, set, frozenset)):
        size += sum(deep_getsizeof(item, seen) for item in obj)
    return size

data = {"users": [{"name": "Alice", "scores": [1,2,3]}] * 1000}
print(f"Deep size: {deep_getsizeof(data) / 1024:.1f} KB")
```

### `tracemalloc` — track memory allocations

```python
import tracemalloc

# Start tracking
tracemalloc.start()

# Your code
data = [list(range(1000)) for _ in range(1000)]

# Get current and peak memory
current, peak = tracemalloc.get_traced_memory()
print(f"Current: {current / 1024 / 1024:.1f} MB")
print(f"Peak:    {peak / 1024 / 1024:.1f} MB")

# Get top memory-consuming lines
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics("lineno")

print("\nTop 10 memory consumers:")
for stat in top_stats[:10]:
    print(f"  {stat}")

tracemalloc.stop()
```

### `memory_profiler` — line-by-line memory

```bash
pip install memory_profiler
```

```python
from memory_profiler import profile

@profile
def memory_heavy():
    a = [i for i in range(1_000_000)]      # ~8 MB
    b = {i: i**2 for i in range(100_000)}   # ~5 MB
    del a                                    # freed
    c = list(range(500_000))                # ~4 MB
    return c
```

```bash
python -m memory_profiler my_script.py

# Line #    Mem usage    Increment   Line Contents
# =============================================
#     3     45.2 MiB     0.0 MiB   @profile
#     4     53.4 MiB     8.2 MiB   a = [i for i in range(1_000_000)]
#     5     58.6 MiB     5.2 MiB   b = {i: i**2 for i in range(100_000)}
#     6     50.4 MiB    -8.2 MiB   del a
#     7     54.2 MiB     3.8 MiB   c = list(range(500_000))
```

---

## 5. `collections` Module — Optimized Data Structures

### `Counter` — count things fast

```python
from collections import Counter

# Already covered in Day 3, but here's the performance angle:
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]

# ❌ Slow manual counting
counts = {}
for word in words:
    counts[word] = counts.get(word, 0) + 1

# ✅ Counter — optimized in C, faster for large datasets
counts = Counter(words)

# Counter operations are O(n) and implemented in C
# For 1M items: Counter is ~2x faster than manual dict
```

### `defaultdict` — skip existence checks

```python
from collections import defaultdict

# ❌ Manual check + initialize
groups = {}
for item in data:
    key = item["category"]
    if key not in groups:
        groups[key] = []
    groups[key].append(item)

# ✅ defaultdict — cleaner AND faster (no double lookup)
groups = defaultdict(list)
for item in data:
    groups[item["category"]].append(item)

# defaultdict(int)   — default 0
# defaultdict(list)  — default []
# defaultdict(set)   — default set()
# defaultdict(lambda: "N/A") — custom default
```

### `deque` — fast queue/stack

```python
from collections import deque

# List vs deque performance
# Operation          list        deque
# ─────────────────  ────────    ─────
# append right       O(1)*       O(1)
# append left        O(n) 🐌     O(1) ⚡
# pop right          O(1)        O(1)
# pop left           O(n) 🐌     O(1) ⚡
# random access      O(1)        O(n)
# * amortized

# Use deque when you need fast append/pop from BOTH ends
q = deque(maxlen=1000)  # bounded — auto-evicts oldest when full

# Sliding window pattern (very common in DE)
def moving_average(data: list[float], window: int) -> list[float]:
    """Calculate moving average using deque as sliding window."""
    window_deque = deque(maxlen=window)
    result = []
    for value in data:
        window_deque.append(value)
        result.append(sum(window_deque) / len(window_deque))
    return result

# Rotate
d = deque([1, 2, 3, 4, 5])
d.rotate(2)   # [4, 5, 1, 2, 3] — move 2 from right to left
d.rotate(-1)  # [5, 1, 2, 3, 4] — move 1 from left to right
```

### `OrderedDict` — dict with order guarantees

```python
from collections import OrderedDict

# Since Python 3.7, regular dict preserves insertion order.
# But OrderedDict still has unique features:

# 1. move_to_end
od = OrderedDict(a=1, b=2, c=3)
od.move_to_end("a")        # a goes to end: b, c, a
od.move_to_end("c", last=False)  # c goes to front: c, b, a

# 2. LRU cache with size limit
class LRUCache(OrderedDict):
    """Least Recently Used cache."""
    def __init__(self, maxsize=128):
        super().__init__()
        self.maxsize = maxsize
    
    def __getitem__(self, key):
        value = super().__getitem__(key)
        self.move_to_end(key)  # mark as recently used
        return value
    
    def __setitem__(self, key, value):
        if key in self:
            self.move_to_end(key)
        super().__setitem__(key, value)
        if len(self) > self.maxsize:
            self.popitem(last=False)  # remove oldest

cache = LRUCache(maxsize=3)
cache["a"] = 1
cache["b"] = 2
cache["c"] = 3
cache["d"] = 4  # "a" is evicted (oldest)
print(list(cache.keys()))  # ["b", "c", "d"]
```

### `ChainMap` — layered lookups

```python
from collections import ChainMap

# Efficient layered configuration (like Day 8 Task 6 config manager)
defaults = {"color": "blue", "size": 10, "debug": False}
env_config = {"debug": True, "log_level": "INFO"}
cli_args = {"size": 20}

config = ChainMap(cli_args, env_config, defaults)
# Searches in order: cli_args → env_config → defaults

print(config["color"])      # "blue" (from defaults)
print(config["debug"])      # True (from env_config)
print(config["size"])       # 20 (from cli_args)

# More efficient than merging dicts: {**defaults, **env_config, **cli_args}
# ChainMap doesn't copy data — O(1) to create
```

---

## 6. Optimization Patterns

### Pattern 1: Use built-in functions (implemented in C)

```python
import timeit

data = list(range(100_000))

# ❌ Python loop
def sum_manual(data):
    total = 0
    for x in data:
        total += x
    return total

# ✅ Built-in sum() — implemented in C
def sum_builtin(data):
    return sum(data)

# sum() is ~10x faster than manual loop
# Same for: min(), max(), any(), all(), sorted(), len()

# ❌ Manual string building
result = ""
for word in words:
    result += word + " "  # O(n²) — creates new string each time!

# ✅ str.join() — O(n)
result = " ".join(words)
```

### Pattern 2: List comprehensions over loops

```python
# ❌ Slow — append in loop
result = []
for i in range(100_000):
    if i % 2 == 0:
        result.append(i ** 2)

# ✅ ~30-50% faster — comprehension
result = [i ** 2 for i in range(100_000) if i % 2 == 0]

# Why faster?
# 1. No .append() method lookup per iteration
# 2. List size is pre-allocated
# 3. Bytecode is optimized for comprehensions
```

### Pattern 3: Generators for large data

```python
import sys

# ❌ Creates entire list in memory
total = sum([x ** 2 for x in range(10_000_000)])  # ~80 MB of memory!

# ✅ Generator — almost zero memory
total = sum(x ** 2 for x in range(10_000_000))  # ~0 MB extra

# Check the difference
list_comp = [x for x in range(1_000_000)]
gen_exp = (x for x in range(1_000_000))
print(f"List: {sys.getsizeof(list_comp) / 1024 / 1024:.1f} MB")  # ~8 MB
print(f"Generator: {sys.getsizeof(gen_exp)} bytes")                # 200 bytes!
```

### Pattern 4: Choose the right data structure

```python
# Membership test: set >> list
big_list = list(range(1_000_000))
big_set = set(range(1_000_000))

# list: O(n) — scans entire list
# set:  O(1) — hash lookup

# Dictionary access: direct >> loop
users = {u["id"]: u for u in user_list}  # build index once
user = users[target_id]                   # O(1) lookup
# vs scanning: O(n) per lookup

# Tuple vs list for fixed data
# Tuples are slightly faster to create and iterate
# Tuples use less memory (no need for resize machinery)
point_list = [1, 2, 3]    # 88 bytes
point_tuple = (1, 2, 3)   # 64 bytes
```

### Pattern 5: Avoid repeated computation

```python
# ❌ len() called every iteration
for i in range(len(data)):
    for j in range(len(data)):
        process(data[i], data[j])

# ✅ Cache the length
n = len(data)
for i in range(n):
    for j in range(n):
        process(data[i], data[j])

# ❌ Method lookup every iteration
for item in data:
    result.append(item.strip().lower())

# ✅ Local variable for method (matters in tight loops)
append = result.append
for item in data:
    append(item.strip().lower())
```

### Pattern 6: `__slots__` for memory-efficient classes

```python
import sys

# Regular class — each instance has a __dict__
class PointRegular:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

# Slotted class — no __dict__, fixed attributes
class PointSlotted:
    __slots__ = ("x", "y", "z")
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

p1 = PointRegular(1, 2, 3)
p2 = PointSlotted(1, 2, 3)

print(f"Regular: {sys.getsizeof(p1) + sys.getsizeof(p1.__dict__)} bytes")  # ~200 bytes
print(f"Slotted: {sys.getsizeof(p2)} bytes")  # ~64 bytes

# ~3x less memory per instance!
# Matters when you have millions of objects (like data records)

# Trade-off: can't add arbitrary attributes
# p2.w = 4  # ❌ AttributeError
```

### Pattern 7: `lru_cache` for expensive repeated calculations

```python
from functools import lru_cache

@lru_cache(maxsize=1024)
def expensive_computation(x: int, y: int) -> float:
    """Called multiple times with same arguments."""
    import math
    return math.sqrt(x ** 2 + y ** 2)

# First call: computes
result = expensive_computation(3, 4)  # 5.0

# Subsequent calls with same args: returns cached result (instant)
result = expensive_computation(3, 4)  # cached!

# Check cache stats
print(expensive_computation.cache_info())
# CacheInfo(hits=1, misses=1, maxsize=1024, currsize=1)
```

---

## 7. String Optimization

```python
# Strings are immutable — concatenation creates new objects

# ❌ O(n²) — creates new string each iteration
result = ""
for i in range(100_000):
    result += str(i)  # copies entire string each time!

# ✅ O(n) — join a list
parts = []
for i in range(100_000):
    parts.append(str(i))
result = "".join(parts)

# ✅ Even cleaner with comprehension
result = "".join(str(i) for i in range(100_000))

# ✅ f-strings for formatting (fastest option for formatting)
name = "Alice"
age = 25
msg = f"{name} is {age}"  # faster than .format() or %

# Benchmark:
# f-string:    ~0.15 μs
# .format():   ~0.40 μs
# % formatting: ~0.30 μs
# + concat:     ~0.20 μs (but only for 2-3 items)

# String interning — Python caches small strings
a = "hello"
b = "hello"
print(a is b)  # True — same object in memory!
# Works for strings that look like identifiers
# Doesn't work for longer or complex strings
```

---

## 8. Algorithm Complexity — Quick Reference

```
Common operations and their Big-O:

list:
  index [i]           O(1)
  append              O(1) amortized
  insert(0, x)        O(n) — shifts everything
  pop()               O(1)
  pop(0)              O(n)
  x in list           O(n) — linear scan
  sort                O(n log n)
  slice [a:b]         O(b-a)

dict:
  d[key]              O(1) average
  d[key] = value      O(1) average
  key in d            O(1) average
  del d[key]          O(1) average
  iteration           O(n)

set:
  x in s              O(1) average
  add(x)              O(1) average
  remove(x)           O(1) average
  union (|)           O(len(s) + len(t))
  intersection (&)    O(min(len(s), len(t)))

deque:
  append              O(1)
  appendleft          O(1)
  pop                 O(1)
  popleft             O(1)
  index [i]           O(n)

Common algorithmic improvements:
  Linear search → dict/set lookup     O(n) → O(1)
  Nested loops → set intersection     O(n²) → O(n)
  Repeated sort → heapq               O(n log n) → O(n log k)
  String concat → join                O(n²) → O(n)
```

---

## 9. `operator` and `itertools` — Functional Performance

### `operator` — avoid lambda overhead

```python
from operator import itemgetter, attrgetter, add
from functools import reduce

users = [
    {"name": "Charlie", "age": 30},
    {"name": "Alice", "age": 25},
    {"name": "Bob", "age": 28},
]

# ❌ Lambda has function call overhead
sorted(users, key=lambda u: u["age"])

# ✅ itemgetter is implemented in C — faster
sorted(users, key=itemgetter("age"))

# Multiple keys
sorted(users, key=itemgetter("age", "name"))

# For objects with attributes
# sorted(objects, key=attrgetter("age"))

# reduce with operator
numbers = [1, 2, 3, 4, 5]
total = reduce(add, numbers)  # faster than reduce(lambda a, b: a + b, numbers)
```

### `itertools` — efficient iteration

```python
from itertools import chain, islice, groupby, product, accumulate, repeat, starmap

# chain — concatenate iterables without copying
all_data = chain(list1, list2, list3)
# vs: list1 + list2 + list3  (creates new list, copies everything)

# islice — slice an iterator
first_100 = list(islice(huge_iterator, 100))
# vs: list(huge_iterator)[:100]  (materializes EVERYTHING first)

# accumulate — running totals
from itertools import accumulate
list(accumulate([1, 2, 3, 4, 5]))  # [1, 3, 6, 10, 15]

# product — cartesian product (replaces nested loops)
from itertools import product
for x, y in product(range(10), range(10)):
    pass
# vs: for x in range(10): for y in range(10):

# groupby — group consecutive elements (data must be sorted!)
from itertools import groupby
data = sorted(records, key=itemgetter("category"))
for category, items in groupby(data, key=itemgetter("category")):
    item_list = list(items)
    print(f"{category}: {len(item_list)} items")

# batched — split into chunks (Python 3.12+)
from itertools import batched
for batch in batched(range(100), 10):
    print(len(batch))  # 10, 10, 10, ..., 10

# Pre-3.12 batch function
def batched_compat(iterable, n):
    it = iter(iterable)
    while True:
        batch = list(islice(it, n))
        if not batch:
            break
        yield batch
```

---

## 10. Profiling a Real Data Pipeline

```python
"""
Example: profiling a data processing pipeline to find bottlenecks.
"""
import cProfile
import pstats
import csv
import json
import re
import time
from collections import Counter, defaultdict
from io import StringIO
from pathlib import Path


def generate_sample_data(n: int = 100_000) -> list[dict]:
    """Generate sample records for profiling."""
    import random
    categories = ["electronics", "clothing", "food", "books", "toys"]
    return [
        {
            "id": i,
            "name": f"Product {i}",
            "price": round(random.uniform(1, 1000), 2),
            "category": random.choice(categories),
            "description": f"This is product {i} " * 10,
            "tags": ",".join(random.sample(["sale", "new", "hot", "limited", "popular"], 3)),
        }
        for i in range(n)
    ]


def clean_text(text: str) -> str:
    """Clean product description."""
    text = text.lower().strip()
    text = re.sub(r"\s+", " ", text)
    return text


def process_pipeline(records: list[dict]) -> dict:
    """Full processing pipeline."""
    # Stage 1: Clean data
    for record in records:
        record["description"] = clean_text(record["description"])
        record["tags"] = record["tags"].split(",")
    
    # Stage 2: Filter (price > 10)
    filtered = [r for r in records if r["price"] > 10]
    
    # Stage 3: Group by category
    groups = defaultdict(list)
    for record in filtered:
        groups[record["category"]].append(record)
    
    # Stage 4: Compute stats per category
    stats = {}
    for category, items in groups.items():
        prices = [item["price"] for item in items]
        stats[category] = {
            "count": len(items),
            "avg_price": sum(prices) / len(prices),
            "max_price": max(prices),
            "min_price": min(prices),
        }
    
    # Stage 5: Tag frequency
    tag_counter = Counter()
    for record in filtered:
        tag_counter.update(record["tags"])
    
    return {
        "total_records": len(records),
        "filtered_records": len(filtered),
        "category_stats": stats,
        "top_tags": tag_counter.most_common(10),
    }


def run_profiled():
    """Run pipeline with profiling."""
    print("Generating data...")
    data = generate_sample_data(100_000)
    
    print("Profiling pipeline...")
    profiler = cProfile.Profile()
    profiler.enable()
    
    result = process_pipeline(data)
    
    profiler.disable()
    
    # Print top 20 time consumers
    stats = pstats.Stats(profiler)
    stats.strip_dirs()
    stats.sort_stats("tottime")
    print("\n" + "=" * 70)
    print("  Top functions by total time:")
    print("=" * 70)
    stats.print_stats(15)
    
    # Print result summary
    print(f"Processed {result['total_records']:,} → {result['filtered_records']:,} records")
    for cat, s in result["category_stats"].items():
        print(f"  {cat}: {s['count']} items, avg ${s['avg_price']:.2f}")


if __name__ == "__main__":
    run_profiled()
```

---

## 11. Quick Wins Checklist

```
Before optimizing, always:
  □ Profile first — don't guess!
  □ Optimize the algorithm before optimizing the code
  □ Measure before AND after changes

Common quick wins:
  □ list → set for membership testing
  □ Loop → list comprehension
  □ String concatenation → "".join()
  □ Manual counting → Counter
  □ Manual grouping → defaultdict(list)
  □ list.pop(0) → deque.popleft()
  □ Repeated dict lookups → local variable
  □ Global variables → local variables
  □ Lambda → operator.itemgetter
  □ Full list → generator expression
  □ Repeated computation → lru_cache
  □ Many small objects → __slots__
  □ Nested loops → set operations

Nuclear options (when Python isn't fast enough):
  □ NumPy for numerical computation
  □ Polars/Pandas for data processing
  □ Cython for C-speed Python
  □ multiprocessing for CPU parallelism
  □ C extension or Rust (PyO3)
```

---

## 📝 Practice Tasks

### Task 1: Benchmark Battle
Compare performance of at least 5 pairs of approaches:
- list vs set for membership testing (10, 1K, 100K, 1M items)
- dict comprehension vs manual loop
- f-string vs .format() vs %
- sorted() with lambda vs operator.itemgetter
- list.append loop vs list comprehension
Create a formatted comparison table with timeit results.

### Task 2: Profile and Optimize
Write a deliberately slow function that processes a list of 100K records (with nested loops, string concatenation, repeated list scans, etc.). Profile it with cProfile. Then optimize it step by step, profiling after each change. Document the speedup at each step.

### Task 3: Memory Optimizer
Create a program that stores 1 million "user" records. Implement three versions:
- Regular dicts
- Regular classes
- Classes with `__slots__`
- NamedTuples
- Dataclasses
Compare memory usage using `tracemalloc`. Print a table.

### Task 4: LRU Cache Implementation
Implement your own LRU cache using `OrderedDict` (like the example above). Benchmark it against `functools.lru_cache` on a recursive fibonacci function. Also compare with an uncached version.

### Task 5: Streaming Aggregator
Process a "large" CSV (generate 1M rows) that doesn't fit in memory. Calculate: sum, average, min, max, count per category — all in a single pass using generators and streaming aggregation. Compare memory with loading everything into a list.

### Task 6: Optimization Challenge
Given this slow code, make it at least 10x faster:
```python
def find_common_friends(users):
    """Find pairs of users who share the most friends."""
    results = []
    for i in range(len(users)):
        for j in range(i+1, len(users)):
            common = []
            for friend in users[i]["friends"]:
                if friend in users[j]["friends"]:
                    common.append(friend)
            if len(common) > 5:
                results.append((users[i]["name"], users[j]["name"], len(common)))
    results.sort(key=lambda x: x[2], reverse=True)
    return results[:10]
```
Profile before and after. Explain each optimization.

---

## 📚 Resources

- [Python Official — timeit](https://docs.python.org/3/library/timeit.html)
- [Python Official — cProfile](https://docs.python.org/3/library/profile.html)
- [Python Official — tracemalloc](https://docs.python.org/3/library/tracemalloc.html)
- [Python Official — collections](https://docs.python.org/3/library/collections.html)
- [Python Official — itertools](https://docs.python.org/3/library/itertools.html)
- [Real Python — Profiling](https://realpython.com/python-profiling/)
- [Real Python — collections](https://realpython.com/python-collections-module/)
- [Python Speed — Performance Tips](https://wiki.python.org/moin/PythonSpeed/PerformanceTips)
- [Python TimeComplexity Wiki](https://wiki.python.org/moin/TimeComplexity)
- [snakeviz — Profile Visualizer](https://jiffyclub.github.io/snakeviz/)

---

## 🔑 Key Takeaways

1. **Profile first, optimize second** — never guess. `cProfile` for functions, `line_profiler` for lines, `tracemalloc` for memory.
2. **Algorithm > micro-optimization** — switching from O(n²) to O(n) matters more than any trick.
3. **Use the right data structure** — `set` for lookups, `deque` for queues, `defaultdict` for grouping.
4. **Built-in functions are C-fast** — `sum()`, `sorted()`, `min()`, `max()` beat Python loops.
5. **List comprehensions > loops** — 30-50% faster, more Pythonic.
6. **Generators save memory** — use `()` instead of `[]` when you iterate once.
7. **`__slots__`** for memory-heavy classes — 3x less memory per instance.
8. **`"".join()`** for string building — O(n) instead of O(n²) concatenation.
9. **`operator.itemgetter`** over `lambda` — faster for sorting and key functions.
10. **`itertools`** for efficient iteration — `chain`, `islice`, `batched` avoid unnecessary copies.

---

> **Tomorrow (Day 14):** Testing & Code Quality — pytest, fixtures, parametrize, mocking, doctests, tox, type checking. The final day of Python core before databases!

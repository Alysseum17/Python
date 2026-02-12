# Day 6: Iterators, Generators & Generator Pipelines

## 🎯 Goal

Master Python's iteration protocols and learn how to build memory-efficient data pipelines using generators. This is where Python shines for data processing.

---

## 1. The Iterator Protocol

Every `for` loop in Python uses the iterator protocol under the hood. Understanding it gives you superpowers.

### How iteration actually works

```python
# When you write:
for item in [1, 2, 3]:
    print(item)

# Python actually does:
iterator = iter([1, 2, 3])  # Get iterator from iterable
while True:
    try:
        item = next(iterator)   # Get next item
        print(item)
    except StopIteration:       # No more items
        break
```

### Key concepts

```python
# Iterable: any object that can return an iterator
# Has __iter__() method
# Examples: list, tuple, dict, set, string, file, range

# Iterator: object that produces values one at a time
# Has __iter__() (returns self) and __next__() methods
# Maintains state, can only go forward, can be exhausted

# Check if something is iterable
from collections.abc import Iterable, Iterator

isinstance([1, 2, 3], Iterable)  # True
isinstance([1, 2, 3], Iterator)  # False (list is iterable, not iterator)

it = iter([1, 2, 3])
isinstance(it, Iterator)         # True
```

### Building custom iterators

```python
# Example: Range-like iterator
class Counter:
    def __init__(self, start, end):
        self.current = start
        self.end = end
    
    def __iter__(self):
        return self  # Iterator returns itself
    
    def __next__(self):
        if self.current >= self.end:
            raise StopIteration
        value = self.current
        self.current += 1
        return value

# Usage
for num in Counter(0, 5):
    print(num)  # 0, 1, 2, 3, 4

# Manual iteration
counter = Counter(0, 3)
print(next(counter))  # 0
print(next(counter))  # 1
print(next(counter))  # 2
# print(next(counter))  # StopIteration error
```

### Why this matters for data engineering

```python
# BAD: Loads entire file into memory
def read_file_bad(filename):
    with open(filename) as f:
        return f.readlines()  # All lines in memory at once!

lines = read_file_bad('huge_log.txt')  # 💥 Could crash with large files
for line in lines:
    process(line)

# GOOD: Iterates line by line
def read_file_good(filename):
    with open(filename) as f:
        for line in f:  # File objects are iterators!
            yield line  # We'll cover yield next

# Only one line in memory at a time
for line in read_file_good('huge_log.txt'):
    process(line)  # ✅ Handles files of any size
```

---

## 2. Generators: Iterators on Steroids

Generators are the **most important concept** in this lesson. They let you create iterators with simple functions.

### The `yield` keyword

```python
# Instead of building a class, just use yield
def count_up_to(n):
    i = 0
    while i < n:
        yield i  # Pauses here, returns value, resumes on next call
        i += 1

# Usage is identical to custom iterator
for num in count_up_to(5):
    print(num)  # 0, 1, 2, 3, 4

# It's a generator object
gen = count_up_to(3)
print(type(gen))    # <class 'generator'>
print(next(gen))    # 0
print(next(gen))    # 1
print(next(gen))    # 2
# print(next(gen))  # StopIteration
```

### Generators maintain state between calls

```python
def fibonacci():
    a, b = 0, 1
    while True:  # Infinite generator!
        yield a
        a, b = b, a + b

# Take only what you need
fib = fibonacci()
for _ in range(10):
    print(next(fib))  # 0, 1, 1, 2, 3, 5, 8, 13, 21, 34

# Or use itertools
from itertools import islice
print(list(islice(fibonacci(), 10)))  # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

### Generator vs Regular Function

```python
# Regular function: returns once, forgets everything
def get_numbers_list(n):
    result = []
    for i in range(n):
        result.append(i * i)
    return result  # Builds entire list in memory

squares = get_numbers_list(1000000)  # All million squares in memory!

# Generator: yields values one at a time, maintains state
def get_numbers_gen(n):
    for i in range(n):
        yield i * i  # Returns one value, remembers where it was

squares = get_numbers_gen(1000000)  # Almost no memory used!
for sq in squares:
    if sq > 100:
        break  # We can stop early without computing everything
```

### Real-world example: Processing large datasets

```python
# You have a 10GB CSV file with 100M rows
# BAD: Load everything
import csv

def load_all_users(filename):
    users = []
    with open(filename) as f:
        reader = csv.DictReader(f)
        for row in reader:
            users.append(row)  # 10GB in memory!
    return users

# GOOD: Process one at a time
def stream_users(filename):
    with open(filename) as f:
        reader = csv.DictReader(f)
        for row in reader:
            yield row  # Only current row in memory

# Filter active users without loading everything
def get_active_users(filename):
    for user in stream_users(filename):
        if user['status'] == 'active':
            yield user

# Chain operations
active_count = sum(1 for _ in get_active_users('users.csv'))
```

---

## 3. Generator Expressions

Like list comprehensions, but lazy (don't build the whole list).

```python
# List comprehension: builds entire list immediately
squares_list = [x**2 for x in range(1000000)]  # All in memory
print(type(squares_list))  # <class 'list'>

# Generator expression: computes on demand
squares_gen = (x**2 for x in range(1000000))   # Almost no memory
print(type(squares_gen))   # <class 'generator'>

# Compare memory usage
import sys
print(sys.getsizeof(squares_list))  # ~8MB
print(sys.getsizeof(squares_gen))   # ~200 bytes!

# Use cases
# Sum without building list
total = sum(x**2 for x in range(1000000))

# Check if any match (stops at first True)
has_large = any(x > 1000 for x in range(10000))

# Get first match (doesn't compute rest)
first_big = next(x for x in range(1000) if x > 500)
```

### When to use which

```python
# Use list comprehension when:
# - Need to iterate multiple times
# - Need len(), indexing, slicing
# - Small dataset that fits in memory
numbers = [x for x in range(100)]
print(numbers[50])  # Can index
print(len(numbers)) # Can get length

# Use generator expression when:
# - One-time iteration
# - Large dataset
# - Feeding to functions like sum(), max(), any(), all()
# - Chaining operations
total = sum(x**2 for x in range(1000000))  # Perfect use case
```

---

## 4. Generator Pipelines (Data Engineering Pattern)

Chain generators to build data processing pipelines. Each stage processes one item at a time.

### Basic pipeline

```python
# Process log files efficiently
def read_logs(filename):
    """Stage 1: Read lines"""
    with open(filename) as f:
        for line in f:
            yield line.strip()

def parse_logs(lines):
    """Stage 2: Parse lines"""
    for line in lines:
        parts = line.split('|')
        if len(parts) == 4:
            timestamp, level, module, message = parts
            yield {
                'timestamp': timestamp,
                'level': level,
                'module': module,
                'message': message
            }

def filter_errors(logs):
    """Stage 3: Filter for errors"""
    for log in logs:
        if log['level'] == 'ERROR':
            yield log

def extract_modules(logs):
    """Stage 4: Extract module names"""
    for log in logs:
        yield log['module']

# Build the pipeline
pipeline = extract_modules(
    filter_errors(
        parse_logs(
            read_logs('app.log')
        )
    )
)

# Process (only happens when we iterate!)
error_modules = set(pipeline)
print(error_modules)
```

### More readable with assignment

```python
# Same pipeline, more readable
lines = read_logs('app.log')
logs = parse_logs(lines)
errors = filter_errors(logs)
modules = extract_modules(errors)

# Still lazy! Nothing computed until:
for module in modules:
    print(module)
```

### Real-world example: ETL pipeline

```python
import json
from datetime import datetime

def extract_from_api(url, batch_size=100):
    """Extract: Fetch data in batches"""
    offset = 0
    while True:
        # Imagine this fetches from API
        response = fetch_api(f"{url}?offset={offset}&limit={batch_size}")
        if not response:
            break
        for record in response:
            yield record
        offset += batch_size

def transform_user_records(records):
    """Transform: Clean and reshape data"""
    for record in records:
        # Skip invalid records
        if not record.get('email'):
            continue
        
        # Transform
        yield {
            'user_id': record['id'],
            'email': record['email'].lower(),
            'signup_date': datetime.fromisoformat(record['created']),
            'is_active': record.get('status') == 'active',
            'metadata': json.dumps(record.get('metadata', {}))
        }

def batch_records(records, size=1000):
    """Helper: Batch records for bulk insert"""
    batch = []
    for record in records:
        batch.append(record)
        if len(batch) >= size:
            yield batch
            batch = []
    if batch:  # Don't forget the last batch!
        yield batch

def load_to_db(batches, db_connection):
    """Load: Bulk insert to database"""
    for batch in batches:
        db_connection.bulk_insert('users', batch)
        yield len(batch)  # Yield progress

# Complete ETL pipeline
raw_data = extract_from_api('https://api.example.com/users')
clean_data = transform_user_records(raw_data)
batched_data = batch_records(clean_data, size=1000)
loaded_counts = load_to_db(batched_data, db)

# Process and show progress
total = sum(loaded_counts)  # Triggers the entire pipeline
print(f"Loaded {total} records")
```

### Why this is powerful for Data Engineering

```python
# Traditional approach (bad for large data):
# 1. Load all data into memory
# 2. Process all data
# 3. Write all data

all_data = fetch_all_from_api()      # 10GB in memory
cleaned = clean_all(all_data)         # Another 10GB
write_all_to_db(cleaned)              # 20GB total!

# Generator pipeline approach:
# Only one record (or small batch) in memory at any time
raw = fetch_streaming()               # Yields one record
clean = transform_streaming(raw)      # Processes one record
load_streaming(clean)                 # Writes one record
# Total memory: ~1KB for current record!
```

---

## 5. `itertools` Module (Must-Know for Data Work)

The standard library's secret weapon for efficient iteration.

```python
from itertools import (
    islice, count, cycle, repeat,
    chain, compress, dropwhile, takewhile,
    groupby, accumulate, product, combinations, permutations
)

# Infinite iterators
# count(start, step) - infinite counter
for i in islice(count(10, 2), 5):
    print(i)  # 10, 12, 14, 16, 18

# cycle(iterable) - infinite repetition
counter = 0
for color in cycle(['red', 'green', 'blue']):
    print(color)
    counter += 1
    if counter >= 7:
        break
# red, green, blue, red, green, blue, red

# repeat(value, times) - repeat a value
list(repeat(10, 3))  # [10, 10, 10]

# Combinatoric iterators
list(product([1, 2], ['a', 'b']))
# [(1, 'a'), (1, 'b'), (2, 'a'), (2, 'b')]

list(combinations([1, 2, 3], 2))
# [(1, 2), (1, 3), (2, 3)]

list(permutations([1, 2, 3], 2))
# [(1, 2), (1, 3), (2, 1), (2, 3), (3, 1), (3, 2)]

# Filtering iterators
# dropwhile(predicate, iterable) - drop while condition is True
list(dropwhile(lambda x: x < 5, [1, 3, 6, 2, 8]))
# [6, 2, 8] - stops dropping after first False

# takewhile(predicate, iterable) - take while condition is True
list(takewhile(lambda x: x < 5, [1, 3, 6, 2, 8]))
# [1, 3] - stops taking at first False

# compress(data, selectors) - filter by boolean mask
list(compress(['a', 'b', 'c', 'd'], [1, 0, 1, 0]))
# ['a', 'c']

# Grouping and accumulating
# groupby(iterable, key) - group consecutive elements
data = [
    {'type': 'A', 'value': 1},
    {'type': 'A', 'value': 2},
    {'type': 'B', 'value': 3},
    {'type': 'B', 'value': 4},
]

for key, group in groupby(data, key=lambda x: x['type']):
    print(key, list(group))
# A [{'type': 'A', 'value': 1}, {'type': 'A', 'value': 2}]
# B [{'type': 'B', 'value': 3}, {'type': 'B', 'value': 4}]

# accumulate(iterable, func) - running totals
from operator import mul
list(accumulate([1, 2, 3, 4]))        # [1, 3, 6, 10] (running sum)
list(accumulate([1, 2, 3, 4], mul))   # [1, 2, 6, 24] (running product)

# Chaining iterables
list(chain([1, 2], [3, 4], [5, 6]))   # [1, 2, 3, 4, 5, 6]
list(chain.from_iterable([[1, 2], [3, 4]]))  # [1, 2, 3, 4]
```

### Real data engineering use case with itertools

```python
from itertools import islice, groupby
from operator import itemgetter

def process_events_efficiently(events):
    """
    Process event stream:
    - Skip first 100 (warmup)
    - Take next 10000
    - Group by user
    - Calculate stats per user
    """
    # Skip warmup events
    events_iter = iter(events)
    skipped = islice(events_iter, 100, None)  # Skip first 100
    
    # Take only next 10000
    limited = islice(skipped, 10000)
    
    # Sort by user_id (required for groupby)
    # In practice, you'd use sorted() here, but that breaks streaming
    # So you'd ensure data is pre-sorted or use different approach
    
    # Group by user
    for user_id, user_events in groupby(limited, key=itemgetter('user_id')):
        events_list = list(user_events)
        
        yield {
            'user_id': user_id,
            'event_count': len(events_list),
            'first_event': events_list[0]['timestamp'],
            'last_event': events_list[-1]['timestamp']
        }

# Usage
user_stats = process_events_efficiently(stream_from_kafka())
for stats in user_stats:
    print(stats)
```

---

## 6. `yield from` (Generator Delegation)

Delegate to another generator - very useful for recursive generators.

```python
# Without yield from
def flatten_bad(nested_list):
    for item in nested_list:
        if isinstance(item, list):
            for subitem in flatten_bad(item):
                yield subitem
        else:
            yield item

# With yield from (cleaner)
def flatten(nested_list):
    for item in nested_list:
        if isinstance(item, list):
            yield from flatten(item)  # Delegate to recursive call
        else:
            yield item

# Usage
nested = [1, [2, 3, [4, 5]], 6, [7, [8, 9]]]
print(list(flatten(nested)))  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Another example: chain multiple generators
def gen1():
    yield 1
    yield 2

def gen2():
    yield 3
    yield 4

def combined():
    yield from gen1()
    yield from gen2()

list(combined())  # [1, 2, 3, 4]
```

---

## 7. Generator Best Practices for Data Engineering

### 1. Always close files properly (use context managers)

```python
# BAD
def read_csv_bad(filename):
    f = open(filename)
    for line in f:
        yield line
    # f never gets closed if consumer stops early!

# GOOD
def read_csv_good(filename):
    with open(filename) as f:
        for line in f:
            yield line
    # Context manager ensures cleanup
```

### 2. Handle errors in pipelines

```python
def robust_pipeline(records):
    for record in records:
        try:
            # Process record
            yield process(record)
        except ValueError as e:
            # Log error but continue
            logger.error(f"Failed to process record: {e}")
            continue
```

### 3. Add progress tracking

```python
def with_progress(iterable, total=None):
    """Wrap any iterable with progress tracking"""
    count = 0
    for item in iterable:
        count += 1
        if count % 1000 == 0:
            print(f"Processed {count} items...")
        yield item

# Usage
records = with_progress(stream_from_db(), total=1000000)
for record in records:
    process(record)
```

### 4. Make generators reusable (when needed)

```python
# Generators are single-use
gen = (x for x in range(3))
list(gen)  # [0, 1, 2]
list(gen)  # [] - exhausted!

# Pattern 1: Wrap in a function
def make_generator():
    return (x for x in range(3))

gen1 = make_generator()
gen2 = make_generator()  # Fresh generator

# Pattern 2: Use itertools.tee (for when you need 2+ independent iterators)
from itertools import tee

gen = (x for x in range(3))
gen1, gen2 = tee(gen)  # Two independent iterators
list(gen1)  # [0, 1, 2]
list(gen2)  # [0, 1, 2]
```

---

## 📝 Practice Tasks

### Task 1: Infinite Sequence Generator
Create a generator that produces the Collatz sequence starting from any number:
- If n is even: n → n/2
- If n is odd: n → 3n + 1
- Stop when reaching 1

Test with n=27 and count how many steps it takes.

### Task 2: CSV Processor Pipeline
Build a pipeline that:
1. Reads a CSV file line by line
2. Parses each line into a dict
3. Filters rows where a numeric column > threshold
4. Transforms data (e.g., normalize strings, convert types)
5. Batches records into groups of N
6. Prints batch summaries

Use generator functions for each stage. Test with a file you create.

### Task 3: Log File Analyzer
Create generators to:
1. Read log file (format: `timestamp|level|message`)
2. Parse into structured data
3. Filter by time range
4. Group by log level
5. Count occurrences

Process a multi-MB log file efficiently.

### Task 4: Fibonacci with Cache
Implement a generator that:
- Produces Fibonacci numbers
- Optionally caches results (hint: use a dict)
- Can resume from any position
- Compare memory usage with list-based approach for first 10,000 numbers

### Task 5: Data Pipeline with itertools
Build a pipeline using `itertools` that:
1. Generates infinite user IDs (count)
2. Cycles through 3 user types
3. Combines them into user records
4. Groups consecutive users by type
5. Takes only first 50 users
6. Calculates statistics per group

Use `count`, `cycle`, `islice`, `groupby`.

### Task 6: File Splitter
Write a generator that:
- Reads a large file
- Yields chunks of N lines
- Each chunk should be writable to a separate file
- Implement both: by line count AND by byte size

Test by splitting a file into 3 pieces.

---

## 🔗 Resources

- [Python Generators Official Tutorial](https://docs.python.org/3/tutorial/classes.html#generators)
- [PEP 255 - Simple Generators](https://www.python.org/dev/peps/pep-0255/)
- [Python itertools Documentation](https://docs.python.org/3/library/itertools.html)
- [Real Python - Generators Guide](https://realpython.com/introduction-to-python-generators/)
- [David Beazley - Generator Tricks for Systems Programmers](http://www.dabeaz.com/generators/) (Classic!)

---

> **Tomorrow (Day 7):** Decorators & Context Managers - Python's "magic" syntax for wrapping functionality.

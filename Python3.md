# Day 3: Data Structures Deep Dive

## 🎯 Goal

Master Python's core data structures: lists, tuples, sets, dicts. Then learn comprehensions — the most "Pythonic" feature that has no real equivalent in Java (and only partial in JS). This day is **critical** for everything that follows, especially Data Engineering.

---

## 1. Lists — Python's Array

Lists are like JS arrays or Java `ArrayList`. Ordered, mutable, allow duplicates.

```python
# Creation
nums = [1, 2, 3, 4, 5]
mixed = [1, "hello", True, 3.14, None]  # any types (like JS)
empty = []
from_range = list(range(10))  # [0, 1, 2, ..., 9]

# Indexing & slicing (same as strings — Day 1)
nums[0]      # 1
nums[-1]     # 5
nums[1:3]    # [2, 3]
nums[::-1]   # [5, 4, 3, 2, 1] (reversed)
```

### List methods — comparison with JS

```python
fruits = ["apple", "banana"]

# Adding
fruits.append("cherry")           # push() in JS → ["apple", "banana", "cherry"]
fruits.insert(1, "blueberry")     # splice(1,0,"blueberry") → inserts at index
fruits.extend(["date", "elderberry"])  # like spread: [...arr, ...other]

# Removing
fruits.remove("banana")           # removes first occurrence (ValueError if not found)
popped = fruits.pop()             # removes & returns last (like JS .pop())
popped = fruits.pop(0)            # removes & returns at index (like JS .shift() for 0)
fruits.clear()                    # empties the list

# Searching
nums = [3, 1, 4, 1, 5, 9, 2, 6]
nums.index(4)         # 2 (index of first occurrence, ValueError if missing)
nums.count(1)         # 2 (how many times 1 appears)
4 in nums             # True (like JS .includes())
4 not in nums         # False

# Sorting
nums.sort()                  # in-place, returns None! (not the list)
nums.sort(reverse=True)      # descending
sorted_nums = sorted(nums)   # returns NEW sorted list (original unchanged)

# ⚠️ IMPORTANT: .sort() returns None, not the list!
# result = nums.sort()  →  result is None!  This catches everyone.
# Use sorted() if you need the result as a value.

# Other
nums.reverse()               # in-place reverse
nums_copy = nums.copy()      # shallow copy (same as nums[:] or list(nums))
len(nums)                    # length (not .length property like JS)
```

### List as stack and queue

```python
# Stack (LIFO) — lists are great for this
stack = []
stack.append("a")    # push
stack.append("b")
stack.pop()          # "b" — pop

# Queue (FIFO) — DON'T use list, it's O(n) for pop(0)
# Use deque instead (Day 13), but for now:
from collections import deque
queue = deque()
queue.append("a")     # enqueue
queue.append("b")
queue.popleft()       # "a" — dequeue, O(1)
```

### Shallow vs deep copy — important!

```python
import copy

original = [[1, 2], [3, 4]]

# Shallow copy — inner lists are still shared!
shallow = original.copy()  # or original[:] or list(original)
shallow[0].append(99)
print(original)  # [[1, 2, 99], [3, 4]] — MODIFIED!

# Deep copy — completely independent
original = [[1, 2], [3, 4]]
deep = copy.deepcopy(original)
deep[0].append(99)
print(original)  # [[1, 2], [3, 4]] — safe!

# Same concept as JS:
# shallow: [...arr] or Array.from(arr)
# deep: structuredClone(arr) or JSON.parse(JSON.stringify(arr))
```

---

## 2. Tuples — Immutable Lists

Tuples are like frozen lists. Once created, they can't be modified. No direct JS/Java equivalent (Java records are close but not the same).

```python
# Creation
point = (3, 4)
single = (42,)       # ⚠️ trailing comma required for single-element tuple!
not_tuple = (42)     # This is just int 42 in parentheses!
empty = ()
from_list = tuple([1, 2, 3])

# Indexing works same as lists
point[0]    # 3
point[-1]   # 4
point[0:1]  # (3,)

# Immutable — can't modify!
# point[0] = 10  # ❌ TypeError: 'tuple' object does not support item assignment
```

### When to use tuples vs lists?

```python
# Use TUPLES when:
# 1. Data shouldn't change (coordinates, RGB colors, database rows)
point = (10, 20)
color = (255, 128, 0)

# 2. Dictionary keys (lists can't be dict keys because they're mutable)
locations = {
    (40.7128, -74.0060): "New York",
    (50.4501, 30.5234): "Kyiv",
}

# 3. Function returns (multiple values)
def get_user():
    return "Alice", 25, "Kyiv"  # returns a tuple (parentheses optional)

name, age, city = get_user()  # tuple unpacking (like JS destructuring)

# 4. Performance — tuples are slightly faster and use less memory
import sys
print(sys.getsizeof([1, 2, 3]))   # 88 bytes (list)
print(sys.getsizeof((1, 2, 3)))   # 64 bytes (tuple)
```

### Tuple unpacking — very Pythonic

```python
# Basic unpacking (like JS destructuring)
x, y = (10, 20)
# JS: const [x, y] = [10, 20]

# Swap variables — no temp needed!
a, b = 1, 2
a, b = b, a  # a=2, b=1 (Python magic, works because right side is evaluated first)

# Star unpacking — like JS rest
first, *rest = [1, 2, 3, 4, 5]
# first = 1, rest = [2, 3, 4, 5]

first, *middle, last = [1, 2, 3, 4, 5]
# first = 1, middle = [2, 3, 4], last = 5

# Ignore values with _
_, _, third = (1, 2, 3)   # only need third
x, *_ = (1, 2, 3, 4, 5)  # only need first

# Nested unpacking
(a, b), (c, d) = (1, 2), (3, 4)
# a=1, b=2, c=3, d=4

# Real-world: iterating pairs
pairs = [(1, "one"), (2, "two"), (3, "three")]
for num, word in pairs:
    print(f"{num} = {word}")
```

### Named tuples — tuples with field names

```python
from collections import namedtuple

# Like a lightweight class / Java record / TS interface
Point = namedtuple("Point", ["x", "y"])
User = namedtuple("User", "name age city")  # string syntax also works

p = Point(3, 4)
print(p.x)      # 3 (access by name)
print(p[0])     # 3 (still works by index)
print(p)        # Point(x=3, y=4)

# Immutable like regular tuples
# p.x = 10  # ❌ AttributeError

# Convert to dict
p._asdict()  # {'x': 3, 'y': 4}

# Create modified copy
p2 = p._replace(x=10)  # Point(x=10, y=4)

# Modern alternative: typing.NamedTuple (with type hints)
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float
    z: float = 0.0  # default value

p = Point(1.0, 2.0)
print(p)  # Point(x=1.0, y=2.0, z=0.0)
```

---

## 3. Sets — Unique Collections

Sets are unordered collections of unique elements. Like Java `HashSet` or JS `Set`.

```python
# Creation
fruits = {"apple", "banana", "cherry"}
empty_set = set()   # ⚠️ NOT {} — that's an empty dict!
from_list = set([1, 2, 2, 3, 3, 3])  # {1, 2, 3} — duplicates removed

# Adding/removing
fruits.add("date")
fruits.discard("banana")   # remove, no error if missing
fruits.remove("banana")    # remove, KeyError if missing
fruits.pop()               # remove and return arbitrary element
```

### Set operations — this is where sets shine

```python
a = {1, 2, 3, 4, 5}
b = {4, 5, 6, 7, 8}

# Union — all elements from both
a | b                # {1, 2, 3, 4, 5, 6, 7, 8}
a.union(b)           # same

# Intersection — only elements in both
a & b                # {4, 5}
a.intersection(b)    # same

# Difference — in a but not in b
a - b                # {1, 2, 3}
a.difference(b)      # same

# Symmetric difference — in either but not both
a ^ b                # {1, 2, 3, 6, 7, 8}
a.symmetric_difference(b)  # same

# Subset/superset
{1, 2}.issubset({1, 2, 3})       # True
{1, 2, 3}.issuperset({1, 2})     # True
{1, 2}.isdisjoint({3, 4})        # True (no common elements)
```

### Practical set use cases

```python
# 1. Remove duplicates (most common use!)
names = ["Alice", "Bob", "Alice", "Charlie", "Bob"]
unique_names = list(set(names))  # ["Alice", "Bob", "Charlie"] (order not guaranteed)

# Preserve order while deduplicating (Python 3.7+, dict preserves insertion order)
unique_ordered = list(dict.fromkeys(names))  # ["Alice", "Bob", "Charlie"]

# 2. Fast membership testing — O(1) vs O(n) for lists!
valid_statuses = {"active", "pending", "suspended"}

status = "active"
if status in valid_statuses:  # O(1) lookup
    print("Valid!")

# vs list — O(n) lookup
# if status in ["active", "pending", "suspended"]:  # slower for large collections

# 3. Find common elements between datasets
users_jan = {"alice", "bob", "charlie", "dave"}
users_feb = {"charlie", "dave", "eve", "frank"}

retained = users_jan & users_feb     # {"charlie", "dave"}
churned = users_jan - users_feb      # {"alice", "bob"}
new_users = users_feb - users_jan    # {"eve", "frank"}

# 4. Compare lists regardless of order
list1 = [3, 1, 2]
list2 = [2, 3, 1]
set(list1) == set(list2)  # True
```

### Frozen sets — immutable sets

```python
# Can be used as dict keys or elements of other sets
fs = frozenset([1, 2, 3])
# fs.add(4)  # ❌ AttributeError — immutable

# Use case: set of sets
groups = {frozenset({1, 2}), frozenset({3, 4})}
```

---

## 4. Dictionaries — Python's Object/Map

Dicts are like JS objects or Java `HashMap`. As of Python 3.7+, they **preserve insertion order**.

```python
# Creation
user = {"name": "Alice", "age": 25, "city": "Kyiv"}
empty = {}
from_pairs = dict([("a", 1), ("b", 2)])    # from list of tuples
from_kwargs = dict(name="Alice", age=25)    # keyword style
from_keys = dict.fromkeys(["a", "b", "c"], 0)  # {"a": 0, "b": 0, "c": 0}

# Access
user["name"]           # "Alice" (KeyError if missing — like Java map.get() without default)
user.get("name")       # "Alice" (None if missing — safe)
user.get("email", "N/A")  # "N/A" (custom default)

# ⚠️ Key difference from JS:
# JS: user.name or user["name"] — both work
# Python: user["name"] only — dot notation is for attributes, not dict keys!
# user.name  →  ❌ AttributeError
```

### Dict methods

```python
user = {"name": "Alice", "age": 25}

# Setting
user["email"] = "alice@example.com"   # add/update
user.update({"age": 26, "city": "Kyiv"})  # merge (like JS Object.assign or {...obj, ...other})

# Python 3.9+ merge operators (like JS spread)
defaults = {"theme": "dark", "lang": "en"}
custom = {"lang": "uk", "font_size": 14}
merged = defaults | custom   # {"theme": "dark", "lang": "uk", "font_size": 14}
# Later values overwrite earlier ones — like JS {...defaults, ...custom}

defaults |= custom  # in-place merge (like Object.assign(defaults, custom))

# Removing
del user["email"]                     # KeyError if missing
removed = user.pop("age")            # removes and returns value (KeyError if missing)
removed = user.pop("missing", None)  # returns None if missing (safe)
last = user.popitem()                # removes last inserted (key, value) pair

# Checking
"name" in user          # True (checks keys, not values!)
"Alice" in user         # False (not a key)
"Alice" in user.values()  # True

# Views
user.keys()    # dict_keys(["name", ...])
user.values()  # dict_values(["Alice", ...])
user.items()   # dict_items([("name", "Alice"), ...])
# These are VIEWS — they update when dict changes (not copies)
```

### Iterating dicts

```python
user = {"name": "Alice", "age": 25, "city": "Kyiv"}

# Keys (default)
for key in user:
    print(key)  # name, age, city

# Values
for value in user.values():
    print(value)  # Alice, 25, Kyiv

# Key-value pairs (most common)
for key, value in user.items():
    print(f"{key}: {value}")

# JS comparison:
# Object.keys(user).forEach(key => ...)
# Object.entries(user).forEach(([key, value]) => ...)
```

### `setdefault` and `defaultdict` — handling missing keys

```python
# setdefault — get value, or set and return default if missing
word_counts = {}
for word in ["apple", "banana", "apple", "cherry", "banana", "apple"]:
    word_counts.setdefault(word, 0)
    word_counts[word] += 1
# {"apple": 3, "banana": 2, "cherry": 1}

# defaultdict — automatic default values (cleaner)
from collections import defaultdict

# Counts
word_counts = defaultdict(int)  # missing keys default to 0
for word in ["apple", "banana", "apple", "cherry"]:
    word_counts[word] += 1

# Grouping
groups = defaultdict(list)  # missing keys default to []
students = [("math", "Alice"), ("physics", "Bob"), ("math", "Charlie")]
for subject, name in students:
    groups[subject].append(name)
# {"math": ["Alice", "Charlie"], "physics": ["Bob"]}

# Nested dicts
nested = defaultdict(lambda: defaultdict(int))
nested["2024"]["January"] += 100
nested["2024"]["February"] += 200
```

### Dict ordering and sorting

```python
# Dicts preserve insertion order since Python 3.7
d = {"b": 2, "a": 1, "c": 3}

# Sort by key
sorted_by_key = dict(sorted(d.items()))
# {"a": 1, "b": 2, "c": 3}

# Sort by value
sorted_by_value = dict(sorted(d.items(), key=lambda item: item[1]))
# {"a": 1, "b": 2, "c": 3}

# Sort by value descending
sorted_desc = dict(sorted(d.items(), key=lambda item: item[1], reverse=True))
# {"c": 3, "b": 2, "a": 1}
```

---

## 5. Comprehensions — The Most Pythonic Feature 🐍

Comprehensions are concise ways to create lists, dicts, and sets. They replace `map`/`filter` patterns and are considered **the** idiomatic way to transform data in Python.

### List comprehensions

```python
# Basic: [expression for item in iterable]
squares = [x ** 2 for x in range(10)]
# [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# With filter: [expression for item in iterable if condition]
even_squares = [x ** 2 for x in range(10) if x % 2 == 0]
# [0, 4, 16, 36, 64]

# JS equivalent:
# const squares = Array.from({length: 10}, (_, i) => i ** 2)
# const evenSquares = [...Array(10).keys()].filter(x => x % 2 === 0).map(x => x ** 2)
# Python is MUCH cleaner here!

# With transformation
names = ["  Alice  ", "BOB", "  charlie"]
clean = [name.strip().title() for name in names]
# ["Alice", "Bob", "Charlie"]

# Nested loops
pairs = [(x, y) for x in range(3) for y in range(3)]
# [(0,0), (0,1), (0,2), (1,0), (1,1), (1,2), (2,0), (2,1), (2,2)]

# Flatten nested lists
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [num for row in matrix for num in row]
# [1, 2, 3, 4, 5, 6, 7, 8, 9]
# Read it as: for row in matrix → for num in row → num

# If-else in expression (not filter — note the position!)
labels = ["even" if x % 2 == 0 else "odd" for x in range(5)]
# ["even", "odd", "even", "odd", "even"]

# ⚠️ Position matters!
# [x for x in items if condition]           ← filter (at the end)
# [a if condition else b for x in items]    ← transform (before for)
```

### Dict comprehensions

```python
# Basic: {key_expr: value_expr for item in iterable}
squares = {x: x ** 2 for x in range(6)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16, 5: 25}

# Swap keys and values
original = {"a": 1, "b": 2, "c": 3}
swapped = {v: k for k, v in original.items()}
# {1: "a", 2: "b", 3: "c"}

# Filter dict
scores = {"Alice": 85, "Bob": 62, "Charlie": 91, "Dave": 58}
passed = {name: score for name, score in scores.items() if score >= 70}
# {"Alice": 85, "Charlie": 91}

# From two lists (like JS Object.fromEntries + zip)
keys = ["name", "age", "city"]
values = ["Alice", 25, "Kyiv"]
user = dict(zip(keys, values))
# {"name": "Alice", "age": 25, "city": "Kyiv"}
# Or with comprehension:
user = {k: v for k, v in zip(keys, values)}

# Count characters
text = "hello world"
char_count = {char: text.count(char) for char in set(text)}
# {"h": 1, "e": 1, "l": 3, "o": 2, " ": 1, "w": 1, "r": 1, "d": 1}
```

### Set comprehensions

```python
# Basic: {expression for item in iterable}
unique_lengths = {len(word) for word in ["hello", "world", "hi", "hey"]}
# {2, 3, 5}

# From a sentence — unique words
words = {word.lower() for word in "The cat sat on the mat".split()}
# {"the", "cat", "sat", "on", "mat"}
```

### Generator expressions — lazy comprehensions

```python
# Generator: (expression for item in iterable)
# Like a list comprehension but LAZY — produces values one at a time
# Saves memory for large datasets!

# List comprehension — creates entire list in memory
sum([x ** 2 for x in range(1_000_000)])  # ~8MB of memory for the list

# Generator expression — almost zero memory
sum(x ** 2 for x in range(1_000_000))    # values computed one at a time

# When to use generators vs lists:
# - Use generator when you're just iterating once (sum, max, min, any, all, for loop)
# - Use list when you need to access items multiple times or by index

# Check if any number is even
any(x % 2 == 0 for x in [1, 3, 5, 7, 8])  # True

# Check if all numbers are positive
all(x > 0 for x in [1, 2, 3, 4, 5])  # True

# Find first match (like JS .find())
numbers = [1, 4, 7, 10, 13, 16]
first_even = next((x for x in numbers if x % 2 == 0), None)
# 4 (second arg is default if nothing found)
```

### Nested comprehensions — when they help and when to stop

```python
# ✅ Good — matrix transpose
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
transposed = [[row[i] for row in matrix] for i in range(3)]
# [[1, 4, 7], [2, 5, 8], [3, 6, 9]]

# ✅ Good — create a multiplication table
table = {(i, j): i * j for i in range(1, 6) for j in range(1, 6)}
# {(1,1): 1, (1,2): 2, ..., (5,5): 25}

# ❌ Too complex — use a regular loop instead!
# result = [transform(x) for group in data for x in group if condition(x) and other(x)]
# If you need more than one for + one if, write a loop:
result = []
for group in data:
    for x in group:
        if condition(x) and other(x):
            result.append(transform(x))
```

---

## 6. `zip` — Combine Iterables in Parallel

```python
# zip combines element-by-element (like a zipper)
names = ["Alice", "Bob", "Charlie"]
ages = [25, 30, 35]
cities = ["Kyiv", "London", "NYC"]

# Basic zip
list(zip(names, ages))
# [("Alice", 25), ("Bob", 30), ("Charlie", 35)]

# Multiple iterables
for name, age, city in zip(names, ages, cities):
    print(f"{name}, {age}, {city}")

# Unequal lengths — stops at shortest
list(zip([1, 2, 3], ["a", "b"]))  # [(1, "a"), (2, "b")] — 3 is dropped!

# Use zip_longest to pad with None
from itertools import zip_longest
list(zip_longest([1, 2, 3], ["a", "b"], fillvalue="?"))
# [(1, "a"), (2, "b"), (3, "?")]

# Unzip (reverse of zip)
pairs = [("Alice", 25), ("Bob", 30), ("Charlie", 35)]
names, ages = zip(*pairs)  # * unpacks the list of tuples
# names = ("Alice", "Bob", "Charlie")
# ages = (25, 30, 35)

# Create dict from two lists
user_dict = dict(zip(names, ages))
# {"Alice": 25, "Bob": 30, "Charlie": 35}

# Enumerate with index + zip
for i, (name, age) in enumerate(zip(names, ages)):
    print(f"{i}: {name} is {age}")
```

---

## 7. Common Patterns & Idioms

### Counting & grouping

```python
from collections import Counter

# Count occurrences
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
counts = Counter(words)
# Counter({"apple": 3, "banana": 2, "cherry": 1})

counts.most_common(2)     # [("apple", 3), ("banana", 2)]
counts["apple"]           # 3
counts["missing"]         # 0 (not KeyError!)
counts.total()            # 6

# Count characters in string
Counter("mississippi")
# Counter({"s": 4, "i": 4, "p": 2, "m": 1})

# Combine counters
c1 = Counter(a=3, b=1)
c2 = Counter(a=1, b=2)
c1 + c2   # Counter({"a": 4, "b": 3})
c1 - c2   # Counter({"a": 2}) — drops zero/negative
```

### Flattening, transposing, grouping

```python
# Flatten one level
nested = [[1, 2], [3, 4], [5, 6]]
flat = [x for sublist in nested for x in sublist]
# [1, 2, 3, 4, 5, 6]

# Group by property
from itertools import groupby

users = [
    {"name": "Alice", "dept": "engineering"},
    {"name": "Bob", "dept": "engineering"},
    {"name": "Charlie", "dept": "marketing"},
    {"name": "Dave", "dept": "marketing"},
]

# ⚠️ groupby requires sorted input!
users.sort(key=lambda u: u["dept"])
for dept, group in groupby(users, key=lambda u: u["dept"]):
    members = [u["name"] for u in group]
    print(f"{dept}: {members}")
# engineering: ["Alice", "Bob"]
# marketing: ["Charlie", "Dave"]

# Simpler alternative with defaultdict (usually better)
from collections import defaultdict
groups = defaultdict(list)
for u in users:
    groups[u["dept"]].append(u["name"])
```

### Unpacking tricks

```python
# Merge dicts (Python 3.9+)
a = {"x": 1}
b = {"y": 2}
merged = a | b          # {"x": 1, "y": 2}

# Merge dicts (Python 3.5+)
merged = {**a, **b}     # {"x": 1, "y": 2} — like JS {...a, ...b}

# Merge lists
c = [1, 2]
d = [3, 4]
merged_list = [*c, *d]  # [1, 2, 3, 4] — like JS [...c, ...d]
```

---

## 8. Data Structure Selection Guide

| Need | Use | Why |
|------|-----|-----|
| Ordered, mutable collection | `list` | Most versatile, general-purpose |
| Immutable sequence / dict key | `tuple` | Hashable, lightweight |
| Unique elements / fast lookup | `set` | O(1) membership, set math |
| Key-value mapping | `dict` | O(1) lookup by key |
| Counting | `Counter` | Built for counting |
| Auto-default dict | `defaultdict` | No KeyError on missing keys |
| Struct-like named fields | `namedtuple` / `dataclass` | Readable, typed |
| Fast queue (FIFO) | `deque` | O(1) both ends |

### Big-O quick reference

| Operation | list | dict/set | deque |
|-----------|------|----------|-------|
| Index `[i]` | O(1) | — | O(n) |
| Search `in` | O(n) | **O(1)** | O(n) |
| Append | O(1) | O(1) | O(1) |
| Insert at 0 | O(n) | — | **O(1)** |
| Pop last | O(1) | — | O(1) |
| Pop first | O(n) | — | **O(1)** |
| Delete | O(n) | O(1) | O(n) |

---

## 📝 Practice Tasks

### Task 1: Word Frequency Analyzer
Given a paragraph of text, build a dict of word frequencies. Print the top 10 most common words. Use both a manual approach (dict) and `Counter`. Ignore punctuation and case.

### Task 2: Set Operations Challenge
Given two lists of student enrollments (course A and course B), find students who are: in both courses, only in A, only in B, in either but not both. Use set operations.

### Task 3: Comprehension Marathon
Rewrite each of these using comprehensions:
```python
# a) Filter and transform
result = []
for n in range(1, 51):
    if n % 3 == 0:
        result.append(n ** 2)

# b) Flatten and deduplicate
data = [[1, 2, 3], [2, 3, 4], [3, 4, 5]]

# c) Invert a dict
original = {"a": 1, "b": 2, "c": 3}

# d) Create a matrix of zeros (5x5)

# e) Extract emails from a list of user dicts (only verified users)
users = [
    {"email": "a@b.com", "verified": True},
    {"email": "c@d.com", "verified": False},
    {"email": "e@f.com", "verified": True},
]
```

### Task 4: Nested Data Transformer
Given this data structure, use comprehensions and dict operations to produce a summary:
```python
sales = [
    {"product": "Widget", "region": "North", "amount": 100},
    {"product": "Gadget", "region": "South", "amount": 200},
    {"product": "Widget", "region": "South", "amount": 150},
    {"product": "Gadget", "region": "North", "amount": 300},
    {"product": "Widget", "region": "North", "amount": 120},
]
# Produce: total per product, total per region, product with highest total
```

### Task 5: Zip & Enumerate
Given parallel lists of names, scores, and grades, create a list of dicts and then sort by score descending. Print a formatted leaderboard using enumerate.

### Task 6: Matrix Operations (no NumPy!)
Using only lists and comprehensions, implement:
- Matrix transpose
- Element-wise addition of two matrices
- Matrix multiplication (challenge!)

---

## 📚 Resources

- [Python Official — Data Structures](https://docs.python.org/3/tutorial/datastructures.html)
- [Real Python — Lists and Tuples](https://realpython.com/python-lists-tuples/)
- [Real Python — Dictionaries](https://realpython.com/python-dicts/)
- [Real Python — Sets](https://realpython.com/python-sets/)
- [Real Python — List Comprehensions](https://realpython.com/list-comprehension-python/)
- [Real Python — When to Use a List Comprehension](https://realpython.com/python-list-comprehension/)
- [Real Python — collections Module](https://realpython.com/python-collections-module/)
- [Python TimeComplexity Wiki](https://wiki.python.org/moin/TimeComplexity)

---

## 🔑 Key Takeaways

1. **Lists** are mutable, **tuples** are immutable — use tuples for data that shouldn't change and as dict keys.
2. **Sets** are your best friend for uniqueness and fast lookups — O(1) vs O(n) for lists.
3. **Dicts** preserve insertion order (3.7+) and support powerful merge operators (`|`, `**`).
4. **Comprehensions** are the Pythonic way — prefer them over `map`/`filter` but don't overdo nesting.
5. **Generator expressions** save memory — use `()` instead of `[]` when you only iterate once.
6. **`Counter`**, **`defaultdict`**, **`namedtuple`** from collections are essential — know when to reach for them.
7. **`zip`** and **unpacking** (`*`, `**`) are used everywhere in Python — master them early.

---

> **Tomorrow (Day 4):** OOP in Python — classes, inheritance, dunder methods, dataclasses, abstract base classes. Coming from Java, you'll feel at home but with much less boilerplate.

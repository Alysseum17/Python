# Day 1: Syntax, Variables, Data Types, Strings

## 🎯 Goal

Get comfortable with Python's core syntax. Since you know JS/TS and Java, this will be fast — focus on **how Python differs**, not on what's the same.

---

## 1. Running Python

```bash
# Check version
python3 --version

# Interactive REPL
python3

# Run a file
python3 main.py
```

> **Key difference from JS/Java:** No semicolons, no curly braces — Python uses **indentation** (4 spaces by convention) to define blocks.

---

## 2. Variables & Data Types

Python is **dynamically typed** (like JS) but **strongly typed** (like Java — no implicit coercion).

```python
# No let/const/var, no type declaration needed
name = "Oleksii"        # str
age = 25                 # int
height = 1.82            # float
is_dev = True            # bool (capital T/F, not true/false like JS)
nothing = None           # None (not null, not undefined)

# Multiple assignment
x, y, z = 1, 2, 3
a = b = c = 0

# Type checking
type(name)       # <class 'str'>
isinstance(age, int)  # True
```

### Type comparison cheat sheet

| Concept | JS/TS | Java | Python |
|---------|-------|------|--------|
| Integer | `number` | `int`, `long` | `int` (unlimited size!) |
| Float | `number` | `double`, `float` | `float` |
| String | `string` | `String` | `str` |
| Boolean | `true`/`false` | `true`/`false` | `True`/`False` |
| Null | `null`/`undefined` | `null` | `None` |
| Array | `[]` | `int[]`, `List<>` | `list` |
| Object/Map | `{}` | `HashMap<>` | `dict` |

### Numbers — Python has unlimited integers

```python
# No integer overflow in Python!
big = 10 ** 100  # This just works, no BigInt needed
print(big)       # 10000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000

# Integer division vs float division (IMPORTANT!)
10 / 3     # 3.3333... (always returns float, like JS)
10 // 3    # 3 (floor division — no equivalent in JS)
10 % 3     # 1 (modulo, same as JS/Java)
2 ** 10    # 1024 (power — in JS it's 2 ** 10 too, or Math.pow())

# Underscores for readability
million = 1_000_000  # same as 1000000
```

---

## 3. Strings

Python strings are **immutable** (like Java). Single and double quotes are identical (unlike JS template literals).

```python
# All equivalent
s1 = 'hello'
s2 = "hello"
s3 = '''hello'''  # triple quotes for multiline

# Multiline strings
poem = """
Roses are red,
Violets are blue,
Python is great,
And so are you.
"""
```

### f-strings (Python's template literals)

```python
name = "World"
age = 25

# f-string — like JS template literals but with f prefix
print(f"Hello, {name}!")              # Hello, World!
print(f"Next year I'll be {age + 1}") # Next year I'll be 26
print(f"{name!r}")                    # 'World' (repr, useful for debugging)
print(f"{age:05d}")                   # 00025 (zero-padded)
print(f"{3.14159:.2f}")               # 3.14 (2 decimal places)

# Comparison with JS:
# JS:     `Hello, ${name}!`
# Python: f"Hello, {name}!"
```

### Common string methods

```python
s = "Hello, World!"

# Familiar from JS
s.upper()            # "HELLO, WORLD!"
s.lower()            # "hello, world!"
s.strip()            # Trims whitespace (like JS .trim())
s.replace("World", "Python")  # "Hello, Python!"
s.split(", ")        # ["Hello", "World!"]
s.startswith("Hello") # True
s.endswith("!")       # True
s.find("World")       # 7 (index, -1 if not found)
s.index("World")      # 7 (raises ValueError if not found)
s.count("l")          # 3

# Python-specific (very useful!)
s.title()            # "Hello, World!"
s.capitalize()       # "Hello, world!"
s.isdigit()          # False
s.isalpha()          # False (has comma and space)
"  ".isspace()       # True

# Join — opposite of split (like JS .join() but on separator)
# JS:     ["a", "b", "c"].join("-")
# Python: "-".join(["a", "b", "c"])  →  "a-b-c"
words = ["Python", "is", "awesome"]
" ".join(words)      # "Python is awesome"
```

### String slicing (no .substring() needed!)

```python
s = "Hello, World!"

# Slicing: s[start:end:step]
s[0]       # "H"
s[-1]      # "!" (negative indexing — no s[s.length - 1] needed)
s[0:5]     # "Hello"
s[:5]      # "Hello" (start defaults to 0)
s[7:]      # "World!" (end defaults to len)
s[-6:]     # "orld!"
s[::2]     # "Hlo ol!" (every 2nd char)
s[::-1]    # "!dlroW ,olleH" (reverse string!)

# This works on lists too (very Pythonic)
```

### String formatting alternatives (know them, prefer f-strings)

```python
# f-strings (modern, preferred) ✅
f"Hello, {name}!"

# .format() (older)
"Hello, {}!".format(name)
"Hello, {0}! You are {1}.".format(name, age)

# % formatting (legacy, you'll see it in old code)
"Hello, %s! You are %d." % (name, age)
```

---

## 4. Conditionals

```python
# No parentheses required, colon instead of curly braces
age = 20

if age >= 18:
    print("Adult")
elif age >= 13:
    print("Teenager")
else:
    print("Child")

# Ternary (like JS ternary but reads like English)
# JS:     const status = age >= 18 ? "adult" : "minor"
# Python:
status = "adult" if age >= 18 else "minor"

# Truthiness — similar to JS but with key differences
# Falsy: False, None, 0, 0.0, "", [], {}, set()
# Everything else is truthy

# Identity vs equality
x = [1, 2, 3]
y = [1, 2, 3]
x == y    # True (value equality, like Java .equals())
x is y    # False (identity/reference, like Java ==)
x is None # Always use 'is' for None checks (not ==)

# Chained comparisons (Python-unique, very clean!)
x = 5
1 < x < 10      # True (no need for: 1 < x && x < 10)
1 < x < 3       # False

# Logical operators: and, or, not (not &&, ||, !)
if age >= 18 and is_dev:
    print("Adult developer")

if not is_dev:
    print("Not a dev")
```

### match-case (Python 3.10+ — like switch but more powerful)

```python
status = 404

match status:
    case 200:
        print("OK")
    case 404:
        print("Not Found")
    case 500:
        print("Server Error")
    case _:
        print("Unknown")  # _ is the default/wildcard
```

---

## 5. Loops

```python
# for loop — iterates over iterables (like JS for...of)
for i in range(5):        # 0, 1, 2, 3, 4
    print(i)

for i in range(2, 10, 2): # 2, 4, 6, 8 (start, stop, step)
    print(i)

# Iterating over a list
fruits = ["apple", "banana", "cherry"]
for fruit in fruits:
    print(fruit)

# With index — enumerate (no need for manual counter)
# JS:  fruits.forEach((fruit, i) => ...)
for i, fruit in enumerate(fruits):
    print(f"{i}: {fruit}")

# While loop
count = 0
while count < 5:
    print(count)
    count += 1   # No ++ in Python!

# Loop control
for i in range(10):
    if i == 3:
        continue   # skip
    if i == 7:
        break      # stop
    print(i)

# for-else (Python-unique! else runs if loop wasn't broken)
for n in range(2, 10):
    for x in range(2, n):
        if n % x == 0:
            break
    else:
        # This runs if the inner loop completed WITHOUT break
        print(f"{n} is prime")
```

---

## 6. Input / Output

```python
# Print
print("Hello")                    # Hello
print("a", "b", "c")             # a b c (space-separated by default)
print("a", "b", sep="-")         # a-b
print("no newline", end="")      # no trailing newline

# Input (always returns string)
name = input("Enter your name: ")
age = int(input("Enter your age: "))  # must cast manually
```

---

## 7. Key Differences Cheat Sheet

| Feature | JS/TS | Java | Python |
|---------|-------|------|--------|
| Blocks | `{ }` | `{ }` | Indentation (4 spaces) |
| Comments | `//` `/* */` | `//` `/* */` | `#` only |
| Constants | `const` | `final` | Convention: `UPPER_CASE` (no enforcement) |
| Print | `console.log()` | `System.out.println()` | `print()` |
| String concat | `` `${x}` `` | `"" + x` | `f"{x}"` |
| Null check | `=== null` | `== null` | `is None` |
| Logical ops | `&&` `\|\|` `!` | `&&` `\|\|` `!` | `and` `or` `not` |
| Increment | `x++` | `x++` | `x += 1` (no `++`) |
| Floor division | `Math.floor(a/b)` | `a / b` (int) | `a // b` |
| Power | `**` or `Math.pow()` | `Math.pow()` | `**` |
| Range loop | `for(let i=0;i<n;i++)` | `for(int i=0;i<n;i++)` | `for i in range(n):` |

---

## 📝 Practice Tasks

Complete these to solidify Day 1. They should take about 1–2 hours total.

### Task 1: Variable Playground
Create variables of each type (int, float, str, bool, None). Print their types and play with type conversion: `int()`, `float()`, `str()`, `bool()`.

### Task 2: String Manipulator
Write a program that takes a user's full name and:
- Prints it in uppercase, lowercase, and title case
- Prints the number of characters (excluding spaces)
- Prints the name reversed
- Prints each word on a separate line with its index

### Task 3: FizzBuzz (the classic, Python style)
Print numbers 1–100. For multiples of 3 print "Fizz", for 5 print "Buzz", for both print "FizzBuzz". Use a ternary or match-case for a Pythonic solution.

### Task 4: Number Guessing Game
Generate a random number (`from random import randint`), let the user guess with hints ("higher"/"lower"), count attempts.

### Task 5: Temperature Converter
Build a converter that handles Celsius ↔ Fahrenheit ↔ Kelvin. Use `match-case` for menu selection.

---

## 📚 Resources

- [Python Official Tutorial — Informal Introduction](https://docs.python.org/3/tutorial/introduction.html)
- [Python Official Tutorial — Control Flow](https://docs.python.org/3/tutorial/controlflow.html)
- [Real Python — Python Basics](https://realpython.com/python-basics/)
- [Real Python — f-strings Guide](https://realpython.com/python-f-strings/)
- [Python String Methods Reference](https://docs.python.org/3/library/stdtypes.html#string-methods)

---

> **Tomorrow (Day 2):** Functions, Lambdas, Scope & Closures — where Python starts to feel really different from Java.

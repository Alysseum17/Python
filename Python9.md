# Day 9: Regular Expressions & Text Processing

## 🎯 Goal

Master Python's `re` module for pattern matching, parsing, and cleaning messy data. Regex is one of those skills that's useful across every language — and as a Data Engineer, you'll use it constantly for parsing logs, validating formats, extracting fields from unstructured text, and cleaning dirty data.

You already know regex basics from JS — Python's flavor is nearly identical (both use PCRE-style). Focus on Python-specific API differences and DE-relevant patterns.

---

## 1. `re` Module Basics

### Key functions overview

```python
import re

text = "My phone is 123-456-7890 and email is alice@example.com"

# re.search()  — find FIRST match anywhere in string
# re.match()   — match only at the BEGINNING of string
# re.fullmatch() — match the ENTIRE string
# re.findall() — find ALL matches, return list of strings
# re.finditer() — find ALL matches, return iterator of Match objects
# re.sub()     — search and replace
# re.split()   — split by pattern
# re.compile() — pre-compile pattern for reuse
```

### `re.search` vs `re.match` vs `re.fullmatch`

```python
import re

text = "Error 404: Page not found"

# search — finds pattern ANYWHERE in string (most common)
result = re.search(r"\d+", text)
print(result.group())  # "404"

# match — only matches at the BEGINNING
result = re.match(r"\d+", text)
print(result)  # None! ("Error" is at the beginning, not digits)

result = re.match(r"Error", text)
print(result.group())  # "Error"

# fullmatch — entire string must match
result = re.fullmatch(r"\d+", "404")
print(result.group())  # "404"

result = re.fullmatch(r"\d+", "Error 404")
print(result)  # None! (entire string isn't digits)

# JS comparison:
# re.search()  ≈ string.match(/pattern/)
# re.match()   ≈ string.match(/^pattern/)
# re.fullmatch() ≈ string.match(/^pattern$/)
```

### Match objects

```python
import re

text = "Date: 2025-02-14, Time: 12:30:00"
match = re.search(r"(\d{4})-(\d{2})-(\d{2})", text)

if match:  # ALWAYS check — search returns None if no match
    match.group()    # "2025-02-14" (entire match)
    match.group(0)   # "2025-02-14" (same as above)
    match.group(1)   # "2025" (first capture group)
    match.group(2)   # "02"
    match.group(3)   # "14"
    match.groups()   # ("2025", "02", "14") — all groups as tuple
    match.start()    # 6 (start index in original string)
    match.end()      # 16 (end index)
    match.span()     # (6, 16) (start, end)

# Named groups (much more readable!)
match = re.search(r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})", text)
if match:
    match.group("year")    # "2025"
    match.group("month")   # "02"
    match.group("day")     # "14"
    match.groupdict()      # {"year": "2025", "month": "02", "day": "14"}
```

---

## 2. Pattern Syntax Quick Reference

You likely know most of this from JS. Here's a focused refresher:

### Characters & anchors

```python
# Literal characters
r"hello"          # matches "hello" exactly

# Metacharacters
.                 # any char except newline
\d                # digit [0-9]
\D                # non-digit [^0-9]
\w                # word char [a-zA-Z0-9_]
\W                # non-word
\s                # whitespace [ \t\n\r\f\v]
\S                # non-whitespace
\b                # word boundary

# Anchors
^                 # start of string (or line with re.MULTILINE)
$                 # end of string (or line with re.MULTILINE)
\A                # absolute start of string (ignores MULTILINE)
\Z                # absolute end of string (ignores MULTILINE)
```

### Quantifiers

```python
*                 # 0 or more (greedy)
+                 # 1 or more (greedy)
?                 # 0 or 1 (greedy)
{n}               # exactly n
{n,}              # n or more
{n,m}             # between n and m

*?                # 0 or more (lazy/non-greedy)
+?                # 1 or more (lazy)
??                # 0 or 1 (lazy)
```

### Groups & alternation

```python
(abc)             # capture group
(?:abc)           # non-capturing group
(?P<name>abc)     # named capture group
a|b               # alternation (a OR b)
```

### Character classes

```python
[abc]             # a, b, or c
[a-z]             # lowercase letter
[A-Za-z]          # any letter
[0-9]             # digit (same as \d)
[^abc]            # NOT a, b, or c
[\w\s]            # word char or whitespace
```

### ⚠️ Always use raw strings!

```python
# In Python, ALWAYS prefix regex with r"..." (raw string)
# Otherwise backslashes get interpreted by Python before regex engine sees them

# ❌ Without raw string
re.search("\d+", text)    # Works by accident, but fragile
re.search("\bword\b", text)  # ❌ BROKEN! \b is backspace in Python strings

# ✅ With raw string
re.search(r"\d+", text)      # Correct
re.search(r"\bword\b", text) # Correct — \b is word boundary

# JS doesn't have this issue because /\d+/ syntax doesn't process escapes
```

---

## 3. Core `re` Functions in Depth

### `re.findall` — find all matches

```python
import re

text = "Prices: $10.50, $23.99, $5.00, and $100"

# Without groups — returns list of strings
prices = re.findall(r"\$[\d.]+", text)
# ["$10.50", "$23.99", "$5.00", "$100"]

# With ONE group — returns list of group contents
prices = re.findall(r"\$([\d.]+)", text)
# ["10.50", "23.99", "5.00", "100"]

# With MULTIPLE groups — returns list of tuples
matches = re.findall(r"(\d+)\.(\d+)", text)
# [("10", "50"), ("23", "99"), ("5", "00")]

# ⚠️ findall with groups is a common gotcha!
# If you have groups but want the full match, use (?:...) non-capturing groups
# or use finditer instead
```

### `re.finditer` — match objects for all matches

```python
import re

text = "Error at 10:30, Warning at 11:45, Error at 14:20"

# Returns iterator of Match objects — more info than findall
for match in re.finditer(r"(\w+) at (\d{2}:\d{2})", text):
    level = match.group(1)
    time = match.group(2)
    pos = match.start()
    print(f"[{pos}] {level} at {time}")

# [0] Error at 10:30
# [16] Warning at 11:45
# [34] Error at 14:20

# Use finditer when you need:
# - Position information (start/end)
# - Named groups
# - Multiple groups without tuple confusion
```

### `re.sub` — search and replace

```python
import re

# Basic replacement
text = "Hello World 123"
result = re.sub(r"\d+", "###", text)
# "Hello World ###"

# Replace with backreference
text = "John Smith, Jane Doe"
result = re.sub(r"(\w+) (\w+)", r"\2, \1", text)
# "Smith, John, Doe, Jane"

# Named backreference
result = re.sub(r"(?P<first>\w+) (?P<last>\w+)", r"\g<last>, \g<first>", text)

# Replace with function — most powerful!
def censor_email(match):
    """Replace email with censored version."""
    email = match.group()
    username, domain = email.split("@")
    return f"{username[0]}***@{domain}"

text = "Contact alice@example.com or bob@company.org"
result = re.sub(r"[\w.]+@[\w.]+", censor_email, text)
# "Contact a***@example.com or b***@company.org"

# Replace with count limit
text = "aaa bbb ccc ddd"
result = re.sub(r"\w+", "X", text, count=2)
# "X X ccc ddd"
```

### `re.split` — split by pattern

```python
import re

# Split by one or more whitespace
text = "hello   world\tfoo  bar"
re.split(r"\s+", text)
# ["hello", "world", "foo", "bar"]

# Split by multiple delimiters
text = "apple,banana;cherry|date"
re.split(r"[,;|]", text)
# ["apple", "banana", "cherry", "date"]

# Keep the delimiters (use capture group)
text = "one, two; three | four"
re.split(r"([\s,;|]+)", text)
# ["one", ", ", "two", "; ", "three", " | ", "four"]

# Limit splits
re.split(r"\s+", "a b c d e", maxsplit=2)
# ["a", "b", "c d e"]
```

### `re.compile` — pre-compile patterns

```python
import re

# Compile once, use many times — faster for repeated use
EMAIL_PATTERN = re.compile(r"[\w.+-]+@[\w-]+\.[\w.]+")
PHONE_PATTERN = re.compile(r"\+?\d{1,3}[-.\s]?\(?\d{1,4}\)?[-.\s]?\d{1,4}[-.\s]?\d{1,9}")

# Use compiled pattern's methods
emails = EMAIL_PATTERN.findall(text)
match = EMAIL_PATTERN.search(text)
cleaned = EMAIL_PATTERN.sub("[REDACTED]", text)

# When to compile:
# ✅ Pattern used in a loop or called repeatedly
# ✅ Pattern stored as a module-level constant
# ❌ One-off usage in a script — no need to compile

# Compiled patterns are also more readable as named constants
DATE_ISO = re.compile(r"\d{4}-\d{2}-\d{2}")
IP_ADDRESS = re.compile(r"\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b")
```

---

## 4. Regex Flags

```python
import re

# re.IGNORECASE (re.I) — case-insensitive
re.findall(r"error", "Error ERROR error", re.IGNORECASE)
# ["Error", "ERROR", "error"]

# re.MULTILINE (re.M) — ^ and $ match line start/end
text = """Line 1
Line 2
Line 3"""
re.findall(r"^Line \d+", text, re.MULTILINE)
# ["Line 1", "Line 2", "Line 3"]
# Without MULTILINE: ["Line 1"] (^ only matches start of entire string)

# re.DOTALL (re.S) — dot matches newlines too
text = "<div>\n  Hello\n</div>"
re.search(r"<div>.*</div>", text)           # None (. doesn't match \n)
re.search(r"<div>.*</div>", text, re.DOTALL) # matches!

# re.VERBOSE (re.X) — allow comments and whitespace in pattern
phone_pattern = re.compile(r"""
    (\+?\d{1,3})?      # optional country code
    [-.\s]?             # optional separator
    \(?(\d{3})\)?       # area code (with optional parens)
    [-.\s]?             # optional separator
    (\d{3})             # first 3 digits
    [-.\s]?             # optional separator
    (\d{4})             # last 4 digits
""", re.VERBOSE)
# MUCH more readable than: r"(\+?\d{1,3})?[-.\s]?\(?(\d{3})\)?[-.\s]?(\d{3})[-.\s]?(\d{4})"

# Combine flags with |
re.findall(r"^error.*", text, re.IGNORECASE | re.MULTILINE)
```

---

## 5. Lookahead & Lookbehind

Assertions that match a position without consuming characters.

```python
import re

# Lookahead: (?=...) — what follows
# "Find digits followed by 'px'"
re.findall(r"\d+(?=px)", "12px 30em 45px 100%")
# ["12", "45"]

# Negative lookahead: (?!...) — what doesn't follow
# "Find digits NOT followed by 'px'"
re.findall(r"\d+(?!px)", "12px 30em 45px 100%")
# ["1", "30", "4", "100"]
# ⚠️ Tricky! "12" → "1" matches (followed by "2", not "px")

# Fix with word boundary:
re.findall(r"\b\d+(?!px)\b", "12px 30em 45px 100%")
# ["30", "100"]

# Lookbehind: (?<=...) — what precedes
# "Find amount after $"
re.findall(r"(?<=\$)\d+\.?\d*", "Price: $49.99, Tax: $5.00")
# ["49.99", "5.00"]

# Negative lookbehind: (?<!...) — what doesn't precede
# "Find numbers not preceded by $"
re.findall(r"(?<!\$)\b\d+\.?\d*", "Price: $49.99, Count: 42, Tax: $5.00")
# ["42"]

# ⚠️ Python lookbehind must be FIXED LENGTH
# (?<=abc)     ✅ 
# (?<=a|bb)    ✅ (alternatives of same length OK in Python 3.6+)
# (?<=a*)      ❌ Variable-length lookbehind not supported
```

---

## 6. Common Patterns for Data Engineering

### Email extraction

```python
import re

EMAIL_PATTERN = re.compile(r"""
    [a-zA-Z0-9._%+-]+    # username
    @                     # @ symbol
    [a-zA-Z0-9.-]+       # domain
    \.[a-zA-Z]{2,}       # TLD
""", re.VERBOSE)

text = "Contact us at support@company.com or sales@company.co.uk"
emails = EMAIL_PATTERN.findall(text)
# ["support@company.com", "sales@company.co.uk"]
```

### URL extraction

```python
URL_PATTERN = re.compile(r"""
    https?://             # protocol
    [\w.-]+               # domain
    (?:\.[a-zA-Z]{2,})+  # TLD
    (?:/[^\s]*)?          # path (optional)
""", re.VERBOSE)

text = "Visit https://example.com/page?q=1 or http://test.org"
urls = URL_PATTERN.findall(text)
```

### IP address

```python
IP_PATTERN = re.compile(r"""
    \b
    (?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}  # first 3 octets
    (?:25[0-5]|2[0-4]\d|[01]?\d\d?)            # last octet
    \b
""", re.VERBOSE)

text = "Server 192.168.1.1 responded, 10.0.0.255 timed out, 999.1.1.1 invalid"
ips = IP_PATTERN.findall(text)
# ["192.168.1.1", "10.0.0.255"] — 999.1.1.1 correctly excluded
```

### Date parsing (multiple formats)

```python
import re

DATE_PATTERNS = {
    "iso": re.compile(r"\b(\d{4})-(\d{2})-(\d{2})\b"),
    "us": re.compile(r"\b(\d{1,2})/(\d{1,2})/(\d{4})\b"),
    "eu": re.compile(r"\b(\d{1,2})\.(\d{1,2})\.(\d{4})\b"),
    "written": re.compile(r"\b(\w+)\s+(\d{1,2}),?\s+(\d{4})\b"),
}

def extract_dates(text: str) -> list[dict]:
    """Extract dates in various formats."""
    results = []
    for fmt, pattern in DATE_PATTERNS.items():
        for match in pattern.finditer(text):
            results.append({
                "format": fmt,
                "raw": match.group(),
                "groups": match.groups(),
                "position": match.start(),
            })
    return results

text = "Events: 2025-02-14, 02/14/2025, 14.02.2025, February 14, 2025"
for date in extract_dates(text):
    print(f"[{date['format']}] {date['raw']}")
```

### Log parsing

```python
import re

# Common log format: "2025-02-14 12:30:45 [ERROR] service.auth: Login failed for user=alice ip=10.0.0.1"
LOG_PATTERN = re.compile(r"""
    (?P<timestamp>\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})  # timestamp
    \s+
    \[(?P<level>\w+)\]                                      # log level
    \s+
    (?P<service>[\w.]+):                                    # service name
    \s+
    (?P<message>.+)                                         # message
""", re.VERBOSE)

log_line = "2025-02-14 12:30:45 [ERROR] service.auth: Login failed for user=alice ip=10.0.0.1"

match = LOG_PATTERN.search(log_line)
if match:
    data = match.groupdict()
    print(data)
    # {
    #   "timestamp": "2025-02-14 12:30:45",
    #   "level": "ERROR",
    #   "service": "service.auth",
    #   "message": "Login failed for user=alice ip=10.0.0.1"
    # }

# Extract key=value pairs from message
KV_PATTERN = re.compile(r"(\w+)=([\w.@]+)")

message = data["message"]
params = dict(KV_PATTERN.findall(message))
# {"user": "alice", "ip": "10.0.0.1"}
```

### Data cleaning patterns

```python
import re

def clean_text(text: str) -> str:
    """Clean messy text data."""
    # Remove HTML tags
    text = re.sub(r"<[^>]+>", "", text)
    
    # Remove URLs
    text = re.sub(r"https?://\S+", "", text)
    
    # Normalize whitespace (multiple spaces/tabs → single space)
    text = re.sub(r"\s+", " ", text)
    
    # Remove special characters but keep basic punctuation
    text = re.sub(r"[^\w\s.,!?;:'-]", "", text)
    
    return text.strip()

def clean_phone(phone: str) -> str:
    """Normalize phone number format."""
    digits = re.sub(r"\D", "", phone)  # remove all non-digits
    if len(digits) == 10:
        return f"({digits[:3]}) {digits[3:6]}-{digits[6:]}"
    if len(digits) == 11 and digits[0] == "1":
        return f"+1 ({digits[1:4]}) {digits[4:7]}-{digits[7:]}"
    return phone  # return original if format unknown

# Test
clean_phone("123-456-7890")    # "(123) 456-7890"
clean_phone("1.234.567.8901")  # "+1 (234) 567-8901"
clean_phone("+1 (234) 567-8901")  # "+1 (234) 567-8901"

def normalize_whitespace(text: str) -> str:
    """Normalize all whitespace variations."""
    text = text.replace("\xa0", " ")  # non-breaking space
    text = text.replace("\t", " ")     # tabs
    text = re.sub(r" {2,}", " ", text) # multiple spaces
    return text.strip()

def extract_numbers(text: str) -> list[float]:
    """Extract all numbers (including decimals and negatives)."""
    pattern = r"-?\d+\.?\d*"
    return [float(x) for x in re.findall(pattern, text)]

extract_numbers("Temp: -5.3°C, Pressure: 1013.25 hPa, Change: +2.1")
# [-5.3, 1013.25, 2.1]
```

---

## 7. String Methods vs Regex

Don't always reach for regex — string methods are faster and more readable for simple tasks.

```python
text = "Hello, World!"

# ✅ Use string methods when possible
text.startswith("Hello")          # not re.match(r"^Hello", text)
text.endswith("!")                # not re.search(r"!$", text)
text.replace("World", "Python")  # not re.sub(r"World", "Python", text)
"World" in text                  # not re.search(r"World", text)
text.strip()                     # not re.sub(r"^\s+|\s+$", "", text)
text.split(",")                  # not re.split(r",", text)
text.lower()                     # not re.sub(r"[A-Z]", lambda m: m.group().lower(), text)
text.isdigit()                   # not re.fullmatch(r"\d+", text)

# ✅ Use regex when you need:
# - Patterns (not fixed strings)
# - Multiple delimiters
# - Capture groups
# - Validation of complex formats
# - Extraction from unstructured text
```

---

## 8. Advanced Patterns

### Greedy vs lazy matching

```python
import re

html = "<div>Hello</div><div>World</div>"

# Greedy (default) — matches as MUCH as possible
re.findall(r"<div>.*</div>", html)
# ["<div>Hello</div><div>World</div>"] — one match, ate everything!

# Lazy (add ?) — matches as LITTLE as possible
re.findall(r"<div>.*?</div>", html)
# ["<div>Hello</div>", "<div>World</div>"] — two separate matches ✅
```

### Non-capturing groups

```python
import re

# (?:...) — group without capturing
text = "2025-02-14"

# With capturing groups — findall returns group contents
re.findall(r"(\d{4})-(\d{2})-(\d{2})", text)
# [("2025", "02", "14")]

# With non-capturing — findall returns full match
re.findall(r"(?:\d{4})-(?:\d{2})-(?:\d{2})", text)
# ["2025-02-14"]
```

### Conditional patterns

```python
import re

# Match with optional parts
# Phone: optionally starts with +, optional area code in parens
phone_re = re.compile(r"""
    (?:\+\d{1,3}\s?)?       # optional country code
    (?:\(\d{3}\)\s?|\d{3}[-.\s])  # area code: (123) or 123-
    \d{3}[-.\s]?             # first 3 digits
    \d{4}                    # last 4 digits
""", re.VERBOSE)

phones = [
    "+1 (234) 567-8901",
    "(234) 567-8901",
    "234-567-8901",
    "234.567.8901",
]

for phone in phones:
    match = phone_re.search(phone)
    print(f"{phone:25s} → {'✅' if match else '❌'}")
```

### Backreferences

```python
import re

# \1 refers back to first capture group
# Find repeated words
text = "the the quick brown fox fox"
re.findall(r"\b(\w+)\s+\1\b", text)
# ["the", "fox"]

# Find matching HTML tags
html = "<b>bold</b> <i>italic</i> <b>also bold</b>"
re.findall(r"<(\w+)>.*?</\1>", html)
# ["b", "i", "b"]
```

---

## 9. Text Processing Beyond Regex

### `str.translate` — character-level replacement (fastest)

```python
# Remove punctuation
import string

text = "Hello, World! How's it going?"
translator = str.maketrans("", "", string.punctuation)
clean = text.translate(translator)
# "Hello World Hows it going"

# Replace specific characters
translator = str.maketrans("àáâãäå", "aaaaaa")
"café résumé".translate(translator)
# "café résumé" → handles accented chars
```

### `textwrap` — format text blocks

```python
import textwrap

long_text = "Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed do eiusmod tempor incididunt ut labore et dolore magna aliqua."

# Wrap to width
print(textwrap.fill(long_text, width=40))
# Lorem ipsum dolor sit amet,
# consectetur adipiscing elit. Sed do
# eiusmod tempor incididunt ut labore
# et dolore magna aliqua.

# Dedent (remove common leading whitespace)
code = """
    def hello():
        print("hi")
"""
print(textwrap.dedent(code))
# def hello():
#     print("hi")

# Shorten
textwrap.shorten(long_text, width=50, placeholder="...")
# "Lorem ipsum dolor sit amet, consectetur..."
```

### `unicodedata` — handle Unicode properly

```python
import unicodedata

# Normalize Unicode (important for comparing text from different sources!)
s1 = "café"           # "é" as single character (NFC)
s2 = "cafe\u0301"     # "e" + combining accent (NFD)
print(s1 == s2)        # False! (different byte sequences)

s1_norm = unicodedata.normalize("NFC", s1)
s2_norm = unicodedata.normalize("NFC", s2)
print(s1_norm == s2_norm)  # True!

# Remove accents
def remove_accents(text: str) -> str:
    """Remove diacritics/accents from text."""
    nfkd = unicodedata.normalize("NFKD", text)
    return "".join(c for c in nfkd if not unicodedata.category(c).startswith("M"))

remove_accents("café résumé naïve")
# "cafe resume naive"

# Get character info
unicodedata.name("€")      # "EURO SIGN"
unicodedata.category("A")  # "Lu" (Letter, uppercase)
unicodedata.category("1")  # "Nd" (Number, decimal)
unicodedata.category(" ")  # "Zs" (Separator, space)
```

---

## 10. Building a Text Parser — Real Example

```python
import re
from dataclasses import dataclass, field
from datetime import datetime
from pathlib import Path

@dataclass
class LogEntry:
    timestamp: datetime
    level: str
    service: str
    message: str
    metadata: dict = field(default_factory=dict)

class LogParser:
    """Parse structured log files into LogEntry objects."""
    
    # Compile patterns once
    LOG_LINE = re.compile(r"""
        (?P<timestamp>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?)
        \s+
        (?P<level>DEBUG|INFO|WARNING|ERROR|CRITICAL)
        \s+
        \[(?P<service>[^\]]+)\]
        \s+
        (?P<message>.+)
    """, re.VERBOSE)
    
    KV_PAIR = re.compile(r"(?P<key>\w+)=(?P<value>[^\s,]+)")
    
    QUOTED_VALUE = re.compile(r'(?P<key>\w+)="(?P<value>[^"]*)"')
    
    def parse_line(self, line: str) -> LogEntry | None:
        """Parse a single log line."""
        match = self.LOG_LINE.match(line.strip())
        if not match:
            return None
        
        data = match.groupdict()
        
        # Extract key=value pairs from message
        metadata = {}
        for kv_match in self.QUOTED_VALUE.finditer(data["message"]):
            metadata[kv_match.group("key")] = kv_match.group("value")
        for kv_match in self.KV_PAIR.finditer(data["message"]):
            key = kv_match.group("key")
            if key not in metadata:  # don't override quoted values
                metadata[key] = kv_match.group("value")
        
        return LogEntry(
            timestamp=datetime.fromisoformat(data["timestamp"]),
            level=data["level"],
            service=data["service"],
            message=data["message"],
            metadata=metadata,
        )
    
    def parse_file(self, filepath: Path):
        """Parse an entire log file, yielding LogEntry objects."""
        with open(filepath, "r", encoding="utf-8") as f:
            for line_num, line in enumerate(f, 1):
                if not line.strip():
                    continue
                entry = self.parse_line(line)
                if entry:
                    yield entry
                else:
                    print(f"Warning: unparseable line {line_num}: {line.strip()[:80]}")
    
    def analyze(self, filepath: Path) -> dict:
        """Analyze a log file and return statistics."""
        from collections import Counter
        
        stats = {
            "total": 0,
            "by_level": Counter(),
            "by_service": Counter(),
            "errors": [],
        }
        
        for entry in self.parse_file(filepath):
            stats["total"] += 1
            stats["by_level"][entry.level] += 1
            stats["by_service"][entry.service] += 1
            
            if entry.level in ("ERROR", "CRITICAL"):
                stats["errors"].append({
                    "timestamp": entry.timestamp.isoformat(),
                    "service": entry.service,
                    "message": entry.message[:200],
                })
        
        return stats

# Usage
parser = LogParser()

# Parse single line
entry = parser.parse_line(
    '2025-02-14T12:30:45.123 ERROR [auth-service] Login failed user=alice ip=10.0.0.1 reason="invalid password"'
)
if entry:
    print(entry.level)      # ERROR
    print(entry.service)    # auth-service
    print(entry.metadata)   # {"user": "alice", "ip": "10.0.0.1", "reason": "invalid password"}
```

---

## 11. Performance Tips

```python
import re

# 1. Compile patterns used in loops
pattern = re.compile(r"\d+")  # compile ONCE

# ❌ Slow — re-compiles every iteration
for line in million_lines:
    re.findall(r"\d+", line)

# ✅ Fast — compiled once
for line in million_lines:
    pattern.findall(line)

# 2. Use string methods for simple checks (10-100x faster!)
import timeit

text = "ERROR: something went wrong"

# String method — fast
timeit.timeit(lambda: text.startswith("ERROR"), number=1_000_000)  # ~0.08s

# Regex — slower
pattern = re.compile(r"^ERROR")
timeit.timeit(lambda: pattern.match(text), number=1_000_000)       # ~0.25s

# 3. Use non-capturing groups when you don't need captures
# (?:...) is slightly faster than (...)

# 4. Be specific — avoid .* when possible
# r"<div>.*?</div>"  — backtracking can be expensive
# r"<div>[^<]*</div>" — much faster (no backtracking)

# 5. Use atomic groups / possessive quantifiers (Python 3.11+)
# Not yet available in Python re module, but available in 'regex' package
# pip install regex
```

---

## 📝 Practice Tasks

### Task 1: Data Validator
Write regex-based validators for: email, phone number (US and UA formats), IPv4 address, URL, credit card number (with format detection: Visa, Mastercard), ISO date (YYYY-MM-DD). Package them in a `Validator` class.

### Task 2: Log Parser
Parse an Apache/Nginx combined log format:
```
127.0.0.1 - frank [10/Oct/2025:13:55:36 -0700] "GET /api/users HTTP/1.1" 200 2326
```
Extract: IP, user, timestamp, method, path, protocol, status code, response size. Calculate statistics: requests per hour, top 10 paths, error rate by status code.

### Task 3: CSV Data Cleaner with Regex
Write a function that takes a messy CSV string/file and:
- Normalizes phone numbers to a standard format
- Validates and normalizes email addresses
- Extracts and standardizes dates from various formats
- Removes HTML tags from text fields
- Flags rows with validation errors

### Task 4: Template Engine
Build a simple template engine that supports:
```python
template = "Hello, {{name}}! You have {{count}} messages. {{#if premium}}Premium user{{/if}}"
render(template, {"name": "Alice", "count": 5, "premium": True})
# "Hello, Alice! You have 5 messages. Premium user"
```
Use regex to find and replace template variables and blocks.

### Task 5: Markdown to HTML Converter (Simplified)
Convert basic Markdown to HTML using regex:
- `# Heading` → `<h1>Heading</h1>` (support h1-h6)
- `**bold**` → `<strong>bold</strong>`
- `*italic*` → `<em>italic</em>`
- `[text](url)` → `<a href="url">text</a>`
- `` `code` `` → `<code>code</code>`
- Blank line → paragraph break

### Task 6: Structured Data Extractor
Given unstructured text like receipts or invoices, extract:
- Dates (multiple formats)
- Currency amounts
- Names/identifiers
- Addresses
Output as structured JSON.

---

## 📚 Resources

- [Python Official — re module](https://docs.python.org/3/library/re.html)
- [Python Official — HOWTO Regex](https://docs.python.org/3/howto/regex.html)
- [Real Python — Regular Expressions](https://realpython.com/regex-python/)
- [regex101.com](https://regex101.com/) — interactive regex tester (select Python flavor!)
- [Regular Expressions Info](https://www.regular-expressions.info/) — comprehensive reference
- [Python Official — string module](https://docs.python.org/3/library/string.html)
- [Python Official — textwrap](https://docs.python.org/3/library/textwrap.html)
- [Python Official — unicodedata](https://docs.python.org/3/library/unicodedata.html)

---

## 🔑 Key Takeaways

1. **Always use raw strings** — `r"pattern"` prevents Python from eating your backslashes.
2. **`re.compile`** for patterns used more than once — faster and more readable as constants.
3. **Named groups** `(?P<name>...)` — always prefer over numbered groups for readability.
4. **`finditer` over `findall`** when you need positions or complex group access.
5. **`re.sub` with function** — the most powerful replacement pattern for data transformation.
6. **`re.VERBOSE`** — always use for complex patterns, add comments.
7. **String methods first** — don't use regex for simple `startswith`, `in`, `split`, `replace`.
8. **Greedy vs lazy** — `.*` is greedy (matches max), `.*?` is lazy (matches min). Know the difference.
9. **Unicode normalize** — always `unicodedata.normalize("NFC", text)` before comparing text from different sources.
10. **`regex101.com`** — test your patterns interactively, always.

---

> **Tomorrow (Day 10):** Virtual Environments, Package Management & Project Structure — venv, pip, Poetry, uv, pyproject.toml, project layout, ruff, black.

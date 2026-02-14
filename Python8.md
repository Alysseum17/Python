# Day 8: File Handling & Serialization

## 🎯 Goal

Learn to read, write, and process files in Python — the bread and butter of Data Engineering. Today covers text files, CSV, JSON, YAML, binary files, `pathlib`, and patterns for handling large files efficiently. You'll use these skills every single day in data pipelines.

---

## 1. Text Files — The Basics

### Reading files

```python
# The standard way — always use 'with' (context manager from Day 7)
with open("data.txt", "r") as f:
    content = f.read()       # entire file as one string

with open("data.txt", "r") as f:
    lines = f.readlines()    # list of lines (includes \n)

with open("data.txt", "r") as f:
    lines = f.read().splitlines()  # list of lines (no \n) — often preferred

# Line by line — memory efficient for large files!
with open("data.txt", "r") as f:
    for line in f:           # f is an iterator — reads one line at a time
        line = line.strip()  # remove \n and whitespace
        process(line)

# Read first N lines
with open("data.txt", "r") as f:
    first_10 = [next(f).strip() for _ in range(10)]
```

### Writing files

```python
# Write (creates new file or overwrites)
with open("output.txt", "w") as f:
    f.write("Hello, World!\n")
    f.write("Second line\n")

# Write multiple lines
lines = ["line 1", "line 2", "line 3"]
with open("output.txt", "w") as f:
    f.writelines(line + "\n" for line in lines)
    # writelines does NOT add newlines — you must add them!

# Append (add to existing file)
with open("log.txt", "a") as f:
    f.write("New log entry\n")

# Print to file
with open("output.txt", "w") as f:
    print("Hello!", file=f)           # print() writes to file
    print("With formatting", 42, file=f)  # auto-adds space and \n
```

### File modes

```python
# Mode string: "r", "w", "a", "x", "b", "t", "+"
# Can be combined: "rb", "w+", "ab"

# r  — read (default). Error if file doesn't exist.
# w  — write. Creates new or TRUNCATES existing.
# a  — append. Creates new or appends to existing.
# x  — exclusive create. Error if file ALREADY exists.
# b  — binary mode (bytes, not str).
# t  — text mode (default).
# +  — read AND write.

# Common combinations:
"r"   # read text (default)
"w"   # write text (overwrites!)
"a"   # append text
"rb"  # read binary
"wb"  # write binary
"r+"  # read and write text (file must exist)
"w+"  # read and write text (creates/truncates)
"x"   # create new text file (fails if exists — safe for avoiding overwrites)
```

### Encoding — always be explicit!

```python
# Default encoding varies by OS! Always specify it.
# UTF-8 is the standard for modern applications.

with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()

with open("output.txt", "w", encoding="utf-8") as f:
    f.write("Привіт, Світе! 🌍")

# Reading files with different encodings (legacy data, Windows files)
with open("legacy.csv", "r", encoding="cp1251") as f:  # Cyrillic Windows
    content = f.read()

with open("latin.txt", "r", encoding="latin-1") as f:
    content = f.read()

# Handle encoding errors gracefully
with open("messy.txt", "r", encoding="utf-8", errors="replace") as f:
    content = f.read()  # invalid bytes → "�"

with open("messy.txt", "r", encoding="utf-8", errors="ignore") as f:
    content = f.read()  # invalid bytes silently dropped

# Detect encoding (for unknown files)
# pip install chardet
import chardet

with open("unknown.txt", "rb") as f:
    raw = f.read()
    result = chardet.detect(raw)
    print(result)  # {'encoding': 'utf-8', 'confidence': 0.99, 'language': ''}
```

---

## 2. `pathlib` — Modern Path Handling

`pathlib` replaces `os.path` — it's object-oriented, cross-platform, and much cleaner.

```python
from pathlib import Path

# Creating paths
p = Path("data/raw/users.csv")
home = Path.home()                    # /home/username
cwd = Path.cwd()                     # current working directory
p = Path("/absolute/path/to/file.txt")
p = Path("relative/path/file.txt")

# Path components
p = Path("/home/user/data/raw/users.csv")
p.name          # "users.csv"
p.stem          # "users" (name without extension)
p.suffix        # ".csv"
p.suffixes      # [".csv"] (or [".tar", ".gz"] for "archive.tar.gz")
p.parent        # Path("/home/user/data/raw")
p.parents[0]    # Path("/home/user/data/raw")
p.parents[1]    # Path("/home/user/data")
p.parts         # ("/", "home", "user", "data", "raw", "users.csv")

# Building paths — use / operator (very Pythonic!)
data_dir = Path("data")
raw_dir = data_dir / "raw"
file_path = raw_dir / "users.csv"
print(file_path)  # data/raw/users.csv

# Compare with os.path:
# os.path.join("data", "raw", "users.csv")  — ugly
# Path("data") / "raw" / "users.csv"        — beautiful
```

### File operations with pathlib

```python
from pathlib import Path

p = Path("data/output.txt")

# Check existence
p.exists()        # True/False
p.is_file()       # True if file
p.is_dir()        # True if directory

# Read/write (one-liners — great for small files)
content = p.read_text(encoding="utf-8")           # read entire file
p.write_text("Hello!", encoding="utf-8")           # write (overwrites)
data = p.read_bytes()                               # read binary
p.write_bytes(b"\x00\x01\x02")                     # write binary

# Create directories
Path("data/processed").mkdir(parents=True, exist_ok=True)
# parents=True  → creates parent dirs if needed (like mkdir -p)
# exist_ok=True → no error if dir already exists

# List files
list(Path("data").iterdir())                     # all items in directory
list(Path("data").glob("*.csv"))                 # all CSV files
list(Path("data").glob("**/*.csv"))              # recursive — all CSV files in all subdirs
list(Path("data").rglob("*.csv"))                # same as above, shorter

# File info
p = Path("data.csv")
p.stat().st_size          # file size in bytes
p.stat().st_mtime         # last modified time (timestamp)

from datetime import datetime
modified = datetime.fromtimestamp(p.stat().st_mtime)
print(f"Last modified: {modified}")

# Rename, move, delete
p.rename("new_name.txt")          # rename
p.replace("destination.txt")      # move (overwrites if exists)
p.unlink()                        # delete file
p.unlink(missing_ok=True)         # delete, no error if missing
```

### pathlib patterns for Data Engineering

```python
from pathlib import Path

# Process all JSON files in a directory tree
data_dir = Path("data/raw")
for json_file in data_dir.rglob("*.json"):
    print(f"Processing: {json_file}")
    data = json.loads(json_file.read_text())
    # ... process data

# Create output path mirroring input structure
input_dir = Path("data/raw")
output_dir = Path("data/processed")

for input_file in input_dir.rglob("*.csv"):
    # Mirror the directory structure
    relative = input_file.relative_to(input_dir)
    output_file = output_dir / relative.with_suffix(".parquet")
    output_file.parent.mkdir(parents=True, exist_ok=True)
    
    print(f"{input_file} → {output_file}")
    # data/raw/2024/jan/users.csv → data/processed/2024/jan/users.parquet

# Find latest file by modification time
latest = max(data_dir.glob("*.csv"), key=lambda p: p.stat().st_mtime)
print(f"Latest file: {latest}")

# Generate unique output filename
from datetime import datetime
timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
output = Path(f"data/output/report_{timestamp}.csv")
```

---

## 3. CSV Files

CSV is the most common data format you'll encounter. Python has a built-in `csv` module, but for Data Engineering, you'll often use `pandas` (Day 27). Still, knowing the stdlib is essential.

### Reading CSV

```python
import csv

# Basic reading
with open("users.csv", "r", encoding="utf-8") as f:
    reader = csv.reader(f)
    header = next(reader)  # first row is header
    for row in reader:
        # row is a list: ["Alice", "25", "Kyiv"]
        name, age, city = row
        print(f"{name}, {age}, {city}")

# As dictionaries (much better — access by column name)
with open("users.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        # row is an OrderedDict: {"name": "Alice", "age": "25", "city": "Kyiv"}
        print(f"{row['name']}, {row['age']}, {row['city']}")
        # ⚠️ All values are STRINGS! You must convert manually.
        age = int(row["age"])
```

### Writing CSV

```python
import csv

# Basic writing
data = [
    ["name", "age", "city"],
    ["Alice", 25, "Kyiv"],
    ["Bob", 30, "London"],
]

with open("output.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f)
    writer.writerows(data)  # write all rows at once
    # or one at a time:
    # writer.writerow(["Charlie", 35, "NYC"])

# ⚠️ newline="" is IMPORTANT on Windows to prevent double newlines

# Dict writing (specify column order)
users = [
    {"name": "Alice", "age": 25, "city": "Kyiv"},
    {"name": "Bob", "age": 30, "city": "London"},
]

with open("output.csv", "w", newline="", encoding="utf-8") as f:
    fieldnames = ["name", "age", "city"]
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()    # write header row
    writer.writerows(users) # write all data rows
```

### Handling CSV quirks

```python
import csv

# Custom delimiter (TSV, pipe-separated, etc.)
with open("data.tsv", "r") as f:
    reader = csv.reader(f, delimiter="\t")
    for row in reader:
        print(row)

# Handle quoting
with open("data.csv", "w", newline="") as f:
    writer = csv.writer(f, quoting=csv.QUOTE_ALL)  # quote everything
    writer.writerow(["name", "bio"])
    writer.writerow(["Alice", 'She said "hello"'])  # handles quotes properly

# Sniff dialect (auto-detect format)
with open("unknown.csv", "r") as f:
    sample = f.read(8192)
    dialect = csv.Sniffer().sniff(sample)
    f.seek(0)
    reader = csv.reader(f, dialect)
    for row in reader:
        print(row)

# Large CSV — process line by line (no memory issues)
def process_large_csv(filepath):
    """Process a multi-GB CSV file with constant memory."""
    with open(filepath, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for i, row in enumerate(reader):
            process_row(row)
            if (i + 1) % 100_000 == 0:
                print(f"Processed {i + 1:,} rows...")
```

---

## 4. JSON

### Reading JSON

```python
import json

# From file
with open("config.json", "r", encoding="utf-8") as f:
    data = json.load(f)  # file → Python dict/list

# From string
json_str = '{"name": "Alice", "age": 25}'
data = json.loads(json_str)  # string → Python dict/list

# Note: load() = from file, loads() = from string (s = string)
```

### Writing JSON

```python
import json

data = {
    "name": "Alice",
    "age": 25,
    "tags": ["python", "data"],
    "address": {"city": "Kyiv", "country": "Ukraine"}
}

# To file
with open("output.json", "w", encoding="utf-8") as f:
    json.dump(data, f, indent=2, ensure_ascii=False)
    # indent=2 → pretty print
    # ensure_ascii=False → allows Ukrainian/emoji characters

# To string
json_str = json.dumps(data, indent=2, ensure_ascii=False)

# Note: dump() = to file, dumps() = to string
```

### JSON type mapping

```python
# Python → JSON
# dict        → object {}
# list, tuple → array []
# str         → string ""
# int, float  → number
# True        → true
# False       → false
# None        → null

# ⚠️ JSON doesn't support:
# - Sets → convert to list first
# - Tuples → become arrays
# - datetime → must serialize manually
# - bytes → must encode (base64)
# - Complex keys → JSON keys must be strings
```

### Custom JSON serialization

```python
import json
from datetime import datetime, date
from pathlib import Path
from decimal import Decimal

# Problem: json.dumps can't handle datetime, Path, Decimal, etc.
# json.dumps({"date": datetime.now()})  # ❌ TypeError

# Solution 1: custom default function
def json_serializer(obj):
    """Handle types that json.dumps can't."""
    if isinstance(obj, (datetime, date)):
        return obj.isoformat()
    if isinstance(obj, Path):
        return str(obj)
    if isinstance(obj, Decimal):
        return float(obj)
    if isinstance(obj, set):
        return list(obj)
    if isinstance(obj, bytes):
        import base64
        return base64.b64encode(obj).decode()
    raise TypeError(f"Type {type(obj)} is not JSON serializable")

data = {
    "timestamp": datetime.now(),
    "price": Decimal("19.99"),
    "tags": {"python", "data"},
    "path": Path("/data/output"),
}

json_str = json.dumps(data, default=json_serializer, indent=2)
print(json_str)
# {
#   "timestamp": "2025-02-14T12:00:00",
#   "price": 19.99,
#   "tags": ["python", "data"],
#   "path": "/data/output"
# }

# Solution 2: custom encoder class
class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        if isinstance(obj, Decimal):
            return float(obj)
        return super().default(obj)

json.dumps(data, cls=CustomEncoder)
```

### JSON Lines (JSONL) — essential for Data Engineering

```python
# JSONL = one JSON object per line
# This is THE format for streaming data, log files, and data pipelines
# Much better than a single JSON array for large datasets because:
# 1. Can process line by line (constant memory)
# 2. Can append without rewriting entire file
# 3. Can parallel process (split by lines)

# Sample data.jsonl:
# {"id": 1, "name": "Alice", "event": "login"}
# {"id": 2, "name": "Bob", "event": "purchase"}
# {"id": 3, "name": "Charlie", "event": "logout"}

import json

# Reading JSONL
def read_jsonl(filepath):
    """Read JSONL file, yield one dict per line."""
    with open(filepath, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if line:  # skip empty lines
                yield json.loads(line)

# Usage
for record in read_jsonl("events.jsonl"):
    print(record["name"], record["event"])

# As a list (if you need all records in memory)
records = list(read_jsonl("events.jsonl"))

# Writing JSONL
def write_jsonl(filepath, records):
    """Write records as JSONL."""
    with open(filepath, "w", encoding="utf-8") as f:
        for record in records:
            f.write(json.dumps(record, ensure_ascii=False) + "\n")

# Append to JSONL
def append_jsonl(filepath, record):
    """Append a single record to a JSONL file."""
    with open(filepath, "a", encoding="utf-8") as f:
        f.write(json.dumps(record, ensure_ascii=False) + "\n")

events = [
    {"id": 1, "event": "click", "timestamp": "2025-02-14T12:00:00"},
    {"id": 2, "event": "view", "timestamp": "2025-02-14T12:01:00"},
]
write_jsonl("events.jsonl", events)
append_jsonl("events.jsonl", {"id": 3, "event": "purchase", "timestamp": "2025-02-14T12:02:00"})
```

---

## 5. YAML

YAML is popular for configuration files (Docker, Kubernetes, CI/CD, Airflow).

```python
# pip install pyyaml
import yaml

# Reading YAML
with open("config.yaml", "r") as f:
    config = yaml.safe_load(f)  # ALWAYS use safe_load, not load!
    # yaml.load() can execute arbitrary Python code — security risk!

# Writing YAML
config = {
    "database": {
        "host": "localhost",
        "port": 5432,
        "name": "mydb"
    },
    "features": ["auth", "logging", "caching"],
    "debug": False
}

with open("config.yaml", "w") as f:
    yaml.dump(config, f, default_flow_style=False, allow_unicode=True)

# Output:
# database:
#   host: localhost
#   name: mydb
#   port: 5432
# debug: false
# features:
# - auth
# - logging
# - caching

# Multiple documents in one YAML file
with open("multi.yaml", "r") as f:
    docs = list(yaml.safe_load_all(f))  # list of dicts
```

### YAML + Pydantic for config management

```python
import yaml
from pydantic import BaseModel

class DatabaseConfig(BaseModel):
    host: str
    port: int = 5432
    name: str
    user: str
    password: str

class AppConfig(BaseModel):
    debug: bool = False
    database: DatabaseConfig
    allowed_origins: list[str] = []

# Load and validate in one step
with open("config.yaml") as f:
    raw = yaml.safe_load(f)

config = AppConfig.model_validate(raw)
# Pydantic validates all fields, converts types, checks constraints
print(config.database.host)
```

---

## 6. Binary Files & Pickle

### Binary file operations

```python
# Reading binary
with open("image.png", "rb") as f:
    data = f.read()      # bytes object
    print(type(data))    # <class 'bytes'>
    print(data[:8])      # first 8 bytes (file header/magic bytes)

# Writing binary
with open("output.bin", "wb") as f:
    f.write(b"\x00\x01\x02\x03")

# Useful for: images, PDFs, compressed files, protobuf, etc.
```

### Pickle — Python object serialization

```python
import pickle

# Pickle can serialize almost ANY Python object
data = {
    "users": [{"name": "Alice", "scores": [95, 87, 92]}],
    "metadata": {"version": 2, "created": datetime.now()},
}

# Save to file
with open("data.pkl", "wb") as f:
    pickle.dump(data, f)

# Load from file
with open("data.pkl", "rb") as f:
    loaded = pickle.load(f)

# Serialize to bytes (for sending over network, caching, etc.)
raw_bytes = pickle.dumps(data)
loaded = pickle.loads(raw_bytes)

# ⚠️ SECURITY WARNING:
# NEVER unpickle data from untrusted sources!
# pickle.load() can execute arbitrary code.
# For untrusted data, use JSON or other safe formats.

# When to use pickle:
# ✅ Caching ML models (sklearn, PyTorch)
# ✅ Saving intermediate pipeline state
# ✅ Inter-process communication
# ❌ Long-term storage (breaks with code changes)
# ❌ Data exchange between languages (Python-only)
# ❌ Untrusted data (security risk)
```

---

## 7. Working with Large Files

### Line-by-line processing (constant memory)

```python
def process_large_file(input_path, output_path):
    """Process a multi-GB text file without loading it all in memory."""
    line_count = 0
    with open(input_path, "r", encoding="utf-8") as fin, \
         open(output_path, "w", encoding="utf-8") as fout:
        for line in fin:          # reads one line at a time
            processed = transform(line.strip())
            fout.write(processed + "\n")
            line_count += 1
            if line_count % 1_000_000 == 0:
                print(f"Processed {line_count:,} lines")
    
    print(f"Total: {line_count:,} lines")
```

### Chunked reading

```python
def read_in_chunks(filepath, chunk_size=8192):
    """Read a file in chunks — good for binary or huge files."""
    with open(filepath, "rb") as f:
        while True:
            chunk = f.read(chunk_size)
            if not chunk:
                break
            yield chunk

# Usage: calculate file hash without loading entire file
import hashlib

def file_hash(filepath, algorithm="sha256"):
    """Calculate hash of a large file efficiently."""
    h = hashlib.new(algorithm)
    for chunk in read_in_chunks(filepath):
        h.update(chunk)
    return h.hexdigest()

print(file_hash("large_file.dat"))
```

### Batch processing CSV

```python
import csv
from itertools import islice

def read_csv_batches(filepath, batch_size=10_000):
    """Read CSV in batches for batch database inserts, API calls, etc."""
    with open(filepath, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        while True:
            batch = list(islice(reader, batch_size))
            if not batch:
                break
            yield batch

# Usage
for i, batch in enumerate(read_csv_batches("huge_data.csv", batch_size=50_000)):
    print(f"Batch {i}: {len(batch)} rows")
    insert_to_database(batch)  # bulk insert
```

### Generator pipeline for large files (Day 6 + Day 8)

```python
import json
import csv
from pathlib import Path

def read_lines(filepath):
    """Stage 1: Read lines lazily."""
    with open(filepath, "r", encoding="utf-8") as f:
        for line in f:
            yield line.strip()

def parse_json_lines(lines):
    """Stage 2: Parse each line as JSON."""
    for line in lines:
        if line:
            try:
                yield json.loads(line)
            except json.JSONDecodeError:
                continue  # skip malformed lines

def filter_events(records, event_type):
    """Stage 3: Filter by event type."""
    for record in records:
        if record.get("event") == event_type:
            yield record

def extract_fields(records, fields):
    """Stage 4: Extract specific fields."""
    for record in records:
        yield {field: record.get(field) for field in fields}

# Build pipeline — nothing executes until iteration!
lines = read_lines("events.jsonl")                          # lazy
records = parse_json_lines(lines)                            # lazy
clicks = filter_events(records, "click")                     # lazy
output = extract_fields(clicks, ["user_id", "timestamp"])    # lazy

# Execute — processes one record at a time through entire pipeline
# Can handle files larger than memory!
for record in output:
    print(record)

# Or write results:
with open("clicks.jsonl", "w") as f:
    for record in output:
        f.write(json.dumps(record) + "\n")
```

---

## 8. `tempfile` — Temporary Files

```python
import tempfile
from pathlib import Path

# Temporary file (auto-deleted when closed)
with tempfile.NamedTemporaryFile(mode="w", suffix=".csv", delete=False) as f:
    f.write("name,age\nAlice,25\n")
    temp_path = f.name
    print(f"Temp file: {temp_path}")

# Temporary directory (auto-deleted when exiting with block)
with tempfile.TemporaryDirectory() as tmpdir:
    temp_path = Path(tmpdir) / "data.json"
    temp_path.write_text('{"key": "value"}')
    # do work with temp files...
# entire directory deleted here

# Useful in Data Engineering for:
# - Downloading files before processing
# - Intermediate pipeline stages
# - Testing
```

---

## 9. `shutil` — High-Level File Operations

```python
import shutil
from pathlib import Path

# Copy file
shutil.copy2("source.txt", "dest.txt")       # preserves metadata
shutil.copy("source.txt", "dest_dir/")        # copy to directory

# Copy entire directory tree
shutil.copytree("source_dir", "dest_dir")

# Move/rename
shutil.move("old_path.txt", "new_path.txt")

# Delete directory tree (CAREFUL!)
shutil.rmtree("directory_to_delete")

# Disk usage
total, used, free = shutil.disk_usage("/")
print(f"Free: {free / (1024**3):.1f} GB")

# Create archive
shutil.make_archive("backup", "zip", "data_dir")   # creates backup.zip
shutil.make_archive("backup", "gztar", "data_dir")  # creates backup.tar.gz

# Extract archive
shutil.unpack_archive("backup.zip", "extracted_dir")
```

---

## 10. Compressed Files

```python
import gzip
import zipfile
import tarfile

# gzip — single file compression (very common in data pipelines)
# Writing
with gzip.open("data.json.gz", "wt", encoding="utf-8") as f:
    json.dump(data, f)

# Reading
with gzip.open("data.json.gz", "rt", encoding="utf-8") as f:
    data = json.load(f)

# Read gzipped CSV line by line
with gzip.open("huge_data.csv.gz", "rt", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        process(row)

# ZIP files — multiple files in one archive
# Reading
with zipfile.ZipFile("archive.zip", "r") as z:
    z.namelist()                          # list files
    z.extractall("output_dir")            # extract all
    with z.open("specific_file.csv") as f:
        content = f.read().decode("utf-8")  # read specific file

# Writing
with zipfile.ZipFile("output.zip", "w", zipfile.ZIP_DEFLATED) as z:
    z.write("file1.csv")
    z.write("file2.csv")
    z.writestr("generated.txt", "Content created in memory")

# TAR files
with tarfile.open("archive.tar.gz", "r:gz") as tar:
    tar.extractall("output_dir")
    members = tar.getmembers()

with tarfile.open("output.tar.gz", "w:gz") as tar:
    tar.add("data_dir")
```

---

## 11. `struct` — Binary Data Packing

```python
import struct

# Pack Python values into bytes (useful for binary protocols, file formats)
packed = struct.pack(">I2sf", 42, b"Hi", 3.14)
# >  = big-endian
# I  = unsigned int (4 bytes)
# 2s = 2-byte string
# f  = float (4 bytes)

# Unpack bytes back to Python values
values = struct.unpack(">I2sf", packed)
print(values)  # (42, b'Hi', 3.140000104904175)

# Read binary file header
with open("data.bin", "rb") as f:
    header = struct.unpack(">4sII", f.read(12))
    magic, version, record_count = header
    print(f"Magic: {magic}, Version: {version}, Records: {record_count}")

# Mostly relevant for:
# - Custom binary file formats
# - Network protocols
# - Low-level data processing
# - Reading headers of images, audio, etc.
```

---

## 12. Complete Data Pipeline Example

```python
"""
Mini ETL pipeline: 
  Extract from multiple JSON files 
  → Transform (clean, validate, enrich)
  → Load to CSV output
"""
import json
import csv
from pathlib import Path
from datetime import datetime
from pydantic import BaseModel, field_validator
from typing import Generator

# --- Schema ---
class RawEvent(BaseModel):
    event_id: str
    user_id: int
    event_type: str
    timestamp: str
    properties: dict = {}
    
    @field_validator("timestamp")
    @classmethod
    def parse_timestamp(cls, v):
        # Validate timestamp format
        datetime.fromisoformat(v)
        return v

class ProcessedEvent(BaseModel):
    event_id: str
    user_id: int
    event_type: str
    timestamp: str
    date: str
    hour: int
    has_properties: bool

# --- Extract ---
def extract_json_files(directory: Path) -> Generator[dict, None, None]:
    """Read all JSON/JSONL files from a directory."""
    for filepath in sorted(directory.rglob("*.jsonl")):
        print(f"Reading: {filepath}")
        with open(filepath, "r", encoding="utf-8") as f:
            for line in f:
                line = line.strip()
                if line:
                    yield json.loads(line)

# --- Transform ---
def validate_and_transform(
    raw_records: Generator[dict, None, None]
) -> tuple[list[ProcessedEvent], list[dict]]:
    """Validate records, transform valid ones, collect errors."""
    valid = []
    errors = []
    
    for raw in raw_records:
        try:
            event = RawEvent.model_validate(raw)
            
            # Enrich with derived fields
            ts = datetime.fromisoformat(event.timestamp)
            processed = ProcessedEvent(
                event_id=event.event_id,
                user_id=event.user_id,
                event_type=event.event_type,
                timestamp=event.timestamp,
                date=ts.strftime("%Y-%m-%d"),
                hour=ts.hour,
                has_properties=bool(event.properties),
            )
            valid.append(processed)
        except Exception as e:
            errors.append({"raw": raw, "error": str(e)})
    
    return valid, errors

# --- Load ---
def load_to_csv(records: list[ProcessedEvent], output_path: Path) -> None:
    """Write processed records to CSV."""
    output_path.parent.mkdir(parents=True, exist_ok=True)
    
    with open(output_path, "w", newline="", encoding="utf-8") as f:
        if not records:
            return
        
        fieldnames = list(records[0].model_dump().keys())
        writer = csv.DictWriter(f, fieldnames=fieldnames)
        writer.writeheader()
        
        for record in records:
            writer.writerow(record.model_dump())

def save_errors(errors: list[dict], output_path: Path) -> None:
    """Save validation errors for debugging."""
    output_path.parent.mkdir(parents=True, exist_ok=True)
    with open(output_path, "w", encoding="utf-8") as f:
        for error in errors:
            f.write(json.dumps(error, ensure_ascii=False) + "\n")

# --- Pipeline ---
def run_pipeline():
    input_dir = Path("data/raw")
    output_dir = Path("data/processed")
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    
    print("=" * 60)
    print(f"Pipeline started at {datetime.now()}")
    print("=" * 60)
    
    # Extract
    raw_records = extract_json_files(input_dir)
    
    # Transform
    valid, errors = validate_and_transform(raw_records)
    
    # Load
    load_to_csv(valid, output_dir / f"events_{timestamp}.csv")
    save_errors(errors, output_dir / f"errors_{timestamp}.jsonl")
    
    # Report
    print(f"\nResults:")
    print(f"  Valid records:  {len(valid):,}")
    print(f"  Error records:  {len(errors):,}")
    print(f"  Success rate:   {len(valid)/(len(valid)+len(errors))*100:.1f}%")
    print(f"\nOutput: {output_dir}")

if __name__ == "__main__":
    run_pipeline()
```

---

## 📝 Practice Tasks

### Task 1: File Statistics Tool
Write a CLI tool that takes a file path and reports: file size (human-readable), line count, word count, character count, encoding (detected), most common words (top 10). Support `.txt`, `.csv`, `.json`, `.jsonl`.

### Task 2: CSV Cleaner
Write a script that reads a messy CSV file (inconsistent quoting, extra whitespace, empty rows, mixed encoding) and outputs a clean, properly formatted CSV. Use `csv.Sniffer` to detect the dialect.

### Task 3: JSON ↔ CSV Converter
Write bidirectional converters:
- `json_to_csv(input_path, output_path)` — flatten nested JSON to CSV
- `csv_to_json(input_path, output_path)` — convert CSV to JSON array or JSONL

Handle nested objects by flattening keys: `{"address": {"city": "Kyiv"}}` → column `address.city`.

### Task 4: Log File Analyzer
Given a JSONL log file with `{"timestamp", "level", "message", "service"}` entries:
- Count events by level (INFO, WARNING, ERROR)
- Find the busiest hour
- Find the most error-prone service
- Output a summary report as both JSON and formatted text

### Task 5: Large File Splitter
Write a tool that splits a large CSV file into smaller chunks:
- `split_csv("huge.csv", max_rows=100_000)` → creates `huge_001.csv`, `huge_002.csv`, etc.
- Each chunk should include the header row
- Report total rows and number of chunks created

### Task 6: Config Manager
Build a configuration system that:
- Reads from YAML config file
- Overrides with environment variables
- Validates with Pydantic
- Supports different environments (dev, staging, prod)
- Writes resolved config back to JSON for auditing

---

## 📚 Resources

- [Python Official — Reading and Writing Files](https://docs.python.org/3/tutorial/inputoutput.html#reading-and-writing-files)
- [Python Official — pathlib](https://docs.python.org/3/library/pathlib.html)
- [Python Official — csv module](https://docs.python.org/3/library/csv.html)
- [Python Official — json module](https://docs.python.org/3/library/json.html)
- [Real Python — Working with Files](https://realpython.com/working-with-files-in-python/)
- [Real Python — Read/Write CSV](https://realpython.com/python-csv/)
- [Real Python — JSON](https://realpython.com/python-json/)
- [Real Python — pathlib](https://realpython.com/python-pathlib/)
- [PyYAML Documentation](https://pyyaml.org/wiki/PyYAMLDocumentation)

---

## 🔑 Key Takeaways

1. **Always use `with`** for file operations — guarantees cleanup.
2. **Always specify `encoding="utf-8"`** — don't rely on system default.
3. **`pathlib` over `os.path`** — cleaner, more Pythonic, use the `/` operator.
4. **JSONL over JSON arrays** for data pipelines — streamable, appendable, memory-efficient.
5. **Process large files line by line** — never `.read()` a multi-GB file.
6. **Generator pipelines** for ETL — chain lazy generators for constant-memory processing.
7. **`csv.DictReader`/`DictWriter`** — always use dict mode for readability.
8. **`yaml.safe_load()`** — NEVER `yaml.load()` (security risk).
9. **Pydantic for validation at boundaries** — validate data as soon as it enters your pipeline.
10. **Separate valid/error records** — the fundamental pattern of data quality in pipelines.

---

> **Tomorrow (Day 9):** Regular Expressions & Text Processing — `re` module deep dive, common patterns, parsing and cleaning messy data.

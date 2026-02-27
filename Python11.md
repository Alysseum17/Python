# Day 11: Threading & Multiprocessing

## 🎯 Goal

Understand how Python handles concurrency and parallelism. This is where Python gets tricky because of the **GIL** — a concept that doesn't exist in Java or JS. You need to know when to use threads, processes, or async (Day 12), and why choosing wrong can make your code SLOWER.

---

## 1. Concurrency vs Parallelism

```
Concurrency: multiple tasks make progress (may alternate on one CPU core)
  → Like a cook switching between stirring soup and chopping vegetables
  → threading, asyncio

Parallelism: multiple tasks run SIMULTANEOUSLY on multiple CPU cores
  → Like two cooks each doing their own task at the same time
  → multiprocessing

JS comparison:
  → JS is single-threaded + event loop (like Python asyncio)
  → JS has Web Workers for parallelism (like Python multiprocessing)

Java comparison:
  → Java threads are TRUE parallel (no GIL)
  → Python threads are concurrent but NOT parallel for CPU work (because of GIL)
```

---

## 2. The GIL — Python's Biggest Limitation

### What is the GIL?

```
GIL = Global Interpreter Lock

The GIL is a mutex (lock) in CPython that allows only ONE thread
to execute Python bytecode at a time.

Even with 8 CPU cores and 8 threads, only 1 thread runs Python code
at any given moment. The others wait.
```

### Why does the GIL exist?

```
CPython's memory management (reference counting) is not thread-safe.
The GIL is the simplest solution — lock everything globally.

It's a design choice from 1992 when multi-core CPUs didn't exist.
Many attempts to remove it have failed because too much C extension
code depends on it. (Python 3.13+ has experimental "free-threaded" mode
that removes the GIL, but it's not production-ready yet.)
```

### What does this mean in practice?

```python
# CPU-bound work (math, data processing, parsing)
# ❌ Threading does NOT help — GIL prevents parallel execution
# ✅ Multiprocessing DOES help — each process has its own GIL

# I/O-bound work (network requests, file reads, database queries)
# ✅ Threading DOES help — GIL is released during I/O waits
# ✅ asyncio ALSO helps — even more efficient for I/O

# Rule of thumb:
# I/O-bound → threading or asyncio
# CPU-bound → multiprocessing
```

### Visual explanation

```
CPU-BOUND with 4 threads (GIL blocks parallelism):

Thread 1: ████████░░░░░░░░████████░░░░░░░░  (runs, then waits for GIL)
Thread 2: ░░░░░░░░████████░░░░░░░░████████  (waits, then runs)
Thread 3: waiting...waiting...waiting...     (barely gets a turn)
Thread 4: waiting...waiting...waiting...
                                              
Total time: SAME as 1 thread (or worse due to GIL switching overhead!)

I/O-BOUND with 4 threads (GIL released during I/O):

Thread 1: ██░░░░░░░░░░░░██  (runs, does I/O wait, runs again)
Thread 2: ░░██░░░░░░░░██░░  (runs while Thread 1 waits)
Thread 3: ░░░░██░░░░██░░░░  (runs while others wait)
Thread 4: ░░░░░░████░░░░░░  (runs while others wait)

Total time: ~4x faster! (I/O waits overlap)
```

---

## 3. `threading` — Concurrent I/O

### Basic threading

```python
import threading
import time

def download_page(url: str) -> None:
    """Simulate downloading a web page."""
    print(f"[{threading.current_thread().name}] Downloading {url}...")
    time.sleep(2)  # simulate network I/O
    print(f"[{threading.current_thread().name}] Done: {url}")

# Sequential — slow
start = time.perf_counter()
for url in ["page1", "page2", "page3", "page4"]:
    download_page(url)
print(f"Sequential: {time.perf_counter() - start:.1f}s")  # ~8 seconds

# Threaded — fast
start = time.perf_counter()
threads = []
for url in ["page1", "page2", "page3", "page4"]:
    t = threading.Thread(target=download_page, args=(url,))
    threads.append(t)
    t.start()  # start the thread

# Wait for all threads to finish
for t in threads:
    t.join()  # blocks until thread completes

print(f"Threaded: {time.perf_counter() - start:.1f}s")  # ~2 seconds
```

### Thread with return values

```python
import threading

# Threads don't return values directly — use shared data structures
results: dict[str, str] = {}
lock = threading.Lock()

def fetch_data(url: str) -> None:
    """Fetch data and store result in shared dict."""
    data = f"Data from {url}"  # simulate work
    with lock:  # thread-safe write
        results[url] = data

threads = []
for url in ["api/users", "api/orders", "api/products"]:
    t = threading.Thread(target=fetch_data, args=(url,))
    threads.append(t)
    t.start()

for t in threads:
    t.join()

print(results)
# {"api/users": "Data from api/users", "api/orders": "...", ...}
```

### Thread safety — Locks

```python
import threading

# ❌ Race condition — counter will be WRONG
counter = 0

def increment_unsafe():
    global counter
    for _ in range(100_000):
        counter += 1  # NOT atomic! Read-modify-write can interleave

threads = [threading.Thread(target=increment_unsafe) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(f"Unsafe counter: {counter}")  # Often less than 400,000!

# ✅ Thread-safe with Lock
counter = 0
lock = threading.Lock()

def increment_safe():
    global counter
    for _ in range(100_000):
        with lock:  # only one thread can execute this block at a time
            counter += 1

threads = [threading.Thread(target=increment_safe) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(f"Safe counter: {counter}")  # Always 400,000

# Java comparison:
# synchronized (lock) { counter++; }
# or AtomicInteger
```

### Other synchronization primitives

```python
import threading

# RLock — reentrant lock (same thread can acquire multiple times)
rlock = threading.RLock()
with rlock:
    with rlock:  # OK! Same thread can re-enter
        pass
# Regular Lock would deadlock here

# Semaphore — allows N threads at once (rate limiting)
semaphore = threading.Semaphore(3)  # max 3 concurrent

def limited_work(task_id):
    with semaphore:  # at most 3 threads in this block
        print(f"Task {task_id} running")
        time.sleep(1)

# Event — signal between threads
event = threading.Event()

def waiter():
    print("Waiting for signal...")
    event.wait()  # blocks until event is set
    print("Got signal!")

def signaler():
    time.sleep(2)
    print("Sending signal!")
    event.set()  # unblocks all waiters

# Barrier — wait for N threads to reach a point
barrier = threading.Barrier(3)

def sync_point(thread_id):
    print(f"Thread {thread_id} reached barrier")
    barrier.wait()  # blocks until all 3 threads arrive
    print(f"Thread {thread_id} continues")
```

### Daemon threads

```python
import threading
import time

def background_task():
    """Runs in background, killed when main thread exits."""
    while True:
        print("Background work...")
        time.sleep(1)

# Daemon thread — doesn't prevent program from exiting
t = threading.Thread(target=background_task, daemon=True)
t.start()

time.sleep(3)
print("Main thread done — daemon will be killed")
# Program exits, daemon thread is killed automatically

# Non-daemon threads (default) block program exit until they complete
```

---

## 4. `multiprocessing` — True Parallelism

Each process gets its own Python interpreter and GIL → true parallel execution.

### Basic multiprocessing

```python
import multiprocessing
import time
import os

def cpu_heavy_task(n: int) -> int:
    """CPU-bound: count primes up to n."""
    count = 0
    for num in range(2, n):
        if all(num % i != 0 for i in range(2, int(num**0.5) + 1)):
            count += 1
    return count

# Sequential
start = time.perf_counter()
results = [cpu_heavy_task(50_000) for _ in range(4)]
print(f"Sequential: {time.perf_counter() - start:.1f}s")  # ~8s

# Multiprocessing
start = time.perf_counter()
with multiprocessing.Pool(processes=4) as pool:
    results = pool.map(cpu_heavy_task, [50_000] * 4)
print(f"Parallel: {time.perf_counter() - start:.1f}s")  # ~2-3s
print(f"Results: {results}")
```

### Process vs Thread — when to use which

```python
import multiprocessing
import threading
import time

def cpu_work():
    """Pure CPU: count to 10 million."""
    total = 0
    for i in range(10_000_000):
        total += i
    return total

def io_work():
    """Simulate I/O: sleep."""
    time.sleep(1)
    return "done"

# ── CPU-bound: multiprocessing wins ──
start = time.perf_counter()
with multiprocessing.Pool(4) as pool:
    pool.map(lambda _: cpu_work(), range(4))
print(f"CPU multiprocessing: {time.perf_counter() - start:.1f}s")  # ~1.5s

start = time.perf_counter()
threads = [threading.Thread(target=cpu_work) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(f"CPU threading: {time.perf_counter() - start:.1f}s")  # ~4s (GIL!)

# ── I/O-bound: threading is fine (and lighter) ──
start = time.perf_counter()
threads = [threading.Thread(target=io_work) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(f"I/O threading: {time.perf_counter() - start:.1f}s")  # ~1s

start = time.perf_counter()
with multiprocessing.Pool(4) as pool:
    pool.map(lambda _: io_work(), range(4))
print(f"I/O multiprocessing: {time.perf_counter() - start:.1f}s")  # ~1s (works but heavier)
```

### Sharing data between processes

```python
import multiprocessing

# ⚠️ Processes don't share memory (unlike threads)!
# Each process gets a COPY of data.

# Option 1: multiprocessing.Value and Array (shared memory)
counter = multiprocessing.Value("i", 0)  # "i" = integer
lock = multiprocessing.Lock()

def increment(shared_counter, shared_lock):
    for _ in range(100_000):
        with shared_lock:
            shared_counter.value += 1

processes = [
    multiprocessing.Process(target=increment, args=(counter, lock))
    for _ in range(4)
]
for p in processes: p.start()
for p in processes: p.join()
print(f"Shared counter: {counter.value}")  # 400,000

# Option 2: multiprocessing.Queue (message passing)
queue = multiprocessing.Queue()

def producer(q, items):
    for item in items:
        q.put(item)

def consumer(q, results_list):
    while not q.empty():
        item = q.get()
        results_list.append(item * 2)

# Option 3: Manager (shared objects — dict, list, etc.)
with multiprocessing.Manager() as manager:
    shared_dict = manager.dict()
    shared_list = manager.list()
    
    def worker(d, name, value):
        d[name] = value
    
    processes = [
        multiprocessing.Process(target=worker, args=(shared_dict, f"key_{i}", i))
        for i in range(5)
    ]
    for p in processes: p.start()
    for p in processes: p.join()
    
    print(dict(shared_dict))  # {"key_0": 0, "key_1": 1, ...}
```

---

## 5. `concurrent.futures` — The High-Level API ⭐

This is what you should use 90% of the time. Same interface for both threads and processes.

### ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def fetch_url(url: str) -> dict:
    """Simulate fetching a URL."""
    time.sleep(1)  # simulate network I/O
    return {"url": url, "status": 200, "size": len(url) * 100}

urls = [f"https://api.example.com/page/{i}" for i in range(10)]

# ── Using submit() + as_completed() ──
# Results come back as they finish (not in order)
start = time.perf_counter()

with ThreadPoolExecutor(max_workers=5) as executor:
    # Submit all tasks
    future_to_url = {executor.submit(fetch_url, url): url for url in urls}
    
    # Process results as they complete
    for future in as_completed(future_to_url):
        url = future_to_url[future]
        try:
            result = future.result()  # get return value (or raise exception)
            print(f"✅ {url}: {result['status']}")
        except Exception as e:
            print(f"❌ {url}: {e}")

print(f"Total: {time.perf_counter() - start:.1f}s")  # ~2s (10 URLs, 5 workers)

# ── Using map() ──
# Results come back IN ORDER (simpler but waits for each in sequence)
with ThreadPoolExecutor(max_workers=5) as executor:
    results = list(executor.map(fetch_url, urls))
    # results is a list in the same order as urls
```

### ProcessPoolExecutor

```python
from concurrent.futures import ProcessPoolExecutor
import math

def compute_factorial(n: int) -> int:
    """CPU-intensive computation."""
    return math.factorial(n)

numbers = [100_000, 90_000, 80_000, 70_000]

# Same API as ThreadPoolExecutor — just swap the class!
with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(compute_factorial, numbers))

for n, result in zip(numbers, results):
    print(f"{n}! has {len(str(result)):,} digits")
```

### Practical pattern: parallel downloads with progress

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def download_file(file_info: dict) -> dict:
    """Simulate downloading a file."""
    time.sleep(file_info["size"] / 100)  # bigger files take longer
    return {
        "name": file_info["name"],
        "size": file_info["size"],
        "status": "ok",
    }

files = [
    {"name": f"file_{i}.csv", "size": (i + 1) * 50}
    for i in range(20)
]

downloaded = 0
failed = 0

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = {executor.submit(download_file, f): f for f in files}
    
    for future in as_completed(futures):
        file_info = futures[future]
        try:
            result = future.result(timeout=30)  # 30s timeout per file
            downloaded += 1
            progress = (downloaded + failed) / len(files) * 100
            print(f"[{progress:5.1f}%] ✅ {result['name']}")
        except Exception as e:
            failed += 1
            print(f"         ❌ {file_info['name']}: {e}")

print(f"\nDone: {downloaded} downloaded, {failed} failed")
```

### Error handling

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import random

def unreliable_task(task_id: int) -> str:
    """Sometimes fails."""
    if random.random() < 0.3:
        raise ConnectionError(f"Task {task_id} failed!")
    return f"Task {task_id} result"

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(unreliable_task, i): i for i in range(10)}
    
    results = []
    errors = []
    
    for future in as_completed(futures):
        task_id = futures[future]
        try:
            result = future.result()
            results.append(result)
        except Exception as e:
            errors.append({"task_id": task_id, "error": str(e)})

print(f"Successes: {len(results)}, Failures: {len(errors)}")
for err in errors:
    print(f"  ❌ Task {err['task_id']}: {err['error']}")
```

---

## 6. Decision Matrix — What to Use When

```
┌─────────────────────────────────────────────────────────────┐
│                    WHAT SHOULD I USE?                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  I/O-bound (network, files, DB)?                             │
│    ├─ Few tasks (< 100):  ThreadPoolExecutor                 │
│    ├─ Many tasks (100+):  asyncio (Day 12)                   │
│    └─ Simple script:      threading.Thread                   │
│                                                              │
│  CPU-bound (math, parsing, compression)?                     │
│    ├─ Parallelizable:     ProcessPoolExecutor                │
│    ├─ Large data:         multiprocessing.Pool               │
│    └─ Very large data:    Use Spark/Dask instead             │
│                                                              │
│  Mixed (I/O + CPU)?                                          │
│    └─ ThreadPool for I/O + ProcessPool for CPU               │
│       or asyncio + ProcessPoolExecutor                       │
│                                                              │
│  Data Engineering specific:                                  │
│    ├─ Parallel API calls:      ThreadPoolExecutor            │
│    ├─ Parallel file processing: ProcessPoolExecutor          │
│    ├─ Parallel DB inserts:     ThreadPoolExecutor            │
│    ├─ ETL batch processing:    multiprocessing.Pool          │
│    └─ Large-scale processing:  Spark / Dask / Ray            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Quick comparison

| Feature | threading | multiprocessing | asyncio (Day 12) |
|---------|-----------|-----------------|-------------------|
| Best for | I/O-bound | CPU-bound | High-concurrency I/O |
| GIL issue | Yes (limited) | No (separate GILs) | N/A (single thread) |
| Memory | Shared | Separate (copies) | Shared |
| Overhead | Low | High (new process) | Very low |
| Max concurrent | ~100s | ~CPU cores | ~10,000s |
| Data sharing | Easy (but locks!) | Hard (pipes, queues) | Easy (single thread) |
| Debugging | Hard | Harder | Medium |

---

## 7. Real-World Data Engineering Patterns

### Pattern 1: Parallel file ingestion

```python
from concurrent.futures import ProcessPoolExecutor, as_completed
from pathlib import Path
import csv
import json

def process_csv_file(filepath: Path) -> dict:
    """Process a single CSV file — runs in separate process."""
    row_count = 0
    error_count = 0
    
    with open(filepath, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            try:
                # validate, transform, etc.
                row_count += 1
            except Exception:
                error_count += 1
    
    return {
        "file": filepath.name,
        "rows": row_count,
        "errors": error_count,
    }

def ingest_all_files(data_dir: Path, max_workers: int = 4) -> list[dict]:
    """Process all CSV files in parallel."""
    csv_files = list(data_dir.glob("*.csv"))
    results = []
    
    with ProcessPoolExecutor(max_workers=max_workers) as executor:
        futures = {executor.submit(process_csv_file, f): f for f in csv_files}
        
        for future in as_completed(futures):
            filepath = futures[future]
            try:
                result = future.result()
                results.append(result)
                print(f"✅ {result['file']}: {result['rows']} rows")
            except Exception as e:
                print(f"❌ {filepath.name}: {e}")
    
    total_rows = sum(r["rows"] for r in results)
    print(f"\nTotal: {len(results)} files, {total_rows:,} rows")
    return results
```

### Pattern 2: Parallel API fetching with rate limiting

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import threading
import time

class RateLimitedFetcher:
    """Fetch data from API with concurrency + rate limiting."""
    
    def __init__(self, max_workers: int = 5, requests_per_second: float = 10):
        self.max_workers = max_workers
        self.min_interval = 1.0 / requests_per_second
        self._last_request = 0.0
        self._lock = threading.Lock()
    
    def _rate_limit(self):
        """Enforce rate limit."""
        with self._lock:
            now = time.time()
            elapsed = now - self._last_request
            if elapsed < self.min_interval:
                time.sleep(self.min_interval - elapsed)
            self._last_request = time.time()
    
    def fetch_one(self, endpoint: str) -> dict:
        """Fetch a single endpoint."""
        self._rate_limit()
        # In real code: requests.get(endpoint)
        time.sleep(0.1)  # simulate request
        return {"endpoint": endpoint, "data": f"response from {endpoint}"}
    
    def fetch_all(self, endpoints: list[str]) -> list[dict]:
        """Fetch all endpoints with rate-limited concurrency."""
        results = []
        errors = []
        
        with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            futures = {
                executor.submit(self.fetch_one, ep): ep
                for ep in endpoints
            }
            
            for future in as_completed(futures):
                endpoint = futures[future]
                try:
                    result = future.result()
                    results.append(result)
                except Exception as e:
                    errors.append({"endpoint": endpoint, "error": str(e)})
        
        return results

# Usage
fetcher = RateLimitedFetcher(max_workers=5, requests_per_second=10)
endpoints = [f"/api/users/{i}" for i in range(50)]
data = fetcher.fetch_all(endpoints)
```

### Pattern 3: Producer-consumer with Queue

```python
import threading
import queue
import time
import json
from pathlib import Path

def producer(file_queue: queue.Queue, data_dir: Path):
    """Read files and put raw data into queue."""
    for filepath in data_dir.glob("*.jsonl"):
        with open(filepath) as f:
            for line in f:
                file_queue.put(line.strip())
    
    # Signal "no more data" to consumers
    file_queue.put(None)  # sentinel value

def consumer(
    file_queue: queue.Queue,
    result_queue: queue.Queue,
    consumer_id: int,
):
    """Process data from queue."""
    processed = 0
    while True:
        item = file_queue.get()
        if item is None:
            # Pass sentinel to next consumer
            file_queue.put(None)
            break
        
        try:
            record = json.loads(item)
            # ... transform ...
            result_queue.put(record)
            processed += 1
        except json.JSONDecodeError:
            pass
        finally:
            file_queue.task_done()
    
    print(f"Consumer {consumer_id} processed {processed} records")

def writer(result_queue: queue.Queue, output_path: Path):
    """Write results from queue to file."""
    with open(output_path, "w") as f:
        while True:
            try:
                record = result_queue.get(timeout=5)
                f.write(json.dumps(record) + "\n")
                result_queue.task_done()
            except queue.Empty:
                break

# Orchestrate
file_q: queue.Queue = queue.Queue(maxsize=1000)  # backpressure!
result_q: queue.Queue = queue.Queue(maxsize=1000)

prod = threading.Thread(target=producer, args=(file_q, Path("data/")))
consumers = [
    threading.Thread(target=consumer, args=(file_q, result_q, i))
    for i in range(4)
]
wrt = threading.Thread(target=writer, args=(result_q, Path("output.jsonl")))

prod.start()
for c in consumers: c.start()
wrt.start()

prod.join()
for c in consumers: c.join()
result_q.put(None)  # signal writer to stop
wrt.join()
```

---

## 8. `multiprocessing.Pool` — Map-Reduce Style

```python
import multiprocessing
from functools import partial

def transform_record(record: dict, multiplier: int = 1) -> dict:
    """Transform a single record."""
    return {
        "id": record["id"],
        "value": record["value"] * multiplier,
        "processed": True,
    }

records = [{"id": i, "value": i * 10} for i in range(1000)]

with multiprocessing.Pool(processes=4) as pool:
    # map — apply function to each item
    results = pool.map(partial(transform_record, multiplier=2), records)
    
    # imap — lazy iterator (memory efficient for large datasets)
    for result in pool.imap(transform_record, records, chunksize=100):
        pass  # process one at a time
    
    # imap_unordered — fastest, results in any order
    for result in pool.imap_unordered(transform_record, records, chunksize=100):
        pass  # order doesn't matter
    
    # starmap — for functions with multiple arguments
    pairs = [(1, 2), (3, 4), (5, 6)]
    sums = pool.starmap(lambda a, b: a + b, pairs)
    # [3, 7, 11]

    # apply_async — single task, non-blocking
    future = pool.apply_async(transform_record, (records[0],))
    result = future.get(timeout=10)  # blocks until result ready
```

### Chunked processing for large data

```python
import multiprocessing
import csv
from itertools import islice
from pathlib import Path

def process_chunk(chunk: list[dict]) -> list[dict]:
    """Process a batch of records (runs in separate process)."""
    results = []
    for record in chunk:
        transformed = {
            "name": record["name"].upper(),
            "value": float(record["value"]) * 2,
        }
        results.append(transformed)
    return results

def chunked_reader(filepath: Path, chunk_size: int = 10_000):
    """Read CSV in chunks."""
    with open(filepath, "r") as f:
        reader = csv.DictReader(f)
        while True:
            chunk = list(islice(reader, chunk_size))
            if not chunk:
                break
            yield chunk

def parallel_process(filepath: Path, output_path: Path, workers: int = 4):
    """Process large CSV in parallel chunks."""
    chunks = list(chunked_reader(filepath, chunk_size=5_000))
    
    all_results = []
    with multiprocessing.Pool(processes=workers) as pool:
        for result_chunk in pool.imap(process_chunk, chunks):
            all_results.extend(result_chunk)
            print(f"Processed batch, total: {len(all_results):,}")
    
    # Write results
    if all_results:
        with open(output_path, "w", newline="") as f:
            writer = csv.DictWriter(f, fieldnames=all_results[0].keys())
            writer.writeheader()
            writer.writerows(all_results)
    
    print(f"Done: {len(all_results):,} records → {output_path}")
```

---

## 9. Thread-Safe Data Structures

```python
import queue
import threading

# queue.Queue — thread-safe FIFO (most important!)
q = queue.Queue(maxsize=100)  # blocks put() when full
q.put("item")
item = q.get()                # blocks if empty
q.get(timeout=5)              # timeout after 5 seconds
q.get_nowait()                # raise queue.Empty immediately

# queue.LifoQueue — thread-safe stack (LIFO)
stack = queue.LifoQueue()

# queue.PriorityQueue — thread-safe priority queue
pq = queue.PriorityQueue()
pq.put((1, "high priority"))
pq.put((10, "low priority"))
item = pq.get()  # (1, "high priority")

# collections.deque — thread-safe for append/pop from both ends
from collections import deque
d = deque(maxlen=1000)
d.append("item")       # thread-safe
d.appendleft("item")   # thread-safe
d.pop()                 # thread-safe
d.popleft()             # thread-safe
# BUT iterating over deque is NOT thread-safe!

# threading.local — thread-local storage
local_data = threading.local()

def worker():
    local_data.value = threading.current_thread().name
    print(f"{local_data.value}")  # each thread has its own .value
```

---

## 10. Common Pitfalls

### Pitfall 1: Deadlock

```python
import threading

lock_a = threading.Lock()
lock_b = threading.Lock()

def worker1():
    with lock_a:
        time.sleep(0.1)
        with lock_b:  # ❌ Waits for lock_b, which worker2 holds
            print("Worker 1 done")

def worker2():
    with lock_b:
        time.sleep(0.1)
        with lock_a:  # ❌ Waits for lock_a, which worker1 holds
            print("Worker 2 done")

# DEADLOCK! Both threads wait for each other forever.

# Fix: Always acquire locks in the same order
def worker1_fixed():
    with lock_a:
        with lock_b:
            print("Worker 1 done")

def worker2_fixed():
    with lock_a:  # same order as worker1
        with lock_b:
            print("Worker 2 done")
```

### Pitfall 2: Forgetting if __name__ == "__main__"

```python
import multiprocessing

# ❌ On Windows/macOS, this will spawn infinite processes!
# (because multiprocessing imports your module to create new processes)

# pool = multiprocessing.Pool(4)  # BAD — at module level

# ✅ Always guard with __name__
if __name__ == "__main__":
    with multiprocessing.Pool(4) as pool:
        results = pool.map(some_func, data)
```

### Pitfall 3: Pickling errors in multiprocessing

```python
import multiprocessing

# multiprocessing needs to PICKLE (serialize) data to send between processes
# Lambdas, inner functions, and some objects can't be pickled

# ❌ Fails
# pool.map(lambda x: x * 2, [1, 2, 3])  # can't pickle lambda

# ✅ Use named top-level functions
def double(x):
    return x * 2

with multiprocessing.Pool(4) as pool:
    results = pool.map(double, [1, 2, 3])
```

---

## 📝 Practice Tasks

### Task 1: Parallel File Downloader
Simulate downloading 20 files with `ThreadPoolExecutor`. Each "download" is a `time.sleep(random.uniform(0.5, 2.0))`. Show progress (X/20), handle occasional failures (random 20% failure rate), report total time vs estimated sequential time.

### Task 2: CPU Benchmark
Write a function that checks if a number is prime. Use `ProcessPoolExecutor` to find all primes in range 1–500,000. Compare times: sequential vs 2/4/8 processes. Print a speedup table.

### Task 3: Thread-Safe Counter
Implement a `SafeCounter` class with `increment()`, `decrement()`, `get_value()` methods. Test with 10 threads each incrementing 100,000 times. Verify the final count is correct. Then implement a version WITHOUT locks and show it's wrong.

### Task 4: Producer-Consumer Pipeline
Build a 3-stage pipeline using `queue.Queue`:
- Producer: reads a CSV file and puts rows into queue1
- Transformer: reads from queue1, transforms, puts into queue2
- Writer: reads from queue2, writes to output file
Use multiple transformer threads. Handle graceful shutdown with sentinel values.

### Task 5: Parallel Web Scraper Simulator
Simulate scraping 100 pages. Each page has a random response time (0.1-1.0s) and 10% chance of failure. Implement with `ThreadPoolExecutor`, max 10 concurrent requests, retry failed pages up to 3 times. Print final statistics.

### Task 6: Multiprocessing vs Threading Race
Write the same computation-heavy task and the same I/O-heavy task. Run each with: sequential, threading (4), multiprocessing (4). Create a formatted comparison table showing times and speedups. Prove that threading helps I/O but not CPU, and multiprocessing helps CPU.

---

## 📚 Resources

- [Python Official — threading](https://docs.python.org/3/library/threading.html)
- [Python Official — multiprocessing](https://docs.python.org/3/library/multiprocessing.html)
- [Python Official — concurrent.futures](https://docs.python.org/3/library/concurrent.futures.html)
- [Real Python — Threading](https://realpython.com/intro-to-python-threading/)
- [Real Python — Multiprocessing](https://realpython.com/python-multiprocessing/)
- [Real Python — concurrent.futures](https://realpython.com/python-concurrency/)
- [Real Python — Speed Up Python with Concurrency](https://realpython.com/python-concurrency/)
- [Python GIL Explained](https://realpython.com/python-gil/)
- [PEP 703 — Making the GIL Optional](https://peps.python.org/pep-0703/) (Python 3.13+)

---

## 🔑 Key Takeaways

1. **GIL blocks CPU parallelism in threads** — Python threads can't use multiple cores for computation.
2. **I/O-bound → `ThreadPoolExecutor`** — GIL is released during I/O, threads overlap waits.
3. **CPU-bound → `ProcessPoolExecutor`** — separate processes, separate GILs, true parallelism.
4. **`concurrent.futures` is your default** — same API for both threads and processes, clean and simple.
5. **`as_completed()` for progress** — get results as they finish, not in order.
6. **Always use locks for shared mutable state** — race conditions are real and hard to debug.
7. **`queue.Queue` is thread-safe** — the best way to pass data between threads.
8. **Guard with `if __name__ == "__main__"`** — required for multiprocessing on Windows/macOS.
9. **Multiprocessing data must be picklable** — no lambdas, no inner functions.
10. **For Data Engineering** — `ThreadPoolExecutor` for API calls and DB queries, `ProcessPoolExecutor` for data transformation and parsing.

---

> **Tomorrow (Day 12):** Async Python — `asyncio`, `async`/`await`, `aiohttp`, async generators. The most efficient way to handle thousands of concurrent I/O operations.

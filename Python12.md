# Day 12: Async Python — asyncio, async/await

## 🎯 Goal

Master Python's async programming model. You already know `async/await` from JavaScript — Python's version is conceptually identical but with different mechanics. Today you'll learn when async beats threading, how the event loop works, and patterns for high-concurrency I/O — essential for Data Engineering (async DB queries, parallel API calls, streaming pipelines).

---

## 1. Why Async?

### The problem async solves

```
You need to make 1,000 API calls. Each takes 0.5s of network waiting.

Sequential:       1,000 × 0.5s = 500 seconds 😴
Threading (100):  ~5 seconds, but 100 OS threads = heavy memory + GIL overhead
Async (1,000):    ~0.5 seconds, ONE thread, minimal memory 🚀

Async is the most efficient way to handle many I/O operations.
No threads, no processes, no GIL issues — just one thread switching
between tasks while they wait for I/O.
```

### Async vs Threading vs Multiprocessing — the final picture

```
┌───────────────────────────────────────────────────────────┐
│  I/O-bound: few tasks (<50)     → ThreadPoolExecutor      │
│  I/O-bound: many tasks (50+)    → asyncio ⭐              │
│  CPU-bound                      → ProcessPoolExecutor     │
│  Mixed I/O + CPU                → asyncio + ProcessPool   │
└───────────────────────────────────────────────────────────┘

Async shines when you have MANY concurrent I/O operations.
Threading is simpler when you have few tasks or need synchronous libraries.
```

---

## 2. Basics — async/await Syntax

### Your first coroutine

```python
import asyncio

# Define a coroutine with 'async def'
async def greet(name: str) -> str:
    print(f"Hello, {name}!")
    await asyncio.sleep(1)  # non-blocking sleep (yields control to event loop)
    print(f"Goodbye, {name}!")
    return f"greeted {name}"

# Run a coroutine
result = asyncio.run(greet("Alice"))
print(result)  # "greeted Alice"

# ⚠️ You CANNOT call a coroutine like a regular function:
# greet("Alice")  → returns a coroutine OBJECT, doesn't execute!
# You MUST await it or pass it to asyncio.run()
```

### JS comparison — almost identical

```javascript
// JavaScript
async function greet(name) {
    console.log(`Hello, ${name}!`);
    await new Promise(resolve => setTimeout(resolve, 1000));
    console.log(`Goodbye, ${name}!`);
    return `greeted ${name}`;
}
```

```python
# Python — same concept, slightly different syntax
async def greet(name: str) -> str:
    print(f"Hello, {name}!")
    await asyncio.sleep(1)  # Python's equivalent of setTimeout promise
    print(f"Goodbye, {name}!")
    return f"greeted {name}"
```

### Key differences from JS async

```
Python                              JavaScript
─────────────────────────────────  ─────────────────────────────────
asyncio.run(main())                Automatic (top-level await, or .then())
await asyncio.sleep(1)             await new Promise(r => setTimeout(r, 1000))
asyncio.gather(*tasks)             Promise.all([...tasks])
asyncio.wait(tasks)                Promise.allSettled([...tasks])
asyncio.create_task(coro())        No direct equivalent (promises start immediately)
async for item in aiter:           for await (const item of aiter)
async with ctx as x:               No equivalent (proposal stage)

Major difference:
  JS: Promises start executing immediately when created
  Python: Coroutines DON'T start until awaited or wrapped in create_task()
```

---

## 3. The Event Loop — How It Works

```
The event loop is a single-threaded scheduler that:
1. Picks a ready task
2. Runs it until it hits an 'await'
3. Suspends that task
4. Picks the next ready task
5. Repeat

It's like a juggler: keeps many balls in the air,
but only touches one at a time.

     Task A: ██░░░░░░██░░░░██████
     Task B: ░░██░░░░░░██░░░░░░░░
     Task C: ░░░░████░░░░██░░░░░░
              ─────── time ──────►
     
     █ = running    ░ = waiting for I/O (suspended)
     
     Only ONE task runs at any moment (single thread),
     but while one waits for I/O, others make progress.
```

```python
import asyncio

async def task(name: str, delay: float) -> str:
    print(f"[{name}] started")
    await asyncio.sleep(delay)  # ← yields control here
    print(f"[{name}] finished after {delay}s")
    return name

async def main():
    # Sequential — waits for each to finish
    result1 = await task("A", 2)
    result2 = await task("B", 1)
    # Total: 3 seconds (2 + 1)

    # Concurrent — runs both "at the same time"
    result1, result2 = await asyncio.gather(
        task("A", 2),
        task("B", 1),
    )
    # Total: 2 seconds (max of 2, 1)

asyncio.run(main())
```

---

## 4. Running Coroutines Concurrently

### `asyncio.gather` — run multiple coroutines, get all results

```python
import asyncio
import time

async def fetch_data(source: str, delay: float) -> dict:
    """Simulate fetching data from different sources."""
    print(f"Fetching from {source}...")
    await asyncio.sleep(delay)
    return {"source": source, "rows": int(delay * 1000)}

async def main():
    start = time.perf_counter()
    
    # All three run concurrently!
    results = await asyncio.gather(
        fetch_data("database", 2.0),
        fetch_data("api", 1.5),
        fetch_data("cache", 0.3),
    )
    
    elapsed = time.perf_counter() - start
    print(f"\nAll done in {elapsed:.1f}s")  # ~2.0s (not 3.8s!)
    for r in results:
        print(f"  {r['source']}: {r['rows']} rows")

asyncio.run(main())
```

### `asyncio.gather` with error handling

```python
async def risky_fetch(url: str) -> str:
    await asyncio.sleep(0.5)
    if "bad" in url:
        raise ConnectionError(f"Failed: {url}")
    return f"Data from {url}"

async def main():
    # return_exceptions=True — exceptions become results instead of propagating
    results = await asyncio.gather(
        risky_fetch("good_url_1"),
        risky_fetch("bad_url"),
        risky_fetch("good_url_2"),
        return_exceptions=True,
    )
    
    for result in results:
        if isinstance(result, Exception):
            print(f"❌ Error: {result}")
        else:
            print(f"✅ {result}")

asyncio.run(main())
# ✅ Data from good_url_1
# ❌ Error: Failed: bad_url
# ✅ Data from good_url_2
```

### `asyncio.create_task` — fire and forget (or collect later)

```python
async def background_job(name: str):
    await asyncio.sleep(2)
    print(f"Background job '{name}' complete")

async def main():
    # create_task starts the coroutine running in the background
    task1 = asyncio.create_task(background_job("sync_cache"))
    task2 = asyncio.create_task(background_job("send_email"))
    
    # Do other work while tasks run in background
    print("Doing main work...")
    await asyncio.sleep(1)
    print("Main work done")
    
    # Wait for background tasks to finish
    await task1
    await task2
    print("Everything done")

asyncio.run(main())
# Doing main work...
# Main work done
# Background job 'sync_cache' complete
# Background job 'send_email' complete
# Everything done
```

### `asyncio.TaskGroup` — structured concurrency (Python 3.11+)

```python
async def fetch(url: str) -> str:
    await asyncio.sleep(1)
    return f"Data from {url}"

async def main():
    # TaskGroup ensures all tasks complete (or all are cancelled on error)
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(fetch("url1"))
        task2 = tg.create_task(fetch("url2"))
        task3 = tg.create_task(fetch("url3"))
    
    # All tasks guaranteed to be done here
    print(task1.result(), task2.result(), task3.result())

    # If ANY task raises, ALL others are cancelled
    # This is safer than gather() — no leaked tasks

asyncio.run(main())
```

---

## 5. Controlling Concurrency

### Semaphore — limit concurrent operations

```python
import asyncio

async def fetch_url(sem: asyncio.Semaphore, url: str) -> dict:
    async with sem:  # at most N concurrent fetches
        print(f"Fetching {url}...")
        await asyncio.sleep(1)  # simulate request
        return {"url": url, "status": 200}

async def main():
    sem = asyncio.Semaphore(5)  # max 5 concurrent requests
    urls = [f"https://api.example.com/page/{i}" for i in range(20)]
    
    tasks = [fetch_url(sem, url) for url in urls]
    results = await asyncio.gather(*tasks)
    
    print(f"Fetched {len(results)} URLs")

asyncio.run(main())
# 20 URLs, but only 5 at a time, ~4 seconds total (not 20s)
```

### `asyncio.wait_for` — timeout

```python
async def slow_operation():
    await asyncio.sleep(10)
    return "done"

async def main():
    try:
        result = await asyncio.wait_for(slow_operation(), timeout=3.0)
    except asyncio.TimeoutError:
        print("Operation timed out after 3 seconds!")

asyncio.run(main())
```

### `asyncio.as_completed` — process results as they finish

```python
import asyncio
import random

async def fetch(url: str) -> dict:
    delay = random.uniform(0.5, 3.0)
    await asyncio.sleep(delay)
    return {"url": url, "time": round(delay, 2)}

async def main():
    urls = [f"url_{i}" for i in range(10)]
    tasks = [fetch(url) for url in urls]
    
    # Process results as they complete (fastest first)
    completed = 0
    for coro in asyncio.as_completed(tasks):
        result = await coro
        completed += 1
        print(f"[{completed}/10] Got: {result['url']} ({result['time']}s)")

asyncio.run(main())
```

---

## 6. Async Iteration & Context Managers

### Async iterators (`async for`)

```python
import asyncio

async def fetch_pages(total: int):
    """Async generator — yields pages one at a time."""
    for page in range(1, total + 1):
        await asyncio.sleep(0.3)  # simulate API call
        data = {"page": page, "items": [f"item_{i}" for i in range(10)]}
        yield data

async def main():
    all_items = []
    async for page_data in fetch_pages(5):
        print(f"Got page {page_data['page']}")
        all_items.extend(page_data["items"])
    
    print(f"Total items: {len(all_items)}")

asyncio.run(main())
```

### Async generator pipeline

```python
import asyncio
import json

async def read_lines(filepath: str):
    """Stage 1: Read lines lazily (simulate async file read)."""
    with open(filepath) as f:
        for line in f:
            await asyncio.sleep(0)  # yield to event loop
            yield line.strip()

async def parse_json(lines):
    """Stage 2: Parse JSON lines."""
    async for line in lines:
        if line:
            try:
                yield json.loads(line)
            except json.JSONDecodeError:
                continue

async def filter_events(records, event_type: str):
    """Stage 3: Filter by event type."""
    async for record in records:
        if record.get("event") == event_type:
            yield record

async def main():
    # Chain async generators — like Day 6 but async!
    lines = read_lines("events.jsonl")
    records = parse_json(lines)
    clicks = filter_events(records, "click")
    
    count = 0
    async for event in clicks:
        count += 1
    print(f"Found {count} click events")

asyncio.run(main())
```

### Async context managers (`async with`)

```python
import asyncio

class AsyncDBConnection:
    """Async context manager for database connections."""
    
    def __init__(self, url: str):
        self.url = url
        self.connection = None
    
    async def __aenter__(self):
        print(f"Connecting to {self.url}...")
        await asyncio.sleep(0.5)  # simulate connection
        self.connection = {"url": self.url, "connected": True}
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print(f"Closing connection to {self.url}")
        await asyncio.sleep(0.1)  # simulate cleanup
        self.connection = None
        return False
    
    async def query(self, sql: str) -> list[dict]:
        await asyncio.sleep(0.2)
        return [{"id": 1, "name": "Alice"}]

async def main():
    async with AsyncDBConnection("postgres://localhost/mydb") as db:
        users = await db.query("SELECT * FROM users")
        print(f"Got {len(users)} users")
    # Connection automatically closed

asyncio.run(main())
```

### Using `contextlib` for async context managers

```python
from contextlib import asynccontextmanager
import asyncio

@asynccontextmanager
async def managed_connection(url: str):
    """Async context manager using decorator (simpler than class)."""
    print(f"Connecting to {url}...")
    conn = await create_connection(url)
    try:
        yield conn
    finally:
        await conn.close()
        print("Connection closed")

# Usage
# async with managed_connection("postgres://...") as conn:
#     result = await conn.query("SELECT 1")
```

---

## 7. Mixing Async with Sync Code

### Running sync functions in async context

```python
import asyncio
import time

def blocking_io():
    """Regular sync function — blocks the event loop!"""
    time.sleep(2)
    return "data"

async def main():
    loop = asyncio.get_event_loop()
    
    # ❌ BAD — blocks the entire event loop for 2 seconds
    # result = blocking_io()
    
    # ✅ GOOD — run in thread pool (doesn't block event loop)
    result = await loop.run_in_executor(None, blocking_io)
    # None = default ThreadPoolExecutor
    
    print(result)

asyncio.run(main())
```

### Running CPU-bound work from async

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
import math

def cpu_heavy(n: int) -> int:
    """CPU-bound work — should run in process pool."""
    return len(str(math.factorial(n)))

async def main():
    loop = asyncio.get_event_loop()
    
    # Run CPU work in process pool from async code
    with ProcessPoolExecutor(max_workers=4) as pool:
        tasks = [
            loop.run_in_executor(pool, cpu_heavy, n)
            for n in [50000, 60000, 70000, 80000]
        ]
        results = await asyncio.gather(*tasks)
    
    for n, digits in zip([50000, 60000, 70000, 80000], results):
        print(f"{n}! has {digits:,} digits")

asyncio.run(main())
```

### Running async from sync code

```python
import asyncio

async def async_function() -> str:
    await asyncio.sleep(1)
    return "result"

# From sync main code:
result = asyncio.run(async_function())

# From inside a sync function that's called from async:
# This is tricky and should be avoided.
# If you need to call async from sync, restructure your code.
```

---

## 8. `aiohttp` — Async HTTP Client

```python
# pip install aiohttp
import asyncio
import aiohttp
import time

async def fetch_url(session: aiohttp.ClientSession, url: str) -> dict:
    """Fetch a single URL using shared session."""
    async with session.get(url) as response:
        data = await response.json()
        return {"url": url, "status": response.status, "data": data}

async def fetch_many(urls: list[str], max_concurrent: int = 10) -> list[dict]:
    """Fetch many URLs with concurrency limit."""
    sem = asyncio.Semaphore(max_concurrent)
    
    async def limited_fetch(session: aiohttp.ClientSession, url: str):
        async with sem:
            return await fetch_url(session, url)
    
    # aiohttp.ClientSession manages connection pooling
    async with aiohttp.ClientSession() as session:
        tasks = [limited_fetch(session, url) for url in urls]
        return await asyncio.gather(*tasks, return_exceptions=True)

async def main():
    urls = [f"https://jsonplaceholder.typicode.com/posts/{i}" for i in range(1, 51)]
    
    start = time.perf_counter()
    results = await fetch_many(urls, max_concurrent=10)
    elapsed = time.perf_counter() - start
    
    success = sum(1 for r in results if not isinstance(r, Exception))
    errors = sum(1 for r in results if isinstance(r, Exception))
    
    print(f"Fetched {len(urls)} URLs in {elapsed:.1f}s")
    print(f"Success: {success}, Errors: {errors}")

# asyncio.run(main())

# Compare:
# Sequential requests: 50 × 0.3s = 15 seconds
# Async with 10 concurrent: ~1.5 seconds (10x faster!)
```

### POST requests and error handling

```python
import asyncio
import aiohttp

async def post_data(session: aiohttp.ClientSession, url: str, data: dict) -> dict:
    """POST with timeout and error handling."""
    timeout = aiohttp.ClientTimeout(total=10)
    try:
        async with session.post(url, json=data, timeout=timeout) as response:
            if response.status >= 400:
                body = await response.text()
                return {"error": f"HTTP {response.status}: {body[:200]}"}
            return await response.json()
    except aiohttp.ClientError as e:
        return {"error": f"Connection error: {e}"}
    except asyncio.TimeoutError:
        return {"error": "Request timed out"}
```

---

## 9. Async Queues — Producer/Consumer Pattern

```python
import asyncio
import json
import random

async def producer(queue: asyncio.Queue, num_items: int):
    """Produce items and put them in the queue."""
    for i in range(num_items):
        await asyncio.sleep(random.uniform(0.01, 0.05))  # simulate work
        item = {"id": i, "value": random.random()}
        await queue.put(item)
        if (i + 1) % 100 == 0:
            print(f"Produced {i + 1} items (queue size: {queue.qsize()})")
    
    # Signal completion
    await queue.put(None)

async def consumer(queue: asyncio.Queue, consumer_id: int, results: list):
    """Consume items from the queue."""
    count = 0
    while True:
        item = await queue.get()
        if item is None:
            await queue.put(None)  # pass sentinel to next consumer
            break
        
        # Process item
        await asyncio.sleep(0.01)
        item["processed_by"] = consumer_id
        results.append(item)
        count += 1
        queue.task_done()
    
    print(f"Consumer {consumer_id} processed {count} items")

async def main():
    queue: asyncio.Queue = asyncio.Queue(maxsize=50)  # backpressure!
    results: list = []
    
    # Start producer and multiple consumers
    producer_task = asyncio.create_task(producer(queue, 500))
    consumer_tasks = [
        asyncio.create_task(consumer(queue, i, results))
        for i in range(4)
    ]
    
    # Wait for everything
    await producer_task
    await asyncio.gather(*consumer_tasks)
    
    print(f"\nTotal processed: {len(results)}")

asyncio.run(main())
```

---

## 10. Real-World Data Engineering Patterns

### Pattern 1: Async ETL — parallel extract from multiple sources

```python
import asyncio
import json
from dataclasses import dataclass

@dataclass
class ExtractResult:
    source: str
    records: list[dict]
    error: str | None = None

async def extract_from_api(url: str) -> ExtractResult:
    """Extract data from REST API."""
    await asyncio.sleep(1)  # simulate API call
    records = [{"id": i, "source": "api"} for i in range(100)]
    return ExtractResult(source=url, records=records)

async def extract_from_db(connection_str: str) -> ExtractResult:
    """Extract data from database."""
    await asyncio.sleep(2)  # simulate DB query
    records = [{"id": i, "source": "db"} for i in range(500)]
    return ExtractResult(source=connection_str, records=records)

async def extract_from_s3(bucket: str) -> ExtractResult:
    """Extract data from S3."""
    await asyncio.sleep(1.5)  # simulate S3 download
    records = [{"id": i, "source": "s3"} for i in range(300)]
    return ExtractResult(source=bucket, records=records)

async def run_extraction():
    """Extract from ALL sources concurrently."""
    results = await asyncio.gather(
        extract_from_api("https://api.example.com/data"),
        extract_from_db("postgres://localhost/analytics"),
        extract_from_s3("s3://data-lake/raw/"),
        return_exceptions=True,
    )
    
    all_records = []
    for result in results:
        if isinstance(result, Exception):
            print(f"❌ Extraction failed: {result}")
        else:
            print(f"✅ {result.source}: {len(result.records)} records")
            all_records.extend(result.records)
    
    print(f"\nTotal: {len(all_records)} records from {len(results)} sources")
    return all_records

# asyncio.run(run_extraction())
# All three extractions happen concurrently!
# Total time: ~2s (max of 1, 2, 1.5) instead of 4.5s sequential
```

### Pattern 2: Batch API ingestion with retries

```python
import asyncio
import random

class AsyncAPIClient:
    """Async API client with concurrency control and retries."""
    
    def __init__(self, max_concurrent: int = 10, max_retries: int = 3):
        self.sem = asyncio.Semaphore(max_concurrent)
        self.max_retries = max_retries
    
    async def fetch_one(self, endpoint: str) -> dict:
        """Fetch with retry logic."""
        async with self.sem:
            last_error = None
            for attempt in range(1, self.max_retries + 1):
                try:
                    await asyncio.sleep(random.uniform(0.05, 0.2))
                    if random.random() < 0.2:
                        raise ConnectionError("Server unavailable")
                    return {"endpoint": endpoint, "data": "ok", "attempt": attempt}
                except ConnectionError as e:
                    last_error = e
                    wait = 2 ** (attempt - 1) * 0.1  # exponential backoff
                    await asyncio.sleep(wait)
            
            return {"endpoint": endpoint, "error": str(last_error)}
    
    async def fetch_many(self, endpoints: list[str]) -> list[dict]:
        """Fetch all endpoints concurrently."""
        tasks = [self.fetch_one(ep) for ep in endpoints]
        return await asyncio.gather(*tasks)

async def main():
    client = AsyncAPIClient(max_concurrent=20, max_retries=3)
    endpoints = [f"/api/records/{i}" for i in range(100)]
    
    results = await client.fetch_many(endpoints)
    
    success = [r for r in results if "error" not in r]
    errors = [r for r in results if "error" in r]
    
    print(f"Success: {len(success)}, Failed: {len(errors)}")
    retried = [r for r in success if r["attempt"] > 1]
    print(f"Succeeded after retry: {len(retried)}")

asyncio.run(main())
```

### Pattern 3: Async database operations

```python
import asyncio

# In real code you'd use: asyncpg, aiosqlite, or SQLAlchemy async
# This simulates the patterns

class AsyncDB:
    """Simulated async database connection."""
    
    async def execute(self, query: str, params: tuple = ()) -> list[dict]:
        await asyncio.sleep(0.05)  # simulate query
        return [{"id": 1}]
    
    async def execute_many(self, query: str, params_list: list[tuple]) -> int:
        await asyncio.sleep(0.1)
        return len(params_list)

async def parallel_queries(db: AsyncDB):
    """Run independent queries in parallel."""
    users, orders, products = await asyncio.gather(
        db.execute("SELECT * FROM users"),
        db.execute("SELECT * FROM orders WHERE date > ?", ("2025-01-01",)),
        db.execute("SELECT * FROM products WHERE active = ?", (True,)),
    )
    return {"users": users, "orders": orders, "products": products}

async def batch_insert(db: AsyncDB, records: list[dict], batch_size: int = 1000):
    """Insert records in parallel batches."""
    batches = [records[i:i+batch_size] for i in range(0, len(records), batch_size)]
    
    async def insert_batch(batch: list[dict]) -> int:
        params = [(r["name"], r["value"]) for r in batch]
        return await db.execute_many("INSERT INTO data VALUES (?, ?)", params)
    
    results = await asyncio.gather(*[insert_batch(b) for b in batches])
    total = sum(results)
    print(f"Inserted {total} records in {len(batches)} batches")
```

---

## 11. Common Mistakes

### Mistake 1: Forgetting to await

```python
# ❌ Common mistake — calling coroutine without await
async def main():
    asyncio.sleep(1)  # Returns coroutine object, does NOT sleep!
    result = fetch_data()  # Returns coroutine, does NOT execute!
    
    # Python warns: "coroutine 'sleep' was never awaited"

# ✅ Always await coroutines
async def main():
    await asyncio.sleep(1)
    result = await fetch_data()
```

### Mistake 2: Blocking the event loop

```python
import time

# ❌ time.sleep() blocks the ENTIRE event loop
async def bad():
    time.sleep(5)  # nothing else can run during this!

# ✅ Use asyncio.sleep()
async def good():
    await asyncio.sleep(5)  # other tasks can run

# ❌ Sync HTTP library blocks
# requests.get("https://example.com")  # blocks event loop

# ✅ Use async HTTP library
# async with aiohttp.ClientSession() as s:
#     await s.get("https://example.com")

# ❌ Sync file read blocks
# data = open("big_file.csv").read()  # blocks event loop

# ✅ Run in executor
# data = await loop.run_in_executor(None, open("big_file.csv").read)
# Or use aiofiles library
```

### Mistake 3: Creating too many tasks without limits

```python
# ❌ 10,000 concurrent requests = overwhelm server + your system
urls = [f"https://api.com/{i}" for i in range(10_000)]
tasks = [fetch(url) for url in urls]
results = await asyncio.gather(*tasks)  # ALL at once!

# ✅ Use semaphore to limit concurrency
sem = asyncio.Semaphore(50)  # max 50 at a time
async def limited_fetch(url):
    async with sem:
        return await fetch(url)

tasks = [limited_fetch(url) for url in urls]
results = await asyncio.gather(*tasks)
```

### Mistake 4: Ignoring task exceptions

```python
# ❌ Exception silently swallowed
async def main():
    task = asyncio.create_task(failing_coroutine())
    await asyncio.sleep(10)
    # task's exception is never retrieved → warning

# ✅ Always await or handle task results
async def main():
    task = asyncio.create_task(failing_coroutine())
    try:
        await task
    except Exception as e:
        print(f"Task failed: {e}")
```

---

## 12. Async Libraries Ecosystem

```
HTTP Clients:
  aiohttp          — most popular async HTTP client/server
  httpx             — modern, supports both sync and async (recommended!)

Databases:
  asyncpg           — fastest PostgreSQL driver
  aiosqlite         — SQLite async wrapper
  motor             — MongoDB async driver
  aiomysql          — MySQL async driver
  SQLAlchemy 2.0    — ORM with async support

Files:
  aiofiles          — async file operations

Web Frameworks:
  FastAPI           — built on async (you'll learn this!)
  Starlette         — ASGI framework FastAPI is built on

Task Queues:
  arq               — async task queue (like Celery but async)
  
Redis:
  aioredis / redis.asyncio — async Redis

For Data Engineering:
  httpx or aiohttp  — async API calls
  asyncpg           — async PostgreSQL
  aiokafka          — async Kafka producer/consumer
```

---

## 📝 Practice Tasks

### Task 1: Parallel Page Fetcher
Simulate fetching 50 web pages. Each page takes 0.2-1.0s (random). Implement with `asyncio.gather` + semaphore (max 10 concurrent). Show progress. Compare time with a sequential version.

### Task 2: Async Retry Decorator
Write an async version of the retry decorator from Day 7:
```python
@async_retry(max_attempts=3, delay=1.0, backoff=2.0)
async def fetch_data(url: str) -> dict:
    ...
```
Support exponential backoff and specific exception types.

### Task 3: Async Producer-Consumer
Build an async pipeline:
- Producer: generates 1000 "events" with random delays
- 4 consumers: process events (validate, transform)
- Writer: collects results and writes to JSONL
Use `asyncio.Queue` with backpressure (maxsize). Print throughput stats.

### Task 4: Concurrent Data Validator
Given a list of 200 records, validate each by "calling" an async validation service (simulated with sleep). Limit to 20 concurrent validations. Collect valid/invalid separately. Show progress percentage.

### Task 5: Mixed Async + Sync Pipeline
Build a pipeline that:
1. Fetches data from 3 async "APIs" concurrently
2. Processes the combined data with CPU-heavy sync function (run in executor)
3. Writes results to file asynchronously
Show that the I/O and CPU parts overlap correctly.

### Task 6: Async Timeout and Cancellation
Write a function that fetches from 5 sources with individual timeouts (2s each). If any source takes too long, cancel it and continue with the others. Report which sources responded and which timed out.

---

## 📚 Resources

- [Python Official — asyncio](https://docs.python.org/3/library/asyncio.html)
- [Real Python — Async IO](https://realpython.com/async-io-python/)
- [Real Python — asyncio Walkthrough](https://realpython.com/python-async-features/)
- [aiohttp Documentation](https://docs.aiohttp.org/)
- [httpx Documentation](https://www.python-httpx.org/)
- [asyncpg Documentation](https://magicstack.github.io/asyncpg/)
- [FastAPI — Async](https://fastapi.tiangolo.com/async/)
- [PEP 492 — Coroutines with async and await](https://peps.python.org/pep-0492/)

---

## 🔑 Key Takeaways

1. **`async/await` is like JS** — same concept, almost same syntax. You'll feel at home.
2. **Coroutines don't run until awaited** — unlike JS Promises that start immediately.
3. **`asyncio.gather`** = `Promise.all` — run coroutines concurrently, get all results.
4. **`asyncio.Semaphore`** — ALWAYS limit concurrency. Don't fire 10,000 tasks at once.
5. **Never block the event loop** — no `time.sleep()`, no `requests.get()`, no sync file I/O.
6. **Use `run_in_executor`** for unavoidable sync code — wraps it in a thread.
7. **`TaskGroup` (3.11+)** is safer than `gather` — automatic cancellation on errors.
8. **Async shines at scale** — 50+ concurrent I/O operations is where it beats threading.
9. **`httpx`** over `aiohttp` for new projects — supports both sync and async modes.
10. **For DE** — async for API ingestion, async DB queries, streaming pipelines. But Spark/Airflow handle most heavy lifting.

---

> **Tomorrow (Day 13):** Performance & Profiling — `cProfile`, `timeit`, memory profiling, `collections` module, optimization patterns. How to find and fix bottlenecks.

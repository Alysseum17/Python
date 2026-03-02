# Day 15: SQL Fundamentals with Python (sqlite3)

## 🎯 Goal

Master SQL through Python's built-in `sqlite3` module. You'll learn SQL from Data Engineering perspective — not just CRUD, but analytical queries, window functions, CTEs, and patterns you'll use daily with PostgreSQL, BigQuery, Snowflake, etc. SQLite is the perfect learning tool: zero setup, same SQL concepts, included with Python.

---

## 1. SQLite + Python — Setup

### Connecting and basic operations

```python
import sqlite3

# Connect (creates file if doesn't exist)
conn = sqlite3.connect("mydb.db")

# In-memory database (great for testing, temp data)
conn = sqlite3.connect(":memory:")

# Get a cursor (executes SQL)
cursor = conn.cursor()

# Execute SQL
cursor.execute("SELECT 1 + 1")
result = cursor.fetchone()
print(result)  # (2,)

# Always close when done
conn.close()

# ✅ Best practice — use context manager
with sqlite3.connect(":memory:") as conn:
    cursor = conn.cursor()
    cursor.execute("SELECT 'hello'")
    print(cursor.fetchone())
# conn.close() is NOT called by context manager!
# But conn.commit() IS called on success, rollback on exception

# For guaranteed close:
conn = sqlite3.connect(":memory:")
try:
    # work
    conn.commit()
finally:
    conn.close()
```

### Row factories — get dicts instead of tuples

```python
import sqlite3

conn = sqlite3.connect(":memory:")

# Default: rows are tuples
# cursor.fetchone()  → (1, "Alice", 25)

# sqlite3.Row — access by name AND index
conn.row_factory = sqlite3.Row
cursor = conn.cursor()
cursor.execute("SELECT 1 as id, 'Alice' as name")
row = cursor.fetchone()
print(row["id"])      # 1
print(row["name"])    # "Alice"
print(row[0])         # 1 (still works by index)
print(dict(row))      # {"id": 1, "name": "Alice"}

# Custom factory — plain dicts
def dict_factory(cursor, row):
    return {col[0]: value for col, value in zip(cursor.description, row)}

conn.row_factory = dict_factory
cursor = conn.cursor()
cursor.execute("SELECT 1 as id, 'Alice' as name")
row = cursor.fetchone()
print(row)  # {"id": 1, "name": "Alice"}
```

---

## 2. DDL — Creating Tables

### Data types in SQLite

```
SQLite has flexible typing (unlike PostgreSQL/MySQL):
  INTEGER  — whole numbers
  REAL     — floating point
  TEXT     — strings
  BLOB     — binary data
  NULL     — null value

SQLite is "type-affinity" — it SUGGESTS types but doesn't strictly enforce.
PostgreSQL, MySQL, BigQuery etc. are strictly typed.
For learning SQL concepts, this doesn't matter.
```

### CREATE TABLE

```python
import sqlite3

conn = sqlite3.connect(":memory:")
cursor = conn.cursor()

# Basic table
cursor.execute("""
    CREATE TABLE users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        age INTEGER CHECK(age >= 0 AND age <= 150),
        city TEXT DEFAULT 'Unknown',
        is_active INTEGER DEFAULT 1,
        created_at TEXT DEFAULT (datetime('now'))
    )
""")

# Table with foreign key
cursor.execute("""
    CREATE TABLE orders (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        product TEXT NOT NULL,
        amount REAL NOT NULL CHECK(amount > 0),
        quantity INTEGER NOT NULL DEFAULT 1,
        status TEXT DEFAULT 'pending' CHECK(status IN ('pending', 'shipped', 'delivered', 'cancelled')),
        ordered_at TEXT DEFAULT (datetime('now')),
        FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
    )
""")

# Enable foreign key enforcement (off by default in SQLite!)
cursor.execute("PRAGMA foreign_keys = ON")

conn.commit()

# CREATE TABLE IF NOT EXISTS — don't error if table already exists
cursor.execute("""
    CREATE TABLE IF NOT EXISTS logs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        level TEXT NOT NULL,
        message TEXT,
        timestamp TEXT DEFAULT (datetime('now'))
    )
""")
```

### Table modifications

```python
# Add column
cursor.execute("ALTER TABLE users ADD COLUMN phone TEXT")

# Rename table
cursor.execute("ALTER TABLE logs RENAME TO audit_logs")

# Drop table
cursor.execute("DROP TABLE IF EXISTS audit_logs")

# Create index (speeds up queries on that column)
cursor.execute("CREATE INDEX idx_users_email ON users(email)")
cursor.execute("CREATE INDEX idx_orders_user_id ON orders(user_id)")
cursor.execute("CREATE INDEX idx_orders_status ON orders(status)")

# Composite index
cursor.execute("CREATE INDEX idx_orders_user_status ON orders(user_id, status)")
```

---

## 3. DML — Insert, Update, Delete

### INSERT

```python
# Single insert
cursor.execute(
    "INSERT INTO users (name, email, age, city) VALUES (?, ?, ?, ?)",
    ("Alice", "alice@example.com", 25, "Kyiv")
)

# ⚠️ ALWAYS use parameterized queries (?) — NEVER string formatting!
# ❌ NEVER DO THIS — SQL injection vulnerability!
# cursor.execute(f"INSERT INTO users (name) VALUES ('{user_input}')")
# If user_input = "'; DROP TABLE users; --"  → your table is gone!

# ✅ ALWAYS parameterized
cursor.execute("INSERT INTO users (name) VALUES (?)", (user_input,))

# Get ID of last inserted row
cursor.execute(
    "INSERT INTO users (name, email, age) VALUES (?, ?, ?)",
    ("Bob", "bob@example.com", 30)
)
new_id = cursor.lastrowid
print(f"New user ID: {new_id}")

# Bulk insert — executemany (much faster than loop)
users_data = [
    ("Charlie", "charlie@example.com", 35, "NYC"),
    ("Diana", "diana@example.com", 28, "Berlin"),
    ("Eve", "eve@example.com", 32, "Kyiv"),
    ("Frank", "frank@example.com", 27, "Paris"),
    ("Grace", "grace@example.com", 29, "Tokyo"),
]
cursor.executemany(
    "INSERT INTO users (name, email, age, city) VALUES (?, ?, ?, ?)",
    users_data,
)
conn.commit()  # don't forget!

# INSERT OR IGNORE — skip if unique constraint violated
cursor.execute(
    "INSERT OR IGNORE INTO users (name, email, age) VALUES (?, ?, ?)",
    ("Alice", "alice@example.com", 25)  # duplicate email — skipped
)

# INSERT OR REPLACE — upsert (replace if exists)
cursor.execute(
    "INSERT OR REPLACE INTO users (id, name, email, age) VALUES (?, ?, ?, ?)",
    (1, "Alice Updated", "alice@example.com", 26)
)
```

### UPDATE

```python
# Update specific rows
cursor.execute(
    "UPDATE users SET age = ?, city = ? WHERE name = ?",
    (26, "London", "Alice")
)

# Update with expression
cursor.execute("UPDATE users SET age = age + 1 WHERE city = 'Kyiv'")

# Check how many rows were affected
cursor.execute("UPDATE users SET is_active = 0 WHERE age < 25")
print(f"Deactivated {cursor.rowcount} users")

# ⚠️ UPDATE without WHERE affects ALL rows!
# cursor.execute("UPDATE users SET is_active = 0")  # EVERYONE deactivated!
```

### DELETE

```python
# Delete specific rows
cursor.execute("DELETE FROM users WHERE is_active = 0")
print(f"Deleted {cursor.rowcount} users")

# Delete with subquery
cursor.execute("""
    DELETE FROM orders 
    WHERE user_id NOT IN (SELECT id FROM users)
""")

# ⚠️ DELETE without WHERE deletes ALL rows!
# cursor.execute("DELETE FROM users")  # EVERYTHING gone!

# Truncate-like (delete all, reset autoincrement)
cursor.execute("DELETE FROM logs")
cursor.execute("DELETE FROM sqlite_sequence WHERE name = 'logs'")
```

---

## 4. SELECT — Querying Data

### Basic queries

```python
# Select all
cursor.execute("SELECT * FROM users")
all_users = cursor.fetchall()  # list of all rows

# Select specific columns
cursor.execute("SELECT name, age FROM users")

# Fetch methods
cursor.fetchone()    # single row (or None)
cursor.fetchall()    # all remaining rows (list)
cursor.fetchmany(5)  # next 5 rows

# ✅ Iterate directly (memory efficient for large results)
cursor.execute("SELECT * FROM users")
for row in cursor:
    print(row)
```

### WHERE — filtering

```python
# Comparison operators
cursor.execute("SELECT * FROM users WHERE age > ?", (30,))
cursor.execute("SELECT * FROM users WHERE city = ?", ("Kyiv",))
cursor.execute("SELECT * FROM users WHERE age BETWEEN ? AND ?", (25, 35))
cursor.execute("SELECT * FROM users WHERE city IN (?, ?, ?)", ("Kyiv", "London", "NYC"))
cursor.execute("SELECT * FROM users WHERE name LIKE ?", ("A%",))  # starts with A
cursor.execute("SELECT * FROM users WHERE email LIKE ?", ("%@example.com",))
cursor.execute("SELECT * FROM users WHERE phone IS NULL")
cursor.execute("SELECT * FROM users WHERE phone IS NOT NULL")

# Logical operators
cursor.execute("""
    SELECT * FROM users 
    WHERE age >= 25 
      AND city = 'Kyiv' 
      AND is_active = 1
""")

cursor.execute("""
    SELECT * FROM users 
    WHERE city = 'Kyiv' OR city = 'London'
""")

cursor.execute("""
    SELECT * FROM users 
    WHERE NOT (age < 25 OR is_active = 0)
""")
```

### ORDER BY, LIMIT, OFFSET

```python
# Sort
cursor.execute("SELECT * FROM users ORDER BY age ASC")
cursor.execute("SELECT * FROM users ORDER BY age DESC")
cursor.execute("SELECT * FROM users ORDER BY city ASC, age DESC")

# Limit results
cursor.execute("SELECT * FROM users ORDER BY age DESC LIMIT 5")

# Pagination
page = 2
page_size = 10
offset = (page - 1) * page_size
cursor.execute(
    "SELECT * FROM users ORDER BY id LIMIT ? OFFSET ?",
    (page_size, offset)
)
```

### DISTINCT

```python
# Unique values
cursor.execute("SELECT DISTINCT city FROM users ORDER BY city")
cities = [row[0] for row in cursor.fetchall()]

# Count distinct
cursor.execute("SELECT COUNT(DISTINCT city) FROM users")
```

---

## 5. Aggregate Functions & GROUP BY

```python
# Basic aggregates
cursor.execute("SELECT COUNT(*) FROM users")
cursor.execute("SELECT AVG(age) FROM users")
cursor.execute("SELECT MIN(age), MAX(age) FROM users")
cursor.execute("SELECT SUM(amount) FROM orders")

# GROUP BY — aggregate per group
cursor.execute("""
    SELECT city, COUNT(*) as user_count, AVG(age) as avg_age
    FROM users
    GROUP BY city
    ORDER BY user_count DESC
""")
for row in cursor:
    print(f"{row['city']}: {row['user_count']} users, avg age {row['avg_age']:.1f}")

# HAVING — filter AFTER grouping (WHERE filters BEFORE)
cursor.execute("""
    SELECT city, COUNT(*) as user_count
    FROM users
    GROUP BY city
    HAVING user_count >= 2
    ORDER BY user_count DESC
""")

# Multiple GROUP BY columns
cursor.execute("""
    SELECT city, status, COUNT(*) as order_count, SUM(amount) as total
    FROM orders o
    JOIN users u ON o.user_id = u.id
    GROUP BY city, status
    ORDER BY city, total DESC
""")
```

### Aggregate with CASE

```python
# Conditional counting
cursor.execute("""
    SELECT 
        COUNT(*) as total_users,
        SUM(CASE WHEN is_active = 1 THEN 1 ELSE 0 END) as active_users,
        SUM(CASE WHEN is_active = 0 THEN 1 ELSE 0 END) as inactive_users,
        ROUND(AVG(CASE WHEN city = 'Kyiv' THEN age END), 1) as kyiv_avg_age
    FROM users
""")
```

---

## 6. JOINs

```python
# Let's add some orders first
orders_data = [
    (1, "Laptop", 999.99, 1, "delivered"),
    (1, "Mouse", 29.99, 2, "delivered"),
    (2, "Keyboard", 79.99, 1, "shipped"),
    (3, "Monitor", 349.99, 1, "pending"),
    (3, "Cable", 9.99, 3, "delivered"),
    (5, "Headphones", 199.99, 1, "cancelled"),
]
cursor.executemany(
    "INSERT INTO orders (user_id, product, amount, quantity, status) VALUES (?, ?, ?, ?, ?)",
    orders_data,
)
conn.commit()

# ── INNER JOIN — only matching rows from both tables ──
cursor.execute("""
    SELECT u.name, o.product, o.amount
    FROM users u
    INNER JOIN orders o ON u.id = o.user_id
    ORDER BY u.name
""")
# Only users who HAVE orders

# ── LEFT JOIN — all rows from left table, matching from right ──
cursor.execute("""
    SELECT u.name, COUNT(o.id) as order_count, COALESCE(SUM(o.amount), 0) as total_spent
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    GROUP BY u.id, u.name
    ORDER BY total_spent DESC
""")
# ALL users, even those with 0 orders

# COALESCE — return first non-null value (like ?? in JS, or Optional.orElse in Java)

# ── Self JOIN — compare rows within same table ──
cursor.execute("""
    SELECT u1.name as user1, u2.name as user2
    FROM users u1
    INNER JOIN users u2 ON u1.city = u2.city AND u1.id < u2.id
    ORDER BY u1.city
""")
# Find pairs of users from the same city

# ── CROSS JOIN — every combination (cartesian product) ──
cursor.execute("""
    SELECT u.name, p.product
    FROM users u
    CROSS JOIN (SELECT DISTINCT product FROM orders) p
    LIMIT 20
""")
```

### JOIN visualization

```
INNER JOIN:     Only overlapping records (A ∩ B)
LEFT JOIN:      All from A + matching from B (A, with B where available)
RIGHT JOIN:     All from B + matching from A (not supported in SQLite)
FULL OUTER JOIN: All from both (not supported in SQLite, emulate with UNION)

Venn diagram:
  ┌──────────────┐
  │  A    ┌──┐   │
  │       │AB│ B  │
  │       └──┘   │
  └──────────────┘
  
  INNER = AB only
  LEFT  = A + AB
  RIGHT = AB + B
  FULL  = A + AB + B
```

---

## 7. Subqueries

```python
# Scalar subquery (returns single value)
cursor.execute("""
    SELECT name, age
    FROM users
    WHERE age > (SELECT AVG(age) FROM users)
""")
# Users older than average

# IN subquery (returns list of values)
cursor.execute("""
    SELECT name
    FROM users
    WHERE id IN (
        SELECT DISTINCT user_id FROM orders WHERE status = 'delivered'
    )
""")
# Users who have at least one delivered order

# NOT IN
cursor.execute("""
    SELECT name
    FROM users
    WHERE id NOT IN (SELECT DISTINCT user_id FROM orders)
""")
# Users with NO orders

# EXISTS (often faster than IN for large datasets)
cursor.execute("""
    SELECT u.name
    FROM users u
    WHERE EXISTS (
        SELECT 1 FROM orders o 
        WHERE o.user_id = u.id AND o.amount > 100
    )
""")
# Users with at least one order over $100

# Subquery in FROM (derived table)
cursor.execute("""
    SELECT city, avg_age
    FROM (
        SELECT city, AVG(age) as avg_age, COUNT(*) as cnt
        FROM users
        GROUP BY city
    ) city_stats
    WHERE cnt >= 2
""")
```

---

## 8. Common Table Expressions (CTEs) — WITH clause

CTEs make complex queries readable. Essential for Data Engineering.

```python
# Basic CTE
cursor.execute("""
    WITH active_users AS (
        SELECT id, name, age, city
        FROM users
        WHERE is_active = 1
    )
    SELECT city, COUNT(*) as count, AVG(age) as avg_age
    FROM active_users
    GROUP BY city
""")

# Multiple CTEs — build up step by step
cursor.execute("""
    WITH 
    user_orders AS (
        -- Step 1: join users with their orders
        SELECT 
            u.id,
            u.name,
            u.city,
            o.amount,
            o.status
        FROM users u
        LEFT JOIN orders o ON u.id = o.user_id
    ),
    user_stats AS (
        -- Step 2: aggregate per user
        SELECT
            id,
            name,
            city,
            COUNT(amount) as order_count,
            COALESCE(SUM(amount), 0) as total_spent,
            COALESCE(AVG(amount), 0) as avg_order
        FROM user_orders
        GROUP BY id, name, city
    ),
    city_stats AS (
        -- Step 3: aggregate per city
        SELECT
            city,
            COUNT(*) as user_count,
            SUM(order_count) as total_orders,
            SUM(total_spent) as city_revenue,
            AVG(total_spent) as avg_user_spent
        FROM user_stats
        GROUP BY city
    )
    -- Final: display results
    SELECT 
        city,
        user_count,
        total_orders,
        ROUND(city_revenue, 2) as revenue,
        ROUND(avg_user_spent, 2) as avg_spent_per_user
    FROM city_stats
    ORDER BY revenue DESC
""")

# CTEs vs subqueries:
# Subqueries can be nested and hard to read
# CTEs are linear and read top-to-bottom (like a pipeline!)
# CTEs can be referenced multiple times in the same query
```

### Recursive CTE (bonus — useful for hierarchical data)

```python
# Create hierarchical data (employees and managers)
cursor.execute("""
    CREATE TABLE employees (
        id INTEGER PRIMARY KEY,
        name TEXT,
        manager_id INTEGER REFERENCES employees(id)
    )
""")
cursor.executemany("INSERT INTO employees VALUES (?, ?, ?)", [
    (1, "CEO Alice", None),
    (2, "VP Bob", 1),
    (3, "VP Charlie", 1),
    (4, "Manager Diana", 2),
    (5, "Manager Eve", 2),
    (6, "Dev Frank", 4),
    (7, "Dev Grace", 4),
])
conn.commit()

# Recursive CTE — traverse the org chart
cursor.execute("""
    WITH RECURSIVE org_chart AS (
        -- Base case: start with CEO (no manager)
        SELECT id, name, manager_id, 0 as level, name as path
        FROM employees
        WHERE manager_id IS NULL
        
        UNION ALL
        
        -- Recursive case: find direct reports
        SELECT e.id, e.name, e.manager_id, oc.level + 1,
               oc.path || ' > ' || e.name
        FROM employees e
        INNER JOIN org_chart oc ON e.manager_id = oc.id
    )
    SELECT level, name, path
    FROM org_chart
    ORDER BY path
""")
for row in cursor:
    indent = "  " * row["level"]
    print(f"{indent}{row['name']}")

# CEO Alice
#   VP Bob
#     Manager Diana
#       Dev Frank
#       Dev Grace
#     Manager Eve
#   VP Charlie
```

---

## 9. Window Functions ⭐

Window functions are THE most important SQL feature for Data Engineering. They compute values across "windows" of rows without collapsing the result (unlike GROUP BY).

```python
# First, let's create a sales table with more data
cursor.execute("""
    CREATE TABLE sales (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        salesperson TEXT,
        region TEXT,
        amount REAL,
        sale_date TEXT
    )
""")
sales_data = [
    ("Alice", "East", 500, "2025-01-05"),
    ("Alice", "East", 300, "2025-01-12"),
    ("Alice", "East", 700, "2025-02-03"),
    ("Bob", "East", 400, "2025-01-08"),
    ("Bob", "East", 600, "2025-01-22"),
    ("Bob", "East", 200, "2025-02-10"),
    ("Charlie", "West", 800, "2025-01-03"),
    ("Charlie", "West", 350, "2025-01-18"),
    ("Charlie", "West", 550, "2025-02-07"),
    ("Diana", "West", 450, "2025-01-10"),
    ("Diana", "West", 900, "2025-01-25"),
    ("Diana", "West", 150, "2025-02-15"),
]
cursor.executemany(
    "INSERT INTO sales (salesperson, region, amount, sale_date) VALUES (?, ?, ?, ?)",
    sales_data,
)
conn.commit()
```

### ROW_NUMBER, RANK, DENSE_RANK

```python
# ROW_NUMBER — unique sequential number within partition
cursor.execute("""
    SELECT 
        salesperson,
        amount,
        sale_date,
        ROW_NUMBER() OVER (
            PARTITION BY salesperson 
            ORDER BY amount DESC
        ) as rank_within_person
    FROM sales
""")
# Each salesperson gets their sales ranked 1, 2, 3...

# RANK — same value gets same rank, with gaps
# DENSE_RANK — same value gets same rank, no gaps
cursor.execute("""
    SELECT 
        salesperson,
        SUM(amount) as total_sales,
        RANK() OVER (ORDER BY SUM(amount) DESC) as rank,
        DENSE_RANK() OVER (ORDER BY SUM(amount) DESC) as dense_rank
    FROM sales
    GROUP BY salesperson
""")

# Top N per group pattern (very common in DE!)
# "Find the top 2 sales per salesperson"
cursor.execute("""
    WITH ranked_sales AS (
        SELECT 
            *,
            ROW_NUMBER() OVER (
                PARTITION BY salesperson 
                ORDER BY amount DESC
            ) as rn
        FROM sales
    )
    SELECT salesperson, amount, sale_date
    FROM ranked_sales
    WHERE rn <= 2
    ORDER BY salesperson, amount DESC
""")
```

### Running totals and moving averages

```python
# Running total (cumulative sum)
cursor.execute("""
    SELECT 
        sale_date,
        salesperson,
        amount,
        SUM(amount) OVER (
            PARTITION BY salesperson 
            ORDER BY sale_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) as running_total
    FROM sales
    ORDER BY salesperson, sale_date
""")

# Moving average (3-row window)
cursor.execute("""
    SELECT 
        sale_date,
        amount,
        ROUND(AVG(amount) OVER (
            ORDER BY sale_date
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ), 2) as moving_avg_3
    FROM sales
    ORDER BY sale_date
""")
```

### LAG and LEAD — access previous/next rows

```python
# Compare each sale with the previous one
cursor.execute("""
    SELECT 
        salesperson,
        sale_date,
        amount,
        LAG(amount) OVER (
            PARTITION BY salesperson ORDER BY sale_date
        ) as prev_amount,
        amount - LAG(amount) OVER (
            PARTITION BY salesperson ORDER BY sale_date
        ) as change
    FROM sales
    ORDER BY salesperson, sale_date
""")
# LAG(amount, 1) = previous row's amount
# LAG(amount, 2) = two rows back
# LEAD(amount)   = next row's amount

# Month-over-month comparison
cursor.execute("""
    WITH monthly AS (
        SELECT 
            strftime('%Y-%m', sale_date) as month,
            SUM(amount) as revenue
        FROM sales
        GROUP BY month
    )
    SELECT 
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) as prev_month,
        ROUND(
            (revenue - LAG(revenue) OVER (ORDER BY month)) * 100.0 
            / LAG(revenue) OVER (ORDER BY month), 
            1
        ) as pct_change
    FROM monthly
""")
```

### FIRST_VALUE, LAST_VALUE, NTH_VALUE

```python
# Compare each sale to the best and worst in the region
cursor.execute("""
    SELECT 
        salesperson,
        region,
        amount,
        FIRST_VALUE(amount) OVER (
            PARTITION BY region ORDER BY amount DESC
        ) as best_in_region,
        ROUND(
            amount * 100.0 / FIRST_VALUE(amount) OVER (
                PARTITION BY region ORDER BY amount DESC
            ), 1
        ) as pct_of_best
    FROM sales
""")
```

### Window frame specification

```
ROWS BETWEEN ... AND ...

Options:
  UNBOUNDED PRECEDING  — from the start
  N PRECEDING          — N rows before current
  CURRENT ROW          — current row
  N FOLLOWING          — N rows after current
  UNBOUNDED FOLLOWING  — to the end

Examples:
  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  — running total
  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW           — 3-row moving window
  ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING            — 3-row centered window
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING — entire partition
```

---

## 10. Transactions

```python
import sqlite3

conn = sqlite3.connect("bank.db")
cursor = conn.cursor()

# By default, sqlite3 auto-commits
# For explicit transactions:

try:
    # Start transaction
    cursor.execute("BEGIN TRANSACTION")
    
    # Transfer $100 from Alice to Bob
    cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice'")
    cursor.execute("UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob'")
    
    # Verify Alice has enough funds
    cursor.execute("SELECT balance FROM accounts WHERE name = 'Alice'")
    if cursor.fetchone()["balance"] < 0:
        raise ValueError("Insufficient funds!")
    
    conn.commit()  # both updates succeed together
    print("Transfer complete")

except Exception as e:
    conn.rollback()  # undo everything
    print(f"Transfer failed: {e}")
```

---

## 11. Python Patterns for SQL

### Building dynamic queries safely

```python
def search_users(
    conn: sqlite3.Connection,
    name: str | None = None,
    city: str | None = None,
    min_age: int | None = None,
    max_age: int | None = None,
    is_active: bool | None = None,
    order_by: str = "name",
    limit: int = 50,
) -> list[dict]:
    """Build a dynamic query with optional filters."""
    
    query_parts = ["SELECT * FROM users WHERE 1=1"]
    params: list = []
    
    if name:
        query_parts.append("AND name LIKE ?")
        params.append(f"%{name}%")
    
    if city:
        query_parts.append("AND city = ?")
        params.append(city)
    
    if min_age is not None:
        query_parts.append("AND age >= ?")
        params.append(min_age)
    
    if max_age is not None:
        query_parts.append("AND age <= ?")
        params.append(max_age)
    
    if is_active is not None:
        query_parts.append("AND is_active = ?")
        params.append(int(is_active))
    
    # Whitelist for ORDER BY (prevent SQL injection!)
    allowed_order = {"name", "age", "city", "created_at"}
    if order_by in allowed_order:
        query_parts.append(f"ORDER BY {order_by}")
    
    query_parts.append("LIMIT ?")
    params.append(limit)
    
    query = " ".join(query_parts)
    
    cursor = conn.cursor()
    cursor.execute(query, params)
    return [dict(row) for row in cursor.fetchall()]

# Usage
users = search_users(conn, city="Kyiv", min_age=25, order_by="age")
```

### Bulk operations with transactions

```python
def bulk_insert_users(conn: sqlite3.Connection, users: list[dict]) -> int:
    """Insert many users in a single transaction (fast!)."""
    cursor = conn.cursor()
    
    try:
        cursor.execute("BEGIN TRANSACTION")
        cursor.executemany(
            "INSERT INTO users (name, email, age, city) VALUES (?, ?, ?, ?)",
            [(u["name"], u["email"], u["age"], u["city"]) for u in users],
        )
        conn.commit()
        return cursor.rowcount
    except Exception:
        conn.rollback()
        raise

# executemany in a transaction is ~100x faster than individual inserts
# For 100,000 rows: individual inserts ~30s, executemany ~0.3s
```

### Repository pattern

```python
class UserRepository:
    """Data access layer for users table."""
    
    def __init__(self, conn: sqlite3.Connection):
        self.conn = conn
        self.conn.row_factory = sqlite3.Row
    
    def create(self, name: str, email: str, age: int, city: str = "Unknown") -> int:
        cursor = self.conn.cursor()
        cursor.execute(
            "INSERT INTO users (name, email, age, city) VALUES (?, ?, ?, ?)",
            (name, email, age, city),
        )
        self.conn.commit()
        return cursor.lastrowid
    
    def get_by_id(self, user_id: int) -> dict | None:
        cursor = self.conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))
        row = cursor.fetchone()
        return dict(row) if row else None
    
    def get_by_email(self, email: str) -> dict | None:
        cursor = self.conn.cursor()
        cursor.execute("SELECT * FROM users WHERE email = ?", (email,))
        row = cursor.fetchone()
        return dict(row) if row else None
    
    def update(self, user_id: int, **fields) -> bool:
        if not fields:
            return False
        
        allowed = {"name", "email", "age", "city", "is_active"}
        fields = {k: v for k, v in fields.items() if k in allowed}
        
        set_clause = ", ".join(f"{k} = ?" for k in fields)
        values = list(fields.values()) + [user_id]
        
        cursor = self.conn.cursor()
        cursor.execute(f"UPDATE users SET {set_clause} WHERE id = ?", values)
        self.conn.commit()
        return cursor.rowcount > 0
    
    def delete(self, user_id: int) -> bool:
        cursor = self.conn.cursor()
        cursor.execute("DELETE FROM users WHERE id = ?", (user_id,))
        self.conn.commit()
        return cursor.rowcount > 0
    
    def list_all(self, limit: int = 100, offset: int = 0) -> list[dict]:
        cursor = self.conn.cursor()
        cursor.execute(
            "SELECT * FROM users ORDER BY id LIMIT ? OFFSET ?",
            (limit, offset),
        )
        return [dict(row) for row in cursor.fetchall()]
    
    def count(self) -> int:
        cursor = self.conn.cursor()
        cursor.execute("SELECT COUNT(*) FROM users")
        return cursor.fetchone()[0]

# Usage
repo = UserRepository(conn)
user_id = repo.create("Alice", "alice@test.com", 25, "Kyiv")
user = repo.get_by_id(user_id)
repo.update(user_id, age=26, city="London")
all_users = repo.list_all()
```

---

## 12. Complete Example — Analytics Dashboard Queries

```python
"""
Real-world analytics queries you'd write as a Data Engineer.
"""
import sqlite3

def setup_analytics_db() -> sqlite3.Connection:
    conn = sqlite3.connect(":memory:")
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()
    
    cursor.executescript("""
        CREATE TABLE events (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            event_type TEXT NOT NULL,
            amount REAL DEFAULT 0,
            created_at TEXT NOT NULL
        );
        
        CREATE INDEX idx_events_user ON events(user_id);
        CREATE INDEX idx_events_type ON events(event_type);
        CREATE INDEX idx_events_date ON events(created_at);
    """)
    
    # Insert sample events
    import random
    from datetime import datetime, timedelta
    
    base = datetime(2025, 1, 1)
    events = []
    for i in range(10_000):
        user_id = random.randint(1, 200)
        event_type = random.choices(
            ["view", "click", "add_to_cart", "purchase"],
            weights=[50, 30, 15, 5],
        )[0]
        amount = round(random.uniform(10, 500), 2) if event_type == "purchase" else 0
        ts = base + timedelta(
            days=random.randint(0, 59),
            hours=random.randint(0, 23),
            minutes=random.randint(0, 59),
        )
        events.append((user_id, event_type, amount, ts.isoformat()))
    
    cursor.executemany(
        "INSERT INTO events (user_id, event_type, amount, created_at) VALUES (?, ?, ?, ?)",
        events,
    )
    conn.commit()
    return conn

conn = setup_analytics_db()
cursor = conn.cursor()

# ── Query 1: Daily active users ──
cursor.execute("""
    SELECT 
        DATE(created_at) as date,
        COUNT(DISTINCT user_id) as dau
    FROM events
    GROUP BY date
    ORDER BY date
""")

# ── Query 2: Conversion funnel ──
cursor.execute("""
    WITH funnel AS (
        SELECT 
            event_type,
            COUNT(*) as event_count,
            COUNT(DISTINCT user_id) as user_count
        FROM events
        GROUP BY event_type
    )
    SELECT 
        event_type,
        event_count,
        user_count,
        ROUND(user_count * 100.0 / FIRST_VALUE(user_count) OVER (
            ORDER BY CASE event_type
                WHEN 'view' THEN 1
                WHEN 'click' THEN 2
                WHEN 'add_to_cart' THEN 3
                WHEN 'purchase' THEN 4
            END
        ), 1) as conversion_pct
    FROM funnel
    ORDER BY CASE event_type
        WHEN 'view' THEN 1
        WHEN 'click' THEN 2
        WHEN 'add_to_cart' THEN 3
        WHEN 'purchase' THEN 4
    END
""")

# ── Query 3: Revenue by week with week-over-week change ──
cursor.execute("""
    WITH weekly_revenue AS (
        SELECT 
            strftime('%Y-W%W', created_at) as week,
            SUM(amount) as revenue,
            COUNT(*) as purchase_count
        FROM events
        WHERE event_type = 'purchase'
        GROUP BY week
    )
    SELECT 
        week,
        ROUND(revenue, 2) as revenue,
        purchase_count,
        ROUND(LAG(revenue) OVER (ORDER BY week), 2) as prev_week,
        ROUND(
            (revenue - LAG(revenue) OVER (ORDER BY week)) * 100.0 
            / LAG(revenue) OVER (ORDER BY week), 
            1
        ) as wow_change_pct
    FROM weekly_revenue
    ORDER BY week
""")

# ── Query 4: Top 10 users by lifetime value ──
cursor.execute("""
    SELECT 
        user_id,
        COUNT(*) as total_events,
        SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) as purchases,
        ROUND(SUM(amount), 2) as lifetime_value,
        MIN(DATE(created_at)) as first_seen,
        MAX(DATE(created_at)) as last_seen
    FROM events
    GROUP BY user_id
    HAVING purchases > 0
    ORDER BY lifetime_value DESC
    LIMIT 10
""")

# ── Query 5: Cohort retention (users who came back) ──
cursor.execute("""
    WITH user_first_week AS (
        SELECT 
            user_id,
            strftime('%Y-W%W', MIN(created_at)) as cohort_week
        FROM events
        GROUP BY user_id
    ),
    user_activity AS (
        SELECT 
            e.user_id,
            ufw.cohort_week,
            strftime('%Y-W%W', e.created_at) as activity_week
        FROM events e
        JOIN user_first_week ufw ON e.user_id = ufw.user_id
    )
    SELECT 
        cohort_week,
        COUNT(DISTINCT user_id) as cohort_size,
        COUNT(DISTINCT CASE WHEN activity_week > cohort_week THEN user_id END) as retained,
        ROUND(
            COUNT(DISTINCT CASE WHEN activity_week > cohort_week THEN user_id END) * 100.0 
            / COUNT(DISTINCT user_id), 
            1
        ) as retention_pct
    FROM user_activity
    GROUP BY cohort_week
    ORDER BY cohort_week
""")
```

---

## 📝 Practice Tasks

### Task 1: E-Commerce Database
Design and create a database with: `products`, `customers`, `orders`, `order_items`, `reviews`. Insert sample data (50+ products, 20+ customers, 100+ orders). Write queries for: best-selling products, customer lifetime value, average review score per product, products with no reviews.

### Task 2: Window Function Challenge
Using the sales table from section 9, write queries for:
- Rank salespeople by total revenue per region
- Running total of sales per month
- 3-day moving average of daily sales
- Each sale as a percentage of the salesperson's total
- First and last sale amount per salesperson

### Task 3: Analytics Dashboard
Create an events table (page views, signups, purchases) with 50K+ rows. Write CTE-based queries for: daily/weekly/monthly active users, conversion funnel, cohort retention, revenue trend with period-over-period comparison.

### Task 4: Repository Pattern
Build a complete `OrderRepository` class with methods: `create_order`, `add_item`, `get_order_with_items`, `get_user_orders`, `cancel_order`, `get_revenue_by_period`. Use transactions where needed. Write pytest tests.

### Task 5: Data Migration Script
Write a script that: reads messy data from CSV (Task 2 from Day 8), validates with regex/rules, inserts valid rows into SQLite tables, logs invalid rows, generates a migration report (rows inserted, failed, by error type).

### Task 6: SQL Query Builder
Build a simple query builder class:
```python
query = QueryBuilder("users") \
    .select("name", "age", "city") \
    .where("age", ">=", 25) \
    .where("city", "=", "Kyiv") \
    .order_by("age", "DESC") \
    .limit(10)

sql, params = query.build()
# sql = "SELECT name, age, city FROM users WHERE age >= ? AND city = ? ORDER BY age DESC LIMIT ?"
# params = [25, "Kyiv", 10]
```

---

## 📚 Resources

- [Python Official — sqlite3](https://docs.python.org/3/library/sqlite3.html)
- [SQLite Official Documentation](https://www.sqlite.org/docs.html)
- [Mode Analytics — SQL Tutorial](https://mode.com/sql-tutorial/)
- [SQLBolt — Interactive SQL Lessons](https://sqlbolt.com/)
- [W3Schools SQL](https://www.w3schools.com/sql/)
- [Use The Index, Luke — SQL Performance](https://use-the-index-luke.com/)
- [Window Functions Explained](https://www.windowfunctions.com/)
- [Real Python — sqlite3](https://realpython.com/python-sqlite-sqlalchemy/)

---

## 🔑 Key Takeaways

1. **Always use parameterized queries (`?`)** — never f-strings for SQL. SQL injection is real.
2. **`sqlite3.Row`** — always set `row_factory` for dict-like access. Tuple indices are unreadable.
3. **CTEs over nested subqueries** — read top-to-bottom like a pipeline. Use `WITH` liberally.
4. **Window functions are essential for DE** — ROW_NUMBER, LAG/LEAD, running totals, ranking. Learn them deeply.
5. **Transactions for data integrity** — multiple related writes should be in one transaction.
6. **`executemany` for bulk inserts** — 100x faster than individual inserts in a loop.
7. **Indexes on columns you filter/join on** — dramatic speedup for large tables.
8. **LEFT JOIN + COALESCE** — the pattern for "all items, even those without matches."
9. **Repository pattern** — separate SQL from business logic. Makes testing easy.
10. **SQLite for development, PostgreSQL for production** — same SQL concepts, different capabilities.

---

> **Tomorrow (Day 16):** SQLAlchemy Core — Python's SQL toolkit. Build queries programmatically, connect to any database, handle connection pooling.

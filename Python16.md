# Day 16: SQLAlchemy Core — Python's SQL Toolkit

## 🎯 Goal

Master SQLAlchemy Core — the foundation layer that lets you build SQL queries programmatically in Python. SQLAlchemy has two layers: **Core** (SQL expression language) and **ORM** (object-relational mapping, Day 17). Today is Core — think of it as a Python query builder that works with any database.

Coming from Java, SQLAlchemy Core is like JOOQ or JDBC with a fluent query builder. Coming from JS, it's like Knex.js. The key advantage: your queries are database-agnostic Python objects, not raw SQL strings.

---

## 1. SQLAlchemy Architecture

```
┌──────────────────────────────────────────────┐
│                 Your Code                      │
├──────────────────────────────────────────────┤
│          SQLAlchemy ORM (Day 17)               │
│     (classes → tables, objects → rows)         │
├──────────────────────────────────────────────┤
│          SQLAlchemy Core (TODAY)                │
│     (Table objects, select(), insert()...)      │
├──────────────────────────────────────────────┤
│          Engine + Connection Pool               │
│     (manages DB connections)                    │
├──────────────────────────────────────────────┤
│          DBAPI (psycopg2, sqlite3, etc.)        │
│     (actual database driver)                    │
├──────────────────────────────────────────────┤
│          Database (PostgreSQL, SQLite, etc.)    │
└──────────────────────────────────────────────┘

Why two layers?
  Core:  SQL power users. Full control. Data Engineering, ETL, analytics.
  ORM:   Application developers. Object-oriented. Web apps, CRUD.
  
  For Data Engineering, Core is often MORE useful than ORM.
  You'll use both — Core for complex queries, ORM for web apps.
```

---

## 2. Engine — Connecting to Databases

### Creating an engine

```python
from sqlalchemy import create_engine

# SQLite (file)
engine = create_engine("sqlite:///mydb.db")

# SQLite (in-memory)
engine = create_engine("sqlite:///:memory:")

# PostgreSQL
engine = create_engine("postgresql://user:password@localhost:5432/mydb")

# PostgreSQL with psycopg2 (explicit driver)
engine = create_engine("postgresql+psycopg2://user:pass@localhost/mydb")

# MySQL
engine = create_engine("mysql+pymysql://user:pass@localhost/mydb")

# Connection URL format:
# dialect+driver://username:password@host:port/database

# With options
engine = create_engine(
    "postgresql://user:pass@localhost/mydb",
    echo=True,         # log all SQL (great for debugging!)
    pool_size=5,       # connection pool size
    max_overflow=10,   # extra connections beyond pool_size
    pool_timeout=30,   # seconds to wait for a connection
    pool_recycle=3600, # recycle connections after 1 hour
)

# echo=True is your best friend while learning!
# It prints every SQL statement SQLAlchemy generates.
```

### Using connections

```python
from sqlalchemy import create_engine, text

engine = create_engine("sqlite:///:memory:", echo=True)

# Modern pattern (SQLAlchemy 2.0): use connect() context manager
with engine.connect() as conn:
    result = conn.execute(text("SELECT 1 + 1 AS answer"))
    row = result.fetchone()
    print(row.answer)  # 2

# ⚠️ Connections are NOT auto-committed in 2.0!
# You must explicitly commit:
with engine.connect() as conn:
    conn.execute(text("INSERT INTO users (name) VALUES ('Alice')"))
    conn.commit()  # required!

# Or use begin() for auto-commit block:
with engine.begin() as conn:
    conn.execute(text("INSERT INTO users (name) VALUES ('Alice')"))
    # auto-commits when block exits successfully
    # auto-rollbacks on exception
```

### `text()` — raw SQL with parameters

```python
from sqlalchemy import text

with engine.connect() as conn:
    # Named parameters with :name syntax
    result = conn.execute(
        text("SELECT * FROM users WHERE age > :min_age AND city = :city"),
        {"min_age": 25, "city": "Kyiv"},
    )
    for row in result:
        print(row.name, row.age)
    
    # Insert with parameters
    conn.execute(
        text("INSERT INTO users (name, email, age) VALUES (:name, :email, :age)"),
        {"name": "Alice", "email": "alice@test.com", "age": 25},
    )
    
    # Bulk insert with executemany semantics
    conn.execute(
        text("INSERT INTO users (name, email, age) VALUES (:name, :email, :age)"),
        [
            {"name": "Bob", "email": "bob@test.com", "age": 30},
            {"name": "Charlie", "email": "charlie@test.com", "age": 35},
        ],
    )
    conn.commit()
```

---

## 3. MetaData & Table — Defining Schema

### Define tables programmatically

```python
from sqlalchemy import (
    MetaData, Table, Column, Integer, String, Float, Boolean,
    DateTime, Text, ForeignKey, UniqueConstraint, Index, CheckConstraint,
)
from sqlalchemy.sql import func

# MetaData holds all table definitions (like a schema registry)
metadata = MetaData()

# Define tables
users = Table(
    "users",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("name", String(100), nullable=False),
    Column("email", String(255), unique=True, nullable=False),
    Column("age", Integer, CheckConstraint("age >= 0 AND age <= 150")),
    Column("city", String(100), default="Unknown"),
    Column("is_active", Boolean, default=True),
    Column("created_at", DateTime, server_default=func.now()),
)

orders = Table(
    "orders",
    metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("user_id", Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=False),
    Column("product", String(200), nullable=False),
    Column("amount", Float, nullable=False),
    Column("quantity", Integer, default=1),
    Column("status", String(20), default="pending"),
    Column("ordered_at", DateTime, server_default=func.now()),
    
    # Table-level constraints
    Index("idx_orders_user_id", "user_id"),
    Index("idx_orders_status", "status"),
    UniqueConstraint("user_id", "product", "ordered_at", name="uq_user_product_time"),
)

products = Table(
    "products",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(200), nullable=False),
    Column("category", String(100)),
    Column("price", Float, nullable=False),
    Column("description", Text),
    Column("in_stock", Boolean, default=True),
)

# Create all tables in the database
engine = create_engine("sqlite:///:memory:", echo=True)
metadata.create_all(engine)
# Generates and executes CREATE TABLE statements for all defined tables
```

### Reflect existing tables (introspection)

```python
from sqlalchemy import MetaData, create_engine

engine = create_engine("sqlite:///existing_db.db")
metadata = MetaData()

# Reflect ALL tables from database
metadata.reflect(bind=engine)

# Access reflected tables
users = metadata.tables["users"]
orders = metadata.tables["orders"]

# Reflect specific table
from sqlalchemy import Table
users = Table("users", metadata, autoload_with=engine)

# Now you can query these tables without defining them manually!
# This is HUGE for Data Engineering — connect to any DB and start querying

print(users.columns.keys())  # ["id", "name", "email", ...]
for col in users.columns:
    print(f"{col.name}: {col.type}")
```

---

## 4. INSERT — Adding Data

```python
from sqlalchemy import insert

# Single insert
stmt = insert(users).values(name="Alice", email="alice@test.com", age=25, city="Kyiv")

with engine.begin() as conn:
    result = conn.execute(stmt)
    print(f"Inserted ID: {result.inserted_primary_key[0]}")

# Insert with dict
user_data = {"name": "Bob", "email": "bob@test.com", "age": 30, "city": "London"}
stmt = insert(users).values(**user_data)

with engine.begin() as conn:
    conn.execute(stmt)

# Bulk insert — list of dicts (fastest way!)
many_users = [
    {"name": "Charlie", "email": "charlie@test.com", "age": 35, "city": "NYC"},
    {"name": "Diana", "email": "diana@test.com", "age": 28, "city": "Berlin"},
    {"name": "Eve", "email": "eve@test.com", "age": 32, "city": "Kyiv"},
    {"name": "Frank", "email": "frank@test.com", "age": 27, "city": "Paris"},
    {"name": "Grace", "email": "grace@test.com", "age": 29, "city": "Tokyo"},
]

with engine.begin() as conn:
    conn.execute(insert(users), many_users)
    # SQLAlchemy generates efficient batch INSERT

# Insert with RETURNING (get back inserted data)
stmt = (
    insert(users)
    .values(name="Hank", email="hank@test.com", age=40)
    .returning(users.c.id, users.c.name)
)

with engine.begin() as conn:
    result = conn.execute(stmt)
    row = result.fetchone()
    print(f"Created: {row.id}, {row.name}")

# INSERT ... ON CONFLICT (upsert) — SQLite and PostgreSQL
from sqlalchemy.dialects.sqlite import insert as sqlite_insert

stmt = sqlite_insert(users).values(
    name="Alice", email="alice@test.com", age=26
)
stmt = stmt.on_conflict_do_update(
    index_elements=["email"],  # conflict on unique email
    set_={"age": stmt.excluded.age, "name": stmt.excluded.name},
)

with engine.begin() as conn:
    conn.execute(stmt)
```

---

## 5. SELECT — Querying Data

This is where SQLAlchemy Core shines — building complex queries programmatically.

### Basic queries

```python
from sqlalchemy import select

# Select all columns
stmt = select(users)
# SELECT users.id, users.name, users.email, ...

# Select specific columns
stmt = select(users.c.name, users.c.age, users.c.city)
# SELECT users.name, users.age, users.city FROM users

# Execute and fetch
with engine.connect() as conn:
    result = conn.execute(select(users))
    
    # Fetch all rows
    all_rows = result.fetchall()
    
    # Or iterate (memory efficient)
    result = conn.execute(select(users))
    for row in result:
        print(row.name, row.age)
    
    # As dicts
    result = conn.execute(select(users))
    for row in result.mappings():
        print(row["name"], row["age"])
    
    # First row or None
    result = conn.execute(select(users))
    first = result.first()

# .c is shorthand for .columns
# users.c.name == users.columns.name
```

### WHERE — filtering

```python
from sqlalchemy import select, and_, or_, not_

# Simple comparison
stmt = select(users).where(users.c.age > 25)
stmt = select(users).where(users.c.city == "Kyiv")
stmt = select(users).where(users.c.name != "Alice")

# Multiple conditions (AND — chained .where())
stmt = (
    select(users)
    .where(users.c.age >= 25)
    .where(users.c.city == "Kyiv")
    .where(users.c.is_active == True)
)
# Equivalent to:
stmt = select(users).where(
    and_(
        users.c.age >= 25,
        users.c.city == "Kyiv",
        users.c.is_active == True,
    )
)

# OR
stmt = select(users).where(
    or_(
        users.c.city == "Kyiv",
        users.c.city == "London",
    )
)

# NOT
stmt = select(users).where(not_(users.c.is_active == True))

# IN
stmt = select(users).where(users.c.city.in_(["Kyiv", "London", "NYC"]))

# NOT IN
stmt = select(users).where(users.c.city.not_in(["Unknown"]))

# BETWEEN
stmt = select(users).where(users.c.age.between(25, 35))

# LIKE
stmt = select(users).where(users.c.name.like("A%"))
stmt = select(users).where(users.c.email.contains("@test.com"))
stmt = select(users).where(users.c.name.startswith("Al"))

# IS NULL / IS NOT NULL
stmt = select(users).where(users.c.city.is_(None))
stmt = select(users).where(users.c.city.isnot(None))

# ORDER BY
stmt = select(users).order_by(users.c.age.desc())
stmt = select(users).order_by(users.c.city.asc(), users.c.age.desc())

# LIMIT / OFFSET
stmt = select(users).order_by(users.c.id).limit(10).offset(20)

# DISTINCT
stmt = select(users.c.city).distinct()
```

### Aggregate functions

```python
from sqlalchemy import func, select

# COUNT
stmt = select(func.count()).select_from(users)
# SELECT count(*) FROM users

# COUNT with condition
stmt = select(func.count()).select_from(users).where(users.c.is_active == True)

# Multiple aggregates
stmt = select(
    func.count().label("total"),
    func.avg(users.c.age).label("avg_age"),
    func.min(users.c.age).label("min_age"),
    func.max(users.c.age).label("max_age"),
)

with engine.connect() as conn:
    row = conn.execute(stmt).first()
    print(f"Total: {row.total}, Avg age: {row.avg_age:.1f}")

# GROUP BY
stmt = (
    select(
        users.c.city,
        func.count().label("user_count"),
        func.avg(users.c.age).label("avg_age"),
    )
    .group_by(users.c.city)
    .order_by(func.count().desc())
)

# HAVING
stmt = (
    select(
        users.c.city,
        func.count().label("user_count"),
    )
    .group_by(users.c.city)
    .having(func.count() >= 2)
)
```

---

## 6. JOINs

```python
from sqlalchemy import select, join, outerjoin

# First insert some orders
with engine.begin() as conn:
    conn.execute(insert(orders), [
        {"user_id": 1, "product": "Laptop", "amount": 999.99, "status": "delivered"},
        {"user_id": 1, "product": "Mouse", "amount": 29.99, "status": "delivered"},
        {"user_id": 2, "product": "Keyboard", "amount": 79.99, "status": "shipped"},
        {"user_id": 3, "product": "Monitor", "amount": 349.99, "status": "pending"},
        {"user_id": 5, "product": "Headphones", "amount": 199.99, "status": "cancelled"},
    ])

# INNER JOIN — using join()
stmt = (
    select(users.c.name, orders.c.product, orders.c.amount)
    .join(orders, users.c.id == orders.c.user_id)
    .order_by(users.c.name)
)
# or shorter — SQLAlchemy infers JOIN condition from ForeignKey:
stmt = (
    select(users.c.name, orders.c.product, orders.c.amount)
    .join(orders)  # auto-detects: users.id == orders.user_id
)

# LEFT OUTER JOIN
stmt = (
    select(
        users.c.name,
        func.count(orders.c.id).label("order_count"),
        func.coalesce(func.sum(orders.c.amount), 0).label("total_spent"),
    )
    .outerjoin(orders)  # LEFT JOIN — all users, even without orders
    .group_by(users.c.id, users.c.name)
    .order_by(func.sum(orders.c.amount).desc().nullslast())
)

# Multiple JOINs
stmt = (
    select(
        users.c.name,
        orders.c.product,
        products.c.category,
        orders.c.amount,
    )
    .join(orders, users.c.id == orders.c.user_id)
    .join(products, orders.c.product == products.c.name)
    .where(orders.c.status == "delivered")
)

# Self-join with aliased tables
from sqlalchemy import alias

u1 = users.alias("u1")
u2 = users.alias("u2")

stmt = (
    select(u1.c.name, u2.c.name)
    .join(u2, and_(u1.c.city == u2.c.city, u1.c.id < u2.c.id))
)
# Users from the same city
```

---

## 7. Subqueries & CTEs

### Subqueries

```python
from sqlalchemy import select, func

# Scalar subquery
avg_age = select(func.avg(users.c.age)).scalar_subquery()

stmt = select(users).where(users.c.age > avg_age)
# SELECT * FROM users WHERE age > (SELECT avg(age) FROM users)

# IN subquery
active_buyer_ids = (
    select(orders.c.user_id)
    .where(orders.c.status == "delivered")
    .distinct()
    .scalar_subquery()
)

stmt = select(users).where(users.c.id.in_(active_buyer_ids))
# SELECT * FROM users WHERE id IN (SELECT DISTINCT user_id FROM orders WHERE ...)

# Derived table (subquery in FROM)
user_stats = (
    select(
        users.c.city,
        func.avg(users.c.age).label("avg_age"),
        func.count().label("cnt"),
    )
    .group_by(users.c.city)
    .subquery()
)

stmt = (
    select(user_stats.c.city, user_stats.c.avg_age)
    .where(user_stats.c.cnt >= 2)
)
```

### CTEs — Common Table Expressions

```python
from sqlalchemy import select, func, literal_column

# Basic CTE
active_users_cte = (
    select(users.c.id, users.c.name, users.c.age, users.c.city)
    .where(users.c.is_active == True)
    .cte("active_users")
)

stmt = (
    select(
        active_users_cte.c.city,
        func.count().label("count"),
        func.avg(active_users_cte.c.age).label("avg_age"),
    )
    .group_by(active_users_cte.c.city)
)

# Multiple CTEs chained
user_orders_cte = (
    select(
        users.c.id,
        users.c.name,
        users.c.city,
        orders.c.amount,
        orders.c.status,
    )
    .outerjoin(orders)
    .cte("user_orders")
)

user_stats_cte = (
    select(
        user_orders_cte.c.id,
        user_orders_cte.c.name,
        user_orders_cte.c.city,
        func.count(user_orders_cte.c.amount).label("order_count"),
        func.coalesce(func.sum(user_orders_cte.c.amount), 0).label("total_spent"),
    )
    .group_by(
        user_orders_cte.c.id,
        user_orders_cte.c.name,
        user_orders_cte.c.city,
    )
    .cte("user_stats")
)

city_stats_cte = (
    select(
        user_stats_cte.c.city,
        func.count().label("user_count"),
        func.sum(user_stats_cte.c.order_count).label("total_orders"),
        func.sum(user_stats_cte.c.total_spent).label("city_revenue"),
    )
    .group_by(user_stats_cte.c.city)
    .cte("city_stats")
)

final_stmt = (
    select(city_stats_cte)
    .order_by(city_stats_cte.c.city_revenue.desc())
)

# SQLAlchemy generates:
# WITH user_orders AS (...),
#      user_stats AS (...),
#      city_stats AS (...)
# SELECT * FROM city_stats ORDER BY city_revenue DESC
```

---

## 8. Window Functions

```python
from sqlalchemy import select, func, over

# Create sales table for examples
from sqlalchemy import Table, Column, Integer, String, Float, DateTime

sales = Table(
    "sales", metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("salesperson", String(100)),
    Column("region", String(50)),
    Column("amount", Float),
    Column("sale_date", String(10)),
)

# ROW_NUMBER
stmt = select(
    sales.c.salesperson,
    sales.c.amount,
    sales.c.sale_date,
    func.row_number().over(
        partition_by=sales.c.salesperson,
        order_by=sales.c.amount.desc(),
    ).label("rank"),
)

# Running total
stmt = select(
    sales.c.sale_date,
    sales.c.salesperson,
    sales.c.amount,
    func.sum(sales.c.amount).over(
        partition_by=sales.c.salesperson,
        order_by=sales.c.sale_date,
        rows=(None, 0),  # UNBOUNDED PRECEDING to CURRENT ROW
    ).label("running_total"),
)

# rows parameter:
# (None, 0)  = UNBOUNDED PRECEDING to CURRENT ROW
# (-2, 0)    = 2 PRECEDING to CURRENT ROW
# (-1, 1)    = 1 PRECEDING to 1 FOLLOWING
# (0, None)  = CURRENT ROW to UNBOUNDED FOLLOWING

# LAG / LEAD
stmt = select(
    sales.c.salesperson,
    sales.c.sale_date,
    sales.c.amount,
    func.lag(sales.c.amount).over(
        partition_by=sales.c.salesperson,
        order_by=sales.c.sale_date,
    ).label("prev_amount"),
)

# RANK and DENSE_RANK
stmt = select(
    sales.c.salesperson,
    func.sum(sales.c.amount).label("total"),
    func.rank().over(
        order_by=func.sum(sales.c.amount).desc(),
    ).label("rank"),
    func.dense_rank().over(
        order_by=func.sum(sales.c.amount).desc(),
    ).label("dense_rank"),
).group_by(sales.c.salesperson)

# Top N per group (CTE + window function)
ranked = (
    select(
        sales,
        func.row_number().over(
            partition_by=sales.c.salesperson,
            order_by=sales.c.amount.desc(),
        ).label("rn"),
    )
    .cte("ranked")
)

stmt = select(ranked).where(ranked.c.rn <= 2)
# Top 2 sales per salesperson
```

---

## 9. UPDATE & DELETE

```python
from sqlalchemy import update, delete

# UPDATE
stmt = (
    update(users)
    .where(users.c.city == "Kyiv")
    .values(is_active=True)
)

with engine.begin() as conn:
    result = conn.execute(stmt)
    print(f"Updated {result.rowcount} rows")

# Update with expression
stmt = (
    update(users)
    .where(users.c.age < 30)
    .values(age=users.c.age + 1)
)

# Update with RETURNING
stmt = (
    update(users)
    .where(users.c.name == "Alice")
    .values(city="London")
    .returning(users.c.id, users.c.name, users.c.city)
)

with engine.begin() as conn:
    result = conn.execute(stmt)
    for row in result:
        print(f"Updated: {row.name} → {row.city}")

# DELETE
stmt = delete(users).where(users.c.is_active == False)

with engine.begin() as conn:
    result = conn.execute(stmt)
    print(f"Deleted {result.rowcount} rows")

# Delete with subquery
stmt = (
    delete(orders)
    .where(
        orders.c.user_id.not_in(
            select(users.c.id)
        )
    )
)
```

---

## 10. Dynamic Query Building

This is where Core really shines over raw SQL — building queries based on runtime conditions.

```python
from sqlalchemy import select, and_, or_, func

def search_users(
    engine,
    name: str | None = None,
    city: str | None = None,
    min_age: int | None = None,
    max_age: int | None = None,
    is_active: bool | None = None,
    order_by: str = "name",
    limit: int = 50,
) -> list[dict]:
    """Build query dynamically based on provided filters."""
    
    stmt = select(users)
    
    # Add filters conditionally
    conditions = []
    if name:
        conditions.append(users.c.name.ilike(f"%{name}%"))
    if city:
        conditions.append(users.c.city == city)
    if min_age is not None:
        conditions.append(users.c.age >= min_age)
    if max_age is not None:
        conditions.append(users.c.age <= max_age)
    if is_active is not None:
        conditions.append(users.c.is_active == is_active)
    
    if conditions:
        stmt = stmt.where(and_(*conditions))
    
    # Dynamic order by (safe — using column objects, not strings)
    order_columns = {
        "name": users.c.name,
        "age": users.c.age,
        "city": users.c.city,
    }
    order_col = order_columns.get(order_by, users.c.name)
    stmt = stmt.order_by(order_col).limit(limit)
    
    with engine.connect() as conn:
        result = conn.execute(stmt)
        return [row._asdict() for row in result]

# Usage — clean and safe!
results = search_users(engine, city="Kyiv", min_age=25, order_by="age")
```

### Building complex reports dynamically

```python
def get_sales_report(
    engine,
    group_by: str = "region",  # "region", "salesperson", "month"
    metric: str = "revenue",    # "revenue", "count", "avg_order"
    min_date: str | None = None,
    max_date: str | None = None,
):
    """Generate different report views from the same base query."""
    
    # Dynamic grouping
    group_columns = {
        "region": sales.c.region,
        "salesperson": sales.c.salesperson,
        "month": func.strftime("%Y-%m", sales.c.sale_date),
    }
    group_col = group_columns[group_by]
    
    # Dynamic metrics
    metrics = {
        "revenue": func.sum(sales.c.amount).label("value"),
        "count": func.count().label("value"),
        "avg_order": func.avg(sales.c.amount).label("value"),
    }
    metric_col = metrics[metric]
    
    stmt = (
        select(group_col.label("group"), metric_col)
        .group_by(group_col)
        .order_by(metric_col.desc())
    )
    
    # Optional date filters
    if min_date:
        stmt = stmt.where(sales.c.sale_date >= min_date)
    if max_date:
        stmt = stmt.where(sales.c.sale_date <= max_date)
    
    with engine.connect() as conn:
        return [row._asdict() for row in conn.execute(stmt)]
```

---

## 11. Transactions & Error Handling

```python
from sqlalchemy import insert, update, select
from sqlalchemy.exc import IntegrityError, SQLAlchemyError

# ── Pattern 1: begin() — auto-commit/rollback ──
try:
    with engine.begin() as conn:
        # Everything here is ONE transaction
        conn.execute(insert(users).values(name="Test", email="test@test.com", age=25))
        conn.execute(
            update(users).where(users.c.name == "Test").values(age=26)
        )
        # Auto-commits if no exception
        # Auto-rollbacks on exception
except IntegrityError as e:
    print(f"Constraint violation: {e}")
except SQLAlchemyError as e:
    print(f"Database error: {e}")

# ── Pattern 2: Manual commit/rollback ──
with engine.connect() as conn:
    try:
        conn.execute(insert(users).values(name="A", email="a@test.com", age=20))
        conn.execute(insert(users).values(name="B", email="b@test.com", age=25))
        conn.commit()
    except IntegrityError:
        conn.rollback()
        print("Rolled back due to constraint violation")

# ── Pattern 3: Savepoints (nested transactions) ──
with engine.begin() as conn:
    conn.execute(insert(users).values(name="Outer", email="outer@test.com", age=30))
    
    savepoint = conn.begin_nested()
    try:
        conn.execute(insert(users).values(name="Inner", email="outer@test.com", age=25))
        # Duplicate email!
        savepoint.commit()
    except IntegrityError:
        savepoint.rollback()
        print("Inner insert failed, but outer still committed")
    
    # "Outer" user is still inserted!
```

---

## 12. Connection Pool Best Practices

```python
from sqlalchemy import create_engine, pool

# Production configuration
engine = create_engine(
    "postgresql://user:pass@localhost/mydb",
    
    # Pool settings
    pool_size=5,            # maintain 5 connections
    max_overflow=10,        # allow 10 additional temporary connections
    pool_timeout=30,        # wait 30s for a connection before error
    pool_recycle=3600,      # recycle connections after 1 hour
    pool_pre_ping=True,     # test connections before using (handles DB restarts)
    
    # Performance
    echo=False,             # disable SQL logging in production
    echo_pool=False,        # disable pool logging
    
    # Execution options
    execution_options={
        "isolation_level": "READ COMMITTED",
    },
)

# For scripts/CLI tools — use NullPool (no pooling)
engine = create_engine(
    "postgresql://user:pass@localhost/mydb",
    poolclass=pool.NullPool,  # new connection each time, close immediately
)

# Dispose engine (close all connections) — for cleanup
engine.dispose()
```

---

## 13. Real-World DE Pattern — ETL with SQLAlchemy Core

```python
"""
Complete ETL pipeline using SQLAlchemy Core.
Extract from source DB → Transform → Load to target DB.
"""
from sqlalchemy import (
    create_engine, MetaData, Table, Column, Integer, String, Float,
    DateTime, select, insert, func, and_,
)
from datetime import datetime

# Source and target engines
source_engine = create_engine("sqlite:///source.db")
target_engine = create_engine("sqlite:///target.db")

# Define target table
target_metadata = MetaData()
daily_summary = Table(
    "daily_summary", target_metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("date", String(10), nullable=False),
    Column("city", String(100)),
    Column("total_users", Integer),
    Column("total_orders", Integer),
    Column("total_revenue", Float),
    Column("avg_order_value", Float),
    Column("loaded_at", DateTime, server_default=func.now()),
)

target_metadata.create_all(target_engine)

# Reflect source tables
source_metadata = MetaData()
source_metadata.reflect(bind=source_engine)
src_users = source_metadata.tables["users"]
src_orders = source_metadata.tables["orders"]


def extract(engine, date: str) -> list[dict]:
    """Extract aggregated data from source."""
    stmt = (
        select(
            func.date(src_orders.c.ordered_at).label("date"),
            src_users.c.city,
            func.count(func.distinct(src_users.c.id)).label("total_users"),
            func.count(src_orders.c.id).label("total_orders"),
            func.coalesce(func.sum(src_orders.c.amount), 0).label("total_revenue"),
            func.coalesce(func.avg(src_orders.c.amount), 0).label("avg_order_value"),
        )
        .join(src_orders, src_users.c.id == src_orders.c.user_id)
        .where(func.date(src_orders.c.ordered_at) == date)
        .group_by(func.date(src_orders.c.ordered_at), src_users.c.city)
    )
    
    with engine.connect() as conn:
        result = conn.execute(stmt)
        return [row._asdict() for row in result]


def transform(records: list[dict]) -> list[dict]:
    """Apply business rules and transformations."""
    transformed = []
    for record in records:
        record["avg_order_value"] = round(record["avg_order_value"], 2)
        record["total_revenue"] = round(record["total_revenue"], 2)
        # Filter out cities with zero revenue
        if record["total_revenue"] > 0:
            transformed.append(record)
    return transformed


def load(engine, records: list[dict]) -> int:
    """Load records into target table."""
    if not records:
        return 0
    
    with engine.begin() as conn:
        conn.execute(insert(daily_summary), records)
    
    return len(records)


def run_etl(date: str):
    """Run the full ETL pipeline."""
    print(f"[ETL] Starting for {date}")
    
    # Extract
    raw = extract(source_engine, date)
    print(f"[Extract] Got {len(raw)} records")
    
    # Transform
    clean = transform(raw)
    print(f"[Transform] {len(clean)} records after cleaning")
    
    # Load
    loaded = load(target_engine, clean)
    print(f"[Load] Inserted {loaded} records")
    
    print(f"[ETL] Complete for {date}")
```

---

## 📝 Practice Tasks

### Task 1: Schema Definition
Define SQLAlchemy Core tables for a blog platform: `authors`, `posts`, `comments`, `tags`, `post_tags` (many-to-many). Create them in SQLite, insert sample data (10 authors, 50 posts, 200 comments, 20 tags). Print the SQL that SQLAlchemy generates (`echo=True`).

### Task 2: Complex Queries
Using the blog schema, write SQLAlchemy Core queries for:
- Top 5 authors by number of posts
- Posts with the most comments
- Tags sorted by popularity (most posts)
- Authors who haven't posted in the last 30 days
- Average comments per post by author

### Task 3: Window Functions in Core
Using a sales/transactions table, implement with SQLAlchemy Core:
- Running total per customer
- Rank customers by total spending
- Month-over-month revenue change (LAG)
- Top 3 transactions per customer (ROW_NUMBER)

### Task 4: Dynamic Report Builder
Build a `ReportBuilder` class that generates different reports from the same data:
```python
builder = ReportBuilder(engine)
report = builder.build(
    group_by="city",        # or "month", "category"
    metrics=["revenue", "order_count", "avg_order"],
    filters={"min_date": "2025-01-01", "status": "delivered"},
    order_by="revenue",
    limit=20,
)
```

### Task 5: Database Reflection Tool
Write a script that connects to any SQLite database, reflects all tables, and generates: table list with column info, row counts, sample data (first 5 rows), foreign key relationships. Output as a formatted report.

### Task 6: ETL Pipeline
Build a complete ETL pipeline using SQLAlchemy Core:
- Extract: read from a CSV file into a staging table
- Transform: clean data, validate, compute derived columns using SQL
- Load: insert into final target table with conflict handling
- Log all steps with timestamps and row counts

---

## 📚 Resources

- [SQLAlchemy 2.0 Documentation](https://docs.sqlalchemy.org/en/20/)
- [SQLAlchemy Core Tutorial](https://docs.sqlalchemy.org/en/20/core/tutorial.html)
- [SQLAlchemy Column Elements](https://docs.sqlalchemy.org/en/20/core/sqlelement.html)
- [SQLAlchemy Engine Configuration](https://docs.sqlalchemy.org/en/20/core/engines.html)
- [Real Python — SQLAlchemy](https://realpython.com/python-sqlite-sqlalchemy/)
- [SQLAlchemy Expression Language Tutorial](https://docs.sqlalchemy.org/en/20/tutorial/)

---

## 🔑 Key Takeaways

1. **Core vs ORM** — Core gives you full SQL control. ORM gives you object mapping. For DE, know both, prefer Core for analytics.
2. **`echo=True`** — always enable while learning. See exactly what SQL is generated.
3. **Engine is NOT a connection** — it's a connection factory + pool. Use `engine.connect()` or `engine.begin()`.
4. **`engine.begin()`** — auto-commit on success, auto-rollback on exception. Use this for writes.
5. **Dynamic query building** — the killer feature. Build filters, grouping, ordering at runtime safely.
6. **`.cte()` for readable queries** — chain CTEs like a pipeline. Much cleaner than nested subqueries.
7. **Window functions work** — `func.row_number().over(partition_by=..., order_by=...)` — same power as raw SQL.
8. **Reflect existing databases** — `metadata.reflect(bind=engine)` to query any DB without defining tables.
9. **`pool_pre_ping=True`** — always enable in production. Handles stale connections gracefully.
10. **Parameters are safe** — SQLAlchemy always parameterizes. No SQL injection risk.

---

> **Tomorrow (Day 17):** SQLAlchemy ORM — map Python classes to database tables. Sessions, relationships, lazy loading, querying with models. The foundation for FastAPI + database apps.

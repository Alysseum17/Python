# Day 18: Database Design & Migrations (Alembic)

## 🎯 Goal

Learn database design principles and Alembic — the migration tool for SQLAlchemy. In production, you never manually `CREATE TABLE`. You write migrations — versioned scripts that evolve your schema over time. This is like Git for your database schema. Coming from Java, this is Flyway/Liquibase. Coming from JS, this is Knex migrations or Prisma Migrate.

---

## 1. Why Migrations?

```
Without migrations:
  Developer A: "I added a column to users table on my machine"
  Developer B: "My app is crashing because that column doesn't exist"
  Production:  "Nobody ran the ALTER TABLE"
  Rollback:    "How do we undo this? Nobody remembers the old schema"

With migrations:
  1. Developer writes a migration file (versioned, checked into Git)
  2. Migration runs automatically on deploy
  3. Every environment has the same schema
  4. Can upgrade AND downgrade (rollback)
  5. Full history of every schema change

Migration = a script that changes your database schema from version N to N+1.
```

---

## 2. Database Design Fundamentals

### Normalization — eliminate data redundancy

```
1NF (First Normal Form):
  - Each column holds atomic (single) values
  - No repeating groups
  
  ❌ Bad:  users(id, name, phones="555-1234,555-5678")
  ✅ Good: users(id, name) + phone_numbers(id, user_id, phone)

2NF (Second Normal Form):
  - 1NF + every non-key column depends on the WHOLE primary key
  - Relevant for composite primary keys
  
  ❌ Bad:  order_items(order_id, product_id, product_name, quantity)
           product_name depends on product_id alone, not the full key
  ✅ Good: order_items(order_id, product_id, quantity)
           products(id, name)

3NF (Third Normal Form):
  - 2NF + no transitive dependencies (non-key → non-key)
  
  ❌ Bad:  employees(id, name, dept_id, dept_name, dept_location)
           dept_name and dept_location depend on dept_id, not employee id
  ✅ Good: employees(id, name, dept_id)
           departments(id, name, location)

For Data Engineering:
  - OLTP (transactional): normalize to 3NF (fewer anomalies, less storage)
  - OLAP (analytical): denormalize (fewer JOINs, faster queries)
  - Data warehouses often use star/snowflake schemas (denormalized)
```

### Common table patterns

```
1. Lookup / Reference tables (rarely change)
   statuses(id, name)
   countries(code, name)
   categories(id, name, parent_id)

2. Entity tables (main business objects)
   users(id, name, email, created_at, updated_at)
   products(id, name, price, category_id)

3. Transaction / Event tables (append-heavy)
   orders(id, user_id, total, status, created_at)
   events(id, user_id, event_type, metadata, timestamp)

4. Junction / Association tables (many-to-many)
   post_tags(post_id, tag_id)
   user_roles(user_id, role_id)

5. Audit / History tables
   user_audit(id, user_id, field, old_value, new_value, changed_at)
```

---

## 3. Index Design

### When to create indexes

```
✅ CREATE index on:
  - Columns in WHERE clauses (frequently filtered)
  - Columns in JOIN conditions (foreign keys!)
  - Columns in ORDER BY
  - Columns in GROUP BY
  - Unique constraints (auto-indexed)

❌ DON'T index:
  - Columns rarely used in queries
  - Tables with few rows (< 1000)
  - Columns with very low cardinality (e.g., boolean — only 2 values)
  - Columns that change frequently (index rebuild cost)

Every INSERT/UPDATE/DELETE must also update indexes.
More indexes = slower writes, faster reads.
```

### Index types in practice

```python
from sqlalchemy import Index, Table, Column, Integer, String, MetaData

# Single column index
Index("idx_users_email", users.c.email)

# Composite index (multi-column)
Index("idx_orders_user_status", orders.c.user_id, orders.c.status)
# Helps queries like: WHERE user_id = ? AND status = ?
# Also helps: WHERE user_id = ?
# Does NOT help: WHERE status = ? (leftmost prefix rule)

# Unique index
Index("uq_users_email", users.c.email, unique=True)

# Partial index (PostgreSQL) — index only a subset
Index(
    "idx_active_users",
    users.c.email,
    postgresql_where=users.c.is_active == True,
)
# Only indexes active users — smaller, faster

# Expression index (PostgreSQL)
Index("idx_users_lower_email", func.lower(users.c.email))
```

### Analyzing query performance

```sql
-- SQLite: explain query plan
EXPLAIN QUERY PLAN SELECT * FROM users WHERE email = 'alice@test.com';

-- PostgreSQL: explain with analysis
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@test.com';

-- What to look for:
-- "SCAN TABLE users" → full table scan (bad for large tables)
-- "SEARCH TABLE users USING INDEX idx_users_email" → index used (good!)
```

---

## 4. Alembic — Setup

### Installation and initialization

```bash
pip install alembic

# Initialize Alembic in your project
cd myproject
alembic init alembic

# Creates:
# myproject/
# ├── alembic/
# │   ├── env.py              ← main config (connects to DB, runs migrations)
# │   ├── script.py.mako      ← template for new migrations
# │   └── versions/            ← migration files live here
# └── alembic.ini              ← connection URL and settings
```

### Configure `alembic.ini`

```ini
# alembic.ini
[alembic]
script_location = alembic

# Database URL — for development
sqlalchemy.url = sqlite:///app.db

# For PostgreSQL:
# sqlalchemy.url = postgresql://user:pass@localhost:5432/mydb

# Better: use environment variable (don't commit passwords!)
# sqlalchemy.url = %(DB_URL)s
```

### Configure `env.py` — connect to your models

```python
# alembic/env.py (key parts)
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context

# ⭐ Import your models' Base metadata
from myproject.models import Base

config = context.config

# Set up logging
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# ⭐ Tell Alembic about your models
target_metadata = Base.metadata

# ... rest of env.py (auto-generated, usually fine as-is)
```

### Using environment variables for the URL

```python
# alembic/env.py — add this near the top
import os

def get_url():
    return os.environ.get("DATABASE_URL", "sqlite:///app.db")

# In run_migrations_online():
def run_migrations_online():
    configuration = config.get_section(config.config_ini_section)
    configuration["sqlalchemy.url"] = get_url()  # override from env
    
    connectable = engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    # ... rest unchanged
```

---

## 5. Creating Migrations

### Auto-generate from model changes

```bash
# After changing your models, generate a migration:
alembic revision --autogenerate -m "create users and orders tables"

# This compares your models (Python classes) with the current DB schema
# and generates a migration script with the differences.
```

```python
# alembic/versions/001_create_users_and_orders.py (auto-generated)
"""create users and orders tables

Revision ID: a1b2c3d4e5f6
Revises:
Create Date: 2025-02-14 12:00:00.000000
"""
from alembic import op
import sqlalchemy as sa

# revision identifiers
revision = 'a1b2c3d4e5f6'
down_revision = None  # first migration
branch_labels = None
depends_on = None


def upgrade():
    """Apply migration — create tables."""
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), autoincrement=True, nullable=False),
        sa.Column('name', sa.String(length=100), nullable=False),
        sa.Column('email', sa.String(length=255), nullable=False),
        sa.Column('age', sa.Integer(), nullable=True),
        sa.Column('city', sa.String(length=100), server_default='Unknown'),
        sa.Column('is_active', sa.Boolean(), server_default=sa.text('1')),
        sa.Column('created_at', sa.DateTime(), server_default=sa.func.now()),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('email'),
    )
    op.create_index('idx_users_email', 'users', ['email'])
    
    op.create_table(
        'orders',
        sa.Column('id', sa.Integer(), autoincrement=True, nullable=False),
        sa.Column('user_id', sa.Integer(), nullable=False),
        sa.Column('product', sa.String(length=200), nullable=False),
        sa.Column('amount', sa.Float(), nullable=False),
        sa.Column('quantity', sa.Integer(), server_default='1'),
        sa.Column('status', sa.String(length=20), server_default='pending'),
        sa.Column('ordered_at', sa.DateTime(), server_default=sa.func.now()),
        sa.PrimaryKeyConstraint('id'),
        sa.ForeignKeyConstraint(['user_id'], ['users.id'], ondelete='CASCADE'),
    )
    op.create_index('idx_orders_user_id', 'orders', ['user_id'])
    op.create_index('idx_orders_status', 'orders', ['status'])


def downgrade():
    """Reverse migration — drop tables."""
    op.drop_index('idx_orders_status')
    op.drop_index('idx_orders_user_id')
    op.drop_table('orders')
    op.drop_index('idx_users_email')
    op.drop_table('users')
```

### Manual migrations

```bash
# Create empty migration (for custom operations)
alembic revision -m "add phone column to users"
```

```python
"""add phone column to users

Revision ID: b2c3d4e5f6a7
Revises: a1b2c3d4e5f6
"""
from alembic import op
import sqlalchemy as sa

revision = 'b2c3d4e5f6a7'
down_revision = 'a1b2c3d4e5f6'


def upgrade():
    op.add_column('users', sa.Column('phone', sa.String(20), nullable=True))

def downgrade():
    op.drop_column('users', 'phone')
```

---

## 6. Running Migrations

```bash
# Apply all pending migrations (upgrade to latest)
alembic upgrade head

# Apply one migration at a time
alembic upgrade +1

# Upgrade to specific revision
alembic upgrade a1b2c3d4e5f6

# Downgrade one step
alembic downgrade -1

# Downgrade to specific revision
alembic downgrade a1b2c3d4e5f6

# Downgrade all the way (empty database)
alembic downgrade base

# Check current version
alembic current

# Show migration history
alembic history
alembic history --verbose

# Show pending migrations (not yet applied)
alembic heads
```

### Migration flow in practice

```bash
# 1. Modify your models (add column, new table, etc.)

# 2. Generate migration
alembic revision --autogenerate -m "add user_profiles table"

# 3. Review the generated migration (ALWAYS review!)
# Check alembic/versions/xxxx_add_user_profiles_table.py
# Auto-generate is good but not perfect — verify the SQL

# 4. Test the migration
alembic upgrade head     # apply
alembic downgrade -1     # rollback
alembic upgrade head     # apply again — should work both ways

# 5. Commit migration file to Git

# 6. Deploy — migration runs automatically
# (In CI/CD: alembic upgrade head before starting the app)
```

---

## 7. Common Migration Operations

### Column operations

```python
from alembic import op
import sqlalchemy as sa

def upgrade():
    # Add column
    op.add_column('users', sa.Column('bio', sa.Text(), nullable=True))
    
    # Add column with default for existing rows
    op.add_column('users', sa.Column(
        'role', sa.String(20), server_default='user', nullable=False
    ))
    
    # Rename column (not supported by all databases)
    op.alter_column('users', 'name', new_column_name='full_name')
    
    # Change column type
    op.alter_column('users', 'age',
        existing_type=sa.Integer(),
        type_=sa.SmallInteger(),
    )
    
    # Make column non-nullable (with existing data!)
    # Step 1: fill NULLs first
    op.execute("UPDATE users SET city = 'Unknown' WHERE city IS NULL")
    # Step 2: then alter
    op.alter_column('users', 'city',
        existing_type=sa.String(100),
        nullable=False,
    )
    
    # Drop column
    op.drop_column('users', 'obsolete_field')

def downgrade():
    op.add_column('users', sa.Column('obsolete_field', sa.String(100)))
    op.alter_column('users', 'city', nullable=True)
    op.alter_column('users', 'age', type_=sa.Integer())
    op.alter_column('users', 'full_name', new_column_name='name')
    op.drop_column('users', 'role')
    op.drop_column('users', 'bio')
```

### Table operations

```python
def upgrade():
    # Create table
    op.create_table(
        'categories',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('name', sa.String(100), nullable=False),
        sa.Column('parent_id', sa.Integer(), sa.ForeignKey('categories.id')),
    )
    
    # Rename table
    op.rename_table('old_name', 'new_name')
    
    # Drop table
    op.drop_table('deprecated_table')

def downgrade():
    op.create_table('deprecated_table', ...)
    op.rename_table('new_name', 'old_name')
    op.drop_table('categories')
```

### Index and constraint operations

```python
def upgrade():
    # Create index
    op.create_index('idx_users_city', 'users', ['city'])
    
    # Composite index
    op.create_index('idx_orders_user_status', 'orders', ['user_id', 'status'])
    
    # Unique constraint
    op.create_unique_constraint('uq_users_email', 'users', ['email'])
    
    # Foreign key
    op.create_foreign_key(
        'fk_orders_user_id',  # constraint name
        'orders',             # source table
        'users',              # target table
        ['user_id'],          # source columns
        ['id'],               # target columns
        ondelete='CASCADE',
    )
    
    # Check constraint
    op.create_check_constraint(
        'ck_users_age_positive',
        'users',
        'age >= 0 AND age <= 150',
    )
    
    # Drop index
    op.drop_index('idx_old_index')
    
    # Drop constraint
    op.drop_constraint('old_constraint', 'table_name')

def downgrade():
    op.drop_check_constraint('ck_users_age_positive', 'users')
    op.drop_constraint('fk_orders_user_id', 'orders', type_='foreignkey')
    op.drop_constraint('uq_users_email', 'users', type_='unique')
    op.drop_index('idx_orders_user_status')
    op.drop_index('idx_users_city')
```

### Data migrations (migrate data, not just schema)

```python
"""Migrate from single name to first_name + last_name."""
from alembic import op
import sqlalchemy as sa

def upgrade():
    # Step 1: Add new columns
    op.add_column('users', sa.Column('first_name', sa.String(50)))
    op.add_column('users', sa.Column('last_name', sa.String(50)))
    
    # Step 2: Migrate data
    # Use raw SQL for data migrations — safer and faster
    op.execute("""
        UPDATE users 
        SET first_name = SUBSTR(name, 1, INSTR(name || ' ', ' ') - 1),
            last_name = SUBSTR(name, INSTR(name || ' ', ' ') + 1)
    """)
    
    # Step 3: Make new columns non-nullable
    op.alter_column('users', 'first_name', nullable=False)
    op.alter_column('users', 'last_name', nullable=False)
    
    # Step 4: Drop old column
    op.drop_column('users', 'name')

def downgrade():
    op.add_column('users', sa.Column('name', sa.String(100)))
    op.execute("""
        UPDATE users SET name = first_name || ' ' || last_name
    """)
    op.alter_column('users', 'name', nullable=False)
    op.drop_column('users', 'last_name')
    op.drop_column('users', 'first_name')
```

---

## 8. Alembic Best Practices

### Always review autogenerated migrations

```python
# Autogenerate is GOOD but NOT PERFECT. It can:
# ✅ Detect: new tables, new columns, dropped tables, dropped columns
# ✅ Detect: type changes, nullable changes, new indexes, foreign keys
# ⚠️ Miss: column renames (sees as drop + add)
# ⚠️ Miss: table renames
# ⚠️ Miss: data migrations
# ❌ Can't: complex logic, conditional changes

# ALWAYS review before running!
```

### Migration naming conventions

```bash
# Descriptive names
alembic revision --autogenerate -m "create users table"
alembic revision --autogenerate -m "add email index to users"
alembic revision --autogenerate -m "add user_profiles table with one-to-one"
alembic revision -m "backfill user roles from legacy data"

# Bad names
alembic revision -m "update"
alembic revision -m "fix"
alembic revision -m "changes"
```

### Never edit migrations after they've been applied in production

```
Migration workflow:
  
  1. Write migration in development
  2. Test upgrade AND downgrade locally
  3. Commit to Git
  4. Applied in staging → test
  5. Applied in production → DONE. This migration is FROZEN.
  
  Need to change something? Write a NEW migration.
  Never edit a migration that has been applied anywhere.
  
  Think of migrations like Git commits — you don't rewrite published history.
```

### Handle multiple developers

```bash
# Two developers create migrations from the same parent?
# Alembic detects this:

# Developer A: a1b2 → xxxx (adds column)
# Developer B: a1b2 → yyyy (adds table)

# After merge, you have two heads:
alembic heads
# Shows: xxxx, yyyy

# Fix: merge heads into one
alembic merge heads -m "merge branch migrations"
# Creates a new migration that depends on BOTH

# Or: one developer rebases their migration
# (delete theirs, regenerate from the merged state)
```

---

## 9. Schema Design Patterns for Data Engineering

### Pattern 1: Star schema (data warehouse)

```python
"""
Star schema: central fact table surrounded by dimension tables.
Optimized for analytical queries (OLAP).
"""
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, Text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

# ── Dimension tables (descriptive, slowly changing) ──

class DimDate(Base):
    __tablename__ = "dim_date"
    
    date_key: Mapped[int] = mapped_column(primary_key=True)  # YYYYMMDD
    full_date: Mapped[str] = mapped_column(String(10))        # 2025-02-14
    year: Mapped[int] = mapped_column(Integer)
    quarter: Mapped[int] = mapped_column(Integer)
    month: Mapped[int] = mapped_column(Integer)
    month_name: Mapped[str] = mapped_column(String(20))
    week: Mapped[int] = mapped_column(Integer)
    day_of_week: Mapped[int] = mapped_column(Integer)
    day_name: Mapped[str] = mapped_column(String(20))
    is_weekend: Mapped[bool] = mapped_column()
    is_holiday: Mapped[bool] = mapped_column(default=False)

class DimProduct(Base):
    __tablename__ = "dim_product"
    
    product_key: Mapped[int] = mapped_column(primary_key=True)
    product_id: Mapped[str] = mapped_column(String(50))  # natural/business key
    name: Mapped[str] = mapped_column(String(200))
    category: Mapped[str] = mapped_column(String(100))
    subcategory: Mapped[str] = mapped_column(String(100))
    brand: Mapped[str] = mapped_column(String(100))
    price: Mapped[float] = mapped_column(Float)

class DimCustomer(Base):
    __tablename__ = "dim_customer"
    
    customer_key: Mapped[int] = mapped_column(primary_key=True)
    customer_id: Mapped[str] = mapped_column(String(50))
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255))
    city: Mapped[str] = mapped_column(String(100))
    country: Mapped[str] = mapped_column(String(100))
    segment: Mapped[str] = mapped_column(String(50))

# ── Fact table (measurable events, foreign keys to dimensions) ──

class FactSales(Base):
    __tablename__ = "fact_sales"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    date_key: Mapped[int] = mapped_column(ForeignKey("dim_date.date_key"))
    product_key: Mapped[int] = mapped_column(ForeignKey("dim_product.product_key"))
    customer_key: Mapped[int] = mapped_column(ForeignKey("dim_customer.customer_key"))
    
    # Measures (numeric values you aggregate)
    quantity: Mapped[int] = mapped_column(Integer)
    unit_price: Mapped[float] = mapped_column(Float)
    total_amount: Mapped[float] = mapped_column(Float)
    discount: Mapped[float] = mapped_column(Float, default=0)
    
    # Degenerate dimension (order ID — no separate table needed)
    order_id: Mapped[str] = mapped_column(String(50))

# Typical analytical query:
# SELECT
#     d.month_name, p.category,
#     SUM(f.total_amount) as revenue,
#     COUNT(DISTINCT f.customer_key) as unique_customers
# FROM fact_sales f
# JOIN dim_date d ON f.date_key = d.date_key
# JOIN dim_product p ON f.product_key = p.product_key
# GROUP BY d.month_name, p.category
```

### Pattern 2: Slowly Changing Dimensions (SCD Type 2)

```python
"""
SCD Type 2: track historical changes to dimension records.
When a customer moves cities, keep BOTH the old and new records.
"""

class DimCustomerSCD2(Base):
    __tablename__ = "dim_customer_scd2"
    
    surrogate_key: Mapped[int] = mapped_column(primary_key=True)
    customer_id: Mapped[str] = mapped_column(String(50))  # natural key (not unique!)
    name: Mapped[str] = mapped_column(String(100))
    city: Mapped[str] = mapped_column(String(100))
    country: Mapped[str] = mapped_column(String(100))
    
    # SCD2 tracking columns
    effective_from: Mapped[str] = mapped_column(String(10))  # 2025-01-01
    effective_to: Mapped[str] = mapped_column(String(10), default="9999-12-31")
    is_current: Mapped[bool] = mapped_column(default=True)

# Example data:
# surrogate_key | customer_id | city   | effective_from | effective_to | is_current
# 1             | C001        | Kyiv   | 2024-01-01     | 2025-06-30   | False
# 2             | C001        | London | 2025-07-01     | 9999-12-31   | True
#
# Customer C001 moved from Kyiv to London on 2025-07-01.
# Historical queries can find their Kyiv address for old orders.
```

### Pattern 3: Event sourcing / append-only

```python
"""
Event sourcing: store every change as an immutable event.
Common in data pipelines, analytics, and audit trails.
"""

class Event(Base):
    __tablename__ = "events"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    entity_type: Mapped[str] = mapped_column(String(50))  # "user", "order"
    entity_id: Mapped[str] = mapped_column(String(50))
    event_type: Mapped[str] = mapped_column(String(50))   # "created", "updated", "deleted"
    payload: Mapped[str] = mapped_column(Text)             # JSON blob
    created_at: Mapped[str] = mapped_column(String(30))
    created_by: Mapped[str | None] = mapped_column(String(100))

# Advantages:
# - Full audit trail
# - Can rebuild any state at any point in time
# - No data loss (never UPDATE or DELETE)
# - Great for analytics pipelines
#
# Used by: Kafka, event-driven architectures, CQRS
```

### Pattern 4: Staging tables for ETL

```python
"""
Staging pattern: load raw data first, validate, then move to final tables.
Standard in data engineering pipelines.
"""

class StagingUsers(Base):
    """Raw data landing zone — no constraints, all TEXT."""
    __tablename__ = "stg_users"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    raw_name: Mapped[str | None] = mapped_column(Text)
    raw_email: Mapped[str | None] = mapped_column(Text)
    raw_age: Mapped[str | None] = mapped_column(Text)  # TEXT — might have garbage
    raw_city: Mapped[str | None] = mapped_column(Text)
    source_file: Mapped[str] = mapped_column(String(255))
    loaded_at: Mapped[str] = mapped_column(String(30))
    is_valid: Mapped[bool | None] = mapped_column(default=None)
    validation_errors: Mapped[str | None] = mapped_column(Text)

# ETL flow:
# 1. TRUNCATE stg_users (or create new staging table per load)
# 2. INSERT raw data from CSV/API into stg_users
# 3. Validate: UPDATE stg_users SET is_valid = ..., validation_errors = ...
# 4. INSERT INTO users SELECT ... FROM stg_users WHERE is_valid = TRUE
# 5. INSERT INTO error_log SELECT ... FROM stg_users WHERE is_valid = FALSE
```

---

## 10. Complete Project Structure with Alembic

```
myproject/
├── pyproject.toml
├── alembic.ini                    ← Alembic configuration
├── alembic/
│   ├── env.py                     ← Connects Alembic to your models
│   ├── script.py.mako             ← Migration file template
│   └── versions/                  ← Migration files
│       ├── 001_create_users.py
│       ├── 002_create_orders.py
│       ├── 003_add_user_phone.py
│       └── 004_create_analytics_views.py
├── src/
│   └── myproject/
│       ├── __init__.py
│       ├── config.py              ← Database URL, settings
│       ├── database.py            ← Engine, session factory
│       ├── models/                ← ORM models
│       │   ├── __init__.py        ← imports Base, all models
│       │   ├── base.py            ← DeclarativeBase + mixins
│       │   ├── user.py
│       │   ├── order.py
│       │   └── product.py
│       └── repositories/          ← Data access layer
│           ├── __init__.py
│           ├── user_repo.py
│           └── order_repo.py
└── tests/
    ├── conftest.py                ← Test fixtures (in-memory DB)
    ├── test_models.py
    └── test_repositories.py
```

### `src/myproject/models/base.py`

```python
from datetime import datetime
from sqlalchemy import Boolean, DateTime
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.now
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.now, onupdate=datetime.now
    )

class SoftDeleteMixin:
    is_deleted: Mapped[bool] = mapped_column(Boolean, default=False)
    deleted_at: Mapped[datetime | None] = mapped_column(DateTime, default=None)
```

### `src/myproject/database.py`

```python
import os
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session

DATABASE_URL = os.environ.get("DATABASE_URL", "sqlite:///app.db")

engine = create_engine(
    DATABASE_URL,
    echo=bool(os.environ.get("SQL_ECHO", False)),
    pool_pre_ping=True,
)

SessionLocal = sessionmaker(bind=engine)

def get_session() -> Session:
    """Get a new database session."""
    return SessionLocal()
```

### `src/myproject/models/__init__.py`

```python
# Import everything so Alembic can see all models
from .base import Base, TimestampMixin, SoftDeleteMixin
from .user import User
from .order import Order
from .product import Product

__all__ = ["Base", "User", "Order", "Product"]
```

### `alembic/env.py` (key configuration)

```python
from myproject.models import Base
from myproject.database import DATABASE_URL

# ... in run_migrations_online():
config.set_main_option("sqlalchemy.url", DATABASE_URL)
target_metadata = Base.metadata
```

---

## 11. Migration Workflow — Day-to-Day

```bash
# ── Starting a new project ──
pip install sqlalchemy alembic
alembic init alembic
# Configure alembic.ini and env.py (see above)
# Write your models
alembic revision --autogenerate -m "initial schema"
alembic upgrade head

# ── Adding a feature that needs DB changes ──
# 1. Modify models (e.g., add a column)
# 2. Generate migration:
alembic revision --autogenerate -m "add bio column to users"
# 3. Review the migration file
# 4. Test:
alembic upgrade head
alembic downgrade -1
alembic upgrade head
# 5. Commit everything to Git

# ── Deploying ──
# In your deployment script or Docker entrypoint:
alembic upgrade head
# Then start the application

# ── Checking status ──
alembic current          # what version is the DB at?
alembic history          # full migration history
alembic heads            # latest migration(s)

# ── Troubleshooting ──
# "Target database is not up to date"
alembic stamp head       # mark current DB as "at head" (dangerous!)

# "Multiple heads"
alembic merge heads -m "merge migrations"
```

---

## 12. Testing Migrations

```python
import pytest
from sqlalchemy import create_engine, inspect, text
from alembic.config import Config
from alembic import command

@pytest.fixture
def alembic_config():
    """Alembic config pointing to test database."""
    config = Config("alembic.ini")
    config.set_main_option("sqlalchemy.url", "sqlite:///:memory:")
    return config

def test_upgrade_to_head(alembic_config):
    """Test that all migrations can be applied."""
    command.upgrade(alembic_config, "head")

def test_downgrade_to_base(alembic_config):
    """Test that all migrations can be reversed."""
    command.upgrade(alembic_config, "head")
    command.downgrade(alembic_config, "base")

def test_upgrade_downgrade_cycle(alembic_config):
    """Test upgrade → downgrade → upgrade."""
    command.upgrade(alembic_config, "head")
    command.downgrade(alembic_config, "base")
    command.upgrade(alembic_config, "head")

def test_tables_exist_after_migration(alembic_config):
    """Verify expected tables are created."""
    command.upgrade(alembic_config, "head")
    
    engine = create_engine("sqlite:///:memory:")
    # Need to connect to same DB as alembic — adjust for real test
    inspector = inspect(engine)
    tables = inspector.get_table_names()
    
    assert "users" in tables
    assert "orders" in tables
```

---

## 📝 Practice Tasks

### Task 1: Design an E-Commerce Schema
Design a complete e-commerce database with proper normalization:
- Customers, products, categories, orders, order_items, reviews, addresses
- Define all models with SQLAlchemy ORM
- Create Alembic migrations
- Include proper indexes for expected query patterns

### Task 2: Migration Practice
Start with a simple schema (users table). Write sequential migrations for:
1. Add `email` column (unique, nullable at first)
2. Backfill emails for existing users
3. Make `email` non-nullable
4. Add `user_profiles` table (one-to-one)
5. Rename `name` to `full_name`
6. Add `tags` table and `user_tags` junction table

### Task 3: Star Schema
Design a star schema for an analytics warehouse:
- Fact table: daily_page_views (measures: views, unique_visitors, avg_time)
- Dimensions: date, page, traffic_source, device_type
- Write the models and migrations
- Write analytical queries using SQLAlchemy Core

### Task 4: Index Optimization
Take the e-commerce schema from Task 1:
- Identify which columns need indexes based on expected queries
- Write a migration that adds all indexes
- Write `EXPLAIN` queries to verify indexes are used
- Document your indexing strategy

### Task 5: Data Migration
Write a migration that:
- Creates a new `user_preferences` table
- Moves data from `users.settings_json` column (a JSON text blob) into properly normalized `user_preferences` rows
- Drops the old column
- Test both upgrade AND downgrade

### Task 6: Full Project Setup
Create a complete project from scratch:
- Models for a task management app (users, projects, tasks, comments)
- Alembic configuration and initial migration
- Repository layer with CRUD operations
- pytest tests for models and repositories
- Makefile with `migrate`, `rollback`, `test` commands

---

## 📚 Resources

- [Alembic Documentation](https://alembic.sqlalchemy.org/)
- [Alembic Tutorial](https://alembic.sqlalchemy.org/en/latest/tutorial.html)
- [Alembic Auto-generate](https://alembic.sqlalchemy.org/en/latest/autogenerate.html)
- [Database Normalization (Wikipedia)](https://en.wikipedia.org/wiki/Database_normalization)
- [Star Schema (Kimball)](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [SQLAlchemy + Alembic Guide](https://docs.sqlalchemy.org/en/20/core/metadata.html)

---

## 🔑 Key Takeaways

1. **Migrations = Git for database schema.** Every change is versioned, reversible, and tracked.
2. **Alembic `--autogenerate`** — compares your models to the DB and generates migration code. Always review it.
3. **Never edit applied migrations** — write new ones. Migrations in production are frozen.
4. **Always test both upgrade AND downgrade** — rollback capability is critical in production.
5. **Data migrations are separate from schema migrations** — add columns first, migrate data, then add constraints.
6. **Indexes on filter/join/order columns** — dramatic performance improvement. But each index slows writes.
7. **Normalize for OLTP, denormalize for OLAP** — different use cases, different designs.
8. **Star schema** — the standard pattern for data warehouses. Fact tables + dimension tables.
9. **Staging tables** — load raw data first, validate, then move to final tables. Standard DE pattern.
10. **Project structure matters** — models, repositories, migrations, tests in separate layers. Clean and maintainable.

---

> 🎉 **Phase 4 Complete!** You've finished Databases & SQL.
>
> **Next up — Phase 5: Data Engineering Tools (Days 19-25)**
> - Day 19: Pandas Fundamentals
> - Day 20: Pandas Advanced + Polars
> - Day 21-23: FastAPI (REST APIs)
> - Day 24: Docker for Python
> - Day 25: CI/CD & Deployment

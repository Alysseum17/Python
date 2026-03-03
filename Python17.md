# Day 17: SQLAlchemy ORM — Object-Relational Mapping

## 🎯 Goal

Master SQLAlchemy ORM — map Python classes to database tables, query with objects instead of raw SQL. This is the layer you'll use in FastAPI, Django-style apps, and any Python project that talks to a database. Coming from Java, this is like JPA/Hibernate. Coming from JS, this is like TypeORM or Prisma.

ORM sits ON TOP of Core (Day 16). Everything from Day 16 still works — ORM just adds object-oriented convenience.

---

## 1. ORM vs Core — When to Use Which

```
Core (Day 16):
  ✅ Complex analytical queries
  ✅ ETL pipelines, batch processing
  ✅ Dynamic report generation
  ✅ Bulk operations (millions of rows)
  ✅ When you think in SQL

ORM (Today):
  ✅ CRUD operations (create, read, update, delete)
  ✅ Web applications (FastAPI, Flask)
  ✅ Domain models with business logic
  ✅ Relationships between entities
  ✅ When you think in objects

In practice, you'll use BOTH:
  - ORM for your web API models and CRUD
  - Core for analytics, reporting, and ETL
  - They can be mixed in the same project
```

---

## 2. Defining Models — Mapped Classes

### Modern style (SQLAlchemy 2.0 — Mapped + mapped_column)

```python
from datetime import datetime
from sqlalchemy import String, Integer, Float, Boolean, Text, ForeignKey, create_engine
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, relationship,
    Session, sessionmaker,
)

# Base class — all models inherit from this
class Base(DeclarativeBase):
    pass

# User model
class User(Base):
    __tablename__ = "users"
    
    # Columns with type hints — clean and modern
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True)
    age: Mapped[int | None] = mapped_column(Integer, default=None)
    city: Mapped[str] = mapped_column(String(100), default="Unknown")
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(default=datetime.now)
    
    # Relationship — access user.orders to get list of orders
    orders: Mapped[list["Order"]] = relationship(back_populates="user", cascade="all, delete-orphan")
    
    def __repr__(self) -> str:
        return f"User(id={self.id}, name='{self.name}', email='{self.email}')"

# Order model
class Order(Base):
    __tablename__ = "orders"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    product: Mapped[str] = mapped_column(String(200))
    amount: Mapped[float] = mapped_column(Float)
    quantity: Mapped[int] = mapped_column(Integer, default=1)
    status: Mapped[str] = mapped_column(String(20), default="pending")
    ordered_at: Mapped[datetime] = mapped_column(default=datetime.now)
    
    # Relationship — access order.user to get the User object
    user: Mapped["User"] = relationship(back_populates="orders")
    
    def __repr__(self) -> str:
        return f"Order(id={self.id}, product='{self.product}', amount={self.amount})"

    @property
    def total(self) -> float:
        """Business logic on the model."""
        return self.amount * self.quantity
```

### Compare with Java JPA / TypeORM

```java
// Java JPA — very similar concept
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue
    private Long id;
    
    @Column(nullable = false, length = 100)
    private String name;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders;
}
```

```typescript
// TypeORM — also similar
@Entity()
class User {
    @PrimaryGeneratedColumn()
    id: number;
    
    @Column({ length: 100 })
    name: string;
    
    @OneToMany(() => Order, order => order.user)
    orders: Order[];
}
```

```python
# SQLAlchemy 2.0 — Pythonic and type-safe
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    orders: Mapped[list["Order"]] = relationship(back_populates="user")
```

---

## 3. Engine & Session Setup

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

# Create engine
engine = create_engine("sqlite:///app.db", echo=True)

# Create all tables
Base.metadata.create_all(engine)

# ── Option 1: Session directly (simple scripts) ──
with Session(engine) as session:
    # work with session
    session.commit()

# ── Option 2: sessionmaker (web apps — preferred) ──
SessionLocal = sessionmaker(bind=engine)

with SessionLocal() as session:
    # work with session
    session.commit()

# ── Option 3: begin() — auto-commit/rollback ──
with Session(engine) as session:
    with session.begin():
        # auto-commits on success
        # auto-rollbacks on exception
        session.add(User(name="Alice", email="alice@test.com"))
    # committed here

# For FastAPI, you'll use a dependency:
# def get_db():
#     db = SessionLocal()
#     try:
#         yield db
#     finally:
#         db.close()
```

### Session lifecycle

```
Session is a "workspace" for ORM operations:

1. Create/open session
2. Query objects (they're tracked by the session)
3. Add/modify/delete objects
4. Flush (SQL sent to DB, but not committed)
5. Commit (changes permanent) or Rollback (undo)
6. Close session

Think of it like a transaction + object cache.

Java equivalent: EntityManager
JS equivalent: Repository (TypeORM) or PrismaClient
```

---

## 4. CREATE — Adding Objects

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    # Create a single object
    alice = User(name="Alice", email="alice@test.com", age=25, city="Kyiv")
    session.add(alice)
    session.commit()
    
    # alice.id is now set! (auto-generated by DB)
    print(f"Created user: {alice.id}")

    # Create multiple objects
    users = [
        User(name="Bob", email="bob@test.com", age=30, city="London"),
        User(name="Charlie", email="charlie@test.com", age=35, city="NYC"),
        User(name="Diana", email="diana@test.com", age=28, city="Berlin"),
    ]
    session.add_all(users)
    session.commit()

    # Create with relationship
    eve = User(name="Eve", email="eve@test.com", age=32, city="Kyiv")
    eve.orders = [
        Order(product="Laptop", amount=999.99, status="delivered"),
        Order(product="Mouse", amount=29.99, status="shipped"),
    ]
    session.add(eve)  # orders are also saved (cascade)
    session.commit()
    
    print(f"Eve has {len(eve.orders)} orders")
```

---

## 5. READ — Querying Objects

### Basic queries with `select()`

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    # Get all users
    stmt = select(User)
    users = session.scalars(stmt).all()
    # scalars() returns model objects (not Row tuples)
    for user in users:
        print(f"{user.name}, {user.age}, {user.city}")
    
    # Get one user by primary key
    alice = session.get(User, 1)  # by ID — fastest lookup
    print(alice.name)
    
    # Get first match
    stmt = select(User).where(User.name == "Alice")
    alice = session.scalars(stmt).first()
    
    # Get exactly one (raises if 0 or 2+)
    stmt = select(User).where(User.email == "alice@test.com")
    alice = session.scalars(stmt).one()
    
    # Get one or None (raises if 2+, returns None if 0)
    stmt = select(User).where(User.email == "nonexistent@test.com")
    user = session.scalars(stmt).one_or_none()
    print(user)  # None
```

### Filtering

```python
with Session(engine) as session:
    # WHERE conditions — same as Core (Day 16)
    stmt = select(User).where(User.age > 25)
    stmt = select(User).where(User.city == "Kyiv", User.is_active == True)
    stmt = select(User).where(User.city.in_(["Kyiv", "London"]))
    stmt = select(User).where(User.name.like("A%"))
    stmt = select(User).where(User.age.between(25, 35))
    stmt = select(User).where(User.age.is_(None))
    
    # AND / OR
    from sqlalchemy import and_, or_
    stmt = select(User).where(
        or_(
            and_(User.city == "Kyiv", User.age >= 25),
            and_(User.city == "London", User.age >= 30),
        )
    )
    
    # ORDER BY
    stmt = select(User).order_by(User.age.desc())
    
    # LIMIT / OFFSET
    stmt = select(User).order_by(User.id).limit(10).offset(20)
    
    # Execute
    users = session.scalars(stmt).all()
```

### Aggregates and GROUP BY

```python
from sqlalchemy import func, select

with Session(engine) as session:
    # Count
    count = session.scalar(select(func.count()).select_from(User))
    print(f"Total users: {count}")
    
    # Group by city
    stmt = (
        select(User.city, func.count().label("count"))
        .group_by(User.city)
        .order_by(func.count().desc())
    )
    # Note: this returns Row tuples, not User objects
    for row in session.execute(stmt):
        print(f"{row.city}: {row.count}")
    
    # Having
    stmt = (
        select(User.city, func.count().label("count"))
        .group_by(User.city)
        .having(func.count() >= 2)
    )
```

---

## 6. Relationships — The Power of ORM

### One-to-Many

```python
# Already defined in our models:
# User.orders → list of Order objects
# Order.user → single User object

with Session(engine) as session:
    # Access orders through user
    alice = session.scalars(select(User).where(User.name == "Alice")).first()
    
    for order in alice.orders:
        print(f"  {order.product}: ${order.amount}")
    
    # Access user through order
    order = session.scalars(select(Order)).first()
    print(f"Order by: {order.user.name}")
    
    # Add order to existing user
    alice.orders.append(Order(product="Keyboard", amount=79.99))
    session.commit()
```

### Many-to-Many

```python
from sqlalchemy import Table, Column, Integer, ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column, relationship

# Association table (no model class needed)
post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", Integer, ForeignKey("posts.id"), primary_key=True),
    Column("tag_id", Integer, ForeignKey("tags.id"), primary_key=True),
)

class Post(Base):
    __tablename__ = "posts"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    content: Mapped[str] = mapped_column(Text, default="")
    
    # Many-to-many relationship
    tags: Mapped[list["Tag"]] = relationship(
        secondary=post_tags, back_populates="posts"
    )
    author: Mapped["User"] = relationship()
    
    def __repr__(self):
        return f"Post(id={self.id}, title='{self.title}')"

class Tag(Base):
    __tablename__ = "tags"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)
    
    posts: Mapped[list["Post"]] = relationship(
        secondary=post_tags, back_populates="tags"
    )
    
    def __repr__(self):
        return f"Tag(name='{self.name}')"

# Usage
with Session(engine) as session:
    python_tag = Tag(name="python")
    data_tag = Tag(name="data-engineering")
    
    post = Post(
        title="SQLAlchemy Guide",
        author_id=1,
        tags=[python_tag, data_tag],
    )
    session.add(post)
    session.commit()
    
    # Access tags from post
    for tag in post.tags:
        print(tag.name)
    
    # Access posts from tag
    for post in python_tag.posts:
        print(post.title)
```

### One-to-One

```python
class UserProfile(Base):
    __tablename__ = "user_profiles"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), unique=True)
    bio: Mapped[str | None] = mapped_column(Text, default=None)
    avatar_url: Mapped[str | None] = mapped_column(String(500), default=None)
    
    user: Mapped["User"] = relationship(back_populates="profile")

# Add to User class:
# profile: Mapped["UserProfile" | None] = relationship(
#     back_populates="user", uselist=False
# )
# uselist=False makes it return single object, not list
```

---

## 7. UPDATE — Modifying Objects

```python
with Session(engine) as session:
    # Get and modify (most common pattern)
    alice = session.get(User, 1)
    alice.age = 26
    alice.city = "London"
    session.commit()
    # SQLAlchemy tracks changes and generates UPDATE automatically
    
    # Bulk update (more efficient for many rows)
    from sqlalchemy import update
    
    stmt = (
        update(User)
        .where(User.city == "Unknown")
        .values(is_active=False)
    )
    result = session.execute(stmt)
    session.commit()
    print(f"Deactivated {result.rowcount} users")
    
    # Update with expressions
    stmt = (
        update(User)
        .where(User.city == "Kyiv")
        .values(age=User.age + 1)
    )
    session.execute(stmt)
    session.commit()
```

---

## 8. DELETE — Removing Objects

```python
with Session(engine) as session:
    # Delete specific object
    user = session.get(User, 1)
    if user:
        session.delete(user)
        session.commit()
        # If cascade="all, delete-orphan", orders are also deleted
    
    # Bulk delete
    from sqlalchemy import delete
    
    stmt = delete(User).where(User.is_active == False)
    result = session.execute(stmt)
    session.commit()
    print(f"Deleted {result.rowcount} users")
    
    # Delete with subquery
    stmt = (
        delete(Order)
        .where(
            Order.user_id.not_in(select(User.id))
        )
    )
    session.execute(stmt)
    session.commit()
```

---

## 9. Eager Loading — Solving the N+1 Problem

### The N+1 problem

```python
# ❌ N+1 PROBLEM — the most common ORM performance issue!
with Session(engine) as session:
    users = session.scalars(select(User)).all()  # 1 query: SELECT * FROM users
    
    for user in users:
        print(user.name, len(user.orders))
        # Each user.orders triggers a SEPARATE query!
        # SELECT * FROM orders WHERE user_id = 1
        # SELECT * FROM orders WHERE user_id = 2
        # SELECT * FROM orders WHERE user_id = 3
        # ... N more queries!
    
    # If you have 100 users: 1 + 100 = 101 queries!
    # This is called "N+1" and it kills performance.
```

### Eager loading solutions

```python
from sqlalchemy.orm import joinedload, selectinload, subqueryload

with Session(engine) as session:
    # ── selectinload — best for collections (one-to-many) ──
    stmt = select(User).options(selectinload(User.orders))
    users = session.scalars(stmt).all()
    # Query 1: SELECT * FROM users
    # Query 2: SELECT * FROM orders WHERE user_id IN (1, 2, 3, ...)
    # Only 2 queries total! Regardless of user count.
    
    for user in users:
        print(user.name, len(user.orders))  # no extra queries!
    
    # ── joinedload — best for single objects (many-to-one) ──
    stmt = select(Order).options(joinedload(Order.user))
    orders = session.scalars(stmt).unique().all()
    # Single query with JOIN:
    # SELECT orders.*, users.* FROM orders JOIN users ON ...
    
    for order in orders:
        print(order.product, order.user.name)  # no extra query!
    
    # ── Nested eager loading ──
    stmt = (
        select(User)
        .options(
            selectinload(User.orders),
            selectinload(User.profile),
        )
    )
    
    # ── Deep nested ──
    stmt = (
        select(User)
        .options(
            selectinload(User.orders).selectinload(Order.items),
        )
    )

# When to use which:
# selectinload → one-to-many (user → orders) — separate IN query
# joinedload   → many-to-one (order → user) — single JOIN
# subqueryload → one-to-many with complex filters — subquery
```

---

## 10. JOINs in ORM

```python
from sqlalchemy import select, func
from sqlalchemy.orm import Session

with Session(engine) as session:
    # Implicit join via relationship
    stmt = (
        select(User.name, Order.product, Order.amount)
        .join(User.orders)  # uses relationship definition
        .order_by(Order.amount.desc())
    )
    
    # Explicit join condition
    stmt = (
        select(User.name, Order.product)
        .join(Order, User.id == Order.user_id)
    )
    
    # LEFT OUTER JOIN
    stmt = (
        select(
            User.name,
            func.count(Order.id).label("order_count"),
            func.coalesce(func.sum(Order.amount), 0).label("total"),
        )
        .outerjoin(Order)
        .group_by(User.id, User.name)
        .order_by(func.sum(Order.amount).desc().nullslast())
    )
    
    for row in session.execute(stmt):
        print(f"{row.name}: {row.order_count} orders, ${row.total:.2f}")
    
    # Join multiple tables
    stmt = (
        select(User.name, Order.product, Tag.name)
        .join(Order, User.id == Order.user_id)
        .join(Post, User.id == Post.author_id)
        .join(post_tags, Post.id == post_tags.c.post_id)
        .join(Tag, post_tags.c.tag_id == Tag.id)
    )
```

---

## 11. Advanced Patterns

### Hybrid properties — computed attributes

```python
from sqlalchemy.ext.hybrid import hybrid_property

class User(Base):
    __tablename__ = "users"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    first_name: Mapped[str] = mapped_column(String(50))
    last_name: Mapped[str] = mapped_column(String(50))
    birth_year: Mapped[int] = mapped_column(Integer)
    
    @hybrid_property
    def full_name(self) -> str:
        """Works on Python objects."""
        return f"{self.first_name} {self.last_name}"
    
    @full_name.expression
    @classmethod
    def full_name(cls):
        """Works in SQL queries."""
        return cls.first_name + " " + cls.last_name
    
    @hybrid_property
    def approximate_age(self) -> int:
        return 2025 - self.birth_year

# Now you can:
user = session.get(User, 1)
print(user.full_name)  # Python property

# AND use in queries:
stmt = select(User).where(User.full_name == "Alice Smith")
# Generates: WHERE first_name || ' ' || last_name = 'Alice Smith'
```

### Model mixins — reusable columns

```python
from datetime import datetime
from sqlalchemy.orm import Mapped, mapped_column

class TimestampMixin:
    """Add created_at and updated_at to any model."""
    created_at: Mapped[datetime] = mapped_column(default=datetime.now)
    updated_at: Mapped[datetime] = mapped_column(
        default=datetime.now, onupdate=datetime.now
    )

class SoftDeleteMixin:
    """Add soft delete capability."""
    is_deleted: Mapped[bool] = mapped_column(Boolean, default=False)
    deleted_at: Mapped[datetime | None] = mapped_column(default=None)

class User(TimestampMixin, SoftDeleteMixin, Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    # Automatically has: created_at, updated_at, is_deleted, deleted_at
```

### Events — hooks on model lifecycle

```python
from sqlalchemy import event

@event.listens_for(User, "before_insert")
def user_before_insert(mapper, connection, target):
    """Normalize data before saving."""
    target.email = target.email.lower().strip()
    target.name = target.name.strip()

@event.listens_for(User, "before_update")
def user_before_update(mapper, connection, target):
    """Track modifications."""
    target.updated_at = datetime.now()

@event.listens_for(Session, "before_commit")
def before_commit(session):
    """Log all pending changes before commit."""
    for obj in session.new:
        print(f"INSERT: {obj}")
    for obj in session.dirty:
        print(f"UPDATE: {obj}")
    for obj in session.deleted:
        print(f"DELETE: {obj}")
```

---

## 12. Repository Pattern with ORM

```python
from sqlalchemy import select, func, and_
from sqlalchemy.orm import Session, selectinload
from typing import Sequence

class UserRepository:
    """Clean data access layer using ORM."""
    
    def __init__(self, session: Session):
        self.session = session
    
    def create(self, name: str, email: str, age: int, city: str = "Unknown") -> User:
        user = User(name=name, email=email, age=age, city=city)
        self.session.add(user)
        self.session.flush()  # get the ID without committing
        return user
    
    def get_by_id(self, user_id: int) -> User | None:
        return self.session.get(User, user_id)
    
    def get_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        return self.session.scalars(stmt).one_or_none()
    
    def get_with_orders(self, user_id: int) -> User | None:
        """Get user with eagerly loaded orders."""
        stmt = (
            select(User)
            .options(selectinload(User.orders))
            .where(User.id == user_id)
        )
        return self.session.scalars(stmt).one_or_none()
    
    def list_all(
        self,
        city: str | None = None,
        min_age: int | None = None,
        is_active: bool | None = None,
        limit: int = 100,
        offset: int = 0,
    ) -> Sequence[User]:
        """List users with optional filters."""
        stmt = select(User)
        
        conditions = []
        if city:
            conditions.append(User.city == city)
        if min_age is not None:
            conditions.append(User.age >= min_age)
        if is_active is not None:
            conditions.append(User.is_active == is_active)
        
        if conditions:
            stmt = stmt.where(and_(*conditions))
        
        stmt = stmt.order_by(User.id).limit(limit).offset(offset)
        return self.session.scalars(stmt).all()
    
    def update(self, user_id: int, **fields) -> User | None:
        user = self.get_by_id(user_id)
        if not user:
            return None
        
        allowed = {"name", "email", "age", "city", "is_active"}
        for key, value in fields.items():
            if key in allowed:
                setattr(user, key, value)
        
        self.session.flush()
        return user
    
    def delete(self, user_id: int) -> bool:
        user = self.get_by_id(user_id)
        if not user:
            return False
        self.session.delete(user)
        self.session.flush()
        return True
    
    def count(self, is_active: bool | None = None) -> int:
        stmt = select(func.count()).select_from(User)
        if is_active is not None:
            stmt = stmt.where(User.is_active == is_active)
        return self.session.scalar(stmt)
    
    def get_top_spenders(self, limit: int = 10) -> list[dict]:
        """Complex query mixing ORM with aggregate functions."""
        stmt = (
            select(
                User.name,
                User.email,
                func.count(Order.id).label("order_count"),
                func.coalesce(func.sum(Order.amount), 0).label("total_spent"),
            )
            .outerjoin(Order)
            .group_by(User.id)
            .order_by(func.sum(Order.amount).desc().nullslast())
            .limit(limit)
        )
        return [row._asdict() for row in self.session.execute(stmt)]

# Usage
with Session(engine) as session:
    repo = UserRepository(session)
    
    user = repo.create("Test User", "test@test.com", 25, "Kyiv")
    print(f"Created: {user}")
    
    users = repo.list_all(city="Kyiv", min_age=20)
    print(f"Found: {len(users)} users")
    
    spenders = repo.get_top_spenders(5)
    for s in spenders:
        print(f"  {s['name']}: ${s['total_spent']:.2f}")
    
    session.commit()
```

---

## 13. Unit of Work — Session Internals

```python
with Session(engine) as session:
    # The session tracks all objects you interact with
    
    alice = User(name="Alice", email="alice@test.com", age=25)
    session.add(alice)
    
    # alice is now "pending" — tracked but not in DB yet
    print(alice in session.new)  # True
    
    session.flush()
    # NOW the INSERT SQL is sent to DB (but not committed)
    # alice.id is available
    print(alice in session.new)      # False
    print(session.is_modified(alice)) # False (just flushed)
    
    alice.age = 26
    print(session.is_modified(alice)) # True (dirty)
    print(alice in session.dirty)     # True
    
    session.commit()
    # Transaction committed — changes are permanent
    
    # Expunge — remove object from session tracking
    session.expunge(alice)
    # alice is now "detached" — changes won't be tracked
    
    # Refresh — reload from database
    bob = session.get(User, 2)
    session.refresh(bob)  # re-reads from DB

    # Merge — take a detached object and re-attach it
    alice.age = 30
    merged_alice = session.merge(alice)
    session.commit()
```

### Identity map

```python
with Session(engine) as session:
    # Session ensures one object per primary key (identity map)
    user1 = session.get(User, 1)
    user2 = session.get(User, 1)
    
    print(user1 is user2)  # True! Same object in memory
    # Second get() doesn't hit the database
    
    # This prevents inconsistencies:
    # If you modify user1, user2 reflects the same changes
    user1.name = "Changed"
    print(user2.name)  # "Changed"
```

---

## 14. Async ORM (for FastAPI)

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy import select

# Async engine (note: different driver!)
async_engine = create_async_engine(
    "sqlite+aiosqlite:///app.db",  # aiosqlite for async SQLite
    # "postgresql+asyncpg://user:pass@localhost/mydb",  # asyncpg for async PostgreSQL
    echo=True,
)

# Async session factory
AsyncSessionLocal = async_sessionmaker(async_engine, class_=AsyncSession)

# Create tables
async def init_db():
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

# Async CRUD
async def create_user(name: str, email: str, age: int) -> User:
    async with AsyncSessionLocal() as session:
        user = User(name=name, email=email, age=age)
        session.add(user)
        await session.commit()
        await session.refresh(user)  # load generated ID
        return user

async def get_users(city: str | None = None) -> list[User]:
    async with AsyncSessionLocal() as session:
        stmt = select(User)
        if city:
            stmt = stmt.where(User.city == city)
        result = await session.scalars(stmt)
        return result.all()

async def get_user_with_orders(user_id: int) -> User | None:
    async with AsyncSessionLocal() as session:
        stmt = (
            select(User)
            .options(selectinload(User.orders))
            .where(User.id == user_id)
        )
        return await session.scalar(stmt)

# FastAPI integration pattern:
# async def get_db():
#     async with AsyncSessionLocal() as session:
#         yield session
#
# @app.get("/users/{user_id}")
# async def read_user(user_id: int, db: AsyncSession = Depends(get_db)):
#     user = await db.get(User, user_id)
#     return user
```

---

## 15. Testing ORM Code

```python
import pytest
from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

@pytest.fixture
def engine():
    """In-memory database for testing."""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    return engine

@pytest.fixture
def session(engine):
    """Fresh session for each test."""
    with Session(engine) as session:
        yield session
        session.rollback()  # undo any changes after each test

@pytest.fixture
def sample_users(session):
    """Pre-populated test data."""
    users = [
        User(name="Alice", email="alice@test.com", age=25, city="Kyiv"),
        User(name="Bob", email="bob@test.com", age=30, city="London"),
        User(name="Charlie", email="charlie@test.com", age=35, city="Kyiv"),
    ]
    session.add_all(users)
    session.flush()
    return users

class TestUserRepository:
    
    def test_create_user(self, session):
        repo = UserRepository(session)
        user = repo.create("Test", "test@test.com", 25, "Kyiv")
        
        assert user.id is not None
        assert user.name == "Test"
        assert user.email == "test@test.com"
    
    def test_get_by_id(self, session, sample_users):
        repo = UserRepository(session)
        user = repo.get_by_id(sample_users[0].id)
        
        assert user is not None
        assert user.name == "Alice"
    
    def test_get_by_id_not_found(self, session):
        repo = UserRepository(session)
        user = repo.get_by_id(999)
        assert user is None
    
    def test_list_by_city(self, session, sample_users):
        repo = UserRepository(session)
        kyiv_users = repo.list_all(city="Kyiv")
        
        assert len(kyiv_users) == 2
        assert all(u.city == "Kyiv" for u in kyiv_users)
    
    def test_update_user(self, session, sample_users):
        repo = UserRepository(session)
        updated = repo.update(sample_users[0].id, age=26, city="London")
        
        assert updated is not None
        assert updated.age == 26
        assert updated.city == "London"
    
    def test_delete_user(self, session, sample_users):
        repo = UserRepository(session)
        result = repo.delete(sample_users[0].id)
        
        assert result is True
        assert repo.get_by_id(sample_users[0].id) is None
    
    def test_count(self, session, sample_users):
        repo = UserRepository(session)
        assert repo.count() == 3
```

---

## 📝 Practice Tasks

### Task 1: Blog Platform Models
Define ORM models for: `Author`, `Post`, `Comment`, `Tag` (many-to-many with Post). Include timestamps mixin, soft delete, proper relationships. Write CRUD operations for each. Test with pytest.

### Task 2: E-Commerce Models
Define: `Customer`, `Product`, `Order`, `OrderItem`, `Review`. Order has many OrderItems. Product has many Reviews. Implement: create order with items, calculate total, get customer order history with items eagerly loaded. Solve N+1.

### Task 3: Repository Pattern
Build complete repositories for Task 1 or 2 with:
- All CRUD operations
- Complex queries (top authors, best-selling products)
- Dynamic filtering (search by multiple optional criteria)
- Pagination support
- Full pytest test coverage

### Task 4: Relationship Deep Dive
Create models that demonstrate all relationship types:
- One-to-one (User ↔ Profile)
- One-to-many (User → Posts)
- Many-to-many (Post ↔ Tags)
- Self-referential (Employee → manager)
Write queries that traverse relationships in both directions.

### Task 5: Migration from Raw SQL
Take the analytics queries from Day 15 (section 12) and rewrite them using ORM: daily active users, conversion funnel, revenue by week with LAG, top users by lifetime value. Compare the generated SQL.

### Task 6: Async ORM API
Build a simple async CRUD service using async SQLAlchemy:
- Async engine + session setup
- Async repository with CRUD methods
- Async function that creates 100 users concurrently
- Test with pytest-asyncio

---

## 📚 Resources

- [SQLAlchemy 2.0 ORM Tutorial](https://docs.sqlalchemy.org/en/20/tutorial/)
- [SQLAlchemy ORM Mapped Classes](https://docs.sqlalchemy.org/en/20/orm/mapping_styles.html)
- [SQLAlchemy Relationships](https://docs.sqlalchemy.org/en/20/orm/relationships.html)
- [SQLAlchemy Session Basics](https://docs.sqlalchemy.org/en/20/orm/session_basics.html)
- [SQLAlchemy Eager Loading](https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html)
- [Real Python — SQLAlchemy ORM](https://realpython.com/python-sqlite-sqlalchemy/)
- [FastAPI + SQLAlchemy Guide](https://fastapi.tiangolo.com/tutorial/sql-databases/)

---

## 🔑 Key Takeaways

1. **ORM = Python classes ↔ database tables.** Objects become rows, attributes become columns.
2. **`Mapped[type]` + `mapped_column()`** — the modern 2.0 way. Type-safe and clean.
3. **`relationship()`** — defines how models connect. `back_populates` for bidirectional access.
4. **Session is a workspace** — tracks changes, flushes SQL, commits transactions.
5. **`session.get(Model, id)`** — fastest way to get by primary key. Uses identity map cache.
6. **`session.scalars(stmt).all()`** — the standard query pattern in 2.0.
7. **N+1 problem is real** — use `selectinload` for collections, `joinedload` for single objects.
8. **Repository pattern** — keep ORM code in one layer. Business logic doesn't touch sessions directly.
9. **`flush()` vs `commit()`** — flush sends SQL (gets IDs), commit makes it permanent.
10. **Test with in-memory SQLite** — fast, isolated, auto-cleaned. Use `session.rollback()` in fixtures.

---

> **Tomorrow (Day 18):** Database Design & Migrations — Alembic for schema migrations, normalization, indexes, and database design patterns for Data Engineering.

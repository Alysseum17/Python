# Day 19: FastAPI Fundamentals

## 🎯 Goal

Build REST APIs with FastAPI — the modern, fast, async Python web framework. FastAPI is built on type hints and Pydantic, so everything you learned on Days 5 (Pydantic) and 12 (async) comes together here. Coming from Java, this is Spring Boot but lighter. Coming from JS, this is Express but with automatic validation and documentation.

FastAPI is the #1 choice for Python APIs in Data Engineering — it powers data platforms, ML model serving, webhook receivers, and internal tools.

---

## 1. Why FastAPI

```
FastAPI advantages:
  ✅ Automatic request validation (via Pydantic)
  ✅ Automatic API docs (Swagger UI + ReDoc) — zero effort
  ✅ Async by default (built on Starlette + uvicorn)
  ✅ Type hints everywhere — IDE autocompletion, fewer bugs
  ✅ Performance — one of the fastest Python frameworks
  ✅ Dependency injection — clean, testable code
  ✅ Easy to learn — minimal boilerplate

Compare:
  Flask:      simple but manual validation, no async, no docs
  Django:     batteries included but heavy, monolithic
  Express.js: minimal like Flask, no built-in validation
  Spring Boot: similar features but MUCH more boilerplate
  FastAPI:    modern sweet spot — fast, validated, documented
```

---

## 2. Hello World

### Installation

```bash
pip install fastapi uvicorn[standard]
# fastapi — the framework
# uvicorn — ASGI server (runs your app)
```

### First app

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello, World!"}

@app.get("/health")
async def health_check():
    return {"status": "ok"}
```

```bash
# Run the server
uvicorn main:app --reload
# main   = filename (main.py)
# app    = FastAPI instance name
# --reload = auto-restart on code changes (dev only)

# Server starts at http://127.0.0.1:8000
# Swagger docs at http://127.0.0.1:8000/docs      ← try this!
# ReDoc docs at http://127.0.0.1:8000/redoc
```

### Compare with Express.js and Spring Boot

```javascript
// Express.js
const app = express();
app.get("/", (req, res) => {
    res.json({ message: "Hello, World!" });
});
app.listen(8000);
```

```java
// Spring Boot
@RestController
public class HelloController {
    @GetMapping("/")
    public Map<String, String> root() {
        return Map.of("message", "Hello, World!");
    }
}
```

```python
# FastAPI — cleanest of all three
@app.get("/")
async def root():
    return {"message": "Hello, World!"}
```

---

## 3. Path Parameters

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

# Basic path parameter
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    # user_id is automatically parsed as int!
    # If someone sends /users/abc → automatic 422 error
    return {"user_id": user_id}

# Multiple path parameters
@app.get("/users/{user_id}/orders/{order_id}")
async def get_user_order(user_id: int, order_id: int):
    return {"user_id": user_id, "order_id": order_id}

# String path parameter
@app.get("/files/{file_path:path}")
async def get_file(file_path: str):
    # :path captures the full remaining path including slashes
    # /files/data/2025/report.csv → file_path = "data/2025/report.csv"
    return {"file_path": file_path}

# Enum path parameter — restrict to specific values
from enum import Enum

class UserRole(str, Enum):
    admin = "admin"
    editor = "editor"
    viewer = "viewer"

@app.get("/roles/{role}")
async def get_role_info(role: UserRole):
    # Only "admin", "editor", "viewer" accepted
    # Anything else → automatic 422 error
    return {"role": role, "permissions": f"permissions for {role.value}"}

# ⚠️ Order matters! Fixed paths BEFORE parameterized paths
@app.get("/users/me")       # this must come FIRST
async def get_current_user():
    return {"user": "current"}

@app.get("/users/{user_id}")  # this comes SECOND
async def get_user(user_id: int):
    return {"user_id": user_id}
# If reversed, /users/me would match {user_id} and fail parsing "me" as int
```

---

## 4. Query Parameters

```python
from fastapi import FastAPI, Query

app = FastAPI()

# Basic query parameters (after ?)
@app.get("/users")
async def list_users(skip: int = 0, limit: int = 10):
    # GET /users?skip=20&limit=10
    return {"skip": skip, "limit": limit}

# Required query parameter (no default value)
@app.get("/search")
async def search(q: str):
    # GET /search?q=hello → works
    # GET /search         → 422 error (q is required)
    return {"query": q}

# Optional query parameter
@app.get("/items")
async def list_items(
    category: str | None = None,
    min_price: float | None = None,
    max_price: float | None = None,
    in_stock: bool = True,
):
    # GET /items?category=electronics&min_price=10&in_stock=true
    filters = {}
    if category:
        filters["category"] = category
    if min_price is not None:
        filters["min_price"] = min_price
    if max_price is not None:
        filters["max_price"] = max_price
    filters["in_stock"] = in_stock
    return {"filters": filters}

# Query parameter validation with Query()
@app.get("/products")
async def list_products(
    q: str | None = Query(
        default=None,
        min_length=2,
        max_length=100,
        description="Search query",
        examples=["laptop"],
    ),
    page: int = Query(default=1, ge=1, description="Page number"),
    size: int = Query(default=20, ge=1, le=100, description="Items per page"),
    sort_by: str = Query(default="name", pattern="^(name|price|rating)$"),
):
    return {"q": q, "page": page, "size": size, "sort_by": sort_by}
    # Validation is automatic:
    # page=0 → 422 (ge=1 means >= 1)
    # size=200 → 422 (le=100 means <= 100)
    # sort_by=invalid → 422 (must match regex pattern)

# List query parameter (multiple values)
@app.get("/filter")
async def filter_items(tags: list[str] = Query(default=[])):
    # GET /filter?tags=python&tags=fastapi&tags=async
    return {"tags": tags}
```

---

## 5. Request Body — Pydantic Models

This is where FastAPI shines. Pydantic models (Day 5) become your request/response schemas.

### Basic request body

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field, EmailStr

app = FastAPI()

# Request schema
class UserCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: str = Field(pattern=r"^[\w.-]+@[\w.-]+\.\w+$")
    age: int = Field(ge=0, le=150)
    city: str = "Unknown"

# Response schema (includes id and timestamps)
class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    age: int
    city: str
    is_active: bool = True

    model_config = {"from_attributes": True}
    # from_attributes=True allows creating from ORM objects:
    # UserResponse.model_validate(db_user)

@app.post("/users", response_model=UserResponse, status_code=201)
async def create_user(user: UserCreate):
    # user is ALREADY validated by Pydantic!
    # If validation fails → automatic 422 with detailed errors
    
    # Simulate DB save
    new_user = {
        "id": 1,
        "name": user.name,
        "email": user.email,
        "age": user.age,
        "city": user.city,
        "is_active": True,
    }
    return new_user

# What happens on invalid input:
# POST /users {"name": "", "email": "not-an-email", "age": -5}
# Response 422:
# {
#   "detail": [
#     {"loc": ["body", "name"], "msg": "String should have at least 1 character"},
#     {"loc": ["body", "email"], "msg": "String should match pattern..."},
#     {"loc": ["body", "age"], "msg": "Input should be greater than or equal to 0"}
#   ]
# }
```

### Separate schemas for Create / Update / Response

```python
from pydantic import BaseModel, Field
from datetime import datetime

# Base schema — shared fields
class UserBase(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: str
    city: str = "Unknown"

# Create — required fields for creation
class UserCreate(UserBase):
    age: int = Field(ge=0, le=150)
    password: str = Field(min_length=8)

# Update — all fields optional (partial update)
class UserUpdate(BaseModel):
    name: str | None = None
    email: str | None = None
    age: int | None = Field(default=None, ge=0, le=150)
    city: str | None = None

# Response — what API returns (no password!)
class UserResponse(UserBase):
    id: int
    age: int
    is_active: bool
    created_at: datetime

    model_config = {"from_attributes": True}

# List response — with pagination metadata
class UserListResponse(BaseModel):
    items: list[UserResponse]
    total: int
    page: int
    size: int
    pages: int

@app.post("/users", response_model=UserResponse, status_code=201)
async def create_user(user: UserCreate):
    # user.password is available here but NOT in response
    ...

@app.patch("/users/{user_id}", response_model=UserResponse)
async def update_user(user_id: int, user: UserUpdate):
    # Only provided fields are updated
    update_data = user.model_dump(exclude_unset=True)
    # {"name": "New Name"} — only fields the client sent
    ...

@app.get("/users", response_model=UserListResponse)
async def list_users(page: int = 1, size: int = 20):
    ...
```

### Nested models

```python
class Address(BaseModel):
    street: str
    city: str
    country: str
    zip_code: str | None = None

class OrderItem(BaseModel):
    product_id: int
    quantity: int = Field(ge=1)
    unit_price: float = Field(gt=0)

class OrderCreate(BaseModel):
    customer_id: int
    shipping_address: Address  # nested object
    items: list[OrderItem] = Field(min_length=1)  # at least 1 item
    notes: str | None = None

@app.post("/orders", status_code=201)
async def create_order(order: OrderCreate):
    total = sum(item.quantity * item.unit_price for item in order.items)
    return {
        "customer_id": order.customer_id,
        "total": total,
        "items_count": len(order.items),
        "shipping_city": order.shipping_address.city,
    }

# POST /orders
# {
#     "customer_id": 1,
#     "shipping_address": {
#         "street": "123 Main St",
#         "city": "Kyiv",
#         "country": "Ukraine"
#     },
#     "items": [
#         {"product_id": 1, "quantity": 2, "unit_price": 29.99},
#         {"product_id": 5, "quantity": 1, "unit_price": 99.99}
#     ]
# }
```

---

## 6. Response Handling

### Status codes

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()

@app.post("/users", status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate):
    return {"id": 1, **user.model_dump()}

@app.delete("/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int):
    # 204 = success with no response body
    return None

# Common status codes:
# 200 OK             — default for GET, PUT, PATCH
# 201 Created        — for POST that creates something
# 204 No Content     — for DELETE
# 400 Bad Request    — client sent wrong data
# 401 Unauthorized   — not authenticated
# 403 Forbidden      — authenticated but not allowed
# 404 Not Found      — resource doesn't exist
# 422 Unprocessable  — validation error (FastAPI default for bad input)
# 500 Internal Error — server bug
```

### HTTPException — error responses

```python
from fastapi import HTTPException

# In-memory "database" for examples
fake_users_db = {
    1: {"id": 1, "name": "Alice", "email": "alice@test.com"},
    2: {"id": 2, "name": "Bob", "email": "bob@test.com"},
}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    if user_id not in fake_users_db:
        raise HTTPException(
            status_code=404,
            detail=f"User {user_id} not found",
        )
    return fake_users_db[user_id]

# Custom error with headers
@app.get("/protected")
async def protected_route():
    raise HTTPException(
        status_code=401,
        detail="Authentication required",
        headers={"WWW-Authenticate": "Bearer"},
    )

# Custom error model
class ErrorResponse(BaseModel):
    error: str
    detail: str
    timestamp: datetime

@app.get("/items/{item_id}", responses={404: {"model": ErrorResponse}})
async def get_item(item_id: int):
    ...
```

---

## 7. Dependency Injection

FastAPI's DI system is simpler than Spring's but equally powerful.

### Basic dependencies

```python
from fastapi import FastAPI, Depends, Query

app = FastAPI()

# A dependency is just a function (or callable)
def pagination_params(
    page: int = Query(default=1, ge=1),
    size: int = Query(default=20, ge=1, le=100),
):
    """Reusable pagination parameters."""
    return {"page": page, "size": size, "skip": (page - 1) * size}

@app.get("/users")
async def list_users(pagination: dict = Depends(pagination_params)):
    return {
        "page": pagination["page"],
        "size": pagination["size"],
        "skip": pagination["skip"],
    }

@app.get("/products")
async def list_products(pagination: dict = Depends(pagination_params)):
    # Same pagination logic reused!
    return {"pagination": pagination}
```

### Class-based dependencies

```python
class PaginationParams:
    """Reusable pagination as a class."""
    
    def __init__(
        self,
        page: int = Query(default=1, ge=1),
        size: int = Query(default=20, ge=1, le=100),
    ):
        self.page = page
        self.size = size
        self.skip = (page - 1) * size
        self.limit = size

@app.get("/users")
async def list_users(pagination: PaginationParams = Depends()):
    # Depends() without arguments uses the class itself
    return {"page": pagination.page, "skip": pagination.skip}
```

### Database dependency (most common pattern)

```python
from sqlalchemy.orm import Session

# Database session dependency
def get_db():
    """Yield a database session, close after request."""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.post("/users", status_code=201)
async def create_user(user: UserCreate, db: Session = Depends(get_db)):
    db_user = User(**user.model_dump())
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user
```

### Dependency chains

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_user_repository(db: Session = Depends(get_db)):
    """Depends on get_db — automatically resolved."""
    return UserRepository(db)

@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    repo: UserRepository = Depends(get_user_repository),
):
    # FastAPI resolves: get_db → get_user_repository → inject into handler
    user = repo.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404)
    return user
```

### Auth dependency

```python
from fastapi import Depends, HTTPException, Header

async def verify_api_key(x_api_key: str = Header()):
    """Simple API key authentication."""
    if x_api_key != "secret-api-key-123":
        raise HTTPException(status_code=401, detail="Invalid API key")
    return x_api_key

@app.get("/protected")
async def protected_route(api_key: str = Depends(verify_api_key)):
    return {"message": "You have access!", "key": api_key}

# Apply dependency to all routes in a router
from fastapi import APIRouter

protected_router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(verify_api_key)],  # all routes require API key
)

@protected_router.get("/stats")
async def admin_stats():
    return {"users": 100, "orders": 500}
```

---

## 8. Routers — Organizing Your API

### Split into modules

```python
# routers/users.py
from fastapi import APIRouter, Depends, HTTPException

router = APIRouter(
    prefix="/users",
    tags=["Users"],  # groups in Swagger docs
)

@router.get("/")
async def list_users():
    return []

@router.get("/{user_id}")
async def get_user(user_id: int):
    return {"id": user_id}

@router.post("/", status_code=201)
async def create_user(user: UserCreate):
    return {"id": 1, **user.model_dump()}
```

```python
# routers/orders.py
from fastapi import APIRouter

router = APIRouter(prefix="/orders", tags=["Orders"])

@router.get("/")
async def list_orders():
    return []

@router.get("/{order_id}")
async def get_order(order_id: int):
    return {"id": order_id}
```

```python
# main.py — assemble the app
from fastapi import FastAPI
from routers import users, orders

app = FastAPI(
    title="My API",
    description="A sample REST API",
    version="1.0.0",
)

app.include_router(users.router)
app.include_router(orders.router)

# API versioning with prefix
# app.include_router(users.router, prefix="/api/v1")
# app.include_router(orders.router, prefix="/api/v1")
```

### Project structure

```
myapi/
├── main.py              ← FastAPI app, include routers
├── config.py            ← Settings (Pydantic BaseSettings)
├── database.py          ← Engine, session, get_db dependency
├── models/              ← SQLAlchemy ORM models
│   ├── __init__.py
│   ├── user.py
│   └── order.py
├── schemas/             ← Pydantic request/response schemas
│   ├── __init__.py
│   ├── user.py
│   └── order.py
├── routers/             ← API route handlers
│   ├── __init__.py
│   ├── users.py
│   └── orders.py
├── repositories/        ← Database access layer
│   ├── __init__.py
│   └── user_repo.py
├── services/            ← Business logic
│   ├── __init__.py
│   └── user_service.py
├── dependencies.py      ← Shared dependencies
├── tests/
│   ├── conftest.py
│   ├── test_users.py
│   └── test_orders.py
└── alembic/             ← Database migrations
```

---

## 9. Testing FastAPI

### Using TestClient

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from main import app
from database import get_db
from models import Base

# Test database
TEST_ENGINE = create_engine("sqlite:///:memory:")
TestSession = sessionmaker(bind=TEST_ENGINE)

@pytest.fixture(autouse=True)
def setup_db():
    """Create tables before each test, drop after."""
    Base.metadata.create_all(TEST_ENGINE)
    yield
    Base.metadata.drop_all(TEST_ENGINE)

@pytest.fixture
def db():
    """Test database session."""
    session = TestSession()
    try:
        yield session
    finally:
        session.rollback()
        session.close()

@pytest.fixture
def client(db):
    """Test client with overridden database."""
    def override_get_db():
        try:
            yield db
        finally:
            pass
    
    app.dependency_overrides[get_db] = override_get_db
    
    with TestClient(app) as c:
        yield c
    
    app.dependency_overrides.clear()
```

```python
# tests/test_users.py

def test_create_user(client):
    response = client.post("/users", json={
        "name": "Alice",
        "email": "alice@test.com",
        "age": 25,
        "password": "securepass123",
    })
    
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Alice"
    assert data["email"] == "alice@test.com"
    assert "password" not in data  # should not be in response!
    assert "id" in data

def test_create_user_invalid_email(client):
    response = client.post("/users", json={
        "name": "Bob",
        "email": "not-an-email",
        "age": 25,
        "password": "securepass123",
    })
    
    assert response.status_code == 422

def test_get_user(client):
    # Create first
    client.post("/users", json={
        "name": "Alice",
        "email": "alice@test.com",
        "age": 25,
        "password": "securepass123",
    })
    
    # Then get
    response = client.get("/users/1")
    assert response.status_code == 200
    assert response.json()["name"] == "Alice"

def test_get_user_not_found(client):
    response = client.get("/users/999")
    assert response.status_code == 404

def test_list_users_with_pagination(client):
    # Create multiple users
    for i in range(25):
        client.post("/users", json={
            "name": f"User {i}",
            "email": f"user{i}@test.com",
            "age": 20 + i,
            "password": "securepass123",
        })
    
    # Test pagination
    response = client.get("/users?page=2&size=10")
    assert response.status_code == 200
    data = response.json()
    assert len(data["items"]) == 10
    assert data["total"] == 25
    assert data["page"] == 2

def test_update_user(client):
    client.post("/users", json={
        "name": "Alice",
        "email": "alice@test.com",
        "age": 25,
        "password": "securepass123",
    })
    
    response = client.patch("/users/1", json={"name": "Alice Updated"})
    assert response.status_code == 200
    assert response.json()["name"] == "Alice Updated"

def test_delete_user(client):
    client.post("/users", json={
        "name": "Alice",
        "email": "alice@test.com",
        "age": 25,
        "password": "securepass123",
    })
    
    response = client.delete("/users/1")
    assert response.status_code == 204
    
    response = client.get("/users/1")
    assert response.status_code == 404
```

---

## 10. Request Extras

### Headers

```python
from fastapi import Header

@app.get("/info")
async def get_info(
    user_agent: str = Header(),
    accept_language: str | None = Header(default=None),
    x_request_id: str | None = Header(default=None),
):
    return {
        "user_agent": user_agent,
        "language": accept_language,
        "request_id": x_request_id,
    }
# FastAPI auto-converts header names:
# X-Request-Id → x_request_id (lowercase, underscores)
```

### Cookies

```python
from fastapi import Cookie
from fastapi.responses import JSONResponse

@app.get("/session")
async def get_session(session_id: str | None = Cookie(default=None)):
    return {"session_id": session_id}

@app.post("/login")
async def login():
    response = JSONResponse(content={"message": "logged in"})
    response.set_cookie(key="session_id", value="abc123", httponly=True)
    return response
```

### Form data and file uploads

```python
from fastapi import File, UploadFile, Form

@app.post("/upload")
async def upload_file(
    file: UploadFile = File(...),
    description: str = Form(default=""),
):
    contents = await file.read()
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents),
        "description": description,
    }

@app.post("/upload-multiple")
async def upload_files(files: list[UploadFile] = File(...)):
    return [
        {"filename": f.filename, "size": f.size}
        for f in files
    ]
```

---

## 11. App Configuration

### Pydantic Settings

```python
# config.py
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    app_name: str = "My API"
    debug: bool = False
    database_url: str = "sqlite:///app.db"
    secret_key: str = "change-me-in-production"
    api_key: str = "dev-key"
    cors_origins: list[str] = ["http://localhost:3000"]
    
    model_config = {"env_file": ".env"}

@lru_cache
def get_settings() -> Settings:
    return Settings()

# main.py
from config import get_settings, Settings
from fastapi import Depends

app = FastAPI()

@app.get("/info")
async def info(settings: Settings = Depends(get_settings)):
    return {
        "app_name": settings.app_name,
        "debug": settings.debug,
    }
```

### Lifespan events (startup/shutdown)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: runs before accepting requests
    print("Starting up...")
    # Initialize DB pool, load ML model, etc.
    
    yield  # app runs here
    
    # Shutdown: runs when app is stopping
    print("Shutting down...")
    # Close DB connections, cleanup resources

app = FastAPI(lifespan=lifespan)
```

---

## 12. Complete CRUD Example

```python
"""
Complete Users CRUD API — everything together.
"""
from fastapi import FastAPI, APIRouter, Depends, HTTPException, Query, status
from pydantic import BaseModel, Field
from datetime import datetime

app = FastAPI(title="Users API", version="1.0.0")

# ── Schemas ──

class UserCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: str = Field(pattern=r"^[\w.-]+@[\w.-]+\.\w+$")
    age: int = Field(ge=0, le=150)
    city: str = "Unknown"

class UserUpdate(BaseModel):
    name: str | None = Field(default=None, min_length=1, max_length=100)
    email: str | None = Field(default=None, pattern=r"^[\w.-]+@[\w.-]+\.\w+$")
    age: int | None = Field(default=None, ge=0, le=150)
    city: str | None = None

class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    age: int
    city: str
    is_active: bool
    created_at: str

class UserListResponse(BaseModel):
    items: list[UserResponse]
    total: int
    page: int
    size: int

# ── In-memory "database" ──

users_db: dict[int, dict] = {}
next_id: int = 1

# ── Router ──

router = APIRouter(prefix="/users", tags=["Users"])

@router.get("/", response_model=UserListResponse)
async def list_users(
    page: int = Query(default=1, ge=1),
    size: int = Query(default=20, ge=1, le=100),
    city: str | None = None,
):
    """List all users with optional filtering and pagination."""
    filtered = list(users_db.values())
    if city:
        filtered = [u for u in filtered if u["city"] == city]
    
    total = len(filtered)
    start = (page - 1) * size
    items = filtered[start : start + size]
    
    return UserListResponse(items=items, total=total, page=page, size=size)

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):
    """Get a single user by ID."""
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail=f"User {user_id} not found")
    return users_db[user_id]

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate):
    """Create a new user."""
    global next_id
    
    # Check duplicate email
    for existing in users_db.values():
        if existing["email"] == user.email:
            raise HTTPException(status_code=409, detail="Email already registered")
    
    new_user = {
        "id": next_id,
        **user.model_dump(),
        "is_active": True,
        "created_at": datetime.now().isoformat(),
    }
    users_db[next_id] = new_user
    next_id += 1
    
    return new_user

@router.patch("/{user_id}", response_model=UserResponse)
async def update_user(user_id: int, user: UserUpdate):
    """Update a user (partial update)."""
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail=f"User {user_id} not found")
    
    update_data = user.model_dump(exclude_unset=True)
    if not update_data:
        raise HTTPException(status_code=400, detail="No fields to update")
    
    users_db[user_id].update(update_data)
    return users_db[user_id]

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int):
    """Delete a user."""
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail=f"User {user_id} not found")
    del users_db[user_id]

app.include_router(router)
```

---

## 📝 Practice Tasks

### Task 1: Books API
Build a CRUD API for a bookstore: `GET /books`, `GET /books/{id}`, `POST /books`, `PATCH /books/{id}`, `DELETE /books/{id}`. Include query params for filtering by author, genre, min/max price. Use Pydantic models with proper validation.

### Task 2: Task Manager API
Build a task management API with: projects and tasks. A project has many tasks. Include: create/list projects, add tasks to project, update task status (todo → in_progress → done), filter tasks by status, assign tasks to users.

### Task 3: Pagination & Filtering
Build a generic `/products` endpoint with: multi-field filtering (category, price range, in_stock, search), sorting (by name, price, rating — ascending/descending), cursor-based and offset-based pagination. Compare the two pagination approaches.

### Task 4: Dependency Injection
Create dependencies for: database session, current user (from API key), pagination params, rate limiter (max 100 requests per minute per API key). Chain them together. Write tests that override dependencies.

### Task 5: File Upload API
Build an API that: accepts CSV file upload, validates the CSV structure (required columns), processes rows (validate, transform), returns a summary (total rows, valid, invalid, errors). Store results and provide a `GET /jobs/{id}` endpoint to check status.

### Task 6: Error Handling
Build a custom exception handling system: custom exception classes (NotFoundError, ConflictError, ValidationError), exception handlers that return consistent error format, logging of all errors, different error detail levels for debug vs production.

---

## 📚 Resources

- [FastAPI Official Documentation](https://fastapi.tiangolo.com/) — excellent, read the tutorial
- [FastAPI Tutorial](https://fastapi.tiangolo.com/tutorial/)
- [Pydantic V2 Documentation](https://docs.pydantic.dev/)
- [Starlette Documentation](https://www.starlette.io/)
- [uvicorn Documentation](https://www.uvicorn.org/)
- [TestClient (Starlette)](https://www.starlette.io/testclient/)
- [Real Python — FastAPI](https://realpython.com/fastapi-python-web-apis/)

---

## 🔑 Key Takeaways

1. **FastAPI = type hints + Pydantic + async.** Everything you've learned combines here.
2. **Pydantic models ARE your validation layer.** Define a schema → validation is automatic.
3. **Swagger docs for free** — just visit `/docs`. Your API is self-documenting.
4. **Dependency injection with `Depends()`** — clean, testable, composable. Use for DB sessions, auth, pagination.
5. **Separate schemas: Create / Update / Response.** Never expose passwords, internal fields.
6. **`HTTPException` for errors** — raise it anywhere, FastAPI handles the response.
7. **Routers for organization** — one router per resource, include in main app.
8. **`TestClient` for testing** — override dependencies for isolated tests.
9. **`response_model` controls output** — even if your function returns extra fields, only schema fields are sent.
10. **`exclude_unset=True`** — the key to partial updates (PATCH). Only update fields the client actually sent.

---

> **Tomorrow (Day 20):** FastAPI Advanced — middleware, background tasks, WebSockets, JWT authentication, CORS, error handling patterns.

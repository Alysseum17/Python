# Day 20: FastAPI Advanced

## 🎯 Goal

Level up your FastAPI skills with production-grade patterns: middleware, background tasks, WebSockets, JWT authentication, CORS, custom exception handling, and rate limiting. After today you can build real, deployable APIs.

---

## 1. Middleware

Middleware runs on EVERY request/response — before the route handler and after. Like Express middleware or Spring filters/interceptors.

### Custom middleware

```python
import time
import uuid
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI()

# ── Timing middleware ──
@app.middleware("http")
async def add_timing(request: Request, call_next):
    """Measure and log request duration."""
    start = time.perf_counter()
    request_id = str(uuid.uuid4())[:8]
    
    # Add request ID to state (accessible in route handlers)
    request.state.request_id = request_id
    
    response = await call_next(request)
    
    duration = time.perf_counter() - start
    response.headers["X-Request-ID"] = request_id
    response.headers["X-Process-Time"] = f"{duration:.4f}"
    
    print(f"[{request_id}] {request.method} {request.url.path} → {response.status_code} ({duration:.3f}s)")
    
    return response
```

### Logging middleware (class-based)

```python
import logging
from starlette.middleware.base import BaseHTTPMiddleware
from fastapi import Request

logger = logging.getLogger("api")

class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Log request
        logger.info(f"→ {request.method} {request.url.path}")
        
        try:
            response = await call_next(request)
            logger.info(f"← {response.status_code} {request.url.path}")
            return response
        except Exception as e:
            logger.error(f"✗ {request.url.path}: {e}")
            raise

app.add_middleware(LoggingMiddleware)
```

### Built-in middleware

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI()

# CORS — allow frontend to call your API
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "https://myapp.com"],
    allow_credentials=True,
    allow_methods=["*"],     # GET, POST, PUT, DELETE, etc.
    allow_headers=["*"],     # Authorization, Content-Type, etc.
)

# GZip — compress responses > 500 bytes
app.add_middleware(GZipMiddleware, minimum_size=500)

# Trusted Host — reject requests with wrong Host header
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["myapp.com", "*.myapp.com", "localhost"],
)
```

### CORS explained

```
CORS = Cross-Origin Resource Sharing

Problem:
  Frontend at https://myapp.com wants to call API at https://api.myapp.com
  Browser blocks this by default (security: Same-Origin Policy)

Solution:
  API sends headers saying "I allow requests from https://myapp.com"
  Browser sees these headers and allows the request

Without CORS middleware:
  → Browser sends preflight OPTIONS request
  → API returns no CORS headers
  → Browser blocks the actual request
  → Frontend gets "CORS error"

With CORS middleware:
  → Browser sends preflight OPTIONS request
  → API returns: Access-Control-Allow-Origin: https://myapp.com
  → Browser allows the actual request
  → Everything works

In development: allow_origins=["*"] (allow everything)
In production: allow_origins=["https://myapp.com"] (specific domains only)
```

---

## 2. JWT Authentication

### How JWT works

```
JWT = JSON Web Token

Flow:
  1. Client sends credentials (email + password) to POST /auth/login
  2. Server verifies credentials
  3. Server creates a JWT token (signed with a secret key)
  4. Client stores the token
  5. Client sends token in every request: Authorization: Bearer <token>
  6. Server verifies the token signature on each request

Token structure (3 parts, base64-encoded, separated by dots):
  header.payload.signature
  
  Header:  {"alg": "HS256", "typ": "JWT"}
  Payload: {"sub": "user@test.com", "exp": 1707955200, "user_id": 1}
  Signature: HMAC-SHA256(header + "." + payload, SECRET_KEY)

The server NEVER stores the token — it just verifies the signature.
This is why JWT is "stateless" — no session storage needed.
```

### Implementation

```python
# auth.py
from datetime import datetime, timedelta, timezone
from typing import Any
import jwt  # pip install PyJWT
from passlib.context import CryptContext  # pip install passlib[bcrypt]
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel

# ── Config ──
SECRET_KEY = "your-super-secret-key-change-in-production"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# ── Password hashing ──
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

# ── JWT tokens ──
def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def decode_token(token: str) -> dict:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token has expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

# ── Schemas ──
class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"

class TokenData(BaseModel):
    email: str
    user_id: int

# ── OAuth2 scheme (tells Swagger where the token goes) ──
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

# ── Dependency: get current user from token ──
async def get_current_user(token: str = Depends(oauth2_scheme)) -> TokenData:
    """Extract and validate user from JWT token."""
    payload = decode_token(token)
    email = payload.get("sub")
    user_id = payload.get("user_id")
    if email is None or user_id is None:
        raise HTTPException(status_code=401, detail="Invalid token payload")
    return TokenData(email=email, user_id=user_id)
```

### Auth routes

```python
# routers/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm

router = APIRouter(prefix="/auth", tags=["Authentication"])

# Fake user DB for example
fake_users = {
    "alice@test.com": {
        "id": 1,
        "email": "alice@test.com",
        "name": "Alice",
        "hashed_password": hash_password("secret123"),
    }
}

@router.post("/login", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """Authenticate and return JWT token."""
    # OAuth2PasswordRequestForm has .username and .password fields
    user = fake_users.get(form_data.username)
    
    if not user or not verify_password(form_data.password, user["hashed_password"]):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect email or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    access_token = create_access_token(
        data={"sub": user["email"], "user_id": user["id"]},
        expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
    )
    
    return Token(access_token=access_token)

@router.get("/me")
async def get_me(current_user: TokenData = Depends(get_current_user)):
    """Get current authenticated user."""
    return {"email": current_user.email, "user_id": current_user.user_id}
```

### Protecting routes

```python
# Any route that needs authentication
@router.get("/users/me/orders")
async def get_my_orders(
    current_user: TokenData = Depends(get_current_user),
    db: Session = Depends(get_db),
):
    orders = db.query(Order).filter(Order.user_id == current_user.user_id).all()
    return orders

# Protect entire router
protected_router = APIRouter(
    prefix="/admin",
    tags=["Admin"],
    dependencies=[Depends(get_current_user)],
)

# Role-based access
async def require_admin(current_user: TokenData = Depends(get_current_user)):
    # In real app, check user's role in DB
    if current_user.email != "admin@test.com":
        raise HTTPException(status_code=403, detail="Admin access required")
    return current_user

@router.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    admin: TokenData = Depends(require_admin),
):
    return {"message": f"User {user_id} deleted by admin {admin.email}"}
```

### How it looks in Swagger

```
After adding OAuth2PasswordBearer, Swagger UI shows:
  🔒 Authorize button (top right)
  Click it → enter username/password
  It calls POST /auth/login → gets token
  All subsequent requests include: Authorization: Bearer <token>
  
This is automatic — zero extra code!
```

---

## 3. Background Tasks

For work that doesn't need to finish before the response (sending emails, processing files, logging analytics).

### Built-in BackgroundTasks

```python
from fastapi import BackgroundTasks
import time

def send_email(to: str, subject: str, body: str):
    """Simulate sending an email (slow operation)."""
    time.sleep(3)  # pretend this takes 3 seconds
    print(f"Email sent to {to}: {subject}")

def log_analytics(event: str, user_id: int):
    """Log analytics event to external service."""
    time.sleep(1)
    print(f"Analytics: {event} by user {user_id}")

@app.post("/users", status_code=201)
async def create_user(
    user: UserCreate,
    background_tasks: BackgroundTasks,
):
    # Create user immediately
    new_user = {"id": 1, **user.model_dump()}
    
    # Queue background work (runs AFTER response is sent)
    background_tasks.add_task(send_email, user.email, "Welcome!", "Thanks for signing up")
    background_tasks.add_task(log_analytics, "user_created", 1)
    
    # Response is sent immediately — client doesn't wait for email/analytics
    return new_user

# BackgroundTasks runs in the same process.
# For heavy work or reliability, use Celery or arq (Day 24 topics).
```

### Background tasks in dependencies

```python
def log_request(background_tasks: BackgroundTasks, request: Request):
    """Dependency that logs every request in background."""
    background_tasks.add_task(
        save_request_log,
        method=request.method,
        path=request.url.path,
        timestamp=datetime.now().isoformat(),
    )

@app.get("/data", dependencies=[Depends(log_request)])
async def get_data():
    return {"data": "hello"}
```

---

## 4. WebSockets

Real-time bidirectional communication. Used for: live dashboards, chat, notifications, streaming data.

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from datetime import datetime

app = FastAPI()

# ── Simple echo WebSocket ──
@app.websocket("/ws/echo")
async def echo(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
    except WebSocketDisconnect:
        print("Client disconnected")

# ── Connection manager for broadcast ──
class ConnectionManager:
    """Manage multiple WebSocket connections."""
    
    def __init__(self):
        self.active: list[WebSocket] = []
    
    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active.append(websocket)
    
    def disconnect(self, websocket: WebSocket):
        self.active.remove(websocket)
    
    async def broadcast(self, message: str):
        """Send message to all connected clients."""
        for ws in self.active:
            await ws.send_text(message)
    
    async def send_personal(self, websocket: WebSocket, message: str):
        await websocket.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws/chat")
async def chat(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            timestamp = datetime.now().strftime("%H:%M:%S")
            await manager.broadcast(f"[{timestamp}] {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast("A user left the chat")

# ── Live data stream (Data Engineering use case) ──
@app.websocket("/ws/metrics")
async def live_metrics(websocket: WebSocket):
    """Stream live metrics to dashboard."""
    await websocket.accept()
    import asyncio
    import random
    
    try:
        while True:
            metrics = {
                "timestamp": datetime.now().isoformat(),
                "cpu_usage": round(random.uniform(10, 90), 1),
                "memory_mb": random.randint(500, 4000),
                "requests_per_sec": random.randint(50, 500),
            }
            await websocket.send_json(metrics)
            await asyncio.sleep(1)  # send every second
    except WebSocketDisconnect:
        print("Dashboard disconnected")
```

### Testing WebSockets

```python
def test_websocket_echo(client):
    with client.websocket_connect("/ws/echo") as ws:
        ws.send_text("hello")
        data = ws.receive_text()
        assert data == "Echo: hello"
```

---

## 5. Custom Exception Handling

### Global exception handlers

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from datetime import datetime
import traceback
import logging

logger = logging.getLogger("api")

app = FastAPI()

# ── Custom exception classes ──

class AppException(Exception):
    """Base exception for our application."""
    def __init__(self, message: str, status_code: int = 500, error_code: str = "INTERNAL_ERROR"):
        self.message = message
        self.status_code = status_code
        self.error_code = error_code

class NotFoundError(AppException):
    def __init__(self, resource: str, resource_id: int | str):
        super().__init__(
            message=f"{resource} with id {resource_id} not found",
            status_code=404,
            error_code="NOT_FOUND",
        )

class ConflictError(AppException):
    def __init__(self, message: str):
        super().__init__(message=message, status_code=409, error_code="CONFLICT")

class ForbiddenError(AppException):
    def __init__(self, message: str = "Access denied"):
        super().__init__(message=message, status_code=403, error_code="FORBIDDEN")

# ── Error response schema ──

class ErrorResponse(BaseModel):
    error_code: str
    message: str
    timestamp: str
    path: str | None = None

# ── Exception handlers ──

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    """Handle all custom application exceptions."""
    logger.warning(f"[{exc.error_code}] {request.method} {request.url.path}: {exc.message}")
    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(
            error_code=exc.error_code,
            message=exc.message,
            timestamp=datetime.now().isoformat(),
            path=str(request.url.path),
        ).model_dump(),
    )

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    """Handle FastAPI HTTPException with consistent format."""
    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(
            error_code=f"HTTP_{exc.status_code}",
            message=str(exc.detail),
            timestamp=datetime.now().isoformat(),
            path=str(request.url.path),
        ).model_dump(),
    )

@app.exception_handler(Exception)
async def unhandled_exception_handler(request: Request, exc: Exception):
    """Catch-all for unexpected errors."""
    logger.error(f"Unhandled: {request.method} {request.url.path}\n{traceback.format_exc()}")
    return JSONResponse(
        status_code=500,
        content=ErrorResponse(
            error_code="INTERNAL_ERROR",
            message="An unexpected error occurred",  # don't expose details
            timestamp=datetime.now().isoformat(),
            path=str(request.url.path),
        ).model_dump(),
    )

# ── Usage in routes ──

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    user = users_db.get(user_id)
    if not user:
        raise NotFoundError("User", user_id)
    return user

@app.post("/users")
async def create_user(user: UserCreate):
    if email_exists(user.email):
        raise ConflictError(f"Email {user.email} is already registered")
    ...
```

---

## 6. Rate Limiting

```python
import time
from collections import defaultdict
from fastapi import Request, HTTPException, Depends

# ── Simple in-memory rate limiter ──

class RateLimiter:
    """Token bucket rate limiter."""
    
    def __init__(self, requests_per_minute: int = 60):
        self.rpm = requests_per_minute
        self.requests: dict[str, list[float]] = defaultdict(list)
    
    def _get_client_id(self, request: Request) -> str:
        """Identify client by IP or API key."""
        api_key = request.headers.get("X-API-Key")
        if api_key:
            return f"key:{api_key}"
        return f"ip:{request.client.host}"
    
    def _cleanup(self, client_id: str):
        """Remove requests older than 1 minute."""
        cutoff = time.time() - 60
        self.requests[client_id] = [
            t for t in self.requests[client_id] if t > cutoff
        ]
    
    def check(self, request: Request) -> None:
        """Check rate limit. Raises HTTPException if exceeded."""
        client_id = self._get_client_id(request)
        self._cleanup(client_id)
        
        if len(self.requests[client_id]) >= self.rpm:
            raise HTTPException(
                status_code=429,
                detail="Rate limit exceeded. Try again later.",
                headers={
                    "Retry-After": "60",
                    "X-RateLimit-Limit": str(self.rpm),
                    "X-RateLimit-Remaining": "0",
                },
            )
        
        self.requests[client_id].append(time.time())

# Global rate limiter instance
rate_limiter = RateLimiter(requests_per_minute=60)

# ── As a dependency ──

async def check_rate_limit(request: Request):
    rate_limiter.check(request)

# Apply to specific routes
@app.get("/api/data", dependencies=[Depends(check_rate_limit)])
async def get_data():
    return {"data": "here"}

# Apply to entire app
# app = FastAPI(dependencies=[Depends(check_rate_limit)])

# For production, use Redis-based rate limiting:
# pip install slowapi
# from slowapi import Limiter
# limiter = Limiter(key_func=get_remote_address)
```

---

## 7. Response Caching

```python
from functools import lru_cache
from fastapi import Request
from fastapi.responses import JSONResponse
import time

# ── Simple in-memory cache ──

class SimpleCache:
    def __init__(self, ttl_seconds: int = 60):
        self.cache: dict[str, tuple[float, any]] = {}
        self.ttl = ttl_seconds
    
    def get(self, key: str):
        if key in self.cache:
            timestamp, value = self.cache[key]
            if time.time() - timestamp < self.ttl:
                return value
            del self.cache[key]
        return None
    
    def set(self, key: str, value):
        self.cache[key] = (time.time(), value)

cache = SimpleCache(ttl_seconds=300)  # 5 min cache

@app.get("/products")
async def list_products(category: str | None = None):
    cache_key = f"products:{category}"
    
    cached = cache.get(cache_key)
    if cached:
        return cached
    
    # Expensive operation
    products = fetch_products_from_db(category)
    
    cache.set(cache_key, products)
    return products

# For production: use Redis
# pip install aioredis
```

### Cache-Control headers

```python
from fastapi.responses import JSONResponse

@app.get("/static-data")
async def get_static_data():
    data = {"version": "1.0", "features": ["a", "b", "c"]}
    response = JSONResponse(content=data)
    response.headers["Cache-Control"] = "public, max-age=3600"  # 1 hour
    return response

@app.get("/user-data")
async def get_user_data():
    data = {"name": "Alice"}
    response = JSONResponse(content=data)
    response.headers["Cache-Control"] = "private, no-cache"  # never cache
    return response
```

---

## 8. Structured Logging

```python
import logging
import json
from datetime import datetime
from fastapi import FastAPI, Request

# ── JSON formatter for structured logs ──

class JSONFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        log_data = {
            "timestamp": datetime.now().isoformat(),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
        }
        # Add extra fields if present
        if hasattr(record, "request_id"):
            log_data["request_id"] = record.request_id
        if hasattr(record, "user_id"):
            log_data["user_id"] = record.user_id
        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_data)

# Setup
def setup_logging():
    logger = logging.getLogger("api")
    logger.setLevel(logging.INFO)
    
    handler = logging.StreamHandler()
    handler.setFormatter(JSONFormatter())
    logger.addHandler(handler)
    
    return logger

logger = setup_logging()

# ── Log middleware ──

@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    import time
    start = time.perf_counter()
    
    response = await call_next(request)
    
    duration = time.perf_counter() - start
    logger.info(
        f"{request.method} {request.url.path} → {response.status_code}",
        extra={
            "request_id": getattr(request.state, "request_id", None),
            "duration": round(duration, 4),
        },
    )
    
    return response

# Output:
# {"timestamp": "2025-02-14T12:00:00", "level": "INFO",
#  "message": "GET /users → 200", "duration": 0.023}
```

---

## 9. API Versioning

```python
from fastapi import FastAPI, APIRouter

app = FastAPI()

# ── Version 1 ──
v1_router = APIRouter(prefix="/api/v1")

@v1_router.get("/users")
async def list_users_v1():
    return [{"id": 1, "name": "Alice"}]  # simple format

# ── Version 2 (new response format) ──
v2_router = APIRouter(prefix="/api/v2")

@v2_router.get("/users")
async def list_users_v2():
    return {
        "data": [{"id": 1, "name": "Alice", "email": "alice@test.com"}],
        "meta": {"total": 1, "page": 1},
    }  # richer format

app.include_router(v1_router, tags=["v1"])
app.include_router(v2_router, tags=["v2"])

# GET /api/v1/users → old format
# GET /api/v2/users → new format
# Both work simultaneously — clients migrate at their own pace
```

---

## 10. Complete Production App Structure

```python
"""
main.py — Production-ready FastAPI application.
"""
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from config import get_settings
from database import engine
from models import Base
from routers import auth, users, orders
from middleware import LoggingMiddleware, TimingMiddleware
from exceptions import setup_exception_handlers

settings = get_settings()

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup and shutdown events."""
    # Startup
    Base.metadata.create_all(bind=engine)
    print(f"🚀 {settings.app_name} starting...")
    yield
    # Shutdown
    engine.dispose()
    print(f"👋 {settings.app_name} shutting down...")

app = FastAPI(
    title=settings.app_name,
    version="1.0.0",
    description="Production-ready API",
    lifespan=lifespan,
    docs_url="/docs" if settings.debug else None,  # disable docs in prod
    redoc_url="/redoc" if settings.debug else None,
)

# Middleware (order matters — last added runs first)
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
app.add_middleware(LoggingMiddleware)
app.add_middleware(TimingMiddleware)

# Exception handlers
setup_exception_handlers(app)

# Routers
app.include_router(auth.router)
app.include_router(users.router)
app.include_router(orders.router)

# Health check
@app.get("/health", tags=["System"])
async def health():
    return {"status": "ok", "version": "1.0.0"}
```

### Running in production

```bash
# Development
uvicorn main:app --reload --port 8000

# Production (with multiple workers)
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4

# Production with Gunicorn (better process management)
gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000

# Options:
# --workers 4           → 4 worker processes (= CPU cores)
# --host 0.0.0.0        → listen on all interfaces (required in Docker)
# --port 8000           → port number
# --access-log -        → access logs to stdout
# --error-log -         → error logs to stdout
```

---

## 11. Async Database Integration Pattern

```python
"""
Full async FastAPI + SQLAlchemy integration.
"""
# database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

ASYNC_DATABASE_URL = "sqlite+aiosqlite:///app.db"

async_engine = create_async_engine(ASYNC_DATABASE_URL, echo=True)
AsyncSessionLocal = async_sessionmaker(async_engine, class_=AsyncSession)

async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# routers/users.py
from fastapi import APIRouter, Depends
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

router = APIRouter(prefix="/users", tags=["Users"])

@router.get("/")
async def list_users(
    db: AsyncSession = Depends(get_db),
    skip: int = 0,
    limit: int = 20,
):
    stmt = select(User).offset(skip).limit(limit)
    result = await db.scalars(stmt)
    return result.all()

@router.get("/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    stmt = (
        select(User)
        .options(selectinload(User.orders))
        .where(User.id == user_id)
    )
    user = await db.scalar(stmt)
    if not user:
        raise NotFoundError("User", user_id)
    return user

@router.post("/", status_code=201)
async def create_user(user: UserCreate, db: AsyncSession = Depends(get_db)):
    db_user = User(**user.model_dump(exclude={"password"}))
    db_user.hashed_password = hash_password(user.password)
    db.add(db_user)
    await db.flush()
    await db.refresh(db_user)
    return db_user
```

---

## 📝 Practice Tasks

### Task 1: Auth System
Build a complete authentication system:
- Registration (POST /auth/register)
- Login (POST /auth/login) → returns JWT
- Get current user (GET /auth/me)
- Password hashing with bcrypt
- Token expiration and refresh
- Protected routes that require valid token
- Role-based access (admin vs regular user)

### Task 2: Middleware Suite
Build a middleware stack with:
- Request ID generation (UUID in every response header)
- Request/response logging (method, path, status, duration)
- Rate limiting (100 req/min per IP)
- CORS configuration
Test that all middleware works together.

### Task 3: WebSocket Chat
Build a chat application with:
- WebSocket endpoint for real-time messages
- Connection manager that tracks rooms
- Join/leave room notifications
- Message history (last 50 messages per room)
- HTML page to test the chat

### Task 4: Error Handling System
Build a consistent error handling system:
- Custom exception hierarchy (AppError → NotFound, Conflict, Validation, Forbidden)
- Global exception handlers returning consistent JSON format
- Different detail levels for debug vs production
- Error logging with stack traces
- Test all error scenarios

### Task 5: Full CRUD with Auth + DB
Combine Days 17-20: Build a complete API with:
- SQLAlchemy async models (User, Post, Comment)
- Alembic migrations
- JWT authentication
- CRUD for all resources
- Only owners can edit/delete their content
- Pagination, filtering, sorting
- Full test suite

### Task 6: Data Ingestion API
Build a data pipeline API:
- POST /ingest — accepts JSON array or CSV file upload
- Validates data with Pydantic
- Processes in background (BackgroundTasks)
- GET /jobs/{id} — check processing status
- GET /jobs/{id}/results — get processed results
- WebSocket /ws/jobs/{id} — live progress updates

---

## 📚 Resources

- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [FastAPI Middleware](https://fastapi.tiangolo.com/tutorial/middleware/)
- [FastAPI WebSockets](https://fastapi.tiangolo.com/advanced/websockets/)
- [FastAPI Background Tasks](https://fastapi.tiangolo.com/tutorial/background-tasks/)
- [FastAPI Testing](https://fastapi.tiangolo.com/tutorial/testing/)
- [PyJWT Documentation](https://pyjwt.readthedocs.io/)
- [passlib Documentation](https://passlib.readthedocs.io/)
- [Real Python — FastAPI Auth](https://realpython.com/python-fastapi/)

---

## 🔑 Key Takeaways

1. **Middleware for cross-cutting concerns** — timing, logging, CORS, rate limiting. Runs on every request.
2. **JWT for stateless auth** — no sessions, no server-side storage. Token contains user info, signed with secret.
3. **`OAuth2PasswordBearer`** — tells Swagger about your auth. Automatic "Authorize" button in docs.
4. **`BackgroundTasks`** — fire-and-forget work after response. Emails, logging, processing. For heavy work use Celery.
5. **WebSockets for real-time** — live dashboards, notifications, chat. `ConnectionManager` pattern for multi-client.
6. **Custom exception hierarchy** — consistent error format across all endpoints. One handler catches all.
7. **Rate limiting protects your API** — in-memory for dev, Redis for production.
8. **API versioning with prefixes** — `/api/v1/`, `/api/v2/`. Both live simultaneously.
9. **Structured JSON logging** — essential for production. Parse logs with ELK, Datadog, etc.
10. **`gunicorn` + `uvicorn` workers** — production deployment. Multiple processes handle concurrent load.

---

> **Tomorrow (Day 21):** FastAPI + SQLAlchemy + Alembic Project — build a complete REST API with database, migrations, authentication, and tests. Putting everything together.

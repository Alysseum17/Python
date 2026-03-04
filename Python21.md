# Day 21: FastAPI + SQLAlchemy + Alembic — Complete Project

## 🎯 Goal

Build a complete, production-grade REST API from scratch. This is the capstone for Phase 5 — everything from Days 5 (Pydantic), 12 (async), 15-18 (SQL/SQLAlchemy/Alembic), and 19-20 (FastAPI) comes together.

We'll build a **Task Management API** (like a simplified Trello/Jira) with:
- User registration & JWT auth
- Projects with ownership
- Tasks with statuses, assignments, priorities
- Comments on tasks
- Full CRUD, pagination, filtering
- Database with migrations
- Tests

---

## 1. Project Architecture

```
task_manager/
├── alembic/                  ← Database migrations
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
├── alembic.ini
├── app/
│   ├── __init__.py
│   ├── main.py              ← FastAPI app entry point
│   ├── config.py             ← Pydantic Settings
│   ├── database.py           ← Engine, session, dependency
│   ├── models/               ← SQLAlchemy ORM models
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── user.py
│   │   ├── project.py
│   │   └── task.py
│   ├── schemas/              ← Pydantic request/response schemas
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── user.py
│   │   ├── project.py
│   │   └── task.py
│   ├── routers/              ← API endpoints
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── projects.py
│   │   └── tasks.py
│   ├── services/             ← Business logic
│   │   └── auth_service.py
│   ├── dependencies.py       ← Shared DI dependencies
│   └── exceptions.py         ← Custom exceptions + handlers
├── tests/
│   ├── conftest.py
│   ├── test_auth.py
│   ├── test_projects.py
│   └── test_tasks.py
└── pyproject.toml
```

### Layer responsibilities

```
Router (routers/)
  → Handles HTTP: parse request, call service, return response
  → Thin layer — no business logic here

Service (services/)
  → Business logic: validation rules, permissions, orchestration
  → Calls DB through session
  → Raises domain exceptions

Model (models/)
  → Database structure: tables, columns, relationships
  → No business logic

Schema (schemas/)
  → API contracts: what comes in, what goes out
  → Validation rules (Pydantic)

Same layered architecture as Spring Boot:
  Controller → Service → Repository → Entity
  Router     → Service → Session     → Model
```

---

## 2. Configuration & Database

### config.py

```python
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    app_name: str = "Task Manager API"
    debug: bool = True
    
    # Database
    database_url: str = "sqlite:///./task_manager.db"
    
    # JWT
    secret_key: str = "dev-secret-key-change-in-production"
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    
    # CORS
    cors_origins: list[str] = ["http://localhost:3000"]
    
    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

### database.py

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from app.config import get_settings

settings = get_settings()

engine = create_engine(
    settings.database_url,
    echo=settings.debug,
    connect_args={"check_same_thread": False},  # SQLite only
)

SessionLocal = sessionmaker(bind=engine, autoflush=False)

def get_db():
    """Database session dependency — yields session, auto-closes."""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

---

## 3. Models

### models/base.py

```python
from datetime import datetime
from sqlalchemy import DateTime
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class TimestampMixin:
    """Adds created_at and updated_at to any model."""
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.now)
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.now, onupdate=datetime.now
    )
```

### models/user.py

```python
from sqlalchemy import String, Boolean
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.models.base import Base, TimestampMixin

class User(TimestampMixin, Base):
    __tablename__ = "users"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(100))
    hashed_password: Mapped[str] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    
    # Relationships
    owned_projects: Mapped[list["Project"]] = relationship(
        back_populates="owner", cascade="all, delete-orphan"
    )
    assigned_tasks: Mapped[list["Task"]] = relationship(
        back_populates="assignee", foreign_keys="Task.assignee_id"
    )
    comments: Mapped[list["Comment"]] = relationship(
        back_populates="author", cascade="all, delete-orphan"
    )
    
    def __repr__(self) -> str:
        return f"User(id={self.id}, email='{self.email}')"
```

### models/project.py

```python
from sqlalchemy import String, Text, ForeignKey, Integer
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.models.base import Base, TimestampMixin

class Project(TimestampMixin, Base):
    __tablename__ = "projects"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text, default=None)
    owner_id: Mapped[int] = mapped_column(Integer, ForeignKey("users.id"), index=True)
    
    # Relationships
    owner: Mapped["User"] = relationship(back_populates="owned_projects")
    tasks: Mapped[list["Task"]] = relationship(
        back_populates="project", cascade="all, delete-orphan"
    )
    
    def __repr__(self) -> str:
        return f"Project(id={self.id}, name='{self.name}')"
```

### models/task.py

```python
import enum
from sqlalchemy import String, Text, ForeignKey, Integer, Enum as SAEnum
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.models.base import Base, TimestampMixin

class TaskStatus(str, enum.Enum):
    TODO = "todo"
    IN_PROGRESS = "in_progress"
    REVIEW = "review"
    DONE = "done"

class TaskPriority(str, enum.Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    URGENT = "urgent"

class Task(TimestampMixin, Base):
    __tablename__ = "tasks"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    description: Mapped[str | None] = mapped_column(Text, default=None)
    status: Mapped[TaskStatus] = mapped_column(SAEnum(TaskStatus), default=TaskStatus.TODO)
    priority: Mapped[TaskPriority] = mapped_column(SAEnum(TaskPriority), default=TaskPriority.MEDIUM)
    project_id: Mapped[int] = mapped_column(Integer, ForeignKey("projects.id"), index=True)
    assignee_id: Mapped[int | None] = mapped_column(
        Integer, ForeignKey("users.id"), default=None, index=True
    )
    
    # Relationships
    project: Mapped["Project"] = relationship(back_populates="tasks")
    assignee: Mapped["User | None"] = relationship(
        back_populates="assigned_tasks", foreign_keys=[assignee_id]
    )
    comments: Mapped[list["Comment"]] = relationship(
        back_populates="task", cascade="all, delete-orphan"
    )
    
    def __repr__(self) -> str:
        return f"Task(id={self.id}, title='{self.title}', status={self.status})"

class Comment(TimestampMixin, Base):
    __tablename__ = "comments"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    content: Mapped[str] = mapped_column(Text)
    task_id: Mapped[int] = mapped_column(Integer, ForeignKey("tasks.id"), index=True)
    author_id: Mapped[int] = mapped_column(Integer, ForeignKey("users.id"), index=True)
    
    # Relationships
    task: Mapped["Task"] = relationship(back_populates="comments")
    author: Mapped["User"] = relationship(back_populates="comments")
```

### models/__init__.py

```python
from app.models.base import Base
from app.models.user import User
from app.models.project import Project
from app.models.task import Task, TaskStatus, TaskPriority, Comment

__all__ = ["Base", "User", "Project", "Task", "TaskStatus", "TaskPriority", "Comment"]
```

---

## 4. Schemas (Pydantic)

### schemas/auth.py

```python
from pydantic import BaseModel, Field

class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"

class TokenData(BaseModel):
    user_id: int
    email: str

class LoginRequest(BaseModel):
    email: str
    password: str
```

### schemas/user.py

```python
from pydantic import BaseModel, Field
from datetime import datetime

class UserCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: str = Field(pattern=r"^[\w.-]+@[\w.-]+\.\w+$")
    password: str = Field(min_length=8, max_length=100)

class UserUpdate(BaseModel):
    name: str | None = Field(default=None, min_length=1, max_length=100)
    email: str | None = Field(default=None, pattern=r"^[\w.-]+@[\w.-]+\.\w+$")

class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    is_active: bool
    created_at: datetime
    
    model_config = {"from_attributes": True}
```

### schemas/project.py

```python
from pydantic import BaseModel, Field
from datetime import datetime
from app.schemas.user import UserResponse

class ProjectCreate(BaseModel):
    name: str = Field(min_length=1, max_length=200)
    description: str | None = None

class ProjectUpdate(BaseModel):
    name: str | None = Field(default=None, min_length=1, max_length=200)
    description: str | None = None

class ProjectResponse(BaseModel):
    id: int
    name: str
    description: str | None
    owner_id: int
    created_at: datetime
    task_count: int = 0
    
    model_config = {"from_attributes": True}

class ProjectDetailResponse(ProjectResponse):
    owner: UserResponse
```

### schemas/task.py

```python
from pydantic import BaseModel, Field
from datetime import datetime
from app.models.task import TaskStatus, TaskPriority
from app.schemas.user import UserResponse

class TaskCreate(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    description: str | None = None
    priority: TaskPriority = TaskPriority.MEDIUM
    assignee_id: int | None = None

class TaskUpdate(BaseModel):
    title: str | None = Field(default=None, min_length=1, max_length=200)
    description: str | None = None
    status: TaskStatus | None = None
    priority: TaskPriority | None = None
    assignee_id: int | None = None

class TaskResponse(BaseModel):
    id: int
    title: str
    description: str | None
    status: TaskStatus
    priority: TaskPriority
    project_id: int
    assignee_id: int | None
    created_at: datetime
    updated_at: datetime
    comment_count: int = 0
    
    model_config = {"from_attributes": True}

class TaskDetailResponse(TaskResponse):
    assignee: UserResponse | None = None
    comments: list["CommentResponse"] = []

class CommentCreate(BaseModel):
    content: str = Field(min_length=1, max_length=5000)

class CommentResponse(BaseModel):
    id: int
    content: str
    author: UserResponse
    created_at: datetime
    
    model_config = {"from_attributes": True}
```

---

## 5. Exceptions

### exceptions.py

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from datetime import datetime
import logging

logger = logging.getLogger("api")

# ── Custom exceptions ──

class AppError(Exception):
    def __init__(self, message: str, status_code: int = 500, error_code: str = "INTERNAL"):
        self.message = message
        self.status_code = status_code
        self.error_code = error_code

class NotFoundError(AppError):
    def __init__(self, resource: str, resource_id):
        super().__init__(f"{resource} {resource_id} not found", 404, "NOT_FOUND")

class ConflictError(AppError):
    def __init__(self, message: str):
        super().__init__(message, 409, "CONFLICT")

class ForbiddenError(AppError):
    def __init__(self, message: str = "Access denied"):
        super().__init__(message, 403, "FORBIDDEN")

class UnauthorizedError(AppError):
    def __init__(self, message: str = "Not authenticated"):
        super().__init__(message, 401, "UNAUTHORIZED")

# ── Error response schema ──

class ErrorResponse(BaseModel):
    error_code: str
    message: str
    timestamp: str

# ── Register handlers ──

def setup_exception_handlers(app: FastAPI):
    @app.exception_handler(AppError)
    async def app_error_handler(request: Request, exc: AppError):
        logger.warning(f"[{exc.error_code}] {request.url.path}: {exc.message}")
        return JSONResponse(
            status_code=exc.status_code,
            content=ErrorResponse(
                error_code=exc.error_code,
                message=exc.message,
                timestamp=datetime.now().isoformat(),
            ).model_dump(),
        )
    
    @app.exception_handler(Exception)
    async def unhandled_handler(request: Request, exc: Exception):
        logger.error(f"Unhandled: {request.url.path}: {exc}", exc_info=True)
        return JSONResponse(
            status_code=500,
            content=ErrorResponse(
                error_code="INTERNAL",
                message="An unexpected error occurred",
                timestamp=datetime.now().isoformat(),
            ).model_dump(),
        )
```

---

## 6. Auth Service & Dependencies

### services/auth_service.py

```python
from datetime import datetime, timedelta, timezone
import jwt
from passlib.context import CryptContext
from sqlalchemy.orm import Session
from sqlalchemy import select

from app.config import get_settings
from app.models.user import User
from app.schemas.auth import TokenData
from app.exceptions import UnauthorizedError, ConflictError

settings = get_settings()
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(user: User) -> str:
    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.access_token_expire_minutes)
    payload = {"sub": user.email, "user_id": user.id, "exp": expire}
    return jwt.encode(payload, settings.secret_key, algorithm=settings.algorithm)

def decode_token(token: str) -> TokenData:
    try:
        payload = jwt.decode(token, settings.secret_key, algorithms=[settings.algorithm])
        return TokenData(user_id=payload["user_id"], email=payload["sub"])
    except jwt.ExpiredSignatureError:
        raise UnauthorizedError("Token has expired")
    except jwt.InvalidTokenError:
        raise UnauthorizedError("Invalid token")

def register_user(db: Session, name: str, email: str, password: str) -> User:
    existing = db.scalars(select(User).where(User.email == email)).first()
    if existing:
        raise ConflictError(f"Email {email} is already registered")
    
    user = User(name=name, email=email.lower().strip(), hashed_password=hash_password(password))
    db.add(user)
    db.commit()
    db.refresh(user)
    return user

def authenticate_user(db: Session, email: str, password: str) -> User:
    user = db.scalars(select(User).where(User.email == email)).first()
    if not user or not verify_password(password, user.hashed_password):
        raise UnauthorizedError("Incorrect email or password")
    if not user.is_active:
        raise UnauthorizedError("Account is deactivated")
    return user
```

### dependencies.py

```python
from fastapi import Depends
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session

from app.database import get_db
from app.models.user import User
from app.services.auth_service import decode_token
from app.exceptions import UnauthorizedError

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db),
) -> User:
    """Extract current user from JWT token."""
    token_data = decode_token(token)
    user = db.get(User, token_data.user_id)
    if not user or not user.is_active:
        raise UnauthorizedError("User not found or deactivated")
    return user
```

---

## 7. Routers

### routers/auth.py

```python
from fastapi import APIRouter, Depends
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session

from app.database import get_db
from app.schemas.auth import Token
from app.schemas.user import UserCreate, UserResponse
from app.services.auth_service import register_user, authenticate_user, create_access_token

router = APIRouter(prefix="/auth", tags=["Authentication"])

@router.post("/register", response_model=UserResponse, status_code=201)
def register(user_data: UserCreate, db: Session = Depends(get_db)):
    """Register a new user."""
    user = register_user(db, user_data.name, user_data.email, user_data.password)
    return user

@router.post("/login", response_model=Token)
def login(form_data: OAuth2PasswordRequestForm = Depends(), db: Session = Depends(get_db)):
    """Login and receive JWT token."""
    user = authenticate_user(db, form_data.username, form_data.password)
    token = create_access_token(user)
    return Token(access_token=token)
```

### routers/projects.py

```python
from fastapi import APIRouter, Depends, Query
from sqlalchemy import select
from sqlalchemy.orm import Session

from app.database import get_db
from app.dependencies import get_current_user
from app.models import User, Project
from app.schemas.project import ProjectCreate, ProjectUpdate, ProjectResponse, ProjectDetailResponse
from app.exceptions import NotFoundError, ForbiddenError

router = APIRouter(prefix="/projects", tags=["Projects"], dependencies=[Depends(get_current_user)])

def _get_own_project(project_id: int, user: User, db: Session) -> Project:
    """Helper: get project, verify ownership."""
    project = db.get(Project, project_id)
    if not project:
        raise NotFoundError("Project", project_id)
    if project.owner_id != user.id:
        raise ForbiddenError("No access to this project")
    return project

@router.get("/", response_model=list[ProjectResponse])
def list_projects(
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=20, ge=1, le=100),
):
    """List current user's projects."""
    stmt = (
        select(Project)
        .where(Project.owner_id == current_user.id)
        .order_by(Project.created_at.desc())
        .offset(skip).limit(limit)
    )
    projects = db.scalars(stmt).all()
    result = []
    for p in projects:
        resp = ProjectResponse.model_validate(p)
        resp.task_count = len(p.tasks)
        result.append(resp)
    return result

@router.post("/", response_model=ProjectResponse, status_code=201)
def create_project(
    data: ProjectCreate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """Create a new project."""
    project = Project(name=data.name, description=data.description, owner_id=current_user.id)
    db.add(project)
    db.commit()
    db.refresh(project)
    return project

@router.get("/{project_id}", response_model=ProjectDetailResponse)
def get_project(
    project_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """Get project details with owner info."""
    project = _get_own_project(project_id, current_user, db)
    resp = ProjectDetailResponse.model_validate(project)
    resp.task_count = len(project.tasks)
    return resp

@router.patch("/{project_id}", response_model=ProjectResponse)
def update_project(
    project_id: int,
    data: ProjectUpdate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """Update project name or description."""
    project = _get_own_project(project_id, current_user, db)
    for key, value in data.model_dump(exclude_unset=True).items():
        setattr(project, key, value)
    db.commit()
    db.refresh(project)
    return project

@router.delete("/{project_id}", status_code=204)
def delete_project(
    project_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """Delete a project and all its tasks."""
    project = _get_own_project(project_id, current_user, db)
    db.delete(project)
    db.commit()
```

### routers/tasks.py

```python
from fastapi import APIRouter, Depends, Query
from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

from app.database import get_db
from app.dependencies import get_current_user
from app.models import User, Project, Task, TaskStatus, TaskPriority, Comment
from app.schemas.task import (
    TaskCreate, TaskUpdate, TaskResponse, TaskDetailResponse,
    CommentCreate, CommentResponse,
)
from app.exceptions import NotFoundError, ForbiddenError

router = APIRouter(
    prefix="/projects/{project_id}/tasks",
    tags=["Tasks"],
    dependencies=[Depends(get_current_user)],
)

def _verify_project_access(project_id: int, user: User, db: Session) -> Project:
    project = db.get(Project, project_id)
    if not project:
        raise NotFoundError("Project", project_id)
    if project.owner_id != user.id:
        raise ForbiddenError("No access to this project")
    return project

def _get_task(task_id: int, project_id: int, db: Session) -> Task:
    task = db.scalars(
        select(Task).where(Task.id == task_id, Task.project_id == project_id)
    ).first()
    if not task:
        raise NotFoundError("Task", task_id)
    return task

@router.get("/", response_model=list[TaskResponse])
def list_tasks(
    project_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
    status: TaskStatus | None = None,
    priority: TaskPriority | None = None,
    assignee_id: int | None = None,
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=50, ge=1, le=100),
):
    """List tasks with optional filters."""
    _verify_project_access(project_id, current_user, db)
    
    stmt = select(Task).where(Task.project_id == project_id)
    if status:
        stmt = stmt.where(Task.status == status)
    if priority:
        stmt = stmt.where(Task.priority == priority)
    if assignee_id is not None:
        stmt = stmt.where(Task.assignee_id == assignee_id)
    
    stmt = stmt.order_by(Task.created_at.desc()).offset(skip).limit(limit)
    tasks = db.scalars(stmt).all()
    
    result = []
    for t in tasks:
        resp = TaskResponse.model_validate(t)
        resp.comment_count = len(t.comments)
        result.append(resp)
    return result

@router.post("/", response_model=TaskResponse, status_code=201)
def create_task(
    project_id: int,
    data: TaskCreate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """Create a task in a project."""
    _verify_project_access(project_id, current_user, db)
    task = Task(
        title=data.title,
        description=data.description,
        priority=data.priority,
        assignee_id=data.assignee_id,
        project_id=project_id,
    )
    db.add(task)
    db.commit()
    db.refresh(task)
    return task

@router.get("/{task_id}", response_model=TaskDetailResponse)
def get_task(
    project_id: int,
    task_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """Get task with comments and assignee."""
    _verify_project_access(project_id, current_user, db)
    stmt = (
        select(Task)
        .options(
            selectinload(Task.comments).selectinload(Comment.author),
            selectinload(Task.assignee),
        )
        .where(Task.id == task_id, Task.project_id == project_id)
    )
    task = db.scalars(stmt).first()
    if not task:
        raise NotFoundError("Task", task_id)
    resp = TaskDetailResponse.model_validate(task)
    resp.comment_count = len(task.comments)
    return resp

@router.patch("/{task_id}", response_model=TaskResponse)
def update_task(
    project_id: int,
    task_id: int,
    data: TaskUpdate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """Update task fields (status, priority, assignee, etc.)."""
    _verify_project_access(project_id, current_user, db)
    task = _get_task(task_id, project_id, db)
    for key, value in data.model_dump(exclude_unset=True).items():
        setattr(task, key, value)
    db.commit()
    db.refresh(task)
    return task

@router.delete("/{task_id}", status_code=204)
def delete_task(
    project_id: int,
    task_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """Delete a task."""
    _verify_project_access(project_id, current_user, db)
    task = _get_task(task_id, project_id, db)
    db.delete(task)
    db.commit()

# ── Comments ──

@router.post("/{task_id}/comments", response_model=CommentResponse, status_code=201)
def add_comment(
    project_id: int,
    task_id: int,
    data: CommentCreate,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """Add a comment to a task."""
    _verify_project_access(project_id, current_user, db)
    _get_task(task_id, project_id, db)
    
    comment = Comment(content=data.content, task_id=task_id, author_id=current_user.id)
    db.add(comment)
    db.commit()
    db.refresh(comment)
    # Trigger lazy load of author for response
    _ = comment.author
    return comment

@router.get("/{task_id}/comments", response_model=list[CommentResponse])
def list_comments(
    project_id: int,
    task_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    """List all comments for a task."""
    _verify_project_access(project_id, current_user, db)
    _get_task(task_id, project_id, db)
    
    stmt = (
        select(Comment)
        .options(selectinload(Comment.author))
        .where(Comment.task_id == task_id)
        .order_by(Comment.created_at.asc())
    )
    return db.scalars(stmt).all()
```

## 8. Main Application

### main.py

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import get_settings
from app.database import engine
from app.models import Base
from app.routers import auth, projects, tasks
from app.exceptions import setup_exception_handlers

settings = get_settings()

app = FastAPI(
    title=settings.app_name,
    version="1.0.0",
    description="Task Management API — JWT auth, projects, tasks, comments",
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Exception handlers
setup_exception_handlers(app)

# Create tables (for dev; use Alembic in production)
Base.metadata.create_all(bind=engine)

# Routers
app.include_router(auth.router)
app.include_router(projects.router)
app.include_router(tasks.router)

@app.get("/health", tags=["System"])
def health():
    return {"status": "ok", "version": "1.0.0"}
```

```bash
# Run the app
uvicorn app.main:app --reload

# Visit http://localhost:8000/docs for Swagger UI
# 1. Register via POST /auth/register
# 2. Login via POST /auth/login (use "Authorize" button with the token)
# 3. Create projects, tasks, comments — all authenticated
```

---

## 9. Alembic Setup

### Initialize Alembic

```bash
alembic init alembic
```

### alembic.ini — set connection

```ini
[alembic]
script_location = alembic
sqlalchemy.url = sqlite:///./task_manager.db
```

### alembic/env.py — connect to models

```python
# Near the top of env.py, add:
from app.models import Base

# Set target_metadata:
target_metadata = Base.metadata

# If using env variable for DB URL:
import os
from app.config import get_settings

def get_url():
    return get_settings().database_url

# In run_migrations_online(), override the URL:
# config.set_main_option("sqlalchemy.url", get_url())
```

### Generate and run initial migration

```bash
# Generate migration from models
alembic revision --autogenerate -m "initial schema: users, projects, tasks, comments"

# Review the generated file in alembic/versions/

# Apply migration
alembic upgrade head

# Verify
alembic current
```

### Future migration workflow

```bash
# 1. Change a model (e.g., add a column)
# 2. Generate migration:
alembic revision --autogenerate -m "add due_date to tasks"

# 3. Review generated file (always!)
# 4. Test both directions:
alembic upgrade head
alembic downgrade -1
alembic upgrade head

# 5. Commit migration file to Git
```

---

## 10. Tests

### tests/conftest.py

```python
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.main import app
from app.database import get_db
from app.models import Base

TEST_ENGINE = create_engine(
    "sqlite:///:memory:",
    connect_args={"check_same_thread": False},
)
TestSession = sessionmaker(bind=TEST_ENGINE)

@pytest.fixture(autouse=True)
def setup_db():
    """Create all tables before each test, drop after."""
    Base.metadata.create_all(TEST_ENGINE)
    yield
    Base.metadata.drop_all(TEST_ENGINE)

@pytest.fixture
def db():
    """Fresh database session."""
    session = TestSession()
    try:
        yield session
    finally:
        session.rollback()
        session.close()

@pytest.fixture
def client(db):
    """Test client with overridden DB dependency."""
    def override_db():
        try:
            yield db
        finally:
            pass
    
    app.dependency_overrides[get_db] = override_db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()

@pytest.fixture
def auth_headers(client) -> dict:
    """Register + login, return auth headers."""
    client.post("/auth/register", json={
        "name": "Test User",
        "email": "test@test.com",
        "password": "password123",
    })
    resp = client.post("/auth/login", data={
        "username": "test@test.com",
        "password": "password123",
    })
    token = resp.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}

@pytest.fixture
def project_id(client, auth_headers) -> int:
    """Create a test project and return its ID."""
    resp = client.post("/projects/", json={
        "name": "Test Project",
        "description": "For testing",
    }, headers=auth_headers)
    return resp.json()["id"]
```

### tests/test_auth.py

```python
def test_register_success(client):
    resp = client.post("/auth/register", json={
        "name": "Alice",
        "email": "alice@test.com",
        "password": "secret123",
    })
    assert resp.status_code == 201
    data = resp.json()
    assert data["name"] == "Alice"
    assert data["email"] == "alice@test.com"
    assert "password" not in data
    assert "hashed_password" not in data

def test_register_duplicate_email(client):
    user = {"name": "Alice", "email": "alice@test.com", "password": "secret123"}
    client.post("/auth/register", json=user)
    resp = client.post("/auth/register", json=user)
    assert resp.status_code == 409
    assert "already registered" in resp.json()["message"]

def test_register_invalid_email(client):
    resp = client.post("/auth/register", json={
        "name": "Alice", "email": "not-email", "password": "secret123",
    })
    assert resp.status_code == 422

def test_register_short_password(client):
    resp = client.post("/auth/register", json={
        "name": "Alice", "email": "alice@test.com", "password": "123",
    })
    assert resp.status_code == 422

def test_login_success(client):
    client.post("/auth/register", json={
        "name": "Alice", "email": "alice@test.com", "password": "secret123",
    })
    resp = client.post("/auth/login", data={
        "username": "alice@test.com", "password": "secret123",
    })
    assert resp.status_code == 200
    data = resp.json()
    assert "access_token" in data
    assert data["token_type"] == "bearer"

def test_login_wrong_password(client):
    client.post("/auth/register", json={
        "name": "Alice", "email": "alice@test.com", "password": "secret123",
    })
    resp = client.post("/auth/login", data={
        "username": "alice@test.com", "password": "wrong",
    })
    assert resp.status_code == 401

def test_login_nonexistent_user(client):
    resp = client.post("/auth/login", data={
        "username": "nobody@test.com", "password": "secret123",
    })
    assert resp.status_code == 401

def test_protected_route_no_token(client):
    resp = client.get("/projects/")
    assert resp.status_code == 401
```

### tests/test_projects.py

```python
def test_create_project(client, auth_headers):
    resp = client.post("/projects/", json={
        "name": "My Project",
        "description": "A cool project",
    }, headers=auth_headers)
    assert resp.status_code == 201
    data = resp.json()
    assert data["name"] == "My Project"
    assert data["description"] == "A cool project"
    assert data["id"] is not None

def test_list_projects(client, auth_headers):
    for i in range(3):
        client.post("/projects/", json={"name": f"Project {i}"}, headers=auth_headers)
    
    resp = client.get("/projects/", headers=auth_headers)
    assert resp.status_code == 200
    assert len(resp.json()) == 3

def test_get_project(client, auth_headers, project_id):
    resp = client.get(f"/projects/{project_id}", headers=auth_headers)
    assert resp.status_code == 200
    assert resp.json()["name"] == "Test Project"
    assert "owner" in resp.json()

def test_get_project_not_found(client, auth_headers):
    resp = client.get("/projects/999", headers=auth_headers)
    assert resp.status_code == 404

def test_update_project(client, auth_headers, project_id):
    resp = client.patch(f"/projects/{project_id}", json={
        "name": "Updated Name",
    }, headers=auth_headers)
    assert resp.status_code == 200
    assert resp.json()["name"] == "Updated Name"

def test_delete_project(client, auth_headers, project_id):
    resp = client.delete(f"/projects/{project_id}", headers=auth_headers)
    assert resp.status_code == 204
    
    resp = client.get(f"/projects/{project_id}", headers=auth_headers)
    assert resp.status_code == 404

def test_cannot_access_other_users_project(client, auth_headers, project_id):
    # Create second user
    client.post("/auth/register", json={
        "name": "Other", "email": "other@test.com", "password": "password123",
    })
    resp = client.post("/auth/login", data={
        "username": "other@test.com", "password": "password123",
    })
    other_headers = {"Authorization": f"Bearer {resp.json()['access_token']}"}
    
    # Can't access user 1's project
    resp = client.get(f"/projects/{project_id}", headers=other_headers)
    assert resp.status_code == 403

    # Can't update
    resp = client.patch(f"/projects/{project_id}", json={"name": "Hacked"}, headers=other_headers)
    assert resp.status_code == 403

    # Can't delete
    resp = client.delete(f"/projects/{project_id}", headers=other_headers)
    assert resp.status_code == 403
```

### tests/test_tasks.py

```python
import pytest

@pytest.fixture
def task_id(client, auth_headers, project_id) -> int:
    resp = client.post(f"/projects/{project_id}/tasks/", json={
        "title": "Test Task",
        "description": "Do something",
        "priority": "high",
    }, headers=auth_headers)
    return resp.json()["id"]

def test_create_task(client, auth_headers, project_id):
    resp = client.post(f"/projects/{project_id}/tasks/", json={
        "title": "Write tests",
        "priority": "urgent",
    }, headers=auth_headers)
    assert resp.status_code == 201
    data = resp.json()
    assert data["title"] == "Write tests"
    assert data["status"] == "todo"
    assert data["priority"] == "urgent"
    assert data["project_id"] == project_id

def test_list_tasks(client, auth_headers, project_id):
    for i in range(5):
        client.post(f"/projects/{project_id}/tasks/", json={
            "title": f"Task {i}",
        }, headers=auth_headers)
    
    resp = client.get(f"/projects/{project_id}/tasks/", headers=auth_headers)
    assert resp.status_code == 200
    assert len(resp.json()) == 5

def test_filter_tasks_by_status(client, auth_headers, project_id, task_id):
    # Update task to "in_progress"
    client.patch(
        f"/projects/{project_id}/tasks/{task_id}",
        json={"status": "in_progress"},
        headers=auth_headers,
    )
    # Create another task (stays "todo")
    client.post(f"/projects/{project_id}/tasks/", json={"title": "Another"}, headers=auth_headers)
    
    # Filter by status
    resp = client.get(
        f"/projects/{project_id}/tasks/?status=in_progress",
        headers=auth_headers,
    )
    assert resp.status_code == 200
    tasks = resp.json()
    assert len(tasks) == 1
    assert tasks[0]["status"] == "in_progress"

def test_filter_tasks_by_priority(client, auth_headers, project_id):
    client.post(f"/projects/{project_id}/tasks/", json={
        "title": "Low", "priority": "low",
    }, headers=auth_headers)
    client.post(f"/projects/{project_id}/tasks/", json={
        "title": "Urgent", "priority": "urgent",
    }, headers=auth_headers)
    
    resp = client.get(
        f"/projects/{project_id}/tasks/?priority=urgent",
        headers=auth_headers,
    )
    assert len(resp.json()) == 1
    assert resp.json()[0]["title"] == "Urgent"

def test_get_task_detail(client, auth_headers, project_id, task_id):
    resp = client.get(
        f"/projects/{project_id}/tasks/{task_id}",
        headers=auth_headers,
    )
    assert resp.status_code == 200
    data = resp.json()
    assert data["title"] == "Test Task"
    assert "comments" in data

def test_update_task_status(client, auth_headers, project_id, task_id):
    resp = client.patch(
        f"/projects/{project_id}/tasks/{task_id}",
        json={"status": "done"},
        headers=auth_headers,
    )
    assert resp.status_code == 200
    assert resp.json()["status"] == "done"

def test_delete_task(client, auth_headers, project_id, task_id):
    resp = client.delete(
        f"/projects/{project_id}/tasks/{task_id}",
        headers=auth_headers,
    )
    assert resp.status_code == 204

def test_task_not_found(client, auth_headers, project_id):
    resp = client.get(
        f"/projects/{project_id}/tasks/999",
        headers=auth_headers,
    )
    assert resp.status_code == 404

# ── Comment tests ──

def test_add_comment(client, auth_headers, project_id, task_id):
    resp = client.post(
        f"/projects/{project_id}/tasks/{task_id}/comments",
        json={"content": "Looks good!"},
        headers=auth_headers,
    )
    assert resp.status_code == 201
    data = resp.json()
    assert data["content"] == "Looks good!"
    assert "author" in data
    assert data["author"]["email"] == "test@test.com"

def test_list_comments(client, auth_headers, project_id, task_id):
    for i in range(3):
        client.post(
            f"/projects/{project_id}/tasks/{task_id}/comments",
            json={"content": f"Comment {i}"},
            headers=auth_headers,
        )
    
    resp = client.get(
        f"/projects/{project_id}/tasks/{task_id}/comments",
        headers=auth_headers,
    )
    assert resp.status_code == 200
    assert len(resp.json()) == 3

def test_delete_project_cascades_tasks(client, auth_headers, project_id, task_id):
    """Deleting project should delete its tasks too."""
    client.delete(f"/projects/{project_id}", headers=auth_headers)
    
    # Task is gone (project gone → 404 on project check)
    resp = client.get(f"/projects/{project_id}/tasks/{task_id}", headers=auth_headers)
    assert resp.status_code == 404
```

---

## 11. Running the Project

```bash
# 1. Install dependencies
pip install fastapi uvicorn[standard] sqlalchemy alembic pydantic-settings \
            pyjwt passlib[bcrypt] pytest httpx

# 2. Initialize Alembic (first time only)
alembic init alembic
# Configure alembic.ini and env.py as shown in section 9

# 3. Generate and apply migrations
alembic revision --autogenerate -m "initial schema"
alembic upgrade head

# 4. Run the app
uvicorn app.main:app --reload

# 5. Open docs
# http://localhost:8000/docs

# 6. Run tests
pytest -v

# 7. Run tests with coverage
pytest --cov=app --cov-report=term-missing
```

### Example API workflow in Swagger

```
1. POST /auth/register  →  {"name": "Alice", "email": "alice@test.com", "password": "secret123"}
2. POST /auth/login     →  {"username": "alice@test.com", "password": "secret123"}
   → Copy the access_token
3. Click 🔒 Authorize button → paste token
4. POST /projects/      →  {"name": "My First Project"}
5. POST /projects/1/tasks/ → {"title": "Write README", "priority": "high"}
6. PATCH /projects/1/tasks/1 → {"status": "in_progress"}
7. POST /projects/1/tasks/1/comments → {"content": "Working on it!"}
8. GET /projects/1/tasks/1  → see task with comments
```

---

## 12. What You've Built — Architecture Summary

```
Request flow:

  Client
    ↓  HTTP request (JSON body, JWT in header)
  FastAPI Middleware
    ↓  CORS, logging, timing
  Router
    ↓  Parse path/query params, validate body (Pydantic)
    ↓  Depends(get_current_user) → verify JWT → load user from DB
    ↓  Depends(get_db) → database session
  Service / Router logic
    ↓  Business logic, permission checks
    ↓  SQLAlchemy queries (select, insert, update, delete)
  SQLAlchemy ORM
    ↓  Generate SQL, execute via engine
  SQLite / PostgreSQL
    ↓  Return data
  Pydantic response_model
    ↓  Serialize to JSON (exclude internal fields)
  Client
    ← HTTP response (JSON)
```

```
Key patterns used:
  ✅ Layered architecture (router → service → DB)
  ✅ Dependency injection (Depends)
  ✅ Repository-like pattern (DB queries in router/service)
  ✅ Schema separation (Create / Update / Response)
  ✅ JWT stateless auth
  ✅ Ownership-based access control
  ✅ Cascade deletes (project → tasks → comments)
  ✅ Eager loading (selectinload to avoid N+1)
  ✅ Custom exception hierarchy
  ✅ Test isolation (in-memory DB, dependency overrides)
```

---

## 📝 Practice Tasks

### Task 1: Extend the API
Add these features to the project:
- `GET /users/me` — return current user profile
- `PATCH /users/me` — update own profile
- `GET /projects/{id}/stats` — return task counts by status and priority
- Add `due_date` field to tasks, write the Alembic migration

### Task 2: Pagination
Replace list endpoints with paginated responses:
```json
{
    "items": [...],
    "total": 42,
    "page": 2,
    "size": 20,
    "pages": 3
}
```
Create a reusable pagination dependency.

### Task 3: Search & Advanced Filtering
Add `GET /tasks/search` endpoint that:
- Searches across all user's projects
- Filters by: status, priority, assignee, date range
- Supports text search in title/description
- Sorts by: created_at, priority, status

### Task 4: Task Status Transitions
Add validation that tasks can only transition in valid order:
- todo → in_progress
- in_progress → review
- review → done or in_progress (sent back)
- done → (cannot change)
Reject invalid transitions with 400 error.

### Task 5: Activity Log
Add an `activities` table that records:
- User created project
- User created/updated/deleted task
- Task status changed (from → to)
- Comment added
Add `GET /projects/{id}/activity` endpoint.

### Task 6: Deploy-Ready
Make the project production-ready:
- Environment-based config (.env.dev, .env.prod)
- Dockerfile + docker-compose.yml
- PostgreSQL instead of SQLite
- Health check with DB connectivity test
- Rate limiting middleware
- Structured JSON logging

---

## 📚 Resources

- [FastAPI Full Stack Template](https://github.com/tiangolo/full-stack-fastapi-template)
- [FastAPI Best Practices](https://github.com/zhanymkanov/fastapi-best-practices)
- [FastAPI + SQLAlchemy Tutorial](https://fastapi.tiangolo.com/tutorial/sql-databases/)
- [Alembic with FastAPI](https://alembic.sqlalchemy.org/en/latest/tutorial.html)
- [FastAPI Testing](https://fastapi.tiangolo.com/tutorial/testing/)
- [JWT Best Practices](https://auth0.com/blog/a-look-at-the-latest-draft-for-jwt-bcp/)
- [12-Factor App](https://12factor.net/)

---

## 🔑 Key Takeaways

1. **Layered architecture** — routers handle HTTP, services handle logic, models handle DB. Don't mix layers.
2. **Schema separation is critical** — `UserCreate` (with password), `UserResponse` (without password), `UserUpdate` (all optional). Never expose internal fields.
3. **Dependency injection chains** — `get_db → get_current_user → route handler`. FastAPI resolves the chain automatically.
4. **Ownership checks everywhere** — verify `project.owner_id == current_user.id` before every operation. Security is not optional.
5. **Cascade deletes** — `cascade="all, delete-orphan"` on relationships. Delete project → tasks → comments automatically.
6. **`selectinload`** — always eagerly load relationships when you need them in response. Prevents N+1.
7. **`model_dump(exclude_unset=True)`** — the key to PATCH (partial updates). Only update fields the client actually sent.
8. **Test fixtures compose** — `client → auth_headers → project_id → task_id`. Each builds on the previous.
9. **`dependency_overrides`** — swap real DB for test DB. Tests run in `:memory:` SQLite instantly.
10. **Alembic for production** — never use `create_all()` in production. Generate migrations, review them, apply them.

---

> 🎉 **Phase 5 Complete!** You've built a production-grade REST API.
>
> **Next up — Phase 6: Data Engineering Foundations (Days 22-28)**
> - Day 22: NumPy
> - Day 23: Pandas Core
> - Day 24: Pandas Advanced
> - Day 25: Data Validation & Quality
> - Day 26: Working with APIs & Web Scraping
> - Day 27: Working with Files at Scale
> - Day 28: Polars

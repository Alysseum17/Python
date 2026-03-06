# FastAPI Authentication & Authorization — Повний Гайд

## Зміст

1. [Основи: як працює автентифікація в вебі](#1-основи)
2. [JWT Authentication — повний цикл](#2-jwt-authentication)
3. [Refresh Tokens — безпечне оновлення сесій](#3-refresh-tokens)
4. [OAuth2 — вхід через Google, GitHub, Facebook](#4-oauth2-соціальний-вхід)
5. [Email Verification — підтвердження пошти](#5-email-verification)
6. [Two-Factor Authentication (2FA) — двофакторна автентифікація](#6-two-factor-authentication-2fa)
7. [Password Reset — скидання пароля](#7-password-reset)
8. [Role-Based Access Control (RBAC) — ролі та дозволи](#8-role-based-access-control-rbac)
9. [API Key Authentication](#9-api-key-authentication)
10. [Rate Limiting для auth endpoints](#10-rate-limiting)
11. [Security Best Practices](#11-security-best-practices)
12. [Повна архітектура auth системи](#12-повна-архітектура)

---

## 1. Основи

### Автентифікація vs Авторизація

```
Автентифікація (Authentication) — ХТО ти?
  → "Я — Олексій, ось мій пароль / токен / відбиток пальця"
  → Перевірка ідентичності

Авторизація (Authorization) — ЩО тобі дозволено?
  → "Олексій має роль admin, тому може видаляти юзерів"
  → Перевірка дозволів

Flow:
  1. Клієнт автентифікується (логін + пароль, OAuth, API key)
  2. Сервер повертає токен (JWT)
  3. Клієнт шле токен з кожним запитом
  4. Сервер перевіряє токен (автентифікація) і права (авторизація)
```

### Методи автентифікації в вебі

```
1. Session-based (класичний)
   → Сервер зберігає сесію в пам'яті/Redis
   → Клієнт отримує session cookie
   → Stateful — сервер пам'ятає стан
   → Підходить для: MPA (multi-page apps), SSR

2. Token-based / JWT (сучасний)
   → Сервер НЕ зберігає нічого
   → Клієнт зберігає JWT токен
   → Stateless — вся інфо в токені
   → Підходить для: SPA, мобільні додатки, мікросервіси, API

3. OAuth2 (делегована)
   → Вхід через Google/GitHub/Facebook
   → Сервер не бачить пароль — його перевіряє провайдер
   → Підходить для: соціальний логін, інтеграції

4. API Key
   → Статичний ключ в заголовку
   → Простий, без expiration (або з ним)
   → Підходить для: server-to-server, public APIs
```

---

## 2. JWT Authentication

### Як працює JWT

```
JWT = JSON Web Token — self-contained токен з інформацією про юзера.

Структура (3 частини, розділені крапками):

  eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxfQ.abc123signature
  ─────── header ───────.────── payload ──────.── signature ──

Header (base64):
  {"alg": "HS256", "typ": "JWT"}

Payload (base64) — claims (твердження):
  {
    "sub": "alice@test.com",      ← subject (ідентифікатор юзера)
    "user_id": 1,                 ← кастомні дані
    "role": "admin",              ← кастомні дані
    "iat": 1707955200,            ← issued at (коли створений)
    "exp": 1707958800             ← expires at (коли спливає)
  }

Signature:
  HMAC-SHA256(
    base64(header) + "." + base64(payload),
    SECRET_KEY
  )

Ключовий принцип:
  → Payload НЕ зашифрований — будь-хто може прочитати!
  → Signature гарантує що payload НЕ підроблений
  → Тому НІКОЛИ не клади в JWT: паролі, кредитки, секрети
  → Тільки: user_id, email, role, expiration
```

### Повна реалізація JWT auth

```python
# ── requirements.txt ──
# fastapi
# uvicorn[standard]
# sqlalchemy
# pyjwt
# passlib[bcrypt]
# pydantic-settings
# python-multipart  ← потрібен для OAuth2PasswordRequestForm
```

#### config.py

```python
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    # JWT
    secret_key: str = "your-super-secret-key-min-32-chars-long!"
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7
    
    # Database
    database_url: str = "sqlite:///./app.db"
    
    # Email (для верифікації та reset)
    smtp_host: str = "smtp.gmail.com"
    smtp_port: int = 587
    smtp_user: str = ""
    smtp_password: str = ""
    
    # OAuth2 providers
    google_client_id: str = ""
    google_client_secret: str = ""
    github_client_id: str = ""
    github_client_secret: str = ""
    
    # App
    frontend_url: str = "http://localhost:3000"
    backend_url: str = "http://localhost:8000"
    
    model_config = {"env_file": ".env"}

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

#### models/user.py

```python
from datetime import datetime
from sqlalchemy import String, Boolean, DateTime, Integer, ForeignKey, Text
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(100))
    hashed_password: Mapped[str | None] = mapped_column(String(255), default=None)
    # None для OAuth-only юзерів (без пароля)
    
    # Статус акаунту
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    is_verified: Mapped[bool] = mapped_column(Boolean, default=False)
    is_superuser: Mapped[bool] = mapped_column(Boolean, default=False)
    
    # Роль
    role: Mapped[str] = mapped_column(String(20), default="user")
    # "user", "editor", "admin", "superadmin"
    
    # 2FA
    totp_secret: Mapped[str | None] = mapped_column(String(32), default=None)
    is_2fa_enabled: Mapped[bool] = mapped_column(Boolean, default=False)
    
    # OAuth провайдери
    oauth_provider: Mapped[str | None] = mapped_column(String(20), default=None)
    oauth_provider_id: Mapped[str | None] = mapped_column(String(255), default=None)
    avatar_url: Mapped[str | None] = mapped_column(String(500), default=None)
    
    # Timestamps
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.now)
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.now, onupdate=datetime.now
    )
    last_login: Mapped[datetime | None] = mapped_column(DateTime, default=None)
    
    # Refresh tokens
    refresh_tokens: Mapped[list["RefreshToken"]] = relationship(
        back_populates="user", cascade="all, delete-orphan"
    )

class RefreshToken(Base):
    __tablename__ = "refresh_tokens"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    token: Mapped[str] = mapped_column(String(500), unique=True, index=True)
    user_id: Mapped[int] = mapped_column(Integer, ForeignKey("users.id"))
    expires_at: Mapped[datetime] = mapped_column(DateTime)
    is_revoked: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.now)
    device_info: Mapped[str | None] = mapped_column(String(500), default=None)
    
    user: Mapped["User"] = relationship(back_populates="refresh_tokens")
```

#### services/auth_service.py — ядро авторизації

```python
from datetime import datetime, timedelta, timezone
import secrets
import jwt
from passlib.context import CryptContext
from sqlalchemy.orm import Session
from sqlalchemy import select, delete

from app.config import get_settings
from app.models.user import User, RefreshToken

settings = get_settings()
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# ── Password хешування ──

def hash_password(password: str) -> str:
    """Хешує пароль через bcrypt (односторонній хеш)."""
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Перевіряє пароль проти хешу."""
    return pwd_context.verify(plain_password, hashed_password)

# ── JWT токени ──

def create_access_token(user: User) -> str:
    """Створює short-lived access token (30 хв)."""
    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.access_token_expire_minutes)
    payload = {
        "sub": str(user.id),       # subject — ID юзера
        "email": user.email,
        "role": user.role,
        "type": "access",           # тип токену
        "iat": datetime.now(timezone.utc),
        "exp": expire,
    }
    return jwt.encode(payload, settings.secret_key, algorithm=settings.algorithm)

def create_refresh_token(user: User, db: Session, device_info: str | None = None) -> str:
    """Створює long-lived refresh token (7 днів) і зберігає в БД."""
    token_value = secrets.token_urlsafe(64)
    expires_at = datetime.now(timezone.utc) + timedelta(days=settings.refresh_token_expire_days)
    
    db_token = RefreshToken(
        token=token_value,
        user_id=user.id,
        expires_at=expires_at,
        device_info=device_info,
    )
    db.add(db_token)
    db.commit()
    
    return token_value

def decode_access_token(token: str) -> dict:
    """Декодує і валідує access token."""
    try:
        payload = jwt.decode(token, settings.secret_key, algorithms=[settings.algorithm])
        if payload.get("type") != "access":
            raise ValueError("Not an access token")
        return payload
    except jwt.ExpiredSignatureError:
        raise ValueError("Token expired")
    except jwt.InvalidTokenError as e:
        raise ValueError(f"Invalid token: {e}")

def verify_refresh_token(token: str, db: Session) -> RefreshToken:
    """Перевіряє refresh token в БД."""
    db_token = db.scalars(
        select(RefreshToken).where(
            RefreshToken.token == token,
            RefreshToken.is_revoked == False,
        )
    ).first()
    
    if not db_token:
        raise ValueError("Invalid refresh token")
    if db_token.expires_at < datetime.now(timezone.utc):
        db_token.is_revoked = True
        db.commit()
        raise ValueError("Refresh token expired")
    
    return db_token

def revoke_refresh_token(token: str, db: Session) -> None:
    """Відкликає конкретний refresh token (logout)."""
    db_token = db.scalars(
        select(RefreshToken).where(RefreshToken.token == token)
    ).first()
    if db_token:
        db_token.is_revoked = True
        db.commit()

def revoke_all_refresh_tokens(user_id: int, db: Session) -> None:
    """Відкликає ВСІ refresh tokens юзера (logout everywhere)."""
    db.execute(
        delete(RefreshToken).where(
            RefreshToken.user_id == user_id,
            RefreshToken.is_revoked == False,
        )
    )
    db.commit()

# ── Реєстрація та логін ──

def register_user(db: Session, name: str, email: str, password: str) -> User:
    """Реєстрація нового юзера."""
    existing = db.scalars(select(User).where(User.email == email.lower())).first()
    if existing:
        raise ValueError(f"Email {email} вже зареєстрований")
    
    user = User(
        name=name.strip(),
        email=email.lower().strip(),
        hashed_password=hash_password(password),
        is_verified=False,  # потрібно підтвердити email
    )
    db.add(user)
    db.commit()
    db.refresh(user)
    return user

def authenticate_user(db: Session, email: str, password: str) -> User:
    """Перевірка credentials."""
    user = db.scalars(select(User).where(User.email == email.lower())).first()
    
    if not user:
        raise ValueError("Невірний email або пароль")
    if not user.hashed_password:
        raise ValueError("Цей акаунт використовує соціальний вхід. Увійдіть через Google/GitHub.")
    if not verify_password(password, user.hashed_password):
        raise ValueError("Невірний email або пароль")
    if not user.is_active:
        raise ValueError("Акаунт деактивовано")
    
    # Оновити last_login
    user.last_login = datetime.now()
    db.commit()
    
    return user
```

#### schemas/auth.py

```python
from pydantic import BaseModel, Field
from datetime import datetime

class RegisterRequest(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: str = Field(pattern=r"^[\w.-]+@[\w.-]+\.\w+$")
    password: str = Field(min_length=8, max_length=100, description="Мінімум 8 символів")

class LoginRequest(BaseModel):
    email: str
    password: str

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int  # секунди до expiration

class RefreshRequest(BaseModel):
    refresh_token: str

class PasswordResetRequest(BaseModel):
    email: str

class PasswordResetConfirm(BaseModel):
    token: str
    new_password: str = Field(min_length=8)

class ChangePasswordRequest(BaseModel):
    current_password: str
    new_password: str = Field(min_length=8)

class Enable2FAResponse(BaseModel):
    secret: str
    qr_code_url: str

class Verify2FARequest(BaseModel):
    code: str = Field(min_length=6, max_length=6)
```

#### dependencies.py

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session

from app.database import get_db
from app.models.user import User
from app.services.auth_service import decode_access_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db),
) -> User:
    """Витягує поточного юзера з JWT токена."""
    try:
        payload = decode_access_token(token)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=str(e),
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    user = db.get(User, int(payload["sub"]))
    if not user or not user.is_active:
        raise HTTPException(status_code=401, detail="Юзер не знайдений або деактивований")
    return user

async def get_current_verified_user(
    current_user: User = Depends(get_current_user),
) -> User:
    """Тільки юзери з підтвердженим email."""
    if not current_user.is_verified:
        raise HTTPException(status_code=403, detail="Email не підтверджений")
    return current_user

async def get_current_admin(
    current_user: User = Depends(get_current_user),
) -> User:
    """Тільки адміни."""
    if current_user.role not in ("admin", "superadmin"):
        raise HTTPException(status_code=403, detail="Потрібні права адміністратора")
    return current_user
```

#### routers/auth.py — основний auth router

```python
from fastapi import APIRouter, Depends, HTTPException, status, Request
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session

from app.database import get_db
from app.dependencies import get_current_user
from app.models.user import User
from app.schemas.auth import (
    RegisterRequest, TokenResponse, RefreshRequest,
    ChangePasswordRequest,
)
from app.services.auth_service import (
    register_user, authenticate_user,
    create_access_token, create_refresh_token,
    verify_refresh_token, revoke_refresh_token,
    revoke_all_refresh_tokens, hash_password, verify_password,
)
from app.config import get_settings

settings = get_settings()
router = APIRouter(prefix="/auth", tags=["Authentication"])

@router.post("/register", status_code=201)
def register(data: RegisterRequest, db: Session = Depends(get_db)):
    """Реєстрація нового акаунту."""
    try:
        user = register_user(db, data.name, data.email, data.password)
    except ValueError as e:
        raise HTTPException(status_code=409, detail=str(e))
    
    # TODO: відправити email для верифікації (section 5)
    
    return {"message": "Акаунт створено. Перевірте email для підтвердження.", "user_id": user.id}

@router.post("/login", response_model=TokenResponse)
def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    request: Request = None,
    db: Session = Depends(get_db),
):
    """Логін — повертає access + refresh токени."""
    try:
        user = authenticate_user(db, form_data.username, form_data.password)
    except ValueError as e:
        raise HTTPException(status_code=401, detail=str(e))
    
    # Якщо увімкнена 2FA — не даємо токен одразу
    if user.is_2fa_enabled:
        # Створюємо тимчасовий токен для 2FA verification
        temp_token = create_access_token(user)  # short-lived
        return TokenResponse(
            access_token="",
            refresh_token="",
            token_type="2fa_required",
            expires_in=0,
        )
        # Клієнт має викликати POST /auth/verify-2fa з кодом
    
    # Створюємо токени
    device_info = request.headers.get("user-agent", "unknown") if request else None
    access_token = create_access_token(user)
    refresh_token = create_refresh_token(user, db, device_info)
    
    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token,
        expires_in=settings.access_token_expire_minutes * 60,
    )

@router.post("/refresh", response_model=TokenResponse)
def refresh_tokens(data: RefreshRequest, db: Session = Depends(get_db)):
    """Оновити access token використовуючи refresh token."""
    try:
        db_token = verify_refresh_token(data.refresh_token, db)
    except ValueError as e:
        raise HTTPException(status_code=401, detail=str(e))
    
    user = db.get(User, db_token.user_id)
    if not user or not user.is_active:
        raise HTTPException(status_code=401, detail="Юзер не знайдений")
    
    # Відкликати старий refresh token (rotation)
    db_token.is_revoked = True
    
    # Створити нові токени
    new_access = create_access_token(user)
    new_refresh = create_refresh_token(user, db)
    db.commit()
    
    return TokenResponse(
        access_token=new_access,
        refresh_token=new_refresh,
        expires_in=settings.access_token_expire_minutes * 60,
    )

@router.post("/logout")
def logout(
    data: RefreshRequest,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
):
    """Logout — відкликає refresh token."""
    revoke_refresh_token(data.refresh_token, db)
    return {"message": "Вихід виконано"}

@router.post("/logout-all")
def logout_all(
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
):
    """Logout з усіх пристроїв — відкликає ВСІ refresh tokens."""
    revoke_all_refresh_tokens(current_user.id, db)
    return {"message": "Вихід з усіх пристроїв виконано"}

@router.post("/change-password")
def change_password(
    data: ChangePasswordRequest,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
):
    """Зміна пароля (потрібен поточний пароль)."""
    if not current_user.hashed_password:
        raise HTTPException(status_code=400, detail="Акаунт через OAuth, пароль не встановлено")
    
    if not verify_password(data.current_password, current_user.hashed_password):
        raise HTTPException(status_code=400, detail="Невірний поточний пароль")
    
    current_user.hashed_password = hash_password(data.new_password)
    # Відкликати всі refresh tokens (security)
    revoke_all_refresh_tokens(current_user.id, db)
    db.commit()
    
    return {"message": "Пароль змінено. Увійдіть заново."}

@router.get("/me")
def get_me(current_user: User = Depends(get_current_user)):
    """Отримати інформацію про поточного юзера."""
    return {
        "id": current_user.id,
        "name": current_user.name,
        "email": current_user.email,
        "role": current_user.role,
        "is_verified": current_user.is_verified,
        "is_2fa_enabled": current_user.is_2fa_enabled,
        "oauth_provider": current_user.oauth_provider,
        "avatar_url": current_user.avatar_url,
    }
```

---

## 3. Refresh Tokens

### Навіщо потрібні refresh tokens

```
Проблема з одним access token:
  → Якщо токен живе 30 хвилин — юзер мусить логінитись кожні 30 хв 
  → Якщо токен живе 30 днів — вкрадений токен працює 30 днів 

Рішення: два типи токенів:

  Access Token:
    → Короткий час життя (15-30 хв)
    → Зберігається в пам'яті (JavaScript variable)
    → Використовується для кожного API запиту
    → Якщо вкрадений — працює лише 30 хв

  Refresh Token:
    → Довгий час життя (7-30 днів)
    → Зберігається в httpOnly cookie або secure storage
    → Використовується ТІЛЬКИ для отримання нового access token
    → Зберігається в БД — можна відкликати!

Flow:
  1. Login → отримав access + refresh
  2. Запити → шлеш access token
  3. Access expired → шлеш refresh → отримав новий access + refresh
  4. Refresh expired → логінся заново
  5. Logout → refresh відкликаний в БД

Refresh Token Rotation:
  → Кожен раз коли використовуєш refresh token, видається НОВИЙ
  → Старий стає невалідним
  → Якщо хтось вкрав refresh token і використав — оригінальний перестає працювати
  → Юзер помітить що його "вибило" і змінить пароль
```

### Діаграма flow

```
Client                          Server                        Database
  │                               │                              │
  ├── POST /auth/login ──────────►│                              │
  │   {email, password}           │── verify password ──────────►│
  │                               │◄── user data ───────────────│
  │                               │── create access token        │
  │                               │── create refresh token ─────►│ (save to DB)
  │◄── {access, refresh} ────────│                              │
  │                               │                              │
  ├── GET /api/data ─────────────►│                              │
  │   Authorization: Bearer {at}  │── decode JWT (no DB!) ──┐   │
  │◄── {data} ───────────────────│◄─────────────────────────┘   │
  │                               │                              │
  │  (access token expired)       │                              │
  ├── POST /auth/refresh ────────►│                              │
  │   {refresh_token}             │── verify in DB ─────────────►│
  │                               │◄── token valid ─────────────│
  │                               │── revoke old refresh ───────►│
  │                               │── create new access          │
  │                               │── create new refresh ───────►│
  │◄── {new_access, new_refresh} │                              │
  │                               │                              │
  ├── POST /auth/logout ─────────►│                              │
  │   {refresh_token}             │── revoke in DB ─────────────►│
  │◄── {ok} ─────────────────────│                              │
```

---

## 4. OAuth2 — Соціальний вхід

### Як працює OAuth2 (Authorization Code Flow)

```
OAuth2 Authorization Code Flow — найбезпечніший flow для веб-додатків.

Учасники:
  → Resource Owner (юзер) — хоче увійти
  → Client (твій додаток) — хоче отримати дані юзера
  → Authorization Server (Google/GitHub) — перевіряє юзера
  → Resource Server (Google API) — має дані юзера

Flow:
  1. Юзер натискає "Login with Google"
  2. Твій сервер перенаправляє на Google:
     https://accounts.google.com/o/oauth2/auth?
       client_id=YOUR_ID&
       redirect_uri=http://yourapp.com/auth/google/callback&
       scope=openid email profile&
       response_type=code&
       state=random_csrf_token
  
  3. Юзер логіниться в Google і дає дозвіл
  
  4. Google перенаправляє назад з authorization code:
     http://yourapp.com/auth/google/callback?code=AUTH_CODE&state=random_csrf_token
  
  5. Твій сервер обмінює code на access token (server-to-server):
     POST https://oauth2.googleapis.com/token
     {client_id, client_secret, code, redirect_uri}
     → отримує Google access_token
  
  6. Твій сервер запитує дані юзера:
     GET https://www.googleapis.com/oauth2/v2/userinfo
     Authorization: Bearer {google_access_token}
     → отримує: email, name, picture
  
  7. Створюєш або знаходиш юзера в своїй БД
  
  8. Видаєш СВІЙ JWT token (не Google-ський!)

Важливо:
  → client_secret ніколи не потрапляє в frontend
  → authorization code одноразовий і short-lived
  → Google access_token використовується тільки на бекенді
  → Юзер отримує ТВІЙ JWT, не Google token
```

### Реєстрація OAuth додатків

```
Google:
  1. Зайди на https://console.cloud.google.com/
  2. APIs & Services → Credentials → Create Credentials → OAuth Client ID
  3. Application type: Web application
  4. Authorized redirect URIs: http://localhost:8000/auth/google/callback
  5. Скопіюй Client ID і Client Secret

GitHub:
  1. Зайди на https://github.com/settings/developers
  2. New OAuth App
  3. Authorization callback URL: http://localhost:8000/auth/github/callback
  4. Скопіюй Client ID і Client Secret

Facebook:
  1. Зайди на https://developers.facebook.com/
  2. Create App → Consumer
  3. Facebook Login → Settings
  4. Valid OAuth Redirect URIs: http://localhost:8000/auth/facebook/callback
  5. Скопіюй App ID і App Secret
```

### Реалізація OAuth2 для Google

```python
# services/oauth_service.py
import httpx
from urllib.parse import urlencode
import secrets

from app.config import get_settings

settings = get_settings()

# ── State management (CSRF protection) ──
# В production використовуй Redis
oauth_states: dict[str, str] = {}  # state → provider

def generate_oauth_state(provider: str) -> str:
    """Генерує унікальний state для CSRF protection."""
    state = secrets.token_urlsafe(32)
    oauth_states[state] = provider
    return state

def verify_oauth_state(state: str, provider: str) -> bool:
    """Перевіряє що state валідний."""
    stored_provider = oauth_states.pop(state, None)
    return stored_provider == provider


# ══════════════════════════════════════════
#  GOOGLE
# ══════════════════════════════════════════

GOOGLE_AUTH_URL = "https://accounts.google.com/o/oauth2/v2/auth"
GOOGLE_TOKEN_URL = "https://oauth2.googleapis.com/token"
GOOGLE_USERINFO_URL = "https://www.googleapis.com/oauth2/v2/userinfo"

def get_google_auth_url() -> str:
    """Генерує URL для перенаправлення на Google."""
    state = generate_oauth_state("google")
    params = {
        "client_id": settings.google_client_id,
        "redirect_uri": f"{settings.backend_url}/auth/google/callback",
        "scope": "openid email profile",
        "response_type": "code",
        "state": state,
        "access_type": "offline",    # щоб отримати refresh_token від Google
        "prompt": "consent",
    }
    return f"{GOOGLE_AUTH_URL}?{urlencode(params)}"

async def exchange_google_code(code: str) -> dict:
    """Обмінює authorization code на user info."""
    # 1. Отримати access token від Google
    async with httpx.AsyncClient() as client:
        token_response = await client.post(GOOGLE_TOKEN_URL, data={
            "client_id": settings.google_client_id,
            "client_secret": settings.google_client_secret,
            "code": code,
            "redirect_uri": f"{settings.backend_url}/auth/google/callback",
            "grant_type": "authorization_code",
        })
        token_data = token_response.json()
        
        if "error" in token_data:
            raise ValueError(f"Google token error: {token_data['error']}")
        
        google_access_token = token_data["access_token"]
        
        # 2. Отримати дані юзера
        userinfo_response = await client.get(
            GOOGLE_USERINFO_URL,
            headers={"Authorization": f"Bearer {google_access_token}"},
        )
        userinfo = userinfo_response.json()
    
    return {
        "provider": "google",
        "provider_id": userinfo["id"],
        "email": userinfo["email"],
        "name": userinfo.get("name", ""),
        "avatar_url": userinfo.get("picture"),
        "is_verified": userinfo.get("verified_email", False),
    }


# ══════════════════════════════════════════
#  GITHUB
# ══════════════════════════════════════════

GITHUB_AUTH_URL = "https://github.com/login/oauth/authorize"
GITHUB_TOKEN_URL = "https://github.com/login/oauth/access_token"
GITHUB_USER_URL = "https://api.github.com/user"
GITHUB_EMAILS_URL = "https://api.github.com/user/emails"

def get_github_auth_url() -> str:
    state = generate_oauth_state("github")
    params = {
        "client_id": settings.github_client_id,
        "redirect_uri": f"{settings.backend_url}/auth/github/callback",
        "scope": "user:email",
        "state": state,
    }
    return f"{GITHUB_AUTH_URL}?{urlencode(params)}"

async def exchange_github_code(code: str) -> dict:
    async with httpx.AsyncClient() as client:
        # 1. Отримати access token
        token_response = await client.post(
            GITHUB_TOKEN_URL,
            data={
                "client_id": settings.github_client_id,
                "client_secret": settings.github_client_secret,
                "code": code,
            },
            headers={"Accept": "application/json"},
        )
        token_data = token_response.json()
        github_token = token_data["access_token"]
        
        headers = {"Authorization": f"Bearer {github_token}"}
        
        # 2. Отримати профіль
        user_response = await client.get(GITHUB_USER_URL, headers=headers)
        user_data = user_response.json()
        
        # 3. Отримати email (може бути приватним)
        email = user_data.get("email")
        if not email:
            emails_response = await client.get(GITHUB_EMAILS_URL, headers=headers)
            emails = emails_response.json()
            primary = next((e for e in emails if e["primary"]), emails[0])
            email = primary["email"]
    
    return {
        "provider": "github",
        "provider_id": str(user_data["id"]),
        "email": email,
        "name": user_data.get("name") or user_data["login"],
        "avatar_url": user_data.get("avatar_url"),
        "is_verified": True,  # GitHub верифікує email
    }


# ══════════════════════════════════════════
#  FACEBOOK
# ══════════════════════════════════════════

FB_AUTH_URL = "https://www.facebook.com/v18.0/dialog/oauth"
FB_TOKEN_URL = "https://graph.facebook.com/v18.0/oauth/access_token"
FB_USER_URL = "https://graph.facebook.com/me"

def get_facebook_auth_url() -> str:
    state = generate_oauth_state("facebook")
    params = {
        "client_id": settings.facebook_app_id,
        "redirect_uri": f"{settings.backend_url}/auth/facebook/callback",
        "scope": "email,public_profile",
        "state": state,
        "response_type": "code",
    }
    return f"{FB_AUTH_URL}?{urlencode(params)}"

async def exchange_facebook_code(code: str) -> dict:
    async with httpx.AsyncClient() as client:
        token_resp = await client.get(FB_TOKEN_URL, params={
            "client_id": settings.facebook_app_id,
            "client_secret": settings.facebook_app_secret,
            "redirect_uri": f"{settings.backend_url}/auth/facebook/callback",
            "code": code,
        })
        fb_token = token_resp.json()["access_token"]
        
        user_resp = await client.get(FB_USER_URL, params={
            "fields": "id,name,email,picture",
            "access_token": fb_token,
        })
        data = user_resp.json()
    
    return {
        "provider": "facebook",
        "provider_id": data["id"],
        "email": data.get("email", ""),
        "name": data.get("name", ""),
        "avatar_url": data.get("picture", {}).get("data", {}).get("url"),
        "is_verified": True,
    }
```

### OAuth роутер

```python
# routers/oauth.py
from fastapi import APIRouter, Depends, HTTPException, Query
from fastapi.responses import RedirectResponse
from sqlalchemy.orm import Session
from sqlalchemy import select

from app.database import get_db
from app.models.user import User
from app.services.auth_service import create_access_token, create_refresh_token
from app.services.oauth_service import (
    get_google_auth_url, exchange_google_code, verify_oauth_state,
    get_github_auth_url, exchange_github_code,
    get_facebook_auth_url, exchange_facebook_code,
)
from app.config import get_settings

settings = get_settings()
router = APIRouter(prefix="/auth", tags=["OAuth2 Social Login"])

async def _handle_oauth_user(oauth_data: dict, db: Session) -> dict:
    """Знайти або створити юзера з OAuth даних."""
    # 1. Шукаємо по provider + provider_id
    user = db.scalars(
        select(User).where(
            User.oauth_provider == oauth_data["provider"],
            User.oauth_provider_id == oauth_data["provider_id"],
        )
    ).first()
    
    # 2. Шукаємо по email
    if not user and oauth_data["email"]:
        user = db.scalars(select(User).where(User.email == oauth_data["email"])).first()
        if user:
            # Лінкуємо існуючий акаунт з OAuth провайдером
            user.oauth_provider = oauth_data["provider"]
            user.oauth_provider_id = oauth_data["provider_id"]
            if oauth_data.get("avatar_url"):
                user.avatar_url = oauth_data["avatar_url"]
    
    # 3. Створюємо нового юзера
    if not user:
        user = User(
            email=oauth_data["email"],
            name=oauth_data["name"],
            oauth_provider=oauth_data["provider"],
            oauth_provider_id=oauth_data["provider_id"],
            avatar_url=oauth_data.get("avatar_url"),
            is_verified=oauth_data.get("is_verified", True),
            hashed_password=None,  # OAuth юзер — без пароля
        )
        db.add(user)
    
    from datetime import datetime
    user.last_login = datetime.now()
    db.commit()
    db.refresh(user)
    
    # 4. Створити наші JWT токени
    access_token = create_access_token(user)
    refresh_token = create_refresh_token(user, db)
    
    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "user_id": user.id,
    }

# ── Google ──

@router.get("/google")
def google_login():
    """Перенаправити юзера на Google для логіну."""
    return RedirectResponse(url=get_google_auth_url())

@router.get("/google/callback")
async def google_callback(
    code: str = Query(...),
    state: str = Query(...),
    db: Session = Depends(get_db),
):
    """Google перенаправляє сюди після логіну."""
    if not verify_oauth_state(state, "google"):
        raise HTTPException(status_code=400, detail="Invalid state (CSRF protection)")
    
    oauth_data = await exchange_google_code(code)
    tokens = await _handle_oauth_user(oauth_data, db)
    
    # Перенаправити на frontend з токенами
    redirect_url = (
        f"{settings.frontend_url}/auth/callback"
        f"?access_token={tokens['access_token']}"
        f"&refresh_token={tokens['refresh_token']}"
    )
    return RedirectResponse(url=redirect_url)

# ── GitHub ──

@router.get("/github")
def github_login():
    return RedirectResponse(url=get_github_auth_url())

@router.get("/github/callback")
async def github_callback(
    code: str = Query(...),
    state: str = Query(...),
    db: Session = Depends(get_db),
):
    if not verify_oauth_state(state, "github"):
        raise HTTPException(status_code=400, detail="Invalid state")
    
    oauth_data = await exchange_github_code(code)
    tokens = await _handle_oauth_user(oauth_data, db)
    
    redirect_url = (
        f"{settings.frontend_url}/auth/callback"
        f"?access_token={tokens['access_token']}"
        f"&refresh_token={tokens['refresh_token']}"
    )
    return RedirectResponse(url=redirect_url)

# ── Facebook ──

@router.get("/facebook")
def facebook_login():
    return RedirectResponse(url=get_facebook_auth_url())

@router.get("/facebook/callback")
async def facebook_callback(
    code: str = Query(...),
    state: str = Query(...),
    db: Session = Depends(get_db),
):
    if not verify_oauth_state(state, "facebook"):
        raise HTTPException(status_code=400, detail="Invalid state")
    
    oauth_data = await exchange_facebook_code(code)
    tokens = await _handle_oauth_user(oauth_data, db)
    
    redirect_url = (
        f"{settings.frontend_url}/auth/callback"
        f"?access_token={tokens['access_token']}"
        f"&refresh_token={tokens['refresh_token']}"
    )
    return RedirectResponse(url=redirect_url)
```

---

## 5. Email Verification

### Flow підтвердження пошти

```
1. Юзер реєструється → create user з is_verified=False
2. Сервер генерує verification token (JWT з type="email_verify", exp=24h)
3. Сервер відправляє email з посиланням:
   https://yourapp.com/verify-email?token=xxx
4. Юзер натискає посилання → frontend шле POST /auth/verify-email {token}
5. Сервер перевіряє token → встановлює is_verified=True
```

### Реалізація

```python
# services/email_service.py
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from datetime import datetime, timedelta, timezone
import jwt

from app.config import get_settings

settings = get_settings()

def create_email_token(user_id: int, email: str, purpose: str, hours: int = 24) -> str:
    """Створює токен для email операцій (verify, reset)."""
    payload = {
        "sub": str(user_id),
        "email": email,
        "purpose": purpose,  # "email_verify" або "password_reset"
        "exp": datetime.now(timezone.utc) + timedelta(hours=hours),
    }
    return jwt.encode(payload, settings.secret_key, algorithm=settings.algorithm)

def decode_email_token(token: str, expected_purpose: str) -> dict:
    """Декодує і перевіряє email токен."""
    try:
        payload = jwt.decode(token, settings.secret_key, algorithms=[settings.algorithm])
        if payload.get("purpose") != expected_purpose:
            raise ValueError("Wrong token purpose")
        return payload
    except jwt.ExpiredSignatureError:
        raise ValueError("Посилання прострочене")
    except jwt.InvalidTokenError:
        raise ValueError("Невалідне посилання")

def send_email(to: str, subject: str, html_body: str) -> None:
    """Відправляє email через SMTP."""
    msg = MIMEMultipart("alternative")
    msg["Subject"] = subject
    msg["From"] = settings.smtp_user
    msg["To"] = to
    msg.attach(MIMEText(html_body, "html"))
    
    with smtplib.SMTP(settings.smtp_host, settings.smtp_port) as server:
        server.starttls()
        server.login(settings.smtp_user, settings.smtp_password)
        server.sendmail(settings.smtp_user, to, msg.as_string())

def send_verification_email(user_id: int, email: str, name: str) -> None:
    """Відправляє email для підтвердження."""
    token = create_email_token(user_id, email, "email_verify", hours=24)
    verify_url = f"{settings.frontend_url}/verify-email?token={token}"
    
    html = f"""
    <h2>Привіт, {name}!</h2>
    <p>Дякуємо за реєстрацію. Підтвердіть email натиснувши кнопку:</p>
    <a href="{verify_url}" 
       style="background: #4CAF50; color: white; padding: 12px 24px; 
              text-decoration: none; border-radius: 4px;">
       Підтвердити Email
    </a>
    <p>Або скопіюйте посилання: {verify_url}</p>
    <p>Посилання діє 24 години.</p>
    """
    
    send_email(email, "Підтвердіть ваш email", html)

def send_password_reset_email(user_id: int, email: str, name: str) -> None:
    """Відправляє email для скидання пароля."""
    token = create_email_token(user_id, email, "password_reset", hours=1)
    reset_url = f"{settings.frontend_url}/reset-password?token={token}"
    
    html = f"""
    <h2>Скидання пароля</h2>
    <p>Привіт, {name}! Ви запросили скидання пароля.</p>
    <a href="{reset_url}"
       style="background: #2196F3; color: white; padding: 12px 24px;
              text-decoration: none; border-radius: 4px;">
       Скинути Пароль
    </a>
    <p>Посилання діє 1 годину.</p>
    <p>Якщо ви не робили цей запит — ігноруйте цей лист.</p>
    """
    
    send_email(email, "Скидання пароля", html)
```

### Endpoint для верифікації

```python
# В routers/auth.py додати:

from app.services.email_service import (
    decode_email_token, send_verification_email, send_password_reset_email,
)

@router.post("/verify-email")
def verify_email(token: str, db: Session = Depends(get_db)):
    """Підтвердити email по токену з листа."""
    try:
        payload = decode_email_token(token, "email_verify")
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    
    user = db.get(User, int(payload["sub"]))
    if not user:
        raise HTTPException(status_code=404, detail="Юзер не знайдений")
    if user.is_verified:
        return {"message": "Email вже підтверджений"}
    
    user.is_verified = True
    db.commit()
    
    return {"message": "Email успішно підтверджено!"}

@router.post("/resend-verification")
def resend_verification(
    current_user: User = Depends(get_current_user),
):
    """Повторно відправити email для підтвердження."""
    if current_user.is_verified:
        raise HTTPException(status_code=400, detail="Email вже підтверджений")
    
    send_verification_email(current_user.id, current_user.email, current_user.name)
    return {"message": "Лист відправлено"}
```

---

## 6. Two-Factor Authentication (2FA)

### Як працює TOTP (Time-based One-Time Password)

```
TOTP — алгоритм який генерує 6-значний код кожні 30 секунд.

Як це працює:
  1. Сервер генерує секретний ключ (base32 string, 16+ символів)
  2. Юзер додає ключ в Google Authenticator (через QR-код)
  3. Обидві сторони (сервер і додаток) генерують одинаковий код
     бо мають однаковий секрет + однаковий час
  4. При логіні юзер вводить код з додатку
  5. Сервер перевіряє що код співпадає

Бібліотека: pyotp
  pip install pyotp qrcode[pil]
```

### Реалізація 2FA

```python
# services/totp_service.py
import pyotp
import qrcode
import io
import base64

def generate_totp_secret() -> str:
    """Генерує секретний ключ для TOTP."""
    return pyotp.random_base32()

def get_totp_uri(secret: str, email: str, app_name: str = "MyApp") -> str:
    """Генерує URI для QR-коду."""
    return pyotp.totp.TOTP(secret).provisioning_uri(
        name=email,
        issuer_name=app_name,
    )

def generate_qr_code_base64(uri: str) -> str:
    """Генерує QR-код як base64 PNG."""
    qr = qrcode.make(uri)
    buffer = io.BytesIO()
    qr.save(buffer, format="PNG")
    return base64.b64encode(buffer.getvalue()).decode()

def verify_totp_code(secret: str, code: str) -> bool:
    """Перевіряє TOTP код. Допускає ±30 секунд."""
    totp = pyotp.TOTP(secret)
    return totp.verify(code, valid_window=1)  # ±1 window (±30 сек)
```

### 2FA ендпоінти

```python
# В routers/auth.py додати:

from app.services.totp_service import (
    generate_totp_secret, get_totp_uri, generate_qr_code_base64, verify_totp_code,
)

@router.post("/2fa/enable")
def enable_2fa(current_user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    """Крок 1: згенерувати секрет і QR-код."""
    if current_user.is_2fa_enabled:
        raise HTTPException(status_code=400, detail="2FA вже увімкнено")
    
    secret = generate_totp_secret()
    uri = get_totp_uri(secret, current_user.email)
    qr_base64 = generate_qr_code_base64(uri)
    
    # Зберігаємо секрет (але 2FA ще не активна!)
    current_user.totp_secret = secret
    db.commit()
    
    return {
        "secret": secret,              # для ручного введення
        "qr_code": f"data:image/png;base64,{qr_base64}",  # для сканування
        "message": "Відскануйте QR-код в Google Authenticator, потім підтвердіть кодом",
    }

@router.post("/2fa/confirm")
def confirm_2fa(
    data: Verify2FARequest,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
):
    """Крок 2: підтвердити що 2FA працює (ввести код з додатку)."""
    if not current_user.totp_secret:
        raise HTTPException(status_code=400, detail="Спочатку виконайте /2fa/enable")
    
    if not verify_totp_code(current_user.totp_secret, data.code):
        raise HTTPException(status_code=400, detail="Невірний код. Спробуйте ще раз.")
    
    current_user.is_2fa_enabled = True
    db.commit()
    
    return {"message": "2FA успішно увімкнено!"}

@router.post("/2fa/disable")
def disable_2fa(
    data: Verify2FARequest,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
):
    """Вимкнути 2FA (потрібен поточний код)."""
    if not current_user.is_2fa_enabled:
        raise HTTPException(status_code=400, detail="2FA не увімкнено")
    
    if not verify_totp_code(current_user.totp_secret, data.code):
        raise HTTPException(status_code=400, detail="Невірний код")
    
    current_user.is_2fa_enabled = False
    current_user.totp_secret = None
    db.commit()
    
    return {"message": "2FA вимкнено"}

@router.post("/2fa/verify")
def verify_2fa_login(
    data: Verify2FARequest,
    temp_token: str,  # тимчасовий токен з /login
    db: Session = Depends(get_db),
):
    """Крок 2 логіну: ввести 2FA код після пароля."""
    try:
        payload = decode_access_token(temp_token)
    except ValueError as e:
        raise HTTPException(status_code=401, detail=str(e))
    
    user = db.get(User, int(payload["sub"]))
    if not user:
        raise HTTPException(status_code=401, detail="Юзер не знайдений")
    
    if not verify_totp_code(user.totp_secret, data.code):
        raise HTTPException(status_code=400, detail="Невірний 2FA код")
    
    # 2FA пройдена — видаємо повні токени
    access_token = create_access_token(user)
    refresh_token = create_refresh_token(user, db)
    
    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token,
        expires_in=settings.access_token_expire_minutes * 60,
    )
```

### Логін з 2FA — повний flow

```
Без 2FA:
  POST /auth/login {email, password} → {access_token, refresh_token}

З 2FA:
  POST /auth/login {email, password}
    → {token_type: "2fa_required", temp_token: "xxx"}
  
  POST /auth/2fa/verify {code: "123456", temp_token: "xxx"}
    → {access_token, refresh_token}
```

---

## 7. Password Reset

```python
# В routers/auth.py додати:

@router.post("/forgot-password")
def forgot_password(email: str, db: Session = Depends(get_db)):
    """Запросити скидання пароля."""
    user = db.scalars(select(User).where(User.email == email.lower())).first()
    
    # ЗАВЖДИ відповідаємо success (не розкриваємо чи існує email)
    if user and user.hashed_password:
        send_password_reset_email(user.id, user.email, user.name)
    
    return {"message": "Якщо email зареєстрований, ви отримаєте лист для скидання пароля."}

@router.post("/reset-password")
def reset_password(data: PasswordResetConfirm, db: Session = Depends(get_db)):
    """Скинути пароль по токену з листа."""
    try:
        payload = decode_email_token(data.token, "password_reset")
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    
    user = db.get(User, int(payload["sub"]))
    if not user:
        raise HTTPException(status_code=404, detail="Юзер не знайдений")
    
    user.hashed_password = hash_password(data.new_password)
    # Відкликати всі refresh tokens (security)
    revoke_all_refresh_tokens(user.id, db)
    db.commit()
    
    return {"message": "Пароль скинуто. Увійдіть з новим паролем."}
```

---

## 8. Role-Based Access Control (RBAC)

### Система ролей

```
Ієрархія:
  superadmin — повний контроль (керування адмінами)
  admin — керування юзерами, контентом
  editor — створення і редагування контенту
  user — базовий доступ (тільки своє)

Permissions (дозволи):
  user:        read:own, write:own
  editor:      read:own, write:own, write:content
  admin:       read:all, write:all, manage:users
  superadmin:  read:all, write:all, manage:users, manage:admins
```

### Реалізація RBAC

```python
# dependencies.py — role-based dependencies

from functools import wraps
from fastapi import Depends, HTTPException

# ── Permission checker ──

class RoleChecker:
    """Dependency для перевірки ролі."""
    
    def __init__(self, allowed_roles: list[str]):
        self.allowed_roles = allowed_roles
    
    def __call__(self, current_user: User = Depends(get_current_user)):
        if current_user.role not in self.allowed_roles:
            raise HTTPException(
                status_code=403,
                detail=f"Потрібна роль: {', '.join(self.allowed_roles)}. "
                       f"Ваша роль: {current_user.role}",
            )
        return current_user

# Готові dependency для різних рівнів
require_user = RoleChecker(["user", "editor", "admin", "superadmin"])
require_editor = RoleChecker(["editor", "admin", "superadmin"])
require_admin = RoleChecker(["admin", "superadmin"])
require_superadmin = RoleChecker(["superadmin"])

# ── Використання в роутерах ──

@router.get("/admin/users")
def list_all_users(
    admin: User = Depends(require_admin),
    db: Session = Depends(get_db),
):
    """Тільки admin і superadmin можуть бачити всіх юзерів."""
    return db.scalars(select(User)).all()

@router.patch("/admin/users/{user_id}/role")
def change_user_role(
    user_id: int,
    new_role: str,
    admin: User = Depends(require_superadmin),
    db: Session = Depends(get_db),
):
    """Тільки superadmin може змінювати ролі."""
    if new_role not in ("user", "editor", "admin"):
        raise HTTPException(status_code=400, detail="Невалідна роль")
    
    user = db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404)
    
    user.role = new_role
    db.commit()
    return {"message": f"Роль {user.email} змінена на {new_role}"}

@router.delete("/admin/users/{user_id}")
def deactivate_user(
    user_id: int,
    admin: User = Depends(require_admin),
    db: Session = Depends(get_db),
):
    """Admin може деактивувати юзерів."""
    user = db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404)
    if user.role in ("admin", "superadmin") and admin.role != "superadmin":
        raise HTTPException(status_code=403, detail="Не можна деактивувати адміна")
    
    user.is_active = False
    revoke_all_refresh_tokens(user.id, db)
    db.commit()
    return {"message": f"Юзер {user.email} деактивовано"}

# ── Resource ownership (комбінація ролі + власності) ──

@router.delete("/posts/{post_id}")
def delete_post(
    post_id: int,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
):
    """Видалити пост — тільки автор або admin."""
    post = db.get(Post, post_id)
    if not post:
        raise HTTPException(status_code=404)
    
    # Власник або адмін
    if post.author_id != current_user.id and current_user.role not in ("admin", "superadmin"):
        raise HTTPException(status_code=403, detail="Тільки автор або адмін може видалити пост")
    
    db.delete(post)
    db.commit()
```

---

## 9. API Key Authentication

```python
# Для server-to-server або public API доступу

from fastapi import Security
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key")

# Простий варіант — статичні ключі
API_KEYS = {
    "key-for-frontend-app": {"name": "Frontend App", "role": "user"},
    "key-for-data-pipeline": {"name": "Data Pipeline", "role": "service"},
    "key-for-admin-tool": {"name": "Admin Tool", "role": "admin"},
}

async def verify_api_key(api_key: str = Security(api_key_header)) -> dict:
    if api_key not in API_KEYS:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return API_KEYS[api_key]

@router.get("/api/data")
def get_data(client: dict = Depends(verify_api_key)):
    return {"data": "sensitive", "client": client["name"]}

# Продакшн варіант — ключі в БД з hashing
# Зберігай hash ключа (як пароль), не plain text
```

---

## 10. Rate Limiting

```python
from collections import defaultdict
import time
from fastapi import Request, HTTPException

class AuthRateLimiter:
    """Захист auth endpoints від brute force."""
    
    def __init__(self):
        self.attempts: dict[str, list[float]] = defaultdict(list)
        self.blocked: dict[str, float] = {}
    
    def _get_key(self, request: Request) -> str:
        return request.client.host
    
    def check(self, request: Request, max_attempts: int = 5, window: int = 300, block_time: int = 900):
        """
        Перевіряє rate limit.
        max_attempts: макс спроб за window секунд
        block_time: час блокування в секундах (15 хв)
        """
        key = self._get_key(request)
        now = time.time()
        
        # Перевірити чи заблоковано
        if key in self.blocked:
            if now < self.blocked[key]:
                remaining = int(self.blocked[key] - now)
                raise HTTPException(
                    status_code=429,
                    detail=f"Забагато спроб. Спробуйте через {remaining} секунд.",
                    headers={"Retry-After": str(remaining)},
                )
            del self.blocked[key]
        
        # Очистити старі спроби
        self.attempts[key] = [t for t in self.attempts[key] if now - t < window]
        
        # Перевірити кількість
        if len(self.attempts[key]) >= max_attempts:
            self.blocked[key] = now + block_time
            raise HTTPException(
                status_code=429,
                detail=f"Забагато спроб входу. Акаунт заблоковано на {block_time // 60} хвилин.",
            )
        
        self.attempts[key].append(now)

auth_limiter = AuthRateLimiter()

# Використання
@router.post("/login")
def login(request: Request, ...):
    auth_limiter.check(request, max_attempts=5, window=300)  # 5 спроб за 5 хвилин
    # ... решта логіки
```

---

## 11. Security Best Practices

```
ПАРОЛІ:
  ✅ Хешуй bcrypt/argon2 (passlib)
  ✅ Мінімум 8 символів
  ✅ Перевіряй leaked passwords (haveibeenpwned API)
  ❌ Ніколи не зберігай plain text
  ❌ Не використовуй MD5/SHA для паролів

JWT ТОКЕНИ:
  ✅ Short-lived access tokens (15-30 хв)
  ✅ Long-lived refresh tokens в БД (можна відкликати)
  ✅ Мінімум інфо в payload (id, role, email)
  ✅ Secret key мінімум 256 біт (32+ символів)
  ✅ Refresh token rotation
  ❌ Не клади секрети в JWT payload
  ❌ Не зберігай access token в localStorage (XSS)

OAUTH2:
  ✅ State parameter (CSRF protection)
  ✅ Обмін code → token тільки на бекенді
  ✅ Перевіряй email верифікацію від провайдера
  ❌ Client secret ніколи на фронтенді

EMAIL:
  ✅ Не розкривай чи існує email (forgot-password)
  ✅ Короткий час життя токенів (1 год для reset, 24 год для verify)
  ✅ Rate limit на відправку (макс 3 листи за годину)

ЗАГАЛЬНЕ:
  ✅ HTTPS обов'язково
  ✅ Rate limiting на auth endpoints (5 спроб за 5 хвилин)
  ✅ Логування спроб входу (failed + successful)
  ✅ CORS — тільки дозволені домени
  ✅ httpOnly cookies для refresh token (якщо веб-додаток)
  ✅ Регулярно чисти expired tokens з БД
```

---

## 12. Повна архітектура

```
┌─────────────────────────────────────────────────────────┐
│                      Frontend                            │
│  (React / Vue / Mobile)                                  │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─── Email/Password ────┐  ┌── OAuth2 ──────────────┐ │
│  │ POST /auth/register   │  │ GET /auth/google       │ │
│  │ POST /auth/login      │  │ GET /auth/github       │ │
│  │ POST /auth/refresh    │  │ GET /auth/facebook     │ │
│  │ POST /auth/logout     │  │ → Callback → JWT       │ │
│  └───────────────────────┘  └────────────────────────┘ │
│                                                          │
│  ┌─── Email Verify ──────┐  ┌── 2FA ────────────────┐ │
│  │ POST /auth/verify     │  │ POST /auth/2fa/enable  │ │
│  │ POST /auth/resend     │  │ POST /auth/2fa/confirm │ │
│  └───────────────────────┘  │ POST /auth/2fa/verify  │ │
│                              └────────────────────────┘ │
│  ┌─── Password Reset ───┐  ┌── RBAC ────────────────┐ │
│  │ POST /forgot-password │  │ Depends(require_admin) │ │
│  │ POST /reset-password  │  │ Depends(require_editor)│ │
│  │ POST /change-password │  │ RoleChecker([...])     │ │
│  └───────────────────────┘  └────────────────────────┘ │
│                                                          │
│  ┌─── Middleware ────────────────────────────────────┐  │
│  │ CORS │ Rate Limit │ Logging │ Auth Dependency     │  │
│  └───────────────────────────────────────────────────┘  │
│                                                          │
│  ┌─── Database ──────────────────────────────────────┐  │
│  │ users │ refresh_tokens │ Alembic migrations       │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 📚 Ресурси

- [FastAPI Security Tutorial](https://fastapi.tiangolo.com/tutorial/security/)
- [OAuth 2.0 Simplified (Aaron Parecki)](https://www.oauth.com/)
- [JWT.io — debugger і документація](https://jwt.io/)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [PyJWT Documentation](https://pyjwt.readthedocs.io/)
- [passlib Documentation](https://passlib.readthedocs.io/)
- [pyotp (TOTP) Documentation](https://pyauth.github.io/pyotp/)
- [Google OAuth2 Documentation](https://developers.google.com/identity/protocols/oauth2)
- [GitHub OAuth Documentation](https://docs.github.com/en/developers/apps/building-oauth-apps)

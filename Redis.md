# Redis + FastAPI — Повний Гайд

> Від основ Redis до OAuth state, сесій, кешування та черг у FastAPI

---

## Зміст

1. [Що таке Redis і навіщо він потрібен](#1-що-таке-redis-і-навіщо-він-потрібен)
2. [Як Redis зберігає дані — структури даних](#2-як-redis-зберігає-дані--структури-даних)
3. [TTL — Time To Live](#3-ttl--time-to-live)
4. [Встановлення та запуск Redis](#4-встановлення-та-запуск-redis)
5. [Підключення до Redis у FastAPI](#5-підключення-до-redis-у-fastapi)
6. [Dependency Injection для Redis](#6-dependency-injection-для-redis)
7. [Патерн Repository для Redis](#7-патерн-repository-для-redis)
8. [OAuth State — детально](#8-oauth-state--детально)
9. [Сесії через Redis](#9-сесії-через-redis)
10. [Кешування відповідей](#10-кешування-відповідей)
11. [Rate Limiting](#11-rate-limiting)
12. [Pub/Sub — черги повідомлень](#12-pubsub--черги-повідомлень)
13. [Redis у Docker Compose](#13-redis-у-docker-compose)
14. [Конфігурація через Settings](#14-конфігурація-через-settings)
15. [Типові помилки та як їх уникнути](#15-типові-помилки-та-як-їх-уникнути)

---

## 1. Що таке Redis і навіщо він потрібен

**Redis** (Remote Dictionary Server) — це in-memory сховище даних типу ключ-значення.

**In-memory** означає, що всі дані зберігаються в оперативній пам'яті (RAM), а не на диску. Тому операції читання/запису відбуваються за мікросекунди.

### Порівняння з PostgreSQL

| Характеристика | PostgreSQL | Redis |
|---|---|---|
| Зберігання | Диск (HDD/SSD) | RAM |
| Швидкість читання | ~1-5 мс | ~0.1 мс |
| Тип даних | Таблиці з рядками | Key-Value структури |
| Persistence | Завжди | Опціонально |
| Призначення | Основні дані | Кеш, сесії, стани |

### Де Redis незамінний

```
┌─────────────────────────────────────────────────────────┐
│                    Де юзати Redis                        │
├─────────────────────────────────────────────────────────┤
│  OAuth State       │ Тимчасове збереження state токена  │
│  Сесії             │ Зберігання JWT refresh токенів      │
│  Кеш               │ Відповіді API, дані користувача     │
│  Rate Limiting     │ Лічильники запитів per user/IP      │
│  Pub/Sub           │ Сповіщення між мікросервісами       │
│  Task Queue        │ Фонові задачі (через Celery)        │
│  Leaderboards      │ Sorted Sets для рейтингів           │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Як Redis зберігає дані — структури даних

Redis — це не просто "словник". Він підтримує 5 основних структур:

### 2.1 String (рядок)

Найпростіша структура. Ключ → значення.

```
SET user:1:name "Daniil"
GET user:1:name       → "Daniil"

SET counter 0
INCR counter          → 1
INCRBY counter 5      → 6
```

**В Python:**
```python
await redis.set("user:1:name", "Daniil")
name = await redis.get("user:1:name")  # b"Daniil"
```

Використовується для: кешу, лічильників, OAuth state, сесій.

---

### 2.2 Hash (словник всередині ключа)

Схожий на Python dict. Один ключ містить кілька полів.

```
HSET user:1 name "Daniil" age "18" city "Bila Tserkva"
HGET user:1 name          → "Daniil"
HGETALL user:1            → {name: Daniil, age: 18, city: ...}
```

**В Python:**
```python
await redis.hset("user:1", mapping={"name": "Daniil", "age": "18"})
user = await redis.hgetall("user:1")  # {b"name": b"Daniil", ...}
```

Використовується для: зберігання об'єктів без серіалізації в JSON.

---

### 2.3 List (двобічна черга)

```
LPUSH queue "task1"   # додати зліва
RPUSH queue "task2"   # додати справа
LPOP queue            → "task1"
LRANGE queue 0 -1     → всі елементи
```

Використовується для: черги задач, логів.

---

### 2.4 Set (множина унікальних значень)

```
SADD active_users "user:1" "user:2" "user:3"
SISMEMBER active_users "user:1"   → 1 (true)
SMEMBERS active_users             → {user:1, user:2, user:3}
```

Використовується для: відстеження онлайн юзерів, тегів.

---

### 2.5 Sorted Set (множина з рейтингом)

```
ZADD leaderboard 1500 "user:1"
ZADD leaderboard 2200 "user:2"
ZRANGE leaderboard 0 -1 WITHSCORES  → user:1=1500, user:2=2200
ZREVRANK leaderboard "user:2"       → 0 (перший у рейтингу)
```

Використовується для: рейтингів, сортованих лідербордів.

---

## 3. TTL — Time To Live

**TTL** — це час життя ключа. Після закінчення Redis автоматично видаляє ключ.

```
SET oauth_state:abc123 "data" EX 300    # видалити через 300 секунд
TTL oauth_state:abc123                   → 287 (залишилось)
PERSIST oauth_state:abc123               # прибрати TTL
PTTL oauth_state:abc123                  → залишилось в мілісекундах
```

**В Python:**
```python
# Встановити з TTL
await redis.set("key", "value", ex=300)        # 300 секунд
await redis.set("key", "value", px=300000)     # 300000 мілісекунд

# Отримати залишок TTL
ttl = await redis.ttl("key")   # int, -1 якщо немає TTL, -2 якщо ключ не існує
```

> **Чому TTL критично важливий для OAuth state:** State токен повинен бути одноразовим і короткоживучим. Якщо юзер не завершив OAuth flow за 5-10 хвилин — токен має самостійно видалитись.

---

## 4. Встановлення та запуск Redis

### Локально (Ubuntu/WSL)

```bash
sudo apt update
sudo apt install redis-server

# Запустити
sudo systemctl start redis
sudo systemctl enable redis   # автозапуск

# Перевірити
redis-cli ping   → PONG
```

### Через Docker

```bash
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7-alpine
```

### Redis CLI — базові команди

```bash
redis-cli

# Всередині CLI:
SET foo "bar"
GET foo
KEYS *              # всі ключі (не юзати в prod!)
KEYS oauth_state:*  # ключі з префіксом
TTL foo
DEL foo
FLUSHDB             # очистити поточну БД (НЕБЕЗПЕЧНО)
```

---

## 5. Підключення до Redis у FastAPI

### Встановити залежності

```bash
# з uv (твій стек)
uv add redis[hiredis]

# або pip
pip install redis[hiredis]
```

> `hiredis` — C-парсер для Redis протоколу. Дає ~2-3x приріст швидкості. Завжди встанови його разом з `redis`.

### Два режими роботи: sync vs async

Оскільки FastAPI async-first, використовуємо **async клієнт**:

```python
import redis.asyncio as aioredis

# Синхронний (НЕ рекомендується у FastAPI)
import redis
r = redis.Redis(host="localhost", port=6379)

# Асинхронний (ПРАВИЛЬНО)
import redis.asyncio as aioredis
r = aioredis.Redis(host="localhost", port=6379)
```

### Connection Pool

Connection Pool — це пул відкритих з'єднань з Redis. Замість того, щоб відкривати нове з'єднання на кожен запит, беремо вже готове з пулу.

```python
# app/core/redis.py
import redis.asyncio as aioredis
from redis.asyncio import ConnectionPool

pool: ConnectionPool | None = None


def create_pool(redis_url: str) -> ConnectionPool:
    return aioredis.ConnectionPool.from_url(
        redis_url,
        max_connections=20,       # максимум 20 одночасних з'єднань
        decode_responses=True,    # автоматично декодувати bytes → str
        socket_timeout=5,         # timeout на операцію (сек)
        socket_connect_timeout=5, # timeout на підключення (сек)
        retry_on_timeout=True,    # повторити при timeout
    )


def get_redis_client() -> aioredis.Redis:
    return aioredis.Redis(connection_pool=pool)
```

> **`decode_responses=True`** — дуже важливий параметр. Без нього Redis повертає `bytes` (`b"value"`), а з ним — звичайні `str` (`"value"`). Завжди встановлюй `True` якщо не працюєш з бінарними даними.

### Lifespan — ініціалізація при старті FastAPI

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.core.redis import create_pool, get_redis_client
import app.core.redis as redis_module


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ===== STARTUP =====
    redis_module.pool = create_pool("redis://localhost:6379/0")
    
    # Перевірити з'єднання
    redis = get_redis_client()
    await redis.ping()
    print("✅ Redis connected")
    
    yield
    
    # ===== SHUTDOWN =====
    await redis_module.pool.disconnect()
    print("🔴 Redis disconnected")


app = FastAPI(lifespan=lifespan)
```

> **Навіщо `lifespan`?** Це рекомендований у FastAPI патерн для startup/shutdown логіки (замість застарілих `@app.on_event`). З'єднання з Redis відкривається один раз при старті та закривається при зупинці.

---

## 6. Dependency Injection для Redis

FastAPI має систему `Depends()` — вона дозволяє "вставляти" залежності в роути автоматично.

```python
# app/dependencies/redis.py
from typing import Annotated
import redis.asyncio as aioredis
from fastapi import Depends
from app.core.redis import get_redis_client


async def get_redis() -> aioredis.Redis:
    return get_redis_client()


RedisDep = Annotated[aioredis.Redis, Depends(get_redis)]
```

**Використання в роутах:**

```python
# app/routers/example.py
from fastapi import APIRouter
from app.dependencies.redis import RedisDep

router = APIRouter()


@router.get("/test-redis")
async def test_redis(redis: RedisDep):
    await redis.set("hello", "world", ex=60)
    value = await redis.get("hello")
    return {"value": value}
```

---

## 7. Патерн Repository для Redis

Замість того щоб писати `redis.set(...)` прямо в роутах, виносимо логіку в окремий клас.

```python
# app/repositories/redis_repository.py
import json
from typing import Any
import redis.asyncio as aioredis


class RedisRepository:
    def __init__(self, redis: aioredis.Redis):
        self.redis = redis

    # ─── String операції ───────────────────────────────────────

    async def set(self, key: str, value: str, ttl: int | None = None) -> None:
        await self.redis.set(key, value, ex=ttl)

    async def get(self, key: str) -> str | None:
        return await self.redis.get(key)

    async def delete(self, key: str) -> bool:
        return bool(await self.redis.delete(key))

    async def exists(self, key: str) -> bool:
        return bool(await self.redis.exists(key))

    async def ttl(self, key: str) -> int:
        return await self.redis.ttl(key)

    # ─── JSON операції ─────────────────────────────────────────

    async def set_json(self, key: str, value: Any, ttl: int | None = None) -> None:
        serialized = json.dumps(value, default=str)
        await self.redis.set(key, serialized, ex=ttl)

    async def get_json(self, key: str) -> Any | None:
        raw = await self.redis.get(key)
        if raw is None:
            return None
        return json.loads(raw)

    # ─── Атомарні операції ─────────────────────────────────────

    async def increment(self, key: str, amount: int = 1) -> int:
        return await self.redis.incrby(key, amount)

    async def set_if_not_exists(self, key: str, value: str, ttl: int) -> bool:
        """SET key value NX EX ttl — атомарна операція.
        Повертає True якщо ключ був створений, False якщо вже існував."""
        result = await self.redis.set(key, value, nx=True, ex=ttl)
        return result is not None

    # ─── Scan (безпечний аналог KEYS) ──────────────────────────

    async def scan_keys(self, pattern: str) -> list[str]:
        """Безпечна ітерація по ключах замість KEYS *"""
        keys = []
        async for key in self.redis.scan_iter(match=pattern):
            keys.append(key)
        return keys
```

---

## 8. OAuth State — детально

### Навіщо потрібен OAuth state?

OAuth 2.0 flow виглядає так:

```
Юзер → [1] Твій бекенд → [2] Google/GitHub (authorize URL)
                                    ↓
Юзер логіниться в Google
                                    ↓
Google → [3] Callback на твій бекенд (?code=xxx&state=yyy)
                                    ↓
Твій бекенд → [4] Обмін code на access_token у Google
```

**Проблема:** На кроці [3] ти отримуєш callback від Google. Але звідки знати, що цей callback справді для твого флоу, а не зловмисна підробка?

**Рішення — CSRF захист через state:**
1. При кроці [1] генеруємо випадковий `state` токен
2. Зберігаємо його в Redis з TTL 10 хвилин
3. Передаємо `state` в URL до Google
4. При кроці [3] Google повертає той самий `state`
5. Ми перевіряємо: чи існує такий `state` в Redis?
6. Якщо так — валідний флоу. Видаляємо `state` (одноразовий!)
7. Якщо ні — атака або прострочений токен

```
┌────────────────────────────────────────────────────────────────┐
│                     OAuth State Flow                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. GET /auth/google/login                                     │
│     → генерувати state = secrets.token_urlsafe(32)            │
│     → Redis.SET oauth_state:{state} "1" EX 600                │
│     → redirect to google.com/oauth?state={state}&...          │
│                                                                │
│  2. Юзер логіниться в Google                                   │
│                                                                │
│  3. GET /auth/google/callback?code=xxx&state=yyy               │
│     → Redis.GET oauth_state:{state}  ← чи існує?             │
│     → якщо немає → 400 Invalid state                          │
│     → Redis.DEL oauth_state:{state}  ← видалити одразу!      │
│     → обміняти code на токени                                  │
│     → повернути JWT                                            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Реалізація OAuth State Repository

```python
# app/repositories/oauth_state_repository.py
import secrets
import redis.asyncio as aioredis


class OAuthStateRepository:
    KEY_PREFIX = "oauth_state"
    DEFAULT_TTL = 600  # 10 хвилин

    def __init__(self, redis: aioredis.Redis):
        self.redis = redis

    def _key(self, state: str) -> str:
        return f"{self.KEY_PREFIX}:{state}"

    async def create(self, ttl: int = DEFAULT_TTL) -> str:
        """Генерує новий state токен і зберігає в Redis."""
        state = secrets.token_urlsafe(32)  # 32 байти = 43 символи base64url
        await self.redis.set(self._key(state), "1", ex=ttl)
        return state

    async def validate_and_consume(self, state: str) -> bool:
        """Перевіряє state і одразу видаляє (одноразовий).
        Повертає True якщо валідний, False якщо ні."""
        key = self._key(state)
        
        # GETDEL — атомарна операція: отримати і видалити одночасно
        # Доступна з Redis 6.2+
        value = await self.redis.getdel(key)
        return value is not None

    async def exists(self, state: str) -> bool:
        """Перевірити без видалення."""
        return bool(await self.redis.exists(self._key(state)))
```

> **Чому `GETDEL` а не `GET` + `DEL`?** Race condition! Якщо між `GET` і `DEL` прийде другий запит з тим самим `state` — він теж пройде перевірку. `GETDEL` атомарна — виконується як одна операція.

### OAuth Router

```python
# app/routers/auth/google.py
import httpx
from fastapi import APIRouter, Depends, HTTPException, Query
from fastapi.responses import RedirectResponse
from app.dependencies.redis import RedisDep
from app.repositories.oauth_state_repository import OAuthStateRepository

router = APIRouter(prefix="/auth/google", tags=["auth"])

GOOGLE_CLIENT_ID = "your-client-id"
GOOGLE_CLIENT_SECRET = "your-client-secret"
GOOGLE_REDIRECT_URI = "http://localhost:8000/auth/google/callback"
GOOGLE_AUTH_URL = "https://accounts.google.com/o/oauth2/v2/auth"
GOOGLE_TOKEN_URL = "https://oauth2.googleapis.com/token"
GOOGLE_USERINFO_URL = "https://www.googleapis.com/oauth2/v2/userinfo"


@router.get("/login")
async def google_login(redis: RedisDep):
    """Крок 1: Редірект на Google OAuth."""
    repo = OAuthStateRepository(redis)
    state = await repo.create(ttl=600)  # 10 хвилин

    params = {
        "client_id": GOOGLE_CLIENT_ID,
        "redirect_uri": GOOGLE_REDIRECT_URI,
        "response_type": "code",
        "scope": "openid email profile",
        "state": state,
        "access_type": "offline",  # для отримання refresh_token
        "prompt": "consent",
    }

    # Будуємо URL вручну
    query_string = "&".join(f"{k}={v}" for k, v in params.items())
    auth_url = f"{GOOGLE_AUTH_URL}?{query_string}"

    return RedirectResponse(url=auth_url)


@router.get("/callback")
async def google_callback(
    redis: RedisDep,
    code: str = Query(...),
    state: str = Query(...),
    error: str | None = Query(None),
):
    """Крок 3: Обробка callback від Google."""
    
    # 0. Перевірити чи не було помилки з боку Google
    if error:
        raise HTTPException(400, f"OAuth error: {error}")

    # 1. Валідувати і одразу видалити state
    repo = OAuthStateRepository(redis)
    is_valid = await repo.validate_and_consume(state)
    
    if not is_valid:
        raise HTTPException(
            status_code=400,
            detail="Invalid or expired OAuth state. Please try again.",
        )

    # 2. Обміняти authorization code на access_token
    async with httpx.AsyncClient() as client:
        token_response = await client.post(
            GOOGLE_TOKEN_URL,
            data={
                "client_id": GOOGLE_CLIENT_ID,
                "client_secret": GOOGLE_CLIENT_SECRET,
                "code": code,
                "grant_type": "authorization_code",
                "redirect_uri": GOOGLE_REDIRECT_URI,
            },
        )
        token_response.raise_for_status()
        tokens = token_response.json()

    access_token = tokens["access_token"]

    # 3. Отримати дані юзера від Google
    async with httpx.AsyncClient() as client:
        user_response = await client.get(
            GOOGLE_USERINFO_URL,
            headers={"Authorization": f"Bearer {access_token}"},
        )
        user_response.raise_for_status()
        google_user = user_response.json()

    # 4. Тут: знайти або створити юзера в БД, видати JWT
    # user = await user_service.find_or_create(google_user)
    # jwt_token = create_jwt(user.id)

    return {
        "message": "OAuth successful",
        "user": {
            "email": google_user["email"],
            "name": google_user["name"],
        },
        # "access_token": jwt_token
    }
```

### Додатковий рівень: зберігати metadata у state

Іноді потрібно прив'язати до state додаткові дані (наприклад, звідки прийшов юзер):

```python
# Розширена версія OAuthStateRepository

async def create_with_metadata(
    self,
    metadata: dict,
    ttl: int = 600,
) -> str:
    """Зберегти state разом з метаданими."""
    state = secrets.token_urlsafe(32)
    value = json.dumps(metadata)
    await self.redis.set(self._key(state), value, ex=ttl)
    return state


async def validate_and_consume_with_metadata(
    self,
    state: str,
) -> dict | None:
    """Повертає metadata якщо валідний, None якщо ні."""
    key = self._key(state)
    raw = await self.redis.getdel(key)
    if raw is None:
        return None
    return json.loads(raw)


# Використання:
state = await repo.create_with_metadata({
    "redirect_after": "/dashboard",
    "user_agent": request.headers.get("user-agent"),
})

# У callback:
metadata = await repo.validate_and_consume_with_metadata(state)
if metadata is None:
    raise HTTPException(400, "Invalid state")
# metadata["redirect_after"] → "/dashboard"
```

---

## 9. Сесії через Redis

Замість stateless JWT access_token, можна зберігати сесії в Redis:

```python
# app/repositories/session_repository.py
import secrets
import json
from datetime import datetime
import redis.asyncio as aioredis


class SessionRepository:
    KEY_PREFIX = "session"
    SESSION_TTL = 60 * 60 * 24 * 7  # 7 днів

    def __init__(self, redis: aioredis.Redis):
        self.redis = redis

    def _key(self, session_id: str) -> str:
        return f"{self.KEY_PREFIX}:{session_id}"

    async def create(self, user_id: int, extra: dict | None = None) -> str:
        session_id = secrets.token_urlsafe(32)
        data = {
            "user_id": user_id,
            "created_at": datetime.utcnow().isoformat(),
            **(extra or {}),
        }
        await self.redis.set(
            self._key(session_id),
            json.dumps(data),
            ex=self.SESSION_TTL,
        )
        return session_id

    async def get(self, session_id: str) -> dict | None:
        raw = await self.redis.get(self._key(session_id))
        if raw is None:
            return None
        return json.loads(raw)

    async def refresh(self, session_id: str) -> bool:
        """Оновити TTL сесії."""
        return bool(await self.redis.expire(self._key(session_id), self.SESSION_TTL))

    async def delete(self, session_id: str) -> bool:
        return bool(await self.redis.delete(self._key(session_id)))

    async def delete_all_user_sessions(self, user_id: int) -> int:
        """Видалити всі сесії юзера (logout з усіх пристроїв)."""
        pattern = f"{self.KEY_PREFIX}:*"
        deleted = 0
        async for key in self.redis.scan_iter(match=pattern):
            raw = await self.redis.get(key)
            if raw:
                data = json.loads(raw)
                if data.get("user_id") == user_id:
                    await self.redis.delete(key)
                    deleted += 1
        return deleted
```

---

## 10. Кешування відповідей

```python
# app/utils/cache.py
import json
import functools
from typing import Callable, Any
import redis.asyncio as aioredis
from app.core.redis import get_redis_client


def cache(ttl: int = 300, key_prefix: str = "cache"):
    """Декоратор для кешування результату async функції."""
    
    def decorator(func: Callable):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            redis = get_redis_client()
            
            # Будуємо ключ з аргументів функції
            cache_key = f"{key_prefix}:{func.__name__}:{args}:{kwargs}"
            
            # Спробувати отримати з кешу
            cached = await redis.get(cache_key)
            if cached is not None:
                return json.loads(cached)
            
            # Виконати функцію
            result = await func(*args, **kwargs)
            
            # Зберегти результат в кеш
            await redis.set(cache_key, json.dumps(result, default=str), ex=ttl)
            
            return result
        
        return wrapper
    return decorator


# Інвалідація кешу
async def invalidate_pattern(pattern: str) -> int:
    redis = get_redis_client()
    deleted = 0
    async for key in redis.scan_iter(match=pattern):
        await redis.delete(key)
        deleted += 1
    return deleted
```

**Використання:**

```python
from app.utils.cache import cache, invalidate_pattern


@cache(ttl=300, key_prefix="user")
async def get_user_profile(user_id: int) -> dict:
    # Важкий запит до БД
    user = await db.get(User, user_id)
    return user.__dict__


# При оновленні профілю — інвалідувати кеш
@router.put("/profile")
async def update_profile(user_id: int, data: ProfileUpdate):
    await user_service.update(user_id, data)
    await invalidate_pattern(f"user:get_user_profile:({user_id},):*")
    return {"message": "Updated"}
```

---

## 11. Rate Limiting

```python
# app/middleware/rate_limit.py
import time
from fastapi import Request, HTTPException
import redis.asyncio as aioredis
from app.core.redis import get_redis_client


class RateLimiter:
    def __init__(
        self,
        max_requests: int = 100,
        window_seconds: int = 60,
    ):
        self.max_requests = max_requests
        self.window = window_seconds

    async def check(self, identifier: str) -> dict:
        """
        Sliding window rate limiter.
        identifier — IP або user_id
        """
        redis = get_redis_client()
        key = f"rate_limit:{identifier}"
        now = int(time.time())
        window_start = now - self.window

        # Видалити старі записи за межами вікна
        await redis.zremrangebyscore(key, 0, window_start)
        
        # Порахувати поточну кількість запитів
        current_count = await redis.zcard(key)
        
        if current_count >= self.max_requests:
            oldest = await redis.zrange(key, 0, 0, withscores=True)
            retry_after = int(oldest[0][1]) + self.window - now if oldest else self.window
            raise HTTPException(
                status_code=429,
                detail="Too many requests",
                headers={"Retry-After": str(retry_after)},
            )

        # Додати поточний запит
        await redis.zadd(key, {f"{now}:{id(object())}": now})
        await redis.expire(key, self.window)

        return {
            "limit": self.max_requests,
            "remaining": self.max_requests - current_count - 1,
            "reset": now + self.window,
        }


# Dependency
async def rate_limit_dependency(request: Request):
    limiter = RateLimiter(max_requests=60, window_seconds=60)
    client_ip = request.client.host
    await limiter.check(client_ip)


# Використання в роуті
@router.post("/login", dependencies=[Depends(rate_limit_dependency)])
async def login(data: LoginRequest):
    ...
```

---

## 12. Pub/Sub — черги повідомлень

Pub/Sub дозволяє компонентам спілкуватись через канали:

```python
# Publisher — відправляє повідомлення
async def publish_event(redis: aioredis.Redis, channel: str, data: dict):
    message = json.dumps(data)
    await redis.publish(channel, message)

# Subscriber — слухає канал
async def listen_events(redis: aioredis.Redis, channel: str):
    pubsub = redis.pubsub()
    await pubsub.subscribe(channel)
    
    async for message in pubsub.listen():
        if message["type"] == "message":
            data = json.loads(message["data"])
            print(f"Received: {data}")
            # обробити подію


# Приклад: сповіщення про нову активність
await publish_event(redis, "user:notifications", {
    "user_id": 42,
    "type": "achievement",
    "data": {"name": "First Login"}
})
```

---

## 13. Redis у Docker Compose

```yaml
# docker-compose.yml
version: "3.9"

services:
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - REDIS_URL=redis://redis:6379/0
      - DATABASE_URL=postgresql+asyncpg://...
    depends_on:
      redis:
        condition: service_healthy
      db:
        condition: service_healthy

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: health_patch
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - pg_data:/var/lib/postgresql/data

volumes:
  redis_data:
  pg_data:
```

### Пояснення команди Redis

```
redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
```

| Параметр | Значення |
|---|---|
| `--appendonly yes` | Записувати всі операції в AOF файл (persistence) |
| `--maxmemory 256mb` | Максимум 256 МБ RAM |
| `--maxmemory-policy allkeys-lru` | При переповненні — видаляти найдавніші ключі |

---

## 14. Конфігурація через Settings

```python
# app/core/config.py
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"
    REDIS_MAX_CONNECTIONS: int = 20
    REDIS_DECODE_RESPONSES: bool = True

    # OAuth
    GOOGLE_CLIENT_ID: str
    GOOGLE_CLIENT_SECRET: str
    GOOGLE_REDIRECT_URI: str = "http://localhost:8000/auth/google/callback"

    GITHUB_CLIENT_ID: str
    GITHUB_CLIENT_SECRET: str

    OAUTH_STATE_TTL: int = 600  # 10 хвилин
    SESSION_TTL: int = 604800   # 7 днів

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"


settings = Settings()
```

```python
# app/core/redis.py — оновлена версія
import redis.asyncio as aioredis
from app.core.config import settings

pool: aioredis.ConnectionPool | None = None


def create_pool() -> aioredis.ConnectionPool:
    return aioredis.ConnectionPool.from_url(
        settings.REDIS_URL,
        max_connections=settings.REDIS_MAX_CONNECTIONS,
        decode_responses=settings.REDIS_DECODE_RESPONSES,
        socket_timeout=5,
        retry_on_timeout=True,
    )


def get_redis_client() -> aioredis.Redis:
    return aioredis.Redis(connection_pool=pool)
```

```bash
# .env
REDIS_URL=redis://localhost:6379/0
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxxxx
GITHUB_CLIENT_ID=Iv1.xxxxx
GITHUB_CLIENT_SECRET=xxxxx
```

---

## 15. Типові помилки та як їх уникнути

### ❌ Не використовувати синхронний клієнт в async коді

```python
# НЕПРАВИЛЬНО
import redis
r = redis.Redis()  # блокує event loop!

# ПРАВИЛЬНО
import redis.asyncio as aioredis
r = aioredis.Redis()
```

### ❌ Не відкривати нове з'єднання на кожен запит

```python
# НЕПРАВИЛЬНО
@router.get("/")
async def handler():
    r = aioredis.Redis(host="localhost")  # новий коннект кожен раз
    ...

# ПРАВИЛЬНО — використовувати ConnectionPool через Depends()
@router.get("/")
async def handler(redis: RedisDep):
    ...
```

### ❌ Не юзати `KEYS *` в production

```python
# НЕПРАВИЛЬНО — блокує Redis на секунди якщо мільйони ключів!
keys = await redis.keys("oauth_state:*")

# ПРАВИЛЬНО — ітеративний scan
async for key in redis.scan_iter(match="oauth_state:*"):
    ...
```

### ❌ Забути видалити OAuth state після перевірки

```python
# НЕПРАВИЛЬНО — state можна використати повторно (replay attack)
value = await redis.get(f"oauth_state:{state}")
if value:
    process_oauth()

# ПРАВИЛЬНО — видалити одразу
value = await redis.getdel(f"oauth_state:{state}")
if value:
    process_oauth()
```

### ❌ Зберігати чутливі дані без TTL

```python
# НЕПРАВИЛЬНО — буде жити вічно
await redis.set("password_reset_token:xxx", user_id)

# ПРАВИЛЬНО
await redis.set("password_reset_token:xxx", user_id, ex=900)  # 15 хвилин
```

### ❌ Не перевіряти `decode_responses`

```python
# Якщо decode_responses=False — отримуєш bytes
value = await redis.get("key")  # b"hello"
if value == "hello":  # False! bytes != str
    ...

# Або явно декодуй
if value and value.decode() == "hello":
    ...

# Або увімкни decode_responses=True при ініціалізації пулу
```

---

## Структура файлів у проєкті

```
app/
├── core/
│   ├── config.py          # Settings з REDIS_URL
│   └── redis.py           # ConnectionPool, get_redis_client
├── dependencies/
│   └── redis.py           # RedisDep = Annotated[Redis, Depends(...)]
├── repositories/
│   ├── redis_repository.py       # Базовий репозиторій
│   ├── oauth_state_repository.py # OAuth state логіка
│   └── session_repository.py     # Сесії
├── routers/
│   └── auth/
│       └── google.py      # /auth/google/login, /callback
├── middleware/
│   └── rate_limit.py      # Rate limiting
└── main.py                # lifespan з Redis init
```

---

## Швидка шпаргалка Redis команд

```python
# Базові операції
await redis.set("key", "value")                    # встановити
await redis.set("key", "value", ex=300)            # з TTL
await redis.set("key", "value", nx=True, ex=300)   # тільки якщо не існує
await redis.get("key")                             # отримати
await redis.getdel("key")                          # отримати і видалити (Redis 6.2+)
await redis.delete("key")                          # видалити
await redis.exists("key")                          # перевірити існування (0 або 1)
await redis.ttl("key")                             # залишок TTL в секундах
await redis.expire("key", 300)                     # оновити TTL

# Числа
await redis.incr("counter")                        # +1
await redis.incrby("counter", 5)                   # +5
await redis.decr("counter")                        # -1

# Hash
await redis.hset("hash", "field", "value")
await redis.hget("hash", "field")
await redis.hgetall("hash")                        # всі поля
await redis.hmset("hash", {"f1": "v1", "f2": "v2"})

# Sorted Set (для rate limiting)
await redis.zadd("key", {"member": score})
await redis.zcard("key")                           # кількість елементів
await redis.zremrangebyscore("key", min, max)      # видалити за score

# Безпечний скан
async for key in redis.scan_iter(match="prefix:*"):
    ...
```

---

*Документ охоплює Redis від основ до production-ready патернів у FastAPI. Ключова концепція: Redis — це не БД, це інструмент для тимчасових станів, де швидкість та TTL важливіші за persistence.*

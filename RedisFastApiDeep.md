# Redis у FastAPI — Повний та Детальний Гайд

> Підключення, ConnectionPool, Depends, Repository патерн, OAuth state, сесії, кешування, rate limiting — все з поясненням кожного рядка

---

## Зміст

1. [Встановлення залежностей](#1-встановлення-залежностей)
2. [Як redis-py працює — sync vs async](#2-як-redis-py-працює--sync-vs-async)
3. [ConnectionPool — що це і навіщо](#3-connectionpool--що-це-і-навіщо)
4. [Ініціалізація через lifespan](#4-ініціалізація-через-lifespan)
5. [Settings — конфігурація через .env](#5-settings--конфігурація-через-env)
6. [Dependency Injection — RedisDep](#6-dependency-injection--redisdep)
7. [Базовий RedisRepository](#7-базовий-redisrepository)
8. [OAuth State — повна реалізація](#8-oauth-state--повна-реалізація)
9. [Session Repository](#9-session-repository)
10. [Кешування — декоратор і вручну](#10-кешування--декоратор-і-вручну)
11. [Rate Limiting](#11-rate-limiting)
12. [Distributed Lock](#12-distributed-lock)
13. [Blacklist для JWT токенів](#13-blacklist-для-jwt-токенів)
14. [Pipeline у FastAPI](#14-pipeline-у-fastapi)
15. [Pub/Sub у FastAPI](#15-pubsub-у-fastapi)
16. [Тестування з Redis](#16-тестування-з-redis)
17. [Docker Compose — повна конфігурація](#17-docker-compose--повна-конфігурація)
18. [Структура проєкту](#18-структура-проєкту)
19. [Типові помилки та антипатерни](#19-типові-помилки-та-антипатерни)

---

## 1. Встановлення залежностей

```bash
# uv (твій стек)
uv add "redis[hiredis]"

# або pip
pip install "redis[hiredis]"
```

### Що таке hiredis і навіщо він

`redis` — це чистий Python клієнт. Він вміє все, але парсинг RESP протоколу написаний на Python.

`hiredis` — це C-розширення (написане командою Redis), яке замінює Python парсер на швидкий C-парсер. Просто встанови `redis[hiredis]` — клієнт автоматично використає hiredis якщо він є.

```
Без hiredis:  ~150,000 ops/sec
З hiredis:    ~400,000 ops/sec  (2-3x швидше)
```

Ніякого додаткового коду не потрібно — просто встанови пакет.

---

## 2. Як redis-py працює — sync vs async

Бібліотека `redis-py` надає **два окремих клієнти**:

```python
# ─── СИНХРОННИЙ КЛІЄНТ ────────────────────────────────────────
import redis

r = redis.Redis(host="localhost", port=6379)
r.set("key", "value")   # блокує потік поки не отримає відповідь

# ─── АСИНХРОННИЙ КЛІЄНТ ───────────────────────────────────────
import redis.asyncio as aioredis

r = aioredis.Redis(host="localhost", port=6379)
await r.set("key", "value")   # не блокує event loop
```

**Чому у FastAPI ОБОВ'ЯЗКОВО async:**

FastAPI побудований на Starlette і використовує asyncio event loop. Коли роут оголошений як `async def`, він виконується в event loop. Якщо всередині роуту зробити **блокуючий** I/O (як синхронний `redis.Redis`), весь event loop замерзає на час операції — жоден інший запит не обробляється.

```
НЕПРАВИЛЬНО — синхронний Redis в async роуті:
┌─────────────────────────────────────────────────────┐
│  Event Loop                                         │
│  Request 1: [████████ чекає Redis ████████]        │
│  Request 2:          [ЗАБЛОКОВАНИЙ!!!]              │
│  Request 3:                    [ЗАБЛОКОВАНИЙ!!!]    │
└─────────────────────────────────────────────────────┘

ПРАВИЛЬНО — async Redis:
┌─────────────────────────────────────────────────────┐
│  Event Loop                                         │
│  Request 1: [──── await Redis ────]                 │
│  Request 2:      [──── await Redis ────]            │
│  Request 3:           [──── await Redis ────]       │
│  (всі виконуються паралельно)                       │
└─────────────────────────────────────────────────────┘
```

---

## 3. ConnectionPool — що це і навіщо

### Проблема без пулу

```python
# ❌ АНТИПАТЕРН — нове з'єднання на кожен запит
@router.get("/user/{id}")
async def get_user(id: int):
    r = aioredis.Redis(host="localhost", port=6379)
    # ↑ кожен запит: TCP handshake → auth → команда → close
    # При 1000 запитів/сек = 1000 нових TCP з'єднань/сек
    data = await r.get(f"user:{id}")
    await r.aclose()   # треба закривати!
    return data
```

Встановлення TCP з'єднання займає ~1-3мс. При 1000 req/s це +1-3 секунди overhead тільки на з'єднання.

### ConnectionPool — рішення

Connection Pool тримає **пул відкритих з'єднань**. Коли потрібне з'єднання — береться з пулу. Після використання — повертається назад.

```
┌─────────────────────────────────────────────────────────────┐
│                     ConnectionPool                          │
│                                                             │
│  Запит 1 ──→ бере з'єднання #1 ──→ повертає з'єднання #1  │
│  Запит 2 ──→ бере з'єднання #2 ──→ повертає з'єднання #2  │
│  Запит 3 ──→ бере з'єднання #1 ──→ (вже повернуте)        │
│                                                             │
│  Пул: [з'єднання #1, з'єднання #2, з'єднання #3, ...]     │
│        (відкриті, готові до роботи)                        │
└─────────────────────────────────────────────────────────────┘
```

```python
# app/core/redis.py
import redis.asyncio as aioredis

# Глобальний пул — один на весь додаток
_pool: aioredis.ConnectionPool | None = None


def create_pool(
    url: str,
    max_connections: int = 20,
    decode_responses: bool = True,
) -> aioredis.ConnectionPool:
    """
    Створити пул з'єднань.
    
    url формат: redis://[:password@]host[:port][/db-number]
    Приклади:
        redis://localhost:6379/0
        redis://:mypassword@localhost:6379/0
        redis://user:password@redis-host:6379/1
    
    max_connections: максимум одночасних з'єднань з Redis.
    Якщо всі зайняті — новий запит чекає поки звільниться.
    Для більшості проєктів 10-20 цілком достатньо.
    
    decode_responses=True: автоматично декодувати bytes → str.
    Без цього redis.get("key") повертає b"value" замість "value".
    Вимкни тільки якщо зберігаєш бінарні дані.
    """
    return aioredis.ConnectionPool.from_url(
        url,
        max_connections=max_connections,
        decode_responses=decode_responses,
        socket_timeout=5.0,          # timeout на виконання команди (сек)
        socket_connect_timeout=5.0,  # timeout на встановлення з'єднання
        socket_keepalive=True,       # TCP keepalive (виявляє "мертві" з'єднання)
        retry_on_timeout=True,       # повторити команду при timeout
        health_check_interval=30,    # перевіряти з'єднання кожні 30с
    )


def get_redis() -> aioredis.Redis:
    """
    Отримати Redis клієнт з пулу.
    Не відкриває нове з'єднання — бере з _pool.
    """
    if _pool is None:
        raise RuntimeError("Redis pool not initialized. Call create_pool() first.")
    return aioredis.Redis(connection_pool=_pool)


async def close_pool() -> None:
    """Закрити всі з'єднання в пулі."""
    if _pool is not None:
        await _pool.disconnect()
```

### Параметри ConnectionPool — детально

```python
aioredis.ConnectionPool.from_url(
    url,

    # ─── Розмір пулу ────────────────────────────────────────────
    max_connections=20,
    # Скільки одночасних з'єднань тримати відкритими.
    # Якщо всі 20 зайняті і прийшов 21-й запит — він чекає.
    # Чекання обмежується socket_timeout.
    # Формула: max_connections ≈ workers × avg_redis_ops_per_request
    # Для FastAPI з 4 воркерами і 3 Redis-ops/запит: 4 × 3 × 2 = 24

    # ─── Декодування ────────────────────────────────────────────
    decode_responses=True,
    # True: get("key") → "value"  (str)
    # False: get("key") → b"value"  (bytes)
    # Рекомендую True для більшості випадків

    # ─── Timeouts ───────────────────────────────────────────────
    socket_timeout=5.0,
    # Скільки секунд чекати відповіді від Redis.
    # При перевищенні — TimeoutError (якщо retry_on_timeout=False)
    # або повторна спроба (якщо retry_on_timeout=True)

    socket_connect_timeout=5.0,
    # Скільки секунд чекати на встановлення TCP з'єднання.
    # Важливо при старті: якщо Redis ще не запустився.

    # ─── Keepalive ──────────────────────────────────────────────
    socket_keepalive=True,
    # TCP keepalive — ОС буде перевіряти чи живе з'єднання.
    # Без цього "мертве" з'єднання може залишатись у пулі.
    socket_keepalive_options={},
    # Системні параметри keepalive (зазвичай не треба міняти)

    # ─── Retry ──────────────────────────────────────────────────
    retry_on_timeout=True,
    # True: при timeout — повторити один раз.
    # Корисно при тимчасових мережевих проблемах.

    retry_on_error=[ConnectionError, TimeoutError],
    # Список типів помилок при яких робити retry.

    # ─── Health check ───────────────────────────────────────────
    health_check_interval=30,
    # Кожні 30 секунд перевіряти з'єднання через PING.
    # Виявляє "мертві" з'єднання до того як вони нам знадобляться.
)
```

---

## 4. Ініціалізація через lifespan

### Що таке lifespan

`lifespan` — це контекстний менеджер FastAPI для startup/shutdown логіки.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # ===== ВСЕ ДО yield = STARTUP =====
    # Виконується один раз при старті додатку
    print("Starting up...")
    
    yield  # ← тут додаток живе і обробляє запити
    
    # ===== ВСЕ ПІСЛЯ yield = SHUTDOWN =====
    # Виконується при зупинці додатку (Ctrl+C, Docker stop, etc.)
    print("Shutting down...")

app = FastAPI(lifespan=lifespan)
```

### Повна ініціалізація Redis через lifespan

```python
# app/core/redis.py
import redis.asyncio as aioredis
import logging

logger = logging.getLogger(__name__)

_pool: aioredis.ConnectionPool | None = None


def create_pool(url: str, max_connections: int = 20) -> aioredis.ConnectionPool:
    return aioredis.ConnectionPool.from_url(
        url,
        max_connections=max_connections,
        decode_responses=True,
        socket_timeout=5.0,
        socket_connect_timeout=5.0,
        socket_keepalive=True,
        retry_on_timeout=True,
        health_check_interval=30,
    )


def get_redis() -> aioredis.Redis:
    if _pool is None:
        raise RuntimeError("Redis pool is not initialized")
    return aioredis.Redis(connection_pool=_pool)


async def close_pool() -> None:
    if _pool is not None:
        await _pool.disconnect()
        logger.info("Redis pool disconnected")
```

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
import app.core.redis as redis_module
from app.core.config import settings
import logging

logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ──────────── STARTUP ────────────
    logger.info("Initializing Redis connection pool...")
    
    # Створюємо пул і зберігаємо в глобальну змінну модуля
    redis_module._pool = redis_module.create_pool(
        url=settings.REDIS_URL,
        max_connections=settings.REDIS_MAX_CONNECTIONS,
    )
    
    # Перевіряємо з'єднання
    redis = redis_module.get_redis()
    try:
        pong = await redis.ping()
        # pong = True
        logger.info(f"✅ Redis connected: {settings.REDIS_URL}")
    except Exception as e:
        logger.error(f"❌ Redis connection failed: {e}")
        raise  # зупинити старт якщо Redis недоступний
    
    # Тут можна ініціалізувати інші ресурси:
    # - SQLAlchemy engine
    # - HTTP client (httpx.AsyncClient)
    # - etc.
    
    yield  # ← додаток живе тут
    
    # ──────────── SHUTDOWN ────────────
    logger.info("Closing Redis connection pool...")
    await redis_module.close_pool()
    logger.info("✅ Redis pool closed")


app = FastAPI(
    title="Health Patch API",
    lifespan=lifespan,
)

# Підключення роутерів
# app.include_router(auth_router)
# app.include_router(users_router)
```

### Чому не `@app.on_event` (застарілий підхід)

```python
# ❌ ЗАСТАРІЛИЙ ПІДХІД (deprecated в FastAPI 0.93+)
@app.on_event("startup")
async def startup():
    ...

@app.on_event("shutdown")
async def shutdown():
    ...

# ✅ ПРАВИЛЬНИЙ ПІДХІД — lifespan
@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    yield
    # shutdown
```

`lifespan` кращий тому що:
- Гарантує виконання shutdown навіть при виключенні в startup
- Краще тестується
- Офіційно рекомендований Starlette/FastAPI

---

## 5. Settings — конфігурація через .env

```python
# app/core/config.py
from pydantic_settings import BaseSettings
from pydantic import field_validator
from functools import lru_cache


class Settings(BaseSettings):
    # ─── Застосунок ─────────────────────────────────────────────
    APP_ENV: str = "development"     # development | staging | production
    DEBUG: bool = False
    SECRET_KEY: str                  # для підпису JWT, сесій тощо

    # ─── Redis ──────────────────────────────────────────────────
    REDIS_URL: str = "redis://localhost:6379/0"
    REDIS_MAX_CONNECTIONS: int = 20
    
    # ─── OAuth TTL ──────────────────────────────────────────────
    OAUTH_STATE_TTL: int = 600        # 10 хвилин
    
    # ─── Сесії ──────────────────────────────────────────────────
    SESSION_TTL: int = 60 * 60 * 24 * 7   # 7 днів
    
    # ─── Кеш ────────────────────────────────────────────────────
    CACHE_DEFAULT_TTL: int = 300      # 5 хвилин
    
    # ─── Rate Limiting ──────────────────────────────────────────
    RATE_LIMIT_REQUESTS: int = 60
    RATE_LIMIT_WINDOW: int = 60       # секунд
    
    # ─── OAuth провайдери ────────────────────────────────────────
    GOOGLE_CLIENT_ID: str = ""
    GOOGLE_CLIENT_SECRET: str = ""
    GOOGLE_REDIRECT_URI: str = "http://localhost:8000/auth/google/callback"
    
    GITHUB_CLIENT_ID: str = ""
    GITHUB_CLIENT_SECRET: str = ""
    GITHUB_REDIRECT_URI: str = "http://localhost:8000/auth/github/callback"

    # ─── JWT ────────────────────────────────────────────────────
    JWT_ALGORITHM: str = "HS256"
    ACCESS_TOKEN_TTL: int = 60 * 15         # 15 хвилин
    REFRESH_TOKEN_TTL: int = 60 * 60 * 24 * 30  # 30 днів

    @field_validator("REDIS_URL")
    @classmethod
    def validate_redis_url(cls, v: str) -> str:
        if not v.startswith(("redis://", "rediss://", "unix://")):
            raise ValueError("REDIS_URL must start with redis://, rediss://, or unix://")
        return v

    model_config = {
        "env_file": ".env",
        "env_file_encoding": "utf-8",
        "case_sensitive": True,
    }


# Singleton — читається один раз при старті
@lru_cache
def get_settings() -> Settings:
    return Settings()


settings = get_settings()
```

```bash
# .env
APP_ENV=development
DEBUG=true
SECRET_KEY=your-secret-key-min-32-chars-long-here

REDIS_URL=redis://localhost:6379/0
REDIS_MAX_CONNECTIONS=20

GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxxxxxxxxx
GITHUB_CLIENT_ID=Iv1.xxxxxxxxxx
GITHUB_CLIENT_SECRET=xxxxxxxxxx

JWT_ALGORITHM=HS256
```

```bash
# .env.docker (для docker-compose)
REDIS_URL=redis://redis:6379/0
# В Docker: hostname = ім'я сервісу в docker-compose.yml
```

---

## 6. Dependency Injection — RedisDep

### Що таке Depends у FastAPI

`Depends` — це механізм dependency injection (DI) FastAPI. Він дозволяє "вставляти" залежності в роути автоматично.

```python
from fastapi import Depends

async def get_current_user(token: str = Depends(get_token)):
    # FastAPI автоматично викликає get_token() і передає результат
    return decode(token)

@router.get("/profile")
async def profile(user = Depends(get_current_user)):
    # FastAPI викликає get_current_user() і передає результат
    return user
```

### Залежність для Redis

```python
# app/dependencies/redis.py
from typing import Annotated
import redis.asyncio as aioredis
from fastapi import Depends
from app.core.redis import get_redis


async def get_redis_dep() -> aioredis.Redis:
    """
    Dependency функція для отримання Redis клієнта.
    
    Повертає клієнт з глобального пулу.
    Не потребує cleanup (з'єднання повертається в пул автоматично).
    
    Чому не використовувати get_redis() напряму в роутах?
    1. Тестування: можна замінити залежність на mock
    2. Консистентність: один спосіб отримати Redis скрізь
    3. Майбутні зміни: можна додати логіку (наприклад, per-request client)
    """
    return get_redis()


# Annotated — зручний shortcut щоб не писати Depends скрізь
RedisDep = Annotated[aioredis.Redis, Depends(get_redis_dep)]
```

### Використання в роутах

```python
# app/routers/example.py
from fastapi import APIRouter
from app.dependencies.redis import RedisDep
import json

router = APIRouter(prefix="/example", tags=["example"])


@router.get("/ping")
async def ping_redis(redis: RedisDep):
    """Перевірити з'єднання з Redis."""
    result = await redis.ping()
    return {"redis": "ok", "pong": result}


@router.get("/set-get")
async def set_and_get(redis: RedisDep):
    """Простий приклад SET і GET."""
    await redis.set("greeting", "Hello from FastAPI!", ex=60)
    value = await redis.get("greeting")
    return {"value": value}


@router.get("/info")
async def redis_info(redis: RedisDep):
    """Інформація про Redis."""
    info = await redis.info("server")
    return {
        "version": info["redis_version"],
        "uptime_seconds": info["uptime_in_seconds"],
        "connected_clients": info.get("connected_clients"),
    }
```

### Чому `Annotated[Redis, Depends(...)]` краще ніж просто `Depends`

```python
# ❌ Традиційний варіант — повторення коду скрізь
@router.get("/a")
async def route_a(redis: aioredis.Redis = Depends(get_redis_dep)):
    ...

@router.get("/b")  
async def route_b(redis: aioredis.Redis = Depends(get_redis_dep)):
    ...

# ✅ З Annotated — лаконічно і без дублювання
RedisDep = Annotated[aioredis.Redis, Depends(get_redis_dep)]

@router.get("/a")
async def route_a(redis: RedisDep):
    ...

@router.get("/b")
async def route_b(redis: RedisDep):
    ...
```

---

## 7. Базовий RedisRepository

Repository патерн — це прошарок між бізнес-логікою і сховищем даних. Замість прямих `redis.set(...)` викликів у роутах — виносимо їх в клас.

**Переваги:**
- Всі Redis ключі в одному місці
- Легко тестувати (можна підмінити репозиторій на mock)
- Перевикористання логіки
- Типізація

```python
# app/repositories/base_redis.py
import json
from typing import Any, TypeVar, Generic
import redis.asyncio as aioredis

T = TypeVar("T")


class BaseRedisRepository:
    """
    Базовий клас для всіх Redis репозиторіїв.
    Надає зручні методи для типових операцій.
    """

    def __init__(self, redis: aioredis.Redis):
        self.redis = redis

    # ─── Прості String операції ──────────────────────────────────

    async def set(
        self,
        key: str,
        value: str,
        ttl: int | None = None,
        nx: bool = False,
        xx: bool = False,
    ) -> bool:
        """
        Встановити значення.
        
        nx=True: встановити тільки якщо ключ НЕ існує
        xx=True: встановити тільки якщо ключ вже існує
        Повертає True якщо встановлено, False якщо ні (при nx/xx).
        """
        result = await self.redis.set(key, value, ex=ttl, nx=nx, xx=xx)
        return result is not None

    async def get(self, key: str) -> str | None:
        """Отримати значення або None якщо ключ не існує."""
        return await self.redis.get(key)

    async def delete(self, *keys: str) -> int:
        """Видалити ключі. Повертає кількість видалених."""
        if not keys:
            return 0
        return await self.redis.delete(*keys)

    async def exists(self, key: str) -> bool:
        """Перевірити чи існує ключ."""
        return bool(await self.redis.exists(key))

    async def ttl(self, key: str) -> int:
        """
        Залишок TTL в секундах.
        -1: ключ існує але без TTL
        -2: ключ не існує
        """
        return await self.redis.ttl(key)

    async def expire(self, key: str, seconds: int) -> bool:
        """Встановити або оновити TTL. Повертає False якщо ключ не існує."""
        return bool(await self.redis.expire(key, seconds))

    async def persist(self, key: str) -> bool:
        """Прибрати TTL (зробити ключ безстроковим)."""
        return bool(await self.redis.persist(key))

    # ─── Атомарні операції ───────────────────────────────────────

    async def getdel(self, key: str) -> str | None:
        """Отримати і видалити атомарно. Повертає None якщо не існував."""
        return await self.redis.getdel(key)

    async def getex(self, key: str, ex: int | None = None) -> str | None:
        """Отримати і оновити TTL атомарно."""
        return await self.redis.getex(key, ex=ex)

    async def incr(self, key: str, amount: int = 1) -> int:
        """Атомарно збільшити числовий ключ."""
        return await self.redis.incrby(key, amount)

    async def decr(self, key: str, amount: int = 1) -> int:
        """Атомарно зменшити числовий ключ."""
        return await self.redis.decrby(key, amount)

    # ─── JSON операції ───────────────────────────────────────────

    async def set_json(
        self,
        key: str,
        value: Any,
        ttl: int | None = None,
    ) -> None:
        """Серіалізувати об'єкт в JSON і зберегти."""
        serialized = json.dumps(value, default=str, ensure_ascii=False)
        await self.redis.set(key, serialized, ex=ttl)

    async def get_json(self, key: str) -> Any | None:
        """Отримати і десеріалізувати JSON. Повертає None якщо не існує."""
        raw = await self.redis.get(key)
        if raw is None:
            return None
        return json.loads(raw)

    async def getdel_json(self, key: str) -> Any | None:
        """Атомарно отримати, десеріалізувати і видалити."""
        raw = await self.redis.getdel(key)
        if raw is None:
            return None
        return json.loads(raw)

    # ─── Масові операції ─────────────────────────────────────────

    async def mset(self, mapping: dict[str, str]) -> None:
        """Встановити кілька ключів атомарно."""
        await self.redis.mset(mapping)

    async def mget(self, *keys: str) -> list[str | None]:
        """Отримати кілька ключів одним запитом."""
        return await self.redis.mget(*keys)

    # ─── Scan (безпечний перебір) ─────────────────────────────────

    async def scan_keys(
        self,
        pattern: str,
        count: int = 100,
    ) -> list[str]:
        """
        Безпечний перебір ключів за патерном.
        Використовує SCAN ітеративно (не блокує Redis).
        
        НЕ використовуй KEYS * в production!
        """
        keys = []
        async for key in self.redis.scan_iter(match=pattern, count=count):
            keys.append(key)
        return keys

    async def delete_by_pattern(self, pattern: str) -> int:
        """Видалити всі ключі за патерном. Повертає кількість видалених."""
        deleted = 0
        async for key in self.redis.scan_iter(match=pattern):
            await self.redis.delete(key)
            deleted += 1
        return deleted

    # ─── Ping / Health ───────────────────────────────────────────

    async def ping(self) -> bool:
        """Перевірити з'єднання."""
        try:
            result = await self.redis.ping()
            return result is True
        except Exception:
            return False
```

---

## 8. OAuth State — повна реалізація

### Що таке OAuth state і навіщо Redis

Нагадаємо flow:

```
1. GET /auth/google/login
   └─ генеруємо state = random token
   └─ Redis.SET oauth_state:{state} data EX 600
   └─ redirect → google.com/oauth?state={state}&client_id=...

2. Юзер логіниться в Google

3. Google → GET /auth/google/callback?code=XXX&state={state}
   └─ Redis.GETDEL oauth_state:{state}  ← чи існує?
   └─ якщо nil → 400 (прострочений або атака)
   └─ якщо є → обмінюємо code на токени
```

**Чому Redis а не просто JWT state:**
- State має жити **не довго** (10 хвилин) і **одноразово** — Redis з TTL ідеально підходить
- State не має сенсу зберігати в БД (мусор після 10 хвилин)
- Redis атомарний `GETDEL` = перевірка + видалення без race condition

### OAuthStateRepository

```python
# app/repositories/oauth_state_repository.py
import secrets
import json
from dataclasses import dataclass, asdict
from datetime import datetime, UTC
import redis.asyncio as aioredis
from app.repositories.base_redis import BaseRedisRepository


@dataclass
class OAuthStateData:
    """Дані що зберігаються разом з OAuth state."""
    provider: str           # "google" | "github" | "facebook"
    redirect_after: str     # куди редіректити після успішного OAuth
    created_at: str         # для дебагу
    ip_address: str | None = None   # опціонально для логування


class OAuthStateRepository(BaseRedisRepository):
    """
    Репозиторій для управління OAuth state токенами.
    
    State токен — це випадковий рядок що:
    1. Генерується при ініціалізації OAuth flow
    2. Зберігається в Redis з TTL
    3. Передається в OAuth провайдер (Google, GitHub тощо)
    4. Повертається провайдером у callback
    5. Перевіряється і одразу видаляється (одноразовий!)
    
    Захищає від CSRF атак на OAuth flow.
    """

    KEY_PREFIX = "oauth_state"
    DEFAULT_TTL = 600  # 10 хвилин — достатньо для завершення OAuth flow

    def _make_key(self, state: str) -> str:
        """
        Формат ключа: oauth_state:{state}
        Приклад: oauth_state:abc123def456xyz789
        """
        return f"{self.KEY_PREFIX}:{state}"

    def _generate_state(self) -> str:
        """
        Генерує криптографічно безпечний токен.
        
        secrets.token_urlsafe(32) генерує 32 байти з /dev/urandom,
        потім кодує в base64url → 43 символи.
        
        base64url безпечний для URL (без +, /, =).
        32 байти = 256 біт ентропії — неможливо підібрати.
        """
        return secrets.token_urlsafe(32)

    async def create(
        self,
        provider: str,
        redirect_after: str = "/",
        ip_address: str | None = None,
        ttl: int = DEFAULT_TTL,
    ) -> str:
        """
        Генерує новий state токен і зберігає в Redis.
        
        Повертає згенерований state (передається в OAuth URL).
        """
        state = self._generate_state()
        
        data = OAuthStateData(
            provider=provider,
            redirect_after=redirect_after,
            created_at=datetime.now(UTC).isoformat(),
            ip_address=ip_address,
        )
        
        # Зберігаємо як JSON з TTL
        # Ключ автоматично видалиться через ttl секунд
        await self.set_json(self._make_key(state), asdict(data), ttl=ttl)
        
        return state

    async def validate_and_consume(self, state: str) -> OAuthStateData | None:
        """
        Перевіряє state і ОДРАЗУ видаляє (одноразовий!).
        
        Повертає OAuthStateData якщо state валідний.
        Повертає None якщо state не існує (прострочений або неправильний).
        
        Використовує GETDEL — атомарна операція.
        Між GET і DEL ніхто не встигне вставитись → no race condition.
        """
        raw = await self.getdel_json(self._make_key(state))
        
        if raw is None:
            return None
        
        return OAuthStateData(**raw)

    async def peek(self, state: str) -> OAuthStateData | None:
        """
        Перевірити state БЕЗ видалення (для дебагу).
        В продакшні використовуй validate_and_consume.
        """
        raw = await self.get_json(self._make_key(state))
        if raw is None:
            return None
        return OAuthStateData(**raw)

    async def revoke(self, state: str) -> bool:
        """Явно видалити state (наприклад при відміні flow)."""
        deleted = await self.delete(self._make_key(state))
        return deleted > 0

    async def get_remaining_ttl(self, state: str) -> int:
        """Скільки секунд залишилось у state."""
        return await self.ttl(self._make_key(state))
```

### Auth Router — Google OAuth

```python
# app/routers/auth/google.py
import httpx
from urllib.parse import urlencode
from fastapi import APIRouter, Depends, HTTPException, Query, Request
from fastapi.responses import RedirectResponse
from app.dependencies.redis import RedisDep
from app.repositories.oauth_state_repository import OAuthStateRepository
from app.core.config import settings
import logging

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/auth/google", tags=["auth:google"])

GOOGLE_AUTH_URL = "https://accounts.google.com/o/oauth2/v2/auth"
GOOGLE_TOKEN_URL = "https://oauth2.googleapis.com/token"
GOOGLE_USERINFO_URL = "https://www.googleapis.com/oauth2/v2/userinfo"


@router.get(
    "/login",
    summary="Ініціювати Google OAuth flow",
    description="Генерує CSRF state, зберігає в Redis, редіректить на Google",
)
async def google_login(
    request: Request,
    redis: RedisDep,
    redirect_after: str = Query(default="/", description="Куди редіректити після логіну"),
):
    """
    Крок 1 OAuth flow.
    
    1. Генеруємо state токен
    2. Зберігаємо в Redis з TTL=10хв
    3. Редіректимо юзера на Google
    """
    repo = OAuthStateRepository(redis)
    
    # Генеруємо state і зберігаємо в Redis
    # Разом з state зберігаємо redirect_after і IP для логування
    state = await repo.create(
        provider="google",
        redirect_after=redirect_after,
        ip_address=request.client.host if request.client else None,
        ttl=settings.OAUTH_STATE_TTL,
    )
    
    logger.info(f"OAuth state created: ...{state[-8:]} for IP {request.client.host if request.client else 'unknown'}")
    
    # Будуємо URL для Google OAuth
    params = {
        "client_id": settings.GOOGLE_CLIENT_ID,
        "redirect_uri": settings.GOOGLE_REDIRECT_URI,
        "response_type": "code",
        "scope": "openid email profile",
        "state": state,
        "access_type": "offline",    # отримати refresh_token
        "prompt": "consent",         # завжди показувати consent screen
    }
    
    auth_url = f"{GOOGLE_AUTH_URL}?{urlencode(params)}"
    
    return RedirectResponse(url=auth_url, status_code=302)


@router.get(
    "/callback",
    summary="Обробка callback від Google",
)
async def google_callback(
    redis: RedisDep,
    code: str = Query(..., description="Authorization code від Google"),
    state: str = Query(..., description="CSRF state токен"),
    error: str | None = Query(default=None, description="Помилка від Google"),
    error_description: str | None = Query(default=None),
):
    """
    Крок 2 OAuth flow.
    
    Google редіректить сюди після того як юзер авторизувався.
    
    1. Перевіряємо error — чи не відмовив юзер
    2. Валідуємо state (CSRF захист) — GETDEL атомарно
    3. Обмінюємо code на access_token
    4. Отримуємо дані юзера від Google
    5. Знаходимо або створюємо юзера в БД
    6. Видаємо JWT
    """

    # ─── Крок 1: Перевірити помилку з боку Google ──────────────
    if error:
        logger.warning(f"Google OAuth error: {error} — {error_description}")
        raise HTTPException(
            status_code=400,
            detail={
                "error": error,
                "description": error_description or "OAuth authorization failed",
            },
        )

    # ─── Крок 2: Валідувати CSRF state ────────────────────────
    repo = OAuthStateRepository(redis)
    state_data = await repo.validate_and_consume(state)
    
    if state_data is None:
        # State не знайдено в Redis — або:
        # - прострочений (>10 хвилин)
        # - вже використаний (replay attack)
        # - підроблений (CSRF атака)
        logger.warning(f"Invalid OAuth state: ...{state[-8:]}")
        raise HTTPException(
            status_code=400,
            detail="Invalid or expired OAuth state. Please start the login process again.",
        )
    
    logger.info(f"OAuth state validated for provider={state_data.provider}")

    # ─── Крок 3: Обміняти code на токени ──────────────────────
    async with httpx.AsyncClient(timeout=10.0) as client:
        try:
            token_response = await client.post(
                GOOGLE_TOKEN_URL,
                data={
                    "client_id": settings.GOOGLE_CLIENT_ID,
                    "client_secret": settings.GOOGLE_CLIENT_SECRET,
                    "code": code,
                    "grant_type": "authorization_code",
                    "redirect_uri": settings.GOOGLE_REDIRECT_URI,
                },
                headers={"Content-Type": "application/x-www-form-urlencoded"},
            )
            token_response.raise_for_status()
            tokens = token_response.json()
        except httpx.HTTPStatusError as e:
            logger.error(f"Google token exchange failed: {e.response.text}")
            raise HTTPException(502, "Failed to exchange authorization code")
        except httpx.RequestError as e:
            logger.error(f"Google token request error: {e}")
            raise HTTPException(502, "Failed to connect to Google")

    access_token = tokens.get("access_token")
    # refresh_token = tokens.get("refresh_token")  # якщо access_type=offline

    # ─── Крок 4: Отримати дані юзера ──────────────────────────
    async with httpx.AsyncClient(timeout=10.0) as client:
        try:
            user_response = await client.get(
                GOOGLE_USERINFO_URL,
                headers={"Authorization": f"Bearer {access_token}"},
            )
            user_response.raise_for_status()
            google_user = user_response.json()
        except httpx.HTTPStatusError as e:
            logger.error(f"Google userinfo failed: {e}")
            raise HTTPException(502, "Failed to get user info from Google")

    # ─── Крок 5 і 6: Знайти/створити юзера і видати JWT ────────
    # В реальному проєкті:
    # user = await user_service.find_or_create_from_google(google_user)
    # jwt = create_access_token({"sub": str(user.id)})
    # return RedirectResponse(f"{state_data.redirect_after}?token={jwt}")

    return {
        "message": "OAuth successful",
        "redirect_after": state_data.redirect_after,
        "user": {
            "google_id": google_user.get("id"),
            "email": google_user.get("email"),
            "name": google_user.get("name"),
            "picture": google_user.get("picture"),
            "email_verified": google_user.get("verified_email"),
        },
    }
```

### Auth Router — GitHub OAuth

```python
# app/routers/auth/github.py
import httpx
from urllib.parse import urlencode
from fastapi import APIRouter, Query, Request, HTTPException
from fastapi.responses import RedirectResponse
from app.dependencies.redis import RedisDep
from app.repositories.oauth_state_repository import OAuthStateRepository
from app.core.config import settings

router = APIRouter(prefix="/auth/github", tags=["auth:github"])

GITHUB_AUTH_URL = "https://github.com/login/oauth/authorize"
GITHUB_TOKEN_URL = "https://github.com/login/oauth/access_token"
GITHUB_USER_URL = "https://api.github.com/user"
GITHUB_EMAILS_URL = "https://api.github.com/user/emails"


@router.get("/login")
async def github_login(
    request: Request,
    redis: RedisDep,
    redirect_after: str = Query(default="/"),
):
    repo = OAuthStateRepository(redis)
    state = await repo.create(
        provider="github",
        redirect_after=redirect_after,
        ip_address=request.client.host if request.client else None,
    )

    params = {
        "client_id": settings.GITHUB_CLIENT_ID,
        "redirect_uri": settings.GITHUB_REDIRECT_URI,
        "scope": "read:user user:email",
        "state": state,
    }

    return RedirectResponse(f"{GITHUB_AUTH_URL}?{urlencode(params)}")


@router.get("/callback")
async def github_callback(
    redis: RedisDep,
    code: str = Query(...),
    state: str = Query(...),
    error: str | None = Query(default=None),
):
    if error:
        raise HTTPException(400, f"GitHub OAuth error: {error}")

    # Валідувати state
    repo = OAuthStateRepository(redis)
    state_data = await repo.validate_and_consume(state)
    if state_data is None:
        raise HTTPException(400, "Invalid or expired state")

    # Обміняти code на token
    async with httpx.AsyncClient() as client:
        token_resp = await client.post(
            GITHUB_TOKEN_URL,
            data={
                "client_id": settings.GITHUB_CLIENT_ID,
                "client_secret": settings.GITHUB_CLIENT_SECRET,
                "code": code,
            },
            headers={"Accept": "application/json"},
        )
        token_resp.raise_for_status()
        token_data = token_resp.json()

    access_token = token_data.get("access_token")
    if not access_token:
        raise HTTPException(502, "Failed to get GitHub access token")

    headers = {
        "Authorization": f"Bearer {access_token}",
        "Accept": "application/vnd.github+json",
    }

    # Отримати дані юзера
    async with httpx.AsyncClient() as client:
        user_resp = await client.get(GITHUB_USER_URL, headers=headers)
        user_resp.raise_for_status()
        github_user = user_resp.json()

        # GitHub може не давати email через /user якщо він приватний
        emails_resp = await client.get(GITHUB_EMAILS_URL, headers=headers)
        emails = emails_resp.json() if emails_resp.status_code == 200 else []

    primary_email = next(
        (e["email"] for e in emails if e.get("primary") and e.get("verified")),
        github_user.get("email"),
    )

    return {
        "user": {
            "github_id": github_user["id"],
            "login": github_user["login"],
            "email": primary_email,
            "name": github_user.get("name"),
            "avatar_url": github_user.get("avatar_url"),
        },
        "redirect_after": state_data.redirect_after,
    }
```

---

## 9. Session Repository

```python
# app/repositories/session_repository.py
import secrets
import json
from datetime import datetime, UTC
from dataclasses import dataclass, asdict
import redis.asyncio as aioredis
from app.repositories.base_redis import BaseRedisRepository


@dataclass
class SessionData:
    user_id: int
    created_at: str
    ip_address: str | None = None
    user_agent: str | None = None
    # Можна додати будь-що: device_type, location, etc.


class SessionRepository(BaseRedisRepository):
    """
    Репозиторій для серверних сесій.
    
    На відміну від JWT (stateless), сесія в Redis дозволяє:
    - Миттєвий logout (просто видалити ключ)
    - Logout з усіх пристроїв (видалити всі сесії юзера)
    - Зберігати довільні дані сесії
    - Бачити активні сесії юзера
    """

    KEY_PREFIX = "session"
    USER_SESSIONS_PREFIX = "user_sessions"
    DEFAULT_TTL = 60 * 60 * 24 * 7  # 7 днів

    def _session_key(self, session_id: str) -> str:
        return f"{self.KEY_PREFIX}:{session_id}"

    def _user_sessions_key(self, user_id: int) -> str:
        # Зберігаємо Set з усіма session_id юзера
        return f"{self.USER_SESSIONS_PREFIX}:{user_id}"

    async def create(
        self,
        user_id: int,
        ip_address: str | None = None,
        user_agent: str | None = None,
        ttl: int = DEFAULT_TTL,
    ) -> str:
        """
        Створити нову сесію.
        Повертає session_id (зберігається в httpOnly cookie).
        """
        session_id = secrets.token_urlsafe(32)

        data = SessionData(
            user_id=user_id,
            created_at=datetime.now(UTC).isoformat(),
            ip_address=ip_address,
            user_agent=user_agent,
        )

        # Зберегти дані сесії
        await self.set_json(self._session_key(session_id), asdict(data), ttl=ttl)

        # Додати session_id до Set активних сесій юзера
        # Це дозволяє видалити всі сесії одним запитом
        user_key = self._user_sessions_key(user_id)
        await self.redis.sadd(user_key, session_id)
        await self.redis.expire(user_key, ttl)  # TTL = найдовша можлива сесія

        return session_id

    async def get(self, session_id: str) -> SessionData | None:
        """Отримати дані сесії або None якщо не існує/прострочена."""
        raw = await self.get_json(self._session_key(session_id))
        if raw is None:
            return None
        return SessionData(**raw)

    async def refresh(self, session_id: str, ttl: int = DEFAULT_TTL) -> bool:
        """
        Продовжити TTL сесії (sliding expiration).
        Викликати при кожному запиті від авторизованого юзера.
        """
        return await self.expire(self._session_key(session_id), ttl)

    async def delete(self, session_id: str, user_id: int) -> bool:
        """Видалити конкретну сесію (logout з одного пристрою)."""
        # Видалити дані сесії
        deleted = await super().delete(self._session_key(session_id))

        # Прибрати з Set активних сесій юзера
        await self.redis.srem(self._user_sessions_key(user_id), session_id)

        return deleted > 0

    async def delete_all_user_sessions(self, user_id: int) -> int:
        """
        Видалити ВСІ сесії юзера (logout з усіх пристроїв).
        Корисно при:
        - зміні пароля
        - блокуванні акаунту
        - security alert
        """
        user_key = self._user_sessions_key(user_id)

        # Отримати всі session_id юзера
        session_ids = await self.redis.smembers(user_key)

        if not session_ids:
            return 0

        # Видалити всі сесії
        # Pipeline = відправити всі команди одним TCP пакетом
        async with self.redis.pipeline() as pipe:
            for sid in session_ids:
                pipe.delete(self._session_key(sid))
            pipe.delete(user_key)
            await pipe.execute()

        return len(session_ids)

    async def get_active_sessions(self, user_id: int) -> list[dict]:
        """Отримати список всіх активних сесій юзера."""
        user_key = self._user_sessions_key(user_id)
        session_ids = await self.redis.smembers(user_key)

        sessions = []
        for sid in session_ids:
            data = await self.get_json(self._session_key(sid))
            if data is not None:  # може бути прострочена
                sessions.append({
                    "session_id": sid[:8] + "...",   # не повертаємо повний ID
                    **data,
                    "ttl_seconds": await self.ttl(self._session_key(sid)),
                })

        return sessions
```

### Використання сесій у роутах

```python
# app/routers/auth/sessions.py
from fastapi import APIRouter, Response, Cookie, HTTPException, Request
from app.dependencies.redis import RedisDep
from app.repositories.session_repository import SessionRepository

router = APIRouter(prefix="/auth", tags=["sessions"])

SESSION_COOKIE = "session_id"
SESSION_TTL = 60 * 60 * 24 * 7  # 7 днів


@router.post("/login")
async def login(
    request: Request,
    response: Response,
    redis: RedisDep,
    # data: LoginRequest,  ← в реальному проєкті
):
    # Після перевірки пароля / OAuth
    user_id = 42  # з БД

    repo = SessionRepository(redis)
    session_id = await repo.create(
        user_id=user_id,
        ip_address=request.client.host if request.client else None,
        user_agent=request.headers.get("user-agent"),
        ttl=SESSION_TTL,
    )

    # Зберегти session_id в httpOnly cookie
    response.set_cookie(
        key=SESSION_COOKIE,
        value=session_id,
        httponly=True,       # недоступний для JavaScript (захист від XSS)
        secure=True,         # тільки HTTPS
        samesite="lax",      # захист від CSRF
        max_age=SESSION_TTL,
    )

    return {"message": "Logged in successfully"}


@router.post("/logout")
async def logout(
    response: Response,
    redis: RedisDep,
    session_id: str | None = Cookie(default=None, alias=SESSION_COOKIE),
):
    if session_id:
        repo = SessionRepository(redis)
        session = await repo.get(session_id)
        if session:
            await repo.delete(session_id, session.user_id)

    response.delete_cookie(SESSION_COOKIE)
    return {"message": "Logged out"}


@router.post("/logout-all")
async def logout_all(
    response: Response,
    redis: RedisDep,
    session_id: str | None = Cookie(default=None, alias=SESSION_COOKIE),
):
    """Logout з усіх пристроїв."""
    if not session_id:
        raise HTTPException(401, "Not authenticated")

    repo = SessionRepository(redis)
    session = await repo.get(session_id)
    if not session:
        raise HTTPException(401, "Invalid session")

    deleted = await repo.delete_all_user_sessions(session.user_id)
    response.delete_cookie(SESSION_COOKIE)

    return {"message": f"Logged out from {deleted} devices"}
```

---

## 10. Кешування — декоратор і вручну

### Ручне кешування в роутах

```python
# app/routers/users.py
import json
from fastapi import APIRouter
from app.dependencies.redis import RedisDep

router = APIRouter(prefix="/users", tags=["users"])
CACHE_TTL = 300  # 5 хвилин


@router.get("/{user_id}/profile")
async def get_profile(user_id: int, redis: RedisDep):
    """
    Патерн Cache-Aside:
    1. Перевірити кеш
    2. Якщо є — повернути
    3. Якщо немає — запитати БД, зберегти в кеш, повернути
    """
    cache_key = f"cache:user:{user_id}:profile"

    # 1. Перевірити кеш
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)

    # 2. Дані не в кеші — запитати БД
    # profile = await db.get(UserProfile, user_id)
    profile = {"id": user_id, "name": "Daniil", "city": "Bila Tserkva"}

    # 3. Зберегти в кеш
    await redis.set(cache_key, json.dumps(profile), ex=CACHE_TTL)

    return profile
```

### CacheRepository — виносимо логіку

```python
# app/repositories/cache_repository.py
import json
import hashlib
from typing import Any, Callable, TypeVar
import redis.asyncio as aioredis
from app.repositories.base_redis import BaseRedisRepository

T = TypeVar("T")


class CacheRepository(BaseRedisRepository):
    """Репозиторій для кешування довільних даних."""

    KEY_PREFIX = "cache"

    def _make_key(self, namespace: str, *parts: Any) -> str:
        """
        Будує ключ кешу.
        Якщо частини занадто довгі — хешує їх для короткого ключа.
        """
        raw_key = f"{self.KEY_PREFIX}:{namespace}:" + ":".join(str(p) for p in parts)

        # Якщо ключ занадто довгий — зробити SHA1 хеш
        if len(raw_key) > 200:
            hash_part = hashlib.sha1(raw_key.encode()).hexdigest()
            return f"{self.KEY_PREFIX}:{namespace}:{hash_part}"

        return raw_key

    async def get_or_set(
        self,
        key: str,
        factory: Callable,
        ttl: int = 300,
    ) -> Any:
        """
        Отримати з кешу або обчислити і зберегти.
        
        key: ключ кешу
        factory: async функція що повертає дані якщо кешу немає
        ttl: час кешування в секундах
        
        Приклад:
            data = await cache.get_or_set(
                key="user:42:stats",
                factory=lambda: db.get_user_stats(42),
                ttl=300,
            )
        """
        # Перевірити кеш
        cached = await self.get_json(key)
        if cached is not None:
            return cached

        # Виконати factory
        result = await factory()

        # Зберегти в кеш
        if result is not None:
            await self.set_json(key, result, ttl=ttl)

        return result

    async def invalidate(self, *keys: str) -> int:
        """Інвалідувати конкретні ключі."""
        return await self.delete(*keys)

    async def invalidate_namespace(self, namespace: str) -> int:
        """
        Інвалідувати всі ключі в namespace.
        Наприклад: invalidate_namespace("user:42") видалить
        всі ключі що починаються з cache:user:42:*
        """
        pattern = f"{self.KEY_PREFIX}:{namespace}:*"
        return await self.delete_by_pattern(pattern)

    def user_profile_key(self, user_id: int) -> str:
        return self._make_key("user", user_id, "profile")

    def user_stats_key(self, user_id: int) -> str:
        return self._make_key("user", user_id, "stats")

    def post_key(self, post_id: int) -> str:
        return self._make_key("post", post_id)

    def feed_key(self, user_id: int, page: int) -> str:
        return self._make_key("feed", user_id, f"page{page}")
```

### Декоратор @cached

```python
# app/utils/cache_decorator.py
import json
import functools
import hashlib
import inspect
from typing import Callable, Any
import redis.asyncio as aioredis
from app.core.redis import get_redis


def cached(
    ttl: int = 300,
    key_prefix: str | None = None,
    skip_cache_if: Callable[..., bool] | None = None,
):
    """
    Декоратор для кешування результату async функції.
    
    ttl: час кешування в секундах
    key_prefix: префікс ключа (за замовчуванням = ім'я функції)
    skip_cache_if: функція що повертає True якщо кеш треба пропустити
    
    Приклад:
        @cached(ttl=300, key_prefix="user_profile")
        async def get_user_profile(user_id: int) -> dict:
            return await db.get(User, user_id)
    """

    def decorator(func: Callable):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            # Перевірити чи треба пропустити кеш
            if skip_cache_if and skip_cache_if(*args, **kwargs):
                return await func(*args, **kwargs)

            redis = get_redis()

            # Будуємо ключ кешу з імені функції та аргументів
            prefix = key_prefix or func.__qualname__
            # Хешуємо аргументи для короткого і безпечного ключа
            args_hash = hashlib.md5(
                json.dumps({"args": args, "kwargs": kwargs}, default=str).encode()
            ).hexdigest()[:12]
            cache_key = f"cache:{prefix}:{args_hash}"

            # Перевірити кеш
            cached_value = await redis.get(cache_key)
            if cached_value is not None:
                return json.loads(cached_value)

            # Виконати функцію
            result = await func(*args, **kwargs)

            # Зберегти результат
            if result is not None:
                await redis.set(
                    cache_key,
                    json.dumps(result, default=str),
                    ex=ttl,
                )

            return result

        # Додати метод для інвалідації
        async def invalidate(*args, **kwargs):
            redis = get_redis()
            prefix = key_prefix or func.__qualname__
            args_hash = hashlib.md5(
                json.dumps({"args": args, "kwargs": kwargs}, default=str).encode()
            ).hexdigest()[:12]
            cache_key = f"cache:{prefix}:{args_hash}"
            await redis.delete(cache_key)

        wrapper.invalidate = invalidate
        return wrapper

    return decorator
```

```python
# Використання декоратора
from app.utils.cache_decorator import cached


@cached(ttl=300, key_prefix="user_profile")
async def get_user_profile(user_id: int) -> dict:
    # Важкий запит до БД
    return {"id": user_id, "name": "Daniil"}


@cached(ttl=60, key_prefix="leaderboard")
async def get_leaderboard(limit: int = 10) -> list:
    return []


# В роуті
@router.get("/profile/{user_id}")
async def profile(user_id: int):
    return await get_user_profile(user_id)


# Інвалідувати кеш після оновлення
@router.put("/profile/{user_id}")
async def update_profile(user_id: int, data: dict):
    # оновити в БД
    await get_user_profile.invalidate(user_id)   # видалити кеш
    return {"message": "Updated"}
```

---

## 11. Rate Limiting

### Sliding Window Rate Limiter

Sliding window — найточніший алгоритм rate limiting:

```
Ліміт: 5 запитів за 10 секунд

Запити: t=1, t=3, t=5, t=8, t=10  → всі пройшли (5 за 10с)
Запит:  t=11  → перевіряємо вікно [1, 11]
         видаляємо t=1 (поза вікном 11-10=1)
         залишилось: t=3,5,8,10 = 4 запити → дозволяємо
Запит:  t=12  → вікно [2, 12]
         видаляємо t=3 (поза вікном)
         залишилось: t=5,8,10,11 = 4 → дозволяємо
```

```python
# app/repositories/rate_limit_repository.py
import time
import uuid
import redis.asyncio as aioredis
from dataclasses import dataclass
from app.repositories.base_redis import BaseRedisRepository


@dataclass
class RateLimitResult:
    allowed: bool
    limit: int
    remaining: int
    reset_at: int        # Unix timestamp коли вікно скинеться
    retry_after: int     # секунд до наступної спроби (якщо blocked)


class RateLimitRepository(BaseRedisRepository):
    """
    Sliding Window Rate Limiter на основі Sorted Set.
    
    Ключ: rate_limit:{identifier}
    Тип: Sorted Set
    Member: унікальний ID запиту (uuid або timestamp+random)
    Score: Unix timestamp запиту
    
    Алгоритм:
    1. ZREMRANGEBYSCORE key 0 (now - window) → видалити старі
    2. ZCARD key → поточна кількість
    3. Якщо < limit → ZADD key now:uuid → дозволити
    4. Якщо >= limit → відмовити
    """

    KEY_PREFIX = "rate_limit"

    def _make_key(self, identifier: str) -> str:
        return f"{self.KEY_PREFIX}:{identifier}"

    async def check(
        self,
        identifier: str,
        limit: int = 60,
        window: int = 60,
    ) -> RateLimitResult:
        """
        Перевірити rate limit для identifier.
        
        identifier: унікальний ідентифікатор (IP, user_id, api_key)
        limit: максимальна кількість запитів за вікно
        window: розмір вікна в секундах
        
        Повертає RateLimitResult з інформацією про поточний стан.
        """
        key = self._make_key(identifier)
        now = time.time()
        window_start = now - window

        # Використовуємо pipeline для мінімальної кількості round-trips
        async with self.redis.pipeline() as pipe:
            # 1. Видалити всі записи старіші за вікно
            pipe.zremrangebyscore(key, 0, window_start)
            # 2. Порахувати поточну кількість запитів
            pipe.zcard(key)
            # 3. Встановити TTL (щоб ключ видалився якщо немає активності)
            pipe.expire(key, window)
            results = await pipe.execute()

        current_count: int = results[1]

        if current_count >= limit:
            # Отримати найстаріший запит (щоб сказати коли retry)
            oldest = await self.redis.zrange(key, 0, 0, withscores=True)
            if oldest:
                _, oldest_score = oldest[0]
                retry_after = int(oldest_score + window - now) + 1
            else:
                retry_after = window

            return RateLimitResult(
                allowed=False,
                limit=limit,
                remaining=0,
                reset_at=int(now) + retry_after,
                retry_after=retry_after,
            )

        # Дозволити — додати поточний запит
        member = f"{now:.6f}:{uuid.uuid4().hex[:8]}"
        await self.redis.zadd(key, {member: now})

        return RateLimitResult(
            allowed=True,
            limit=limit,
            remaining=limit - current_count - 1,
            reset_at=int(now + window),
            retry_after=0,
        )

    async def reset(self, identifier: str) -> None:
        """Скинути лічильник (для адмінів або тестів)."""
        await self.delete(self._make_key(identifier))
```

### Rate Limit Middleware

```python
# app/middleware/rate_limit.py
from fastapi import Request, Response
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
from app.core.redis import get_redis
from app.repositories.rate_limit_repository import RateLimitRepository


class RateLimitMiddleware(BaseHTTPMiddleware):
    """
    Глобальний Rate Limit Middleware.
    Застосовується до ВСІХ запитів.
    """

    def __init__(self, app, limit: int = 60, window: int = 60):
        super().__init__(app)
        self.limit = limit
        self.window = window

    async def dispatch(self, request: Request, call_next):
        # Визначити ідентифікатор: IP адреса
        client_ip = request.client.host if request.client else "unknown"

        redis = get_redis()
        repo = RateLimitRepository(redis)

        result = await repo.check(
            identifier=f"ip:{client_ip}",
            limit=self.limit,
            window=self.window,
        )

        # Додати rate limit headers у відповідь
        response_headers = {
            "X-RateLimit-Limit": str(result.limit),
            "X-RateLimit-Remaining": str(result.remaining),
            "X-RateLimit-Reset": str(result.reset_at),
        }

        if not result.allowed:
            return JSONResponse(
                status_code=429,
                content={
                    "detail": "Too Many Requests",
                    "retry_after": result.retry_after,
                },
                headers={
                    **response_headers,
                    "Retry-After": str(result.retry_after),
                },
            )

        response = await call_next(request)

        # Додати headers до реальної відповіді
        for key, value in response_headers.items():
            response.headers[key] = value

        return response
```

### Rate Limit як Dependency (для окремих роутів)

```python
# app/dependencies/rate_limit.py
from typing import Annotated
from fastapi import Depends, HTTPException, Request
from app.dependencies.redis import RedisDep
from app.repositories.rate_limit_repository import RateLimitRepository


def make_rate_limiter(limit: int = 60, window: int = 60):
    """
    Фабрика dependencies для rate limiting.
    
    Використання:
        @router.post("/login", dependencies=[Depends(make_rate_limiter(5, 60))])
        async def login(...):
            # максимум 5 запитів за 60 секунд
    """
    async def rate_limit_dep(request: Request, redis: RedisDep):
        repo = RateLimitRepository(redis)

        # Ідентифікатор: IP + endpoint
        identifier = f"ip:{request.client.host}:{request.url.path}"

        result = await repo.check(identifier=identifier, limit=limit, window=window)

        if not result.allowed:
            raise HTTPException(
                status_code=429,
                detail="Too many requests. Please try again later.",
                headers={"Retry-After": str(result.retry_after)},
            )

    return rate_limit_dep


# Ліміт для login/register (захист від brute force)
LoginRateLimit = Depends(make_rate_limiter(limit=5, window=60))

# Загальний API ліміт
ApiRateLimit = Depends(make_rate_limiter(limit=100, window=60))
```

```python
# Використання в роутах
from app.dependencies.rate_limit import LoginRateLimit, make_rate_limiter

@router.post("/login", dependencies=[LoginRateLimit])
async def login(data: LoginRequest, redis: RedisDep):
    # максимум 5 спроб за хвилину
    ...

@router.post("/register", dependencies=[Depends(make_rate_limiter(3, 3600))])
async def register(data: RegisterRequest):
    # максимум 3 реєстрації за годину
    ...
```

---

## 12. Distributed Lock

Розподілене блокування — гарантує що одна операція виконується тільки один раз одночасно.

```python
# app/repositories/lock_repository.py
import asyncio
import secrets
import redis.asyncio as aioredis
from contextlib import asynccontextmanager
from app.repositories.base_redis import BaseRedisRepository


# Lua скрипт для безпечного звільнення lock
# Перевіряє що ми власники перед видаленням (атомарно!)
RELEASE_LOCK_SCRIPT = """
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
"""


class LockRepository(BaseRedisRepository):
    """
    Distributed Lock через Redis.
    
    Патерн: SET key owner NX EX ttl
    - NX = тільки якщо не існує (атомарна перевірка + встановлення)
    - EX = автоматичне звільнення якщо власник впав
    - owner = унікальний ID власника (щоб не видалити чужий lock)
    """

    KEY_PREFIX = "lock"

    def _lock_key(self, resource: str) -> str:
        return f"{self.KEY_PREFIX}:{resource}"

    async def acquire(
        self,
        resource: str,
        ttl: int = 30,
        retry_times: int = 0,
        retry_delay: float = 0.1,
    ) -> str | None:
        """
        Захопити lock для ресурсу.
        
        resource: ім'я ресурсу (наприклад, "payment:order_99")
        ttl: максимальний час lock (сек). Після нього — автозвільнення.
        retry_times: кількість повторних спроб якщо lock зайнятий
        retry_delay: затримка між спробами (сек)
        
        Повертає lock_token якщо захопив, None якщо lock зайнятий.
        lock_token потрібен для release!
        """
        lock_token = secrets.token_urlsafe(16)
        key = self._lock_key(resource)

        for attempt in range(retry_times + 1):
            # SET key token NX EX ttl — атомарна операція
            result = await self.redis.set(key, lock_token, nx=True, ex=ttl)

            if result is not None:
                # Lock захоплено!
                return lock_token

            if attempt < retry_times:
                # Lock зайнятий — почекати і повторити
                await asyncio.sleep(retry_delay)

        return None  # Не вдалось захопити

    async def release(self, resource: str, lock_token: str) -> bool:
        """
        Звільнити lock.
        
        ВАЖЛИВО: передавати той самий lock_token що повернув acquire().
        Lua скрипт перевіряє що ми власники перед видаленням.
        
        Без цієї перевірки: якщо lock прострочив і його захопив інший —
        ми б видалили чужий lock.
        """
        key = self._lock_key(resource)
        result = await self.redis.eval(
            RELEASE_LOCK_SCRIPT,
            1,        # кількість ключів
            key,      # KEYS[1]
            lock_token,  # ARGV[1]
        )
        return bool(result)

    async def extend(self, resource: str, lock_token: str, ttl: int) -> bool:
        """
        Продовжити TTL lock (якщо операція займає більше часу ніж очікувалось).
        """
        key = self._lock_key(resource)
        # Перевірити що ми власники
        current_owner = await self.redis.get(key)
        if current_owner != lock_token:
            return False
        # Продовжити TTL
        return await self.expire(key, ttl)

    @asynccontextmanager
    async def lock(
        self,
        resource: str,
        ttl: int = 30,
        retry_times: int = 3,
        retry_delay: float = 0.1,
    ):
        """
        Context manager для distributed lock.
        
        Використання:
            async with lock_repo.lock("payment:order_99"):
                # критична секція
                await process_payment(order_id)
        """
        lock_token = await self.acquire(resource, ttl, retry_times, retry_delay)

        if lock_token is None:
            raise RuntimeError(f"Failed to acquire lock for '{resource}'")

        try:
            yield lock_token
        finally:
            await self.release(resource, lock_token)
```

```python
# Використання
@router.post("/payments/{order_id}/process")
async def process_payment(order_id: int, redis: RedisDep):
    from app.repositories.lock_repository import LockRepository

    lock_repo = LockRepository(redis)

    # Гарантуємо що платіж обробляється тільки раз
    try:
        async with lock_repo.lock(f"payment:{order_id}", ttl=30):
            # Тільки один воркер зайде сюди одночасно
            # await payment_service.process(order_id)
            return {"status": "processed"}
    except RuntimeError:
        raise HTTPException(409, "Payment is already being processed")
```

---

## 13. Blacklist для JWT токенів

При logout JWT залишається валідним до закінчення TTL. Blacklist дозволяє інвалідувати токен миттєво.

```python
# app/repositories/token_blacklist_repository.py
from app.repositories.base_redis import BaseRedisRepository


class TokenBlacklistRepository(BaseRedisRepository):
    """
    Blacklist для відкликаних JWT токенів.
    
    При logout: додаємо JTI (JWT ID) в blacklist з TTL = залишок часу токена.
    При кожному запиті: перевіряємо чи JTI не в blacklist.
    
    JTI — унікальний ідентифікатор токена (claims["jti"]).
    """

    KEY_PREFIX = "token_blacklist"

    def _make_key(self, jti: str) -> str:
        return f"{self.KEY_PREFIX}:{jti}"

    async def revoke(self, jti: str, remaining_ttl: int) -> None:
        """
        Додати токен в blacklist.
        
        jti: JWT ID з payload токена
        remaining_ttl: скільки секунд залишилось у токена
                       (після закінчення токен і так не буде валідний)
        """
        if remaining_ttl <= 0:
            return  # Токен вже прострочений, не треба додавати

        # Зберігаємо "1" як значення (нам важлива тільки наявність ключа)
        await self.set(self._make_key(jti), "1", ttl=remaining_ttl)

    async def is_revoked(self, jti: str) -> bool:
        """Перевірити чи токен відкликаний."""
        return await self.exists(self._make_key(jti))

    async def revoke_all_user_tokens(
        self,
        user_id: int,
        token_ttl: int,
    ) -> None:
        """
        Відкликати всі токени юзера через 'generation' підхід.
        
        Замість списку всіх токенів — зберігаємо "token generation".
        При кожному запиті перевіряємо що generation у токені
        збігається з поточним generation юзера.
        """
        key = f"token_gen:{user_id}"
        await self.redis.incr(key)
        await self.expire(key, token_ttl)
```

```python
# app/core/jwt.py
from datetime import datetime, timedelta, UTC
import uuid
import jwt as pyjwt
from app.core.config import settings
from app.repositories.token_blacklist_repository import TokenBlacklistRepository
import redis.asyncio as aioredis


def create_access_token(user_id: int) -> tuple[str, str]:
    """
    Створити JWT токен.
    Повертає (token, jti) — jti потрібен для blacklist.
    """
    jti = str(uuid.uuid4())   # унікальний ID токена
    now = datetime.now(UTC)
    exp = now + timedelta(seconds=settings.ACCESS_TOKEN_TTL)

    payload = {
        "sub": str(user_id),
        "jti": jti,           # JWT ID для blacklist
        "iat": now,
        "exp": exp,
    }

    token = pyjwt.encode(payload, settings.SECRET_KEY, algorithm=settings.JWT_ALGORITHM)
    return token, jti


async def verify_token(
    token: str,
    redis: aioredis.Redis,
) -> dict:
    """
    Перевірити токен: підпис, термін дії, і чи не в blacklist.
    """
    try:
        payload = pyjwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.JWT_ALGORITHM],
        )
    except pyjwt.ExpiredSignatureError:
        raise ValueError("Token expired")
    except pyjwt.InvalidTokenError:
        raise ValueError("Invalid token")

    # Перевірити blacklist
    jti = payload.get("jti")
    if jti:
        repo = TokenBlacklistRepository(redis)
        if await repo.is_revoked(jti):
            raise ValueError("Token has been revoked")

    return payload
```

---

## 14. Pipeline у FastAPI

Pipeline дозволяє відправити кілька команд одним TCP пакетом.

```python
# app/routers/stats.py
from fastapi import APIRouter
from app.dependencies.redis import RedisDep

router = APIRouter()


@router.get("/dashboard/{user_id}")
async def get_dashboard(user_id: int, redis: RedisDep):
    """
    Отримати всі дані дашборду одним pipeline.
    Без pipeline: 5 round-trips → ~1-2мс кожен → ~5-10мс загалом
    З pipeline:   1 round-trip → ~1-2мс загалом
    """

    # Ключі для отримання
    keys = {
        "profile": f"cache:user:{user_id}:profile",
        "stats": f"cache:user:{user_id}:stats",
        "notifications": f"cache:user:{user_id}:notifications",
        "level": f"game:user:{user_id}:level",
        "xp": f"game:user:{user_id}:xp",
    }

    # Один pipeline замість 5 окремих GET
    async with redis.pipeline(transaction=False) as pipe:
        # transaction=False = просто batching, не транзакція
        for key in keys.values():
            pipe.get(key)
        results = await pipe.execute()

    # Зіставити результати з ключами
    data = {}
    for (field, _), value in zip(keys.items(), results):
        data[field] = value

    return data


@router.post("/bulk-update")
async def bulk_update(updates: list[dict], redis: RedisDep):
    """Масове оновлення через pipeline."""
    async with redis.pipeline(transaction=False) as pipe:
        for update in updates:
            pipe.set(
                f"cache:{update['key']}",
                update['value'],
                ex=300,
            )
        results = await pipe.execute()

    return {"updated": sum(1 for r in results if r)}
```

### Pipeline з транзакцією (MULTI/EXEC)

```python
@router.post("/transfer")
async def transfer_xp(
    from_user: int,
    to_user: int,
    amount: int,
    redis: RedisDep,
):
    """
    Перенести XP між юзерами атомарно.
    Використовуємо WATCH + MULTI/EXEC для атомарності.
    """
    from_key = f"game:user:{from_user}:xp"
    to_key = f"game:user:{to_user}:xp"

    async with redis.pipeline(transaction=True) as pipe:
        while True:
            try:
                # Стежити за змінами
                await pipe.watch(from_key, to_key)

                # Перевірити баланс
                current_xp = await pipe.get(from_key)
                if current_xp is None or int(current_xp) < amount:
                    raise HTTPException(400, "Insufficient XP")

                # Почати транзакцію
                pipe.multi()
                pipe.decrby(from_key, amount)
                pipe.incrby(to_key, amount)

                await pipe.execute()
                break  # успіх

            except Exception as e:
                if "WATCH" in str(e):
                    continue  # ключ змінився, повторити
                raise

    return {"transferred": amount}
```

---

## 15. Pub/Sub у FastAPI

```python
# app/services/notification_service.py
import json
import asyncio
import redis.asyncio as aioredis
from app.core.redis import get_redis


class NotificationService:
    """Сервіс сповіщень через Redis Pub/Sub."""

    CHANNEL_PREFIX = "notifications"

    def user_channel(self, user_id: int) -> str:
        return f"{self.CHANNEL_PREFIX}:user:{user_id}"

    def broadcast_channel(self) -> str:
        return f"{self.CHANNEL_PREFIX}:broadcast"

    async def send_to_user(
        self,
        redis: aioredis.Redis,
        user_id: int,
        event_type: str,
        data: dict,
    ) -> int:
        """Надіслати сповіщення конкретному юзеру."""
        message = json.dumps({
            "type": event_type,
            "data": data,
        })
        return await redis.publish(self.user_channel(user_id), message)

    async def broadcast(
        self,
        redis: aioredis.Redis,
        event_type: str,
        data: dict,
    ) -> int:
        """Надіслати всім підписникам."""
        message = json.dumps({"type": event_type, "data": data})
        return await redis.publish(self.broadcast_channel(), message)
```

### SSE — Server-Sent Events через Pub/Sub

```python
# app/routers/notifications.py
import json
import asyncio
import redis.asyncio as aioredis
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
from app.dependencies.redis import RedisDep
from app.core.redis import get_redis

router = APIRouter(prefix="/notifications", tags=["notifications"])


@router.get("/stream/{user_id}")
async def notification_stream(user_id: int, redis: RedisDep):
    """
    Server-Sent Events endpoint.
    Клієнт підключається і отримує сповіщення в реальному часі.
    
    Використання на фронтенді:
        const events = new EventSource('/notifications/stream/42');
        events.onmessage = (e) => console.log(JSON.parse(e.data));
    """

    async def event_generator():
        # Підписуємось на канали
        pubsub = redis.pubsub()
        await pubsub.subscribe(
            f"notifications:user:{user_id}",
            "notifications:broadcast",
        )

        try:
            async for message in pubsub.listen():
                if message["type"] == "message":
                    # SSE формат: "data: {...}\n\n"
                    yield f"data: {message['data']}\n\n"

                # Heartbeat кожні 30с щоб тримати з'єднання
                # (в реальному коді треба додати asyncio.sleep + check)

        except asyncio.CancelledError:
            # Клієнт відключився
            pass
        finally:
            await pubsub.unsubscribe()
            await pubsub.aclose()

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",   # для nginx
        },
    )
```

---

## 16. Тестування з Redis

```python
# tests/conftest.py
import pytest
import pytest_asyncio
import redis.asyncio as aioredis
from fastapi.testclient import TestClient
from httpx import AsyncClient, ASGITransport
from app.main import app
import app.core.redis as redis_module


@pytest_asyncio.fixture
async def redis_client():
    """
    Реальний Redis для тестів.
    Використовуємо окрему БД (db=1) щоб не засмічувати продакшн (db=0).
    """
    client = aioredis.Redis(
        host="localhost",
        port=6379,
        db=1,             # ← окрема БД для тестів!
        decode_responses=True,
    )
    yield client

    # Cleanup: видалити всі ключі після тесту
    await client.flushdb()
    await client.aclose()


@pytest_asyncio.fixture
async def app_with_redis(redis_client):
    """
    Підмінити Redis в додатку на тестовий клієнт.
    """
    # Підмінити пул
    redis_module._pool = redis_client.connection_pool

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        yield client

    redis_module._pool = None


# Тести
@pytest.mark.asyncio
async def test_oauth_state_flow(redis_client):
    """Тест створення і валідації OAuth state."""
    from app.repositories.oauth_state_repository import OAuthStateRepository

    repo = OAuthStateRepository(redis_client)

    # Створити state
    state = await repo.create(provider="google", redirect_after="/dashboard")
    assert len(state) > 30  # достатньо довгий

    # Перевірити що існує в Redis
    data = await repo.peek(state)
    assert data is not None
    assert data.provider == "google"
    assert data.redirect_after == "/dashboard"

    # Validate and consume
    consumed = await repo.validate_and_consume(state)
    assert consumed is not None
    assert consumed.provider == "google"

    # Повторна спроба — має повернути None
    again = await repo.validate_and_consume(state)
    assert again is None


@pytest.mark.asyncio
async def test_rate_limiting(redis_client):
    """Тест rate limiting."""
    from app.repositories.rate_limit_repository import RateLimitRepository

    repo = RateLimitRepository(redis_client)

    # Перші 5 запитів — дозволені
    for i in range(5):
        result = await repo.check("test_user", limit=5, window=60)
        assert result.allowed is True
        assert result.remaining == 4 - i

    # 6-й запит — заблокований
    result = await repo.check("test_user", limit=5, window=60)
    assert result.allowed is False
    assert result.remaining == 0
    assert result.retry_after > 0


@pytest.mark.asyncio
async def test_cache_repository(redis_client):
    """Тест кешування."""
    from app.repositories.cache_repository import CacheRepository

    cache = CacheRepository(redis_client)
    key = "test:user:42:profile"

    # Кеш порожній
    assert await cache.get_json(key) is None

    # Зберегти
    data = {"id": 42, "name": "Daniil"}
    await cache.set_json(key, data, ttl=60)

    # Отримати
    cached = await cache.get_json(key)
    assert cached == data

    # Інвалідувати
    await cache.invalidate(key)
    assert await cache.get_json(key) is None
```

```bash
# pyproject.toml — залежності для тестів
[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "httpx>=0.27",
]

# pytest.ini або pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

---

## 17. Docker Compose — повна конфігурація

```yaml
# docker-compose.yml
version: "3.9"

services:

  # ─── FastAPI додаток ──────────────────────────────────────────
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - APP_ENV=development
      - REDIS_URL=redis://redis:6379/0
      - DATABASE_URL=postgresql+asyncpg://postgres:postgres@db:5432/health_patch
    env_file:
      - .env.docker
    depends_on:
      redis:
        condition: service_healthy
      db:
        condition: service_healthy
    volumes:
      - .:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    networks:
      - backend

  # ─── Redis ───────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"   # для локальної розробки та redis-cli
    volumes:
      - redis_data:/data
      - ./redis.conf:/etc/redis/redis.conf   # кастомна конфігурація
    command: redis-server /etc/redis/redis.conf
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s
    networks:
      - backend

  # ─── PostgreSQL ──────────────────────────────────────────────
  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: health_patch
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - backend

volumes:
  redis_data:
  pg_data:

networks:
  backend:
    driver: bridge
```

```ini
# redis.conf (для Docker)
bind 0.0.0.0
port 6379

# Без пароля для локальної розробки
# requirepass your_password   # розкоментувати для prod

# Persistence
appendonly yes
appendfsync everysec

# Пам'ять
maxmemory 256mb
maxmemory-policy allkeys-lru

# Логування
loglevel notice

# Slowlog
slowlog-log-slower-than 10000
slowlog-max-len 128
```

---

## 18. Структура проєкту

```
app/
│
├── main.py                      # FastAPI app + lifespan
│
├── core/
│   ├── config.py                # Settings (BaseSettings)
│   └── redis.py                 # ConnectionPool, get_redis()
│
├── dependencies/
│   ├── redis.py                 # RedisDep = Annotated[Redis, Depends(...)]
│   └── rate_limit.py            # LoginRateLimit, ApiRateLimit
│
├── repositories/
│   ├── base_redis.py            # BaseRedisRepository (базові операції)
│   ├── oauth_state_repository.py # OAuth CSRF state
│   ├── session_repository.py    # Серверні сесії
│   ├── cache_repository.py      # Кешування
│   ├── rate_limit_repository.py # Rate limiting (Sliding Window)
│   ├── lock_repository.py       # Distributed Lock
│   └── token_blacklist_repository.py  # JWT blacklist
│
├── routers/
│   └── auth/
│       ├── google.py            # /auth/google/login + /callback
│       ├── github.py            # /auth/github/login + /callback
│       └── sessions.py          # /auth/login, /logout, /logout-all
│
├── services/
│   └── notification_service.py  # Pub/Sub сповіщення
│
├── middleware/
│   └── rate_limit.py            # Глобальний RateLimit Middleware
│
└── utils/
    └── cache_decorator.py       # @cached декоратор

tests/
├── conftest.py                  # pytest fixtures (redis_client, app)
├── test_oauth_state.py
├── test_rate_limit.py
└── test_cache.py

docker-compose.yml
redis.conf
.env
.env.docker
```

---

## 19. Типові помилки та антипатерни

### ❌ Синхронний клієнт в async коді

```python
# НЕПРАВИЛЬНО — блокує весь event loop
import redis
r = redis.Redis()

@router.get("/")
async def handler():
    r.set("key", "val")   # ← БЛОКУЄ! Нікакий інший запит не пройде

# ПРАВИЛЬНО
import redis.asyncio as aioredis
```

---

### ❌ Нове з'єднання на кожен запит

```python
# НЕПРАВИЛЬНО — тисячі з'єднань = Redis падає
@router.get("/")
async def handler():
    r = aioredis.Redis(host="localhost")   # ← нове з'єднання кожного разу!
    ...
    await r.aclose()

# ПРАВИЛЬНО — ConnectionPool через Depends
async def handler(redis: RedisDep):
    ...
```

---

### ❌ Відсутній TTL у тимчасових ключів

```python
# НЕПРАВИЛЬНО — накопичуються вічно
await redis.set("oauth_state:abc", "data")        # без TTL!
await redis.set("password_reset:xyz", user_id)    # без TTL!

# ПРАВИЛЬНО
await redis.set("oauth_state:abc", "data", ex=600)
await redis.set("password_reset:xyz", user_id, ex=900)
```

---

### ❌ GET + DEL замість GETDEL (race condition)

```python
# НЕПРАВИЛЬНО — між GET і DEL може зайти другий запит
value = await redis.get("oauth_state:abc")
if value:
    await redis.delete("oauth_state:abc")
    process(value)
# Проблема: два паралельних запити з однаковим state — обидва пройдуть!

# ПРАВИЛЬНО — атомарна операція
value = await redis.getdel("oauth_state:abc")
if value:
    process(value)
```

---

### ❌ KEYS * в production

```python
# НЕПРАВИЛЬНО — блокує Redis на секунди при мільйонах ключів
keys = await redis.keys("user:*")

# ПРАВИЛЬНО — ітеративний SCAN
async for key in redis.scan_iter(match="user:*", count=100):
    ...
```

---

### ❌ Не закривати PubSub з'єднання

```python
# НЕПРАВИЛЬНО — з'єднання залишається відкритим
pubsub = redis.pubsub()
await pubsub.subscribe("channel")
# якщо виключення — з'єднання не закриється

# ПРАВИЛЬНО — try/finally
pubsub = redis.pubsub()
await pubsub.subscribe("channel")
try:
    async for message in pubsub.listen():
        ...
finally:
    await pubsub.unsubscribe()
    await pubsub.aclose()
```

---

### ❌ Зберігати великі об'єкти без думки про пам'ять

```python
# НЕПРАВИЛЬНО — кешувати весь список з 10,000 записів
all_users = await db.get_all_users()  # 10,000 юзерів
await redis.set("cache:all_users", json.dumps(all_users))
# 10,000 × ~200 байт = 2 MB в Redis × 100 запитів = 200 MB

# ПРАВИЛЬНО — кешувати з пагінацією
await redis.set("cache:users:page:1", json.dumps(users_page_1), ex=300)
await redis.set("cache:users:page:2", json.dumps(users_page_2), ex=300)
```

---

### ❌ Використовувати одну БД для тестів і prod

```python
# НЕПРАВИЛЬНО — тести видаляють продакшн дані!
r = aioredis.Redis(host="localhost", port=6379, db=0)  # db=0 = prod!
await r.flushdb()  # видалив всі продакшн ключі!

# ПРАВИЛЬНО — окрема БД для тестів
r_test = aioredis.Redis(host="localhost", port=6379, db=1)  # db=1 = tests
```

---

### ❌ Не обробляти ConnectionError

```python
# НЕПРАВИЛЬНО — при падінні Redis весь додаток падає
@router.get("/profile/{id}")
async def profile(id: int, redis: RedisDep):
    cached = await redis.get(f"user:{id}")   # ConnectionError якщо Redis впав

# ПРАВИЛЬНО — graceful degradation
@router.get("/profile/{id}")
async def profile(id: int, redis: RedisDep):
    try:
        cached = await redis.get(f"user:{id}")
        if cached:
            return json.loads(cached)
    except Exception:
        pass  # Redis недоступний — йдемо в БД
    
    # Fallback до БД
    return await db.get_user(id)
```

---

*Цей документ описує повну production-ready інтеграцію Redis у FastAPI: від ConnectionPool до OAuth state, сесій, кешування та rate limiting з правильними патернами і без типових помилок.*

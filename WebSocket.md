# 4. WebSockets у FastAPI

## Що таке WebSocket?

**HTTP** — це як листування поштою: клієнт надсилає запит → сервер відповідає → з'єднання закривається.

**WebSocket** — це як телефонний дзвінок: з'єднання відкривається **один раз** і обидві сторони можуть говорити **в будь-який момент**.

```
── HTTP (Request-Response) ──

Клієнт                 Сервер
  │── GET /data ─────→ │
  │← { "value": 1 } ──│       ← з'єднання закрилось
  │                    │
  │── GET /data ────���→ │
  │← { "value": 2 } ──│       ← знову закрилось
  │                    │
  (кожен запит — нове з'єднання)


── WebSocket (Bidirectional) ──

Клієнт                 Сервер
  │── Handshake ─────→ │
  │← Accepted ─────────│       ← з'єднання ВІДКРИТЕ
  │                    │
  │── "hello" ───────→ │       ← клієнт може слати будь-коли
  │← "world" ──────────│       ← сервер може слати будь-коли
  │← "update: ..." ───│       ← сервер може слати БЕЗ запиту
  │── "bye" ─────────→ │
  │                    │
  (одне з'єднання весь час)
```

---

## Коли використовувати WebSocket?

| Задача | HTTP чи WebSocket? | Чому? |
|---|---|---|
| Отримати список юзерів | ✅ HTTP | Один запит — одна відповідь |
| Чат в реальному часі | ✅ WebSocket | Повідомлення приходять миттєво |
| Форма реєстрації | ✅ HTTP | Стандартний POST запит |
| Live-дашборд з метриками | ✅ WebSocket | Дані оновлюються кожну секунду |
| Сповіщення (notifications) | ✅ WebSocket | Сервер пушить без запиту клієнта |
| Завантаження файлу | ✅ HTTP | Один запит — один файл |
| Онлайн-гра | ✅ WebSocket | Потрібна мінімальна затримка |
| Стрімінг даних (ETL pipeline) | ✅ WebSocket | Потік даних в реальному часі |

---

## Базовий приклад: Echo-сервер

Найпростіший WebSocket — отримує повідомлення і відправляє його назад:

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws/echo")
async def echo(websocket: WebSocket):
    # Крок 1: Прийняти з'єднання (handshake)
    await websocket.accept()

    try:
        # Крок 2: Нескінченний цикл — чекаємо повідомлення
        while True:
            data = await websocket.receive_text()        # ← чекаємо від клієнта
            await websocket.send_text(f"Echo: {data}")   # ← відправляємо назад
    except WebSocketDisconnect:
        # Крок 3: Клієнт закрив вкладку / втратив з'єднання
        print("Client disconnected")
```

### Що тут відбувається покроково:

```
1. Клієнт підключається   →  websocket.accept()     ← "Привіт, я приймаю з'єднання"
2. Клієнт надсилає "hi"   →  websocket.receive_text() повертає "hi"
3. Сервер відповідає       →  websocket.send_text("Echo: hi")
4. Повтор кроків 2-3       →  while True
5. Клієнт від'єднується    →  WebSocketDisconnect → виходимо з циклу
```

### Типи даних, які можна надсилати:

```python
# Текст
data = await websocket.receive_text()
await websocket.send_text("hello")

# JSON (автоматична серіалізація/десеріалізація)
data = await websocket.receive_json()       # ← отримати dict
await websocket.send_json({"key": "value"}) # ← надіслати dict

# Бінарні дані (файли, зображення)
data = await websocket.receive_bytes()
await websocket.send_bytes(b"\x00\x01\x02")
```

---

## Чат: Менеджер з'єднань (ConnectionManager)

Проблема: Echo працює з **одним** клієнтом. Для чату потрібно надсилати повідомлення **всім**.

### Навіщо пот��ібен ConnectionManager?

```
Без менеджера:                   З менеджером:

Клієнт A ←→ Сервер               Клієнт A ←→ ┐
Клієнт B ←→ Сервер               Клієнт B ←→ ├── ConnectionManager ←→ Сервер
                                  Клієнт C ←→ ┘
A не знає про B                   A надіслав → всі отримали (broadcast)
```

### Реалізація:

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from datetime import datetime

app = FastAPI()


class ConnectionManager:
    """Керує WebSocket-з'єднаннями всіх клієнтів."""

    def __init__(self):
        # Список ВСІХ активних з'єднань
        self.active: list[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        """Нове з'єднання: прийняти + додати до списку."""
        await websocket.accept()
        self.active.append(websocket)

    def disconnect(self, websocket: WebSocket):
        """Клієнт від'єднався: прибрати зі списку."""
        self.active.remove(websocket)

    async def broadcast(self, message: str):
        """Надіслати повідомлення УСІМ підключеним клієнтам."""
        for ws in self.active:
            await ws.send_text(message)

    async def send_personal(self, websocket: WebSocket, message: str):
        """Надіслати повідомлення ОДНОМУ клієнту."""
        await websocket.send_text(message)


# Один менеджер на весь додаток (глобальний стан)
manager = ConnectionManager()
```

### Ендпоінт чату:

```python
@app.websocket("/ws/chat")
async def chat(websocket: WebSocket):
    # 1. Новий клієнт підключився
    await manager.connect(websocket)

    try:
        while True:
            # 2. Чекаємо повідомлення від цього клієнта
            data = await websocket.receive_text()

            # 3. Додаємо час і відправляємо УСІМ
            timestamp = datetime.now().strftime("%H:%M:%S")
            await manager.broadcast(f"[{timestamp}] {data}")

    except WebSocketDisconnect:
        # 4. Клієнт від'єднався — прибрати і повідомити інших
        manager.disconnect(websocket)
        await manager.broadcast("A user left the chat")
```

### Як це працює з трьома клієнтами:

```
Клієнт A підключився     →  active: [A]
Клієнт B підключився     →  active: [A, B]
Клієнт C підключився     →  active: [A, B, C]

Клієнт A надсилає "Привіт"
  → broadcast("[10:30:15] Привіт")
  → A отримає "[10:30:15] Привіт"
  → B отримає "[10:30:15] Привіт"
  → C отримає "[10:30:15] Привіт"

Клієнт B закриває вкладку
  → disconnect(B)         →  active: [A, C]
  → broadcast("A user left the chat")
  → A отримає "A user left the chat"
  → C отримає "A user left the chat"
```

---

## Data Engineering: Live метрики

Реальний кейс — дашборд, який показує метрики сервера в реальному часі:

```python
import asyncio
import random
from datetime import datetime

@app.websocket("/ws/metrics")
async def live_metrics(websocket: WebSocket):
    """Стрімінг метрик на дашборд кожну секунду."""
    await websocket.accept()

    try:
        while True:
            # Генеруємо метрики (в реальності — з бази/моніторингу)
            metrics = {
                "timestamp": datetime.now().isoformat(),
                "cpu_usage": round(random.uniform(10, 90), 1),
                "memory_mb": random.randint(500, 4000),
                "requests_per_sec": random.randint(50, 500),
            }

            # Надсилаємо JSON клієнту
            await websocket.send_json(metrics)

            # Чекаємо 1 секунду до наступного оновлення
            await asyncio.sleep(1)

    except WebSocketDisconnect:
        print("Dashboard disconnected")
```

### Що отримує клієнт кожну секунду:

```json
{"timestamp": "2026-03-06T14:30:01", "cpu_usage": 45.3, "memory_mb": 2100, "requests_per_sec": 230}
{"timestamp": "2026-03-06T14:30:02", "cpu_usage": 52.1, "memory_mb": 2150, "requests_per_sec": 245}
{"timestamp": "2026-03-06T14:30:03", "cpu_usage": 38.7, "memory_mb": 2080, "requests_per_sec": 210}
...
```

### Чому WebSocket, а не HTTP polling?

```
── HTTP Polling (погано для live даних) ──

Клієнт кожну секунду:
  GET /metrics → відповідь → закрити з'єднання
  GET /metrics → відповідь → закрити з'єднання
  GET /metrics → відповідь → закрити з'єднання
  (кожен запит — новий handshake, нові заголовки, overhead)


── WebSocket (ефективно) ──

Клієнт один раз:
  connect → дані → дані → дані → дані → ...
  (одне з'єднання, мінімальний overhead)
```

| Метрика | HTTP Polling (1/сек) | WebSocket |
|---|---|---|
| З'єднань за хвилину | 60 | 1 |
| Затримка | ~100-500ms | ~1-5ms |
| Трафік (заголовки) | ~60KB/хв | ~2KB/хв |

---

## Тестування WebSockets

FastAPI надає зручний тестовий клієнт:

```python
from fastapi.testclient import TestClient

client = TestClient(app)

def test_websocket_echo():
    # websocket_connect відкриває WebSocket-з'єднання
    with client.websocket_connect("/ws/echo") as ws:
        ws.send_text("hello")           # надіслати
        data = ws.receive_text()         # отримати відповідь
        assert data == "Echo: hello"

def test_websocket_json():
    with client.websocket_connect("/ws/metrics") as ws:
        data = ws.receive_json()         # отримати JSON
        assert "cpu_usage" in data
        assert "timestamp" in data

def test_websocket_disconnect():
    with client.websocket_connect("/ws/echo") as ws:
        ws.send_text("test")
        ws.receive_text()
    # Вийшли з `with` → з'єднання автоматично закрилося
```

---

## Клієнтський код (JavaScript)

Щоб підключитися до WebSocket з браузера:

```javascript
// Підключення до echo
const ws = new WebSocket("ws://localhost:8000/ws/echo");

// Коли з'єднання відкрите
ws.onopen = () => {
    console.log("Connected!");
    ws.send("Привіт, сервер!");
};

// Коли отримали повідомлення
ws.onmessage = (event) => {
    console.log("Отримано:", event.data);
    // "Echo: Привіт, сервер!"
};

// Коли з'єднання закрилось
ws.onclose = () => {
    console.log("Disconnected");
};

// Коли помилка
ws.onerror = (error) => {
    console.error("WebSocket error:", error);
};
```

### Для live метрик (JSON):

```javascript
const ws = new WebSocket("ws://localhost:8000/ws/metrics");

ws.onmessage = (event) => {
    const metrics = JSON.parse(event.data);
    // metrics = { timestamp: "...", cpu_usage: 45.3, ... }

    document.getElementById("cpu").textContent = metrics.cpu_usage + "%";
    document.getElementById("memory").textContent = metrics.memory_mb + " MB";
};
```

---

## Повний life-cycle WebSocket-з'єднання

```
Клієнт (браузер)                          Сервер (FastAPI)
      │                                        │
      │── HTTP GET /ws/echo ─────────────────→ │  ← Звичайний HTTP запит
      │   Headers:                             │
      │     Upgrade: websocket                 │  ← "Хочу перейти на WebSocket"
      │     Connection: Upgrade                │
      │                                        │
      │← HTTP 101 Switching Protocols ────────│  ← websocket.accept()
      │                                        │
      │ ═══════ WebSocket з'єднання ═════════ │
      │                                        │
      │── text: "hello" ────────────────────→  │  ← receive_text()
      │← text: "Echo: hello" ─────────────────│  ← send_text()
      │                                        │
      │── text: "world" ────────────────────→  │  ← receive_text()
      │← text: "Echo: world" ─────────────────│  ← send_text()
      │                                        │
      │── close frame ──────────────────────→  │  ← WebSocketDisconnect
      │← close frame ────────────────���────────│
      │                                        │
      │ ═══════ З'єднання закрите ═══════════ │
```

---

## Важливі нюанси

### 1. WebSocket не масштабується просто так

```python
# ❌ ПРОБЛЕМА: ConnectionManager зберігає з'єднання в пам'яті
# Якщо у вас 2 сервери за балансувальником — вони НЕ бачать з'єднання один одного

Сервер 1: active = [Клієнт A, Клієнт B]
Сервер 2: active = [Клієнт C]
# Клієнт A надсилає → broadcast → тільки A і B отримають, C — ні!

# ✅ РІШЕННЯ: використовувати Redis Pub/Sub або інший брокер
# pip install redis
```

### 2. Автентифікація в WebSocket

```python
# WebSocket НЕ підтримує заголовки я�� HTTP
# Токен передається через query-параметр

@app.websocket("/ws/chat")
async def chat(websocket: WebSocket, token: str = Query(...)):
    # Валідація токена
    try:
        user = decode_token(token)
    except Exception:
        await websocket.close(code=1008)  # Policy Violation
        return

    await websocket.accept()
    ...
```

```javascript
// Клієнт передає токен в URL
const ws = new WebSocket("ws://localhost:8000/ws/chat?token=eyJhbGci...");
```

### 3. Heartbeat / Ping-Pong

```python
# З'єднання може "зависнути" без реального розриву
# Потрібен heartbeat для перевірки

@app.websocket("/ws/reliable")
async def reliable(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            # Якщо клієнт не відповідає 30 сек — він "мертвий"
            data = await asyncio.wait_for(
                websocket.receive_text(),
                timeout=30.0
            )
            await websocket.send_text(f"Got: {data}")
    except asyncio.TimeoutError:
        print("Client timed out")
        await websocket.close()
    except WebSocketDisconnect:
        print("Client disconnected")
```

---

## Коротко

```
WebSocket = постійне двостороннє з'єднання
  ├── websocket.accept()         → відкрити з'єднання
  ├── websocket.receive_text()   → отримати від клієнта
  ├── websocket.send_text()      → надіслати клієнту
  ├── websocket.send_json()      → надіслати JSON
  ├── WebSocketDisconnect        → клієнт від'єднався
  └── ConnectionManager          → broadcast усім клієнтам
```

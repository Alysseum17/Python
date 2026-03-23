# Redis — Повний та Детальний Гайд

> Від того як Redis влаштований всередині до кожного методу кожної структури даних

---

## Зміст

1. [Що таке Redis — глибоко](#1-що-таке-redis--глибоко)
2. [Як Redis працює всередині](#2-як-redis-працює-всередині)
3. [Встановлення та CLI](#3-встановлення-та-cli)
4. [Неймінг ключів — конвенції](#4-неймінг-ключів--конвенції)
5. [Глобальні команди для ключів](#5-глобальні-команди-для-ключів)
6. [String — рядки](#6-string--рядки)
7. [Integer та Float через String](#7-integer-та-float-через-string)
8. [Hash — словники](#8-hash--словники)
9. [List — черги та стеки](#9-list--черги-та-стеки)
10. [Set — множини](#10-set--множини)
11. [Sorted Set — відсортовані множини](#11-sorted-set--відсортовані-множини)
12. [TTL та Expiration — детально](#12-ttl-та-expiration--детально)
13. [Transactions — MULTI/EXEC](#13-transactions--multiexec)
14. [Lua Scripts](#14-lua-scripts)
15. [Pub/Sub — публікація і підписка](#15-pubsub--публікація-і-підписка)
16. [Persistence — як Redis зберігає дані](#16-persistence--як-redis-зберігає-дані)
17. [Eviction Policies — що робити коли закінчується пам'ять](#17-eviction-policies--що-робити-коли-закінчується-память)
18. [Redis Databases — числові БД](#18-redis-databases--числові-бд)
19. [Конфігурація redis.conf](#19-конфігурація-redisconf)
20. [Патерни використання Redis](#20-патерни-використання-redis)

---

## 1. Що таке Redis — глибоко

### Офіційне визначення vs реальність

Офіційно Redis — це **Remote Dictionary Server**. Але насправді це набагато більше ніж просто словник.

Redis — це **in-memory data structure store**. Ключове слово: **структури даних**. Redis не просто зберігає рядки — він розуміє типи даних і надає операції специфічні для кожного типу.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Redis — це не просто кеш                    │
│                                                                     │
│  ✅ In-memory база даних          (з опціональним збереженням)      │
│  ✅ Кеш                           (найпоширеніший юзкейс)           │
│  ✅ Message broker                (Pub/Sub, Streams)                │
│  ✅ Session store                 (OAuth state, сесії юзерів)       │
│  ✅ Rate limiter                  (лічильники з TTL)                │
│  ✅ Leaderboard engine            (Sorted Sets)                     │
│  ✅ Task queue                    (Lists як черги)                  │
│  ✅ Distributed lock              (SET NX)                          │
└─────────────────────────────────────────────────────────────────────┘
```

### Чому Redis такий швидкий

**1. Все в RAM**

Оперативна пам'ять (RAM) vs SSD vs HDD:

```
RAM:  ~100 наносекунд  (0.0001 мс)
SSD:  ~100 мікросекунд (0.1 мс)
HDD:  ~10 мілісекунд   (10 мс)

Тобто RAM в ~1000x швидша за SSD і ~100,000x швидша за HDD
```

**2. Однопоточна модель з event loop**

Redis обробляє запити в **один потік**. Це звучить як мінус, але насправді це перевага:

- Немає race conditions між потоками
- Немає витрат на синхронізацію (mutex, locks)
- Порядок виконання команд завжди передбачуваний

```
Клієнт 1: SET foo 1  ──┐
Клієнт 2: SET bar 2  ──┤──→ [Черга запитів] ──→ [Один потік Redis] ──→ відповіді
Клієнт 3: GET foo    ──┘
```

> Один потік обробляє ~1,000,000 операцій на секунду. Для порівняння: PostgreSQL з пулом потоків — ~50,000-100,000 simple queries/s.

**3. Прості структури даних**

Redis не парсить SQL, не будує execution plans, не join-ить таблиці. Кожна операція — це пряма маніпуляція зі структурою в пам'яті.

**4. Non-blocking I/O**

Redis використовує `epoll`/`kqueue` (залежно від ОС) — механізм, який дозволяє одному потоку одночасно "слухати" тисячі клієнтів без блокування.

---

## 2. Як Redis працює всередині

### Архітектура

```
┌─────────────────────────────────────────────────────────────────┐
│                        Redis Server                             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Network Layer                        │   │
│  │  TCP port 6379  │  Unix socket  │  TLS (опціонально)   │   │
│  └─────────────────────────┬───────────────────────────────┘   │
│                            │                                   │
│  ┌─────────────────────────▼───────────────────────────────┐   │
│  │                   Event Loop (ae.c)                     │   │
│  │           epoll / kqueue / select                       │   │
│  └─────────────────────────┬───────────────────────────────┘   │
│                            │                                   │
│  ┌─────────────────────────▼───────────────────────────────┐   │
│  │                  Command Processor                      │   │
│  │    Парсинг RESP протоколу → виконання команди           │   │
│  └─────────────────────────┬───────────────────────────────┘   │
│                            │                                   │
│  ┌─────────────────────────▼───────────────────────────────┐   │
│  │                   Data Structures                       │   │
│  │   String │ Hash │ List │ Set │ Sorted Set │ Stream      │   │
│  └─────────────────────────┬───────────────────────────────┘   │
│                            │                                   │
│  ┌─────────────────────────▼───────────────────────────────┐   │
│  │                    Persistence                          │   │
│  │            RDB (snapshot) │ AOF (append-only)           │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### RESP — Redis Serialization Protocol

Коли ти пишеш `SET foo bar` в redis-cli, він кодує це в **RESP протокол**:

```
*3\r\n          ← масив з 3 елементів
$3\r\n          ← bulk string довжиною 3
SET\r\n
$3\r\n          ← bulk string довжиною 3
foo\r\n
$3\r\n          ← bulk string довжиною 3
bar\r\n
```

Redis відповідає:
```
+OK\r\n         ← Simple String
```

Типи відповідей:
```
+OK             → Simple String (success)
-ERR message   → Error
:42             → Integer
$6\r\nfoobar   → Bulk String
*2\r\n...       → Array
$-1             → Null (ключ не існує)
```

Знати це не обов'язково для роботи з Redis, але пояснює чому Redis такий швидкий — протокол мінімалістичний і легко парсується.

### Keyspace — як Redis зберігає ключі

Всередині Redis ключі зберігаються в **hash table** (словник у C). Кожен запис:

```c
typedef struct dictEntry {
    void *key;           // ключ (завжди SDS рядок)
    union {
        void *val;       // вказівник на значення (тип залежить від структури)
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; // для collision chaining
} dictEntry;
```

**SDS (Simple Dynamic Strings)** — власний формат рядків Redis:

```c
struct sdshdr {
    int len;     // поточна довжина
    int free;    // вільне місце (pre-allocated)
    char buf[];  // самі дані + '\0'
};
```

Перевага SDS над C-рядками:
- `O(1)` для отримання довжини (замість `O(n)` strlen)
- Безпечний для бінарних даних (може містити `\0`)
- Pre-allocation зменшує кількість realloc

---

## 3. Встановлення та CLI

### Встановлення

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install redis-server

# macOS
brew install redis

# Перевірити версію
redis-server --version
# Redis server v=7.2.x

# Запустити як сервіс
sudo systemctl start redis-server
sudo systemctl enable redis-server  # автозапуск при ребуті
sudo systemctl status redis-server  # перевірити статус
```

### Docker (найкращий варіант для розробки)

```bash
# Простий запуск
docker run -d --name redis -p 6379:6379 redis:7-alpine

# З persistence
docker run -d \
  --name redis \
  -p 6379:6379 \
  -v $(pwd)/redis-data:/data \
  redis:7-alpine \
  redis-server --appendonly yes

# Підключитись до Redis всередині контейнера
docker exec -it redis redis-cli
```

### Redis CLI — інтерфейс командного рядка

```bash
# Підключитись до локального Redis
redis-cli

# Підключитись до конкретного хосту/порту
redis-cli -h 192.168.1.10 -p 6379

# З паролем
redis-cli -a your_password

# Виконати одну команду і вийти
redis-cli SET foo bar
redis-cli GET foo

# Ping — перевірити з'єднання
redis-cli ping
# → PONG

# Ping з повідомленням
redis-cli ping "hello"
# → "hello"
```

### Корисні CLI команди для адміністрування

```bash
# Інформація про сервер
redis-cli INFO server

# Інформація про пам'ять
redis-cli INFO memory

# Статистика
redis-cli INFO stats

# Клієнти
redis-cli INFO clients

# Persistence
redis-cli INFO persistence

# Все разом
redis-cli INFO all

# Моніторинг в реальному часі (всі команди що виконуються)
redis-cli MONITOR

# Повільні запити (slowlog)
redis-cli SLOWLOG GET 10     # останні 10 повільних команд
redis-cli SLOWLOG LEN        # кількість записів у slowlog
redis-cli SLOWLOG RESET      # очистити slowlog

# Налагодження
redis-cli DEBUG SLEEP 0.1    # симулювати затримку (для тестів)
redis-cli DEBUG JMAP         # дамп пам'яті (для розробки)
```

### redis-cli в interactive mode — корисні tricks

```bash
redis-cli

# Переключитись на БД 1
SELECT 1

# Переключитись назад на БД 0
SELECT 0

# Показати кількість ключів в поточній БД
DBSIZE

# Автодоповнення — натисни Tab після початку команди
SET  → [Tab] підказує аргументи

# Режим повторення команди (як watch)
redis-cli --interval 1 INFO memory
# оновлює кожну секунду

# Latency monitor
redis-cli --latency
redis-cli --latency-history
redis-cli --latency-dist
```

---

## 4. Неймінг ключів — конвенції

Redis не нав'язує формат ключів — це просто рядки. Але спільнота виробила конвенції:

### Роздільник `:` (двокрапка)

```
object:id:field

user:42:name
user:42:email
user:42:sessions

post:17:comments
post:17:likes

oauth_state:abc123def456
session:xyz789
rate_limit:192.168.1.1
```

### Префікси для типів

```
cache:users:42          ← кешовані дані
session:abc123          ← сесія
lock:payment:order_99   ← distributed lock
queue:emails            ← черга
pubsub:notifications    ← pub/sub канал
```

### Правила хорошого тону

```
✅ user:42:profile           → читабельно, ієрархічно
✅ oauth_state:abc123         → зрозумілий префікс
✅ rate_limit:ip:192.168.1.1  → includes entity type

❌ u42p                       → незрозуміло
❌ USER_42_PROFILE             → зайві великі літери
❌ user-42-profile             → дефіс плутає з мінусом
❌ user.42.profile             → крапка не конвенційна
```

### Максимальна довжина ключа

Redis підтримує ключі до **512 MB**. Але практично:
- Тримай ключі до 100-200 символів
- Довгі ключі займають більше пам'яті та повільніше порівнюються
- Якщо ключ дуже довгий — зроби хеш через SHA1/SHA256

```python
import hashlib
long_key = "some:very:long:key:that:exceeds:reasonable:length:abc123xyz"
short_key = f"cache:{hashlib.sha1(long_key.encode()).hexdigest()}"
# cache:da39a3ee5e6b4b0d3255bfef95601890afd80709
```

---

## 5. Глобальні команди для ключів

Ці команди працюють з **будь-яким типом** ключа.

### EXISTS — перевірити існування

```bash
SET foo "bar"
EXISTS foo          → 1  (існує)
EXISTS nonexistent  → 0  (не існує)

# Перевірити кілька ключів одразу
SET a "1"
SET b "2"
EXISTS a b c        → 2  (a і b існують, c ні)
# Увага: якщо передати один ключ двічі — рахується двічі
EXISTS a a          → 2
```

```python
await redis.exists("foo")           # → 1 або 0
await redis.exists("a", "b", "c")   # → 2
```

---

### DEL — видалення

```bash
SET a "1"
SET b "2"
SET c "3"

DEL a           → 1  (видалено 1 ключ)
DEL b c         → 2  (видалено 2 ключі)
DEL nonexistent → 0  (нічого не видалено)
```

```python
await redis.delete("a")         # → 1
await redis.delete("b", "c")    # → 2
```

---

### UNLINK — асинхронне видалення

```bash
UNLINK big_key   → 1
```

`UNLINK` на відміну від `DEL` не видаляє пам'ять одразу — воно помічає ключ як "видалений" і звільняє пам'ять у фоновому потоці. Це важливо для великих ключів (великі Hash, List з мільйонами елементів), де `DEL` може заблокувати event loop на мілісекунди.

```python
await redis.unlink("big_key")   # асинхронне видалення
await redis.delete("big_key")   # синхронне видалення (блокує)
```

---

### TYPE — отримати тип значення

```bash
SET str_key "hello"
HSET hash_key field "value"
LPUSH list_key "item"
SADD set_key "member"
ZADD zset_key 1.0 "member"

TYPE str_key    → string
TYPE hash_key   → hash
TYPE list_key   → list
TYPE set_key    → set
TYPE zset_key   → zset
TYPE nonexist   → none
```

```python
t = await redis.type("str_key")   # → b"string" або "string" (якщо decode_responses)
```

---

### RENAME / RENAMENX

```bash
SET old_name "value"
RENAME old_name new_name   → OK
# Якщо new_name вже існує — він перезаписується!

# Безпечний варіант — перейменувати тільки якщо new_name не існує
RENAMENX old_name new_name  → 1  (успішно)
RENAMENX old_name new_name  → 0  (new_name вже існує)
```

```python
await redis.rename("old", "new")
result = await redis.renamenx("old", "new")  # True або False
```

---

### COPY — скопіювати ключ

```bash
SET source "hello"
COPY source destination       → 1  (успішно)
COPY source destination       → 0  (destination вже існує)
COPY source destination REPLACE → 1  (перезаписати якщо існує)

# Копіювати в іншу БД
COPY source destination DB 1  → 1
```

---

### OBJECT ENCODING — дізнатись внутрішнє кодування

```bash
SET small_int 42
OBJECT ENCODING small_int     → int

SET small_str "hello"
OBJECT ENCODING small_str     → embstr

SET big_str "a very long string that exceeds 44 bytes in length definitely"
OBJECT ENCODING big_str       → raw

HSET small_hash f1 v1 f2 v2
OBJECT ENCODING small_hash    → listpack  (якщо маленький)
# або → ziplist (старі версії Redis)
# або → hashtable (якщо великий)
```

Redis автоматично оптимізує кодування залежно від розміру даних:

| Структура | Маленька | Велика |
|---|---|---|
| String | `int` або `embstr` | `raw` |
| Hash | `listpack` (≤128 полів, ≤64 байти) | `hashtable` |
| List | `listpack` (≤128 елем, ≤64 байти) | `quicklist` |
| Set | `listpack` або `intset` | `hashtable` |
| Sorted Set | `listpack` | `skiplist` |

Це важливо для розуміння пам'яті: `listpack` займає в 5-10x менше пам'яті ніж `hashtable`.

---

### OBJECT REFCOUNT / IDLETIME / FREQ / HELP

```bash
SET foo "hello"
OBJECT REFCOUNT foo    → 1  (кількість посилань — для garbage collection)
OBJECT IDLETIME foo    → 0  (секунд з останнього звернення)
OBJECT FREQ foo        → 0  (частота звернень — для LFU eviction)
OBJECT HELP            → список підкоманд
```

---

### DUMP / RESTORE — серіалізація ключа

```bash
SET foo "hello"
DUMP foo    → "\x00\x05hello\n\x00..." (бінарний RDB формат)

# Відновити на іншому сервері
RESTORE new_key 0 "\x00\x05hello\n\x00..."  # 0 = TTL (без обмеження)
RESTORE new_key 5000 "..."                   # 5000ms TTL
```

Корисно для міграції або копіювання ключів між Redis-серверами.

---

### SCAN — безпечний перебір ключів

```bash
# KEYS * — НЕБЕЗПЕЧНО! Блокує Redis поки перебирає всі ключі
KEYS *              # Не юзай в production!
KEYS user:*         # Не юзай в production!

# SCAN — ітеративний, не блокує
SCAN 0               → "34" ["key1", "key2", ...]
#    ^cursor           ^new cursor  ^batch of keys
SCAN 34              → "89" [...]
SCAN 89              → "0"  [...]  ← cursor=0 означає кінець
```

SCAN повертає **курсор** і **пачку ключів**. Продовжуй поки курсор не повернеться до `0`.

```bash
# Фільтрація за патерном
SCAN 0 MATCH user:*        → повертає тільки ключі user:*
SCAN 0 MATCH user:* COUNT 100  # підказка скільки брати за раз (не гарантія)

# SCAN по типу (Redis 6.0+)
SCAN 0 MATCH * TYPE string    # тільки String ключі
SCAN 0 MATCH * TYPE hash      # тільки Hash ключі
```

```python
# Асинхронний ітератор — найкращий спосіб
async for key in redis.scan_iter(match="user:*", count=100):
    print(key)

# Або вручну
cursor = 0
while True:
    cursor, keys = await redis.scan(cursor, match="user:*", count=100)
    for key in keys:
        print(key)
    if cursor == 0:
        break
```

---

### WAIT — очікувати реплікації

```bash
# Дочекатись поки N реплік отримають останні записи (timeout у мс)
WAIT 1 100    → 1  (1 репліка підтвердила за 100мс)
```

---

## 6. String — рядки

String — найпростіша і найчастіше використовувана структура. Але "String" в Redis — це не лише текст. Це **бінарна послідовність байт** до 512 MB.

Що можна зберегти в String:
- Текст: `"Hello, World!"`
- Числа: `"42"`, `"3.14"`
- JSON: `'{"id": 1, "name": "Daniil"}'`
- Бінарні дані: зображення, серіалізовані об'єкти
- Будь-що до 512 MB

### SET — встановити значення

```bash
SET key value
```

```bash
SET greeting "Hello, World!"
SET user_count 0
SET is_active true
SET config '{"theme": "dark", "lang": "uk"}'
```

**Опції SET:**

```bash
# EX — TTL в секундах
SET session:abc "user_data" EX 3600     # видалити через 1 годину

# PX — TTL в мілісекундах
SET cache:key "data" PX 30000           # видалити через 30 секунд

# EXAT — TTL як Unix timestamp (секунди) [Redis 6.2+]
SET key "value" EXAT 1735689600         # видалити о конкретній даті

# PXAT — TTL як Unix timestamp (мілісекунди) [Redis 6.2+]
SET key "value" PXAT 1735689600000

# NX — встановити ТІЛЬКИ якщо ключ НЕ існує
SET lock:resource "owner_id" NX EX 30   # distributed lock!
# → OK   (якщо встановлено)
# → nil  (якщо ключ вже існував)

# XX — встановити ТІЛЬКИ якщо ключ вже існує
SET user:42:name "Daniil" XX
# → OK   (якщо ключ існує і оновлений)
# → nil  (якщо ключ не існував)

# GET — повернути старе значення при SET [Redis 6.2+]
SET counter 10
SET counter 20 GET   → "10"  (повернуло старе значення "10")

# KEEPTTL — не змінювати TTL при оновленні [Redis 6.0+]
SET session:abc "user_data" EX 3600
SET session:abc "new_data" KEEPTTL     # TTL залишиться 3600
```

```python
# Python еквіваленти
await redis.set("key", "value")
await redis.set("key", "value", ex=3600)
await redis.set("key", "value", px=30000)
await redis.set("key", "value", nx=True)         # → True або None
await redis.set("key", "value", xx=True)         # → True або None
await redis.set("key", "value", keepttl=True)
await redis.set("key", "value", get=True)        # → старе значення
```

---

### GET — отримати значення

```bash
SET foo "bar"
GET foo          → "bar"
GET nonexistent  → nil   (ключ не існує)
```

```python
value = await redis.get("foo")   # → "bar" або None
```

---

### GETSET — встановити і повернути старе значення (застарілий)

```bash
SET counter 10
GETSET counter 20   → "10"  (повернуло 10, встановило 20)
GET counter         → "20"
```

> **Застарілий з Redis 6.2.** Використовуй `SET key value GET` замість цього.

---

### GETDEL — отримати і видалити [Redis 6.2+]

```bash
SET oauth_state:abc "user_data"
GETDEL oauth_state:abc   → "user_data"  (ключ видалений)
GET oauth_state:abc      → nil
```

```python
value = await redis.getdel("oauth_state:abc")   # → value або None
```

**Атомарна операція** — ключ ніхто не встигне прочитати між GET і DEL.

---

### GETEX — отримати і встановити/прибрати TTL [Redis 6.2+]

```bash
SET session:abc "data" EX 3600

# Отримати і продовжити TTL
GETEX session:abc EX 3600    → "data"  (TTL оновлено до 3600)
GETEX session:abc EXAT 1735689600  → "data"  (TTL як Unix timestamp)
GETEX session:abc PERSIST    → "data"  (прибрати TTL — ключ стане "вічним")
```

```python
# Sliding session expiration
value = await redis.getex("session:abc", ex=3600)  # продовжити сесію
```

---

### MSET / MGET — масові операції

```bash
# Встановити кілька ключів АТОМАРНО
MSET user:1:name "Alice" user:1:age "25" user:2:name "Bob"
# → OK

# Отримати кілька ключів одним запитом
MGET user:1:name user:1:age user:2:name user:99:name
# → ["Alice", "25", "Bob", nil]
```

```python
# MSET
await redis.mset({"user:1:name": "Alice", "user:1:age": "25"})

# MGET
values = await redis.mget("user:1:name", "user:1:age", "nonexistent")
# → ["Alice", "25", None]
```

> **Продуктивність:** `MGET` з 100 ключами набагато швидше ніж 100 окремих `GET`. Один round-trip до Redis замість 100.

---

### MSETNX — встановити кілька, тільки якщо жоден не існує

```bash
MSETNX key1 "val1" key2 "val2"   → 1  (встановлено, бо жоден не існував)
MSETNX key1 "new" key3 "val3"    → 0  (не встановлено, бо key1 вже існує)
# Атомарно: або всі встановлюються, або жоден
```

---

### APPEND — додати до рядка

```bash
SET log ""
APPEND log "2024-01-01 10:00 Login\n"      → 24  (довжина рядка)
APPEND log "2024-01-01 10:05 Search\n"     → 48
APPEND log "2024-01-01 10:10 Logout\n"     → 72
GET log
# "2024-01-01 10:00 Login\n2024-01-01 10:05 Search\n..."
```

```python
new_len = await redis.append("log", "new entry\n")
```

---

### STRLEN — довжина рядка

```bash
SET greeting "Hello"
STRLEN greeting     → 5
STRLEN nonexistent  → 0
```

```python
length = await redis.strlen("greeting")
```

---

### SETRANGE / GETRANGE — операції зі зрізами

```bash
SET greeting "Hello World"

# Замінити частину рядка починаючи з offset
SETRANGE greeting 6 "Redis"
GET greeting    → "Hello Redis"

# Отримати підрядок [start, end] включно
GETRANGE greeting 0 4    → "Hello"
GETRANGE greeting 6 10   → "Redis"
GETRANGE greeting 0 -1   → "Hello Redis"  (весь рядок)
GETRANGE greeting -5 -1  → "Redis"        (з кінця)
```

```python
await redis.setrange("greeting", 6, "Redis")
substring = await redis.getrange("greeting", 0, 4)
```

---

### SUBSTR — застарілий псевдонім GETRANGE

```bash
SUBSTR greeting 0 4   → "Hello"
# Використовуй GETRANGE замість цього
```

---

## 7. Integer та Float через String

Redis зберігає числа як рядки, але розпізнає їх і надає атомарні математичні операції.

### INCR / DECR

```bash
SET counter 10

INCR counter    → 11
INCR counter    → 12
DECR counter    → 11

# Якщо ключ не існує — вважається 0
INCR new_counter   → 1
INCR new_counter   → 2

# Якщо значення не число — помилка
SET name "hello"
INCR name   → ERR value is not an integer or out of range
```

```python
new_val = await redis.incr("counter")     # → int
new_val = await redis.decr("counter")     # → int
```

**Атомарність** — гарантована. Навіть якщо 1000 клієнтів одночасно виконають `INCR counter` — жодне збільшення не буде втрачено.

---

### INCRBY / DECRBY

```bash
SET score 100

INCRBY score 50    → 150  (збільшити на 50)
INCRBY score -20   → 130  (зменшити через від'ємне число)
DECRBY score 30    → 100  (зменшити на 30)
```

```python
await redis.incrby("score", 50)
await redis.decrby("score", 30)
```

---

### INCRBYFLOAT

```bash
SET price 10.00

INCRBYFLOAT price 1.50    → 11.5
INCRBYFLOAT price -0.5    → 11
INCRBYFLOAT price 0.001   → 11.001

# Важливо: float може мати проблеми з точністю
INCRBYFLOAT price 3.1415926535   → 14.1415926535

# Немає DECRBYFLOAT — просто передай від'ємне число
INCRBYFLOAT price -1.0    → 13.1415926535
```

```python
await redis.incrbyfloat("price", 1.50)
await redis.incrbyfloat("price", -0.5)
```

---

### SETNX — встановити якщо не існує (застарілий)

```bash
SETNX lock:resource "owner"   → 1  (встановлено)
SETNX lock:resource "other"   → 0  (вже існує, не змінено)
```

> **Застарілий.** Використовуй `SET key value NX` — вона підтримує TTL в одній атомарній операції.

---

### SETEX / PSETEX — встановити з TTL (застарілі)

```bash
SETEX session:abc 3600 "user_data"    # TTL в секундах
PSETEX session:abc 3600000 "data"     # TTL в мілісекундах
```

> **Застарілі.** Використовуй `SET key value EX 3600` або `SET key value PX 3600000`.

---

## 8. Hash — словники

Hash дозволяє зберігати в одному ключі **словник поле → значення**. Ідеально для об'єктів.

```
user:42  →  {
    name:  "Daniil"
    email: "d@example.com"
    age:   "18"
    city:  "Bila Tserkva"
}
```

### Коли Hash кращий за кілька String ключів

```bash
# ❌ Кілька String ключів — 4 операції, 4 записи в keyspace
SET user:42:name  "Daniil"
SET user:42:email "d@example.com"
SET user:42:age   "18"
SET user:42:city  "Bila Tserkva"

# ✅ Один Hash — 1 операція, 1 запис в keyspace
HSET user:42  name "Daniil"  email "d@example.com"  age "18"  city "Bila Tserkva"
```

Hash з невеликою кількістю полів (≤128) зберігається як `listpack` — дуже компактний формат. Кілька String ключів — кожен з overhead на метадані.

---

### HSET — встановити одне або кілька полів

```bash
# Одне поле
HSET user:42 name "Daniil"   → 1  (кількість НОВИХ полів)

# Кілька полів
HSET user:42 name "Daniil" email "d@example.com" age "18"
→ 2  (2 нових поля; name вже існувало — не рахується)
```

```python
# Одне поле
await redis.hset("user:42", "name", "Daniil")

# Кілька через mapping
await redis.hset("user:42", mapping={
    "name": "Daniil",
    "email": "d@example.com",
    "age": "18",
})
```

---

### HGET — отримати одне поле

```bash
HGET user:42 name    → "Daniil"
HGET user:42 phone   → nil  (поле не існує)
HGET nonexist field  → nil  (ключ не існує)
```

```python
name = await redis.hget("user:42", "name")   # → "Daniil" або None
```

---

### HMGET — отримати кілька полів

```bash
HMGET user:42 name email age nonexistent
→ ["Daniil", "d@example.com", "18", nil]
```

```python
values = await redis.hmget("user:42", "name", "email", "nonexistent")
# → ["Daniil", "d@example.com", None]
```

---

### HGETALL — отримати всі поля і значення

```bash
HGETALL user:42
→ ["name", "Daniil", "email", "d@example.com", "age", "18", "city", "Bila Tserkva"]
# Повертає чергуючись: поле, значення, поле, значення...
```

```python
user = await redis.hgetall("user:42")
# → {"name": "Daniil", "email": "d@example.com", ...}  (dict)
```

---

### HKEYS / HVALS / HLEN

```bash
HKEYS user:42    → ["name", "email", "age", "city"]   (тільки ключі)
HVALS user:42    → ["Daniil", "d@example.com", "18", "Bila Tserkva"]  (тільки значення)
HLEN user:42     → 4   (кількість полів)
```

```python
keys = await redis.hkeys("user:42")
vals = await redis.hvals("user:42")
count = await redis.hlen("user:42")
```

---

### HEXISTS — перевірити існування поля

```bash
HEXISTS user:42 name    → 1  (існує)
HEXISTS user:42 phone   → 0  (не існує)
```

```python
exists = await redis.hexists("user:42", "name")   # → True або False
```

---

### HDEL — видалити поля

```bash
HDEL user:42 age city    → 2  (видалено 2 поля)
HDEL user:42 nonexist    → 0
```

```python
deleted = await redis.hdel("user:42", "age", "city")   # → int
```

---

### HSETNX — встановити поле тільки якщо не існує

```bash
HSETNX user:42 name "Alice"   → 0  (name вже існує, не змінено)
HSETNX user:42 phone "+380"   → 1  (phone не існувало, встановлено)
```

```python
result = await redis.hsetnx("user:42", "phone", "+380")   # → True або False
```

---

### HINCRBY / HINCRBYFLOAT — збільшити числове поле

```bash
HSET stats views 100 likes 50

HINCRBY stats views 10        → 110
HINCRBY stats views -5        → 105
HINCRBYFLOAT stats rating 0.5 → 0.5  (якщо поле не існувало — починає з 0)
```

```python
await redis.hincrby("stats", "views", 10)
await redis.hincrbyfloat("stats", "rating", 0.5)
```

---

### HSCAN — ітерація по полях (для великих Hash)

```bash
HSCAN user:42 0            → "0" ["name", "Daniil", "email", "d@..."]
HSCAN big_hash 0 COUNT 10  → cursor + batch
HSCAN big_hash 0 MATCH "e*"  → тільки поля що починаються з 'e'
```

```python
async for field, value in redis.hscan_iter("user:42"):
    print(f"{field}: {value}")

# З фільтром
async for field, value in redis.hscan_iter("user:42", match="e*"):
    print(f"{field}: {value}")
```

---

### HRANDFIELD — випадкове поле [Redis 6.2+]

```bash
HRANDFIELD user:42           → "name"  (одне випадкове поле)
HRANDFIELD user:42 3         → ["name", "age", "city"]  (3 випадкових)
HRANDFIELD user:42 -3        → ["age", "age", "name"]   (3 з повторами)
HRANDFIELD user:42 2 WITHVALUES → ["name", "Daniil", "age", "18"]
```

```python
field = await redis.hrandfield("user:42")
fields = await redis.hrandfield("user:42", count=3)
fields_with_vals = await redis.hrandfield("user:42", count=2, withvalues=True)
```

---

## 9. List — черги та стеки

Redis List — це **linked list** (двобічний зв'язний список). Підтримує операції з обох кінців за `O(1)`.

```
LEFT (голова)                        RIGHT (хвіст)
     ↕                                    ↕
[ "task3" | "task2" | "task1" | "task0" ]
  ↑ index 0    index 1   index 2    index 3
  або -4        -3         -2         -1 (від кінця)
```

### LPUSH / RPUSH — додати елементи

```bash
# L = Left (додає зліва, тобто на початок)
LPUSH queue "task1"          → 1
LPUSH queue "task2"          → 2
LPUSH queue "task3"          → 3
# Список: [task3, task2, task1]

# Можна передати кілька — додаються зліва по черзі
LPUSH queue "a" "b" "c"
# Список: [c, b, a, task3, task2, task1]

# R = Right (додає справа, тобто в кінець)
RPUSH queue "last"           → 7
# Список: [c, b, a, task3, task2, task1, last]
```

```python
await redis.lpush("queue", "task1")
await redis.lpush("queue", "task2", "task3")  # додає task3 потім task2 зліва
await redis.rpush("queue", "last")
```

---

### LPOP / RPOP — витягнути елементи

```bash
# Поточний список: [task3, task2, task1]

LPOP queue     → "task3"   (забрали зліва)
# Список: [task2, task1]

RPOP queue     → "task1"   (забрали справа)
# Список: [task2]

# Витягнути кілька одразу (Redis 6.2+)
RPUSH nums 1 2 3 4 5
LPOP nums 3    → ["1", "2", "3"]
RPOP nums 2    → ["5", "4"]
```

```python
item = await redis.lpop("queue")
item = await redis.rpop("queue")
items = await redis.lpop("queue", count=3)  # список
```

---

### BLPOP / BRPOP — блокуючі POP (для черг)

```bash
# Чекати поки з'явиться елемент (блокує)
BLPOP queue 0      # чекати нескінченно
BLPOP queue 30     # чекати максимум 30 секунд
# → ["queue", "task1"]   (повертає ім'я ключа + значення)
# → nil якщо timeout

# Чекати на першому непорожньому з кількох списків
BLPOP queue1 queue2 queue3 10
```

```python
# Блокуючий pop — ідеально для воркерів
result = await redis.blpop("task_queue", timeout=30)
if result:
    queue_name, task = result
    process(task)
```

**Патерн Worker Queue:**
```
Producer:  RPUSH task_queue "job_data"
Worker:    BLPOP task_queue 0  ← чекає і бере задачу
```

---

### LLEN — довжина списку

```bash
LLEN queue     → 3
LLEN nonexist  → 0
```

```python
length = await redis.llen("queue")
```

---

### LRANGE — отримати зріз

```bash
RPUSH nums 1 2 3 4 5

LRANGE nums 0 -1    → ["1", "2", "3", "4", "5"]  (весь список)
LRANGE nums 0 2     → ["1", "2", "3"]              (перші 3)
LRANGE nums 1 3     → ["2", "3", "4"]
LRANGE nums -3 -1   → ["3", "4", "5"]              (останні 3)
LRANGE nums 0 999   → ["1", "2", "3", "4", "5"]   (більше ніж є — ок)
```

```python
all_items = await redis.lrange("nums", 0, -1)
first_10 = await redis.lrange("nums", 0, 9)
```

---

### LINDEX — отримати елемент за індексом

```bash
RPUSH letters a b c d e

LINDEX letters 0    → "a"
LINDEX letters 2    → "c"
LINDEX letters -1   → "e"   (останній)
LINDEX letters 99   → nil   (не існує)
```

```python
item = await redis.lindex("letters", 0)
last = await redis.lindex("letters", -1)
```

---

### LSET — змінити елемент за індексом

```bash
RPUSH letters a b c

LSET letters 1 "B"   → OK
LRANGE letters 0 -1  → ["a", "B", "c"]

LSET letters 99 "x"  → ERR index out of range
```

```python
await redis.lset("letters", 1, "B")
```

---

### LINSERT — вставити перед/після елемента

```bash
RPUSH letters a c

LINSERT letters BEFORE "c" "b"  → 3  (нова довжина)
LRANGE letters 0 -1              → ["a", "b", "c"]

LINSERT letters AFTER "b" "b2"  → 4
LRANGE letters 0 -1              → ["a", "b", "b2", "c"]

LINSERT letters BEFORE "x" "y"  → -1  (pivot не знайдено)
```

```python
await redis.linsert("letters", "BEFORE", "c", "b")
await redis.linsert("letters", "AFTER", "b", "b2")
```

---

### LREM — видалити елементи за значенням

```bash
RPUSH log "error" "info" "error" "warn" "error" "info"

LREM log 2 "error"    → 2  (видалено 2 перших "error" зліва)
# Список: ["info", "warn", "error", "info"]

LREM log -1 "info"    → 1  (видалено 1 "info" справа)
# Список: ["info", "warn", "error"]

LREM log 0 "warn"     → 1  (видалити всі "warn")
```

```
count > 0: видаляти зліва, max count штук
count < 0: видаляти справа, max |count| штук
count = 0: видалити всі входження
```

```python
await redis.lrem("log", 2, "error")   # видалити 2 зліва
await redis.lrem("log", -1, "info")   # видалити 1 справа
await redis.lrem("log", 0, "warn")    # видалити всі
```

---

### LTRIM — обрізати список

```bash
RPUSH nums 1 2 3 4 5 6 7 8 9 10

LTRIM nums 0 4    → OK   (залишити тільки перші 5)
LRANGE nums 0 -1  → ["1", "2", "3", "4", "5"]
```

**Патерн "Circular Buffer" — останні N логів:**

```bash
RPUSH logs "entry1"
RPUSH logs "entry2"
...
RPUSH logs "entry101"
LTRIM logs -100 -1   # зберігати тільки останні 100 записів
```

```python
await redis.rpush("logs", entry)
await redis.ltrim("logs", -100, -1)
```

---

### LMOVE — перемістити елемент між списками [Redis 6.2+]

```bash
# LMOVE source destination LEFT|RIGHT LEFT|RIGHT
RPUSH pending "task1" "task2" "task3"

# Взяти з правого кінця pending і додати зліва в processing
LMOVE pending processing RIGHT LEFT   → "task3"
LRANGE pending 0 -1     → ["task1", "task2"]
LRANGE processing 0 -1  → ["task3"]
```

**Атомарна** операція — ідеально для reliable queue (якщо воркер впав — задача залишається в processing).

```python
task = await redis.lmove("pending", "processing", "RIGHT", "LEFT")
```

---

### LPOS — знайти позицію елемента [Redis 6.0.6+]

```bash
RPUSH letters a b c b a b

LPOS letters "b"           → 1      (перше входження)
LPOS letters "b" RANK 2    → 3      (друге входження)
LPOS letters "b" COUNT 0   → [1, 3, 5]  (всі входження)
LPOS letters "b" RANK -1   → 5      (останнє входження)
```

```python
pos = await redis.lpos("letters", "b")
positions = await redis.lpos("letters", "b", count=0)  # всі
```

---

### RPOPLPUSH — застарілий (використовуй LMOVE)

```bash
RPOPLPUSH source destination   # = LMOVE source destination RIGHT LEFT
```

---

## 10. Set — множини

Set — це **невпорядкована** колекція **унікальних** рядків.

```
tags:post:42  →  { "redis", "python", "backend", "fastapi" }
```

### SADD — додати елементи

```bash
SADD tags:post:42 "redis" "python" "backend"   → 3  (додано 3 нових)
SADD tags:post:42 "redis"                       → 0  (redis вже є, не додано)
SADD tags:post:42 "fastapi"                     → 1  (додано 1 новий)
```

```python
added = await redis.sadd("tags:post:42", "redis", "python", "backend")
```

---

### SREM — видалити елементи

```bash
SREM tags:post:42 "python" "nonexistent"   → 1  (видалено 1 елемент)
```

```python
removed = await redis.srem("tags:post:42", "python", "nonexistent")
```

---

### SMEMBERS — всі елементи

```bash
SMEMBERS tags:post:42   → {"redis", "backend", "fastapi"}
# Порядок не гарантований!
```

```python
members = await redis.smembers("tags:post:42")   # → set
```

---

### SISMEMBER — перевірити приналежність

```bash
SISMEMBER tags:post:42 "redis"    → 1  (є в множині)
SISMEMBER tags:post:42 "java"     → 0  (немає)
```

```python
is_member = await redis.sismember("tags:post:42", "redis")   # → True/False
```

---

### SMISMEMBER — перевірити кілька [Redis 6.2+]

```bash
SMISMEMBER tags:post:42 "redis" "java" "fastapi"   → [1, 0, 1]
```

```python
results = await redis.smismember("tags:post:42", "redis", "java", "fastapi")
# → [True, False, True]
```

---

### SCARD — розмір множини

```bash
SCARD tags:post:42   → 3
```

```python
count = await redis.scard("tags:post:42")
```

---

### SRANDMEMBER — випадковий елемент

```bash
SRANDMEMBER tags:post:42        → "redis"  (один випадковий)
SRANDMEMBER tags:post:42 2      → ["redis", "fastapi"]  (2 без повторів)
SRANDMEMBER tags:post:42 -5     → ["redis", "redis", "backend", "fastapi", "redis"]
#                          ^від'ємне = з повторами
```

```python
item = await redis.srandmember("tags:post:42")
items = await redis.srandmember("tags:post:42", count=2)
```

---

### SPOP — витягнути і видалити випадковий елемент

```bash
SPOP tags:post:42     → "fastapi"  (видалено з множини)
SPOP tags:post:42 2   → ["redis", "backend"]  (видалити 2)
```

```python
item = await redis.spop("tags:post:42")
items = await redis.spop("tags:post:42", count=2)
```

---

### SMOVE — перемістити елемент між множинами

```bash
SADD set1 "a" "b" "c"
SADD set2 "d" "e"

SMOVE set1 set2 "b"   → 1  (переміщено)
SMEMBERS set1          → {"a", "c"}
SMEMBERS set2          → {"d", "e", "b"}
```

```python
result = await redis.smove("set1", "set2", "b")   # → True/False
```

---

### Операції над множинами — UNION, INTER, DIFF

```bash
SADD set_a "a" "b" "c" "d"
SADD set_b "c" "d" "e" "f"

# Об'єднання (union) — A ∪ B
SUNION set_a set_b       → {"a", "b", "c", "d", "e", "f"}
SUNIONSTORE dest set_a set_b  → 6  (зберегти результат у dest)

# Перетин (intersection) — A ∩ B
SINTER set_a set_b       → {"c", "d"}
SINTERSTORE dest set_a set_b  → 2

# Різниця (difference) — A \ B
SDIFF set_a set_b        → {"a", "b"}   (є в A але не в B)
SDIFF set_b set_a        → {"e", "f"}   (є в B але не в A)
SDIFFSTORE dest set_a set_b  → 2

# Розмір перетину без повернення елементів [Redis 7.0+]
SINTERCARD 2 set_a set_b         → 2
SINTERCARD 2 set_a set_b LIMIT 1 → 1  (обмежити результат)
```

```python
union = await redis.sunion("set_a", "set_b")
inter = await redis.sinter("set_a", "set_b")
diff = await redis.sdiff("set_a", "set_b")

# Зберегти в новий ключ
await redis.sunionstore("dest", "set_a", "set_b")
await redis.sinterstore("dest", "set_a", "set_b")
await redis.sdiffstore("dest", "set_a", "set_b")
```

---

### SSCAN — ітерація по великих множинах

```bash
SSCAN tags:post:42 0            → "0" ["redis", "python", ...]
SSCAN tags:post:42 0 MATCH "f*" → тільки елементи що починаються з 'f'
SSCAN tags:post:42 0 COUNT 100  → підказка про розмір батчу
```

```python
async for member in redis.sscan_iter("big_set", match="prefix:*"):
    print(member)
```

---

## 11. Sorted Set — відсортовані множини

Sorted Set (ZSet) — це множина унікальних елементів, де кожен елемент має **числовий score** (float). Елементи завжди відсортовані за score від меншого до більшого.

```
leaderboard  →  { "user:3": 500, "user:1": 1200, "user:2": 2500 }
                        ↑ score        ↑ member
# Відсортовано: user:3 (500) < user:1 (1200) < user:2 (2500)
```

**Де використовується:**
- Рейтинги та leaderboards
- Rate limiting (score = timestamp)
- Пріоритетні черги
- Геопросторові індекси (GEO API побудований поверх ZSet)
- Delayed jobs (score = час виконання)

---

### ZADD — додати елементи

```bash
ZADD leaderboard 2500 "user:2"   → 1  (1 новий елемент)
ZADD leaderboard 1200 "user:1"   → 1
ZADD leaderboard 500  "user:3"   → 1

# Кілька одразу
ZADD leaderboard 2500 "user:2" 1200 "user:1" 500 "user:3"   → 3

# Оновити score існуючого елемента
ZADD leaderboard 3000 "user:2"   → 0  (0 нових, але score змінено)
```

**Опції ZADD:**

```bash
# NX — додати тільки якщо не існує (не оновлювати)
ZADD leaderboard NX 9999 "user:1"   → 0  (не змінено, user:1 вже є)
ZADD leaderboard NX 9999 "user:new" → 1  (доданий новий)

# XX — оновити тільки якщо вже існує
ZADD leaderboard XX 3000 "user:1"    → 0  (оновлено)
ZADD leaderboard XX 3000 "new_user"  → 0  (не додано, бо не існував)

# GT — оновити тільки якщо новий score БІЛЬШИЙ [Redis 6.2+]
ZADD leaderboard 1200 "user:1"  # зараз 1200
ZADD leaderboard GT 1000 "user:1"  → 0  (не змінено, 1000 < 1200)
ZADD leaderboard GT 1500 "user:1"  → 0  (оновлено до 1500)

# LT — оновити тільки якщо новий score МЕНШИЙ [Redis 6.2+]
ZADD leaderboard LT 1000 "user:1"  → 0  (оновлено, 1000 < 1500)
ZADD leaderboard LT 9999 "user:1"  → 0  (не змінено, 9999 > 1000)

# CH — змінити повернення: рахувати також оновлені елементи
ZADD leaderboard CH 2000 "user:1" 100 "new_user"  → 2  (1 оновлений + 1 новий)

# INCR — збільшити score (замість ZINCRBY)
ZADD leaderboard INCR 500 "user:1"  → нове значення score
```

```python
await redis.zadd("leaderboard", {"user:1": 1200, "user:2": 2500})
await redis.zadd("leaderboard", {"user:1": 500}, nx=True)
await redis.zadd("leaderboard", {"user:1": 1500}, gt=True)
```

---

### ZSCORE — отримати score

```bash
ZSCORE leaderboard "user:1"    → "1500"  (завжди рядок)
ZSCORE leaderboard "nobody"    → nil
```

```python
score = await redis.zscore("leaderboard", "user:1")   # → float або None
```

---

### ZINCRBY — збільшити score

```bash
ZINCRBY leaderboard 100 "user:1"   → "1600"  (новий score)
ZINCRBY leaderboard -50 "user:1"   → "1550"
# Якщо елемент не існував — score починається з 0 + increment
ZINCRBY leaderboard 100 "new_user" → "100"
```

```python
new_score = await redis.zincrby("leaderboard", 100, "user:1")  # → float
```

---

### ZRANK / ZREVRANK — позиція в рейтингу

```bash
# ZRANK — позиція з 0, відсортовано від найменшого
ZRANK leaderboard "user:3"   → 0   (найменший score = перший)
ZRANK leaderboard "user:1"   → 1
ZRANK leaderboard "user:2"   → 2   (найбільший score = останній в ASC)
ZRANK leaderboard "nobody"   → nil

# ZREVRANK — позиція з 0, відсортовано від найбільшого (для leaderboard!)
ZREVRANK leaderboard "user:2" → 0  (перший у рейтингу)
ZREVRANK leaderboard "user:1" → 1
ZREVRANK leaderboard "user:3" → 2  (останній)

# З поверненням score (Redis 7.2+)
ZRANK leaderboard "user:1" WITHSCORE  → [1, "1550"]
```

```python
rank = await redis.zrank("leaderboard", "user:1")
rev_rank = await redis.zrevrank("leaderboard", "user:1")
```

---

### ZRANGE — отримати елементи за позицією

```bash
# ASC (від найменшого)
ZRANGE leaderboard 0 -1                → ["user:3", "user:1", "user:2"]
ZRANGE leaderboard 0 -1 WITHSCORES    → ["user:3", "500", "user:1", "1550", ...]
ZRANGE leaderboard 0 2                 → перші 3 елементи
ZRANGE leaderboard -3 -1               → останні 3 елементи

# DESC (від найбільшого) [Redis 6.2+]
ZRANGE leaderboard 0 -1 REV           → ["user:2", "user:1", "user:3"]

# За score
ZRANGE leaderboard 1000 2000 BYSCORE  → елементи зі score між 1000 і 2000
ZRANGE leaderboard "(1000" 2000 BYSCORE  # "(1000" = більше ніж 1000 (виключно)
ZRANGE leaderboard -inf +inf BYSCORE  → всі елементи за score

# З LIMIT (для пагінації)
ZRANGE leaderboard 0 +inf BYSCORE LIMIT 0 10  → перша сторінка (0-9)
ZRANGE leaderboard 0 +inf BYSCORE LIMIT 10 10 → друга сторінка (10-19)

# За лексикографічним порядком (коли всі scores однакові)
ZRANGE names "[A" "[Z" BYLEX
ZRANGE names "[A" "(C" BYLEX  # "(C" = менше ніж C
```

```python
# Всі елементи з score ASC
members = await redis.zrange("leaderboard", 0, -1)
members_with_scores = await redis.zrange("leaderboard", 0, -1, withscores=True)
# → [("user:3", 500.0), ("user:1", 1550.0), ("user:2", 2500.0)]

# DESC
members = await redis.zrange("leaderboard", 0, -1, rev=True)

# За score
members = await redis.zrange("leaderboard", 1000, 2000, byscore=True)
```

---

### ZRANGEBYSCORE / ZREVRANGEBYSCORE — застарілі

```bash
ZRANGEBYSCORE leaderboard 1000 2000              → між 1000 і 2000
ZRANGEBYSCORE leaderboard -inf +inf WITHSCORES   → всі
ZRANGEBYSCORE leaderboard "(1000" 2000           → більше 1000 (виключно)
ZRANGEBYSCORE leaderboard 1000 +inf LIMIT 0 10   → пагінація

ZREVRANGEBYSCORE leaderboard +inf -inf           → всі в DESC порядку
```

> **Застарілі з Redis 6.2.** Використовуй `ZRANGE ... BYSCORE`.

---

### ZRANGEBYLEX / ZREVRANGEBYLEX — за алфавітом (застарілі)

Використовується коли всі scores однакові (наприклад, 0 для всіх):

```bash
ZADD names 0 "Alice" 0 "Bob" 0 "Charlie" 0 "Dave"

ZRANGEBYLEX names "[A" "[C"        → ["Alice", "Bob"]  (від A до C, вкл.)
ZRANGEBYLEX names "[A" "(C"        → ["Alice", "Bob"]  (до C, виключно)
ZRANGEBYLEX names "-" "+"          → всі (- = -∞, + = +∞)
ZRANGEBYLEX names "[B" "+"         → ["Bob", "Charlie", "Dave"]
```

> **Застарілі.** Використовуй `ZRANGE ... BYLEX`.

---

### ZRANGESTORE — ZRANGE і зберегти результат [Redis 6.2+]

```bash
ZRANGESTORE dest leaderboard 0 -1        → 3  (скопійовано в dest)
ZRANGESTORE top3 leaderboard 0 2 REV     → 3  (топ-3 в dest)
```

---

### ZCOUNT / ZLEXCOUNT / ZCARD

```bash
# Кількість елементів зі score в діапазоні
ZCOUNT leaderboard 1000 2000    → 1  (тільки user:1)
ZCOUNT leaderboard -inf +inf    → 3  (всі)
ZCOUNT leaderboard "(1000" +inf → 1  (більше 1000 виключно)

# Кількість елементів за лексикографічним порядком
ZADD names 0 "Alice" 0 "Bob" 0 "Charlie"
ZLEXCOUNT names "-" "+"          → 3  (всі)
ZLEXCOUNT names "[A" "(C"        → 2  (Alice, Bob)

# Загальна кількість
ZCARD leaderboard   → 3
```

```python
count = await redis.zcount("leaderboard", 1000, 2000)
count = await redis.zcount("leaderboard", "-inf", "+inf")
total = await redis.zcard("leaderboard")
```

---

### ZREM / ZREMRANGEBYSCORE / ZREMRANGEBYRANK / ZREMRANGEBYLEX

```bash
# Видалити елементи за іменем
ZREM leaderboard "user:3" "user:1"   → 2  (видалено 2)

# Видалити всі зі score в діапазоні
ZREMRANGEBYSCORE leaderboard 0 999   → 1

# Видалити за позицією (rank)
ZREMRANGEBYRANK leaderboard 0 1      → 2  (видалити 1-й і 2-й)

# Видалити за лекс. діапазоном
ZREMRANGEBYLEX names "[A" "(C"       → 2
```

```python
await redis.zrem("leaderboard", "user:3")
await redis.zremrangebyscore("leaderboard", 0, 999)
await redis.zremrangebyrank("leaderboard", 0, 1)
```

---

### ZPOPMIN / ZPOPMAX — витягнути з мінімальним/максимальним score

```bash
ZPOPMIN leaderboard      → ["user:3", "500"]   (мін score)
ZPOPMAX leaderboard      → ["user:2", "2500"]  (макс score)
ZPOPMIN leaderboard 2    → 2 елементи з найменшим score
```

```python
items = await redis.zpopmin("leaderboard")        # → [(member, score)]
items = await redis.zpopmax("leaderboard", 3)     # → топ-3
```

---

### BZPOPMIN / BZPOPMAX — блокуючий POP

```bash
BZPOPMIN priority_queue 0    # чекати нескінченно
BZPOPMIN priority_queue 10   # чекати 10 секунд
# → ["priority_queue", "job:low", "1"]
```

---

### ZUNION / ZINTER / ZDIFF — операції над ZSet [Redis 6.2+]

```bash
ZADD skills:alice 5 "python" 3 "redis" 4 "fastapi"
ZADD skills:bob   3 "python" 5 "java"  2 "redis"

# Об'єднання (сума scores за замовчуванням)
ZUNION 2 skills:alice skills:bob WITHSCORES
→ [python:8, redis:5, fastapi:4, java:5]

# WEIGHTS — множник для кожного set
ZUNION 2 skills:alice skills:bob WEIGHTS 2 1 WITHSCORES
# alice scores × 2, bob scores × 1

# AGGREGATE — як комбінувати scores
ZUNION 2 skills:alice skills:bob AGGREGATE MIN WITHSCORES
→ [python:3, redis:2, fastapi:4, java:5]   (мінімум зі спільних)

ZUNION 2 skills:alice skills:bob AGGREGATE MAX WITHSCORES
→ [python:5, redis:3, fastapi:4, java:5]   (максимум)

# Перетин — тільки спільні елементи
ZINTER 2 skills:alice skills:bob WITHSCORES
→ [redis:5, python:8]   (сума scores)

# Різниця — є в першому але не в інших
ZDIFF 2 skills:alice skills:bob WITHSCORES
→ [fastapi:4]

# Зберегти результат
ZUNIONSTORE dest 2 skills:alice skills:bob
ZINTERSTORE dest 2 skills:alice skills:bob
ZDIFFSTORE dest 2 skills:alice skills:bob
```

```python
await redis.zunion(["skills:alice", "skills:bob"], withscores=True)
await redis.zinter(["skills:alice", "skills:bob"], aggregate="MIN")
await redis.zunionstore("dest", ["skills:alice", "skills:bob"], weights=[2, 1])
```

---

### ZRANDMEMBER — випадковий елемент [Redis 6.2+]

```bash
ZRANDMEMBER leaderboard        → "user:1"
ZRANDMEMBER leaderboard 2      → ["user:1", "user:2"]
ZRANDMEMBER leaderboard -3     → з повторами
ZRANDMEMBER leaderboard 2 WITHSCORES → [user:1, 1550, user:2, 2500]
```

---

### ZSCAN — ітерація по великих ZSet

```bash
ZSCAN leaderboard 0            → cursor + [member, score, ...]
ZSCAN leaderboard 0 MATCH "user:*" COUNT 10
```

```python
async for member, score in redis.zscan_iter("leaderboard"):
    print(f"{member}: {score}")
```

---

## 12. TTL та Expiration — детально

### Як Redis реалізує TTL

Redis не перевіряє TTL кожен тик. Замість цього використовує **дві стратегії**:

**1. Lazy expiration (ледача)**
При кожному зверненні до ключа Redis перевіряє чи не прострочений він. Якщо прострочений — видаляє і повертає `nil`.

**2. Active expiration (активна)**
Кожні 100 мс Redis бере 20 випадкових ключів з тих що мають TTL. Видаляє прострочені. Якщо більше 25% з них були прострочені — повторює. Це гарантує що пам'ять звільняється навіть якщо ніхто не звертається до ключів.

### Команди TTL

```bash
# EXPIRE — встановити TTL в секундах
EXPIRE key 300         → 1  (успішно)
EXPIRE nonexist 300    → 0  (ключ не існує)

# PEXPIRE — TTL в мілісекундах
PEXPIRE key 300000     → 1

# EXPIREAT — TTL як Unix timestamp (секунди)
EXPIREAT key 1735689600

# PEXPIREAT — TTL як Unix timestamp (мілісекунди)
PEXPIREAT key 1735689600000

# Опції EXPIRE/PEXPIRE (Redis 7.0+):
EXPIRE key 300 NX    # встановити TTL тільки якщо немає TTL
EXPIRE key 300 XX    # встановити TTL тільки якщо вже є TTL
EXPIRE key 300 GT    # встановити тільки якщо новий TTL більший
EXPIRE key 300 LT    # встановити тільки якщо новий TTL менший

# TTL — залишок в секундах
TTL key        → 287    (залишилось 287 секунд)
TTL key        → -1     (ключ існує але без TTL)
TTL nonexist   → -2     (ключ не існує)

# PTTL — залишок в мілісекундах
PTTL key       → 287423

# PERSIST — видалити TTL (зробити "вічним")
PERSIST key    → 1  (TTL видалено)
PERSIST key    → 0  (TTL не було)

# EXPIRETIME — отримати Unix timestamp закінчення (Redis 7.0+)
EXPIRETIME key   → 1735689600   (секунди)
PEXPIRETIME key  → 1735689600000  (мілісекунди)
```

```python
await redis.expire("key", 300)
await redis.pexpire("key", 300000)
await redis.expireat("key", 1735689600)

ttl = await redis.ttl("key")     # int: секунди, -1, або -2
pttl = await redis.pttl("key")   # int: мілісекунди

await redis.persist("key")       # прибрати TTL
```

### Поширені TTL значення

```python
# Зручні константи для проєкту
class TTL:
    MINUTE      = 60
    FIVE_MINUTES = 5 * 60
    FIFTEEN_MINUTES = 15 * 60
    HOUR        = 3600
    SIX_HOURS   = 6 * 3600
    DAY         = 86_400
    WEEK        = 7 * 86_400
    MONTH       = 30 * 86_400

# OAuth state — 10 хвилин
await redis.set("oauth_state:abc", "1", ex=TTL.FIFTEEN_MINUTES)

# Сесія — 7 днів
await redis.set("session:xyz", data, ex=TTL.WEEK)

# Кеш — 1 година
await redis.set("cache:user:42", json_data, ex=TTL.HOUR)

# Rate limit — 1 хвилина
await redis.set("rate:ip:1.2.3.4", "0", ex=TTL.MINUTE)
```

---

## 13. Transactions — MULTI/EXEC

Redis підтримує транзакції через блок `MULTI`/`EXEC`. Всі команди всередині виконуються **атомарно** — або всі, або жодна (при `DISCARD`).

```bash
MULTI              → OK   (починаємо транзакцію)
SET counter 0      → QUEUED  (команди ставляться в чергу, не виконуються)
INCR counter       → QUEUED
INCR counter       → QUEUED
EXEC               → [OK, 1, 2]  (виконати всі команди)
```

```bash
# Відмінити транзакцію
MULTI
SET key "value"
DISCARD            → OK  (транзакцію скасовано, нічого не виконалось)
```

### Помилки всередині EXEC

```bash
MULTI
SET counter 10
INCR counter         # OK, буде виконано
HSET counter f v     # WRONGTYPE error — але транзакція продовжиться!
INCR counter         # виконається (= 11)
EXEC
→ [OK, 11, (error) WRONGTYPE..., 12]
```

> **Важливо:** Redis транзакції НЕ повертаються при помилці всередині EXEC. Якщо одна команда провалилась — решта виконуються.

Єдиний спосіб "відкату" — синтаксична помилка ДО EXEC:

```bash
MULTI
SET counter 10
NOTACOMMAND        → (error) ERR unknown command
INCR counter
EXEC               → EXECABORT  (вся транзакція скасована)
```

### WATCH — оптимістичне блокування

`WATCH` дозволяє реалізувати **Compare-And-Swap (CAS)**:

```bash
# Класичний приклад: перевести кошти між рахунками
WATCH account:alice account:bob   # стежити за змінами
MULTI
DECRBY account:alice 100
INCRBY account:bob 100
EXEC
# → nil якщо account:alice або account:bob змінились між WATCH і EXEC
# → [OK, OK] якщо не змінились
```

```python
async with redis.pipeline(transaction=True) as pipe:
    while True:
        try:
            # Стежити за ключем
            await pipe.watch("counter")
            
            current = int(await pipe.get("counter") or 0)
            
            # Почати транзакцію
            pipe.multi()
            pipe.set("counter", current + 1)
            
            # Виконати (поверне None якщо ключ змінився)
            result = await pipe.execute()
            break  # успіх — виходимо з циклу
        except redis.WatchError:
            # Хтось змінив ключ — повторити
            continue
```

### Pipeline — пакетна відправка команд

Pipeline не є транзакцією, але дозволяє відправити кілька команд одним TCP пакетом (без гарантії атомарності):

```python
# Без pipeline: 3 round-trips
await redis.set("a", 1)
await redis.set("b", 2)
await redis.set("c", 3)

# З pipeline: 1 round-trip
async with redis.pipeline() as pipe:
    pipe.set("a", 1)
    pipe.set("b", 2)
    pipe.set("c", 3)
    results = await pipe.execute()
    # → [True, True, True]
```

---

## 14. Lua Scripts

Lua скрипти виконуються всередині Redis **атомарно** — ніяка інша команда не виконується поки скрипт працює.

```bash
# EVAL script numkeys key [key ...] arg [arg ...]
EVAL "return 'Hello from Lua'" 0

EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey myvalue

# Складніший приклад — getdel вручну
EVAL "
local val = redis.call('GET', KEYS[1])
redis.call('DEL', KEYS[1])
return val
" 1 oauth_state:abc
```

### EVALSHA — виконати скрипт за хешем

```bash
# Завантажити скрипт
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
→ "e0e1f9fabfa9d353e7d65e1f93bdcf0272c12988"  (SHA1 хеш)

# Виконати за хешем (швидше ніж передавати скрипт кожного разу)
EVALSHA e0e1f9fabfa9d353e7d65e1f93bdcf0272c12988 1 mykey

# Перевірити чи скрипт існує
SCRIPT EXISTS sha1 sha2
→ [1, 0]  (перший є, другий ні)

# Очистити кеш скриптів
SCRIPT FLUSH
```

### Приклад: атомарний distributed lock

```lua
-- Acquire lock
local key = KEYS[1]
local owner = ARGV[1]
local ttl = ARGV[2]

local result = redis.call('SET', key, owner, 'NX', 'EX', ttl)
if result then
    return 1  -- lock acquired
else
    return 0  -- lock already held
end
```

```python
LOCK_SCRIPT = """
local key = KEYS[1]
local owner = ARGV[1]
local ttl = tonumber(ARGV[2])
local result = redis.call('SET', key, owner, 'NX', 'EX', ttl)
if result then return 1 else return 0 end
"""

UNLOCK_SCRIPT = """
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
"""

# Acquire
result = await redis.eval(LOCK_SCRIPT, 1, "lock:resource", "owner_id", 30)
if result == 1:
    # критична секція
    pass

# Release (тільки якщо ми власники!)
await redis.eval(UNLOCK_SCRIPT, 1, "lock:resource", "owner_id")
```

---

## 15. Pub/Sub — публікація і підписка

Pub/Sub — це **messaging** патерн де:
- **Publisher** публікує повідомлення в **канал**
- **Subscriber** підписується на канал і отримує всі повідомлення

> **Важливо:** Pub/Sub в Redis — це **fire and forget**. Якщо підписник не підключений в момент публікації — повідомлення **загубиться**. Для надійної доставки використовуй Redis Streams.

```bash
# Підписатись на канал (блокуюча операція)
SUBSCRIBE notifications
# → ["subscribe", "notifications", 1]  (підписано на 1 канал)

# Підписатись на кілька каналів
SUBSCRIBE ch1 ch2 ch3

# Підписатись за патерном
PSUBSCRIBE user:*           # всі канали що починаються з user:
PSUBSCRIBE user:* news:*    # кілька патернів

# Відписатись
UNSUBSCRIBE notifications
PUNSUBSCRIBE user:*

# Список активних підписок
PUBSUB CHANNELS             # всі активні канали
PUBSUB CHANNELS user:*      # за патерном
PUBSUB NUMSUB ch1 ch2       # кількість підписників на кожен канал
PUBSUB NUMPAT               # кількість патернів
```

```bash
# В окремому з'єднанні — публікація
PUBLISH notifications '{"type": "alert", "message": "Hello!"}'
→ 3  (кількість підписників що отримали повідомлення)
```

```python
# Publisher
async def publish(redis, channel: str, data: dict):
    message = json.dumps(data)
    subscribers = await redis.publish(channel, message)
    return subscribers  # кількість отримувачів

# Subscriber
async def subscribe(redis, channels: list[str]):
    pubsub = redis.pubsub()
    await pubsub.subscribe(*channels)
    
    async for message in pubsub.listen():
        # message = {
        #     "type": "message" | "subscribe" | "unsubscribe",
        #     "pattern": None,
        #     "channel": "notifications",
        #     "data": '{"type": "alert", ...}'
        # }
        if message["type"] == "message":
            data = json.loads(message["data"])
            await handle_message(data)

# Pattern subscriber
async def pattern_subscribe(redis):
    pubsub = redis.pubsub()
    await pubsub.psubscribe("user:*")
    
    async for message in pubsub.listen():
        if message["type"] == "pmessage":
            print(f"Channel: {message['channel']}")
            print(f"Pattern: {message['pattern']}")
            print(f"Data: {message['data']}")
```

---

## 16. Persistence — як Redis зберігає дані

Redis — in-memory, але може зберігати дані на диск. Два механізми:

### RDB — Redis Database (snapshot)

Redis робить **знімок** всіх даних і записує в бінарний файл `dump.rdb`.

```
┌──────────────────────────────────────────────────────┐
│                    RDB Snapshot                      │
│                                                      │
│  RAM: [all data]  ──→  fork()  ──→  write dump.rdb  │
│                         ↑                           │
│              дочірній процес (не блокує Redis)       │
└──────────────────────────────────────────────────────┘
```

**Налаштування в redis.conf:**
```
# save <seconds> <changes>
save 3600 1     # зберегти якщо за 1 год відбулась ≥1 зміна
save 300 100    # зберегти якщо за 5 хв відбулось ≥100 змін
save 60 10000   # зберегти якщо за 1 хв відбулось ≥10000 змін

# Вимкнути RDB:
save ""

dbfilename dump.rdb
dir /var/lib/redis
```

**Ручний snapshot:**
```bash
SAVE       # синхронно (блокує Redis)
BGSAVE     # асинхронно (fork + запис у фоні)
LASTSAVE   # Unix timestamp останнього успішного SAVE
```

**Плюси RDB:**
- Компактний бінарний формат
- Швидке відновлення (один файл)
- Мінімальний вплив на продуктивність

**Мінуси RDB:**
- Можлива втрата даних між snapshot-ами (до 5 хвилин)
- Великий fork() може бути дорогим при великому обсязі даних

---

### AOF — Append Only File

AOF записує **кожну команду** що модифікує дані в файл `appendonly.aof`.

```
SET foo bar       ─→  *3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n
INCR counter      ─→  *2\r\n$4\r\nINCR\r\n$7\r\ncounter\r\n
...
```

**Налаштування:**
```
appendonly yes
appendfilename "appendonly.aof"

# Частота fsync (запису на диск):
appendfsync always    # кожну команду — повна безпека, повільно
appendfsync everysec  # кожну секунду — баланс (за замовчуванням, рекомендовано)
appendfsync no        # коли ОС вирішить — швидко, але не безпечно
```

**AOF Rewrite — стиснення файлу:**

З часом AOF файл розростається. Redis може його переписати, зберігши лише мінімальний набір команд для відновлення поточного стану:

```bash
BGREWRITEAOF   # запустити rewrite у фоні
```

```
# Автоматичний rewrite:
auto-aof-rewrite-percentage 100   # якщо файл виріс удвічі
auto-aof-rewrite-min-size 64mb    # і більше 64 MB
```

**Плюси AOF:**
- Менша втрата даних (максимум 1 секунда)
- Людиночитабельний формат

**Мінуси AOF:**
- Більший розмір файлу
- Повільніший restart (потрібно "програти" всі команди)

---

### RDB + AOF разом (рекомендовано)

```
# redis.conf
save 3600 1
save 300 100
appendonly yes
appendfsync everysec
```

Якщо обидва увімкнені — при restart Redis використовує AOF (він повніший).

---

## 17. Eviction Policies — що робити коли закінчується пам'ять

```
maxmemory 256mb
maxmemory-policy <policy>
```

| Policy | Опис |
|---|---|
| `noeviction` | Не видаляти нічого, повертати error при записі (за замовчуванням) |
| `allkeys-lru` | Видаляти найдавніше використані ключі (з усіх) |
| `volatile-lru` | Видаляти найдавніше використані ключі (тільки з TTL) |
| `allkeys-lfu` | Видаляти найменш часто використані (з усіх) [Redis 4.0+] |
| `volatile-lfu` | Видаляти найменш часто використані (тільки з TTL) |
| `allkeys-random` | Видаляти випадкові ключі (з усіх) |
| `volatile-random` | Видаляти випадкові ключі (тільки з TTL) |
| `volatile-ttl` | Видаляти ключі з найменшим TTL першими |

**Рекомендовані:**
- **Кеш (не критичні дані):** `allkeys-lru` або `allkeys-lfu`
- **Сесії (важливі дані з TTL):** `volatile-lru`
- **Критичні дані без TTL:** `noeviction`

```python
# Перевірити скільки пам'яті використовується
redis-cli INFO memory | grep used_memory_human
# used_memory_human: 2.50M
```

---

## 18. Redis Databases — числові БД

Redis підтримує кілька логічних баз даних (0-15 за замовчуванням):

```bash
SELECT 0   # переключитись на БД 0 (за замовчуванням)
SELECT 1   # переключитись на БД 1
SELECT 15  # максимальна за замовчуванням

# Кожна БД — окремий keyspace
SELECT 0
SET foo "bar"

SELECT 1
GET foo    → nil   # БД 1 не бачить ключів БД 0
```

**У URL:**
```
redis://localhost:6379/0   # БД 0
redis://localhost:6379/1   # БД 1
```

**Рекомендовані стратегії:**
```
БД 0: production дані (кеш, сесії)
БД 1: тести
БД 2: dev environment
```

```bash
# Очистити поточну БД
FLUSHDB

# Очистити ВСІ БД
FLUSHALL

# Перемістити ключ в іншу БД
MOVE mykey 1   → 1  (переміщено в БД 1)
```

---

## 19. Конфігурація redis.conf

```ini
# ─── Мережа ────────────────────────────────────────────────────
bind 127.0.0.1           # слухати тільки localhost (безпечно)
# bind 0.0.0.0           # слухати всі інтерфейси (для Docker)
port 6379
# unixsocket /tmp/redis.sock   # Unix socket замість TCP (швидше локально)

# ─── Безпека ───────────────────────────────────────────────────
requirepass your_strong_password   # пароль
rename-command FLUSHALL ""         # заборонити небезпечні команди
rename-command DEBUG ""
rename-command CONFIG ""

# ─── Пам'ять ───────────────────────────────────────────────────
maxmemory 256mb
maxmemory-policy allkeys-lru

# ─── Persistence (RDB) ─────────────────────────────────────────
save 3600 1
save 300 100
save 60 10000
dbfilename dump.rdb
dir /var/lib/redis

# ─── Persistence (AOF) ─────────────────────────────────────────
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# ─── Логування ─────────────────────────────────────────────────
loglevel notice          # debug | verbose | notice | warning
logfile /var/log/redis/redis-server.log

# ─── Клієнти ───────────────────────────────────────────────────
maxclients 10000         # максимум клієнтів
timeout 300              # disconnect після 300с бездіяльності
tcp-keepalive 300        # TCP keepalive

# ─── Slowlog ───────────────────────────────────────────────────
slowlog-log-slower-than 10000   # логувати команди > 10мс (мікросекунди)
slowlog-max-len 128             # зберігати останні 128 записів

# ─── Оптимізація структур ──────────────────────────────────────
hash-max-listpack-entries 128
hash-max-listpack-value 64
list-max-listpack-size -2       # max 8KB per node
set-max-intset-entries 512
zset-max-listpack-entries 128
zset-max-listpack-value 64
```

---

## 20. Патерни використання Redis

### Патерн 1: Distributed Lock

```bash
# Захопити lock (SET NX EX — атомарна операція)
SET lock:payment:order_99 "server1:process123" NX EX 30
# → OK    (lock захоплено)
# → nil   (lock вже зайнятий)

# Звільнити lock ТІЛЬКИ якщо ти власник (через Lua!)
# Не можна: GET + DEL (між ними може зайти інший)
```

### Патерн 2: Sliding Window Rate Limit

```
Ключ: rate:user:42
Тип: Sorted Set
Score: Unix timestamp
Member: унікальний ID запиту

- Видаляємо всі entries старіші за вікно
- Рахуємо скільки залишилось
- Якщо < ліміт → додаємо поточний → дозволяємо
- Якщо ≥ ліміт → відмовляємо
```

### Патерн 3: Cache-Aside

```
1. Запит до кешу → є? → повернути
2. Запит до кешу → немає?
3. Запит до БД
4. Зберегти в кеш з TTL
5. Повернути
```

### Патерн 4: Write-Through Cache

```
1. Запис в БД
2. Одночасно оновити кеш
Гарантує консистентність між БД і кешем
```

### Патерн 5: Session Token Blacklist

```
При logout додаємо JWT токен в blacklist:
SET blacklist:{jti} "1" EX {token_remaining_ttl}

При кожному запиті перевіряємо:
EXISTS blacklist:{jti} → 1 = відхилити, 0 = дозволити
```

### Патерн 6: Idempotency Key

```
Для платежів/важливих операцій:
SET idempotency:{key} "{result}" NX EX 86400
→ OK   = перший запит, виконати
→ nil  = дублікат, повернути збережений результат
```

### Патерн 7: Pub/Sub Fan-out

```
Подія → PUBLISH channel data
         ↓        ↓        ↓
   Worker1   Worker2   Worker3
(кожен отримує копію повідомлення)
```

### Патерн 8: Leaderboard

```
ZADD  leaderboard {score} {user_id}    ← оновити score
ZINCRBY leaderboard {delta} {user_id}  ← додати до score
ZREVRANK leaderboard {user_id}          ← позиція в рейтингу
ZREVRANGE leaderboard 0 9 WITHSCORES   ← топ-10
```

---

## Шпаргалка: яку структуру вибрати

```
┌─────────────────────────────────────────────────────────────────┐
│                   Яку структуру вибрати?                        │
├─────────────────────────────────────────────────────────────────┤
│  ЗАДАЧА                          │  СТРУКТУРА                   │
├──────────────────────────────────┼──────────────────────────────┤
│  Кеш відповіді API               │  String (JSON)               │
│  OAuth state / CSRF token        │  String (NX + EX)            │
│  Сесія юзера                     │  String (JSON + EX)          │
│  Лічильник (views, likes)        │  String (INCR)               │
│  Distributed lock                │  String (SET NX EX)          │
│  Зберегти об'єкт юзера           │  Hash                        │
│  Атомічно оновити поле об'єкта   │  Hash (HINCRBY)              │
│  Черга задач (FIFO)              │  List (RPUSH + BLPOP)        │
│  Стек                            │  List (LPUSH + LPOP)         │
│  Останні N елементів             │  List (RPUSH + LTRIM)        │
│  Унікальні теги/категорії        │  Set                         │
│  Спільні друзі                   │  Set (SINTER)                │
│  Активні юзери                   │  Set (SADD/SISMEMBER)        │
│  Rate limiting                   │  Sorted Set (score=timestamp)│
│  Leaderboard / рейтинг           │  Sorted Set (score=points)   │
│  Пріоритетна черга               │  Sorted Set (score=priority) │
│  Delayed jobs                    │  Sorted Set (score=exec_time)│
│  Pub/Sub сповіщення              │  Pub/Sub channels            │
└──────────────────────────────────┴──────────────────────────────┘
```

---

*Наступний документ: Redis в FastAPI — підключення, dependency injection, repositories, OAuth state, кешування та rate limiting.*

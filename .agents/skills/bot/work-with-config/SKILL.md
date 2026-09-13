---
name: work-with-config
description: >-
  Use when adding, changing, reviewing, or debugging my-study-bot config:
  load_dotenv in main.py / app/database.py, TG_TOKEN / DB_URL, win32
  AiohttpSession SSL+IPv4 bypass vs clean Bot(token=…), Dispatcher /
  DbSessionMiddleware / init_db / include_router / scheduler startup-shutdown
  hooks / polling startup, or keeping business logic out of main.py. Not for
  handler product flows, expiry window math, or Docker deploy secrets alone.
---

# Work With Config

Use this skill for **runtime configuration and process startup** in
`my-study-bot` (Telegram long-polling bot, aiogram v3).

There is **no** HTTP server config, CORS, or brand merge. Contour values and
secrets live in `.env` (loaded by `python-dotenv`); process wiring lives in
`main.py`; DB URL and engine live in `app/database.py`.

Stack: aiogram 3.22 (`Bot`, `Dispatcher`, polling), `python-dotenv`,
SQLAlchemy 2.0 + `aiosqlite`, Python 3.13 in Docker (`python:3.13-slim`).

Entry: `main.py` → `asyncio.run(main())` → `dp.start_polling(bot)`. Prefer
keeping Windows-only session hacks inside the `sys.platform == "win32"`
branch. Prefer keeping business logic out of `main.py` (handlers,
keyboards, states, models).

**Temp skills path:** this skill lives under `.agents/skills/bot/` in this
package for now. Canonical skills are intended to live in
`my-study-bot-meta/.agents/skills/bot/` once meta is available — prefer
that path when choosing skills if it exists.

Related package: `../my-study-bot-meta` (docs, OpenSpec, skills).

## Quick Reference

| Case | Preferred pattern |
| --- | --- |
| Env load | `load_dotenv()` in `main.py` **and** `app/database.py` |
| Secrets / contour | Env only — `TG_TOKEN`, `DB_URL`; never hardcode; do not commit `.env` / `.env.server` |
| Bot construction | `win32`: custom `AiohttpSession` (SSL verify off + IPv4); else `Bot(token=…)` |
| SSL bypass | **Local Windows VPN/debug only** — never copy into Linux / Docker / production |
| Startup wiring | `Dispatcher` → `DbSessionMiddleware` → `init_db()` → `include_router` → hooks (`start_scheduler` / `stop_scheduler`) → `start_polling` |
| Business logic | Handlers / keyboards / states / models / `app/scheduler.py` — **not** only inside `main.py` |
| DB default | `DB_URL` default `sqlite+aiosqlite:///data/db.sqlite3` |

## How Env Loading Works

```
.env  (gitignored)
   ↓
load_dotenv() in main.py          → os.getenv("TG_TOKEN") for Bot
load_dotenv() in app/database.py  → os.getenv("DB_URL", default …) for engine
   ↓
Dispatcher + middleware + init_db + router + polling
```

Rules:

1. Call `load_dotenv()` in both `main.py` (inside `main()`) and at module
   level in `app/database.py` so token and DB URL resolve regardless of
   import order.
2. Production secrets stay on the VPS (Compose `env_file: .env`) — do **not**
   commit real values; `.env` and `.env.server` are gitignored.
3. Read `os.getenv` at wiring sites (`main.py`, `app/database.py`). Do not
   invent a second config module unless product needs grow beyond these two.
4. No committed `.env.example` yet — document new vars in `AGENTS.md` Config
   table; add `.env.example` when convenient.

## Env Vars

| Variable | Role |
| --- | --- |
| `TG_TOKEN` | Telegram Bot API token (required) |
| `DB_URL` | SQLAlchemy async URL (default `sqlite+aiosqlite:///data/db.sqlite3`) |

`data/` is gitignored and mounted in Compose (`./data:/app/data`) so SQLite
survives container restarts.

### Env vs `main.py` / `database.py`

| Belongs in **env** | Belongs in **code** |
| --- | --- |
| `TG_TOKEN`, `DB_URL` | `main.py`: Bot/session branch, Dispatcher, middleware, `init_db`, router, polling |
| Anything that must change without a code change | `app/database.py`: engine, `async_session`, models, `init_db()` |
| | Platform gate: `sys.platform == "win32"` session hack vs clean Bot |

## Platform Session (Windows vs Linux/Docker)

| Contour | Construction |
| --- | --- |
| Local Windows (`sys.platform == "win32"`) | Custom `AiohttpSession`: SSL `check_hostname=False`, `verify_mode=CERT_NONE`, connector `family=AF_INET`; `Bot(token=…, session=session)` |
| Linux / Docker / VPS | Clean `Bot(token=os.getenv("TG_TOKEN"))` — default session |

Rules:

1. Windows SSL verify off + IPv4 is **ONLY** for local VPN/debug.
2. **Never** copy the Windows SSL bypass into Linux, Docker images, or
   production entrypoints.
3. Keep the bypass inside the `if sys.platform == "win32":` branch; the
   `else` branch stays the clean Bot constructor.

```python
if sys.platform == "win32":
    # Local VPN/debug only — SSL verify off + IPv4
    session = AiohttpSession()
    session._connector_init.update({
        "family": socket.AF_INET,
        "ssl": ssl_context,  # CERT_NONE
    })
    bot = Bot(token=os.getenv("TG_TOKEN"), session=session)
else:
    bot = Bot(token=os.getenv("TG_TOKEN"))
```

## `main.py` Startup Shape

Order inside `async def main()`:

1. `load_dotenv()`
2. Build `Bot` (platform branch above)
3. `dp = Dispatcher()`
4. `dp.update.middleware(DbSessionMiddleware())`
5. `await init_db()`
6. `dp.include_router(router)` from `app.handlers`
7. Register `startup` / `shutdown` hooks (`start_scheduler(bot)` /
   `stop_scheduler()` from `app.scheduler`)
8. `await dp.start_polling(bot)`

Process entry:

```python
if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("Bot stopped by user...")
```

Prefer thin `main.py`: no study/subscription product logic or expiry loops
here — put them in `app/handlers.py`, `app/keyboards.py`, `app/states.py`,
`app/database.py`, `app/scheduler.py`.

## Patterns for Changing Config

### 1. New env variable

1. Document in `AGENTS.md` Config table (and `.env.example` when added).
2. Set in local `.env` / server `.env` (not committed secrets).
3. Read `os.getenv("MY_VAR")` at the wiring site (`main.py` or
   `app/database.py`).
4. Keep `load_dotenv()` calls so the var is available.

### 2. Change Bot / session construction

1. Edit only the platform branch that applies.
2. Do **not** enable SSL bypass outside `win32`.
3. Smoke: local `python main.py` on Windows; Docker/Linux must use clean Bot.

### 3. Change startup wiring

1. Keep middleware → `init_db` → `include_router` → polling order unless there
   is a deliberate reason to change it.
2. New routers: `include_router` in `main.py`; handler bodies stay in `app/`.
3. Do not move product flows into `main.py`.

### 4. Change DB URL / engine

1. Prefer `DB_URL` env override; keep the SQLite default for local/Compose.
2. Engine and `async_session` stay in `app/database.py`.
3. `init_db()` runs `create_all` then `_ensure_sqlite_user_columns` (DDL in
   `_SQLITE_USER_COLUMN_DDL`). New `User` columns must be registered there so the
   VPS volume gets ALTER on startup; deploy does not wipe SQLite.

## Examples

### Env load at both wiring sites

```python
# main.py (inside main())
load_dotenv()
bot = Bot(token=os.getenv("TG_TOKEN"))  # or win32 session branch

# app/database.py (module level)
load_dotenv()
DB_URL = os.getenv("DB_URL", "sqlite+aiosqlite:///data/db.sqlite3")
```

### Keep business logic out of `main.py`

```python
# main.py — wiring only
dp.update.middleware(DbSessionMiddleware())
await init_db()
dp.include_router(router)

# app/handlers.py — product behavior
@router.message(Command("start"))
async def cmd_start(message: Message, session: AsyncSession):
    ...
```

### Add a secret

1. Document `MY_SECRET` in `AGENTS.md` (and `.env.example` when present).
2. Set value on the deploy contour only.
3. Read `os.getenv("MY_SECRET")` where needed — never commit the real value.

## Common Mistakes

| Mistake | Why it hurts | Fix |
| --- | --- | --- |
| SSL bypass in Docker/Linux | Insecure production Telegram TLS | Keep bypass inside `win32` only |
| Business logic only in `main.py` | Hard to test; fights modular routers | Handlers / keyboards / states / models |
| Missing `load_dotenv()` in one of the two sites | Token or DB URL silently wrong/empty | Call in both `main.py` and `database.py` |
| Hardcoded `TG_TOKEN` / `DB_URL` | Leak via git; wrong contour | Env + defaults only |
| Committing `.env` / `.env.server` | Credential leak | Keep gitignored; secrets on VPS only |
| Skipping `init_db` before polling | Missing tables on first run | Await `init_db()` before `start_polling` |
| Forgetting `DbSessionMiddleware` | Handlers lack `session` | Register on `dp.update` before routers |
| Second config framework without need | Split sources of truth | Env + existing two load sites |

## Checklist

When changing or reviewing config-related work:

- [ ] `TG_TOKEN` / `DB_URL` (and any new knobs) documented; no real secrets committed
- [ ] `load_dotenv()` still present in `main.py` and `app/database.py`
- [ ] Windows SSL+IPv4 bypass remains `win32`-only; Linux/Docker stay clean Bot
- [ ] Startup still: middleware → `init_db` → `include_router` → polling
- [ ] Product behavior not dumped into `main.py`
- [ ] Compose still mounts `./data` and uses `env_file: .env` when deploy-relevant
- [ ] Agent can still run `python main.py` / `pip install -r requirements.txt` from package root

## Agent Workflow

1. Confirm whether the task is env, Bot/session platform branch, or startup
   wiring (`Dispatcher` / middleware / DB / router / polling).
2. Edit only the files the change requires (see map below).
3. Run Python commands from the bot package root; fix failures before claiming
   done.

Typical commands (agent runs):

```bash
pip install -r requirements.txt
python main.py
```

Docker smoke when image/compose changed:

```bash
docker compose build
docker compose up -d
```

## Where Things Live

| Concern | Location |
| --- | --- |
| Env files | `.env`, `.env.server` (gitignored); no `.env.example` yet |
| Env load | `load_dotenv()` in `main.py`, `app/database.py` |
| Process entry / Bot / polling | `main.py` |
| DB URL, engine, models, `init_db` | `app/database.py` |
| Session injection | `app/middlewares.py` (`DbSessionMiddleware`) |
| Routers / commands | `app/handlers.py` |
| High-level notes | `AGENTS.md` → Config And Env / How Startup Is Organized |
| Deploy cwd / Compose | `docker-compose.yml`, `.github/workflows/deploy.yml` |

Related skills (by name only): `work-with-database`, `work-with-middleware`,
`work-with-handlers`, `work-with-structure` (when present under bot skills).

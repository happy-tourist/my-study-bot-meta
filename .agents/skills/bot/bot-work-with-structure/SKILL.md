---
name: bot-work-with-structure
description: >-
  Use when placing or moving code in the my-study-bot Telegram package: main.py
  entry, handlers/routers, SQLAlchemy models, DbSessionMiddleware, keyboards,
  FSM states, app/scheduler.py APScheduler expiry jobs, data/sqlite, tests/,
  Docker/Compose/CI, or deciding where command logic vs DB vs shared UI belong.
  Prefer extending User + injected session; no parallel data path.
---

# Work With Structure

## Overview

Use this skill when creating or relocating code under the bot package root (and
related `data/` / Docker / deploy wiring). This is a **Telegram long-polling
bot** (aiogram v3), not an HTTP API server: Telegram updates flow through the
Dispatcher; handlers own UX replies; SQLite is the source of truth for users /
subscription fields.

Stack: aiogram 3.22 (`Router`, `Dispatcher`, polling), `python-dotenv`,
SQLAlchemy 2.0 + `aiosqlite`, Python `3.13` in Docker (`python:3.13-slim`).

Entry: `main.py` → `asyncio.run(main())` → `dp.start_polling(bot)`. Build
`Bot` / `Dispatcher`, attach middleware, `init_db()`, include router — prefer
not stuffing business logic into `main.py`.

Path style: package imports from repo root, e.g.
`from app.handlers import router`, `from app.database import User`.

**Temp skills path:** this skill lives under
`.agents/skills/bot/` in this package for now. Canonical skills are
intended to live in `my-study-bot-meta/.agents/skills/bot/` once meta
is available — prefer that path when choosing skills if it exists.

Sibling meta: resolve via `project-map.md` key `my-study-bot-meta`
(`../my-study-bot-meta`) for OpenSpec / docs / skills.

## Core Rules

1. Before adding folders, inspect a nearby peer in the **same layer** and match its file name, export style, and import path.
2. Create only the files the feature needs. Do not add empty stubs “for later” (`app/states.py` is still a stub — extend it; `app/keyboards.py` already has topic-menu builders — extend those).
3. Place code by layer role (entry vs middleware vs handlers vs DB vs keyboards vs FSM).
4. Respect **allowed dependency direction** (see below). Never invert layers.
5. **Handlers** own command/callback UX and orchestration. **database** owns models + engine/session factory. **middlewares** inject `session` only.
6. Prefer extending the existing `User` model and middleware session injection rather than inventing a parallel data path.
7. Keep Windows SSL/IPv4 session hacks inside `sys.platform == "win32"` in `main.py` — do not copy them into Docker/Linux/production.
8. Keep Russian user-facing strings consistent with existing replies unless product copy is being redesigned.

## Folder Map

| Layer | Path | Role |
|-------|------|------|
| Entry | `main.py` | `Bot`, `Dispatcher`, polling, win32 session; wire middleware + router |
| Handlers | `app/handlers.py` | Routers / commands / callbacks (thin orchestration) |
| Database | `app/database.py` | Engine, `async_session`, `User` model, `init_db()` |
| Middleware | `app/middlewares.py` | `DbSessionMiddleware` injects `session: AsyncSession` |
| Keyboards | `app/keyboards.py` | Inline topic-menu builders (`main_menu_kb`, `back_to_menu_kb`, `menu:*`; temp `subscription_kb`) |
| Scheduler | `app/scheduler.py` | APScheduler expiry job (`check_subscriptions`); start/stop from `main` hooks |
| FSM | `app/states.py` | aiogram FSM states / groups |
| Tests | `tests/` | pytest + pytest-asyncio (`test_subscription_expiry.py`, …) |
| Runtime DB | `data/` | SQLite file (gitignored; Compose volume `./data:/app/data`) |
| Deploy | `Dockerfile`, `docker-compose.yml`, `.github/workflows/` | Image + VPS compose deploy |

Env: `.env` / `.env.server` (gitignored). Vars: `TG_TOKEN` (required), `DB_URL` (default `sqlite+aiosqlite:///data/db.sqlite3`). No committed `.env.example` yet — document when adding.

### Layer roles (detail)

| Layer | Owns | Does not own |
|-------|------|--------------|
| **`main.py`** | Process entry, platform Bot session, middleware registration, `init_db`, include router, startup/shutdown hooks (incl. scheduler) | Command replies, queries, FSM steps, keyboard markup, expiry loops |
| **`handlers.py`** | Filters, handlers, greetings / study flow UX, commit via injected session | Engine creation; second ORM session factory; SSL hacks; cron jobs |
| **`database.py`** | `User` columns, engine/URL, `async_session`, `init_db` / `create_all` | Telegram replies; Router registration |
| **`middlewares.py`** | Open/close session per update; put `session` in handler `data` | Business rules; user upsert logic; background jobs |
| **`keyboards.py`** | Shared reply/inline builders | DB access; long handler bodies |
| **`scheduler.py`** | APScheduler job, reminder/expire windows, `Bot.send_message` for expiry DMs | Router handlers; middleware session lifecycle |
| **`states.py`** | FSM state groups for multi-step dialogs | Persistence; Telegram send calls |
| **`data/`** | Runtime SQLite file | Source code; secrets |
| **Docker / CI** | Image, compose, GHCR push + SSH deploy | Product handlers |

## Dependency Direction

Typical path:

```text
Telegram update → Dispatcher middleware → handler → session/DB
keyboards / states ← handlers (shared UI)
main.py → Bot / Dispatcher / middleware / init_db / include_router
```

Allowed:

```text
main.py        →  app.handlers, app.database, app.middlewares, app.scheduler (wiring only)
middlewares    →  app.database (async_session factory)
handlers       →  app.database (models), app.keyboards, app.states; session via injection
scheduler      →  app.database (async_session / User), aiogram Bot; own session in job
keyboards      →  aiogram types only (no DB)
states         →  aiogram FSM only (no DB / no handlers)
database       →  SQLAlchemy / aiosqlite / dotenv (no aiogram handlers)
```

**Forbidden inversions:**

| Wrong | Why |
|-------|-----|
| `database` → `handlers` / keyboards | Models must stay free of Telegram UX |
| `keyboards` / `states` → `database` | Shared UI is not persistence |
| Growing `main.py` with command logic | Keep entry/wiring only; put UX in handlers |
| New SQLite/ORM path beside `User` + middleware session | One data path: injected `AsyncSession` |
| Copying win32 SSL bypass into Dockerfile / Linux branch | Production uses clean `Bot(token=…)` |
| Fat “services” layer that reopens its own sessions | Use middleware-injected session |

## Where New Code Belongs

Decide in this order:

1. **New command / callback?** → `app/handlers.py` (or split routers later under `app/` matching peers); register via existing `router` included from `main.py`.
2. **Need DB in handler?** → declare `session: AsyncSession`; middleware injects it — do not open a second session.
3. **New user / subscription field?** → extend `User` in `app/database.py`; migrate carefully if SQLite already has rows (`create_all` does not alter columns).
4. **Reply / inline keyboard?** → builder in `app/keyboards.py`; import from handlers.
5. **Multi-step dialog?** → states in `app/states.py`; handlers use `FSMContext`.
6. **Startup wiring (middleware, router, init_db, scheduler start/stop)?** → `main.py` only for registration — not business replies or expiry loops.
7. **Background subscription expiry / cron?** → `app/scheduler.py`; wire `start_scheduler` / `stop_scheduler` from `main` hooks only.
8. **Env / DB URL?** → `.env` locally; Compose `env_file: .env` on VPS; default SQLite under `data/`.
9. **Deploy / image?** → `Dockerfile`, `docker-compose.yml`, `.github/workflows/` — not product logic.
10. **Tests?** → `tests/` with pytest + pytest-asyncio; cover expiry scheduler and registration / gates.

### Handlers vs database vs UI — what belongs where

**Put in handlers**

- `/start` and future study/subscription commands and callbacks.
- Upsert / read `User` via injected `session`.
- Choose reply text and attach keyboards; drive FSM transitions.

**Put in database**

- `User` model: `id` (Telegram BigInteger PK), `username`, `subscription_end`, `is_active`, `created_at`.
- Engine, `async_session`, `init_db()`.

**Put in middlewares**

- Per-update session lifecycle and injection into handler `data`.

**Put in keyboards / states**

- Reusable markup builders; FSM groups for forms / multi-step flows.

**Put in scheduler**

- APScheduler job registration, reminder/expire window math, background DMs and `is_active` deactivation.
- Own `async_session` (or test `session_factory`) — not middleware-injected `session`.

**Put in main.py**

- `load_dotenv`, Bot construction (win32 branch vs default), Dispatcher, middleware, `init_db`, `include_router`, startup/shutdown scheduler hooks, polling.

### What NOT to put

| Avoid | Prefer |
|-------|--------|
| Business replies inside `main.py` | `app/handlers.py` |
| Opening `AsyncSession` ad hoc in handlers | Injected `session` from middleware |
| Second user table / parallel SQLite helper | Extend `User` + existing engine |
| Keyboard markup inline duplicated everywhere | `app/keyboards.py` |
| FSM state strings scattered as magic values | `app/states.py` |
| SSL verify disable outside win32 local debug | Clean Bot session on Linux/Docker |
| Committing `.env` / `data/db.sqlite3` | gitignore; secrets on VPS only |
| Empty HTTP FastAPI “BFF” beside the bot | This package is polling-only |

## Naming

| Kind | Convention | Examples |
|------|------------|----------|
| Package modules | snake_case under `app/` | `handlers.py`, `database.py` |
| Models / classes | PascalCase | `User`, `DbSessionMiddleware` |
| Handlers | `async def` + command prefix | `cmd_start` |
| Router export | module-level `router = Router()` | imported in `main.py` |
| FSM | StatesGroup subclasses in `states.py` | match peer style when added |
| Keyboards | functions returning markup | `app.keyboards as kb` |
| Env vars | `SCREAMING_SNAKE` | `TG_TOKEN`, `DB_URL` |
| Deploy | Compose service `bot`; image `ghcr.io/.../my-study-bot:latest` | workflow on `main` |

Match existing peers; keep a single included router until a real split is needed.

## Typical Shapes

### Entry

```text
main.py    # Bot, Dispatcher, middleware, init_db, include_router, start_polling
```

### App package

```text
app/
├── handlers.py      # Router + commands/callbacks
├── database.py      # engine, async_session, User, init_db
├── middlewares.py   # DbSessionMiddleware
├── keyboards.py     # markup builders
├── scheduler.py     # APScheduler expiry job
└── states.py        # FSM StatesGroup stubs / groups
```

### Outside app

```text
tests/                       # pytest suite (subscription expiry, …)
data/db.sqlite3              # runtime (gitignored)
Dockerfile
docker-compose.yml           # service bot, env_file, volume ./data:/app/data
.github/workflows/deploy.yml # GHCR build/push + SSH compose
requirements.txt             # includes apscheduler
```

## Real Composition Examples

**Startup** — `main.py` loads dotenv, builds Bot (win32 custom session else default), `Dispatcher`, `dp.update.middleware(DbSessionMiddleware())`, `await init_db()`, `dp.include_router(router)`, startup → `start_scheduler(bot)`, shutdown → `stop_scheduler()`, `start_polling`.

**Handler + DB** — `cmd_start(message, session: AsyncSession)` selects `User` by Telegram id; creates row on first visit; Russian greet strings.

**Middleware** — opens `async_session`, sets `data["session"]`, closes after handler.

**Deploy** — push `main` → build/push GHCR → SSH `/home/deploy/my-study-bot` → `docker compose pull && up -d`.

## Creating New Pieces — Checklist

**New command / callback**

1. Add handler on `router` in `app/handlers.py` (or a new router module included from `main.py`).
2. If DB needed, take `session: AsyncSession` — do not create your own.
3. Reuse builders from `app/keyboards.py` when markup is shared.
4. Keep copy in Russian consistent with existing replies.

**New model field / entity**

1. Extend `User` (or add a related model) in `app/database.py`.
2. Plan SQLite migration — `create_all` won’t alter existing tables.
3. Read/write only through the injected session in handlers.

**New keyboard**

1. Add builder in `app/keyboards.py`.
2. Import from handlers; avoid duplicating `InlineKeyboardMarkup` trees.

**New FSM flow**

1. Define StatesGroup in `app/states.py`.
2. Handlers set/clear state via `FSMContext`; persist durable data via session, not only FSM memory.

**Deploy / env change**

1. Touch Dockerfile / compose / workflow only for runtime process.
2. Keep secrets in server `.env`; volume preserves `data/`.

## Domain Anchors

| Domain | Primary paths | Notes |
|--------|---------------|-------|
| Process entry | `main.py` | polling; win32 SSL/IPv4 local only |
| Commands / UX | `app/handlers.py` | thin; session injected |
| Users / subscription columns | `app/database.py` `User` | extend, don’t fork |
| Session injection | `app/middlewares.py` | per update |
| Shared markup | `app/keyboards.py` | builders |
| Multi-step dialogs | `app/states.py` | FSM |
| Subscription expiry cron | `app/scheduler.py` | APScheduler; day windows + 10:00 Moscow |
| Tests | `tests/` | pytest-asyncio |
| SQLite file | `data/` | gitignored; Compose volume |
| VPS runtime | `docker-compose.yml` | `/home/deploy/my-study-bot` |
| Meta / OpenSpec / skills | `my-study-bot-meta` via `project-map.md` | canonical docs/skills |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Growing `main.py` with `/start` or study logic | Move to `app/handlers.py` |
| `session = async_session()` inside every handler | Use middleware-injected `session` |
| New sqlite3/`requests` persistence beside SQLAlchemy | Extend `User` + existing engine |
| Keyboard/FSM logic mixed into `database.py` | Keep models persistence-only |
| Putting markup only inside one handler forever | Extract to `app/keyboards.py` when reused |
| Copying win32 SSL bypass into Docker image | Linux branch: plain `Bot(token=…)` |
| Committing `.env` or `data/db.sqlite3` | Keep gitignored; secrets on VPS |
| Assuming `create_all` migrates columns | Plan explicit SQLite migration |
| Inventing a web API in this package for the same bot | Stay long-polling unless product asks |

## Related Skills

- Canonical copies → `my-study-bot-meta/.agents/skills/bot/`
- Expiry cron / windows → `work-with-scheduler`
- Meta agent index → `my-study-bot-meta/.agents/AGENTS.md`
- Package always-on context → repo root `AGENTS.md` + `project-map.md`

## Verification

For structure-only placement tasks, confirm:

- [ ] Correct layer (`main` / `handlers` / `database` / `middlewares` / `keyboards` / `scheduler` / `states` / `tests` / deploy)
- [ ] Dependency direction respected (update → middleware → handler → session/DB)
- [ ] No parallel data path; handlers use injected `AsyncSession`
- [ ] Business logic not stuffed into `main.py`
- [ ] Keyboards/states stay free of DB access
- [ ] win32 SSL hacks not leaked to Docker/Linux
- [ ] Secrets and `data/` not committed
- [ ] New fields extend `User` (or related models) in `database.py`

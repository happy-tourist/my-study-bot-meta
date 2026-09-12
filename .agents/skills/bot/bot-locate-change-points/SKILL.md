---
name: bot-locate-change-points
description: Finds files and exact places that need to be changed or where new files should be added in the my-study-bot aiogram Telegram study bot based on a task description. Use when the user asks to analyze a task, locate implementation points, find affected files, or identify where changes should be made without editing code.
---

# Locate Change Points

Use this skill when the user gives a task statement and wants only analysis of where changes are needed in this bot package (`my-study-bot`).

Stack: aiogram 3.22 (Router, Dispatcher, long-polling), python-dotenv, SQLAlchemy 2.0 + aiosqlite (SQLite), Python 3.13 in Docker. Skills path for now: `.agents/skills/bot/` in this repo (canonical copy may later live under `my-study-bot-meta`). Runtime paths below are relative to this bot repo root; sibling meta/docs/OpenSpec is `../my-study-bot-meta`.

## Hard Rule

Do NOT edit, create, delete, format, or commit files.

Only inspect the codebase and report:

- existing files that likely need changes;
- exact handlers, FSM states, keyboards, User/DB fields, middleware, deploy/env involved;
- places where new files should be added, if the task requires new code;
- open questions when the implementation location cannot be determined confidently.

## Bot Layer Map

Search and assign ownership top-down along the Telegram update path.

| Layer | Path | Role |
|-------|------|------|
| Entry | `main.py` | Bot, Dispatcher, polling, platform session |
| Handlers | `app/handlers.py` | Router, commands, callbacks |
| DB | `app/database.py` | engine, User, init_db |
| Middleware | `app/middlewares.py` | DbSessionMiddleware |
| Keyboards | `app/keyboards.py` | reply/inline |
| FSM | `app/states.py` | FSM states |
| Deploy | `Dockerfile`, `docker-compose.yml`, `.github/workflows/deploy.yml` | GHCR + VPS |
| Env | `.env` (gitignored), `TG_TOKEN`, `DB_URL` | secrets / DB URL |

Allowed dependency direction:

Telegram update → Dispatcher middleware (`DbSessionMiddleware`) → handler (`session` injected) → SQLAlchemy `AsyncSession` → SQLite under `data/`. Shared UI in `keyboards.py`; multi-step dialogs in `states.py`. Prefer thin handlers; do not put business logic only inside `main.py`.

Prefer not editing `main.py` unless Bot/Dispatcher/polling/middleware registration or Windows-only session setup requires it — product flows live in `app/handlers.py` (+ states/keyboards/DB).

### Current vs intended product (scaffold)

Handlers are still thin; prefer extending existing layers rather than inventing a parallel data path.

| Expectation | Today |
|-------------|--------|
| Register user on `/start` | Implemented (`User` upsert + Russian greet + inline topic menu) |
| Subscription / study features | Schema has `subscription_end`, `is_active`; Subscription menu stub (no gate); no real study handlers yet |
| FSM forms / keyboards | `app/keyboards.py` has `main_menu_kb` / `back_to_menu_kb`; `app/states.py` still a stub |
| Modular routers | Single `router` in `app/handlers.py`, included from `main.py` |
| `.env.example` | Missing — vars documented in `AGENTS.md` |
| Tests | None yet — add pytest + aiogram helpers when critical flows appear |

### Handler / UX surface (today)

| Trigger | Behavior |
|---------|----------|
| `/start` (`CommandStart`) | Create `User` if missing; greet; attach inline topic menu |
| `menu:cars` / `menu:houses` / `menu:subscription` | Stub section + Back button |
| `menu:back` | Restore section-choice + main menu |

User-facing strings are Russian; keep them consistent unless copy is being redesigned.

### User model (today)

| Column | Notes |
|--------|-------|
| `id` | Telegram BigInteger PK |
| `username` | Optional string |
| `subscription_end` | Optional datetime — subscription gate (unused in handlers yet) |
| `is_active` | Boolean, default `True` |
| `created_at` | utcnow default |

Handlers that need DB must declare `session: AsyncSession` (injected by middleware).

## Decision Guidance

Use these rules to pick the layer before naming files.

### Command / callback vs FSM vs keyboard

| If the change is… | Prefer |
|-------------------|--------|
| New or changed Telegram command / message filter / callback | `app/handlers.py` on existing `router` (or new router module included from `main.py` if splitting grows) |
| Multi-step dialog / form (states, wait for next input) | `app/states.py` (StatesGroup) + handlers using `FSMContext` |
| Reply or inline keyboard markup | `app/keyboards.py` builders; wire in handlers |
| Greeting / Russian copy on `/start` | `app/handlers.py` `cmd_start` |
| Startup/shutdown hooks, middleware registration, polling | `main.py` |

**Do not** put study/subscription business logic only in `main.py`. Keep handlers thin; keyboards and FSM states in their modules.

### DB / middleware vs env

| If the change is… | Prefer |
|-------------------|--------|
| Persist user / subscription fields | Extend `User` in `app/database.py`; use injected `session` in handlers — do not invent a parallel store |
| Create tables on boot | `init_db()` already `create_all` — note schema changes may need migration strategy (none yet) |
| How handlers get a DB session | `app/middlewares.py` `DbSessionMiddleware`; registered on `dp.update` in `main.py` |
| Token / DB path | `.env` (gitignored): `TG_TOKEN` (required), `DB_URL` (default `sqlite+aiosqlite:///data/db.sqlite3`); document in `.env.example` when adding one |
| Windows local SSL/IPv4 VPN debug | `main.py` `sys.platform == "win32"` branch only — never copy into Docker/Linux production |

### Deploy / Docker

| If the change is… | Prefer |
|-------------------|--------|
| Image build, Python version, CMD | `Dockerfile` (`python:3.13-slim`, `CMD ["python", "main.py"]`) |
| Runtime env file / SQLite volume | `docker-compose.yml` (`env_file: .env`, `./data:/app/data`) |
| CI build/push GHCR + SSH compose pull/up | `.github/workflows/deploy.yml` |
| VPS secrets on server | Server `.env` only under `/home/deploy/my-study-bot` — do not commit |

### Tests

| If the change is… | Prefer |
|-------------------|--------|
| Registration, subscription gates, critical flows | New `tests/` (or similar) with pytest + aiogram testing helpers — none exist yet |

## Domain Hotspots

| Domain | Start here |
|--------|------------|
| Process entry / polling / Bot session | `main.py` |
| Commands, callbacks, message handlers | `app/handlers.py` |
| User registration `/start` | `app/handlers.py` `cmd_start` + `User` |
| User / subscription schema | `app/database.py` `User` |
| DB session injection | `app/middlewares.py` → `main.py` middleware register |
| Keyboards (reply/inline) | `app/keyboards.py` |
| FSM multi-step flows | `app/states.py` + handlers |
| Deps | `requirements.txt` |
| Docker image | `Dockerfile` |
| Compose / volume / env_file | `docker-compose.yml` |
| CI / GHCR / VPS deploy | `.github/workflows/deploy.yml` |
| Secrets / DB URL | `.env` (`TG_TOKEN`, `DB_URL`); optional future `.env.example` |
| Product specs / OpenSpec | Sibling `../my-study-bot-meta` when present — cite under Связанные места; do not invent meta files from this skill |

## Workflow

1. Read the task statement carefully and extract:
   - target feature or behavior;
   - entities (commands/callbacks, FSM, keyboards, User/subscription, middleware, deploy/env);
   - whether the task changes existing behavior or adds a new flow;
   - whether OpenSpec / meta docs in `../my-study-bot-meta` are involved.

2. Search the codebase by domain terms from the task:
   - entry (`Bot`, `Dispatcher`, `start_polling`, `include_router`, win32 session);
   - handlers (`Router`, `CommandStart`, `Command`, `CallbackQuery`, `F.`, `message.answer`);
   - DB (`User`, `subscription_end`, `is_active`, `init_db`, `async_session`, `DB_URL`);
   - middleware (`DbSessionMiddleware`, `session`);
   - keyboards (`InlineKeyboardMarkup` / `ReplyKeyboardMarkup`, builders, `menu:*` constants);
   - FSM (`FSMContext`, `StatesGroup`, `State`);
   - deploy (`Dockerfile`, `docker-compose`, `GHCR`, `appleboy/ssh-action`);
   - env (`TG_TOKEN`, `DB_URL`, `load_dotenv`).

3. Inspect nearby files to understand ownership along the layering path:
   - `main.py` middleware + router → `app/handlers.py` → `session` / `User`;
   - new UX → `keyboards.py` / `states.py` + handler wiring;
   - schema fields → `database.py` and any handlers that read/write them;
   - deploy/env when runtime or secrets change;
   - sibling `../my-study-bot-meta` when product contracts/specs must stay aligned — do not invent meta files here.

4. Apply decision guidance to classify each hit as edit vs add, and note related handler/DB/FSM/env/deploy touch points.

5. Stop after identifying the change points. Do not continue into implementation.

## Search Tips

| Look for… | Where / how |
|-----------|-------------|
| Bot / Dispatcher / polling | `main.py` |
| Router registration | `main.py` `include_router` + `app/handlers.py` `router` |
| `/start` registration | `app/handlers.py` `cmd_start` |
| User columns / engine | `app/database.py` |
| Session injection | `app/middlewares.py`; register in `main.py` |
| Keyboards | `app/keyboards.py` |
| FSM states | `app/states.py` |
| Env vars | `.env` / `os.getenv("TG_TOKEN")`, `DB_URL` in `database.py` |
| Docker / Compose | `Dockerfile`, `docker-compose.yml` |
| CI deploy | `.github/workflows/deploy.yml` |
| Deps | `requirements.txt` |

## Output Format

Respond in this structure (Russian headings):

```markdown
## Где править
- `path/to/file.py` — why this file is relevant and what kind of change is expected.

## Где добавить
- `path/to/new_file.py` — why a new file belongs here.

## Связанные места
- `path/to/related-file.py` — why it should be checked during implementation.

## Вопросы
- Clarifying question, only if needed.
```

Omit empty sections. Keep each bullet specific and actionable. Prefer paths under `app/`, `main.py`, plus deploy/env files when relevant. For OpenSpec / product-contract work, cite `../my-study-bot-meta` under Связанные места (do not edit it from this skill).

## Confidence

Prefer saying "likely" when the location is inferred from naming or nearby patterns. Say "confirmed" only when the inspected code directly proves the relationship.

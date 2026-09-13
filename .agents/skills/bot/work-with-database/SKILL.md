---
name: work-with-database
description: >-
  Use when adding, changing, reviewing, or debugging the SQLite user store,
  SQLAlchemy User model, DB_URL / data/db.sqlite3 paths, DbSessionMiddleware
  session injection, init_db, _SQLITE_USER_COLUMN_DDL / _ensure_sqlite_user_columns
  startup ALTER, or subscription / admin fields (subscription_end, is_active,
  trial_used, is_admin, is_banned) in my-study-bot so Telegram /start
  registration and handlers keep working.
---

# Work With Database

Use this skill for the **SQLite user store** in `my-study-bot`: SQLAlchemy 2.0 async + aiosqlite, the `User` model, and how handlers get an `AsyncSession` via middleware.

**Temp path:** this skill lives under `.agents/skills/bot/` in this repo for now; canonical copy may later move to `my-study-bot-meta/.agents/skills/bot/`. Runtime paths below are relative to this bot repo root.

Driver: **aiosqlite**. ORM: **SQLAlchemy 2.0** (`DeclarativeBase`, `mapped_column`, `async_sessionmaker`). No Colyseus, GameDatabase, or drizzle — do not invent those patterns here.

## Map Of Pieces

| Piece | Path | Role |
|-------|------|------|
| Connection + model | `app/database.py` | `DB_URL`, `engine`, `async_session`, `Base`, `User`, `init_db()`, `_SQLITE_USER_COLUMN_DDL`, `_ensure_sqlite_user_columns` |
| Session injection | `app/middlewares.py` | `DbSessionMiddleware` opens `async_session()`, sets `data["session"]` |
| Wire-up | `main.py` | `dp.update.middleware(DbSessionMiddleware())` then `await init_db()` |
| Handlers | `app/handlers.py`, `app/handlers_admin.py` | Declare `session: AsyncSession`; upsert `User` on `/start`; admin mutations |
| Env | `.env` (no `.env.example` yet) | `DB_URL` (default `sqlite+aiosqlite:///data/db.sqlite3`); `TG_TOKEN` unrelated to DB |
| Deploy volume | `docker-compose.yml` | `./data:/app/data` — SQLite file survives container restarts |
| Ignore | `.gitignore` | `data/` is gitignored — DB is not committed |

## Current Schema Facts

`app/database.py` — table `users` (`User`):

| Field | Column / type | Notes |
|-------|---------------|-------|
| `id` | `BigInteger` PK | Telegram user id (not autoincrement) |
| `username` | `String(64)`, nullable | Telegram username at register/update time |
| `subscription_end` | `DateTime`, nullable | Expiry instant; used by `app/scheduler.py` + `app/auth.py` gates |
| `is_active` | `Boolean`, `default=True` | Active flag |
| `trial_used` | `Boolean`, `default=False` | One-time free trial already consumed |
| `is_admin` | `Boolean`, `default=False` | DB admin role (OR with bootstrap `ADMIN_IDS`) |
| `is_banned` | `Boolean`, `default=False` | Full learner access block («доступ закрыт») |
| `created_at` | `DateTime`, `default=datetime.utcnow` | Row creation time |

These are **Telegram user** fields (identity + subscription + operator flags), not HTTP auth users. There is no separate admin HTTP API or JWT user store in this package — admin UX is Telegram-only (`handlers_admin`).

`init_db()` runs `Base.metadata.create_all`, then `_ensure_sqlite_user_columns` — idempotent `ALTER TABLE` for known new columns (e.g. `trial_used`, `is_admin`, `is_banned`) on an existing SQLite `users` table so VPS volume survives deploy without manual SSH wipe.

## Relation To Handlers / Identity

```text
main.py: DbSessionMiddleware + init_db()
        │
        ▼
  each Telegram update → async with async_session() as session
        │
        ▼
  handler(..., session: AsyncSession)
        │
        ▼
  select/add/commit User (PK = Telegram id)
```

- Identity is the Telegram user id (`message.from_user.id`), stored as `User.id`.
- `/start` creates the row if missing (`session.add` + `commit`); returning users are greeted without rewrite.
- Prefer the **injected** `session` in handlers — do not open a second engine/sessionmaker for the same request path.
- Subscription / study features should extend this `User` model and the same middleware injection, not a parallel DB stack.
- Auth for Telegram is Bot API token (`TG_TOKEN`); DB holds persisted profile/subscription data only.

## How To Add Columns Safely

1. Edit `User` in `app/database.py` only for new persisted user fields.
2. Prefer nullable columns or columns with a Python/`mapped_column` `default=...` so existing insert paths (e.g. `/start`) need not supply every field.
3. When adding a column: update the `User` model **and** append DDL to `_SQLITE_USER_COLUMN_DDL` in `app/database.py`. On next process/container start `init_db` runs create_all then idempotent ALTER for missing names. Deploy does **not** wipe `data/db.sqlite3`.
4. Keep using `async_session` from `app/database.py` and `DbSessionMiddleware`; do not add a second engine.
5. Document `DB_URL` if env/path behavior changes (add `.env.example` when convenient).
6. Compose volume `./data:/app/data` means the host `data/` directory is authoritative on the VPS — rows survive push; only missing columns from `_SQLITE_USER_COLUMN_DDL` are added.
7. Cover new DDL with a test like `tests/test_database_schema.py` (legacy table without the column → ensure → column present; second run idempotent).

Example pattern (same style as existing fields):

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)  # Telegram ID
    username: Mapped[str | None] = mapped_column(String(64), nullable=True)
    subscription_end: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    trial_used: Mapped[bool] = mapped_column(Boolean, default=False)
    is_admin: Mapped[bool] = mapped_column(Boolean, default=False)
    is_banned: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    # new_field: Mapped[str | None] = mapped_column(String(64), nullable=True)
```

## Do

- Extend `User` in `app/database.py` for new persisted learner/subscription fields.
- Give new columns a safe default or make them nullable so `/start` inserts keep working.
- Register every new `users` column in `_SQLITE_USER_COLUMN_DDL` so VPS volume gets ALTER on startup.
- Use injected `session: AsyncSession` in handlers that need DB.
- Keep SQLite under `data/` (default URL) and rely on Compose volume for persistence (no wipe on deploy).
- Treat subscription fields as server-side truth; do not trust client-only claims for access gates.

## Don't

- Invent Colyseus `GameDatabase`, drizzle schemas, or HTTP `/auth/register` — this is an aiogram bot + SQLAlchemy.
- Open a parallel engine/sessionmaker in handlers instead of middleware injection.
- Put FSM / ephemeral dialog state in the users table — that belongs in aiogram FSM (`app/states.py`).
- Commit `data/db.sqlite3`, secrets, or production `.env` with real credentials.
- Add a `User` mapped column without a matching `_SQLITE_USER_COLUMN_DDL` entry (prod volume will break on SELECT).
- Delete prod/local SQLite on every deploy — wipe only for intentional reset (e.g. re-test trial).
- Bypass the existing `User` model with a second user table unless there is a strong, explicit reason.

## Checklist

When changing DB-related code:

- [ ] New columns are nullable or have defaults compatible with `/start` insert
- [ ] New columns registered in `_SQLITE_USER_COLUMN_DDL` (+ model) and covered by schema ensure test
- [ ] Handlers that need DB still take `session: AsyncSession` from middleware
- [ ] No second engine / parallel ORM stack introduced
- [ ] Understood: create_all creates tables; column adds come from `_ensure_sqlite_user_columns` (not from wipe)
- [ ] Compose `./data:/app/data` and gitignore of `data/` still understood
- [ ] No Colyseus/drizzle patterns copied from other projects

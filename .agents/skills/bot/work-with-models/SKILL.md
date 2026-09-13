---
name: work-with-models
description: >-
  Use when creating, changing, reviewing, or debugging SQLAlchemy ORM models in
  the my-study-bot Telegram study bot — DeclarativeBase, Mapped / mapped_column,
  User table users, subscription_end / is_active / trial_used / is_admin / is_banned
  fields, init_db create_all + _SQLITE_USER_COLUMN_DDL ensure, or extending
  persistence for subscription / study / admin without parallel stores.
---

# Work With Models

Use this skill when editing **SQLAlchemy ORM models** (persistence layer) in
`my-study-bot`.

**Temp skills path:** this skill lives under `.agents/skills/bot/` in this
package for now. Canonical skills are intended to live in
`my-study-bot-meta/.agents/skills/bot/` once meta is available — prefer that
path when choosing skills if it exists.

Runtime paths below are relative to this bot repo root. Sibling meta/docs/OpenSpec:
`../my-study-bot-meta`.

Models are **persistence only**. Authoritative study/subscription rules, gating,
and side effects live in handlers / study helpers — not in model definitions.

## Overview

| Concern | Path / pattern |
| --- | --- |
| ORM models | `app/database.py` |
| API | SQLAlchemy 2.0: `DeclarativeBase`, `Mapped`, `mapped_column` |
| Session wiring | `DbSessionMiddleware` injects `session: AsyncSession` into handlers |
| Boot | `init_db()` → `create_all` + `_ensure_sqlite_user_columns` (DDL from `_SQLITE_USER_COLUMN_DDL`) |
| Engine / URL | `DB_URL` (default `sqlite+aiosqlite:///data/db.sqlite3`) |

Stack: SQLAlchemy 2.0 + `aiosqlite` (see `requirements.txt`).

## Current Vs Intended

**Today** (`app/database.py`) — `User` on table `users`:

```python
from sqlalchemy import BigInteger, Boolean, DateTime, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass


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
```

**Intended** (product growth; keep one store):

| Column | Role | Notes |
| --- | --- | --- |
| `id` | Telegram user id | `BigInteger` PK |
| `username` | Optional Telegram username | `String(64)`, nullable |
| `subscription_end` | Subscription expiry | `DateTime`, nullable — gate in handlers / `app/auth.py` |
| `is_active` | Active flag | `Boolean`, default `True` |
| `trial_used` | One-time free trial consumed | `Boolean`, default `False` |
| `is_admin` | DB admin role | `Boolean`, default `False` — OR with bootstrap `ADMIN_IDS` |
| `is_banned` | Full learner access block | `Boolean`, default `False` — independent of `is_active` |
| `created_at` | Row created | `DateTime`, `utcnow` default |

Prefer **extend `User`** for subscription/study/admin persistence rather than parallel
stores or a second user table.

### Column types (stable)

| Column | SQLAlchemy type | Meaning |
| --- | --- | --- |
| `id` | `BigInteger` | Telegram id (PK) |
| `subscription_end` | `DateTime` | When subscription ends (`None` = none / unset) |
| `is_active` | `Boolean` | Soft active flag |
| `trial_used` | `Boolean` | One-time free trial consumed |
| `is_admin` | `Boolean` | DB admin role (OR with `ADMIN_IDS`) |
| `is_banned` | `Boolean` | Learner access block («доступ закрыт») |

Keep these encodings stable; change only with handlers that read/write them.

## Python Pattern

Always use **SQLAlchemy 2.0 declarative** style (`Mapped` + `mapped_column`), not
classic `Column(...)` without annotations:

```python
from datetime import datetime

from sqlalchemy import BigInteger, Boolean, DateTime, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    username: Mapped[str | None] = mapped_column(String(64), nullable=True)
    subscription_end: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    trial_used: Mapped[bool] = mapped_column(Boolean, default=False)
    is_admin: Mapped[bool] = mapped_column(Boolean, default=False)
    is_banned: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

Rules:

- One `Base(DeclarativeBase)` shared by all models in `app/database.py`.
- Annotate every column: `field: Mapped[T] = mapped_column(...)`.
- Telegram ids stay `BigInteger` (not plain `Integer`) — ids exceed 32-bit range.
- Nullable business fields use `Mapped[T | None]` + `nullable=True`.
- Defaults for booleans/timestamps belong on `mapped_column`, not in handlers as
  the only source of truth for column defaults.

## Extending Persistence (Subscription / Study)

### Prefer extend `User`

Add study/subscription columns on `User` / `users` when the data is per-Telegram-user:

```python
class User(Base):
    __tablename__ = "users"
    # existing columns...
    # new: study_streak: Mapped[int] = mapped_column(Integer, default=0)
```

Handlers mutate via injected `session` (not raw engine in handlers):

```python
async def some_handler(..., session: AsyncSession):
    user = await session.get(User, message.from_user.id)
    # validate / apply rules here — then assign fields and commit
```

### New tables — only when needed

Add another `Base` subclass in `app/database.py` only when the entity is not a
user attribute (e.g. lessons, attempts). Keep FKs explicit; still no business
rules inside the model class body.

### Schema boot / migrations

```python
async def init_db():
    """Создаёт таблицы и догоняет недостающие колонки на существующем SQLite."""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
        await conn.run_sync(_ensure_sqlite_user_columns)
```

- **Today:** `create_all` creates missing tables; `_ensure_sqlite_user_columns` runs idempotent `ALTER TABLE` for names listed in `_SQLITE_USER_COLUMN_DDL` (e.g. `trial_used`, `is_admin`, `is_banned`) so the Compose volume on VPS picks up new columns on container start without SSH wipe.
- **When adding a column:** update the `User` model **and** append DDL to `_SQLITE_USER_COLUMN_DDL`. Cover with `tests/test_database_schema.py`-style ensure test. Do **not** wipe `data/db.sqlite3` on every deploy — wipe only for intentional reset.

## Do

- Keep models to **what must persist** (user id, subscription, study fields).
- Mutate rows from **handlers** (or study helpers) after validating intents.
- Prefer extending `User` over parallel stores for subscription/study data.
- Use `DeclarativeBase` + `Mapped` + `mapped_column` consistently.
- Use `BigInteger` for Telegram `id`, `DateTime` for `subscription_end`,
  `Boolean` for `is_active` / `trial_used` / `is_admin` / `is_banned`.
- Declare `session: AsyncSession` in handlers; rely on `DbSessionMiddleware`.
- Register new `users` columns in `_SQLITE_USER_COLUMN_DDL` when changing the model.

## Don't

- Put subscription gates, study rules, or Telegram UX copy inside model files.
- Invent a second user/profile store alongside `User` / `users`.
- Use classic undecorated `Column` style mixed with 2.0 `Mapped` in this package.
- Replace `BigInteger` Telegram PK with `Integer`.
- Assume `create_all` alone migrates existing tables — column adds need `_SQLITE_USER_COLUMN_DDL` + `_ensure_sqlite_user_columns`.
- Add a mapped column without DDL registration (prod volume will break).
- Commit secrets or the live `data/db.sqlite3` file.

## Alignment Checklist

When changing models:

1. Columns still match handler expectations (`id`, `subscription_end`, `is_active`, `trial_used`, `is_admin`, `is_banned`, …).
2. Types stay: Telegram id `BigInteger`, `subscription_end` `DateTime`, `is_active` / `trial_used` / `is_admin` / `is_banned` `Boolean`.
3. Business rules stay in handlers / study helpers — models remain persistence-only.
4. Prefer extend `User`; justify any new table.
5. If columns change on an existing DB: model + `_SQLITE_USER_COLUMN_DDL` (+ schema ensure test); no deploy wipe.
6. Middleware still injects `session`; handlers still use `AsyncSession`.

## Related

- Where to edit for a task: `bot-locate-change-points`
- Package overview: `AGENTS.md` (Database And Middleware, Current vs intended product)
- Sibling meta (when present): `../my-study-bot-meta` — OpenSpec / product contracts

---
name: bot-work-with-auth
description: >-
  Use when adding, changing, reviewing, or debugging Telegram bot access and
  subscription gates: User upsert on /start, message.from_user.id as identity,
  SQLAlchemy User.subscription_end / is_active checks, background expiry via
  app/scheduler.py (reminders + deactivation), DbSessionMiddleware session
  injection, or gated handlers in this aiogram my-study-bot package.
---

# Work With Auth

Use this skill for **access / subscription** in the study bot
(`my-study-bot`).

Auth is **Telegram user id** + **SQLAlchemy `User` row**. Identity comes from
`message.from_user.id` (or `callback_query.from_user.id`). Access to gated
features is decided by `User.is_active` and `User.subscription_end`.

There is **no** JWT, OAuth, cookies, Redis sessions, Bearer tokens, or
express-session layer.

**Temp skills path:** this skill lives under `.agents/skills/bot/` in this
package for now. Canonical skills are intended to live in
`my-study-bot-meta/.agents/skills/bot/` once meta is available — prefer
that path when choosing skills if it exists. Runtime `app/…` and `main.py`
paths are relative to this bot repo root.

Related skills (by name — load when that area is in scope):
`work-with-scheduler`, `bot-work-with-errors`, `bot-work-with-structure`,
`bot-locate-change-points`, `bot-verify-code` (when present in meta).

## Core Rule

Identity is **Telegram BigInteger id** stored as `User.id` (PK). Registration
happens on `/start` upsert via the injected `session`. Gated handlers must
load the `User` row and check `is_active` + `subscription_end` — not invent
tokens or trust client-supplied “premium” flags in message text.

Keep:

- User model + engine in `app/database.py`
- Session injection in `app/middlewares.py` (`DbSessionMiddleware`)
- Registration / greeting in `app/handlers.py` (`CommandStart`)
- Subscription / active checks in handlers (or a shared helper) before gated UX

Do not invent JWT middleware, OAuth providers, cookie sessions, or Redis
auth stores. Do not put study/product business rules only inside `main.py`.

## Map Of Pieces

| Layer | Path | Role |
|-------|------|------|
| Entry | `main.py` | `Bot` + `Dispatcher`; registers `DbSessionMiddleware` on updates; `init_db()`; includes router |
| Middleware | `app/middlewares.py` | Opens `async_session` per update; injects `session: AsyncSession` into handler `data` |
| User model | `app/database.py` | `User`: `id` (Telegram PK), `username`, `subscription_end`, `is_active`, `trial_used`, `created_at` |
| DB init | `app/database.py` `init_db()` | `create_all` + `_ensure_sqlite_user_columns` (DDL in `_SQLITE_USER_COLUMN_DDL`) |
| Register | `app/handlers.py` `/start` | Upsert by `message.from_user.id`; first visit grants one-time trial; greet + inline topic menu |
| Gate | `app/auth.py` + handlers | `has_active_subscription`; Cars/Houses gated; Subscription/tariffs always open |
| Expiry job | `app/scheduler.py` | Temporary minute windows + minutely cron (restore day + 10:00 MSK with ЮKassa) |
| Env | `.env` (local / VPS) | `TG_TOKEN` (required), `DB_URL` (default SQLite under `data/`) |
| Storage | `data/db.sqlite3` | Runtime SQLite (gitignored; Compose volume `./data:/app/data`) |

Prefer extending the existing `User` model and middleware session injection
rather than a parallel identity store.

## End-To-End Auth Flow

```text
Telegram update (message / callback)
        │
        ▼
  DbSessionMiddleware
        │
        └─ opens async_session → data["session"]
        │
        ▼
  Handler (e.g. /start or gated command)
        │
        ├─ telegram_id = event.from_user.id
        ├─ SELECT User WHERE id == telegram_id
        │
        ├─ /start + missing row
        │     → INSERT User(id, username=…)
        │     → commit → greet first visit + inline topic menu
        │
        ├─ /start + existing row
        │     → greet return + inline topic menu
        │
        └─ gated feature
              → require user row
              → require is_active is True
              → require subscription_end is not None
                 and subscription_end > now (UTC-aware policy as coded)
              → else refuse with Russian copy; do not run study flow
```

There is no separate “login” HTTP step: Telegram already authenticated the
user to the Bot API; the bot only **maps** that id to a local `User` and
**gates** by subscription fields.

### Step-by-step (happy path)

1. `main.py` registers `DbSessionMiddleware` so every handler can declare `session: AsyncSession`.
2. User sends `/start` → handler reads `message.from_user.id` (+ optional `username`).
3. If no `User` row: create with `trial_used=True`, `is_active=True`, `subscription_end=utcnow()+3 minutes`; commit; first-visit greeting (trial notice) + `kb.main_menu_kb()`.
4. If row exists: return greeting + same topic menu (do not recreate; do not re-grant trial).
5. For topic sections Cars/Houses: load `User`; `has_active_subscription` → allow stub or refuse with Russian copy + CTA to Subscription.
6. Subscription section / `tariff:*` grants: always reachable; write `subscription_end` / `is_active` on grant (no payment provider yet).
7. Background: `app/scheduler.py` (temporary minute windows, cron `minute="*"`) reminds before expiry and deactivates (`is_active=False`) when `subscription_end < now`.

## User Model (Access Fields)

File: `app/database.py`.

| Field | Type | Role |
|-------|------|------|
| `id` | `BigInteger` PK | Telegram user id — **the** identity key |
| `username` | `String(64)` \| None | Snapshot from Telegram; may change; not for auth |
| `subscription_end` | `DateTime` \| None | Access until this instant; `None` = no paid access |
| `is_active` | `bool` (default `True`) | Soft disable / ban without deleting the row |
| `trial_used` | `bool` (default `False`) | One-time free trial already consumed |
| `created_at` | `DateTime` | Registration timestamp |

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)  # Telegram ID
    username: Mapped[str | None] = mapped_column(String(64), nullable=True)
    subscription_end: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    trial_used: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

When adding new **required** columns, give safe defaults and register DDL in
`_SQLITE_USER_COLUMN_DDL` so `init_db` ALTERs existing SQLite on the next start.
so `/start` upsert and existing rows do not break.

## Registration (`/start`)

File: `app/handlers.py`.

```python
@router.message(CommandStart())
async def cmd_start(message: Message, session: AsyncSession):
    user_id = message.from_user.id
    user = await _load_user(session, user_id)

    if user is None:
        user = User(
            id=user_id,
            username=message.from_user.username,
            trial_used=True,
            is_active=True,
            subscription_end=datetime.utcnow() + timedelta(minutes=3),
        )
        session.add(user)
        await session.commit()
        await message.answer(
            "Привет! Я тебя запомнил 👋\n\n"
            "Тебе активирован пробный период на 3 дня (3 мин).\n\n"
            "Выбери раздел:",
            reply_markup=kb.main_menu_kb(),
        )
    else:
        await message.answer(
            f"С возвращением, {message.from_user.first_name}!\n\nВыбери раздел:",
            reply_markup=kb.main_menu_kb(),
        )
```

- Always key off `from_user.id`, never username alone (usernames change / may be absent).
- Keep Russian user-facing strings consistent unless product copy is being redesigned.
- After upsert/greet, attach the shared inline topic menu (`main_menu_kb`); Cars/Houses are gated via `has_active_subscription`; Subscription opens tariffs without gate.
- `/start` should remain the registration path; do not require a separate “sign up” command for MVP.
- Shared helper lives in `app/auth.py` — import it; do not duplicate gate math in each handler.

## How To Gate Handlers

Pattern for any subscription-protected command / callback:

1. Declare `session: AsyncSession` (middleware must stay registered).
2. Resolve `telegram_id` from `message.from_user.id` or `callback.from_user.id`.
3. `select(User).where(User.id == telegram_id)` → if missing, ask to `/start` (or upsert, per product).
4. If `not user.is_active` → refuse (banned / disabled).
5. If `user.subscription_end is None` or `user.subscription_end <= now` → refuse (no / expired subscription).
6. Only then run the study / paid flow.

Suggested shared helper (`app/auth.py`):

```python
from datetime import datetime

from app.database import User


def has_active_subscription(user: User | None, *, now: datetime | None = None) -> bool:
    now = now or datetime.utcnow()
    if user is None or not user.is_active:
        return False
    if user.subscription_end is None:
        return False
    return user.subscription_end > now
```

- Use the same clock basis as stored `subscription_end` (today: naive UTC via `datetime.utcnow` in the model).
- Prefer injecting `session` and loading `User` in the handler; do not trust FSM state or keyboard `callback_data` as proof of subscription.
- Admin / grant flows that set `subscription_end` should update the `User` row in SQLite — that **is** the source of truth.
- Background expiry (reminders + deactivation) belongs in `app/scheduler.py` — see `work-with-scheduler`. Do not duplicate window math in handlers.

## Middleware And Session

File: `app/middlewares.py`.

```python
class DbSessionMiddleware(BaseMiddleware):
    async def __call__(self, handler, event, data):
        async with async_session() as session:
            data["session"] = session
            return await handler(event, data)
```

- Handlers that touch auth/DB **must** declare `session: AsyncSession`.
- Do not open a second engine/session ad hoc inside handlers for the same update.
- Middleware is registered on the Dispatcher in `main.py` — removing it breaks registration and gates.

## Env And Storage

| Variable | Role |
|----------|------|
| `TG_TOKEN` | Telegram Bot API token (required) |
| `DB_URL` | SQLAlchemy async URL (default `sqlite+aiosqlite:///data/db.sqlite3`) |

No JWT / OAuth / session secrets. Do not commit `.env` or production tokens.
`data/` is gitignored and mounted in Compose so SQLite survives restarts.

## Changing Auth Safely

1. Keep `DbSessionMiddleware` on the Dispatcher or handlers lose `session`.
2. Keep `User.id` as Telegram BigInteger PK — do not switch PK to username.
3. Preserve `/start` upsert as the registration entry (or explicitly migrate the product).
4. Gate paid features via `is_active` + `subscription_end`, not message text or client claims.
5. New required `User` columns → defaults + `_SQLITE_USER_COLUMN_DDL` entry (startup ALTER; no deploy wipe).
6. Keep Russian refuse / greet copy consistent with existing handlers.
7. Prefer a shared `has_active_subscription` (or middleware gate) over duplicated checks.
8. Do not introduce JWT, OAuth, cookies, or Redis “for completeness”.

## Common Mistakes

| Mistake | Why it hurts |
|---------|----------------|
| Keying users by `username` | Unstable / nullable; wrong identity |
| Skipping `session` / middleware | No upsert, no subscription read |
| Treating `subscription_end is None` as unlimited | Today means **no** paid access |
| Ignoring `is_active` | Soft-disable never takes effect |
| Trusting FSM / callback payload for “premium” | Spoofable; not the DB source of truth |
| Inventing JWT / OAuth / Redis sessions | Wrong stack for this Telegram bot |
| Creating a second users table | Diverges from `User` + middleware path |
| Putting gate logic only in `main.py` | Breaks layering; handlers become untestable |
| Comparing timezone-aware vs naive datetimes carelessly | False allow / deny on expiry |
| Committing `.env` / `TG_TOKEN` | Secret leak |

## Change Checklist

When touching auth / access:

1. Which piece? `database.User` / middleware / `/start` / gate helper / handler / scheduler expiry / env / deploy volume.
2. `DbSessionMiddleware` still registered; handlers still receive `session`.
3. Identity still `from_user.id` → `User.id`.
4. `/start` still upserts safely (no duplicate PK errors).
5. Gated handlers still check `is_active` + `subscription_end`.
6. `None` subscription still means denied (unless product explicitly changes that).
7. New `User` columns have safe defaults and `_SQLITE_USER_COLUMN_DDL` registration.
8. No JWT / OAuth / cookie / Redis auth introduced.
9. Russian user-facing refuse/greet strings still consistent.
10. SQLite path / `DB_URL` / `data/` volume still valid for local and VPS.

## Do / Don't

| Do | Don't |
|----|--------|
| Use Telegram `from_user.id` as identity | Use username or display name as PK |
| Upsert `User` on `/start` via injected `session` | Open ad-hoc sessions or skip middleware |
| Gate with `is_active` + `subscription_end` | Invent JWT / OAuth / cookie / Redis auth |
| Treat `subscription_end is None` as no access | Assume `None` means lifetime free |
| Extend existing `User` + middleware path | Parallel identity tables or stores |
| Keep refuse/greet copy in Russian (current product) | Port Colyseus / HTTP BFF auth patterns |
| Share one subscription-check helper | Scatter inconsistent expiry logic |
| Keep secrets in `.env` only (`TG_TOKEN`, `DB_URL`) | Commit tokens or invent unused auth secrets |

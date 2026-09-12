---
name: bot-work-with-auth
description: >-
  Use when adding, changing, reviewing, or debugging Telegram bot access and
  subscription gates: User upsert on /start, message.from_user.id as identity,
  SQLAlchemy User.subscription_end / is_active checks, DbSessionMiddleware
  session injection, or gated handlers in this aiogram my-study-bot package.
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
`bot-work-with-errors`, `bot-work-with-structure`, `bot-locate-change-points`,
`bot-verify-code` (when present in meta).

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
| User model | `app/database.py` | `User`: `id` (Telegram PK), `username`, `subscription_end`, `is_active`, `created_at` |
| DB init | `app/database.py` `init_db()` | `Base.metadata.create_all` if tables missing |
| Register | `app/handlers.py` `/start` | Upsert by `message.from_user.id`; greet first visit vs return; attach inline topic menu (`kb.main_menu_kb()`) |
| Gate | Handlers (or helper) | Load `User` by Telegram id; require `is_active` and valid `subscription_end` |
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
3. If no `User` row: create with defaults (`is_active=True`, `subscription_end=None`); commit; first-visit greeting + `kb.main_menu_kb()`.
4. If row exists: return greeting + same topic menu (do not recreate).
5. For paid / study features: reload or reuse `User`; check `is_active` and `subscription_end`; allow or refuse. Menu stubs today are **not** gated.

## User Model (Access Fields)

File: `app/database.py`.

| Field | Type | Role |
|-------|------|------|
| `id` | `BigInteger` PK | Telegram user id — **the** identity key |
| `username` | `String(64)` \| None | Snapshot from Telegram; may change; not for auth |
| `subscription_end` | `DateTime` \| None | Access until this instant; `None` = no paid access |
| `is_active` | `bool` (default `True`) | Soft disable / ban without deleting the row |
| `created_at` | `DateTime` | Registration timestamp |

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)  # Telegram ID
    username: Mapped[str | None] = mapped_column(String(64), nullable=True)
    subscription_end: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
```

When adding new **required** columns, give safe defaults (or migrate carefully)
so `/start` upsert and existing rows do not break.

## Registration (`/start`)

File: `app/handlers.py`.

```python
@router.message(CommandStart())
async def cmd_start(message: Message, session: AsyncSession):
    user_id = message.from_user.id

    result = await session.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()

    if user is None:
        user = User(id=user_id, username=message.from_user.username)
        session.add(user)
        await session.commit()
        await message.answer(
            "Привет! Я тебя запомнил 👋\n\nВыбери раздел:",
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
- After upsert/greet, attach the shared inline topic menu (`main_menu_kb`); section stubs are not gated yet — do not treat menu callbacks as proof of subscription.
- `/start` should remain the registration path; do not require a separate “sign up” command for MVP.

## How To Gate Handlers

Pattern for any subscription-protected command / callback:

1. Declare `session: AsyncSession` (middleware must stay registered).
2. Resolve `telegram_id` from `message.from_user.id` or `callback.from_user.id`.
3. `select(User).where(User.id == telegram_id)` → if missing, ask to `/start` (or upsert, per product).
4. If `not user.is_active` → refuse (banned / disabled).
5. If `user.subscription_end is None` or `user.subscription_end <= now` → refuse (no / expired subscription).
6. Only then run the study / paid flow.

Suggested shared helper (prefer one place over copy-paste):

```python
from datetime import datetime

def has_active_subscription(user: User, *, now: datetime | None = None) -> bool:
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
5. New required `User` columns → defaults or migration before deploy.
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

1. Which piece? `database.User` / middleware / `/start` / gate helper / handler / env / deploy volume.
2. `DbSessionMiddleware` still registered; handlers still receive `session`.
3. Identity still `from_user.id` → `User.id`.
4. `/start` still upserts safely (no duplicate PK errors).
5. Gated handlers still check `is_active` + `subscription_end`.
6. `None` subscription still means denied (unless product explicitly changes that).
7. New `User` columns have safe defaults or a migration plan.
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

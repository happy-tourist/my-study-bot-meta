---
name: bot-work-with-auth
description: >-
  Use when adding, changing, reviewing, or debugging Telegram bot access and
  subscription gates in my-study-bot: /start upsert + trial + Menu sync,
  topic gate/ban/slash unregistered, ADMIN_IDS / is_admin / is_banned,
  promote/demote Menu sync. Core in SKILL.md; topic details in start-register.md
  and gate.md.
---

# Work With Auth

Use this skill for **access / subscription** in the study bot
(`my-study-bot`).

Auth is **Telegram user id** + **SQLAlchemy `User` row**. Identity comes from
`message.from_user.id` (or `callback_query.from_user.id`). Access to gated
features is decided by `User.is_active` and `User.subscription_end`.

There is **no** JWT, OAuth, cookies, Redis sessions, Bearer tokens, or
express-session layer.

Runtime `app/…` and `main.py` paths are relative to the bot repo root
(`../my-study-bot` via projects-map).

Related skills: `work-with-scheduler`, `bot-work-with-errors`,
`bot-work-with-structure`, `bot-locate-change-points`, `bot-verify-code`.

## Specialized Topics

Read the matching file in this folder when the change involves that area
(do not load every file at once):

| Topic | File |
|-------|------|
| `/start` upsert, trial, greet, Menu Button sync on register/return; promote/demote sync | [start-register.md](start-register.md) |
| Topic gate (Cars/Houses/Computers/Chinese), ban, slash unregistered, `has_active_subscription` | [gate.md](gate.md) |

## Core Rule

Identity is **Telegram BigInteger id** stored as `User.id` (PK). Registration
happens on `/start` upsert via the injected `session`. Gated handlers must
load the `User` row and check `is_active` + `subscription_end` — not invent
tokens or trust client-supplied “premium” flags in message text.

Keep:

- User model + engine in `app/database.py`
- Session injection in `app/middlewares.py` (`DbSessionMiddleware`)
- Registration / greeting in `app/handlers.py` (`CommandStart`) — details in [start-register.md](start-register.md)
- Subscription / active checks via `app/auth.py` before gated UX — details in [gate.md](gate.md)

Do not invent JWT middleware, OAuth providers, cookie sessions, or Redis
auth stores. Do not put study/product business rules only inside `main.py`.

## Map Of Pieces

| Layer | Path | Role |
|-------|------|------|
| Entry | `main.py` | `Bot` + `Dispatcher`; registers `DbSessionMiddleware`; `init_db()`; includes routers |
| Middleware | `app/middlewares.py` | Opens `async_session` per update; injects `session` |
| User model | `app/database.py` | `User` identity + subscription / admin / ban columns |
| Register | `app/handlers.py` `/start` | Upsert + trial + Menu sync + topic menu — [start-register.md](start-register.md) |
| Gate | `app/auth.py` + handlers | `has_active_subscription` / ban; Cars…Chinese stubs — [gate.md](gate.md) |
| Menu commands | `app/bot_commands.py` | Learner/admin `BotCommand` lists; chat-scoped sync |
| Admin | `app/handlers_admin.py` | Panel; `ADMIN_IDS` + `is_admin`; promote/demote → Menu sync |
| Expiry job | `app/scheduler.py` | Temporary minute windows + minutely cron |
| Env / storage | `.env` / `data/db.sqlite3` | `TG_TOKEN`, `DB_URL`, `ADMIN_IDS`; gitignored SQLite volume |

## End-To-End Auth Flow

```text
Telegram update → DbSessionMiddleware → handler
  /start missing row → INSERT trial User → sync Menu → greet + topic menu
  /start existing    → sync Menu → return greet (+ ban → «Доступ закрыт.»)
  slash topic/sub without row → «Сначала нажми /start»
  gated topic (inline/slash) → ban → has_active_subscription → stub or refuse
```

No separate HTTP login: Telegram authenticated the user; the bot maps id → `User`
and gates by subscription fields. Step detail: [start-register.md](start-register.md),
[gate.md](gate.md).

## User Model (Access Fields)

File: `app/database.py`.

| Field | Role |
|-------|------|
| `id` | Telegram BigInteger PK — **the** identity key |
| `username` | Snapshot only; not for auth |
| `subscription_end` | Access until; `None` = no paid access |
| `is_active` | Soft disable on expiry/revoke (scheduler); not the ban flag |
| `is_admin` / `is_banned` | DB admin role (OR `ADMIN_IDS`); full learner block |
| `trial_used` | One-time free trial consumed |
| `created_at` | Registration timestamp |

New **required** columns → safe defaults + `_SQLITE_USER_COLUMN_DDL` so `init_db`
ALTERs existing SQLite.

## Middleware And Session

Handlers that touch auth/DB **must** declare `session: AsyncSession`. Do not
open a second engine/session for the same update. Middleware stays registered
on the Dispatcher in `main.py`.

## Env And Storage

| Variable | Role |
|----------|------|
| `TG_TOKEN` | Bot API token (required) |
| `DB_URL` | Async SQLAlchemy URL (default SQLite under `data/`) |
| `ADMIN_IDS` | CSV bootstrap admin Telegram ids |

No JWT / OAuth secrets. Do not commit `.env` or tokens.

## Changing Auth Safely

1. Keep `DbSessionMiddleware` registered; handlers still receive `session`.
2. Keep `User.id` as Telegram BigInteger PK.
3. Preserve `/start` upsert (or explicitly migrate product) — [start-register.md](start-register.md).
4. Gate paid topics via `is_active` + `subscription_end` — [gate.md](gate.md).
5. New `User` columns → defaults + `_SQLITE_USER_COLUMN_DDL`.
6. Prefer shared `has_active_subscription` over duplicated checks.
7. Do not introduce JWT, OAuth, cookies, or Redis auth.

## Common Mistakes

| Mistake | Why it hurts |
|---------|----------------|
| Keying users by `username` | Unstable / nullable |
| Skipping `session` / middleware | No upsert, no subscription read |
| `subscription_end is None` as unlimited | Means **no** paid access |
| Ignoring `is_active` / ban order | Soft-disable or ban never takes effect |
| Trusting FSM / callback for “premium” | Spoofable; not DB truth |
| Parallel users table / JWT stack | Wrong for this Telegram bot |
| Gate logic only in `main.py` | Untestable handlers |

## Change Checklist

1. Piece: `User` / middleware / `/start` / gate / handler / scheduler / env?
2. `session` still injected; identity still `from_user.id` → `User.id`.
3. `/start` upsert + Menu sync still safe ([start-register.md](start-register.md)).
4. Topic gates still ban → `has_active_subscription` ([gate.md](gate.md)).
5. `None` subscription still denied; Russian copy consistent.
6. No JWT / OAuth / Redis auth; secrets stay in `.env`.

## Do / Don't

| Do | Don't |
|----|--------|
| Telegram `from_user.id` as identity | Username as PK |
| Upsert on `/start` via injected `session` | Ad-hoc sessions / skip middleware |
| Gate with `is_active` + `subscription_end` | Invent JWT / OAuth / cookie auth |
| Extend `User` + middleware path | Parallel identity stores |
| Share one subscription-check helper | Scatter inconsistent expiry logic |

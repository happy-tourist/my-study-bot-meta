---
name: work-with-handlers
description: >-
  Use when adding, changing, or reviewing aiogram Router handlers in
  my-study-bot: message/callback handlers in app/handlers.py, filters
  (CommandStart, Command, F, CallbackQuery), trial/tariffs/gate callbacks,
  DB session injection, or wiring via dp.include_router in main.py. Keep
  handlers thin — UI in keyboards, gate helper in app/auth.py, multi-step
  dialogs in states.
---

# Work With Handlers

Use this skill when adding or changing **Telegram update handlers** in
`my-study-bot`.

This is a **Telegram long-polling bot** (aiogram v3), not an HTTP API. Prefer
Router message/callback handlers for user flows. There is no REST surface —
updates arrive via polling only.

Stack: **aiogram** 3.22 (`Router`, `Dispatcher`, filters), SQLAlchemy 2.0 async
+ aiosqlite, Python 3.13 (Docker).

**Temp skills path:** this skill lives under `.agents/skills/bot/` in this
package for now. Canonical skills are intended to live in
`my-study-bot-meta/.agents/skills/bot/` once meta is available — prefer that
path when choosing skills if it exists.

Configure handlers in `app/handlers.py`. Prefer not editing `main.py` beyond
`dp.include_router(...)` / middleware / polling wiring unless startup itself
must change.

## Quick Reference

| Topic | Pattern |
|-------|---------|
| Where | `app/handlers.py` — single `router = Router()` |
| Wire-up | `main.py` → `dp.include_router(router)` |
| Commands | `@router.message(CommandStart())` / `Command("…")` |
| Text / media | `@router.message(F.text)` / other `F` filters |
| Inline callbacks | `@router.callback_query(F.data == "…")` |
| DB access | declare `session: AsyncSession` — injected by `DbSessionMiddleware` |
| Shared UI | builders in `app/keyboards.py` |
| Multi-step | FSM states in `app/states.py` + `FSMContext` |
| User copy | Russian strings, match existing replies |

## Rules

| Do | Don't |
|----|--------|
| Keep handlers thin (reply / DB upsert / call keyboard) | Put keyboard markup trees or FSM state graphs only inside handlers |
| Add handlers on the existing `router` in `app/handlers.py` | Invent a parallel HTTP API or second Dispatcher |
| Declare `session: AsyncSession` when the handler needs DB | Open a new engine/session inside the handler |
| Put reply/inline keyboards in `app/keyboards.py` | Inline huge `InlineKeyboardMarkup([...])` blobs in every handler |
| Put multi-step dialogs in `app/states.py` + FSM handlers | Encode long wizards as nested if/else in one command handler |
| Keep user-facing strings in Russian | Switch to English copy unless product redesign asks |
| Extend `User` + middleware session for study/subscription | Invent a second persistence path beside SQLite/`User` |

## Handler surface in `app/handlers.py`

### Single router

```python
from aiogram import Router, F
from aiogram.types import Message, CallbackQuery
from aiogram.filters import CommandStart, Command
from sqlalchemy.ext.asyncio import AsyncSession

router = Router()


@router.message(CommandStart())
async def cmd_start(message: Message, session: AsyncSession):
    ...
```

- One module-level `router`; include it once from `main.py`.
- Prefer extending this router until product clearly needs split routers
  (then still `include_router` from `main.py` — do not start a second bot
  process).

### Filters

| Filter | Use |
|--------|-----|
| `CommandStart()` | `/start` |
| `Command("name")` | other bot commands |
| `F.…` | text, content type, callback data, etc. |
| `CallbackQuery` handlers | `@router.callback_query(...)` |

### DB session

`DbSessionMiddleware` injects `session` into handler `data`. Handlers that
touch SQLite must accept `session: AsyncSession` and use the injected session
(open/close is middleware-owned).

### Keyboards and FSM

- Shared UI → `app/keyboards.py` (`ReplyKeyboardMarkup` / `InlineKeyboardMarkup`
  builders).
- Multi-step flows → `app/states.py` (FSM group) + `FSMContext` in handlers.

## Handler catalog (current)

| Trigger | Handler | Notes |
|---------|---------|-------|
| `/start` (`CommandStart`) | `cmd_start` | Upsert by Telegram id; first visit grants one-time trial (`trial_used`, 3 min); greet + `kb.main_menu_kb()` |
| `menu:cars` / `menu:houses` | `menu_cars` / `menu_houses` | Load `User`; `has_active_subscription` → stub content + `back_to_menu_kb`, else refuse + `subscription_required_kb` |
| `menu:subscription` | `menu_subscription` | Always open; tariffs list via `tariffs_kb()` (no gate) |
| `tariff:*` | `tariff_grant` | Grant minutes from `kb.TARIFFS` onto `subscription_end`; set `is_active=True` |
| `menu:back` | `menu_back` | Restore section-choice prompt + `main_menu_kb()` |

Keyboards live in `app/keyboards.py` (`menu:*`, `TARIFF_PREFIX` / `TARIFFS`). Gate helper: `app/auth.py` `has_active_subscription`. `app/states.py` is still a stub (no FSM for this menu). Study topic content remains stubs; access gate is live.

## Layering

```
Telegram update
  Dispatcher + DbSessionMiddleware
    → Router handler (thin)
      → optional AsyncSession / User
      → keyboards.py for UI
      → states.py for multi-step
    → NEVER an HTTP route or REST BFF
```

Canonical product path: user sends command/callback → handler → reply (and
optional DB write). Not `POST /api/...`.

## Reply shapes

| Kind | Pattern | Examples |
|------|---------|----------|
| Text reply | `await message.answer("…")` | `/start` greetings |
| Markup | `reply_markup=` from `keyboards.py` | topic menu on `/start` |
| Callback answer | `await callback.answer(...)` + edit/send | section stubs / back |
| Errors | short user-facing RU text; keep simple | no HTTP status envelopes |

## Adding a new handler

1. Ask: is this a Telegram UX step (command / message / callback / FSM)? If
   someone asks for REST — stop; this package has no HTTP API.
2. Add `@router.message(...)` or `@router.callback_query(...)` on the existing
   `router` in `app/handlers.py`.
3. If DB is needed, declare `session: AsyncSession` and use middleware injection.
4. Keep the handler short; move keyboards to `keyboards.py`, FSM to `states.py`.
5. Match existing Russian copy style.
6. Prefer extending `User` for subscription/study fields rather than a new table
   unless the product clearly needs one.
7. Do not put business logic only in `main.py`. Leave the Windows SSL/IPv4
   branch untouched unless the task is about local VPN/debug connectivity.
8. Smoke with `python main.py` from the bot package root when useful (`.venv`,
   `.env` with `TG_TOKEN`); fix failures before claiming done.

Do **not** create Flask/FastAPI route trees, webhook-only dual stacks, or a
second SQLite access layer that bypasses middleware.

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Adding Express/FastAPI routes “for the bot” | Use Router handlers only |
| Opening `async_session()` inside every handler | Declare `session: AsyncSession` |
| Fat handlers with inline keyboard + FSM + SQL | Split to `keyboards.py` / `states.py` |
| English user strings next to Russian ones | Keep RU consistent with `/start` |
| Parallel user store besides `User` | Extend `User` + injected session |
| Forgetting `dp.include_router` for a new Router | Wire from `main.py` if you split routers |
| Copying Windows SSL bypass into Docker/Linux | Keep it in `sys.platform == "win32"` only |

## Checklist for a new or changed handler

1. Lives in `app/handlers.py` on the shared `router` (or a router included from `main.py`).
2. Handler is thin; UI/FSM extracted when non-trivial.
3. Correct filter (`CommandStart` / `Command` / `F` / callback).
4. `session: AsyncSession` only when DB is used; no ad-hoc engines.
5. User-facing text is Russian and matches nearby tone.
6. No HTTP/API surface introduced.
7. `main.py` only touched for include_router / middleware / startup if required.

## Related skills

- `work-with-middleware` — `DbSessionMiddleware`, update middleware order (when present)
- `work-with-database` — `User` model, `init_db`, `DB_URL` (when present)
- `work-with-keyboards` — reply / inline builders in `keyboards.py` (when present)
- `work-with-states` — FSM groups and multi-step dialogs (when present)
- `work-with-structure` — where handlers vs DB vs main belong (when present)

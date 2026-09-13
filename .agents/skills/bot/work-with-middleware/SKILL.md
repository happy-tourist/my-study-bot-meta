---
name: work-with-middleware
description: >-
  Use when adding, changing, reviewing, or debugging aiogram update middleware
  in my-study-bot: DbSessionMiddleware, dp.update.middleware registration,
  session injection into handlers, middleware order (DB / auth-subscription /
  logging), or AsyncSession lifecycle. Not Express CORS and not Colyseus
  Room.onAuth.
---

# Work With Middleware

Use this skill when working with **aiogram update middleware** for this bot
package (`my-study-bot`).

Stack: aiogram 3.22 (Router, Dispatcher, long-polling), SQLAlchemy 2.0 +
aiosqlite, python-dotenv. Skills path for now: `.agents/skills/bot/` in this
repo (canonical copy may later live under `my-study-bot-meta`). Runtime paths
below are relative to this bot repo root.

Related skills (by name only — load when that area is in scope):
`bot-locate-change-points` (where to edit), handlers / DB / FSM skills when
present.

## Out Of Scope

This is **not** an HTTP / Express middleware layer. Do **not** invent or port:

- Express `app.use` / CORS / `OPTIONS` 204 / `Access-Control-*`
- Colyseus `Room.onAuth` / JWT room guards
- BFF `checkSession` / `checkCaptcha` / Redis session middleware
- HTTP route guards that `res.status(401|403).send(...)`

Auth and subscription checks for this bot belong as **aiogram middleware** (or
handler-level checks using the injected DB session), not as web-server middleware.

## Core Model

Custom update middleware lives in `app/middlewares.py` and is registered on the
Dispatcher in `main.py` via `dp.update.middleware(...)`. Order matters.

```
Telegram update
  → Dispatcher update middleware chain
      → (preferred) logging — outermost, sees all updates
      → (preferred) auth / subscription gate — before handlers that need access
      → DbSessionMiddleware — opens session, injects data["session"]
  → handler (declares session: AsyncSession when it needs DB)
```

| Piece | Where | Role |
|-------|--------|------|
| `DbSessionMiddleware` | `app/middlewares.py` | `async with async_session()` → `data["session"]` → await handler → session closes |
| Registration | `main.py` | `dp.update.middleware(DbSessionMiddleware())` before polling |
| Handler consumption | `app/handlers.py`, `app/handlers_admin.py` | declare `session: AsyncSession`; use injected session |

Today only `DbSessionMiddleware` exists. Future middlewares (logging, auth /
subscription gate) should follow the preferred order above.

## Patterns

### DbSessionMiddleware (session lifecycle)

```python
from typing import Any, Awaitable, Callable, Dict

from aiogram import BaseMiddleware
from aiogram.types import TelegramObject

from app.database import async_session


class DbSessionMiddleware(BaseMiddleware):
    """Передаёт сессию БД в аргументы хендлеров."""

    async def __call__(
        self,
        handler: Callable[[TelegramObject, Dict[str, Any]], Awaitable[Any]],
        event: TelegramObject,
        data: Dict[str, Any],
    ) -> Any:
        async with async_session() as session:
            data["session"] = session
            return await handler(event, data)
```

Rules:

- Open a session with `async with async_session() as session`.
- Inject via `data["session"] = session` before calling the handler.
- Session closes when the `async with` exits (after the handler returns or
  raises) — do not leave sessions open across updates.
- Keep the class in `app/middlewares.py`; register it in `main.py`, do not
  open `async_session()` ad hoc inside handlers.

### Register on Dispatcher

```python
dp = Dispatcher()
dp.update.middleware(DbSessionMiddleware())
await init_db()

dp.include_router(router)
```

Rules:

- Register update middleware on `dp` **before** `start_polling`.
- Prefer `dp.update.middleware(...)` so every update type gets the same chain.
- Product middleware classes stay in `app/middlewares.py`; `main.py` only
  wires them (same as today’s `DbSessionMiddleware` import).

### Handlers declare `session: AsyncSession`

```python
from sqlalchemy.ext.asyncio import AsyncSession

@router.message(CommandStart())
async def cmd_start(message: Message, session: AsyncSession):
    result = await session.execute(select(User).where(User.id == user_id))
    # ...
```

Rules:

- Handlers that need DB **must** declare `session: AsyncSession` — aiogram
  injects from `data["session"]`.
- Do not create a second engine/session factory in the handler.
- Commit/rollback intentionally in the handler; middleware only owns open/close.

### Preferred middleware order (future)

When adding more update middlewares, register in this order (outer → inner):

1. **Logging** — outermost; observe every update (and failures) without depending
   on DB or auth outcome.
2. **Auth / subscription gate** — reject or short-circuit unauthorized /
   expired-subscription users before business handlers run. May need DB: either
   run **after** `DbSessionMiddleware` or open a narrow session itself — prefer
   after DB middleware so it reuses `data["session"]`.
3. **DbSessionMiddleware** — innermost of the shared infrastructure; always
   available to handlers that declare `session`.

Practical registration (aiogram: last registered ≈ outer, first registered ≈
closer to handler — verify against current aiogram 3.x docs when adding a
second middleware; keep one clear order comment in `main.py`).

If a subscription gate needs `session`, register it so it runs **with** an
already-injected session (inner relative to logging, outer or same chain as DB
depending on whether the gate reads `data["session"]`). Document the chosen
order next to the `dp.update.middleware(...)` calls.

## Wiring Changes

1. Implement middleware classes in `app/middlewares.py` (subclass
   `BaseMiddleware`, implement `async def __call__(self, handler, event, data)`).
2. Register on `dp.update` in `main.py` — prefer not scattering registration
   across routers unless a middleware is truly router-scoped.
3. Handlers that need DB declare `session: AsyncSession`; do not bypass
   middleware with manual `async_session()` in handlers.
4. Extend `User` / subscription fields in `app/database.py` for gate logic; do
   not invent a parallel auth store.
5. Do **not** port Express CORS or Colyseus `onAuth` patterns.
6. Run the bot / relevant checks from this package root when verifying; fix
   failures before claiming done.

## Checklist

- [ ] Middleware class lives in `app/middlewares.py` and subclasses `BaseMiddleware`.
- [ ] `DbSessionMiddleware` uses `async with async_session()`, sets
      `data["session"]`, awaits `handler`, then closes.
- [ ] Registered with `dp.update.middleware(...)` in `main.py` before polling.
- [ ] Handlers that touch DB declare `session: AsyncSession`.
- [ ] New middlewares follow preferred order: logging → auth/subscription → DB
      (or documented equivalent that still injects session for handlers).
- [ ] No Express CORS / Colyseus `onAuth` / HTTP session-guard patterns added.
- [ ] No ad-hoc session factories inside handlers.

## Common Mistakes

- Forgetting to register middleware on `dp.update` — handlers get no `session`.
- Opening `async_session()` inside every handler instead of using middleware.
- Omitting `session: AsyncSession` on the handler signature — injection fails.
- Registering subscription/auth middleware without access to DB when the gate
  needs `User.subscription_end` / `is_active`.
- Putting middleware-only wiring or business gates solely in `main.py` beyond
  registration calls.
- Porting Express `app.use` CORS or Colyseus room `onAuth` into this Telegram bot.
- Leaving sessions open (not using `async with`) or sharing one session across
  concurrent updates.

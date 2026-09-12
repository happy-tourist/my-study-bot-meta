---
name: bot-work-with-errors
description: >-
  Use when adding, changing, reviewing, or debugging error handling in the
  my-study-bot aiogram v3 Telegram bot — handler exceptions, DB session
  failures, invalid FSM input, subscription denials, Telegram API errors, or
  aligning expected failures with clear Russian message.answer /
  callback.answer UX.
---

# Work With Errors

Use this skill when working with handler, DB, FSM, subscription, and Telegram
API failures in this package (`my-study-bot`).

Stack: aiogram 3.22 (`Router` / `Dispatcher` / long polling), SQLAlchemy 2.0 +
`aiosqlite`, Python 3.13 (Docker). This is a **Telegram long-polling bot**, not
an HTTP API / BFF and not a Colyseus game server.

**Paths:** this skill currently lives in **this bot repo** at
`.agents/skills/bot/` (temporary). Canonical skills/OpenSpec will move to
**my-study-bot-meta** when present (`project-map.md` key `my-study-bot-meta`).
Runtime `app/…` / `main.py` paths are relative to **this repository root**.

Related skills (by name only — load when that area is in scope):
`bot-work-with-handlers`, `bot-work-with-middleware`, `bot-work-with-database`,
`bot-work-with-fsm`, `bot-work-with-keyboards`, `bot-work-with-structure`,
`bot-work-with-test`.

## Core Model

There is **no** `ServiceError`, **no** `{ errorCode, errorMessage }` envelope,
**no** Express/BFF error middleware catalog, and **no** Colyseus JWT / room /
`client.send("error")` patterns. Do not invent those here.

Failures split by channel:

| Channel | What fails | How the user sees it |
|---------|------------|----------------------|
| Expected UX | Bad FSM input, no subscription, wrong step | Clear Russian `message.answer` / `callback.answer` |
| Handler / DB | Unexpected exception, commit failure | Log with context; optional short generic Russian reply |
| Telegram API | Network, flood, bad request from Bot API | Log; retry only when safe; do not treat Windows SSL bypass as a fix |
| Middleware session | Session open/close / broken transaction | Middleware owns lifecycle; do not leave broken sessions open |

Prefer **clear, channel-appropriate feedback** over inventing a BFF-style code map.

```
Expected:   validate → message.answer / callback.answer (Russian) → return
Unexpected: log with user id + handler → optional generic reply → do not swallow
DB:         use injected session; middleware opens/closes; commit only on success
Telegram:   let aiogram surface API errors; log context; no SSL-verify hacks in prod
```

## Expected Failures (User-Facing)

For bad input, missing subscription, wrong FSM step, or other **anticipated**
denials:

1. Do **not** throw (or catch-and-ignore).
2. Reply in **Russian**, consistent with existing copy in `app/handlers.py`.
3. Prefer `message.answer(...)` for messages; for callbacks use
   `callback.answer(...)` (toast / alert when appropriate) and/or edit/answer
   a follow-up message.
4. Return early; do not mutate DB state after a denied check.

```python
@router.message(SomeState.waiting_answer)
async def on_answer(message: Message, state: FSMContext, session: AsyncSession):
    text = (message.text or "").strip()
    if not text:
        await message.answer("Пожалуйста, отправь текстовый ответ.")
        return

    user = await session.get(User, message.from_user.id)
    if user is None or not user.is_active:
        await message.answer("Нет доступа. Нажми /start или оформи подписку.")
        return

    # … success path only
```

Subscription gates (when product handlers land): check `User.subscription_end` /
`is_active` via the injected session, then deny with a clear Russian reply — do
not invent a parallel auth envelope.

## Unexpected Exceptions

Unexpected failures (bugs, DB driver errors, unexpected `None`):

- **Log** with context: Telegram `user id`, handler name / update type, and the
  exception (`logging.exception` or `logger.exception`).
- Keep bare `except:` / empty `except Exception: pass` **rare**. Never swallow
  silently.
- Optional: one short generic Russian reply (“Что-то пошло не так…”) so the chat
  is not silent — do not dump stack traces to the user.
- Prefer letting aiogram’s error handlers / outer logging catch fatals rather
  than wrapping every handler in a broad try/except that hides root causes.

```python
import logging

logger = logging.getLogger(__name__)

try:
    await session.commit()
except Exception:
    logger.exception(
        "commit failed user_id=%s handler=%s",
        message.from_user.id,
        "cmd_start",
    )
    await message.answer("Не удалось сохранить данные. Попробуй ещё раз позже.")
    raise  # or return after rollback — do not pass silently
```

If you catch to send a user reply, still log; re-raise or ensure the session is
rolled back cleanly (see middleware).

## DB Session And Middleware

`DbSessionMiddleware` (`app/middlewares.py`) opens an `AsyncSession` per update
and injects `session` into handler `data`. Middleware **owns** open/close via
`async with async_session()`.

- Handlers declare `session: AsyncSession` and use the injected session — do not
  open a second long-lived session in the handler for the same update.
- On failure after partial work: rely on context-manager teardown; call
  `await session.rollback()` when you catch and continue in the same handler.
- Do not hold references to the session after the handler returns.
- Do not “fix” DB errors by disabling constraints or inventing a second SQLite
  path outside `DB_URL` / `data/`.

## FSM Input Failures

`app/states.py` is a stub today; when forms land:

- Invalid or empty input → Russian `message.answer` + stay in state (or clear
  deliberately).
- Cancel / `/start` mid-flow → clear state when appropriate; do not leave orphan
  FSM data without a path out.
- Do not treat validation misses as unexpected exceptions.

## Telegram API Errors

Network timeouts, flood limits, and Bot API rejections are infrastructure, not
product envelopes:

- Log with user id / chat id when available.
- Retry only idempotent, safe operations; respect flood limits.
- Do **not** treat the Windows SSL verify disable in `main.py` as production
  error handling. That branch (`sys.platform == "win32"`) is **local VPN/debug
  only**. Linux / Docker / VPS use clean `Bot(token=…)`. Never copy SSL
  `CERT_NONE` into images or “fix SSL errors” playbooks for the server.

## Logging

Today startup/shutdown use `print` in `main.py`. Prefer module `logging` for
handler/DB failures as features grow:

- Include `user_id` and handler name on unexpected errors.
- `warning` for expected denials that are worth ops visibility (optional).
- `exception` / `error` for unexpected failures.

Do **not** introduce pino, Prometheus, or BFF-style metrics stacks unless there
is an explicit observability decision.

## Checklist For New Failures

1. Identify the channel: expected UX / unexpected / DB session / Telegram API.
2. Expected (bad input, no subscription): Russian `message.answer` /
   `callback.answer`; return; no silent drop.
3. Unexpected: log with user id + handler; never empty catch; optional generic reply.
4. DB: injected session only; middleware owns open/close; rollback on caught failure.
5. No `ServiceError` / `{ errorCode, errorMessage }` / Colyseus-style envelopes.
6. Do not “fix” TLS with production SSL verify disable (Windows debug only).
7. Keep Russian copy consistent with existing handler replies.
8. Add tests when registration / subscription gate contracts become critical
   (`bot-work-with-test` when present).

## Common Mistakes

- Porting `ServiceError` + `{ errorCode, errorMessage }` BFF envelopes into the bot.
- Silently `except Exception: pass` around handlers or commits.
- Opening/closing DB sessions inside handlers instead of using middleware injection.
- Leaving a failed session dirty and continuing to use it in the same update.
- Copying Windows SSL `CERT_NONE` / IPv4 hacks into Docker or VPS as “error handling”.
- Throwing (or logging only) on expected bad FSM input / subscription denial with
  no user-facing Russian reply.
- Dumping English stack traces or error codes into Telegram chats.
- Treating this package like Colyseus (`onAuth`, `client.send("error")`, JWT).

## Key Files

| Path | Role |
|------|------|
| `app/handlers.py` | Routers / commands; user-facing Russian replies |
| `app/middlewares.py` | `DbSessionMiddleware` — session open/close per update |
| `app/database.py` | Engine, `async_session`, `User`, `init_db` |
| `app/states.py` | FSM states (stub) — validate input here when forms land |
| `app/keyboards.py` | Inline topic-menu builders (`menu:*`) |
| `main.py` | Bot / Dispatcher / polling; Windows SSL bypass is debug-only |

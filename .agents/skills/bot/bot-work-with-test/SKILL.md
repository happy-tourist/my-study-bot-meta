---
name: bot-work-with-test
description: >-
  Use when planning or writing pytest + pytest-asyncio tests for my-study-bot
  (aiogram 3): User upsert on /start, subscription_end/is_active gates,
  app/scheduler.py expiry (day/minute windows, check_subscriptions), FSM
  transitions, or DbSessionMiddleware session injection. Core workflow: test
  plan (mocks/verify) → write/extend tests/ → run pytest from bot repo root and
  fix failures. Suite already exists under tests/; extend it for new critical
  flows — do not invent Jest/mocha/Colyseus patterns.
trigger: slash
---

# Work With Test

Use this skill to plan and write tests for the Telegram study bot
(`my-study-bot`, aiogram 3). A **pytest suite already exists** under `tests/`
(e.g. `tests/test_subscription_expiry.py`, `tests/conftest.py`). Extend it when
adding critical flows (registration, subscription gates, scheduler expiry, FSM).
Start with a test plan, then implement tests from that plan.

**Paths:** canonical skill in **my-study-bot-meta** under
`.agents/skills/bot/`. Runtime `app/…`, `main.py`, and `tests/…` paths are
relative to the **bot repository root** (`../my-study-bot`). Sibling meta:
`../my-study-bot-meta`.

Stack: **pytest**, **pytest-asyncio**, **aiogram 3** (handlers as async
callables; `Dispatcher.feed_raw_update` for routing / middleware / DI),
**SQLAlchemy async** + **aiosqlite** (prefer in-memory SQLite for tests),
**APScheduler** helpers via `app/scheduler.py` (call `check_subscriptions` /
window helpers directly — do not require a running scheduler in unit tests).
Assert with pytest `assert`. Tests live under `tests/` as `test_*.py`.

Core principle: exercise the public contract handlers and scheduler expose to
users (DMs, `User` field updates, `/start` greet), not private helpers unless
they are the documented pure API (`classify_subscription_action`,
`reminder_window`). Prefer documented aiogram 3 testing approaches over inventing
Jest, mocha, Vitest, or Colyseus harnesses.

## Pytest / aiogram Testing Setup

| Piece | Role |
|-------|------|
| Runner | `pytest` from bot repo root (optionally `pytest tests/…`) |
| Async | `pytest-asyncio` (`asyncio_mode = auto` in `pytest.ini` / `pyproject.toml`) |
| Framework under test | aiogram 3 Router / Dispatcher — call handlers directly or `dp.feed_raw_update` |
| FSM | `MemoryStorage` + `FSMContext` / `StorageKey` (no Redis in unit tests) |
| DB | In-memory `sqlite+aiosqlite:///:memory:` (or temp file); `Base.metadata.create_all` |
| Telegram API | Mock `Message` / `CallbackQuery` / `Bot.send_message` (`AsyncMock`) — no real `TG_TOKEN`, no polling |
| Scheduler | Call `check_subscriptions(..., session_factory=, now=, unit=)` — no live cron |

Dev deps (ensure present in the bot env; pin reasonably):

```text
pytest
pytest-asyncio
```

`aiosqlite` is already in `requirements.txt`; reuse it for the test DB.

Do **not** add `jest.config`, mocha, `@colyseus/testing`, Vitest, or Node test
runners.

Agent **runs** `pytest` from the bot package root after writing or changing
tests; fix failures before claiming done. Install missing test deps into `.venv`
if needed (`pip install pytest pytest-asyncio`).

## File Layout

| Subject | Test path |
|---------|-----------|
| `app/handlers.py` (`/start`, future commands) | `tests/test_handlers_start.py` (or `tests/test_handlers.py`) |
| Subscription gates (`subscription_end`, `is_active`) | `tests/test_subscription.py` (add with the gate code) |
| `app/scheduler.py` (windows, expire, production defaults) | `tests/test_subscription_expiry.py` |
| `app/states.py` / FSM dialogs | `tests/test_fsm_*.py` |
| `app/middlewares.py` (`DbSessionMiddleware`) | `tests/test_middleware_db.py` |
| Shared fixtures (engine, session_factory, …) | `tests/conftest.py` |

Mirror feature growth under `tests/`; keep imports as `from app.…`. Prefer
`tests/` over a single giant file once coverage spans more than one area.

```python
import pytest
from unittest.mock import AsyncMock

from aiogram import Dispatcher
from aiogram.fsm.storage.memory import MemoryStorage
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
```

## Workflow

1. Read the SUT (`app/handlers.py`, `app/scheduler.py`, `app/database.py` `User`,
   middleware, `app/states.py` when FSM exists) and note the user-facing contract.
2. Produce a test plan with two sections: **What needs to be mocked / stubbed**
   and **What to verify**.
3. Extend existing `tests/` (reuse `conftest.py` fixtures). Only bootstrap
   `pytest` / `pytest.ini` if the suite is missing on a fresh clone.
4. Cover success and failure paths (first `/start` vs return, remind/expire
   windows with `unit="day"` or explicit `unit="minute"`, failed DM does not
   abort batch, production defaults stay day + daily cron, FSM / middleware).
5. Run `pytest` from the bot repo root; fix failures before claiming done.

## Test Plan

### What Needs To Be Mocked / Stubbed

Prefer a real in-memory DB + real router registration over mocking SQLAlchemy
away. Stub only what must not run:

- Telegram Bot API network calls — use `AsyncMock` on `message.answer` /
  `callback.answer`, or a fake bot token with mocked session; never call a live
  bot or start long polling in tests.
- External side effects beyond SQLite (payments, HTTP APIs) when those land.
- Production `DB_URL` / `data/db.sqlite3` — override with in-memory engine in
  fixtures; do not mutate the developer’s real DB file.

Do **not** mock away `User` persistence when testing `/start` upsert — assert
rows via a real `AsyncSession`. Do **not** skip `DbSessionMiddleware` when the
SUT under test is session injection (feed updates through Dispatcher +
middleware).

### What To Verify

Use these categories only when the SUT has relevant behavior:

- `/start` registration: first visit creates `User` (`id` = Telegram id,
  `username` set); reply is the first-visit Russian greeting.
- `/start` return: existing user is not duplicated; return greeting uses
  `first_name`.
- Subscription gates (when implemented): `is_active` / `subscription_end` allow
  or block study handlers; expired / inactive gets the deny path.
- FSM transitions (when implemented): state moves as designed; out-of-order
  messages do not corrupt state; `MemoryStorage` reflects expected state/data.
- Middleware: `DbSessionMiddleware` puts `session: AsyncSession` into handler
  `data` and closes/scopes the session per update (handler can query/commit).

Formulation rules:

- Name the condition being tested.
- Assert outward contract (reply text / state / DB row), not private locals.
- Cover both success and failure when the SUT branches on them.

Examples:

```text
Verify /start User upsert:
- Unknown Telegram id: User row created; answer starts with "Привет! Я тебя запомнил 👋" and includes "Выбери раздел:"; `reply_markup` from `main_menu_kb()`
- Existing id: no second row; answer contains first_name welcome + same topic menu

Verify subscription gate (when implemented):
- is_active True and subscription_end in future: study handler proceeds
- expired or is_active False: deny reply / no privileged side effect

Verify FSM (when implemented):
- Valid step sequence: state advances; collected data in FSMContext
- Message in wrong state: ignored or error reply; state unchanged

Verify DbSessionMiddleware:
- After feed_update / middleware call: handler receives usable AsyncSession
- Handler commit persists User visible in a fresh session from same engine
```

## Coverage Topics

### User upsert on `/start`

Canonical approaches:

1. **Direct handler call** (fast unit): build `AsyncMock` message with
   `from_user.id` / `username` / `first_name`, pass a real test `AsyncSession`,
   await `cmd_start`, assert `message.answer` and DB row.
2. **Dispatcher feed** (routing + DI): `Dispatcher`, `include_router(router)`,
   register middleware or pass `session=` into `feed_raw_update`, feed a raw
   `/start` update dict.

```python
@pytest.mark.asyncio
async def test_start_creates_user(session: AsyncSession):
    message = AsyncMock()
    message.from_user.id = 1001
    message.from_user.username = "learner"
    message.from_user.first_name = "Ann"
    message.answer = AsyncMock()

    from app.handlers import cmd_start

    await cmd_start(message, session)

    message.answer.assert_awaited()
    # assert User(id=1001) exists via select
```

Match Russian copy already in `app/handlers.py` unless product copy changes in
the same change.

### Subscription gates (`subscription_end` / `is_active`)

When product handlers start checking subscription:

- Seed `User` rows with future / past `subscription_end` and `is_active`
  True/False.
- Assert allow vs deny without hitting Telegram.
- Prefer one shared gate helper tested once, then thin handler tests — but do
  not invent a parallel data path; use the existing `User` model.

### FSM transitions

When `app/states.py` gains real `StatesGroup`s:

- Use `MemoryStorage` and `FSMContext` with a `StorageKey`.
- Feed messages through Dispatcher with storage attached, or call handlers with
  an injected `FSMContext`.
- Assert `get_state()` / `get_data()` after each step; cover cancel/clear if
  the product defines them.

### Middleware session injection

```python
@pytest.mark.asyncio
async def test_db_middleware_injects_session():
    # Build Dispatcher + DbSessionMiddleware wired to test async_sessionmaker
    # Feed a simple update to a handler that requires session: AsyncSession
    # Assert handler ran and could execute select/commit
    ...
```

Point middleware at the **test** sessionmaker (patch or construct middleware
with a test factory) so production `DB_URL` is unused.

## Fixtures (`tests/conftest.py`)

Keep fixtures small and local to the bot:

```python
import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.database import Base


@pytest_asyncio.fixture
async def engine():
    eng = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with eng.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield eng
    await eng.dispose()


@pytest_asyncio.fixture
async def session(engine):
    maker = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
    async with maker() as session:
        yield session
```

Add `bot` / `dp` fixtures when Dispatcher-style tests appear. Prefer
`asyncio_mode = auto` so bare async tests work without repeating markers
everywhere (markers still fine).

## Test File Structure

```python
import pytest
from unittest.mock import AsyncMock
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import User
from app.handlers import cmd_start


@pytest.mark.asyncio
async def test_start_registers_new_user(session: AsyncSession):
    message = AsyncMock()
    message.from_user.id = 42
    message.from_user.username = "u"
    message.from_user.first_name = "Test"
    message.answer = AsyncMock()

    await cmd_start(message, session)

    result = await session.execute(select(User).where(User.id == 42))
    assert result.scalar_one_or_none() is not None
    message.answer.assert_awaited()
```

Conventions:

- Files named `test_*.py`; functions `test_*`.
- One focused behavior per test; share setup via fixtures.
- Use pytest `assert` — not Jest `expect`, not Node `assert` modules as the
  primary style guide.
- No real polling (`dp.start_polling`) and no real Telegram network in CI/local
  unit tests.

## Helpers (optional, keep in-file)

Prefer small local helpers over new packages:

```python
def make_message(*, user_id: int, username: str | None = None, first_name: str = "T", text: str = "/start"):
    message = AsyncMock()
    message.text = text
    message.from_user.id = user_id
    message.from_user.username = username
    message.from_user.first_name = first_name
    message.answer = AsyncMock()
    message.bot = AsyncMock()
    return message
```

Share helpers only when a second file needs them (`tests/helpers.py`); default
is a single `SKILL.md` and in-file / `conftest` helpers — no separate md
required.

## Checklist Before Finishing

- Test plan covered; success and failure for `/start` (and gates/FSM when
  present).
- Suite introduced if missing: `pytest`, `pytest-asyncio`, `tests/`, runnable
  from bot repo root.
- In-memory (or temp) DB — production `data/db.sqlite3` untouched.
- No real `TG_TOKEN` / polling / Telegram HTTP in unit tests.
- No Jest / mocha / Colyseus / Vitest APIs or config files.
- Ran `pytest` from the bot package root; failures fixed before claiming done.

## Anti-Patterns

```python
# Jest / mocha / Colyseus — wrong stack
# import { describe, it, expect } from "@jest/globals"
# const colyseus = await boot(appConfig)

# Hitting production SQLite or live Telegram
# DB_URL=sqlite+aiosqlite:///data/db.sqlite3 in tests without override
# Bot(token=os.environ["TG_TOKEN"]); await dp.start_polling(bot)

# Asserting only mocks while claiming upsert works
# message.answer.assert_awaited()  # without checking User row for /start

# Skipping pytest after adding or changing tests
```

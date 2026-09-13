---
name: work-with-fsm
description: >-
  Use when adding, changing, reviewing, or debugging aiogram FSM dialogs in
  my-study-bot: StatesGroup / State in app/states.py, FSMContext in handlers,
  state filters, set/get/update/clear data, MemoryStorage vs Redis, or
  multi-step study/subscription/admin forms (not SQLite User truth in FSM only).
---

# Work With FSM

Use this skill for **aiogram 3 dialog / session state** in `my-study-bot`.

FSM state definitions live in **`app/states.py`**. Handlers that drive steps live in **`app/handlers.py`** (learner) or **`app/handlers_admin.py`** (admin search). Storage is configured on **`Dispatcher`** in **`main.py`**. Do **not** treat FSM data as the source of truth for `User` / subscription fields — persist those in SQLite via the injected `session`.

Skills path (this package): `.agents/skills/bot/`. Runtime paths below are relative to this bot repo root. Canonical OpenSpec / shared skills may also live in sibling `my-study-bot-meta` when that package exists.

Coordinate with sibling skills when they exist: `work-with-handlers`, `work-with-keyboards`, `work-with-database`, `bot-work-with-auth`.

## Map Of Pieces

| Layer | Path | Role |
|-------|------|------|
| Storage | `main.py` | `Dispatcher(storage=…)` — default MemoryStorage if omitted |
| States | `app/states.py` | `StatesGroup` + `State` definitions for multi-step flows |
| Learner handlers | `app/handlers.py` | Enter / step / cancel for learner forms |
| Admin handlers | `app/handlers_admin.py` | Admin search FSM (`AdminSearchForm`) |
| Keyboards | `app/keyboards.py` | Reply / inline UI for form steps |
| Persistence | `app/database.py` | `User` model — durable fields (`subscription_end`, `is_active`, `is_admin`, …) |
| Session inject | `app/middlewares.py` | `DbSessionMiddleware` → handler `session: AsyncSession` |

Today: `app/states.py` defines `AdminSearchForm.waiting_query` for admin user search; topic-menu navigation remains callback `edit_text` (not FSM). Learner study/subscription multi-step forms are still future. `Dispatcher()` in `main.py` uses aiogram’s default in-memory storage.

## Lifecycle Flow

```text
user sends /command or callback that starts a form
        │
        ▼
  handler: state.set(SomeGroup.step1)
        │  optionally state.update_data(...)
        │  prompt + keyboard
        ▼
  next message / callback  (filter: SomeGroup.step1)
        │  state.get_data() / update_data(...)
        │  validate input → set next State or finish
        ▼
  … more steps …
        │
        ▼
  finish: persist durable fields via session (SQLite)
        │  await state.clear()
        │  confirm to user
```

### Responsibility split

| Piece | Do here | Don't |
|-------|---------|--------|
| `StatesGroup` / `State` | Name dialog steps; one group per flow | Put business rules or DB writes in `states.py` |
| Enter handler | `set` first state; seed `update_data`; send prompt | Skip `clear` when restarting an interrupted flow |
| Step handler | Filter by exact `State`; validate; `update_data`; advance | Trust FSM-only copy of subscription / identity |
| Cancel / `/start` interrupt | `state.clear()` (or explicit reset policy) | Leave orphan state after user abandons the form |
| Finish handler | Write `User` (and related) via `session`; then `clear` | Store only in FSM data and call it “saved” |
| Storage | MemoryStorage for single-process bot; Redis if multi-instance / durable dialogs | Add Redis “just in case” without a product need |

## Current Scaffold

### `app/states.py` (grow here)

```python
from aiogram.fsm.state import State, StatesGroup


class AdminSearchForm(StatesGroup):
    waiting_query = State()


class SubscribeForm(StatesGroup):
    waiting_plan = State()
    waiting_confirm = State()


class StudyForm(StatesGroup):
    waiting_topic = State()
    waiting_answer = State()
```

- One `StatesGroup` per user-facing multi-step flow (admin search, study, subscription, …).
- Keep group names product-oriented; avoid a single mega-group for unrelated dialogs.
- Drive admin search steps from `app/handlers_admin.py`; future learner forms from `app/handlers.py`.

### Handler pattern (`app/handlers_admin.py` — admin search)

```python
from aiogram.fsm.context import FSMContext

from app.states import AdminSearchForm


@admin_only.callback_query(F.data == kb.ADMIN_SEARCH)
async def admin_search_start(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    await state.set_state(AdminSearchForm.waiting_query)
    await callback.message.edit_text("Введите id или username:")
```

### Handler pattern (`app/handlers.py` — future learner forms)

```python
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import StateFilter

from app.states import SubscribeForm


@router.message(Command("subscribe"))
async def subscribe_start(message: Message, state: FSMContext):
    await state.clear()
    await state.set_state(SubscribeForm.waiting_plan)
    await state.update_data(plan=None)
    await message.answer("Выбери план…")  # + keyboard from app.keyboards


@router.message(SubscribeForm.waiting_plan)
async def subscribe_plan(message: Message, state: FSMContext, session: AsyncSession):
    await state.update_data(plan=message.text)
    await state.set_state(SubscribeForm.waiting_confirm)
    await message.answer("Подтверди…")


@router.message(SubscribeForm.waiting_confirm)
async def subscribe_confirm(message: Message, state: FSMContext, session: AsyncSession):
    data = await state.get_data()
    # persist User.subscription_end / is_active via session — not FSM-only
    await state.clear()
    await message.answer("Готово.")
```

- Declare `state: FSMContext` when the handler reads or writes dialog state.
- Prefer state-as-filter (`SubscribeForm.waiting_plan`) over loose `StateFilter` unless combining conditions.
- Russian user-facing strings: stay consistent with existing `/start` copy style unless product copy is being redesigned.

### Storage in `main.py`

```python
from aiogram.fsm.storage.memory import MemoryStorage
# from aiogram.fsm.storage.redis import RedisStorage  # only if product needs it

dp = Dispatcher(storage=MemoryStorage())
# Equivalent today: Dispatcher() with no storage arg → MemoryStorage default
```

| Choice | When |
|--------|------|
| **MemoryStorage** (prefer default) | Single bot process (local, one Docker replica). Dialogs reset on restart — OK for short forms. |
| **RedisStorage** | Multiple replicas, or dialogs must survive process restart. Requires Redis URL in env; document in `.env` / deploy notes. |

Do not introduce Redis until multi-instance or durable-dialog requirements are explicit.

## FSMContext API (use these)

| Call | Purpose |
|------|---------|
| `await state.set_state(Group.step)` | Move user into a step (or `None` to clear state only) |
| `await state.get_state()` | Inspect current state (debug / branching) |
| `await state.update_data(**kwargs)` | Merge ephemeral form fields |
| `await state.get_data()` | Read accumulated form data |
| `await state.set_data(dict)` | Replace entire data dict |
| `await state.clear()` | Clear state **and** data — use on finish, cancel, or form restart |

Ephemeral only in FSM data: draft answers, selected plan id before confirm, wizard step flags. Durable: `User.id`, `username`, `subscription_end`, `is_active` → SQLite via middleware `session`.

## Intended Product Flows

Align with AGENTS.md / product when filling stubs:

| Flow | FSM role | Persist where |
|------|----------|---------------|
| `/start` register / greet | Usually **no** FSM | `User` upsert in SQLite (already implemented) |
| Subscription wizard | Multi-step `StatesGroup` | `subscription_end`, `is_active` on `User` at confirm |
| Study / lesson steps | Multi-step answers, topic pick | Lesson progress tables or `User` fields when added — not FSM-only |
| Cancel / interrupt | `clear` on cancel command or competing entry | — |

Keyboards for steps belong in `app/keyboards.py`; keep handlers thin.

## Do / Don't

| Do | Don't |
|----|--------|
| Define steps in `app/states.py` as `StatesGroup` | Scatter magic state strings in handlers |
| Filter step handlers by concrete `State` | Handle all text globally then branch on `get_state()` without need |
| `clear` before starting or after finishing a form | Leave users stuck in a state after success/cancel |
| Persist subscription / identity in SQLite | Put DB truth only in FSM data |
| Prefer MemoryStorage unless product needs Redis | Add Redis storage without deploy + env story |
| Reuse injected `session: AsyncSession` | Open ad-hoc engines inside FSM handlers |
| Keep Russian replies consistent with existing tone | Invent a parallel English-only UX unless asked |

## Change Checklist

1. Belongs in FSM dialog state — not a new parallel “session” store outside aiogram.
2. New/changed `StatesGroup` lives in `app/states.py`; handlers import it.
3. Entry sets state; each step filtered; finish/cancel calls `state.clear()`.
4. Durable fields written through `session` to `User` (or future models), not FSM-only.
5. Storage choice documented: MemoryStorage default, or Redis with env if required.
6. Keyboards for steps in `app/keyboards.py` when UI is more than plain text.
7. No secrets in FSM data; tokens stay in `.env` (`TG_TOKEN`, `DB_URL`, …).
8. Run the bot from this package root (`python main.py`); fix failures before claiming done.

## Related

- Handlers / commands: `.agents/skills/bot/work-with-handlers/SKILL.md` (when present)
- Keyboards: `.agents/skills/bot/work-with-keyboards/SKILL.md` (when present)
- Database / `User`: `.agents/skills/bot/work-with-database/SKILL.md` (when present)
- Bot overview: `AGENTS.md` (handlers, `AdminSearchForm`, SQLite)
- Meta OpenSpec / skills index: `my-study-bot-meta/.agents/AGENTS.md` (when meta exists)

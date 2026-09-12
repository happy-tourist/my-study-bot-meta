---
name: work-with-keyboards
description: >-
  Use when adding, changing, reviewing, or debugging Telegram reply/inline
  keyboards in the my-study-bot aiogram study bot — especially builders in
  `app/keyboards.py`, `reply_markup=` wiring from handlers (`import app.keyboards
  as kb`), Reply vs Inline choice, callback_data conventions, or Russian button
  labels. No direct server analog; needed for aiogram UX. Not for FSM state
  machines alone (`app/states.py`) or DB/subscription logic.
---

# Work With Keyboards

Use this skill for **Telegram keyboards** in `my-study-bot` (aiogram v3 reply / inline markup).

Canonical skill path: `my-study-bot-meta/.agents/skills/bot/work-with-keyboards/`. Runtime paths below are relative to sibling bot repo root (`../my-study-bot`).

Coordinate with: `bot-locate-change-points` (where to edit), handlers / FSM in `app/handlers.py` + `app/states.py`, package overview in `{bot}/AGENTS.md`.

## Scope Boundary

| In scope | Out of scope |
|----------|--------------|
| Builders in `app/keyboards.py` (`InlineKeyboardMarkup` / `ReplyKeyboardMarkup`, `InlineKeyboardBuilder` / `ReplyKeyboardBuilder`, buttons) | Heavy business logic (subscription checks, DB writes, study rules) |
| Attaching markup via `reply_markup=` in handlers (`import app.keyboards as kb`) | FSM state definitions alone — `app/states.py` |
| Reply vs Inline choice; `callback_data` shape | Bot token / polling / Windows SSL hacks — `main.py` |
| Russian button labels aligned with message copy | Admin web UI / HTTP API (none in this package) |

## Bot Today

- `app/keyboards.py` — builders: `main_menu_kb()`, `back_to_menu_kb()`; shared `menu:*` callback constants (`MENU_CARS`, `MENU_HOUSES`, `MENU_SUBSCRIPTION`, `MENU_BACK`).
- `app/handlers.py` — `import app.keyboards as kb`; `/start` attaches `main_menu_kb()`; section stubs navigate via `edit_text` + `callback.answer()`.
- Prefer **factory functions** in `keyboards.py` that return markup; handlers only call them and pass `reply_markup=`. Keep callback filters in sync with shared constants (`F.data == kb.MENU_…`).

## Placement Pattern

```python
# app/keyboards.py — build markup only
def main_menu_kb() -> InlineKeyboardMarkup:
    ...

# app/handlers.py — attach only
await message.answer("…", reply_markup=kb.main_menu_kb())
```

### Hard rules

1. **Builders live in `app/keyboards.py`.** Do not inline large `InlineKeyboardMarkup([...])` trees inside handlers unless truly one-off and tiny.
2. **Handlers attach** with `reply_markup=kb.<builder>(…)`. Keep `import app.keyboards as kb`.
3. **No heavy business logic in builders.** No DB sessions, subscription gates, or study scoring inside keyboard factories. Pass already-decided flags/labels in, or decide in the handler / service layer first.
4. **Russian labels** must match existing user-facing message tone (`Привет!…`, `С возвращением…`) unless product copy is being redesigned.

## Reply vs Inline

| Use | When | Notes |
|-----|------|-------|
| **Inline** (`InlineKeyboardMarkup` / inline builder) | **Current product:** post-`/start` topic menu, section stubs, back navigation; also contextual actions, toggles, confirmations | Does not replace the reply keyboard; drives `CallbackQuery` + usually `edit_text` |
| **Reply** (`ReplyKeyboardMarkup` / reply builder) | Persistent “always visible” actions when product asks for a reply keyboard | Replaces the user’s keyboard; not used for today’s topic menu |

Prefer **one clear job** per keyboard. Do not mix unrelated actions on the same row without a product reason.

Today’s topic menu is intentionally **inline under the greet message** (not a reply keyboard). Prefer extending `main_menu_kb` / `back_to_menu_kb` + `menu:*` constants before inventing a parallel menu surface.

Remove or replace reply keyboards explicitly when leaving a flow (`ReplyKeyboardRemove` or a new menu) so stale buttons do not linger.

## Callback Data Conventions

Inline buttons need stable `callback_data` handled by `CallbackQuery` filters in handlers.

| Rule | Do | Don't |
|------|----|--------|
| Stable | Fixed prefixes that survive refactors (`menu:`, `sub:`, `study:`) | Ad-hoc strings that change with every copy tweak |
| Short | Stay well under Telegram’s **64-byte** limit | Stuff long Russian labels or JSON blobs into `callback_data` |
| Namespaced | `area:action` or `area:action:id` (e.g. `study:next`, `sub:buy:1m`) | Bare verbs (`ok`, `yes`) shared across features |
| Parseable | Split on `:` (or a single delimiter) in the handler | Free-form prose |

Button **text** is user-facing Russian; **callback_data** is machine-facing English/slug namespace. Never put secrets in callback_data.

Wire handlers with filters such as `F.data == "…"` or `F.data.startswith("study:")` — keep the string literals in sync with the builder (prefer shared constants next to the builders if duplication appears).

## Builder Shape (aiogram 3)

Prefer aiogram keyboard builders when rows grow; plain markup constructors are fine for tiny static boards.

```python
from aiogram.types import InlineKeyboardMarkup
from aiogram.utils.keyboard import InlineKeyboardBuilder

MENU_CARS = "menu:cars"
# … MENU_HOUSES, MENU_SUBSCRIPTION, MENU_BACK …

def main_menu_kb() -> InlineKeyboardMarkup:
    b = InlineKeyboardBuilder()
    b.button(text="🚗 Машины", callback_data=MENU_CARS)
    # …
    b.adjust(1)
    return b.as_markup()

def lesson_actions(lesson_id: int) -> InlineKeyboardMarkup:
    b = InlineKeyboardBuilder()
    b.button(text="Дальше", callback_data=f"study:next:{lesson_id}")
    b.button(text="Назад", callback_data="study:back")
    b.adjust(2)
    return b.as_markup()
```

Prefer `InlineKeyboardBuilder` / `ReplyKeyboardBuilder` in `app/keyboards.py`; drop unused imports.

Dynamic labels (e.g. include a name) are OK when the **structure** stays in the builder and values are passed in as arguments — still no DB I/O inside the factory.

## Do / Don't

| Do | Don't |
|----|--------|
| Add/change builders in `app/keyboards.py` | Embed markup construction deep inside unrelated helpers |
| Attach with `reply_markup=kb.…` from handlers | Put subscription/DB logic inside keyboard factories |
| Prefer Inline for the post-`/start` topic menu (current product) | Invent a second parallel menu surface beside `menu:*` without reason |
| Keep `callback_data` short, stable, namespaced | Put Russian UI copy or PII into `callback_data` |
| Match Russian button text to message copy | Invent English button labels while messages stay Russian |
| Share constants for callback prefixes when reused | Duplicate magic strings across many modules without need |

## Change Checklist

1. Builder added or updated in `app/keyboards.py` (not only inlined in the handler).
2. Handler imports `app.keyboards as kb` and passes `reply_markup=…`.
3. Reply vs Inline choice matches the UX (today: inline topic menu after `/start`; reply only when product asks).
4. Inline `callback_data` is stable, short, namespaced; handler filter matches exactly.
5. Russian button labels consistent with existing replies / product copy.
6. No DB/session/subscription business logic inside the keyboard module.
7. Stale reply keyboards cleared or replaced when leaving a flow, if applicable.
8. Callback handlers registered on the same router flow that expects those buttons.
9. Run the bot / relevant checks from this package root as appropriate; fix failures before claiming done.

## Related

- Where to edit: `.agents/skills/bot/bot-locate-change-points/SKILL.md`
- Handlers: `app/handlers.py`
- FSM stubs: `app/states.py`
- Package overview / UX notes: `{bot}/AGENTS.md`

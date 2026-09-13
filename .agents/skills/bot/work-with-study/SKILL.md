---
name: work-with-study
description: >-
  Use when implementing, changing, reviewing, or debugging product study-domain
  logic (lessons, study flows, subscription-gated learning features) in the
  my-study-bot Telegram study bot — handlers, FSM states, User/subscription
  fields, or related business rules.
---

# Work With Study

Use this skill for **продуктовая логика обучения (study flows)** in `my-study-bot`.

Bot owns study/subscription truth in SQLite via the `User` model and injected sessions. Telegram is the UX surface only. Product is still early — guide implementation toward the intended product below; do not treat unimplemented study features as shipped.

Pair with:
- Subscription / access gate: `bot-work-with-auth` (when present)
- Multi-step lessons/forms: `work-with-fsm` (when present)
- Russian user-facing copy: `work-with-messages` (when present)

Skills path for now: `.agents/skills/bot/` in this repo (canonical copy may later live under `my-study-bot-meta`). Runtime paths below are relative to this bot repo root.

## Overview

| Surface | Path | Role |
| --- | --- | --- |
| Handlers | `app/handlers.py` | Study commands/callbacks; thin glue over session + rules |
| FSM | `app/states.py` | Multi-step lesson / form states (`StatesGroup`) |
| Keyboards | `app/keyboards.py` | Reply/inline markup only — no business rules |
| User / DB | `app/database.py` | `User` (+ future study-related columns); extend, don't fork |
| Middleware | `app/middlewares.py` | `DbSessionMiddleware` → `session: AsyncSession` in handlers |
| Entry | `main.py` | Bot/Dispatcher/polling/router include only — not study rules |

Scaffold today: `/start` registers `User` (first visit grants one-time trial), greets in Russian, and shows an inline topic menu. Cars/Houses are gated via `app/auth.py` `has_active_subscription`; Subscription opens tariffs (grant without payment). Study topic **content** is still stubs — do not claim lessons/progress as shipped. `app/keyboards.py` has topic/tariff/gate CTA builders; `app/states.py` is still a stub.

### Current vs intended product (scaffold)

| Expectation | Today |
| --- | --- |
| Register user on `/start` | Implemented (`User` upsert + trial on first visit + topic menu) |
| Subscription / study features | Tariffs + Cars/Houses gate live; study lesson content still stubs |
| FSM forms / keyboards | Topic menu, tariffs, subscription-required CTA; `app/states.py` still a stub |
| Modular routers | Single `app/handlers.py` + `app/auth.py` gate helper |

When adding study/subscription behavior, prefer extending the existing `User` model and middleware session injection rather than inventing a parallel data path.

## Authority

- SQLite `User` (and future study tables/columns) is the only durable truth for registration, subscription, and study progress once implemented.
- Handlers read/write via injected `session: AsyncSession` — do not open ad-hoc engines or bypass middleware for normal update handling.
- Subscription gate **before** study features: use `has_active_subscription` (`app/auth.py`) — `is_active` + `subscription_end > now`. Gate policy for topic sections is fixed; lesson-level rules remain product decisions when real study flows land.
- Do not put study business logic in `main.py` or only inside keyboard builders.

## Domain Surfaces

### Handlers (`app/handlers.py`)

- Own Telegram triggers for study: commands, callbacks, message filters that advance a lesson/form.
- Keep handlers thin: parse update → gate (auth/subscription) → call shared study helpers or FSM transitions → answer with Russian copy.
- Prefer extending the existing `router` until size justifies a dedicated study router included from `main.py` (registration only in `main.py`).

### FSM (`app/states.py`) — pair with `work-with-fsm`

- Multi-step lessons, quizzes, and forms live in `StatesGroup` / `State` definitions here; handlers use `FSMContext`.
- Do not encode lesson branching only as nested if-chains in one handler without states when the flow is multi-step.
- Scaffold is nearly empty today — adding study states is expected growth, not a shipped library of lessons.

### Keyboards (`app/keyboards.py`)

- Build reply/inline markup only.
- Callback data / button labels may identify steps; **rules** (what answer is correct, whether the user may proceed, subscription checks) stay in handlers / shared study helpers, not in keyboard modules alone.

### Persistence (`app/database.py` + middleware)

| Column (today) | Notes |
| --- | --- |
| `id` | Telegram BigInteger PK |
| `username` | Optional |
| `subscription_end` | Optional datetime — gate via `has_active_subscription` |
| `is_active` | Soft active / deactivate on expiry |
| `trial_used` | One-time free trial consumed |
| `is_active` | Boolean, default `True` |
| `created_at` | utcnow default |

- Extend `User` (or add clearly related models) for progress, lesson position, etc. Prefer one coherent schema path.
- Handlers that need DB must declare `session: AsyncSession`.
- Schema migrations: `init_db` = `create_all` + `_ensure_sqlite_user_columns`. New `User` columns → model + `_SQLITE_USER_COLUMN_DDL` (startup ALTER on VPS volume; no wipe on push).

### Entry (`main.py`)

- Register middleware, routers, polling. Windows SSL/IPv4 hacks stay in the `win32` branch only.
- **No** lesson rules, subscription checks, or FSM state definitions here.

## Architecture Preference

```
Telegram update
  → DbSessionMiddleware (session)
  → subscription / auth gate (bot-work-with-auth)
  → handler (thin) + FSM states (work-with-fsm)
  → study rules helpers (pure / testable where practical)
  → User / DB via session
  → Russian replies (work-with-messages) + keyboards (markup only)
```

- Prefer small pure helpers for “may proceed / next step / score” so handlers stay thin and testable without Telegram I/O.
- Shared UI in `keyboards.py`; multi-step dialogs in `states.py`; durable fields on `User` (or related models).
- Do not invent a second store (files, in-memory global dicts as source of truth) alongside SQLite for product study state.

## Implementation Checklist (scaffold → product)

- [ ] Subscription gate before study entry (`has_active_subscription`; Cars/Houses already gated — extend when real lessons land).
- [ ] Study commands/callbacks in handlers; FSM states for multi-step lessons/forms.
- [ ] Extend `User` / related models for progress only as product defines — do not invent a full LMS schema without requirements.
- [ ] Keyboards for navigation only; business rules outside keyboard-only files.
- [ ] Russian copy consistent with existing `/start` tone unless copy is redesigned (`work-with-messages`).
- [ ] Keep `main.py` free of study domain logic.
- [ ] When critical flows land: pytest + aiogram testing helpers for gate + happy-path lesson steps.
- [ ] Align product contracts with `../my-study-bot-meta` OpenSpec when present — do not invent meta files from this skill.

## Do

- Extend `User` + middleware `session`; put study flows in handlers / FSM / states.
- Gate study features on subscription/access before lesson UX (`app/auth.py`).
- Keep study rules out of `main.py` and out of keyboards-only modules.
- Use FSM for multi-step lessons/forms; Russian user-facing strings.
- Mark undecided product rules (lesson model, progress shape, payments) as open questions.
- Stay scaffold-aware: describe **target** behavior; do not claim unimplemented study lessons as shipped.

## Don't

- Bypass `has_active_subscription` for new study entry points — topic gate is live; keep it.
- Put authoritative study logic only in `main.py` or only in `app/keyboards.py`.
- Invent a parallel user/progress database outside the SQLAlchemy `User` path without an explicit product decision.
- Ship English learner-facing copy by default; match existing Russian replies unless redesigning copy.
- Assert a full lesson catalog, scoring engine, or payment flow exists in code when it does not.

## Related

- Change-point map: `.agents/skills/bot/bot-locate-change-points/SKILL.md`
- Auth / subscription gate: `bot-work-with-auth` (when present under `.agents/skills/bot/`)
- FSM: `work-with-fsm` (when present)
- Messages / copy: `work-with-messages` (when present)
- Package overview: `AGENTS.md` (Current vs intended product)
- Specs / OpenSpec: sibling `../my-study-bot-meta` when present

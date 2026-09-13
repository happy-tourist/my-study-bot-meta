---
name: work-with-scheduler
description: >-
  Use when adding, changing, reviewing, or debugging my-study-bot background
  jobs: APScheduler AsyncIOScheduler in app/scheduler.py, subscription expiry
  reminders (3/2/1) and is_active deactivation, EXPIRY_WINDOW_UNIT day vs minute,
  Europe/Moscow daily cron, start_scheduler/stop_scheduler in main startup/shutdown,
  or tests that pass unit=/now=/session_factory into check_subscriptions.
---

# Work With Scheduler

Use this skill for **background subscription expiry** in `my-study-bot`
(APScheduler + SQLAlchemy), not for Telegram update handlers or Docker deploy.

Canonical skill path: `my-study-bot-meta/.agents/skills/bot/work-with-scheduler/`.
Runtime paths (`app/scheduler.py`, `main.py`, `tests/…`) are relative to the
sibling bot repo root (`../my-study-bot`).

Stack: **APScheduler** `AsyncIOScheduler` (pinned in `requirements.txt`),
aiogram `Bot.send_message`, SQLAlchemy async `async_session` / optional
`session_factory` injection for tests. Timezone of the scheduler:
`Europe/Moscow`.

Related skills: `bot-work-with-auth` (gates / `User` fields),
`bot-work-with-structure` (placement), `work-with-config` (startup hooks),
`bot-work-with-test` (pytest coverage).

## Map Of Pieces

| Piece | Path | Role |
|-------|------|------|
| Scheduler module | `app/scheduler.py` | `AsyncIOScheduler`, window math, `check_subscriptions`, start/stop |
| Wire-up | `main.py` | `startup` → `start_scheduler(bot)`; `shutdown` → `stop_scheduler()` |
| User fields | `app/database.py` `User` | `subscription_end`, `is_active` — source of truth |
| Tests | `tests/test_subscription_expiry.py` | Windows, classify, delivery, production defaults |

## Production Defaults (do not regress)

| Setting | Production value | Notes |
|---------|------------------|-------|
| `EXPIRY_WINDOW_UNIT` | `"day"` | Module default; minute only via explicit `unit=` (tests / temp harness) |
| Cron | `hour=10`, `minute=0` | Daily at 10:00 Europe/Moscow — **not** `minute="*"` |
| Reminders | N ∈ {3, 2, 1} | Window `[now+N, now+N+1)` in the active unit |
| Expire | `subscription_end < now` | Sets `is_active=False`, then notifies |

`start_scheduler` must keep the daily cron. Reverting to every-minute cron is a
temporary harness only — restore day + `hour=10`/`minute=0` before shipping.

## Core Flow

```text
main startup
   └─ start_scheduler(bot)
         └─ cron 10:00 Europe/Moscow → check_subscriptions(bot)
               ├─ for N in 3,2,1: users in reminder window → send_message
               └─ active users with subscription_end < now
                     → is_active=False + commit → send expired notice
main shutdown
   └─ stop_scheduler()  # shutdown(wait=False) if running
```

`check_subscriptions` opens its **own** session via `async_session` (or injected
`session_factory`). It does **not** use `DbSessionMiddleware` — background jobs
are outside Telegram updates.

## API Surface (`app/scheduler.py`)

| Symbol | Role |
|--------|------|
| `ExpiryWindowUnit` | `"day"` \| `"minute"` |
| `EXPIRY_WINDOW_UNIT` | Module default (`"day"` in production) |
| `reminder_window(now, n, *, unit=)` | UTC half-open window for remind-N |
| `classify_subscription_action(...)` | Pure: `remind_3`/`remind_2`/`remind_1`/`expire`/`None` |
| `check_subscriptions(bot, *, session_factory=, now=, unit=)` | Side effects: DM + deactivate |
| `start_scheduler(bot)` / `stop_scheduler()` | Lifecycle; job id `check_subscriptions` |

Prefer extending these helpers over duplicating window math in handlers.

## Placement Rules

| Do | Don't |
|----|--------|
| Keep job logic in `app/scheduler.py` | Stuff expiry loops into `main.py` or handlers |
| Wire start/stop only from `main` hooks | Start a second `AsyncIOScheduler` elsewhere |
| Use `async_session` / injected `session_factory` in the job | Expect middleware `session` inside the cron |
| Pass `unit="minute"` / fixed `now=` in tests | Flip module default to `"minute"` permanently |
| Keep Russian reminder / expired copy in scheduler | Invent English-only system DMs |

Do **not** reintroduce temporary minute grant buttons in the subscription stub
unless a new OpenSpec change asks for a verification harness. Product subscription
UX is a separate change; the scheduler owns reminders and deactivation only.

## Testing

- Suite: `tests/test_subscription_expiry.py` (+ `tests/conftest.py` in-memory engine).
- Call `check_subscriptions(..., session_factory=..., now=..., unit="day"|"minute")`.
- Assert production defaults separately (e.g. `EXPIRY_WINDOW_UNIT == "day"`, cron
  source has `hour=10` / `minute=0`, no `minute="*"`).
- Mock `Bot.send_message`; failed delivery must not abort the rest of the batch.

Run from bot root: `pytest` (or `pytest tests/test_subscription_expiry.py`).

## Change Checklist

1. Defaults still day + daily Moscow 10:00 unless an explicit harness task says otherwise.
2. `startup`/`shutdown` still call start/stop; no orphaned scheduler after stop.
3. Job still uses `User.subscription_end` / `is_active` — no parallel store.
4. Window math stays half-open `[now+N, now+N+1)`.
5. Exceptions on `send_message` are logged; other users still processed.
6. Tests updated for any contract change; `pytest` green from bot root.

## Related

- Auth gates in handlers → `bot-work-with-auth`
- Folder map / dependency direction → `bot-work-with-structure`
- Startup wiring → `work-with-config`
- Pytest patterns → `bot-work-with-test`

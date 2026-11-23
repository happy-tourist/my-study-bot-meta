# Topic gate / ban / slash unregistered

Read when changing Cars/Houses/Computers/Chinese (or new topic stubs),
`has_active_subscription`, ban overrides, or slash-vs-inline parity.

## Shared openers

| Surface | Helper | Notes |
|---------|--------|-------|
| Inline `menu:cars` / `menu:houses` / `menu:computers` / `menu:chinese` | `_open_gated_topic_stub` | Ban → gate → stub + `back_to_menu_kb`, else refuse + `subscription_required_kb` |
| Slash `/cars` `/houses` `/computers` `/chinese` | `_cmd_gated_topic` → `_send_gated_topic` | Unregistered → «Сначала нажми /start»; else same ban → gate → stub via `message.answer` |
| Gate math | `app/auth.py` `has_active_subscription` | `is_active` + `subscription_end > now`; do not duplicate in each handler |
| Ban | `is_banned_user` | «Доступ закрыт.» before subscription CTA |

Chinese mirrors Computers: `MENU_CHINESE` / `cmd_chinese` / `menu_chinese`, section title «Курсы китайского».

## How to gate a handler

1. Declare `session: AsyncSession` (middleware must stay registered).
2. Resolve `telegram_id` from `message.from_user.id` or `callback.from_user.id`.
3. `select(User).where(User.id == telegram_id)` → if missing, ask to `/start` (slash) or refuse per product.
4. If banned → «Доступ закрыт.»
5. If `not has_active_subscription(user)` → subscription-required copy + CTA.
6. Only then run the stub / study flow.

```python
from datetime import datetime

from app.database import User


def has_active_subscription(user: User | None, *, now: datetime | None = None) -> bool:
    now = now or datetime.utcnow()
    if user is None or not user.is_active:
        return False
    if user.subscription_end is None:
        return False
    return user.subscription_end > now
```

- Same clock basis as stored `subscription_end` (naive UTC via `datetime.utcnow` today).
- Do not trust FSM state or `callback_data` as proof of subscription.
- Subscription / `tariff:*` stays open when registered and not banned; `/help` needs no `/start`.
- Expiry reminders / deactivation: `app/scheduler.py` (`work-with-scheduler`) — do not duplicate window math here.

## Adding a topic section

1. `MENU_*` + button in `main_menu_kb` (order: topics before Subscription).
2. Thin `menu_*` + `cmd_*` wrappers → shared gated helpers.
3. `BotCommand` in `learner_bot_commands` with matching RU/emoji label.
4. Tests: allow / refuse / ban + Menu Button / slash (see `bot-work-with-test`).

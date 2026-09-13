## Context

Пакет: **bot** (`../my-study-bot`). Мотивация — `proposal.md`. Поведение — delta-specs `start/register`, `subscription/tariffs`, `subscription/gate`, `subscription/expiry`.

Сейчас: `User` без `trial_used`; `/start` без триала; «Подписка» — stub; темы без gate; `EXPIRY_WINDOW_UNIT = "day"`, cron `hour=10, minute=0`; ветка `unit="minute"` уже есть в `app/scheduler.py`. Чеклист — `tasks.md`.

## Goals / Non-Goals

**Goals:**

- Поле `trial_used` + разовый триал 3 минуты на первом `/start`.
- Каталог `TARIFFS` + inline-кнопки; grant без ЮKassa; stacking от `max(now, subscription_end)`.
- Gate: «Машины»/«Дома» только при активной подписке; «Подписка» всегда.
- Минутный expiry: `EXPIRY_WINDOW_UNIT = "minute"`, cron `minute="*"`.
- Сброс локальной SQLite; pytest (trial, tariffs, gate, expiry).

**Non-Goals:**

- ЮKassa, инвойсы, webhooks.
- Возврат дневного cron/окон в этом change.
- Наполнение stubs «Машины»/«Дома» (кроме gate).

## Decisions

### D1 — Capability paths

| ID | Роль |
|----|------|
| `start/register` | триал + меню; темы с учётом gate |
| `subscription/tariffs` | каталог и grant |
| `subscription/gate` | allow/deny тем |
| `subscription/expiry` | временный minute mode |

### D2 — Модель и сброс БД

- `User.trial_used: Mapped[bool] = mapped_column(Boolean, default=False)`.
- При apply удалить локальный файл БД по `DB_URL` (типично `data/db.sqlite3`), затем `init_db`.

### D3 — Триал на `/start`

Ветка `user is None`: `trial_used=True`, `is_active=True`, `subscription_end = utcnow() + 3 minutes`; текст «3 дня (3 мин)» + topic-меню. Returning без нового триала.

### D4 — Каталог тарифов и UI

| id | title (UI) | minutes | price display |
|----|------------|---------|---------------|
| `1_month` | 1 месяц (30 мин) | 30 | 99 ₽ (9900) |
| `3_months` | 3 месяца (90 мин) | 90 | 249 ₽ |
| `forever` | Навсегда (36500 мин) | 36500 | 999 ₽ |

- `callback_data`: `tariff:1_month` / `tariff:3_months` / `tariff:forever`.
- `menu:subscription` → тарифы + Back (без gate).
- Forever = 36500 минут.

### D5 — Grant без оплаты

`tariff:*` → stacking `base = max(now, future end)` → `+ minutes`; `is_active=True`; русское подтверждение.

### D6 — Expiry minute harness (временно)

`EXPIRY_WINDOW_UNIT = "minute"`; cron `minute="*"`; тесты SC-EXP на минуты. Дни + daily 10:00 — later с ЮKassa.

### D7 — Gate (новый в scope)

По канону `bot-work-with-auth`:

```text
has_active_subscription(user, now):
  user is not None
  and user.is_active
  and user.subscription_end is not None
  and user.subscription_end > now
```

- Часы: naive UTC (`datetime.utcnow`), как у `subscription_end`.
- Shared helper (предпочтительно маленький модуль/`app/auth.py` или рядом с handlers — без дублирования в каждом handler).
- `menu:cars` / `menu:houses`: если false → русский отказ («нужна подписка»), показать/сохранить путь к «Подписка» (кнопка или текст), **не** открывать stub; `callback.answer`.
- Если true → текущий stub + Back.
- `menu:subscription` и `tariff:*` — **без** gate.
- Не middleware на весь update (не ломать `/start` и подписку); проверка в handlers тем или общий helper, вызываемый из них.

Альтернатива «прятать кнопки тем» отклонена: кнопки остаются, отказ при нажатии яснее для UX оплаты.

### D8 — Файлы и слои

| Файл | Изменение |
|------|-----------|
| `app/database.py` | `trial_used` |
| `app/handlers.py` | триал; тарифы; gate на cars/houses |
| helper auth | `has_active_subscription` |
| `app/keyboards.py` | builders тарифов |
| `app/scheduler.py` | minute unit + minutely cron |
| `tests/` | expiry, trial, tariff, gate |
| Локальный DB | удалить при apply |

### D9 — Technical prerequisites — закрыты

Grant / minute / DB reset / forever / **gate в этом change** — зафиксированы. Открытых блокеров нет.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Забыть вернуть дни после ЮKassa | Out of scope + comment в scheduler |
| Gate дублируется в handlers | Один helper |
| Отказ без CTA на подписку | В refuse — явный намёк / кнопка «Подписка» |
| Сброс БД на VPS | Только локальный `DB_URL` |

## Migration Plan

1. Сброс SQLite → схема с `trial_used`.
2. Триал + тарифы + gate + minute cron.
3. Проверка: `/start` → темы открываются; после expire → отказ; «Подписка» → тариф → снова доступ.
4. Позже: ЮKassa + day windows.

## Open Questions

Нет.

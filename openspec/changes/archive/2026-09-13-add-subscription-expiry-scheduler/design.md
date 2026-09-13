## Context

Пакет: **bot** (`../my-study-bot`). Мотивация — `proposal.md`. Поведение — `specs/subscription/expiry/spec.md`.

Сделано: `apscheduler`, `app/scheduler.py`, wiring в `main.py`, `tzdata`, pytest по дневным окнам. **Финал:** дневные окна 3/2/1 UTC, cron `hour=10, minute=0` Europe/Moscow; минутный харнесс и кнопки выдачи 1м/5м после проверки **сняты** (раздел «Подписка» снова stub). Чеклист — `tasks.md`.

## Goals / Non-Goals

**Goals:**

- Прод-логика expiry: дневные окна 3/2/1 UTC + daily cron 10:00 Europe/Moscow + deactivate.
- Ошибки Telegram на одного user не рвут прогон.
- После живой проверки: код без minute-cron / minute-windows и без временных grant-кнопок.

**Non-Goals:**

- Gate, оплата, multi-instance lock, dedup напоминаний.
- Продуктовый UX подписки / постоянные кнопки выдачи срока.
- Менять stubs «Машины» / «Дома» / оставлять тестовые кнопки в «Подписка».

## Decisions

### D1 — Часы и расписание (hybrid) — финал

- Сравнение окон и `subscription_end`: naive **UTC**.
- **Прод-cron (итог change):** `hour=10, minute=0`, timezone `Europe/Moscow`.
- **Docker:** `tzdata` в образе.

### D2 — Capability

- `subscription/expiry` (не смешивать с `subscription/gate`).

### D3 — Deactivate без gate

- Пишем `is_active=False` + сообщение; gate — отдельный change.

### D4 — Один процесс, без dedup

- Один контейнер. Нет колонки «уже напомнили». Ускоренный cron и минутные окна были только для проверки и **возвращены** к дневному режиму до archive.

### D5 — Стек и файлы (ядро expiry)

| Что | Решение |
|-----|---------|
| Библиотека | `apscheduler==3.10.4`, `AsyncIOScheduler` |
| Модуль | `app/scheduler.py`: `check_subscriptions`, `start_scheduler`, `stop_scheduler`, чистые хелперы окон |
| Сессия БД | `async with async_session()` внутри job |
| Модель | только `User` |
| Wiring | `startup` → `start_scheduler(bot)`; `shutdown` → `stop_scheduler()` |
| Прод-выборка remind | end ∈ `[now+N days, now+N+1 day)` для `N in (3,2,1)` |
| Expire | `subscription_end < now` → `is_active=False`, commit, сообщение |
| Send | per-user `try/except` |

### D6 — Ускоренная проверка (исторически) → заменена D8

D6 (SQL seed +2d + временный cron) частично выполнен; живой SC-EXP-02 через SQL на этой машине блокировался сетью. Дальнейшая проверка шла через **D8** (кнопки + минутные окна), затем revert.

### D7 — Тесты

- Pytest чистой логики по **дневным** окнам SC-EXP-01…06 — канон; финальный код и тесты на дневной семантике.

### D8 — Временный тест-харнесс (исторически; снят)

Использовался для живой проверки, затем **полностью снят** из shipping:

1. Только для теста; постоянные тарифы — later.
2. Минутные remind-окна и `minute="*"` — временно, затем revert к дням / daily cron.
3. Две кнопки «На 1 минуту» / «На 5 минут» — временно; **post-verify cleanup:** удалены из runtime, раздел «Подписка» снова stub.

| Элемент | Итог |
|---------|------|
| Cron на время проверки | был `minute="*"` → **revert** к `hour=10, minute=0` |
| Окна remind на время проверки | были минуты N∈{3,2,1} → **revert** к дням |
| Кнопки 1м/5м | были для стенда → **удалены** после проверки |
| Финал | дневные окна + daily Moscow cron; без grant-кнопок |

**Альтернативы отклонены:** one-shot job на user; постоянные минутные окна в проде; оставлять тест-кнопки как shipping UX.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Забыли вернуть дни / daily cron → спам и неверные remind | Явные tasks revert; verify `rg` перед archive |
| Тест-кнопки остались бы на проде | Post-verify cleanup: кнопки удалены; stub «Подписка» |
| Pytest дневных окон краснеет при minute-unit в коде | Финальный код и тесты — дневные |

## Migration Plan

1. Ядро scheduler + wiring + tzdata + дневные тесты.
2. Временно: тест-харнесс D8 → deploy → проверить 1м и 5м в Telegram.
3. Revert окон и cron к прод-значениям; удалить тест-кнопки (stub «Подписка»).
4. Sync/archive OpenSpec; product UX подписки — отдельный change.

## Technical prerequisites (from explore)

D1–D5 закрыты. D8 выполнен и снят. Открытых блокеров нет.

## Open Questions

Нет.

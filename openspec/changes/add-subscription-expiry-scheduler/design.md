## Context

Пакет: **bot** (`../my-study-bot`). Мотивация — `proposal.md`. Поведение — `specs/subscription/expiry/spec.md`.

Уже сделано (частичный apply): `apscheduler`, `app/scheduler.py`, wiring в `main.py`, `tzdata`, pytest по дневным окнам. На main сейчас временно `minute="*"` (проверка на проде). Stub «Подписка» ещё без кнопок выдачи. Чеклист — `tasks.md`.

## Goals / Non-Goals

**Goals:**

- Прод-логика expiry: дневные окна 3/2/1 UTC + daily cron 10:00 Europe/Moscow + deactivate.
- Временный тест-харнесс: minute-cron; remind-окна в **минутах** N∈{3,2,1}; две кнопки в «Подписка» (1 мин / 5 мин).
- После живой проверки: вернуть дневные окна и прод-cron в коде (MUST перед закрытием change).
- Ошибки Telegram на одного user не рвут прогон.

**Non-Goals:**

- Gate, оплата, multi-instance lock, dedup напоминаний.
- Финальный продуктовый UX подписки (кнопки — временный стенд).
- Менять stubs «Машины» / «Дома».

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

- Один контейнер. Нет колонки «уже напомнили». Ускоренный cron и минутные окна **обязаны** быть возвращены к дневному режиму до archive.

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

D6 (SQL seed +2d + временный cron) частично выполнен; живой SC-EXP-02 через SQL на этой машине блокировался сетью. Дальнейшая проверка — через **D8** (кнопки + минутные окна).

### D7 — Тесты

- Pytest чистой логики по **дневным** окнам SC-EXP-01…06 — уже есть; после временного переключения на минуты тесты должны снова зеленеть на **дневной** семантике в финале (или параметризовать unit окна, но прод-канон = дни).

### D8 — Временный тест-харнесс (меню + минуты)

Решения из explore (закрыты разработчиком):

1. Только для теста; постоянные тарифы — later.
2. Минутные remind-окна — временно, чтобы проверить.
3. **Две** кнопки (не три).

| Элемент | Решение |
|---------|---------|
| Вход | Раздел «Подписка» (`menu:subscription`): текст + две inline-кнопки + «Назад» |
| Кнопка 1 | «На 1 минуту» → `callback` namespaced (напр. `sub:test:1m`): `is_active=True`, `subscription_end = utcnow()+1 minute` |
| Кнопка 2 | «На 5 минут» → `sub:test:5m`: `is_active=True`, `subscription_end = utcnow()+5 minutes` |
| Cron на время проверки | `minute="*"` |
| Окна remind на время проверки | те же N∈{3,2,1}, но шаг **минута**: `[now+N minutes, now+N+1 minute)` |
| Тексты remind | временно допустимо «через N минут» / упрощённо то же семейство copy; финал снова «дни» |
| Ожидание 1м | после истечения — сообщение об истечении (SC-EXP-05 по смыслу) |
| Ожидание 5м | когда до конца ~1 минута — remind за 1 единицу окна (SC-EXP-03 в минутном режиме); затем expire |
| После проверки | вернуть дневные окна + `hour=10, minute=0`; убрать `minute="*"` из итогового кода |
| Кнопки после проверки | оставить до отдельного product-change (не блокируют archive scheduler), либо убрать по желанию apply — default **оставить** с пометкой «тест» в copy |

**Альтернативы отклонены:** one-shot job на user (второй механизм); постоянные минутные окна в проде.

Константа/флаг `EXPIRY_WINDOW_UNIT=day|minute` допустима, если упрощает revert; иначе явный временный edit + revert как в D6.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Забыли вернуть дни / daily cron → спам и неверные remind | Явные tasks revert; verify `rg` перед archive |
| Кнопка 1м сразу попадает в окно remind_1 | Принять возможный remind перед expire на первом тике; либо выставлять end чуть меньше 1м — предпочтительно документировать: 1м-сценарий целится в **expire** |
| Тест-кнопки на проде доступны всем | Временно ок; later — убрать / заменить оплатой |
| Pytest дневных окон краснеет при minute-unit в коде | Финальный код и тесты — дневные; минутный режим только на окно проверки |

## Migration Plan

1. Ядро scheduler + wiring + tzdata + дневные тесты (уже).
2. Включить тест-харнесс D8 → deploy → проверить 1м и 5м в Telegram.
3. Revert окон и cron к прод-значениям → deploy.
4. Sync/archive OpenSpec; product UX подписки — отдельный change.

## Technical prerequisites (from explore)

D1–D5, D8 закрыты решениями выше. Открытых блокеров нет.

## Open Questions

Нет (не блокеры): точный wording кнопок; оставлять ли кнопки после revert cron.

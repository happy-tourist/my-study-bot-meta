## Context

Пакет: **bot** (`../my-study-bot`). Мотивация — `proposal.md`. Поведение — `specs/subscription/expiry/spec.md`.

Сейчас: `User.subscription_end` / `is_active` в `app/database.py`; часы модели — naive UTC (`datetime.utcnow`); long-polling в `main.py` с hooks `startup` / `shutdown`; планировщика и gate нет. Чеклист apply — `tasks.md`.

## Goals / Non-Goals

**Goals:**

- В процессе бота: `AsyncIOScheduler` + модуль проверки подписок.
- Напоминания за 3/2/1 UTC-день и деактивация истекших с уведомлением.
- Старт/стоп планировщика в lifecycle бота.
- Локальная ускоренная проверка (cron каждую минуту + тестовый user с `subscription_end ≈ now+2d`), затем возврат к прод-расписанию до завершения change.
- Ошибки Telegram на одного user не рвут весь прогон.

**Non-Goals:**

- Gate в handlers, оплата, multi-instance lock, таблица dedup напоминаний.
- Менять семантику `/start` / topic menu.

## Decisions

### D1 — Часы и расписание (hybrid)

- **Сравнение окон дней и `subscription_end`:** naive **UTC** (как в модели и `bot-work-with-auth`).
- **Прод-cron:** `AsyncIOScheduler(timezone="Europe/Moscow")`, job `cron hour=10, minute=0` (утро по Москве).
- **Альтернативы:** всё в UTC (проще Docker, хуже «10:00 МСК»); всё в Moscow (нужно хранить/конвертировать даты) — отклонены в пользу hybrid.
- **Docker:** установить `tzdata` (apt или эквивалент), иначе `Europe/Moscow` на slim-образе может падать.

### D2 — Capability

- `subscription/expiry` (не смешивать с будущим `subscription/gate`).

### D3 — Deactivate без gate

- В этом change пишем `is_active=False` + сообщение; отказ в handlers — отдельный change. Иначе «отключение» только в данных + UX-уведомление.

### D4 — Один процесс, без dedup

- Один контейнер/процесс как сейчас. Нет колонки «уже напомнили». Прод: один fire в сутки. Ускоренный cron **обязан** быть возвращён, иначе спам.

### D5 — Стек и файлы

| Что | Решение |
|-----|---------|
| Библиотека | `apscheduler==3.10.4`, `AsyncIOScheduler` |
| Модуль | новый `app/scheduler.py`: `scheduler`, `check_subscriptions(bot)`, `start_scheduler(bot)`, `stop_scheduler()` |
| Сессия БД | `async with async_session()` внутри job (не middleware update) |
| Модель | только существующий `User`; без параллельного store |
| Wiring | `main.py`: после создания `bot` — в `startup` вызвать `start_scheduler(bot)`; в `shutdown` — `stop_scheduler()` |
| Выборка remind | `is_active == True`, `subscription_end IS NOT NULL`, end ∈ `[now+N days, now+N+1 day)` для `N in (3,2,1)` |
| Выборка expire | `is_active == True`, `subscription_end IS NOT NULL`, `subscription_end < now` → `is_active=False`, commit |
| Send | per-user `try/except` вокруг `bot.send_message` |
| Тексты | русские, смысл как в исходном черновике (дни / истекла) |

**Не** класть бизнес-логику expiry только в `main.py`.

### D6 — Локальная ускоренная проверка (обязательный шаг apply)

1. Временно заменить прод-trigger на `cron minute="*"` (каждую минуту); **не** коммитить/не оставлять в финале.
2. Создать/обновить тестового пользователя (Telegram id того, кто запускает бота): `is_active=True`, `subscription_end = datetime.utcnow() + timedelta(days=2)` (попадает в окно N=2).
3. Запустить бота, дождаться одного срабатывания, убедиться, что пришло напоминание «через 2 дн.».
4. Сразу вернуть `hour=10, minute=0` (Europe/Moscow); убрать временный cron из итогового кода.
5. Тестовую строку в БД можно оставить или сбросить — на усмотрение apply; прод-расписание в коде MUST быть daily.

Опционально: флаг/константа `SCHEDULER_DEBUG_EVERY_MINUTE` только на время шага 1–3, по умолчанию `False` в смерженном коде — предпочтительно явный временный edit + revert, без постоянного debug-флага в проде.

### D7 — Тесты

- Покрыть чистую логику отбора user id / действий (без реального APScheduler и без Telegram), когда появится `tests/`; SC-ID из spec.
- Пока suite нет — Traceability `pending`; ускоренная проверка D6 закрывает риск «job не шлёт».

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| UTC day buckets ≠ «календарный день МСК» у границы суток | Зафиксировано в D1/spec; приемлемо до product TZ |
| `minute="*"` забыли вернуть → спам | Явный task revert; verify перед archive |
| Нет gate → `is_active=False` слабо влияет на UX | Out of scope; сообщение об истечении всё же уходит |
| Два процесса бота → дубли сообщений | Один compose-сервис; не масштабировать replicas |
| Telegram flood при большой базе | Сейчас мало users; при росте — throttle later |
| `tzdata` отсутствует в image | Добавить в Dockerfile |

## Migration Plan

1. Зависимость + `app/scheduler.py` + hooks в `main.py` + `tzdata` в Docker.
2. Локально: D6 (every minute → seed → observe → revert).
3. Deploy обычным pipeline; первый прод-fire — в 10:00 Europe/Moscow после выкладки.
4. Rollback: убрать start scheduler / revert commit; данные `is_active` при необходимости править вручную.

## Technical prerequisites (from explore)

Закрыты решениями D1–D6 выше. Открытых блокеров для propose/apply нет.

## Open Questions

Нет (не блокеры): точные emoji в copy; нужен ли env для часа cron позже.

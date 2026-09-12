## 1. Зависимости и модуль планировщика

- [x] 1.1 Прочитать `design.md` (D1–D6), `specs/subscription/expiry/spec.md`, skills `bot-work-with-structure`, `bot-work-with-auth`, `work-with-database`, `work-with-config` — сверить точки врезки
- [x] 1.2 В sibling `../my-study-bot`: добавить `apscheduler==3.10.4` в `requirements.txt`, выполнить `pip install -r requirements.txt` — импорт `apscheduler` успешен
- [x] 1.3 Создать `app/scheduler.py` по `design.md` (AsyncIOScheduler Europe/Moscow, `check_subscriptions`, start/stop, выборки 3/2/1 и expire, per-user try/except) — `python -c "from app.scheduler import start_scheduler, stop_scheduler"` из корня bot успешен

## 2. Wiring и deploy

- [x] 2.1 Подключить start/stop планировщика в `main.py` (`startup` / `shutdown`) по `design.md` — при старте процесса job зарегистрирован, при shutdown scheduler останавливается
- [x] 2.2 Добавить `tzdata` в Docker-образ (`Dockerfile`) — сборка `docker compose build` (или `docker build`) не падает; timezone Europe/Moscow резолвится в runtime-образе

## 3. Тесты логики выборки

- [x] 3.1 Добавить pytest (+ pytest-asyncio при необходимости) и тест(ы) чистой логики отбора/действий с ID сценариев `SC-EXP-01`…`SC-EXP-06` (без реального Telegram/APScheduler) — `pytest` из корня bot зелёный; Traceability в delta-spec обновить на covered где применимо

## 4. Ускоренная проверка (design D6) и фиксация прод-cron

- [x] 4.1 Временно выставить trigger `cron` с `minute="*"` (не оставлять в финале) — в коде job срабатывает каждую минуту
- [x] 4.2 Создать/обновить тестового пользователя в SQLite: `is_active=True`, `subscription_end = utcnow()+2 days` (Telegram id аккаунта, куда должен прийти бот) — строка видна в БД
- [ ] 4.3 Запустить `python main.py`, дождаться одного срабатывания job — в Telegram приходит напоминание об истечении через 2 дня (SC-EXP-02)
- [x] 4.4 Вернуть прод-расписание `hour=10, minute=0` (Europe/Moscow), убедиться что `minute="*"` отсутствует в итоговом коде — `rg 'minute="\\*"' app/scheduler.py main.py` пусто; daily cron на месте

## 5. Закрытие

- [ ] 5.1 `openspec validate add-subscription-expiry-scheduler` — без ошибок
- [ ] 5.2 После apply: `/opsx-sync` (или sync-specs) и archive по workflow meta — delta `subscription/expiry` в main specs

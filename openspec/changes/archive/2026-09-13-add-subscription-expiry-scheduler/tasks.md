## 1. Зависимости и модуль планировщика

- [x] 1.1 Прочитать `design.md` (D1–D6), `specs/subscription/expiry/spec.md`, skills `bot-work-with-structure`, `bot-work-with-auth`, `work-with-database`, `work-with-config` — сверить точки врезки
- [x] 1.2 В sibling `../my-study-bot`: добавить `apscheduler==3.10.4` в `requirements.txt`, выполнить `pip install -r requirements.txt` — импорт `apscheduler` успешен
- [x] 1.3 Создать `app/scheduler.py` по `design.md` (AsyncIOScheduler Europe/Moscow, `check_subscriptions`, start/stop, выборки 3/2/1 и expire, per-user try/except) — `python -c "from app.scheduler import start_scheduler, stop_scheduler"` из корня bot успешен

## 2. Wiring и deploy

- [x] 2.1 Подключить start/stop планировщика в `main.py` (`startup` / `shutdown`) по `design.md` — при старте процесса job зарегистрирован, при shutdown scheduler останавливается
- [x] 2.2 Добавить `tzdata` в Docker-образ (`Dockerfile`) — сборка `docker compose build` (или `docker build`) не падает; timezone Europe/Moscow резолвится в runtime-образе

## 3. Тесты логики выборки

- [x] 3.1 Добавить pytest (+ pytest-asyncio при необходимости) и тест(ы) чистой логики отбора/действий с ID сценариев `SC-EXP-01`…`SC-EXP-06` (без реального Telegram/APScheduler) — `pytest` из корня bot зелёный; Traceability в delta-spec обновить на covered где применимо

## 4. Ускоренная проверка (design D6) — частично; живой SC-EXP-02 через SQL superseded секцией 6

- [x] 4.1 Временно выставить trigger `cron` с `minute="*"` (не оставлять в финале) — в коде job срабатывает каждую минуту
- [x] 4.2 Создать/обновить тестового пользователя в SQLite: `is_active=True`, `subscription_end = utcnow()+2 days` (Telegram id аккаунта, куда должен прийти бот) — строка видна в БД
- [x] 4.3 Запустить `python main.py`, дождаться одного срабатывания job — в Telegram приходит напоминание об истечении через 2 дня (SC-EXP-02) — **superseded** секцией 6 (проверка через кнопки + минутные окна; SQL+2d не требуется)
- [x] 4.4 Вернуть прод-расписание `hour=10, minute=0` (Europe/Moscow), убедиться что `minute="*"` отсутствует в итоговом коде — выполнено в 6.4

## 5. Закрытие

- [x] 5.1 `openspec validate add-subscription-expiry-scheduler` — без ошибок
- [x] 5.2 После apply: `/opsx-sync` (или sync-specs) — delta `subscription/expiry` в main specs (`openspec/specs/subscription/expiry/`); archive отложен на end-implement-change

## 6. Временный тест-харнесс (design D8) — historical; снят после verify

- [x] 6.1 В `app/scheduler.py` временно переключить remind-окна на минуты (`N in (3,2,1)` минут) и оставить/подтвердить cron `minute="*"` — классификация для end≈now+5m даёт remind_1 примерно за минуту до конца; copy согласован с минутами или приемлемо для теста
- [x] 6.2 В разделе «Подписка»: две inline-кнопки «На 1 минуту» / «На 5 минут» + handlers (`sub:test:1m` / `sub:test:5m`) выставляют `is_active=True` и `subscription_end = utcnow()+1m` / `+5m` — кнопки видны, после нажатия строка User обновлена
- [x] 6.3 На стенде/проде после деплоя: нажать «На 1 минуту» → дождаться сообщения об истечении; нажать «На 5 минут» → remind roughly за 1 мин до конца, затем expire — SC-EXP-T01…T03 наблюдаемы в Telegram (подтверждено пользователем)
- [x] 6.4 После успешной проверки: вернуть дневные окна 3/2/1 и прод-cron `hour=10, minute=0`; `rg 'minute="\\*"' app/scheduler.py main.py` пусто; `pytest` по SC-EXP-01…06 зелёный — финальный код без minute-cron и без minute-windows
- [x] 6.5 Post-verify cleanup: удалить временные grant-кнопки 1м/5м и handlers `sub:test:*`; раздел «Подписка» снова stub — в runtime нет shipping grant UX

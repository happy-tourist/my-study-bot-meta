## 1. Модель и сброс локальной БД

- [x] 1.1 Прочитать `design.md` (D2–D3), delta `specs/start/register/spec.md`, skills `work-with-models`, `work-with-database` — сверить поле `trial_used` и сброс SQLite
- [x] 1.2 В sibling `../my-study-bot`: добавить `User.trial_used` (Boolean, default False) в `app/database.py` по design D2 — импорт модели успешен; поле видно в классе `User`
- [x] 1.3 Удалить локальный файл SQLite по `DB_URL` (типично `data/db.sqlite3`); не трогать прод/VPS — файла нет; следующий старт бота создаёт чистую схему с `trial_used`

## 2. Триал на `/start`

- [x] 2.1 Прочитать skills `work-with-handlers`, `bot-work-with-auth`, `work-with-messages` и SC-START-01…03 — сверить тексты триала
- [x] 2.2 В `app/handlers.py` на первом `/start`: `trial_used=True`, `is_active=True`, `subscription_end = utcnow()+3 minutes`, русский текст триала «3 дня (3 мин)» + меню; returning без нового триала — логика соответствует design D3

## 3. Тарифы и grant

- [x] 3.1 Прочитать delta `specs/subscription/tariffs/spec.md`, design D4–D5, skills `work-with-keyboards`, `work-with-handlers` — сверить каталог и callback
- [x] 3.2 В `app/keyboards.py` (и при необходимости тонкий модуль тарифов): dict `TARIFFS` 30/90/36500 мин, builders с подписями «день (N мин)» и ценой, `tariff:*` callback_data — клавиатура строится без ошибок импорта
- [x] 3.3 `menu:subscription` открывает экран тарифов + Back; handlers `tariff:*` делают grant + stacking по design D5, русское подтверждение — неизвестный tariff id не меняет User

## 4. Gate тематических разделов

- [x] 4.1 Прочитать delta `specs/subscription/gate/spec.md`, design D7, skill `bot-work-with-auth` — сверить `has_active_subscription`
- [x] 4.2 Добавить shared helper активной подписки (`is_active` + `subscription_end > now`) — unit-тест или прямой вызов helper зелёный на allow/deny кейсах
- [x] 4.3 В handlers `menu:cars` / `menu:houses`: при отсутствии активной подписки — русский отказ + ack, stub не открывать; при наличии — stub + Back; «Подписка» без gate — соответствует SC-GATE-01…04

## 5. Expiry: минутный режим

- [x] 5.1 Прочитать delta `specs/subscription/expiry/spec.md`, design D6, skill `work-with-scheduler` — сверить unit и cron
- [x] 5.2 В `app/scheduler.py`: `EXPIRY_WINDOW_UNIT = "minute"`; cron `minute="*"` вместо `hour=10, minute=0` — при старте job minutely; remind-тексты минутные

## 6. Тесты

- [x] 6.1 Прочитать skill `bot-work-with-test` и Traceability во всех delta-specs — сверить SC-ID
- [x] 6.2 Обновить `tests/test_subscription_expiry.py` под минутные окна и SC-EXP-T04 — `pytest tests/test_subscription_expiry.py` зелёный
- [x] 6.3 Добавить pytest на триал `/start` (SC-START-01…), grant тарифов (SC-TAR-02 и/или SC-TAR-05) и gate (SC-GATE-03 и allow SC-GATE-01 или SC-MENU-01) — `pytest` из корня bot зелёный; Traceability обновить на covered где применимо

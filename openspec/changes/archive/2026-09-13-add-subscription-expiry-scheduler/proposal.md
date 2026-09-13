## Why

У пользователей с платной подпиской дата окончания уже хранится в профиле, но бот не предупреждает заранее и не помечает истекшие подписки. Нужен фоновый проход: напоминания до окончания и отключение неоплативших.

## What Changes

- Фоновая проверка подписок в процессе long-polling бота.
- **Прод (итог change):** напоминания за 3/2/1 **день** (UTC) и деактивация по истечении; cron раз в сутки в 10:00 Europe/Moscow.
- **Проверка (исторически, не shipping):** временно использовались минутные окна remind, cron каждую минуту и кнопки выдачи 1м/5м в разделе «Подписка»; после живой проверки всё это **снято** — в runtime снова stub «Подписка», прод-окна и daily cron.
- **Capability ID:** `subscription/expiry` (gate и оплата не входят).

## Scope

- Пакет: **bot** (`my-study-bot`).
- Capability: **`subscription/expiry`**.
- Поведение (финал): напоминания за 3/2/1 день; деактивация при `subscription_end` в прошлом; без даты окончания — не трогать; daily cron 10:00 Europe/Moscow.
- Временный тест-харнесс (minute-cron, минутные окна, кнопки 1м/5м) — только верификация в рамках change; **не** требование к поставке; кнопки удалены post-verify.
- Один экземпляр процесса бота.

## Out of scope

- Оплата, продление, платёжный провайдер и CTA «оплатить».
- Handler/middleware **gate** платных функций (`subscription/gate`).
- Продуктовый UX тарифов / постоянные кнопки выдачи подписки (отдельный change).
- Multi-replica / распределённый lock / таблица «уже напомнили».
- Изменение семантики `/start` и stubs «Машины» / «Дома» / «Подписка» (stub остаётся stub).

## Capabilities

### New Capabilities

- `subscription/expiry`: фоновые напоминания об истечении подписки и деактивация по истечении срока (прод: дневные окна + daily Moscow cron).

### Modified Capabilities

- (нет)

## Impact

- Runtime: sibling `../my-study-bot` — планировщик, поля `User.subscription_end` / `is_active`, исходящие сообщения; раздел «Подписка» снова stub (без тест-кнопок).
- Зависимость: `apscheduler`.
- Deploy: `tzdata` в образе; финальный cron — daily 10:00 Europe/Moscow (не `minute="*"`).
- Main specs после sync: `openspec/specs/subscription/expiry/`.

## References

- Sibling: `../my-study-bot/AGENTS.md`.
- Meta: `docs/projects-map.md`, `openspec/config.yaml`.
- Explore / verify: минутный харнесс и кнопки 1м/5м — historical only; после проверки сняты.

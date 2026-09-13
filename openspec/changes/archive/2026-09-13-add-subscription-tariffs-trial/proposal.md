## Why

Пользователям нужны несколько тарифов подписки и разовый бесплатный триал, а раздел «Подписка» пока заглушка. Разделы «Машины» и «Дома» должны открываться только при активной подписке. Платёжный провайдер ещё не подключаем: для проверки UX и цепочки remind/expire длительности считаем в минутах (как «сжатые дни»), в интерфейсе оставляем дневные названия с минутами в скобках; после ЮKassa вернём календарные дни.

## What Changes

- Разовый пробный период при первой регистрации на `/start` (3 минуты; в тексте — «3 дня» с пометкой минут); для legacy-пользователей с `trial_used=False` — кнопка «Получить пробный период» в «Подписка».
- Каталог тарифов «1 месяц» / «3 месяца» / «Навсегда» в разделе «Подписка»; по нажатию — мгновенная выдача срока, как будто оплата прошла (без ЮKassa).
- Gate: «Машины» и «Дома» только при активной подписке; отказ по-русски + CTA на «Подписка»; «Подписка» без gate.
- Временный минутный режим expiry: окна remind 3/2/1 и cron каждую минуту (до возврата дневного прод-режима с ЮKassa).
- Схема: колонка `trial_used` + idempotent SQLite ALTER при старте; для локальной проверки допустим сброс файла БД.
- Capability ID: `start/register`, `subscription/tariffs`, `subscription/gate`, `subscription/expiry`.

## Scope

- Пакет: **bot** (`my-study-bot`).
- Capabilities: **`start/register`**, **`subscription/tariffs`** (новый), **`subscription/gate`** (новый), **`subscription/expiry`**.
- Триал: первый `/start` или разовый claim в «Подписка» при `trial_used=False`; повторный `/start` / повторный claim не выдают второй триал.
- Тарифы: длительности 30 / 90 / 36500 минут; подписи кнопок с дневными названиями и минутами в скобках; цены можно показывать, списания нет.
- Grant по клику: активная подписка с новым `subscription_end` (stacking — в design).
- Gate: доступ к тематическим разделам по `is_active` + действующий `subscription_end`; раздел «Подписка» всегда доступен.
- Expiry на время change: минутные окна и minutely cron; тексты remind в минутах.
- Русский UX.

## Out of scope

- ЮKassa / любой платёжный провайдер, инвойсы, webhooks.
- Возврат дневных окон и daily cron 10:00 MSK (отдельный change после ЮKassa).
- Содержимое stubs «Машины» / «Дома» (остаются заглушки, но за gate).
- Multi-replica / lock / дедуп напоминаний.

## Capabilities

### New Capabilities

- `subscription/tariffs`: каталог тарифов в разделе «Подписка» и мгновенная выдача срока по выбору тарифа (без реальной оплаты).
- `subscription/gate`: доступ к тематическим разделам только при активной подписке; отказ с русским сообщением.

### Modified Capabilities

- `start/register`: первый `/start` активирует разовый триал; «Подписка» ведёт к тарифам; «Машины»/«Дома» открываются только при прохождении gate.
- `subscription/expiry`: временный минутный режим окон remind 3/2/1 и проверки каждую минуту вместо дневного прод-канона (до ЮKassa).

## Impact

- Runtime: sibling `../my-study-bot` — модель (`trial_used` + ensure-columns), `/start`, меню/тарифы/claim-trial, `app/auth.py`, планировщик expiry.
- Локальная SQLite: ensure `trial_used`; опциональный сброс файла БД при apply.
- Зависимости: без новых пакетов; ЮKassa не добавляем.
- Main specs после sync: `start/register`, `subscription/tariffs`, `subscription/gate`, `subscription/expiry`.

## References

- Sibling: `../my-study-bot/AGENTS.md`; skill `bot-work-with-auth` (правило gate).
- Meta: `docs/projects-map.md`, `openspec/config.yaml`.
- Main specs: `openspec/specs/start/register/`, `openspec/specs/subscription/expiry/`.
- Explore: D1=grant без оплаты, D2=minute harness, D3=сброс БД, D4=forever = 36500 минут; gate добавлен в scope по запросу.

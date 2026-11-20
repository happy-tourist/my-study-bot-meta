## Context

Пакет: **bot** (`../my-study-bot`). Мотивация — `proposal.md`. Поведение — delta-specs `admin/panel`, `start/register`, `subscription/gate`. Чеклист — `tasks.md`.

Сейчас: один `app/handlers.py` router; `User` без ролей/бана; env только `TG_TOKEN` / `DB_URL`; `main_menu_kb()` одинаков для всех; gate в `app/auth.py` только по подписке; сессия через `DbSessionMiddleware`.

## Goals / Non-Goals

**Goals:**

- Bootstrap `ADMIN_IDS` + колонки `is_admin` / `is_banned` на `User` (+ ensure DDL).
- Фильтр/хелпер «является админом»; отдельный admin-router; кнопка «Админ» в меню.
- Статистика, пагинация, поиск (FSM), карточка с admin/ban/unban/тарифы/revoke.
- Бан блокирует learner UX («доступ закрыт»); защиты ролей из specs.

**Non-Goals:**

- Веб-UI, audit log, сброс `trial_used`, свободный ввод длительности, платежи.
- Полный split всего `handlers.py` на пакет (только добавить admin-модуль рядом).

## Decisions

### D1 — Файловая раскладка

- Новый `app/handlers_admin.py` (или `app/admin/` если разрастётся) с `Router`, filter на админа для message/callback admin-префикса.
- `main.py`: `dp.include_router(admin_router)` **после** learner router (или до — не важно, если фильтры не пересекаются).
- Не открывать свой `async_session()` в handlers — только injected `session`.
- UI-кнопки админки — в `app/keyboards.py` (префиксы `admin:`); поиск — FSM в `app/states.py`.
- Хелперы: `app/auth.py` расширить (`is_admin_user`, `is_banned_user`) или тонкий `app/admin_access.py` — предпочтительно рядом с auth, без второго user store.

**Альтернатива:** монолит в `handlers.py` — отвергнуто (объём UX). Пакет `app/handlers/` с `__init__` — отложено, YAGNI.

### D2 — Модель и env

- `User.is_admin: bool` default False; `User.is_banned: bool` default False.
- `_SQLITE_USER_COLUMN_DDL` + ensure при `init_db` (как `trial_used`).
- Env: `ADMIN_IDS=463353358` (CSV числовых id). Парсинг при старте/чтении хелпера; документировать в sibling `AGENTS.md`.
- Админ = `id in ADMIN_IDS` **или** `user.is_admin`. Bootstrap id всегда админ, даже если флаг в БД сбросили (снять bootstrap через UI нельзя).

### D3 — Ban vs `is_active`

- Бан только через `is_banned`. Scheduler expiry продолжает трогать `is_active` / reminders и **не** снимает бан.
- Learner handlers (`/start`, menu, tariffs, claim): early-return «доступ закрыт» если banned (кроме того, что админ забанен не должен случаться — админов не баним).
- Gate: banned → access closed до проверки подписки.

### D4 — Admin UX flow

```
/admin | menu:admin
  -> home (кнопки: статистика, пользователи, поиск, забаненные)
  -> stats (агрегаты SQL count)
  -> users list / banned list (page size ~8, prev/next)
  -> search FSM: текст id или @username
  -> card(user_id): статус + действия
```

- Продление: те же `TARIFFS` / минуты, stacking как learner grant, но target = карточка.
- Revoke: `subscription_end=None`, `is_active=False` (не трогать `trial_used`, `is_admin`, `is_banned`).
- Защиты в одном месте (service-хелпер): cannot ban if target admin; cannot promote if banned; cannot demote self / bootstrap.

### D5 — Меню

- `main_menu_kb(*, show_admin: bool = False)` — четвёртая кнопка «Админ».
- Все места, где сейчас `main_menu_kb()` (`/start`, back-to-menu): передавать `show_admin=is_admin(...)`.

### D6 — Prerequisite из explore (закрыты)

| ID | Решение |
|----|---------|
| Bootstrap id | `463353358` |
| Capability | `admin/panel` |
| Роли | env + `is_admin` |
| Бан | `is_banned`, полный блок |
| Продление | кнопки тарифов + revoke |
| Поиск | id / `@username` + пагинация |

## Risks / Trade-offs

- [Username не уникален в Telegram на 100%] → Mitigation: поиск по `@username` exact match в нашей таблице; при нескольких — показать список; предпочтителен numeric id.
- [Lockout если снять всех админов] → Mitigation: bootstrap из env нельзя снять через UI; нельзя снять себя.
- [Админский callback без фильтра] → Mitigation: router-level filter + повторная проверка прав на мутирующих действиях.
- [Объём callback_data] → Mitigation: короткие префиксы `admin:u:{id}`, `admin:ban:{id}`; id Telegram влезает в лимит.

## Migration Plan

1. Добавить колонки через ensure DDL (volume SQLite на VPS не вайпать).
2. Прописать `ADMIN_IDS=463353358` в локальный и серверный `.env`.
3. Задеплоить образ; рестарт compose.
4. Rollback: откат кода; колонки безопасно оставить; убрать `ADMIN_IDS` вернёт только DB-админов (bootstrap пропадёт — держать env).

## Open Questions

Нет блокирующих. Размер страницы списка (8) и точная формулировка копирайта — на apply по стилю существующих русских строк.

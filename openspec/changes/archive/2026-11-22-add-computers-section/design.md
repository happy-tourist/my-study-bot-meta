## Context

Пакет **bot** (`../my-study-bot`). Сейчас тематические разделы — Машины и Дома: константы/`main_menu_kb` в `app/keyboards.py`, handlers в `app/handlers.py` с `is_banned_user` → `has_active_subscription` → stub + `back_to_menu_kb`, тесты в `tests/test_handlers_subscription.py`. См. proposal.md — Why.

Технических prerequisite / блокеров из explore нет: паттерн уже в проекте, новых deps/env нет.

## Goals / Non-Goals

**Goals:**

- Третья тема «Компьютеры» в том же UX-слое, что Машины/Дома.
- Тот же gate (подписка / бан) через существующие helpers в `app/auth.py`.
- Покрытие новых SC pytest-ами рядом с текущими menu/gate тестами.

**Non-Goals:**

- FSM / уроки / контент раздела.
- Рефакторинг общего «topic router» (допустимо тонкое копирование handler Cars/Houses).
- Правки scheduler, тарифов, admin, Docker.

## Decisions

1. **Зеркало Cars/Houses, не новая capability `study/*`.**  
   Rationale: stub без учебного контента — расширение меню + gate.  
   Alternative: завести `study/computers` — отвергнуто (нет lesson-домена в runtime).

2. **UI: `MENU_COMPUTERS = "menu:computers"`, кнопка «💻 Компьютеры» в `main_menu_kb`.**  
   Порядок кнопок тем: Машины → Дома → Компьютеры → Подписка (далее Админ при необходимости); `adjust(1)` как сейчас.  
   Rationale: единый префикс `menu:` и русский label из explore; порядок как в delta `start/register`.  
   Alternative: отдельное подменю — out of scope.

3. **Handler в `app/handlers.py` по образцу `menu_cars` / `menu_houses`.**  
   Rationale: thin handler + injected `AsyncSession`; UI в keyboards; gate в `auth.has_active_subscription` / `is_banned_user`.  
   Stub-текст: `Раздел «Компьютеры»\nЗдесь будет контент.` + `back_to_menu_kb()`.

4. **Тесты: расширить `tests/test_handlers_subscription.py`.**  
   Импорт `menu_computers` рядом с `menu_cars`/`menu_houses`; новые сценарии и Traceability — в delta-specs `start/register` и `subscription/gate` (не дублировать таблицу SC→тест здесь).  
   Меню-лейблы: в текущем suite явные «Машины»/«Дома» уже в тестах admin-start и Back — добавить «Компьютеры»; first-visit сейчас только `reply_markup is not None` — добавить ассерты лейблов тем включая Computers.

5. **Точки изменений (bot):**  
   - `app/keyboards.py` — константа + кнопка  
   - `app/handlers.py` — callback `menu_computers`  
   - `tests/test_handlers_subscription.py` — сценарии  
   - опционально краткая строка в sibling `AGENTS.md` (список тем; task 4.1)  
   Чеклист — `tasks.md`.

## Risks / Trade-offs

- [Дублирование handler] → Mitigation: принять как у Cars/Houses; общий helper — отдельный refactor вне scope.
- [Меню теснее на узких клиентах] → Mitigation: одна кнопка; раскладку не усложнять.
- [Пропуск обновления старых ассертов клавиатуры] → Mitigation: явные tasks на START/MENU тесты.

## Migration Plan

- Деплой обычный (образ bot); схема БД не меняется.
- Rollback: убрать кнопку/handler/тесты; данные пользователей не затрагиваются.

## Open Questions

Нет.

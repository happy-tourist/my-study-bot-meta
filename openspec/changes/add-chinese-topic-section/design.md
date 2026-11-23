## Context

Пакет **bot** (`../my-study-bot`). Тематические разделы — Машины, Дома, Компьютеры: константы/`main_menu_kb` в `app/keyboards.py`, общий `_send_gated_topic` + inline/slash в `app/handlers.py`, Menu Button в `app/bot_commands.py`, тесты в `tests/test_handlers_subscription.py` и `tests/test_bot_commands.py`. См. proposal.md — Why.

Технических prerequisite / блокеров из explore нет: паттерн уже в проекте, новых deps/env нет. Контент курсов — out of scope.

## Goals / Non-Goals

**Goals:**

- Четвёртая тема «Курсы китайского» в том же UX-слое, что Машины/Дома/Компьютеры (inline + slash + Menu Button).
- Тот же gate (подписка / бан) через существующие helpers в `app/auth.py` и `_send_gated_topic`.
- Покрытие новых SC pytest-ами рядом с текущими menu/gate/command тестами.

**Non-Goals:**

- FSM / уроки / контент раздела (`study/lesson`).
- Рефакторинг общего topic registry (допустимо тонкое копирование паттерна Computers).
- Правки scheduler, тарифов, admin, Docker.

## Decisions

1. **Зеркало Computers, не новая capability `study/*`.**  
   Rationale: stub без учебного контента — расширение меню + gate + Menu Button.  
   Alternative: завести `study/chinese` — отвергнуто (нет lesson-домена в runtime; out of scope proposal).

2. **UI: `MENU_CHINESE = "menu:chinese"`, кнопка «🇨🇳 Курсы китайского» в `main_menu_kb`.**  
   Порядок кнопок тем: Машины → Дома → Компьютеры → Курсы китайского → Подписка (далее Админ при необходимости); `adjust(1)` как сейчас.  
   Rationale: единый префикс `menu:` и русский label из explore; порядок как в proposal.  
   Alternative: отдельное подменю курсов — out of scope.

3. **Handlers: thin wrappers как Computers — inline `_open_gated_topic_stub(..., section="Курсы китайского")`, slash `Command("chinese")` → `_cmd_gated_topic` с тем же `section`.**  
   Rationale: общий `_send_gated_topic` уже даёт ban → gate → stub (`f"Раздел «{section}»\nЗдесь будет контент."`); не дублировать ветки и не хардкодить отдельную строку stub в handler.  
   **Canon `section=`:** ровно `Курсы китайского` **без** emoji (emoji только на кнопке/`BotCommand.description`). Иначе exact-text SC-MENU-07 / SC-GATE-09 / SC-CMD-10 разъедутся с `Раздел «🇨🇳 …»`.  
   Stub-текст: `Раздел «Курсы китайского»\nЗдесь будет контент.` + `back_to_menu_kb()`.  
   Slash ban / refuse для Chinese **не** получают отдельные SC: тот же `_cmd_gated_topic`, что у Cars/Houses/Computers; репрезентативные slash — SC-CMD-03/04; для Chinese — SC-CMD-10 (allow) + inline SC-GATE-10/11.

4. **Menu Button: добавить `BotCommand(command="chinese", description="🇨🇳 Курсы китайского")` в `learner_bot_commands` сразу после `computers`.**  
   Rationale: SC-CMD-01 / параллельный вход как у остальных тем; `/help` берёт список из тех же builders.  
   Sync на `/start` менять не нужно — уже вызывает `learner_bot_commands` / `admin_bot_commands` (SC-CMD-09 / help подхватят `chinese` после обновления builder и тестовых констант).

5. **Тесты:**  
   - `tests/test_handlers_subscription.py` — `_assert_main_topic_labels` (+ substring «Курсы китайского»; emoji на кнопке не мешает); меню/gate SC-START-01/02/05, SC-MENU-04/07, SC-GATE-09/10/11 (зеркало Computers: `menu_chinese` + `MENU_CHINESE`).  
   - `tests/test_bot_commands.py` — `LEARNER_COMMANDS` / `SECTION_DESCRIPTIONS` (+ `chinese` сразу после `computers`, description «🇨🇳 Курсы китайского»); SC-CMD-01, SC-CMD-10 (`cmd_chinese` active stub, exact text); SC-CMD-09 / SC-HELP-01 / admin-prefix — через те же константы (startup `main.py` уже зовёт `learner_bot_commands()`).  
   Traceability SC→тест — только в delta-specs (не дублировать таблицу здесь).

6. **Точки изменений (bot):**  
   - `app/keyboards.py` — константа + кнопка  
   - `app/handlers.py` — inline + slash Chinese  
   - `app/bot_commands.py` — learner command `chinese`  
   - `tests/test_handlers_subscription.py`, `tests/test_bot_commands.py`  
   - опционально краткая строка в sibling `AGENTS.md` (список тем; task)  
   Чеклист — `tasks.md`.

## Risks / Trade-offs

- [Дублирование тонких handlers] → Mitigation: принять как у Computers; общий registry — отдельный refactor вне scope.
- [Меню длиннее на узких клиентах] → Mitigation: одна кнопка; раскладку не усложнять.
- [Пропуск обновления ассертов Menu Button / help] → Mitigation: явные tasks на SC-CMD-01 и slash.

## Migration Plan

- Деплой обычный (образ bot); схема БД не меняется.
- Rollback: убрать кнопку/handlers/команду/тесты; данные пользователей не затрагиваются.

## Open Questions

Нет.

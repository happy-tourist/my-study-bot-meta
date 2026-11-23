## 1. Keyboard — кнопка «Курсы китайского»

- [x] 1.1 Перед работой: прочитать `design.md` (решение 2), delta `specs/start/register/spec.md` (меню + SC-MENU-07), аналоги `MENU_COMPUTERS` в `app/keyboards.py`; skills `bot-work-with-keyboards`, `work-with-messages`.
- [x] 1.2 Добавить `MENU_CHINESE = "menu:chinese"` и кнопку «🇨🇳 Курсы китайского» в `main_menu_kb` сразу после Компьютеров и перед Подпиской; убедиться, что клавиатура строится с новой кнопкой (импорт/вызов builder без ошибок).

## 2. Handlers — stub, gate, slash

- [x] 2.1 Перед работой: `design.md` (решение 3), delta `specs/subscription/gate/spec.md` (SC-GATE-09/10/11) и `specs/start/register/spec.md` (SC-MENU-07 / SC-CMD-10), аналоги Computers в `app/handlers.py` (`_open_gated_topic_stub` / `_cmd_gated_topic`); skills `bot-work-with-handlers`, `bot-work-with-auth`.
- [x] 2.2 Добавить thin wrappers (как `menu_computers` / `cmd_computers`): inline на `MENU_CHINESE` и slash `Command("chinese")` с `section="Курсы китайского"` **без emoji** через `_open_gated_topic_stub` / `_cmd_gated_topic` (ban → gate → stub из `_send_gated_topic` ровно `Раздел «Курсы китайского»\nЗдесь будет контент.` + `back_to_menu_kb()`); проверить регистрацию на router.

## 3. Menu Button — команда `chinese`

- [x] 3.1 Перед работой: `design.md` (решение 4), delta `specs/start/register/spec.md` (SC-CMD-01 / SC-CMD-10), аналог `computers` в `app/bot_commands.py`; skills `work-with-handlers` / Menu Button раздел в sibling `AGENTS.md`.
- [x] 3.2 Добавить `BotCommand(command="chinese", description="🇨🇳 Курсы китайского")` в `learner_bot_commands` сразу после `computers`; admin list наследует через `*learner_bot_commands()`.

## 4. Тесты меню, gate и slash

- [x] 4.1 Перед работой: Traceability в обеих delta-specs; существующие тесты Computers в `tests/test_handlers_subscription.py` и `tests/test_bot_commands.py`; skill `bot-work-with-test`.
- [x] 4.2 Обновить/добавить тесты: в `test_handlers_subscription.py` — `_assert_main_topic_labels` (+ substring «Курсы китайского»; кнопка с 🇨🇳 проходит) для SC-START-01/02/05 и SC-MENU-04; SC-MENU-07 / SC-GATE-09/10/11 (зеркало Computers: import `menu_chinese`, callback `MENU_CHINESE`); в `test_bot_commands.py` — `LEARNER_COMMANDS` / `SECTION_DESCRIPTIONS` (+ `chinese` после `computers`, description «🇨🇳 Курсы китайского») для SC-CMD-01 (+ side-effect SC-CMD-09 / SC-HELP-01 / admin-prefix через те же константы); SC-CMD-10 (`cmd_chinese` active stub, exact text); при необходимости обновить шапки модулей тестов.
- [x] 4.3 Запустить `pytest` из корня `../my-study-bot` — suite зелёный; в delta Traceability заменить `pending` на `covered` с именами тестов.

## 5. Sibling docs (опционально)

- [x] 5.1 Перед работой: `design.md` Decision 6; текущие упоминания Cars/Houses/Computers в sibling `AGENTS.md`.
- [x] 5.2 При необходимости одна строка в sibling `AGENTS.md`: тема Chinese / `menu:chinese` / `/chinese` рядом с Computers.

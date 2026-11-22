## 1. Keyboard — кнопка «Компьютеры»

- [x] 1.1 Перед работой: прочитать `design.md` (решения 2–3), delta `specs/start/register/spec.md` (меню + SC-MENU-06), аналоги `MENU_CARS`/`MENU_HOUSES` в `app/keyboards.py`; skills `bot-work-with-keyboards`, `work-with-messages`.
- [x] 1.2 Добавить `MENU_COMPUTERS = "menu:computers"` и кнопку «💻 Компьютеры» в `main_menu_kb` сразу после Домов и перед Подпиской; убедиться, что клавиатура строится с новой кнопкой (импорт/вызов builder без ошибок).

## 2. Handler — stub и gate

- [x] 2.1 Перед работой: `design.md` (handler mirror), delta `specs/subscription/gate/spec.md` (SC-GATE-06/07/08), аналоги `menu_cars`/`menu_houses` в `app/handlers.py`; skills `bot-work-with-handlers`, `bot-work-with-auth`.
- [x] 2.2 Добавить callback-handler `menu_computers` (зеркало Cars/Houses): `is_banned_user` → `has_active_subscription` → stub-текст ровно `Раздел «Компьютеры»\nЗдесь будет контент.` + `back_to_menu_kb()`; иначе те же отказы (`_refuse_access_closed` / `_refuse_without_subscription`); проверить регистрацию на router.

## 3. Тесты меню и gate

- [x] 3.1 Перед работой: Traceability в обеих delta-specs; существующие тесты Cars/Houses в `tests/test_handlers_subscription.py`; skill `bot-work-with-test`.
- [x] 3.2 Импортировать `menu_computers`; обновить ассерты меню: SC-START-05 и SC-MENU-04 (уже проверяют «Машины»/«Дома») — добавить «Компьютеры»; SC-START-01 (сейчас только `reply_markup is not None`) — добавить проверки лейблов тем включая Computers; добавить тесты SC-MENU-06, SC-GATE-06, SC-GATE-07, SC-GATE-08 (зеркало Cars/Houses, stub-текст ровно как в design Decision 3 / SC-MENU-06); при необходимости обновить шапку модуля тестов (диапазоны SC-MENU / SC-GATE).
- [x] 3.3 Запустить `pytest` из корня `../my-study-bot` — suite зелёный; в delta Traceability заменить `pending` на `covered` с именами тестов.

## 4. Sibling docs (опционально)

- [x] 4.1 При необходимости одна строка в sibling `AGENTS.md`: тема Computers / `menu:computers` рядом с Cars/Houses (как в `design.md` Decision 5).

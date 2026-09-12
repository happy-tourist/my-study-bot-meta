## 1. Keyboards

- [x] 1.1 Прочитать `design.md` (D2–D3) и skill `work-with-keyboards`; в sibling `../my-study-bot/app/keyboards.py` добавить `main_menu_kb()` и `back_to_menu_kb()` с `callback_data` `menu:cars` / `menu:houses` / `menu:subscription` / `menu:back` и русскими лейблами из design — проверить, что модуль импортируется (`python -c "import app.keyboards as kb; kb.main_menu_kb(); kb.back_to_menu_kb()"` из корня bot)

## 2. Handlers — /start и callbacks

- [x] 2.1 Прочитать `specs/start/register/spec.md` (SC-START-*) и skill `work-with-handlers`; в `app/handlers.py` к обеим веткам `/start` добавить приглашение «Выбери раздел:» и `reply_markup=kb.main_menu_kb()`, сохранив upsert User — проверить diff: обе ветки с markup, commit/логика User без регрессии
- [x] 2.2 Прочитать design D4–D5 и skill `work-with-messages`; добавить callback-хендлеры для `menu:cars`, `menu:houses`, `menu:subscription`, `menu:back` (`edit_text` + stub/меню + `callback.answer()`, plain text) — проверить, что фильтры совпадают с builders и каждый handler вызывает `callback.answer()`
- [x] 2.3 Из корня bot выполнить `python -m compileall app` и убедиться в exit code 0

## 3. Финализация планирования (после кода — вне apply-блока кода)

- [x] 3.1 Выполнить `openspec validate add-topic-menu` в meta и устранить ошибки валидации, если появятся после правок артефактов

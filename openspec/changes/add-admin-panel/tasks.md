## 1. Модель, env и доступ

- [x] 1.1 Прочитать `design.md` (D2–D3, D6), `proposal.md` Scope, skills `work-with-models`, `work-with-database`, `work-with-config`, `bot-work-with-auth` — сверить `is_admin` / `is_banned` / `ADMIN_IDS`
- [x] 1.2 В sibling `../my-study-bot`: добавить `User.is_admin`, `User.is_banned` (Boolean, default False) и idempotent ensure-columns в `app/database.py` — импорт модели успешен; поля видны в классе `User`
- [x] 1.3 Расширить `tests/test_database_schema.py` (или аналог) на ensure новых колонок — `pytest` на schema-тест зелёный
- [x] 1.4 Хелперы парсинга `ADMIN_IDS` и `is_admin_user` / проверка бана (в `app/auth.py` или рядом по design D1) — unit-тесты: bootstrap id админ; DB-флаг админ; не-админ false; banned true/false
- [x] 1.5 Документировать `ADMIN_IDS` в sibling `AGENTS.md` (таблица Config); значение bootstrap `463353358` для локального `.env` при необходимости — таблица env содержит `ADMIN_IDS`

## 2. Меню, ban на learner-потоке, wiring

- [x] 2.1 Прочитать delta `specs/start/register/spec.md`, `specs/subscription/gate/spec.md`, skills `work-with-keyboards`, `work-with-handlers`, `work-with-messages` — сверить SC-START-05/06, SC-MENU-05, SC-GATE-05
- [x] 2.2 `main_menu_kb(show_admin=...)` + все вызовы меню; `/start` и learner handlers отказывают banned («доступ закрыт»); gate Cars/Houses/Subscription учитывает ban — логика соответствует specs
- [x] 2.3 Создать `app/handlers_admin.py` с `Router`, подключить в `main.py` через `include_router` — импорт и регистрация без ошибок

## 3. Админ-панель UX

- [x] 3.1 Прочитать delta `specs/admin/panel/spec.md`, design D4–D5, skills `work-with-handlers`, `work-with-fsm`, `work-with-keyboards` — сверить SC-ADM-01…14
- [x] 3.2 Keyboards/callbacks админки (home, stats, lists+pagination, card actions) в `app/keyboards.py` — builders импортируются; callback_data в лимите Telegram
- [x] 3.3 FSM поиска пользователя в `app/states.py` + handlers: `/admin` и `menu:admin` только админам; не-админ — «доступ закрыт» (SC-ADM-01/02); статистика (SC-ADM-03); поиск/списки (SC-ADM-04/13)
- [x] 3.4 Мутации с карточки: promote/demote с защитами (SC-ADM-05/06/10/14), ban/unban (SC-ADM-07/08/09), grant тарифом + revoke (SC-ADM-11/12); injected `session` — отказы и подтверждения по-русски

## 4. Тесты

- [x] 4.1 Прочитать skill `bot-work-with-test` и Traceability в delta-specs — сверить SC-ID
- [x] 4.2 Pytest: доступ `/admin`, статистика, promote/demote/ban защиты, grant/revoke, banned `/start` и gate (SC-ADM-* / SC-START-05/06 / SC-GATE-05 по покрытию) — `pytest` из корня bot зелёный
- [x] 4.3 Обновить Traceability в delta-specs с `pending` на `covered` где есть тесты — таблица согласована с именами тестов

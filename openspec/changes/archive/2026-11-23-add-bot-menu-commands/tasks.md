## 1. Bot commands module + startup

- [x] 1.1 Перед работой: `design.md` (Decisions 1–3, 6), delta `specs/start/register` (SC-CMD-01), proposal Scope; skills `work-with-config`, `bot-work-with-structure`; аналоги `main.py` `startup`.
- [x] 1.2 Добавить `app/bot_commands.py`: `learner_bot_commands()`, `admin_bot_commands()`, `sync_commands_for_user(bot, chat_id, *, is_admin)`; описания разделов/`admin` с эмодзи как у inline; `start`/`help` — канон из design Decision 1; без `admin` в learner.
- [x] 1.3 В `main.py` `startup`: `set_my_commands` learner на `BotCommandScopeDefault` + `MenuButtonCommands`; без `AllPrivateChats`; проверить импорт/`await` без ошибок при старте модуля (или точечный тест builders в блоке 4).

## 2. Slash handlers + sync на /start

- [x] 2.1 Перед работой: `design.md` (Decisions 4–5), delta `specs/start/register` (SC-CMD-02…09, SC-HELP-01…03); аналоги `_open_gated_topic_stub` / `menu_subscription` / «Сначала нажми /start»; skills `bot-work-with-handlers`, `bot-work-with-auth`, `work-with-messages`.
- [x] 2.2 Вынести/переиспользовать gate+stub/tariffs для `Message.answer` (не только `edit_text`); хендлеры `Command` для `cars`/`houses`/`computers`/`subscription`/`help`; **slash order:** unregistered → «Сначала нажми /start» (не копировать inline None→gate/тарифы), затем ban как inline, затем topics gate/stub или `/subscription` тарифы; `/help` — без требования `/start`, в т.ч. banned.
- [x] 2.3 В `cmd_start` после того как user известен (новый после commit / returning после load) вызвать `sync_commands_for_user` по `is_admin_user(..., telegram_id=)` **до** ответа на **всех** ветках, включая banned (SC-CMD-09); admin → chat scope с `admin`, иначе learner на chat.

## 3. Admin promote/demote sync

- [x] 3.1 Перед работой: `design.md` (Decision 3), delta `specs/admin/panel` (SC-ADM-CMD-01…03); `admin_promote`/`admin_demote` в `app/handlers_admin.py`; skills `bot-work-with-auth`, `bot-work-with-handlers`.
- [x] 3.2 После успешного promote/demote + commit вызвать sync команд для **target** user id (promote → admin list; demote → **set learner** на chat scope, не `delete_my_commands`); `bot` из inject `bot: Bot` или `callback.bot`.

## 4. Tests + Traceability

- [x] 4.1 Перед работой: Traceability в обеих delta-specs; паттерны `tests/test_handlers_subscription.py` / `tests/test_handlers_admin.py`; skill `bot-work-with-test`.
- [x] 4.2 Тесты builders команд (learner без `admin`, admin с `admin` + эмодзи-описания; `start`/`help` short RU descriptions без эмодзи разделов) — SC-CMD-01 / SC-ADM-CMD-01/02 на уровне списков; slash: SC-CMD-02…08, SC-HELP-01…03; `/start` sync (в т.ч. banned; admin = bootstrap id или admin flag) — SC-CMD-09; promote/demote — mock `callback.bot.set_my_commands` (**set** learner на demote, не `delete_my_commands`) — SC-ADM-CMD-03.
- [x] 4.3 Запустить `pytest` из корня `../my-study-bot` — suite зелёный; в delta Traceability заменить `pending` на `covered` с именами тестов.

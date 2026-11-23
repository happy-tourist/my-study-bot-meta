## Context

Пакет **bot** (`../my-study-bot`). Сейчас: `/start` + inline `main_menu_kb`; темы через callback + gate; `/admin` уже есть как `Command("admin")`, но **нет** `set_my_commands` / Menu Button sync. См. proposal.md — Why. Explore: блокеров нет (D1=A commands; Mini App out of scope).

## Goals / Non-Goals

**Goals:**

- Default learner Bot Commands + per-chat admin override с `/admin`.
- Слэш-вход в разделы / help с тем же gate/copy, что inline (новое сообщение, не `edit`).
- Обновление команд целевого чата на promote/demote и согласование на `/start`.
- Тесты: builders списков команд + slash handlers + sync на promote/demote (mock Bot API).

**Non-Goals:**

- Mini App / reply-клавиатура / смена inline UX.
- Новые pip-зависимости, схема БД, Docker.
- Живой e2e клиента Telegram Menu Button UI.

## Decisions

1. **Модуль списков команд (например `app/bot_commands.py`), не размазывать по handlers.**  
   Функции: `learner_bot_commands()` → list[`BotCommand`]; `admin_bot_commands()` = learner + `admin`.  
   Описания разделов/админки — как у inline: `🚗 Машины`, `🏠 Дома`, `💻 Компьютеры`, `💳 Подписка`, `🛠 Админ`.  
   Канон для `start` / `help` (короткие RU, без эмодзи): `start` → `Запуск и главное меню`; `help` → `Справка и список команд`.  
   Rationale: один источник правды для startup, `/start`, promote/demote, `/help` text.  
   Alternative: хардкод в `main.py` — отвергнуто (дубли с admin sync).

2. **Scopes Telegram:**  
   - Startup: `set_my_commands(learner, scope=BotCommandScopeDefault())` — **только** Default (без `BotCommandScopeAllPrivateChats` в этом change).  
   - `MenuButtonCommands` явно через `set_chat_menu_button` (default) — закрепить type=commands.  
   - Админ: `set_my_commands(admin_list, scope=BotCommandScopeChat(chat_id=user_id))` в private chat.  
   - Снятие админа: **`set_my_commands(learner, scope=BotCommandScopeChat(...))`** (не `delete_my_commands`), чтобы chat-override явно стал learner-списком.  
   Rationale: chat-scope override без публикации `/admin` всем; один предсказуемый demote-путь.  
   Alternative: только BotFather — отвергнуто (нет sync после promote).

3. **Синхронизация моментами:**  
   - `startup` в `main.py`: default learner + menu button commands.  
   - `cmd_start`: после того как user известен (новый — после commit; returning — после load) вызвать `sync_commands_for_user(bot, telegram_id, is_admin=is_admin_user(..., telegram_id=))` **до** ответа, в том числе для banned (бургер синхронизируется даже при «доступ закрыт»).  
   - `admin_promote` / `admin_demote` после успешного commit — sync для **target** `user.id`.  
   - Ban/unban **не** меняют список команд (out of scope): бургер может показывать разделы, handlers по-прежнему отказывают.  
   Rationale: Q6; иначе бургер устаревает после promote; bootstrap `ADMIN_IDS` без `User.is_admin` ловится через `is_admin_user`.  
   Alternative: только startup — отвергнуто.

4. **Slash section handlers в `app/handlers.py` рядом с callback; общая логика без дубля gate.**  
   Вынести ответ «тема stub / отказ / тарифы» в helpers, принимающие либо `Message` (`answer`), либо путь callback (`edit_text`) — или тонкие message-обёртки, зовущие тот же ban→gate→stub, что `_open_gated_topic_stub`, но с `message.answer`.  
   **Порядок для slash sections** (`cars`/`houses`/`computers`/`subscription`): (1) нет строки User → RU «Сначала нажми /start» (`message.answer`, без stub/тарифов); (2) иначе ban → «доступ закрыт»; (3) topics → subscription gate → stub; `/subscription` после ban → тарифы **без** отказа по статусу подписки.  
   Copy «Сначала нажми /start» — тот же текст, что у trial/tariff callback alerts; это **slash-first** шаг. Не считать текущий inline topic/`menu:subscription` каноном для missing User: сегодня `_open_gated_topic_stub` при `user is None` уходит в subscription-refuse, а `menu_subscription` может открыть тарифы — **менять inline в этом change не нужно**; slash обязан явно проверить unregistered (SC-CMD-06).  
   Rationale: SC-CMD-02…05/07/08 = те же ban/gate outcomes, что inline для **зарегистрированных**; SC-CMD-06 = отдельный unregistered-path для slash.  
   Alternative: полностью отдельные handlers — риск расхождения copy/gate.

5. **`/help`:** короткий RU абзац + нумерованный/маркированный список команд из того же builder (learner vs admin). Без inline-клавиатуры разделов.  
   Отвечает **любому** отправителю (в т.ч. unregistered и banned): список по `is_admin_user` (admin → с `admin`, иначе learner); не открывает stub/тарифы и не требует предварительного `/start`.  
   Rationale: вариант B из explore; справка не gated-контент.

6. **Точки изменений (bot):**  
   - `main.py` — startup: set commands + menu button  
   - новый `app/bot_commands.py` (или согласованное имя) — builders + `sync_commands_for_user`  
   - `app/handlers.py` — `/cars`…`/subscription`, `/help`; sync на `/start` (оба early-return: новый user после commit и banned returning — **до** `answer`)  
   - `app/handlers_admin.py` — sync после promote/demote (`callback.bot` или inject `bot: Bot`)  
   - `tests/` — новые/расширенные pytest: mock `Bot.set_my_commands` / `set_chat_menu_button`; для promote/demote — `callback.bot = AsyncMock` с `set_my_commands` (текущий `_make_callback` bot не ставит). Demote ассертит **set** learner-списка на chat scope, не `delete_my_commands`.  
   Чеклист — `tasks.md`. Traceability SC→тест — только в specs.

7. **Тесты Menu Button:** не гонять реальный Telegram; ассертить вызовы API и состав `BotCommand`; slash — как текущие handler-тесты с AsyncSession.

## Risks / Trade-offs

- [Клиент кэширует список команд] → Mitigation: sync на `/start` + promote/demote; при жалобе — повторный `/start`.
- [Дублирование message vs callback UI] → Mitigation: shared helpers; copy из одного места.
- [Не-админ вводит `/admin` вручную] → Mitigation: уже есть refuse; Menu Button просто не рекламирует.
- [Bootstrap `ADMIN_IDS` без строки User] → Mitigation: sync по `is_admin_user(..., telegram_id=)` на `/start`, не только по `User.is_admin`.
- [Banned видит разделы в бургере] → Accept: ban/unban не sync’ят команды; handlers отказывают; sync на `/start` всё же выравнивает admin/learner список.

## Migration Plan

- Деплой обычный; после рестарта polling — default commands применяются.
- Rollback: убрать set_my_commands/handlers; при необходимости `delete_my_commands` вручную через Bot API / BotFather.
- Схема БД не меняется.

## Open Questions

Нет (Q4–Q7 закрыты в explore).

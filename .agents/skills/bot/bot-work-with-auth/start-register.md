# Registration (`/start`) + trial + Menu Button sync

Read when changing `/start` upsert, first-visit trial, greet copy, or
per-chat Menu Button sync on register/return.

## Handler sketch

File: `app/handlers.py`.

```python
@router.message(CommandStart())
async def cmd_start(message: Message, session: AsyncSession):
    user_id = message.from_user.id
    user = await _load_user(session, user_id)

    if user is None:
        user = User(
            id=user_id,
            username=message.from_user.username,
            trial_used=True,
            is_active=True,
            subscription_end=datetime.utcnow() + timedelta(minutes=3),
        )
        session.add(user)
        await session.commit()
        # sync_commands_for_user(…) before greet — see bot_commands
        await message.answer(
            "Привет! Я тебя запомнил 👋\n\n"
            "Тебе активирован пробный период на 3 дня (3 мин).\n\n"
            "Выбери раздел:",
            reply_markup=kb.main_menu_kb(),
        )
    else:
        await message.answer(
            f"С возвращением, {message.from_user.first_name}!\n\nВыбери раздел:",
            reply_markup=kb.main_menu_kb(),
        )
```

## Rules

- Always key off `from_user.id`, never username alone.
- First visit: one-time trial (`trial_used=True`, `subscription_end=now+3 min` / «3 дня» copy).
- Return: no second row; do not re-grant trial; same topic menu (`main_menu_kb(show_admin=…)`).
- Sync Menu Button (`sync_commands_for_user`) **before** greet; banned users still sync, then «Доступ закрыт.»
- `/start` remains the registration path — no separate sign-up command for MVP.
- Russian greet/refuse strings stay consistent unless product copy changes.

## Promote / demote Menu sync

After admin promote/demote commit, call `sync_commands_for_user` for the **target**
chat (demote must **set** learner list — not only `delete_my_commands`).
Details: `app/bot_commands.py`, `app/handlers_admin.py`.

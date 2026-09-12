---
name: work-with-messages
description: >-
  Use when adding, changing, reviewing, or debugging Telegram bot replies and
  user-facing copy in my-study-bot — especially `message.answer`, `message.reply`,
  `edit_text`, captions, `callback.answer`, Russian UX strings, parse_mode
  (HTML/Markdown), or keeping secrets out of chat text. Not for Colyseus /
  WebSocket room messages (that is a different product’s skill).
---

# Work With Messages

Use this skill for **ответы бота / тексты пользователю** in `my-study-bot` (aiogram v3 handlers).

**Not** Colyseus room WS messages. Server `work-with-messages` elsewhere means `this.onMessage` / room protocol — ignore that domain here.

Skills path for now: `.agents/skills/bot/` in this repo (canonical copy may later live under `my-study-bot-meta`). Runtime paths below are relative to this bot repo root.

Coordinate with: `bot-locate-change-points` (where to edit), keyboards (`app/keyboards.py`), FSM (`app/states.py`). Handlers stay thin; copy is UX, not business logic.

## Scope Boundary

| In scope | Out of scope |
|----------|--------------|
| User-visible Telegram text: answers, edits, captions, alerts | Colyseus / WS room message contracts |
| Choosing `answer` vs `reply` vs `edit_text` / `callback.answer` | DB schema / subscription logic (except what the user sees) |
| Russian tone and consistency with existing greets | Keyboard layout builders (markup lives in `keyboards.py`; button labels are copy — keep style aligned) |
| `parse_mode` safety; never leaking secrets into chat | Windows SSL/IPv4 session hacks in `main.py` |

## Existing Copy (source of truth today)

From `app/handlers.py`:

| Situation | Text |
|-----------|------|
| First `/start` (user created) | `"Привет! Я тебя запомнил 👋\n\nВыбери раздел:"` (+ `main_menu_kb`) |
| Returning user | `f"С возвращением, {message.from_user.first_name}!\n\nВыбери раздел:"` (+ `main_menu_kb`) |
| Section stubs | `"Раздел «Машины|Дома|Подписка»\nЗдесь будет контент."` (+ back keyboard) |
| Back to menu | `"Выбери раздел:"` (+ `main_menu_kb`) |

Keep Russian user-facing strings consistent with these unless product copy is being redesigned. Plain text (no `parse_mode`) for menu flows.

## Where Copy Lives

| Kind | Prefer |
|------|--------|
| Short one-liners (greets, confirms, errors) | Inline in the handler next to `message.answer` / `callback.answer` — OK today |
| Long multi-paragraph help, lessons, terms, repeated blocks | Centralize (e.g. `app/texts.py` or constants module) when copy grows; import into handlers |
| Button labels | `app/keyboards.py` builders; same Russian tone as answers |

Do not scatter the same long paragraph across handlers — extract once and reuse.

## Sending Patterns (aiogram 3)

| Method | When |
|--------|------|
| `message.answer(...)` | Default for new bot replies (no quote of the user’s message). Prefer this for greets and most UX. |
| `message.reply(...)` | Only when quoting the user’s message is intentional (thread clarity). Avoid as the default. |
| `message.edit_text(...)` / `callback.message.edit_text(...)` | Update an existing bot message (menus, status) instead of spamming a new one. |
| `callback.answer(...)` | Always acknowledge inline-button presses. Use empty / short text for silent ack; `show_alert=True` only for important notices. Does **not** replace a chat reply when the user needs visible content — pair with `edit_text` or `answer` as needed. |
| Captions | Same rules as body text: Russian, no secrets, careful `parse_mode`. |

### Hard rules

1. **Russian UX** — match existing greets’ tone (friendly, short). Do not switch to English or redesign copy casually.
2. **No secrets in messages** — never send `TG_TOKEN`, `.env` values, DB URLs, session dumps, stack traces with credentials, or internal IDs the user should not see. Log internals server-side; user gets a safe Russian phrase.
3. **`parse_mode` caution** — default to plain text unless formatting is required. If using `HTML` or `Markdown`/`MarkdownV2`, escape user-controlled fragments (`first_name`, usernames, free text) or prefer `html.escape` / aiogram helpers. Unescaped `<`, `>`, `&`, `*`, `_` in names break messages or open injection-style garbled markup.
4. **Personalization** — interpolating `first_name` is fine (as today); still escape if `parse_mode` is set.
5. **Callbacks** — answer the callback query so the client spinner stops; keep alert text short.

## Do / Don't

| Do | Don't |
|----|--------|
| Prefer `message.answer` for new replies | Use `reply` everywhere by habit |
| Keep short greets in handlers; extract when copy gets long | Duplicate long lesson/help text in multiple handlers |
| Preserve existing Russian greets unless redesigning | Invent parallel English copy “just for now” |
| Escape dynamic text when `parse_mode` is on | Pass raw `first_name` / user input into HTML/Markdown |
| `callback.answer()` on every button handler | Leave callbacks unanswered |
| Put tokens and errors in logs only | Echo `TG_TOKEN`, DB paths, or tracebacks into chat |

## Change Checklist

1. New/changed user text is Russian and consistent with `"Привет! Я тебя запомнил 👋"` / `"С возвращением, …"` tone.
2. Short strings stay near the handler; long/repeated copy is centralized.
3. Correct API: `answer` vs `reply` vs `edit_text` vs `callback.answer`.
4. If `parse_mode` is used, dynamic fragments are escaped; otherwise prefer plain text.
5. No secrets, env values, or internal diagnostics in the message body/caption/alert.
6. Inline keyboards: labels in `keyboards.py` match the same UX voice; callback handlers acknowledge with `callback.answer`.
7. Smoke-check the flow in Telegram (or note if untestable) before claiming done.

## Related

- Where to edit: `.agents/skills/bot/bot-locate-change-points/SKILL.md`
- Handlers / existing greets: `app/handlers.py`
- Keyboards: `app/keyboards.py`
- FSM: `app/states.py`
- Package overview: `AGENTS.md` (Handlers And UX Surface)

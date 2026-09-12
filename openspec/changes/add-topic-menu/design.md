## Context

Пакет **bot** (`../my-study-bot`). Сейчас `/start` в `app/handlers.py` делает upsert `User` и шлёт приветствие без markup; `app/keyboards.py` — stub (импорты без builders). Motivation — `proposal.md`. Поведение — `specs/start/register/spec.md`. Чеклист реализации — `tasks.md`.

Prerequisite из explore (закрыты решениями ниже): темы = stubs; подписка = stub без gate; `callback_data` namespaced `menu:*`.

## Goals / Non-Goals

**Goals:**

- Factory-функции инлайн-клавиатур в `app/keyboards.py`
- Прикрепить меню к обеим веткам `/start` (новый / возврат)
- Callback-хендлеры разделов + «назад» через `edit_text` + `callback.answer()`
- Plain text (без HTML/`parse_mode`), чтобы не трогать escape `first_name`

**Non-Goals:**

- FSM (`app/states.py`), новые таблицы, middleware, deploy
- Реальный study-контент и subscription gate
- Введение pytest suite в этом change (coverage = pending в Traceability)

## Decisions

### D1. Inline под сообщением, не Reply

- **Выбор:** `InlineKeyboardMarkup` под ответом `/start`; переключение экранов через `callback.message.edit_text`.
- **Почему:** меню привязано к одному сообщению; не заменяет клавиатуру чата; совпадает с запросом продукта.
- **Альтернатива:** Reply-меню — отложено (Out of scope).

### D2. Builders в `keyboards.py`, handlers только attach

- **Файлы:** `app/keyboards.py` — `main_menu_kb()`, `back_to_menu_kb()`; `app/handlers.py` — `reply_markup=kb.…` (уже есть `import app.keyboards as kb`).
- **Почему:** канон `work-with-keyboards`; без DB в builders.

### D3. Namespaced `callback_data`

| Кнопка (RU label) | `callback_data` |
|-------------------|-----------------|
| 🚗 Машины | `menu:cars` |
| 🏠 Дома | `menu:houses` |
| 💳 Подписка | `menu:subscription` |
| ⬅️ Назад в меню | `menu:back` |

- **Почему:** skill требует namespace (`menu:`), короткие стабильные slug’и; фильтры `F.data == "…"`.
- **Альтернатива:** Habr-литералы `topic_cars` / `subscription` — отклонены.

### D4. Stub copy, plain text

Пример stub: `Раздел «Машины»` + строка «Здесь будет контент.» (аналогично для домов и подписки). Главное меню: greeting + `\n\nВыбери раздел:`. Без `<b>` и без `parse_mode`.

### D5. Upsert User без изменений

Точка врезки: после существующего `session.commit` / ветки returning — только текст ответа и `reply_markup`. Session в callback-хендлерах разделов **не нужна**.

### D6. Один router

Оставить текущий `router` в `app/handlers.py`; отдельный router для меню не вводить.

## Точки изменений

| Файл | Что сделать |
|------|-------------|
| `app/keyboards.py` | Добавить builders + при необходимости константы callback |
| `app/handlers.py` | Меню на `/start`; 4 callback handler’а |
| `main.py` / DB / middleware | без изменений |

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| `edit_text` недоступен (старое/нетекстовое сообщение) | В happy-path `/start` ок; при ошибке — лог/краткий fallback later, не в этом change |
| Stub-темы зафиксируют ожидания пользователей | Proposal/Out of scope: заменить в отдельном study-change |
| Нет автотестов | Traceability = pending; smoke вручную / `compileall` в tasks |

## Migration Plan

- Деплой обычный (код bot); схема БД не меняется; rollback = предыдущий образ без меню.

## Open Questions

Нет (D1–D3 из explore закрыты: stubs / stub subscription / `menu:*`).

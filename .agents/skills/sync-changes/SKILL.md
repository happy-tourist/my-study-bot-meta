---
name: sync-changes
description: >-
  Compares OpenSpec config and orchestration skills (align-code, check-changes,
  implement-change, end-implement-change, commit and their referenced skills)
  between this meta repo and another project path/URL the user provides; proposes
  additions for both sides. Use when the user runs sync-changes, asks to sync
  skills with another project, or compare agent workflow skills across repos.
---

# Sync Changes — сравнение orchestration-скиллов с другим проектом

Сравнить **канон orchestration** текущего `my-study-bot-meta` с **другим проектом** (путь или ссылка от пользователя) и выдать предложения: **что добавить сюда** и **что добавить туда**.

Скилл **только анализирует и предлагает** — не копирует файлы, не правит, не коммитит.

## Когда применять

- Пользователь запускает `sync-changes` / просит синхронизировать / сравнить skills с другим проектом
- Нужен двусторонний gap-анализ orchestration-слоя (OpenSpec config + workspace-скиллы пайплайна)

Не использовать для сравнения runtime-кода bot или произвольных docs вне scope ниже.

## Вход: другой проект (обязателен)

1. Путь / URL / workspace-folder из запроса пользователя.
2. Иначе из однозначного контекста диалога (одно имя/путь).
3. **Иначе — спросить один раз:** «С чем сравнивать?» (локальный путь к корню репо, sibling-путь или clone URL + куда склонировать/уже склонировано).

Не угадывать второй проект. Не начинать сравнение, пока корень **peer** не резолвится.

### Резолв peer root

| Форма входа | Действие |
|-------------|----------|
| Абсолютный / относительный путь к git-корню | Использовать как есть, если есть `.git` или явный корень meta-like (есть `openspec/` или `.agents/skills/`) |
| `file://` / путь из Cursor workspace | Нормализовать к filesystem path |
| HTTPS/SSH git URL | Спросить (или взять из запроса) локальный checkout; **не** клонировать без явного согласия. Если уже есть clone рядом — использовать его |
| Имя sibling-папки | Резолвить относительно родителя текущего meta (`../<name>`) |

Объявить:

```text
Current: <abs-path my-study-bot-meta>
Peer:    <abs-path other>
```

Если peer недоступен / не похож на meta (нет ни `openspec/config.yaml`, ни `.agents/skills/`) — сообщить и спросить уточнение; не выдумывать структуру.

## Scope сравнения (фиксированный)

Сравнивать **только** эти якоря и их **связанные** скиллы (см. ниже):

| # | Якорь (относительный путь от корня проекта) | Связанные |
|---|-----------------------------------------------|-----------|
| 1 | `openspec/config.yaml` | нет (файл как есть) |
| 2 | `.agents/skills/align-code/` | скиллы, на которые ссылается `align-code` **и** транзитивно связанные из этих ссылок **в текущем** и **в peer** (отдельно по каждому корню) |
| 3 | `.agents/skills/check-changes/` | то же правило ссылок |
| 4 | `.agents/skills/implement-change/` | прямые ссылки из SKILL (обычно apply / align / check / commit) — включить в сравнение как связанные |
| 5 | `.agents/skills/end-implement-change/` | прямые ссылки (update / sync / archive / commit) |
| 6 | `.agents/skills/commit/` | прямые ссылки, если есть |

Имена каталогов skills могут отличаться в peer (другой префикс пакета) — сопоставлять по **роли** (align-оркестратор, verify, check-changes, implement/end-implement, commit, openspec-*), не только по точному имени папки.

### Как собирать «связанные» скиллы

Для каждого якорного `SKILL.md` в **current** и отдельно в **peer**:

1. Прочитать `SKILL.md` (и соседние файлы skill-каталога, если на них есть относительные ссылки).
2. Извлечь ссылки на другие skills: markdown-ссылки на `**/SKILL.md`, пути вида `.agents/skills/...`, таблицы «Делегируемые скиллы».
3. Добавить найденные skill-каталоги в **closure** этого якоря для данного корня.
4. Для `align-code` и `check-changes` — пройти **один уровень транзитивности** от делегируемых (например `bot-align-code` → skills, которые он явно требует читать Always-include / «обязательно прочитать»). Не раздувать до всего `.agents/skills/bot/` без явной ссылки из якоря или его прямых делегатов.
5. Для `implement-change` / `end-implement-change` / `commit` — включить **прямые** делегаты из таблиц скилла; глубже — только если делегат сам в списке якорей (избежать полного дампа всех `openspec-*`).

Итог: два множества путей — `current_set` и `peer_set` (якоря + связанные), плюс `openspec/config.yaml` с обеих сторон (если файл есть).

## Workflow

```
Sync-Changes Progress:
- [ ] 1. Resolve peer (ask if missing)
- [ ] 2. Build closures (current + peer)
- [ ] 3. Pair by role / path
- [ ] 4. Diff content (presence + meaningful deltas)
- [ ] 5. Report: add to current / add to peer
```

### 1–2. Closures

Построить таблицы:

| Role / path | Current | Peer |
|-------------|---------|------|
| openspec/config.yaml | yes/no + briefly | yes/no + briefly |
| align-code (+ refs…) | … | … |
| … | … | … |

Кратко показать пользователю состав closures **до** глубокого diff (сколько файлов с каждой стороны).

### 3. Pairing

Сопоставление:

1. Точный относительный путь (`.agents/skills/align-code/SKILL.md`).
2. Иначе — по `name:` во frontmatter.
3. Иначе — по роли из description / заголовка (orchestrator align, verify package, check-changes, implement-change, …).
4. Неспаренное → кандидат на **добавление** на сторону, где отсутствует.

Проект-специфичные куски (имена пакетов `bot`, пути `../my-study-bot`, доменные Always-include) **не** предлагать копировать слепо: помечать как «адаптировать под peer/current layout», не как verbatim paste.

### 4. Diff rules

Для каждой пары файлов:

- **Есть только в current** → предложение **добавить в peer** (с адаптацией имён/путей peer).
- **Есть только в peer** → предложение **добавить в current** (с адаптацией под my-study-bot-meta / bot).
- **Есть в обоих**, но peer содержит полезный паттерн (шаг workflow, safety rule, state-файл, формат отчёта, делегат), которого нет в current → **добавить в current** (описать *что* перенести, не обязательно весь файл).
- **Есть в обоих**, но current содержит полезный паттерн, которого нет в peer → **добавить в peer**.
- Игнорировать косметику: порядок абзацев, синонимы, локальные имена продуктов без поведенческой разницы.
- Не предлагать удаление в этом скилле (только добавления / перенос идей). Для delete — `check-changes` или ручной review.
- `openspec/config.yaml`: сравнивать ключи/секции (`context`, `projectsMap`, rules, schema defaults). Предлагать добавить **недостающие ключи/секции-идеи**, не перезаписывать целиком чужой config.

### 5. Report

Язык: **русский**. Формат:

```markdown
# Sync Changes — отчёт

Current: `<path>`
Peer: `<path or url → local>`

## Состав сравнения
| Якорь | Current closure | Peer closure |
|-------|-----------------|--------------|
| openspec/config.yaml | … | … |
| align-code | N files | N files |
| check-changes | … | … |
| implement-change | … | … |
| end-implement-change | … | … |
| commit | … | … |

## Что добавить в current (my-study-bot-meta)
- **`<артефакт / идея>`** (из peer `путь`): что взять и как адаптировать
- … или «ничего»

## Что добавить в peer (`<имя peer>`)
- **`<артефакт / идея>`** (из current `путь`): что взять и как адаптировать
- … или «ничего»

## Заметки
- неспаренные роли / нужна ручная адаптация путей
- что сознательно не предлагалось (проект-специфика)
```

Каждый пункт — actionable: конкретный skill/файл/секция и суть добавления. Без длинных цитат.

Приоритет: отсутствующие якорные скиллы → отсутствующие делегаты → пробелы в `openspec/config.yaml` → улучшения внутри существующих SKILL.md.

## Примеры триггеров

- «Запусти sync-changes с `../other-meta`»
- «Сравни orchestration skills с https://github.com/org/other-meta»
- «sync-changes» (без пути) → спросить «С чем сравнивать?»

## Не делать

- Не править файлы current/peer без отдельной просьбы применить предложения.
- Не клонировать remote без согласия.
- Не сравнивать весь `.agents/skills/**` и runtime — только scope якорей + closure.
- Не предлагать удаление / rename как основной output.
- Не запускать `commit`, `implement-change`, pip/pytest как часть этого скилла.

---
name: sync-changes
description: >-
  Bidirectional gap analysis between this meta repo and another project
  path/URL: deep-compare OpenSpec config + openspec-* + orchestration skills,
  then scan remaining workspace/package skills, AGENTS indexes, and meta
  workflow patterns for transferable ideas both ways. Use when the user runs
  sync-changes, asks to sync skills with another project, or compare agent
  workflow / OpenSpec / meta practices across repos.
---

# Sync Changes — двусторонний gap-анализ meta с другим проектом

Сравнить текущий `my-study-bot-meta` с **другим проектом** (путь или ссылка от пользователя) и выдать предложения: **что добавить / перенять сюда** и **что добавить / перенять туда**.

Два слоя:

1. **Deep-pass** — фиксированные orchestration-якоря (полный content-diff).
2. **Discovery-pass** — всё остальное полезное в meta другого проекта (и наоборот): skills вне якорей, структура индексов, паттерны workflow/safety/отчётов, docs/maps/kilo — с оценкой переносимости.

Скилл **только анализирует и предлагает** — не копирует файлы, не правит, не коммитит.

## Когда применять

- Пользователь запускает `sync-changes` / просит синхронизировать / сравнить skills или agent-workflow с другим проектом
- Нужен двусторонний gap-анализ: не только «есть ли у нас те же orchestration-скиллы», но и «что ещё полезного у них / у нас»

Не использовать для сравнения **runtime-кода** bot как продукта (aiogram handlers, модели и т.п.) — только meta-слой и **переносимые идеи** из package-skills (роль/паттерн, не доменный код).

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

---

## Слой A — Deep-pass (orchestration, обязателен)

Сравнивать эти якоря и их **связанные** скиллы:

| # | Якорь (относительный путь от корня проекта) | Связанные |
|---|-----------------------------------------------|-----------|
| 1 | `openspec/config.yaml` | нет (файл как есть) |
| 2 | `.agents/skills/openspec-*/` | **все** каталоги skills с префиксом `openspec-` в current и peer; прямые ссылки — в closure |
| 3 | `.agents/skills/align-code/` | ссылки + один уровень транзитивности от делегатов |
| 4 | `.agents/skills/check-changes/` | то же |
| 5 | `.agents/skills/implement-change/` | прямые делегаты (apply / *-align-code / *-verify-code / verify-mock / check / commit…) |
| 6 | `.agents/skills/end-implement-change/` | прямые делегаты (update / sync / archive / commit) |
| 7 | `.agents/skills/commit/` | прямые ссылки, если есть |
| 8 | `.agents/skills/prepare-changes/` | если есть |
| 9 | `.agents/skills/review-change/` | brief-bot → bot-align-implementability → align-meta (peer: brief-server → brief-client → align-back → align-front → align-meta; заменяет бывшие brief-change / align-change) |
| 10 | `.agents/skills/prepare-mock/` | если есть |
| 11 | `.agents/skills/verify-mock/` | если есть |
| 12 | `.agents/skills/server/align-implementability-back/` | если есть (или workspace-root / peer эквивалент по роли; у нас — `bot/bot-align-implementability`) |
| 13 | `.agents/skills/client/align-implementability-front/` | если есть (или workspace-root / peer эквивалент по роли; у нас — тот же bot-align-implementability) |
| 14 | `.agents/skills/align-meta/` | если есть |
| 15 | `.agents/skills/sync-changes/` | сам этот скилл — сравнить scope/report с peer, если есть |

Если якорь отсутствует в одном корне — кандидат на добавление. Сопоставление: точный путь → `name:` frontmatter → **роль** (не только имя папки; `bot-align-code` ↔ `client-align-code` + `server-align-code` по роли).

### Closure deep-pass

Для каждого якорного `SKILL.md` в current и peer (если есть):

1. Прочитать `SKILL.md` (+ соседние файлы skill-каталога по относительным ссылкам).
2. Извлечь ссылки: `**/SKILL.md`, `.agents/skills/...`, таблицы делегатов.
3. Добавить в closure этого якоря.
4. Для `align-code` / `check-changes` — **один** уровень транзитивности от делегатов (Always-include / «обязательно прочитать»). Не раздувать до всего `bot/` / `client/` / `server/` без явной ссылки.
5. `openspec-*`: полный набор с обеих сторон; спарить по имени; неспаренное → добавить.
6. Делегаты, уже покрытые другим якорем — не дублировать diff, учесть в таблице closures.

Итог deep-pass: `current_deep` / `peer_deep` + content-diff по Diff rules ниже.

---

## Слой B — Discovery-pass (всё полезное сверх якорей)

Цель: найти **переносимые** артефакты и идеи, которых нет в deep-pass, в обе стороны.

### Что инвентаризировать (оба корня)

| Область | Где смотреть | Что искать |
|---------|--------------|------------|
| Workspace skills | `.agents/skills/*/SKILL.md` (корень, не package) | скиллы вне deep-set: имя, `description`, роль |
| Package skills | `.agents/skills/<pkg>/` (`bot`/`client`/`server`/…) | **роли** (align, verify, locate, structure, auth, errors, test, …), не доменный текст целиком |
| Индексы | `AGENTS.md`, `.agents/AGENTS.md` | таблицы skills, workflow-строки, discovery roots |
| OpenSpec meta | `openspec/config.yaml` (уже в A; здесь — соседние conventions), наличие `changes/` / `specs/` layout | rules.*, context hints, schema defaults |
| Docs / maps | `docs/projects-map.md`, `docs/README.md`, `*.local.yaml.example` | паттерны maps, local overrides |
| Tooling | `kilo.template.jsonc` / `kilo.jsonc`, skill path hints | nested skills.paths, lazy-loading |
| Прочее meta | корневые agent/cursor rules, если явно про workflow | только если переносимо без runtime |

### Как оценивать переносимость

Для каждого кандидата (skill / секция / паттерн), которого нет или он слабее на другой стороне:

| Вердикт | Критерий | В отчёте |
|---------|----------|----------|
| **Перенести / добавить** | Универсальный workflow, safety, формат отчёта, state-файл, оркестрация, OpenSpec rule, структура skill (topic split, progress checklist) — работает в другом стеке после замены имён пакетов/путей | actionable пункт + как адаптировать |
| **Адаптировать идею** | Паттерн полезен, но завязан на стек (aiogram vs Vue/Colyseus; mock Visual Spec) — взять *идею* (чеклист, gate, отчёт), не файл целиком | «идея: … → воплотить как … в current/peer» |
| **Не предлагать** | Чистая доменная специфика продукта, runtime API, уникальные фичи без аналога; косметика; дубликат уже покрытый deep-pass | только кратко в «Заметки», без пункта в add-list |

**Не** предлагать слепой copy всего `.agents/skills/client/` в `bot/` или наоборот. Предлагать:

- отсутствующую **роль** skill (например peer имеет `*-locate-change-points` с лучшим форматом отчёта → усилить наш locate);
- отсутствующий **workspace** skill (peer имеет уникальный оркестратор / sync / prepare — оценить для нас);
- паттерн внутри существующей роли (bloat scoring, resume state, confirm gates, dual-repo commit+push);
- индекс/docs/kilo convention, которая упрощает discovery.

### Inventory → pairing (discovery)

1. Собрать списки: все `**/SKILL.md` под `.agents/skills/` с обеих сторон (+ ключевые index/docs файлы).
2. Исключить пути, уже полностью разобранные в deep-pass (не повторять тот же diff; можно сослаться «см. deep»).
3. Спарить оставшееся по роли / `name:` / description.
4. Неспаренное + спаренное с заметным gap → прогнать через таблицу переносимости.
5. Ограничить объём: приоритет **высокоценным** предложениям (обычно ≤15 пунктов на сторону в discovery; если больше — сгруппировать и оставить top по impact). Не устраивать dump всех файлов peer.

---

## Workflow

```
Sync-Changes Progress:
- [ ] 1. Resolve peer (ask if missing)
- [ ] 2. Deep-pass: build closures + pair + content diff
- [ ] 3. Discovery-pass: inventory skills/indexes/docs + transferability
- [ ] 4. Merge proposals (dedupe deep vs discovery)
- [ ] 5. Report: add to current / add to peer
```

Кратко показать состав deep closures **до** длинного diff; discovery — краткий inventory (N skills / N package roots) до детальных пунктов.

### Diff rules (оба слоя)

- **Есть только в current** → предложение **добавить в peer** (с адаптацией).
- **Есть только в peer** → предложение **добавить в current** (с адаптацией под my-study-bot-meta / bot).
- **Есть в обоих**, полезный паттерн только в peer → **добавить в current** (что перенести, не обязательно весь файл).
- **Есть в обоих**, полезный паттерн только в current → **добавить в peer**.
- Игнорировать косметику (порядок абзацев, синонимы, имена продуктов без поведенческой разницы).
- Не предлагать удаление / rename как основной output (для delete — `check-changes`).
- `openspec/config.yaml`: недостающие ключи/секции-идеи, не overwrite чужого config.
- Package-skills: предпочитать «усилить наш skill роли X идеей Y», а не «скопировать `work-with-fsm`».

### Priority в отчёте

1. Отсутствующие якорные / openspec-* skills  
2. Пробелы в `openspec/config.yaml` / orchestration workflow  
3. Discovery: отсутствующие **роли** workspace/package skills с высокой переносимостью  
4. Улучшения внутри существующих SKILL (safety, state, report, gates)  
5. Indexes / docs / kilo conventions  

---

## Report

Язык: **русский**. Формат:

```markdown
# Sync Changes — отчёт

Current: `<path>`
Peer: `<path or url → local>`

## Deep-pass (orchestration)
| Якорь | Current | Peer |
|-------|---------|------|
| openspec/config.yaml | … | … |
| openspec-* | N: … | N: … |
| align-code / check / implement / end / commit | … | … |
| prepare-changes / review-change / mock / align-* | … | … |
| sync-changes | … | … |

## Discovery-pass (inventory)
| Область | Current | Peer |
|---------|---------|------|
| workspace skills (вне deep) | N: … | N: … |
| package roots | bot | client, server / … |
| package skill roles | align, verify, locate, … | … |
| indexes / docs / kilo | … | … |

## Что добавить в current (my-study-bot-meta)
### Orchestration / OpenSpec
- **`<артефакт / идея>`** (из peer `путь`): что взять и как адаптировать
### Прочее полезное (discovery)
- **`<артефакт / идея>`** (из peer `путь`): перенести / адаптировать идею → …
- … или «ничего» в секции

## Что добавить в peer (`<имя peer>`)
### Orchestration / OpenSpec
- …
### Прочее полезное (discovery)
- …
- … или «ничего» в секции

## Заметки
- неспаренные роли / нужна ручная адаптация
- сознательно не предлагалось (чистая доменка, runtime, косметика)
```

Каждый пункт — actionable. Без длинных цитат.

## Примеры триггеров

- «Запусти sync-changes с `../happy-tourist-meta`»
- «Сравни skills и workflow с другим meta — что у них полезного взять»
- «sync-changes» (без пути) → спросить «С чем сравнивать?»

## Не делать

- Не править файлы current/peer без отдельной просьбы применить предложения.
- Не клонировать remote без согласия.
- Не сравнивать runtime-код продукта и не предлагать перенос доменных фич «как есть».
- Не dump’ить весь `.agents/skills/**` в отчёт — только gaps с вердиктом переносимости.
- Не предлагать удаление / rename как основной output.
- Не запускать `commit`, `implement-change`, pip/pytest как часть этого скилла.

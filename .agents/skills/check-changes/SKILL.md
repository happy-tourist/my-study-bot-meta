---
name: check-changes
description: >-
  Scans unstaged (and untracked) changes in my-study-bot, then recommends
  whether to add, update, or delete skills, fix projects-map / project-map,
  docs, or AGENTS.md across the ecosystem. Use when the user runs
  check-changes, asks to review unstaged bot diffs for skill/docs/agent drift,
  or wants a «что добавить / что изменить / что удалить» list after local coding.
---

# Check Changes — навыки, maps, docs, AGENTS

Проверить **незастейдженные** (и untracked) изменения в **bot**, сопоставить их с каноном skills / projects-map / docs / `AGENTS.md`, и выдать список предложений: **что добавить**, **что изменить**, **что удалить**.

Скилл **только анализирует и предлагает** — не правит файлы, не стейджит, не коммитит (для коммита — [`commit`](../commit/SKILL.md)).

## Когда применять

- Пользователь запускает `check-changes` / просит проверить незастейдженные изменения bot на drift skills/docs/agents
- После локальной разработки, до `commit`, когда нужно понять, не устарели ли навыки и документация агентов

Не вызывать «на всякий случай» после каждой мелкой правки без запроса.

## Репозитории и пути

Разрешить пути от корня `my-study-bot-meta` через [`docs/projects-map.md`](../../../docs/projects-map.md) (+ `projects-map.local.yaml` если есть):

| Ключ | Репозиторий | Default path | Что сканировать |
|------|-------------|--------------|-----------------|
| `bot` | `my-study-bot` | `../my-study-bot` | **да** — unstaged + untracked |
| meta | `my-study-bot-meta` | `.` | **нет** как источник diff; **да** как канон для сверки |

Если путь не найден — спросить пользователя, не угадывать.

### Канон для сверки (meta + sibling)

| Область | Где смотреть |
|---------|--------------|
| Индекс skills | [`.agents/AGENTS.md`](../../AGENTS.md) |
| Bot skills | `.agents/skills/bot/*/SKILL.md` |
| Workspace skills | `.agents/skills/` (`commit`, `openspec-*`, этот скилл) |
| Projects map | `docs/projects-map.md`, `projects-map.local.yaml.example` |
| Child maps | `{bot}/project-map.md` |
| Docs | `docs/README.md` и связанные `docs/**` |
| AGENTS.md | meta корень + `.agents/AGENTS.md` + `{bot}/AGENTS.md` |

## Scope diff

Только **незастейдженное** working tree bot:

```bash
# в bot:
git status -sb
git status --porcelain
git diff                 # unstaged tracked
git ls-files --others --exclude-standard   # untracked
```

Для untracked — прочитать содержимое новых файлов (или `git diff --no-index /dev/null <file>` где доступно).

**Не включать** в основной анализ:

- `git diff --cached` (staged) — если staged есть, кратко упомянуть: «есть staged, этот скилл их не разбирает; застейджьте/расстейджьте или скажите расширить scope»
- коммиты ветки vs merge-base (это зона `*-verify-code` / `*-align-code`)

Если bot чист по unstaged/untracked — остановиться: «незастейдженных изменений нет».

## Workflow

```
Check-Changes Progress:
- [ ] 1. Resolve paths
- [ ] 2. Collect unstaged/untracked (bot)
- [ ] 3. Summarize change themes
- [ ] 4. Map themes → skills / maps / docs / AGENTS
- [ ] 5. Decide add vs change vs delete vs none
- [ ] 6. Report «Что добавить» / «Что изменить» / «Что удалить»
```

### 1. Resolve paths

Из корня meta: projects-map (+ local). Проверить, что bot — git work tree.

### 2. Collect diffs

В bot — команды из Scope. Собрать:

| Repo | Unstaged files | Untracked | Themes (1–3 bullets) |
|------|----------------|-----------|----------------------|

Показать пользователю краткую сводку **до** глубокого анализа.

### 3. Summarize change themes

По путям и содержимому diff сгруппировать темы, например:

- новый домен / слой (study flow, subscription gate, новый FSM, новая keyboard)
- смена контракта (User fields, env vars, middleware injection)
- новый каталог / тип файлов, которого нет в structure-skills
- смена путей репозиториев / layout workspace
- правки только стиля / copy / тестов без нового паттерна

Игнорировать шум: форматирование без смысловых паттернов, сгенерированные артефакты.

### 4. Map → канон

Для каждой темы проверить покрытие:

| Вопрос | Куда смотреть |
|--------|----------------|
| Есть skill с этим concern? | bot skill descriptions + `.agents/AGENTS.md` таблицы |
| Skill описывает **устаревший** API/путь/паттерн относительно diff? | соответствующий `SKILL.md` |
| Skill **потерял актуальность** (домен/путь/стек убраны diff’ом)? | skill body + diff + факт наличия кода в bot |
| Новые пути/ключи сервисов? | `docs/projects-map.md`, child `project-map.md` |
| Новые команды сборки / стек / домены пакета? | `{bot\|meta}/AGENTS.md`, `.agents/AGENTS.md` |
| Канон docs устарел или нет ссылки? | `docs/README.md`, профильные `docs/**` |

Читать только релевантные skill/AGENTS/docs (не все подряд), но **не пропускать** индекс `.agents/AGENTS.md` и корневые AGENTS затронутых пакетов. Для кандидатов на удаление — сверить skill с **текущим** деревом bot (не только с diff): пути/API из skill ещё существуют?

### 5. Decide: add / change / delete / none

Правила решения:

**Добавить новый skill**, если:

- появился устойчивый повторяемый паттерн работы агента (новый домен: study, payments, admin, …);
- ни один существующий skill не покрывает concern даже после расширения description/body;
- паттерн достаточно стабилен, чтобы не быть one-off хаком.

Имя/расположение: `bot-*` / `work-with-*` под `.agents/skills/bot/`; workspace-уровень — рядом с `commit` / `check-changes`.

**Изменить существующий skill**, если:

- diff меняет канонический API, пути файлов, сообщения, env, lifecycle, которые skill ещё нужен, но описывает устаревше;
- description skills не триггерится на новую терминологию из diff;
- в skill есть запрет/пример, противоречащий новым изменениям;
- skill частично устарел — достаточно точечной правки, **не** удаления.

**Удалить skill** (предложить удаление каталога `…/SKILL.md` + строки из индексов AGENTS), если:

- diff **убирает** домен / слой / контракт / стек, ради которого skill существовал, и в bot **не осталось** кода/пути, на который skill опирается;
- skill целиком про чужой стек/продукт (остаток копипаста) и не применим к текущему bot;
- skill полностью дублирует другой актуальный skill без уникального concern (предложить удалить **дубликат**, оставить канон);
- description/body skill триггерятся на работу, которой в проекте больше нет — агент будет следовать мёртвым правилам.

**Не предлагать удаление**, если:

- skill про будущий/плановый домен, явно ещё в AGENTS/product scope («stub», «planned»);
- устарела только часть — тогда **изменить**, не удалять;
- нет связи с текущим unstaged diff и нет явных признаков мёртвого skill (не устраивать полный purge «на всякий случай»).

При предложении удаления всегда указать: **зачем skill был**, **почему больше не актуален** (ссылка на diff/отсутствие кода), **что почистить в индексах** (`.agents/AGENTS.md`, корневой `AGENTS.md` при упоминании).

**Поправить projects-map / project-map**, если:

- новые ключи путей для OpenSpec/агентов;
- сменился layout siblings / имя репо;
- child `project-map.md` отсутствует или ключ `my-study-bot-meta` неверен;
- ключи map указывают на удалённые пути — поправить или убрать (часто рядом с delete skill).

**Поправить docs**, если:

- канон в `docs/` описывает старый контракт/архитектуру;
- в оглавлении нет нового важного документа, который стоит завести или на который сослаться;
- docs ссылаются на skill/путь, который предлагается удалить.

**Поправить AGENTS.md** (meta корень, `.agents/AGENTS.md`, bot), если:

- новый skill нужно внести в индекс;
- skill предложен к удалению — убрать из таблиц/discovery;
- сменились стек, команды, домены, ссылки на skills/docs;
- always-on инструкции расходятся с фактическим diff-паттерном.

**Ничего не предлагать**, если изменение полностью покрыто актуальными skills/docs/AGENTS и maps, и мёртвых skills по diff не видно. Явно написать: «покрытие достаточное».

Не предлагать дубликаты skills «на всякий случай». Не предлагать правки runtime-кода — только agent/docs/map артефакты. Не удалять skills самому в этом прогоне — только рекомендовать.

### 6. Report

Формат ответа пользователю (три основные секции + краткий контекст):

```markdown
## Сводка изменений
| Repo | Unstaged / untracked | Темы |
|------|----------------------|------|
| bot | … | … |

## Что добавить
- **`<артефакт>`** (`путь`): зачем (1 предложение, связь с diff)
- … или «ничего»

## Что изменить
- **`<артефакт>`** (`путь`): что именно поправить и почему
- … или «ничего»

## Что удалить
- **`<skill или артефакт>`** (`путь`): почему потерял актуальность (diff / нет кода); что убрать из AGENTS/docs
- … или «ничего»

## Вне scope / заметки
- staged ignored; опциональные follow-ups (verify-code, commit) — коротко
```

Каждый пункт — actionable: конкретный файл/skill-name и суть правки/удаления. Без длинных цитат diff.

Приоритет в списке: skills (add → change → delete) → AGENTS индексы → projects-map → docs.

## Примеры триггеров

- «Запусти check-changes»
- «Проверь незастейдженные изменения bot — нужны ли новые skills?»
- «Какие skills устарели и можно удалить?»
- «Что поправить в AGENTS/docs после моих локальных правок?»

## Не делать

- Не редактировать и не удалять skills/docs/AGENTS/maps в этом скилле без отдельной просьбы применить предложения.
- Не предлагать массовое удаление skills без связи с diff или явной мёртвости относительно текущего bot.
- Не анализировать staged / branch commits как основной scope.
- Не сканировать meta working tree как источник «изменений продукта» (кроме сверки канона).
- Не запускать pip/pytest/Docker build как часть этого скилла.
- Не путать с `bot-verify-code` / `bot-align-code` (compliance кода) и `commit` (stage/commit/push).

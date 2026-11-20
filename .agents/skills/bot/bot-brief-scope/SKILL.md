---
name: bot-brief-scope
description: >-
  Read-only bot scope brief from an OpenSpec change: contract summary
  (handlers/FSM/User/subscription/keyboards/scheduler), planned bot behavior,
  reuse vs adapt vs new, plus review cards for расхождения/пробелы/неоднозначности.
  Use when the user asks to brief bot scope, разбери постановку на бота, what
  the bot must do from the change, or to understand bot work before coding.
  Chat report only by default; does not prepare a per-file change plan
  (bot-locate-change-points) or audit code (bot-align-code).
---

# Brief Bot Scope

Read-only brief of **what the bot must do and enforce**, grounded in an OpenSpec change (+ Parent/Child), plus a short **reuse / adapt / new** readout vs `my-study-bot`.

**Home:** `.agents/skills/bot/` in **my-study-bot-meta**. Runtime evidence: projects-map key `bot`.

## Hard Boundary

- Do **not** edit, create, delete, format, or commit files — except when called from [`review-change`](../../review-change/SKILL.md) (override: after report, apply meta fixes) or the user **explicitly** asks to change named meta files / this skill.
- Deliver the **entire** result **in chat only** (unless override above).
- Do **not** run `bot-locate-change-points` or produce a «куда класть файлы» plan.
- Do **not** run `bot-align-implementability` / `bot-align-code` (implementability / code vs AC).
- **Allowed for reuse:** light read of `my-study-bot` (via projects-map) and sibling changes — as analogs, not as an implementation checklist of new paths.

**Report language: Russian.** Keep handler names, callback_data, FSM states, User fields as in the sources.

## When To Use

- Clear bot picture from a change before implementation / align
- Triggers: brief bot scope / разбери постановку на бота / что должен сделать bot / digest постановки
- After OpenSpec artifacts exist

## Relation To Other Skills

| Skill | Role relative to this |
|-------|------------------------|
| [`review-change`](../../review-change/SKILL.md) | Orchestrator: brief → implementability → align-meta |
| `bot-align-implementability` | Can we build it? — not this |
| `bot-locate-change-points` | Where to edit files — not this |
| `prepare-changes` / `openspec-propose` | Create/slice change — not this |

## Input

Resolve in this order (ask only if still nothing resolves):

1. **Explicit** OpenSpec change name and/or pasted excerpts.
2. **Current git branch (default)** — slug hint from branch in `my-study-bot-meta`, or the single active change if unambiguous; else ask.
3. Pasted excerpts when paths unavailable.

Optional filters (honor when stated):

| Filter | Effect |
|--------|--------|
| «только change» | Sibling changes out of scope for **Расхождения/Пробелы/Неоднозначности** (OK for **Переиспользование**) |
| «без sibling» | Do not read sibling OpenSpec changes for issues (reuse may still peek bot) |

## Source Priority

1. **OpenSpec change** wins (`proposal`, `design`, `specs`, `tasks`).
2. Conflicts **inside** the change → **Расхождения** cards.
3. For **Переиспользование**: sibling changes + light bot read; do not invent reuse.

Do not invent handlers, FSM states, User fields, or algorithms not in the loaded docs.

## Workflow

1. Resolve change root (`openspec status --change "<name>" --json` when useful).
2. Read bot-relevant artifacts: proposal, design, delta specs, tasks. Honor exclusions («не смотри design»).
3. Extract bot-facing scope: /start & upsert, subscription gate, trial/tariffs, FSM, keyboards, admin, scheduler expiry, study flows.
4. **Reuse scan:** docs + siblings + light bot look → already exists / adapt / new.
5. Collect **расхождения / пробелы / неоднозначности** → review cards.
6. Write the chat report. Stop (unless review-change override).

## Output Format

Stacked lists. **Do not use markdown tables** in the report body (cards OK with bullets).

```markdown
## Контекст
- change: …
- Parent/Child: … (если есть)

## Контракт / поверхности
- команды / callbacks — что bot принимает
- User / subscription / trial / admin fields — ключевые поля
- FSM states — когда нужны
- keyboards / русский UX — ключевые сообщения
- scheduler — reminders / deactivation если в постановке

## Поведение bot
- правила / исходы из постановки
- ошибки / deny / invalid input — только если описаны

## Переиспользование
- Уже есть / as-is
- Доработать
- С нуля
(пустой список — «нет»)

## Расхождения
… (карточки; опустить если пусто)

## Пробелы
… (опустить если пусто)

## Неоднозначности
… (если пусто — «не найдено»)
```

### Карточки: Расхождения / Пробелы / Неоднозначности

Формат как в `align-meta`. Якорь — **одно** место правки (`файл` + `Lnn` или `Lstart–Lend`). В `сообщение`: цитата «здесь» + (если нужно) другая цитата + одна фраза сути. **Запрещено:** «выше»/«ниже», markdown-ссылки.

| Раздел | Когда |
|--------|--------|
| **Расхождения** | Два+ места в change tree говорят разное |
| **Пробелы** | Bot нужно поведение, а в change его нет / не дописано «как» |
| **Неоднозначности** | Текст есть, но два разумных чтения без канона |

```markdown
### <короткий заголовок>
- **Что не так:** …
- **Предлагаю:** …
- **Ревью**
  - файл: `openspec/changes/<name>/…`
  - строки: Lnn
  - сообщение: <paste-ready без markdown-ссылок>
```

Не выдумывать API/поля в **Контракт** / **Поведение**.

## Не делать

- Не plan файлов, не apply runtime, не commit.
- Не путать с implementability audit.
- Не предлагать follow-up skills списком в конце (кроме краткого «далее: locate / review-change» если пользователь спросил next step).

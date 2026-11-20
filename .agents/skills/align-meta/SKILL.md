---
name: align-meta
description: >-
  Use when aligning my-study-bot-meta branch and working-tree changes (docs,
  OpenSpec, skills, AGENTS, projects-map, kilo) for quality of created/changed
  meta artifacts: contradictions inside the change, gaps vs pasted AC/OpenSpec
  intent, broken cross-refs, capability/naming, index drift, extras. Reports
  three tiers (hard / warnings / recommendations). Finding card: Что не так /
  Предлагаю / Ревью (файл + одна строка-якорь + paste-ready сообщение).
  Does not judge sibling implementability (see bot-align-implementability). Does not edit files.
---

# Align Meta

Perform a read-only alignment audit of **created / changed my-study-bot-meta artifacts** and report findings in Russian.

Output **one scope** with **three tiers** (hard → warnings → recommendations):

1. **Created / changed meta files** — contradictions, gaps, extras, broken links, naming/index defects inside the branch artifacts.

Every finding is a **review card** (**Что не так** / **Предлагаю** / **Ревью** with paste-ready `сообщение`). Keep the align-meta **tier** (hard / warning / recommendation) as a label on the card.

For realizability vs sibling code/contracts, use **`bot-align-implementability`** (can run after or separately).

## Runtime roots

- **Subject:** only `my-study-bot-meta` working tree and branch diff (this repo).
- Resolve sibling paths via [`docs/projects-map.md`](../../../docs/projects-map.md) (+ `projects-map.local.yaml` if present) **only** when needed to classify wrong mounts or projects-map keys — do **not** run a full implementability audit here.

## Required Axes (this skill)

Always complete:

1. **Requirements fit** — meta artifacts vs pasted AC / dialog intent / OpenSpec source (when a source exists).
2. **Completeness / edge cases** — missed paths, owners, states, exceptions, indexes that the change should have covered **inside meta**.
3. **Extras / consistency** — unjustified new content, contradictions inside meta, broken links, index drift, forbidden capability/naming patterns.

Do **not** score sibling implementability (`FEASIBLE` / `Код проекта`) here — that belongs to `bot-align-implementability`.

## Hard Boundary

Do not edit, create, delete, format, or commit files — except when called from [`review-change`](../review-change/SKILL.md) (override: after report, apply meta fixes) or the user **explicitly** asks to change named meta files / this skill. Do not apply fixes from standalone align mode. Deliver the complete result in chat.

## Input And Store

Accept pasted requirements and an optional OpenSpec change name. If the branch clearly targets an OpenSpec change / docs task and no requirements source is given, audit from the change artifacts; ask only when the goal of the branch cannot be inferred.

If a standalone OpenSpec store is named, run `openspec store list --json`, resolve its id, and pass `--store <id>`. Otherwise use **`my-study-bot-meta/openspec/`**.

## Requirements Sources

When available, load:

- pasted requirements / acceptance criteria as provided in the dialog;
- OpenSpec change artifacts already on the branch (`proposal`, `specs`, `design`, `tasks`, assets);
- pasted UX notes when the change is message/keyboard-driven (optional).

Treat explicit prohibitions and exceptions as atomic requirements: "не отображать", "не добавлять", "запрещено", "только для…", "кроме…".

A tester's guess/question is only a lead. Treat a comment as clarification only when it explicitly states or updates expected behavior.

## What Counts As Meta Artifacts

Classify every changed path:

| Class | Typical paths |
|-------|----------------|
| Docs | `docs/**`, root `AGENTS.md`, `.agents/AGENTS.md` |
| OpenSpec | `openspec/changes/**`, `openspec/specs/**`, `openspec/config.yaml` |
| Skills | `.agents/skills/**/SKILL.md` (+ skill support files) |
| Maps / indexes | `docs/projects-map.md`, `projects-map.local.yaml*` (local only — never require commit), `kilo.jsonc` / `kilo.template.jsonc` |
| Other meta | scripts, examples, templates under meta that instruct agents |

Ignore noise unless it changes agent behavior: lockfile churn without semantic change, pure formatting with no claim change — mention briefly, do not inflate into hard gaps.

## Load Branch Work

Inspect read-only **in my-study-bot-meta** (repo root):

```bash
git status --short
git diff
git diff --cached
git branch -vv
```

Resolve merge-base: prefer `origin/master`, else `origin/main`, `master`, `main` — first ref that exists.

When the branch has commits beyond base:

```bash
git merge-base HEAD <base-ref>
git log --oneline <merge-base>..HEAD
git diff --stat <merge-base>...HEAD
git diff <merge-base>...HEAD
```

Also collect untracked: `git ls-files --others --exclude-standard` and read those files.

Union of commits + staged + unstaged + untracked is the file set. Do not ignore unstaged work. Narrow only when the user explicitly scopes to a path or working-tree-only.

## Axis A — Requirements

Judge meta content first, then optional planning completeness.

Report hard gaps only as:

- `missing` — explicitly required artifact/statement absent from the change;
- `docs-only` — promised by complete artifacts (e.g. tasks/proposal) but not present where it should land in meta;
- `meta-only` — clear contradiction between two meta sources in the change (or meta vs explicit AC text);
- `extra` — clearly forbidden or unjustified content (wrong capability ID = ticket/change name, inventing APIs, scope outside the task).

Leave **proven meta↔sibling code contradictions** (`code-only` / runtime `gap`) to **`bot-align-implementability`**; here at most a one-line pointer if spotted while reading requirements.

Ambiguous wording, unresolved Open Questions, or unproven leans → **Warnings**, not hard omissions.

Score **Постановка: N/10** only from hard omissions (`missing` / `docs-only` / `meta-only` / `extra`), not from Warnings or Recommendations.

## Axis C — Completeness / Edge Cases

Use evidence in this order:

1. explicit requirements / OpenSpec AC;
2. strong untouched meta analogues (similar skills, specs, docs, archived changes);
3. deterministic structure rules of my-study-bot-meta (capability IDs, index tables, OpenSpec config);
4. (optional hint only) sibling contracts — if used, do not turn into full implementability scoring.

For reachable documentation scope, examine:

- empty/null/permission/reconnect branches the docs claim to cover;
- handler/FSM/subscription capability mapping when UX is mentioned;
- parent/child OpenSpec links (`## Child changes` / `## Parent change`) when either side is edited;
- skill discovery indexes (`.agents/AGENTS.md`, root `AGENTS.md`, `kilo.template.jsonc` / `kilo.jsonc` paths) when skills are added/renamed/removed;
- cross-links of the form `my-study-bot-meta/docs/...` and projects-map keys;
- relative markdown links between change artifacts (targets must resolve on disk);
- Russian UX / keyboard callback notes when the change claims message surfaces.

Report hard `defect` only when a changed meta artifact deterministically misleads an agent or developer into wrong path, wrong sibling, wrong API, or incomplete mandatory checklist — with condition, current text, expected invariant/evidence, file, and minimum fix description.

Verdict:

- `READY` — no proven deterministic hard defect found;
- `BLOCKED` — at least one hard defect must be resolved or explicitly characterized before relying on the meta change.

Unresolved Open Questions that do not yet prove wrong guidance → **Warnings**, not `BLOCKED`.

## Axis D — Extras / Consistency / Preservation

Classify every changed hunk:

- `task` — directly justified by an atomic requirement or named meta goal;
- `incidental` — rename/cleanup/index sync required by the task;
- `neighbor` — unrelated rewritten guidance in a touched file.

Report hard `extra` / `regression` when proven:

- new guidance contradicts untouched canon without justification;
- removed/renamed skill or doc breaks index or `skills.paths` without update;
- capability ID uses ticket number or change name as folder;
- unjustified scope expansion (admin skill rules inside a pure learner-doc task, etc.);
- broken relative links introduced by the change (link target missing in meta);
- layer mixing: implementation checklist dumped into `proposal`, SC→test table into `design`, etc. against OpenSpec config rules.

Do not double-count the same issue as both `defect` and `extra` unless evidence and impact differ.

### OpenSpec-specific checks

When the diff touches `openspec/`:

1. Capability ID = product path (`start/register`, `subscription/gate`, `study/lesson`, …) — not ticket / change name;
2. Delta specs under `openspec/changes/<slug>/specs/<capability-id>/`;
3. Parent/child change links consistent when either side is edited;
4. Artifact roles respected (proposal intent / specs behavior / design how / tasks checklist);
5. `tasks.md` blocks (`## N. …`) coherent with design when both exist.

### Skills / AGENTS checks

When the diff touches skills or indexes:

1. New skill has `name` + `description` (when to use); listed in `.agents/AGENTS.md` if it is a discoverable project skill; root `AGENTS.md` discovery line updated when appropriate;
2. Renames/deletes update all meta references and kilo nested paths if applicable;
3. Skill must not instruct editing from align/verify read-only modes unless that is its purpose (orchestrators like `implement-change` / `review-change` may override for subagents);
4. Bot skills remain under `.agents/skills/bot/`.

## Severity

Every finding goes into exactly one tier. Do not silence softer items; do not inflate them into hard gaps.

| Tier | When | Align cues |
|------|------|------------|
| **Hard gaps** | Explicit requirement / forbidden content / deterministic wrong guidance / proven unjustified extra or broken index | `missing`, `docs-only`, `meta-only`, `extra`, `defect`, `regression` |
| **Warnings** | Strong lean without full proof; ambiguous AC; desirable analogue parity not mandatory; Open Question that can change expected meta text | `[warning]` |
| **Recommendations** | Optional polish vs analogue; sync unchecked OpenSpec tasks; soft wording; extra examples; non-blocking index niceties | `[recommendation]` |

Examples:

- New skill added but missing from `.agents/AGENTS.md` discovery table → hard `[missing]` / `[defect]` if discoverable; else **Warning**.
- Capability folder named `ht-123` or `redesign-task-set-cards` → hard `[extra]` / `[defect]` (forbidden pattern — use product path).
- Proposal promises keyboard/UX copy that is absent while change is message-driven → hard `[docs-only]` / `[missing]`.
- Two meta docs disagree on required fields → hard `[meta-only]`.
- OpenSpec `tasks.md` still unchecked while artifacts exist → **Recommendation** (docs sync), not hard unless AC demanded those tasks for this planning pass.
- Spec claims handler outcome `x` but bot proves `y` → **not this skill** → `bot-align-implementability`.

## Finding card (обязательный формат)

Формат как в `bot-brief-scope` (карточки ревью). Каждое замечание — отдельная карточка. **Не** писать замечание без файла и строки-якоря (кроме «файл отсутствует» — ожидаемый путь и `строки: —`).

В чате перед карточкой можно кратко показать цитаты источников (с **bold** на спорном фрагменте). В **сообщение** ревью цитаты обязательны.

```markdown
### <N>. <короткий заголовок>
- Метка: `[missing|docs-only|meta-only|extra|defect|regression|warning|recommendation]`
- **Что не так:** <факт: что написано сейчас / чего нет / чему противоречит>
- **Предлагаю:** <конкретная правка текста/контракта/индекса>  
  **или** Неясно как делать: <варианты A/B без выбора канона постановки>
- **Ревью**
  - файл: `path/from/repo/root.ext`   # относительный путь текстом, без markdown-ссылки
  - строки: Lnn                       # РОВНО ОДИН якорь (одна строка или один непрерывный фрагмент Lstart–Lend)
  - сообщение: <paste-ready текст без markdown-ссылок>
```

**Правила якоря (`файл` + `строки`):**

- Комментарий ревью вешают на **одно** место — один файл и одна строка (или один непрерывный фрагмент `Lstart–Lend`). Не писать «L88 и L153» как якорь.
- Якорь = место, которое **правят**, или главный конфликт. Остальные места — внутри `сообщение`.
- Путь — от корня `my-study-bot-meta`, plain-text в backticks (не markdown-link в блоке **Ревью**).
- Номера строк — по актуальному содержимому файла на диске.

**Правила `сообщение` (обязательно):**

- Самодостаточно и **коротко**: цитата «здесь» + (если нужно) цитата из другого файла + **одна фраза** сути + предложение.
- Другие файлы — plain-text путь + `Lnn` **и короткая цитата**: `` `path` Lnn: «…» ``. Не ограничиваться номером строки без цитаты.
- Суть — одной фразой, без простыни. Не сваливать три длинные цитаты подряд.
- **Запрещено:** «выше», «ниже», markdown-ссылки `[текст](url)`.
- Не дублировать якорь в сообщении как «здесь L153», если и так ясно; цитата с якоря — «Здесь: «…»».

**Пример (хорошо):**

```text
Здесь: «specs/ht-123/spec.md». Capability ID = номер задачи — запрещено. Нужен продуктовый путь (start/register, study/lesson, …).
```

**`Предлагаю`:**

- Рекомендация ревью / default для правки meta; **не** выдавать за уже принятый канон постановки, если канон не выбран.
- Если канон не выбран — **Неясно как делать** + 2 коротких варианта A/B, не маскировать под hard «исправить так».
- Actionable (что заменить/добавить/удалить), не «разобраться».

## Блок замечаний

Все findings — в одном блоке ниже, три яруса по порядку:

1. **Ошибки (hard)** — только hard cues; если пусто: `- нет`
2. **Предупреждения** — если пусто: `- нет`
3. **Рекомендации** — если пусто: `- нет`

Не смешивать пункты и строку `- нет` в одном ярусе. Опускать пустой ярус **нельзя** — всегда три подзаголовка.

## Report

Write in Russian.

```markdown
## Вердикт
- Постановка: N/10 — <основание; только hard>
- Готовность опираться на meta: READY | BLOCKED — <основание; только hard defect>
- Лишнее / регрессии канона: нет | N — <основание; только hard>
- Режим: requirements-only | docs/openspec | skills | mixed | bot-only
- Источник: <source>
- Работа в ветке meta: <staged / unstaged / untracked / commits>
- Следующий шаг: при необходимости `bot-align-implementability`

## Просмотренные файлы (meta)
- `path` — <added|modified|untracked|from commit>

## Замечания по созданным / изменённым файлам

### Ошибки (hard)
<!-- карточки 1..N в формате Finding card (с Ревью); если нет — единственная строка: -->
- нет

### Предупреждения
- нет

### Рекомендации
- нет

## Карта аналогов (meta)
- `path` — <зачем смотреть>

## Требуется решение до merge meta
- <только для BLOCKED; ссылки на N>

## Что делать дальше
- <hard → warnings → recommendations; затем опционально bot-align-implementability>
```

Never apply fixes from standalone align mode. Карточки описывают правки текстом (и paste-ready `сообщение` для ревью); файлы не менять — кроме override от `review-change` или явной просьбы править этот skill / named docs.

## Not This Skill

| Skill | Difference |
|-------|------------|
| `bot-align-implementability` | Bot/handlers/DB/FSM realizability of meta claims vs sibling |
| `align-code` | Runtime branch vs requirements — not meta-only diff |
| `check-changes` | Unstaged bot → propose skill/docs drift fixes |
| `openspec-verify-change` | Implementation vs OpenSpec artifacts |
| `prepare-changes` | Scope slicing before propose — not quality audit of created artifacts |
| `openspec-propose` / `openspec-update-change` | Creates/updates planning artifacts — not audit |

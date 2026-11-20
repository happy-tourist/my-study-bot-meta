---
name: bot-align-implementability
description: >-
  Use when judging whether my-study-bot-meta bot claims are implementable
  (possibility to build only — not meta doc quality, not that bot already
  shipped). Against my-study-bot via projects-map. Meta is a постановка for
  future apply: do NOT warn that planned handlers/FSM/User fields are absent in
  sibling HEAD. Verdict FEASIBLE|PARTIAL|INFEASIBLE|UNKNOWN. Finding card: Что
  не так / Предлагаю / Ревью (файл + одна строка-якорь + paste-ready сообщение).
  Not align-meta. Does not edit files.
---

# Align Implementability (Bot)

Perform a read-only **bot implementability** audit: can the **my-study-bot-meta** change be built **as written** against **my-study-bot**? Report findings in Russian.

**Scope of judgment — only possibility to implement.** Answers «можно ли реализовать bot по этим claims», not «идеальны ли AC», not «уже ли лежит готовый код в sibling». Completeness/quality of meta → **`align-meta`**.

### Постановка ≠ недоделанный bot (запрет замечаний)

Meta/OpenSpec change — **постановка к будущей реализации**. Sibling `my-study-bot` HEAD **ещё не обязан** содержать описываемый handler / FSM state / User field / scheduler job.

**Запрещено** писать finding-карточки вида «в bot ещё нет X из spec» / «надо добавить, потому что в HEAD нет». Отсутствие planned bot-поверхности в sibling — **норма**. Классифицируй claim как **`extends`** (или `aligns`, если паттерн/код уже есть) в вердикте / доказательной базе **без** отдельной карточки.

Карточку пиши только если **нельзя** реализовать как написано: нет аналога паттерна / **conflicts** с доказанным контрактом или кодом / критический `unproven` без пути реализации.

Output **one scope** with **three tiers** (hard → warnings → recommendations). Every finding is a **card**: file + line + what is wrong + fix proposal (or «неясно как делать»).

## Runtime roots

- **Claims subject:** changed / relevant `my-study-bot-meta` artifacts that assert **bot** runtime: handlers, middleware, User/SQLAlchemy, FSM, keyboards, scheduler, auth/subscription/admin, study flows, bot skills.
- **Evidence code:** resolve via [`docs/projects-map.md`](../../../../docs/projects-map.md) (+ `projects-map.local.yaml`). Primary: `bot` → `../my-study-bot/`. Skills: `.agents/skills/bot/`.
- Do **not** treat sibling working trees as the primary diff to “fix”.

## Scope filter (claims)

**Include** atomic claims about:

1. Commands / callbacks / Router wiring;
2. User fields / subscription_end / is_active / trial / admin / ban;
3. Middleware / AsyncSession injection;
4. FSM states and transitions;
5. Keyboards / Russian UX outcomes;
6. Scheduler expiry reminders / deactivation;
7. Study-domain rules owned by bot;
8. Bot skill procedures / conventions.

## Hard Boundary

Do not edit, create, delete, format, or commit files — except when called from [`review-change`](../../review-change/SKILL.md) (override: after report, apply meta fixes) or the user **explicitly** asks to change named meta files / this skill. Deliver the complete result in chat.

## Input And Store

Accept pasted requirements and an optional OpenSpec change name. If the branch clearly targets an OpenSpec change, audit from the change artifacts + bot evidence. Otherwise use **`my-study-bot-meta/openspec/`**.

## Load Branch Work (claims set)

Inspect read-only **in my-study-bot-meta**:

```bash
git status --short
git diff
git diff --cached
git branch -vv
```

Resolve merge-base: prefer `origin/master`, else `origin/main`, `master`, `main`.

When the branch has commits beyond base:

```bash
git merge-base HEAD <base-ref>
git log --oneline <merge-base>..HEAD
git diff --stat <merge-base>...HEAD
git diff <merge-base>...HEAD
```

Also collect untracked: `git ls-files --others --exclude-standard`. Union of commits + staged + unstaged + untracked is the claim file set; filter to bot-relevant claims.

## Load Sibling Evidence (bot)

For each atomic bot claim:

1. Resolve `bot` via projects-map (+ local YAML).
2. Grep/read `app/handlers*.py`, `app/database.py`, `app/states.py`, `app/keyboards.py`, `app/scheduler.py`, `app/auth.py`, `main.py`, tests that prove or refute the claim.
3. Prefer current HEAD of the sibling clone; if missing, **Warning** (`sibling-missing`), not a hard gap.

Before deep reads — `{bot}/AGENTS.md`. Do not invent sibling APIs.

## Axis — Bot implementability judgment

Compare independently per claim:

1. **Handlers / callbacks** — pattern exists or clear analogue for **extends**;
2. **User / DB** — fields/patterns match SQLAlchemy User store;
3. **FSM** — StatesGroup patterns;
4. **Keyboards / UX** — builders + Russian message outcomes;
5. **Scheduler** — APScheduler expiry patterns;
6. **Auth/subscription/admin** — gate helpers / ADMIN_IDS;
7. **Skill procedure** — conventions under `.agents/skills/bot/`.

Classify each claim: **aligns** | **extends** | **conflicts** | **unproven**.

Hard `gap` only with concrete sibling evidence + meta misstatement + required parity. **Absence of the new surface in sibling HEAD is not a hard gap** when meta clearly specifies an **extends** path.

Score **Код проекта: N/10** only from hard `gap` / `conflicts` / required `code-only` items.

| Вердикт | Когда |
|---------|--------|
| `FEASIBLE` | No hard `conflicts`; claims **aligns** or **extends**; remaining risks are Warnings only |
| `PARTIAL` | Mixed aligns/extends vs hard gap/conflicts, or critical pieces **unproven** |
| `INFEASIBLE` | Hard **conflicts** / required `gap` — **не** только потому что «в HEAD ещё нет» при ясном extends |
| `UNKNOWN` | Not enough sibling evidence — do **not** invent APIs |

## Severity

| Tier | When | Align cues |
|------|------|------------|
| **Hard gaps** | Required parity missing or claim conflicts proven sibling evidence | `gap`, `conflicts`, `code-only` |
| **Warnings** | Strong lean without full proof; sibling clone missing; critical path `unproven` | `[warning]`; `unproven`; `[sibling-missing]` |
| **Recommendations** | Optional polish vs analogue; soft wording | `[recommendation]` |

## Finding card (обязательный формат)

```markdown
### <N>. <короткий заголовок>
- Метка: `[gap|code-only|warning|recommendation]` (+ обязательно `aligns|extends|conflicts|unproven` для runtime-claim)
- **Что не так:** <факт: meta / sibling / противоречие>
- **Предлагаю:** <конкретная правка meta-контракта или уточнение extends>
  **или** Неясно как делать: <варианты A/B без выбора канона постановки>
- **Ревью**
  - файл: `path/from/repo/root.ext`
  - строки: Lnn
  - сообщение: <paste-ready текст без markdown-ссылок>
```

Якорь — одно место правки в meta. Sibling — plain-text в `сообщение` (`../my-study-bot/...` Lnn: «…»).

## Report

Write in Russian.

```markdown
## Вердикт
- Код проекта (bot): N/10 — <основание; только hard>
- Реализуемость bot (только возможность): FEASIBLE | PARTIAL | INFEASIBLE | UNKNOWN — <кратко; новый extends ≠ INFEASIBLE>
- Режим: docs/openspec | skills | mixed | bot-only
- Источник claims: <OpenSpec change / docs / skill>
- Работа в ветке meta: <staged / unstaged / untracked / commits>
- Предыдущий шаг: при необходимости сначала `align-meta`

## Claims (meta, bot)
- `path` — <какие bot runtime-утверждения проверяли>

## Доказательная база (bot sibling)
- `sibling-path` — <что подтвердили / не нашли / clone missing>

## Замечания по возможности реализовать (bot)

### Ошибки (hard)
- нет

### Предупреждения
- нет

### Рекомендации
- нет

## Карта аналогов (bot)
- `path` — <зачем смотреть>

## Что делать дальше
- <hard → warnings → recommendations; meta-quality → align-meta>
```

Never apply fixes in standalone mode — except override from `review-change`.

## Not This Skill

| Skill | Difference |
|-------|------------|
| `align-meta` | Quality of meta artifacts — not sibling realizability |
| `align-code` / `bot-align-code` | Runtime branch vs requirements after code exists |
| `openspec-verify-change` | Implementation vs OpenSpec (after code exists) |
| `bot-verify-code` | Bot code compliance, not meta claims vs sibling |

---
name: bot-align-code
description: Use when aligning branch and working-tree changes against the active OpenSpec change (primary), optional pasted clarifications, repository analogues, test readiness, preservation of previous behavior, aiogram handlers (/start upsert+greet), User model (subscription_end / is_active), DbSessionMiddleware session injection, FSM states, reply/inline keyboards, Russian user-facing messages, Windows-only SSL/IPv4 Bot session branch, and Docker Compose / GHCR deploy for my-study-bot.
---

# Align Code

Perform a read-only code alignment audit for the Telegram study bot (`my-study-bot`) and report findings in Russian across three tiers: hard gaps, warnings, and recommendations.

Stack context: aiogram 3.22 (Router / Dispatcher / long-polling), SQLAlchemy 2.0 + aiosqlite, python-dotenv, Python 3.13 in Docker (`python:3.13-slim`). Entry: `main.py` → `asyncio.run(main())` → `Bot` / `Dispatcher` / `dp.start_polling(bot)`. Windows-only SSL verify bypass + IPv4 `AiohttpSession` lives strictly under `sys.platform == "win32"`; Linux / Docker / VPS use clean `Bot(token=…)`. Authoritative user/subscription data: SQLite (`DB_URL`, default `sqlite+aiosqlite:///data/db.sqlite3`); handlers receive `session: AsyncSession` via `DbSessionMiddleware`. Users interact in Telegram only — no HTTP API / admin UI in this package.

**Paths:** this skill currently lives in **this bot repo** at `.agents/skills/bot/` (temporary). Canonical skills/OpenSpec will move to **my-study-bot-meta** when present (`project-map.md` key `my-study-bot-meta` → sibling `../my-study-bot-meta/.agents/skills/bot/`). Runtime `app/…`, `main.py` paths are relative to **this repository root**. Outside align-only mode, the agent runs `python main.py` / `pip install -r requirements.txt` (and Docker build/compose when verifying deploy) from this repo root; fix failures before claiming done.

## Four Required Axes

Always complete all four:

1. **Requirements fit** — code vs **активный OpenSpec change** (primary); paste/диалог только как явные уточнения.
2. **Codebase fit** — changed behavior vs strong repository analogues.
3. **Test readiness** — deterministic defects in reachable states before deriving tests.
4. **Behavior preservation** — unjustified regressions vs base in refactors, shared helpers, and neighboring edits.

The primary subject is the current working tree and branch diff: staged, unstaged, relevant untracked files, and commits vs base. The **active OpenSpec change** is the primary requirements source; paste/dialog are secondary clarifications only.

## Hard Boundary

Do not edit, create, delete, format, or commit files. Do not run tests for preservation analysis. Deliver the complete result in chat.

## Input And Store

**Primary:** resolve and load one **active OpenSpec change**. Accept an optional change name from the user; otherwise auto-select the single active change. Paste/dialog wording may refine AC only when it does not contradict the change (or the user explicitly overrides).

Ask when the change cannot be resolved (0 or >1 active without a clear pick, and no name given). Do not treat paste alone as a substitute for an active change unless the user confirms `OpenSpec: none`.

OpenSpec lives under **my-study-bot-meta**. Resolve meta via `project-map.md` key `my-study-bot-meta` (from bot: sibling `../my-study-bot-meta`). If OpenSpec/meta is unavailable and the user did not confirm fallback — ask; do not invent specs. Allow explicit `OpenSpec: none`.

### Resolve OpenSpec change

Resolve **one** change name before Axes A–D (including edge-case / test-readiness checks):

1. Use the name the user passed (argument, slash-command, explicit mention).
2. Else infer from conversation context if unambiguous.
3. Else from meta root: `openspec list --json` — auto-select if exactly **one** active change.
4. Else if several active changes — ask the user to pick one.
5. Else if none — ask for a change name or explicit `OpenSpec: none` (then paste/dialog/repo only).

Announce: `Using OpenSpec change: <name>` (or `OpenSpec: none`). Override: user passes another name.

That change is **the primary requirement evidence** for atomic checklists, Axis A, Axis C edge cases / scenarios, and docs↔code contradictions — not an optional afterthought.

## Requirements Sources

Load **in this order**:

1. **Active OpenSpec change** (required when resolved): `proposal`, delta `specs/**/spec.md`, `design`, `tasks` — scenarios (SC-*), acceptance wording, design decisions, task scope.
2. Optional paste / dialog clarifications that explicitly update expected behavior without inventing a parallel spec.
3. Repository / `AGENTS.md` / strong analogues — conventions only, never as replacements for change evidence.

Treat explicit prohibitions and exceptions as atomic requirements: "не возвращать", "не добавлять", "не принимать", "только для…", "кроме…", "без подписки", "stub".

`/start` registration, `User` subscription fields, middleware `session` injection, FSM/keyboards, and Russian reply copy are first-class when AC names them. Authoritative subscription/user truth lives in SQLite via injected session — do not treat client-side Telegram UI alone as fulfilment of DB AC.

A tester's guess/question is only a lead. Treat dialog wording as clarification only when it explicitly states or updates expected behavior relative to the active change.

## Atomic Requirement Checklist

### User / subscription fields

For every stated `User` column or subscription rule, independently verify:

- present vs intentionally omitted on the model / DB;
- source/path and transformation (Telegram update → handler → `AsyncSession` → SQLite);
- requiredness for product/AC;
- type and format (`BigInteger` Telegram id PK, `username` nullable String, `subscription_end` DateTime|null, `is_active` bool, `created_at`);
- allowed range, nullability, and defaults;
- default / empty / first-visit vs return behavior;
- who may mutate (bot handlers via injected session — prefer extending `User`, not a parallel store);
- gate semantics when AC demands subscription checks (`subscription_end`, `is_active`).

A persistence field is not covered until all explicit properties are covered. Known scaffold: schema has `subscription_end` / `is_active`; product subscription handlers may still be absent — do not treat column presence alone as fulfilment of subscription-feature AC.

### Handlers / commands / messages

For every command, message filter, or callback handler, independently verify:

1. triggering user action (`/start`, command, text, callback_data) and when the bot is ready (polling, middleware registered);
2. exact filter / command string (product today: `CommandStart()` → `/start`);
3. every payload field from Telegram update used (user id, username, first_name, callback data, FSM data);
4. handler registration on `Router` and inclusion via `dp.include_router(router)` in `main.py`;
5. DB / subscription / FSM guards before side effects;
6. authoritative persistence (upsert `User`, commit, read-after-write);
7. outbound reply text / keyboard after accept (Russian copy consistency with existing replies unless AC redesigns copy);
8. reject / no-op / error feedback when gated (what user observes in chat);
9. FSM enter/leave / state clear interaction with multi-step flows when AC demands forms;
10. middleware: handlers that need DB declare `session: AsyncSession` and use injected session (open/close per update).

### Middleware / session / startup

For every middleware / startup requirement, independently verify:

1. `DbSessionMiddleware` on `dp.update` opens `async_session`, injects `data["session"]`, closes after handler;
2. env contract: `TG_TOKEN` (required), `DB_URL` (default SQLite path under `data/`);
3. `load_dotenv()` + `init_db()` (`create_all` + `_ensure_sqlite_user_columns`) before polling;
4. Windows SSL/IPv4 hack only inside `sys.platform == "win32"` — must not leak into Docker/Linux production path;
5. handlers do not open a second engine/session path that bypasses middleware when AC expects injection;
6. missing/invalid `TG_TOKEN` → bot cannot poll (not silent “works without token”).

### FSM / keyboards

For every FSM or keyboard requirement, independently verify:

1. states live in `app/states.py` (or documented module); handlers use `FSMContext` correctly;
2. reply / inline keyboards built in `app/keyboards.py` (shared UI, not duplicated ad hoc unless AC allows);
3. callback_data / button text match handler filters;
4. state transitions and clear on cancel/complete when AC defines them;
5. stubs (`states.py` imports only; empty keyboard module) do not fulfil product AC that demands real forms/menus — note: topic-menu builders in `keyboards.py` are real product UI, not a stub.

Trace the concrete chain:

```text
Telegram update → Dispatcher → DbSessionMiddleware (session)
  → Router handler (filters / CommandStart / FSM)
  → SQLAlchemy User read/write → commit
  → message.answer / edit + optional keyboard
  ↔ callback / next FSM state → same middleware → handler
```

Example shape (`/start`):

```python
# user: /start
# bot: select User by telegram id → create if missing → greet + inline topic menu
#      first visit: "Привет! Я тебя запомнил 👋\n\nВыбери раздел:" + main_menu_kb()
#      return: "С возвращением, {first_name}!\n\nВыбери раздел:" + main_menu_kb()
# menu:* callbacks: edit_text section stub / back — not a subscription gate
```

Example shape (subscription gate — when AC adds it):

```python
# handler: load User via session → check subscription_end / is_active
#          → allow study flow or deny with Russian reply
```

Example shape (FSM — when AC adds it):

```python
# states.py: StatesGroup → handler sets state → keyboards.py Reply/Inline
#            → callback/message advances or clears FSMContext
```

Method presence or "looks compatible" alone is insufficient. Track each contract fact separately so one correct layer cannot hide another mismatch.

Prefer extending existing `User` + middleware session + `app/handlers.py` / `states.py` / `keyboards.py` rather than inventing parallel data paths or putting product logic only in `main.py` — unless AC explicitly says otherwise.

Use `AGENTS.md` and strong in-repo analogues for conventions; never as replacements for requirement evidence.

## Load Branch Work

Inspect read-only (from this bot repo root):

```bash
git status --short
git diff
git diff --cached
git branch -vv
```

When the branch has commits beyond base, determine merge-base and inspect:

```bash
git log --oneline <base>..HEAD
git diff --stat <base>...HEAD
git diff <base>...HEAD
```

Read relevant untracked files and full current Python modules when surrounding behavior matters. Prefer `main.py`, `app/handlers.py`, `app/database.py`, `app/middlewares.py`, `app/keyboards.py`, `app/states.py`, deploy (`Dockerfile`, `docker-compose.yml`, `.github/workflows/deploy.yml`), and env notes (`AGENTS.md`; no committed `.env.example` yet). Do not ignore unstaged work.

## Load Planning Scope

After resolving the change name (see **Resolve OpenSpec change**), from meta:

```bash
openspec status --change "<name>" --json
```

Read concrete artifact paths from the result (`proposal`, `specs`, `design`, `tasks`). Use them as **primary** requirements for **all four axes**, including Axis C edge cases and scenario IDs from delta specs. Report clear docs↔code contradictions. If no change resolved — only after user confirms `OpenSpec: none`, audit from paste/dialog and repository evidence; otherwise stop and ask. Do not invent specs.

## Axis A — Requirements

Judge implementation first, then planning artifacts.

Report hard gaps only as:

- `missing` — explicitly required and absent;
- `docs-only` — promised by complete artifacts but absent from implemented feature;
- `code-only` — clear docs/code contradiction;
- `extra` — clearly forbidden or unjustified behavior/data/outbound side effect.

Ambiguous CR wording, unresolved Open Questions, or unproven leans → **Warnings**, not hard omissions.

Wrong `User` field source/constraint, missing `/start` upsert, handler without injected `session` when AC requires DB, subscription gate absent while AC demands it, FSM/keyboard missing while AC demands forms, Windows SSL hack copied into non-win32/Docker path, or inventing a parallel user store are explicit omissions.

Score **Постановка: N/10** only from hard omissions (`missing` / `docs-only` / `code-only` / `extra`), not from Warnings or Recommendations.

Known scaffold gap vs product contract (repository fact — elevate to hard only when AC/docs demand the product surface): `/start` upsert + one-time trial + topic menu exists; Cars/Houses gated via `app/auth.py` `has_active_subscription`; Subscription shows tariffs (grant without payment); study **content** still stubs; `app/states.py` still a stub; expiry is on a temporary minute harness. Do not treat `states.py` alone as fulfilment of study AC; do not treat topic stubs as paid lesson content.

## Axis B — Codebase

Find strong untouched analogues for the same domain/flow. Prefer same layer:

| Concern | Look in |
|---------|---------|
| Entry / Bot / polling | `main.py` (`load_dotenv`, win32 session branch, `Dispatcher`, middleware, `init_db`, `include_router`, `start_polling`) |
| Handlers / commands | `app/handlers.py` (`Router`, `CommandStart`, tariffs / gate callbacks) |
| Auth / gate helper | `app/auth.py` (`has_active_subscription`) |
| User model / engine | `app/database.py` (`User` incl. `trial_used`, `async_session`, `init_db`, `_SQLITE_USER_COLUMN_DDL`) |
| DB session injection | `app/middlewares.py` (`DbSessionMiddleware`) |
| Keyboards | `app/keyboards.py` (topic menu, tariffs, gate CTA) |
| Scheduler | `app/scheduler.py` (expiry job; temporary minute harness) |
| FSM | `app/states.py` |
| Tests | `tests/` (expiry, auth, trial/tariffs/gate) |
| Deps | `requirements.txt` (aiogram 3.22, SQLAlchemy, aiosqlite, python-dotenv) |
| Env | local `.env` / server `.env` (gitignored); document `TG_TOKEN`, `DB_URL` — do not commit secrets |
| Deploy | `Dockerfile`, `docker-compose.yml`, `.github/workflows/deploy.yml` |
| Meta / OpenSpec | `../my-study-bot-meta` when present |

Compare `/start` upsert pattern, middleware session usage, Russian reply style, `User` column reuse, FSM/keyboard placement, win32-only SSL branch, and compose `env_file` + `./data:/app/data` volume only when the analogue proves the behavior.

Prefer thin handlers + injected session + shared keyboards/states — flag business logic only in `main.py`, duplicate engines, or bypassing middleware as codebase-implied when analogues keep that boundary.

Report `gap` only with:

- concrete analogue path;
- changed file lacking the behavior;
- why parity is required rather than optional polish.

Label analogue-driven findings as codebase-implied. Score **Код проекта: N/10** only from hard `gap` items (not Warnings / Recommendations).

## Axis C — Test Readiness

Use evidence in this order:

1. explicit requirements — including the **resolved OpenSpec change** (delta specs scenarios/SC-*, design constraints, task acceptance) when present; do not skip change artifacts and invent generic edge cases instead;
2. handler/command contracts, `User` fields, middleware session, FSM/keyboard shapes, env/deploy wiring;
3. strong analogues;
4. deterministic runtime semantics (first `/start`, repeat `/start`, missing username, expired subscription, inactive user, FSM cancel, callback without state).

For reachable behavior, examine permitted empty/null username, duplicate `/start`, subscription_end null vs past vs future, `is_active=False`, handler missing `session` arg while middleware expects injection, FSM stuck state, keyboard callback_data mismatch, win32 vs Linux Bot construction branches, compose volume path for SQLite — **and** every edge/negative path named or implied by the resolved change's specs/design.

Runtime facts that often create defects:

- handler uses DB but omits `session: AsyncSession` → runtime TypeError / missing data key;
- parallel engine/session bypasses middleware → connection leaks or uncommitted writes;
- subscription AC added but only columns exist, no gate in handlers → access always open;
- FSM states defined but handlers never set/clear state → stuck dialogs;
- empty keyboard module / missing builders while AC demands menus (topic menu builders already exist — extend them);
- Windows SSL bypass present outside `win32` branch → insecure production image;
- `TG_TOKEN` / `DB_URL` renamed in code but not in compose/`AGENTS.md` → deploy/runtime miss;
- `data/` not mounted in compose while SQLite path assumes `/app/data` → empty DB on restart;

### Handler / lifecycle feedback loops (обязательно)

Когда diff/ветка трогает handlers (`message` / `callback_query`), FSM transitions, middleware, startup/shutdown hooks, timers/`asyncio.create_task`/`asyncio.sleep` loops, или код, который из handler снова шлёт сообщение / вызывает тот же handler path — **отдельно** проверь, нет ли закальцованности (шторм сообщений, рекурсивных answer, бесконечных task ticks). Ожидание «один inbound update → конечное малое число outbound» — дефолт, пока AC явно не требует push/polling-имитации.

Для каждой такой цепочки независимо проверь:

1. **Триггер** — inbound message/command/callback, FSM state, timer, startup hook.
2. **Outbound side effect** — `message.answer` / `edit_text` / `bot.send_message`, другой handler path, DB write that re-enters the same flow, `asyncio` task без отмены.
3. **Повторный вход** — может ли side effect снова попасть в тот же handler без нового внешнего update (self-send, mirrored bot message handled by same filter, middleware re-entry).
4. **Идемпотентность / guard** — dedupe, «already registered», subscription already applied, task cancel on shutdown; отсутствие guard при доказанном re-entry → defect.
5. **Shutdown / polling stop** — background tasks cancelled in `shutdown`; иначе накопление work после остановки.
6. **Кратность** — на один `/start` / один callback ожидается конечный ответ, не каскад N сообщений без новых inputs.

Типичные петли для флага:

```text
handler / middleware → answer / send / DB
  → same handler again → …   (or uncleared asyncio task → storm)
```

- handler отвечает текстом, который снова матчит тот же фильтр и обрабатывается ботом;
- callback handler снова эмитит тот же `callback_data` path без смены state;
- FSM transition пишет данные, снова триггерящие тот же state handler;
- `asyncio.create_task` / periodic loop на startup без cancel в `shutdown` → tick storm;
- retry без backoff/max на Telegram/DB failure внутри handler path.

Report as hard `[defect]` when a reachable path deterministically storms messages/DB writes or spins a task/handler loop. If the chain looks risky but proof is incomplete → **Warning** with the suspected cycle edges.

Report hard `defect` only when a reachable state deterministically causes wrong reply, runtime failure, invalid DB state, stuck FSM, unsafe side effect (including message/timer storms), or contract violation.

Each hard defect includes condition, current behavior, expected invariant/evidence, file/symbol, and minimum scenario. Verdict:

- `READY` — no proven deterministic hard defect found;
- `BLOCKED` — at least one hard defect must be resolved or explicitly characterized before tests.

Unresolved Open Questions / ambiguous error-branch splits that do not yet prove wrong bot behavior go to **Warnings**, not `BLOCKED`, unless a reachable wrong reply/state is already deterministic.

Note: no automated test suite yet; when tests appear, prefer pytest + aiogram testing helpers. Cite missing coverage as **Recommendations** unless the missing test would be the only way to prove a hard defect already evidenced in code. Do not run `python main.py` / pytest here for preservation analysis; when verifying outside align-only mode, run commands from this repo root and fix failures before claiming done.

## Axis D — Behavior Preservation

Classify every changed hunk:

- `task` — directly justified by an atomic requirement;
- `incidental` — refactor/rename/cleanup/shared-helper change;
- `neighbor` — unrelated changed behavior in a task file.

Compare before vs after using diff/base, contracts, and callers. Report `regression` only for proven prior behavior changed without task justification: `/start` upsert/greet copy, `User` public columns, middleware session injection, router inclusion, win32-only Bot session branch, compose env/volume, shared-helper results, or FSM/keyboard public callback contracts.

Do not run tests. Do not double-count the same issue as both defect and regression unless the evidence and impact differ.

### Changed handlers, FSM, and message contract

Refactors of handler/FSM/keyboard surfaces often break Telegram UX quietly. When a changed file touches `app/handlers.py`, `app/states.py`, `app/keyboards.py`, or router wiring in `main.py` — audit the user-facing contract separately from internal helpers.

For every removed, renamed, or reshaped command/state/keyboard, independently verify:

1. **Old public surface** — command string, reply text, keyboard buttons/callback_data, success/deny behavior previously observed by users/tests.
2. **New resolution path** — which handler, state, or keyboard builder now owns the behavior.
3. **Caller migration** — grep handlers/keyboards/FSM for old command filters, callback_data, state names; untouched call sites are strong regression evidence.
4. **Effective UX** — trace what the user actually receives in chat; renamed callback with old button is a break.
5. **Indirect paths** — shared helpers used by `/start` and future study/subscription paths.

Typical regression patterns to flag:

- `/start` no longer upserts or changes greet copy without AC;
- `User` columns renamed/omitted while handlers still read `subscription_end` / `is_active`;
- middleware removed or session key renamed → handlers break;
- FSM state group renamed without handler updates;
- keyboard `callback_data` drift vs `F.data` filters;
- router no longer `include_router`'d from `main.py`.

Report as `regression` or `defect` when old paths deterministically stop greeting, persisting users, or advancing FSM. Mention approximate handler/keyboard hit count when grep proves it.

### Middleware, DB, env, and deploy wiring

Session injection, env, and deploy are easy to break in "cleanup" hunks. Treat them as public cross-layer contracts.

When a changed hunk touches `DbSessionMiddleware`, `app/database.py` / `User`, `TG_TOKEN` / `DB_URL`, win32 Bot branch, or Docker/GHCR deploy, independently verify:

1. **Session invariants** — handlers still receive `session`; middleware still opens/closes per update.
2. **Startup side effects** — `init_db` still runs; dotenv still loaded; polling still starts.
3. **User model** — columns and defaults still compatible with `/start` upsert and planned subscription gates.
4. **Platform branch** — SSL/IPv4 hack still win32-only; Linux/Docker path stays clean.
5. **Runtime vs deploy** — compose `env_file: .env` and `./data:/app/data` still match `DB_URL` default; GHCR image/tag and VPS cwd `/home/deploy/my-study-bot` assumptions not silently broken when AC touches ops.
6. **Secrets** — `.env` / `.env.server` remain uncommitted; no tokens in workflow scripts beyond existing GHCR/SSH secret usage.
7. **No handler storms** — thin command/callback handlers stay one-shot per update; no self-reentry or uncleared tasks (see **Handler / lifecycle feedback loops**).

Typical regression patterns to flag:

- unused-variable cleanup deletes session injection or `/start` commit;
- win32 SSL bypass copied into else/Docker branch;
- `User.is_active` default removed → unexpected inactive users;
- greet strings changed while AC only asked for subscription gate;
- compose volume dropped → SQLite wiped each deploy;
- new handler path that re-invokes itself or leaves a background task running after shutdown.

Report as `regression` or `defect` when users deterministically lose registration, session, subscription gate, or deploy persistence after the change, or when a reachable path storms messages. Name the field/command/keyboard/env key, before/after call sites, and consumers (handlers and/or future tests).

## Severity

Every finding goes into exactly one tier. Do not silence softer items; do not inflate them into hard gaps.

| Tier | When | Align cues |
|------|------|------------|
| **Hard gaps** | Explicit requirement / forbidden behavior / required analogue parity / deterministic wrong state / proven unjustified regression | `missing`, `docs-only`, `code-only`, `extra`, required `gap`, `defect`, `regression`; scores and `BLOCKED` use **only** this tier |
| **Warnings** | Strong lean without full proof; ambiguous AC with a clear risk; desirable analogue parity not proven mandatory; Open Question that can change expected behavior; likely docs↔bot mismatch pending аналитика | `[warning]`; does **not** lower scores or force `BLOCKED` alone |
| **Recommendations** | Optional polish vs analogue; docs/OpenSpec sync; unchecked planning tasks; test-coverage ideas that are not defects; soft parity with neighbouring handlers | `[recommendation]`; informational only |

Examples:

- AC requires subscription gate on study command and handler is absent → hard `[missing]`.
- AC unclear whether expired subscription gets renew CTA or plain deny; code picks one and no wrong state is proven → **Warning** until discriminant is settled.
- Analogue handler always uses injected `session` and CR says «как /start» → hard `[gap]` if that file is the cited evidence; if CR is silent → **Recommendation**.
- OpenSpec `tasks.md` still unchecked while code exists → **Recommendation** (docs sync), not hard requirements gap.
- Ambiguous CR wording that was misread once already → **Warning** with both readings, ask for confirmation rather than hard `missing`.
- AC demands FSM form and only stub `states.py` exists → hard `[missing]` / `[docs-only]`; do not treat import stub as fulfilment.
- Columns `subscription_end` / `is_active` exist while AC demands product subscription UX → hard `[missing]` / `[docs-only]` if handlers never gate.
- Handler / task re-enters itself (or uncleared asyncio loop after shutdown) and storms outbound work → hard `[defect]`.
- Suspected re-entry without a proven storm path → **Warning** with cycle edges.

Hard sections below still omit when empty. **Warnings** and **Recommendations** always appear; if empty, a single line `- нет`.

## Report

Write in Russian.

```markdown
## Вердикт
- Постановка: N/10 — <основание; только hard>
- Код проекта: N/10 — <основание; только hard>
- Готовность к тестам: READY | BLOCKED — <основание; только hard defect>
- Регрессии: нет | N — <основание; только hard>
- Режим: requirements-only | post-propose | in-progress
- Источник: активный OpenSpec change <name> (+ уточнения | none + fallback)
- OpenSpec change: <name | none; passed | active | inferred>
- Работа в ветке: <staged / unstaged / untracked / commits>

## Просмотренные файлы
- `path` — <state>

## Пробелы относительно постановки
- [missing|docs-only|code-only|extra] <gap and evidence>

## Пробелы относительно кода проекта
- [gap] <gap> — аналог: `path` — нет в: `changed-file`

## Расхождения docs и кода
- <proven contradiction>

## Явные дефекты перед написанием тестов
- [defect] <condition/current/expected/evidence/file/scenario>

## Регрессии прежнего поведения (без прогона тестов)
- [regression] <hunk class/before/after/justification/file/caller>

## Изменившийся deploy / env / middleware
- [regression|defect] <old env/volume/middleware/platform branch → new; runtime/deploy impact>

## Handlers / FSM / subscription / messages
- [regression|defect] <command|User field|session|FSM state|keyboard/callback|reply copy; before/after; consumers; why chain breaks>

## Handler / lifecycle-петли
- [defect|warning] <trigger → outbound → re-entry / uncleared task; expected 1 effect vs storm; file/symbol>

## Предупреждения
- [warning] <risk / ambiguity / likely gap; evidence; what would promote it to hard>
- нет

## Рекомендации
- [recommendation] <optional polish / docs sync / soft parity / test coverage>
- нет

## Карта аналогов
- `path` — <finding supported>

## Требуется решение перед тестами
- <only for BLOCKED from hard defects>

## Что делать дальше
- <hard first; then warnings; recommendations last>
```

Omit empty hard detail sections. Always keep **Предупреждения** and **Рекомендации** (use `- нет` when empty; do not mix items and `нет`).

Never apply fixes from align mode.

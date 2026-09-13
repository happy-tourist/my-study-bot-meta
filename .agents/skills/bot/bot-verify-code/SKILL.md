---
name: bot-verify-code
description: >-
  Use when the user asks to verify code, check skill compliance, audit a branch
  diff vs master/main/merge-base, audit local diffs, or after implementing a
  change in the my-study-bot Telegram (aiogram) package. Also when checking
  Router/handlers wiring, middleware session injection, User model / SQLite
  path, FSM/keyboards stubs, Windows SSL branch, Russian UX strings, deploy
  env, or DRY / KISS / YAGNI balance on changed bot files. Reports three tiers:
  Violations, Warnings, Recommendations.
---

# Verify Code

Use this skill to check **all production code changed on the current branch** in
the `my-study-bot` package against **all code-related** project skills under
`.agents/skills/bot/`, plus the **built-in** checks in this file (Bot
conventions, and DRY / KISS / YAGNI with conflict-aware judgment).

Default scope is the **full branch diff**: commits vs merge-base **and**
uncommitted working-tree changes. Do not stop at staged/unstaged/untracked.

This skill is primarily an **orchestrator**: domain rules live in the code
skills listed below — **read those skill files** when present and apply them; do
not restate or invent parallel domain rules here. Built-in sections below fill
gaps when a listed skill file is missing, and own bot conventions (aiogram
Router, middleware session, User model, Windows SSL branch, Russian UX, no
parallel DB path) plus DRY / KISS / YAGNI balance.

**Paths:** skills currently live in **this repo** at `.agents/skills/bot/`
(temporary; later move to **my-study-bot-meta**). Runtime code and `app/…` /
`main.py` paths are relative to **this bot repo root** (`my-study-bot`).
Sibling meta package is **`../my-study-bot-meta`**, not a path under skills.

**Stack assumptions:** aiogram 3.22 (Router, Dispatcher, long polling),
python-dotenv, SQLAlchemy 2.0 + aiosqlite (async ORM, SQLite under `data/`),
Python 3.13 in Docker (`python:3.13-slim`); local may be 3.10+. No npm.
Package manager: pip (`requirements.txt`). Tests: none yet; future pytest +
aiogram helpers when added.

## When To Use

- User asks to verify code / skill compliance / convention check
- After implementing a feature or refactor, before commit
- When reviewing the current branch (or a named path) for convention drift

## Scope: code skills only

Skills live under `.agents/skills/bot/<name>/SKILL.md` in this repo
(temporary; later `my-study-bot-meta/.agents/skills/bot/`).

### Always include (read and apply each when the file exists)

Verify against **every** skill in this set for the checked code — not a subset
guessed from the path. Path routing below only helps prioritize deeper reading;
it does **not** allow skipping skills from this list.

| Skill | Concern |
|-------|---------|
| `bot-work-with-errors` | error handling, empty `except`, user-facing failure replies |
| `bot-work-with-auth` | Telegram identity / subscription gates / access checks |
| `bot-work-with-structure` | `main.py` vs `app/` layout, package ownership |
| `work-with-config` | Bot/Dispatcher wiring, env-driven config |
| `work-with-handlers` | aiogram Router handlers, commands, filters |
| `work-with-middleware` | update middleware (e.g. `DbSessionMiddleware`) |
| `work-with-fsm` | FSM states / multi-step dialogs (`app/states.py`) |
| `work-with-models` | SQLAlchemy `User` / models (`app/database.py`) |
| `work-with-messages` | user-facing message copy and reply patterns |
| `work-with-study` | study / subscription product flows |
| `work-with-database` | engine, session, SQLite path, no parallel DB |
| `work-with-env-deploy` | `.env*`, secrets, Docker, Compose, GHCR, Actions |
| `work-with-keyboards` | reply / inline keyboards (`app/keyboards.py`) |

If a new code skill appears under `.agents/skills/bot/` (same kind: how to
write app code — handlers, middleware, FSM, models, messages, study, DB, env,
keyboards), include it too. Prefer reading one extra skill over missing a rule.

If a listed skill file is **missing**, do not invent a parallel rulebook — apply
**Built-in: Bot conventions** for that concern and continue.

### Always exclude

Do **not** use these for bot-verify-code (unless the user explicitly asks):

| Skill | Why excluded |
|-------|----------------|
| `bot-work-with-test` | test-writing / test conventions (not used to audit prod code) |
| `bot-locate-change-points` | planning where to edit |
| `bot-verify-code` | this orchestrator |
| `bot-align-code` | alignment / docs authorship |
| `openspec` / opsx skills | change workflow / specs |
| `commit` | commit messages |

Skip verifying files that are **only** tests (`tests/**`, `test/**`,
`**/test_*.py`, `**/*_test.py`, `**/__tests__/**`) unless the user explicitly
asks to include them. Focus on production/source code under `app/` and `main.py`
(and deploy/runtime config touched by the change: `Dockerfile`,
`docker-compose.yml`, `.github/workflows/**`, `requirements.txt`,
`.env.example` when present — not secret `.env` / `.env.server` contents).

## What To Check

**Always** inspect the full branch change set. Working tree alone is not enough.

Run git from the **repository root** (this bot package is the git root).

1. Resolve merge-base: `git merge-base HEAD <base-ref>`. Prefer `origin/main`
   (this repo), else `origin/master`, `main`, `master`, `develop` — first that
   exists.
2. Committed on the branch: `git diff --name-only <merge-base>...HEAD` and
   `git diff <merge-base>...HEAD`
3. Staged: `git diff --cached --name-only` and `git diff --cached`
4. Unstaged: `git diff --name-only` and `git diff`
5. Untracked: `git ls-files --others --exclude-standard` (read those files fully)

Union of 2–5 is the file set (filter to bot production sources unless the user
widened scope). Never skip step 2 because the working tree looks small.

Narrow only when the user **explicitly** asks for working-tree-only / only staged,
or names a **folder or path**.

Skip unrelated noise (lockfiles, coverage, binary assets, `data/db.sqlite3*`,
`__pycache__`, `.venv`) unless the user asked to include them. Do not commit or
demand review of production secrets in `.env` / `.env.server`.

## Commands (before done)

Run commands from the **bot package root** (`my-study-bot`). Fix failures before
claiming done. Do not skip tooling after changes unless the user explicitly
asked for a report-only pass with no tooling.

There is **no npm** in this package. When verification is part of an
implementation you own, or when the user asked to verify-and-fix / “make sure
it passes”, run:

| Command | Purpose |
|---------|---------|
| `pip install -r requirements.txt` | ensure deps match `requirements.txt` (when env/deps changed or imports fail) |
| `python -m compileall -q app main.py` | syntax / bytecode check for changed Python package |
| `pytest` (when present) | future automated suite; run only if pytest is installed and tests exist |

Prefer `python -m compileall` for syntax confidence. Do not invent a test suite
or npm scripts. When `pytest` / aiogram test helpers appear under `tests/`, run
them from this repo root. For interactive smoke, you may start `python main.py`
when needed (requires `.env` with `TG_TOKEN`); prefer finite gate commands
(compileall / pytest). Do not treat passing compile/pytest as a substitute for
skill checks.

Report failed command output under **Violations** (tooling) with the command
and a short failure summary. Do not invent flake8/mypy rules beyond the project
config and skill text.

## Workflow

1. Collect the file set: merge-base…HEAD **plus** staged, unstaged, untracked
   (or the path the user named). Exclude test-only files per Scope.
2. **Read all skills from the Always include table** that exist on disk (do not
   rely on memory; do not skip because the path “looks unrelated”).
3. For each source file, apply every code skill whose rules can touch that file.
   When unsure, apply the skill. If missing, use Built-in Bot conventions.
4. Apply **Built-in: Bot conventions** to all checked production files.
5. Apply **Built-in: DRY / KISS / YAGNI** to all checked production files.
   Resolve principle conflicts with the order in that section **before**
   classifying or fixing — never emit opposing principle fixes for the same hunk.
6. Classify every finding into Violations / Warnings / Recommendations
   (see Severity). Each item: file path, what is wrong, which skill/rule, and
   the expected pattern. Soft Prefer guidance → Warnings or Recommendations,
   not silence.
7. Fix **Violations** when the user asked to verify-and-fix, or when
   verification runs as part of an implementation task you own. Fix **Warnings**
   on verify-and-fix when the preferred pattern clearly fits. Fix
   **Recommendations** only if the user asked to tidy / apply soft order.
   On principle fixes, obey **Built-in: DRY / KISS / YAGNI** conflict order so
   DRY does not fight KISS/YAGNI (and vice versa). Otherwise list findings and
   wait.
8. When implementing / verify-and-fix: **run** `python -m compileall -q app main.py`
   (and `pip install -r requirements.txt` / `pytest` when relevant) from the
   bot package root; fix failures before claiming done.

Do not praise compliant code. Do not turn this into a product/bug review skill
or `bot-align-code`. Do not report `bot-work-with-test` findings unless asked.

## Built-in: Bot conventions

Grounded in `AGENTS.md` and current `app/` + `main.py` layout. Report as
`(skill: bot-verify-code / Bot conventions)`. Apply always; sibling skills win
when they exist and conflict on a detail.

### Entry and Dispatcher wiring

- Process entry: `main.py` → `asyncio.run(main())` → `dp.start_polling(bot)`.
- Startup order: `load_dotenv()` → build `Bot` → `Dispatcher` +
  `DbSessionMiddleware` on updates → `await init_db()` →
  `dp.include_router` for learner (`app.handlers`) and admin (`app.handlers_admin`) →
  startup/shutdown hooks → polling.
- Prefer keeping product handlers in `app/handlers.py` / `app/handlers_admin.py`
  (or future routers under `app/`), not stuffing study/subscription/admin logic
  only into `main.py`.

**Violations when:** business logic / handlers live only in `main.py` while
`app/handlers.py` / `app/handlers_admin.py` already own that surface; a second ad-hoc polling/bot
bootstrap bypasses the existing Dispatcher + middleware path without an
explicit request.

### Windows SSL branch (win32 only)

- Local Windows path may disable SSL verify and force IPv4 for VPN/debug
  (`sys.platform == "win32"` + custom `AiohttpSession`).
- Linux / Docker / VPS must use the clean `Bot(token=…)` branch.
- Do **not** copy the Windows SSL bypass into the Dockerfile, Compose image, or
  production runtime.

**Violations when:** SSL verify disable / IPv4 connector hacks appear outside
the `win32` branch, or are baked into Docker/production images.

### Middleware session injection

- `app/middlewares.py` — `DbSessionMiddleware` opens `async_session()`, sets
  `data["session"]`, and closes per update.
- Handlers that need DB must declare `session: AsyncSession` and use the
  injected session — do not open a parallel engine/session inside handlers
  without an explicit reason.

**Violations when:** handlers create their own SQLAlchemy engine/session
alongside middleware injection; middleware session lifecycle is broken
(session not closed / shared across updates incorrectly).

### User model and persistence

- `app/database.py` — `User` columns: `id` (Telegram `BigInteger` PK),
  `username`, `subscription_end`, `is_active`, `created_at`.
- Default DB URL: `sqlite+aiosqlite:///data/db.sqlite3` via `DB_URL`.
- Prefer extending the existing `User` model and middleware session injection
  for study/subscription behavior.

**Violations when:** a parallel user store / second SQLite path / alternate ORM
stack is introduced for the same Telegram users without an explicit request.
**Warnings when:** new NOT NULL columns or required fields break `/start`
upsert without a clear migration/default path.

### Handlers and Russian UX

- `/start` upserts `User`, greets (first visit vs return), and attaches inline topic menu (`main_menu_kb`).
- Keep **Russian** user-facing strings consistent with existing replies unless
  product copy is being redesigned.
- Shared UI builders belong in `app/keyboards.py` (topic menu is real product UI);
  multi-step dialogs in `app/states.py` (still a stub — extend rather than invent parallel modules).

**Violations when:** user-visible copy switches language inconsistently without
an explicit redesign; keyboard/FSM logic is duplicated in handlers while
`keyboards.py` / `states.py` should own it and the change clearly fits those
modules. **Warnings when:** English-only new UX is mixed into Russian flows
without a stated product decision.

### Layering (hard direction)

Typical paths:

```text
Telegram update → Dispatcher middleware → handler (session injected)
Persistence → SQLAlchemy AsyncSession → SQLite under data/
```

| Area | Location | Owns |
|------|----------|------|
| Entry | `main.py` | Bot, Dispatcher, polling, win32 session branch |
| Handlers | `app/handlers.py` | Learner Router / commands |
| Admin | `app/handlers_admin.py` | Admin panel Router |
| Auth | `app/auth.py` | subscription + admin/ban helpers |
| DB | `app/database.py` | engine, `User`, `init_db` |
| Middleware | `app/middlewares.py` | session injection |
| Keyboards | `app/keyboards.py` | reply / inline builders |
| FSM | `app/states.py` | FSM states (`AdminSearchForm`) |

**Do not:** invent a parallel DB path; put study logic only in `main.py`;
invert layers (e.g. models importing Telegram handlers unnecessarily).

### Env, secrets, deploy

- Env vars: `TG_TOKEN` (required), `DB_URL` (optional default above), `ADMIN_IDS` (CSV bootstrap admins).
- Do not commit secrets (`.env`, `.env.server` are gitignored).
- Deploy: GHCR image `ghcr.io/happy-tourist/my-study-bot:latest`, Compose
  service `bot` with `env_file: .env` and volume `./data:/app/data`, VPS cwd
  `/home/deploy/my-study-bot`, Actions on `main`.

**Violations when:** secrets hardcoded in source or committed env files;
Windows SSL bypass copied into production image. **Warnings when:** deploy
paths/image names drift from the established Compose/Actions contract without
an explicit rename.

### Soft style

- Soft tidy (Recommendations): group imports; keep handlers thin; avoid
  drive-by reformat of untouched hunks.

### Agent commands

- Run `pip install -r requirements.txt`, `python -m compileall -q app main.py`,
  and `pytest` when present, from the bot package root; fix failures before
  claiming done. Do not skip verification or assume pass without running.

### Severity cues for built-in conventions

| Tier | Examples |
|------|----------|
| **Violations** | Parallel DB/user store; Windows SSL bypass in Docker/prod; handlers skip middleware session; business logic only in `main.py`; secrets in source; empty `except` on registration/DB paths |
| **Warnings** | Prefer extend `User` but a one-off parallel table/field was added; env URL/path hardcoded when `DB_URL` already covers it; mixed-language UX without redesign |
| **Recommendations** | Import grouping tidy; minor formatting; soft consistency with nearby handler/keyboard analogues; mild DRY/KISS polish |

## Built-in: DRY / KISS / YAGNI

Report as `(skill: bot-verify-code / DRY-KISS-YAGNI)`. Apply to all checked
production files. Principles are **judgment lenses**, not absolute mandates.

**Core:** Prefer the simplest correct code that meets the real requirement.
Project skills and Bot conventions **win** over DRY, KISS, and YAGNI. When
principles pull opposite ways, choose **one** outcome using the conflict order
below — never “satisfy DRY” by violating KISS/YAGNI or a domain skill, and never
“simplify” by deleting a required shared contract.

### Verify lens

| Principle | Flag when | Do not flag when |
|-----------|-----------|------------------|
| **DRY** | Same *knowledge/rule* is duplicated in the change set and will drift if only one side changes | Similar-looking code with different domain meaning; intentional parallel handlers a skill requires; trivial short copies that stay clearer separate |
| **KISS** | New indirection, factory, or clever layer that obscures the change without payoff | Required layering from skills (Router, middleware session, models, FSM, keyboards) |
| **YAGNI** | Abstraction, extension point, or helper built for hypothetical future call sites (0–1 real uses of that shape) | Small helper with 2+ real call sites of the *same* shape already in the change |

### Conflict resolution (required before classify / fix)

Apply **in order**:

1. **Domain skills / Bot conventions / contracts** — do not dedupe or simplify away a required pattern (e.g. keep middleware session injection; do not merge distinct handler/FSM/keyboard concerns into one grab-bag).
2. **YAGNI** — remove or do not introduce unused / future-only abstractions.
3. **KISS** — prefer direct code over shared machinery when duplication is small or meanings differ.
4. **DRY** — extract only when identical knowledge would otherwise drift; the extraction must stay simple.

**Anti-conflict rule:** Do not report both “extract for DRY” and “inline for KISS/YAGNI” on the same hunk. Pick one using the order above. On verify-and-fix, apply that single outcome. Prefer **one finding per hunk** that names the winning principle (mention secondary principles only as supporting reason in the same bullet).

### Severity cues

| Tier | When |
|------|------|
| **Violations** | Rare for pure principles. Only if a new premature abstraction **breaks** a required skill/contract, or critical contract knowledge is duplicated **and already diverges** in the same change set |
| **Warnings** | Identical-knowledge duplication likely to drift; unused/future-only abstraction; complexity that obscures a preferred skill pattern |
| **Recommendations** | Mild duplication or slightly overcomplicated local code where a simpler safe shape is obvious |

### Red flags — STOP and re-resolve

- “Merge for DRY” when a skill or contract says keep the split
- Shared util with one call site “for later” (YAGNI)
- Inlining a module that multiple real callers already need (false KISS)
- Opposing principle findings on the same hunk without conflict resolution

## Path hints (optional prioritization only)

Use only to decide **where to look harder**, never to drop a skill from Always include.

| Change / path signals | Look harder at |
|----------------------|----------------|
| `main.py` | Bot conventions (startup order, win32 SSL branch, Dispatcher wiring) |
| `app/handlers.py` | `work-with-handlers`, `work-with-messages`, `work-with-study` |
| `app/handlers_admin.py` | `work-with-handlers`, `bot-work-with-auth`, `work-with-fsm` |
| `app/auth.py` | `bot-work-with-auth` |
| `app/middlewares.py` | `work-with-middleware`, session injection |
| `app/database.py` | `work-with-models`, `work-with-database` |
| `app/states.py` | `work-with-fsm` |
| `app/keyboards.py` | `work-with-keyboards` |
| auth / subscription gates | `bot-work-with-auth` |
| `.env*`, `Dockerfile`, `docker-compose.yml`, `.github/workflows/**` | `work-with-env-deploy` |
| error handling / empty `except` | `bot-work-with-errors` |
| `app/` layout ownership | `bot-work-with-structure`, `work-with-config` |

## High-signal checks (examples, not a full rulebook)

Reminders to **open the skill** (or Built-in) — skill text wins.

- **Entry**: `main.py` wires Bot/Dispatcher/middleware/`init_db`/router; keep product logic in `app/`.
- **win32 SSL**: only inside `sys.platform == "win32"`; never in Docker/prod.
- **Session**: handlers use injected `session: AsyncSession`; no parallel DB path.
- **User**: extend existing `User` model; SQLite under `data/` via `DB_URL`.
- **UX**: Russian user-facing strings; keyboards/FSM in dedicated modules.
- **Deploy**: secrets not committed; Compose volume `./data:/app/data`.
- **Tooling**: `pip install -r requirements.txt` / `python -m compileall` / future `pytest` from bot repo root — no npm.
- **DRY/KISS/YAGNI**: conflict order — skills → YAGNI → KISS → DRY; one fix per hunk.

## Severity

Map each skill finding by how the source skill phrases the rule. Skill text wins;
when wording mixes levels, pick the strongest that still applies.

| Tier | When | Skill wording cues (examples) |
|------|------|-------------------------------|
| **Violations** | Hard break of a required pattern / contract | Do / Don't, Must, Never, Always, Hard Rules, parallel DB path, win32 SSL in prod, missing middleware session |
| **Warnings** | Preferred pattern clearly fits; allowed exception does **not** apply | Prefer, Should, “use X instead of Y” when X fits |
| **Recommendations** | Soft tidy / style / optional polish | usually, Soft, import grouping, optional analogue consistency, mild DRY/KISS |

Do not invent findings outside the code skills and the built-in sections above.
Do not inflate Prefer into Violations.

## Output

Report language: **Russian** for prose; keep paths, symbols, and code as-is.
Section headings stay **Violations** / **Warnings** / **Recommendations** (and
`## Verify Code`) as in the template below.

Always include all three finding sections. If a section has no items, put a
single line `- none` (do not mix items and `none` in the same section).

Format:

```text
## Verify Code

Scope: branch vs <merge-base> (<base-ref>) + staged + unstaged + untracked [+ path if used]
Skills: all code skills (excl. test/locate/align/openspec/commit) + Bot conventions + DRY/KISS/YAGNI
Tooling: <commands run and pass/fail summary, or skipped with reason>

### Violations
- `path` — <что не так> (skill: `<name>`)
  Expected: <краткий правильный паттерн или секция skill>

### Warnings
- `path` — <что не так> (skill: `<name>`)
  Expected: <краткий правильный паттерн или секция skill>

### Recommendations
- none
```

Empty-section example: only `- none` under that heading. Item shape is the same
in every tier. Do not collapse Warnings/Recommendations into Violations or omit
soft Prefer findings.

## Red Flags — collect the branch first

These mean STOP and re-collect files vs merge-base before reporting:

- Ran only `git diff` / `--cached` / untracked
- “User didn’t ask for the whole branch”
- “Working tree has the real work”
- “I’ll check working tree first, branch later”
- Report Scope without `branch vs <merge-base>`

## Related Skills

All skills in the Always include table. For tests use `bot-work-with-test`
when present (and when the user asks to include tests). For requirements /
analogue alignment use `bot-align-code`. Bot conventions and DRY / KISS /
YAGNI balance are owned by this skill when sibling files are absent.

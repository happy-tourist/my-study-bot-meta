# Индекс агента — my-study-bot-meta

Канонический каталог документации и skills для AI-агентов в метарепозитории My Study Bot (Telegram study bot).

Краткие always-on инструкции (Cursor): [`AGENTS.md`](../AGENTS.md) в корне репозитория.

Runtime-код: sibling  
[`../my-study-bot`](../my-study-bot) (aiogram 3 + SQLAlchemy / SQLite, long-polling).  
Контекст пакета:

| Пакет | AGENTS.md |
|-------|-----------|
| Bot | [`../my-study-bot/AGENTS.md`](../my-study-bot/AGENTS.md) |

---

## Документация

| Документ | Назначение |
|----------|------------|
| [`docs/projects-map.md`](../docs/projects-map.md) | Сервис → sibling-путь; local YAML; Child `project-map.md` (ключ `my-study-bot-meta`) |
| [`docs/README.md`](../docs/README.md) | Оглавление документации |

Локальные оверрайды путей: [`projects-map.local.yaml`](../projects-map.local.yaml) / шаблон [`projects-map.local.yaml.example`](../projects-map.local.yaml.example) в корне meta.

OpenSpec: [`openspec/config.yaml`](../openspec/config.yaml); артефакты в `openspec/changes/` и `openspec/specs/`.

---

## Skills

### OpenSpec / workspace (`.agents/skills/`)

| Skill | Когда |
|-------|--------|
| [`align-code`](skills/align-code/SKILL.md) | Bot: `bot-align-code` + `bot-verify-code` → сводный отчёт |
| [`commit`](skills/commit/SKILL.md) | Status → stage → commit → **push** по meta + bot (main/master) |
| [`check-changes`](skills/check-changes/SKILL.md) | Unstaged bot → предложения: добавить/изменить/**удалить** skills, maps, docs, AGENTS.md |
| [`sync-changes`](skills/sync-changes/SKILL.md) | Сравнение orchestration (config.yaml, align/check/implement/end/commit + связанные) с другим проектом → предложения добавить сюда и туда |
| [`implement-change`](skills/implement-change/SKILL.md) | Активный change: apply (блоки tasks в субагентах) → align → check-changes → commit |
| [`end-implement-change`](skills/end-implement-change/SKILL.md) | Закрытие change: update → sync-specs → archive → commit (без вопросов) |
| [`openspec-propose`](skills/openspec-propose/SKILL.md) | Propose: proposal → specs → design → tasks; при активном change по той же теме — править его, не создавать новый |
| [`openspec-new-change`](skills/openspec-new-change/SKILL.md) | Создать change и начать артефакты по схеме |
| [`openspec-continue-change`](skills/openspec-continue-change/SKILL.md) | Продолжить создание недостающих артефактов change |
| [`openspec-update-change`](skills/openspec-update-change/SKILL.md) | Правка существующих артефактов change без кода |
| [`openspec-apply-change`](skills/openspec-apply-change/SKILL.md) | Реализация по `tasks.md` |
| [`openspec-verify-change`](skills/openspec-verify-change/SKILL.md) | Проверка реализации против артефактов |
| [`openspec-archive-change`](skills/openspec-archive-change/SKILL.md) | Архивация закрытого change |
| [`openspec-explore`](skills/openspec-explore/SKILL.md) | Explore mode до/во время change; явно флагает «⛔ нужно решение разработчика» если фича ещё не implementable в проекте |
| [`openspec-sync-specs`](skills/openspec-sync-specs/SKILL.md) | Delta → main specs без archive |

### Discovery (другие корни)

| Корень | Назначение |
|--------|------------|
| [`.agents/skills/bot/`](skills/bot/) | Bot-wide skills (meta); runtime: `../my-study-bot/` |

#### Bot-wide skills (`.agents/skills/bot/`)

Стек: aiogram 3.22 (Router / Dispatcher / long-polling), SQLAlchemy 2.0 + aiosqlite, APScheduler (expiry cron), python-dotenv, Python 3.13 в Docker (`python:3.13-slim`). Runtime: `../my-study-bot/`. Entry: `main.py` → `asyncio.run(main())` → `dp.start_polling(bot)`. Windows-only SSL/IPv4 `AiohttpSession` — строго под `sys.platform == "win32"`.

| Skill | Когда |
|-------|--------|
| [`bot-align-code`](skills/bot/bot-align-code/SKILL.md) | Audit ветки / диффа против OpenSpec change, аналогов, test readiness; handlers / User / middleware / FSM / keyboards / deploy |
| [`bot-locate-change-points`](skills/bot/bot-locate-change-points/SKILL.md) | Где править / куда класть новые файлы (без правок кода) |
| [`bot-verify-code`](skills/bot/bot-verify-code/SKILL.md) | Проверка кода / compliance skills + DRY/KISS/YAGNI |
| [`bot-work-with-auth`](skills/bot/bot-work-with-auth/SKILL.md) | `/start` upsert + trial, gate/ban, `ADMIN_IDS` / `is_admin` / `is_banned`, admin panel (`handlers_admin`) |
| [`bot-work-with-errors`](skills/bot/bot-work-with-errors/SKILL.md) | Handler/DB/FSM/Telegram errors → русские `message.answer` / `callback.answer` |
| [`bot-work-with-structure`](skills/bot/bot-work-with-structure/SKILL.md) | `main.py` / learner+admin handlers / auth / models / middleware / keyboards / states / Docker |
| [`bot-work-with-test`](skills/bot/bot-work-with-test/SKILL.md) | pytest + pytest-asyncio (trial/tariffs/gate, admin panel, auth helper, expiry) |
| [`work-with-config`](skills/bot/work-with-config/SKILL.md) | `load_dotenv`, `TG_TOKEN` / `DB_URL` / `ADMIN_IDS`, win32 session vs clean `Bot`, dual-router startup |
| [`work-with-database`](skills/bot/work-with-database/SKILL.md) | SQLite User store, `DB_URL`, `init_db`, session injection, `trial_used` / `is_admin` / `is_banned` |
| [`work-with-models`](skills/bot/work-with-models/SKILL.md) | SQLAlchemy `User` / DeclarativeBase / subscription + admin/ban columns |
| [`work-with-middleware`](skills/bot/work-with-middleware/SKILL.md) | `DbSessionMiddleware`, registration, AsyncSession lifecycle |
| [`work-with-handlers`](skills/bot/work-with-handlers/SKILL.md) | Learner + admin routers, filters, trial/tariffs/gate/`admin:*`, `include_router` |
| [`work-with-messages`](skills/bot/work-with-messages/SKILL.md) | `message.answer` / reply / edit / captions, русский UX, parse_mode |
| [`work-with-keyboards`](skills/bot/work-with-keyboards/SKILL.md) | Reply/Inline builders в `app/keyboards.py`, `menu:*` / `tariff:*` / `admin:*` |
| [`work-with-fsm`](skills/bot/work-with-fsm/SKILL.md) | `StatesGroup` / `FSMContext`, `AdminSearchForm` + future study forms (не подмена SQLite) |
| [`work-with-study`](skills/bot/work-with-study/SKILL.md) | Продуктовая study-логика, lessons, subscription-gated learning |
| [`work-with-scheduler`](skills/bot/work-with-scheduler/SKILL.md) | APScheduler: expiry reminders / deactivation; temporary minute harness vs day+10:00 MSK |
| [`work-with-env-deploy`](skills/bot/work-with-env-deploy/SKILL.md) | `.env` (`TG_TOKEN` / `DB_URL` / `ADMIN_IDS`) / Docker Compose / VPS / GHCR |

Runtime-пути в skills (`app/…`, `main.py`) — относительно корня sibling-репозитория `my-study-bot`.

Kilo Code: при появлении канона [`../kilo.template.jsonc`](../kilo.template.jsonc); локально `cp kilo.template.jsonc kilo.jsonc` (ignored). Nested skills — `skills.paths` в рабочем `kilo.jsonc`.

---

## OpenSpec / workflow

- Конфиг: [`openspec/config.yaml`](../openspec/config.yaml) (`projectsMap`, `context`, capability rules).
- Capability ID = продуктовый путь (start/register, subscription/gate, study/lesson, …), не имя change.
- Delta: `openspec/changes/<slug>/specs/<capability-id>/spec.md`
- Main: `openspec/specs/<capability-id>/spec.md`
- Типичный Cursor chat workflow: `/opsx-explore` → `/opsx-propose` → review → `/opsx-apply` → `/opsx-sync` → `/opsx-archive`.
- Артефакты живут в **my-study-bot-meta**, не в sibling. Пути к runtime при apply — через [`docs/projects-map.md`](../docs/projects-map.md) (+ local YAML).
- Always-on: [`AGENTS.md`](../AGENTS.md).

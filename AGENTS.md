# AGENTS.md — my-study-bot-meta

Метарепозиторий экосистемы My Study Bot (Telegram-бот для обучения): каноническая документация и skills для AI-агентов.  
**Runtime-код здесь не живёт** — исходники в sibling-репозитории `../my-study-bot` (bot).

Подробный индекс docs и skills: [`.agents/AGENTS.md`](.agents/AGENTS.md).

## Обязательно перед работой

1. Из корня meta: [`docs/projects-map.md`](docs/projects-map.md) — таблица «Пути для OpenSpec / агентов»; при наличии — смержить [`projects-map.local.yaml`](projects-map.local.yaml) (local перекрывает канон). Шаблон: [`projects-map.local.yaml.example`](projects-map.local.yaml.example).
2. Если работаешь в sibling-репозитории — сначала локальный `project-map.md` (ключ `my-study-bot-meta` → `..`), затем `my-study-bot-meta/docs/projects-map.md` (+ local YAML в meta).
3. Пути к коду — sibling через projects-map (+ local):
   - Bot: `../my-study-bot`
4. Канон документации: [`docs/README.md`](docs/README.md). В child markdown — ссылки вида `my-study-bot-meta/docs/...` (резолв через `project-map.md`).
5. **Контекст пакета:** перед правками читай `AGENTS.md` целевого репозитория:
   - Bot (aiogram 3 + SQLAlchemy / SQLite): [`../my-study-bot/AGENTS.md`](../my-study-bot/AGENTS.md)
6. **Перед выбором скилла:** skills из `my-study-bot-meta/.agents/skills/` — [`bot/`](.agents/skills/bot/) для runtime Telegram-бота, плюс OpenSpec (`openspec-*`). Локальных канонических `.agents/skills/` в sibling-репозитории нет (в bot мог остаться временный дубликат до переноса).

## OpenSpec

- Конфиг: [`openspec/config.yaml`](openspec/config.yaml).
- Capability ID — продуктовый путь (start/register, subscription/gate, study/lesson, …), не номер тикета и не имя change.
- **Запрещено** класть в `openspec/specs/` номер задачи или имя change как capability.
- Артефакты OpenSpec создаются и архивируются **в my-study-bot-meta**, не в sibling runtime-репозитории.
- Пути к runtime при apply — через [`docs/projects-map.md`](docs/projects-map.md) (+ local YAML).

## Работа с OpenSpec

- **task** — schema по умолчанию `spec-driven` (proposal → specs → design → tasks).
- Типичный Cursor chat workflow: `/opsx-explore` → `/opsx-propose` → review → `/opsx-apply` → `/opsx-sync` → `/opsx-archive`.

## Работа с Git

- Runtime-код коммитить в sibling `my-study-bot`; docs/OpenSpec/skills meta — в `my-study-bot-meta`.
- Если начинаем новую задачу, но есть локальные изменения — спроси, что с ними делать.
- Не пушить и не создавать PR/MR без явной просьбы пользователя.

## Skills (discovery)

- OpenSpec / workspace (этот репозиторий): `.agents/skills/` (`openspec-*`, [`align-code`](.agents/skills/align-code/SKILL.md), [`check-changes`](.agents/skills/check-changes/SKILL.md), [`sync-changes`](.agents/skills/sync-changes/SKILL.md), [`commit`](.agents/skills/commit/SKILL.md), [`implement-change`](.agents/skills/implement-change/SKILL.md), [`end-implement-change`](.agents/skills/end-implement-change/SKILL.md))
- Bot-wide (meta): [`.agents/skills/bot/`](.agents/skills/bot/) — runtime Telegram bot: `../my-study-bot/`

Детали и перечень: [`.agents/AGENTS.md`](.agents/AGENTS.md).

## Сборка и команды

Команды `pip` / запуск / тесты / Docker запускает **агент** из каталога sibling-репо (`my-study-bot`). Не ждать подтверждения пользователя; при падении — починить до завершения задачи.

Типичные команды (из корня `my-study-bot`):

- `python -m venv .venv` / активация `.venv`
- `pip install -r requirements.txt`
- `python main.py` — long-polling бот (нужен `.env` с `TG_TOKEN`)
- `pytest` — suite в `{bot}/tests/` (запускать из корня sibling `my-study-bot`)
- Docker: `docker compose build` / `docker compose up -d`

## Kilo Code: lazy loading nested skills

При появлении канона: [`kilo.template.jsonc`](kilo.template.jsonc) → локально `cp kilo.template.jsonc kilo.jsonc` (ignored).  
В `kilo.jsonc` пути nested skills (`skills.paths`: `.agents/skills/bot`).  
Root `.agents/skills/` (`openspec-*`, `commit`, …) обнаруживаются автоматически — не дублировать в `skills.paths`.

Скилл [`commit`](.agents/skills/commit/SKILL.md): незакоммиченные изменения в meta + bot → stage → commit на `main`/`master` → **всегда push** (запуск скилла = разрешение на push).

# Документация экосистемы My Study Bot

Канон для AI-агентов и разработчиков. Исходники живут в sibling-репозитории `../my-study-bot` (bot) — не внутри meta.

## Как читать

0. [projects-map.md](projects-map.md) — карта сервис → sibling-путь; local YAML.
1. OpenSpec: [`openspec/config.yaml`](../openspec/config.yaml) — context, rules, apply/verify files.
2. Пакетный контекст runtime:
   - Bot: `{bot}/AGENTS.md` → `../my-study-bot/AGENTS.md`
3. Skills и индекс агентов: [`.agents/AGENTS.md`](../.agents/AGENTS.md) (`.agents/skills/bot/`, `openspec-*`).

Пути вида `{ключ}/…` и `my-study-bot-meta/docs/...` резолвить через [projects-map.md](projects-map.md) (+ `projects-map.local.yaml`) и child `project-map.md` (ключ `my-study-bot-meta`).

## Каталог docs (meta)

| Документ | Содержание |
|----------|------------|
| [projects-map.md](projects-map.md) | Карта сервис → sibling-путь (+ local YAML); таблица «Пути для OpenSpec / агентов» |

Общие code/test rules живут в пакетном `AGENTS.md` и skills (отдельного `docs/bot/` нет).

OpenSpec: [`openspec/config.yaml`](../openspec/config.yaml); changes/specs в [`openspec/`](../openspec/). Domain-capability-map / information-system — опционально позже.

## Sibling runtime

| Ключ projects-map | Путь (от meta) | Документация / контекст |
|-------------------|----------------|-------------------------|
| `bot` | `../my-study-bot/` | `{bot}/AGENTS.md` — продукт, стек, handlers / DB / scheduler / FSM / tests |

OpenSpec-артефакты: [`openspec/`](../openspec/) в **my-study-bot-meta** (не в sibling). Конфиг: [`openspec/config.yaml`](../openspec/config.yaml).

Always-on инструкции meta: [`AGENTS.md`](../AGENTS.md).

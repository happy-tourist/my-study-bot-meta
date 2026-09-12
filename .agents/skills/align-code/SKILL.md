---
name: align-code
description: >-
  Runs bot-align-code + bot-verify-code for the my-study-bot sibling, then
  prints a combined Bot report. Use when the user asks for align-code, full
  bot align, or to audit the bot against requirements and skill compliance
  in one pass.
---

# Align Code — bot

Оркестратор: для **bot** последовательно выполнить **align-code** и **verify-code**, затем выдать **единый отчёт**.

Скилл **только анализирует и отчитывается** — не правит runtime-код, не коммитит. Правки — только по явной просьбе после отчёта.

## Когда применять

- Пользователь запускает `align-code` / просит полный align по боту
- Нужен один проход: соответствие постановке (**align**) + compliance skills (**verify**) на sibling bot

Не подменять собой одиночные `bot-align-code` / `bot-verify-code`, если пользователь явно ограничил scope одним из них.

## Делегируемые скиллы (обязательно прочитать и следовать)

| Пакет | Align | Verify |
|-------|-------|--------|
| Bot | [`.agents/skills/bot/bot-align-code/SKILL.md`](../bot/bot-align-code/SKILL.md) | [`.agents/skills/bot/bot-verify-code/SKILL.md`](../bot/bot-verify-code/SKILL.md) |

Правила осей, severity, scope diff и формат секций живут **в дочерних скиллах**. Здесь — только оркестрация, пути и сводный отчёт. Не дублировать и не ослаблять дочерние правила.

## Репозитории и пути

Разрешить пути от корня `my-study-bot-meta` через [`docs/projects-map.md`](../../../docs/projects-map.md) (+ `projects-map.local.yaml` если есть):

| Ключ | Репозиторий | Default path | Skills root |
|------|-------------|--------------|-------------|
| `bot` | `my-study-bot` | `../my-study-bot` | `.agents/skills/bot/` |

Если путь не найден — спросить пользователя, не угадывать.

Перед работой по пакету — локальный `{bot}/AGENTS.md`.

## Вход

**Основной источник требований — активный OpenSpec change** в meta (`proposal` / `specs` / `design` / `tasks`). Paste и формулировки из диалога — только уточнения поверх change, не замена.

Принимать (и пробрасывать в bot align):

- имя OpenSpec change (если пользователь назвал)
- опционально: вставленный текст / уточнения из диалога

**Резолв OpenSpec change** (один раз на весь прогон, до Align):

1. Имя, переданное пользователем.
2. Иначе — из контекста диалога, если однозначно.
3. Иначе из meta: `openspec list --json` — автовыбор, если ровно **один** active change.
4. Иначе при нескольких — спросить, какой active change использовать.
5. Иначе при нуле active — спросить имя change или явно подтвердить audit без OpenSpec (`OpenSpec: none`, тогда только paste/диалог/repo).

Объявить: `Using OpenSpec change: <name>` (или `OpenSpec: none`). То же имя передать в bot-align. В дочернем align этот change — **primary evidence** для осей A/C (включая краевые случаи из SC-*/design); paste/диалог не перекрывают артефакты change без явного решения пользователя.

Если active change не резолвится и пользователь не дал fallback — спросить **один раз** до шага Align.

## Hard Boundary

- Не править, не создавать, не удалять, не форматировать runtime-файлы.
- Не коммитить / не пушить.
- Дочерний align-скилл остаётся read-only (не `pip`/`pytest` для preservation analysis). Verify: агент сам запускает `pip` / `python` / `pytest` / Docker из корня bot при необходимости, исправляет сбои; report-only verify (без «исправить») — всё равно запускать tooling и включать результаты в отчёт.
- Align остаётся read-only; verify в этом оркестраторе — **report-only** (не verify-and-fix), если пользователь явно не попросил «исправить».

## Workflow

```
Align-Code Progress:
- [ ] 1. Resolve paths (projects-map + local)
- [ ] 2. Resolve requirements source (shared)
- [ ] 3. Bot: read + run bot-align-code
- [ ] 4. Bot: read + run bot-verify-code
- [ ] 5. Combined report (Bot)
```

### 1–2. Paths and requirements

Из корня meta: projects-map (+ local). Проверить, что bot — git work tree.

Собрать общий источник требований. **Primary = активный OpenSpec change** (резолв как в **Вход**), затем `openspec status --change "…" --json` из meta; артефакты change обязательны для Align осей A/C (краевые случаи / SC-*). Paste/диалог — вторичные уточнения, не основной канон.

### 3–4. Bot

Рабочий cwd / git / чтение `app/…`, `main.py` — **корень bot**. Skills читать из meta `.agents/skills/bot/`.

1. Полностью выполнить `bot-align-code` (все оси, формат отчёта дочернего скилла).
2. Полностью выполнить `bot-verify-code` (ветка vs merge-base + working tree; все Always-include skills).

Сохранить результаты для сводки; **не** публиковать отдельным финальным ответом до шага 5 (допустимы краткие прогресс-апдейты).

### 5. Combined report

Язык сводки: **русский**. Секции Align / Verify — как в дочерних скиллах (не переписывать язык дочернего шаблона).

Выдать **один** отчёт:

```markdown
# Align Code — отчёт

Источник требований: активный OpenSpec change <name> (+ paste/диалог только как уточнения | none + fallback)
OpenSpec change: <name | none; passed | active | inferred>
Scope: bot

## Сводка
| Пакет | Постановка | Код проекта | Тесты | Регрессии | Verify Violations | Verify Warnings |
|-------|------------|-------------|-------|-----------|-------------------|-----------------|
| Bot | N/10 | N/10 | READY\|BLOCKED | нет\|N | N | N |

## Bot

### Align
<полный отчёт bot-align-code>

### Verify
<полный отчёт bot-verify-code>

## Что делать дальше
- <hard / Violations сначала; затем Warnings; Recommendations последними>
- <выполненные pip/python/pytest/Docker-команды bot и результаты, если уместны>
```

Если пакет пропущен (нет пути) — секция с одной строкой `пропущен: <причина>`, без выдуманных findings.

Пустые hard-секции Align опускать по правилам дочернего скилла. Пустые tier-секции Verify — `- none` / `- нет` по дочернему шаблону.

## Не делать

- Не изобретать параллельные критерии align/verify.
- Не считать «чистый» без реального прогона дочерних скиллов (если в пакете есть branch/working-tree изменения).
- Не запускать `commit` / правки docs skills из этого скилла.

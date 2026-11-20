---
name: review-change
description: >-
  Orchestrates full meta review of an active OpenSpec change: bot-brief-scope →
  bot-align-implementability → align-meta — each in its own subagent, each
  followed by «Добавь и поправь что считаешь нужным». Use when the user asks to
  review-change, brief+align a change, or run постановка digest then
  implementability/meta aligns without pausing. Meta artifacts only — does not
  propose, apply, or edit runtime bot code.
---

# Review Change — brief (bot) → align (implementability → meta)

Оркестратор **разбора и выравнивания** уже существующего OpenSpec change: сначала digest постановки (bot), затем реализуемость и качество meta (implementability → meta). Parent **не** пишет brief/align сам: делегирует субагентам и гонит пайплайн до конца.

**Граница:** только meta (`openspec/`, связанные docs/indexes при правках). **Не** запускать `openspec-propose`, `openspec-apply-change`, `implement-change`; не трогать runtime bot.

Одиночные package-скиллы (`bot-brief-scope`, `bot-align-implementability`, `align-meta`) остаются и вызываются напрямую, если нужна одна фаза.

## Когда применять

- Пользователь запускает `review-change` / просит brief+align / «прогони постановку и align» по активному change
- Нужен end-to-end: brief-bot → align-implementability → align-meta (+ правки meta по находкам) без промежуточного review
- Артефакты change уже есть (после propose / update) — этот скилл их **не** создаёт

Не подменять одиночный `bot-brief-scope` / `bot-align-implementability` / `align-meta`, если пользователь явно ограничил одной фазой.

После прерывания — снова `review-change`: читает `.review-change-state.yaml` и резюмирует с сохранённой фазы.

## Жёсткое правило прерывания

**Прервать можно ТОЛЬКО если есть задачи которые невозможно выполнить без решения человека.**

Не останавливаться ради «лучше спросить», мелкой неоднозначности с default, промежуточного OK, удобства parent-агента.

Останавливаться только когда без выбора человека нельзя: нет/несколько active change без контекста; нет артефактов; секрет; конфликт без default; субагент вернул `HUMAN_BLOCKER`.

При стопе — записать state, кратко: что блокирует; после решения — снова `review-change` (resume). Не продолжать следующие фазы.

## State-файл (для resume)

Путь: `openspec/changes/<name>/.review-change-state.yaml` (от корня meta).

```yaml
change: <name>
phase: brief-bot | align-implementability | align-meta | done
blocker: null | "<short reason>"
updated: "<ISO-8601>"
```

- Старт → `phase: brief-bot`
- После brief-bot → `phase: align-implementability`
- После align-implementability → `phase: align-meta`
- После align-meta → `phase: done`, `blocker: null`
- При стопе → текущая `phase` + `blocker`

Не коммитить обязательность этого файла в продуктовый смысл change — оркестраторский маркер.

## Делегируемые скиллы (обязательно прочитать перед фазой)

| Фаза | Skill |
|------|--------|
| 1. Brief bot | [`.agents/skills/bot/bot-brief-scope/SKILL.md`](../bot/bot-brief-scope/SKILL.md) |
| 2. Align implementability | [`.agents/skills/bot/bot-align-implementability/SKILL.md`](../bot/bot-align-implementability/SKILL.md) |
| 3. Align meta | [`.agents/skills/align-meta/SKILL.md`](../align-meta/SKILL.md) |

## Вход / выбор change

1. Имя от пользователя, иначе из контекста диалога.
2. Иначе `openspec list --json` — автовыбор, если ровно **один** active change.
3. Иначе при 0 или >1 без однозначности — **прервать**.

Объявить: `Using change: <name>`.

Убедиться, что `openspec/changes/<name>/` существует и есть базовые артефакты — иначе `HUMAN_BLOCKER` (сначала propose / continue-change).

Пути runtime — через [`docs/projects-map.md`](../../../docs/projects-map.md) (+ local YAML) — субагентам для reuse/evidence, не для правок runtime.

## Workflow

```
Review-Change Progress:
- [ ] 1. Select active change (parent)
- [ ] 2. Brief-bot-scope → субагент (+ правка)
- [ ] 3. Align-implementability → субагент (+ правка)
- [ ] 4. Align-meta → субагент (+ правка)
- [ ] 5. Short final report
```

### 1. Подготовка (parent)

1. Выбрать change + при resume прочитать state.
2. Записать state текущей фазы, `change: <name>`.
3. Кратко передать субагентам: change name, projects-map paths, что уже сделано.

### 2–4. Brief / Align phases (отдельный субагент каждая)

Для **brief-bot**, затем **align-implementability**, затем **align-meta**:

1. Прочитать и выполнить соответствующий skill для этого change.
2. Затем **в том же субагенте** выполнить промпт дословно:

   > Добавь и поправь что считаешь нужным

   По отчёту — правки только в **meta** артефакты change; не ждать подтверждения; **не** runtime.
3. Вернуть parent краткий итог или `HUMAN_BLOCKER`. Обновить state → следующая фаза.

Порядок **фиксирован**: brief-bot → align-implementability → align-meta.

**Override read-only** у brief/align-скиллов: в рамках **этого** оркестратора субагент после отчёта **обязан** применить разумные правки по промпту выше (только meta). Не трогать секреты и не расширять scope за пределы findings. Не запускать propose / apply / commit / archive / implement-change.

### 5. Финальный отчёт (parent)

```markdown
# Review Change — итог

**Change:** <name>
**Brief-bot:** OK | HUMAN_BLOCKER | skipped
**Align-implementability:** OK | HUMAN_BLOCKER | skipped
**Align-meta:** OK | HUMAN_BLOCKER | skipped
**Stopped:** нет | <причина только human-blocker>
**Resume:** снова `review-change` (если Stopped ≠ нет; читает state)
**Next:** review артефактов (человек) → `/opsx-apply` или `implement-change`
```

## Субагенты — общие правила

- Каждый шаг 2–4 — **новый** субагент; не смешивать фазы в одном.
- Передавать полный контекст: change name, map-пути meta/bot, краткий итог предыдущих фаз.
- `run_in_background: false` — ждать результат перед следующей фазой.
- Не резюмировать длинно вывод субагента пользователю между фазами — только прогресс одной строкой.
- Порядок фаз **фиксирован**: brief-bot → align-implementability → align-meta.

## Не делать

- Не пропускать фазы «чтобы быстрее».
- Не запускать propose / apply / implement-change / commit / archive в этом скилле.
- Не править runtime bot.
- Не ослаблять safety (секреты, force push).

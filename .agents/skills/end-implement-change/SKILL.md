---
name: end-implement-change
description: >-
  Closes an active OpenSpec change end-to-end: openspec-update-change →
  openspec-sync-specs → openspec-archive-change. Use when the user asks to
  end-implement-change, finalize/close a change after implementation, or run
  update → sync → archive without pausing for questions.
---

# End Implement Change — update → sync → archive

Оркестратор закрытия **активного** OpenSpec change после реализации. Запускает три скилла **подряд**, без пауз и без вопросов пользователю.

## Когда применять

- Пользователь запускает `end-implement-change` / просит закрыть change после implement
- Нужен end-to-end: update артефактов → sync delta в main specs → archive

Не подменять одиночный `openspec-update-change` / `openspec-sync-specs` / `openspec-archive-change`, если пользователь явно хочет только один шаг.

## Жёсткое правило: без уточнений

**Ничего не спрашивать.** На любой prompt / confirmation / choice из дочерних скиллов — решить самому и продолжить.

Defaults:

| Вопрос дочернего скилла | Решение |
|-------------------------|---------|
| Какой change? | Имя от пользователя → иначе из контекста → иначе единственный active → иначе самый недавно изменённый из `openspec list --json` |
| Coherence / что обновлять в artifacts? | Полный coherence-проход: согласовать существующие артефакты с реализованным состоянием |
| Неполные artifacts при archive? | Продолжить |
| Неполные tasks при archive? | Продолжить |
| Sync delta перед archive? | Уже сделан в фазе 2 → **Archive now** (не Cancel, не «без sync»). Если после фазы 2 ещё есть drift — **Sync now**, затем archive |
| Sync anyway / Cancel? | **Archive now** |

Override «ask the user» / «prompt for selection» / «confirm» из дочерних скиллов **не действует** в этом оркестраторе.

Останавливаться только при жёстком blocker: CLI/IO error, archive target уже существует, sync validation failed и повтор не помог, нет ни одного active change.

## Делегируемые скиллы (обязательно прочитать перед фазой)

| Фаза | Skill |
|------|--------|
| 1. Update | [`.agents/skills/openspec-update-change/SKILL.md`](../openspec-update-change/SKILL.md) |
| 2. Sync | [`.agents/skills/openspec-sync-specs/SKILL.md`](../openspec-sync-specs/SKILL.md) |
| 3. Archive | [`.agents/skills/openspec-archive-change/SKILL.md`](../openspec-archive-change/SKILL.md) |

Правила update/sync/archive живут в дочерних скиллах. Здесь — оркестрация, порядок и auto-answers.

## Вход / выбор change

1. Имя от пользователя, иначе из контекста диалога.
2. Иначе `openspec list --json` — если ровно один active → его.
3. Иначе → самый недавно изменённый (`lastModified`), без вопроса.

Объявить: `Using change: <name>`. Один и тот же `<name>` передать во все три фазы.

## Workflow

```
End-Implement-Change Progress:
- [ ] 1. Select active change
- [ ] 2. openspec-update-change
- [ ] 3. openspec-sync-specs
- [ ] 4. openspec-archive-change
- [ ] 5. Short final report
```

### 1. Update

Прочитать и выполнить `openspec-update-change` для `<name>`:

- Coherence review существующих planning-артефактов (не создавать новые).
- Писать сразу (`rules.update`), без confirmation gate.
- Не править runtime-код.

### 2. Sync

Прочитать и выполнить `openspec-sync-specs` для того же `<name>`:

- Синхронизировать **все** delta specs из `artifactPaths.specs.existingOutputPaths`.
- Intelligent merge в main specs; validate; краткий summary.
- Если delta нет — зафиксировать «no delta» и идти дальше (не стоп).

### 3. Archive

Прочитать и выполнить `openspec-archive-change` для того же `<name>`:

- Incomplete artifacts/tasks → warning в уме, **proceed**.
- Sync assessment: после фазы 2 ожидать already synced → **Archive now**.
- Не Cancel. Не «Archive without syncing», если sync ещё не сделан (сначала sync).
- Inline sync из archive **не** дублировать, если фаза 2 уже успешно смержила все delta; только если verification показывает drift — sync, verify, затем move.
- `mv` change в `archive/YYYY-MM-DD-<name>` (или имя уже с датой — без второго префикса).

### 4. Финальный отчёт

Кратко на русском:

```markdown
# End Implement Change — итог

**Change:** <name>
**Update:** OK | skipped-coherent | failed
**Sync:** OK | no-delta | failed
**Archive:** OK | failed → <path>
**Stopped:** нет | <причина>
```

## Не делать

- Не спрашивать пользователя и не ждать OK между фазами.
- Не пропускать update или sync «чтобы быстрее» (кроме «уже coherent» / «no delta»).
- Не архивировать, пока sync (фаза 2 или inline recovery) не завершён или явно no-delta.
- Не трогать runtime-код в этом скилле.
- Не создавать PR и не коммитить (для commit — отдельный `commit` / `implement-change`).

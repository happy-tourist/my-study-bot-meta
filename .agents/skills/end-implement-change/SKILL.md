---
name: end-implement-change
description: >-
  Closes an active OpenSpec change end-to-end: openspec-update-change →
  openspec-sync-specs → openspec-archive-change → commit. Use when the user
  asks to end-implement-change, finalize/close a change after implementation,
  or run update → sync → archive → commit without pausing for questions.
---

# End Implement Change — update → sync → archive → commit

Оркестратор закрытия **активного** OpenSpec change после реализации. Запускает четыре скилла **подряд**, без пауз и без вопросов пользователю.

## Когда применять

- Пользователь запускает `end-implement-change` / просит закрыть change после implement
- Нужен end-to-end: update артефактов → sync delta в main specs → archive → commit (+ push)

Не подменять одиночный `openspec-update-change` / `openspec-sync-specs` / `openspec-archive-change` / `commit`, если пользователь явно хочет только один шаг.

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
| Commit messages / подтверждение commit? | Составить сообщения по diff (как в `commit`); не ждать OK. Запуск `end-implement-change` = разрешение на stage → commit → push |

Override «ask the user» / «prompt for selection» / «confirm» из дочерних скиллов **не действует** в этом оркестраторе.

Останавливаться только при жёстком blocker: CLI/IO error, archive target уже существует, sync validation failed и повтор не помог, нет ни одного active change, git/auth blocker у `commit`, который нельзя обойти без человека.

## Делегируемые скиллы (обязательно прочитать перед фазой)

| Фаза | Skill |
|------|--------|
| 1. Update | [`.agents/skills/openspec-update-change/SKILL.md`](../openspec-update-change/SKILL.md) |
| 2. Sync | [`.agents/skills/openspec-sync-specs/SKILL.md`](../openspec-sync-specs/SKILL.md) |
| 3. Archive | [`.agents/skills/openspec-archive-change/SKILL.md`](../openspec-archive-change/SKILL.md) |
| 4. Commit | [`.agents/skills/commit/SKILL.md`](../commit/SKILL.md) |

Правила update/sync/archive/commit живут в дочерних скиллах. Здесь — оркестрация, порядок и auto-answers.

## Вход / выбор change

1. Имя от пользователя, иначе из контекста диалога.
2. Иначе `openspec list --json` — если ровно один active → его.
3. Иначе → самый недавно изменённый (`lastModified`), без вопроса.

Объявить: `Using change: <name>`. Один и тот же `<name>` передать в фазы 1–3.

## Workflow

```
End-Implement-Change Progress:
- [ ] 1. Select active change
- [ ] 2. openspec-update-change
- [ ] 3. openspec-sync-specs
- [ ] 4. openspec-archive-change
- [ ] 5. commit (stage → commit → push)
- [ ] 6. Short final report
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

### 4. Commit

Прочитать и выполнить [`commit`](../commit/SKILL.md) целиком (stage → commit → push по dirty-репо экосистемы meta + bot).

- Запуск `end-implement-change` = явное разрешение на commit/push (как у `implement-change`).
- Если оба репо чистые и нет unpushed commits — зафиксировать «commit: nothing to do» и идти к отчёту.
- Если commit/push требует решения человека (чужая ветка, секреты в diff, auth rejected) — остановиться с причиной в отчёте.
- Не ослаблять safety `commit` (force, amend, секреты).

### 5. Финальный отчёт

Кратко на русском:

```markdown
# End Implement Change — итог

**Change:** <name>
**Update:** OK | skipped-coherent | failed
**Sync:** OK | no-delta | failed
**Archive:** OK | failed → <path>
**Commit:** OK | nothing-to-do | failed → <причина>
**Stopped:** нет | <причина>
```

## Не делать

- Не спрашивать пользователя и не ждать OK между фазами.
- Не пропускать update, sync или commit «чтобы быстрее» (кроме «уже coherent» / «no delta» / «nothing to do»).
- Не архивировать, пока sync (фаза 2 или inline recovery) не завершён или явно no-delta.
- Не пропускать фазу commit после успешного archive.
- Не трогать runtime-код в фазах update/sync/archive (runtime коммитится в фазе 4, если уже изменён ранее).
- Не создавать PR/MR в этом скилле.

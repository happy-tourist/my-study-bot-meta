---
name: implement-change
description: >-
  Orchestrates full implementation of an active OpenSpec change: apply task
  blocks via subagents, then align-code, check-changes, and commit. Use when
  the user asks to implement-change, fully implement an active change, or run
  apply → align → check → commit end-to-end without pausing for review.
---

# Implement Change — apply → align → check → commit

Оркестратор полного цикла по **активному** OpenSpec change. Parent **не** пишет runtime-код сам: делегирует шаги субагентам и гонит пайплайн до конца.

## Когда применять

- Пользователь запускает `implement-change` / просит полностью реализовать активный change
- Нужен end-to-end: apply tasks → align → check-changes → commit без промежуточного review

Не подменять одиночный `openspec-apply-change`, если пользователь явно хочет только apply.

После прерывания продолжение — снова запустить этот скилл (`implement-change`): он читает `.implement-change-state.yaml` и резюмирует с сохранённой фазы (не начинать apply с нуля без нужды).

## Жёсткое правило прерывания

**Прервать можно ТОЛЬКО если есть задачи которые невозможно выполнить без решения человека.**

Не останавливаться ради:

- «лучше спросить» / «на всякий случай подтвердить»
- мелкой неоднозначности, если есть разумный default из spec/design/skills
- желания показать промежуточный отчёт и ждать OK
- удобства parent-агента

Останавливаться только когда без выбора человека нельзя продолжить, например: нет/несколько active change без однозначного контекста; секрет/credential, которого нет в env; конфликт требований без default в артефактах; git/auth blocker у `commit`, который нельзя обойти без человека.

При таком стопе — записать state (ниже), кратко: что блокирует, какие варианты, что уже сделано; после решения человека — снова `implement-change` (resume по state). Не продолжать следующие фазы пайплайна.

## State-файл (для resume)

Путь: `openspec/changes/<name>/.implement-change-state.yaml` (от корня meta).

Писать/обновлять **в начале** (после выбора change), **после каждого** завершённого apply-блока и фазы align/check/commit, и **при** `HUMAN_BLOCKER`.

```yaml
change: <name>
phase: apply | align | check | commit | done
apply_block: "<N-or-null>"   # текущий/следующий блок при phase=apply; иначе null
blocker: null | "<short reason>"
updated: "<ISO-8601>"
```

- После успешного блока apply с ещё pending → `phase: apply`, `apply_block` = следующий pending.
- После всего apply → `phase: align`, `apply_block: null`.
- После align → `phase: check`; после check → `phase: commit`; после commit → `phase: done`, `blocker: null`.
- При стопе → оставить текущую `phase` / `apply_block`, заполнить `blocker`.

Не коммитить обязательность этого файла в продуктовый смысл change — это оркестраторский маркер; если попадёт в commit meta вместе с tasks — допустимо.

## Делегируемые скиллы (обязательно прочитать перед фазой)

| Фаза | Skill |
|------|--------|
| 1. Apply | [`.agents/skills/openspec-apply-change/SKILL.md`](../openspec-apply-change/SKILL.md) |
| 2. Align | [`.agents/skills/align-code/SKILL.md`](../align-code/SKILL.md) |
| 3. Check | [`.agents/skills/check-changes/SKILL.md`](../check-changes/SKILL.md) |
| 4. Commit | [`.agents/skills/commit/SKILL.md`](../commit/SKILL.md) |

Правила apply/align/check/commit живут в дочерних скиллах. Здесь — оркестрация, субагенты и override паузы.

## Вход / выбор change

Как в `openspec-apply-change`:

1. Имя от пользователя, иначе из контекста диалога.
2. Иначе `openspec list --json` — автовыбор, если ровно **один** active change.
3. Иначе при 0 или >1 без однозначности — **прервать** (нужно решение человека).

Объявить: `Using change: <name>`.

Пути runtime — через [`docs/projects-map.md`](../../../docs/projects-map.md) (+ local YAML).

## Workflow

```
Implement-Change Progress:
- [ ] 1. Select active change + read openspec-apply-change
- [ ] 2. Status + apply instructions + contextFiles (parent)
- [ ] 3. Apply: каждый блок tasks.md → отдельный субагент
- [ ] 4. Align-code → отдельный субагент (+ правка)
- [ ] 5. Check-changes → отдельный субагент (+ правка)
- [ ] 6. Commit → отдельный субагент
- [ ] 7. Short final report
```

### 1–2. Подготовка (parent)

1. Прочитать `openspec-apply-change` целиком.
2. `openspec status --change "<name>" --json`
3. `openspec instructions apply --change "<name>" --json`
4. Прочитать все `contextFiles`.
5. Если `state: "blocked"` (нет артефактов) — **прервать** (нужен человек / continue-change).
6. Если `state: "all_done"` и незавершённых задач нет — пропустить фазу 3, сразу 4→5→6.
7. Показать кратко: schema, progress N/M, список блоков tasks.

**Override паузы apply:** пункты «Pause if unclear / ask / wait» из apply-скилла **не** действуют в этом оркестраторе, кроме правила прерывания выше. Субагенты apply должны доводить блок до конца или вернуть явный human-blocker.

### 3. Apply — блоки tasks в субагентах

**Блок** = секция `## N. …` в `tasks.md` (и все её `- [ ]` / `- [x]` пункты). Порядок — как в файле.

Для каждого блока, где есть незавершённые `- [ ]`:

1. Запустить **отдельный** субагент (`generalPurpose`, дождаться завершения).
2. В prompt субагента обязательно:
   - прочитать и следовать `openspec-apply-change`;
   - change name, schema, пути `contextFiles`, projects-map;
   - **только этот блок** (номер + заголовок + полный список пунктов блока);
   - реализовать все pending пункты; отмечать `- [x]` сразу после каждого;
   - держать изменения минимальными и в scope блока;
   - **не** спрашивать пользователя; при невозможности без человека — вернуть `HUMAN_BLOCKER: …` и остановиться;
   - в ответе parent: что сделано, какие checkbox обновлены, `HUMAN_BLOCKER` или `OK`.
3. Блоки запускать **последовательно** (следующий после завершения предыдущего) — типичные зависимости bot layers → meta docs/OpenSpec.
4. Если субагент вернул `HUMAN_BLOCKER` — **прервать** весь пайплайн (не align/check/commit).
5. После всех блоков — `openspec instructions apply --change "<name>" --json` (или status): если остались pending без blocker — один retry-проход по оставшимся; иначе продолжить.

Parent **не** реализует задачи сам, кроме микро-фикса checkbox/пути, если субагент явно попросил и это не код продукта.

### 4. Align-code (отдельный субагент)

Один субагент:

1. Прочитать и выполнить [`align-code`](../align-code/SKILL.md) для этого change (передать имя change).
2. Затем **в том же субагенте** выполнить промпт дословно:

   > Добавь и поправь что считаешь нужным

   То есть по отчёту align/verify — внести правки в runtime/docs/skills, которые субагент считает нужными; не ждать подтверждения.
3. Вернуть parent краткий итог: findings + что поправлено, или `HUMAN_BLOCKER`.

### 5. Check-changes (отдельный субагент)

Один субагент:

1. Прочитать и выполнить [`check-changes`](../check-changes/SKILL.md).
2. Затем **в том же субагенте** выполнить промпт дословно:

   > Добавь и поправь что считаешь нужным

   То есть добавить/обновить skills, maps, docs, AGENTS по своим же рекомендациям; не ждать подтверждения.
3. Вернуть parent краткий итог или `HUMAN_BLOCKER`.

Override «только анализ» у align-code / check-changes: в рамках **этого** оркестратора субагент после отчёта **обязан** применить разумные правки по промпту выше. Не трогать секреты и не расширять scope за пределы findings.

### 6. Commit (отдельный субагент)

Один субагент: прочитать и выполнить [`commit`](../commit/SKILL.md) целиком (stage → commit → push по dirty-репо экосистемы).

Если commit/push требует решения человека (чужая ветка, секреты в diff, auth rejected) — `HUMAN_BLOCKER`, прервать.

### 7. Финальный отчёт (parent)

Кратко на русском:

```markdown
# Implement Change — итог

**Change:** <name>
**Apply:** N/M tasks · блоки: …
**Align:** OK | HUMAN_BLOCKER | skipped
**Check-changes:** OK | HUMAN_BLOCKER | skipped
**Commit:** OK | HUMAN_BLOCKER | skipped
**Stopped:** нет | <причина только human-blocker>
**Resume:** снова `implement-change` (если Stopped ≠ нет; читает state)
```

## Субагенты — общие правила

- Каждый шаг 3/4/5/6 — **новый** субагент; не смешивать фазы в одном.
- Передавать полный контекст: change name, абсолютные/map-пути meta/bot, что уже сделано в предыдущих фазах (кратко).
- `run_in_background: false` — ждать результат перед следующей фазой.
- Не резюмировать длинно вывод субагента пользователю между фазами — только прогресс одной строкой, затем следующая фаза.

## Не делать

- Не пропускать align / check-changes / commit «чтобы быстрее».
- Не запускать commit, если apply остановился на `HUMAN_BLOCKER`.
- Не архивировать change и не создавать PR в этом скилле.
- Не ослаблять safety `commit` (force, amend, секреты).

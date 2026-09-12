---
name: commit
description: >-
  Scans uncommitted changes across my-study-bot-meta and my-study-bot; always
  stages, commits on main/master, and pushes. Use when the user runs the commit
  skill or asks to commit across the ecosystem («закоммитить»); push is always
  included — no separate push request needed.
---

# Commit — два репозитория экосистемы

Проверить незакоммиченные изменения в meta и bot, застейджить, закоммитить и **всегда запушить**. Запуск этого скилла = полный цикл `stage → commit → push`. Отдельное слово «пуш» в запросе не требуется.

Рабочие ветки: **main** или **master**.

## Когда применять

Только если пользователь **явно** запускает commit / просит закоммитить по экосистеме (этот скилл). Не вызывать «на всякий случай» после правок кода.

## Репозитории

Разрешить пути от корня `my-study-bot-meta` через [`docs/projects-map.md`](../../../docs/projects-map.md) (+ `projects-map.local.yaml` если есть):

| Ключ | Репозиторий | Default path | Default branch |
|------|-------------|--------------|----------------|
| meta (этот репо) | `my-study-bot-meta` | `.` | `main` |
| `bot` | `my-study-bot` | `../my-study-bot` | `main` |

Если путь не найден — спросить пользователя, не угадывать.

## Safety (жёстко)

- Не менять `git config`.
- Не `--force` / `--force-with-lease` на `main`/`master` без явной просьбы.
- Не `--amend`, rebase `-i`, `reset --hard`, `--no-verify` без явной просьбы.
- Не коммитить секреты: `.env*`, `credentials.json`, ключи, токены. Если пользователь просит — предупредить и исключить.
- **Push обязателен** после успешного commit в каждом dirty-репо (и для репо, где уже есть unpushed commits на `main`/`master`). Не спрашивать отдельно про push.
- Пустой коммит не создавать: репо без изменений и без unpushed commits пропускать.

## Workflow

Копируй чеклист и отмечай прогресс:

```
Commit Progress:
- [ ] 1. Resolve paths
- [ ] 2. Status scan (all 2)
- [ ] 3. Branch check (main/master)
- [ ] 4. Draft messages + confirm if needed
- [ ] 5. Stage → commit (per dirty repo)
- [ ] 6. Push (always)
- [ ] 7. Final status report
```

### 1. Resolve paths

Из корня meta прочитать projects-map (+ local YAML). Убедиться, что каждый корень — git work tree (`git rev-parse --is-inside-work-tree`).

### 2. Status scan (параллельно по двум репо)

В **каждом** репо выполнить:

```bash
git status -sb
git status --porcelain
git diff
git diff --cached
git log -5 --oneline
```

Собрать таблицу:

| Repo | Branch | Dirty? | Ahead/behind | Summary of changes |
|------|--------|--------|--------------|--------------------|

Показать пользователю кратко **до** stage/commit (что будет закоммичено в каком репо). Если оба чистые — остановиться: «нечего коммитить».

### 3. Branch check

Для каждого dirty-репо:

1. `git branch --show-current` → должна быть `main` или `master` (или tracking `origin/main` / `origin/master`).
2. Если ветка другая — **не** коммитить молча: спросить, переключиться / продолжить на текущей / пропустить репо.
3. Meta: допускается первый коммит на пустом `main` (ещё нет `HEAD`).

### 4. Commit messages

- Если пользователь дал одно общее сообщение — использовать его во всех репо с изменениями (допустимо лёгкий префикс `bot:` / `meta:` только если это улучшает ясность).
- Если сообщения нет — для **каждого** dirty-репо составить своё по diff (1–2 предложения, фокус на **why**).
- Стиль: краткий английский или русский — как в недавнем `git log` этого репо; по умолчанию короткое английское описание.
- Не копировать шаблон «Generated with …» / Co-authored-by без просьбы.

Если запрос неоднозначен (какие файлы включать, чужие WIP, секреты) — спросить перед шагом 5. Если нет секретов/чужой ветки — сразу 5 → 6 (push без отдельного подтверждения).

### 5. Stage → commit (по каждому dirty-репо)

Последовательно (не смешивать cwd):

```bash
git add -A
# перепроверить: git status --porcelain / git diff --cached
git commit -m "$(cat <<'EOF'
<message>

EOF
)"
```

- Если hook отклонил коммит — исправить и сделать **новый** commit (не amend).
- Не стейджить ignored-секреты через `git add -f`.

### 6. Push (всегда)

После успешных коммитов — сразу push во всех затронутых репо. Также запушить репо на `main`/`master`, где working tree чистый, но есть unpushed commits (`ahead`).

```bash
git push -u origin HEAD
```

- Если remote tracking уже есть — достаточно `git push`.
- При ошибке auth/rejected — показать вывод, не форсить; спросить пользователя.
- Не пропускать push «потому что пользователь не сказал запушить» — в этом скилле push входит в контракт.

### 7. Final report

Кратко:

```
| Repo | Result | Branch | Commit | Push |
|------|--------|--------|--------|------|
| meta | committed / clean / skipped | … | <short sha or —> | ok / skipped / fail |
| bot | … | … | … | … |
```

## Примеры триггеров

- «Закоммить» / «Сделай commit» / запуск скилла `commit`
- «Проверь незакоммиченные изменения в meta/bot и закоммить»
- «Сделай commit по экосистеме»

Во всех случаях результат: commit (где нужно) **и** push.

## Не делать

- Не создавать PR/MR в этом скилле (отдельный запрос).
- Не мержить ветки и не переключать branch без согласия.
- Не трогать только один репо, если пользователь просил «оба» — просканировать все; коммитить только dirty.

---
name: work-with-env-deploy
description: >-
  Use when changing bot env vars, Docker Compose, VPS deploy, GHCR image push,
  or DB_URL / TG_TOKEN for my-study-bot — .env / .env.server, Dockerfile,
  docker-compose.yml, or .github/workflows/deploy.yml.
---

# Work With Env And Deploy

Use this skill for **env vars**, **Docker/Compose**, and **VPS deploy via GitHub Actions + GHCR** of the Telegram study bot (`my-study-bot`).

Skills path for now: `.agents/skills/bot/` in this repo (canonical copy may later live under `my-study-bot-meta`). Runtime paths below are relative to this bot repo root.

Deploy target: VPS under `/home/deploy/my-study-bot`, Docker Compose + image from GHCR (`ghcr.io/happy-tourist/my-study-bot:latest`). Trigger: push to `main` (or `workflow_dispatch`).

## Hard rules

| Do | Don't |
|----|--------|
| Keep production secrets only on the server (`.env` under `/home/deploy/my-study-bot`) | Commit secrets (`.env`, `.env.server` are gitignored) |
| Document env keys (`.env.example` when added; until then document in `AGENTS.md` / this skill) | Invent CI-injected app secrets — `TG_TOKEN` stays on the VPS `.env` |
| Use GH secrets `SSH_*` only for deploy SSH | Put `TG_TOKEN` in GitHub Actions vars/secrets for the bot process |
| Mount `./data:/app/data` so SQLite survives restarts | Bake `data/` or `.env` into the image (`.dockerignore` excludes them) |
| Keep Windows SSL/IPv4 bypass only in `main.py` `win32` branch | Copy Windows SSL verify-disable into the production Docker image |
| Run local `pip` / `docker` / `python main.py` from bot package root | Skip verification or assume pass without running |

## Env vars

`python-dotenv` loads `.env` from `main.py` / `app/database.py`. Compose injects the same file via `env_file: .env`.

| Variable | Role | Local | Prod (server file) |
|----------|------|-------|--------------------|
| `TG_TOKEN` | Telegram Bot API token (required) | `.env` | `/home/deploy/my-study-bot/.env` |
| `DB_URL` | SQLAlchemy async URL | default `sqlite+aiosqlite:///data/db.sqlite3` | same default; override in server `.env` if needed |

No committed `.env.example` yet — document vars here / in `AGENTS.md`; add `.env.example` when convenient. Do not commit secrets (`.env`, `.env.server` are gitignored).

When adding a new env key:

1. Document it (prefer adding `.env.example` with a comment; until then update `AGENTS.md`).
2. Add to local `.env` and to the VPS `/home/deploy/my-study-bot/.env`.
3. Do **not** rely on GitHub Actions to inject app secrets — update the server `.env` manually (or via secure SSH), then `docker compose up -d` (or recreate the container) so Compose reloads `env_file`.
4. Prefer Compose `env_file` for secrets; do not hardcode tokens in the image or workflow.

## Docker image (`Dockerfile`)

| Setting | Value |
|---------|-------|
| Base | `python:3.13-slim` |
| Extra apt | `ca-certificates` (SSL trust) |
| Deps | `pip install -r requirements.txt` (layer-cached) |
| Workdir | `/app` |
| Data dir | `mkdir -p /app/data` (volume usually mounts over it) |
| CMD | `python main.py` |

`.dockerignore` excludes: `.venv/`, `.git/`, `.github/`, `__pycache__/`, `*.pyc`, `.env`, `data/`, `*.md`.

Linux / Docker / VPS use the clean `Bot(token=…)` branch — do not copy the Windows SSL bypass into the production image.

## Compose (`docker-compose.yml`)

| Setting | Value |
|---------|-------|
| Service | `bot` |
| Image | `ghcr.io/happy-tourist/my-study-bot:latest` |
| Container name | `my-study-bot` |
| Restart | `unless-stopped` |
| Env | `env_file: .env` |
| Volume | `./data:/app/data` (SQLite survives restarts **and** deploys — not wiped by push) |

On container start `init_db()` runs `create_all` then `_ensure_sqlite_user_columns` (DDL in `_SQLITE_USER_COLUMN_DDL` in `app/database.py`). New columns land on the existing VPS file without SSH. Do **not** delete `data/db.sqlite3` in the deploy workflow.

## GitHub Actions deploy

Workflow: `.github/workflows/deploy.yml` (`Deploy to VPS`).

Triggers: `push` to `main`, `workflow_dispatch`.

Image name in workflow: `ghcr.io/${{ github.repository }}:latest` (resolves to `ghcr.io/happy-tourist/my-study-bot:latest` for this repo). Compose pins the same image tag.

### Pipeline (must stay in sync)

1. Checkout → `docker/login-action` to `ghcr.io` with `GITHUB_TOKEN` (packages: write).
2. `docker/build-push-action` context `.` → push `ghcr.io/<github.repository>:latest`.
3. SSH via `appleboy/ssh-action` with secrets: `SSH_HOST`, `SSH_USER`, `SSH_KEY`.
4. Remote: `cd /home/deploy/my-study-bot` → `docker login ghcr.io` (actor + `GITHUB_TOKEN`) → `docker compose pull` → `docker compose up -d` → `docker image prune -f`.

CI does **not** ship `.env` or `data/` — those live only on the VPS. Compose file on the server must already exist (typically checked out / copied once).

### Repo / Actions / VPS checklist

- GitHub secrets: `SSH_HOST`, `SSH_USER`, `SSH_KEY` (workflow also uses `GITHUB_TOKEN` for GHCR).
- On VPS once: Docker + Compose, directory `/home/deploy/my-study-bot` with `docker-compose.yml` and `.env` (`TG_TOKEN`, optional `DB_URL`), writable `./data` for SQLite.
- Confirm `.dockerignore` keeps `.env` and `data/` out of the image.

## Files map

| Concern | Path |
|---------|------|
| Env template | `.env.example` (missing — document vars; add when convenient) |
| Local env | `.env` (gitignored) |
| Prod env (server only) | `/home/deploy/my-study-bot/.env` — not committed with secrets |
| Image build | `Dockerfile`, `.dockerignore` |
| Runtime compose | `docker-compose.yml` |
| CI / GHCR / SSH deploy | `.github/workflows/deploy.yml` |
| Env load / Bot entry | `main.py` |
| DB URL wiring | `app/database.py` |
| Deploy notes | `AGENTS.md` |

## Agent workflow

1. Confirm whether the task is local env, prod secrets on VPS, `DB_URL` / volume, Dockerfile/Compose, or CI/GHCR/SSH.
2. Edit only the files in the map above that the change requires.
3. Keep “secrets on server only” and volume-mounted `data/` unless the user explicitly changes the deploy model.
4. Run local pip/docker verification from the bot package root; fix failures before claiming done. Remote prod deploy / VPS smoke (SSH credentials) — propose steps to the user when the agent cannot execute them.

Typical commands (agent runs locally from bot root):

```bash
pip install -r requirements.txt
python main.py
docker build -t my-study-bot:local .
docker compose config
```

Prod smoke (VPS — propose when agent lacks SSH access):

```bash
cd /home/deploy/my-study-bot
docker compose ps
docker compose logs --tail 50 bot
```

## Anti-patterns

- Committing `.env` / `.env.server` with real `TG_TOKEN`.
- Putting `TG_TOKEN` in GitHub Actions `vars`/`secrets` and expecting the container to see it without a server-side `.env`.
- Removing `.env` or `data/` from `.dockerignore` (leaks secrets / bakes local DB into the image).
- Dropping the `./data:/app/data` volume (SQLite wiped on every recreate).
- Copying Windows `main.py` SSL verify-disable / IPv4 hacks into the Dockerfile or Linux runtime.
- Changing `DB_URL` without ensuring the SQLite path exists and is writable under the mounted `data/` volume on the VPS.
- Assuming a local `python main.py` run updates production (production only changes after GHCR push + compose pull/up).

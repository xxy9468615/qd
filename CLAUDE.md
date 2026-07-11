# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

This repo is the **QD (qiandao)** framework — a Tornado-based HTTP scheduled-task framework (upstream: [qd-today/qd](https://github.com/qd-today/qd)) — with a project-specific template `sophnet.har` (auto check-in for [sophnet.com](https://sophnet.com)) and Railway deployment config layered on top.

QD executes "templates": each template is a HAR file. QD renders it through a **Jinja2 sandboxed** template engine, then runs the HTTP requests in sequence. Each entry has `comment`, `request`, and `rule` (success/failure asserts + variable extraction). Variables extracted from one entry (`{{var}}`) are available to later entries.

## Architecture (big picture — requires reading multiple files)

- **`run.py`** — entry point. Builds the Tornado `Application`, binds port, and starts the worker. Selects worker mode from `config.worker_method` (`Queue` or `Batch`).
- **`worker.py`** — `BaseWorker` + `BatchWorker` (time-based cron) and `QueueWorker` (Redis-backed queue). Runs tasks, clears old logs, handles batch push.
- **`config.py`** — central settings, all read from env vars with defaults. This is the single source of truth for behavior; **change behavior via env vars, not by editing this file**.
- **`web/`** — the Tornado web app (UI for templates / tasks / users / push). `web/app.py` builds `Application`; handlers live under `web/handlers/` (not `web/router`).
- **`libs/`** — core engine: `fetcher.py` (HTTP execution), `funcs.py` (`Cal` for `api://` utilities, `Pusher` for notifications), `safe_eval.py`, `parse_url.py`, `cookie_utils.py`, `mcrypto.py`.
- **`db/`** — SQLAlchemy data layer. `db/__init__.py` exposes `DB` with `.user`, `.tpl`, `.task`, `.tasklog`, `.redis`, `.site`, etc. Backend is **sqlite3 by default**; set `DB_TYPE=mysql` (+ `JAWSDB_MARIA_URL`) to switch.
- **`config/`** — runtime config dir; mounted as a Railway volume (`qd-config`) so settings survive redeploys.

Two execution modes matter: **`Queue`** (default, needs Redis — `redis` class / `REDISCLOUD_URL`) and **`Batch`** (no Redis dependency, periodic `CHECK_TASK_LOOP`). This deployment uses **Batch** (see `.env.railway.example`), so Redis is not required.

## Running

```bash
pip install -r requirements.txt   # or: pipenv install
python run.py                     # serves on $PORT (default 8923); -p N overrides
```

There are no unit tests and no lint/build CI in this repo. Style is enforced by `.flake8` (max-line-length 120, noqa E501). Framework is third-party — prefer upstream conventions over local refactors.

## Deployment (Railway)

- `railway.json` builds `Dockerfile`, starts `python run.py`, mounts `/usr/src/app/config` as the `qd-config` volume, healthcheck `/`.
- `.env.railway.example` → copy to real env. Key overrides for this deployment: `WORKER_METHOD=Batch`, `DISPLAY_IMPORT_WARNING=False`, `USER0ISADMIN=True`, `PORT=80`.
- DB is sqlite3 (the `config` volume persists it). Set `DB_TYPE=mysql` + `JAWSDB_MARIA_URL` only if you switch backends.
- `Dockerfile.ja3` / `Dockerfile.lite` are alternative images (JA3 fingerprinting / minimal); the default `Dockerfile` is what Railway uses.

## The sophnet.har template

Custom contribution. Flow is 4 sequential steps:

1. **GET /api/sys/user/me** — extract `user_name`.
2. **GET /api/sys/checkin/welfare** — extract stats (`todayCheckedIn`, `continuousDays`, `totalDays`, `tokenBalance`, `todayReward`).
3. **POST /api/sys/checkin/do** — perform check-in; **both 200 and 422 are success** (422 = already checked in today).
4. **POST api://util/unicode** — renders the human-readable log line via Jinja2 inside the `data` field.

### Template gotchas (non-obvious, learned the hard way)

- **Auth uses `Authorization: Bearer {{access_token}}`** (JWT) instead of raw cookies, to avoid Jinja2 escaping issues with URL-encoded cookie values. Extra cookies (`_c_WBKFRo`, `Hm_lvt`, `HMACCOUNT`, `acw_tc`, `authorized-token`, `multiple-tabs`, `Hm_lpvt`) are stored as separate QD custom variables.
- **QD's Jinja2 sandbox parses EVERY header/cookie/body value.** A `{%` sequence inside URL-encoded JSON causes `TemplateSyntaxError`. Never embed raw cookie/JSON strings containing `{%`.
- **422 from `checkin/do` is expected and must be treated as success** (already checked in).
- **Do NOT extract variables from the check-in POST response** — its 422 body carries stale `dailyReward` values that overwrite the correct welfare data extracted in step 2.
- Login uses 阿里云 smart captcha (slider); it must be done manually in a browser. The token expires ~14 days — re-extract `access_token` and the cookie variables and update them when it does. They are QD custom variables, not secrets in this repo.

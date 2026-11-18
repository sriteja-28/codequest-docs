# Local development

One command sets everything up, one command runs it. Works on Windows
(`scripts/dev.ps1`) and macOS/Linux (`scripts/dev.sh`); both scripts have the
same commands.

## Prerequisites

| Tool | Version | Notes |
|---|---|---|
| Python | 3.10+ | `py -3` / `python` / `python3` |
| Node.js | 20+ | Next.js 16 needs 20.9 or newer |
| Docker | Desktop or Engine with Compose v2 | must be running |
| RAM | ~4 GB free | the Next.js dev server alone can use 1-2 GB |

## First run

```powershell
# Windows (PowerShell)
.\scripts\dev.ps1 setup
.\scripts\dev.ps1 dev
```

```bash
# macOS / Linux
bash scripts/dev.sh setup
bash scripts/dev.sh dev
```

`setup` is safe to re-run. It:

1. checks Python, Node and Docker;
2. creates `backend/.env`, `judge/.env` and `frontend/.env.local` from the
   `.example` files (random `SECRET_KEY`, one random judge key shared by backend
   and judge); existing files are never overwritten;
3. creates or reuses the virtualenvs (`backend/.venv`, or an existing
   `backend/leet_venv`; `judge/venv`) and installs dependencies;
4. starts the dev infrastructure, applies migrations, seeds sample data;
5. builds the four judge language images if they are missing.

`dev` starts the infrastructure plus three processes (logs in `.dev/logs/`):

| Process | URL |
|---|---|
| Frontend (Next.js dev server) | http://localhost:3000 |
| API (uvicorn, auto-reload) | http://localhost:8000/api/ (health: `/api/health/`) |
| Django admin | http://localhost:8000/django-admin/ |
| Judge worker | consumes the RabbitMQ queue, runs code in Docker |
| RabbitMQ management UI | http://localhost:15673 (codequest / codequest_dev) |

Always open the app as **localhost**, never 127.0.0.1: sign-in cookies belong
to one host name.

## Everyday commands

| Command | What it does |
|---|---|
| `dev` | start everything (`-NoWorker` / `-NoFrontend` on Windows, `NO_WORKER=1` / `NO_FRONTEND=1` on bash) |
| `stop` | stop the processes and containers; **data is kept** |
| `stop frontend` | stop one process (`backend`, `worker`, `frontend`) |
| `status` | containers, processes and the health check |
| `logs backend -Follow` | stream one log (`logs backend -f` on bash) |
| `test` | backend tests, judge tests, frontend type-check and lint |
| `migrate`, `seed` | apply migrations / load sample data (idempotent) |
| `manage <args>` | any `manage.py` command, e.g. `manage grant_role you@example.com ADMIN` |

## Dev infrastructure

`infra/docker-compose.dev.yml` (compose project `codequest-dev`) runs only the
backing services. Ports are deliberately not the defaults, so the stack can run
next to databases you already have, and everything binds to 127.0.0.1:

| Service | Address | Credentials |
|---|---|---|
| Postgres 16 | 127.0.0.1:5433, db `codequest` | codequest / codequest_dev |
| Redis 7 | 127.0.0.1:6380 | none |
| RabbitMQ 3.13 | 127.0.0.1:5673 (UI :15673) | codequest / codequest_dev |

These are development-only credentials. Production uses its own compose file
and secrets (see `docs/DEPLOYMENT.md`).

## Function-only problems

The seeded problems use the function-only judge: open one and the editor shows
just `class Solution` (or a bare JavaScript function). There is no `main`, no
stdin parsing and nothing to print; the judge calls your function with the test
case and compares the returned value.

- Admin-editable test-case inputs are JSON objects keyed by parameter name, e.g.
  `{"nums": [2, 7, 11, 15], "target": 9}`. The sample panel renders that as
  `nums = [2, 7, 11, 15], target = 9`.
- A problem with an empty `function_signature` keeps the legacy stdin/stdout
  behaviour, so mixed catalogues work while problems are migrated.
- Databases seeded before this change need
  `.\scripts\dev.ps1 manage seed_data --refresh-starters` (or
  `bash scripts/dev.sh manage seed_data --refresh-starters`). It rewrites the
  five seeded problems' signatures, starters and test inputs in place.
- Saved editor code is versioned per problem mode: pre-function-mode
  stdin/stdout code is not reused for function-mode problems.

## Contest lifecycle

- Author new contest problems **unpublished** and add them through the contest's
  Django admin inline. Statements open through the contest when it starts.
  Registered users submit for points; anyone signed in can run sample cases.
- After the end, finalization freezes standings and releases eligible problems
  into practice. Public contests default to `release_problems=true`; private
  contests and opt-outs stay unpublished. A shared problem still used by another
  upcoming/live round is held until that round is finalized, then released.
- Judging may continue for up to `CONTEST_FINALIZE_GRACE_SECONDS` after end
  (defaults to `JUDGE_STALE_SUBMISSION_SECONDS`, normally 900). Reaped jobs can
  recover during this window; verdicts after finalization never alter final ranks.
- API requests finalize lazily. For predictable timing, schedule the command
  below periodically (e.g. every minute) using your OS/service scheduler:

```powershell
.\scripts\dev.ps1 manage finalize_contests --dry-run
.\scripts\dev.ps1 manage finalize_contests
# One ended contest, without waiting for the remaining judging:
.\scripts\dev.ps1 manage finalize_contests --slug weekly-contest-1 --force
```

On macOS/Linux use `bash scripts/dev.sh manage finalize_contests` with the same
options. This is idempotent; a Django admin action also supports force-finalizing.
The schedule, mappings and standings are read-only after finalization.

Change **Contest recent days** on the Platform settings row in Django admin to
adjust the Recent/Archive split (1–365, default 30 days). Virtual contests are a
coming-soon entry point.

## Destructive commands (they ask you to type a word first)

| Command | Effect |
|---|---|
| `reset-data` | deletes the dev stack's volumes (Postgres, Redis, RabbitMQ data), then migrates and seeds again. Type `RESET`. |
| `cleanup-old` | removes the containers of the **old** compose setup (project `infra`: `codequest_postgres`, `codequest_redis`, `codequest_rabbitmq`). Type `REMOVE`. Their volumes (`infra_*_data`) are kept unless you add `-IncludeVolumes` (`INCLUDE_VOLUMES=1`) and confirm again with `DELETE-VOLUMES`. |

Nothing outside the `codequest-dev` project and the processes recorded in
`.dev/pids` is ever stopped or deleted.

## Using a different database

The defaults point at the dev stack. To use another Postgres (for example one
you already run on 5432), set `DATABASE_URL` in `backend/.env`. **Back that
database up first**, then run `migrate`: the app needs its latest migrations.

## Sign-in during development

Production sign-in is Google or GitHub OAuth only; there is no public password
sign-up or password login. A provider button appears on the login page when its
`*_CLIENT_ID` and `*_CLIENT_SECRET` are set in `backend/.env`.

For local multi-user testing, development settings enable `DEV_PASSWORD_AUTH`.
The login page then shows a **Development accounts** panel and
`POST /api/auth/dev-login/` accepts the seeded accounts:

| Account | Password | Role |
|---|---|---|
| `admin@codequest.dev` | `admin123` | Admin (also `/django-admin/`) |
| `pro@codequest.dev` | `user123` | Pro |
| `free@codequest.dev` | `user123` | Free |

The endpoint answers 404 whenever `DEV_PASSWORD_AUTH` is false, and production
settings force it off. To test the production path locally, set
`DEV_PASSWORD_AUTH=false` in `backend/.env` and restart the backend.

To use the real providers, set the client IDs/secrets in `backend/.env` and
register these callback URLs in each provider console (no code changes):

- Google: `http://localhost:3000/auth/callback/google`
- GitHub: `http://localhost:3000/auth/callback/github`

Use Google's OAuth consent screen in testing mode and GitHub's OAuth App
settings. If a provider is not configured its button stays disabled with a
short explanation.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Port 5433 is already used` | another program uses that port; stop it or set `CQ_DEV_POSTGRES_PORT` (and `DATABASE_URL`) |
| `Port 8000/3000 is in use` | a server you started yourself is running; stop it (the script never kills processes it didn't start) |
| Submissions stay "Queued" | `status`: the worker must be running and Docker images built (`setup` builds them) |
| "The paging file is too small" (Windows) | memory is exhausted; close apps or stop the frontend (`stop frontend`) |
| Seed prints garbled characters | run through the scripts (they force UTF-8 output) |

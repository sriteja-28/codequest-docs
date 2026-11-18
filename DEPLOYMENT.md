# Deployment

How to take CodeQuest to a single server with Docker Compose and Caddy. The
stack is provider-agnostic: any Linux box (or VM) with Docker and two open
ports works — a small VPS, a home server, or a cloud instance.

## What you deploy

| Piece | Runs as | Notes |
|---|---|---|
| Caddy | container | the only thing exposed: ports 80 + 443, automatic HTTPS |
| Frontend | container | Next.js standalone server (same-origin) |
| Backend | container | uvicorn (Django ASGI: HTTP + WebSocket), 2 workers |
| `migrate` | one-shot | schema migrations + admin static files |
| `finalizer` | container | runs `finalize_contests` every minute (idempotent) |
| Judge worker | container | consumes the queue, runs code via the host's Docker |
| Postgres / Redis / RabbitMQ | containers | internal only, data in named volumes |

Everything except Caddy sits on one internal Docker network with no host
ports. Volumes are prefixed `codequest-prod_*`, so the stack never touches the
`codequest-dev` development volumes.

## Prerequisites

| Tool | Notes |
|---|---|
| Docker Engine + Compose v2 | the only hard requirement (`docker compose version`) |
| 2 GB RAM minimum | 4 GB recommended: the worker runs Docker-in-Docker-style sandbox containers |
| DNS | an `A`/`AAAA` record for your domain pointing at the server |
| Ports 80 + 443 | open inbound (Caddy uses them for Let's Encrypt + HTTP/3) |

The judge language images (python/cpp/java/js) are built by the same compose
file, so nothing else is needed.

## First deployment

```bash
cd infra
cp .env.production.example .env.production
# edit .env.production: fill in every value (secrets, your domain, OAuth ids)
```

`infra/.env.production` is the single source of truth: Compose reads it for
variable interpolation **and** injects it into the backend and judge
containers. The `DJANGO_SETTINGS_MODULE=config.settings.production` that makes
Django enforce every `[PROD-REQUIRED]` check is set by the compose file.

Generate secrets:

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(48))"   # SECRET_KEY, JUDGE_INTERNAL_API_KEY
```

Remember to change the example passwords in **both** places they appear
(`POSTGRES_PASSWORD` + the `DATABASE_URL` line; `RABBITMQ_PASSWORD` + the
`RABBITMQ_URL` line — the example file notes where).

Then build and start:

```bash
cd infra
docker compose --env-file .env.production -f docker-compose.prod.yml build
docker compose --env-file .env.production -f docker-compose.prod.yml --profile images build
docker compose --env-file .env.production -f docker-compose.prod.yml up -d
```

The `--profile images` build creates the four judge language images
(`codequest-python`, `codequest-cpp`, `codequest-java`, `codequest-js`) on the
host's Docker daemon — the worker needs them through the mounted socket.
Re-run that build after pulling judge changes; it is a no-op when unchanged.

On first boot: `migrate` applies migrations and fills the static-files volume,
`backend` waits for it, Caddy obtains certificates for `CQ_DOMAIN` (a public
domain needs working DNS first; `CQ_DOMAIN=localhost` gets a local certificate
and one browser warning).

Watch the first boot:

```bash
docker compose --env-file .env.production -f docker-compose.prod.yml logs -f migrate backend caddy
```

## Post-deploy checklist

```bash
# every service healthy (or migrate exited 0)
docker compose --env-file .env.production -f docker-compose.prod.yml ps

# through Caddy on the real origin
curl -s https://<your-domain>/api/health/
# {"database":"ok","cache":"ok","broker":"ok",...}

curl -sI https://<your-domain>/ | head -n1        # 200, the Next.js landing page
```

Then create the first admin (sign-in is Google/GitHub for the public, but
staff can use the Django password form):

```bash
docker compose --env-file .env.production -f docker-compose.prod.yml \
  exec backend python manage.py createsuperuser
```

Optional sample data (skip for a real launch): `... exec backend python
manage.py seed_data` — with production settings it refuses unless you add
`--force` (seed users have public passwords; only for disposable databases).

Register OAuth callbacks with each provider (from `.env.production` values):

- Google: `https://<your-domain>/auth/callback/google`
- GitHub: `https://<your-domain>/auth/callback/github`

A provider button appears on the login page once its id **and** secret are set;
otherwise it shows "Not configured on this server".

## Maintenance

| Task | Command |
|---|---|
| Logs | `docker compose ... logs -f backend` (`judge-worker`, `frontend`, `caddy`) |
| Migrations after an image update | re-run: they run in the `migrate` service on every `up` |
| Update the app | `git pull`, then the three commands above again |
| Django admin | `https://<your-domain>/django-admin/` (staff password sign-in) |
| RabbitMQ UI | not published; use an SSH tunnel if needed (see compose file comment) |
| Backups | see below |

**Database backup** (the only stateful data that matters day to day):

```bash
docker compose --env-file .env.production -f docker-compose.prod.yml \
  exec postgres pg_dump -U codequest codequest | gzip > backup-$(date +%F).sql.gz
```

Restore: `gunzip -c backup.sql.gz | docker compose ... exec -T postgres psql -U codequest codequest`.

RabbitMQ/Redis data live in volumes (`codequest-prod_rabbitmq`,
`codequest-prod_redisdata`) — snapshot them or rely on the app's tolerance
(unjudged submissions are reaped to INTERNAL_ERROR; Redis only holds cache and
sessions, which re-authenticate).

**Contest finalization** is automated by the `finalizer` service (every
minute, idempotent). No host cron needed.

## Resource notes

- The worker runs untrusted user code inside sandbox containers limited to
  1 CPU / 256 MB per test (see `judge/.env.example`). Give the host enough
  headroom for concurrent runs; scale by adding workers — RabbitMQ
  round-robins jobs, and duplicate deliveries are safe (the backend applies
  the first verdict only).
- `backend` runs 2 uvicorn workers handling HTTP and WebSocket together;
  raise `--workers` in `infra/docker-compose.prod.yml` for bigger loads
  (Redis is already the shared channel layer, cache and session store).
- The whole stack needs no root after install: Caddy owns ports 80/443 via
  the container's own capabilities. The judge worker container runs as root
  only to access the Docker socket.

## Local smoke test of the production stack

You can rehearse the full deployment on your own machine with throwaway
secrets — same commands, `CQ_DOMAIN=localhost` in `.env.production`, then open
`https://localhost` (one browser warning for the local certificate). On Docker
Desktop for Windows also set the `SANDBOX_*` values as shown in
`.env.production.example`, or the judge cannot run code. Tear it down
completely (including its volumes) with:

```bash
docker compose --env-file .env.production -f docker-compose.prod.yml down -v
```

`down -v` removes only `codequest-prod_*` volumes; development data in
`codequest-dev_*` is untouched.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `migrate` exits non-zero | `docker compose ... logs migrate` — usually a wrong `DATABASE_URL` password |
| Caddy gets no certificate | DNS must point at this server and 80/443 must be reachable; check `docker compose ... logs caddy` |
| Sign-in buttons say "Not configured" | set the provider id **and** secret in `.env.production`, then restart: `docker compose ... up -d backend` |
| Submissions stay "Queued" | is `judge-worker` running, and do the four `codequest-*` language images exist on the host (`docker images`)? Rebuild with `--profile images build` |
| Judge health checks fail with `cannot open /code/.cq_run.sh` | the sandbox payload dir is not host-visible: `SANDBOX_DIR` must be a path the Docker daemon resolves on the host (Linux defaults are correct; see `SANDBOX_*` in `.env.production.example`) |
| `503 feature_disabled` / `maintenance` | a platform feature or maintenance mode is on — Platform settings in `/django-admin/` |
| OAuth redirect mismatch | the provider console callback must be exactly `https://<your-domain>/auth/callback/<provider>` |
| 413 / payload errors | Caddy's default request body limit is generous; submissions are capped by `MAX_CODE_BYTES` anyway |

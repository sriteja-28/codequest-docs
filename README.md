# CodeQuest: coding practice platform

A self-hostable platform for practising algorithms and data structures:

- online judge: function-only editor in Python, C++, Java and JavaScript, sandboxed in Docker
- practice lists, courses, progress tracking, daily problem, contests with an archive
- Google / GitHub sign-in only (no passwords); mock providers for local testing
- admin dashboard and Django admin: content changes never need code changes

## Architecture

```
 Browser ──HTTP/WS──▶ Next.js (frontend)        Django API (DRF + Channels) ◀── judge callbacks
    │                                            │        │         │
    └──────────── /api, /ws ─────────────────────┘     Postgres    Redis (cache, sessions, WS)
                                                         │
                                          RabbitMQ ◀─────┘ submission jobs ──▶ judge worker ──▶ Docker sandbox
```

In production one reverse proxy serves the frontend, `/api` and `/ws` on a
single domain (same origin), so cookies, CORS and WebSockets need no special
configuration.

## Documentation

| Topic | File |
|---|---|
| Local development (setup, daily commands, troubleshooting) | [LOCAL_SETUP.md](LOCAL_SETUP.md) |
| Deployment (production compose, reverse proxy, OAuth apps) | [DEPLOYMENT.md](DEPLOYMENT.md) |
| REST / WebSocket API | [api.md](api.md) |
| System design notes | [architecture.md](architecture.md) |

## Quick start

```powershell
.\scripts\dev.ps1 setup    # Windows   (macOS/Linux: bash scripts/dev.sh setup)
.\scripts\dev.ps1 dev      #           (bash scripts/dev.sh dev)
```

Then open http://localhost:3000. Details in [LOCAL_SETUP.md](LOCAL_SETUP.md).

## Judge contract

The worker fetches test cases and posts verdicts through the backend's
internal API (shared `JUDGE_INTERNAL_API_KEY`), so it needs no database
credentials. Backend and judge must be deployed together: the job and callback
formats are versioned with the worker. Jobs failing with infrastructure errors
are requeued up to `MAX_JOB_RETRIES` times, then dead-lettered to
`<JUDGE_QUEUE_NAME>.dead`.

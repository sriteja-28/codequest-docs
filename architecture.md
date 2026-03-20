# CodeQuest — Architecture

## Service Map

```
                        ┌──────────────────────────────────┐
                        │          Users / Browser          │
                        └──────────────┬───────────────────┘
                                       │ HTTPS / WSS
                        ┌──────────────▼───────────────────┐
                        │     Frontend (Next.js 14)         │
                        │  Port 3000 | App Router | TS      │
                        │  TanStack Query + Zustand          │
                        │  Monaco Editor + Recharts          │
                        └──────────────┬───────────────────┘
                                       │ REST + WebSocket
                        ┌──────────────▼───────────────────┐
                        │    Backend (Django + Channels)    │
                        │  Port 8000 | DRF | JWT cookies    │
                        │  ASGI via uvicorn / daphne        │
                        └──┬────────┬──────────┬───────────┘
                           │        │          │
          ┌────────────────▼──┐  ┌──▼────┐  ┌─▼────────────────┐
          │   PostgreSQL 16   │  │ Redis │  │    RabbitMQ 3.13  │
          │   Port 5432       │  │  6379 │  │    Port 5672      │
          │   Primary DB      │  │Channel│  │    Job Queue      │
          │                   │  │Layer  │  │                   │
          └───────────────────┘  └───────┘  └────────┬─────────┘
                                                       │ AMQP
                                          ┌────────────▼─────────┐
                                          │   Judge Worker(s)     │
                                          │   Python process(es)  │
                                          │   Spawns per-lang     │
                                          │   Docker containers   │
                                          └──────────────────────┘
```

## Services

### frontend/
**Tech:** Next.js 14 (App Router), TypeScript, Tailwind CSS, TanStack Query, Zustand, Monaco Editor, Recharts

**Responsibilities:**
- All user-facing UI (problems, submissions, contests, discussions)
- Admin UI under `/admin` (separate layout, ADMIN role guard)
- No business logic — all rules enforced server-side
- Talks to backend via REST API (axios with cookie auth) and WebSocket (native + reconnecting wrapper)
- Ad slots rendered via layout API; Pro users always receive empty placement list

**Env vars:** `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_WS_URL`

---

### backend/
**Tech:** Django 5, Django REST Framework, Django Channels, daphne/uvicorn, JWT (HTTP-only cookies)

**Apps:**
| App | Purpose |
|-----|---------|
| `users` | Custom AbstractBaseUser, JWT cookie auth, plan/role logic |
| `billing` | Plan change audit log, admin plan override, Stripe hook stubs |
| `content` | AdPlacement, AdCreative, FeatureFlag — CMS for ads and flags |
| `problems` | Problem, Section, Tag, TestCase, Solution — full CRUD with admin endpoints |
| `submissions` | Submission lifecycle, RabbitMQ publisher, judge callback receiver, WS broadcast |
| `contests` | Contest, ContestProblem, ContestParticipant, leaderboard broadcast |
| `discussions` | DiscussionThread, DiscussionComment, flag/moderation endpoints |
| `ai_assist` | Hint, explanation, chat, complexity — mocked service classes, Redis usage counters |
| `admin_panel` | Aggregate stats endpoints consumed by admin UI |

**Auth flow:**
1. POST `/api/auth/login/` → sets `access_token` + `refresh_token` HTTP-only cookies
2. All subsequent requests include cookies automatically
3. DRF reads `access_token` cookie via `CookieJWTAuthentication`
4. On 401, frontend intercepts and POSTs `/api/auth/refresh/` for silent refresh

**Env vars:** `SECRET_KEY`, `DATABASE_URL`, `REDIS_URL`, `RABBITMQ_URL`, `JUDGE_INTERNAL_API_KEY`, `ALLOWED_HOSTS`, `CORS_ORIGINS`

---

### judge/
**Tech:** Python, pika (AMQP), docker SDK, psycopg2

**Files:**
| File | Purpose |
|------|---------|
| `worker.py` | RabbitMQ consumer loop; one process = one worker; scale by running N copies |
| `runner.py` | Fetches test cases, runs each through sandbox, posts callback to Django |
| `sandbox.py` | Wraps Docker SDK; enforces CPU/memory/network/PID limits per container |
| `comparator.py` | Normalises and compares actual vs expected output |
| `languages/*/Dockerfile` | Minimal images for Python, C++, Java, JavaScript |

**Job flow:**
```
Django  ──AMQP publish──▶  RabbitMQ queue
                                 │
                                 ▼ AMQP consume (prefetch=1)
                           Judge worker
                                 │
                           ┌─────▼──────────────────────────┐
                           │  For each test case:           │
                           │  1. Write code to temp dir     │
                           │  2. docker run (isolated)      │
                           │  3. Compare stdout vs expected │
                           │  4. Collect time + memory      │
                           └─────────────────────────────────┘
                                 │
                           POST /api/submissions/{id}/judge-callback/
                           (signed with X-Judge-Api-Key header)
                                 │
                           Django updates DB + broadcasts via Redis Channels
                                 │
                           WebSocket push → browser
```

**Security constraints per container:**
- `--network none` (no internet)
- `--read-only` filesystem + `/tmp` tmpfs (64 MB, noexec)
- `--memory` + `--memswap-limit` (configurable, default 256 MB)
- `--pids-limit 64` (no fork bombs)
- `--cpu-quota` derived from CPU time limit
- `user=nobody` (non-root)

**Env vars:** `DATABASE_URL`, `RABBITMQ_URL`, `BACKEND_API_URL`, `JUDGE_INTERNAL_API_KEY`, `CPU_TIME_LIMIT`, `MEMORY_LIMIT_MB`

---

### infra/
`docker-compose.yml` runs PostgreSQL 16, Redis 7, RabbitMQ 3.13 for local development.
In production, replace with managed services (see Scaling section).

---

## Data Flows

### Flow 1: View a Problem
```
Browser → GET /api/problems/{slug}/
Django: ProblemViewSet.retrieve()
  → checks is_published=True
  → serializes Problem + sample test cases + tags + section
  → for PRO solutions, checks user.is_pro via SolutionSerializer
  → returns JSON
Browser: renders ReactMarkdown statement, Monaco editor pre-populated with starter code
```

### Flow 2: Submit a Solution (full path)
```
Browser → POST /api/submissions/ {problem_slug, language, code}
Django:
  1. Creates Submission (status=QUEUED)
  2. Increments problem.total_submissions
  3. publisher.publish_submission_job() → RabbitMQ (durable message)
  4. Returns {submission_id, status="QUEUED"}

Browser:
  1. Opens WebSocket ws/submissions/{id}/
  2. Polls useSubmission hook (refetchInterval=1.5s as fallback)

Judge worker:
  1. AMQP consume → job payload
  2. POST callback: status=RUNNING
  3. Fetches test cases from DB
  4. For each test case → Sandbox.run() → Docker container
  5. Compares output → determines per-TC status
  6. POST callback to Django with full results

Django (judge-callback):
  1. Updates Submission.status, runtime_ms, memory_kb
  2. Creates SubmissionResult rows
  3. If ACCEPTED: updates problem.accepted_submissions + UserSolvedProblem
  4. Channels group_send to ws group "submission_{id}"

Browser WebSocket handler: updates UI status in real-time
```

### Flow 3: Contest Leaderboard (live)
```
Admin/judge updates ContestParticipant.final_score (or scoring worker does)
  → calls _broadcast_leaderboard(contest_slug)
  → Channels group_send to "leaderboard_{slug}"

All browsers on /contests/{slug}/leaderboard:
  → LeaderboardConsumer receives event
  → sends JSON to each WebSocket connection
  → React state updated → re-render
```

### Flow 4: AI Hint request
```
Browser → POST /api/ai/hint/ {problem_slug, hint_level, user_code}
Django:
  1. check_and_increment_ai_limit() → Redis INCR with 24h TTL
  2. If FREE user and limit exceeded → 429 with upgrade prompt
  3. HintService.get_hint(problem, code, level) → returns hint JSON
  4. [TODO: replace with OpenAI/Anthropic API call]
Browser: displays hint in AiPanel
```

---

## Plan Enforcement

All restrictions are **server-side only**. Frontend only reads `user.plan`/`user.is_pro` for UX hints:

| Feature | FREE | PRO | Where enforced |
|---------|------|-----|----------------|
| AI Hints | 5/day | 50/day | `check_and_increment_ai_limit()` in Redis |
| AI Chat | 10/day | 200/day | Same |
| PRO solutions | Preview only | Full | `SolutionSerializer.get_is_locked()` |
| Ads | Shown | Hidden | `LayoutPlacementsView` returns `[]` for PRO |
| Contests | Open | Open | No restriction currently |

---

## Scaling to Cloud

### Recommended architecture (AWS example)

```
Route 53 → CloudFront → S3 (Next.js static export)
                      → ALB
                            → ECS Fargate: backend (N replicas)
                            → ECS Fargate: daphne/channels (M replicas, sticky sessions for WS)

RDS PostgreSQL (Multi-AZ) ← backend
ElastiCache Redis (cluster) ← backend Channels + cache
Amazon MQ (RabbitMQ) ← backend publisher + judge workers
ECR (Docker images for judge languages)
ECS Tasks (judge workers, auto-scaled on queue depth via CloudWatch)
```

### Scaling each service

**Frontend:**
- Export as static site to S3 + CloudFront, or use Vercel.
- Zero scaling needed — CDN handles it.

**Backend (REST):**
- Run N replicas behind ALB.
- Stateless (JWT in cookies, sessions in Redis).
- Scale up reads by adding read replicas to RDS.

**Backend (WebSockets):**
- Django Channels uses Redis channel layer — all nodes share pub/sub.
- Need sticky sessions on ALB for the WS upgrade handshake duration (or use connection tokens).
- Scale independently from REST if needed.

**Judge workers:**
- Pure consumers — run as many as needed.
- Auto-scale based on RabbitMQ queue depth.
- Each worker needs Docker daemon access — use EC2 with Docker or Fargate privileged mode.
- Isolate on dedicated node group (no other workloads share the host).

**Database:**
- Use connection pooling (PgBouncer) in front of RDS.
- Add read replica for analytics/admin queries.

**Queue:**
- Amazon MQ (managed RabbitMQ) or CloudAMQP.
- Use separate queues for priority levels if needed.
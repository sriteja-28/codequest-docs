# CodeQuest — Architecture

## Service Map

```
                        ┌──────────────────────────────────┐
                        │          Users / Browser          │
                        └──────────────┬───────────────────┘
                                       │ HTTPS / WSS
                        ┌──────────────▼───────────────────┐
                        │     Frontend (Next.js 16)         │
                        │  Port 3000 | App Router | TS      │
                        │  TanStack Query + Zustand          │
                        │  Monaco Editor + Recharts          │
                        │  next-themes (light + dark)        │
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
          │   Primary DB      │  │Cache +│  │    Job Queue      │
          │                   │  │Channel│  │                   │
          └───────────────────┘  └───────┘  └────────┬─────────┘
                                                       │ AMQP
                                          ┌────────────▼─────────┐
                                          │   Judge Worker(s)     │
                                          │   Python process(es)  │
                                          │   Spawns per-lang     │
                                          │   Docker containers   │
                                          └──────────────────────┘
```

Ports above are production defaults. The local dev stack binds alternate ports
to protect any pre-existing services: Postgres `127.0.0.1:5433`, Redis `6380`,
RabbitMQ `5673` (UI `15673`). See `docs/LOCAL_SETUP.md` and `scripts/dev.ps1`.


## Services

### frontend/
**Tech:** Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS 4, TanStack Query, Zustand, Monaco Editor, Recharts, next-themes

**Responsibilities:**
- All user-facing UI (home progress, problems, practice lists, courses, submissions, contests, discussions)
- Admin UI under `/admin` (separate layout, ADMIN role guard)
- No business logic — all rules enforced server-side
- Talks to the backend through a same-origin `/api` proxy (cookies stay first-party) and WebSocket (native + reconnecting wrapper)
- Ad slots rendered via layout API; Pro users always receive empty placement list
- Light and dark themes from shared design tokens; theme choice persists client-side

**Routes:**
| Route | Page |
|---|---|
| `/` | Landing for visitors (server-rendered); progress dashboard when signed in |
| `/practice`, `/practice/[slug]` | Curated practice lists, bookmarks; one list with progress + difficulty filter |
| `/roadmap` | Topic graph from section prerequisites (`?section=` opens a topic's problems) |
| `/problems` | Catalogue table: search, difficulty, status, topic group, topic tags, sort, pagination |
| `/problems/[slug]`, `/contests/[slug]/problems/[problemSlug]` | The same split workspace (`components/workspace/ProblemWorkspace.tsx`) |
| `/courses`, `/courses/[slug]`, `/courses/[slug]/lessons/[lessonSlug]` | Catalogue, course outline, lesson player |
| `/contests`, `/contests/[slug]`, `/contests/[slug]/leaderboard` | Contest index, detail (register, problems, countdown), live standings |
| `/submissions`, `/submissions/[id]` | History with filters; verdict, per-test results, code |
| `/discuss`, `/discuss/thread/[id]` | All threads (paginated); thread with nested replies |
| `/profile` (`?tab=settings`) | Progress overview; profile form, theme, connected accounts, saved-code reset |
| `/upgrade` | Plans (checkout shows "coming soon") |
| `/admin/*` | Staff dashboard, rebuilt on the design tokens (light + dark): schema-driven resource CRUD, editors, settings, features, audit log |

**Shared building blocks:** `components/ui/*` (tokens-only kit), `ProblemTable`, `ProgressPanels`
(home + profile widgets from one cached `/users/me/progress/` payload), `PracticeListCard`,
`CourseCard`, `ContestCard`/`Countdown`, `ThreadRow`/`CommentTree`, `Markdown` + `CodeBlock`.

**Theming notes:**
- `lib/theme.ts`: Light / Dark / System via next-themes, applied everywhere including `/admin/*`.
- Markdown uses `prose prose-sm|prose-base prose-cq`. `.prose-cq` in `globals.css` is deliberately
  unlayered: the typography plugin emits `.prose` into `@layer utilities`, which would otherwise
  override the token colours.
- The theme is unknown during SSR. Anything whose markup depends on it (highlighted code, theme
  toggle, the signed-in home) renders a neutral version until `useIsClient()` is true, because
  React keeps server-rendered attributes when hydrating.
- Features without a client yet (AI chat/explain/complexity, checkout, virtual contests, custom
  test input) call `useComingSoon(key)`, which shows the server's message from `/api/config/`
  and sends no request.

**Env vars:** `NEXT_PUBLIC_API_URL` (empty in the default same-origin proxy setup), `NEXT_PUBLIC_WS_URL`, `NEXT_PUBLIC_SITE_NAME`

---

### backend/
**Tech:** Django 5, Django REST Framework, Django Channels, daphne/uvicorn, JWT (HTTP-only cookies)

**Apps:**
| App | Purpose |
|-----|---------|
| `users` | Custom user model, JWT cookie auth, OAuth identities, plan/role logic, cached progress aggregation, daily activity |
| `billing` | Plan change audit log, admin plan override, payment-provider stubs |
| `content` | AdPlacement, AdCreative, FeatureFlag — CMS for ads and flags |
| `problems` | Problem, Section, Tag, TestCase, Solution; practice lists, stars, daily problem; full admin CRUD |
| `submissions` | Submission lifecycle, RabbitMQ publisher, judge callback receiver, WS broadcast, daily-activity side effects |
| `contests` | Contest, ContestProblem, ContestParticipant, leaderboard broadcast |
| `courses` | Category → Course → Chapter → Lesson catalogue and per-user lesson progress |
| `discussions` | DiscussionThread, DiscussionComment, flag/moderation endpoints |
| `ai_assist` | Authored hints; explanation/chat/complexity report `ai_not_configured` until a provider is wired |
| `admin_panel` | Unified audit trail (`AdminLog`), runtime `PlatformSettings`, feature-registry admin endpoints, aggregate stats consumed by the admin UI |

**Auth flow:**
1. `GET /api/auth/oauth/{google|github}/start/` returns the provider authorize URL
   (PKCE S256, state cookie). A provider button only renders when its client id
   and secret are configured.
2. The provider redirects to `/auth/callback/{provider}`; the frontend posts the
   code to `/api/auth/oauth/{provider}/callback/`.
3. Django verifies the identity, signs in or creates exactly one user, and sets
   `access_token` + `refresh_token` HTTP-only cookies. Google requires a verified
   email; GitHub requires a verified primary email. Matching emails are never
   auto-merged across providers — linking is an explicit authenticated action.
4. All subsequent requests include cookies automatically. DRF reads
   `access_token` via `CookieJWTAuthentication`.
5. On 401, the frontend calls `POST /api/auth/refresh/` once and retries.
6. Django admin uses a normal staff session (separate from the JWT cookies).
   Public email/password login does not exist; `DEV_PASSWORD_AUTH` adds
   `/api/auth/dev-login/` in development and test only.

**Env vars:** `SECRET_KEY`, `DATABASE_URL`, `REDIS_URL`, `RABBITMQ_URL`, `JUDGE_INTERNAL_API_KEY`, `ALLOWED_HOSTS`, `CORS_ORIGINS`, `GOOGLE_OAUTH_CLIENT_ID/SECRET`, `GITHUB_OAUTH_CLIENT_ID/SECRET`, `OAUTH_REDIRECT_BASE`, `DEV_PASSWORD_AUTH`, `PROGRESS_CACHE_TTL`

---

### judge/
**Tech:** Python, pika (AMQP), docker SDK, psycopg2

**Files:**
| File | Purpose |
|------|---------|
| `worker.py` | RabbitMQ consumer loop; one process = one worker; scale by running N copies |
| `runner.py` | Fetches test cases, runs each through sandbox, posts callback to Django |
| `sandbox.py` | Wraps Docker SDK; enforces CPU/memory/network/PID limits per container |
| `codegen.py` | Validates function signatures, encodes JSON test-case input as line tokens, generates per-language harnesses |
| `comparator.py` | Normalises and compares actual vs expected output (`EXACT`, `FLOAT`, `UNORDERED`) |
| `executor.py` | Compiles/runs user code inside the sandbox (treated as frozen; do not edit) |
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
                           │  2. Function mode: encode JSON │
                           │     input as line tokens and   │
                           │     append generated harness   │
                           │  3. docker run (isolated)      │
                           │  4. Split __CQ_RESULT__ line,  │
                           │     compare vs expected        │
                           │  5. Collect time + memory      │
                           └─────────────────────────────────┘
                                 │
                           POST /api/submissions/{id}/judge-callback/
                           (signed with X-Judge-Api-Key header)
                                 │
                           Django updates DB + broadcasts via Redis Channels
                                 │
                           WebSocket push → browser
```

Users write only the function/`class Solution`. The generated harness reads the
line-token input, calls the function and prints the return value on a line
tagged `__CQ_RESULT__`; user debug stdout is ignored. Legacy problems with no
`function_signature` keep the stdin/stdout contract.


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
`docker-compose.yml` runs PostgreSQL 16, Redis 7 and RabbitMQ 3.13 for local
development. The dev stack uses alternate host ports (5433/6380/5673) and a
separate Compose project so it never collides with an existing install. In
production these are replaced by managed services (see Scaling below). The
frontend and backend run as ordinary long-lived Next.js/Django servers — no
static export.

---

## Learning Data Model (B4)

```
Section ──< Problem
   │           │
   │           ├──< ProblemListItem >── ProblemList   (ordered practice path)
   │           ├──< Star (user bookmark)
   │           └──< DailyProblem (one published problem per UTC date)
   │
Category ──< Course ──< Chapter ──< Lesson ──? Problem
                                      │
                                      └──< LessonProgress (user × lesson)

User ──< UserSolvedProblem
     ──< UserDailyActivity (per-UTC-day submissions/accepted)
     ──< Submission
     ──< LessonProgress
```

- `Section` is admin-managed, not an enum. `name` is the stable seed key,
  `slug` is the URL key, and `prerequisites` is a self-M2M used for ordering
  the roadmap. New sections need no code change.
- `ProblemList` items are ordered rows; unpublished problems are excluded at
  read time so a draft problem never leaks into a practice path.
- `DailyProblem` has a unique date. Staff can schedule a date in Django admin or
  the admin API; otherwise the public endpoint creates a deterministic row.
- `UserDailyActivity` is updated once per terminal judged submission inside the
  verdict transaction. `manage.py backfill_daily_activity` rebuilds it from
  submission history.
- `LessonProgress` is unique per (user, lesson); completing is idempotent.
- The dashboard aggregate is cached per user; verdict commits invalidate it.

---

## Data Flows

### Flow 1: View a Problem
```
Browser → GET /api/problems/{slug}/
Django: ProblemViewSet.retrieve()
  → checks is_published=True
  → serializes Problem + sample test cases + tags + section + starter code
  → for function-mode problems, generates templates from function_signature
  → for PRO solutions, checks user.is_pro via SolutionSerializer
  → returns JSON
Browser: renders ReactMarkdown statement, Monaco editor pre-populated with starter code
```

### Flow 2: Submit a Solution (full path)
```
Browser → POST /api/submissions/ {problem_slug, language, code}
Django:
  1. Creates Submission (status=QUEUED)
  2. publisher.publish_submission_job() → RabbitMQ (durable message)
  3. Returns {submission_id, status="QUEUED"}

Browser:
  1. Opens WebSocket ws/submissions/{id}/
  2. Polls useSubmission hook (refetchInterval=1.5s as fallback)

Judge worker:
  1. AMQP consume → job payload
  2. POST callback: status=RUNNING
  3. Fetches test cases + signature/compare config from the backend
  4. For each test case → encode input → Sandbox.run() → Docker container
  5. Splits __CQ_RESULT__, compares output → determines per-TC status
  6. POST callback to Django with full results

Django (judge-callback, one transaction):
  1. Updates Submission.status, runtime_ms, memory_kb
  2. Creates SubmissionResult rows
  3. If terminal: updates problem counters, UserSolvedProblem (on ACCEPTED),
     streak, today's UserDailyActivity row
  4. On commit: invalidates the user's cached progress payload
  5. Channels group_send to ws group "submission_{id}"

Browser WebSocket handler: updates UI status in real-time
```

### Flow 2b: Progress dashboard
```
Browser → GET /api/users/me/progress/
Django: MyProgressView → get_user_progress(user)
  → cache hit? return it (TTL PROGRESS_CACHE_TTL, default 300s)
  → otherwise aggregate:
      • published problems vs UserSolvedProblem (total/all/EASY/MEDIUM/HARD)
      • per-section totals and solved counts
      • judged submission counters (sample runs excluded)
      • 365-day UserDailyActivity heatmap (zero-filled)
      • streak and five most recent judged submissions
  → cache and return
Any judged verdict invalidates the cache after its transaction commits.
`manage.py backfill_daily_activity` rebuilds heatmap rows from history.
```

### Flow 3: Contest standings and lifecycle
```
Judge callback → locks submission → problem → user/activity → contest
  → rebuilds participant/problem stats in submission-time order
  → ranks by score, penalty, last accept, registration time, ID
  → after commit, broadcasts a top-50 snapshot (status + finalized included)
  → Channels group "leaderboard_{sha256(slug)}" (bounded group name)

All browsers on /contests/{slug}/leaderboard:
  → consumer checks public/admin visibility at connect and before every push
  → page 1 takes the pushed top-50 snapshot in place; other pages (50 rows) refetch
  → 30-second polling backs up the socket until finalized
Contest pages poll only within a minute of a start/end and while finalizing.

Contest ends → finalize_contests command / lazy API or socket access
  → lock problems in ID order, then contest (compatible with verdict locks)
  → wait for in-window queued/running/reaped jobs, up to the grace deadline
    (checked lock-free first, so polling during the grace period takes no locks)
  → freeze ranks + stamp finalized_at + system audit log
  → publish problems owed to practice: in some finalized public contest with
    release_problems, and not used by an upcoming/live round (held ones are
    released when that round is finalized, even a private one)
  → late verdicts still affect practice progress, never frozen contest stats
```

`Contest.finalized_at` and `release_problems` persist the lifecycle. Finalized
standings are read-only in the REST and Django admin surfaces. Registration and
submission creation recheck eligibility under a contest lock. The end instant
is inclusive for submissions; finalization starts strictly after it. The judging
grace period defaults to 900 seconds (`CONTEST_FINALIZE_GRACE_SECONDS`).

Contest problems can remain unpublished: their contest-scoped detail/Run/Hint
paths resolve them after start. Public practice paths still require publication.
After end the regular Pro restriction returns; after release the practice workspace
accepts ordinary submissions. Editorials and discussions stay locked during live
contests. Virtual entry points show the coming-soon toast.

The index uses paginated `current`, `recent`, and searchable `archive` buckets.
`PlatformSettings.contest_recent_days` controls the recent/archive cutoff (default
30 days, range 1–365); `/api/config/` shares it with the UI.

### Flow 4: AI Hint request
```
Browser → POST /api/ai/hint/ {problem_slug, hint_level, user_code}
Django:
  1. Hint levels come from the problem's authored hints (hints_md, split on '---')
  2. Pro-only levels/problems → 403 pro_required; past the last level → 404 no_more_hints
  3. Per-plan daily quota (AI_HINTS_FREE_DAILY / AI_HINTS_PRO_DAILY) → 429 quota_exceeded
  4. Returns {hint, level, has_more, used_today, daily_limit, source: "authored"}
Browser: workspace "Hint" dialog reveals one level at a time (its "Ask the AI tutor"
button shows the ai_chat coming-soon message; no LLM is called)
```

---

## Admin Platform (B7)

One audit trail, one settings row, one feature registry:

**Runtime settings** — `admin_panel.PlatformSettings` is a singleton (`pk=1`,
enforced by a check constraint). It is the single source for the site name and
description, the site-wide announcement (text + level), maintenance mode and its
message, hint quotas (`ai_hints_free_daily` / `ai_hints_pro_daily`, blank = the
`AI_HINTS_*_DAILY` environment default) and `contest_recent_days`. The public
config cache (`core:site-config:v2`) is invalidated immediately and on
transaction commit, so dashboard edits reach `/api/config/` without a restart.

**Feature registry** — `apps/core/features.py` is authoritative for what the UI
may offer: implemented features can be switched off at runtime via
`content.FeatureFlag` rows (Django admin or `PATCH /api/admin/features/{key}/`);
unimplemented ones always answer **501 coming_soon**. Public endpoints are gated
with the `FeatureEnabled` permission (**503 feature_disabled**, admins exempt),
and `maintenance_mode` pauses Run/Submit for non-admins with **503 maintenance**.

**Unified audit trail** — `AdminLog` records every staff change from three
sources: `api` (dashboard viewsets via `AuditedViewSetMixin` / `audited_update`),
`django` (a post_save mirror of Django admin `LogEntry`) and `system` (scheduled
finalization, OAuth account events, `grant_role`). Rows carry a field-level
`changes` diff and `object_repr`, and are exposed read-only at
`/api/admin/audit-log/` with action / source / object-type / search filters.

**Durable contest corrections** — `ContestParticipant.score_adjustment` +
`adjustment_reason`: `final_score = solved points + adjustment` on every rescore,
so admin corrections survive later verdicts. Finalization freezes standings and
rejects further participant mutations with **409**; the finalize endpoint answers
**409 judging_pending** (with a `force` path) while in-window submissions lack
verdicts.

**Admin frontend** — `frontend/src/app/admin/*` is generated from
`components/admin/resource-config.ts`: `EditorForm` renders schema-driven forms
(signature / code / markdown / relation fields), `ResourceManager` drives generic
lists + CRUD dialogs, and the per-domain editors (problems, courses, contests)
add test cases, editorials, outline reordering, participants and finalize.
ADMIN sees all sections; MODERATOR is restricted to Discussions; the dashboard
follows the user's theme (the dark-only admin pin was removed).

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
Route 53 → CloudFront → S3 (Next.js static assets)
                      → ALB
                            → ECS Fargate: frontend (next start, N replicas)
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
- Run `next start` (standalone output) in one or more containers behind the load balancer/CDN.
- Cache immutable build assets at the CDN; dynamic pages stay server-rendered so they can call the same-origin `/api` proxy.
- No static export: the app relies on server rewrites/cookies.

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

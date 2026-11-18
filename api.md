# CodeQuest — API Reference

Base URL: `http://localhost:8000/api`

Auth: JWT in HTTP-only cookies (`access_token`, `refresh_token`). Set by the OAuth
callback (or the development-only password endpoint). Internal judge calls use the
`X-Judge-Api-Key` header instead (shared secret, no JWT, not throttled).

## Conventions

**Pagination.** Standard lists use `?page=N` (default 20 per page). `?page_size=M` is honored where supported, capped:

| Endpoint group | Max `page_size` |
|---|---|
| Most lists (threads, comments, users, …) | 100 |
| Problems list, tags, submission history | 1000 |

**Error envelope.** Errors return `{"detail": "..."}` and, where a machine-readable
code helps the client branch, a `code` field:

| Code | Meaning |
|---|---|
| `pro_required` | Feature/problem requires a Pro plan (HTTP 403) |
| `no_more_hints` | Hint sequence exhausted (HTTP 404) |
| `ai_not_configured` | LLM feature not enabled on the server (HTTP 503) |
| `not_implemented` | Honest stub, see Billing (HTTP 501) |
| `queue_full` | Judge queue at hard limit (HTTP 503) |
| `rate_limited` | Throttled (HTTP 429) |

**WebSockets** validate the `Origin` header against `WS_ALLOWED_ORIGINS` and authenticate
via the `access_token` cookie. Application close codes: **4401** (unauthenticated) and
**4403** (forbidden/not yours) — clients must not auto-reconnect on these.

---

## Auth

Public sign-in is Google or GitHub OAuth. There is no public password registration or
password login. A provider is enabled only when the server has its client id + secret;
`GET /config/` reports the available providers.

### GET /auth/oauth/&lt;provider&gt;/start/
Begin a sign-in (`provider` is `google` or `github`). Stores a signed, HTTP-only state
cookie (state + nonce + PKCE verifier) valid for `OAUTH_STATE_MAX_AGE` seconds.
`?intent=connect` (signed-in users only) links another provider; `?next=/path` is the
same-site page to return to.

**Response 200:**
```json
{ "authorization_url": "https://accounts.google.com/o/oauth2/v2/auth?..." }
```
`?redirect=1` or `Accept: text/html` answers **302** instead (used by the Django admin
login page). Unknown provider → 404; missing credentials → 503.

---

### POST /auth/oauth/&lt;provider&gt;/callback/
Exchange the provider's authorization code, resolve the account, and set the JWT cookies.

**Body:** `{ "code": "...", "state": "..." }` — the state must match the signed cookie,
and each state is accepted once.

**Response 200:** sets `access_token` (60 min) and `refresh_token` (14 day) HTTP-only
cookies.
```json
{
  "message": "Signed in.", "user": { "id": 4, "username": "alice", "plan": "FREE", ... },
  "next": "/problems/two-sum", "created": true, "linked_legacy": false
}
```
With `intent=connect` the response is 201 and carries `identity` instead of `user`.
Account-linking rules: a returning provider identity signs into its user; a new identity
creates one account; a matching email on another provider is refused (the user connects
the provider explicitly from the profile); an identity owned by another user is never
taken over. Banned/inactive accounts are refused before any cookie is set.

---

### GET /auth/identities/
Linked sign-in methods for the signed-in user (**401** anonymously). The provider subject
is never exposed.

```json
[{ "id": 2, "provider": "google", "provider_display": "Google", "email": "alice@example.com", "email_verified": true, ... }]
```

### DELETE /auth/identities/&lt;provider&gt;/
Disconnect one provider. **204** on success; **409 `last_credential`** when it is the
only way into the account.

---

### POST /auth/dev-login/
Local development only: email + password for seeded accounts. Answers **404** unless
`DEV_PASSWORD_AUTH` is on (development/test); production settings force it off.

**Body:** `{ "email": "free@codequest.dev", "password": "user123" }`

**Response 200:** sets the same JWT cookies as the OAuth callback. Throttled (`THROTTLE_LOGIN`).

---

### GET /auth/me/
Returns the authenticated user's full profile. **401** for anonymous visitors (no refresh dance on this endpoint client-side; `useMe` resolves `null`).

**Response 200:**
```json
{
  "id": 4, "email": "alice@example.com", "username": "alice",
  "plan": "FREE", "is_pro": false, "plan_expires_at": null,
  "problems_solved": 12, "current_streak": 3,
  "ai_hint_limit": 5, "ai_chat_limit": 10
}
```

---

### POST /auth/logout/
Blacklists the refresh token and clears both cookies. Works unauthenticated too (idempotent). **Response 200.**

---

### POST /auth/refresh/
Rotates the refresh token (old one is blacklisted) and resets the access cookie using the refresh cookie. **Response 200** or **401** (client then redirects to login).

---

## Problems

### GET /problems/
Paginated list with filters.

**Query params:** `page`, `page_size` (≤1000), `difficulty` (EASY|MEDIUM|HARD), `section` (e.g. HASH_MAP), `topic` (tag slug), `company` (tag slug), `search`

**Response 200:**
```json
{
  "count": 42, "next": "...?page=2", "previous": null,
  "results": [
    {
      "id": 1, "slug": "two-sum", "title": "Two Sum", "number": 1,
      "difficulty": "EASY", "section": { "name": "HASH_MAP", "display_name": "Hash Map / Set" },
      "tags": [{ "id": 1, "name": "Array", "tag_type": "TOPIC" }],
      "acceptance_rate": 47.2, "is_solved": false
    }
  ]
}
```
`acceptance_rate` is a pre-computed, DB-annotated value (never null); ordering is stable.

---

### GET /problems/{slug}/
Full problem detail including statement, sample test cases and starter code.

**Response 200:**
```json
{
  "id": 1, "slug": "two-sum", "title": "Two Sum",
  "statement_md": "## Problem\n\nGiven an array...",
  "difficulty": "EASY",
  "time_complexity_average": "O(n)", "space_complexity": "O(n)",
  "test_case_count": 11,
  "function_signature": {
    "name": "twoSum",
    "params": [
      { "name": "nums", "type": "int[]" },
      { "name": "target", "type": "int" }
    ],
    "return_type": "int[]"
  },
  "compare_mode": "UNORDERED",
  "float_tolerance": 0.00001,
  "starter_code_python": "class Solution:\n    def twoSum(self, nums: List[int], target: int) -> List[int]:\n        ...",
  "starter_code_cpp": "...", "starter_code_java": "...", "starter_code_javascript": "...",
  "sample_test_cases": [
    {
      "id": 1,
      "input_data": "{\"nums\": [2, 7, 11, 15], \"target\": 9}",
      "display_input": "nums = [2, 7, 11, 15], target = 9",
      "expected_output": "[0,1]", "explanation": "..."
    }
  ],
  "hints": [{ "index": 0, "available": true }]
}
```

**Function-only judging.** `function_signature` is optional. When present, the
editor shows only the function/class: the judge parses the test-case input and
prints the return value, so users never write stdin/stdout boilerplate. When it
is `null`, the problem keeps the legacy stdin/stdout contract.

- Parameter and return types: `int`, `long`, `double`, `bool`, `string`, plus up
  to two `[]` dimensions (`int[][]` and `string[][]` are the only nested arrays).
- Function-mode test-case `input_data` is a JSON object keyed by parameter name;
  `display_input` is the same data rendered for humans (the frontend shows it in
  the test-case panel).
- `compare_mode` is `EXACT` (JSON-aware text), `FLOAT` (numeric tolerance
  `float_tolerance`) or `UNORDERED` (top-level JSON array, any order).
- Blank starter fields are generated from the signature by the API. Templates
  are function/class only; no `main` or stdin parsing.

**Response 403** (Pro-only problem, non-Pro user):
```json
{ "detail": "This problem requires a Pro plan.", "code": "pro_required", "title": "..." }
```
The frontend renders an upgrade card on this response.

---

### GET /problems/{slug}/solutions/
Editorial solutions. PRO-only solutions return truncated content + `is_locked: true` for non-Pro users.

Editorials for problems that belong to a **live contest** are locked for everyone except admins until the contest ends (`is_locked: true`, reason included).

**403 `pro_required`** if the problem itself is Pro-only.

---

### GET /problems/sections/
Admin-managed catalogue sections, ordered by `order_index`. Counts only include
published problems; `solved_count` is per-user and `0` when signed out.

**Response 200:**
```json
{
  "count": 12,
  "results": [
    {
      "id": 1, "name": "HASH_MAP", "slug": "hash_map",
      "display_name": "Hash Map / Set", "icon": "🗂️",
      "description_md": "", "order_index": 1,
      "prerequisites": ["ARRAY"],
      "problem_count": 4, "solved_count": 1
    }
  ]
}
```

### POST /problems/{slug}/star/
Toggle the current user's bookmark. **Auth required** (401 when signed out).
**Response 200:** `{ "slug": "two-sum", "starred": true }` (`false` when this
call removed the star).

### GET /problems/starred/
The current user's starred problems, newest star first. Same shape as
`GET /problems/`. **Auth required.**

### GET /problems/daily/
Today's featured problem (UTC date). When staff have not scheduled one, the
server deterministically picks a published problem and stores the row, so the
same problem is returned all day.

**Response 200:**
```json
{
  "date": "2026-11-16",
  "problem": {
    "slug": "two-sum", "title": "Two Sum", "number": 1,
    "difficulty": "EASY", "section": { "name": "HASH_MAP", "slug": "hash_map" },
    "tags": [], "acceptance_rate": 47.2, "is_solved": false
  }
}
```
**404** when the catalogue has no published problems.

---

### GET /problems/tags/?tag_type=TOPIC
All tags filtered by type (TOPIC or COMPANY).

---

## Practice Lists

Curated, ordered study paths (NeetCode-style). Both endpoints are public and
paginated; progress is included when the caller is signed in.

### GET /problems/lists/
**Response 200:**
```json
{
  "count": 2,
  "results": [
    {
      "id": 1, "slug": "array-string-warmup", "title": "Array & String Warm-up",
      "description_md": "...", "order_index": 1,
      "problem_count": 11, "solved_count": 3,
      "difficulties": { "EASY": 9, "MEDIUM": 1, "HARD": 1 }
    }
  ]
}
```

### GET /problems/lists/{slug}/
List detail plus the ordered problems (unpublished problems are omitted from
both the count and the payload):

```json
{
  "slug": "array-string-warmup", "title": "Array & String Warm-up",
  "problem_count": 11, "solved_count": 3,
  "difficulties": { "EASY": 9, "MEDIUM": 1, "HARD": 1 },
  "problems": [
    { "slug": "contains-duplicate", "title": "Contains Duplicate", "difficulty": "EASY", "is_solved": true }
  ]
}
```

---

## Progress

### GET /users/me/progress/
Cached dashboard aggregate for the signed-in user. The payload covers the
home progress ring, the 365-day submission heatmap, streaks, per-section
progress and the five most recent judged submissions.

**Response 200:**
```json
{
  "totals": {
    "all":    { "total": 25, "solved": 1 },
    "EASY":   { "total": 17, "solved": 1 },
    "MEDIUM": { "total": 6,  "solved": 0 },
    "HARD":   { "total": 2,  "solved": 0 }
  },
  "solve_rate": 4.0,
  "submissions": {
    "total": 12, "accepted": 4, "acceptance_rate": 33.3,
    "today": 1, "accepted_today": 0
  },
  "streak": { "current": 1, "best": 4, "last_active_date": "2026-11-16" },
  "sections": [
    { "id": 1, "name": "HASH_MAP", "slug": "hash_map", "display_name": "Hash Map / Set",
      "icon": "🗂️", "total": 4, "solved": 1, "percent": 25.0 }
  ],
  "activity": [{ "date": "2025-11-17", "submissions": 0, "accepted": 0 }],
  "recent_submissions": [
    { "id": "uuid", "problem_slug": "two-sum", "problem_title": "Two Sum",
      "status": "WRONG_ANSWER", "language": "python", "created_at": "..." }
  ]
}
```

- `activity` always has exactly 365 entries, oldest first, zero-filled.
- Internal judge errors are excluded from submission stats.
- The payload is cached per user for `PROGRESS_CACHE_TTL` seconds (default 300)
  and invalidated when a judged verdict is committed. Completed sample runs
  ("Run") never change progress.

---

## Courses

Category → Course → Chapter → Lesson. Public catalog; completed-lesson flags
require auth and default to `false` when signed out.

### GET /courses/categories/
```json
{ "count": 2, "results": [
  { "id": 1, "name": "Interview Prep", "slug": "interview-prep", "icon": "🎯", "course_count": 1 }
] }
```

### GET /courses/
Query params: `category` (category slug), `difficulty`
(BEGINNER|INTERMEDIATE|ADVANCED), `search`, `page`, `page_size`.

```json
{
  "slug": "python-interview-foundations",
  "title": "Python Interview Foundations",
  "subtitle": "Complexity, hash maps and the sliding-window habit",
  "difficulty": "BEGINNER", "cover_icon": "🐍", "estimated_minutes": 90,
  "category": { "id": 1, "name": "Interview Prep", "slug": "interview-prep" },
  "lesson_count": 4, "completed_count": 1, "progress_percent": 25.0
}
```

### GET /courses/{slug}/
Course detail plus `chapters[]`; every chapter carries its published
`lessons[]`, and every lesson may link a practice `problem`
(`{slug, title, number, difficulty}` or `null`).

### POST /courses/lessons/{id}/complete/ · DELETE /courses/lessons/{id}/complete/
Mark/unmark a lesson complete. **Auth required.** Completing twice is
idempotent. Response:
```json
{
  "lesson_id": 3, "completed": true,
  "course": { "slug": "python-interview-foundations",
              "lesson_count": 4, "completed_count": 1, "progress_percent": 25.0 }
}
```

---

## Submissions

### POST /submissions/
Create a judged submission. Enqueues to RabbitMQ. Returns immediately.

**Body:** `{ "problem_slug": "two-sum", "language": "python", "code": "def twoSum(...):\n    ..." }` (code ≤ `MAX_CODE_BYTES` bytes UTF-8)

Optional `"contest": <id>` scores the submission in that contest: the problem is resolved through the contest (so unpublished contest problems work), the contest must be live, and the caller registered and not disqualified (**403** otherwise; **400** when the problem is not in the contest). Eligibility is rechecked under the contest lock when the row is created.

**Response 201:**
```json
{ "submission_id": "550e8400-e29b-41d4-a716-446655440000", "status": "QUEUED" }
```

**Response 503** (broker down or queue over the hard limit):
```json
{ "detail": "Judge queue is full. Please try again shortly.", "code": "queue_full" }
```
A persisted `INTERNAL_ERROR` submission row is created in that case, so the failure shows up in history.

Plan-scoped throttles: `THROTTLE_SUBMIT_FREE` (3/min) vs `THROTTLE_SUBMIT_PRO` (10/min), plus a sliding burst guard (`THROTTLE_BURST`).

---

### POST /submissions/run-code/
Sample-run against the problem's sample test cases only. Same body/limits; result rows are flagged `is_sample_run` and excluded from history and stats. Same throttle family (`THROTTLE_RUN_*`). With `"contest": <id>` the problem is opened through that contest (unpublished contest problems, no registration needed); runs are never attached to the contest and never score.

---

### GET /submissions/{id}/
Full submission detail with per-testcase results. Stale `QUEUED/RUNNING` rows older than `JUDGE_STALE_SUBMISSION_SECONDS` are reaped to `INTERNAL_ERROR` on read.

Hidden test cases are **redacted for non-admins**: hidden rows expose `status`, timing and error flags but blank `actual_output`/`expected_output`/`input_data`, with `is_hidden: true`.

**Response 200:**
```json
{
  "id": "550e8400...", "problem_slug": "two-sum", "language": "python",
  "status": "ACCEPTED", "runtime_ms": 48, "memory_kb": 15200,
  "results": [
    { "id": 1, "test_case": 1, "status": "ACCEPTED", "actual_output": "[0,1]", "time_ms": 12, "is_hidden": false }
  ]
}
```
Only the owner (or an admin) can read a submission; others get **403**.

**Status enum:** `QUEUED`, `RUNNING`, `ACCEPTED`, `WRONG_ANSWER`, `RUNTIME_ERROR`, `TIME_LIMIT`, `MEMORY_LIMIT`, `COMPILE_ERROR`, `INTERNAL_ERROR`

---

### GET /submissions/history/
Current user's judged submissions (sample runs excluded), newest first. Optional `?problem_slug=two-sum` filter; `?page_size=` up to 1000 (used for activity charts).

---

## Judge ↔ Backend (internal, `X-Judge-Api-Key`)

Shared-secret endpoints used by the judge worker (`JUDGE_INTERNAL_API_KEY` on both sides).
A wrong/missing key yields **403**. These routes bypass JWT auth and throttling on purpose.

> **Deploy note:** the callback contract below is versioned with the worker.
> Deploy `backend/` and `judge/` together.

### POST /submissions/{submission_id}/judge-callback/
Write results back. **Idempotent**: duplicate or out-of-order callbacks return 200 with `{"ok": true, "applied": false}` — only the first RUNNING callback and the first terminal callback mutate state; counters (solved counts, problem stats) move exactly once, at verdict time, inside a transaction.

**Body:**
```json
{
  "status": "RUNNING" | "ACCEPTED" | "WRONG_ANSWER" | "RUNTIME_ERROR" | "TIME_LIMIT"
          | "MEMORY_LIMIT" | "COMPILE_ERROR" | "INTERNAL_ERROR",
  "runtime_ms": 48, "memory_kb": 15200, "error_message": "",
  "results": [
    { "test_case_id": 1, "status": "ACCEPTED", "actual_output": "[0,1]",
      "error_output": "", "time_ms": 12, "memory_kb": 15000 }
  ]
}
```
Validation: `status` must be a known enum value (unknown values → 400, not silently stored); ≤1000 result rows; duplicate `test_case_id` → 400.

**Responses:** `200 {"ok": true, "applied": true|false, "final_status": "..."}` · `400` (invalid payload/transition) · `403` (bad key) · `404` (unknown submission).

**Retry semantics:** transient backend outages during the callback are retried by the worker (`CALLBACK_RETRIES`); jobs that fail repeatedly are requeued up to `MAX_JOB_RETRIES` then dead-lettered to `<queue>.dead`.

### GET /submissions/judge/problems/{problem_id}/testcases/[?sample_only=1]
Test cases and judge configuration for a problem, ordered by `order_index` — so
the worker needs no database credentials.

**Response 200:**
```json
{
  "problem_id": 1,
  "function_signature": {
    "name": "twoSum",
    "params": [
      { "name": "nums", "type": "int[]" },
      { "name": "target", "type": "int" }
    ],
    "return_type": "int[]"
  },
  "compare_mode": "UNORDERED",
  "float_tolerance": 0.00001,
  "test_cases": [
    {
      "id": 1,
      "input_data": "{\"nums\": [2, 7, 11, 15], \"target\": 9}",
      "expected_output": "[0,1]",
      "is_hidden": false,
      "is_sample": true
    }
  ]
}
```

`function_signature` is `null` for legacy stdin/stdout problems. In function
mode the worker encodes each JSON parameter value into line tokens, appends a
generated harness to the user's code, and keeps the result printed on the
`__CQ_RESULT__` line (user debug output is ignored). **404** for unknown problems.

---

## Contests

### GET /contests/
Paginated public contests (`page`, `page_size`, optional case-insensitive `search`).
`bucket` selects:

| Bucket | Contents / ordering |
|---|---|
| `current` | live + upcoming, earliest start first |
| `recent` | ended within the recent window, latest end first |
| `archive` | ended before the recent window, latest end first |
| `ended` | recent + archive |
| omitted | all public contests, latest start first |

The recent window is `PlatformSettings.contest_recent_days` (1–365, default 30),
also returned as `GET /config/ → contests.recent_days`. The cutoff belongs to
Recent; a contest exactly at its `end_at` is still live. Rows include `status`,
`finalized`, `problem_count`, `participant_count`, and `user_result` (null or
`{rank, final_score, is_disqualified}` for the signed-in participant).

---

### GET /contests/{slug}/
Contest detail with problem list.

```json
{
  "id": 1, "name": "Weekly Contest #1", "slug": "weekly-contest-1",
  "status": "upcoming", "start_at": "2026-03-20T14:00:00Z", "duration_minutes": 120,
  "is_registered": false, "participant_count": 12,
  "finalized": false, "finalized_at": null, "user_result": null,
  "problems_hidden": true,
  "contest_problems": []
}
```

Contest problems are **hidden until the contest starts** — before `start_at`,
`contest_problems` is `[]` and `problems_hidden` is `true` (admins may preview).
Each revealed problem row includes `practice_available` and the caller's contest
solve/attempt data. Unknown/private contests return 404; admins may view private contests.

### GET /contests/{slug}/problems/{problem_slug}/
Contest-scoped problem detail, including unpublished problems. Before start:
**403** `contest_not_started` (admin preview allowed). While live: anyone can read
the statement regardless of plan; authenticated users can Run samples with
`contest: <id>`. After end: archived statement remains readable, with the usual
Pro gate restored. **404** for a problem outside the contest or a private contest.
Returns the normal problem detail plus `practice_available`; catalogue neighbours
are null. Once released, use `/problems/{slug}/` for practice submissions.

---

### POST /contests/{slug}/register/
Register the current user. **201** created; **200** `{ "detail": "Already registered." }`; **400** if the contest has ended or is full (`max_participants` enforced with a row lock); **404** unknown/private contest.

---

### GET /contests/{slug}/leaderboard/
Paginated (`page`, `page_size`; the UI uses 50). Ordered by ICPC rules:
`final_score` desc, `penalty_minutes` asc, last accepted submission time,
registration time, then participant ID. Penalty = Σ over solved problems of
(solve minutes + 10 per penalized wrong attempt before the first accepted
submission). Compile/internal errors are not penalized. Verdicts arriving out of
order use submission timestamps. Unknown/private contests return **404** except
for admins. Finalized contests return the frozen result.

Also available via WebSocket at `ws/contests/{slug}/leaderboard/`.

### Finalization
Ended contests finalize after in-window judging finishes, or after
`CONTEST_FINALIZE_GRACE_SECONDS` (defaults to the stale-job timeout, 900 seconds).
Reaped jobs remain pending during that grace period because a legitimate verdict
can still replace them. Later verdicts update practice progress but never frozen
contest standings. Finalization assigns ranks, clears disqualified ranks, records
a system audit entry and publishes eligible problems. It is transactional and
idempotent. List/detail/problem/standings requests and socket connections finalize
lazily; lists handle at most ten overdue contests per request.

`manage.py finalize_contests` processes all due contests. Options: `--dry-run`,
`--slug <slug>`, `--force` (skip remaining judging). A Django admin action offers
the same force-finalize operation. Public contests release problems by default;
`release_problems=false` and private contests keep them unpublished. A problem is
released once any finalized public contest with `release_problems` contains it,
but held while an upcoming/live contest still uses it; that round's finalization
then releases it, even if that round is private or opts out itself.

### Admin: contests
All require `role=ADMIN`:
- `GET/POST /contests/admin/contests/`, `GET/PATCH/DELETE /contests/admin/contests/{slug}/` — full CRUD (`ContestAdminSerializer`). The ordered problem set (`problem_entries`) is locked once a contest starts; finalization additionally freezes the schedule and release policy (**400**).
- `GET /contests/admin/contests/{slug}/participants/` — every participant, including disqualified ones (`?search=`).
- `PATCH /contests/admin/contests/{slug}/participants/{user_id}/` — disqualify (`is_disqualified`, `disqualified_reason`) or correct a score with `{ "score_adjustment": 50, "adjustment_reason": "Judge lost the first accept" }`. `final_score = solved points + score_adjustment`, so the correction survives every later verdict and rescore, and ranks are recomputed immediately. **204** on delete (also for an unknown contest); finalized standings reject mutations with **409**.
- `POST /contests/admin/contests/{slug}/finalize/` — `{ "force": true|false }`. **409** `judging_pending` (body carries the pending count) while in-window submissions still lack verdicts inside the grace period; `{ "force": true }` finalizes without them. **200** `{ "outcome": "finalized"|"already_finalized"|"not_ended"|"judging_pending", "released", "pending", "participants" }`.

---

## Discussions

Unpublished problem threads are hidden. During a live contest, non-admins cannot
read or post its problem discussions: problem-scoped endpoints return **403**
`contest_locked`; global lists omit them and direct thread/comment URLs return
**404**. Access returns when the contest ends and the problem is published.

### GET /discuss/problems/{slug}/threads/
All threads for a problem.

### POST /discuss/problems/{slug}/threads/create/
Create a new thread. **Body:** `{ "title": "My approach for Two Sum" }` **Auth required.**

### GET /discuss/threads/{id}/
Thread detail with comments.

### GET /discuss/threads/{id}/comments/
Comments for a thread including nested replies (`comment_count` is de-duplicated — a comment's counts no longer multiply by reply count).

### POST /discuss/threads/{id}/comments/create/
Post a comment. **Body:** `{ "body_md": "Great solution!", "parent": null }` **Auth required.**

### POST /discuss/comments/{id}/upvote/ · POST /discuss/comments/{id}/accept/
Upvote / mark accepted answer. **Auth required** (author-of-thread for accept).

### POST /discuss/comments/{id}/flag/
Flag a comment for moderation. **Auth required.**

### PATCH /discuss/admin/comments/{id}/moderate/
Hide or clear flag: `{ "is_hidden": true }` or `{ "is_flagged": false }`. **MODERATOR/ADMIN only.**

### Admin moderation CRUD (role=MODERATOR or ADMIN)
- `GET/POST /discuss/admin/threads/` + `GET/PATCH/DELETE /discuss/admin/threads/{id}/`
- `GET/POST /discuss/admin/comments/` + `GET/PATCH/DELETE /discuss/admin/comments/{id}/`
All writes (including pin/lock and hidden/flag changes) are audited.

---

## Layout / Ads

### GET /layout/placements/?route=/problems
Returns active ad creatives for the given route **and the current user's plan** — placements can target FREE/ANON/PRO. PRO users always receive `{ "placements": [] }`.

**Response 200:**
```json
{
  "placements": [
    {
      "key": "sidebar_left",
      "position": "sidebar_left",
      "creatives": [{ "id": 1, "html_snippet": "<div>...</div>", "priority": 10 }]
    }
  ]
}
```
Creative HTML/links are validated server-side (`validate_link`: only `http(s)` schemes, no `javascript:`); the frontend renders snippets inside a **sandboxed iframe** (`sandbox="allow-sandbox allow-scripts"`-equivalent attribute set, no same-origin).

### GET /layout/pages/ · GET /layout/pages/{slug}/
CMS content pages (markdown); only `published` rows are public. Published pages
render on the frontend at `/pages/{slug}/` and the Footer links them automatically.

### Admin (role=ADMIN)
- `GET/POST /layout/admin/pages/` + `GET/PATCH/DELETE /layout/admin/pages/{id}/` — content pages (`published`, `order_index`).
- `GET/POST /layout/admin/placements/` + `GET/PATCH/DELETE /layout/admin/placements/{id}/`
- `GET/POST /layout/admin/creatives/` + `GET/PATCH/DELETE /layout/admin/creatives/{id}/`

---

## AI Assist

Auth required. Quotas are **per user per UTC day**, enforced with a row lock (`AI_HINTS_FREE_DAILY`=5, `AI_HINTS_PRO_DAILY`=50, `AI_CHAT_FREE_DAILY`=10, `AI_CHAT_PRO_DAILY`=200). Hints are **authored content** (stored, level-gated) — no LLM call involved.

### POST /ai/hint/
Progressive hints for a problem.

**Body:** `{ "problem_slug": "two-sum", "hint_level": 0, "user_code": "def twoSum(...):" }`

Optional `"contest": <id>` (requires `problem_slug`) resolves the problem through its contest, so unpublished contest problems have hints; while the contest is live the first hint is open to every plan. Before the start: **403** `contest_not_started`.

**Response 200:**
```json
{
  "hint": "Think about what data structure allows O(1) lookups.",
  "level": 0, "has_more": true,
  "next_requires_pro": false,
  "used_today": 1, "daily_limit": 5,
  "source": "authored"
}
```
`next_requires_pro` tells the client up front that the next hint needs Pro.

**Errors:** 403 `pro_required` (Pro-gated hint or problem) · 404 `no_more_hints` · 429 quota exhausted (`{ "detail": "Daily hint limit reached (5/day). Upgrade to Pro for more hints.", "used_today": 5, "daily_limit": 5 }`).

---

### POST /ai/chat/ · POST /ai/explain-solution/ · POST /ai/analyze-complexity/
LLM-backed endpoints. **They are honest stubs**: without a configured provider they return

**Response 503:**
```json
{ "detail": "AI chat is not configured on this server yet.", "code": "ai_not_configured" }
```
(Pro gating and daily quotas are still enforced before the 503.)

---

## Billing

### GET /billing/plans/
Plan catalog derived from the enforced limits in settings (`THROTTLE_*`, AI quotas, judge limits).

**Response 200:** `{ "plans": [ { "id": "FREE", "name": "Free", "price_monthly": 0, "features": [...], "limits": {...} }, { "id": "PRO", ... } ] }`

### POST /billing/subscribe/ · POST /billing/cancel/
**501 `{ "detail": "...", "code": "not_implemented" }`** — no payment processor is integrated; the API refuses to pretend otherwise. The frontend `/upgrade` page explains this and points to the admin grant path.

### PATCH/POST /billing/admin/users/{id}/plan/   (role=ADMIN)
Change a user's plan: `{ "plan": "PRO", "days": 30, "notes": "Support grant" }`. `days` sets `plan_expires_at`; omit for no expiry. Returns the updated admin user object.

---

## Admin Endpoints

All require `role=ADMIN`.

### GET /admin/stats/overview/
```json
{
  "users": { "total": 1200, "pro": 145, "free": 1055, "new_this_week": 34, "pro_pct": 12.1 },
  "submissions": { "total": 48000, "today": 320, "accepted_today": 190, "acceptance_rate_today": 59.4, "per_day": [...] },
  "platform": { "total_problems": 42, "draft_problems": 2, "practice_lists": 6, "courses": 3, "active_contests": 1, "upcoming_contests": 2, "flagged_comments": 3 }
}
```
`per_day` is always 14 entries (zero-filled) — `{ "day": "2026-11-01", "count": 0, "accepted": 0 }`.

### GET /admin/stats/problems/
Per-problem acceptance rates, busiest first: `{ "problems": [{ "slug", "title", "difficulty", "is_published", "total_submissions", "accepted_submissions", "acceptance_rate" }] }`

### GET /admin/stats/contests/{slug}/
Contest analytics: `{ "contest", "status", "participants", "submissions", "accepted", "problems": [{ "problem_slug", "title", "score", "submissions", "solved_by" }] }`

### GET /admin/users/?search=alice&plan=PRO&role=USER
Filterable, paginated user list (`plan=PRO` respects non-expired Pro status; `search` matches email/username/display_name). Stable newest-first ordering.

### GET /admin/users/{id}/ · PATCH /admin/users/{id}/
Detail / update: `is_banned`, `banned_reason`, `role`, `display_name`, `is_active`. Self-protection: an admin cannot demote or ban themselves (**400**). `plan` is read-only here — billing endpoint owns it.

### GET /admin/discussions/flags/
Flagged comments still visible: `{ "flagged_comments": [{ "id", "body_preview", "created_by", "thread_id", "problem_slug", "created_at" }], "count": n }` (moderators allowed).

### GET /admin/settings/ · PATCH /admin/settings/
The single runtime settings row (`PlatformSettings`, created with defaults on
first use): `site_name`, `site_description`, `announcement_text`,
`announcement_level` (`info|warning|success`), `maintenance_mode`,
`maintenance_message`, `ai_hints_free_daily`, `ai_hints_pro_daily` (blank =
environment default, exposed as `hint_defaults`), `contest_recent_days`.
PATCH is partial and audited; `/api/config/` reflects it immediately (double cache
invalidation) — no restart needed. Maintenance mode pauses Run/Submit for
non-admins with **503 `maintenance`**. POST answers **405**.

### GET /admin/features/ · PATCH /admin/features/{key}/
The feature registry: every capability with `{ "key", "label", "implemented", "default_on", "override", "state", "message" }`, where `state` is `enabled`, `disabled` or `coming_soon`. PATCH takes `{ "enabled": true|false|null }` — `null` removes the override (back to the registry default). Features that are not built yet answer **400 `not_toggleable`**; unknown keys **404**. Every change is audited.

### GET /admin/audit-log/?action=&object_type=&object_id=&source=api|django|system&admin=&search=
The unified audit trail (paginated, newest first): one row per dashboard API
change, Django admin change (mirrored `LogEntry`), or system action. Rows carry
`{ "action", "object_type", "object_id", "object_repr", "description", "changes", "source", "admin" }`, where `changes` maps field → `[old, new]`.
`GET /admin/audit-log/types/` lists the object types present (filter options).

---

## Problems — Admin CRUD (role=ADMIN)

- `GET/POST /problems/admin/problems/`, `GET/PATCH/DELETE /problems/admin/problems/{slug}/`
  The list endpoint returns light rows; the detail is fully editable. Slugs left
  blank are generated from the title and made unique (`two-sum`, `two-sum-2`).
  Deleting a problem with submissions or contest entries answers **409
  `in_use`** — unpublish it instead (its history stays intact).
- `GET/POST /problems/admin/problems/{slug}/testcases/`, `GET/PUT/PATCH/DELETE .../testcases/{id}/`
  For function-mode problems (a `function_signature` is set), `input_data` and
  `expected_output` are validated against the signature exactly as the judge
  reads them (parameter-keyed JSON); violations answer **400** with the field name.
- `GET/POST /problems/admin/problems/{slug}/solutions/`, `GET/PUT/PATCH/DELETE .../solutions/{id}/`
- `GET/POST /problems/admin/sections/`, `GET/POST /problems/admin/tags/` (+ retrieve/update/destroy)
  `prerequisite_ids` rejects self-references and cycles (the roadmap is a DAG).
- `GET/POST /problems/admin/lists/`, `GET/PATCH/DELETE /problems/admin/lists/{slug}/` —
  practice lists. `problem_ids: [1, 2, 3]` sets the ordered membership in one
  call; omitted on PATCH leaves the order untouched.
- `GET/POST /problems/admin/daily/`, `GET/PATCH/DELETE /problems/admin/daily/{id}/` —
  schedule the daily problem. Write `{ "date": "2026-11-20", "problem_id": 4 }`;
  one row per date. `?from=YYYY-MM-DD` limits the list to that date onward.

### Django admin

Sections (with prerequisites), practice lists (inline ordered items), daily
problems, stars, courses, chapters, lessons and lesson progress are also
editable in Django admin at `/django-admin/` for staff sessions, so catalogue
work needs no code change.

---

## WebSockets

### ws/submissions/{submission_id}/
Real-time status for one submission. Owner or admin only — anyone else is closed with **4403**; anonymous with **4401**.

**Messages received:**
```json
{ "type": "connected", "submission_id": "550e8400..." }
{ "type": "status_update", "submission_id": "550e8400...", "status": "ACCEPTED", "runtime_ms": 48, "memory_kb": 15200 }
```
A `status_update` snapshot is pushed immediately on connect, so a verdict that landed before the socket opened is not missed. Do not reconnect on 4401/4403.

---

### ws/contests/{contest_slug}/leaderboard/
Live top-50 snapshots, with the same visibility policy as REST. Unknown/private
contests close with **4404**; do not reconnect. Visibility is rechecked before
each broadcast. The standings page applies each pushed snapshot to page 1 (the
same top 50 rows) and refetches other pages; until finalization, polling every 30
seconds backs up the socket. Contest pages otherwise poll only within a minute of a
start/end (clock skew) and every 10 seconds while results are being finalized.

**Messages received:**
```json
{ "type": "connected", "contest_slug": "weekly-contest-1" }
{ "type": "leaderboard_update", "status": "ended", "finalized": true, "entries": [{ "rank": 1, "username": "alice", "final_score": 250, "penalty_minutes": 42 }] }
```

---

## Rate limiting summary

| Scope | Rate (default) |
|---|---|
| Anonymous API | 1000/hour |
| Authenticated API | 5000/hour |
| Login / Register | 10/min / 20/hour |
| Run code (FREE / PRO) | 10/min / 30/min |
| Submit (FREE / PRO) | 3/min / 10/min |
| Run+submit burst guard | 3 per 10s |

429 responses include `{ "detail" }`; the client surfaces it verbatim where it matters (AI panel, console).

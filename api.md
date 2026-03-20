# CodeQuest — API Reference

Base URL: `http://localhost:8000/api`

Auth: JWT in HTTP-only cookies (`access_token`, `refresh_token`). Set by login/register.
Internal judge calls use `X-Judge-Api-Key` header instead.

---

## Auth

### POST /auth/register/
Register a new user.

**Body:**
```json
{ "email": "alice@example.com", "username": "alice", "password": "secret123", "password_confirm": "secret123" }
```

**Response 201:**
```json
{ "message": "Registration successful.", "user": { "id": 4, "username": "alice", "plan": "FREE", "is_pro": false, ... } }
```

---

### POST /auth/login/
Login and receive JWT cookies.

**Body:** `{ "email": "alice@example.com", "password": "secret123" }`

**Response 200:** Sets `access_token` (60 min) and `refresh_token` (14 day) HTTP-only cookies.
```json
{ "message": "Login successful.", "user": { "id": 4, "username": "alice", "plan": "FREE", ... } }
```

---

### GET /auth/me/
Returns the authenticated user's full profile.

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
Blacklists refresh token and clears cookies. **Response 200.**

---

### POST /auth/refresh/
Silently refreshes the access token using the refresh cookie. **Response 200.**

---

## Problems

### GET /problems/
Paginated list with filters.

**Query params:** `page`, `difficulty` (EASY|MEDIUM|HARD), `section` (e.g. HASH_MAP), `topic` (tag slug), `company` (tag slug), `search`

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

---

### GET /problems/{slug}/
Full problem detail including statement and sample test cases.

**Response 200:**
```json
{
  "id": 1, "slug": "two-sum", "title": "Two Sum",
  "statement_md": "## Problem\n\nGiven an array...",
  "difficulty": "EASY",
  "time_complexity_average": "O(n)", "space_complexity": "O(n)",
  "sample_test_cases": [
    { "id": 1, "input_data": "[2,7,11,15]\n9", "expected_output": "[0,1]", "explanation": "..." }
  ],
  "hints": [{ "index": 0, "available": true }]
}
```

---

### GET /problems/{slug}/solutions/
Editorial solutions. PRO-only solutions return truncated content + `is_locked: true` for non-Pro users.

---

### GET /problems/sections/
All problem sections. **Response 200:** `[{ "id": 1, "name": "HASH_MAP", "display_name": "Hash Map / Set", "icon": "🗂️", "order_index": 1 }]`

---

### GET /problems/tags/?tag_type=TOPIC
All tags filtered by type (TOPIC or COMPANY).

---

## Submissions

### POST /submissions/
Create a submission. Enqueues to RabbitMQ. Returns immediately.

**Body:** `{ "problem_slug": "two-sum", "language": "python", "code": "def twoSum(...):\n    ..." }`

**Response 201:**
```json
{ "submission_id": "550e8400-e29b-41d4-a716-446655440000", "status": "QUEUED" }
```

---

### GET /submissions/{id}/
Full submission detail with per-testcase results.

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

---

### GET /submissions/history/
Current user's submission history. Optional `?problem_slug=two-sum` filter.

---

## Contests

### GET /contests/
All public contests. Each has `status`: `upcoming | live | ended`.

---

### GET /contests/{slug}/
Contest detail with problem list.

**Response 200:**
```json
{
  "id": 1, "name": "Weekly Contest #1", "slug": "weekly-contest-1",
  "status": "upcoming", "start_at": "2026-03-20T14:00:00Z", "duration_minutes": 120,
  "is_registered": false, "participant_count": 12,
  "contest_problems": [{ "id": 1, "problem": { "title": "Two Sum", ... }, "score": 100 }]
}
```

---

### POST /contests/{slug}/register/
Register current user for contest. **Response 201** (or 200 if already registered).

---

### GET /contests/{slug}/leaderboard/
Ordered by rank. Also available via WebSocket at `ws/contests/{slug}/leaderboard/`.

---

## Discussions

### GET /discuss/problems/{slug}/threads/
All threads for a problem.

### POST /discuss/problems/{slug}/threads/
Create a new thread. **Body:** `{ "title": "My approach for Two Sum" }` **Auth required.**

### GET /discuss/threads/{id}/comments/
Comments for a thread including nested replies.

### POST /discuss/threads/{id}/comments/
Post a comment. **Body:** `{ "body_md": "Great solution!", "parent": null }` **Auth required.**

---

## Layout / Ads

### GET /layout/placements/?route=/problems
Returns active ad creatives for the given route and current user's plan.
PRO users always receive `{ "placements": [] }`.

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

---

## AI Assist

### POST /ai/hint/
Returns a progressive hint. Rate-limited (FREE: 5/day, PRO: 50/day).

**Body:** `{ "problem_slug": "two-sum", "hint_level": 0, "user_code": "def twoSum(...):" }`

**Response 200:**
```json
{ "hint": "Think about what data structure allows O(1) lookups.", "level": 0, "has_more": true, "used_today": 1, "daily_limit": 5 }
```

**Response 429:** `{ "detail": "Daily hint limit reached (5/day). Upgrade to Pro for more hints.", "used": 5, "limit": 5 }`

---

### POST /ai/explain-solution/
Explains code with time/space complexity.

**Body:** `{ "problem_slug": "two-sum", "code": "...", "language": "python" }`

**Response 200:**
```json
{ "explanation": "...", "time_complexity": "O(n)", "space_complexity": "O(n)", "time_justification": "...", "improvement": "..." }
```

---

### POST /ai/chat/
Problem-scoped conversational AI. Maintains history in Redis per `conversation_id`.

**Body:** `{ "problem_slug": "two-sum", "message": "How do I approach this?", "conversation_id": "optional-uuid" }`

**Response 200:**
```json
{ "response": "Let's think step by step...", "conversation_id": "chat:4:two-sum", "used_today": 1, "daily_limit": 10 }
```

---

### POST /ai/analyze-complexity/
Analyzes time/space complexity of arbitrary code.

**Body:** `{ "code": "...", "language": "python" }` **Response 200:** `{ "time_complexity": "O(n)", "space_complexity": "O(n)", "confidence": "medium" }`

---

## Admin Endpoints

All require `role=ADMIN`.

### GET /admin/stats/overview/
```json
{
  "users": { "total": 1200, "pro": 145, "free": 1055, "new_this_week": 34, "pro_pct": 12.1 },
  "submissions": { "total": 48000, "today": 320, "accepted_today": 190, "per_day": [...] },
  "platform": { "total_problems": 42, "active_contests": 1, "flagged_comments": 3 }
}
```

### GET /admin/stats/problems/
Per-problem acceptance rates, ordered by total submissions.

### GET /admin/stats/contests/{slug}/
Contest-specific analytics: participants, submissions, top 10.

### GET /admin/users/?search=alice&plan=PRO
Filterable user list with full admin fields.

### PATCH /admin/users/{id}/
Update user: `is_banned`, `banned_reason`, `role`, `display_name`.

### PATCH /billing/admin/users/{id}/plan/
Change user plan: `{ "plan": "PRO", "days": 30, "notes": "Support grant" }`

### GET /admin/discussions/flags/
Flagged comments awaiting moderation.

### PATCH /discuss/admin/comments/{id}/moderate/
Hide or clear flag: `{ "is_hidden": true }` or `{ "is_flagged": false }`

---

## WebSockets

### ws/submissions/{submission_id}/
Subscribe to real-time submission status updates.

**Messages received:**
```json
{ "type": "connected", "submission_id": "550e8400..." }
{ "type": "status_update", "submission_id": "550e8400...", "status": "ACCEPTED", "runtime_ms": 48, "memory_kb": 15200 }
```

---

### ws/contests/{contest_slug}/leaderboard/
Subscribe to live leaderboard updates.

**Messages received:**
```json
{ "type": "connected", "contest": "weekly-contest-1" }
{ "type": "leaderboard_update", "entries": [{ "rank": 1, "username": "alice", "final_score": 250 }, ...] }
```
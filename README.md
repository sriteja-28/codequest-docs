# CodeQuest — LeetCode-style Coding Platform

A full-stack, self-hostable competitive programming platform with:
- Online Judge (multi-language, sandboxed Docker execution)
- Contests, leaderboards, company/topic tags
- Free vs Pro plans with AI-powered hints and chat
- CMS-style admin — no code changes needed for content

## Architecture

```
┌─────────────┐    REST/WS    ┌──────────────────┐
│  Frontend   │ ─────────────▶│  Backend (Django) │
│  (Next.js)  │               │  DRF + Channels   │
└─────────────┘               └────────┬─────────┘
                                        │ ORM
                               ┌────────▼─────────┐
                               │   PostgreSQL DB   │
                               └──────────────────┘
                                        │
                    ┌───────────────────┼──────────────────┐
                    │                   │                   │
             ┌──────▼──────┐   ┌───────▼──────┐  ┌────────▼──────┐
             │  RabbitMQ   │   │    Redis      │  │  Judge Worker │
             │  (job queue)│   │ (Channels/WS) │  │  (sandboxed)  │
             └─────────────┘   └──────────────┘  └───────────────┘
```

## Prerequisites

- **Docker & Docker Compose** (v2+)
- **Python 3.11+**
- **Node.js 20+**
- **Make** (optional, for shortcuts)

## Quick Start

### 1. Start Infrastructure

```bash
cd infra
docker compose up -d
docker compose start 
```

This starts: PostgreSQL (5432), Redis (6379), RabbitMQ (5672, management UI 15672).

### 2. Backend Setup

```bash
cd backend
python -m venv .venv # python -m venv leet_venv
source .venv/bin/activate        # Windows: .\leet_venv\Scripts\activate
pip install -r requirements.txt
       or
python -m pip install -r requirements.txt

cp .env.example .env             # Edit as needed

python manage.py migrate
python manage.py createsuperuser
python manage.py seed_data       # Loads sample problems, contests, ads

uvicorn config.asgi:application --reload --port 8000

daphne -b 0.0.0.0 -p 8000 config.asgi:application
```

### 3. Judge Worker Setup

```bash
cd judge
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env             # Edit as needed

# Build language runner images first
docker build -t codequest-python ./languages/python
docker build -t codequest-cpp    ./languages/cpp
docker build -t codequest-java   ./languages/java
docker build -t codequest-js     ./languages/javascript

python worker.py
```

### 4. Frontend Setup

```bash
cd frontend
npm install
cp .env.local.example .env.local  # Edit as needed
npm run dev                        # Starts on http://localhost:3000
```

## Default Credentials

After running `seed_data`:

| Role | Email | Password |
|------|-------|----------|
| Admin | admin@codequest.dev | admin123 |
| Pro user | pro@codequest.dev | user123 |
| Free user | free@codequest.dev | user123 |

RabbitMQ management UI: http://localhost:15672 (guest / guest)

## End-to-End Test Flows

### Flow 1: Submit a Solution
1. Login as free@codequest.dev
2. Go to `/problems` → pick "Two Sum"
3. Write a Python solution in the editor → click Submit
4. Watch status update live (QUEUED → RUNNING → ACCEPTED)

### Flow 2: Create a Problem (Admin)
1. Login as admin@codequest.dev
2. Go to `/admin/problems` → "New Problem"
3. Fill statement, add testcases, set tags → Publish
4. Verify it appears on `/problems`

### Flow 3: Contest Flow
1. Admin creates a contest at `/admin/contests`
2. Free user registers at `/contests/{slug}`
3. Submit solutions during contest window
4. Watch leaderboard update in real time at `/contests/{slug}/leaderboard`

### Flow 4: Free vs Pro Ad Behavior
1. Login as free@codequest.dev → observe left sidebar ad on problem pages
2. Login as pro@codequest.dev → no ads visible
3. Free user hits AI hint limit → sees upgrade prompt

## Service URLs

| Service | URL |
|---------|-----|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:8000/api/ |
| Django Admin (built-in) | http://localhost:8000/django-admin/ |
| RabbitMQ UI | http://localhost:15672 |
| PostgreSQL | localhost:5432 |
| Redis | localhost:6379 |

## Environment Variables

### backend/.env
```
SECRET_KEY=change-me-in-production
DEBUG=True
DATABASE_URL=postgresql://codequest:codequest@localhost:5432/codequest
REDIS_URL=redis://localhost:6379/0
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
JUDGE_INTERNAL_API_KEY=judge-secret-key-change-me
ALLOWED_HOSTS=localhost,127.0.0.1
CORS_ORIGINS=http://localhost:3000
```

### judge/.env
```
DATABASE_URL=postgresql://codequest:codequest@localhost:5432/codequest
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
BACKEND_API_URL=http://localhost:8000
JUDGE_INTERNAL_API_KEY=judge-secret-key-change-me
JUDGE_QUEUE=code_submissions
CPU_TIME_LIMIT=10
MEMORY_LIMIT_MB=256
```

### frontend/.env.local
```
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_WS_URL=ws://localhost:8000
```

## Scaling to Cloud

See `docs/architecture.md` for detailed cloud deployment guidance.

In short:
- **Frontend** → Vercel / CloudFront + S3
- **Backend** → ECS Fargate or K8s (multiple replicas behind ALB)
- **Judge** → ECS tasks or K8s Jobs, auto-scaled on queue depth
- **PostgreSQL** → Amazon RDS
- **Redis** → Amazon ElastiCache
- **RabbitMQ** → Amazon MQ or CloudAMQP
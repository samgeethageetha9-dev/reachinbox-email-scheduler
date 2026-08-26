# ReachInbox Hiring Assignment

This repository contains a full-stack email scheduler built with Express, PostgreSQL, Redis, BullMQ, and a React + Tailwind dashboard.

## Features implemented

### Backend
- Express + TypeScript API server
- PostgreSQL persistence via Prisma
- BullMQ job queue backed by Redis
- Delayed scheduling without cron jobs
- Ethereal Email fake SMTP sending
- Rate limiting enforced with Redis-backed counters
- Worker concurrency configuration via env vars
- Google OAuth authentication flow
- API endpoints to schedule, list, and inspect jobs
- Idempotency protection per recipient + subject + scheduled time

### Frontend
- Google login screen
- Dashboard header with user details and logout
- Scheduled Emails and Sent Emails tabs
- Compose New Email modal
- CSV/text lead upload and parser
- Loading states and empty states
- Basic error handling

---

## Local setup

### 1) Start Redis and PostgreSQL

```bash
docker compose up -d
```

### 2) Backend setup

```bash
cd backend
cp .env.example .env
npm install
npx prisma generate
npx prisma db push
npm run dev
```

The backend runs on:
- http://localhost:4000

### 3) Frontend setup

```bash
cd frontend
cp .env.example .env
npm install
npm run dev
```

The frontend runs on:
- http://localhost:5173

---

## Environment variables

### Backend

```env
PORT=4000
FRONTEND_URL=http://localhost:5173
SESSION_SECRET=replace_me
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/reachinbox
REDIS_URL=redis://localhost:6379
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_CALLBACK_URL=http://localhost:4000/auth/google/callback
ETHEREAL_USER=
ETHEREAL_PASS=
WORKER_CONCURRENCY=2
DELAY_BETWEEN_EMAILS_MS=2000
MAX_EMAILS_PER_HOUR=200
SMTP_FROM="ReachInbox <noreply@reachinbox.ai>"
```

### Frontend

```env
VITE_API_URL=http://localhost:4000
```

---

## Ethereal Email setup

Ethereal is used as a fake SMTP provider for development and testing.

1. Visit https://ethereal.email/
2. Create or login to a test account
3. Copy the SMTP username and password into the backend `.env` as `ETHEREAL_USER` and `ETHEREAL_PASS`
4. The application automatically creates a test account if those values are not configured, and the generated credentials are used for mail delivery

The app logs the preview URL from Ethereal for verification.

---

## Architecture overview

### Scheduling model

The scheduler uses BullMQ with Redis as the durable queue backend. When a job is scheduled, the backend stores a record in PostgreSQL and also enqueues a delayed BullMQ job with the recipient payload. The delay corresponds to the future send time. Because the queue is backed by Redis, jobs survive a backend restart and continue to execute when the delay expires.

### Persistence and restart safety

- The email record is saved to PostgreSQL before queueing.
- BullMQ stores delayed jobs in Redis.
- On server restart, Redis still keeps the job queue and delayed timestamps intact.
- The worker rebuilds the queue connection and resumes processing future jobs without resetting from Day 1.
- Idempotency is enforced with a deterministic hash based on sender + recipient + subject + send time, so the same logical send request is not queued twice.

### Rate limiting and concurrency

The system uses a Redis-backed hourly counter per sender. Each worker checks the current hourly bucket before sending. If the count is at or above the configured hourly threshold, the job is moved back into the delayed queue for the next available hour and is not dropped.

Configuration used:
- `WORKER_CONCURRENCY`: number of BullMQ worker threads
- `DELAY_BETWEEN_EMAILS_MS`: minimum gap between each email send (default: 2000ms)
- `MAX_EMAILS_PER_HOUR`: global per-sender daily/hourly budget (default: 200)

This avoids relying on in-memory counters, which would be lost on restart. The Redis lock ensures concurrent workers do not race and double-send under load.

### Behavior under load

When many jobs are queued for the same time window, the BullMQ worker processes them sequentially according to concurrency and delay settings. If the configured hourly send limit is reached, remaining jobs are pushed into the next available bucket rather than being permanently failed or lost. This preserves order as much as possible while keeping the system safe across multiple worker instances.

---

## API flow

### Authentication
- `GET /auth/google` starts Google OAuth login
- `GET /auth/google/callback` completes login and redirects to the dashboard
- `GET /api/auth/me` returns the authenticated user
- `POST /api/auth/logout` logs out the current user

### Scheduling
- `POST /api/jobs/schedule`
  Allowed payload:
  ```json
  {
    "subject": "Hello",
    "body": "This is an outreach email",
    "recipients": ["a@example.com", "b@example.com"],
    "sendAt": "2026-08-26T15:00:00.000Z",
    "sender": "team@company.com",
    "delayBetweenMs": 2000,
    "hourlyLimit": 200
  }
  ```

- `GET /api/jobs?status=scheduled|sent|failed|all`
  Returns the current jobs from PostgreSQL.

---

## Notes / trade-offs

- This project uses a simple Redis + BullMQ pattern for queue durability and rate limiting rather than cron-based scheduling.
- The app uses Ethereal Email for test delivery realism without external email credentials.
- Google OAuth is implemented as a real OAuth flow and requires the user to set valid `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` values.
- The scheduler keeps each recipient as its own job row, which makes the dashboard and tracking straightforward.

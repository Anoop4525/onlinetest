# Assignment & Testing Platform

A production-ready online assignment and candidate testing platform.

Administrators build timed assessments, assign them to candidates and issue a
one-time access password. Candidates sit the test one question at a time under
**server-controlled timers**, with answers saved automatically, and submit
exactly once. Results, answer review, integrity signals and feedback are
available to the administrator.

- **Frontend** — React 18, Vite, JSX, Tailwind CSS, React Router, Redux Toolkit,
  React Redux, Axios, React Icons
- **Backend** — Node.js, Express, REST, JavaScript (CommonJS)
- **Database** — MySQL 8 (`mysql2`, connection pooling, parameterised queries,
  SQL migrations)

---

## Table of contents

1. [Features](#features)
2. [Requirements](#requirements)
3. [Quick start](#quick-start)
4. [Environment variables](#environment-variables)
5. [Database setup](#database-setup)
6. [Running in development](#running-in-development)
7. [Production build & deployment](#production-build--deployment)
8. [Project structure](#project-structure)
9. [API reference](#api-reference)
10. [Security model](#security-model)
11. [Troubleshooting](#troubleshooting)

---

## Features

### Administrator

- Secure sign-in with HTTP-only cookie sessions and CSRF protection
- Dashboard with live, database-backed metrics and charts (submissions over
  time, creation trend, completion breakdown, score distribution)
- Test management: create, edit, view, duplicate, activate, close, archive,
  search, filter and paginate
- Visual question builder — unlimited questions, five question types, **per
  question time limits**, marks, required flag, reorder, duplicate, delete
- Candidate management with server-side search, filtering and pagination
- Guided *Assign Test* workflow that generates a cryptographically secure
  one-time password, shown exactly once with copy-to-clipboard
- Regenerate or disable access; a submitted test can never be reopened
- Results dashboard and per-attempt review including correct answers,
  per-question time spent and timeout status
- Feedback dashboard and a full audit log

### Candidate

- Sign in with the assigned email and the one-time password
- Confirm personal details (the assigned email is read-only)
- Read and acknowledge instructions before the timer starts
- One question at a time, with a **server-authoritative countdown**
- Automatic answer saving (debounced, periodic, on navigation and on page hide)
- Copy / paste / cut / drag deterrents inside the answer field
- Safe refresh and resume — reloading never restarts the test or grants time
- Automatic advance on question timeout and auto-submit on overall timeout
- Transactional, idempotent final submission with server-side scoring
- Post-test feedback, then a permanently closed session

---

## Requirements

| Software | Version |
| -------- | ------- |
| Node.js  | 18.18+ (20 LTS or newer recommended) |
| npm      | 9+ |
| MySQL    | 8.0+ (MariaDB 10.6+ also works) |

---

## Quick start

```bash
# 1. Install dependencies for both applications
npm run install:all

# 2. Create the environment files and fill them in
cp server/.env.example server/.env
cp client/.env.example client/.env

# 3. Generate two strong secrets and paste them into server/.env
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"

# 4. Create the schema (the database itself is created if missing)
npm run migrate

# 5. Create the first administrator from SEED_ADMIN_* in server/.env
npm run seed

# 6. Start the API and the React dev server together
npm run dev
```

Then open <http://localhost:5173/admin/login> (or <http://localhost:5021/admin/login> after `npm run build`).

---

## Environment variables

Both `.env` files are git-ignored. **Never commit real secrets**, and never put
database or JWT secrets in the client — anything prefixed `VITE_` is embedded in
the public JavaScript bundle.

### `server/.env`

| Variable | Required | Description |
| --- | --- | --- |
| `NODE_ENV` | yes | `development` or `production` |
| `PORT` | no | API port (default `5021`) |
| `TRUST_PROXY` | prod | Set to `1` behind Nginx so client IPs and secure cookies work |
| `DB_HOST` / `DB_PORT` | yes | MySQL host and port |
| `DB_USER` / `DB_PASSWORD` | yes | MySQL credentials |
| `DB_NAME` | yes | Database name (created by `npm run migrate` if absent) |
| `DB_CONNECTION_LIMIT` | no | Pool size (default `10`) |
| `CLIENT_URL` | yes in prod | Allowed browser origin, e.g. `https://assigmenttest.felinemedia.tech` |
| `CLIENT_URL_WWW` | no | Second allowed origin (the `www` host) |
| `CORS_ORIGINS` | no | Extra allowed origins, comma separated |
| `COOKIE_DOMAIN` | no | e.g. `.felinemedia.tech`; leave empty for host-only cookies |
| `JWT_SECRET` | yes | ≥32 random chars in production |
| `SESSION_SECRET` | yes | ≥32 random chars, **different** from `JWT_SECRET` |
| `ADMIN_TOKEN_TTL_HOURS` | no | Admin session lifetime (default `8`) |
| `CANDIDATE_SESSION_TTL_HOURS` | no | Candidate session lifetime (default `6`) |
| `SEED_ADMIN_NAME/EMAIL/PASSWORD` | for seeding | Initial administrator; the password needs ≥10 chars with upper, lower and a digit |
| `LOG_LEVEL` | no | `error`, `warn`, `info` or `debug` |
| `BCRYPT_ROUNDS` | no | Default `12` |

The server refuses to start on invalid configuration rather than serving
half-working endpoints.

### `client/.env`

| Variable | Description |
| --- | --- |
| `VITE_API_BASE_URL` | `/api` in development (Vite proxies it); the full origin in production, e.g. `https://assigmenttest.felinemedia.tech/api` |
| `VITE_DEV_API_TARGET` | Optional dev proxy target (default `http://localhost:5021`) |

---

## Database setup

You do **not** need to create tables by hand in phpMyAdmin.

```bash
npm run migrate          # create database (if needed) + apply all migrations
npm run migrate:status   # show applied / pending migrations
npm run seed             # create the initial administrator
```

The runner records applied files in `schema_migrations` and applies each file
exactly once, in filename order.

| # | Migration | Table |
| --- | --- | --- |
| 001 | `001_create_admins.sql` | `admins` |
| 002 | `002_create_tests.sql` | `tests` |
| 003 | `003_create_questions.sql` | `questions` |
| 004 | `004_create_question_options.sql` | `question_options` |
| 005 | `005_create_candidates.sql` | `candidates` |
| 006 | `006_create_test_assignments.sql` | `test_assignments` |
| 007 | `007_create_test_attempts.sql` | `test_attempts` |
| 008 | `008_create_candidate_answers.sql` | `candidate_answers` |
| 009 | `009_create_feedback.sql` | `feedback` |
| 010 | `010_create_sessions.sql` | `sessions` |
| 011 | `011_create_integrity_events.sql` | `test_integrity_events` |
| 012 | `012_create_audit_logs.sql` | `audit_logs` |

Key constraints that enforce the product rules at the storage layer:

- `test_attempts.assignment_id` is **UNIQUE** → one attempt per assignment
- `test_assignments (candidate_id, test_id)` is **UNIQUE** → one assignment of a
  test per candidate
- `candidate_answers (attempt_id, question_id)` is **UNIQUE** → safe upserts, no
  duplicate answers
- `feedback.attempt_id` is **UNIQUE** → one feedback record per attempt
- `questions (test_id, question_order)` is **UNIQUE** → deterministic ordering
- Assignments and attempts use `ON DELETE RESTRICT` so submitted history can
  never be cascade-deleted. Deleting a test that already has assignments
  archives it instead.

The MySQL user needs `CREATE, ALTER, INDEX, SELECT, INSERT, UPDATE, DELETE`
(plus `CREATE` on the server if you want the migration to create the database).

---

## Running in development

```bash
npm run dev       # API (:5021) + Vite dev server (:5173) together
npm run server    # API only
npm run client    # frontend only
npm run lint      # ESLint over the React source
```

Vite proxies `/api` to the API, so the browser sees a single origin and cookies
behave exactly as they do in production.

| Area | Vite dev server | Single-origin (Node serves `client/dist`) |
| --- | --- | --- |
| Admin | <http://localhost:5173/admin/login> | <http://localhost:5021/admin/login> |
| Candidate | <http://localhost:5173/login> | <http://localhost:5021/login> |
| Health check | — | <http://localhost:5021/api/health> |

Both work locally. The single-origin column matches production exactly: run
`npm run build` once, then `npm run server`, and browse the API's own port.

---

## Production build & deployment

### 1. Build the frontend

```bash
# client/.env
# VITE_API_BASE_URL=https://assigmenttest.felinemedia.tech/api
npm run build          # emits client/dist
```

### 2. Run the API

```bash
cd server
NODE_ENV=production npm start
```

Use a process manager so the API restarts on failure and on boot. A systemd
unit is provided at `deploy/assignment-api.service.example`; with PM2:

```bash
pm2 start server/src/server.js --name assignment-api --env production
pm2 save && pm2 startup
```

### 3. Reverse proxy

```
Browser ──HTTPS──> Nginx ─┬─ /     -> client/dist (SPA fallback)
                          └─ /api  -> 127.0.0.1:5021 (Express)
                                          |
                                        MySQL
```

A complete example is in `deploy/nginx.conf.example`. It terminates TLS, serves
the SPA with a history fallback, proxies `/api`, sets security headers and a
strict CSP, and redirects `www` to the canonical apex host.

### 4. Deploying an update

An update has **two halves, and they fail independently**. The frontend is a
folder of files, so copying it is enough. The API is a running process: new
source on disk changes nothing until that process is restarted, because Node
loaded the old modules into memory at boot.

Shipping only the first half is the usual cause of "it works locally but not on
the server" — the new screens arrive, the endpoints they call do not, and the
page renders with data the old API never sent.

```bash
# 1. Frontend - rebuild, then copy the result
cd client && npm run build
rsync -a --delete client/dist/ user@host:/var/www/assigmenttest/client/dist/

# 2. API - copy the source, install any new dependencies, run new migrations
rsync -a --delete --exclude node_modules --exclude .env --exclude uploads \
      server/ user@host:/var/www/assigmenttest/server/
ssh user@host 'cd /var/www/assigmenttest/server && npm ci --omit=dev && npm run migrate'

# 3. Restart the API  <-- the step that is easy to forget
ssh user@host 'sudo systemctl restart assignment-api'
```

`--exclude uploads` matters: candidate submissions and question images live
there, and `--delete` without it would wipe them.

Confirm both halves actually landed:

```bash
curl -s https://your-host/api/health          # environment must read "production"
curl -s https://your-host/ | grep -o 'assets/index-[^"]*'   # hash matches your local build
```

If the API half did not restart, `/api/health` still answers and the site still
loads — nothing looks broken, which is exactly why this one is worth checking
explicitly after every deploy.

### Production checklist

- [ ] `NODE_ENV=production` and `TRUST_PROXY=1`
- [ ] `JWT_SECRET` and `SESSION_SECRET` are long, random and different
- [ ] `CLIENT_URL` (and `CLIENT_URL_WWW`) list the real origins — CORS is an
      allowlist and never `*`
- [ ] HTTPS is active, so cookies are issued with `Secure`
- [ ] `server/.env` is `chmod 600` and owned by the service user
- [ ] `npm run migrate` has been run against the production database
- [ ] Database backups are scheduled

Both production hosts are supported:

- `https://assigmenttest.felinemedia.tech`
- `https://www.assigmenttest.felinemedia.tech`

---

## Project structure

```
assignment-test-platform/
├── client/
│   ├── public/
│   ├── src/
│   │   ├── components/{common,admin,candidate}/
│   │   ├── features/          # Redux Toolkit slices
│   │   │   ├── auth/ tests/ questions/ candidates/
│   │   │   └── assignments/ attempt/ results/ feedback/ dashboard/ ui/
│   │   ├── hooks/             # useServerTimer, useAutoSave, useCopyPasteBlock…
│   │   ├── layouts/           # AdminLayout, CandidateLayout
│   │   ├── pages/{admin,candidate}/
│   │   ├── routes/            # route table + guards
│   │   ├── services/          # axios instance + endpoint wrappers
│   │   ├── store/             # configureStore
│   │   ├── utils/
│   │   ├── App.jsx  main.jsx  index.css
│   ├── .env.example  vite.config.js  tailwind.config.js
│
├── server/
│   ├── src/
│   │   ├── config/            # env, db pool, logger
│   │   ├── controllers/{admin,candidate}/
│   │   ├── services/          # business rules
│   │   ├── repositories/      # parameterised SQL only
│   │   ├── middleware/        # auth, csrf, rate limits, validation, errors
│   │   ├── validators/        # Zod schemas
│   │   ├── routes/            # adminRoutes, candidateRoutes
│   │   ├── migrations/        # 001…012 .sql
│   │   ├── seeders/  scripts/ utils/
│   │   └── app.js  server.js
│   └── .env.example
│
├── deploy/                    # nginx + systemd examples
├── package.json               # workspace scripts
└── README.md
```

Responsibilities are strictly separated: controllers handle HTTP, services hold
business rules, repositories own SQL, middleware handles auth/validation/limits.

---

## API reference

All responses share one envelope:

```jsonc
// success
{ "success": true, "message": "Test created successfully.", "data": { } }

// list endpoints add pagination
{ "success": true, "data": [], "pagination": { "page": 1, "limit": 20, "total": 154, "totalPages": 8 } }

// error — machine-readable code, no stack traces in production
{ "success": false, "code": "TEST_ALREADY_SUBMITTED", "message": "Your test has already been submitted." }
```

### Admin

| Method | Endpoint |
| --- | --- |
| `POST` | `/api/admin/auth/login` |
| `POST` | `/api/admin/auth/logout` |
| `GET` | `/api/admin/auth/me` |
| `PUT` | `/api/admin/auth/profile` · `/api/admin/auth/password` |
| `GET` | `/api/admin/dashboard/summary` · `/api/admin/dashboard/charts` |
| `GET/POST` | `/api/admin/tests` |
| `GET/PUT/DELETE` | `/api/admin/tests/:id` |
| `PATCH` | `/api/admin/tests/:id/status` |
| `POST` | `/api/admin/tests/:id/duplicate` |
| `POST` | `/api/admin/tests/:testId/questions` · `/questions/reorder` |
| `PUT/DELETE` | `/api/admin/questions/:id` |
| `GET/POST` | `/api/admin/candidates` |
| `GET/PUT` | `/api/admin/candidates/:id` |
| `GET/POST` | `/api/admin/assignments` |
| `GET` | `/api/admin/assignments/:id` |
| `POST` | `/api/admin/assignments/:id/regenerate-password` · `/disable` · `/enable` |
| `PUT` | `/api/admin/assignments/:id/access` |
| `GET` | `/api/admin/results` · `/api/admin/results/:attemptId` |
| `GET` | `/api/admin/feedback` · `/api/admin/feedback/:id` |
| `GET` | `/api/admin/activity` · `/api/admin/activity/actions` |

### Candidate

| Method | Endpoint |
| --- | --- |
| `POST` | `/api/candidate/login` · `/api/candidate/logout` |
| `GET` | `/api/candidate/me` |
| `PUT` | `/api/candidate/details` |
| `GET` | `/api/candidate/test` |
| `POST` | `/api/candidate/test/start` |
| `GET` | `/api/candidate/test/current-question` |
| `PUT` | `/api/candidate/test/answers` |
| `POST` | `/api/candidate/test/next` |
| `GET` | `/api/candidate/test/review` |
| `POST` | `/api/candidate/test/submit` |
| `POST` | `/api/candidate/test/integrity-event` |
| `POST` | `/api/candidate/feedback` |

`GET /api/candidate/test/current-question` returns only what the candidate is
allowed to see — never a correct answer, a future question, scoring metadata or
another candidate's data:

```json
{
  "attemptId": 125,
  "questionNumber": 3,
  "totalQuestions": 10,
  "question": { "id": 25, "text": "…", "type": "LONG_ANSWER", "timeLimitSeconds": 300, "marks": 10 },
  "remainingSeconds": 247
}
```

Common error codes: `INVALID_CREDENTIALS`, `TEST_ALREADY_SUBMITTED`,
`TEST_ACCESS_EXPIRED`, `ACCESS_NOT_STARTED`, `ACCESS_DISABLED`,
`TEST_NOT_AVAILABLE`, `QUESTION_TIME_EXPIRED`, `QUESTION_CHANGED`,
`SESSION_EXPIRED`, `CSRF_TOKEN_INVALID`, `VALIDATION_ERROR`, `RATE_LIMITED`.

---

## Security model

**The browser is never trusted.** Score, timers, question sequence, attempt
ownership, submission state and role are all resolved from authenticated server
state on every request.

- **Passwords** — bcrypt (12 rounds) for administrators and for candidate access
  passwords. Only hashes are stored. The generated test password is returned in
  exactly one response and never again.
- **Credential generation** — Node's `crypto` CSPRNG with rejection sampling for
  a uniform distribution; 12 characters with guaranteed upper/lower/digit and no
  ambiguous glyphs (`0/O`, `1/l/I`).
- **Sessions** — administrators use a signed JWT in an HTTP-only cookie;
  candidates use an opaque token whose **HMAC hash** is stored, so a leaked
  database cannot be used to forge a session. Sessions are revocable, and only
  one candidate session per assignment can be live at a time.
- **CSRF** — double-submit token: a JS-readable cookie must be echoed in the
  `X-CSRF-Token` header on every state-changing request.
- **One attempt per assignment** — enforced by a UNIQUE index, a row lock and
  service-level checks. Regenerating a password cannot reopen a submitted test.
- **Idempotent submission** — the submit endpoint runs in a transaction with
  `SELECT … FOR UPDATE`; a repeat request returns the existing result rather than
  creating a second one.
- **Server-side timing** — every deadline is derived from database timestamps.
  A question's remaining time is `limit − accumulated − elapsed-since-entry`,
  so refreshing never grants extra time. Expiry, auto-advance and auto-submit are
  all decided by the server; the browser countdown is purely visual.
- **SQL injection** — every query is parameterised. The only interpolated values
  are integer-clamped `LIMIT`/`OFFSET` and an allowlist-validated sort column.
  `LIKE` wildcards in search terms are escaped.
- **XSS** — all user content renders as React text. `dangerouslySetInnerHTML` is
  not used anywhere.
- **Transport** — Helmet, HSTS in production, a strict CORS allowlist (never
  `*`), a 256 KB body limit, and compression.
- **Rate limiting** — strict on sign-in and credential operations, deliberately
  generous on autosave so a candidate never loses an answer.
- **Logging** — passwords, hashes, tokens and CSRF values are redacted. Clients
  get a generic message on 5xx; details stay in the server log.
- **Copy/paste blocking** is a deterrent scoped to the answer field, not a
  security control, and it does not affect keyboard accessibility elsewhere.

---

## Troubleshooting

**`Invalid environment configuration`** — a required variable is missing. The
message lists exactly which ones.

**`ECONNREFUSED` / `DATABASE_UNAVAILABLE` on startup** — MySQL is not reachable.
Check that the server is running and that `DB_HOST`, `DB_PORT`, `DB_USER`,
`DB_PASSWORD` are correct. The API deliberately refuses to start without a
working database connection.

**`Access denied for user`** — the MySQL user lacks privileges, or lacks
`CREATE` if you expect the migration to create the database. Create it manually
and re-run `npm run migrate`.

**Admin login returns 401 immediately** — run `npm run seed` first. If it says
the admin already exists, the password in `.env` is not what is stored; change
it from *Settings* after signing in.

**Login succeeds but the next request is 401** — a cookie problem. In production
make sure HTTPS is active, `TRUST_PROXY=1` is set, `CLIENT_URL` matches the
browser origin exactly (scheme + host, no trailing slash) and, if the API is on
a different subdomain, `COOKIE_DOMAIN` is set.

**`CSRF_TOKEN_INVALID`** — the `atp_csrf` cookie was not sent back. This is
normally a third-party-cookie or origin mismatch; serve the API and the SPA from
the same origin as in the sample Nginx configuration.

**CORS errors in the browser console** — the origin is not in the allowlist. Add
it to `CLIENT_URL`, `CLIENT_URL_WWW` or `CORS_ORIGINS` and restart the API.

**Candidate sees "Test already submitted"** — expected behaviour. One assignment
allows exactly one attempt. To let someone retake a test, assign the test again
(which issues a fresh assignment and password).

**The question timer looks wrong after a refresh** — it is recalculated from the
server. The displayed value is the authoritative remaining time; the local
countdown only animates between requests.

**A 404 on refresh of a deep admin route** — the SPA history fallback is
missing. Ensure `try_files $uri $uri/ /index.html;` is present.

# onlinetest

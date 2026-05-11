# Bank Licensing & Compliance Portal

A portal for the National Bank of Rwanda to manage the licensing of commercial banks and financial institutions. The existing process runs entirely on email and spreadsheets: no audit trail, no single source of truth, no visibility into where an application is at any given moment. This replaces it end to end.

**Live demo:** http://49.12.66.239:4000

**Stack:** Node.js + Express backend, PostgreSQL, React 19 + Redux Toolkit frontend (Vite).

Supplementary reading:
- [`docs/DESIGN_DOCUMENT.md`](docs/DESIGN_DOCUMENT.md): architecture decisions, data model, state machine, roles
- [`docs/bnr_api.postman_collection.json`](docs/bnr_api.postman_collection.json): complete API collection

## Table of Contents

1. [Docker (recommended)](#docker-recommended)
2. [Manual setup](#manual-setup-no-docker)
3. [Demo accounts](#demo-accounts)
4. [Running the test suite](#running-the-test-suite)
5. [API documentation](#api-documentation)
6. [Repository structure](#repository-structure)


## Docker (recommended)

This is the quickest way to get the system running. You need Docker Desktop and nothing else, no local Node or PostgreSQL required.

### 1. Install and start Docker Desktop

Download for your OS:

| OS | Link |
|:---|:-----|
| macOS | https://docs.docker.com/desktop/install/mac-install/ |
| Windows | https://docs.docker.com/desktop/install/windows-install/ |
| Linux | https://docs.docker.com/desktop/install/linux/ |

After installing, open Docker Desktop and wait for the whale icon in your taskbar to stop animating. That means the daemon is up. You can double-check with:

```bash
docker info
```

If it prints engine details rather than an error, you're good to go.

### 2. Clone with submodules

The backend and frontend live in separate git submodules under `moduleRepos/`. You need `--recurse-submodules` or they'll be empty directories.

```bash
git clone --recurse-submodules https://github.com/ichristine180/bank-licensing-compliance-portal.git
cd bank-licensing-compliance-portal
```

Already cloned without the flag? Fix it with:

```bash
git submodule update --init --recursive
```

### 3. Start everything

```bash
docker-compose up --build
```

This starts three services in the right order:

1. **PostgreSQL 16**: creates `bnr_compliance` (app) and `bnr_compliance_test` (tests)
2. **Backend**: applies the schema, sets up the `bnr_app` database role with scoped permissions, then seeds demo data before the API starts listening
3. **Frontend**: Vite dev server, proxied to the backend

The first run pulls images and installs npm dependencies, so give it about two minutes. Restarts after that are fast.

### 4. Open the app

| Service | URL |
|:--------|:----|
| Frontend | http://localhost:4000 |
| API | http://localhost:3000/api |

Log in with any of the [demo accounts](#demo-accounts) below.

### 5. Run the tests

```bash
docker-compose exec backend npm test
```

The container already has a `.env.test` pointing at the `bnr_compliance_test` database, so no extra setup is needed.

### 6. Other useful commands

```bash
# Re-run the seed without restarting
docker-compose exec backend node scripts/seed.js

# Stop but keep your data volumes
docker-compose down

#stops everything and wipes the database volumes
docker-compose down -v
```

> **Heads up:** If you restart without `-v` and find that login doesn't work for the seeded accounts, the database volume is likely in a state from a previous run. Running `docker-compose down -v && docker-compose up --build` gives you a clean slate.


## Manual setup (no Docker)

You'll need Node.js 18+, PostgreSQL 14+, and npm.

### 1. Clone with submodules

```bash
git clone --recurse-submodules https://github.com/ichristine180/bank-licensing-compliance-portal.git
cd bank-licensing-compliance-portal
```

Or if you already cloned:

```bash
git submodule update --init --recursive
```

### 2. Backend

From `moduleRepos/backend/`:

```bash
cd moduleRepos/backend
npm install
```

**Create the two databases.** One for the app, one for tests:

```bash
psql -U postgres -c "CREATE DATABASE bnr_compliance;"
psql -U postgres -c "CREATE DATABASE bnr_compliance_test;"
```

**Create `.env`** in `moduleRepos/backend/`:

```env
DATABASE_ADMIN_URL=postgres://postgres:yourpassword@localhost:5432/bnr_compliance
DATABASE_URL=postgres://bnr_app:yourapppassword@localhost:5432/bnr_compliance
JWT_SECRET=replace_with_a_long_random_string_at_least_32_chars
JWT_EXPIRES_IN=30m
REFRESH_TOKEN_EXPIRES_IN=7d
```

**Create `.env.test`** in the same directory (points at the test database):

```env
DATABASE_ADMIN_URL=postgres://postgres:yourpassword@localhost:5432/bnr_compliance_test
DATABASE_URL=postgres://bnr_app:yourapppassword@localhost:5432/bnr_compliance_test
JWT_SECRET=test_secret
JWT_EXPIRES_IN=30m
REFRESH_TOKEN_EXPIRES_IN=7d
```

A few things worth knowing:
- `DATABASE_ADMIN_URL` uses your superuser. It's only needed at startup to apply the schema and manage grants; the app never uses it at runtime.
- `DATABASE_URL` connects as `bnr_app`, a limited role the startup script creates automatically on first run.
- Special characters in passwords (like `@`) need to be URL-encoded; `@` becomes `%40`.

**Seed the database.** This applies the schema, creates reference data, demo accounts, and two sample applications:

```bash
npm run seed
```

**Start the backend:**

```bash  
npm start    
```

The API listens on `http://localhost:3000`. Schema application runs on every startup and is fully idempotent, so there's no separate migration step to worry about.

### 3. Frontend

From `moduleRepos/frontend/`:

```bash
cd moduleRepos/frontend
npm install
npm run dev
```

The frontend starts at `http://localhost:4000`. All `/api` requests are proxied to `http://localhost:3000` by the Vite dev server config, so no extra environment variables are needed for local development.


## Demo accounts

The seed script creates one account per role. All of these work immediately after seeding.

| Role | Email | Password |
|:-----|:------|:---------|
| ADMIN | admin@bnr.rw | Admin@2026 |
| REVIEWER | reviewer@bnr.rw | Reviewer@2026 |
| APPROVER | approver@bnr.rw | Approver@2026 |
| APPLICANT | applicant@rtn.rw | Applicant@2026 |

Two sample applications come pre-loaded so you can see the workflow without having to submit one manually:

| Institution | Status |
|:------------|:-------|
| Kigali Commercial Bank Ltd | DRAFT |
| Rwanda Microfinance Cooperative | UNDER_REVIEW (already assigned to the seeded REVIEWER) |

> **Note on creating staff accounts:** `/api/auth/register` only accepts APPLICANT registrations by design. Adding a REVIEWER, APPROVER, or ADMIN account through the API requires an admin user-management endpoint that I didn't get to within the time available for this submission. The seeded accounts above cover all staff workflows.


## Running the test suite

**With Docker (services already running):**

```bash
docker-compose exec backend npm test
```

**Without Docker**, from `moduleRepos/backend/` with `.env.test` in place:

```bash
npm test
```

### What's tested

85 tests across 4 suites. All of them are integration tests running against a real PostgreSQL instance (`bnr_compliance_test`) with no database mocking. The test runner applies the schema on startup and truncates every table before each individual test.

| Suite | Tests | What it covers |
|:------|------:|:---------------|
| `auth.test.js` | 19 | Registration validation, login, token rotation, logout |
| `middleware.test.js` | 14 | JWT verification, role enforcement, self-approval block |
| `applications.test.js` | 22 | CRUD, role-based visibility, edit restrictions by status |
| `stateMachine.test.js` | 29 | All 7 state transitions, guard conditions, concurrent claim |

The concurrent access test in `stateMachine.test.js` is worth calling out specifically: it fires two simultaneous `POST /applications/:id/start-review` requests against the same application via `Promise.all` and asserts that exactly one gets a 200 and the other gets a 400. This is what verifies the `SELECT FOR UPDATE` locking actually works under real concurrency, not just in theory.


## API documentation

The Postman collection at `docs/bnr_api.postman_collection.json` covers every endpoint.

Import it into Postman, then set the `base_url` collection variable to `http://localhost:3000/api` (same whether you're running Docker or manually).

The login requests are set up to auto-populate `access_token` and `refresh_token` into collection variables. Create-application and document-upload requests do the same for `application_id` and `document_id`, so you can run them in sequence without copying values between requests.


## Repository structure

This is a monorepo with the backend and frontend as separate git submodules.

```
bank-licensing-compliance-portal/
├── docs/
│   ├── DESIGN_DOCUMENT.md               # Architecture, data model, state machine, roles
│   ├── data_model.png                   # ER diagram
│   └── bnr_api.postman_collection.json  # API collection
│
├── docker-compose.yml                   # Wires up db, backend, frontend
├── docker/
│   └── postgres-init/01-init.sql        # Creates bnr_compliance_test on first boot
│
├── moduleRepos/
│   ├── backend/
│   │   ├── src/
│   │   │   ├── db/          # Schema SQL, connection pool, startup init
│   │   │   ├── middleware/  # JWT auth, role guards
│   │   │   ├── models/      # Query layer, no business logic here
│   │   │   ├── routes/      # Route definitions
│   │   │   └── services/    # Business logic, state machine transitions
│   │   ├── scripts/seed.js  # Idempotent seed
│   │   ├── tests/           # 85 integration tests
│   │   └── Dockerfile
│   │
│   └── frontend/
│       ├── src/
│       │   ├── features/    # Redux slices (auth, applications, admin)
│       │   ├── pages/       # Applicant and admin views
│       │   ├── services/    # Axios client with token refresh interceptor
│       │   └── utils/       # Role helpers, route guards
│       └── Dockerfile
│
└── README.md
```

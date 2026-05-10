# Bank Licensing & Compliance Portal

A full-stack portal for the National Bank of Rwanda (BNR) to manage the licensing of commercial banks and financial institutions. Replaces a manual email-and-spreadsheet process with a structured workflow, enforced role separation, and an append-only audit trail.

---

## Repository Structure

This is a monorepo using Git submodules.

```
bank-licensing-compliance-portal/
├── docs/                        # Design document
├── moduleRepos/
│   ├── backend/                 # Node.js + Express API (PostgreSQL)
│   └── frontend/                # React frontend (see note below)
└── README.md
```

## Prerequisites

- Node.js 18+
- PostgreSQL 14+
- npm

---

## Backend Setup

All commands below run from `moduleRepos/backend/` unless stated otherwise.

### 1. Clone the repo

```bash
git clone --recurse-submodules https://github.com/ichristine180/bank-licensing-compliance-portal.git
cd bank-licensing-compliance-portal/moduleRepos/backend
```

If you already cloned without `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

### 2. Install dependencies

```bash
npm install
```

### 3. Create the databases

You need two PostgreSQL databases: one for the application and one for tests. The application also needs a dedicated runtime user with limited permissions (the setup script handles the grants automatically).

```bash
psql -U postgres -c "CREATE DATABASE bnr_compliance;"
psql -U postgres -c "CREATE DATABASE bnr_compliance_test;"
```

### 4. Configure environment variables

Create a `.env` file in `moduleRepos/backend/`:

```
DATABASE_ADMIN_URL=postgres://postgres:yourpassword@localhost:5432/bnr_compliance
DATABASE_URL=postgres://bnr_app:yourapppassword@localhost:5432/bnr_compliance
JWT_SECRET=replace_with_a_long_random_string
JWT_EXPIRES_IN=30m
REFRESH_TOKEN_EXPIRES_IN=7d
```

Create a `.env.test` file in the same directory for the test database:

```
DATABASE_ADMIN_URL=postgres://postgres:yourpassword@localhost:5432/bnr_compliance_test
DATABASE_URL=postgres://bnr_app:yourapppassword@localhost:5432/bnr_compliance_test
JWT_SECRET=test_secret
JWT_EXPIRES_IN=30m
REFRESH_TOKEN_EXPIRES_IN=7d
```

A few things worth noting:
- `DATABASE_ADMIN_URL` uses your PostgreSQL superuser. This is only used at startup to apply the schema and manage permissions.
- `DATABASE_URL` uses `bnr_app`, a limited role the startup script creates automatically. The application runs as this user at runtime.
- If your password contains special characters (like `@`), URL-encode them (e.g. `@` becomes `%40`).

### 5. Apply the schema and seed the database

The seed script applies the schema first and then inserts all reference and test data in one step:

```bash
npm run seed
```

This creates:
- 10 document types (8 mandatory, 2 optional) required for a BNR banking license
- One user per role (APPLICANT, REVIEWER, APPROVER, ADMIN)
- Two sample applications in different workflow states

**Seed credentials:**

| Role      | Email                   | Password        |
|:----------|:------------------------|:----------------|
| ADMIN     | admin@bnr.rw            | Admin@2026      |
| REVIEWER  | reviewer@bnr.rw         | Reviewer@2026   |
| APPROVER  | approver@bnr.rw         | Approver@2026   |
| APPLICANT | applicant@rtn.rw   | Applicant@2026  |

> **Note on staff accounts:** The `/api/auth/register` endpoint only creates APPLICANT accounts. Creating REVIEWER, APPROVER, and ADMIN accounts through the API requires an admin user management endpoint that was not implemented in this submission due to time constraints. Use the seeded accounts above to test staff workflows.

### 6. Start the server

```bash
npm run dev       # development (nodemon, restarts on file changes)
npm start         # production
```

The server starts on `http://localhost:3000`. The schema is re-applied on every startup (idempotent), so there is no separate migration step.

---

## Running Tests

```bash
npm test
```

Tests run against `bnr_compliance_test` using the credentials in `.env.test`. The test runner:
- Applies the schema fresh on startup
- Truncates all tables before each test
- Does not mock the database

**Current coverage: 85 tests across 4 suites.**

| Suite                  | Tests | What it covers |
|:-----------------------|------:|:---------------|
| auth.test.js           |    19 | Registration, login, token rotation, logout |
| middleware.test.js     |    14 | Token verification, role enforcement, self-approval block |
| applications.test.js   |    22 | CRUD operations, role-based visibility |
| stateMachine.test.js   |    29 | All 7 transitions, guard conditions, concurrent claim |

The concurrent access test explicitly demonstrates `SELECT FOR UPDATE` locking — two simultaneous `start-review` requests on the same application, asserting exactly one 200 and one 400.

---

## API Documentation

A Postman collection is at `moduleRepos/backend/docs/bnr_api.postman_collection.json`.

Import it into Postman and set the `base_url` variable to `http://localhost:3000/api`. The login requests auto-populate `access_token` and `refresh_token` variables, and create/upload requests auto-populate `application_id` and `document_id`.



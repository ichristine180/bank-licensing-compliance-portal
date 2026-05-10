# Bank Licensing & Compliance Portal: Design Document

## The Problem

BNR's licensing process runs through email threads, spreadsheets, and informal sign-offs. A decision that determines whether an institution can operate in Rwanda gets made through a chain of messages with no reliable record of who reviewed what or when. When a decision is disputed (and in banking, they do get disputed), there is nothing to point to.

Three things the current process doesn't do: it doesn't show anyone where an application stands without asking the person managing it, it doesn't enforce who can make decisions and in what order, and it leaves no trail that could survive scrutiny. This portal addresses all three, though the audit trail and workflow correctness were the ones I wasn't willing to compromise on. The rest was shaped around getting those right.

---

## Decisions I Made (and Why)

I'm putting this early because these decisions shaped everything else in the design. If you disagree with any of them, the rest of the document will make more sense once you know what I was optimizing for.

**Monolith over services.** I thought about splitting the audit writes into a separate service, give it its own database, its own write path. But that creates a distributed consistency problem: if the workflow update succeeds and the audit write fails, you have diverged state and history, which is catastrophically bad in a regulatory context. A monolith keeps both writes in the same transaction. Simple, correct, easy to reason about. Deployment will get more unwieldy as the codebase grows, but that's a future problem and not one I want to solve prematurely.

**JWT over sessions.** The revocation gap bothers me a little. A logout kills the refresh token but the access token runs until expiry, up to 30 minutes. For an internal tool used during business hours by a small team, I'm comfortable with that. Sessions would fix it but require shared server-side state, which complicates anything beyond a single instance. If this goes to production at scale, a Redis blocklist for access tokens is the next step.

**Separate REVIEWER and APPROVER roles.** The spec has a hard rule: the person who reviews cannot be the same person who approves. I could have made one "staff" role and checked `reviewer_id != user.id` everywhere an approver acts, but that's asking for trouble because there are too many places to forget that check and no compiler to catch the miss. Separate roles make it structural: a REVIEWER account physically cannot reach the approval endpoints. The ADMIN role is also excluded from workflow actions intentionally. Giving ADMIN the ability to approve applications creates a privileged back channel that's difficult to audit.

**Self-registration is APPLICANT only.** Reviewer, approver, and admin accounts must be created by an existing admin. There's no public API path to a privileged role. One thing to watch: the `/register` endpoint will need rate limiting before it sees real traffic. I haven't added that yet and it's a gap worth noting.

**Reviewer assignment survives resubmit.** When an applicant resubmits after being asked for more information, `reviewer_id` stays unchanged. I went back and forth on this. You could argue the application should go back to the pool, but re-pooling means another reviewer picks it up with no context, and the reviewer who asked for the information is the one best positioned to evaluate whether it was actually provided. Keeps the implementation simpler too, since no re-claim step is needed.

**Document types are predefined, not free-text.** Letting applicants type their own document type labels creates inconsistency that makes a reviewer's job harder — "Cert of Inc", "Certificate of Incorporation", and "Incorporation Certificate" are all the same thing but look different in the database. The `document_types` table seeds a controlled vocabulary of 10 types realistic for a BNR banking license. The upload endpoint validates the submitted type against this table and returns a 400 with a pointer to `GET /api/document-types` if it doesn't match. The `mandatory` flag is present and seeded but not enforced as a hard blocker on submission yet. Full enforcement would check that every mandatory type has an `is_current = true` document before allowing submit, and would surface the missing types to the applicant in the API response. Admin management of the type list (adding new types, changing mandatory status, deprecating old ones) is the obvious next piece but was left out of this submission. The schema supports it cleanly; it's just a CRUD endpoint away.

**Admin user management is not implemented.** The `/api/auth/register` endpoint only creates APPLICANT accounts. Creating staff accounts (REVIEWER, APPROVER, ADMIN) through the API requires a separate admin-only endpoint — something like `POST /api/admin/users` — that I didn't get to in this submission. The schema, role system, and middleware all support it; it's a CRUD endpoint with an `requireRole('ADMIN')` guard. For testing, the seed script creates one account per role with known credentials. That covers everything  needed to exercise the full workflow. 
**Comments schema is in but the endpoints aren't.** Internal reviewer notes and applicant-facing messaging are obviously useful in a real licensing workflow, but building them out would have meant shipping a less-tested state machine. The `application_comments` table is in the schema. The API is not.

---

## Architecture

React frontend, Node.js/Express API, PostgreSQL. The API is the only way in. No direct database access from the browser.

```
Browser (React)
    |  HTTPS + JSON
    v
Express API (Node.js)
    |  Routes -> Middleware -> Services -> Models
    v
PostgreSQL
    |  bnr_app (runtime, limited grants)
    |  admin user (schema init only)
    v
Local Filesystem (document storage)
```

I'm fairly strict about keeping routes thin. Routes parse the request and hand off to a service. If there's a conditional in a route handler that isn't just "call service, return result", that logic belongs somewhere else. Services hold all the business rules. Models are SQL only. This discipline matters when you have to change a transition rule later: you change it in one place, not scattered across route handlers.

Documents land on the local filesystem under UUID filenames. All the metadata lives in PostgreSQL. The `file_path` column is designed to be swappable for an S3 key in production without touching anything else.

---

## Data Model

Seven tables. I'll call out the non-obvious design choices inline; the schema itself is fairly readable.

### users

```sql
id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
email         TEXT UNIQUE NOT NULL,
password_hash TEXT NOT NULL,
full_name     TEXT NOT NULL,
role          TEXT NOT NULL,  -- APPLICANT | REVIEWER | APPROVER | ADMIN
organization  TEXT,
is_active     BOOLEAN DEFAULT true,
created_at    TIMESTAMPTZ DEFAULT now()
```

Accounts are deactivated rather than deleted. Audit entries reference `actor_id`, and deleting a user would orphan that history. A reviewer who leaves BNR should lose login access, not disappear from the record of everything they approved or rejected.

### applications

```sql
id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
applicant_id        UUID NOT NULL REFERENCES users(id),
institution_name    TEXT NOT NULL,
institution_type    TEXT NOT NULL,
registration_number TEXT,
license_type        TEXT NOT NULL,
contact_email       TEXT,
contact_phone       TEXT,
status              TEXT NOT NULL DEFAULT 'DRAFT',
reviewer_id         UUID REFERENCES users(id),  -- assigned once, stays through resubmits
approver_id         UUID REFERENCES users(id),  -- set only at final decision
review_notes        TEXT,
decision_reason     TEXT,
submitted_at        TIMESTAMPTZ,
decided_at          TIMESTAMPTZ,
created_at          TIMESTAMPTZ DEFAULT now(),
updated_at          TIMESTAMPTZ DEFAULT now()
```

Valid statuses: `DRAFT`, `SUBMITTED`, `UNDER_REVIEW`, `ADDITIONAL_INFO_REQUIRED`, `REVIEWED`, `APPROVED`, `REJECTED`.

Status is stored, not derived. `reviewer_id` and `approver_id` being separate columns is what makes the four-eyes check possible. The service compares them before any approval or rejection commits. A `set_updated_at` trigger keeps `updated_at` current automatically.

### document_types

```sql
id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
name        varchar(100) UNIQUE NOT NULL,
description text         NOT NULL,
mandatory   boolean      DEFAULT false,
created_at  timestamptz  DEFAULT now()
```

This table is the reference list of document types BNR accepts for a licensing application. The upload endpoint validates the submitted `document_type` against this table and rejects anything not on the list. Applicants call `GET /api/document-types` to get the list before uploading, so the frontend can render a proper dropdown rather than a free-text field.

Eight types are seeded as mandatory (Certificate of Incorporation, Memorandum and Articles of Association, Business Plan, Audited Financial Statements, Proof of Minimum Capital, Fit and Proper Declaration, AML/CFT Policy Document, Source of Funds Declaration) and two as optional (Organizational Chart, IT Systems and Infrastructure Plan). The mandatory flag is informational in this submission. The intended behavior with more time is described in the decisions section.

### documents

```sql
id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
application_id  UUID NOT NULL REFERENCES applications(id),
uploaded_by     UUID NOT NULL REFERENCES users(id),
original_name   TEXT NOT NULL,
stored_name     TEXT NOT NULL,  -- UUID-based, safe for the filesystem
file_path       TEXT NOT NULL,
file_size       BIGINT NOT NULL,
mime_type       TEXT NOT NULL,
document_type   TEXT NOT NULL,
version         INTEGER NOT NULL DEFAULT 1,
is_current      BOOLEAN DEFAULT true,
uploaded_at     TIMESTAMPTZ DEFAULT now()
```

Nothing is deleted. On resubmit, existing rows flip to `is_current = false` and new uploads come in as `is_current = true`. The submit and resubmit guards check only `is_current = true`. You can't satisfy the document requirement with files from a previous cycle that were already reviewed. Both generations stay in the database permanently, which matters if a decision is ever challenged and the reviewer needs to show what documents they actually had.

The 5MB limit is enforced by Multer's `limits.fileSize` before the file reaches the route handler. The application layer never sees an oversized file.

### audit_log

```sql
id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
application_id   UUID NOT NULL REFERENCES applications(id),
actor_id         UUID NOT NULL REFERENCES users(id),
actor_role       TEXT NOT NULL,
action           TEXT NOT NULL,
previous_status  TEXT,
new_status       TEXT,
previous_state   JSONB,   -- full snapshot of the row before the change
new_state        JSONB,   -- full snapshot after
metadata         JSONB,   -- reason strings, extra context per action
created_at       TIMESTAMPTZ DEFAULT now()
```

Every audit write happens in the same transaction as the state change. `previous_state` and `new_state` are full JSONB snapshots. Given just this table, you can reconstruct the complete history of any application without touching anything else. The spec asks for this and JSONB makes it straightforward.

### application_comments

```sql
id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
application_id  UUID NOT NULL REFERENCES applications(id),
author_id       UUID NOT NULL REFERENCES users(id),
author_role     TEXT NOT NULL,
content         TEXT NOT NULL,
is_internal     BOOLEAN DEFAULT false,  -- true = BNR staff only
created_at      TIMESTAMPTZ DEFAULT now()
```

Schema in, endpoints out. See the decisions section.

### refresh_tokens

```sql
id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
user_id     UUID NOT NULL REFERENCES users(id),
token_hash  TEXT UNIQUE NOT NULL,
expires_at  TIMESTAMPTZ NOT NULL,
created_at  TIMESTAMPTZ DEFAULT now()
```

Raw token goes to the client, SHA-256 hash is stored. On each refresh, a new token is issued and the old one is invalidated immediately.

---

## Roles and Permissions

All enforced in Express middleware on the server. The frontend hides unavailable actions, and the backend refuses them regardless. Someone who strips the frontend and hits the API directly gets exactly the same restrictions.

**APPLICANT:** can self-register, create applications, edit their own DRAFTs, upload documents to their own applications, and submit. That's it. They can see their own applications in any state, including after a final decision.

**REVIEWER:** can claim SUBMITTED applications for review, request additional information from the applicant (assigned only), and complete the review (assigned only). They see all non-DRAFT applications.

**APPROVER:** can approve or reject REVIEWED applications, with one hard constraint: they cannot act on an application they personally reviewed. They see applications in REVIEWED, APPROVED, and REJECTED states.

**ADMIN:** creates and manages staff accounts. Sees everything. Cannot perform any workflow action. An admin's job is operational, not regulatory. Giving ADMIN workflow access would create a side channel that bypasses the review process.

The four-eyes rule from the spec (reviewer cannot be approver) is enforced at two levels: role-level (REVIEWER accounts cannot reach approval endpoints) and data-level (the service checks `reviewer_id != user.id` before any final decision commits).

---

## State Machine

I'd normally put this in Miro with a proper diagram, but here's the text version, which covers all transitions.

```
              DRAFT
                |
             submit (applicant, needs >= 1 doc)
                |
           SUBMITTED
                |
          start-review (any reviewer)
                |
    +------UNDER_REVIEW-------+
    |           |             |
complete-review |        request-info (assigned reviewer, reason required)
    |           |             |
 REVIEWED       |    ADDITIONAL_INFO_REQUIRED
  /    \        |             |
approve reject  |          resubmit (applicant, needs >= 1 doc)
  /       \     |             |
APPROVED REJECTED +-----------+
(terminal) (terminal)
```

The transitions:

**submit** (DRAFT -> SUBMITTED): owner only, at least one `is_current` document required, sets `submitted_at`.

**start-review** (SUBMITTED -> UNDER_REVIEW): any REVIEWER, runs inside `SELECT FOR UPDATE` to handle concurrent claims, sets `reviewer_id`.

**request-info** (UNDER_REVIEW -> ADDITIONAL_INFO_REQUIRED): assigned reviewer only. Reason is required and goes into the audit entry's `metadata` field. I wanted it in the audit trail, not just a status field.

**resubmit** (ADDITIONAL_INFO_REQUIRED -> UNDER_REVIEW): owner only, at least one new document required, `reviewer_id` preserved. I originally thought about sending the application back to the unassigned pool here, but that means a fresh reviewer picks it up with no context and possibly asks for the same things all over again. Keeping the assignment made more sense.

**complete-review** (UNDER_REVIEW -> REVIEWED): assigned reviewer only, `review_notes` required.

**approve** (REVIEWED -> APPROVED): any APPROVER where `reviewer_id != user.id`. This is the hard rule from the spec. Sets `approver_id` and `decided_at`.

**reject** (REVIEWED -> REJECTED): same constraints as approve, plus rejection reason required. Sets `approver_id`, `decided_at`, `decision_reason`.

No API endpoint accepts APPROVED or REJECTED as a source state. There's no reversal path.

---

## Concurrent Access

The spec explicitly requires handling two users acting on the same application simultaneously without producing an inconsistent state. The clearest case: two reviewers see the same SUBMITTED application and both send start-review at the same moment.

I looked at two approaches. Optimistic locking (adding a `version` column and rejecting updates where the version doesn't match) would work, but it puts the retry burden on the caller. A 409 saying "version mismatch, please retry" is a worse API contract than a 400 saying "this application is no longer available for review." The second approach, pessimistic locking with `SELECT FOR UPDATE`, gives the cleaner outcome:

```sql
BEGIN;
SELECT * FROM applications WHERE id = $1 FOR UPDATE;
-- guard check runs here against the locked row
UPDATE applications SET status = 'UNDER_REVIEW', reviewer_id = $2 WHERE id = $1;
COMMIT;
```

`FOR UPDATE` holds an exclusive lock on the row for the duration of the transaction. The second reviewer's query blocks at that line until the first commits. By then the status is `UNDER_REVIEW`, the guard check fails, and they get a 400.

All seven transitions go through `withLock(id, fn)` in the model layer. It's the only path to running a transition, not an opt-in. An integration test fires two simultaneous start-review requests via `Promise.all` and asserts the response codes are exactly `[200, 400]`.

---

## Audit Trail

Every audit entry is written in the same transaction as the action it records. If the action rolls back, the audit entry rolls back too. The two cannot diverge.

Append-only is enforced at the database level:

```sql
REVOKE UPDATE, DELETE ON audit_log FROM bnr_app;
```

One thing that caught me during testing: this REVOKE only works correctly when the table is owned by the admin user and `bnr_app` has only been granted INSERT. If `bnr_app` owns the table, it can always modify its own objects regardless of REVOKE. The two-user setup (admin for schema ownership, `bnr_app` for runtime) is what makes this hold. It's not a convention or a code promise. The database refuses the operation.

Each entry captures: acting user (`actor_id`, `actor_role`), action name, status transition, full JSONB snapshots of the application before and after, and action-specific context in `metadata` (rejection reasons, info request reasons). Timestamp comes from the database server, not the client.

---

## Documents

Uploads go through Multer first. Anything over 5MB is rejected before the route handler runs. The application layer never sees it. Files hit the disk with UUID names; a metadata row goes into `documents` with the original name, path, size, MIME type, uploader, and document type.

Versioning: nothing is deleted. When an applicant resubmits, all existing rows for that application flip to `is_current = false`, new uploads come in as `is_current = true`. The document requirement guards on submit and resubmit check only `is_current = true`. Documents from a previous cycle that were already reviewed don't count toward the requirement.

Download access is scoped by application permission. Applicants can only pull documents from their own applications, staff from applications in their permitted scope.

Not yet implemented: virus scanning, real MIME type verification (the declared type is trusted), encryption at rest.

---

## Testing

Integration tests with Jest and Supertest against `bnr_compliance_test`. Schema applied fresh on startup, every table truncated before each test. No mocking of database calls.

**auth.test.js (19 tests):** registration validation, duplicate email rejection, login, refresh token rotation, logout.

**middleware.test.js (14 unit tests):** Jest mocks, no database. Token verification against missing/bad/expired tokens. Role enforcement returning 403, not 404, not 500. The self-approval block.

**applications.test.js (22 tests):** CRUD visibility and edit rules across roles.

**stateMachine.test.js (29 tests):** every valid transition, every guard failure, terminal state enforcement, and the concurrent claim test: two simultaneous start-review requests via `Promise.all`, result must be exactly `[200, 400]`.

Gaps worth noting: document upload endpoints (tested manually during development), admin user management endpoints, refresh token expiry, audit log read access. I'd close these before a production deployment.

---

## API Documentation

Postman collection at `docs/BNR_Compliance_Portal.postman_collection.json`. Covers authentication, application CRUD, all seven state machine transition endpoints, and document upload.

Set `{{base_url}}` to `http://localhost:3000` and `{{access_token}}` to the value from the login response.

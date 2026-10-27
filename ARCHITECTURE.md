# Clinic Booking API: Architecture & Design (v2, after gap audit)

Take-home for the Full Stack Developer Intern role (HireSol). Brief: clinics publish time slots, patients book them, a slot can never be double-booked even under simultaneous requests, input is validated with helpful errors, and the booking rules have automated tests.

This document is the planning artifact. It does three things in order:

1. **Audits the first architecture** and lists every gap found (Section 1).
2. **Fixes the design** (Sections 2 to 7).
3. **Turns it into a build plan** with quality gates, cut lines and a rubric trace (Sections 8 to 13).

Scoring weights from the brief: Correctness 3, API design 2, Code quality 2, Testing 2. Reviewers check booking rules first, so every decision below is judged by one question: does it make the rules provably correct, and can a reviewer see that in under ten minutes?

---

## 1. Gap audit of the first architecture

Severity reflects the damage if shipped as-is.

| ID | Sev | Gap in v1 | Resolution |
|---|---|---|---|
| G1 | High | `SELECT ... FOR UPDATE` was the locking story, but SQLAlchemy silently ignores it on SQLite, and pysqlite does not open a transaction until the first write. Check-then-insert would race on SQLite. | SQLite connections run `BEGIN IMMEDIATE` via an engine event, with WAL and `busy_timeout`. Postgres uses row locks. CI runs both. (§5.5) |
| G2 | High | "Patient cannot hold overlapping bookings" was listed as a rule, but a unique index cannot enforce it. As check-then-insert it is a race, and the brief never asked for it. | Moved to Tier 2. If built, it takes a patient row lock under a global lock order. (§5.4) |
| G3 | High | Idempotency was a one-liner. Key scope, same key with a different body, two concurrent requests with the same key, and the replay status code were all undefined. | Unique `(patient_id, key)`, lookup performed inside the slot lock, 422 on mismatch, 200 plus `Idempotent-Replay` on replay. (§5.4) |
| G4 | High | Identity by `X-Patient-Id` header is spoofable, and accepting `patient_id` in the booking body is an insecure-direct-object-reference bug (book as anyone). | Opaque Bearer API keys, stored hashed. Identity always comes from the key. The booking body contains only `slot_id`. (§6.3) |
| G5 | Medium | Inconsistent verbs: `DELETE /slots/{id}` but `POST /bookings/{id}/cancel`. Both are soft cancels that keep history. | Both are `POST .../cancel`. (§6.1) |
| G6 | High | Overlap prevention had no race-proof mechanism on SQLite, and the Postgres exclusion constraint needs `btree_gist`. It also silently assumed one provider per clinic. | Exclusion constraint on Postgres, `IMMEDIATE` serialization on SQLite, extension created in the migration, and the single-calendar assumption written down (A1). |
| G7 | Medium | A `timezone` column on clinics that nothing used. | Removed. The API is UTC-only and says so. |
| G8 | High | SQLite drops timezone information. Round-tripping an aware datetime returns a naive one, which breaks every time comparison. | `UTCDateTime` type decorator that enforces aware-in, aware-out, plus a round-trip test. |
| G9 | Medium | No lock ordering, so deadlocks were possible once more than one lock was taken. | Global order: patient, then slot, then booking. The slot row is the per-slot mutex. Read, lock, re-read pattern. (§5.3) |
| G10 | Medium | Behavior undefined for cancelling after a slot has started, and the order in which checks run (404 vs 403 vs 409 vs idempotent 200). | Explicit rules and a fixed check precedence. (§5.4, §6.5) |
| G11 | Medium | The concurrency test would not have been truly concurrent: a sync test client with a shared session proves nothing. | Two levels: service-level threads with their own sessions behind a barrier, and a live-server HTTP race. No `sleep` anywhere. (§9) |
| G12 | Medium | Per-test transaction rollback cannot be used with tests that commit across several connections. | Truncate-based isolation, schema built from migrations only. (§9.3) |
| G13 | Medium | Alembic autogenerate does not reliably emit partial indexes or exclusion constraints, and `create_all` would bypass them entirely. | Hand-written migration, `create_all` banned, and a test that asserts the constraints exist. |
| G14 | Medium | Scope against the time box: v1 listed many extras with no cut line. | Tiers 0/1/2 with cut lines at fixed hours. (§2, §11) |
| G15 | Medium | Availability filter, listing indexes and pagination were unspecified. | Derived status via `NOT EXISTS`, indexes defined, keyset cursor pagination. (§4, §6.4) |
| G16 | Medium | No clinic-facing view of bookings, and no rule on who may see patient data. | `GET /clinics/{id}/bookings`. Public slot views never reveal who booked. |
| G17 | Medium | Error handling covered business rules only. Nothing for database outage, unexpected exceptions, deadlock, or traceability. | Request IDs, a 503 mapping, a safe 500 body, bounded retry on serialization failure. (§7) |
| G18 | Medium | The reviewer's path to running the project was assumed to work. A Docker-only test setup fails reviewers without Docker. | SQLite is the zero-setup default for run and test. Docker Compose gives Postgres. A fresh-clone check is a gate. |
| G19 | Medium | The correctness evidence existed only inside the test suite, which reviewers skim. | `make demo` fires 20 concurrent bookings and prints "1 x 201, 19 x 409". |
| G20 | Low | "Slot must be in the future" is untestable without a controllable clock. | Injectable `Clock`. |
| G21 | Low | Unknown JSON fields silently ignored, so typos produce confusing success. | `extra="forbid"` plus field-path error mapping. |
| G22 | Low | OpenAPI would show only success responses. | Declare 401/403/404/409/422 response models and examples. |
| G23 | Low | Operational basics unspecified: health/readiness, env-based config, non-root container, no secrets in the repo. | Section 7. |
| G24 | Medium | What "booked" means to a public viewer of a slot was undefined. | Derived `status` enum: `available`, `booked`, `cancelled`. |
| G25 | Low | Reschedule, multiple providers, capacity above 1 were silently out of scope. | Listed explicitly as non-goals with extension notes. (§2.3) |
| G26 | Medium | Cancelling a booked slot cascades, but the response would not tell the caller what else changed. | Slot-cancel response includes `cancelled_booking`. |

Two of these (G1, G8) are the kind of bug that only appears when a reviewer runs the tests on a machine other than yours. That is why the zero-setup SQLite path and the Postgres path are both first-class and both in CI.

---

## 2. Scope

### 2.1 Tiers and cut lines

| Tier | Contents | Rule |
|---|---|---|
| **Tier 0: required by the brief** | Create/list/cancel slots. Book/cancel bookings. No double-booking under concurrency. Validation with helpful errors. Tests for the rules. README with setup, endpoints, examples, test instructions. | Must be complete and green by hour 3:45. |
| **Tier 1: differentiators, low risk** | API-key auth with ownership checks. Idempotency-Key. Slot overlap prevention. Keyset pagination and filters. Request IDs and structured logs. Live-server race test. `make demo`. Docker Compose, CI matrix, DESIGN notes. | Only after Tier 0 is green. |
| **Tier 2: stretch** | Patient-overlap rule. Atomic batch slot creation. Cancellation cutoff window. | Only if 30+ minutes remain at hour 5. Otherwise listed under "next steps". |

If time runs short, cut in this order: Tier 2, then batch/filters polish, then API-key auth (fall back to `X-Clinic-Id` / `X-Patient-Id` headers, documented as a placeholder). Never cut tests or the concurrency mechanism.

### 2.2 Domain assumptions (state them in the README)

- **A1. One clinic is one calendar.** One slot is one appointment with one provider, and a clinic's open slots cannot overlap. Extension: add `provider_id` and scope the exclusion constraint to it.
- **A2. Capacity is exactly 1.** A slot holds one confirmed booking at a time.
- **A3. Cancelling keeps history.** Slots and bookings are never deleted. A cancelled booking frees the slot for rebooking.
- **A4. A clinic cancelling a booked slot cancels the booking** in the same transaction (`cancelled_by = clinic`).
- **A5. A started slot can no longer change state; repeating a completed cancel stays 200.** Once `now >= starts_at`, bookings and slot state transitions are rejected with `SLOT_STARTED`; a cancel of an already-cancelled resource still returns 200 (§14.3).
- **A6. All timestamps are UTC.** Input must carry an offset and is normalized to UTC. Naive datetimes are rejected.

### 2.3 Deliberate non-goals

Frontend, queues, caches, Redis, microservices, email/SMS notifications, rate limiting, payments, atomic reschedule (two-slot transaction, a hard concurrency problem of its own), multi-provider clinics, recurring-slot rules. Each is a sensible "next step" in the README. Leaving them out is a judgment call that should be visible.

---

## 3. System overview

```
                    HTTP (JSON)
   Client ───────────────────────────────►  FastAPI app (sync)
                                             │
        ┌────────────────────────────────────┴───────────────────────────────┐
        │ Middleware: request-id, access log, exception → error envelope      │
        ├─────────────────────────────────────────────────────────────────────┤
        │ API layer      routers + Pydantic schemas + auth dependency         │
        │                (no business rules, no SQL)                          │
        ├─────────────────────────────────────────────────────────────────────┤
        │ Service layer  SlotService, BookingService, AccountService          │
        │                orchestrates: lock → rules → persist → commit        │
        ├───────────────────────────────┬─────────────────────────────────────┤
        │ Domain (pure)                 │ Repositories (SQLAlchemy 2.0)       │
        │ rules.py, errors.py, clock    │ locking queries, keyset listing     │
        ├───────────────────────────────┴─────────────────────────────────────┤
        │ Unit of Work (one transaction per use case)                         │
        └─────────────────────────────┬───────────────────────────────────────┘
                                      │
                       PostgreSQL 16 (primary)  /  SQLite (zero-setup)
                       constraints are the last line of defense
```

Rules of the road:

- Routers call services. Services call repositories and domain rules. Nothing calls upward.
- `domain/` has no imports from SQLAlchemy or FastAPI, so its rules are unit-testable in microseconds.
- One use case equals one transaction, managed by the Unit of Work. Services never commit mid-way.

**Sync vs async decision.** The service is synchronous FastAPI with the sync SQLAlchemy engine. Correctness here depends on transaction and lock semantics, which are easiest to reason about and test with plain threads. FastAPI runs sync endpoints in a thread pool, so concurrent requests are genuinely concurrent at the database. Async would add complexity without improving a single guarantee.

---

## 4. Data model

### 4.1 Schema (Postgres form; SQLite equivalents noted)

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;            -- Postgres only

CREATE TABLE clinics (
  id            uuid PRIMARY KEY,
  name          text        NOT NULL CHECK (char_length(name) BETWEEN 1 AND 120),
  api_key_hash  char(64)    NOT NULL UNIQUE,          -- sha256 of the key
  created_at    timestamptz NOT NULL
);

CREATE TABLE patients (
  id            uuid PRIMARY KEY,
  name          text        NOT NULL CHECK (char_length(name) BETWEEN 1 AND 120),
  email         text        NOT NULL,                 -- stored lowercased
  api_key_hash  char(64)    NOT NULL UNIQUE,
  created_at    timestamptz NOT NULL,
  CONSTRAINT patients_email_unique UNIQUE (email)
);

CREATE TABLE slots (
  id            uuid PRIMARY KEY,
  clinic_id     uuid        NOT NULL REFERENCES clinics(id),
  starts_at     timestamptz NOT NULL,
  ends_at       timestamptz NOT NULL,
  status        text        NOT NULL DEFAULT 'open'
                  CHECK (status IN ('open','cancelled')),
  created_at    timestamptz NOT NULL,
  cancelled_at  timestamptz,
  CHECK (ends_at > starts_at),
  CHECK ((status = 'cancelled') = (cancelled_at IS NOT NULL)),
  -- no overlapping open slots in one clinic; adjacent slots allowed ('[)')
  EXCLUDE USING gist (clinic_id WITH =,
                      tstzrange(starts_at, ends_at, '[)') WITH &&)
          WHERE (status = 'open')
);
CREATE INDEX slots_clinic_time ON slots (clinic_id, starts_at, id);

CREATE TABLE bookings (
  id               uuid PRIMARY KEY,
  slot_id          uuid        NOT NULL REFERENCES slots(id),
  patient_id       uuid        NOT NULL REFERENCES patients(id),
  status           text        NOT NULL DEFAULT 'confirmed'
                     CHECK (status IN ('confirmed','cancelled')),
  cancelled_by     text        CHECK (cancelled_by IN ('patient','clinic')),
  idempotency_key  text,
  created_at       timestamptz NOT NULL,
  cancelled_at     timestamptz,
  CHECK ((status = 'cancelled') = (cancelled_at IS NOT NULL)),
  CHECK ((status = 'cancelled') = (cancelled_by IS NOT NULL))
);

-- THE guarantee: at most one confirmed booking per slot, ever.
CREATE UNIQUE INDEX one_confirmed_booking_per_slot
  ON bookings (slot_id) WHERE status = 'confirmed';

CREATE UNIQUE INDEX booking_idempotency
  ON bookings (patient_id, idempotency_key) WHERE idempotency_key IS NOT NULL;

CREATE INDEX bookings_patient_time ON bookings (patient_id, created_at DESC, id);
CREATE INDEX bookings_slot         ON bookings (slot_id);
```

Notes:

- A partial unique index (not a plain unique constraint) is what makes cancel-then-rebook work while still forbidding two live bookings. SQLite supports partial unique indexes, so the guarantee holds on both databases.
- There is no `booked` flag on slots. "Booked" is derived (`EXISTS` confirmed booking), so it can never disagree with the bookings table.
- The exclusion constraint exists only on Postgres. On SQLite the same rule is checked in the service inside an `IMMEDIATE` transaction (§5.5).
- UUID v4 primary keys. Timestamps are `timestamptz`; on SQLite a `UTCDateTime` type decorator guarantees aware-in/aware-out (G8).

### 4.2 State machines

```
Slot:     open ──(clinic cancels)──► cancelled        (terminal)
Booking:  confirmed ──(patient or clinic cancels)──► cancelled   (terminal)
          slot cancelled ⇒ its confirmed booking → cancelled (cancelled_by = clinic)
```

Invariant to protect and test: **a cancelled slot never has a confirmed booking, and an open slot has at most one.**

### 4.3 Derived slot status shown to clients

`cancelled` if `status='cancelled'`, else `booked` if a confirmed booking exists, else `available`. Query shape:

```sql
SELECT s.*, CASE WHEN s.status='cancelled' THEN 'cancelled'
                 WHEN EXISTS (SELECT 1 FROM bookings b
                              WHERE b.slot_id = s.id AND b.status='confirmed')
                 THEN 'booked' ELSE 'available' END AS derived_status
FROM slots s WHERE s.clinic_id = :cid AND (s.starts_at, s.id) > (:t, :id)
ORDER BY s.starts_at, s.id LIMIT :n;
```

The `EXISTS` is served by the partial unique index, so listing stays an index scan.

---

## 5. Concurrency design (the heart of the submission)

### 5.1 Principle

Three independent layers, so a bug in one cannot cause a double-booking:

1. **Serialization:** every state change on a slot happens while holding that slot's row lock. The slot row is the per-slot mutex (on SQLite, serialization is the global writer lock taken by `BEGIN IMMEDIATE`; §5.5).
2. **Rule checks inside the lock:** the service verifies status, time and existing booking while holding it, which gives clean errors and correct decisions.
3. **Database backstop:** the database enforces exactly two booking-related invariants: **at most one confirmed booking per slot** (partial unique index) and **no overlapping open slots per clinic** (Postgres exclusion constraint). Any write that would violate either is rejected even if the code is wrong. A test proves the first by bypassing the service.

The invariant "a cancelled slot never has a confirmed booking" is **not** enforced by a database constraint. It is enforced by the service inside the slot lock (the cascade) and proven by the R2 test. See §14.1.

### 5.2 Race matrix

Every pair of concurrent operations that could break an invariant, the mechanism that resolves it, and the test that proves it.

| # | Concurrent operations | Mechanism | Required outcome | Test |
|---|---|---|---|---|
| R1 | book × book (same slot) | slot row lock; second sees confirmed booking; unique index backstop | exactly one 201, the rest 409 `SLOT_ALREADY_BOOKED` | `test_n_parallel_bookings_one_winner` (service and HTTP) |
| R2 | book × cancel slot | both lock the slot, one goes first | either (booked, then cancelled with cascade) or (cancelled, then book gets 409 `SLOT_CANCELLED`); never a confirmed booking on a cancelled slot | `test_book_vs_cancel_slot_never_leaves_confirmed_on_cancelled` |
| R3 | rebook × cancel booking | both lock the slot | either order is a valid linearization; invariant holds | `test_rebook_races_with_cancel` |
| R4 | cancel booking × cancel booking | slot lock, second finds it already cancelled | both 200, one real transition, original `cancelled_by` preserved | `test_double_cancel_is_idempotent` |
| R5 | cancel slot × cancel slot | same | both 200, one cascade | same style |
| R6 | create slot × create overlapping slot | Postgres exclusion constraint; SQLite `IMMEDIATE` plus in-service overlap check | one 201, one 409 `SLOT_OVERLAP` | `test_parallel_overlapping_slot_creation` |
| R7 | same Idempotency-Key × same Idempotency-Key | slot lock then key lookup inside the lock; unique `(patient_id, key)` backstop | one booking created; the other request replays it | `test_concurrent_same_key_creates_one_booking` |
| R8 (T2) | one patient books two overlapping slots at once | patient row lock taken first | only one succeeds | `test_patient_overlap_race` |

### 5.3 Lock protocol

- **Global lock order: patient → slot → booking.** Only the Tier-2 patient-overlap path takes the patient lock, and it takes it first. Everything else starts at the slot.
- **Read, lock, re-read.** Endpoints that start from a booking id (cancel) read the booking unlocked to learn `slot_id`, lock the slot, then re-read the booking under the lock and decide. This avoids deciding on stale data.
- Locks are held only for the duration of a short transaction. No network calls or sleeps happen inside.

### 5.4 Use-case pseudocode

**Book a slot**

```
book(patient, slot_id, idem_key):
  BEGIN
    slot = slots.get_for_update(slot_id)            or raise SlotNotFound (404)
    if idem_key:
        prev = bookings.find(patient.id, idem_key)
        if prev: 
            if prev.slot_id != slot_id: raise IdempotencyKeyMismatch (422)
            return Replay(prev)                      # 200 + Idempotent-Replay: true
    rules.assert_bookable(slot, now)                 # SLOT_CANCELLED, SLOT_STARTED
    if bookings.active_for_slot(slot.id): raise SlotAlreadyBooked (409)
    booking = bookings.add(slot, patient, idem_key)
  COMMIT                                             # IntegrityError on the partial index
                                                     # is translated to SlotAlreadyBooked
```

**Cancel a booking** (patient owner or the slot's clinic): read booking, lock slot, re-read booking, authorize, return 200 unchanged if already cancelled, else apply rule A5 (`SLOT_STARTED`), then set `cancelled`, `cancelled_by`, `cancelled_at`.

**Cancel a slot** (owning clinic): lock slot, authorize, no-op if already cancelled, reject if started, set `cancelled`, then cancel its confirmed booking if any (`cancelled_by = clinic`). Return `{slot, cancelled_booking}`.

**Create a slot** (owning clinic): validate the window with pure rules, check for overlap with open slots, insert. A Postgres exclusion violation (`23P01`) is translated to `SLOT_OVERLAP`.

### 5.5 Per-database notes

**PostgreSQL**
- Default `READ COMMITTED` plus explicit row locks. SERIALIZABLE was rejected: it needs app-wide retry logic and is harder to reason about than "lock the slot".
- Defensive wrapper: bounded retry (max 3) on SQLSTATE `40001`/`40P01`. Under the lock order above this should never trigger, but it is cheap insurance.
- Pool sized above the race test's thread count so the test measures locking, not pool starvation.

**SQLite**
- `with_for_update()` is ignored by SQLAlchemy on SQLite. Instead an engine `begin` listener emits `BEGIN IMMEDIATE` for all service transactions, which takes the write lock up front and serializes writers. Check-then-insert is therefore safe.
- Pragmas on connect: `journal_mode=WAL`, `busy_timeout=5000`, `foreign_keys=ON`.
- Use a file database for tests (in-memory databases are per-connection and break multi-threaded tests).
- This asymmetry is documented honestly in the README: Postgres has finer-grained locks, SQLite serializes all writers. The guarantee is identical.

---

## 6. API specification

Base path `/v1`. JSON only. All times ISO-8601 with offset, returned in UTC (`Z`).

### 6.1 Endpoints

| Method & path | Auth | Success | Notes |
|---|---|---|---|
| `POST /v1/clinics` | none | 201 `{clinic, api_key}` | Key shown once. Open registration is a demo simplification. |
| `POST /v1/patients` | none | 201 `{patient, api_key}` | 409 `EMAIL_ALREADY_REGISTERED` on duplicate email. |
| `POST /v1/clinics/{clinic_id}/slots` | clinic (own) | 201 slot, `Location` header | Validation and overlap rules. |
| `GET /v1/clinics/{clinic_id}/slots` | none | 200 `{items, next_cursor}` | Filters: `status`, `from`, `to`, `limit`, `cursor`. |
| `GET /v1/slots/{slot_id}` | none | 200 slot | Never reveals who booked. |
| `POST /v1/slots/{slot_id}/cancel` | clinic (owner) | 200 `{slot, cancelled_booking}` | Idempotent. |
| `POST /v1/bookings` | patient | 201 booking, `Location` | Body `{slot_id}`. Optional `Idempotency-Key`. Replay returns 200. |
| `GET /v1/bookings/{booking_id}` | patient owner or slot's clinic | 200 booking | |
| `POST /v1/bookings/{booking_id}/cancel` | patient owner or slot's clinic | 200 booking | Idempotent. |
| `GET /v1/patients/{patient_id}/bookings` | patient (self) | 200 paged | |
| `GET /v1/clinics/{clinic_id}/bookings` | clinic (self) | 200 paged | Includes patient name and email. |
| `GET /health` | none | 200 | Liveness. |
| `GET /ready` | none | 200 / 503 | Executes `SELECT 1`. |

### 6.2 Resource shapes

```json
// Slot
{"id":"…","clinic_id":"…","starts_at":"2026-11-03T09:00:00Z","ends_at":"2026-11-03T09:30:00Z",
 "status":"available","created_at":"…","cancelled_at":null}

// Booking
{"id":"…","slot_id":"…","patient_id":"…","status":"confirmed","cancelled_by":null,
 "created_at":"…","cancelled_at":null,
 "slot":{"clinic_id":"…","starts_at":"…","ends_at":"…"}}
```

### 6.3 Authentication and authorization

- `Authorization: Bearer clk_<random>` for clinics and `pat_<random>` for patients. Keys are 32 random bytes (urlsafe), stored as SHA-256 hashes (high-entropy tokens do not need a slow hash). Lookup is by indexed hash.
- A single FastAPI dependency resolves the key to an `Actor(kind, id)`.
- Authorization is an explicit table, enforced in services (not scattered in routers):

| Action | Allowed actor |
|---|---|
| Publish a slot | the clinic whose id is in the path |
| Cancel a slot | the clinic that owns the slot |
| Book | any authenticated patient, acting as themselves |
| Read or cancel a booking | the booking's patient, or the clinic that owns its slot |
| List a patient's bookings | that patient |
| List a clinic's bookings | that clinic |

- Status semantics: 401 for missing or invalid key, 403 for authenticated but not permitted (used uniformly, for easy explanation and testing).
- The key prefix selects the lookup table (`clk_` → clinics, `pat_` → patients). An unknown key returns 401; a valid key of the wrong kind returns 403.

### 6.4 Validation rules

| Field | Rules |
|---|---|
| `starts_at`, `ends_at` | ISO-8601 **with offset** (naive rejected), normalized to UTC, `ends_at > starts_at` |
| Duration | 5 minutes to 8 hours (configurable) |
| Start time | strictly in the future, no more than 365 days ahead (configurable) |
| Overlap | no overlap with another open slot in the same clinic; touching ends allowed |
| `name` | trimmed, 1 to 120 characters |
| `email` | valid, at most 254 characters, lowercased |
| `slot_id`, path ids | valid UUID |
| `Idempotency-Key` | 1 to 64 characters of `[A-Za-z0-9_-]` |
| Listing | `limit` 1 to 100 (default 20), `from < to`, `status` in `available|booked|cancelled`, default is all statuses |
| Unknown JSON fields | rejected (`extra="forbid"`) |
| Cursor | opaque base64url of `(starts_at, id)`. Invalid gives 422 `INVALID_CURSOR`. Keyset pagination. |

### 6.5 Check precedence (so errors are predictable)

Book: authenticate (401) → body validation (422) → slot exists (404) → **take lock** → idempotent replay → slot cancelled → slot started → already booked → insert.

Cancel: authenticate (401) → resource exists (404) → authorize (403) → already cancelled? return 200 → started? (409) → apply.

### 6.6 Error envelope and catalog

One shape for everything, including framework-generated 422s and unexpected errors:

```json
{"error":{"code":"SLOT_ALREADY_BOOKED",
          "message":"This slot has already been booked.",
          "request_id":"01J…",
          "details":{"slot_id":"…"}}}
```

Validation errors carry field paths:

```json
{"error":{"code":"VALIDATION_ERROR","message":"Request validation failed.","request_id":"…",
          "details":{"fields":[{"field":"ends_at","issue":"must be after starts_at"}]}}}
```

| HTTP | Code | When |
|---|---|---|
| 401 | `UNAUTHENTICATED` | missing, malformed or unknown API key |
| 403 | `FORBIDDEN` | acting on another party's resource |
| 404 | `CLINIC_NOT_FOUND`, `SLOT_NOT_FOUND`, `BOOKING_NOT_FOUND`, `PATIENT_NOT_FOUND` | unknown id |
| 409 | `SLOT_ALREADY_BOOKED` | confirmed booking exists |
| 409 | `SLOT_CANCELLED` | booking a cancelled slot |
| 409 | `SLOT_STARTED` | booking or cancelling once the slot has started |
| 409 | `SLOT_OVERLAP` | clinic already has an open slot in that window |
| 409 | `EMAIL_ALREADY_REGISTERED` | duplicate patient email |
| 422 | `VALIDATION_ERROR`, `INVALID_CURSOR`, `IDEMPOTENCY_KEY_MISMATCH` | bad input |
| 500 | `INTERNAL_ERROR` | unexpected. Body has only `request_id`, no stack trace. |
| 503 | `SERVICE_UNAVAILABLE` | database unreachable |

Codes are stable strings that clients can branch on. Messages are written for humans and say what to do next (for example, "choose a different slot").

---

## 7. Code architecture and engineering standards

### 7.1 Repository layout

```
clinic-booking-api/
├── app/
│   ├── main.py                  # app factory, routers, exception handlers
│   ├── api/
│   │   ├── deps.py              # auth dependency, UoW dependency, clock
│   │   ├── routes_accounts.py  routes_slots.py  routes_bookings.py  routes_health.py
│   │   └── schemas.py           # request/response models, error models
│   ├── services/
│   │   ├── slots.py  bookings.py  accounts.py
│   │   └── authz.py             # the authorization table as code
│   ├── domain/
│   │   ├── rules.py             # pure functions: window validation, bookable?, cancellable?
│   │   ├── errors.py            # DomainError hierarchy with code + http mapping
│   │   └── clock.py             # Clock protocol, SystemClock
│   ├── repositories/            # slots.py bookings.py accounts.py (locking + keyset queries)
│   ├── db/
│   │   ├── models.py  session.py  uow.py
│   │   └── types.py             # UTCDateTime
│   ├── core/                    # config (pydantic-settings), logging, request-id
│   └── security.py              # key generation + hashing
├── migrations/                  # Alembic; hand-written initial migration
├── tests/
│   ├── unit/                    # domain rules, security, cursor
│   ├── service/                 # rules through real DB, incl. threaded races
│   ├── api/                     # status codes, envelope, auth, pagination
│   ├── http_race/               # live-server concurrency test
│   ├── schema/                  # constraints exist and bite
│   └── conftest.py
├── scripts/demo_race.py         # 20 concurrent bookings, prints the verdict
├── docs/DESIGN.md               # decisions and race matrix (condensed from this file)
├── docker-compose.yml  Dockerfile  Makefile  pyproject.toml  .env.example
├── .github/workflows/ci.yml
└── README.md
```

### 7.2 Principles

- **Domain errors, one translator.** Services raise `DomainError` subclasses carrying `code`, `http_status` and `details`. A single exception handler maps them to the envelope. No `HTTPException` below the API layer.
- **Pure rules.** `rules.assert_bookable(slot, now)` takes plain values and returns or raises. Everything time-dependent receives `now` from the injected `Clock`.
- **Unit of Work.** `with uow.begin() as tx:` yields repositories on one session. It commits on exit, rolls back on exception, and applies the dialect-specific begin (`IMMEDIATE` on SQLite).
- **Typed throughout.** `mypy --strict` on `app/`, `ruff` for lint and format. Pydantic models for every input and output, with examples for OpenAPI.
- **Small functions, no clever code.** A service method fits on a screen and reads like the pseudocode in 5.4.
- **No dead flexibility.** No generic repository base class, no plugin system, no settings nobody uses.

### 7.3 Configuration and operations

- Settings via environment (`pydantic-settings`): `DATABASE_URL` (default `sqlite:///./clinic.db`), `MIN_SLOT_MINUTES`, `MAX_SLOT_MINUTES`, `MAX_BOOKING_HORIZON_DAYS`, `LOG_LEVEL`. `.env.example` committed, no secrets in the repo.
- Structured JSON logs with `request_id` on every line. Business events (booking created, cancelled) logged at INFO with ids only, never email or names.
- Middleware assigns `X-Request-ID` and echoes it in responses and error bodies.
- Handlers: `DomainError` → mapped status; `RequestValidationError` → 422 envelope; `OperationalError` (connection) → 503; anything else → 500 with `request_id` and a logged stack trace.
- Docker: slim Python 3.12 image, non-root user, healthcheck against `/ready`. Compose waits for Postgres health, runs `alembic upgrade head`, then starts Uvicorn.
- Schema changes only through Alembic. `create_all` is never used (G13).

---

## 8. Testing strategy

### 8.1 Pyramid

| Layer | Purpose | Speed |
|---|---|---|
| Unit (pure rules, cursor codec, key hashing) | every validation and state rule, no DB | milliseconds |
| Service (real DB) | rules with locks and constraints, including threaded races | fast |
| API | status codes, envelope shape, auth matrix, pagination, OpenAPI sanity | fast |
| HTTP race (live server) | end-to-end proof that concurrent HTTP requests cannot double-book | about 1–2 s |
| Schema | the constraints exist in migrated databases and reject bad writes directly | fast |

### 8.2 The named tests (each maps to a rule)

**Booking rules**
- `test_book_available_slot_returns_201`
- `test_book_already_booked_slot_returns_409_slot_already_booked`
- `test_book_cancelled_slot_returns_409`
- `test_book_started_slot_returns_409` (uses `FrozenClock`)
- `test_cancel_booking_frees_slot_and_rebooking_succeeds`
- `test_cancel_booking_is_idempotent_and_keeps_first_cancelled_by`
- `test_cancel_after_slot_started_returns_409`
- `test_cancel_slot_cascades_to_booking_and_reports_it`
- `test_patient_cannot_cancel_someone_elses_booking_403`

**Slots and validation**
- `test_create_slot_rejects_naive_datetime`, `..._end_before_start`, `..._in_past`, `..._too_short`, `..._too_long`
- `test_overlapping_slot_rejected_adjacent_slot_allowed`
- `test_unknown_field_rejected`
- `test_validation_error_lists_field_paths`
- `test_list_slots_filter_and_cursor_pagination_is_stable`

**Concurrency**
- `test_n_parallel_bookings_one_winner` (service level: 20 threads, own sessions, `Barrier`)
- `test_http_race_one_201_nineteen_409` (live server, 20 threads)
- `test_book_vs_cancel_slot_never_leaves_confirmed_on_cancelled`
- `test_rebook_races_with_cancel`
- `test_parallel_overlapping_slot_creation`
- `test_concurrent_same_idempotency_key_creates_one_booking`

**Idempotency**
- `test_same_key_same_slot_replays_original_with_header`
- `test_same_key_different_slot_returns_422`

**Schema and platform**
- `test_unique_index_rejects_second_confirmed_booking_inserted_directly` (bypasses the service, proving the DB backstop)
- `test_migrated_schema_has_expected_constraints`
- `test_utc_roundtrip_preserves_timezone_on_sqlite` (G8)

About 30 tests, each named after the rule it protects. The suite reads as documentation of the booking rules.

### 8.3 Test infrastructure

- Schema comes from `alembic upgrade head` on a temp SQLite file, or on Postgres when `TEST_DATABASE_URL` is set. The test and production schemas are the same artifact.
- Isolation is by `TRUNCATE`/`DELETE` between tests, not rollback, because race tests commit across connections.
- Race tests use `threading.Barrier` to release all workers together, and assert on outcome counts and on database state afterwards (exactly one confirmed row). No `time.sleep` for synchronization.
- `FrozenClock` is injected through a FastAPI dependency override. Factory helpers build clinics, patients and slots in one line.
- Postgres-only tests carry `@pytest.mark.postgres`, which is skipped with a clear message on SQLite.
- Quality gate: the booking services and domain rules are covered at a high branch-coverage level (report only, no chasing a global percentage).

---

## 9. Developer experience, CI and documentation

### 9.1 One-command workflows

| Command | Result |
|---|---|
| `make install` | venv and dev dependencies |
| `make run` | migrate and start on SQLite at `:8000` |
| `make test` | full suite on SQLite, no Docker needed |
| `make test-pg` | full suite against Postgres via Compose |
| `make lint` | ruff and mypy |
| `make demo` | starts the app, fires 20 concurrent bookings, prints the outcome |
| `docker compose up` | API and Postgres, ready in one step |

### 9.2 CI (GitHub Actions)

Jobs: `lint` (ruff, mypy), `test-sqlite`, `test-postgres` (service container `postgres:16`), `docker-build` smoke. A passing-badge in the README tells the reviewer the repo works on a clean machine.

### 9.3 `make demo` output (target)

```
Registered clinic and 20 patients
Published slot 2026-11-03T09:00Z–09:30Z
Firing 20 simultaneous booking requests...
  201 Created ........ 1
  409 SLOT_ALREADY_BOOKED ... 19
Database check: confirmed bookings for slot = 1  ✓
```

This turns the correctness claim into something a reviewer can see in seconds.

### 9.4 README outline

1. One-paragraph summary, CI badge, and a link to the architecture notes
2. Quick start: `make run` (no Docker) and `docker compose up` (Postgres)
3. Run the tests: `make test`, `make test-pg`
4. **How double-booking is prevented**: the three layers and the race matrix in about 20 lines
5. Endpoint table, then end-to-end `curl` walk-through: register, publish, book, double-book (409), cancel, rebook
6. Error format and code catalog
7. Business rules and domain assumptions (A1–A6)
8. Design decisions and trade-offs (sync vs async, SQLite vs Postgres semantics, API keys)
9. Next steps: patient-overlap rule, batch publishing, cancellation window, reschedule, multi-provider, rate limiting

`docs/DESIGN.md` carries the decision log (Section 12) and the race matrix for interviewers who want depth.

---

## 10. Rubric traceability

| Criterion (weight) | What a reviewer looks for | Evidence in the repo | Where to find it fast |
|---|---|---|---|
| Correctness (3) | No double-booking, rules hold under concurrency | Slot lock, partial unique index, race matrix R1–R7, `make demo` | README §4, `tests/service`, `tests/http_race` |
| API design (2) | Sensible resources, correct status codes, helpful errors | Consistent verbs, 201/200/401/403/404/409/422 used precisely, single error envelope, stable codes, OpenAPI with error models | README endpoint table, `/docs` |
| Code quality (2) | Structure, separation of concerns | Layered package, pure domain rules, Unit of Work, strict typing, lint in CI | `app/` layout, CI badge |
| Testing (2) | Meaningful tests of important rules | About 30 tests named after rules, threaded and HTTP races, constraint tests that bypass the service | `tests/`, README test section |

A fifth thing is not scored but shapes the interview: the decision log shows trade-offs made on purpose.

---

## 11. Build plan (time-boxed, about 6 hours)

| Time | Work | Exit criterion |
|---|---|---|
| 0:00–0:30 | Scaffold: `pyproject`, ruff/mypy, config, DB session with SQLite pragmas, `UTCDateTime`, Alembic init, Makefile | `make lint` passes on an empty app |
| 0:30–1:30 | Hand-written migration (both dialects), models, repositories, domain rules and errors, unit tests for rules | schema tests green on SQLite |
| 1:30–2:45 | Slot and booking services with the lock protocol, Unit of Work, service-level race test | R1, R2, R3 tests green |
| 2:45–3:45 | Routers, schemas, error envelope, auth dependency, listing with cursor | API tests green. **Cut line A: Tier 0 complete** |
| 3:45–4:45 | Tier 1: idempotency, request IDs and logs, `/ready`, live-server HTTP race, `scripts/demo_race.py` | R6, R7 and HTTP race green |
| 4:45–5:30 | Dockerfile, Compose, CI matrix, README, `docs/DESIGN.md` | CI green on both databases |
| 5:30–6:00 | Fresh-clone verification, final polish, tag | Definition of done (Section 13) |
| Spare time only | Tier 2 items | Never at the cost of the above |

If behind at 3:45: drop API keys for header identity, skip batch and filters polish. If behind at 4:45: cut `make demo` and keep the HTTP race test.

---

## 12. Decision log (condensed)

| # | Decision | Why | Rejected alternative |
|---|---|---|---|
| D1 | Sync FastAPI and sync SQLAlchemy | Guarantees depend on transaction semantics, which threads test most directly | Async: complexity with no gain in correctness |
| D2 | Row lock **and** DB constraint | Lock gives clean decisions, constraint is the unbreakable backstop | App check only (racy); constraint only (poor errors, no cascade safety) |
| D3 | READ COMMITTED plus locks | Simple reasoning | SERIALIZABLE (retry logic everywhere) |
| D4 | Soft cancel with history | Audit, safe rebooking | Hard delete |
| D5 | `POST .../cancel` for both resources | Consistent, supports idempotent 200 with a body | `DELETE` for one, `POST` for the other |
| D6 | Derived slot status | Cannot disagree with bookings | Stored `booked` flag |
| D7 | Bearer API keys, identity from the key | Removes the spoofable-header and body-`patient_id` bugs | Header ids |
| D8 | UTC-only API | One time model, no hidden conversions | Per-clinic timezone nobody uses |
| D9 | Single calendar per clinic (A1) | Matches the brief | Provider model (scope creep) |
| D10 | Cursor (keyset) pagination | Stable under inserts, index-friendly | Offset pagination |
| D11 | SQLite default plus Postgres via Compose | Zero-setup for reviewers, production-grade option | Postgres only (breaks without Docker) |
| D12 | Migrations are the schema source | Constraints cannot be skipped | `create_all` |
| D13 | Cascade-cancel booking when a clinic cancels a slot | Matches reality, keeps the invariant | Refuse to cancel booked slots (strands clinics) |

---

## 13. Risks and definition of done

### 13.1 Build risks and mitigations

| Risk | Mitigation |
|---|---|
| Alembic autogenerate drops partial indexes or exclusion constraints | Hand-write the migration, assert constraints in a schema test |
| `btree_gist` unavailable on a managed Postgres | **Hard requirement** on Postgres: the migration fails fast if `CREATE EXTENSION btree_gist` cannot succeed. The official image includes it; README notes the requirement. |
| Flaky race tests | Barrier release, outcome-count assertions, no sleeps, pool larger than thread count |
| SQLite "database is locked" under threads | WAL, `busy_timeout`, `IMMEDIATE`, file DB for tests |
| Naive datetimes sneaking in through SQLite | `UTCDateTime` plus a dedicated test |
| Time-dependent tests drifting | Everything uses `Clock`, never `datetime.now()` in rules |
| Over-building and missing the time box | Tier cut lines in Section 11 |
| Reviewer's Python version | `requires-python >= 3.11`, CI on 3.11 and 3.12 |

### 13.2 Definition of done (check before submitting)

- [ ] Fresh clone, then `make test` passes with no Docker and no manual setup
- [ ] `docker compose up` serves the API on Postgres, and `make test-pg` passes
- [ ] `make demo` prints exactly one 201 and nineteen 409s
- [ ] The race matrix R1–R7 each have a passing test
- [ ] Bypass test proves the database rejects a second confirmed booking
- [ ] Every error response uses the envelope and carries a `request_id`
- [ ] `/docs` shows all endpoints with error responses and examples
- [ ] `ruff` and `mypy --strict` clean, CI green on SQLite and Postgres
- [ ] README has setup, endpoint table, `curl` examples (including a failing one), test instructions, assumptions, trade-offs
- [ ] No secrets, no `*.db` files, no stray debug code committed
- [ ] Commit history is clean, with meaningful messages (scaffold, schema, services, API, tests, docs)
- [ ] Repository is public (or access is granted) and the link is submitted on HireSol

---

## 14. Errata and decisions (v2.1)

This section supersedes any conflicting statement in Sections 1–13. `CLAUDE.md` records the rule that §14 overrides earlier sections. Tier 2 remains out of scope; only R1–R7 are required.

### 14.1 Backstop scope and Postgres requirements

- `btree_gist` is a **hard requirement** on Postgres. The migration fails fast if `CREATE EXTENSION btree_gist` cannot succeed. The earlier claim that the service-level overlap check still works without it is withdrawn: slot creation takes no row lock, so without the exclusion constraint two concurrent overlapping inserts can both succeed.
- Slot creation is protected by the Postgres exclusion constraint or by SQLite `BEGIN IMMEDIATE`. It takes no row lock and is **outside the lock order**.
- The database enforces exactly two booking-related invariants: **at most one confirmed booking per slot** (partial unique index) and **no overlapping open slots per clinic** (Postgres exclusion constraint).
- "A cancelled slot never has a confirmed booking" is enforced by the service inside the slot lock (the cascade) and proven by the R2 test. It is **not** enforced by a database constraint.

### 14.2 Idempotency

- Idempotency keys are scoped per patient, immutable, and never expire.
- A replay returns **200** with the original booking in its **current** state, even if it has since been cancelled. The response carries `Idempotent-Replay: true` and no `Location` header.
- To book again after a cancellation, the client must use a new key.
- Same key and same slot returns the replay; same key and a different slot returns **422 `IDEMPOTENCY_KEY_MISMATCH`**. Keys are case-sensitive.
- Concurrent same key, different slots, on Postgres: on an `IntegrityError` for `booking_idempotency`, roll back, open a **new** transaction, and re-read by `(patient_id, key)`. Same slot returns the replay; different slot returns 422. A test covers this.

### 14.3 Cancel truth table

For both booking cancel and slot cancel, in this order:

1. If the resource is already cancelled: return **200** with a no-op body, even if the slot has started.
2. Otherwise, if `now >= starts_at`: return **409 `SLOT_STARTED`**.
3. Otherwise: apply the transition.

A slot cancel returns `{slot, cancelled_booking}`. `cancelled_booking` is the full booking object cancelled by this slot cancellation, or `null`. The slot and the cascaded booking use the **same** `cancelled_at` value, so a repeat call returns the same body.

### 14.4 Dialect DDL and time

- DDL is written per dialect. Use `length()` (available on both databases) instead of `char_length()`. There is no `EXCLUDE` constraint on SQLite. The migration branches on dialect.
- On SQLite, `UTCDateTime` stores one fixed canonical UTC string form so that string ordering matches chronological ordering. It returns aware datetimes, and API output serializes with `Z`. A round-trip test and an ordering test cover this.
- `started` means `now >= starts_at`.
- The 365-day horizon and the 5- and 480-minute duration limits are inclusive.

### 14.5 Error mapping

- Map to **503 `SERVICE_UNAVAILABLE`** only connection-level failures and SQLite "database is locked" after `busy_timeout` is exhausted. Other `OperationalError`s are **500 `INTERNAL_ERROR`**.
- Translate `IntegrityError` by constraint or index name in **one central function**, on both dialects:
  - `one_confirmed_booking_per_slot` → `SlotAlreadyBooked`
  - `booking_idempotency` → replay or `IDEMPOTENCY_KEY_MISMATCH` (see §14.2)
  - the patients email unique constraint → `EmailAlreadyRegistered`
  - the Postgres exclusion constraint → `SlotOverlap`

### 14.6 SQLite transactions

- Use a plain deferred session for the auth key lookup and for read-only GETs.
- Use `BEGIN IMMEDIATE` only for write use cases, issued by the Unit of Work.
- Use SQLAlchemy's documented pysqlite workaround: set the driver `isolation_level=None` in the connect event and emit an explicit `BEGIN` in the begin event.
- Add a test proving a write use case holds the write lock: while it is open, a second connection with `busy_timeout=0` attempting `BEGIN IMMEDIATE` gets "database is locked".

### 14.7 Race tests

- One session per worker, a file database, and a pool size greater than the thread count (or `NullPool`). No shared test-client session, no sleeps.

### 14.8 Scope and lock order

- Tier 2 is out of scope entirely; `CLAUDE.md` wins. Only R1–R7 are required.
- The global lock order remains patient → slot → booking. The **patient and booking levels are reserved and unused**.
- Slot creation is outside the lock order (§14.1).

### 14.9 Authentication

- Bearer API keys only. The header-based fallback (`X-Clinic-Id` / `X-Patient-Id`) is deleted.
- The key prefix selects the lookup table (`clk_` → clinics, `pat_` → patients). An unknown key returns **401**; a valid key of the wrong kind returns **403**.
- 403-after-existence reveals that a resource exists. The README states explicitly that this is a deliberate trade-off.

### 14.10 Listing, filters and pagination

- Slot listing `from`/`to` means `starts_at` ∈ `[from, to)`, half-open.
- A slot cursor is a position only: `(starts_at, id)`. It is not bound to filters. A malformed cursor returns **422 `INVALID_CURSOR`**.
- Booking lists (patient and clinic) are ordered by `created_at DESC, id DESC` with a `(created_at, id)` cursor.
- An unknown clinic on `GET /v1/clinics/{clinic_id}/slots` returns **404 `CLINIC_NOT_FOUND`**.

### 14.11 Registration, readiness and developer workflows

- Concurrent duplicate patient email: catch the unique violation and return **409 `EMAIL_ALREADY_REGISTERED`**. Normalize the address with the email validator, then lowercase it.
- `/ready` runs `SELECT 1` and verifies that `alembic_version` equals the head revision.
- `make test-pg` requires `TEST_DATABASE_URL`. The README documents `docker compose up -d db` and the URL to use. Every Make target also has a raw PowerShell and bash command in the README.

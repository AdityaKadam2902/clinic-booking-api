# CLAUDE.md: Clinic Booking API

You are building a take-home submission for a Full Stack Developer Intern role. Reviewers score Correctness (3), API design (2), Code quality (2), Testing (2), and they check the booking rules first. The goal is not a long feature list. The goal is a small system whose guarantees are **provably correct, easy to verify, and clearly explained**.

## Source of truth

`ARCHITECTURE.md` in this repo is the spec. Read the sections named in each phase prompt before writing code. If the spec and these rules conflict, these rules win. If the spec is silent or ambiguous, stop and ask. Do not guess on anything that touches concurrency, time, or the schema.

## Stack (fixed, do not substitute)

Python 3.11+, FastAPI (sync endpoints), SQLAlchemy 2.0 (sync), Alembic, Pydantic v2, pydantic-settings, psycopg 3 for Postgres, SQLite for zero-setup, pytest, httpx, ruff, mypy.

## Non-negotiable rules

1. **The double-booking guarantee is three layers:** slot row lock, rule checks inside the lock, and the partial unique index as backstop. Never rely on an application check alone. Never remove a layer to make a test pass.
2. **Schema comes from Alembic migrations only.** Never call `Base.metadata.create_all()`, not in app code and not in tests.
3. **SQLite needs `BEGIN IMMEDIATE`.** `with_for_update()` does nothing on SQLite. Service transactions must run `IMMEDIATE` there via an engine event, with WAL, `busy_timeout` and `foreign_keys=ON`.
4. **All timestamps are timezone-aware UTC.** Reject naive datetimes at the API. Use the `UTCDateTime` type so SQLite round-trips stay aware.
5. **Never call `datetime.now()` / `datetime.utcnow()` in rules or services.** Take `now` from the injected `Clock`.
6. **Lock order is patient, then slot, then booking.** Endpoints that start from a booking id use read, lock slot, re-read.
7. **Layering:** routers call services; services call repositories and `domain/` rules; `domain/` imports no FastAPI or SQLAlchemy. No `HTTPException` below the API layer. Services raise `DomainError` subclasses; one handler maps them to the error envelope.
8. **One error envelope for everything**, including framework 422s and unexpected 500s, always with `request_id` and a stable `code`. Never leak a stack trace.
9. **Identity comes from the Bearer API key only.** Never accept `patient_id` or `clinic_id` in a body as the acting identity. Keys are stored as SHA-256 hashes and shown once.
10. **Never log or return PII beyond what an endpoint's spec allows.** Public slot views never reveal who booked.
11. **Tests:**
    - No `time.sleep` for synchronization. Races use `threading.Barrier`.
    - Never weaken, skip, or delete a failing test to get green. Fix the code, or stop and explain.
    - Race tests assert both the response counts and the database state afterwards.
    - Every test is named after the rule it protects.
12. **No scope creep.** Tier 2 items in the spec (patient-overlap, batch create, cancellation window) are off limits unless the prompt says so. No extra frameworks, caches, queues, or generic base classes.
13. **No dead flexibility.** No config option, abstraction, or parameter that nothing uses.

## Code standards

- `mypy --strict` clean on `app/`. `ruff check` and `ruff format` clean.
- Every public function and class has a type signature. Docstrings only where the *why* is not obvious (locking, ordering, dialect differences). Comments explain intent, never restate code.
- Service methods read like the pseudocode in ARCHITECTURE.md §5.4 and fit on one screen.
- Names are domain words: `SlotAlreadyBooked`, `assert_bookable`, `cancel_slot`. No `utils.py`, no `helpers.py`, no `manager`.
- Dependencies are pinned by lower bound in `pyproject.toml`. Add nothing the spec does not call for.

## Commands (define in the Makefile in Phase 0)

| Command | Purpose |
|---|---|
| `make install` | venv and dev dependencies |
| `make run` | migrate, then serve on SQLite at :8000 |
| `make test` | full suite on SQLite |
| `make test-pg` | full suite on Postgres (needs `TEST_DATABASE_URL`) |
| `make lint` | ruff check, ruff format --check, mypy |
| `make demo` | race demo script |

Run `make lint test` before reporting any phase complete.

## Working method

- Work **one phase at a time** and stop at the end of it. Do not start the next phase.
- At the start of a phase, post a plan in 10 lines or fewer, then execute.
- Keep commits small and meaningful, one logical change each, imperative messages (`Add partial unique index for confirmed bookings`). No "WIP" or "fix" commits.
- At the end of a phase, report: what you built, which spec sections it covers, test results (counts and the command used), **any deviation from the spec and why**, and anything you were unsure about. Never claim something works without having run it.
- If a command fails, show the error and fix the cause. Do not paper over it.
- If you find a flaw in the spec, say so plainly and propose a fix before implementing anything different.

## Definition of done

The project is done only when the checklist in ARCHITECTURE.md §13.2 is fully ticked, including a fresh-clone run of `make test` and `make demo` in a different directory.

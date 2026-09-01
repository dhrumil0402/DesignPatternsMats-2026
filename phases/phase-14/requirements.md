# Phase 14 — Tests, docs, demo (Requirements) *(Optional enrichment)*

This is the **assignment**. Implement from these requirements. If you get stuck or want to check your design, use the [guided check](guided-check.md) (outlines and expected coverage — not a complete solution dump).

> **Optional enrichment** — This phase is not required for course completion.
> Complete Phases 1–12 first (Phase 13 recommended but not required). Do this if you want proof-and-demo rigor.

This phase is **not** a GoF pattern lab. It proves the system as an integrated whole.

## Purpose

Show that each pattern’s **extension point** is understandable, that tests cover the seams, and that a reviewer can follow **code + docs** from Phase 1 through 12 (and optional enrichment if completed) without an oral walkthrough.

## Scope and naming rules

- Integration tests use a **test database** (or transactional rollback) with migrations applied — not production data.
- Do not skip migration smoke: empty DB to head must work in CI or a documented local script.
- Pattern docs live under `docs/patterns/`; demo narrative is README or `docs/demo.md`.

## Outcome required at end of phase

- Unit tests for each major pattern seam (Strategy swap, Builder validation, State transitions, Decorator policies, plus earlier factory/adapter/command/observer as you have them).
- Backend integration tests: HTTP + DB.
- Migration smoke: upgrade head on empty Postgres.
- Optional frontend component tests; lightweight E2E or API scenario script for a happy path.
- Demo narrative touching many patterns in one story.
- Each `docs/patterns/*.md` has problem, intent, code pointers, and a “try this extension” idea.
- A reviewer can start from docs without undocumented tribal knowledge.

---

## Prerequisites

- **Required track complete:** Phase 12 minimum.
- **Recommended:** Phase 13 (UI exercises the backend realistically)—not required for this optional phase.

---

## Required architecture for this phase

- **Tests:** unit (domain), integration (API + migrated DB), optional Playwright/Cypress or API-only scenario.
- **Docs:** patterns folder complete; demo story; README links.
- **CI (if you have it):** pytest + migration job. If no CI, a script in README that a reviewer can run.

---

## Suggested file locations

Paths are relative to **`yourpath\project\`**.

```text
yourpath\project\
├── backend/tests/
│   ├── test_sensor_creators.py
│   ├── test_family_factory.py
│   ├── test_location_config_builder.py
│   ├── test_adapters.py
│   ├── test_strategies.py
│   ├── test_state_transitions.py
│   ├── test_decorators.py
│   ├── test_commands.py
│   ├── test_observer_alerts.py
│   └── integration/
│       ├── conftest.py
│       ├── test_locations_api.py
│       ├── test_readings_api.py
│       ├── test_automation_api.py
│       └── test_commands_api.py
├── docs/patterns/*.md
└── docs/demo.md                  # or a Demo section in README.md
```

## Type hints

| Item | Hint |
|------|------|
| Test `DATABASE_URL` | `str` — separate DB name, e.g. `postgresql+psycopg://...@localhost:5432/greenhouse_test` |
| `PYTHONPATH` | `src` when running pytest from `backend/` |
| Integration client | FastAPI `TestClient`; override `get_db` to yield the test session |

Do not invent new domain entities in this phase; reuse types from Phases 2–11.

---

## Step-by-step implementation requirements

## Step 1 — Unit tests (pattern seams)

Minimum coverage (add any missing from earlier phases):

| Seam | What to assert |
|------|----------------|
| Factory Method | Distinct creator defaults |
| Abstract Factory | Family kits differ |
| Builder | Invalid config rejected |
| Adapter | Vendor payload normalized |
| Strategy | Two keys, same context, different decide |
| Facade | (optional unit with fakes) overview keys |
| State | Illegal transition raises |
| Decorator | Policy can block inner |
| Command | Rejected does not change state |
| Observer | Subscriber runs on publish |

Acceptance criteria:

- These tests run without a live UI.
- Failures name the pattern/module clearly.

## Step 2 — Integration / HTTP + DB

- Test database URL distinct from dev if possible.
- Run migrations before the suite.
- Truncate or rollback per test.
- Cover at least: create location config; take a reading; evaluate automation; post a command.

Acceptance criteria:

- Suite is reliable locally (no sleep-based flakes as the only sync).

## Step 3 — Migration smoke

Documented command: empty Postgres → `alembic upgrade head` succeeds.

Acceptance criteria:

- A reviewer following README can reproduce this.

## Step 4 — Frontend / E2E (lightweight)

At least one of:

- Component tests (Strategy panel / disabled controls), or
- Playwright/Cypress, or
- API-only script documenting the click path.

Scenario to cover if E2E/API script:

1. Health OK.
2. Evaluate automation → recommendation visible.
3. Start pump → state `running` → event in feed (or command log).
4. Optional: moisture recovers / alert path if modeled.

Acceptance criteria:

- The path is written down even if automation is manual for the demo recording.

## Step 5 — Demo narrative

Document in README or `docs/demo.md` a single story, for example:

1. Sensor read via Adapter → Observer publishes `reading.created`.
2. Strategy recommends irrigation; Facade shows it on overview.
3. Operator Command; State allows Idle → Running; Decorator policies apply.
4. Event feed + WebSocket update UI.
5. Moisture recovers; alert handling (optional Phase 13 polish, if implemented).

Acceptance criteria:

- Each step points at a UI area **and** a module path.

## Step 6 — Pattern docs cleanup

Each `docs/patterns/*.md` includes:

- Problem in greenhouse terms
- Pattern name + intent
- Where to look in code
- “Try this extension” homework idea

Acceptance criteria:

- Phases 2–11 each have a matching pattern file.

---

## Definition of done

- Tests described above exist and pass (or skipped items are listed with reason).
- Migration smoke is documented and works.
- Demo narrative exists.
- Pattern docs are complete.
- This is the **final optional enrichment** phase; further work (hardware, auth, multi-tenant) is after this closure point.

---

## Common pitfalls to avoid

- Testing only mocks and never hitting Postgres.
- Demo story that does not match the running app.
- Pattern docs that copy textbook text instead of *this* repo’s modules.

---

## Handoff

**Optional enrichment track complete.** Required course completion is at Phase 12. Index: [`docs/phases/README.md`](../README.md).

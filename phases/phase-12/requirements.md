# Phase 12 — Production API, WebSocket, schema hardening (Requirements)

This is the **assignment**. Implement from these requirements. If you get stuck or want to check your design, use the [guided check](guided-check.md) (signatures and partial snippets — not a complete solution).

This phase is **not** a GoF pattern lab. It hardens the API, adds WebSocket fan-out from the Observer bus, and tightens the schema that already grew in Phases 2–11.

## Purpose

The database **already exists**. This phase focuses on:

- A **production API** surface (stable `/api` routes). Scalar at `/scalar` remains the reference.
- **WebSocket** fan-out from Observer (no duplicate business logic).
- **Schema hardening:** indexes, FK `ON DELETE` policies, optional seeds — not “first time we add a database.”

## Scope and naming rules

- Remove or gate any `/dev/*` routes; standardize on `/api/*` (optional `/api/v1/*` if you version).
- Align React `api.ts` with the same paths — one client module, not scattered URLs.
- WebSocket subscriber **listens to the EventBus**; it must not re-implement Strategy, Command, or Adapter.
- Keep `location_id` naming.

## Outcome required at end of phase

- Fresh `alembic upgrade head` from an empty database creates the full schema through this phase’s hardening revision.
- No critical dashboard flow depends on in-memory-only stores.
- WebSocket pushes at least `reading.created`, `actuator.state_changed`, `alert.created`.
- UI connection indicator reflects WS status; EventFeed can subscribe instead of poll-only.
- Scalar still documents REST; WS is described in README or phase notes.
- Seed data (optional but recommended) is idempotent.

---

## Prerequisites

- Phase 11 completed: event bus, alerts table, polling feed, `realtime.ts` stub.

---

## Required architecture for this phase

- **Application:** connection manager for WS clients; a subscriber that forwards bus events as JSON messages.
- **API:** REST path cleanup; `WS /api/ws` (or documented equivalent).
- **Infrastructure:** hardening migration (indexes, FKs, seeds). No new domain tables unless you found a gap — then add a new revision, do not rewrite old ones.
- **Frontend:** `realtime.ts` real implementation; header connection indicator; `api.ts` base paths updated.

---

## Suggested file locations

Paths are relative to **`yourpath\project\`**.

```text
yourpath\project\
├── backend/src/
│   ├── application/realtime/connection_manager.py
│   ├── application/realtime/ws_subscriber.py    # EventBus → broadcast
│   └── interfaces/api/ws.py
└── frontend/src/services/
    ├── api.ts                                   # /api only
    └── realtime.ts                              # WebSocket client
```

## Type hints

WebSocket JSON envelope:

| Field | Hint |
|-------|------|
| `type` | `str` — at least `reading.created`, `actuator.state_changed`, `alert.created` |
| `payload` | `object` (event-specific; keep ids as UUID strings) |

Frontend:

| Name | Hint |
|------|------|
| `RealtimeEvent` | `{ type: string; payload: unknown }` |
| `subscribeRealtime` | `(onEvent: (e: RealtimeEvent) => void) => () => void` |
| WS URL | derive from `VITE_API_BASE_URL` (`ws://` / `wss://`), path `/api/ws` |

Connection indicator states: `connected` \| `connecting` \| `down` (`str`).

---

## Step-by-step implementation requirements

## Step 1 — Route convergence

Inventory all routes. Every feature the dashboard uses must live under `/api/...`.

Minimum capabilities to keep reachable:

| Capability | Example route |
|------------|-----------------|
| Overview | `GET /api/locations/{id}/overview` |
| Sensors | `GET /api/sensors`, `POST /api/sensors/{id}/read` |
| Config | `GET/POST /api/locations/.../config` |
| Automation | `POST /api/locations/{id}/automation/evaluate` |
| Commands | `POST /api/commands`, `GET /api/commands` |
| Alerts | `GET /api/alerts`, optional `PATCH /api/alerts/{id}` |
| Executions | `GET /api/actuators/executions` |

Acceptance criteria:

- No production UI call hits `/dev`.
- Scalar lists the production paths.

## Step 2 — Schema hardening migration

Forward-only revision (name such as `schema_hardening`):

- Indexes you deferred (e.g. readings `(device_id, recorded_at)`, alerts `(status, created_at)`).
- Documented `ON DELETE` behavior on FKs.
- Optional NOT NULL tightening only if the app already supplies defaults.
- Optional **idempotent** seed: one demo location, devices, one reading.

Acceptance criteria:

- `alembic upgrade head` from empty Postgres succeeds through this revision.
- You do not edit applied Phase 2–11 revisions.

## Step 3 — WebSocket

- Endpoint `WS /api/ws` (auth stub allowed: accept all in course mode).
- Server: Observer subscriber → connection manager broadcast.
- Message types include at least reading created, actuator state changed, alert created.
- Client: implement `realtime.ts` (connect, parse JSON, reconnect/backoff can be minimal here; optional Phase 13 can deepen reconnect/backoff UX).
- EventFeed prefers WS; polling may remain as fallback (document which).

Acceptance criteria:

- Trigger a read (or command) → UI updates **without** full page refresh **and** without relying solely on the Phase 11 poll interval if WS is connected.
- Disconnecting the server shows a disconnected indicator (even a simple badge).

## Step 4 — Frontend path + indicator

- Centralize API base URL; fix any leftover `/dev` or inconsistent hosts.
- Header (or layout) shows WS connected / connecting / down.

Acceptance criteria:

- All Phase 2–11 features still work after a fresh migrate + seed.

## Step 5 — Documentation

- Short note in root README: WS URL, how to smoke-test.
- Optional `docs/schema.md` listing migration chain — or a table in `docs/phases/README.md`.

Acceptance criteria:

- A reviewer can migrate empty DB and open dashboard against seed/demo data.

---

## Definition of done

- Production `/api` only for UI.
- Hardening migration applied; empty-DB migrate works.
- WS fan-out from the same EventBus.
- `realtime.ts` is no longer a no-op stub.
- Features still work after migrate + seed.

---

## Common pitfalls to avoid

- Duplicating alert logic in the WS handler instead of subscribing to the bus.
- Breaking Scalar by disabling OpenAPI.
- Hand-editing old migration files that classmates already applied.
- Forgetting to update `api.ts` after renaming routes.

---

## Course completion (required track)

You have met the **required course** when the Definition of done above passes. No further phases are mandatory for credit.

Checklist:

- [ ] Production `/api` only for UI
- [ ] Hardening migration applied; empty-DB migrate works
- [ ] WS fan-out from the same EventBus
- [ ] `realtime.ts` is no longer a no-op stub
- [ ] Features still work after migrate + seed

---

## Optional next steps

Want operator-grade polish or proof-and-demo rigor? These phases are **not required** for course completion:

- [Phase 13 — Dashboard polish (optional)](../phase-13/requirements.md) · [guided check](../phase-13/guided-check.md) — charts, empty/error states, UX cohesion
- [Phase 14 — Tests, docs, demo (optional)](../phase-14/requirements.md) · [guided check](../phase-14/guided-check.md) — test pyramid, pattern docs, demo narrative

# Phase 11 — Observer (Requirements)

This is the **assignment**. Implement from these requirements. If you get stuck or want to check your design, use the [guided check](guided-check.md) (signatures and partial snippets — not a complete solution).

After you implement, answer the [questions](./questions.md).

## Purpose

Introduce an in-process **event bus** (publish/subscribe). Persist **alerts** so the event feed survives refresh. Phase 12 will fan the same events out over WebSocket — do not implement WS here beyond a client **stub**.

## Scope and naming rules

- Publishers (reading ingest, command executor, state transitions) must not import UI or HTTP. They publish domain events.
- Every successful ingest publishes `reading.created` with `device_id`, `value`, `unit`, `source`, and `recorded_at`. Skip that publish when `tracking_enabled` is false (the row may still be stored). Manual read, the simulation sampler, and a translated MQTT payload all use this same publish rule.
- Subscribers handle persistence (alerts) and optional metrics. Alert rules subscribe to the bus; they do not read the MQTT socket. Do not add a broker client or the device readings route in this phase.
- `location_id` remains the site scope on alerts when known.

## Outcome required at end of phase

- Event bus with at least an alert-persistence subscriber.
- Ingest publishes `reading.created` only while that device’s `tracking_enabled` is true.
- Low moisture (or threshold-crossed) reading creates an `alerts` row visible in the feed after poll.
- `GET` alerts/events API supports listing (and optionally `since` / `status=active`).
- Dashboard event feed polls the API. Sensor cards may keep the Phase 5 readings poll.
- `realtime.ts` interface exists as a stub for Phase 12.
- Scalar documents alerts endpoints.
- Pattern note at `docs/patterns/observer.md`.

---

## Prerequisites

- Phase 10: commands and state changes to react to.
- Phase 5: ingest path (`ReadingIngest`) that already stores readings from manual read, the sampler, and MQTT translation.

---

## Required architecture for this phase

- **Domain:** event types; `EventBus` subscribe/publish; subscriber interface.
- **Application:** subscribers that persist alerts; publishers called from existing services (ingest, command). Ingest publishes `reading.created` after a successful insert when `tracking_enabled` is true.
- **Infrastructure:** `alerts` table + repository.
- **API:** list alerts (filter by status).
- **Frontend:** EventFeed polling; realtime module stub.

---

## Suggested file locations

Paths are relative to **`yourpath\project\`**.

```text
yourpath\project\
├── backend/src/
│   ├── domain/events/events.py           # ReadingCreated, ThresholdCrossed, CommandFailed, ...
│   ├── domain/events/bus.py              # EventBus
│   ├── application/alerts/subscribers.py
│   ├── application/alerts/dto.py
│   ├── infrastructure/persistence/models.py              # AlertRow
│   ├── infrastructure/persistence/alert_repository.py
│   └── interfaces/api/alerts.py
└── frontend/src/
    ├── services/realtime.ts              # stub
    └── .../EventFeed.tsx                 # #events
```

## Type hints

Domain events (plain objects/dataclasses): include `device_id: UUID | None`, `location_id: UUID | None`, and event-specific fields (e.g. moisture `float` on threshold crossed).

ORM `alerts`:

| Column | SQL hint |
|--------|----------|
| `id` | UUID PK |
| `location_id` | UUID FK, **nullable** |
| `device_id` | UUID FK, **nullable** |
| `severity` | VARCHAR — `"info"` \| `"warning"` \| `"critical"` |
| `message` | TEXT |
| `event_type` | VARCHAR (e.g. `threshold.crossed`, `command.failed`) |
| `status` | VARCHAR — `"active"` \| `"resolved"` |
| `created_at` | timestamptz |
| `acknowledged_at` | timestamptz, nullable |

Alert DTO JSON: `{ id, location_id, device_id, severity, message, event_type, status, created_at }`.

Frontend stub: `RealtimeEvent = { type: string; payload: unknown }`; `subscribeRealtime` returns an unsubscribe `() => void`.

---

## Step-by-step implementation requirements

## Step 1 — Database

`alerts`:

| Column | Intent |
|--------|--------|
| Identifier | PK |
| `location_id` | FK, nullable |
| `device_id` | FK, nullable |
| `severity` | `info` / `warning` / `critical` |
| `message` | Text |
| `event_type` | e.g. `threshold.crossed`, `command.failed` |
| `status` | `active` / `resolved` |
| `created_at` | Timestamptz |
| `acknowledged_at` | Nullable |

Optional extra table for append-only domain events is **not** required.

Autogenerate and apply.

Acceptance criteria:

- Schema uses `location_id` if the FK is present.

## Step 2 — Bus and subscribers

- `EventBus` with subscribe and publish.
- `AlertPersistenceSubscriber` inserts into `alerts` for relevant event types (at least threshold crossed / low moisture, and command failed).
- Optional `MetricsSubscriber` with no table.

Publish from: every successful ingest (`reading.created` with `device_id`, `value`, `unit`, `source`, `recorded_at`), command results, and important state transitions. Do not publish `reading.created` when `tracking_enabled` is false.

Acceptance criteria:

- Publishers do not call the alerts repository directly (they publish).
- Subscriber insert is covered by a test.
- A sampler or `POST /read` on a tracked device publishes `reading.created`. The same ingest with tracking off stores the row and does not publish.

## Step 3 — API and UI

- `GET /api/alerts?status=active` and/or `GET /api/events?since=`.
- Event feed in `#events` polling every few seconds (interval documented). Empty state.
- `frontend/src/services/realtime.ts`: exported functions/types for subscribe — implementation can be no-op or poll wrapper; comment that Phase 12 replaces this with WebSocket. Leave the Phase 5 readings poll in place.

Acceptance criteria:

- Trigger a low moisture read (or seed) → new alert row → visible in feed after poll.
- Refresh still shows alerts (DB-backed).

## Step 4 — Tests and docs

- Subscriber inserts on published event.
- Ingest with tracking on publishes `reading.created`; tracking off does not.
- Integration: publish → GET alerts nonempty.
- `docs/patterns/observer.md`.

---

## Definition of done

- Bus + alert subscriber working.
- `reading.created` is published from ingest only when tracking is on.
- `alerts` table filled from events.
- Polling feed on the dashboard. Readings poll may remain.
- Realtime stub in place.
- Tests and pattern doc complete.

---

## Common pitfalls to avoid

- Fetching alerts inside the reading adapter (tight coupling).
- WebSocket implementation in this phase (belongs in Phase 12).
- Alerts only in React state.
- Publishing `reading.created` for a device with `tracking_enabled` false.
- A separate publish path for the sampler versus `POST /read`.

---

## Handoff to next phases

- Phase 12 attaches a dashboard WebSocket subscriber to the same bus, replaces the sensor-card readings poll while that socket is up, and may deliver readings on the device HTTP route or from an optional MQTT broker. Neither transport may become a second publisher; each calls ingest, which already publishes.

→ [Phase 12 — Production API, WebSocket, hardening (requirements)](../phase-12/requirements.md) · [guided check](../phase-12/guided-check.md)

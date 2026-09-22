# Phase 12 — Production API, WebSocket, schema hardening (Requirements)

This is the **assignment**. Implement from these requirements. If you get stuck or want to check your design, use the [guided check](guided-check.md) (signatures and partial snippets — not a complete solution).

This phase is **not** a GoF pattern lab. It hardens the API, adds WebSocket fan-out from the Observer bus, and tightens the schema that already grew in Phases 2–11.

## Purpose

The database **already exists**. This phase focuses on:

- A **production API** surface (stable `/api` routes). Scalar at `/scalar` remains the reference.
- **WebSocket** fan-out from Observer (no duplicate business logic). Sensor cards follow `reading.created` and stop the Phase 5 readings poll while the socket is up. This socket is the dashboard channel, not a device channel.
- **Device HTTP** for ESP32-style controllers with `protocol: http`. The device GETs its sampling config and POSTs readings. The route calls the Phase 5 translator and then ingest. It does not parse a second reading type and it does not publish domain events. There is no device WebSocket.
- **Optional MQTT subscriber** for `protocol: mqtt`. It starts only when a broker URL is set. It calls the same translator and ingest. It does not parse readings itself and it does not publish domain events itself.
- **Schema hardening:** indexes, FK `ON DELETE` policies, optional seeds — not “first time we add a database.”

## Scope and naming rules

- Remove or gate any `/dev/*` routes; standardize on `/api/*` (optional `/api/v1/*` if you version).
- Align React `api.ts` with the same paths — one client module, not scattered URLs.
- WebSocket subscriber **listens to the EventBus**; it must not re-implement Strategy, Command, or Adapter. Do not accept ESP32 connections on `WS /api/ws`.
- `POST /api/devices/{device_id}/readings` calls the Phase 5 translator with the body dict and `source` `http`, then `ReadingIngest`. `GET /api/devices/{device_id}/sampling` returns the stored interval and tracking flag. Unknown `device_id`: log and drop. Do not publish `reading.created` from the route.
- MQTT subscriber listens on the broker only when a broker URL is set, and calls the same translator with `source` `mqtt`, then `ReadingIngest`. Unknown `device_id`: log and drop. Do not open the broker socket from the domain. If the Phase 5 translator always sets `source` to `mqtt`, add an optional source argument that defaults to `mqtt` so existing tests stay valid.
- Course definition of done stays green with only simulation devices, no ESP32, and no broker. Device HTTP is a smoke test that GETs sampling and POSTs one payload. MQTT is that same adapter test plus a README smoke test that publishes one payload when the broker profile is up.
- Keep `location_id` naming. Do not introduce `greenhouse_id` (the topic prefix `greenhouse/` is a channel name, not a column).

## Outcome required at end of phase

- Fresh `alembic upgrade head` from an empty database creates the full schema through this phase’s hardening revision.
- No critical dashboard flow depends on in-memory-only stores.
- WebSocket pushes at least `reading.created`, `actuator.state_changed`, `alert.created`. `reading.created` and `alert.created` include `zone_id` when the device has one so the zone device list can update. MQTT topics stay `greenhouse/devices/{device_id}/...`.
- Sensor cards update from `realtime.ts` and stop the Phase 5 readings poll while the socket is connected. Polling remains the documented fallback when the socket is down.
- UI connection indicator reflects WS status; EventFeed can subscribe instead of poll-only.
- `protocol: http` devices use `GET /api/devices/{device_id}/sampling` and `POST /api/devices/{device_id}/readings`. Sampling PATCH for those devices only updates the columns that GET returns. The device applies the interval locally. The backend does not poll the ESP32 and does not push config.
- Mosquitto (or equivalent) is a Compose **profile**, not part of a normal `docker compose up`. The subscriber starts only when a broker URL is set. Topic `greenhouse/devices/{device_id}/reading`. Sampling PATCH for `protocol=mqtt` publishes a retained `{ sampling_interval_seconds, tracking_enabled }` on `greenhouse/devices/{device_id}/config` only when that broker is configured.
- Scalar still documents REST, including the device GET and POST. Dashboard WS and optional MQTT topics are described in README or phase notes.
- Seed data is idempotent and includes one simulation sensor with `sampling_interval_seconds` of `5` and `tracking_enabled` true, so the dashboard moves with no hardware.

---

## Prerequisites

- Phase 11 completed: event bus, alerts table, polling feed, `realtime.ts` stub.

---

## Required architecture for this phase

- **Application:** connection manager for dashboard WS clients; a subscriber that forwards bus events as JSON messages; device HTTP handler that translates then calls ingest; MQTT subscriber that does the same only when a broker URL is set.
- **API:** REST path cleanup; `WS /api/ws` (or documented equivalent) for the dashboard only; `GET /api/devices/{device_id}/sampling` and `POST /api/devices/{device_id}/readings`. Sampling PATCH for `protocol=http` updates columns only. Sampling PATCH for `protocol=mqtt` also publishes the retained config message when a broker is configured.
- **Infrastructure:** hardening migration (indexes, FKs, seeds); Mosquitto in a Compose profile. No new domain tables unless you found a gap — then add a new revision, do not rewrite old ones. No device WebSocket.
- **Frontend:** `realtime.ts` real implementation; sensor cards consume `reading.created`; header connection indicator; `api.ts` base paths updated.

---

## Suggested file locations

Paths are relative to **`yourpath\project\`**.

```text
yourpath\project\
├── backend/src/
│   ├── application/realtime/connection_manager.py
│   ├── application/realtime/ws_subscriber.py    # EventBus → broadcast
│   ├── infrastructure/mqtt/subscriber.py        # optional broker → translator → ingest
│   ├── interfaces/api/ws.py                     # dashboard WS only
│   └── interfaces/api/devices.py                # GET sampling, POST readings
└── frontend/src/services/
    ├── api.ts                                   # /api only
    └── realtime.ts                              # WebSocket client
```

## Type hints

WebSocket JSON envelope:

| Field | Hint |
|-------|------|
| `type` | `str` — at least `reading.created`, `actuator.state_changed`, `alert.created` |
| `payload` | `object` (event-specific; keep ids as UUID strings; include `zone_id` on reading and alert payloads when the device has a zone) |

Frontend:

| Name | Hint |
|------|------|
| `RealtimeEvent` | `{ type: string; payload: unknown }` |
| `subscribeRealtime` | `(onEvent: (e: RealtimeEvent) => void) => () => void` |
| WS URL | derive from `VITE_API_BASE_URL` (`ws://` / `wss://`), path `/api/ws` |

Connection indicator states: `connected` \| `connecting` \| `down` (`str`).

Device HTTP:

| Name | Hint |
|------|------|
| POST body | `{ value: float, unit: str }` |
| GET sampling | `{ sampling_interval_seconds: int, tracking_enabled: bool }` |
| Stored `source` | `"http"` on this route; `"mqtt"` on the broker path |

---

## Step-by-step implementation requirements

## Step 1 — Route convergence

Inventory all routes. Every feature the dashboard uses must live under `/api/...`.

Minimum capabilities to keep reachable:

| Capability | Example route |
|------------|-----------------|
| Overview | `GET /api/locations/{id}/overview` |
| Sensors | `GET /api/sensors`, `POST /api/sensors/{id}/read` |
| Device HTTP | `GET /api/devices/{id}/sampling`, `POST /api/devices/{id}/readings` |
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
- Optional **idempotent** seed: one demo location, devices, one reading, and one simulation sensor with `sampling_interval_seconds` `5` and `tracking_enabled` true.

Acceptance criteria:

- `alembic upgrade head` from empty Postgres succeeds through this revision.
- You do not edit applied Phase 2–11 revisions.

## Step 3 — WebSocket

- Endpoint `WS /api/ws` (auth stub allowed: accept all in course mode).
- Server: Observer subscriber → connection manager broadcast.
- Message types include at least reading created, actuator state changed, alert created.
- Client: implement `realtime.ts` (connect, parse JSON, reconnect/backoff can be minimal here; optional Phase 13 can deepen reconnect/backoff UX).
- Sensor cards apply `reading.created` in place and **stop** the Phase 5 `GET .../readings` poll while the socket is connected. The zone device list updates from `reading.created` and `alert.created` when `zone_id` is present. If the socket is down, resume that poll (document this fallback).
- EventFeed prefers WS; alert polling may remain as fallback (document which).

Acceptance criteria:

- With the seeded simulation sensor and tracking on, the card value changes **without** “Read now” and **without** the Phase 5 poll while WS is connected.
- Trigger a read (or command) → UI updates **without** full page refresh **and** without relying solely on the Phase 11 poll interval if WS is connected.
- Disconnecting the server shows a disconnected indicator (even a simple badge).
- A device with tracking off does not produce live `reading.created` frames.

## Step 3a — Device HTTP

- `GET /api/devices/{device_id}/sampling` returns `{ sampling_interval_seconds, tracking_enabled }` from the device columns. Missing device → 404.
- `POST /api/devices/{device_id}/readings` body `{ value, unit }` calls the Phase 5 translator with `source` `http`, then `ReadingIngest`. Unknown device: log and drop (404). Do not publish `reading.created` in the route, and do not open a device WebSocket.
- The ESP32 applies the interval from GET and posts on its own clock. The backend does not poll the device. Sampling PATCH for `protocol=http` only updates those columns.
- The simulation sampler still skips devices whose protocol is not `simulation`, including `http` and `mqtt`.

Acceptance criteria:

- GET then POST of one payload inserts a `sensor_readings` row with `source` `http` and, when tracking is on, a `reading.created` frame.
- Tracking off stores the row and does not publish.
- The course demo does not require this call.

## Step 3b — MQTT subscriber

- Add Mosquitto (or equivalent) as a Compose **profile**. A normal `docker compose up` does not start it. Start the subscriber only when a broker URL is set. The course demo must still run if you never publish to it and never start the profile.
- Subscribe to `greenhouse/devices/{device_id}/reading`. Resolve the device, run the same Phase 5 translator with `source` `mqtt`, then `ReadingIngest`. Unknown device: log and drop. Do not insert from the subscriber with a second code path.
- When `PATCH /api/devices/{id}/sampling` targets `protocol=mqtt` **and** a broker is configured, publish a **retained** message on `greenhouse/devices/{device_id}/config`:

```json
{ "sampling_interval_seconds": 30, "tracking_enabled": true }
```

Document that the ESP32 on the broker path should follow that retained config. An `http` device uses GET instead. The backend clock does not poll the controller. Simulation devices keep using `SimulationSampler` and do not need the broker or the device routes.

Acceptance criteria:

- Adapter test still translates a dict with no broker.
- README smoke test, with the broker profile running: publish one payload to the reading topic and see a `sensor_readings` row plus a `reading.created` frame when tracking is on.
- Definition of done passes with only the seeded simulation sensor running, and with the broker profile stopped.

## Step 4 — Frontend path + indicator

- Centralize API base URL; fix any leftover `/dev` or inconsistent hosts.
- Header (or layout) shows WS connected / connecting / down.

Acceptance criteria:

- All Phase 2–11 features still work after a fresh migrate + seed.

## Step 5 — Documentation

- Short note in root README: dashboard WS URL, device HTTP smoke test (`GET .../sampling`, `POST .../readings`), optional MQTT topics (`.../reading`, retained `.../config`) and how to start the broker profile, and that simulation alone is enough for the course demo.
- Optional `docs/schema.md` listing migration chain — or a table in `docs/phases/README.md`.

Acceptance criteria:

- A reviewer can migrate empty DB and open dashboard against seed/demo data.

---

## Definition of done

- Production `/api` only for UI.
- Hardening migration applied; empty-DB migrate works.
- WS fan-out from the same EventBus.
- Sensor cards follow `reading.created` and pause the Phase 5 poll while connected.
- Device HTTP calls the Phase 5 translator then ingest. `source` is `http`. No device WebSocket.
- MQTT subscriber starts only when a broker URL is set; retained config is published for `protocol=mqtt` sampling updates only then.
- Idempotent seed includes a simulation sensor on a 5-second interval with tracking on.
- `realtime.ts` is no longer a no-op stub.
- Features still work after migrate + seed with no hardware.

---

## Common pitfalls to avoid

- Duplicating alert logic in the WS handler instead of subscribing to the bus.
- Parsing device HTTP or MQTT payloads in the route or subscriber instead of the Phase 5 translator, or publishing `reading.created` from either instead of ingest.
- Adding a device WebSocket, or accepting an ESP32 on the dashboard socket.
- Starting Mosquitto on every `docker compose up`, or failing the course demo when the broker URL is unset.
- Broadcasting readings for devices with tracking off.
- Breaking Scalar by disabling OpenAPI.
- Hand-editing old migration files that classmates already applied.
- Forgetting to update `api.ts` after renaming routes.
- Requiring a physical ESP32 for the course demo.

---

## Course completion (required track)

You have met the **required course** when the Definition of done above passes. No further phases are mandatory for credit.

Checklist:

- [ ] Production `/api` only for UI
- [ ] Hardening migration applied; empty-DB migrate works
- [ ] WS fan-out from the same EventBus
- [ ] Sensor cards live on `reading.created`; Phase 5 poll only as fallback
- [ ] Device HTTP smoke test; no device WebSocket
- [ ] Optional MQTT subscriber + retained config when the broker profile is up; simulation seed moves the dashboard with no hardware and no broker
- [ ] `realtime.ts` is no longer a no-op stub
- [ ] Features still work after migrate + seed

---

## Optional next steps

Want operator-grade polish or proof-and-demo rigor? These phases are **not required** for course completion:

- [Phase 13 — Dashboard polish (optional)](../phase-13/requirements.md) · [guided check](../phase-13/guided-check.md) — charts, empty/error states, UX cohesion
- [Phase 14 — Tests, docs, demo (optional)](../phase-14/requirements.md) · [guided check](../phase-14/guided-check.md) — test pyramid, pattern docs, demo narrative

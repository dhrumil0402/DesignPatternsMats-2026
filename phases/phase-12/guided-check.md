# Phase 12 — API & WebSocket hardening (Guided check)

Complete the [requirements](requirements.md) first. Use this document if you are stuck or to check signatures and JSON. Snippets are **not** a complete solution.

This phase **completes the required course track.** Phases 13–14 are optional.

## Pattern

**API + Observer (WS)** — one `/api` surface; WebSocket is another **subscriber**, not a second domain.

---

## Target layout

```text
interfaces/api/          # all HTTP routers under /api
interfaces/ws/
  hub.py                 # connection manager
  routes.py              # websocket endpoint
main.py                  # include routers + ws
```

---

## Step 1 — Route convergence

Inventory (adjust if you named paths slightly differently; keep one `/api` prefix):

| Area | Example paths |
|------|----------------|
| Health | `GET /health` (may stay unprefixed) |
| Locations | `GET/POST /api/locations`, `GET /api/locations/{id}` |
| Devices | `GET /api/devices`, `GET /api/devices/{id}` |
| Sensors | `POST /api/sensors/{id}/read`, `GET .../readings` |
| Device HTTP | `GET /api/devices/{id}/sampling`, `POST /api/devices/{id}/readings` |
| Automation | `PUT .../automation`, `POST .../evaluate` |
| Overview | `GET /api/locations/{id}/overview` |
| Commands | `POST /api/devices/{id}/commands`, `GET /api/commands` |
| Alerts | `GET /api/alerts` |
| Executions | `GET /api/actuators/executions` |
| WS | `WS /ws` (not under REST `/api` unless you document it) |

Remove leftover `/greenhouse` prefixes. Scalar lists the same contracts.

**Check:** browser/Scalar uses only `/api/...` for REST; no duplicate unprefixed resource routes.

---

## Step 2 — Schema hardening migration

- FKs and indexes from earlier phases actually applied (`alembic upgrade head`).
- Indexes that matter: `sensor_readings (device_id, recorded_at)`, `alerts (location_id, created_at)`, `command_log (created_at)`.
- Optional: drop or gate `/dev` seed behind `DEBUG` / env flag.

**Check:** `\d` (or equivalent) shows indexes; production-like env does not expose unrestricted seed if you gated it.

---

## Step 3 — WebSocket

```python
class ConnectionManager:
    async def connect(self, websocket: WebSocket) -> None: ...
    async def broadcast(self, message: dict) -> None: ...

# subscriber
def on_alert_created(event: AlertCreated) -> None:
    # schedule/broadcast JSON { "type": "alert.created", "payload": { ... } }
    ...
```

`WS /ws` (or `/ws/alerts`). Subscribe the hub in the **same** composition root as Phase 11 — do not open sockets inside use cases. Frontend: `useAlertsSocket` with reconnect.

Minimal payload:

```json
{ "type": "alert.created", "payload": { "id": "<uuid>", "severity": "warning", "message": "..." } }
```

Also broadcast `reading.created` (`device_id`, `value`, `unit`, `source`, `recorded_at`). Sensor cards apply that frame and stop the Phase 5 readings poll while the socket is connected; resume the poll when it is down.

**Check:** creating an alert (via a reading that fires the rule) pushes a message to an open WS client without the WS handler inserting into `alerts`. A tracked simulation sensor changes the card without “Read now”. Tracking off does not emit `reading.created`. This socket is not an ESP32 channel.

---

## Step 3a — Device HTTP

`GET /api/devices/{device_id}/sampling` returns the stored interval and tracking flag. `POST /api/devices/{device_id}/readings` body `{ "value": 0.41, "unit": "vwc" }` calls the Phase 5 translator with `source` `http`, then `ReadingIngest`. Unknown device: log and drop. Do not publish from the route. Do not add a device WebSocket.

Sampling PATCH for `protocol=http` only updates the columns GET returns. The device posts on its own clock. The backend does not poll it. The sampler still skips non-simulation protocols.

**Check:** GET then POST inserts a row with `source` `http` and, when tracking is on, a `reading.created` frame. Tracking off stores the row and does not publish. The course demo does not require this call.

---

## Step 3b — MQTT subscriber

Compose **profile**: Mosquitto. A normal `docker compose up` does not start it. Subscriber starts only when a broker URL is set. Topic `greenhouse/devices/{device_id}/reading` → the same translator with `source` `mqtt` → `ReadingIngest`. Unknown device: log and drop.

Sampling PATCH for `protocol=mqtt`, and only when a broker is configured, publishes a retained message on `greenhouse/devices/{device_id}/config`:

```json
{ "sampling_interval_seconds": 30, "tracking_enabled": true }
```

An `http` device uses GET instead of that retained message. Simulation devices keep using `SimulationSampler` and do not need the broker. Idempotent seed: one simulation sensor, interval `5`, `tracking_enabled` true.

**Check:** course demo runs with the broker profile stopped and with no published MQTT payload. README smoke test, with the profile up, publishes one reading and a row appears when tracking is on.

---

## Step 4 — Frontend path + indicator

- `VITE_API_BASE` (or equivalent) includes `/api` if the backend is mounted that way — one constant, no mixed prefixes.
- Connection badge: connected / retrying / offline (Tailwind).
- Optional: append pushed alerts to the panel; still allow GET as fallback.

**Check:** stop the backend → badge shows disconnected/retrying; start again → reconnects without a full page rewrite of alert logic.

---

## Step 5 — Documentation

README (or `docs/` note):

- REST lives under `/api`
- `WS /api/ws` (dashboard only; path and JSON `type` field, including `reading.created`)
- Device HTTP: `GET /api/devices/{device_id}/sampling` and `POST /api/devices/{device_id}/readings`
- Optional MQTT `greenhouse/devices/{device_id}/reading` and retained `.../config`, and how to start the broker profile
- How to run backend + frontend + `alembic upgrade head` without the broker
- Simulation seed (5-second interval) is enough; ESP32 and Mosquitto are optional
- Schema note: indexes / FKs that support live traffic

**Check:** a classmate can start the stack from README without asking for the WS path.

---

## Phase 12 completion checklist

- [ ] Consistent `/api` REST surface
- [ ] Migrations + useful indexes applied
- [ ] WS subscriber on the existing bus; sensor cards follow `reading.created`
- [ ] Device HTTP calls the translator then ingest; no device WebSocket
- [ ] MQTT subscriber calls ingest only when a broker URL is set; retained config for `protocol=mqtt`
- [ ] UI live indicator; Phase 5 poll only when the socket is down
- [ ] README documents REST + dashboard WS + device HTTP + optional MQTT; simulation seed needs no hardware and no broker

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Frontend 404 on resources | Base URL + `/api` prefix match `include_router` |
| WS handler writes AlertRow | Persist in Observer subscriber; WS only broadcasts |
| Use case imports WebSocket | Inject bus; hub is a subscriber |
| No message on new alert | Subscribe hub at startup; publish `AlertCreated` after insert |
| Card still polls while WS is up | Stop the Phase 5 readings poll on connect; resume on disconnect |
| Device route or MQTT subscriber inserts rows itself | Call the Phase 5 translator, then `ReadingIngest` |
| ESP32 connected on the dashboard socket | Device traffic is HTTP; `WS /api/ws` is the UI only |
| Demo fails with no broker | Subscriber starts only when a broker URL is set |
| Live frames while tracking is off | Ingest skips `reading.created` when `tracking_enabled` is false |

---

## Course complete (required track)

You have finished **Phases 1–12**. Optional: [Phase 13](../phase-13/requirements.md) (dashboard polish), [Phase 14](../phase-14/requirements.md) (tests, docs, demo).

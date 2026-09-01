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

**Check:** creating an alert (via a reading that fires the rule) pushes a message to an open WS client without the WS handler inserting into `alerts`.

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
- `WS /ws` (path and JSON `type` field)
- How to run backend + frontend + `alembic upgrade head`
- Schema note: indexes / FKs that support live traffic

**Check:** a classmate can start the stack from README without asking for the WS path.

---

## Phase 12 completion checklist

- [ ] Consistent `/api` REST surface
- [ ] Migrations + useful indexes applied
- [ ] WS subscriber on the existing bus
- [ ] UI live indicator
- [ ] README documents REST + WS

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Frontend 404 on resources | Base URL + `/api` prefix match `include_router` |
| WS handler writes AlertRow | Persist in Observer subscriber; WS only broadcasts |
| Use case imports WebSocket | Inject bus; hub is a subscriber |
| No message on new alert | Subscribe hub at startup; publish `AlertCreated` after insert |

---

## Course complete (required track)

You have finished **Phases 1–12**. Optional: [Phase 13](../phase-13/requirements.md) (dashboard polish), [Phase 14](../phase-14/requirements.md) (tests, docs, demo).

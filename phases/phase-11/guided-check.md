# Phase 11 — Observer (Guided check)

Complete the [requirements](requirements.md) first. Use this document if you are stuck or to check signatures and JSON. Snippets are **not** a complete solution.

## Pattern

**Observer** — `EventBus` + subscribers (in-process). WebSocket is Phase 12.

---

## Target layout

```text
domain/events/
  events.py             # ReadingTaken, CommandExecuted, ...
  bus.py                # EventBus
application/alerts/
  subscriber.py         # persist Alert from events
  dto.py
infrastructure/persistence/
  models.py             # AlertRow
  alert_repository.py
interfaces/api/alerts.py
frontend/.../AlertsPanel.tsx
```

---

## Step 1 — Database

```text
alerts(
  id, location_id FK, severity, message, created_at, acknowledged_at NULL
)
```

```powershell
alembic revision --autogenerate -m "alerts"
alembic upgrade head
```

**Check:** table exists; `location_id` FK is valid (not `greenhouse_id`).

---

## Step 2 — Bus and subscribers

```python
class EventBus:
    def subscribe(self, event_type: type, handler: Callable) -> None: ...
    def publish(self, event: object) -> None: ...

class AlertSubscriber:
    def on_reading(self, event: ReadingTaken) -> None: ...
    def on_command(self, event: CommandExecuted) -> None: ...
```

**Publish sites** (application services, not routers):

- After a persisted sensor read → `ReadingTaken`
- After command executed / failed → `CommandExecuted` (or equivalent)

**Composition root:** construct one `EventBus`, `subscribe` alert handlers, inject the **same** bus into reading and command services. Publishers must **not** import `AlertRepository`.

Example rule: moisture below zone `low` → `severity="warning"` alert. Command `failed` → alert. Keep rules in the subscriber.

**Check:** taking a reading that crosses the rule inserts an `alerts` row without the reading router calling the alerts repo.

---

## Step 3 — API and UI

`GET /api/alerts?location_id=`

```json
{
  "id": "<uuid>",
  "location_id": "<uuid>",
  "severity": "warning",
  "message": "Moisture below zone low",
  "created_at": "...",
  "acknowledged_at": null
}
```

Optional `POST /api/alerts/{id}/ack`. UI: poll every 5s **or** a clearly labeled stub (“live push in Phase 12”). Badge by severity. Tailwind.

**Check:** Scalar shows GET; triggering a read that should alert increases the list after poll/refresh.

---

## Step 4 — Tests and docs

- `test_low_moisture_publishes_alert`
- `test_publisher_does_not_import_alert_repository` (architecture / grep-style or unit with a fake bus)
- `docs/patterns/observer.md`: subject vs subscribers, vs Phase 12 WS

---

## Phase 11 completion checklist

- [ ] Bus + at least one subscriber
- [ ] Alerts persisted
- [ ] List API + UI (poll or stub)
- [ ] Publishers do not write alerts directly
- [ ] Tests + pattern doc

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Alerts only from the alerts router | Publish from application services |
| Reading service imports AlertRow | Inject EventBus; subscriber persists |
| No events after HTTP success | Call `bus.publish` after persist |
| Live WS in this phase | Poll or stub; sockets are Phase 12 |

---

## Next

→ [Phase 12 — API & WebSocket (requirements)](../phase-12/requirements.md) · [guided check](../phase-12/guided-check.md)

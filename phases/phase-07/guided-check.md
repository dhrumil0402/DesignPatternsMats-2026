# Phase 7 — Facade (Guided check)

Complete the [requirements](requirements.md) first. Use this document if you are stuck or to check signatures and JSON. Snippets are **not** a complete solution.

## Pattern

**Facade** — one entry point over subsystems.

---

## Target layout

```text
application/overview/
  dto.py                 # LocationOverviewDto
  facade.py              # LocationOverviewFacade
interfaces/api/overview.py
frontend/.../OverviewPanel.tsx   # mount in #overview on DashboardPage
```

---

## Step 1 — Overview DTO contract

```python
class LocationOverviewDto(BaseModel):  # or equivalent
    location: LocationSummaryDto          # id, name
    device_counts: DeviceCountsDto        # sensor: int, actuator: int (assigned only)
    zones: list[OverviewZoneDto]          # every zone, including an empty one
    strategy_key: str | None
    last_recommendation: RecommendationDto | None  # action, reason; null when no key
```

Each zone: `id`, `name`, `moisture_threshold_low`, `moisture_threshold_high`, `devices`, `latest_readings`.

Each device: `id`, `device_type`, `role`, `display_name`.

Each reading: `device_id`, `value`, `unit`, `recorded_at`, `tracking_enabled`, `sampling_interval_seconds`. Those last two come from the device. Include a reading only when that device has a `sensor_readings` row. Do not put `latest_readings` on the root DTO.

JSON field names must stay stable and match Scalar.

**Check:** missing location → 404. No saved strategy → 200 with `strategy_key` and `last_recommendation` null.

---

## Step 2 — Facade implementation

```python
class LocationOverviewFacade:
    def get_overview(self, location_id: UUID) -> LocationOverviewDto:
        # location + every zone
        # assigned devices only (this location_id and a zone_id)
        # one newest sensor_readings row per those devices (any source)
        # saved strategy_key
        # if a key is saved: last_recommendation from the automation evaluate service
        # if no key is saved: leave both strategy fields null; do not call decide()
        ...
```

Router injects **only** the facade — not `DeviceRepository`, `ReadingRepository`, and `AutomationService` together. The response model and a not-found mapping may live on the route.

Watch N+1: one assigned-device query and one newest-reading query for those ids — not a query per device in a Python loop. `evaluate` may still load moisture once per zone; leave that inside the automation service.

**Check:** after a new reading or strategy change, the next GET reflects DB state. The overview UI does not merge other API responses to build the snapshot.

---

## Step 3 — Database

**No migration required.** Optional view `location_overview_v1` (prefer location terminology). If you skip the view, the pattern note says existing tables are enough.

**Check:** Phase 6 schema unchanged unless you added an optional view revision.

---

## Step 4 — API and frontend

`GET /api/locations/{location_id}/overview`

Example response when a strategy key is saved:

```json
{
  "location": { "id": "<uuid>", "name": "Lab Site A" },
  "device_counts": { "sensor": 1, "actuator": 0 },
  "zones": [
    {
      "id": "<uuid>",
      "name": "Bench 1",
      "moisture_threshold_low": 0.2,
      "moisture_threshold_high": 0.45,
      "devices": [
        {
          "id": "<uuid>",
          "device_type": "moisture_sensor",
          "role": "sensor",
          "display_name": "Bench moisture"
        }
      ],
      "latest_readings": [
        {
          "device_id": "<uuid>",
          "value": 0.31,
          "unit": "vwc",
          "recorded_at": "...",
          "tracking_enabled": true,
          "sampling_interval_seconds": 300
        }
      ]
    },
    {
      "id": "<uuid>",
      "name": "Bench 2",
      "moisture_threshold_low": 0.3,
      "moisture_threshold_high": 0.6,
      "devices": [],
      "latest_readings": []
    }
  ],
  "strategy_key": "conservative",
  "last_recommendation": { "action": "wait", "reason": "..." }
}
```

When no key is saved, the same shape returns HTTP 200 with `strategy_key` and `last_recommendation` set to `null`.

Rows in `latest_readings` may come from the sampler or MQTT ingest, not only from `POST /read`. A device with no row stays in `devices` and is omitted from `latest_readings`. Unassigned devices do not appear.

`#overview`: the location dropdown may use the existing locations list. Cards and Refresh use one `fetchOverview(locationId)` — counts, strategy, recommendation, and one card per zone (thresholds, devices, reading or a “no reading” state).

**Check:** Scalar shows the schema. Selecting a location fetches overview once; Refresh repeats that call.

---

## Step 5 — Tests and docs

- `test_overview_returns_strategy_and_readings` — seeded location, two zones, one assigned sensor with a reading, at least one unassigned device. Assert 404, null strategy before save, assigned counts, unassigned ids absent, reading fields, an empty zone, then `last_recommendation` matching evaluate after a key is saved.
- `docs/patterns/facade.md`: why one entry point, what it hides, computed-not-cached recommendation, N+1 note (one newest-reading query; per-zone moisture stays in evaluate), no new table.

---

## Phase 7 completion checklist

- [ ] Single overview endpoint + UI
- [ ] Handler uses only the facade
- [ ] No required new tables
- [ ] Tests + pattern doc

---

## Troubleshooting

| Symptom                        | Fix                                         |
| ------------------------------ | ------------------------------------------- |
| Router lists five Depends(...) | Inject the facade only                      |
| Overview stale after a read    | Facade must query DB (or evaluate) each GET |
| Many queries per device        | Join / window for latest readings           |
| Scoped by greenhouse id        | Use `location_id`                           |

---

## Next

→ [Phase 8 — State (requirements)](../phase-08/requirements.md) · [guided check](../phase-08/guided-check.md)

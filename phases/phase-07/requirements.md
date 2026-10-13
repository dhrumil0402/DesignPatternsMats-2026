# Phase 7 — Facade (Requirements)

This is the **assignment**. Implement from these requirements. If you get stuck or want to check your design, use the [guided check](guided-check.md) (signatures and partial snippets — not a complete solution).

After you implement, answer the [questions](./questions.md).

## Purpose

Provide a **single read API** for the dashboard that aggregates existing tables through repositories. **No new business schema** — compose what Phases 2–6 already persist.

## Scope and naming rules

- Facade is the only collaborator the overview HTTP handler should call (not five repositories in the router). A response model and a not-found mapping are fine. Do not import reading, device, location, or automation repositories in the handler.
- Overview is scoped by **`location_id`**. There is no `greenhouse_id`.
- Include **every zone** on that location, including a zone with no devices.
- A device is assigned only when it has this `location_id` and a non-null `zone_id`. Unassigned devices are absent from `device_counts`, from `zones[].devices`, and from `zones[].latest_readings`.
- `device_counts` splits those assigned devices by `role` into `sensor` and `actuator`. Family counts are optional and not required.
- Each zone lists its assigned devices and, separately, the newest reading for each of those devices that has one. `tracking_enabled` and `sampling_interval_seconds` on a reading summary come from the device.
- Latest readings may come from a manual read, the Phase 5 sampler, or MQTT ingest. Do not filter them to `POST /read`. Do not add a scheduler, MQTT client, a second assignment API, or the device readings route in this phase.
- Optional SQL view is allowed; not required.

## Outcome required at end of phase

- `GET` overview for a location returns one DTO: location, device counts of assigned devices, zones (devices and latest readings), saved `strategy_key`, and `last_recommendation`.
- `last_recommendation` is computed on that request by the existing automation evaluate service when a key is saved. No saved key is still HTTP 200 with `strategy_key` and `last_recommendation` null. Do not store the recommendation in a new table.
- Dashboard **Overview** section (`#overview`) renders cards from that DTO. A location dropdown may use the existing locations list. Selecting a location performs one overview fetch; Refresh repeats that same fetch.
- Scalar documents the overview schema.
- Pattern note at `docs/patterns/facade.md`.

---

## Prerequisites

- Phase 6: automation rules + readings in DB (any Phase 5 ingress).
- Devices, locations, zones exist.

---

## Required architecture for this phase

- **Domain / application:** facade class `get_overview(location_id)` assembling a read model. It may call existing repositories and the automation service. It must not duplicate Strategy `decide` or Adapter translation.
- **Infrastructure:** queries in the repository layer. Prefer one assigned-device query and one newest-reading query for those device ids, not a reading query per device in a Python loop.
- **API:** one overview route. The handler calls **only** the facade.
- **Frontend:** `#overview` cards from that one overview response. Do not merge device, reading, and automation calls in React.

---

## Suggested file locations

Paths are relative to **`yourpath\project\`**.

```text
yourpath\project\
├── backend/src/
│   ├── application/overview/dto.py       # LocationOverviewDto
│   ├── application/overview/facade.py    # LocationOverviewFacade
│   └── interfaces/api/overview.py
└── frontend/src/components/overview/OverviewPanel.tsx   # mount in #overview
```

No new table required.

## Type hints

`LocationOverviewDto` (read model):

| Field                     | Hint                                                                                                                                                                                 |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `location`                | `{ id: UUID, name: str }`                                                                                                                                                            |
| `device_counts`           | `{ sensor: int, actuator: int }` of **assigned** devices only (family counts optional, not required)                                                                                 |
| `zones`                   | Every zone on the location, including an empty one. Each: `{ id, name, moisture_threshold_low, moisture_threshold_high, devices, latest_readings }`                                  |
| `zones[].devices`         | Assigned devices in that zone: `{ id, device_type, role, display_name }`                                                                                                             |
| `zones[].latest_readings` | Newest `sensor_readings` row per device in that zone that has one: `{ device_id, value, unit, recorded_at, tracking_enabled, sampling_interval_seconds }`. Omit devices with no row. |
| `strategy_key`            | Saved automation key, or `str \| None`                                                                                                                                               |
| `last_recommendation`     | `{ action: str, reason: str } \| None` — present only when a key is saved                                                                                                            |

Path param: `location_id: UUID`.

`recorded_at` orders “newest.” Do not filter readings by `source`. Copy `tracking_enabled` and `sampling_interval_seconds` from the device onto each reading summary; do not add those columns to `sensor_readings`.

---

## Step-by-step implementation requirements

## Step 1 — Overview DTO contract

Define a read DTO that includes:

- Location id and name.
- `device_counts.sensor` and `device_counts.actuator` for devices assigned to zones in this location (`location_id` set and `zone_id` not null). Do not count unassigned devices.
- A `zones` list for **every** zone on the location. Each zone has id, name, both moisture thresholds, its assigned devices, and `latest_readings` for those devices. A zone with no devices has empty `devices` and empty `latest_readings`.
- A device with no `sensor_readings` row stays in that zone’s `devices` and is left out of `latest_readings`.
- `latest_readings` is one newest row per assigned device (`recorded_at`). A manual read, the sampler, or MQTT ingest all qualify.
- Active `strategy_key`, or null when none is saved.
- `last_recommendation` only when a key is saved. Compute it by calling the existing automation evaluate service, which already calls `decide`. Do not reimplement the policy, do not call the evaluate HTTP route from the facade, and do not persist the recommendation. No saved key is HTTP 200 with both `strategy_key` and `last_recommendation` null.

Acceptance criteria:

- JSON field names match the table above and appear in Scalar.
- Missing location → 404.
- No saved strategy → 200 with both strategy fields null.

## Step 2 — Facade implementation

Implement `get_overview(location_id)` in the application layer. It coordinates existing subsystems. It is not a “god” ORM model.

Prefer this query shape:

- One load of the location and its zones.
- One query for devices assigned to that location.
- One query for the newest reading of those device ids.
- One read of the saved strategy key.
- Call evaluate only when a key exists.

Do not query readings once per device in a Python loop. `evaluate` may still load moisture once per zone. Leave that lookup in the strategy service, and mention it in the pattern note instead of reimplementing `decide` to avoid it.

Requirements:

- The router does not import reading, strategy, device, or location repositories.
- The handler’s only collaborator is the facade.

Acceptance criteria:

- After a new reading is stored, or after the strategy key changes, the next overview GET reflects the database.
- The overview UI does not stitch device, reading, and automation responses together to build that snapshot.

## Step 3 — Database

**No migration required.** Optional view `location_overview_v1` (prefer location terminology; avoid `greenhouse_overview_v1`) if you want to teach views.

Acceptance criteria:

- Phase 6 schema unchanged unless you add an optional view migration.
- If you skip the view, the pattern note says existing tables are enough.

## Step 4 — API and frontend

- `GET /api/locations/{location_id}/overview` with the overview DTO as `response_model`, so Scalar shows the schema.
- Map a missing location to 404. Do not turn “no strategy saved” into 404.
- Replace the `#overview` placeholder with cards bound to the DTO.
- A location dropdown may call the existing locations list so the operator can pick a `location_id`. Selecting a location performs **one** overview fetch. Refresh repeats that same fetch.
- Cards from that response:
  - Assigned sensor and actuator counts.
  - Strategy key, or a clear empty state when it is null.
  - Recommendation action and reason, or a clear empty state when it is null.
  - One card per zone: name, moisture thresholds, and each assigned device’s display name, role, and type. Show value, unit, tracking, and sampling interval when a reading exists; otherwise a clear “no reading” state.

Acceptance criteria:

- Overview works end-to-end; Scalar shows the schema.
- The panel does not call device, reading, and automation endpoints to assemble the cards.

## Step 5 — Tests and docs

Integration test `test_overview_returns_strategy_and_readings` against a seeded database. Use a location with two zones, one assigned sensor that has a reading, and at least one unassigned device. Assert:

- Unknown location id → 404.
- Location name matches the seed.
- Before a key is saved, `strategy_key` and `last_recommendation` are null and the status is 200.
- `device_counts` counts assigned devices only.
- Unassigned device ids are absent from every zone’s `devices`.
- The assigned sensor’s reading includes value, unit, `tracking_enabled`, and `sampling_interval_seconds`.
- The zone with no assigned devices has empty `devices` and empty `latest_readings`.
- After saving a known strategy key, `last_recommendation.action` and `last_recommendation.reason` match evaluate for that same location.

`docs/patterns/facade.md` covers:

- Why one entry point, and which subsystems it hides (locations, assigned devices, readings, saved strategy).
- That `last_recommendation` is computed by evaluate on each GET, not cached in a new table.
- The query shape from Step 2, including that evaluate may still look up moisture once per zone.
- That no new table was required. If you skipped the optional view, say existing tables are enough.

---

## Definition of done

- Single overview endpoint + UI.
- Handler uses only the facade.
- Tests and pattern doc complete.
- No required new tables.

---

## Common pitfalls to avoid

- Re-implementing strategy inside the facade instead of calling the automation evaluate service.
- A flat `latest_readings` list on the root DTO that drops which zone a device belongs to.
- Counting unassigned devices, or omitting a zone that has no devices.
- Returning 404 when no strategy is saved.
- Five queries per nested loop (a reading query per device) without a comment or a fix in the repository.
- Overview scoped by something other than `location_id`.
- The overview panel calling readings and automation itself and merging those responses in React.

---

## Handoff to next phases

- Phase 8 (State) will add actuator lifecycle fields that overview can later include.

→ [Phase 8 — State (requirements)](../phase-08/requirements.md) · [guided check](../phase-08/guided-check.md)

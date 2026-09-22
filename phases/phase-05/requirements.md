# Phase 5 — Adapter (Requirements)

This is the **assignment**. Implement from these requirements. If you get stuck or want to check your design, use the [guided check](guided-check.md) (signatures and partial snippets — not a complete solution).

After you implement, answer the [questions](./questions.md).

## Purpose

Read sensors through **port** abstractions and **adapters**, then **store each reading** so history and later Strategy use real persisted data rather than one-off in-memory values. Simulation and vendor-stub adapters are the course’s **mocked** device drivers—no lab hardware required.

Simulation devices also **generate values in code** on `sampling_interval_seconds` while tracking is on. An MQTT adapter **translates** an inbound payload into the same `Reading` so a later phase can accept it over device HTTP or from an optional broker. This phase does not run either transport.

## Scope and naming rules

- Adapters translate vendor, simulation, or MQTT payloads into a unified reading shape. Domain/application code talks to **ports**, not vendor SDKs or a broker client.
- Persist every successful read in `sensor_readings` keyed by `device_id`. One ingest path writes that table: manual read, the simulation sampler, and (from Phase 12) device HTTP or the optional MQTT subscriber.
- `sampling_interval_seconds` and `tracking_enabled` live on `devices`. The sampler uses them. Phase 2 may already store `sampling_interval_seconds` inside `default_config`; this phase promotes it to a column and backfills from that JSON.
- `default_config.protocol` selects the sensor adapter: `simulation` or `mqtt`. Family `simulation` stays the in-process generator. Family `edge` with `protocol: mqtt` is the real-device kit. The vendor stub remains a translation exercise; it is not the ESP32 path.
- When `tracking_enabled` is false, do not sample that device and do not publish a live reading event (Phase 11). A one-shot `POST /read` may still store a row.
- Keep `location_id` as the site scope from Phase 4; do not introduce `greenhouse_id`. A device’s zone is `devices.zone_id`. Sensor cards may show that zone’s name. Do not add another assignment API. The sampler still selects by `protocol` and `tracking_enabled`, not by zone.
- Domain ports must not depend on FastAPI, SQLAlchemy, or Pydantic. Do not open a broker socket in the domain.

## Outcome required at end of phase

- At least two adapters implement the same sensor port (simulation plus a stub “vendor”). An MQTT sensor adapter translates a payload dict into the same `Reading` (no broker in this phase).
- `ActuatorPort` and a **simulation actuator adapter** exist (in-memory apply or log-only—no GPIO); Phase 9 Decorator wraps this port.
- `devices.sampling_interval_seconds` and `devices.tracking_enabled` exist. `PATCH /api/devices/{id}/sampling` updates them.
- `POST` read on a sensor triggers an adapter, inserts a reading, and returns a normalized DTO.
- A simulation sampler records readings for enabled `protocol: simulation` sensors when their interval has elapsed. MQTT devices and disabled devices are skipped.
- Sensor cards can “Read now”, edit interval and tracking, and show value + source. Until Phase 12, cards poll the latest stored reading. History survives refresh.
- Scalar documents the read/history and sampling endpoints.
- Pattern note exists at `docs/patterns/adapter.md`.

---

## Prerequisites

Before starting, all of the following must be true:

- Phase 4 completed: `locations` / `zones` with `location_id`.
- Phase 2–3: sensor `device_id`s exist in `devices`.
- Alembic is at Phase 4 head.

If any prerequisite is missing, fix it first.

---

## Required architecture for this phase

- **Domain layer**
  - `SensorPort` and `ActuatorPort` defining unified read/apply operations (actuator: simulation adapter only—no real GPIO).
  - Normalized reading value object (value, unit, source, timestamp).
  - No HTTP or ORM types.

- **Application layer**
  - `ReadingIngest.record` is the only writer of `sensor_readings`: load device → select adapter or accept an already translated reading → persist → map to DTO.
  - `SimulationSampler.run_once(now)` loads simulation sensors with tracking on and records when the interval has elapsed.
  - Reading and sampling DTOs.

- **Infrastructure layer**
  - Concrete sensor adapters (simulation; vendor stub with a different raw shape; MQTT payload translator).
  - `SimulationActuatorAdapter` implementing `ActuatorPort` (stub apply for Phases 9–10).
  - ORM for `sensor_readings`, sampling columns on `devices`, and a readings repository.
  - Adapter selection by `default_config.protocol` (`simulation` or `mqtt`), with the vendor stub selected by its own documented flag. Document the rule.

- **API layer**
  - Trigger read and list recent readings under `/api/sensors/...`.
  - `PATCH /api/devices/{id}/sampling`.
  - Missing device → 404; adapter/domain errors and an interval below 5 seconds → 400-level with detail.

- **Frontend**
  - Read action, sampling interval, and tracking toggle on sensor cards; value + source badge.
  - Temporary poll of the latest reading (a few seconds) until Phase 12 WebSocket replaces it. Say in the UI copy or a code comment that the poll is temporary.

---

## Suggested file locations

Paths are relative to **`yourpath\project\`**.

```text
yourpath\project\
├── backend/src/
│   ├── domain/sensors/ports.py              # SensorPort
│   ├── domain/actuators/ports.py            # ActuatorPort
│   ├── domain/sensors/reading.py            # Reading
│   ├── application/readings/dto.py
│   ├── application/readings/service.py      # ReadingIngest
│   ├── application/readings/sampler.py      # SimulationSampler
│   ├── infrastructure/adapters/sensors/       # simulation.py, vendor_stub.py, mqtt.py
│   ├── infrastructure/adapters/actuators/simulation.py  # SimulationActuatorAdapter
│   ├── infrastructure/persistence/models.py # ReadingRow + sampling columns
│   ├── infrastructure/persistence/reading_repository.py
│   └── interfaces/api/sensors.py            # + read/history + sampling
└── frontend/src/features/sensors/           # Read now, interval, tracking
```

## Type hints

Domain `Reading`:

| Field | Hint |
|-------|------|
| `device_id` | `UUID` |
| `value` | `float` |
| `unit` | `str` (e.g. `vwc`, `lux`) |
| `source` | `str` — `"simulation"`, `"mqtt"`, or `"vendor"` (vendor stub only) |
| `recorded_at` | timezone-aware `datetime` |

Device sampling columns:

| Column | Hint |
|--------|------|
| `sampling_interval_seconds` | `int`, not null, server default `300`, minimum `5` |
| `tracking_enabled` | `bool`, not null, server default `true` |

PATCH body: `{ sampling_interval_seconds: int, tracking_enabled: bool }`.

ORM `sensor_readings`:

| Column | SQL hint |
|--------|----------|
| `id` | UUID PK |
| `device_id` | UUID FK → `devices` |
| `value` | NUMERIC |
| `unit` | VARCHAR |
| `source` | VARCHAR |
| `recorded_at` | timestamptz |
| Index | `(device_id, recorded_at DESC)` |

API `ReadingDto` JSON: `{ device_id, value: number, unit, source, recorded_at: ISO-8601 string }`.

---

## Step-by-step implementation requirements

## Step 1 — Database model

`sensor_readings` table:

| Column | Intent |
|--------|--------|
| Identifier | Primary key |
| `device_id` | FK → `devices` |
| `value` | Numeric reading |
| `unit` | Unit string |
| `source` | `simulation`, `mqtt`, or `vendor` |
| `recorded_at` | Timestamptz |

On `devices` in the same revision:

| Column | Intent |
|--------|--------|
| `sampling_interval_seconds` | Integer, not null, server default `300` |
| `tracking_enabled` | Boolean, not null, server default `true` |

Backfill `sampling_interval_seconds` from `default_config` when that JSON already has a numeric `sampling_interval_seconds`; otherwise leave the server default. After this phase the columns are the source of truth (creators may still copy the same numbers into `default_config`).

Index `(device_id, recorded_at)` (descending if your dialect supports it) for latest-row and later charts.

Acceptance criteria:

- FK from readings to devices exists.
- Index supports “latest reading per device”.
- Existing devices have a non-null interval and tracking flag.

## Step 2 — Generate migration via Alembic

Autogenerate from the ORM model (message such as `sensor_readings`). Review create table + index + FK and the two new `devices` columns. Add the JSON backfill in the reviewed revision if autogenerate did not emit it. Apply. Do not edit applied revisions.

Acceptance criteria:

- `alembic current` is the new revision.
- Table inspection shows `sensor_readings` and the sampling columns on `devices`.

## Step 3 — Ports and adapters

Define a sensor port with a read operation that returns a normalized reading.

Implement:

- **Simulation adapter** — generates a plausible value in code from `device_type`. Use ranges such as moisture `0.2`–`0.6` `vwc` and light `200`–`2000` `lux`. `source` is `simulation`.
- **Vendor stub adapter** — accepts a different raw payload or internal structure and **translates** it to the same normalized type. Translation is the point of Adapter. `source` is `vendor`. This is not the ESP32 path.
- **MQTT sensor adapter** — translates an inbound dict such as `{ "value": 0.41, "unit": "vwc" }` into `Reading` with `source` `mqtt`. Tests pass the dict. Do not connect to a broker here.
- **Actuator port + simulation actuator adapter** — define `ActuatorPort.apply(device_id, command, payload)` and a `SimulationActuatorAdapter` that records intent in memory or logs only (no GPIO). Phase 9 will wrap this port with decorators.

Selector: `default_config.protocol` of `simulation` or `mqtt`. Document how the vendor stub is chosen so it does not collide with those two.

Acceptance criteria:

- Application code depends on the port, not on adapter classes (except a factory/selector).
- Simulation and vendor adapters can yield different `source` values for the same logical read.
- Simulation values fall inside the documented range for that device type.
- MQTT translation does not require a running broker.
- `ActuatorPort` and `SimulationActuatorAdapter` compile and can be called from a test or thin demo route (full command flow comes in Phases 9–10).

## Step 4 — Persistence and application service

Repository inserts a reading and lists recent readings for a device (limit / latest).

`ReadingIngest`:

- Load device; 404-equivalent if missing.
- For a one-shot read: select adapter, call the port, persist, return DTO.
- For an already translated reading (MQTT adapter output): persist that reading. Phase 12’s subscriber will call this; this phase only needs the method and a test.

`SimulationSampler`, started from the application lifespan:

- Each tick loads sensors with `protocol` `simulation` and `tracking_enabled` true.
- Record when `now - last recorded_at` is at least that device’s `sampling_interval_seconds` (no previous row counts as elapsed).
- Skip MQTT devices and devices with tracking off.
- Tests call `run_once(now)` with a fake clock. Do not sleep in tests.

Acceptance criteria:

- Each successful API read appends a row (history must exist).
- DTO includes value, unit, source, recorded_at, device_id.
- A second `run_once` inside the interval does not insert another row; a call after the interval does.
- Disabled and MQTT devices gain no sampler rows.

## Step 5 — API endpoints

- Trigger read: `POST /api/sensors/{id}/read`.
- `GET /api/sensors/{id}/readings?limit=`.
- `PATCH /api/devices/{id}/sampling` body `{ sampling_interval_seconds, tracking_enabled }`.

HTTP: 200/201 on success; 404 missing device; 400 adapter/validation failures and when `sampling_interval_seconds` is below `5`.

Acceptance criteria:

- Scalar documents the endpoints and reading schema.
- Repeated reads increase row count.
- PATCH persists interval and tracking; a too-small interval returns 400.

## Step 6 — Frontend

On sensor cards (Phase 2 list or equivalent):

- “Read now” calls the read endpoint.
- Show latest value and a **source** badge (`simulation`, `mqtt`, or `vendor`).
- Controls to set `sampling_interval_seconds` and turn `tracking_enabled` off or on for that device (calls PATCH).
- Poll `GET /api/sensors/{id}/readings?limit=1` every few seconds so sampler rows show up without a manual read. Comment in code that Phase 12 replaces this poll with WebSocket.
- Tailwind conventions; loading and error states.

Acceptance criteria:

- Refresh still shows the last stored value (from DB, not only React state).
- Turning tracking off stops new sampler rows for that device; turning it on resumes them after the interval.
- Changing the interval changes how often new sampler rows appear.

## Step 7 — Tests

- Adapter unit tests: vendor/raw shape maps to normalized fields.
- Simulation adapter stays inside the documented range and sets `source` to `simulation`.
- MQTT adapter maps `{ value, unit }` to a `Reading` with `source` `mqtt` and does not open a socket.
- Sampler: `run_once` inserts when the interval has elapsed, skips a second call inside the interval, and skips disabled and MQTT devices.
- Integration: read inserts a `sensor_readings` row.

Acceptance criteria:

- Translation and sampler tests do not require HTTP or a broker.

## Step 8 — Documentation

`docs/patterns/adapter.md`: problem (incompatible APIs), solution (port + adapters), where in code, extension (third vendor).

---

## Definition of done

Phase 5 is done when all items below are true:

- `sensor_readings` exists with FK and index.
- `sampling_interval_seconds` and `tracking_enabled` exist on `devices` and are backfilled.
- At least two adapters implement the sensor port, plus an MQTT translator tested with a dict.
- `ActuatorPort` and `SimulationActuatorAdapter` exist (stub apply; no GPIO).
- Read API persists and returns a normalized DTO.
- Sampler records enabled simulation devices on their interval.
- UI can trigger a read, edit sampling, and show value + source from persisted data (poll until Phase 12).
- Tests cover translation, generation range, sampler clock, and insert.
- Pattern doc exists.

---

## Common pitfalls to avoid

- Calling vendor-shaped types from the API router.
- Storing readings only in memory.
- Skipping the index needed for “latest” queries.
- One adapter class with `if source ==` sprawl instead of separate adapter types.
- Opening a broker connection inside the domain or in this phase’s sampler.
- Treating `default_config.sampling_interval_seconds` as authoritative after the columns exist.
- Broadcasting or sampling devices whose `tracking_enabled` is false.

---

## Handoff to next phases

- Phase 6 (Strategy) evaluates each zone with that zone’s thresholds and the latest moisture among sensors whose `zone_id` matches. Those rows may come from a manual read, the simulation sampler, or (later) MQTT ingest. Do not use a global latest reading from an unassigned device. Do not add another sampler in Phase 6.
- Phase 9 (Decorator) wraps the **Phase 5 `ActuatorPort`**—do not skip the simulation actuator adapter.
- Phase 11 (Observer) will publish `reading.created` from this ingest path when tracking is on.
- Phase 12 may deliver this same payload dict on the device HTTP route or from the optional broker, and it pushes `reading.created` over the dashboard WebSocket, replacing the card poll. Do not add that route or a broker client here.
- Optional Phase 13 charts query this table.

### Beyond the course (other hardware)

ESP32-style controllers are in this track (translation now). Phase 12 adds device HTTP, and an optional broker. The backend stores `sampling_interval_seconds`. An `http` device reads it with GET. An `mqtt` device gets a retained config message when a broker is configured. The backend does not poll the device. The controller honors the interval.

Other buses stay optional later. Keep application services unchanged:

1. Add new adapter classes implementing `SensorPort` / `ActuatorPort` (e.g. GPIO, Modbus, DMX).
2. Extend the selector: map `device_family`, `protocol`, or env flag to the live adapter.
3. Optionally add a live `DeviceFamilyFactory`; keep `sensor_readings` schema as-is.
4. Do not move irrigation policy or state rules into adapters—translation and apply only.

→ [Phase 6 — Strategy (requirements)](../phase-06/requirements.md) · [guided check](../phase-06/guided-check.md)

# Phase 4 — Builder (Requirements)

This is the **assignment**. Implement from these requirements. If you get stuck or want to check your design, use the [guided check](guided-check.md) (signatures and partial snippets — not a complete solution).

After you implement, answer the [questions](./questions.md).

## Purpose

This phase introduces the **Builder** pattern for creating a valid **location configuration** and persists it into PostgreSQL.

## Scope and naming rules

- Use **location** terminology in data and APIs.
- Use **`location_id`** in zones and related records.
- A device belongs to at most one zone. `devices.zone_id` is that assignment. On assign, set `devices.location_id` from the zone’s `location_id` in the same write. On unassign, clear both. The client does not send a location that can disagree with the zone.
- Do **not** introduce `greenhouse_id` in this phase.
- Keep the product context as smart greenhouse; only the relational resource naming changes to location.
- `LocationConfigBuilder` only sets the location name, adds zones, and `build()` returns an unsaved config or raises `ConfigurationError`. Do not add `add_device`. Devices already exist from Phases 2–3, and a zone has no id until the config is saved. Assignment is a later update through its own service, not a rebuild of the location.

## Outcome required at end of phase

- A Builder workflow can create one location with one or more zones.
- Builder validation blocks invalid configurations before persistence.
- A persisted location configuration can be retrieved by location id.
- A user can assign an existing device to a zone in that location, list the devices in that zone, and clear the assignment. Unassigned devices stay valid.
- Dashboard has a configuration UI for creating location + zones, plus a zone picker on device cards.
- API reference (Scalar) documents the location config endpoints, zone assignment, and the zone device list.

---

## Prerequisites

Before starting, all of the following must be true:

- Phase 3 completed successfully.
- Devices and family provisioning endpoints are functional.
- Alembic is configured and the database is at Phase 3 head revision.
- Dashboard has a placeholder configuration section.

If any prerequisite is missing, fix it first; this phase assumes that foundation.

---

## Required architecture for this phase

Implement the phase with clean layer separation:

- **Domain layer**
  - Owns the Builder and validation rules.
  - Contains location/zone configuration entities.
  - Must not depend on FastAPI, SQLAlchemy, or Pydantic.

- **Application layer**
  - Defines request/response DTOs for location configuration.
  - Maps DTOs to Builder calls and maps persistence results back to DTOs.
  - Orchestrates save/retrieve of the built config through repositories.
  - A separate assignment service sets or clears `devices.zone_id` after the zone row exists. It does not call the builder.

- **Infrastructure layer**
  - Owns ORM models and repository persistence logic.
  - Persists location and zones transactionally.

- **API layer**
  - Exposes endpoints under `/api/locations/...`.
  - Uses DTOs as request/response contracts.
  - Converts domain/application errors to HTTP responses.

---

## Suggested file locations

Paths are relative to **`yourpath\project\`**.

```text
yourpath\project\
├── backend/src/
│   ├── domain/locations/entity.py            # Location, Zone, LocationConfig
│   ├── domain/locations/config_builder.py
│   ├── domain/locations/errors.py            # ConfigurationError
│   ├── application/locations/dto.py
│   ├── application/locations/mappers.py
│   ├── application/locations/config_service.py
│   ├── application/locations/zone_assignment_service.py
│   ├── infrastructure/persistence/models.py  # LocationRow, ZoneRow, devices.zone_id
│   ├── infrastructure/persistence/location_repository.py
│   └── interfaces/api/locations.py           # config + zone device list + assign
└── frontend/src/components/config/LocationConfigWizard.tsx
```

## Type hints

Domain:

| Type | Fields |
|------|--------|
| `Zone` | `name: str`, `moisture_threshold_low: float`, `moisture_threshold_high: float`, `schedule: dict`, `id: UUID \| None` |
| `Location` | `name: str`, `zones: tuple[Zone, ...]`, `id: UUID \| None` |
| `LocationConfig` | `location: Location` (builder output) |
| `ConfigurationError` | subclass of `ValueError` |

ORM:

| Table / column | SQL / mapping hint |
|----------------|-------------------|
| `locations.id` | UUID PK |
| `locations.name` | `String(128)`, not null |
| `locations.created_at` | timestamptz |
| `zones.id` | UUID PK |
| `zones.location_id` | UUID FK → `locations.id`, not null, index |
| `zones.name` | `String(128)` |
| `zones.moisture_threshold_low` / `_high` | `Numeric` (e.g. 5,4) stored as float in domain |
| `zones.schedule` | JSONB |
| `devices.zone_id` | nullable UUID FK → `zones.id`, `ON DELETE SET NULL`, indexed |
| `devices.location_id` | nullable UUID FK; written only from the zone’s `location_id` when `zone_id` is set |

API JSON:

| DTO | Fields |
|-----|--------|
| Request | `location_name: str`, `zones: [{ name, moisture_threshold_low: number, moisture_threshold_high: number, schedule?: object }]` |
| Response | `location: { id: UUID, name: str }`, `zones: [{ id, location_id, name, moisture_threshold_low, moisture_threshold_high, schedule }]` |

Thresholds in this product: `float` in **0.0–1.0** (VWC); low must be strictly less than high.

---

## Step-by-step implementation requirements

## Step 1 — Database model design

Define the relational model needed for location configuration:

- `locations`
  - Identifier, name, created timestamp.
- `zones`
  - Identifier, **`location_id`** foreign key to `locations`.
  - Zone name.
  - Moisture threshold low/high values.
  - Schedule payload (JSON).
- Device assignment columns on `devices`:
  - `zone_id` nullable FK → `zones.id`, `ON DELETE SET NULL`, indexed.
  - `location_id` nullable FK. Set it from the zone in the assignment write. Do not accept it as a separate client field.

Validation and constraints expectations:

- Each zone belongs to exactly one location.
- `zones.location_id` must reference an existing location.
- Index zones by `location_id` for reads.

Acceptance criteria:

- Schema design uses `location_id` consistently.
- No schema artifact contains `greenhouse_id`.

## Step 2 — Generate migration via Alembic

Use Alembic autogenerate from model changes:

- Generate a new revision with a clear message such as `locations_and_zones`.
- Review generated migration carefully:
  - `locations` table creation exists.
  - `zones` table creation exists.
  - FK is from `zones.location_id` to `locations.id`.
  - `devices.zone_id` and `devices.location_id` are nullable FKs. `zone_id` uses `ON DELETE SET NULL`.
- Apply migration to the local database.

Acceptance criteria:

- `alembic current` points to the new revision.
- Database inspection shows `zones.location_id` and `devices.zone_id`.
- Migration chain is forward-only; no edits to already-applied revisions.

## Step 3 — Builder domain behavior

Implement the Builder behavior in domain logic with these guarantees:

- Builder can set location name.
- Builder can add one or more zones.
- `build()` validates and either returns a complete configuration or raises a configuration error.

Required validation rules:

- Location name is required and non-empty.
- At least one zone is required.
- For each zone, low threshold must be strictly less than high threshold.
- Threshold ranges must be constrained to your selected domain limits (for example 0–1 for VWC).

Acceptance criteria:

- Invalid config is rejected before any database write.
- Valid config builds deterministically.
- The builder has no method that attaches a device. Assignment happens only after `zones.id` exists.

## Step 4 — DTO contracts

Define DTOs for this phase in application/API contracts:

- Request DTO for creating a location configuration:
  - Location name.
  - List of zones with thresholds and schedule.
- Response DTO for location configuration:
  - Location metadata.
  - Zone list including **`location_id`**.

Requirements:

- DTO names should clearly indicate intent (request/read models).
- DTO field names must match API JSON contract.
- Domain entities and DTOs remain separate.

Acceptance criteria:

- Endpoint contracts are typed and documented in Scalar.
- Zone response includes `location_id` in every row.

## Step 5 — Repository and transaction semantics

Implement persistence behavior with a transaction boundary:

- Save location first.
- Save all zones linked by `location_id`.
- Commit as one unit; failures rollback all writes.

Read behavior:

- Retrieve full config by location id including child zones.

Acceptance criteria:

- Partial writes do not occur on invalid or failed saves.
- Fetch returns consistent location + zones payload.

## Step 6 — API endpoints

Expose location configuration endpoints:

- Create config endpoint under `/api/locations/...`.
- Read config endpoint under `/api/locations/{location_id}/...`.

HTTP behavior requirements:

- Validation/configuration errors return 400-level responses with useful details.
- Missing location returns 404.
- Successful create returns created representation.

Acceptance criteria:

- Endpoints visible in Scalar.
- Endpoints return DTO schemas exactly as defined.

## Step 6b — Assign a device to a zone

After the zone row exists, a separate assignment service (not the builder) updates the device:

- `PATCH /api/devices/{id}/zone` body `{ "zone_id": "<uuid>" | null }`.
- When `zone_id` is set, load the zone and write `devices.zone_id` plus `devices.location_id` copied from `zones.location_id` in the same transaction.
- When `zone_id` is null, clear both columns.
- `GET /api/locations/{location_id}/zones/{zone_id}/devices` lists sensors and actuators with that `zone_id`. 404 if the zone is missing or does not belong to that location.
- 404 if the device or zone is missing. Do not accept a separate `location_id` in the body.

Acceptance criteria:

- Assigning two devices to zone 2 makes the zone list return only those devices.
- A device assigned to another zone is absent from that list.
- Unassign clears `zone_id` and `location_id`.
- Scalar documents both routes.
- The PATCH handler does not call `LocationConfigBuilder`.

## Step 7 — Frontend configuration wizard

Implement a dashboard config wizard in the configuration section:

- Collect location name.
- Collect one or more zones with thresholds and schedule.
- Submit to location config create endpoint.
- Display saved result (including returned location id and zone list).
- On device cards, a zone picker calls the assign endpoint and shows the current zone name. Empty state: the device is unassigned. The picker is available after a location config exists; it does not run inside `build()`.

UI requirements:

- Show inline validation messages for obvious user input issues.
- Show API error state and success state.
- Preserve Tailwind conventions established in earlier phases.

Acceptance criteria:

- User can create a location config end-to-end from UI.
- User can place two devices in one zone and see them on that zone’s list.
- UI reflects server validation failures clearly.

## Step 8 — Test requirements

Minimum test set:

- Builder unit tests:
  - Success path.
  - Missing location.
  - No zones.
  - Invalid threshold ordering.
- Integration/API tests:
  - Create config persists location + zones.
  - Get config returns expected structure with `location_id`.
  - Invalid payload handled with 400.
  - Assign two devices to one zone; the zone device list returns only those devices.
  - A device in another zone is absent from that list.
  - Unassign clears `zone_id` and `location_id`.
  - Builder tests do not call the assignment service.

Migration checks:

- CI or local check runs migrations from empty DB to head.

Acceptance criteria:

- Tests pass reliably.
- Migration from empty database succeeds.

## Step 9 — Documentation requirements

Update learning docs:

- Add/refresh `docs/patterns/builder.md` with:
  - Builder intent in this project.
  - Why this is Builder (not Factory Method / Abstract Factory).
  - Where validation lives.
  - Why `location_id` naming is used.
  - Why device assignment is not a builder method.

Update phase references where needed:

- Ensure adjacent phases refer to location/location_id consistently.

---

## Definition of done

Phase 4 is done when all items below are true:

- Database has `locations` and `zones` with FK `zones.location_id`.
- `devices.zone_id` exists. Assignment copies `location_id` from the zone and clears both on unassign.
- Builder enforces required validations before persistence and does not attach devices.
- Location config DTOs exist and are used by API.
- Endpoints for create/read location config, zone assignment, and the zone device list are live and documented in Scalar.
- UI wizard can create and display persisted config. Device cards can assign a zone.
- Automated tests cover builder validation, API persistence, and zone assignment.
- Documentation references `location_id` consistently.

---

## Common pitfalls to avoid

- Mixing domain entities and API DTOs directly.
- Letting builder validation happen only in API layer.
- Using `greenhouse_id` in any new schema or endpoint.
- Treating migration generation as optional manual SQL.
- Returning partially saved data on transactional failure.
- Adding `add_device` to `LocationConfigBuilder`, or rebuilding the location when a device moves.
- Storing a `location_id` that does not match the assigned zone.
- Treating an unassigned device as a member of every zone.

---

## Handoff to next phases

- Phase 5 (Adapter) stores readings on `device_id`. Sensor cards may show the zone name. Sampling does not filter by zone.
- Phase 6 (Strategy) evaluates each zone using that zone’s thresholds and only the sensors whose `zone_id` matches. Do not use a global latest reading from an unassigned device.
- Phase 7 overview lists zones with their assigned devices and those devices’ latest readings.
- Later alert flows copy `location_id` and `zone_id` from the device. Unassigned devices leave both null.

→ [Phase 5 — Adapter (requirements)](../phase-05/requirements.md) · [guided check](../phase-05/guided-check.md)


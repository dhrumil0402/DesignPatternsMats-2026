# Phase 4 — Builder (Guided check)

Complete the [requirements](requirements.md) first. Use this document if you are stuck or to check signatures, layout, and JSON. Snippets are **not** a complete solution.

**Time estimate:** 4–6 hours.

## Pattern

**Builder** — construct a complex `LocationConfig` incrementally; `build()` validates and returns an immutable result (or raises `ConfigurationError`).

**Naming:** `locations` / `location_id` — not `greenhouses` / `greenhouse_id`.

```text
LocationConfigBuilder → LocationConfig (domain)
       ↓ build()
LocationRepository → LocationRow + ZoneRow (ORM)
       ↓
LocationConfigDto (API)
```

---

## Target repository layout (end state)

```text
backend/src/
├── domain/locations/
│   ├── entity.py               # Location, Zone, LocationConfig
│   ├── config_builder.py
│   └── errors.py               # ConfigurationError
├── application/locations/
│   ├── dto.py
│   ├── mappers.py
│   ├── config_service.py
│   └── zone_assignment_service.py
├── infrastructure/persistence/
│   ├── models.py               # LocationRow, ZoneRow, devices.zone_id
│   └── location_repository.py
└── interfaces/api/locations.py # config, assign, zone device list
frontend/src/components/config/LocationConfigWizard.tsx
```

---

## Step 0 — Phase 3 baseline

```powershell
cd "yourpath\project\"
docker compose up --build -d
docker compose exec backend alembic current
```

Confirm `/api/devices` and dashboard families still work.

---

## Step 1–2 — ORM + migration

**`LocationRow`:** `id` (UUID), `name`, `created_at`; relationship to zones.

**`ZoneRow`:** `id`, **`location_id`** FK → `locations.id` (`ondelete="CASCADE"`), `name`, `moisture_threshold_low/high` (numeric), `schedule` (JSONB). Index `ix_zones_location_id`.

**`devices`:** nullable `zone_id` FK → `zones.id` (`ondelete="SET NULL"`, index) and nullable `location_id` (`ondelete="SET NULL"`). Assignment writes both; the client sends only `zone_id`. Deleting a zone still requires the service to clear `location_id` on those devices, because that column points at the location.

```powershell
docker compose exec backend alembic revision --autogenerate -m "locations_and_zones"
docker compose exec backend alembic upgrade head
```

**Check:** `\d zones` shows **`location_id`**, not `greenhouse_id`. `\d devices` shows nullable `zone_id`.

---

## Step 3 — Domain + Builder (signatures)

```python
class ConfigurationError(ValueError): ...

class LocationConfigBuilder:
    def with_location_name(self, name: str) -> "LocationConfigBuilder": ...
    def add_zone(
        self,
        name: str,
        moisture_threshold_low: float,
        moisture_threshold_high: float,
        schedule: dict | None = None,
    ) -> "LocationConfigBuilder": ...
    def build(self) -> LocationConfig:
        ...  # reject empty name, zero zones, low >= high, thresholds outside 0–1
```

**Check:** invalid configs never reach the repository. The builder has no `add_device`. Assignment is Step 6b, after the zone has an id.

---

## Step 4 — DTOs

Request: `location_name` + `zones[]` with `name`, `moisture_threshold_low/high`, `schedule`.

Response: `location: { id, name }` + `zones[]` each including **`location_id`**.

Domain entities stay separate from Pydantic models.

---

## Step 5 — Repository + service

```python
class LocationRepository:
    def save_config(self, config: LocationConfig) -> tuple[LocationRow, list[ZoneRow]]:
        ...  # flush location id, insert zones, one commit

    def get_config(self, location_id) -> tuple[LocationRow, list[ZoneRow]] | None:
        ...

class LocationConfigService:
    def build_and_save(self, request: BuildLocationConfigRequestDto) -> LocationConfigDto:
        builder = LocationConfigBuilder().with_location_name(request.location_name)
        for z in request.zones:
            builder.add_zone(...)
        config = builder.build()
        loc_row, zone_rows = self._repo.save_config(config)
        return location_config_to_dto(...)
```

**Check:** failed save does not leave a location without zones.

---

## Step 6 — REST

Prefix `/api/locations`, tags `["locations"]`.

| Method | Path | Status |
|--------|------|--------|
| POST | `/config` | 201 |
| GET | `""` | 200, `[{ id, name }]`, empty `[]` |
| GET | `/{location_id}/config` | 200 / 404 |
| DELETE | `/{location_id}` | 204 / 404 |
| POST | `/{location_id}/zones` | 201 / 400 / 404 |
| PATCH | `/{location_id}/zones/{zone_id}` | 200 / 400 / 404 |
| DELETE | `/{location_id}/zones/{zone_id}` | 204 / 400 last zone / 404 |

Validation errors → 400.

Example create body:

```json
{
  "location_name": "Lab Site A",
  "zones": [
    {
      "name": "Bench 1",
      "moisture_threshold_low": 0.2,
      "moisture_threshold_high": 0.45,
      "schedule": { "watering": "08:00" }
    }
  ]
}
```

**Check:** Scalar **locations** tag; every zone in the response has `location_id`. List returns every created location. Delete removes zones with the location and leaves assigned devices with both ids null. These handlers do not call `LocationConfigBuilder`.

Zone add/update bodies use `name`, `moisture_threshold_low`, `moisture_threshold_high`, and optional `schedule`. Same rules as `build()`. Delete of the only zone returns 400. Delete of a zone clears `zone_id` and `location_id` on devices that were in it before the row is removed (`ON DELETE SET NULL` clears `zone_id` only).

---

## Step 6b — Assign devices to a zone

```python
class ZoneAssignmentService:
    def assign(self, device_id: UUID, zone_id: UUID | None) -> None:
        ...  # zone_id set → copy zones.location_id; null → clear both
    def list_devices(self, location_id: UUID, zone_id: UUID) -> list[Device]:
        ...
```

| Method | Path |
|--------|------|
| PATCH | `/api/devices/{id}/zone` body `{ "zone_id": "<uuid>" }` or `{ "zone_id": null }` |
| GET | `/api/locations/{location_id}/zones/{zone_id}/devices` |

404 missing device, missing zone, or zone not in that location. Do not send `location_id` in the PATCH body. The handler must not call `LocationConfigBuilder`.

**Check:** two devices assigned to zone 2 are the only rows in that list. Unassign clears `zone_id` and `location_id`.

---

## Step 7 — Frontend wizard

Mount in `#configuration`. On load, `GET /api/locations` fills a location list that survives refresh. Create adds a location and selects it. Select loads `GET .../config`. Confirm before delete; deleting the selection clears the zone picker.

On the selected location: add a zone, edit name, thresholds, and schedule, and delete a zone. The last zone cannot be deleted. Inline validation + API error banner. Tailwind cards from earlier phases.

Device cards: zone picker calling PATCH. Options are every saved location’s zones, grouped by location and labeled `{location name} — {zone name}`, or an unassigned empty state. After a zone delete, devices that were in it show as unassigned. The picker runs after save, not inside `build()`.

Types in `api.ts` must match JSON keys (`location_id`, not `greenhouse_id`).

---

## Step 8 — Tests (names)

- `test_build_success`
- `test_build_requires_name` / `test_build_requires_zones`
- `test_build_rejects_invalid_thresholds`
- Optional API: POST 201 + GET structure with `location_id`
- `test_assign_devices_to_zone`
- `test_unassign_clears_zone_and_location`
- `test_list_locations`
- `test_delete_location_clears_assignments`
- `test_add_zone_to_location`
- `test_update_zone_rejects_invalid_thresholds`
- `test_delete_zone_clears_assignments`
- `test_delete_last_zone_is_rejected`

```powershell
$env:PYTHONPATH = "src"
pytest tests -q
```

---

## Step 9 — Pattern doc

`docs/patterns/builder.md`: problem, fluent `build()`, vs Factory Method / Abstract Factory, `location_id` naming, extension idea.

---

## Phase 4 completion checklist

- [ ] `locations` + `zones`; FK `zones.location_id`
- [ ] `devices.zone_id`; assign copies `location_id`; unassign clears both
- [ ] Builder validates before persist and does not attach devices
- [ ] POST/GET location config, location list/delete, zone add/edit/delete, PATCH assign, and zone device list live in Scalar
- [ ] Location list survives refresh; select and delete work; zones on the selection can be added, edited, and deleted
- [ ] Wizard creates config; device cards assign a zone labeled with its location name
- [ ] Builder and assignment tests pass
- [ ] Pattern doc written

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Autogenerate empty | Import new models in `env.py` |
| `greenhouse_id` in migration | Rename models; regenerate |
| 400 on POST | Read `detail` — thresholds or empty zones |
| Zones missing on GET | Load relationship after commit |
| Assign sets a mismatched location | Copy `location_id` from the zone row in the same write |
| Builder test touches devices | Keep assignment in `ZoneAssignmentService` |
| Delete leaves `location_id` set | Clear both device columns before the zone row is removed |
| Last zone disappears | Return 400 and keep that zone |

---

## Non-goals

Requiring every device to be assigned; putting `add_device` on the builder; calling the builder to list, delete, or edit a saved location or zone; renaming a location; replacing a whole saved config through `build()`; strategy execution (Phase 6).

---

## Next

→ [Phase 5 — Adapter (requirements)](../phase-05/requirements.md) · [guided check](../phase-05/guided-check.md)

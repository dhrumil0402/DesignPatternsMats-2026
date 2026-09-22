# Phase 5 — Adapter (Guided check)

Complete the [requirements](requirements.md) first. Use this document if you are stuck or to check signatures and JSON. Snippets are **not** a complete solution.

## Pattern

**Adapter** — vendor/simulation/MQTT payloads → unified `SensorPort`; actuator stub → `ActuatorPort`. One ingest path writes `sensor_readings`. The simulation sampler calls that path on an interval. No broker in this phase.

```text
POST /read → SensorPort.read() → ReadingIngest → sensor_readings → ReadingDto
SimulationSampler.run_once(now) → same ingest, only protocol=simulation and tracking_enabled
MqttSensorAdapter.translate(payload) → Reading (source=mqtt); device HTTP or an optional broker arrives in Phase 12
```

---

## Target repository layout (end state)

```text
backend/src/
├── domain/sensors/
│   ├── ports.py          # SensorPort
│   └── reading.py        # normalized Reading
├── domain/actuators/
│   └── ports.py          # ActuatorPort
├── application/readings/
│   ├── dto.py
│   ├── service.py        # ReadingIngest
│   └── sampler.py        # SimulationSampler
├── infrastructure/
│   ├── adapters/sensors/         # simulation.py, vendor_stub.py, mqtt.py
│   ├── adapters/actuators/simulation.py  # SimulationActuatorAdapter
│   └── persistence/
│       ├── models.py     # ReadingRow + sampling columns on devices
│       └── reading_repository.py
└── interfaces/api/sensors.py   # + read/history + PATCH sampling
```

---

## Step 1 — Database model

```text
sensor_readings(id, device_id FK, value, unit, source, recorded_at)
Index: (device_id, recorded_at DESC)

devices.sampling_interval_seconds  int NOT NULL default 300
devices.tracking_enabled           bool NOT NULL default true
```

ORM lives in infrastructure (`ReadingRow`), not in domain. Backfill interval from `default_config->>'sampling_interval_seconds'` when that value is numeric; otherwise keep `300`. Columns are the source of truth after this revision.

**Check:** model includes FK to `devices`, an index that supports “latest reading per device,” and both sampling columns.

---

## Step 2 — Generate migration via Alembic

```powershell
alembic revision --autogenerate -m "sensor_readings"
alembic upgrade head
```

Review `upgrade()`: create table, index, FK, and the two `devices` columns. Add the JSON backfill if autogenerate omitted it. Do not edit applied revisions.

**Check:** `alembic current` is the new revision; `\d sensor_readings` shows the table; `devices` has the sampling columns.

---

## Step 3 — Ports and adapters

```python
class SensorPort(ABC):
    @abstractmethod
    def read(self, device: Device) -> Reading:
        ...

class SimulationSensorAdapter(SensorPort):
    def read(self, device: Device) -> Reading:
        ...  # generate in code; source="simulation"
        # moisture_sensor: value in 0.2–0.6, unit "vwc"
        # light_sensor: value in 200–2000, unit "lux"

class VendorStubSensorAdapter(SensorPort):
    def read(self, device: Device) -> Reading:
        ...  # translate a different raw dict/object; source="vendor"

class MqttSensorAdapter:
    def translate(self, device: Device, payload: dict) -> Reading:
        ...  # payload {"value": 0.41, "unit": "vwc"} → source="mqtt"
```

`Reading` fields: `device_id`, `value`, `unit`, `source`, `recorded_at`.

Selector: `default_config.protocol` is `simulation` or `mqtt`. Document a separate flag for the vendor stub so it does not collide. Application code depends on `SensorPort`, not on adapter classes (except a factory/selector). Do not open a broker socket here.

**Check:** simulation and vendor adapters can yield different `source` values; simulation values stay in range; MQTT `translate` needs only a dict; vendor adapter **translates**, it does not decide irrigation policy.

---

## Step 3b — Actuator port (simulation stub)

Define `ActuatorPort` and a simulation adapter now so Phase 9 can wrap it with decorators. No GPIO or real hardware.

```python
class ActuatorPort(ABC):
    @abstractmethod
    def apply(self, device_id: UUID, command: str, payload: dict) -> None:
        ...

class SimulationActuatorAdapter(ActuatorPort):
    def apply(self, device_id: UUID, command: str, payload: dict) -> None:
        ...  # log or in-memory record only; no physical output
```

Selector can mirror sensors (`device_family` / `protocol`). A unit test or thin demo route that calls `apply` is enough this phase.

**Check:** `ActuatorPort` and `SimulationActuatorAdapter` exist; Phase 9 can import the innermost adapter from `infrastructure/adapters/actuators/simulation.py`.

---

## Step 4 — Persistence and application service

```python
class ReadingRepository:
    def insert(self, reading: Reading) -> Reading: ...
    def list_for_device(self, device_id: UUID, limit: int = 20) -> list[Reading]: ...

class ReadingIngest:
    def take_reading(self, device_id: UUID) -> ReadingDto: ...
    def record(self, device_id: UUID, reading: Reading) -> ReadingDto: ...
    def list_readings(self, device_id: UUID, limit: int = 20) -> list[ReadingDto]: ...

class SimulationSampler:
    def run_once(self, now: datetime) -> None: ...
```

Ingest: load device (missing → not-found) → select adapter → `port.read()` → persist → map DTO. `record` persists an already translated reading (MQTT). Start the sampler from the app lifespan; tests call `run_once(now)` only.

Sampler loads sensors with `protocol` `simulation` and `tracking_enabled` true. Insert when there is no prior row or `now - last recorded_at` ≥ `sampling_interval_seconds`. Skip MQTT devices and tracking-off devices.

**Check:** each successful read **appends** a row. A second `run_once` inside the interval does not insert. Disabled and MQTT devices gain no sampler rows.

---

## Step 5 — API endpoints

| Method | Path |
|--------|------|
| POST | `/api/sensors/{id}/read` |
| GET | `/api/sensors/{id}/readings?limit=1` |
| PATCH | `/api/devices/{id}/sampling` |

PATCH body: `{ "sampling_interval_seconds": 30, "tracking_enabled": true }`. `400` when the interval is below `5`.

Response:

```json
{
  "device_id": "<uuid>",
  "value": 0.31,
  "unit": "vwc",
  "source": "simulation",
  "recorded_at": "2026-08-28T09:00:00Z"
}
```

404 if device missing; 400 for adapter/validation failures. Scalar **sensors** tag updates automatically.

**Check:** two POSTs increase row count; PATCH round-trips interval and tracking; Scalar shows the reading schema.

---

## Step 6 — Frontend

Sensor card: “Read now”, latest value, source badge (`simulation`, `mqtt`, or `vendor`), inputs for interval and tracking (PATCH). Loading and error states. After refresh, show last **DB** reading.

Poll `GET .../readings?limit=1` every few seconds so sampler rows appear without “Read now”. Comment that Phase 12 replaces this poll with WebSocket.

**Check:** refresh still shows the stored value, not only React state. Tracking off stops new sampler rows for that card; a shorter interval inserts sooner.

---

## Step 7 — Tests

- `test_vendor_adapter_normalizes_raw_payload`
- `test_simulation_adapter_value_in_range`
- `test_mqtt_adapter_translates_payload`
- `test_sampler_respects_interval_and_tracking`
- `test_read_inserts_sensor_reading`

Translation and sampler tests should not require HTTP or a broker. Use a fake clock for `run_once(now)`.

```powershell
$env:PYTHONPATH = "src"
pytest tests -q
```

---

## Step 8 — Documentation

`docs/patterns/adapter.md`: problem (incompatible APIs), solution (port + adapters), where in code, extension (third vendor).

---

## Phase 5 completion checklist

- [ ] Table + index applied
- [ ] `sampling_interval_seconds` + `tracking_enabled` on `devices` (backfilled)
- [ ] Simulation + vendor adapters behind `SensorPort`; MQTT `translate` from a dict
- [ ] `ActuatorPort` + `SimulationActuatorAdapter` (stub apply; no GPIO)
- [ ] POST read persists a row; sampler inserts only when interval elapsed and tracking is on
- [ ] UI shows value + source from DB, plus interval and tracking controls; temporary poll
- [ ] Tests + pattern doc

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Autogenerate empty | Import `ReadingRow` / models in `alembic/env.py` |
| Latest reading is slow / wrong | Index `(device_id, recorded_at DESC)` |
| Router imports vendor XML/JSON types | Depend on `SensorPort` only |
| UI loses value on refresh | Persist each read; fetch last row from DB |
| Sampler inserts forever | Honor `sampling_interval_seconds` and `tracking_enabled`; skip `protocol=mqtt` |
| MQTT test needs a broker | `translate(payload)` only; device HTTP and the optional subscriber are Phase 12 |

---

## Next

→ [Phase 6 — Strategy (requirements)](../phase-06/requirements.md) · [guided check](../phase-06/guided-check.md)

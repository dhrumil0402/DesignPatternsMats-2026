# Phase 2 — Factory Method (Guided check)

Complete the [requirements](requirements.md) first. Use this document if you are stuck or to check signatures, layout, and JSON. Snippets are **not** a complete solution.

**Time estimate:** 3–5 hours.

## Pattern

**Factory Method** — creator interface (`create_sensor`); concrete creators choose type and defaults. Callers depend on the abstract creator, not `MoistureSensor(...)` in HTTP handlers.

---

## Target repository layout (end state)

Paths are relative to **`yourpath\project\`** (replace with your clone path).

```text
yourpath\project\
├── docker-compose.yml              # postgres + backend + frontend
├── backend/
│   ├── Dockerfile                  # NEW
│   ├── alembic/versions/
│   │   ├── 001_baseline.py
│   │   └── <rev>_devices.py        # NEW — autogenerate
│   └── src/
│       ├── domain/sensors/
│       │   ├── entity.py
│       │   └── creators.py
│       ├── application/sensors/service.py
│       ├── infrastructure/persistence/
│       │   ├── base.py
│       │   ├── models.py
│       │   └── device_repository.py
│       └── interfaces/api/sensors.py
└── frontend/
    ├── Dockerfile                  # NEW
    └── src/
        ├── services/api.ts
        └── features/sensors/SensorList.tsx
```

---

## Step 0 — Working Phase 1 stack (host is fine)

Confirm Phase 1 still works the Phase 1 way (Postgres via Compose; backend/frontend on the host) **before** you containerize the apps in Step 1.

```powershell
cd "yourpath\project\"
docker compose up -d
```

Optional host check: venv → `uvicorn` on `:8000`; `npm run dev` on `:5173`. Health and dashboard OK.

---

## Step 1 — Full-stack Compose

Add Dockerfiles and extend Compose. Intent: **one** `docker compose up --build` runs DB + API + UI with bind mounts for reload.

**`backend/Dockerfile`** (shape — adjust paths to match your layout):

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY pyproject.toml ./
COPY src ./src
COPY alembic ./alembic
COPY alembic.ini ./
COPY tests ./tests
RUN pip install --no-cache-dir -e ".[dev]"
ENV PYTHONPATH=/app/src
WORKDIR /app
CMD ["uvicorn", "main:app", "--reload", "--host", "0.0.0.0", "--port", "8000", "--app-dir", "src"]
```

**`frontend/Dockerfile`:**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
EXPOSE 5173
CMD ["sh", "-c", "npm ci && npm run dev -- --host 0.0.0.0 --port 5173"]
```

**Compose services** (add alongside existing `postgres`; names can vary):

```yaml
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+psycopg://${POSTGRES_USER:-greenhouse}:${POSTGRES_PASSWORD:-greenhouse}@postgres:5432/${POSTGRES_DB:-greenhouse}
      CORS_ORIGINS: http://localhost:5173
    volumes:
      - ./backend/src:/app/src
      - ./backend/alembic:/app/alembic
      - ./backend/tests:/app/tests
    depends_on:
      postgres:
        condition: service_healthy

  frontend:
    build: ./frontend
    ports:
      - "5173:5173"
    environment:
      VITE_API_BASE_URL: http://localhost:8000
    volumes:
      - ./frontend:/app
      - frontend_node_modules:/app/node_modules
    depends_on:
      - backend

volumes:
  # keep existing postgres volume
  frontend_node_modules:
```

Inside the backend container the DB host is **`postgres`**, not `localhost`. The browser still calls `http://localhost:8000`.

```powershell
docker compose up --build -d
docker compose ps
curl http://localhost:8000/health
```

**Check:** three services running; health `"db": "ok"`; dashboard at http://localhost:5173/dashboard.

---

## Step 2 — Persistence layer (before migration)

**`base.py`**

```python
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

**`DeviceRow`** — columns to include (write the mapped types yourself):

| Column | Notes |
|--------|--------|
| `id` | UUID PK, `gen_random_uuid()` server default |
| `device_type` | String(64), not null |
| `role` | String(32), default `"sensor"` |
| `display_name` | String(128), nullable |
| `default_config` | JSONB, default `{}` |
| `created_at` | timestamptz, `now()` |
| Index | `ix_devices_role` on `role` |

**Alembic `env.py`** — replace `target_metadata = None` with `Base.metadata`, and import `models` so autogenerate is not empty.

**`get_db`** — yield a `SessionLocal()` and close it in `finally`.

---

## Step 3 — Autogenerate

From the project root with Compose up:

```powershell
docker compose exec backend alembic revision --autogenerate -m "devices"
```

If Alembic’s cwd is wrong inside the image, set `working_dir` / `command` so `alembic.ini` is found (often `/app`, not `/app/src`). Review: `down_revision` is Phase 1 id; `upgrade()` contains `op.create_table("devices", ...)`. If `upgrade()` is `pass` only, metadata is not wired — delete the file and regenerate.

```powershell
docker compose exec backend alembic upgrade head
docker exec -it greenhouse-postgres psql -U greenhouse -d greenhouse -c "\d devices"
```

Discipline for later phases: change models → autogenerate → review → upgrade (via `docker compose exec backend`). Never edit applied revisions.

---

## Step 4 — Domain + Factory Method

**`Sensor`** fields: `id: UUID | None`, `device_type`, `display_name`, `default_config`.

Creator surface you should have:

```python
class SensorCreator(ABC):
    @abstractmethod
    def create_sensor(self, display_name: str | None = None) -> Sensor:
        ...

class MoistureSensorCreator(SensorCreator):
    def create_sensor(self, display_name: str | None = None) -> Sensor:
        ...  # device_type="moisture_sensor"; config includes unit/threshold/interval

class LightSensorCreator(SensorCreator):
    def create_sensor(self, display_name: str | None = None) -> Sensor:
        ...  # device_type="light_sensor"; different unit/interval

def get_creator(sensor_type: str) -> SensorCreator:
    ...  # registry: "moisture" | "light"; unknown → ValueError
```

**Check:** moisture vs light defaults are not the same dict.

---

## Step 5 — Repository + service

```python
class DeviceRepository:
    def save_sensor(self, sensor: Sensor) -> Sensor: ...
    def list_sensors(self) -> list[Sensor]: ...  # role == "sensor"

class SensorService:
    def create_sensor(self, sensor_type: str, display_name: str | None = None) -> Sensor:
        creator = get_creator(sensor_type)
        prototype = creator.create_sensor(display_name=display_name)
        return self._repo.save_sensor(prototype)

    def list_sensors(self) -> list[Sensor]:
        ...
```

**Check:** `commit()` happens; id is populated after save.

---

## Step 6 — REST

Prefix `/api/sensors`, tags `["sensors"]`.

Request: `{ "type": "moisture" | "light", "display_name": string | null }`

Response:

```json
{
  "id": "<uuid>",
  "device_type": "moisture_sensor",
  "display_name": "Soil moisture sensor",
  "default_config": { "sampling_interval_seconds": 300, "unit": "vwc" }
}
```

`POST` → 201; unknown type → 400. `include_router` in `main.py`.

```powershell
curl http://localhost:8000/api/sensors
curl -X POST http://localhost:8000/api/sensors -H "Content-Type: application/json" -d "{\"type\":\"moisture\"}"
```

**Check:** Scalar **sensors** tag; GET still returns rows after `docker compose restart backend`.

---

## Step 7 — Frontend

`api.ts` types/functions: `SensorDto`, `fetchSensors()`, `createSensor(type, displayName?)`.

`SensorList` responsibilities: load on mount, buttons for moisture/light, error banner, empty state, list with name + type + config. Mount in `#sensors`; leave other placeholders.

**Check:** add both types, refresh, both still listed.

---

## Step 8 — Tests (names, not full files)

- `test_moisture_creator_defaults` — type `moisture_sensor`; threshold key present
- `test_light_creator_defaults` — type `light_sensor`; unit differs
- Optional API test for GET/POST

```powershell
docker compose exec backend pytest tests -q
```

(Host venv + `PYTHONPATH=src` remains fine for unit tests.)

---

## Step 9 — Pattern doc

`docs/patterns/factory-method.md`: problem, solution, code paths, exercise (temperature creator).

---

## Phase 2 completion checklist

- [ ] Compose runs postgres + backend + frontend; health + dashboard OK
- [ ] Autogenerate reviewed and applied via Compose exec
- [ ] `\d devices` shows expected columns
- [ ] POST moisture/light → 201 with distinct `default_config`
- [ ] GET persists across backend restart
- [ ] Scalar sensors tag
- [ ] Dashboard Sensors section works
- [ ] Creator tests pass
- [ ] Pattern doc written

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Backend cannot reach DB | `DATABASE_URL` host must be `postgres` in Compose |
| Empty autogenerate | Import `Base` + `models` in `env.py` |
| `relation "devices" does not exist` | `docker compose exec backend alembic upgrade head` |
| 404 on `/api/sensors` | `include_router` |
| Empty list after create | `commit()` in repository |
| Frontend can’t reach API | Publish `8000`; `VITE_API_BASE_URL=http://localhost:8000` |
| Test import errors | `PYTHONPATH=src` or run pytest in the backend container |

---

## Non-goals

`device_family`, actuators, live readings, locations/zones.

---

## Next

→ [Phase 3 — Abstract Factory (requirements)](../phase-03/requirements.md) · [guided check](../phase-03/guided-check.md)

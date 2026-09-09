# Phase 14 — Tests, docs, demo (Guided check) *(Optional enrichment)*

Complete the [requirements](requirements.md) first. Use this document if you are stuck or to check coverage and file names. This is **not** a paste-ready test suite.

> **Optional enrichment** — Not required for course completion. Complete Phases 1–12 first.

This phase is **not** a GoF pattern lab.

---

## Suggested test layout

```text
backend/tests/
  test_sensor_creators.py
  test_family_factory.py
  test_location_config_builder.py
  test_adapters.py
  test_strategies.py
  test_state_transitions.py
  test_decorators.py
  test_commands.py
  test_observer_alerts.py
  integration/
    conftest.py              # test DB session, migrate, rollback
    test_locations_api.py
    test_readings_api.py
    test_automation_api.py
    test_commands_api.py
```

```powershell
$env:PYTHONPATH = "src"
$env:DATABASE_URL = "postgresql+psycopg://greenhouse:greenhouse@localhost:5432/greenhouse_test"
pytest tests -q
```

Use a **separate** database name for tests when you can; create it once in Compose or `createdb`.

---

## Integration fixture sketch

```python
@pytest.fixture
def client(db_session):
    def override_get_db():
        yield db_session
    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

Fill `db_session` with a transactional session against a migrated test DB — do not copy a full conftest from a blog as the submission; wire it to **your** `get_db`.

---

## Migration smoke

```powershell
docker compose up --build -d
docker compose exec backend alembic upgrade head
docker compose exec backend alembic current
```

Host venv + published Postgres port remains fine for local pytest. Optional: `docker compose exec backend pytest`.

CI job equivalent: start Compose stack → upgrade head → pytest.

---

## E2E / scenario (checklist)

If using Playwright, names to aim for:

- `health badge shows db ok`
- `evaluate automation shows recommendation`
- `start pump enables running badge`
- `event feed shows new item`

API-only alternative: a `scripts/demo_scenario.py` that prints each step’s HTTP status — acceptable if documented in `docs/demo.md`.

---

## Pattern doc skeleton (every file)

1. Problem (greenhouse)
2. Intent (your words)
3. Module paths
4. Extension exercise

Files: `factory-method.md`, `abstract-factory.md`, `builder.md`, `adapter.md`, `strategy.md`, `facade.md`, `state.md`, `decorator.md`, `command.md`, `observer.md`.

---

## Demo narrative headings you can reuse

1. Read (Adapter) → event (Observer)
2. Strategy + Facade overview
3. Command + State + Decorator
4. WebSocket feed (Phase 12)
5. Alert resolve (optional Phase 13, if implemented)

---

## Done

**Optional enrichment track complete.** Required course completion is at Phase 12.

→ [Phase overview](../README.md)

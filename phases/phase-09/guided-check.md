# Phase 9 — Decorator (Guided check)

Complete the [requirements](requirements.md) first. Use this document if you are stuck or to check signatures and JSON. Snippets are **not** a complete solution.

## Pattern

**Decorator** — layered `ActuatorPort` implementations.

```text
LoggingDecorator → MaxRunTimeDecorator → SimulationActuatorAdapter
```

---

## Target layout

```text
domain/actuators/ports.py
infrastructure/adapters/actuators/
  simulation.py
  logging_decorator.py
  max_runtime_decorator.py
infrastructure/persistence/
  models.py                 # ActuatorExecutionLogRow
  execution_log_repository.py
interfaces/api/actuators.py
frontend/.../ExecutionLog.tsx
```

---

## Step 1 — Database

```text
actuator_execution_log(
  id, device_id FK, success BOOLEAN,
  policy_applied JSONB,     # e.g. ["maxRunTime","logging"]
  message TEXT, created_at
)
```

```powershell
alembic revision --autogenerate -m "actuator_execution_log"
alembic upgrade head
```

**Check:** table exists; FK to `devices` is valid.

---

## Step 2 — Decorator stack

```python
class ActuatorPort(ABC):
    @abstractmethod
    def apply(self, device_id: UUID, command: str, payload: dict) -> None:
        ...

class LoggingActuatorDecorator(ActuatorPort):
    def __init__(self, inner: ActuatorPort, log_repo: ...) -> None: ...
    def apply(self, ...): ...  # try inner; always write a log row

class MaxRunTimeActuatorDecorator(ActuatorPort):
    def apply(self, ...): ...  # if running too long → raise; do not call inner
```

Compose in the application service / composition root (not in the router with nested `if`). Make order explicit (e.g. logging outermost). Max-run-time may use persisted `actuator_states.updated_at` or a documented clock.

**Check:** blocked policy does not call inner (prove in a test); failed policy still writes `success=false` with policy name in JSON.

---

## Step 3 — API and UI

`GET /api/actuators/executions?limit=20`

```json
{
  "id": "<uuid>",
  "device_id": "<uuid>",
  "success": false,
  "policy_applied": ["maxRunTime", "logging"],
  "message": "Max run time exceeded",
  "created_at": "..."
}
```

Optional apply/demo endpoint until Command. UI table: time, device, success badge, policies, message. Empty state. Tailwind.

Demo: force a running state older than the threshold, then apply — expect `success=false` after refresh.

**Check:** Scalar lists the endpoint; blocked attempt appears in the UI from the table.

---

## Step 4 — Tests and docs

- `test_max_runtime_does_not_call_inner`
- `test_failed_policy_writes_execution_row`
- `docs/patterns/decorator.md`: wrapping, order, vs inheritance

---

## Phase 9 completion checklist

- [ ] Decorated port in use
- [ ] Execution log persisted and shown
- [ ] Policy failure visible as unsuccessful row
- [ ] Tests + pattern doc

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Each decorator has a different method name | Share `ActuatorPort.apply` |
| Log only in stdout | Insert `actuator_execution_log` |
| Policy `if` in the router | Wrapper classes in the stack |

---

## Next

→ [Phase 10 — Command (requirements)](../phase-10/requirements.md) · [guided check](../phase-10/guided-check.md)

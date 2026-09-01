# Phase 10 — Command (Guided check)

Complete the [requirements](requirements.md) first. Use this document if you are stuck or to check signatures and JSON. Snippets are **not** a complete solution.

## Pattern

**Command** — encapsulate requests; persist audit.

---

## Target layout

```text
domain/commands/
  command.py            # Command ABC
  start_pump.py
  stop_pump.py
  noop.py               # optional
application/commands/
  invoker.py
  dto.py
  service.py
infrastructure/persistence/
  models.py             # CommandLogRow
  command_log_repository.py
interfaces/api/commands.py
frontend/.../CommandPanel.tsx
```

---

## Step 1 — Database

```text
command_log(
  id, device_id FK, command_type, payload JSONB,
  status,               # accepted | rejected | executed | failed
  error_message, created_at
)
```

```powershell
alembic revision --autogenerate -m "command_log"
alembic upgrade head
```

**Check:** table exists; rejected commands still insert a row (`status=rejected`).

---

## Step 2 — Command objects

```python
class Command(ABC):
    type: str
    def execute(self) -> None: ...

class StartPumpCommand(Command): ...
class StopPumpCommand(Command): ...

class CommandInvoker:
    def submit(self, command: Command) -> CommandLogDto: ...
```

Execute path (implement in invoker / command, not in the router):

1. **Guard** — load actuator state; if command not in `allowed_commands` → persist `rejected`, HTTP 409/400, **stop**.
2. **Port** — `ActuatorPort.apply(...)` (decorated stack from Phase 9).
3. **State** — legal transition (e.g. start → running, stop → idle).
4. **Log** — `status=executed` or `failed` with `error_message`.

Unknown `command_type` → 400. Invoker should not contain `if command_type == "start_pump":` business logic — map type → class, then `execute()`.

**Check:** illegal-state submit writes `rejected` and does **not** change `actuator_states`; legal start changes state to running.

---

## Step 3 — API

`POST /api/devices/{id}/commands`

```json
{ "command_type": "start_pump", "payload": {} }
```

Responses: 202/200 + log DTO; 409 illegal state; 404 missing device; 400 unknown type.

`GET /api/commands?limit=20` — newest first.

**Check:** Scalar shows both routes; rejected POST still appears on GET.

---

## Step 4 — Frontend

`#commands`: disabled buttons from `allowed_commands`; POST on click; last result; recent log table. Confirm dialog optional until Phase 13.

**Check:** button disabled when command not allowed; after start, stop becomes available (via refreshed `allowed_commands`).

---

## Step 5 — Tests and docs

- `test_illegal_command_rejected_logged`
- `test_start_pump_transitions_to_running`
- `docs/patterns/command.md`: object vs function, invoker, audit

---

## Phase 10 completion checklist

- [ ] Command classes + invoker
- [ ] State-aware reject + persist
- [ ] Decorated port used on execute
- [ ] UI + tests + pattern doc

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Router implements start/stop | Command object + invoker |
| Rejected command not in history | Insert `command_log` before returning 409 |
| State changes on reject | Guard first; save state only after success |
| Decorators skipped | Call `ActuatorPort`, not a raw adapter |

---

## Next

→ [Phase 11 — Observer (requirements)](../phase-11/requirements.md) · [guided check](../phase-11/guided-check.md)

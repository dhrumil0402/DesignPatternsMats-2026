# Phase 10 — Command (Requirements)

This is the **assignment**. Implement from these requirements. If you get stuck or want to check your design, use the [guided check](guided-check.md) (signatures and partial snippets — not a complete solution).

After you implement, answer the [questions](./questions.md).

## Purpose

Encapsulate actuator actions as **command objects** with **auditable history** in PostgreSQL. Integrate Phase 8 state guards and the Phase 9 decorator stack.

## Scope and naming rules

- HTTP handlers create/execute commands via an invoker; they do not call actuator adapters directly.
- Persist every attempt in `command_log` (`success`, `failed`, `rejected`).
- Rejected (illegal state) commands must **not** change `actuator_states`.

## Outcome required at end of phase

- Commands exist for at least start pump, stop pump, and set light level (or equivalent named operations).
- `POST /api/commands` executes; `GET /api/commands` lists history.
- UI controls respect `allowed_commands`; command history table is visible.
- Successful start updates state to `running` and writes `command_log`.
- Scalar documents command endpoints.
- Pattern note at `docs/patterns/command.md`.

---

## Prerequisites

- Phase 9: decorated ports + execution log.
- Phase 8: transition guards and `allowed_commands`.
- Sensor readings may arrive from the sampler or MQTT ingest, not only `POST /read`. Do not add a scheduler, MQTT client, or the device readings route in this phase.

---

## Required architecture for this phase

- **Domain:** command interface `execute()`; concrete commands; invoker.
- **Application:** DTOs for enqueue/result; service wires state check → decorated port → state update → command log.
- **Infrastructure:** `command_log` table + repository.
- **API:** POST command, GET history.
- **Frontend:** actuator controls + history; disable buttons not in `allowed_commands`.

---

## Suggested file locations

Paths are relative to **`yourpath\project\`**.

```text
yourpath\project\
├── backend/src/
│   ├── domain/commands/base.py           # Command
│   ├── domain/commands/pump.py           # StartPumpCommand, StopPumpCommand
│   ├── domain/commands/light.py          # SetLightLevelCommand
│   ├── domain/commands/invoker.py
│   ├── application/commands/dto.py
│   ├── application/commands/service.py
│   ├── infrastructure/persistence/models.py              # CommandLogRow
│   ├── infrastructure/persistence/command_log_repository.py
│   └── interfaces/api/commands.py
└── frontend/src/
    ├── .../ActuatorControls.tsx          # #controls
    └── .../CommandHistory.tsx
```

## Type hints

Domain:

| Type | Hint |
|------|------|
| `command_type` | `str` — e.g. `start_pump`, `stop_pump`, `set_light_level` |
| `payload` | `dict` |
| `status` | `str` — `"success"` \| `"failed"` \| `"rejected"` |
| `CommandResult` | `status`, optional `error_message: str` |

ORM `command_log`:

| Column | SQL hint |
|--------|----------|
| `id` | UUID PK |
| `command_type` | VARCHAR |
| `device_id` | UUID FK → `devices` |
| `payload` | JSONB |
| `status` | VARCHAR |
| `error_message` | TEXT, nullable |
| `created_at` | timestamptz |

POST body JSON: `{ command_type: string, device_id: UUID, payload?: object }`.

---

## Step-by-step implementation requirements

## Step 1 — Database

`command_log`:

| Column | Intent |
|--------|--------|
| Identifier | PK |
| `command_type` | VARCHAR |
| `device_id` | FK → `devices` |
| `payload` | JSONB |
| `status` | `success` / `failed` / `rejected` |
| `error_message` | Nullable text |
| `created_at` | Timestamptz |

Autogenerate and apply.

Acceptance criteria:

- Table exists with FK to devices.

## Step 2 — Command objects

Implement `StartPumpCommand`, `StopPumpCommand`, `SetLightLevelCommand` (names may vary), each with `execute()`.

Invoker runs a command and does not contain device-specific if-ladders for every type (registry or polymorphism).

Execute path:

1. Validate current state / `allowed_commands`.
2. If illegal → status `rejected`, no state change, no inner apply (or apply not reached).
3. If legal → decorated port `apply` → update `actuator_states` → log `success` or `failed`.

Acceptance criteria:

- Rejected command test: status `rejected`, state unchanged.
- Success: pump start → `running` + log row `success`.

## Step 3 — API

- `POST /api/commands` body: command type, device id, optional payload.
- `GET /api/commands?limit=50`.

404 missing device; 400 unknown command type; rejected can be 409 or 200 with `status=rejected` — pick one, document it, keep it consistent.

Acceptance criteria:

- Scalar documents request/response.
- GET returns newest first.

## Step 4 — Frontend

- **Actuator controls** in `#controls`: POST command; disable when `allowed_commands` omits the action.
- **Command history** from `command_log`.
- Confirm destructive actions if you already have a pattern; otherwise optional Phase 13 can add a dialog.

Acceptance criteria:

- User can start/stop from UI when state allows; history updates.

## Step 5 — Tests and docs

- Rejected illegal state.
- Successful execute updates state + log.
- `docs/patterns/command.md`: intent, invoker, vs calling the port from the router.

---

## Definition of done

- Command objects + invoker in use.
- `command_log` written for success/fail/reject.
- UI controls + history work with `allowed_commands`.
- Tests and pattern doc complete.

---

## Common pitfalls to avoid

- Router calling `ActuatorPort` directly, bypassing commands.
- Rejected commands still flipping state.
- Ignoring decorator failures (should be `failed` in `command_log` and execution log).

---

## Handoff to next phases

- Phase 11 (Observer) should publish events when commands succeed or fail.

→ [Phase 11 — Observer (requirements)](../phase-11/requirements.md) · [guided check](../phase-11/guided-check.md)

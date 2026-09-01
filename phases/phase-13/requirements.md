# Phase 13 — Dashboard polish (Requirements) *(Optional enrichment)*

This is the **assignment**. Implement from these requirements. If you get stuck or want to check your design, use the [guided check](guided-check.md) (signatures and partial snippets — not a complete solution).

> **Optional enrichment** — This phase is not required for course completion.
> Complete Phases 1–12 first. Do this if you want deeper UX polish (charts, empty/error states, visual cohesion).

This phase is **not** a GoF pattern lab. It brings the incrementally built dashboard to **operator-ready** quality.

## Purpose

Improve layout cohesion, charts, empty/error states, and consistent UX. This is not a greenfield React rewrite.

## Scope and naming rules

- Keep Tailwind v4; prefer `@theme` or shared utility patterns over new CSS frameworks.
- Charts read **historical** `sensor_readings` (Phase 5), not random client-side numbers.
- Full auth/roles are **out of scope** unless the course requires them.
- Complex rule-editor UI is out of scope.

## Outcome required at end of phase

- Unified visual language (spacing, typography, status colors).
- Responsive sensor cards and control panel (usable at a mobile width and desktop).
- Empty states and loading skeletons on major panels.
- At least one historical chart (moisture recommended) with a time range query.
- Alerts can be filtered (active vs all); acknowledge/resolve if `PATCH` exists.
- WS reconnect/backoff and a visible connection indicator (if Phase 12 delivered WebSocket; deepen polish here if desired).
- Optional: edit existing location config (not only create).

---

## Prerequisites

- **Required track complete:** Phase 12 (production REST + WebSocket; panels wired to real APIs).
- This optional phase assumes Phase 12 is done; it is not required for course completion.

---

## Required architecture for this phase

- **Frontend:** design tokens; chart component; error boundary or route fallback; typed API models aligned with OpenAPI.
- **Backend:** `GET /api/sensors/{id}/readings?from=&to=` if not already present; optional index migration for chart queries.
- **No new business tables** unless required for reporting.

---

## Suggested file locations

Paths are relative to **`yourpath\project\`**.

```text
yourpath\project\
├── backend/src/interfaces/api/sensors.py     # readings ?from=&to= if missing
├── frontend/src/
│   ├── index.css                             # tokens / @theme
│   ├── services/api.ts
│   ├── services/realtime.ts                  # reconnect/backoff
│   └── .../MoistureChart.tsx                 # or equivalent chart component
└── frontend/src/components/AppLayout.tsx     # WS indicator if not already there
```

## Type hints

Chart query:

| Item | Hint |
|------|------|
| Path | `GET /api/sensors/{id}/readings` |
| `from` / `to` | ISO-8601 `str` query params |
| Item | `{ value: number, unit: string, recorded_at: string }` (sort oldest-to-newest or document) |

Alert PATCH body: `{ "status": "resolved" }` (`status: str`). Filter query: `status=active` or omit for all.

WS backoff: delay `number` milliseconds, cap e.g. 15000; `attempt: number`.

---

## Step-by-step implementation requirements

## Step 1 — Visual / UX polish

- Shared tokens (colors for ok/warn/error, card chrome, type scale).
- Responsive grids for sensors and controls.
- Empty states: “No alerts”, “No readings yet”, etc.
- Skeleton loaders while fetching.

Acceptance criteria:

- Dashboard looks consistent across sections added in Phases 2–12.
- Narrow viewport does not overflow hopelessly (stack cards).

## Step 2 — Historical chart

- Query readings with `from` / `to` (ISO timestamps).
- Render at least moisture over time (library of your choice, or SVG).
- Optional second series (light) if time allows.

Acceptance criteria:

- Chart data matches rows in `sensor_readings` for that device and window.
- Empty range shows an empty state, not a broken canvas.

## Step 3 — Optional database

If chart queries are slow, add a forward-only index migration. No new domain tables.

Acceptance criteria:

- If you skip the migration, document that existing Phase 5 index is enough.

## Step 4 — Alerts workflow

- Filter active vs all.
- If API supports `PATCH /api/alerts/{id}`, acknowledge/resolve from UI and persist.

Acceptance criteria:

- Filter changes the list without a full reload.
- Resolve stays resolved after refresh.

## Step 5 — Engineering hardening

- Types in `api.ts` match Scalar/OpenAPI (fix drift).
- `realtime.ts`: reconnect with backoff; header indicator.
- Error boundary or per-route error UI.

Acceptance criteria:

- Killing the WS server then restarting reconnects without a manual full reload (within your backoff window).

## Step 6 — Panel checklist

| Area | Polish task |
|------|-------------|
| Overview | Single refresh; stale-data indicator |
| Sensors | Sparkline or last-value trend on cards |
| Automation | Show strategy from persisted config |
| Controls | Confirm dialog for pump start |
| Events | Scrollable feed, severity sorting |
| Config | Edit existing location (stretch if timeboxed — mark done/not done) |

Acceptance criteria:

- Each row is either implemented or explicitly deferred in a short README note.

---

## Definition of done

- Visual system is consistent.
- One real historical chart works.
- Empty/error/loading states exist on major panels.
- Alert filter (and patch if implemented) works.
- WS reconnect is visible and functional.
- Non-goals (auth, rule editor) were not accidentally scoped in.

---

## Common pitfalls to avoid

- Rewriting the whole frontend.
- Charting mock Math.random instead of API history.
- Introducing a second CSS framework.

---

## Handoff to optional next phase

- [Phase 14 — Tests, docs, demo (optional)](../phase-14/requirements.md) · [guided check](../phase-14/guided-check.md) — proof-and-demo rigor.

Return to the phase index: [`docs/phases/README.md`](../README.md)

# Student learning guides

These guides are **theory and coaching chapters** for each phase. Read the guide first, then implement the matching lab under [`docs/phases/phase-NN/`](../../phases/). Each lab folder has **requirements** (start here), a **guided check** (if you are stuck or to compare shapes), and — for Phases 1–11 — **questions**.

Tone matches course theory materials: professional theory first, then hands-on follow-along, then a short bridge to your greenhouse lab. Guides paraphrase textbooks; they do **not** copy textbook text.

## Course tracks

| Track                   | Phases | Guides                                                                  |
| ----------------------- | ------ | ----------------------------------------------------------------------- |
| **Required**            | 1–12   | Guides 01–12 (skeleton, patterns, production API / WebSocket hardening) |
| **Optional enrichment** | 13–14  | Guides 13–14 (dashboard polish; tests, docs, demo)                      |

**Course completion** is at Phase 12. Guides 13–14 are for students who want extra polish or proof-and-demo rigor.

## How examples relate to the project

- **Your product** is a smart greenhouse / location control system (sensors, actuators, automation, dashboard).
- **Teaching examples** in Guides 02–14 use _other_ domains (courier notifications, flight itineraries, auctions, ArenaTickets, and so on) on purpose.
- That way you learn the pattern’s _shape_ without copying a ready-made solution into the Phase lab.
- Each guide ends with **Bridge to your lab** — a mapping table (teaching type → greenhouse type) plus one do/don’t for the seam. Bridges do **not** contain FastAPI, Alembic, or React solutions. For routes, columns, and acceptance criteria, follow the phase **requirements** first; the guided check is optional extra help, not a complete solution.

## How to use each week

1. **Read** the learning guide for the phase (this folder) — start with the **Theory** section; it is designed to stand alone.
2. **Follow along** (Part 2) — run the problem demo, build the solution in steps, run the complete script, compare expected output (stdlib Python for pattern guides).
3. **Implement** from the phase **requirements** in `docs/phases/phase-NN/`. Open the guided check afterward to verify signatures and JSON — do not treat it as a copy-paste solution.
4. **Answer** the `questions.md` in that same folder (Phases 1–11) in your own words. The same items may reappear on the exam.
5. **Reflect** — optional short note in `docs/patterns/` (from Phase 2 onward) or your course journal.

Lab commands use **`yourpath\project\`** as a placeholder for the folder that contains `docker-compose.yml`, `backend/`, and `frontend/`. Replace it with **your** clone path. Do not copy someone else’s `C:\Users\...` location.

## Chapter shape (what to expect)

Most guides follow this two-part spine:

**Part 1 — Theory**

1. Narrative hook and learning objectives
2. **Theory** — standalone professional introduction: intent, problem, analogy, structure, when to use, related patterns (Guide 12: required engineering topic; Guides 13–14: optional engineering topics)

**Part 2 — Follow along**

3. Sticky problem — small annotated snippets + a **runnable “before”** demo
4. Pattern in practice — short bridge from theory to the runnable example
5. Building the solution — step-by-step snippets with teaching comments
6. **Complete worked solution** — one self-contained script you can save and run, plus expected output
7. Compare & contrast, traps, key terms
8. Reading assignments, bridge to your lab, summary, next guide

Pattern demos (Guides 02–11) use **stdlib-only Python** so you can paste into a `.py` file and run with `python your_file.py` without installing the course stack.

## Guide index — required (Phases 1–12)

| Phase | Guide                                                                            | Lab (requirements first)                                                                                                                           |
| ----: | -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
|     1 | [Course intro, design patterns & skeleton](./01-course-patterns-and-skeleton.md) | [Requirements](../../phases/phase-01/requirements.md) · [Guided check](../../phases/phase-01/guided-check.md) · [Questions](../../phases/phase-01/questions.md)                                 |
|     2 | [Factory Method](./02-factory-method.md)                                         | [Requirements](../../phases/phase-02/requirements.md) · [Guided check](../../phases/phase-02/guided-check.md) · [Questions](../../phases/phase-02/questions.md)                     |
|     3 | [Abstract Factory](./03-abstract-factory.md)                                     | [Requirements](../../phases/phase-03/requirements.md) · [Guided check](../../phases/phase-03/guided-check.md) · [Questions](../../phases/phase-03/questions.md)                 |
|     4 | [Builder](./04-builder.md)                                                       | [Requirements](../../phases/phase-04/requirements.md) · [Guided check](../../phases/phase-04/guided-check.md) · [Questions](../../phases/phase-04/questions.md)                                   |
|     5 | [Adapter](./05-adapter.md)                                                       | [Requirements](../../phases/phase-05/requirements.md) · [Guided check](../../phases/phase-05/guided-check.md) · [Questions](../../phases/phase-05/questions.md)                                   |
|     6 | [Strategy](./06-strategy.md)                                                     | [Requirements](../../phases/phase-06/requirements.md) · [Guided check](../../phases/phase-06/guided-check.md) · [Questions](../../phases/phase-06/questions.md)                                 |
|     7 | [Facade](./07-facade.md)                                                         | [Requirements](../../phases/phase-07/requirements.md) · [Guided check](../../phases/phase-07/guided-check.md) · [Questions](../../phases/phase-07/questions.md)                                     |
|     8 | [State](./08-state.md)                                                           | [Requirements](../../phases/phase-08/requirements.md) · [Guided check](../../phases/phase-08/guided-check.md) · [Questions](../../phases/phase-08/questions.md)                                       |
|     9 | [Decorator](./09-decorator.md)                                                   | [Requirements](../../phases/phase-09/requirements.md) · [Guided check](../../phases/phase-09/guided-check.md) · [Questions](../../phases/phase-09/questions.md)                               |
|    10 | [Command](./10-command.md)                                                       | [Requirements](../../phases/phase-10/requirements.md) · [Guided check](../../phases/phase-10/guided-check.md) · [Questions](../../phases/phase-10/questions.md)                                   |
|    11 | [Observer](./11-observer.md)                                                     | [Requirements](../../phases/phase-11/requirements.md) · [Guided check](../../phases/phase-11/guided-check.md) · [Questions](../../phases/phase-11/questions.md)                                 |
|    12 | [API, WebSocket & hardening](./12-api-websocket-hardening.md)                    | [Requirements](../../phases/phase-12/requirements.md) · [Guided check](../../phases/phase-12/guided-check.md) |

## Guide index — optional enrichment (Phases 13–14)

| Phase | Guide                                                      | Lab (requirements first)                                                                                                         |
| ----: | ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
|    13 | [Dashboard polish](./13-dashboard-polish.md) _(optional)_  | [Requirements](../../phases/phase-13/requirements.md) · [Guided check](../../phases/phase-13/guided-check.md)     |
|    14 | [Tests, docs & demo](./14-tests-docs-demo.md) _(optional)_ | [Requirements](../../phases/phase-14/requirements.md) · [Guided check](../../phases/phase-14/guided-check.md) |

## Reference bookshelf

- _Head First Design Patterns_ (Freeman & Robson) — course PDF under [`docs/materials/`](../)
- [Refactoring Guru — Design Patterns](https://refactoring.guru/design-patterns)
- Gang of Four (_Design Patterns: Elements of Reusable Object-Oriented Software_) — classic intent definitions

Always prefer your own wording in assignments and reflection notes.

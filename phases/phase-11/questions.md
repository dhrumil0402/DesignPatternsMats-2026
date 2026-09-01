# Phase 11 — Observer questions

**Pattern / focus:** Observer.

**Read first:** [Guide 11](../../materials/guides/11-observer.md) · [Requirements](requirements.md)

## How to answer

- Use your own wording. Do not paste teaching-example types (for example live auction bids) as if they were your greenhouse classes.
- When a question asks about *this application*, refer to the event bus, alerts, publishers, and the event feed from the lab.
- Short answers are fine when the question is narrow. Write a few sentences when it asks you to explain or compare.

## A. Pattern

1. State the intent of Observer in plain language. What is a **one-to-many** dependency here, and why must the subject notify dependents **without** knowing their concrete classes?

2. Name the participants (**subject / event bus**, **observer / subscriber**, **concrete observers**, **event**, **publisher**). What does `subscribe` / `unsubscribe` / `publish` (or `notify`) each do?

3. When should you use Observer, and when should you skip it (only one listener forever; or you need a strict multi-step workflow with guaranteed order)?

## B. This phase of the application

4. Give at least two **domain events** you publish in this lab (for example reading created, threshold crossed, command failed). Who is the **publisher** in each case, and why must that publisher **not** import the alerts repository or the UI?

5. What does `AlertPersistenceSubscriber` (or your equivalent) do when it handles an event? Why do alerts live in the `alerts` table so the event feed still shows them after a **refresh**?

6. The dashboard Event feed **polls** `GET /api/alerts` (or events). `services/realtime.ts` is only a **stub**. Why is full WebSocket **not** required in this phase? What should Phase 12 plug into so you do **not** re-implement pub/sub inside socket handlers?

7. Describe the must-demo path: a low-moisture / threshold-crossed reading leads to an alert row that appears in the feed after poll. Which objects collaborate (reading pipeline → bus → subscriber → API → UI), and which object must **not** write the alert directly?

## C. Compare, contrast, and scenarios

8. Contrast Observer with **Facade**. Facade simplifies **calling into** a subsystem. Observer **fans out** notifications after something happened. How do both appear in this greenhouse (Phase 7 overview vs this phase)?

9. Contrast Observer with **Command**. Command captures **intent to act**. Observer propagates **outcomes**. How can `execute()` publish an event without the command class knowing who persists alerts?

10. One subscriber throws during `notify` / `publish` and aborts the loop, so later subscribers never run. Why is that a trap? What should the bus do so a failing alert writer does not block metrics (or later WebSocket) subscribers?

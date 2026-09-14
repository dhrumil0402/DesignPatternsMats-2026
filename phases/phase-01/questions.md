# Phase 1 — Skeleton questions

**Pattern / focus:** Course intro and an empty-but-running three-tier skeleton (no GoF pattern this phase).

**Read first:** [Guide 01](../../materials/guides/01-course-patterns-and-skeleton.md) · [Requirements](requirements.md)

## How to answer

- Use your own wording. Do not paste textbook definitions or teaching-example class names as if they were your greenhouse types.
- When a question asks about *this application*, refer to what you built (or what the lab required): layers, routes, and tooling.
- Short answers are fine when the question is narrow. Write a few sentences when it asks you to explain or compare.
- Write each answer inside the matching **Your Answer** note. Replace the placeholder; leave the question text unchanged.

## A. Pattern

1. In your own words, what is a design pattern? What is it *not*?

> [!NOTE]
> ***Your Answer***
>
> _(Write your answer here.)_

2. Name the three GoF pattern families. For each family, give one-sentence: what kind of design problem it addresses. Then place **Factory Method** and **Strategy** into the correct family.

> [!NOTE]
> ***Your Answer***
>
> _(Write your answer here.)_

3. A teammate wants to add a pattern “because it is on the course list,” even though the feature is small and unlikely to grow. When should you **skip** a pattern? What risk do you take if you apply one too early?

> [!NOTE]
> ***Your Answer***
>
> _(Write your answer here.)_

## B. This phase of the application

4. Why does Phase 1 ship a vertical slice that does almost no greenhouse business logic? What does “empty but running” prove that a folder of unimplemented classes would not?

> [!NOTE]
> ***Your Answer***
>
> _(Write your answer here.)_

5. List the four backend layer packages used in this course (`domain`, `application`, `infrastructure`, `interfaces/api`). For each, state what belongs there and give one example of something that must **not** live in `domain`.

> [!NOTE]
> ***Your Answer***
>
> _(Write your answer here.)_

6. What does `GET /health` return, and why does it check the database instead of only reporting that the HTTP process is up? Why is API documentation served at `/scalar`, and why is `/docs` disabled?

> [!NOTE]
> ***Your Answer***
>
> _(Write your answer here.)_

7. Phase 1 requires Alembic (or equivalent) with a **baseline** migration and **no** business tables such as `devices`. Why introduce the migration toolchain before any product schema? What would go wrong if you created tables by hand in Postgres and only added migrations later?

> [!NOTE]
> ***Your Answer***
>
> _(Write your answer here.)_

## C. Compare, contrast, and scenarios

8. Explain **dependency direction** in this skeleton: which layers may import which? Why must domain code not import FastAPI, SQLAlchemy, or Pydantic models used as HTTP schemas?

> [!NOTE]
> ***Your Answer***
>
> _(Write your answer here.)_

9. The frontend cannot show a healthy badge. A classmate blames “the patterns.” What should you check first (stack, CORS/proxy, health JSON), and why is that a Phase 1 concern rather than a later pattern concern?

> [!NOTE]
> ***Your Answer***
>
> _(Write your answer here.)_

10. Course completion is at **Phase 12**, not Phase 1. What is still missing after a successful skeleton, and how do later phases add behaviour without rewriting the foundations you laid here?

> [!NOTE]
> ***Your Answer***
>
> _(Write your answer here.)_

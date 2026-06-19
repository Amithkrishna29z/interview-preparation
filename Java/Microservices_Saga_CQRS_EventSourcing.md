# Microservices: Saga, CQRS & Event Sourcing — Awareness Notes

> **Scope note (junior job prep):** These are **advanced microservices patterns deferred for later study**. This file is trimmed to a one-line-per-concept awareness section — enough to recognize the terms in an interview. Junior-level microservices *awareness* (monolith vs microservices, API gateway, service discovery, circuit breaker) is in **`System_Design_Microservices_Interview_Questions.md`**. The full deep-dive remains in git history.

---

## The Problem They Solve

In microservices, each service owns its own database, so you **can't use one ACID transaction** across services. These patterns are different ways to keep data consistent without a single distributed transaction.

## Recognize These Terms

- **Saga pattern** — a multi-step business transaction split into local transactions across services. If a step fails, earlier steps are undone with **compensating transactions** (e.g., "release inventory"). Two styles: **choreography** (services react to each other's events, no coordinator) and **orchestration** (a central coordinator drives the steps).
- **CQRS** (Command Query Responsibility Segregation) — separate the **write** model (commands) from the **read** model (queries), often with different data stores optimized for each.
- **Event Sourcing** — instead of storing current state, store the **sequence of events** that led to it; current state is rebuilt by replaying events. Read models ("projections") are derived from the event stream.
- **Outbox pattern** — reliably publish events by writing them to an "outbox" table in the *same* DB transaction as the business data, then relaying them to a broker (avoids the dual-write problem).
- **BASE vs ACID** — distributed systems often trade ACID's strong consistency for BASE (Basically Available, Soft state, Eventually consistent).

> **Interview soundbite:** "I know Saga handles distributed transactions across microservices with compensating actions, and that CQRS separates reads from writes while event sourcing stores events as the source of truth. I haven't built these in production — my hands-on experience is single-service Spring Boot apps."

---

*Trimmed to awareness level for junior job prep. Restore the full Saga/CQRS/Event-Sourcing deep-dive from version control when you're ready to study it.*

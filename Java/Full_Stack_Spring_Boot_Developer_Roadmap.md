# Full-Stack Spring Boot Developer — Roadmap & Index

A guided learning path through this repository for landing a **junior → mid full-stack Spring Boot developer** role. Every item links to a study guide in this folder.

## How to use this

- Follow the phases **top to bottom** — each builds on the previous one.
- Priority markers tell you where to spend your time:
  - 🎯 **Must-know** — core to a junior Spring Boot interview. Do not skip.
  - ⭐ **Should-know** — common follow-ups; learn after the must-knows.
  - 🔭 **Awareness** — senior/advanced; know it *exists* and the one-line "why". (These guides were trimmed to awareness depth on purpose.)
- Tick the boxes as you go. Revisit [`MASTER_CHEAT_SHEET.md`](MASTER_CHEAT_SHEET.md) the week before an interview.

---

## Phase 1 — Core Java Foundations 🎯

The language itself. Most junior interviews are won or lost here.

- [ ] 🎯 [Java Data Types](Java_Data_Types_Notes.md)
- [ ] 🎯 [Collections Framework — Internal Working](Java_Collections_Framework_Internal_Working.md)
- [ ] 🎯 [Exception Handling](Java_Exception_Handling_Guide.md)
- [ ] 🎯 [Generics & Wildcards](Java_Generics_And_Wildcards_Guide.md)
- [ ] 🎯 [`equals()` & `hashCode()`](Java_Equals_And_HashCode_Guide.md)
- [ ] 🎯 [Enums](Java_Enums_Guide.md)
- [ ] 🎯 [Streams & Lambdas](Java_Streams_And_Lambdas_Guide.md)
- [ ] ⭐ [Multithreading & Concurrency](Java_Multithreading_Concurrency_Guide.md)
- [ ] ⭐ [Modern Java Features (9–21)](Java_Modern_Features_9_to_21.md)
- [ ] ⭐ [IO & NIO](Java_IO_And_NIO_Guide.md)
- [ ] ⭐ [Reflection & Annotations](Java_Reflection_And_Annotations_Guide.md)
- [ ] ⭐ [Serialization](Java_Serialization_Guide.md)
- [ ] ⭐ [JVM Internals, Memory & GC](JVM_Internals_Memory_Management_GC.md)
- [ ] 🎯 [Java Developer Interview Questions](Java_Developer_Interview_Questions.md) — broad Q&A drill

---

## Phase 2 — Spring & Spring Boot 🎯

The framework you'll be hired to use.

- [ ] 🎯 [Spring Framework Study Guide](Java_Spring_Framework_Study_Guide.md) — IoC, DI, the container
- [ ] 🎯 [Spring Boot — Junior Developer Interview Questions](Spring_Boot_Junior_Developer_Interview_Questions.md)
- [ ] 🎯 [AOP & Bean Lifecycle](Spring_AOP_And_Bean_Lifecycle.md)
- [ ] 🎯 [Bean Validation](Spring_Bean_Validation_Guide.md)
- [ ] ⭐ [Embedded Tomcat in Spring Boot](Tomcat_In_Spring_Boot_Guide.md)

---

## Phase 3 — Persistence & Databases 🎯

How your app stores and queries data.

- [ ] 🎯 [JPA & Hibernate](JPA_Hibernate_Interview_Questions.md)
- [ ] 🎯 [Database Concepts](Database_Concepts_Interview_Questions.md) — ACID, transactions, isolation, normalization
- [ ] 🎯 Pick **one** relational DB and go deep:
  - [ ] [PostgreSQL](PostgreSQL_Interview_Questions.md)
  - [ ] [MySQL](MySQL_Interview_Questions.md)
- [ ] ⭐ [MongoDB](MongoDB_Interview_Questions.md) — document/NoSQL basics
- [ ] ⭐ [Redis](Redis_Interview_Questions.md) — caching
- [ ] ⭐ [SQL — Advanced Window Functions](SQL_Advanced_Window_Functions.md)
- [ ] 🔭 [Database Schema Design Patterns](Database_Schema_Design_Patterns.md)

---

## Phase 4 — REST APIs & Security 🎯

Exposing and protecting your backend.

- [ ] 🎯 [REST API & HTTP](REST_API_HTTP_Interview_Questions.md)
- [ ] 🎯 [Spring Security & JWT](Spring_Security_JWT_Interview_Questions.md)
- [ ] 🔭 [OAuth2, OIDC & Advanced Security](Spring_OAuth2_OIDC_Security.md)

---

## Phase 5 — Testing 🎯

Junior devs are expected to write tests.

- [ ] 🎯 [Testing — JUnit & Mockito](Java_Testing_JUnit_Mockito_Guide.md)
- [ ] 🔭 [Advanced Testing — TestContainers & Contract](Testing_Advanced_TestContainers_And_Contract.md)

---

## Phase 6 — Frontend (the "full-stack" half) ⭐

Enough frontend to build and reason about a UI on top of your APIs.

- [ ] 🎯 [HTML](HTML_Interview_Questions.md)
- [ ] 🎯 [CSS](CSS_Interview_Study_Guide.md)
- [ ] 🎯 [JavaScript](JavaScript_Interview_Questions.md)
- [ ] ⭐ [TypeScript](typescript.md)
- [ ] 🎯 [React](React_Interview_Questions.md)
- [ ] ⭐ [Next.js](NextJS_Interview_Questions.md)

---

## Phase 7 — DevOps, Cloud & Tooling ⭐

Getting your app built, shipped, and run.

- [ ] 🎯 [Git, Docker & DevOps Interview Questions](Git_Docker_DevOps_Interview_Questions.md)
- [ ] 🎯 [Docker Concepts](Docker_Concepts_Study_Guide.md)
- [ ] ⭐ [DevOps Core Concepts](DevOps_Core_Concepts.md)
- [ ] ⭐ [CI/CD Pipelines](CI_CD_Pipelines_Deep_Dive.md)
- [ ] ⭐ [Kubernetes — Learning Guide](Kubernetes_Learning_Guide.md)
- [ ] ⭐ [Kubernetes — Interview Questions](Kubernetes_Interview_Questions.md)
- [ ] ⭐ [AWS Cloud](AWS_Cloud_Interview_Questions.md)
- [ ] 🔭 [Terraform on AWS](TERRAFORM_AWS_STUDY_GUIDE.md)
- [ ] ⭐ [Networking — Concepts for Cloud/DevOps](Networking_Concepts_Cloud_DevOps.md)
- [ ] ⭐ [Networking — Interview Questions](Networking_Interview_Questions.md)
- [ ] 🔭 [Observability — Tracing, Metrics, Logging](Observability_Tracing_Metrics_Logging.md)
- [ ] 🔭 [Vim / Nano Editors](Vim_Nano_Editor_Study_Guide.md)

---

## Phase 8 — Messaging, Reactive & Microservices 🔭

Common in real codebases; mostly awareness-level for a junior.

- [ ] ⭐ [Kafka](Kafka_Interview_Questions.md)
- [ ] 🔭 [Kafka vs RabbitMQ](Kafka_RabbitMQ_Interview_Questions.md)
- [ ] 🔭 [Spring WebFlux — Reactive](Spring_WebFlux_Reactive.md)
- [ ] 🔭 [System Design — Microservices](System_Design_Microservices_Interview_Questions.md)
- [ ] 🔭 [Microservices — Saga, CQRS, Event Sourcing](Microservices_Saga_CQRS_EventSourcing.md)
- [ ] 🔭 [Distributed Systems — Core Concepts](Distributed_Systems_Core_Concepts_Study_Guide.md)

---

## Phase 9 — Design, Architecture & Engineering Craft ⭐

What separates a good junior from a forgettable one.

- [ ] 🎯 [GoF Design Patterns](GoF_Design_Patterns_Complete.md)
- [ ] 🎯 [Software Engineering Principles](Software_Engineering_Principles_Interview_Questions.md) — SOLID, etc.
- [ ] ⭐ [Clean Architecture](Clean_Architecture_Study_Guide.md)
- [ ] ⭐ [Architecture Topics for Java Developers](Architecture_Topics_For_Java_Developers.md)
- [ ] 🔭 [Software Architect Study Guide](Software_Architect_Study_Guide.md)
- [ ] 🔭 [Backend Engineering Mastery Roadmap](Backend_Engineering_Mastery_Roadmap.md)
- [ ] 🔭 [World-Class Software Engineer Roadmap](World_Class_Software_Engineer_Roadmap.md)

---

## Phase 10 — Problem Solving (DSA) ⭐

For the coding round.

- [ ] 🎯 [DSA Interview Questions](DSA_Interview_Questions.md)
- [ ] ⭐ [Dynamic Programming](DSA_Dynamic_Programming.md)

---

## Phase 11 — Behavioral & Final Prep 🎯

- [ ] 🎯 [HR & Behavioral Interview Questions](HR_Behavioral_Interview_Questions.md)
- [ ] 🎯 [Master Cheat Sheet](MASTER_CHEAT_SHEET.md) — final-week revision

---

## Suggested study order at a glance

```
Core Java  ─►  Spring Boot  ─►  Persistence  ─►  REST + Security  ─►  Testing
   (P1)          (P2)            (P3)             (P4)               (P5)
                                                    │
                                                    ▼
                              Frontend (P6)  ─►  DevOps/Cloud (P7)
                                                    │
                                                    ▼
                  Messaging/Microservices (P8)  +  Design/Architecture (P9)
                                                    │
                                                    ▼
                                   DSA (P10)  ─►  Behavioral (P11)
```

**Minimum viable junior backend path (if short on time):** P1 → P2 → P3 → P4 → P5 → P10 → P11, then layer in P6 frontend for the "full-stack" label.

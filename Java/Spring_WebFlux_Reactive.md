# Spring WebFlux & Reactive Programming — Awareness Notes

> **Scope note (junior job prep):** Reactive programming is an **advanced topic deferred for later study**. This file has been trimmed to a short awareness section — enough to not be blindsided in an interview. The full deep-dive (Mono/Flux operators, backpressure, Schedulers, R2DBC, reactive security/testing, hot/cold publishers) was removed to keep the study set focused on junior full-stack essentials. For day-to-day junior work, you'll almost always use **Spring MVC**, not WebFlux.

---

## What It Is (the 30-second version)

- **Spring MVC** (what you should know cold) uses the blocking Servlet API: each HTTP request holds one Tomcat thread for its whole duration, including time spent waiting on the DB or other services.
- **Spring WebFlux** is the **non-blocking, reactive** alternative. A small fixed pool of event-loop threads (Netty) can serve thousands of concurrent I/O-bound requests because a thread starts I/O and immediately moves on instead of waiting.
- Return types change from `User` / `ResponseEntity<User>` to `Mono<T>` (0–1 items) and `Flux<T>` (0–N items), built on **Project Reactor**.

## MVC vs WebFlux (the one comparison worth knowing)

| Aspect | Spring MVC | Spring WebFlux |
|---|---|---|
| I/O model | Blocking (Servlet API) | Non-blocking (Reactive Streams) |
| Default server | Tomcat (thread-per-request) | Netty (event loop) |
| Return types | `String`, `ResponseEntity<T>` | `Mono<T>`, `Flux<T>` |
| Database | JDBC / JPA | R2DBC |
| HTTP client | `RestTemplate` / `RestClient` | `WebClient` |
| Learning curve | Low | High |

## When You'd Use It (and when you wouldn't)

- **Use WebFlux** for true streaming (Server-Sent Events, WebSocket), very high-concurrency I/O-bound services, or when the whole stack is already reactive.
- **Stay on Spring MVC** for standard CRUD apps, when using JDBC/Hibernate, or when the team isn't reactive-experienced. On **Java 21+, virtual threads** give MVC much of WebFlux's throughput for CRUD with far simpler code.

> **Interview soundbite:** "I understand WebFlux conceptually — it's non-blocking and good for high-concurrency streaming — but I've focused on Spring MVC, which fits most CRUD apps. Mixing blocking JDBC into WebFlux gives all the complexity with none of the benefit."

---

*Trimmed to awareness level for junior job prep. Restore the full reactive deep-dive from version control when you're ready to study it.*

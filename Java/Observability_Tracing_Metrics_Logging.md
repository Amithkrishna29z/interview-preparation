# Observability: Logs, Metrics & Traces — Awareness Notes

> **Scope note (junior job prep):** Deep observability (Micrometer, Prometheus, Grafana, Zipkin/Jaeger, OpenTelemetry, ELK/Loki, SLI/SLO/SLA, distributed tracing) is **deferred for later study** — it's an SRE/platform concern beyond junior scope. This file is trimmed to the three-pillars awareness. Practical **application logging** in Spring Boot is kept in **`Spring_Boot_Logging_Guide.md`**. The full deep-dive remains in git history.

---

## The Three Pillars (recognize these)

- **Logs** — timestamped text records of events ("what happened"). In Spring Boot: SLF4J + Logback; log at the right level (ERROR/WARN/INFO/DEBUG) and include context.
- **Metrics** — numeric measurements over time ("how much / how many"): request rate, latency, error rate, memory. Spring Boot exposes these via **Actuator** (`/actuator/metrics`, `/actuator/health`), often scraped by **Prometheus** and graphed in **Grafana**.
- **Traces** — follow a single request as it hops across services ("where did the time go"). Tools: Zipkin, Jaeger, OpenTelemetry.

**Monitoring vs observability:** monitoring tells you *that* something is wrong (dashboards/alerts on known metrics); observability lets you ask *why* by exploring logs, metrics, and traces together.

The one piece a junior does touch: **Spring Boot Actuator** — add `spring-boot-starter-actuator`, and `/actuator/health` and `/actuator/metrics` come for free.

> **Interview soundbite:** "Observability has three pillars — logs, metrics, traces. I know Spring Boot Actuator exposes health and metrics endpoints, and that Prometheus/Grafana and tracing tools build on that. Hands-on I've used logging; the full observability stack is something I'd grow into."

---

*Trimmed to awareness level for junior job prep. Restore the full observability deep-dive from version control when you're ready to study it.*

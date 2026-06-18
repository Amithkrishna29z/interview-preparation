# Spring Boot Logging with SLF4J + Logback & Actuator Basics

> **How to use this guide (junior dev):** The single most asked question here is **"How do you log in Spring Boot and why not use `System.out.println`?"** — start with [section 2](#2-the-logging-stack-slf4j--logback) and be able to explain SLF4J vs Logback out loud. After that, cover log levels and parameterized logging — you will use both every single day. Actuator health checks come up in every Kubernetes/cloud interview.

---

## Table of Contents

1. [Overview](#1-overview)
2. [The Logging Stack: SLF4J + Logback](#2-the-logging-stack-slf4j--logback)
3. [Creating a Logger](#3-creating-a-logger)
4. [Log Levels](#4-log-levels)
5. [Parameterized Logging — The Right Way](#5-parameterized-logging--the-right-way)
6. [Configuring Logging in application.yml](#6-configuring-logging-in-applicationyml)
7. [Per-Environment Config with logback-spring.xml](#7-per-environment-config-with-logback-springxml)
8. [Best Practices & Security](#8-best-practices--security)
9. [Spring Boot Actuator Basics](#9-spring-boot-actuator-basics)
10. [Common Mistakes & Pitfalls](#10-common-mistakes--pitfalls)
11. [Common Interview Questions](#11-common-interview-questions)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. Overview

Every production application needs structured, leveled logging. `System.out.println` has no levels, no timestamps, and no filtering — it is not acceptable in production code. Spring Boot ships with a full logging stack out of the box via `spring-boot-starter-logging` (bundled in every standard starter).

For advanced topics — distributed trace IDs, MDC correlation, Micrometer metrics, Prometheus/Grafana — see `Observability_Tracing_Metrics_Logging.md`.

---

## 2. The Logging Stack: SLF4J + Logback

Spring Boot's logging stack has two layers that you must understand separately:

| Layer | What it is | Your role |
|---|---|---|
| **SLF4J** (`org.slf4j`) | Facade / API — defines the `Logger` interface | **Code against this always** |
| **Logback** (`ch.qos.logback`) | Default implementation — does the actual writing | Configured via XML/yml; never imported in app code |

**The rule:** Import only `org.slf4j.Logger` and `org.slf4j.LoggerFactory` in your Java classes — never import Logback directly. This keeps your code decoupled from the implementation; swapping to Log4j2 requires one dependency change, no class changes. Spring Boot auto-configures Logback via `spring-boot-starter-logging` (bundled in every standard starter).

---

## 3. Creating a Logger

### Option A — Manual (plain Java, always works)

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class OrderService {

    // One logger per class — use the class itself as the name
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public void createOrder(Order order) {
        log.info("Creating order for user {}", order.getUserId());
    }
}
```

### Option B — Lombok `@Slf4j` (preferred in teams that use Lombok)

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j          // Lombok generates: private static final Logger log = LoggerFactory.getLogger(OrderService.class);
@Service
public class OrderService {

    public void createOrder(Order order) {
        log.info("Creating order for user {}", order.getUserId());
        // 'log' is the generated field name — use it directly
    }
}
```

Both produce the same SLF4J `Logger` — `@Slf4j` just saves the boilerplate.

---

## 4. Log Levels

Spring Boot supports five log levels, from most severe to least:

| Level | When to use | Default visible? |
|---|---|---|
| `ERROR` | Unrecoverable errors — exceptions that break a flow, failed external calls | Yes |
| `WARN` | Unexpected but recoverable — deprecated usage, retried operations, config oddities | Yes |
| `INFO` | Normal business events — order created, user logged in, service started | Yes |
| `DEBUG` | Developer detail — method entry/exit, SQL params, intermediate values | No (dev only) |
| `TRACE` | Very fine-grained — loop iterations, raw HTTP bodies | No (rarely used) |

**Default level is INFO** — DEBUG and TRACE are suppressed unless explicitly enabled. Use `INFO` and above in production; enable `DEBUG` for your own package only in local dev (never the whole app).

---

## 5. Parameterized Logging — The Right Way

SLF4J uses `{}` placeholders. The string is only built if the log level is active — faster and safer than concatenation.

### Good — parameterized (always do this)

```java
// String built only if INFO is enabled
log.info("Order {} created for user {}", orderId, userId);

// Multiple params — same pattern
log.debug("Processing item {} of {} in batch {}", current, total, batchId);

// Exception with context — pass the Throwable as the last arg (no placeholder needed)
log.error("Failed to process order {}", orderId, ex);
```

### Bad — string concatenation (never do this)

```java
// BAD: string is always built, even when DEBUG is off — wastes CPU
log.debug("Processing item " + current + " of " + total);

// BAD: toString() called even if level is suppressed
log.info("Order: " + order.toString());

// BAD: Using System.out — no levels, no timestamps, no log routing
System.out.println("Order created: " + orderId);
```

**Why it matters:** In high-throughput code, unnecessary string concatenation in suppressed log statements adds measurable GC pressure.

---

## 6. Configuring Logging in application.yml

For most junior projects, `application.yml` is all you need.

```yaml
logging:
  # Root level applies to everything not explicitly overridden
  level:
    root: INFO                          # production default

    # Your package: DEBUG in local dev, INFO in prod (override per environment)
    com.myapp: DEBUG

    # Third-party packages — raise to WARN to reduce noise
    org.hibernate.SQL: DEBUG            # shows generated SQL
    org.springframework.web: WARN

  # Write logs to a file (optional — in containers, stdout is usually enough)
  file:
    name: logs/myapp.log               # creates the file; rotates automatically

  # Customize the console pattern (optional)
  pattern:
    console: "%d{HH:mm:ss} %-5level [%thread] %logger{36} - %msg%n"
    # %d = timestamp, %-5level = padded level, %logger{36} = short class name
```

> Logging to a file is optional in containerized apps (Docker/Kubernetes) because container runtimes capture stdout. Only add `logging.file.name` if you need a persistent local file.

---

## 7. Per-Environment Config with logback-spring.xml

For more control — JSON output in prod, pretty console in dev — create `src/main/resources/logback-spring.xml`. The `<springProfile>` tag lets you switch config based on the active Spring profile.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- DEV: readable colored console + DEBUG for your package -->
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss} %highlight(%-5level) [%cyan(%logger{36})] - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO"><appender-ref ref="CONSOLE"/></root>
        <logger name="com.myapp" level="DEBUG"/>
    </springProfile>

    <!-- PROD: structured JSON output (ELK / Loki parse this) -->
    <springProfile name="prod">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
        </appender>
        <root level="INFO"><appender-ref ref="CONSOLE"/></root>
    </springProfile>
</configuration>
```

> `logback-spring.xml` (with `-spring`) gives you `<springProfile>` support. Plain `logback.xml` does not — Spring Boot cannot inject profile awareness into it.

For JSON output in prod, add `logstash-logback-encoder` (version 7.4+) to `pom.xml` — see Maven Central for the latest version.

---

## 8. Best Practices & Security

### Never log sensitive data

```java
// BAD — passwords, tokens, card numbers must never appear in logs
log.info("User login: username={}, password={}", username, password);
log.debug("Calling payment API with token={}", apiToken);

// GOOD — log only what you need for debugging; mask or omit secrets
log.info("User login attempt for username={}", username);
log.debug("Calling payment API (token present: {})", apiToken != null);
```

Logs flow into log aggregation systems (ELK, Splunk, Loki) accessible to many people — a leaked secret in a log line is a security incident.

### Correlation / Trace IDs

In microservices, add a trace ID to every log line so you can follow a single request across services. Spring Boot's Micrometer Tracing auto-populates `traceId` and `spanId` into the MDC when configured — they appear in every log line for that request automatically. See `Observability_Tracing_Metrics_Logging.md` for setup.

---

## 9. Spring Boot Actuator Basics

Actuator adds production-ready HTTP endpoints to your app — health checks, metrics, and runtime log level changes — with no extra code.

### Add the dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Expose endpoints in application.yml

By default only `/actuator/health` and `/actuator/info` are exposed over HTTP. Expose more explicitly:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, loggers   # never use "*" in production
  endpoint:
    health:
      show-details: when-authorized               # hide internals from anonymous callers
```

### Key endpoints

| Endpoint | What it returns | Junior use case |
|---|---|---|
| `GET /actuator/health` | `{"status":"UP"}` or `DOWN` with component details | Kubernetes liveness/readiness probes — see `Kubernetes_Learning_Guide.md` |
| `GET /actuator/info` | App name, version, git commit (configured in yml) | Shows build info in dashboards |
| `GET /actuator/metrics` | List of metric names (timers, counters, memory) | Entry point; drill in via `/actuator/metrics/{name}` |
| `GET /actuator/loggers` | All logger names and their current levels | Inspect logging config at runtime |
| `POST /actuator/loggers/{name}` | Change a logger's level at runtime — no restart | Temporarily enable DEBUG in prod for one package |

### Change log level at runtime (no restart)

```bash
# POST to /actuator/loggers/{package} — reverts on restart
curl -X POST http://localhost:8080/actuator/loggers/com.myapp \
  -H "Content-Type: application/json" -d '{"configuredLevel": "DEBUG"}'
```

### Securing Actuator + Kubernetes health checks

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info    # whitelist — never use "*" in prod
  endpoint:
    health:
      show-details: when-authorized
  server:
    port: 8081    # optional: separate internal port, firewalled from public traffic
  health:
    livenessstate:
      enabled: true              # /actuator/health/liveness  — K8s liveness probe
    readinessstate:
      enabled: true              # /actuator/health/readiness — K8s readiness probe
```

> `/actuator/env` and `/actuator/heapdump` can leak secrets — never expose them publicly. For full Kubernetes probe config see `Kubernetes_Learning_Guide.md`; for Micrometer metrics + Prometheus see `Observability_Tracing_Metrics_Logging.md`.

---

## 10. Common Mistakes & Pitfalls

| Mistake | Why it's wrong | Fix |
|---|---|---|
| `import ch.qos.logback.*` in app code | Couples your code to Logback; breaks if you switch implementations | Import `org.slf4j.Logger` and `LoggerFactory` only |
| `"Value: " + obj` in log statements | String built even when level is off — wasted CPU | Use `log.debug("Value: {}", obj)` |
| Logging passwords, tokens, PII | Security incident — logs are widely readable | Log only non-sensitive identifiers; mask secrets |
| Leaving `DEBUG` on in production | Log volume explodes; sensitive internals exposed | Set `logging.level.root=INFO` in prod profile |
| `management.endpoints.web.exposure.include=*` in prod | Exposes heap dumps, env vars with secrets, thread dumps | Whitelist only: `health, info` — add others behind auth |
| Using `System.out.println` | No levels, no routing, no filtering | Always use SLF4J `log.*` methods |
| One logger for the whole app (wrong class) | Loses the class context; filtering by package breaks | `LoggerFactory.getLogger(ThisClass.class)` per class |

---

## 11. Common Interview Questions

**Q: What is SLF4J and why does Spring Boot use it instead of Logback directly?**
SLF4J is a logging facade — it defines the API (`Logger`, `LoggerFactory`) but does no actual logging itself. Spring Boot defaults to Logback as the implementation behind SLF4J. Coding against SLF4J means your application code is decoupled from the logging implementation, so the implementation can be swapped (e.g., to Log4j2) without changing any application classes.

**Q: What is the default log level in Spring Boot, and how do you change it?**
The default root log level is INFO, which means DEBUG and TRACE are suppressed. You change it per-package in `application.yml` with `logging.level.com.myapp=DEBUG` for local development, or by POSTing to `/actuator/loggers/{name}` at runtime without restarting.

**Q: Why use `log.info("User {}", userId)` instead of `log.info("User " + userId)`?**
With placeholders, SLF4J only builds the string if the INFO level is active. String concatenation always evaluates — even if DEBUG is suppressed — wasting CPU on string allocation and garbage collection in high-throughput services. Placeholders also eliminate accidental NPE if `toString()` throws.

**Q: What is Spring Boot Actuator and which endpoints matter for a junior developer?**
Actuator adds production-ready HTTP endpoints with no extra code. The most important are `/health` (used by Kubernetes liveness/readiness probes), `/loggers` (change log levels at runtime), and `/metrics` (application and JVM metrics). In production, only expose the endpoints you need and secure them — never expose all endpoints via the wildcard.

**Q: How do you prevent sensitive data from appearing in logs?**
Never pass passwords, tokens, or personal data as log arguments — log only non-sensitive identifiers (user IDs, order IDs). Review log statements in code review the same way you review SQL queries.

---

## 12. Quick Reference Cheat Sheet

```
LOGGING STACK
  SLF4J  = API facade   → import org.slf4j.Logger / LoggerFactory
  Logback = default impl → configured via application.yml / logback-spring.xml
  Lombok  = @Slf4j       → generates: private static final Logger log = ...

LOG LEVELS (most → least severe)
  ERROR  WARN  INFO  DEBUG  TRACE
  Default: INFO (DEBUG/TRACE suppressed)
  Prod:    root=INFO   |   Dev: com.myapp=DEBUG

PARAMETERIZED LOGGING
  log.info("Order {} for user {}", orderId, userId);   ✓
  log.info("Order " + orderId + " for " + userId);     ✗

CHANGE LEVEL AT RUNTIME (no restart)
  POST /actuator/loggers/com.myapp
  Body: {"configuredLevel": "DEBUG"}

application.yml
  logging.level.root: INFO
  logging.level.com.myapp: DEBUG
  logging.file.name: logs/app.log      # optional in containers

logback-spring.xml
  <springProfile name="dev"> ... </springProfile>    # console
  <springProfile name="prod"> ... </springProfile>   # JSON/file

ACTUATOR KEY ENDPOINTS
  /actuator/health    → UP/DOWN — used by K8s probes
  /actuator/info      → app version / build info
  /actuator/metrics   → JVM + app metrics
  /actuator/loggers   → inspect + change log levels

ACTUATOR SECURITY
  expose:
    include: health, info          # whitelist only what you need
  management.server.port: 8081     # separate internal port

NEVER DO
  - import ch.qos.logback.* in app code
  - log passwords / tokens / PII
  - leave DEBUG on in prod
  - expose all actuator endpoints (include: "*") unsecured
  - System.out.println in production code

CROSS-REFERENCES
  Metrics, Prometheus, Grafana, MDC/trace IDs → Observability_Tracing_Metrics_Logging.md
  Kubernetes liveness/readiness probes        → Kubernetes_Learning_Guide.md
  Spring profiles (dev/prod)                  → Spring_Boot_Configuration_And_Profiles_Guide.md
```

---

*Last Updated: 2026-06-18*

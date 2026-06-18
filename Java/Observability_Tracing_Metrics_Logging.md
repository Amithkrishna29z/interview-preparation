# Observability: Tracing, Metrics, and Logging — Full Stack Java Developer Interview Guide

## Overview

Observability is the ability to understand what is happening inside a system by examining its external outputs. For microservices, it is non-negotiable — without it, debugging production issues becomes guesswork. This guide covers the three pillars (logs, metrics, traces), the tools (Micrometer, Prometheus, Grafana, Zipkin, Jaeger, ELK, OpenTelemetry), and Spring Boot integration.

---

## Table of Contents

1. [The Three Pillars of Observability](#1-the-three-pillars-of-observability)
2. [Logs — Deep Dive](#2-logs--deep-dive)
3. [Metrics — Deep Dive](#3-metrics--deep-dive)
4. [Distributed Tracing — Deep Dive](#4-distributed-tracing--deep-dive)
5. [OpenTelemetry](#5-opentelemetry)
6. [Spring Boot Actuator](#6-spring-boot-actuator)
7. [Prometheus and Grafana](#7-prometheus-and-grafana)
8. [Log Aggregation: ELK, EFK, Loki](#8-log-aggregation-elk-efk-loki)
9. [Alerting and SLI/SLO/SLA](#9-alerting-and-slislosla)
10. [Debugging a Slow Request in Microservices](#10-debugging-a-slow-request-in-microservices)
11. [Interview Questions and Answers](#11-interview-questions-and-answers)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. The Three Pillars of Observability

### 1.1 Monitoring vs Observability

| Monitoring | Observability |
|---|---|
| Tells you **when** something is wrong | Tells you **why** something is wrong |
| Reacts to known failure modes | Explores unknown unknowns |
| "Is the service up?" | "Why is this specific user's request slow?" |

**Analogy:** Monitoring is a car dashboard — it alerts you when something specific goes wrong. Observability is a flight data recorder — you can reconstruct exactly what happened for any event, even ones you never anticipated.

---

### 1.2 The Three Pillars

```
┌───────────────────────────────────────────────────────────────────┐
│                         OBSERVABILITY                             │
│   ┌─────────────┐   ┌─────────────┐   ┌───────────────────────┐  │
│   │    LOGS     │   │   METRICS   │   │        TRACES         │  │
│   │ What        │   │ How many?   │   │ Where did this        │  │
│   │ happened?   │   │ How fast?   │   │ request go?           │  │
│   └─────────────┘   └─────────────┘   └───────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

| Pillar | Questions Answered | Storage | Query Style |
|---|---|---|---|
| Logs | What happened? What was the error? | Text/JSON, Elasticsearch | Full-text search |
| Metrics | Error rate? Latency? Memory usage? | Time-series DB (Prometheus) | Aggregation, rate, percentile |
| Traces | Which service is slow? What is the call chain? | Jaeger, Zipkin | Trace lookup, span analysis |

---

### 1.3 Logs

**Structured vs Unstructured:**

Unstructured: `2024-01-15 10:23:45 ERROR com.example.OrderService - Failed to process order 12345`

Structured (JSON):
```json
{
  "timestamp": "2024-01-15T10:23:45.123Z",
  "level": "ERROR",
  "message": "Failed to process order",
  "orderId": "12345",
  "userId": "john@example.com",
  "traceId": "a1b2c3d4e5f6",
  "service": "order-service"
}
```

Structured logging is machine-parseable, filterable, and correlatable via `traceId`.

**Log Levels:**

| Level | Use Case |
|---|---|
| TRACE | Method entry/exit — disabled in production |
| DEBUG | Developer diagnostics — off in production |
| INFO | Normal business events (order created, user logged in) |
| WARN | Unexpected but handled — system continues (retried, used fallback) |
| ERROR | Failure that affects a specific operation — needs investigation |
| FATAL | System-level failure, service cannot continue |

> **Interview Tip:** ERROR = something went wrong for a user. WARN = system degraded but compensated.

---

### 1.4 Metrics

**Four core meter types:**

| Type | Definition | Example |
|---|---|---|
| Counter | Monotonically increasing; never decreases | Total requests, total errors |
| Gauge | Current value; can go up or down | JVM heap in use, active connections |
| Histogram | Samples observations in buckets; exposes sum and count | Request duration distribution |
| Summary | Calculates quantiles client-side | p50, p95, p99 latency |

**The Cardinality Problem:**
```
# GOOD — low cardinality
http_requests_total{method="GET", status="200", endpoint="/api/orders"}

# BAD — one time series per user → Prometheus crashes
http_requests_total{userId="user-12345"}
```
Never use user IDs, order IDs, or any unbounded value as a metric label.

---

### 1.5 Traces

```
traceId: a1b2c3d4e5f67890

  Span A: API Gateway          [0ms ──────────────────── 350ms]
    Span B: Order Service        [5ms ────────────── 340ms]
      Span C: DB Query             [10ms ──── 80ms]
      Span D: Inventory Call        [90ms ──────────── 200ms]
        Span E: Inventory DB          [95ms ── 180ms]
      Span F: Payment Call            [210ms ──── 320ms]
```

**Span fields:** `traceId` (same across all services), `spanId`, `parentSpanId`, `name`, `startTime`, `endTime`, `tags`, `status`.

---

## 2. Logs — Deep Dive

### 2.1 Structured Logging in Spring Boot

**pom.xml:**
```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

**logback-spring.xml:**
```xml
<configuration>
    <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <customFields>{"service":"order-service","env":"${SPRING_PROFILES_ACTIVE}"}</customFields>
        </encoder>
    </appender>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %-5level %logger{36} [%X{traceId}] - %msg%n</pattern>
        </encoder>
    </appender>

    <springProfile name="prod,staging">
        <root level="INFO"><appender-ref ref="JSON_CONSOLE"/></root>
    </springProfile>
    <springProfile name="dev,default">
        <root level="DEBUG"><appender-ref ref="CONSOLE"/></root>
    </springProfile>
</configuration>
```

---

### 2.2 MDC (Mapped Diagnostic Context)

MDC is a thread-local map that Logback automatically appends to every log statement on that thread.

```java
MDC.put("requestId", UUID.randomUUID().toString());
MDC.put("userId", "user-789");
log.info("Starting order processing");  // includes requestId and userId automatically
MDC.clear(); // CRITICAL: always clear to prevent ThreadLocal leaks
```

**MDC Filter (production pattern):**
```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class MdcRequestFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        try {
            String requestId = Optional.ofNullable(httpRequest.getHeader("X-Request-ID"))
                .orElse(UUID.randomUUID().toString());
            MDC.put("requestId", requestId);
            MDC.put("httpMethod", httpRequest.getMethod());
            MDC.put("uri", httpRequest.getRequestURI());
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

When `micrometer-tracing` is on the classpath, Spring Boot auto-populates `traceId` and `spanId` into MDC for every request:
```
INFO [order-service,a1b2c3d4e5f67890,f1e2d3c4] OrderService - Order created
```

---

### 2.3 Logging Best Practices

```java
@Service
@Slf4j
public class OrderService {
    public Order createOrder(CreateOrderRequest request) {
        log.info("Creating order for user={}, itemCount={}, total={}",
            request.getUserId(), request.getItems().size(), request.getTotal());
        try {
            Order order = orderRepository.save(buildOrder(request));
            log.info("Order created successfully orderId={}, userId={}", order.getId(), request.getUserId());
            return order;
        } catch (InsufficientInventoryException e) {
            log.warn("Insufficient inventory userId={}, productId={}", request.getUserId(), e.getProductId());
            throw e;
        } catch (Exception e) {
            log.error("Unexpected error creating order userId={}", request.getUserId(), e);
            throw new OrderCreationException("Order creation failed", e);
        }
    }
}
```

**Anti-patterns:**
```java
log.error("Error occurred");                                    // BAD: no context
log.debug("Processing " + items.size() + " items for " + userId); // BAD: string concat
log.debug("Processing {} items for {}", items.size(), userId);    // GOOD: parameterized
log.info("Payment cardNumber={}", creditCard.getNumber());        // BAD: sensitive data
```

---

## 3. Metrics — Deep Dive

### 3.1 Micrometer

Micrometer is the instrumentation facade for JVM apps — the SLF4J equivalent for metrics. Write instrumentation once, switch backends (Prometheus, Datadog, CloudWatch) by changing a dependency.

---

### 3.2 Spring Boot Actuator + Prometheus Setup

**pom.xml:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

**application.yml:**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus, loggers
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
    tags:
      application: ${spring.application.name}
```

**Auto-configured metrics:**

| Metric | Description |
|---|---|
| `jvm.memory.used` | JVM heap and non-heap usage |
| `jvm.gc.pause` | GC pause durations |
| `http.server.requests` | Per-endpoint request count, duration, errors |
| `hikaricp.connections.active` | Active DB connections |
| `hikaricp.connections.pending` | Threads waiting for a DB connection |

---

### 3.3 Custom Metrics

```java
@Component
public class OrderMetrics {
    private final Counter ordersCreated;
    private final Timer orderProcessingTime;
    private final AtomicInteger activeOrders = new AtomicInteger(0);

    public OrderMetrics(MeterRegistry registry) {
        ordersCreated = Counter.builder("orders.created")
            .description("Total orders created")
            .register(registry);

        orderProcessingTime = Timer.builder("orders.processing.duration")
            .description("Order processing time")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .register(registry);

        Gauge.builder("orders.active", activeOrders, AtomicInteger::get)
            .description("Orders currently being processed")
            .register(registry);
    }

    public <T> T recordOrderProcessing(Supplier<T> orderLogic) {
        return orderProcessingTime.record(orderLogic);
    }
}
```

**`@Timed` annotation shortcut:**
```java
@Timed(value = "payment.processing.time", percentiles = {0.5, 0.95, 0.99}, histogram = true)
public PaymentResult processPayment(PaymentRequest request) { ... }
```
> `@Timed` requires a `TimedAspect` bean registered when not using the Spring Boot AOP starter.

**Two more meter types:**
- **DistributionSummary** — like Timer but for non-time values (bytes, batch size): `DistributionSummary.builder("http.request.size").baseUnit("bytes")...record(size)`.
- **LongTaskTimer** — for long-running tasks (minutes/hours); tracks tasks *currently in progress*.

---

## 4. Distributed Tracing — Deep Dive

### 4.1 How Distributed Tracing Works

```
traceId: 1a2b3c4d5e6f7890  (same across ALL services)

  [API Gateway]     spanId: 0001  parentSpanId: null
  |  0ms ──────────────────────────────────────── 420ms

  [Order Service]   spanId: 0002  parentSpanId: 0001
  |  10ms ─────────────────────────────── 410ms

  [Inventory Svc]   spanId: 0004  parentSpanId: 0002
  |  55ms ─────────────────── 200ms

  [Inventory DB]    spanId: 0005  parentSpanId: 0004
  |  57ms ──────────── 190ms   ← SLOW — root cause
```

**Context Propagation:** Service A injects trace context into HTTP headers; Service B extracts it, reuses the same `traceId`, creates a new child span. Spring Boot handles this automatically.

W3C `traceparent` header format:
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^  ^traceId (32 hex)                ^spanId (16 hex) ^flags
```

---

### 4.2 Sampling Strategies

- **Head-based sampling**: decision made at the start of the request, propagated downstream. Set in Spring Boot: `management.tracing.sampling.probability: 0.1` (10%). Simple but may drop rare errors.
- **Tail-based sampling**: decision deferred until the trace completes — keep all errors/slow traces, sample the rest. More accurate but requires buffering (OTel Collector, Grafana Tempo). An SRE/platform concern.

---

### 4.3 Spring Boot 3 + Micrometer Tracing

Spring Boot 3 replaced Spring Cloud Sleuth with Micrometer Tracing.

**pom.xml:**
```xml
<!-- Zipkin/Brave bridge -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

**application.yml:**
```yaml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% dev, 0.1 prod
spring:
  zipkin:
    base-url: http://zipkin:9411
logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

**Auto-instrumented:** `@RestController` endpoints, `RestTemplate`, `WebClient`, `@KafkaListener`, `KafkaTemplate.send()`, `@Scheduled`.

**Manual span creation:**
```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final Tracer tracer;

    public Order processOrder(CreateOrderRequest request) {
        Span span = tracer.nextSpan()
            .name("order.process")
            .tag("order.userId", request.getUserId());

        try (Tracer.SpanInScope ws = tracer.withSpan(span.start())) {
            span.event("validation.started");
            validateOrder(request);
            Order saved = orderRepository.save(buildOrder(request));
            span.tag("order.id", saved.getId().toString());
            return saved;
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

> **Note:** Across async boundaries (`@Async`, thread pools) trace context is NOT inherited by default (ThreadLocal). Micrometer 1.10+ fixes this with context-propagating executor wrappers.

---

### 4.4 Zipkin vs Jaeger

| | Zipkin | Jaeger |
|---|---|---|
| Origin | Twitter | Uber |
| OTel compatibility | Via compatibility layer | Native |
| Sampling | Fixed / per-service | Adaptive |
| UI port | 9411 | 16686 |

**Docker quick start:**
```bash
# Zipkin
docker run -d -p 9411:9411 openzipkin/zipkin

# Jaeger (all-in-one, with OTLP)
docker run -d --name jaeger -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 -p 4317:4317 jaegertracing/all-in-one:latest
```

---

## 5. OpenTelemetry

OpenTelemetry (OTel) is a vendor-neutral CNCF standard for generating, collecting, and exporting telemetry (logs, metrics, traces). It merges the older OpenTracing and OpenCensus projects.

**OTel vs Micrometer:**

| | OpenTelemetry | Micrometer |
|---|---|---|
| Scope | Traces + Metrics + Logs | Primarily Metrics + Tracing bridge |
| Ecosystem | CNCF, language-agnostic | JVM ecosystem |
| Auto-instrumentation | Java agent (zero code change) | Spring auto-configuration |

**OTel Java Agent (zero-code instrumentation):**
```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=order-service \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
     -Dotel.traces.sampler=parentbased_traceidratio \
     -Dotel.traces.sampler.arg=0.1 \
     -jar order-service.jar
```

Auto-instruments: Spring MVC/WebFlux, JDBC, Redis, Kafka, gRPC, HTTP clients, scheduled tasks — no code changes.

**OTel Collector:** A YAML pipeline (receivers → processors → exporters) that sits between your apps and backends. Point your app's OTLP endpoint at it; the platform team configures it to fan out to Jaeger, Prometheus, Loki, etc.

---

## 6. Spring Boot Actuator

### 6.1 Key Endpoints

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus, loggers  # production
  endpoint:
    health:
      show-details: when-authorized
    env:
      keys-to-sanitize: password,secret,key,token
```

| Endpoint | Description |
|---|---|
| `/actuator/health` | Health status — used by Kubernetes probes |
| `/actuator/prometheus` | Prometheus scrape endpoint |
| `/actuator/metrics/{name}` | Details for a specific metric |
| `/actuator/loggers` | View/change log levels at runtime |
| `/actuator/threaddump` | Debug thread deadlocks |
| `/actuator/heapdump` | Debug OOM issues |

**Change log level at runtime (no restart):**
```bash
curl -X POST http://localhost:8080/actuator/loggers/com.example.orderservice \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel":"DEBUG"}'
```

---

### 6.2 Health Indicators

**Custom health indicator:**
```java
@Component
public class ExternalPaymentApiHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = restTemplate.getForEntity(
                "https://api.stripe.com/v1/charges?limit=1", String.class);
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up().withDetail("paymentGateway", "Stripe").build();
            }
            return Health.down().withDetail("reason", "Unexpected status: " + response.getStatusCode()).build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

Spring Boot auto-detects any `HealthIndicator` bean and includes it in `/actuator/health`. If any component is DOWN, overall status is DOWN.

---

### 6.3 Liveness vs Readiness Probes

```
Liveness  → /actuator/health/liveness
  "Is this container alive?"
  Fail → Kubernetes KILLS and RESTARTS the pod
  Use for: deadlocks, infinite loops, corrupted state
  Do NOT include external dependency health here

Readiness → /actuator/health/readiness
  "Is this container ready for traffic?"
  Fail → Kubernetes REMOVES pod from load balancer (pod keeps running)
  Use for: DB disconnects, startup not finished, dependencies unavailable
```

**application.yml:**
```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

**Kubernetes deployment.yaml:**
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 5
  failureThreshold: 3
```

---

## 7. Prometheus and Grafana

### 7.1 Prometheus Architecture

```
order-svc:8080/actuator/prometheus  ◄── PULL (every 15s)
payment-svc:8081/actuator/prometheus ◄──            Prometheus Server
inventory-svc:8082/actuator/prometheus ◄──          (TSDB + PromQL + alert rules)
                                                        │
                                          ┌─────────────┴────────────┐
                                          ▼                          ▼
                                    Alertmanager                  Grafana
```

**prometheus.yml (static config):**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-services'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['order-service:8080', 'payment-service:8081']

rule_files:
  - "alerting_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

---

### 7.2 PromQL — Essential Queries

```promql
# Request rate (RPS)
sum(rate(http_server_requests_seconds_count{application="order-service"}[5m]))

# Error rate %
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count[5m])) * 100

# p95 latency
histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le))

# JVM heap usage %
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# DB pending connections
hikaricp_connections_pending
```

---

### 7.3 Alerting Rules and Grafana

**Alerting rule example:**
```yaml
groups:
  - name: spring-boot-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          / sum(rate(http_server_requests_seconds_count[5m])) * 100 > 5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.application }}"
```

**The Four Golden Signals (Google SRE):**

| Signal | What to Measure |
|---|---|
| **Latency** | Time to serve requests (success vs error separately) |
| **Traffic** | Requests per second |
| **Errors** | Rate of failed requests |
| **Saturation** | How full is your service (CPU %, DB pool) |

**RED Method (services):** Rate, Errors, Duration  
**USE Method (infrastructure):** Utilization, Saturation, Errors

---

## 8. Log Aggregation: ELK, EFK, Loki

### 8.1 ELK Stack

```
Application servers (JSON logs)
  └── Filebeat (ships logs)
       └── Logstash (parse, transform, enrich)
            └── Elasticsearch (store, index, search)
                 └── Kibana (visualize, query, alert)
```

A Logstash pipeline: `input` (Beats on 5044) → `filter` (parse JSON, fix timestamps, drop health-check noise) → `output` (Elasticsearch, daily indices). Platform-team work; a junior understands the flow.

---

### 8.2 EFK Stack (Kubernetes-Preferred)

EFK swaps Logstash for **Fluentd/Fluent Bit**: runs as a DaemonSet, reads container stdout, ships to Elasticsearch. Preferred in Kubernetes — far less memory than Logstash.

| | Fluentd | Logstash |
|---|---|---|
| Memory | ~40MB | ~500MB |
| Kubernetes | Native | Extra config needed |
| Best for | K8s, resource-constrained | Complex transformations |

---

### 8.3 Grafana Loki

Loki is "Prometheus but for logs" — indexes only labels (not full content), stores compressed raw logs.

| | Elasticsearch | Loki |
|---|---|---|
| Indexing | All fields | Labels only |
| Storage cost | High | Low |
| Query language | Lucene / KQL | LogQL |
| Integration | Kibana | Grafana |

```
Application → Promtail (agent) → Loki → Grafana
```

**LogQL examples:**
```logql
{app="order-service", env="production"} |= "ERROR"
{app="order-service"} | json | traceId = "a1b2c3d4e5f6"
```

---

### 8.4 Cloud Logging Solutions

**AWS CloudWatch Logs**, GCP Cloud Logging, and Azure Monitor provide managed log aggregation without running ELK/Loki yourself. They index structured JSON fields (`level`, `traceId`, `service`) for filtering and dashboards — useful when you don't want to operate logging infrastructure.

---

## 9. Alerting and SLI/SLO/SLA

**SLI** — a concrete measurement (e.g., fraction of successful requests, p99 latency).  
**SLO** — a target for your SLI (e.g., "99.9% of requests succeed over 30 days") — your internal commitment.  
**SLA** — a contractual commitment to customers with penalties. SLA ≤ SLO (internal target must be tighter).  
**Error budget** = `1 - SLO`. A 99.9% SLO → ~43 minutes downtime/month. When exhausted, teams freeze deployments.

**Alert on symptoms, not causes:**
```
BAD:  Alert: CPU > 80%  (may not affect users)
GOOD: Alert: Error rate > 1%  (users are experiencing errors)
GOOD: Alert: p99 latency > 1s  (users are waiting too long)
```

**Alertmanager** receives Prometheus firing alerts, groups/deduplicates/silences them, and routes by severity to Slack or PagerDuty. Writing the config in detail is an SRE/platform task.

---

## 10. Debugging a Slow Request in Microservices

```
1. IDENTIFY
   User reports slow checkout / alert fires for p99 > 2s.
   Get traceId from logs: grep "userId=user-789" order-service.log

2. OPEN TRACE
   Go to Jaeger/Zipkin UI → search by traceId.
   View waterfall:
     API Gateway       [0ms ───────────────────────── 4,200ms]
     Order Service     [10ms ─────────────────────── 4,180ms]
     Inventory Call    [15ms ────────────────────── 3,900ms]  ← SLOW
     Inventory DB      [20ms ──────────────────── 3,850ms]  ← ROOT CAUSE

3. ANALYZE SPAN
   Click "Inventory DB" span → tags show:
     db.statement: SELECT * FROM inventory WHERE product_id = ?
     db.rows_affected: 50000
     db.execution_time: 3800ms
   → Full table scan — missing index

4. CROSS-REFERENCE METRICS
   Grafana: HikariCP pending connections spiked? → DB contention vs isolated query.

5. FIX AND VERIFY
   Add index → query drops to 3ms.
   Monitor Grafana: p99 returns to normal. Confirm in traces.
```

**Common findings:**

| Symptom in Trace | Root Cause | Fix |
|---|---|---|
| Many DB spans | N+1 query | JOIN fetch or batch fetch |
| Single slow DB span | Missing index or bad query plan | Add index, rewrite query |
| Slow external API span | Third-party latency | Timeout, circuit breaker, cache |
| Long gap between spans | Thread pool wait or serialization | More workers, optimize serialization |
| Root span slow, children fast | CPU-bound application logic | Profile with async-profiler |

---

## 11. Interview Questions and Answers

**Q1: What are the three pillars of observability?**

Logs, Metrics, and Traces. Logs record discrete events (what happened). Metrics are aggregated numerical measurements over time (how many, how fast). Traces follow a single request end-to-end across all services (where did time go, where did it fail).

---

**Q2: What is the difference between monitoring and observability?**

Monitoring is reactive — you define thresholds in advance and alert when they breach. It handles known failure modes. Observability is proactive — the system produces enough telemetry to answer arbitrary questions, including ones you never anticipated. Monitoring asks "is this broken?"; observability asks "what is actually happening?"

---

**Q3: What is distributed tracing? What is a span?**

Distributed tracing tracks a single request as it flows through multiple services. A span is the fundamental unit — one operation (one HTTP call, one DB query). Each span has a `traceId` (shared across all services), a unique `spanId`, `parentSpanId`, a name, start/end times, and key-value tags. A trace is a tree of spans sharing the same trace ID.

---

**Q4: How does a trace ID propagate across microservices?**

Service A injects trace context into HTTP headers. The W3C standard is `traceparent: 00-{traceId}-{spanId}-{flags}`. Service B extracts these, creates a child span using the same `traceId`, and sets `parentSpanId` to what it received. In Spring Boot with Micrometer Tracing or the OTel Java agent, this is automatic. The traceId is also placed in MDC so it appears in all log statements.

---

**Q5: What is OpenTelemetry?**

A vendor-neutral CNCF framework providing a unified API and SDK for generating logs, metrics, and traces in any language. Instrument once using OTel APIs, then export to any backend (Jaeger, Prometheus, Datadog) by swapping exporters. The OTel Java agent provides zero-code auto-instrumentation — attach the JAR and it instruments Spring, JDBC, Kafka, and HTTP clients automatically.

---

**Q6: What is the difference between Zipkin and Jaeger?**

Both are distributed tracing backends. Zipkin (Twitter origin) is simpler with fewer moving parts — good for getting started. Jaeger (Uber origin) is more feature-rich, supports adaptive sampling, and has native OpenTelemetry support. For new projects, Jaeger with OTel is increasingly the standard.

---

**Q7: What is MDC? Why is it used?**

MDC (Mapped Diagnostic Context) is a thread-local map provided by SLF4J/Logback that automatically appends key-value pairs to every log statement on the current thread. Used for correlating all log lines for a single request (requestId, userId) and for including traceId/spanId (Micrometer does this automatically). Critical rule: always call `MDC.clear()` in a `finally` block to prevent ThreadLocal leaks in thread pools.

---

**Q8: What is structured logging?**

Structured logging outputs log records as machine-parseable JSON instead of plain text. Each field (timestamp, level, message, traceId, userId) is a separate JSON key. This lets aggregation systems index individual fields, run filtered queries, and build dashboards. In Spring Boot, use `logstash-logback-encoder` with `LogstashEncoder`.

---

**Q9: What is Micrometer?**

The instrumentation facade for JVM apps — the SLF4J equivalent for metrics. Write instrumentation once using `MeterRegistry` API (Counter, Gauge, Timer), then switch backends by adding a different dependency. Spring Boot auto-configures Micrometer and provides dozens of built-in metrics (JVM, Tomcat, HikariCP). Also serves as the API layer for Micrometer Tracing in Spring Boot 3+.

---

**Q10: What are the meter types in Micrometer? Counter vs Gauge?**

Counter: monotonically increasing — counts events (total orders, total errors). Gauge: current snapshot value that can go up or down (JVM heap, active connections). Timer: measures count and duration (HTTP response times). DistributionSummary: like Timer but for arbitrary units (bytes, items). LongTaskTimer: tracks long-running tasks currently in progress.

---

**Q11: What is Prometheus? How does it collect metrics?**

Prometheus is an open-source time-series database. It uses a **pull model**: scrapes `/actuator/prometheus` on your services every 15 seconds. Spring Boot exposes metrics there via `micrometer-registry-prometheus`. Prometheus stores data in its TSDB, evaluates alerting rules, and allows querying via PromQL. The pull model means Prometheus detects service downtime (scrape fails), and services don't need to know where Prometheus is.

---

**Q12: What is PromQL? Give a query for error rate.**

PromQL is the Prometheus query language. Key functions: `rate()` (per-second rate of a counter), `sum()`, `histogram_quantile()` (percentiles from histogram data).

```promql
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count[5m])) * 100
```

---

**Q13: What is the RED method?**

A methodology for monitoring microservices: **R**ate (requests/sec), **E**rrors (error rate %), **D**uration (latency p95/p99). Contrast with USE (Utilization, Saturation, Errors) for infrastructure resources. The Four Golden Signals (Google SRE) add Saturation to RED.

---

**Q14: What is the ELK stack?**

Elasticsearch (search/storage) + Logstash (parse/transform/route) + Kibana (UI). Common flow: Application → Filebeat → Logstash → Elasticsearch → Kibana. Modern variants use Fluentd instead of Logstash (EFK stack), especially in Kubernetes.

---

**Q15: How do you add a custom health indicator in Spring Boot?**

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        try {
            boolean reachable = externalService.ping();
            return reachable ? Health.up().withDetail("externalApi", "reachable").build()
                             : Health.down().withDetail("reason", "unreachable").build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```
Spring Boot auto-detects the bean and includes it in `/actuator/health`. Any DOWN component rolls up to overall DOWN.

---

**Q16: What is the difference between liveness and readiness probes?**

**Liveness** (`/actuator/health/liveness`): "Is this pod alive?" Failure → Kubernetes kills and restarts the pod. Use for unrecoverable states (deadlocks). Do NOT include external dependency health here.

**Readiness** (`/actuator/health/readiness`): "Is this pod ready for traffic?" Failure → Kubernetes removes it from the load balancer (pod keeps running). Use for DB connectivity, startup, temporary overload. Enable both with `management.endpoint.health.probes.enabled: true`.

---

**Q17: How would you debug a slow request in a microservices system?**

1. Get the trace ID from logs filtered by userId/endpoint.
2. Open Jaeger/Zipkin UI, search by trace ID, find the slowest span in the waterfall.
3. Inspect that span's tags for the DB query, parameters, or external API call.
4. Common findings: single slow DB span → missing index; many DB spans → N+1 problem; slow external API → add timeout/circuit breaker.
5. Cross-reference Grafana: was DB pool pending? CPU spike? GC pause?
6. Reproduce, fix, verify via traces.

---

**Q18: What are SLI, SLO, and SLA?**

SLI = a concrete measurement (fraction of successful requests, p99 latency). SLO = internal target for the SLI ("99.9% of requests succeed over 30 days"). SLA = contractual commitment to customers with penalties — less strict than your SLO. Error budget = `1 - SLO`; for 99.9% SLO → ~43 min downtime/month.

---

**Q19: What is cardinality in metrics? Why is it a problem?**

Cardinality = number of unique time series from different label value combinations. Using `userId` as a label with 1 million users creates 1 million time series — Prometheus must hold each active series in memory, causing memory exhaustion. Labels must have small, bounded value sets (status codes, HTTP methods, service names). Use logs or traces for per-user data.

---

**Q20: What is head-based vs tail-based sampling?**

Head-based: trace/don't-trace decision at request start, propagated downstream via sampling flag. Simple, low overhead. May drop rare interesting failures. Configured in Spring Boot via `management.tracing.sampling.probability`.

Tail-based: decision deferred until the full trace completes — always keep errors and slow traces. More accurate but requires buffering (OTel Collector, Grafana Tempo). For most teams, head-based at 10-20% with error force-sampling is sufficient to start.

---

**Q21: How does Spring Boot 3 Micrometer Tracing differ from Spring Cloud Sleuth?**

Sleuth was the Spring Boot 2.x tracing solution (Brave-only). Spring Boot 3 replaced it with Micrometer Tracing — a vendor-neutral facade supporting both Brave (`micrometer-tracing-bridge-brave`) and OpenTelemetry (`micrometer-tracing-bridge-otel`). Migration: remove `spring-cloud-sleuth-*`, add the Brave or OTel bridge dependencies, move `spring.sleuth.*` config to `management.tracing.*`.

---

**Q22: What is the OTel Collector and why use it?**

A vendor-agnostic proxy between your apps and backends. Benefits: fan out to multiple backends without redeploying services; add/remove attributes, sample traces, filter noisy spans; buffer and retry when backends are unavailable; reduce egress costs to cloud vendors.

---

**Q23: How do you propagate trace context in Kafka messages?**

OTel and Brave inject trace context into Kafka message headers on the producer side (e.g., `traceparent`). The consumer extracts the headers and creates a child span linked to the producer's span. With Spring Kafka + Micrometer Tracing or the OTel Java agent, this is automatic — you see the full trace from the HTTP request through Kafka to the consumer.

---

**Q24: What is a "Gauge" backed by a weakly-referenced object?**

Micrometer holds a weak reference to the object a Gauge measures. If that object is garbage collected, the Gauge silently stops reporting (returns NaN). Fix: keep a strong reference in your component or back the Gauge with an `AtomicInteger`/`AtomicLong` field your component owns.

---

**Q25: What is the difference between Timer and DistributionSummary?**

Both measure value distributions. Timer measures elapsed time (nanoseconds internally, displayed as seconds/ms) — use for HTTP response times, DB query durations. DistributionSummary measures arbitrary values in any unit — use for request body sizes, batch sizes. Timer has `record(Duration)` / `record(Runnable)`; DistributionSummary has `record(double amount)`.

---

**Q26: How do you correlate logs and traces?**

Micrometer Tracing automatically places `traceId` into MDC for each request. Your logback pattern/JSON encoder reads MDC and includes `traceId` in every log line. In Grafana with both Loki (logs) and Jaeger (traces), configure "derived fields" to extract the traceId from a log line as a clickable link to the trace — so from an error log you can jump directly to the full distributed trace.

---

**Q27: What metrics would you put on a production dashboard?**

Service health (RED): RPS, error rate %, p50/p95/p99 latency by endpoint. JVM: heap %, GC pause rate, thread count. Database: HikariCP active/max connections, pending connections (alert if > 0), acquisition p99. Kafka (if used): consumer lag. Infra: CPU, memory.

Typical thresholds: error rate > 1% warn / > 5% critical; p99 > 500ms warn / > 2s critical; heap > 80% warn; DB pending > 0 for 1 min critical.

---

**Q28: Loki vs Elasticsearch for log aggregation?**

Elasticsearch indexes every field — fast on any field, expensive in storage. Loki indexes only labels (like Prometheus), stores compressed log chunks — cheap, scales easily, integrates natively with Grafana. Loki is preferred for Kubernetes-native stacks and cost-sensitive setups. Elasticsearch wins for complex analytics and compliance/audit logging requiring ad-hoc queries on arbitrary fields.

---

**Q29: How do you secure Actuator endpoints in production?**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus
  endpoint:
    health:
      show-details: when-authorized
  server:
    port: 8081          # Separate management port
    address: 127.0.0.1  # Internal only
```

Require authentication for non-health actuator endpoints via Spring Security. Expose `/actuator/prometheus` only on an internal network interface.

---

**Q30: What is context propagation across async threads?**

Trace context and MDC live in ThreadLocals — new threads don't inherit them. Solutions: (1) wrap your `ThreadPoolTaskExecutor` with Micrometer's context-propagating wrapper (1.10+); (2) for MDC, copy with `MDC.getCopyOfContextMap()` before submitting and restore inside the task; (3) with OTel, pass `Context.current()` explicitly to the async task.

---

**Q31: What is a span event vs a span tag?**

A **tag (attribute)** is static key-value metadata describing the span itself (e.g., `db.statement`, `http.status`). A **span event** is a timestamped annotation marking something that happened at a specific moment during the span (e.g., `"cache.miss"`, `"validation.completed"`). Tags describe the operation; events create a timeline within the span.

---

**Q35: What observability stack would you recommend for a new Spring Boot project?**

Metrics: Spring Boot Actuator + Micrometer → Prometheus + Grafana. Traces: Spring Boot 3 Micrometer Tracing (OTel bridge) → Jaeger or Grafana Tempo. Logs: structured JSON (logstash-logback-encoder) → Promtail → Grafana Loki. Grafana unifies all three with trace-to-log linking. Fully open-source and cloud-agnostic. If budget allows, Datadog or Honeycomb give better out-of-the-box experience with less setup.

---

## 12. Quick Reference Cheat Sheet

### Spring Boot Observability Dependencies

```xml
<!-- Actuator -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- Prometheus metrics -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<!-- Tracing with Zipkin/Brave -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
<!-- Structured logging -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

### Essential application.yml

```yaml
spring:
  application:
    name: order-service
  zipkin:
    base-url: http://zipkin:9411

management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus, loggers
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
  tracing:
    sampling:
      probability: 1.0  # 100% dev, 0.1 prod
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
    tags:
      application: ${spring.application.name}

logging:
  pattern:
    level: "%5p [${spring.application.name},%X{traceId:-},%X{spanId:-}]"
```

### Key PromQL Queries

```promql
# Request rate
sum(rate(http_server_requests_seconds_count[5m]))

# Error rate %
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count[5m])) * 100

# p99 latency
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le))

# Heap usage %
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# DB pending connections
hikaricp_connections_pending
```

### Observability Quick Comparison

| Question | Use | Tool |
|---|---|---|
| What is the error rate right now? | Metrics | Prometheus + Grafana |
| Which service is causing this slow request? | Traces | Jaeger / Zipkin |
| What exactly happened in this specific request? | Logs | ELK / Loki |
| Is this service healthy? | Health | Spring Actuator |
| When should I page someone? | Alerting | Alertmanager / Grafana |

### MDC Pattern Summary

```java
// Filter — add context
MDC.put("requestId", extractOrGenerate(request));
MDC.put("userId", getAuthenticatedUser());
// traceId and spanId auto-added by Micrometer Tracing

// Logger — automatically includes MDC
log.info("Processing order {}", orderId);

// Filter finally — always clean up
finally { MDC.clear(); }
```

### Span Lifecycle

```java
Span span = tracer.nextSpan().name("operation.name").start();
try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
    span.tag("key", "value");
    span.event("checkpoint");
    doWork();
} catch (Exception e) {
    span.error(e);
    throw e;
} finally {
    span.end();
}
```

---

*Last Updated: 2026-06-18*

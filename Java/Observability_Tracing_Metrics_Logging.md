# Observability: Tracing, Metrics, and Logging — Full Stack Java Developer Interview Guide

## Overview

Observability is the ability to understand what is happening inside a system by examining its external outputs. For microservices architectures, observability is non-negotiable — without it, debugging production issues becomes guesswork. This guide covers the three pillars (logs, metrics, traces), the tools (Micrometer, Prometheus, Grafana, Zipkin, Jaeger, ELK, OpenTelemetry), and deep Spring Boot integration.

---

## 1. The Three Pillars of Observability

### 1.1 What is Observability?

**Observability** is the ability to infer the internal state of a system solely from its external outputs (logs, metrics, traces). The term comes from control theory.

**The key distinction:**

| Monitoring | Observability |
|---|---|
| Tells you **when** something is wrong | Tells you **why** something is wrong |
| Dashboards for known failure modes | Answers questions you didn't anticipate |
| Reacts to known unknowns | Explores unknown unknowns |
| "Is the service up?" | "Why is this specific user's request slow?" |
| Alert fires when threshold breached | Arbitrary queries on system state |

**Real-world analogy:**
- Monitoring is like a car dashboard — it shows speed, temperature, fuel. It alerts you when something specific goes wrong.
- Observability is like having a flight data recorder — you can reconstruct exactly what happened and why, even for events you never anticipated.

**Why observability matters in microservices:**

In a monolith, a stack trace tells you everything. In a microservices system, a single user request may touch 10+ services across 50+ containers. A failure in Service G may have been caused by a slow response from Service B, which was waiting on a database query triggered by Service A. Without distributed tracing, finding this is nearly impossible.

```
User Request → API Gateway → Order Service → Inventory Service → DB
                                          → Payment Service → Stripe API
                                          → Notification Service → Email Provider
```

A bug in this chain could be at any hop. Observability lets you follow the entire journey.

**The debuggability principle:**
A system is observable if an engineer can answer any question about its behavior at any point in time using only the telemetry data it produces — without needing to SSH into servers, add new logging, or redeploy.

---

### 1.2 The Three Pillars

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY                                     │
│                                                                     │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────────────────┐  │
│   │    LOGS     │   │   METRICS   │   │        TRACES           │  │
│   │             │   │             │   │                         │  │
│   │ Discrete    │   │ Aggregated  │   │ End-to-end journey      │  │
│   │ events with │   │ numerical   │   │ of a single request     │  │
│   │ context     │   │ data points │   │ across all services     │  │
│   │             │   │ over time   │   │                         │  │
│   │ "What       │   │ "How many?" │   │ "Where did this         │  │
│   │  happened?" │   │ "How fast?" │   │  request go?"           │  │
│   └─────────────┘   └─────────────┘   └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

**Each pillar answers different questions:**

| Pillar | Questions Answered | Storage | Query Style |
|---|---|---|---|
| Logs | What happened? What was the error? What was the state? | Text/JSON files, Elasticsearch | Full-text search, filter |
| Metrics | How many requests? What is the error rate? How much memory? | Time-series DB (Prometheus, InfluxDB) | Aggregation, rate, percentile |
| Traces | Which service is slow? What is the call chain? How long did the DB query take? | Jaeger, Zipkin, Tempo | Trace lookup, span analysis |

---

### 1.3 Logs

**What logs capture:**
- Discrete events: a user logged in, an order was placed, an exception was thrown
- State changes: payment status changed from PENDING to COMPLETED
- Errors with full context: stack traces, input values, system state at failure time
- Audit trails: who did what, when

**Structured vs Unstructured Logging:**

Unstructured (traditional):
```
2024-01-15 10:23:45 ERROR com.example.OrderService - Failed to process order 12345 for user john@example.com: Connection timeout
```

Structured (JSON):
```json
{
  "timestamp": "2024-01-15T10:23:45.123Z",
  "level": "ERROR",
  "logger": "com.example.OrderService",
  "message": "Failed to process order",
  "orderId": "12345",
  "userId": "john@example.com",
  "errorType": "ConnectionTimeout",
  "durationMs": 5000,
  "traceId": "a1b2c3d4e5f6",
  "spanId": "f1e2d3",
  "service": "order-service",
  "env": "production"
}
```

**Why structured logging wins:**
- Machine-parseable: Elasticsearch, Splunk, CloudWatch can index every field
- Filterable: `level:ERROR AND service:order-service AND userId:john@example.com`
- Aggregatable: count errors by service, by user, by error type
- Correlatable: join logs from multiple services using traceId

**Log Levels — When to Use Each:**

| Level | Use Case | Example |
|---|---|---|
| TRACE | Extremely granular execution flow, method entry/exit. Disabled in production. | `TRACE: Entering hashPassword(), algorithm=bcrypt` |
| DEBUG | Developer diagnostics, intermediate values, algorithm steps. Off in production. | `DEBUG: Cache miss for key=product:123, fetching from DB` |
| INFO | Normal business events worth recording. Low volume, high value. | `INFO: Order 456 created by user 789, total=$99.00` |
| WARN | Unexpected but handled situation. System continues. Investigate later. | `WARN: Retry attempt 2/3 for payment gateway, latency=2000ms` |
| ERROR | Failure that affects a specific operation. Needs investigation. | `ERROR: Failed to charge card for order 456, charge declined` |
| FATAL | System-level failure, service cannot continue. Rare. | `FATAL: Database connection pool exhausted, shutting down` |

> **Interview Tip:** Interviewers often ask "what would you log at WARN vs ERROR?" The key: ERROR means something went wrong for a user/operation. WARN means something is degraded but the system compensated (retried, used fallback, returned cached data).

---

### 1.4 Metrics

**What metrics capture:**
- Aggregated numerical data over time
- How many requests per second, what is the error rate, how much memory is used
- Metrics do NOT tell you why — they tell you that something changed

**The four core meter types:**

| Type | Definition | Use Case | Example |
|---|---|---|---|
| Counter | Monotonically increasing value. Never decreases (except on restart). | Total requests, total errors, total orders | `http_requests_total = 10,042` |
| Gauge | Current value. Can go up or down. | JVM heap in use, active connections, queue depth | `jvm_memory_used_bytes = 512MB` |
| Histogram | Samples observations, counts them in configurable buckets, exposes sum and count | Request duration distribution, response size | Buckets: [0-10ms: 500, 10-50ms: 200, 50-100ms: 30] |
| Summary | Like histogram but calculates quantiles client-side. Less flexible for aggregation. | p50, p95, p99 latency | `p99 = 450ms` |

**Difference between metrics and logs:**

| Dimension | Metrics | Logs |
|---|---|---|
| Granularity | Aggregated (count, sum, rate) | Per-event detail |
| Cardinality | Low — pre-defined labels | High — arbitrary fields |
| Storage cost | Very low | High |
| Query speed | Fast (time-series DB) | Slower (full-text search) |
| Best for | "Is the system healthy?" | "What happened in this specific request?" |

**The Cardinality Problem:**

Cardinality = number of unique label value combinations for a metric.

```
# Good: low cardinality
http_requests_total{method="GET", status="200", endpoint="/api/orders"}

# BAD: high cardinality — one time series per user!
http_requests_total{method="GET", status="200", userId="user-12345"}
# If you have 1 million users, you have 1 million time series
# Prometheus will run out of memory and crash
```

> **Best Practice:** Never use user IDs, order IDs, session IDs, or any unbounded identifier as a metric label. Labels should have a bounded, small set of values (status codes, HTTP methods, service names, regions).

---

### 1.5 Traces

**What traces capture:**
- The end-to-end journey of a single request across all services
- Which services were called, in what order, how long each took
- Parent-child relationships between operations
- Errors and slowness at each hop

**Key concepts:**

```
Trace (one complete request journey):
─────────────────────────────────────────────────────────────────
traceId: a1b2c3d4e5f67890

  Span A: API Gateway          [0ms ──────────────────── 350ms]
    Span B: Order Service        [5ms ────────────── 340ms]
      Span C: DB Query             [10ms ──── 80ms]
      Span D: Inventory Call        [90ms ──────────── 200ms]
        Span E: Inventory DB          [95ms ── 180ms]
      Span F: Payment Call            [210ms ──── 320ms]
─────────────────────────────────────────────────────────────────
```

**Span:** A single unit of work. Has:
- `traceId`: same for all spans in the request
- `spanId`: unique ID for this operation
- `parentSpanId`: ID of the calling span (null for root span)
- `name`: operation name (e.g., "HTTP POST /api/orders")
- `startTime` and `endTime`
- `tags/attributes`: key-value metadata
- `events/logs`: timestamped events within the span
- `status`: OK, ERROR

**Difference between traces and logs:**

| Dimension | Logs | Traces |
|---|---|---|
| Scope | Single service, single event | Entire request across all services |
| Structure | Flat (list of events) | Hierarchical (tree of spans) |
| Timing | Event timestamp | Start + end time = duration |
| Correlation | Via traceId field | Native parent-child structure |
| Best for | "What happened in Service X?" | "Why is the total request slow?" |

---

## 2. Logs — Deep Dive

### 2.1 Setting Up Structured Logging in Spring Boot

**Dependencies (pom.xml):**
```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

**logback-spring.xml:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- JSON appender for production -->
    <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>requestId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <!-- Add service metadata to every log line -->
            <customFields>{"service":"order-service","env":"${SPRING_PROFILES_ACTIVE}"}</customFields>
        </encoder>
    </appender>

    <!-- Human-readable appender for local development -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{traceId}] - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- Use JSON in prod, plain text in dev -->
    <springProfile name="prod,staging">
        <root level="INFO">
            <appender-ref ref="JSON_CONSOLE"/>
        </root>
    </springProfile>

    <springProfile name="dev,default">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>
</configuration>
```

**Sample JSON log output:**
```json
{
  "@timestamp": "2024-01-15T10:23:45.123Z",
  "@version": "1",
  "message": "Order 12345 created successfully",
  "logger_name": "com.example.OrderService",
  "level": "INFO",
  "level_value": 20000,
  "traceId": "a1b2c3d4e5f67890abcd",
  "spanId": "f1e2d3c4",
  "requestId": "req-550e8400",
  "userId": "user-789",
  "service": "order-service",
  "env": "production"
}
```

---

### 2.2 MDC (Mapped Diagnostic Context)

MDC is a thread-local map of key-value pairs that Logback/Log4j2 automatically appends to every log statement on that thread.

**How it works:**
```java
// MDC stores values in a ThreadLocal map
// Every log statement on this thread will include these values
MDC.put("requestId", UUID.randomUUID().toString());
MDC.put("userId", "user-789");

log.info("Starting order processing");    // includes requestId and userId
log.warn("Inventory low for product 42"); // includes requestId and userId
log.info("Order completed");              // includes requestId and userId

MDC.clear(); // CRITICAL: always clear MDC to prevent ThreadLocal leaks
```

**MDC in a Spring Filter (production pattern):**
```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class MdcRequestFilter implements Filter {

    private static final String REQUEST_ID_HEADER = "X-Request-ID";
    private static final String USER_ID_HEADER = "X-User-ID";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        try {
            // Use existing request ID from upstream, or generate new one
            String requestId = Optional
                .ofNullable(httpRequest.getHeader(REQUEST_ID_HEADER))
                .orElse(UUID.randomUUID().toString());

            MDC.put("requestId", requestId);
            MDC.put("httpMethod", httpRequest.getMethod());
            MDC.put("uri", httpRequest.getRequestURI());

            // If authenticated, add user context
            String userId = httpRequest.getHeader(USER_ID_HEADER);
            if (userId != null) {
                MDC.put("userId", userId);
            }

            // Pass request ID downstream to other services
            chain.doFilter(request, response);
        } finally {
            MDC.clear(); // Always clear — prevents ThreadLocal leaks in thread pools
        }
    }
}
```

**Spring Boot + Micrometer auto-populates traceId/spanId:**

When you have `micrometer-tracing` on the classpath, Spring Boot automatically puts `traceId` and `spanId` into MDC for every request. Your logs automatically include trace context without any manual wiring.

```
# Log output with auto-populated trace context:
INFO [order-service,a1b2c3d4e5f67890,f1e2d3c4] OrderService - Order created
      ^service name   ^traceId           ^spanId
```

> **Best Practice:** Always propagate the `X-Request-ID` header from your API gateway through all downstream services. Store it in MDC. This allows you to correlate logs across services for a single user request even before you have full distributed tracing set up.

---

### 2.3 Correlation ID Propagation Pattern

```
┌────────────────────────────────────────────────────────────────────┐
│  Client                                                            │
│    │  GET /api/orders                                              │
│    │  X-Request-ID: req-550e8400                                   │
└────┼───────────────────────────────────────────────────────────────┘
     │
┌────▼───────────────────────────────────────────────────────────────┐
│  API Gateway                                                       │
│    MDC: requestId=req-550e8400                                     │
│    Logs: [req-550e8400] Routing to order-service                   │
│    Forwards: X-Request-ID: req-550e8400                            │
└────┼───────────────────────────────────────────────────────────────┘
     │
┌────▼───────────────────────────────────────────────────────────────┐
│  Order Service                                                     │
│    MDC: requestId=req-550e8400, traceId=abc123, spanId=def456      │
│    Logs: [req-550e8400][abc123] Fetching order from DB             │
└────────────────────────────────────────────────────────────────────┘
```

**Propagating request ID with RestTemplate:**
```java
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.getInterceptors().add((request, body, execution) -> {
            // Forward correlation ID to downstream services
            String requestId = MDC.get("requestId");
            if (requestId != null) {
                request.getHeaders().set("X-Request-ID", requestId);
            }
            return execution.execute(request, body);
        });
        return restTemplate;
    }
}
```

---

### 2.4 Logging Best Practices

```java
@Service
@Slf4j // Lombok annotation — generates: private static final Logger log = LoggerFactory.getLogger(...)
public class OrderService {

    public Order createOrder(CreateOrderRequest request) {
        // GOOD: Log with structured context at INFO for business events
        log.info("Creating order for user={}, itemCount={}, total={}",
            request.getUserId(), request.getItems().size(), request.getTotal());

        try {
            Order order = orderRepository.save(buildOrder(request));

            // GOOD: Log success with outcome details
            log.info("Order created successfully orderId={}, userId={}",
                order.getId(), request.getUserId());

            return order;

        } catch (InsufficientInventoryException e) {
            // GOOD: WARN for expected/handled failures
            log.warn("Insufficient inventory for order userId={}, productId={}",
                request.getUserId(), e.getProductId());
            throw e;

        } catch (Exception e) {
            // GOOD: ERROR with exception for unexpected failures
            log.error("Unexpected error creating order userId={}", request.getUserId(), e);
            throw new OrderCreationException("Order creation failed", e);
        }
    }
}
```

**Anti-patterns to avoid:**
```java
// BAD: No context — useless for debugging
log.error("Error occurred");

// BAD: String concatenation — evaluated even when log level is off
log.debug("Processing " + items.size() + " items for user " + userId);
// GOOD: Use parameterized logging — evaluated lazily
log.debug("Processing {} items for user {}", items.size(), userId);

// BAD: Logging sensitive data
log.info("Payment processed cardNumber={}", creditCard.getNumber());

// BAD: Logging in a tight loop — floods logs, destroys signal
for (Item item : items) {
    log.debug("Processing item {}", item); // Could log millions of lines
}
```

---

## 3. Metrics — Deep Dive

### 3.1 Micrometer

Micrometer is an instrumentation facade for JVM-based applications. The relationship:

```
Application Code
      │
      │ uses Micrometer API
      ▼
┌─────────────────┐
│   Micrometer    │  ← Vendor-neutral instrumentation layer
│   (facade)      │    (like SLF4J for metrics)
└─────────────────┘
      │
      │ pluggable registry adapters
      ▼
┌──────────┐ ┌──────────┐ ┌───────────┐ ┌──────────┐ ┌──────────┐
│Prometheus│ │Datadog   │ │CloudWatch │ │InfluxDB  │ │New Relic │
└──────────┘ └──────────┘ └───────────┘ └──────────┘ └──────────┘
```

**Key advantage:** Write your instrumentation once using Micrometer's API, then switch backends by changing a dependency and configuration. No code changes needed.

---

### 3.2 Spring Boot Actuator + Prometheus Setup

**pom.xml:**
```xml
<!-- Spring Boot Actuator -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Micrometer Prometheus registry -->
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
      base-path: /actuator
  endpoint:
    health:
      show-details: always
      show-components: always
    prometheus:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    distribution:
      percentiles-histogram:
        http.server.requests: true        # Enable histogram for HTTP requests
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99  # Export these percentiles
      slo:
        http.server.requests: 100ms, 500ms, 1s  # SLO buckets
    tags:
      application: ${spring.application.name}  # Tag all metrics with app name
      env: ${SPRING_PROFILES_ACTIVE:dev}
```

**Auto-configured metrics in Spring Boot:**

| Metric | Description |
|---|---|
| `jvm.memory.used` | JVM heap and non-heap usage |
| `jvm.gc.pause` | GC pause durations |
| `jvm.threads.live` | Live thread count |
| `http.server.requests` | Per-endpoint request count, duration, errors |
| `hikaricp.connections.active` | Active DB connections |
| `hikaricp.connections.pending` | Threads waiting for a DB connection |
| `spring.data.repository.invocations` | JPA repository method calls |
| `cache.gets` | Cache hit/miss counts |
| `kafka.consumer.lag` | Kafka consumer lag (if Kafka is used) |

---

### 3.3 Custom Metrics

```java
@Component
public class OrderMetrics {

    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Timer orderProcessingTime;
    private final AtomicInteger activeOrders = new AtomicInteger(0);

    public OrderMetrics(MeterRegistry registry) {
        // Counter: total orders created, broken down by region
        ordersCreated = Counter.builder("orders.created")
            .description("Total number of orders successfully created")
            .tag("region", "us-east-1")
            .register(registry);

        // Counter: failed orders broken down by failure reason
        ordersFailed = Counter.builder("orders.failed")
            .description("Total number of orders that failed")
            .register(registry);

        // Timer: measures both count and duration, with percentiles
        orderProcessingTime = Timer.builder("orders.processing.duration")
            .description("Time taken to process an order end-to-end")
            .publishPercentiles(0.5, 0.95, 0.99)      // Client-side percentiles
            .publishPercentileHistogram()               // Server-side histogram for PromQL
            .sla(Duration.ofMillis(100), Duration.ofMillis(500)) // SLO buckets
            .register(registry);

        // Gauge: current snapshot value (backed by AtomicInteger)
        Gauge.builder("orders.active", activeOrders, AtomicInteger::get)
            .description("Number of orders currently being processed")
            .register(registry);

        // Gauge from a service bean (lazy evaluation)
        Gauge.builder("orders.queue.depth", orderQueue, Queue::size)
            .description("Orders waiting in processing queue")
            .register(registry);
    }

    public void recordOrderCreated(String region) {
        // Tags can be dynamic at record time (but must be bounded!)
        ordersCreated.increment(Tags.of("region", region));
    }

    public <T> T recordOrderProcessing(Supplier<T> orderLogic) {
        return orderProcessingTime.record(orderLogic);
    }

    public void orderStarted() { activeOrders.incrementAndGet(); }
    public void orderFinished() { activeOrders.decrementAndGet(); }
}
```

**Using @Timed annotation:**
```java
@Service
public class PaymentService {

    // @Timed automatically wraps the method with a Timer
    @Timed(value = "payment.processing.time",
           description = "Time to process payment",
           percentiles = {0.5, 0.95, 0.99},
           histogram = true)
    public PaymentResult processPayment(PaymentRequest request) {
        // Method body — timing is handled automatically
        return stripeClient.charge(request);
    }
}
```

> **Note:** `@Timed` needs a `TimedAspect` bean (`new TimedAspect(registry)`) registered when you're not relying on the Spring Boot AOP starter.

---

### 3.4 DistributionSummary and LongTaskTimer

Two more meter types worth knowing by name:
- **DistributionSummary** — like a Timer but for arbitrary (non-time) values such as request body size or batch size: `DistributionSummary.builder("http.request.size").baseUnit("bytes")...record(size)`.
- **LongTaskTimer** — for long-running tasks (batch jobs, exports that take minutes/hours); unlike Timer it tracks tasks *currently in progress*. Start a sample before the work and `sample.stop()` in `finally`.

---

## 4. Distributed Tracing — Deep Dive

### 4.1 How Distributed Tracing Works

**The anatomy of a trace:**

```
User: POST /api/checkout
──────────────────────────────────────────────────────────────────
traceId: 1a2b3c4d5e6f7890  (same across ALL services)

  [API Gateway]     spanId: 0001  parentSpanId: null  (root span)
  |  0ms ──────────────────────────────────────────── 420ms

  [Order Service]   spanId: 0002  parentSpanId: 0001
  |  10ms ─────────────────────────────────── 410ms

  [DB: SELECT]      spanId: 0003  parentSpanId: 0002
  |  12ms ──── 50ms

  [Inventory Svc]   spanId: 0004  parentSpanId: 0002
  |  55ms ─────────────────── 200ms

  [Inventory DB]    spanId: 0005  parentSpanId: 0004
  |  57ms ──────────── 190ms   ← SLOW! This is the bottleneck

  [Payment Svc]     spanId: 0006  parentSpanId: 0002
  |  210ms ────────── 380ms

  [Stripe API]      spanId: 0007  parentSpanId: 0006
  |  215ms ──────── 370ms
──────────────────────────────────────────────────────────────────
```

**Context Propagation — How trace IDs travel across services:**

When Service A calls Service B, it injects the trace context into HTTP headers. Service B extracts it, reuses the same `traceId`, and creates a new child span. In Spring Boot this happens automatically — you rarely wire it by hand.

The standard header is W3C `traceparent` (the older Zipkin format is `X-B3-*`):
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^  ^traceId (32 hex chars)             ^spanId (16hex) ^flags
             version
```

---

### 4.2 Sampling Strategies

Tracing every request at 100% is expensive, so production samples a fraction. As a junior, know the two approaches at an awareness level:

- **Head-based sampling**: the decision (trace or not) is made at the start of the request and propagated downstream. Simple and cheap, but can miss rare failures. This is what you set in Spring Boot: `management.tracing.sampling.probability: 0.1` (10%).
- **Tail-based sampling**: the decision is deferred until the trace completes, so you can keep all errors/slow traces and sample the rest. More accurate but needs buffering — handled by the OTel Collector, Tempo, or Honeycomb, not the app.

A common production setup is head-based at 10-20% with errors always sampled.

---

### 4.3 Spring Boot 3 + Micrometer Tracing

Spring Boot 3 replaced Spring Cloud Sleuth with Micrometer Tracing.

**pom.xml:**
```xml
<!-- Micrometer Tracing with Brave (Zipkin) bridge -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>

<!-- OR: Micrometer Tracing with OpenTelemetry bridge -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

**application.yml:**
```yaml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% in dev/staging, 0.1 in prod

spring:
  zipkin:
    base-url: http://zipkin:9411
    sender:
      type: web  # HTTP sender (vs Kafka sender for high volume)

logging:
  pattern:
    # Include traceId and spanId in log pattern (Micrometer auto-populates MDC)
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

**What gets auto-instrumented:**
- `@RestController` endpoints — inbound HTTP spans
- `RestTemplate` calls — outbound HTTP spans
- `WebClient` calls — outbound HTTP spans (reactive)
- `@KafkaListener` — consumer spans
- `KafkaTemplate.send()` — producer spans
- `@Scheduled` methods — span per execution
- Spring Data repository calls (with additional config)

**Manual span creation:**
```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final Tracer tracer;           // Micrometer Tracer
    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;

    public Order processOrder(CreateOrderRequest request) {
        // Create a child span for a specific sub-operation
        Span span = tracer.nextSpan()
            .name("order.process")
            .tag("order.userId", request.getUserId())
            .tag("order.itemCount", String.valueOf(request.getItems().size()));

        // withSpan makes this span the "current" span for this try block
        try (Tracer.SpanInScope ws = tracer.withSpan(span.start())) {

            span.event("validation.started");
            validateOrder(request);
            span.event("validation.completed");

            // Child spans in called methods will automatically be children
            // of the current span
            Order saved = orderRepository.save(buildOrder(request));
            span.tag("order.id", saved.getId().toString());

            return saved;

        } catch (Exception e) {
            span.tag("error", e.getMessage());
            span.error(e); // Marks span as error in Zipkin/Jaeger
            throw e;
        } finally {
            span.end(); // Always end the span
        }
    }
}
```

With Micrometer Tracing, the `traceId` is automatically placed in MDC, so a normal `log.info(...)` call already prints `[service,traceId,spanId]` — no extra wiring needed.

> **Awareness:** Across async boundaries (`@Async`, `CompletableFuture`, thread pools) the trace context is NOT inherited by default, because it lives in a ThreadLocal. Micrometer 1.10+ fixes this when you wrap your executor with a context-propagating wrapper (`ContextExecutorService` / `ContextSnapshot`). You usually only need to know this exists.

---

### 4.4 Zipkin

Zipkin is a tracing backend: instrumented services report spans to a Zipkin **collector**, which stores them (in-memory for dev; MySQL/Elasticsearch/Cassandra for production) and shows them in a web UI on port 9411. The UI lets you search traces by service/duration/tags, view the span waterfall, and see a service dependency graph.

**Running Zipkin with Docker:**
```bash
# Quick start (in-memory storage)
docker run -d -p 9411:9411 openzipkin/zipkin
```

---

### 4.5 Jaeger

Jaeger (originally from Uber) is another tracing backend with the same collector → storage → UI shape, but it's more feature-rich, natively supports OpenTelemetry/OTLP, and offers adaptive sampling (auto-tuning sample rates by traffic). Its UI runs on port 16686.

**Running Jaeger with Docker (all-in-one for dev):**
```bash
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 4317:4317 \
  jaegertracing/all-in-one:latest
```

**Jaeger vs Zipkin (the points an interviewer wants):**

| | Zipkin | Jaeger |
|---|---|---|
| Origin | Twitter | Uber |
| OTel compatibility | Via compatibility layer | Native |
| Sampling | Fixed / per-service | Adaptive |
| Best for | Simpler setups | Large-scale systems |

---

## 5. OpenTelemetry

### 5.1 What is OpenTelemetry?

OpenTelemetry (OTel) is a vendor-neutral, open-source observability framework for generating, collecting, and exporting telemetry data (logs, metrics, traces). It is the merger of OpenTracing and OpenCensus projects and is now the CNCF standard.

```
┌────────────────────────────────────────────────────────────────────┐
│  OpenTelemetry                                                     │
│                                                                    │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │     API     │  │     SDK      │  │    Instrumentation       │ │
│  │             │  │              │  │    Libraries             │ │
│  │  Interfaces │  │Implementation│  │   (auto-instrument       │ │
│  │  TracerAPI  │  │  Samplers    │  │   Spring, JDBC, Kafka,   │ │
│  │  MeterAPI   │  │  Exporters   │  │   HTTP clients, etc.)   │ │
│  │  LoggerAPI  │  │  Processors  │  │                          │ │
│  └─────────────┘  └──────────────┘  └──────────────────────────┘ │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │              OTel Collector (optional but recommended)      │  │
│  │  Receives → Processes → Exports to multiple backends        │  │
│  └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

**OTel vs Micrometer:**

| | OpenTelemetry | Micrometer |
|---|---|---|
| Scope | Traces + Metrics + Logs | Primarily Metrics + Tracing bridge |
| Ecosystem | CNCF standard, language-agnostic | JVM ecosystem |
| Spring Boot | Via micrometer-tracing-bridge-otel | Native Spring Boot support |
| Auto-instrumentation | Java agent (zero code change) | Spring auto-configuration |

---

### 5.2 OTel Java Agent (Zero-Code Instrumentation)

```bash
# Add to JVM startup flags — no code changes required
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=order-service \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
     -Dotel.exporter.otlp.protocol=grpc \
     -Dotel.resource.attributes=env=production,team=payments \
     -Dotel.traces.sampler=parentbased_traceidratio \
     -Dotel.traces.sampler.arg=0.1 \
     -jar order-service.jar
```

**What the Java agent auto-instruments:** Spring MVC/WebFlux, JDBC (SQL queries become spans), Redis, Kafka (producer and consumer), gRPC, HTTP clients, and scheduled tasks — with no code changes.

> **Awareness (SRE-level):** In larger setups, services send telemetry to an **OTel Collector** (a YAML pipeline of receivers → processors → exporters) which fans data out to Jaeger, Prometheus, Loki, etc. You configure this once at the platform level; as a junior you just point your app's OTLP endpoint at it. See section 11 Q22 for why it's used.

### 5.3 Manual OTel Instrumentation

Manual instrumentation with the raw OTel API mirrors the Micrometer `Tracer` example in 4.3: build a span with `tracer.spanBuilder(...)`, set attributes, `makeCurrent()` in a try-with-resources, `recordException`/`setStatus(ERROR)` on failure, and always `span.end()` in `finally`. In Spring Boot you normally use Micrometer's `Tracer` (4.3) rather than the raw OTel API directly.

---

## 6. Spring Boot Actuator

### 6.1 Key Endpoints

**application.yml (expose all endpoints — restrict in production):**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"          # All endpoints (dev only!)
        # include: health,info,metrics,prometheus  # Production
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized  # Only show details to authorized users
      show-components: always
    env:
      keys-to-sanitize: password,secret,key,token,.*credentials.*  # Mask sensitive values
    loggers:
      enabled: true
```

| Endpoint | Description | Example Use |
|---|---|---|
| `/actuator/health` | Application health status, component health | Kubernetes liveness/readiness probes |
| `/actuator/info` | Application info (version, build time) | Display in monitoring dashboards |
| `/actuator/metrics` | List all available metrics | Browse what's instrumented |
| `/actuator/metrics/{name}` | Details for specific metric | `GET /actuator/metrics/jvm.memory.used` |
| `/actuator/prometheus` | Prometheus scrape endpoint | Prometheus scrapes this URL |
| `/actuator/loggers` | View/change log levels at runtime | Change DEBUG/INFO without restart |
| `/actuator/env` | Environment properties | Debug config values |
| `/actuator/beans` | All Spring beans | Debug Spring context |
| `/actuator/mappings` | All HTTP endpoint mappings | See what URLs are registered |
| `/actuator/threaddump` | Current thread dump | Debug thread deadlocks |
| `/actuator/heapdump` | Heap dump file | Debug OOM issues |
| `/actuator/httptrace` | Last 100 HTTP requests | Quick recent request history |
| `/actuator/scheduledtasks` | Scheduled task details | Debug cron jobs |

**Changing log level at runtime (no restart):**
```bash
# Change order-service package to DEBUG without restarting
curl -X POST http://localhost:8080/actuator/loggers/com.example.orderservice \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel":"DEBUG"}'

# Revert to INFO
curl -X POST http://localhost:8080/actuator/loggers/com.example.orderservice \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel":"INFO"}'
```

---

### 6.2 Health Indicators

**Built-in health indicators (auto-configured when dependencies present):**
- `DataSourceHealthIndicator` — DB connectivity
- `RedisHealthIndicator` — Redis connectivity
- `KafkaHealthIndicator` — Kafka broker connectivity
- `DiskSpaceHealthIndicator` — Disk space
- `MongoHealthIndicator` — MongoDB connectivity

**Custom health indicator:**
```java
@Component
public class ExternalPaymentApiHealthIndicator implements HealthIndicator {

    private final PaymentGatewayClient paymentClient;
    private final RestTemplate restTemplate;

    @Override
    public Health health() {
        try {
            // Check if external payment API is reachable
            ResponseEntity<String> response = restTemplate.getForEntity(
                "https://api.stripe.com/v1/charges?limit=1", String.class);

            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("paymentGateway", "Stripe")
                    .withDetail("status", "reachable")
                    .withDetail("responseTime", "< 500ms")
                    .build();
            }
            return Health.down()
                .withDetail("reason", "Unexpected status: " + response.getStatusCode())
                .build();

        } catch (Exception e) {
            return Health.down(e)
                .withDetail("paymentGateway", "Stripe")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

**Health response example:**
```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": { "database": "PostgreSQL", "validationQuery": "isValid()" }
    },
    "redis": { "status": "UP", "details": { "version": "7.0.5" } },
    "externalPaymentApi": {
      "status": "UP",
      "details": { "paymentGateway": "Stripe", "status": "reachable" }
    },
    "diskSpace": {
      "status": "UP",
      "details": { "total": 499963174912, "free": 100000000000, "threshold": 10485760 }
    }
  }
}
```

---

### 6.3 Liveness vs Readiness Probes

This is a very common interview question, especially in Kubernetes contexts.

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Kubernetes Pod                             │
│                                                                     │
│  Liveness Probe → /actuator/health/liveness                        │
│  Question: "Is this container alive? Should it be restarted?"      │
│  Fail → Kubernetes KILLS and RESTARTS the container                 │
│  Use: Detect deadlocks, infinite loops, corrupted internal state    │
│                                                                     │
│  Readiness Probe → /actuator/health/readiness                      │
│  Question: "Is this container ready to receive traffic?"           │
│  Fail → Kubernetes REMOVES pod from Service load balancer          │
│  (pod keeps running, just no traffic)                               │
│  Use: Detect DB disconnects, dependency unavailability, startup     │
└─────────────────────────────────────────────────────────────────────┘
```

**Spring Boot Kubernetes probe configuration:**
```yaml
# application.yml
management:
  endpoint:
    health:
      probes:
        enabled: true  # Enables /actuator/health/liveness and /readiness
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
  initialDelaySeconds: 30    # Wait 30s before first check (app startup)
  periodSeconds: 10          # Check every 10 seconds
  failureThreshold: 3        # Restart after 3 consecutive failures
  timeoutSeconds: 5

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 5
  failureThreshold: 3
  timeoutSeconds: 3
```

**Custom liveness/readiness contributor:**
```java
@Component
public class OrderServiceReadinessContributor implements ReactiveHealthContributor {

    private final DatabaseConnectionPool dbPool;

    @Override
    public Health health() {
        // Mark as NOT READY if DB connection pool is exhausted
        int available = dbPool.getAvailableConnections();
        if (available == 0) {
            return Health.outOfService()
                .withDetail("reason", "DB connection pool exhausted")
                .build();
        }
        return Health.up().build();
    }
}
```

> **Interview Tip:** The key distinction: liveness failure = "this pod is broken, kill it". Readiness failure = "this pod is alive but can't serve traffic right now, route traffic elsewhere". Readiness probe failure is graceful; liveness probe failure is disruptive.

---

## 7. Prometheus and Grafana

### 7.1 Prometheus Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ order-svc    │  │payment-svc   │  │inventory-svc │             │
│  │:8080/actuator│  │:8081/actuator│  │:8082/actuator│             │
│  │  /prometheus │  │  /prometheus │  │  /prometheus │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│         │                 │                 │                      │
│         │ ◄── PULL ───────┼─────────────────┘                      │
│         │   (Prometheus scrapes every 15s)                         │
│         ▼                                                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Prometheus Server                                           │  │
│  │  - Scrapes /actuator/prometheus                              │  │
│  │  - Stores time-series data (TSDB)                            │  │
│  │  - Evaluates alerting rules                                  │  │
│  │  - Exposes query API (PromQL)                                │  │
│  └────────────────────────────────┬─────────────────────────────┘  │
│                                   │                                 │
│         ┌─────────────────────────┤                                 │
│         ▼                         ▼                                 │
│  ┌──────────────┐         ┌──────────────┐                         │
│  │ Alertmanager │         │   Grafana    │                         │
│  │ (route/page) │         │ (dashboards) │                         │
│  └──────────────┘         └──────────────┘                         │
└──────────────────────────────────────────────────────────────────────┘
```

**prometheus.yml:**
```yaml
global:
  scrape_interval: 15s       # Scrape every 15 seconds
  evaluation_interval: 15s   # Evaluate alerting rules every 15s

scrape_configs:
  - job_name: 'spring-boot-services'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets:
          - 'order-service:8080'
          - 'payment-service:8081'
          - 'inventory-service:8082'
    # For Kubernetes: use kubernetes_sd_configs instead of static_configs

  - job_name: 'spring-boot-k8s'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

rule_files:
  - "alerting_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

---

### 7.2 PromQL — Essential Queries

A junior should recognize these core patterns (a fuller list is in the cheat sheet). `rate()` gives per-second rate of a counter; `histogram_quantile()` derives percentiles from histogram buckets.

```promql
# Request rate (RPS) for a service
sum(rate(http_server_requests_seconds_count{application="order-service"}[5m]))

# Error rate as a percentage (5xx / total)
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count[5m])) * 100

# p95 latency (requires percentiles-histogram enabled)
histogram_quantile(0.95,
  sum(rate(http_server_requests_seconds_bucket[5m])) by (le))

# JVM heap usage %
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# DB pending connections — alarm if > 0 consistently
hikaricp_connections_pending
```

> **Awareness:** There are many more queries (per-endpoint error breakdowns, GC pause rate, connection-acquisition p99, Kafka consumer lag, etc.). They follow the same `rate()` / `histogram_quantile()` / `sum by (...)` building blocks.

---

### 7.3 Prometheus Alerting Rules

A Prometheus alerting rule pairs a PromQL `expr` with a `for:` duration (how long it must stay true before firing) and a severity label. Example:

```yaml
groups:
  - name: spring-boot-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          / sum(rate(http_server_requests_seconds_count[5m])) * 100 > 5
        for: 2m            # must be true for 2 minutes before firing
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.application }}"
```

The same pattern covers high latency (p99 > 2s), high JVM heap (> 80%), and DB pool exhaustion (`hikaricp_connections_pending > 0`).

---

### 7.4 Grafana Dashboards

**Key Spring Boot dashboard panels:**

```
┌─────────────────────────────────────────────────────────────────────┐
│               Spring Boot Service Dashboard                         │
├──────────────┬──────────────┬──────────────┬────────────────────────┤
│  RPS          │  Error Rate  │  p99 Latency │  Active Instances      │
│  245 req/s   │  0.2%        │  340ms       │  3                     │
├──────────────┴──────────────┴──────────────┴────────────────────────┤
│  Request Rate (time series)    │  Error Rate (time series)           │
│  ────────────────────────────  │  ─────────────────────────────────  │
│  /api/orders     ────────      │  5xx ──                             │
│  /api/payments   ──────        │  4xx ─────────                      │
├────────────────────────────────┼────────────────────────────────────┤
│  Latency Heatmap (p50/95/99)  │  JVM Heap Usage                     │
│  ───────────────────────────  │  ─────────────────────────────────  │
│  p99 ────────────────────     │  Used ──────────────────────────    │
│  p95 ────────────────         │  Max ─────────────────────────────  │
│  p50 ───────────              │                                      │
├────────────────────────────────┼────────────────────────────────────┤
│  DB Pool: Active/Pending      │  Kafka Consumer Lag                  │
│  ───────────────────────────  │  ─────────────────────────────────  │
│  Active ────────────────      │  orders-topic ─────────────────     │
│  Pending (should be 0) ─      │  payments-topic ───────────────     │
└────────────────────────────────┴────────────────────────────────────┘
```

**The Four Golden Signals (Google SRE):**

| Signal | What to Measure | Prometheus Query |
|---|---|---|
| **Latency** | Time to serve requests (success vs error separately) | `histogram_quantile(0.99, ...)` |
| **Traffic** | Demand on your system (requests per second) | `rate(http_server_requests_seconds_count[5m])` |
| **Errors** | Rate of failed requests | Error rate % query above |
| **Saturation** | How full your service is (CPU %, DB pool usage) | `hikaricp_connections_pending` |

**RED Method (for services):**
- **R**ate: requests per second
- **E**rrors: errors per second (or error rate %)
- **D**uration: latency distribution

**USE Method (for infrastructure/resources):**
- **U**tilization: how busy is the resource? (CPU %, memory %)
- **S**aturation: how much extra work is queued? (run queue, DB pending)
- **E**rrors: error events (disk errors, network drops)

> **Interview Tip:** RED is for services (like microservices endpoints). USE is for resources (CPU, memory, disk, network). For an order service, apply RED. For the server it runs on, apply USE.

---

## 8. Log Aggregation: ELK, EFK, Loki

### 8.1 ELK Stack

```
┌────────────────────────────────────────────────────────────────────┐
│  Application Servers                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ order-svc    │  │ payment-svc  │  │inventory-svc │           │
│  │ app.log (JSON│  │ app.log (JSON│  │ app.log (JSON│           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         │ Filebeat         │ Filebeat         │ Filebeat          │
└─────────┼──────────────────┼──────────────────┼───────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │ Beats protocol
                             ▼
                    ┌──────────────────┐
                    │    Logstash      │  ← Parse, transform, enrich
                    │  Input: Beats    │
                    │  Filter: grok    │
                    │  Output: ES      │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  Elasticsearch   │  ← Store, index, search
                    │  (distributed    │
                    │   search engine) │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │     Kibana       │  ← Visualize, query, alert
                    │  (UI dashboard)  │
                    └──────────────────┘
```

A Logstash pipeline has three stages: `input` (e.g. Beats on port 5044), `filter` (parse the JSON message, fix the timestamp, rename `traceId`, drop noisy health-check lines), and `output` (write to Elasticsearch, typically into daily indices like `app-logs-{service}-YYYY.MM.dd`). The config is a Ruby-style DSL; the details are platform-team work, not something a junior writes from scratch.

---

### 8.2 EFK Stack (Kubernetes-Preferred)

EFK swaps Logstash for **Fluentd/Fluent Bit**: a lightweight agent runs as a DaemonSet on each Kubernetes node, reads container stdout/stderr, and ships logs to Elasticsearch → Kibana. It's preferred in Kubernetes because the agent uses far less memory than Logstash.

**Fluentd vs Logstash:**

| | Fluentd | Logstash |
|---|---|---|
| Language | Ruby + C | JRuby (JVM) |
| Memory | ~40MB | ~500MB |
| Config format | YAML-like | Ruby DSL |
| Kubernetes | Native support | Needs extra config |
| Plugins | ~1000 | ~200 |
| Best for | Kubernetes, resource-constrained | Complex transformations |

---

### 8.3 Grafana Loki

Loki is "Prometheus but for logs" — designed for cost-effective log aggregation.

**How Loki differs from Elasticsearch:**

| | Elasticsearch | Loki |
|---|---|---|
| Indexing | Indexes all log fields | Indexes only labels (like Prometheus) |
| Storage cost | High (full-text index) | Low (compressed raw logs) |
| Query speed | Fast for any field | Fast for labels, slower for full-text |
| Query language | Lucene / KQL | LogQL |
| Integration | Kibana | Grafana (same tool as metrics) |
| Best for | Complex log analytics | High-volume logs, K8s, cost-sensitive |

**Loki architecture:**
```
Application → Promtail (agent) → Loki → Grafana
```

**LogQL query examples:** filter by label first (fast, uses the index), then optionally by content:
```logql
# Errors for a service in production
{app="order-service", env="production"} |= "ERROR"

# Find all logs for one request (crucial for debugging)
{app="order-service"} | json | traceId = "a1b2c3d4e5f6"
```

> **Awareness:** A **Promtail** agent (configured in YAML, similar to Prometheus scrape config) ships logs from files/Kubernetes pods to Loki and extracts labels like `level` and `service`. Platform-team setup, not junior code.

---

### 8.4 Cloud Logging Solutions

Managed cloud platforms provide log aggregation without running ELK/Loki yourself: **AWS CloudWatch Logs** (ship via a Logback appender, query with CloudWatch Insights), GCP Cloud Logging, and Azure Monitor. They index your structured JSON fields (`level`, `traceId`, `service`) so you can filter errors, count by service, and find slow requests the same way. Useful when you don't want to operate the logging infrastructure.

---

## 9. Alerting and SLI/SLO/SLA

### 9.1 SLI, SLO, SLA Definitions

```
┌─────────────────────────────────────────────────────────────────────┐
│  SLI (Service Level Indicator)                                      │
│  A specific, measurable metric that indicates service health        │
│  Example: "The percentage of HTTP requests that return 2xx"        │
│           "The p99 latency of the checkout endpoint"               │
│           "The percentage of Kafka messages processed within 30s"  │
├─────────────────────────────────────────────────────────────────────┤
│  SLO (Service Level Objective)                                      │
│  A TARGET value for an SLI — your internal commitment              │
│  Example: "99.9% of requests return 2xx over 30 days"              │
│           "p99 latency < 500ms, measured over 7 days"              │
│  SLOs are aspirational targets you set for yourself                │
├─────────────────────────────────────────────────────────────────────┤
│  SLA (Service Level Agreement)                                      │
│  A CONTRACTUAL commitment to customers with consequences           │
│  Example: "We guarantee 99.9% uptime. If we miss, you get credits"│
│  SLA ≤ SLO (internal target should be tighter than the contract)  │
├─────────────────────────────────────────────────────────────────────┤
│  Error Budget                                                       │
│  How much failure you're allowed before breaching the SLO          │
│  99.9% SLO → 0.1% error budget → 43.8 min downtime/month          │
│  99.99% SLO → 0.01% → 4.38 min downtime/month                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Error budget** = `1 - SLO`. A 99.9% SLO allows ~43 minutes of downtime per month (43,200 min × 0.001); when the budget is exhausted, teams typically freeze new deployments. (The deeper SLO/error-budget math is an SRE concern — junior level just needs the concept.)

**Alert on symptoms, not causes:**

```
BAD (cause-based alerting):
- Alert: CPU > 80%
- Alert: JVM heap > 75%
- Alert: DB queries > 100ms
→ These may not affect users at all

GOOD (symptom-based alerting):
- Alert: Error rate > 1% (users are experiencing errors)
- Alert: p99 latency > 1s (users are waiting too long)
- Alert: Availability < 99.9% in the last 30 days (SLO breach imminent)
→ These directly measure user experience
```

---

### 9.2 Alertmanager Configuration

Prometheus sends firing alerts to **Alertmanager**, which then groups related alerts, deduplicates, silences/inhibits noise (e.g. suppress latency alerts when the whole service is down), and routes by severity to receivers like Slack or PagerDuty (critical → page the on-call engineer). The config is a YAML `route` tree with per-team `receivers`. Writing it in detail is an SRE/platform task; a junior should know Alertmanager is the piece that turns alerts into notifications.

---

## 10. Debugging a Slow Request in Microservices

This is a common interview scenario question: "How would you debug a slow request?"

**Step-by-step methodology:**

```
1. IDENTIFY
   └── User reports slow checkout (or alert fires for p99 > 2s)
   └── Get the trace ID from the user's request log:
       grep "userId=user-789" order-service.log | grep "POST /checkout"
       → Found: traceId=a1b2c3d4e5f67890

2. OPEN TRACE
   └── Go to Jaeger/Zipkin UI
   └── Search by traceId: a1b2c3d4e5f67890
   └── See the full waterfall:
       API Gateway        [0ms ─────────────────────────────── 4,200ms]
       Order Service      [10ms ──────────────────────────── 4,180ms]
       Inventory Call     [15ms ────────────────────────── 3,900ms]  ← SLOW
       Inventory DB       [20ms ──────────────────────── 3,850ms]  ← ROOT CAUSE

3. ANALYZE SPANS
   └── Click on "Inventory DB" span
   └── See tags:
       db.statement: SELECT * FROM inventory WHERE product_id = ?
       db.rows_affected: 50000
       db.execution_time: 3800ms
   └── AHA: Full table scan — missing index

4. CHECK METRICS
   └── Grafana: HikariCP pending connections spiked at same time?
       → If yes: DB contention, not just one query
       → If no: isolated slow query

5. REPRODUCE
   └── Run the same query with EXPLAIN ANALYZE locally
   └── Confirm missing index on inventory.product_id

6. FIX AND VERIFY
   └── Add index: CREATE INDEX idx_inventory_product_id ON inventory(product_id);
   └── Re-run query: 3ms instead of 3800ms
   └── Monitor Grafana: p99 latency returns to normal
   └── Check traces: Inventory DB span is now < 10ms
```

**Common culprits found via tracing:**

| Symptom in Trace | Root Cause | Fix |
|---|---|---|
| Many DB spans (N+1) | N+1 query problem | Add JOIN fetch or batch fetch |
| Single DB span is slow | Missing index or bad query plan | Add index, rewrite query |
| External API span slow | Third-party latency | Add timeout, circuit breaker, cache |
| Long gap between spans | Serialization, thread pool wait | Add more workers, optimize serialization |
| Span repeats many times | Retry loop or pagination issue | Tune retry policy |
| Root span is slow, all children fast | Application logic (CPU) | Profile with async-profiler |

---

## 11. Interview Questions and Answers

### Pillar and Concept Questions

**Q1: What are the three pillars of observability?**

A: The three pillars are:
1. **Logs**: Discrete, time-stamped records of events. Tell you what happened at a specific moment. Best for debugging specific errors.
2. **Metrics**: Aggregated numerical measurements over time (counters, gauges, histograms). Tell you the health and performance trends of a system. Best for alerting and dashboards.
3. **Traces**: End-to-end journey of a single request across all services. Tell you where time was spent and where failures occurred in a distributed call chain. Best for debugging latency in microservices.

---

**Q2: What is the difference between monitoring and observability?**

A: Monitoring is reactive — you define what to watch in advance and get alerted when it crosses a threshold. It handles "known unknowns" (you know your DB can be slow, so you monitor query time). Observability is proactive — the system produces enough telemetry that you can answer arbitrary questions about its behavior, including questions you didn't anticipate when building it. It handles "unknown unknowns". In practice: monitoring asks "is this thing I know about broken?", observability asks "what is actually happening in my system right now?"

---

**Q3: What is distributed tracing? What is a span?**

A: Distributed tracing tracks the path of a single request as it flows through multiple services in a microservices system. A **span** is the fundamental unit of a trace — it represents a single operation (one HTTP call, one DB query, one message processing). Each span has a unique ID, a reference to its parent span, a name, start/end timestamps, and key-value tags. A trace is a tree of spans, all sharing the same trace ID.

---

**Q4: How does a trace ID propagate across microservices?**

A: When Service A makes an HTTP call to Service B, it injects the trace context into HTTP headers. The W3C standard header is `traceparent: 00-{traceId}-{spanId}-{flags}`. The Zipkin/B3 format uses `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-ParentSpanId`. Service B reads these headers, creates a new child span using the same trace ID but a new span ID, and sets the parent span ID to what it received. This propagation is automatic in Spring Boot with Micrometer Tracing or the OTel Java agent. The trace ID is also placed in MDC so it appears in all log statements.

---

**Q5: What is OpenTelemetry?**

A: OpenTelemetry is a vendor-neutral, open-source observability framework maintained by the CNCF (Cloud Native Computing Foundation). It provides a unified API and SDK for generating logs, metrics, and traces from any application in any language. The key benefit is that you instrument your code once using OTel APIs and then export to any backend (Jaeger, Zipkin, Prometheus, Datadog, New Relic) by swapping exporters in configuration. The OTel Java agent provides zero-code auto-instrumentation — attach the JAR as a javaagent and it instruments Spring, JDBC, Kafka, HTTP clients, and more automatically.

---

**Q6: What is the difference between Zipkin and Jaeger?**

A: Both are distributed tracing backends. Zipkin originated at Twitter and is simpler with fewer moving parts — good for getting started quickly. Jaeger originated at Uber, is more feature-rich, supports adaptive sampling (automatically adjusts sample rates based on traffic), and has native OpenTelemetry support. Zipkin uses B3 propagation natively; Jaeger supports multiple formats including W3C. For new projects, Jaeger with OTel is increasingly the standard. In Spring Boot, Micrometer Tracing supports both via different bridge dependencies.

---

**Q7: What is MDC? Why is it used?**

A: MDC (Mapped Diagnostic Context) is a thread-local map provided by SLF4J/Logback that automatically appends key-value pairs to every log statement on the current thread. You use it by calling `MDC.put("traceId", "abc123")` early in request processing (typically in a Filter). Then every subsequent `log.info(...)` call on that thread automatically includes `traceId=abc123` in the output — you don't need to pass it to every method. It's used for: correlating all log lines for a single request, adding traceId/spanId (Micrometer does this automatically), adding userId and requestId context. Critical rule: always call `MDC.clear()` in a `finally` block to prevent ThreadLocal leaks in thread pools.

---

**Q8: What is structured logging?**

A: Structured logging outputs log records as machine-parseable data (usually JSON) instead of plain text strings. Each field — timestamp, level, message, service, traceId, userId — is a separate key in the JSON document. This enables log aggregation systems (Elasticsearch, CloudWatch, Splunk) to index individual fields, run faceted queries, and build dashboards. Example: instead of `"ERROR 10:23 - Failed to process order 123 for user john"`, structured logging produces `{"level":"ERROR","message":"Failed to process order","orderId":123,"userId":"john","traceId":"abc"}`. In Spring Boot, use `logstash-logback-encoder` with `LogstashEncoder`.

---

**Q9: What is Micrometer?**

A: Micrometer is the instrumentation facade for JVM applications — the SLF4J equivalent for metrics. You write your instrumentation code once using Micrometer's `MeterRegistry` API (`Counter`, `Gauge`, `Timer`, `DistributionSummary`), and then switch backends by adding a different dependency (e.g., `micrometer-registry-prometheus` vs `micrometer-registry-datadog`). Spring Boot auto-configures Micrometer and provides dozens of built-in metrics (JVM, Tomcat, Spring MVC, HikariCP). It also serves as the API layer for Micrometer Tracing in Spring Boot 3+.

---

**Q10: What are the meter types in Micrometer? What is the difference between a Counter and a Gauge?**

A: Micrometer has five core types:
- **Counter**: A monotonically increasing value — it can only go up (or reset to zero on restart). Use for counting events: total orders, total errors, total logins.
- **Gauge**: The current value of something that can go up or down. Use for snapshots: JVM heap in use, active database connections, queue depth.
- **Timer**: Measures both count and duration of events. Records latency distribution. Use for HTTP response times, DB query durations.
- **DistributionSummary**: Like a Timer but for arbitrary units (bytes, items, not time). Use for request sizes, items per batch.
- **LongTaskTimer**: For tasks that take a long time (minutes/hours). Tracks currently running tasks. Use for batch jobs, data exports.

The Counter/Gauge distinction: Counter for "how many times did X happen total", Gauge for "how many Xs exist right now".

---

**Q11: What is Prometheus? How does it collect metrics?**

A: Prometheus is an open-source time-series database and monitoring system. Unlike most monitoring tools, Prometheus uses a **pull model**: it periodically scrapes (HTTP GET) configured `/metrics` endpoints on your services. In Spring Boot, the `micrometer-registry-prometheus` dependency exposes all metrics at `/actuator/prometheus` in Prometheus text format. Prometheus scrapes this every 15 seconds (configurable), stores the data in its time-series database (TSDB), and allows querying via PromQL. It also evaluates alerting rules and sends alerts to Alertmanager. The pull model means Prometheus can detect if a service has gone down (scrape fails), and services don't need to know where Prometheus is.

---

**Q12: What is PromQL? Give a query for error rate.**

A: PromQL (Prometheus Query Language) is the query language for Prometheus. Key functions:
- `rate()`: calculates per-second rate of a counter over a time window
- `sum()`, `avg()`, `max()`, `min()`: aggregation
- `histogram_quantile()`: calculates quantiles from histogram data
- `by (label)`: group by label values

Error rate query:
```promql
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count[5m]))
* 100
```
This gives the percentage of requests returning 5xx over the last 5 minutes.

---

**Q13: What is the RED method?**

A: RED is a methodology for monitoring microservices, proposed by Tom Wilkie at Weaveworks:
- **R**ate: How many requests per second is your service handling?
- **E**rrors: How many requests are failing per second (or as a percentage)?
- **D**uration: How long are requests taking (latency distribution, especially p95/p99)?

These three metrics give you a complete picture of service health from the user's perspective. Contrast with USE (Utilization, Saturation, Errors) which is designed for infrastructure resources like CPU and disk. The Four Golden Signals from Google SRE are similar: Latency, Traffic, Errors, Saturation — where Saturation is added to RED.

---

**Q14: What is the ELK stack?**

A: ELK stands for Elasticsearch + Logstash + Kibana:
- **Elasticsearch**: Distributed search and analytics engine. Stores logs as JSON documents, indexes all fields for fast full-text and structured search.
- **Logstash**: Data processing pipeline. Ingests logs from various sources (files, Beats agents, Kafka), transforms/enriches them (parse timestamps, add fields, filter noise), and ships to Elasticsearch.
- **Kibana**: Web UI for Elasticsearch. Search logs with KQL, create visualizations and dashboards, set up alerts.
- **Filebeat/Beats**: Lightweight log shippers that run on each server and send logs to Logstash or Elasticsearch.

The common flow: Application → Filebeat → Logstash → Elasticsearch → Kibana. Modern variants use Fluentd instead of Logstash (EFK stack), especially in Kubernetes.

---

**Q15: How do you add a custom health indicator in Spring Boot?**

A:
```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        try {
            // Check the external dependency
            boolean reachable = externalService.ping();
            if (reachable) {
                return Health.up().withDetail("externalApi", "reachable").build();
            }
            return Health.down().withDetail("reason", "External API unreachable").build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```
Spring Boot auto-detects any bean implementing `HealthIndicator` and includes it in `/actuator/health`. The status rolls up — if any component is DOWN, the overall status is DOWN.

---

**Q16: What is the difference between liveness and readiness probes?**

A: Both are Kubernetes health checks configured on pods:

**Liveness probe** (`/actuator/health/liveness`): "Is this container alive?" If it fails, Kubernetes kills and restarts the container. Use it to detect unrecoverable states: deadlocks, infinite loops, corrupted internal state. Do NOT include external dependency health here — you don't want to restart a pod just because the DB is temporarily unreachable.

**Readiness probe** (`/actuator/health/readiness`): "Is this container ready to receive traffic?" If it fails, Kubernetes removes the pod from the Service's load balancer endpoints — traffic stops being routed to it, but the pod keeps running. Use it to signal "I'm starting up" or "I'm temporarily overloaded/dependencies unavailable". Include DB connectivity, required services here.

In Spring Boot, enable with `management.endpoint.health.probes.enabled: true`.

---

**Q17: How would you debug a slow request in a microservices system?**

A:
1. Obtain the trace ID — from the user's logs, from error logs filtered by userId, or from the response header `traceparent`.
2. Open Jaeger or Zipkin UI, search by trace ID. Examine the waterfall view to find the slowest span.
3. Click on the slow span: check its tags (which DB query, which endpoint, what parameters), check events (timeline of what happened inside the span).
4. Common findings:
   - Single DB span slow → Missing index, table lock, bad query plan → Run EXPLAIN ANALYZE
   - Many DB spans (20+ for one request) → N+1 query problem → Fix with JOIN fetch
   - External API span slow → Third-party latency → Add timeout, cache, circuit breaker
   - Gap between spans → Thread pool exhaustion, GC pause, serialization → Check thread metrics, GC logs
5. Cross-reference with Grafana: Did the DB pool have pending connections at that time? Was CPU spiking? Was there a GC pause?
6. Reproduce, fix, verify via traces that the slow span is now fast.

---

**Q18: What are SLI, SLO, and SLA?**

A:
- **SLI (Service Level Indicator)**: A concrete measurement of a service behavior from the user's perspective. Examples: the fraction of successful requests, the p99 latency, the percentage of messages processed within 30 seconds.
- **SLO (Service Level Objective)**: A target for your SLI. "99.9% of requests will succeed." "p99 latency will be < 500ms." This is your internal goal.
- **SLA (Service Level Agreement)**: A contractual commitment to customers, typically with financial penalties if violated. SLAs are usually less strict than SLOs (e.g., SLO: 99.95% availability, SLA: 99.9%) to give buffer.
- **Error budget**: `1 - SLO`. For a 99.9% availability SLO, you have 0.1% error budget = ~43 minutes of downtime per month.

---

**Q19: What is cardinality in metrics? Why is it a problem?**

A: Cardinality refers to the number of unique time series created by different combinations of label values. For example, if you have a metric `http_requests_total` with labels `{status, endpoint, userId}` and 1 million users, you get millions of time series. Prometheus (and other TSDBs) must keep each active time series in memory. High cardinality (millions of series) causes memory exhaustion and query slowdowns — potentially crashing Prometheus or making it unusable. The rule: labels must have a small, bounded set of values. HTTP status codes (5 values), HTTP methods (5 values), endpoint names (tens of values) are fine. User IDs, order IDs, session IDs, IP addresses are NOT. If you need per-user data, use logs or traces — not metrics.

---

**Q20: What is head-based vs tail-based sampling in distributed tracing?**

A: Head-based sampling makes the trace/don't-trace decision at the very start of a trace (when the first span is created, typically at the edge service). The decision propagates to all downstream services via the sampling flag in the `traceparent` header. Simple, low overhead. Downside: you may accidentally drop traces for rare, interesting failures.

Tail-based sampling defers the decision until the entire trace is complete. All spans are collected, and then you decide what to keep based on the full trace: keep all errored traces, keep slow traces (p99+), sample the rest at 1%. This ensures you always capture important traces. It requires buffering all spans before making the decision, which adds latency and infrastructure complexity. It's implemented in the OTel Collector, Grafana Tempo, and Honeycomb. For most teams starting out, head-based sampling at a meaningful rate (10-20%) with force-sampling for errors is sufficient.

---

**Q21: How does Spring Boot 3 Micrometer Tracing differ from Spring Cloud Sleuth?**

A: Spring Cloud Sleuth was the tracing solution for Spring Boot 2.x. It was a Spring-specific library that auto-configured Brave (Zipkin's tracing library). In Spring Boot 3, Sleuth was deprecated and replaced by Micrometer Tracing, which provides a vendor-neutral tracing facade (like Micrometer for metrics). Micrometer Tracing works with both Brave (bridge: `micrometer-tracing-bridge-brave`) and OpenTelemetry (bridge: `micrometer-tracing-bridge-otel`). The API is similar but the dependency coordinates changed. If you're migrating: remove `spring-cloud-sleuth-*`, add `micrometer-tracing-bridge-brave` and `zipkin-reporter-brave`. Move `spring.sleuth.*` config to `management.tracing.*`.

---

**Q22: What is the OTel Collector and why would you use it?**

A: The OTel Collector is a vendor-agnostic proxy for telemetry data that sits between your applications and your backends. Instead of having each service send traces directly to Jaeger and metrics directly to Prometheus, all services send to the Collector, which fans out to multiple destinations. Benefits:
1. **Vendor flexibility**: Change backends without redeploying services
2. **Data processing**: Add/remove attributes, sample traces, filter noisy spans
3. **Fan-out**: Send same data to multiple backends simultaneously (Jaeger + Tempo + Zipkin)
4. **Buffering**: The Collector can buffer and retry when backends are unavailable
5. **Reduced egress**: Aggregate and compress data before sending to cloud vendors

---

**Q23: How do you propagate trace context in Kafka messages?**

A: Kafka doesn't natively support header-based context propagation like HTTP. OpenTelemetry and Brave/Spring Kafka handle this by injecting trace context into Kafka message headers. On the producer side, the current span context is serialized into message headers (e.g., `traceparent`). On the consumer side, the listener extracts the headers and creates a child span linked to the producer's span. In Spring Kafka with Micrometer Tracing or the OTel Java agent, this happens automatically. The result is that you can see the full trace from the HTTP request that produced the message, through Kafka, to the consumer that processed it — all as one connected trace.

---

**Q24: What is a "Gauge" backed by a weakly-referenced object?**

A: Micrometer holds a **weak reference** to the object a Gauge measures. If that object is garbage collected (no strong reference kept), the Gauge silently stops reporting (returns NaN) — a common bug. Fix: keep a strong reference in your component, or back the Gauge with an `AtomicInteger`/`AtomicLong` field your component owns.

---

**Q25: What is the difference between Timer and DistributionSummary in Micrometer?**

A: Both measure distributions of recorded values, but Timer is specifically for measuring elapsed time (nanoseconds internally, displayed in seconds or milliseconds), while DistributionSummary is for measuring arbitrary values in any unit (bytes, items, dollars). Timer has a `record(Duration)` or `record(Runnable)` API; DistributionSummary has a `record(double amount)` API. Both support publishing percentiles and histograms. Use Timer for: HTTP response times, DB query durations, message processing time. Use DistributionSummary for: request body sizes, batch sizes, payment amounts.

---

**Q26: How do you correlate logs and traces?**

A: The bridge is the `traceId` field. When Micrometer Tracing (or OTel) is active:
1. For each HTTP request, a trace is created with a traceId
2. The traceId is automatically placed into MDC (Mapped Diagnostic Context)
3. Your logback pattern or JSON encoder reads MDC and includes traceId in every log line
4. Now logs say: `INFO [order-service,a1b2c3d4,e5f67890] Order created`

In Grafana, with both Loki (logs) and Tempo/Jaeger (traces) as data sources, you can configure "derived fields" — a regex that extracts the traceId from a log line and creates a clickable link to the trace in Jaeger. So from a single error log line, you can jump directly to the complete distributed trace.

---

**Q27: What metrics would you put on a production dashboard for a Java microservice?**

A: The essentials:
- **Service health (RED)**: request rate (RPS), error rate % (5xx/total), p50/p95/p99 latency — by endpoint.
- **JVM**: heap usage %, GC pause rate/duration, live thread count.
- **Database**: HikariCP active vs max connections, pending connections (alert if > 0), acquisition time p99.
- **Kafka (if used)**: consumer lag, messages/sec. **Infra**: CPU and memory.

Typical alert thresholds: error rate > 1% warning / > 5% critical; p99 > 500ms warning / > 2s critical; heap > 80% warning / > 95% critical; DB pending > 0 for 1 min critical.

---

**Q28: What is the difference between Loki and Elasticsearch for log aggregation?**

A: Elasticsearch indexes every field of every log document, enabling fast queries on any field. This is powerful but expensive in storage and compute. Loki only indexes **labels** (key-value pairs like `app=order-service, env=production`) and stores log content as compressed chunks. Querying by label is fast (like Prometheus); querying log content requires scanning the chunks for matching lines (slower). Loki is much cheaper to operate, integrates natively with Grafana, and scales horizontally like Prometheus. Elasticsearch is better for complex analytics, ad-hoc queries on arbitrary fields, and teams that already have Kibana expertise. For Kubernetes-native stacks with Grafana, Loki is increasingly preferred. For compliance/audit logging with complex querying needs, Elasticsearch wins.

---

**Q29: How do you secure Actuator endpoints in production?**

A:
```yaml
# application.yml — restrict Actuator to internal network
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus  # Expose only what's needed
  endpoint:
    health:
      show-details: when-authorized  # Details only for authenticated requests
```

```java
// SecurityConfig — require authentication for all actuator endpoints except health
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/health/**").permitAll()  // Kubernetes probes
            .requestMatchers("/actuator/prometheus").hasIpAddress("10.0.0.0/8")  // Internal only
            .requestMatchers("/actuator/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        );
        return http.build();
    }
}
```

Best practice: expose `/actuator/prometheus` only on an internal network interface or a separate management port:
```yaml
management:
  server:
    port: 8081  # Management on different port from app traffic
    address: 127.0.0.1  # Only accessible from localhost (or internal network)
```

---

**Q30: What is context propagation across async threads?**

A: When you use `@Async`, `CompletableFuture`, or thread pools, the new thread doesn't inherit the trace context or MDC values from the calling thread (they live in ThreadLocals). Solutions: (1) wrap your `ThreadPoolTaskExecutor` with Micrometer's context-propagating wrapper (1.10+), which captures and restores context automatically; (2) for plain MDC, copy it with `MDC.getCopyOfContextMap()` before submitting and `MDC.setContextMap(...)` inside the task (clearing in `finally`); (3) with the OTel API, pass `Context.current()` to the async task explicitly.

---

**Q31: What is a span event vs a span tag?**

A: Both are metadata attached to a span, but they serve different purposes.

A **tag (attribute)** is a key-value pair that describes the span itself — it's static metadata about the operation. Examples: `db.statement=SELECT * FROM orders`, `http.status=200`, `order.id=123`. Tags are shown in the span detail view and can be searched/filtered.

A **span event** is a timestamped annotation within the span — it records something that happened at a specific moment during the span's execution. Examples: `"validation.completed"` at 15ms, `"cache.miss"` at 20ms, `"db.query.started"` at 22ms. Events create a timeline within the span and are useful for understanding sequential steps within a single operation.

```java
span.tag("order.id", "123");           // attribute — describes the span
span.event("cache.miss");              // event — happened at this point in time
span.event("db.fallback.triggered");   // event — another timestamped moment
```

---

**Q32: What is the difference between a trace exporter and a trace reporter?**

A: Mostly the same idea under different vendors' terms. In Brave/Zipkin, a **Reporter** buffers spans and sends them in batches to a Sender. In OpenTelemetry, the equivalent is a **SpanExporter** (e.g. `OtlpGrpcSpanExporter`, `ZipkinSpanExporter`), wrapped by a `BatchSpanProcessor`. "Exporter" is the OTel term; "reporter" is the Zipkin/Brave term.

---

**Q33: When would you choose Prometheus over a commercial APM (Datadog, New Relic)?**

A: Choose **Prometheus + Grafana** (open-source) when cost matters, you want no vendor lock-in / on-prem data control, you're already in Kubernetes (kube-prometheus-stack), and the team can run the infrastructure. Choose a **commercial APM** (Datadog, New Relic, Dynatrace) when you want faster time-to-value, integrated logs+metrics+traces in one tool, built-in anomaly detection, and no infrastructure to manage.

---

**Q34: How does Micrometer help with the cardinality problem?**

A: Micrometer's `MeterFilter` API lets you guard against runaway cardinality: `MeterFilter.maximumAllowableTags(...)` caps the number of unique tag values for a meter and denies new ones past the limit, instead of silently exhausting memory. Filters can also deny meters, rename tags, add common tags, or bucket unbounded values.

---

**Q35: What observability tools would you recommend for a new Spring Boot microservices project?**

A: A solid open-source stack for a modern Spring Boot project:
- **Metrics**: Spring Boot Actuator + Micrometer → Prometheus + Grafana
- **Traces**: Spring Boot 3 + Micrometer Tracing (OTel bridge) → Jaeger or Grafana Tempo
- **Logs**: structured JSON (logstash-logback-encoder) → Promtail → Grafana Loki
- **Unified view**: Grafana ties all three pillars together with trace-to-log linking. On Kubernetes, kube-prometheus-stack bundles Prometheus + Grafana.

It's fully open-source and cloud-agnostic. If budget and operational overhead allow, Datadog or Honeycomb give a better out-of-the-box experience with less setup.

---

## 12. Quick Reference Cheat Sheet

### Spring Boot Observability Dependencies

```xml
<!-- Actuator (metrics, health) -->
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
        enabled: true   # Kubernetes liveness/readiness
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
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) / sum(rate(http_server_requests_seconds_count[5m])) * 100

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
| Which service is causing this request to be slow? | Traces | Jaeger / Zipkin |
| What exactly happened in this specific request? | Logs | ELK / Loki |
| Is this service healthy? | Health | Spring Actuator |
| How do I know when to page someone? | Alerting | Alertmanager / Grafana |
| Is this a known failure pattern? | Monitoring | Prometheus alert rules |
| Is this an unknown failure I need to investigate? | Observability | All three pillars together |

### MDC Pattern Summary

```java
// Filter (add context)
MDC.put("requestId", extractOrGenerate(request));
MDC.put("userId", getAuthenticatedUser());
// traceId and spanId auto-added by Micrometer Tracing

// Logger (automatically includes MDC)
log.info("Processing order {}", orderId);

// Filter finally block (always clean up)
finally { MDC.clear(); }
```

### Span Lifecycle

```java
Span span = tracer.nextSpan().name("operation.name").start();
try (Tracer.SpanInScope ws = tracer.withSpan(span.start())) {
    span.tag("key", "value");   // metadata
    span.event("checkpoint");   // timestamped event
    doWork();
} catch (Exception e) {
    span.error(e);              // mark as error
    throw e;
} finally {
    span.end();                 // always end
}
```

---

*This guide covers the observability stack for Java/Spring Boot interviews. As a junior, focus on: the three pillars, Actuator + Micrometer + Prometheus basics, structured logging with traceId correlation, and walking through how you'd debug a slow request using traces, metrics, and logs together.*

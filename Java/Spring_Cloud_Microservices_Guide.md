# Spring Cloud Microservices — Building Blocks Guide

## Overview

This is an **awareness-level guide for junior Spring Boot developers**. You are not expected to architect a microservices system from scratch. The bar is: explain what each Spring Cloud component does and why it exists, read a basic config, and write a simple `@FeignClient`. Designing the full topology is a senior/architect concern.

This guide focuses on the **Spring-specific tooling**. For microservices theory (why microservices, decomposition strategies, CAP theorem) see the **System_Design_Microservices_Interview_Questions.md** and **Distributed_Systems_Core_Concepts_Study_Guide.md**. For Saga/CQRS patterns see **Microservices_Saga_CQRS_EventSourcing.md**. For observability see **Observability_Tracing_Metrics_Logging.md**.

---

## Table of Contents

1. [The Problem Spring Cloud Solves](#1-the-problem-spring-cloud-solves)
2. [Service Discovery — Netflix Eureka](#2-service-discovery--netflix-eureka)
3. [Declarative REST Client — OpenFeign](#3-declarative-rest-client--openfeign)
4. [API Gateway — Spring Cloud Gateway](#4-api-gateway--spring-cloud-gateway)
5. [Centralized Config — Spring Cloud Config Server](#5-centralized-config--spring-cloud-config-server)
6. [Resilience — Resilience4j](#6-resilience--resilience4j)
7. [Brief Mentions — Load Balancing & Distributed Tracing](#7-brief-mentions--load-balancing--distributed-tracing)
8. [Component Summary Table](#8-component-summary-table)
9. [Common Interview Questions](#common-interview-questions)
10. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## 1. The Problem Spring Cloud Solves

In a single Spring Boot monolith, everything is one process — you call a method, no network. In microservices, services are **separate processes** on different hosts/ports and those addresses change constantly (containers restart, scale up/down, move). Spring Cloud is a toolkit that solves the operational problems this creates:

| Problem | Spring Cloud Solution |
|---|---|
| How does Service A find Service B's URL? | Eureka (Service Discovery) |
| How do I call Service B cleanly from Java? | OpenFeign (REST Client) |
| How do clients reach any service without knowing ports? | Spring Cloud Gateway (API Gateway) |
| How do I change config across 10 services without redeploying? | Spring Cloud Config Server |
| What happens when Service B is down or slow? | Resilience4j (Circuit Breaker / Retry) |
| Which instance of Service B gets the request? | Spring Cloud LoadBalancer |
| How do I trace a request across 5 services? | Micrometer Tracing |

---

## 2. Service Discovery — Netflix Eureka

### The Problem It Solves

In a static world you'd hardcode `http://order-service:8081`. In Kubernetes or Docker containers, ports and IPs change. **Eureka** is a registry — services register themselves on startup, and other services look up addresses by name at runtime.

> Analogy: Eureka is the company directory. Instead of memorising everyone's desk phone, you look up "John from Payments" and the directory gives you his current number.

### Setting Up the Eureka Server

Add dependency `spring-cloud-starter-netflix-eureka-server`:

```java
// Eureka Server — its own small Spring Boot app
@SpringBootApplication
@EnableEurekaServer          // turns this app into the registry
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml for the Eureka Server
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false   # server doesn't register itself
    fetch-registry: false
```

### Registering a Service (Eureka Client)

Any service that adds `spring-cloud-starter-netflix-eureka-client` auto-registers on startup — no extra annotation needed in modern Spring Cloud.

```yaml
# application.yml of order-service
spring:
  application:
    name: order-service           # this is the name other services use to look it up

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### Looking Up Another Service by Name

You don't call Eureka directly. OpenFeign and Spring Cloud LoadBalancer resolve names automatically using Eureka behind the scenes (see sections 3 and 7).

**Junior bar**: know that `@EnableEurekaServer` creates the registry; services register via config; other services discover them by `spring.application.name`.

---

## 3. Declarative REST Client — OpenFeign

### The Problem It Solves

Without Feign you'd write `RestTemplate` or `WebClient` boilerplate for every inter-service call: build the URL, set headers, handle errors, deserialise JSON. **OpenFeign** lets you declare an interface annotated like a Spring MVC controller and it generates the HTTP client for you.

> Analogy: OpenFeign is like ordering food via an app — you just pick items (call the method); the app handles the actual HTTP request to the restaurant.

### Minimal Setup

Dependency: `spring-cloud-starter-openfeign`

```java
// Enable Feign clients in your main class (or any @Configuration)
@SpringBootApplication
@EnableFeignClients
public class PaymentServiceApplication { ... }
```

### Defining a Feign Client

```java
// Declare an interface — Feign generates the implementation at startup
@FeignClient(name = "order-service")   // "order-service" must match spring.application.name in Eureka
public interface OrderClient {

    // Mirror the endpoint signature from order-service's controller
    @GetMapping("/api/orders/{orderId}")
    OrderDTO getOrder(@PathVariable("orderId") Long orderId);

    @PostMapping("/api/orders")
    OrderDTO createOrder(@RequestBody CreateOrderRequest request);
}
```

### Using the Client in a Service

```java
@Service
public class PaymentService {

    private final OrderClient orderClient;  // Spring injects the Feign-generated proxy

    public PaymentService(OrderClient orderClient) {
        this.orderClient = orderClient;
    }

    public void processPayment(Long orderId) {
        // This looks like a local method call; Feign makes the HTTP GET under the hood
        OrderDTO order = orderClient.getOrder(orderId);
        // ... process payment using order details
    }
}
```

**Junior bar**: This is the component you are most likely to actually write in a junior role. Be able to declare a `@FeignClient` interface, map methods to HTTP endpoints, and inject it like any Spring bean.

### Common Feign Mistakes

- Forgetting `@EnableFeignClients` — results in "No qualifying bean of type 'OrderClient'" at startup.
- Mismatching the `name` in `@FeignClient` with the service's `spring.application.name` — Eureka won't resolve it.
- Not adding a fallback — if `order-service` is down, the call throws an exception unless Resilience4j wraps it (see section 6).

---

## 4. API Gateway — Spring Cloud Gateway

### The Problem It Solves

Without a gateway, clients need to know each service's host and port — and your frontend would make CORS calls to 10 different origins. The **API Gateway** is the single entry point: all traffic enters through port 8080 (for example) and the gateway routes each request to the correct backend service. It also centralises cross-cutting concerns: authentication, rate limiting, logging, and SSL termination.

> Analogy: The gateway is the hotel reception desk — guests (clients) talk to one desk; reception directs them to the right room (service).

### Dependency

`spring-cloud-starter-gateway` — note: this uses Spring WebFlux (reactive), not the servlet stack.

### Basic Route Configuration

```yaml
# application.yml of the gateway service
spring:
  application:
    name: api-gateway

  cloud:
    gateway:
      routes:
        - id: order-service-route
          uri: lb://order-service          # lb:// means "use load balancer + Eureka to resolve"
          predicates:
            - Path=/api/orders/**          # any request to /api/orders/** goes to order-service
          filters:
            - StripPrefix=1                # optional: strip the first path segment before forwarding

        - id: payment-service-route
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
```

### Built-in Filters (Common Ones)

| Filter | What it Does |
|---|---|
| `AddRequestHeader` | Injects a header before forwarding (e.g., correlation ID) |
| `RequestRateLimiter` | Rate-limits by IP or user using Redis |
| `RewritePath` | Rewrites the URL path before forwarding |
| `AuthenticationFilter` (custom) | Validates JWT before allowing the request through |

**Junior bar**: Understand that `uri: lb://service-name` routes to a service discovered via Eureka. Be able to read a route config. Writing custom filters is not expected.

---

## 5. Centralized Config — Spring Cloud Config Server

### The Problem It Solves

If you have 10 microservices each with their own `application.yml`, changing a database URL or feature flag means redeploying all 10. **Spring Cloud Config Server** externalises configuration into a single Git repository. All services pull their config from the Config Server at startup (and optionally refresh at runtime).

> Analogy: Instead of every employee keeping their own policy handbook, the company publishes one handbook on a shared drive — updates appear everywhere immediately.

### Config Server Setup

Dependency: `spring-cloud-config-server`

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication { ... }
```

```yaml
# application.yml for config server
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo   # git repo holding all configs
          default-label: main
```

### Client Service Setup

Dependency: `spring-cloud-starter-config`

```yaml
# bootstrap.yml (or application.yml with spring.config.import) in each service
spring:
  application:
    name: order-service
  config:
    import: optional:configserver:http://localhost:8888
```

The Config Server looks for a file named `order-service.yml` (or `order-service-prod.yml` for the `prod` profile) in the Git repo and serves it to the client.

**Junior bar**: Know the concept — one Git-backed store, all services pull from it. Know that `@EnableConfigServer` sets it up and that services declare `spring.config.import` to consume it.

---

## 6. Resilience — Resilience4j

### The Problem It Solves

In microservices, downstream services fail. If `order-service` is slow or down and `payment-service` keeps calling it, all threads in `payment-service` back up — causing cascading failures that take down the whole system. **Resilience4j** provides patterns to handle this gracefully.

> Analogy: A circuit breaker in your house trips when there's a fault, cutting the circuit to prevent a fire. Resilience4j does the same for service calls.

### Key Resilience4j Patterns

| Pattern | What It Does |
|---|---|
| **Circuit Breaker** | Opens the circuit (stops calls) after X% failures; tries again after a wait |
| **Retry** | Automatically retries a failed call N times with optional backoff |
| **Rate Limiter** | Limits how many calls per second your service makes to a downstream |
| **Bulkhead** | Limits concurrent calls to isolate one slow downstream from affecting others |
| **Time Limiter** | Cancels a call that takes longer than a configured timeout |

Resilience4j **replaced Hystrix** (Netflix, now in maintenance mode).

### Dependency

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<!-- Also needs spring-boot-starter-aop -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### Circuit Breaker with Fallback

```java
@Service
public class PaymentService {

    private final OrderClient orderClient;

    public PaymentService(OrderClient orderClient) {
        this.orderClient = orderClient;
    }

    // @CircuitBreaker wraps this method; if it fails repeatedly, fallbackMethod is called instead
    @CircuitBreaker(name = "orderService", fallbackMethod = "getOrderFallback")
    public OrderDTO getOrder(Long orderId) {
        return orderClient.getOrder(orderId);   // Feign call to order-service
    }

    // Fallback — same return type, extra Throwable parameter
    public OrderDTO getOrderFallback(Long orderId, Throwable ex) {
        // Return a safe default rather than propagating the error
        return new OrderDTO(orderId, "UNKNOWN", BigDecimal.ZERO);
    }
}
```

### Circuit Breaker Configuration in `application.yml`

```yaml
resilience4j:
  circuitbreaker:
    instances:
      orderService:                        # matches name in @CircuitBreaker
        sliding-window-size: 10            # evaluate last 10 calls
        failure-rate-threshold: 50         # open if 50% or more fail
        wait-duration-in-open-state: 10s   # stay open for 10 seconds then try again
        permitted-number-of-calls-in-half-open-state: 3
```

### Retry Example

```java
@Retry(name = "orderService", fallbackMethod = "getOrderFallback")
public OrderDTO getOrder(Long orderId) {
    return orderClient.getOrder(orderId);
}
```

```yaml
resilience4j:
  retry:
    instances:
      orderService:
        max-attempts: 3
        wait-duration: 500ms
```

**Junior bar**: Know that circuit breakers prevent cascading failures; know that `@CircuitBreaker(name=..., fallbackMethod=...)` is the annotation; be able to explain OPEN/CLOSED/HALF-OPEN states. Writing a fallback method is realistic; tuning thresholds is a senior task.

### Circuit Breaker States

```
CLOSED  →  normal, all calls pass through
  ↓ (failure rate exceeds threshold)
OPEN    →  calls immediately return fallback (no network call)
  ↓ (after waitDuration)
HALF-OPEN → a few test calls allowed through
  ↓ (test calls succeed)        ↓ (test calls fail)
CLOSED (back to normal)       OPEN (wait again)
```

---

## 7. Brief Mentions — Load Balancing & Distributed Tracing

### Spring Cloud LoadBalancer

**Problem**: When Eureka returns 3 instances of `order-service`, which one do you call?  
**Solution**: Spring Cloud LoadBalancer (the modern replacement for Netflix Ribbon) automatically distributes calls across instances using round-robin by default. It is used transparently by Feign and Gateway — you don't interact with it directly.

For deeper reading, see **System_Design_Microservices_Interview_Questions.md**.

### Micrometer Tracing (Distributed Tracing)

**Problem**: A slow API call touches 5 services. How do you trace which service was the bottleneck?  
**Solution**: Micrometer Tracing (formerly Spring Cloud Sleuth) injects a `traceId` and `spanId` into every log line and propagates them across service calls via HTTP headers. You visualise traces in Zipkin or Jaeger.

For setup, spans, and Zipkin/Jaeger integration see **Observability_Tracing_Metrics_Logging.md**.

---

## 8. Component Summary Table

| Component | Dependency Artifact | Problem It Solves | Key Annotation / Config |
|---|---|---|---|
| Eureka Server | `spring-cloud-starter-netflix-eureka-server` | Service registry — services find each other by name | `@EnableEurekaServer` |
| Eureka Client | `spring-cloud-starter-netflix-eureka-client` | Auto-register with the registry | `spring.application.name` |
| OpenFeign | `spring-cloud-starter-openfeign` | Declarative inter-service HTTP calls | `@FeignClient`, `@EnableFeignClients` |
| Spring Cloud Gateway | `spring-cloud-starter-gateway` | Single entry point, routing, auth/rate-limit filters | `spring.cloud.gateway.routes` in YAML |
| Config Server | `spring-cloud-config-server` | Centralised external config from Git | `@EnableConfigServer` |
| Config Client | `spring-cloud-starter-config` | Pull config from Config Server at startup | `spring.config.import` |
| Resilience4j | `resilience4j-spring-boot3` | Circuit breaker, retry, rate limiter, bulkhead | `@CircuitBreaker`, `@Retry` |
| Spring Cloud LoadBalancer | included with Eureka client | Client-side load balancing across instances | Transparent — used by Feign/Gateway |
| Micrometer Tracing | `micrometer-tracing-bridge-*` | Distributed trace propagation across services | Auto-configured; view in Zipkin/Jaeger |

---

## Common Interview Questions

**Q: What is service discovery and why is it needed in microservices?**  
In microservices, services run in containers whose IPs and ports change dynamically. Service discovery (Eureka) lets each service register its address at startup and lets other services look it up by name at runtime, so you never hardcode URLs. Without it, a single IP change would break all callers.

**Q: What is an API Gateway and what does it do?**  
An API Gateway is a single entry point for all client traffic. It routes requests to the correct backend service using path-based rules (e.g., `/api/orders/**` → order-service). It also centralises cross-cutting concerns such as authentication, rate limiting, and CORS, so individual services don't each implement them.

**Q: How does OpenFeign differ from RestTemplate?**  
RestTemplate requires you to manually build URLs, set headers, and handle serialisation in every call. OpenFeign lets you declare an interface with Spring MVC annotations — Feign generates the HTTP client implementation. It also integrates with Eureka (resolves service names automatically) and Resilience4j (adds circuit breaking), making it cleaner for service-to-service calls.

**Q: What is a circuit breaker and what problem does it solve?**  
A circuit breaker monitors calls to a downstream service. When the failure rate exceeds a threshold it "opens" — subsequent calls immediately return a fallback response instead of making a network call. This prevents one slow or failed service from exhausting threads and cascading failures through the whole system. After a wait period, it transitions to HALF-OPEN to test if the downstream has recovered.

**Q: What replaced Hystrix in Spring Cloud?**  
Resilience4j replaced Hystrix, which Netflix put into maintenance mode. Resilience4j is modular (circuit breaker, retry, rate limiter, bulkhead, time limiter are separate), has no external dependencies, and integrates natively with Spring Boot 3 via `resilience4j-spring-boot3`.

**Q: What is Spring Cloud Config Server?**  
It is a centralised configuration service backed by a Git repository. All microservices pull their configuration from it at startup instead of bundling `application.yml` files individually. Changing a property in Git (and triggering a refresh) propagates the change to all services without redeployment.

---

## Quick Reference Cheat Sheet

```
EUREKA
  Server:  @EnableEurekaServer | port 8761 | register-with-eureka: false
  Client:  spring.application.name: my-service | service-url.defaultZone: http://localhost:8761/eureka/

OPENFEIGN
  Enable:  @EnableFeignClients on @SpringBootApplication
  Declare: @FeignClient(name = "target-service")
           @GetMapping("/path") ResponseType method(@PathVariable id);
  Inject:  Constructor-inject the interface — Spring provides the proxy

GATEWAY
  Route:   uri: lb://service-name  (lb = load balanced via Eureka)
           predicates: Path=/api/orders/**
  Filters: AddRequestHeader, RequestRateLimiter, RewritePath

CONFIG SERVER
  Server:  @EnableConfigServer | git.uri: https://github.com/org/config-repo
  Client:  spring.config.import: optional:configserver:http://localhost:8888

RESILIENCE4J
  Circuit Breaker: @CircuitBreaker(name = "svc", fallbackMethod = "fallback")
  Retry:           @Retry(name = "svc", fallbackMethod = "fallback")
  Config:          resilience4j.circuitbreaker.instances.svc.failure-rate-threshold: 50
  States:          CLOSED → OPEN → HALF-OPEN → CLOSED

LOAD BALANCER
  Transparent — used internally by Feign (lb://) and Gateway (lb://)
  Round-robin by default; no annotation needed

TRACING
  Add micrometer-tracing bridge → traceId/spanId auto-injected in logs
  Visualise in Zipkin (http://localhost:9411) or Jaeger
  See: Observability_Tracing_Metrics_Logging.md
```

---

*Last Updated: 2026-06-18*

# Spring Cloud Microservices — Awareness Notes

> **Scope note (junior job prep):** Spring Cloud tooling (Eureka, Gateway, Config Server, Resilience4j) is **microservices infrastructure a junior rarely builds — deferred for later study**. Most junior Spring jobs are single-service apps. The junior-level microservices *awareness* (what an API gateway / service discovery / circuit breaker is, monolith vs microservices) is kept in **`System_Design_Microservices_Interview_Questions.md`**. The full Spring Cloud component guide remains in git history.

---

## Components to Recognize (one-liner each)

| Component | What it does |
|---|---|
| **Eureka** | Service discovery — services register themselves so others can find them by name instead of hardcoded IPs |
| **OpenFeign** | Declarative REST client — call another service via a Java interface (the one bit a junior might actually write) |
| **Spring Cloud Gateway** | API gateway — single entry point that routes requests to the right service, handles auth/rate-limiting |
| **Config Server** | Centralized external configuration for all services |
| **Resilience4j** | Circuit breaker / retry / rate limiter — stops cascading failures when a downstream service is down |

The one piece you might touch as a junior — a Feign client:

```java
@FeignClient(name = "order-service")
public interface OrderClient {
    @GetMapping("/api/orders/user/{userId}")
    List<Order> getOrdersByUser(@PathVariable Long userId);
}
```

> **Interview soundbite:** "I know Spring Cloud provides the microservices plumbing — Eureka for discovery, Gateway for routing, Config Server for central config, Resilience4j for circuit breaking, and Feign for service-to-service calls. I've built single Spring Boot services; wiring a full microservices topology is something I'd grow into."

---

*Trimmed to awareness level for junior job prep. Restore the full Spring Cloud guide from version control when you're ready to study it.*

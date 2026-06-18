# Embedded Tomcat in Spring Boot — Study Guide

## Overview

Every Spring Boot web app you run with `java -jar app.jar` starts a **web server inside itself** — almost always **Apache Tomcat**. Understanding Tomcat separates "I can write a controller" from "I understand how my HTTP request gets handled" — and interviewers probe the second part.

This guide covers what Tomcat is, why Spring Boot embeds it, how a request flows through it, and the key settings you'll be asked to tune.

---

## Table of Contents

1. [What Is Tomcat?](#what-is-tomcat)
2. [Servlet, Servlet Container & the Servlet API](#servlet-servlet-container--the-servlet-api)
3. [Embedded vs External (Traditional) Tomcat](#embedded-vs-external-traditional-tomcat)
4. [How Spring Boot Starts Tomcat](#how-spring-boot-starts-tomcat)
5. [The Request Lifecycle (End to End)](#the-request-lifecycle-end-to-end)
6. [Tomcat's Thread Pool — The Most Important Concept](#tomcats-thread-pool--the-most-important-concept)
7. [Key Configuration Properties](#key-configuration-properties)
8. [DispatcherServlet — Where Tomcat Hands Off to Spring](#dispatcherservlet--where-tomcat-hands-off-to-spring)
9. [The Other Servers — Jetty, Undertow & Netty](#the-other-servers--jetty-undertow--netty)
10. [Connectors, Ports & HTTPS](#connectors-ports--https)
11. [Common Mistakes & Pitfalls](#common-mistakes--pitfalls)
12. [Common Interview Questions](#common-interview-questions)
13. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What Is Tomcat?

**Apache Tomcat** is a **web server** and **servlet container**. It listens on a port, accepts HTTP requests, and hands them to your Java code to produce a response.

**Analogy:** Tomcat is the restaurant building, front door, and waiters. Your Spring controllers are the kitchen. Tomcat doesn't cook — it routes requests to the kitchen and responses back to customers.

Without a server like Tomcat, your Java code has no way to hear HTTP requests. Plain Java has no built-in HTTP listener for web apps.

---

## Servlet, Servlet Container & the Servlet API

- **Servlet** — a Java class that handles HTTP requests. Has a `service()` method that receives a request and writes a response.
- **Servlet Container** — runs servlets, manages their lifecycle, feeds them requests. **Tomcat is a servlet container.**
- **Servlet API** — standard interfaces (`HttpServletRequest`, `HttpServletResponse`) that containers and servlets agree on.

**Analogy:** The Servlet API is the standard socket shape. Tomcat is the wall socket. A servlet is any appliance you plug in. Because everyone agrees on the shape, you can swap Tomcat for Jetty without changing your code.

```java
// A raw servlet — you almost NEVER write this in Spring Boot,
// but it shows what Spring is doing for you under the hood.
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.setContentType("text/plain");
        resp.getWriter().write("Hello World");
    }
}
```

> In Spring Boot you write `@RestController` classes, not servlets. Under the hood, Spring registers **one** master servlet (the `DispatcherServlet`) with Tomcat, which routes requests to your controllers.

**Jakarta vs javax:** The servlet package was renamed from `javax.servlet` to `jakarta.servlet` in Spring Boot 3 / Tomcat 10+. Older code uses `javax.servlet.*`.

---

## Embedded vs External (Traditional) Tomcat

**The old way (External Tomcat):**
1. Build a **WAR** file. Install Tomcat separately on a server. Deploy the WAR into `webapps/`. One Tomcat can host several WARs.

**The Spring Boot way (Embedded Tomcat):**
1. Tomcat is a **JAR dependency** bundled inside your app. Build one executable ("fat") JAR. Run `java -jar app.jar` — the app starts Tomcat itself.

| Aspect | Embedded (Spring Boot default) | External / Traditional |
|---|---|---|
| Packaging | Executable JAR (fat JAR) | WAR file |
| Server install | None — bundled in the JAR | Install & manage Tomcat separately |
| Run command | `java -jar app.jar` | Deploy WAR, start Tomcat |
| Apps per server | One app = one process | Many WARs in one Tomcat |
| Best for | Microservices, containers, cloud | Legacy multi-app shared servers |
| Version control | Pinned in `pom.xml` | Set by ops team |

> Embedded is the default and the modern choice — it fits perfectly into Docker containers (one app, one process, one port).

---

## How Spring Boot Starts Tomcat

Adding `spring-boot-starter-web` pulls in embedded Tomcat as a transitive dependency.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- pulls in spring-boot-starter-tomcat → tomcat-embed-core -->
</dependency>
```

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
        // 1. Creates the Spring application context
        // 2. Detects a web starter on the classpath
        // 3. Auto-configures and STARTS embedded Tomcat on port 8080
    }
}
```

Spring Boot's auto-configuration detects `tomcat-embed-core` on the classpath and creates a `TomcatServletWebServerFactory` bean that builds and starts Tomcat. You never call Tomcat APIs directly.

> **Takeaway:** "It's on the classpath" is why Tomcat starts. Remove the web starter and no server starts.

---

## The Request Lifecycle (End to End)

```
1. Browser sends:  GET /users/42  →  to your server's IP on port 8080
        │
        ▼
2. Tomcat's CONNECTOR is listening on 8080. It accepts the TCP connection
   and parses the raw bytes into an HttpServletRequest object.
        │
        ▼
3. Tomcat grabs a worker THREAD from its thread pool to handle this request.
   (If all threads are busy, the request waits in the accept queue.)
        │
        ▼
4. Tomcat passes the request to the DispatcherServlet (Spring's front controller).
        │
        ▼
5. DispatcherServlet asks HandlerMapping: "which controller handles GET /users/42?"
   → finds UserController.getUser(42)
        │
        ▼
6. Your controller runs (calls services, DB, etc.) and returns a User object.
        │
        ▼
7. Jackson converts the User object to JSON.
        │
        ▼
8. Tomcat sends the HTTP response back, then RETURNS the worker thread to
   the pool so it can handle the next request.
```

**Critical detail:** By default, **one request = one thread, held for the entire request duration** (the "thread-per-request" model). This is why thread pool size directly caps how many requests you can handle at once.

---

## Tomcat's Thread Pool — The Most Important Concept

Tomcat keeps a pool of **worker threads**. Each request borrows one thread for its whole lifetime, then returns it. Pool size caps your concurrency.

```properties
server.tomcat.threads.max=200       # max concurrent requests processed
server.tomcat.threads.min-spare=10  # threads kept alive when idle (for burst readiness)
server.tomcat.accept-count=100      # waiting queue when all 200 threads are busy
server.tomcat.max-connections=8192  # max simultaneous connections accepted
server.tomcat.connection-timeout=20000  # ms before giving up on a slow/idle connection
```

**Beyond `threads.max`:** requests wait in the accept queue. **Beyond `threads.max + accept-count`:** connection refused.

### The blocking trap

In thread-per-request, a thread is **stuck** for the whole request — including time spent waiting on a slow DB or external API.

```
If every request calls a slow API that takes 2 seconds:
  - Each thread is blocked for 2 full seconds.
  - With 200 threads → only ~100 requests/second throughput.
  - Request #201+ waits, then eventually times out.
```

Don't blindly raise `threads.max` to fix slowness — each thread costs ~512KB–1MB of stack memory. Fix the slow downstream call first.

---

## Key Configuration Properties

```properties
# ---- Networking ----
server.port=8080                        # port Tomcat listens on (0 = random, great for tests)
server.address=0.0.0.0                  # bind to all network interfaces

# ---- Context path ----
server.servlet.context-path=/api        # prefix ALL endpoints: /users → /api/users

# ---- Thread pool ----
server.tomcat.threads.max=200
server.tomcat.threads.min-spare=10
server.tomcat.accept-count=100

# ---- Timeouts ----
server.tomcat.connection-timeout=20000  # ms; slow-client guard
server.tomcat.keep-alive-timeout=60000  # ms a connection stays open for reuse

# ---- Limits ----
server.tomcat.max-http-form-post-size=2MB
server.max-http-request-header-size=8KB

# ---- Access logs ----
server.tomcat.accesslog.enabled=true
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b

# ---- Graceful shutdown ----
server.shutdown=graceful                # finish in-flight requests before stopping
spring.lifecycle.timeout-per-shutdown-phase=30s
```

### Programmatic customization (when properties aren't enough)

```java
@Component
public class TomcatCustomizer
        implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {
    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.setPort(9090);
        factory.addConnectorCustomizers(connector ->
            connector.setProperty("maxThreads", "300")
        );
    }
}
```

---

## DispatcherServlet — Where Tomcat Hands Off to Spring

Tomcat doesn't know anything about `@RestController`. The bridge between Tomcat and Spring MVC is the **`DispatcherServlet`** — Spring's single front controller.

```
Tomcat (servlet container)
   └── registers ONE servlet:  DispatcherServlet  (mapped to "/")
          └── HandlerMapping     → finds which @Controller method matches the URL
          └── HandlerAdapter     → invokes that method
          └── HttpMessageConverter (Jackson) → object ⇄ JSON
          └── ViewResolver       → (for server-rendered HTML; skipped for REST)
```

Spring Boot auto-registers the `DispatcherServlet` — you don't configure it manually.

> **Tomcat** handles network + threads + HTTP parsing. **DispatcherServlet** handles routing to your code. They're separate layers — a frequent interview confusion point.

---

## The Other Servers — Jetty, Undertow & Netty

Spring Boot officially supports four embedded servers. Knowing why you'd swap is a classic "do you understand it's pluggable?" question.

### Servlet servers (Tomcat, Jetty, Undertow)

All implement the Servlet API — fully interchangeable for Spring MVC apps.

- **Tomcat** — the default. Battle-tested, largest community, safest choice.
- **Jetty** — lightweight, long history of embedding, fine-grained programmatic config.
- **Undertow** — very lightweight, high throughput (JBoss/Red Hat). Good for high concurrency with low memory.

### Reactive server (Netty)

**Netty** is not a servlet container — it's an asynchronous, non-blocking, event-driven framework. It's the default server for **Spring WebFlux**, not Spring MVC. A small number of threads handle thousands of connections because no thread blocks waiting. You get Netty automatically by using the reactive starter:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### Switching servlet servers (Tomcat → Jetty/Undertow)

```xml
<!-- Step 1: Exclude Tomcat -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- Step 2: Add the server you want -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>  <!-- or -starter-jetty -->
</dependency>
```

No code changes needed — that's the whole point of the Servlet API standard.

| Server | Type | Character | Pick when |
|---|---|---|---|
| **Tomcat** | Servlet | Default, battle-tested | Almost always |
| **Jetty** | Servlet | Lightweight, embeddable | Fine-grained control |
| **Undertow** | Servlet | Memory-efficient, high conns | High-throughput, low memory |
| **Netty** | Reactive | Non-blocking event loop | WebFlux / massive concurrency |

> Key interview point: you can swap servlet servers because they all implement the Servlet API. Netty is the odd one out — you get it by choosing the WebFlux stack, not by excluding Tomcat.

---

## Connectors, Ports & HTTPS

A **connector** is the Tomcat component that owns a network port and protocol. The default is one HTTP connector on port 8080.

```properties
server.port=8443
server.ssl.enabled=true
server.ssl.key-store=classpath:keystore.p12
server.ssl.key-store-password=changeit
server.ssl.key-store-type=PKCS12
```

**Real-world note:** In production behind a load balancer or reverse proxy (Nginx, AWS ALB), TLS is often terminated at the proxy. Set `server.forward-headers-strategy=framework` so Spring correctly reads the original client IP and scheme from `X-Forwarded-*` headers.

---

## Common Mistakes & Pitfalls

- **"Port 8080 is already in use."** A previous run still holds the port. Kill it, or use `server.port=0` in tests.
- **Cranking `threads.max` to "fix" slowness.** More threads ≠ faster. Fix the slow downstream call — don't just create thousands of blocked threads and run out of memory.
- **Confusing the accept queue with the thread pool.** `accept-count` is the waiting line; `threads.max` is the number of workers.
- **Forgetting graceful shutdown in containers.** Without `server.shutdown=graceful`, a Kubernetes pod restart drops in-flight requests mid-response.
- **Assuming you still build WARs.** Modern Spring Boot ships executable JARs with embedded Tomcat. WAR is only for legacy external servers.
- **Thinking `DispatcherServlet` IS Tomcat.** Tomcat is the server; DispatcherServlet is Spring's router running *inside* it.
- **Blocking the request thread on long work.** A 30-second job in a controller holds a thread for 30 seconds. Offload to `@Async` or return `202 Accepted`.

---

## Common Interview Questions

**Q1: What is embedded Tomcat and why does Spring Boot use it?**
Embedded Tomcat is Tomcat bundled as a library inside your JAR, started by your app at runtime. Spring Boot uses it so apps are self-contained — build one executable JAR and run it with `java -jar`, no separate server install needed. The Tomcat version is pinned in `pom.xml` alongside your code.

**Q2: WAR vs JAR — what changed?**
Traditionally you built a WAR and deployed it into an externally-installed Tomcat. Spring Boot builds an executable fat JAR containing your code and the server. One JAR = one self-running app. WARs are only needed for legacy shared/external servers.

**Q3: What is a servlet container?**
A program that runs servlets, manages their lifecycle, and feeds them HTTP requests via the Servlet API. Tomcat is the most common one. In Spring Boot, all requests pass through a single servlet (the DispatcherServlet) that Tomcat hosts.

**Q4: Walk me through what happens when a request hits a Spring Boot app.**
Tomcat's connector accepts the connection on port 8080 and parses the HTTP bytes into an `HttpServletRequest`. A worker thread is grabbed from the pool. The request goes to the DispatcherServlet, which finds the matching controller method, invokes it, serializes the response to JSON via Jackson, and Tomcat sends it back — then returns the thread to the pool.

**Q5: How does Tomcat handle concurrent requests?**
Thread-per-request: a pool (default max 200) of worker threads each handle one request for its whole duration. Up to `threads.max` run concurrently; extras wait in the accept queue (`accept-count` = 100); beyond that, connections are refused.

**Q6: What's the difference between `threads.max` and `accept-count`?**
`threads.max` is how many requests are *actively processed* at once. `accept-count` is the *waiting queue* used when all workers are busy. Workers = staff serving customers; accept-count = the queue waiting to be served.

**Q7: How do you change the port / why set it to 0?**
`server.port=8080` in `application.properties`. Setting it to `0` makes Tomcat pick a random free port — useful in integration tests so parallel runs don't collide.

**Q8: What other servers can Spring Boot use, and how do you switch?**
Four options: **Tomcat** (default), **Jetty**, and **Undertow** are servlet containers for Spring MVC; **Netty** is the non-blocking server for Spring WebFlux. Swap servlet servers by excluding `spring-boot-starter-tomcat` and adding the one you want — no code changes, since they all implement the Servlet API. Get Netty by using `spring-boot-starter-webflux` instead of `-web`.

**Q9: What is the DispatcherServlet and how does it relate to Tomcat?**
It's Spring MVC's front controller — the single servlet Tomcat hosts. Tomcat handles networking and threads; it forwards every request to the DispatcherServlet, which routes it to the right `@Controller` method. They're separate layers working together.

**Q10: Your app is slow under load — what Tomcat-related things do you check?**
Are worker threads blocked on slow DB/external calls? (Fix the slow call first.) Is the accept queue overflowing? Are timeouts configured sensibly? Is the pool sized for available memory? For heavily I/O-bound, high-concurrency workloads consider reactive WebFlux.

**Q11: How do you make Tomcat shut down without dropping requests?**
Set `server.shutdown=graceful` and `spring.lifecycle.timeout-per-shutdown-phase`. Tomcat stops accepting new requests but lets in-flight ones finish within the timeout — essential for zero-downtime deploys in Kubernetes.

---

## Quick Reference Cheat Sheet

```
WHAT IS IT
  Tomcat        → web server + servlet container (listens on a port, runs servlets)
  Servlet       → Java class that handles an HTTP request (low-level)
  Container     → program that runs servlets (Tomcat IS one)
  Servlet API   → standard interfaces (HttpServletRequest/Response) — the "socket shape"

EMBEDDED vs EXTERNAL
  Embedded (default) → Tomcat is a JAR inside your app; build fat JAR; java -jar app.jar
  External           → install Tomcat separately; deploy a WAR; many WARs per server

WHY IT STARTS
  spring-boot-starter-web → pulls in spring-boot-starter-tomcat → tomcat-embed-core
  On classpath → auto-config creates TomcatServletWebServerFactory → starts Tomcat :8080

REQUEST FLOW
  Connector(:8080) → worker thread → DispatcherServlet → HandlerMapping
    → @Controller method → Jackson (object→JSON) → response → thread returned to pool
  Model: ONE request = ONE thread, held for the whole request (thread-per-request)

THREAD POOL (the big one)
  server.tomcat.threads.max=200        → max concurrent requests processed
  server.tomcat.threads.min-spare=10   → idle threads kept ready
  server.tomcat.accept-count=100       → waiting queue when all threads busy
  server.tomcat.max-connections=8192   → max simultaneous connections
  Beyond max → queue; beyond max+accept-count → connection refused
  More threads ≠ faster. Fix slow downstream calls first.

KEY PROPERTIES
  server.port=8080                        → listen port (0 = random, great for tests)
  server.servlet.context-path=/api        → prefix all endpoints
  server.tomcat.connection-timeout=20000  → slow-client guard (ms)
  server.shutdown=graceful                → finish in-flight requests on shutdown
  server.ssl.enabled=true + key-store     → enable HTTPS

DISPATCHERSERVLET
  Spring's single front-controller servlet that Tomcat hosts
  Tomcat = network + threads;  DispatcherServlet = route to your @Controller

THE OTHER SERVERS (Boot supports 4)
  Tomcat   → servlet, default, battle-tested        (Spring MVC)
  Jetty    → servlet, lightweight, embeddable       (Spring MVC)
  Undertow → servlet, memory-efficient, high conns  (Spring MVC)
  Netty    → REACTIVE, non-blocking event loop      (Spring WebFlux default)
  Swap servlet servers: exclude -tomcat, add -jetty/-undertow (no code change).
  Get Netty: use spring-boot-starter-webflux instead of -web.

PACKAGE RENAME
  javax.servlet.*   → Spring Boot 2 / Tomcat 9
  jakarta.servlet.* → Spring Boot 3 / Tomcat 10+
```

---

*Last Updated: 2026-06-18*

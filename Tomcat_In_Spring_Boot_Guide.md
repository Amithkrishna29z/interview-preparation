# Embedded Tomcat in Spring Boot — Study Guide

## Overview

Every Spring Boot web app you run with `java -jar app.jar` is secretly starting a **web server inside itself**. That server is almost always **Apache Tomcat**. Understanding Tomcat is what separates "I can write a controller" from "I understand how my HTTP request actually gets handled" — and that second part is what interviewers probe.

This guide explains what Tomcat is, why Spring Boot embeds it, how a request flows through it, and the handful of settings (thread pool, timeouts, port) you will be asked to tune and explain.

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

**Apache Tomcat** is a **web server** and **servlet container**. Its job is simple to state: listen on a network port, accept incoming HTTP requests, and hand them to your Java code so you can produce a response.

**Think of it like a restaurant:**

- **Tomcat** = the **building, the front door, and the waiters**. It receives customers (HTTP requests), seats them, and carries orders back and forth.
- **Your Spring code** (controllers, services) = the **kitchen**. It does the actual cooking (business logic).
- Tomcat does **not** cook. It just makes sure requests get to the kitchen and responses get back to the customer — reliably, and many at once.

Without a server like Tomcat, your Java code has no way to "hear" HTTP requests coming over the network. Plain Java has no built-in HTTP listener for web apps — Tomcat fills that gap.

---

## Servlet, Servlet Container & the Servlet API

These three words come up constantly. Here's the plain-English version.

- **Servlet** = a Java class that handles HTTP requests. The original, low-level way to write web code in Java. It has a `service()` method that receives a request and writes a response.
- **Servlet Container** = the program that **runs** servlets, manages their lifecycle, and feeds them requests. **Tomcat is a servlet container.**
- **Servlet API** = the standard set of interfaces (`HttpServletRequest`, `HttpServletResponse`, etc.) that the container and your servlets agree to speak.

**Think of it like electrical sockets:** The **Servlet API** is the standard socket shape. **Tomcat** is the wall socket (the container) providing power. A **servlet** is any appliance you plug in. Because everyone agrees on the socket shape, you can swap the wall socket (Tomcat → Jetty) without rewiring your appliance.

```java
// A raw servlet — you almost NEVER write this in Spring Boot,
// but knowing it exists explains what Spring is doing for you.
public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.setContentType("text/plain");        // set response header
        resp.getWriter().write("Hello World");     // write the response body
    }
}
```

> **Key insight:** In Spring Boot you write `@RestController` classes, not servlets. But under the hood, Spring registers **one** master servlet (the `DispatcherServlet`) with Tomcat, and that servlet routes requests to your controllers. More on this below.

**Jakarta vs javax note:** The servlet package was renamed from `javax.servlet` to `jakarta.servlet` (Spring Boot 3 / Tomcat 10+). If you see `import javax.servlet.*` it's an older codebase (Spring Boot 2 / Tomcat 9).

---

## Embedded vs External (Traditional) Tomcat

This is the single most important distinction to understand for interviews.

**The old way (External Tomcat):**
1. You build a **WAR** file (Web Application aRchive).
2. You install Tomcat separately on a server.
3. You **deploy** the WAR into Tomcat's `webapps/` folder.
4. Tomcat runs your app. One Tomcat can host several WARs.

**The Spring Boot way (Embedded Tomcat):**
1. Tomcat is just a **library (a JAR dependency)** bundled inside your app.
2. You build a single **executable JAR** (a "fat JAR") containing your code *and* Tomcat.
3. You run `java -jar app.jar`. Your app **starts Tomcat itself**, from inside `main()`.
4. The server lives and dies with the application.

**Think of it like transport:**
- **External Tomcat** = taking a **public bus**. The bus (server) exists independently; you and other passengers (WARs) board it. You depend on the bus being there and configured correctly.
- **Embedded Tomcat** = owning your **own car**. The engine (Tomcat) is built into the vehicle (your app). You turn the key (`java -jar`) and go. Self-contained.

| Aspect | Embedded (Spring Boot default) | External / Traditional |
|---|---|---|
| Packaging | Executable JAR (fat JAR) | WAR file |
| Server install | None — bundled in the JAR | Install & manage Tomcat separately |
| Run command | `java -jar app.jar` | Deploy WAR, start Tomcat |
| Apps per server | One app = one process | Many WARs in one Tomcat |
| Best for | Microservices, containers, cloud | Legacy multi-app shared servers |
| Version control | Tomcat version pinned in `pom.xml` | Tomcat version set by ops team |

> Embedded is the default and the modern choice — it's why Spring Boot apps "just run" and fit perfectly into Docker containers (one app, one process, one port).

---

## How Spring Boot Starts Tomcat

When you add `spring-boot-starter-web`, you automatically get embedded Tomcat as a transitive dependency.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- This pulls in spring-boot-starter-tomcat behind the scenes -->
</dependency>
```

You can confirm it's there: `spring-boot-starter-web` depends on `spring-boot-starter-tomcat`, which depends on `tomcat-embed-core`.

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
        // ↑ This single line:
        //   1. Creates the Spring application context
        //   2. Sees that a web starter is on the classpath
        //   3. Auto-configures and STARTS an embedded Tomcat
        //   4. Tomcat begins listening on port 8080 (the default)
    }
}
```

**What makes this happen:** Spring Boot's auto-configuration detects that a servlet web server class (`tomcat-embed-core`) is on the classpath and creates a `ServletWebServerFactory` bean (specifically `TomcatServletWebServerFactory`). That factory builds and starts the Tomcat instance. You never call Tomcat APIs directly — auto-configuration does it for you.

> **One-line takeaway:** "It's on the classpath" is *why* Tomcat starts. Remove the web starter and no server starts at all (it becomes a plain app).

---

## The Request Lifecycle (End to End)

This is the flow interviewers love. Walk it slowly.

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
5. DispatcherServlet asks the HandlerMapping: "which controller method
   handles GET /users/42?"  →  finds UserController.getUser(42)
        │
        ▼
6. Your controller runs (calls services, DB, etc.) and returns a User object.
        │
        ▼
7. A message converter (Jackson) turns the User object into JSON.
        │
        ▼
8. DispatcherServlet writes JSON into the HttpServletResponse.
        │
        ▼
9. Tomcat sends the HTTP response back over the connection, then RETURNS the
   worker thread to the pool so it can handle the next request.
```

**Think of it like a call center:** The phone lines (the **connector**) are always ringing. A pool of agents (**worker threads**) is on standby. When a call comes in, a free agent picks it up (one thread per request), looks up who can help (**DispatcherServlet → controller**), handles it, hangs up, and goes back to standby for the next call. If every agent is busy, new callers wait on hold (the **accept queue**). If the hold queue is full too, callers get a busy signal (connection refused).

**The critical detail:** by default, **one request = one thread, held for the entire duration of that request** (the "thread-per-request" model). This is why the thread pool size directly limits how many requests you can handle at once.

---

## Tomcat's Thread Pool — The Most Important Concept

If you remember one tunable thing about Tomcat, make it this.

Tomcat keeps a pool of **worker threads**. Each incoming request borrows one thread for its whole lifetime, then gives it back. The pool size caps your concurrency.

```properties
# application.properties — Tomcat thread pool settings (with Spring Boot defaults)

server.tomcat.threads.max=200          # max worker threads. Up to 200 requests
                                       # can be processed AT THE SAME TIME.

server.tomcat.threads.min-spare=10     # threads kept alive even when idle,
                                       # ready to handle sudden traffic instantly.

server.tomcat.accept-count=100         # the WAITING QUEUE size. If all 200 threads
                                       # are busy, up to 100 more requests wait here.

server.tomcat.max-connections=8192     # max simultaneous CONNECTIONS Tomcat accepts
                                       # (connections waiting + being processed).

server.tomcat.connection-timeout=20000 # ms to wait for the request data to arrive
                                       # before giving up on a slow/idle connection.
```

**Think of it like a restaurant with 200 tables (`threads.max`):**
- 200 diners can eat simultaneously.
- If full, up to 100 more can wait in the lobby (`accept-count`).
- If the lobby is also full, the next person is turned away at the door (connection refused).
- 10 tables (`min-spare`) are always set and ready so the first guests don't wait for setup.

### Why this matters: the blocking trap

In the thread-per-request model, a thread is **stuck** for the entire request — including time spent waiting on a slow database query or a slow external API call.

```
If every request calls a slow API that takes 2 seconds:
  - Each thread is blocked for 2 full seconds.
  - With 200 threads, you can only finish ~100 requests/second.
  - Request #201+ waits in the queue, then eventually times out.
```

This is the #1 reason interviewers bring up **reactive / WebFlux** (which uses few threads and doesn't block) — but for most apps, simply right-sizing the Tomcat pool and keeping downstream calls fast is enough. Don't blindly crank `threads.max` to 5000: each thread costs ~512KB–1MB of stack memory, and too many threads cause CPU context-switching overhead.

---

## Key Configuration Properties

All Tomcat tuning in Spring Boot is done through `application.properties` / `application.yml` — you almost never touch a `server.xml` file (that's the external-Tomcat world).

```properties
# ---- Networking ----
server.port=8080                       # port Tomcat listens on. Use 0 for a RANDOM
                                       # free port (handy in tests).
server.address=0.0.0.0                 # bind to all network interfaces (default).

# ---- Context path ----
server.servlet.context-path=/api       # prefix ALL endpoints. /users becomes /api/users.

# ---- Thread pool (see section above) ----
server.tomcat.threads.max=200
server.tomcat.threads.min-spare=10
server.tomcat.accept-count=100

# ---- Timeouts ----
server.tomcat.connection-timeout=20000 # ms; slow-client / idle-connection guard.
server.tomcat.keep-alive-timeout=60000 # ms a connection stays open for reuse.

# ---- Limits (security & stability) ----
server.tomcat.max-http-form-post-size=2MB   # cap request body size for form posts.
server.max-http-request-header-size=8KB     # cap header size (anti-abuse).

# ---- Access logs (who hit my server?) ----
server.tomcat.accesslog.enabled=true
server.tomcat.accesslog.directory=logs
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b
                                       # %h=client IP, %t=time, %r=request line,
                                       # %s=status code, %b=bytes sent.

# ---- Graceful shutdown ----
server.shutdown=graceful               # finish in-flight requests before stopping.
spring.lifecycle.timeout-per-shutdown-phase=30s   # how long to wait for them.
```

> **YAML equivalent** (same keys, nested):
> ```yaml
> server:
>   port: 8080
>   tomcat:
>     threads:
>       max: 200
>       min-spare: 10
>     accept-count: 100
> ```

### Programmatic customization (when properties aren't enough)

```java
@Component
public class TomcatCustomizer
        implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.setPort(9090);                 // override the port in code
        factory.addConnectorCustomizers(connector ->
            connector.setProperty("maxThreads", "300")  // raw Tomcat connector tuning
        );
    }
}
```

This bean is auto-detected by Spring Boot and applied while building the embedded server. Use it only for settings not exposed as simple properties.

---

## DispatcherServlet — Where Tomcat Hands Off to Spring

Tomcat doesn't know anything about `@RestController`. The bridge between Tomcat (servlet world) and Spring MVC (controller world) is the **`DispatcherServlet`**.

**Think of it like a mailroom in a big office building:** Tomcat is the **post office** that delivers all mail to the building's **single mailroom** (`DispatcherServlet`). The mailroom clerk reads each envelope's address (`URL + HTTP method`) and routes it to the correct department (`@Controller` method). Every letter goes through the one mailroom first — that's why it's called the **front controller**.

```
Tomcat (servlet container)
   └── registers ONE servlet:  DispatcherServlet  (mapped to "/")
          └── HandlerMapping     → finds which @Controller method matches the URL
          └── HandlerAdapter     → invokes that controller method
          └── HttpMessageConverter (Jackson) → object ⇄ JSON
          └── ViewResolver       → (for server-rendered HTML; skipped for REST/JSON)
```

Spring Boot auto-registers the `DispatcherServlet` for you — you don't configure it manually. So the full picture is:

> **Tomcat** handles the *network + threads + HTTP parsing*. **DispatcherServlet** handles the *routing to your code*. They're partners, not the same thing — a frequent point of confusion in interviews.

---

## The Other Servers — Jetty, Undertow & Netty

Tomcat is the default, but it's not the only option. Spring Boot officially supports **four** embedded servers. Knowing they exist — and *why* you'd swap — is a favorite "do you really understand it's pluggable?" question.

**Think of it like car engines:** Your car (the app) needs an engine, but you can choose which one. They all turn the wheels (serve HTTP), but differ in fuel economy, power, and character. Spring Boot lets you drop in a different engine without rebuilding the car.

### The three servlet servers

These all implement the **Servlet API**, so they work with Spring MVC (`@Controller`, blocking, thread-per-request) and are fully interchangeable.

- **Tomcat** — the **default**. Battle-tested since 1999, the largest community, the most documentation and Stack Overflow answers. When in doubt, this is the safe choice. ~200-thread pool, thread-per-request model (everything earlier in this guide).

- **Jetty** — a lightweight server with a long history of being **embedded** inside other applications (it was doing embedded long before Tomcat made it popular). Known for a small footprint and fine-grained, programmatic configuration. Popular when you want tight control or are embedding inside another product.

- **Undertow** — a very lightweight, high-performance server from **JBoss/Red Hat** (it powers WildFly). Built on non-blocking I/O (XNIO) under the hood, so it's **memory-efficient and handles a very large number of concurrent connections well**. A common pick when you need high throughput with low memory — e.g. an API gateway holding many open connections.

### The reactive server

- **Netty** — fundamentally different. Netty is **not** a servlet container; it's an **asynchronous, event-driven, non-blocking** network framework. It's the **default server for Spring WebFlux** (the reactive stack), *not* for Spring MVC.

  **Think of it like the call center again:** Tomcat assigns one agent (thread) per call and that agent is stuck until the call ends. Netty uses a **small team of agents who juggle many calls at once** — they start one call, and while waiting on hold (a slow DB), they pick up another. A handful of threads handle thousands of connections because no thread ever sits idle blocking. This is the **event loop** model.

  You don't swap Netty in via exclusions — you get it automatically when you use the reactive starter instead of the web starter:

  ```xml
  <!-- Reactive stack → Netty is the default server -->
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-webflux</artifactId>
  </dependency>
  ```

  See `Spring_WebFlux_Reactive.md` for the reactive programming side of this.

### Servlet (Tomcat/Jetty/Undertow) vs Reactive (Netty), at a glance

| | Servlet servers (Tomcat, Jetty, Undertow) | Netty (reactive) |
|---|---|---|
| Stack | Spring **MVC** | Spring **WebFlux** |
| Model | Thread-per-request (blocking) | Event loop (non-blocking) |
| Threads | Many (~200) — one held per request | Few — shared across thousands of requests |
| Best when | Normal apps; blocking JDBC/JPA | Massive concurrency, mostly I/O-bound, non-blocking DB |
| Difficulty | Simple, familiar | Steeper — must avoid *any* blocking call |

> For most apps — and almost every junior role — **Tomcat + Spring MVC is the right default**. Reach for Netty/WebFlux only when you genuinely have huge concurrency and a fully non-blocking stack. Adding reactive complexity to a normal CRUD app is a classic over-engineering mistake.

### Switching the servlet server (Tomcat → Jetty/Undertow)

Because Tomcat is just a swappable library, you change it in a few lines of build config.

```xml
<!-- Step 1: EXCLUDE Tomcat from the web starter -->
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

<!-- Step 2: ADD the server you want instead -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>  <!-- or -starter-jetty -->
</dependency>
```

No code changes are needed — that's the whole point of the Servlet API standard.

| Server | Type | Character | Typical reason to pick |
|---|---|---|---|
| **Tomcat** | Servlet | The default, battle-tested, huge community | "It just works" — almost always the right choice |
| **Jetty** | Servlet | Lightweight, long history of embedding | Highly embeddable, fine-grained control |
| **Undertow** | Servlet | Very lightweight, high throughput (JBoss) | Memory-efficient, good for many connections |
| **Netty** | Reactive | Non-blocking event loop (WebFlux default) | Massive concurrency on a non-blocking stack |

> For a junior interview, the key point isn't *which* to pick — it's understanding *why you can swap at all*: the three servlet servers all implement the Servlet API, so Spring Boot uses whichever is on the classpath. Netty is the odd one out — it's not a servlet container, and you get it by choosing the reactive (WebFlux) stack rather than by excluding Tomcat.

---

## Connectors, Ports & HTTPS

A **connector** is the Tomcat component that owns a network port and protocol. The default is one HTTP connector on port 8080.

```properties
# ---- Enable HTTPS (TLS) on the main connector ----
server.port=8443
server.ssl.enabled=true
server.ssl.key-store=classpath:keystore.p12   # file holding your certificate + key
server.ssl.key-store-password=changeit
server.ssl.key-store-type=PKCS12
server.ssl.key-alias=tomcat
```

```
Browser  --HTTPS-->  [ Tomcat connector :8443 ]  decrypts TLS  -->  your controllers
```

**Real-world note:** In production behind a load balancer or reverse proxy (Nginx, AWS ALB), TLS is often **terminated at the proxy**, and Tomcat just receives plain HTTP internally. In that case you set `server.forward-headers-strategy=framework` so Spring correctly reads the original client IP and scheme from the `X-Forwarded-*` headers the proxy adds.

---

## Common Mistakes & Pitfalls

- **"Port 8080 is already in use."** Another process (often a previous run of your own app) still holds the port. Kill it, or set `server.port=0` for a random port in tests.
- **Cranking `threads.max` sky-high to "fix" slowness.** More threads ≠ more speed. If requests are slow because of a slow DB, you just create 5000 blocked threads and run out of memory. Fix the slow downstream call first.
- **Confusing the accept queue with the thread pool.** `accept-count` is the *waiting line*, `threads.max` is the *number of workers*. Requests beyond `threads.max` wait; requests beyond `threads.max + accept-count` are rejected.
- **Forgetting graceful shutdown in containers.** Without `server.shutdown=graceful`, a Kubernetes pod restart can kill in-flight requests mid-response, causing user-facing errors during every deploy.
- **Assuming you still build WARs.** Modern Spring Boot ships executable JARs with embedded Tomcat. You only need a WAR for legacy external servers.
- **Thinking the `DispatcherServlet` IS Tomcat.** They're separate layers — Tomcat is the server, DispatcherServlet is Spring's router that runs *inside* it.
- **Blocking the request thread on long work.** A 30-second job inside a controller holds a Tomcat thread for 30 seconds. Offload long tasks to `@Async` / a queue, or return a `202 Accepted`.

---

## Common Interview Questions

**Q1: What is embedded Tomcat and why does Spring Boot use it?**
Embedded Tomcat is the Tomcat server bundled as a library *inside* your application JAR, started by your app at runtime. Spring Boot uses it so apps are self-contained: you build one executable JAR and run it with `java -jar` — no separate server to install or deploy a WAR into. This makes apps portable, container-friendly, and lets you pin the exact Tomcat version in your build.

**Q2: WAR vs JAR — what changed?**
Traditionally you packaged a WAR and deployed it into an externally-installed Tomcat. Spring Boot packages an executable ("fat") JAR that contains your code *and* the server. One JAR = one self-running app on one port. WARs are now only for legacy shared/external servers.

**Q3: What is a servlet container?**
A program that runs servlets — managing their lifecycle and feeding them HTTP requests via the Servlet API. Tomcat is the most common one. Your Spring controllers ultimately run behind a single servlet (the DispatcherServlet) that Tomcat hosts.

**Q4: Walk me through what happens when a request hits a Spring Boot app.**
Tomcat's connector accepts the connection on port 8080 and parses the HTTP request. Tomcat assigns a worker thread from its pool. The request goes to the DispatcherServlet, which uses HandlerMapping to find the matching controller method, invokes it, converts the returned object to JSON via Jackson, writes the response, and Tomcat sends it back and returns the thread to the pool. (See [The Request Lifecycle](#the-request-lifecycle-end-to-end).)

**Q5: How does Tomcat handle concurrent requests?**
Thread-per-request: it keeps a pool (default max 200) of worker threads; each request borrows one thread for its whole duration, then returns it. Up to `threads.max` requests run concurrently; extras wait in the accept queue (`accept-count`, default 100); beyond that, connections are refused.

**Q6: What's the difference between `threads.max` and `accept-count`?**
`threads.max` is how many requests can be *actively processed* at once. `accept-count` is the size of the *waiting queue* used only when all worker threads are busy. Workers → people serving; accept-count → the line waiting to be served.

**Q7: How do you change the port / why would you set it to 0?**
`server.port=8080` in `application.properties`. Setting it to `0` makes Tomcat pick a random free port — useful in integration tests so parallel test runs don't collide on a fixed port.

**Q8: What other servers can Spring Boot use, and how do you switch?**
Spring Boot supports four: **Tomcat** (default), **Jetty**, and **Undertow** are servlet containers (for Spring MVC); **Netty** is a non-blocking server used by the reactive stack (Spring WebFlux). To swap among the servlet ones, exclude `spring-boot-starter-tomcat` and add `spring-boot-starter-jetty` (or `-undertow`) — no code changes, since they all implement the Servlet API. Netty isn't swapped in that way; you get it by using `spring-boot-starter-webflux` instead of `-web`. Undertow is picked for memory-efficiency/high connection counts, Jetty for embeddability, Netty for non-blocking high concurrency.

**Q9: What is the DispatcherServlet and how does it relate to Tomcat?**
It's Spring MVC's front controller — the single servlet that Tomcat hosts. Tomcat handles networking and threads; it forwards every request to the DispatcherServlet, which routes it to the right `@Controller` method. They're separate layers working together.

**Q10: Your app gets slow under load — what Tomcat-related things do you check?**
Are all worker threads blocked on slow DB/external calls? (Fix the slow call, don't just add threads.) Is the accept queue overflowing (requests refused)? Are connection/keep-alive timeouts sane? Is the pool sized for the workload and memory available? For heavily I/O-bound, high-concurrency workloads, consider reactive WebFlux which avoids one-thread-per-request blocking.

**Q11: How do you make Tomcat shut down without dropping requests?**
Set `server.shutdown=graceful` and a `spring.lifecycle.timeout-per-shutdown-phase`. On shutdown Tomcat stops accepting new requests but lets in-flight ones finish within the timeout — essential for zero-downtime deploys in Kubernetes.

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
  SpringApplication.run(...) is what boots it

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
  server.port=8080                     → listen port (0 = random, great for tests)
  server.servlet.context-path=/api     → prefix all endpoints
  server.tomcat.connection-timeout=20000  → slow-client guard (ms)
  server.shutdown=graceful             → finish in-flight requests on shutdown
  server.ssl.enabled=true + key-store  → enable HTTPS

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
  javax.servlet.*  → Spring Boot 2 / Tomcat 9
  jakarta.servlet.* → Spring Boot 3 / Tomcat 10+
```

---

*Last Updated: 2026-06-13*

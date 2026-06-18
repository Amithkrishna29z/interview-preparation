# Spring WebFlux & Reactive Programming — Full Stack Java Developer Interview Guide

---

## 1. Reactive Programming Fundamentals

### 1.1 Why Reactive Programming?

In Spring MVC, every HTTP request occupies one thread for its entire duration — including all waiting time (DB, HTTP calls). With a 200-thread Tomcat pool and 500ms DB calls, max throughput is only 400 req/s, and slow downstream services stall the whole app.

In the reactive model, a thread initiates I/O and **immediately returns** to handle other work. A handful of threads can serve thousands of concurrent I/O-bound requests.

#### Event Loop vs Thread-Per-Request

| Aspect | Thread-Per-Request (MVC/Tomcat) | Event Loop (WebFlux/Netty) |
|---|---|---|
| Threads needed | 1 per active request | Small fixed pool (~CPU cores) |
| Blocking I/O effect | Thread hangs idle | Non-blocking — thread stays available |
| 10,000 concurrent slow I/O | 10,000 threads | ~8–16 threads |
| Programming model | Straightforward sequential | Declarative, operator chains |

#### The Reactive Manifesto

| Property | How WebFlux addresses it |
|---|---|
| **Responsive** | Non-blocking I/O keeps latency low under load |
| **Resilient** | Error handling operators, circuit breakers, fallbacks |
| **Elastic** | Event loop scales without proportional thread growth |
| **Message Driven** | Publisher/Subscriber model — decoupled producers and consumers |

> **Common Pitfall:** The Reactive Manifesto is about system architecture, not just reactive libraries. You can build a reactive system with Kafka + Spring MVC. WebFlux is one approach, not the only one.

---

## 2. Reactive Streams Specification

### 2.1 The Four Interfaces

```java
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}

public interface Subscriber<T> {
    void onSubscribe(Subscription s);  // called first, once
    void onNext(T t);                  // called for each item
    void onError(Throwable t);         // terminal — on failure
    void onComplete();                 // terminal — when done
}

public interface Subscription {
    void request(long n);   // subscriber asks for n items
    void cancel();
}

public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {}
```

### 2.2 Backpressure — The Core Concept

Backpressure lets a subscriber tell the publisher "slow down." Reactive Streams uses a **PULL model** — the subscriber drives the rate by calling `request(N)`.

```
Without backpressure: 1M items/s → 100 items/s consumer → OutOfMemoryError
With backpressure:    consumer.request(100) → producer sends 100 → consumer.request(100) → ...
```

> **Interview Insight:** This pull model is what distinguishes Reactive Streams from simple callbacks or Futures.

### 2.3 Contract Rules

- `onSubscribe` is called exactly once before any other signal
- `onNext` is called at most N times where N is the total requested count
- `onError` and `onComplete` are mutually exclusive terminal signals
- A Subscriber MUST call `request(N)` or it will receive nothing

---

## 3. Project Reactor — Deep Dive

### 3.1 Mono and Flux

```java
// MONO — 0 or 1 asynchronous item
Mono<String> just         = Mono.just("hello");
Mono<String> empty        = Mono.empty();
Mono<String> error        = Mono.error(new RuntimeException());
Mono<String> fromCallable = Mono.fromCallable(() -> blockingDbCall());  // lazy, runs on subscribe
Mono<String> fromFuture   = Mono.fromFuture(CompletableFuture.supplyAsync(() -> "result"));

// FLUX — 0 to N asynchronous items
Flux<Integer> range    = Flux.range(1, 10);
Flux<String>  fromList = Flux.fromIterable(List.of("a", "b", "c"));
Flux<Long>    interval = Flux.interval(Duration.ofSeconds(1));  // 0, 1, 2... every second
```

**Nothing happens until you subscribe.** Reactive chains are lazy.

### 3.2 Subscribing

```java
flux.subscribe(
    item -> System.out.println("Next: " + item),
    err  -> System.err.println("Error: " + err),
    ()   -> System.out.println("Complete!")
);

// block() — converts to blocking. NEVER call inside a WebFlux handler.
String result   = mono.block();
List<Integer> l = flux.collectList().block();
```

> **Common Pitfall:** `block()` inside a WebFlux handler throws `IllegalStateException`. Event loop threads must not block.

### 3.3 Operators

#### Transformation

```java
// map: synchronous 1-to-1
Flux<String> upper = Flux.just("a", "b", "c").map(String::toUpperCase);

// flatMap: async 1-to-many, CONCURRENT — results may interleave
Flux<Order> orders = Flux.fromIterable(userIds)
    .flatMap(id -> orderRepository.findByUserId(id));  // all queries start at once

// concatMap: sequential flatMap — waits for each inner publisher
Flux<Order> ordered = Flux.fromIterable(userIds)
    .concatMap(id -> orderRepository.findByUserId(id));

// switchMap: cancels previous inner publisher when new item arrives (e.g. search-as-you-type)
```

#### Filtering & Aggregation

```java
Flux<Integer> evens      = Flux.range(1, 10).filter(n -> n % 2 == 0);
Flux<Integer> first5     = Flux.range(1, 100).take(5);
Flux<Integer> distinct   = Flux.just(1, 2, 1, 3).distinct();
Mono<Integer> sum        = Flux.range(1, 5).reduce(0, Integer::sum);  // Mono<15>
Mono<List<Integer>> list = Flux.range(1, 5).collectList();
```

#### Combining

```java
// zip: combine items from N publishers pairwise
Mono<String> combined = Mono.zip(fetchUser(id), fetchOrder(id),
    (user, order) -> user.getName() + ": " + order.getTotal());

// merge: subscribe to all concurrently, interleave by timing
Flux<String> merged = Flux.merge(flux1, flux2);

// concat: subscribe sequentially — flux1 must complete before flux2 starts
Flux<String> concatenated = Flux.concat(flux1, flux2);
```

#### Side Effects & Fallbacks

```java
flux.doOnNext(item -> log.debug("Processing: {}", item))
flux.doOnError(err  -> log.error("Failed", err))
flux.doFinally(sig  -> closeResources())  // always called, like finally

// Fallbacks
Mono<User> user = findById(id).switchIfEmpty(Mono.error(new NotFoundException("not found")));
Mono<User> safe = findById(id).defaultIfEmpty(User.guestUser());
```

### 3.4 Error Handling

```java
Mono<String> safe = riskyOp().onErrorReturn("default");
Mono<User>   safe = findUser(id)
    .onErrorResume(NotFoundException.class, e -> Mono.just(User.guest()))
    .onErrorMap(DataAccessException.class, e -> new ServiceException("DB error", e));

Mono<String> withRetry   = callApi().retry(3);
Mono<String> withTimeout = callApi().timeout(Duration.ofSeconds(5)).onErrorReturn("timed-out");
```

> **Common Pitfall:** Prefer `onErrorResume` over `onErrorContinue`. `onErrorContinue` affects upstream operators in surprising ways.

### 3.5 Backpressure Strategies

```java
fastProducer.onBackpressureBuffer(1000)   // buffer up to 1000, error if exceeded
fastProducer.onBackpressureDrop()         // silently drop items when overwhelmed
fastProducer.onBackpressureLatest()       // keep only the most recent item
```

### 3.6 Schedulers — Thread Control

```java
// publishOn: switch thread for ALL DOWNSTREAM operators from this point
Flux.range(1, 10)
    .map(i -> "item-" + i)               // subscriber thread
    .publishOn(Schedulers.boundedElastic())
    .map(String::toUpperCase)            // boundedElastic thread

// subscribeOn: switch thread for the SOURCE (position in chain doesn't matter)
Mono.fromCallable(() -> blockingDbCall())
    .subscribeOn(Schedulers.boundedElastic())  // blocking call on safe pool
    .map(result -> transform(result))

// Scheduler types:
Schedulers.parallel()        // CPU cores — for CPU-bound work
Schedulers.boundedElastic()  // expandable pool — for blocking I/O
Schedulers.single()          // single reusable thread
```

> **Interview Tip:** Always use `Schedulers.boundedElastic()` for blocking code. Never block on `Schedulers.parallel()` threads — you'll starve the CPU pool.

### 3.7 Context — Reactive Equivalent of ThreadLocal

`ThreadLocal` breaks in reactive code because the thread changes mid-chain. Use `Context` instead.

```java
Mono<String> result = Mono.deferContextual(ctx -> {
        String traceId = ctx.getOrDefault("traceId", "unknown");
        return performWork(traceId);
    })
    .contextWrite(Context.of("traceId", "abc123"));
```

> **Common Pitfall:** Context propagation is **backwards** — you write it downstream and read it upstream. Spring Security's reactive support uses this internally.

### 3.8 Testing with StepVerifier

```java
StepVerifier.create(Flux.just(1, 2, 3))
    .expectNext(1).expectNext(2).expectNext(3)
    .expectComplete()
    .verify();

StepVerifier.create(findUser(-1L))
    .expectError(NotFoundException.class)
    .verify();

// Virtual time for interval-based sequences
StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofHours(1)).take(3))
    .thenAwait(Duration.ofHours(3))
    .expectNext(0L, 1L, 2L)
    .expectComplete()
    .verify();
```

---

## 4. Spring WebFlux

### 4.1 Architecture Overview

```
HTTP Request → Netty (NioEventLoopGroup ~8 threads)
    → DispatcherHandler
    → HandlerMapping → HandlerAdapter
    → Controller returns Mono<ResponseEntity<T>> or Flux<T>
    → ResponseBodyResultHandler → writes to channel (non-blocking)
```

### 4.2 Spring WebFlux vs Spring MVC

| Aspect | Spring MVC | Spring WebFlux |
|---|---|---|
| I/O Model | Blocking (Servlet API) | Non-blocking (Reactive Streams) |
| Default Server | Tomcat (thread-per-request) | Netty (event loop) |
| Return Types | `String`, `ResponseEntity<T>` | `Mono<T>`, `Flux<T>` |
| Database | JDBC | R2DBC |
| HTTP Client | `RestTemplate`, `RestClient` | `WebClient` |
| Security | `@EnableWebSecurity` | `@EnableWebFluxSecurity` |
| Testing | `MockMvc` | `WebTestClient` |
| Learning curve | Low | High |

**When to choose WebFlux:** streaming large datasets (SSE, WebSocket), very high concurrency with I/O-bound ops, microservices making many parallel HTTP calls, all dependencies are reactive.

**When to STAY with Spring MVC:** standard CRUD apps, team unfamiliar with reactive, using JDBC/Hibernate, Java 21+ with virtual threads (matches WebFlux throughput for CRUD with simpler code).

> **Interview Tip:** Knowing when NOT to use WebFlux is as important as knowing when to use it. Mixing blocking JDBC in WebFlux gives all the complexity with none of the benefit.

### 4.3 Annotated Controllers

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    // GET single — Mono<ResponseEntity> for proper 404 handling
    @GetMapping("/{id}")
    public Mono<ResponseEntity<User>> getUser(@PathVariable Long id) {
        return userRepository.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    // GET list
    @GetMapping
    public Flux<User> getAllUsers() {
        return userRepository.findAll();
    }

    // POST
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<User> createUser(@RequestBody @Valid Mono<CreateUserRequest> requestMono) {
        return requestMono
            .map(req -> new User(req.name(), req.email()))
            .flatMap(userRepository::save);
    }

    // Server-Sent Events — streaming
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<User>> streamUsers() {
        return userRepository.findAll()
            .map(user -> ServerSentEvent.<User>builder()
                .id(user.getId().toString())
                .event("user-update")
                .data(user)
                .build());
    }

    // Parallel calls — fetch user and orders simultaneously
    @GetMapping("/{id}/with-orders")
    public Mono<UserWithOrders> getUserWithOrders(@PathVariable Long id) {
        return Mono.zip(
            userRepository.findById(id).switchIfEmpty(Mono.error(new NotFoundException("User not found"))),
            orderService.findByUserId(id).collectList()
        ).map(tuple -> new UserWithOrders(tuple.getT1(), tuple.getT2()));
    }
}
```

### 4.4 Functional Router API

An alternative to annotation controllers — useful for simple handlers and easier unit testing.

```java
@Configuration
public class UserRouter {
    @Bean
    public RouterFunction<ServerResponse> userRoutes(UserHandler handler) {
        return RouterFunctions.route()
            .GET("/api/users/{id}", handler::getUser)
            .GET("/api/users",      handler::getAllUsers)
            .POST("/api/users",     handler::createUser)
            .build();
    }
}

@Component
public class UserHandler {
    public Mono<ServerResponse> getUser(ServerRequest request) {
        Long id = Long.parseLong(request.pathVariable("id"));
        return userRepository.findById(id)
            .flatMap(user -> ServerResponse.ok().bodyValue(user))
            .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> createUser(ServerRequest request) {
        return request.bodyToMono(CreateUserRequest.class)
            .flatMap(req -> userRepository.save(new User(req.name(), req.email())))
            .flatMap(saved -> ServerResponse.created(URI.create("/api/users/" + saved.getId()))
                .bodyValue(saved));
    }
}
```

### 4.5 WebClient — Reactive HTTP Client

`WebClient` is the non-blocking replacement for `RestTemplate`.

```java
// Create — use a singleton @Bean to share the connection pool
WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
    .codecs(c -> c.defaultCodecs().maxInMemorySize(16 * 1024 * 1024))
    .build();

// GET single item
Mono<User> user = client.get()
    .uri("/users/{id}", userId)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, r ->
        r.bodyToMono(ErrorResponse.class).flatMap(e -> Mono.error(new NotFoundException(e.getMessage()))))
    .bodyToMono(User.class);

// GET list
Flux<Product> products = client.get().uri("/products").retrieve().bodyToFlux(Product.class);

// POST
Mono<Order> created = client.post()
    .uri("/orders")
    .bodyValue(new CreateOrderRequest(userId, items))
    .retrieve()
    .bodyToMono(Order.class);
```

> **Common Pitfall:** `WebClient.create()` creates a new HTTP client with a new connection pool each call. Always use a singleton `WebClient` bean.

---

## 5. R2DBC — Reactive Relational Database

### 5.1 What is R2DBC?

R2DBC is a non-blocking database driver specification. Unlike JDBC (blocks the calling thread), R2DBC returns `Mono`/`Flux` immediately.

### 5.2 Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb
    username: user
    password: pass
    pool:
      max-size: 20
```

### 5.3 Entities and Repositories

```java
// Entity — different from JPA: no @Entity, no @GeneratedValue, no relationships
@Table("users")
public class User {
    @Id
    private Long id;
    @Column("full_name")
    private String name;
    private String email;
}

// Repository — extends ReactiveCrudRepository
public interface UserRepository extends ReactiveCrudRepository<User, Long> {
    Flux<User> findByActiveTrue();
    Mono<User> findByEmail(String email);

    @Query("SELECT * FROM users WHERE created_at > :since ORDER BY created_at DESC")
    Flux<User> findRecentUsers(@Param("since") LocalDateTime since);
}
```

### 5.4 Custom Queries and Joins

R2DBC has no relationships — joins are manual via `DatabaseClient`.

```java
public Flux<UserWithOrderCount> findUsersWithOrderCount() {
    return databaseClient.sql("""
            SELECT u.id, u.name, COUNT(o.id) as order_count
            FROM users u LEFT JOIN orders o ON u.id = o.user_id
            GROUP BY u.id, u.name
            """)
        .map(row -> new UserWithOrderCount(
            row.get("id", Long.class),
            row.get("name", String.class),
            row.get("order_count", Long.class)))
        .all();
}
```

### 5.5 R2DBC vs JPA

| Feature | JPA / Hibernate | R2DBC |
|---|---|---|
| I/O Model | Blocking (JDBC) | Non-blocking |
| Lazy loading | Yes | No |
| Relationships | @OneToMany, etc. | Manual joins |
| L2 Cache | Yes | No |
| Entity tracking | Yes (dirty checking) | No |
| Transactions | @Transactional | @Transactional (reactive) |
| Production maturity | Very mature | Maturing |

> **Common Pitfall:** R2DBC is closer to raw JDBC than Hibernate in terms of features. Only use it if your full stack is non-blocking and you're prepared to manage joins manually.

### 5.6 Reactive Transactions

```java
@Transactional
public Mono<Order> createOrder(CreateOrderRequest request) {
    return userRepository.findById(request.getUserId())
        .switchIfEmpty(Mono.error(new NotFoundException("User not found")))
        .flatMap(user -> orderRepository.save(new Order(user.getId(), request.getProductId())));
    // Rolls back on any error in the chain
}
```

---

## 6. WebFlux Security

### 6.1 Configuration

```java
@Configuration
@EnableWebFluxSecurity  // NOT @EnableWebSecurity (that's for Spring MVC)
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .authorizeExchange(auth -> auth
                .pathMatchers("/api/public/**").permitAll()
                .pathMatchers("/api/admin/**").hasRole("ADMIN")
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .csrf(csrf -> csrf.disable())
            .build();
    }
}
```

### 6.2 Accessing Security Context

```java
// SecurityContextHolder doesn't work in reactive — use ReactiveSecurityContextHolder

@GetMapping("/me")
public Mono<User> getCurrentUser(@AuthenticationPrincipal Mono<JwtAuthenticationToken> tokenMono) {
    return tokenMono
        .map(token -> token.getTokenAttributes().get("sub").toString())
        .flatMap(userRepository::findByUsername);
}
```

---

## 7. WebFlux Testing

```java
@WebFluxTest(UserController.class)
class UserControllerTest {

    @Autowired  WebTestClient webTestClient;
    @MockBean   UserRepository userRepository;

    @Test
    void shouldReturnUser_whenExists() {
        when(userRepository.findById(1L)).thenReturn(Mono.just(new User(1L, "John", "john@example.com")));

        webTestClient.get().uri("/api/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(User.class)
            .value(user -> assertThat(user.getName()).isEqualTo("John"));
    }

    @Test
    void shouldReturn404_whenUserNotFound() {
        when(userRepository.findById(999L)).thenReturn(Mono.empty());

        webTestClient.get().uri("/api/users/999")
            .exchange()
            .expectStatus().isNotFound();
    }
}

// Service-layer unit test with StepVerifier
@Test
void shouldThrowNotFound_whenUserMissing() {
    when(userRepository.findById(99L)).thenReturn(Mono.empty());

    StepVerifier.create(userService.findById(99L))
        .expectError(NotFoundException.class)
        .verify();
}
```

---

## 8. Reactive Patterns

### 8.1 Parallel Calls

```java
@GetMapping("/{id}/full-profile")
public Mono<FullProfile> getFullProfile(@PathVariable Long id) {
    return Mono.zip(
        userService.findById(id),
        orderService.findByUserId(id).collectList(),
        reviewService.findByUserId(id).collectList()
    ).map(tuple -> FullProfile.builder()
        .user(tuple.getT1()).orders(tuple.getT2()).reviews(tuple.getT3()).build());
}
```

### 8.2 Circuit Breaker with Resilience4j

```java
return webClient.post().uri("/payments").bodyValue(request).retrieve()
    .bodyToMono(PaymentResult.class)
    .transformDeferred(CircuitBreakerOperator.of(circuitBreaker))
    .onErrorReturn(CallNotPermittedException.class,
        PaymentResult.circuitOpen("Payment service unavailable"));
```

### 8.3 Hot vs Cold Publishers

**Cold publisher:** each subscriber gets its own independent sequence from the beginning. Examples: `Flux.just()`, database queries, WebClient calls.

**Hot publisher:** emits regardless of subscribers; subscribers only see items emitted after they subscribe. Examples: WebSocket messages, Kafka streams, `Sinks.Many`.

```java
// Convert cold to hot
ConnectableFlux<Integer> hot = Flux.range(1, 10).publish();
hot.subscribe(i -> System.out.println("Sub1: " + i));
hot.subscribe(i -> System.out.println("Sub2: " + i));
hot.connect();  // both subscribers receive items simultaneously

// share() — auto-connects on first subscriber
Flux<StockPrice> liveStream = stockPriceService.getLivePrices().share();

// Sinks — push to hot publisher programmatically
Sinks.Many<String> sink = Sinks.many().multicast().onBackpressureBuffer();
sink.tryEmitNext("event1");
```

### 8.4 Caching

```java
Mono<Config> cached      = configService.loadConfig().cache();
Mono<Config> timedCache  = configService.loadConfig().cache(Duration.ofMinutes(5));
```

---

## 9. Interview Questions & Answers

**Q1: What is the difference between Spring MVC and Spring WebFlux?**
Spring MVC uses the blocking Servlet API — each request occupies a thread from a fixed pool. WebFlux uses a non-blocking event loop (Reactor + Netty) where a small number of threads handle thousands of concurrent requests. Same annotations (`@GetMapping`) but different return types (`Mono<T>`, `Flux<T>`), different HTTP client (`WebClient`), and different DB driver (R2DBC).

---

**Q2: What is backpressure and why does it matter?**
Backpressure lets a slow subscriber control the rate of a fast publisher. Without it, a 1M items/s producer overwhelms a 100 items/s consumer, causing `OutOfMemoryError`. Reactive Streams uses a pull model: the subscriber calls `request(N)` to ask for exactly N items. This makes reactive systems resilient under load, not just fast in ideal conditions.

---

**Q3: What is the difference between `flatMap` and `concatMap`?**
Both transform each item into a Publisher and flatten results. `flatMap` subscribes to inner publishers **concurrently** — results interleave, good for parallel DB queries. `concatMap` subscribes **sequentially** — each inner publisher must complete before the next starts, guarantees order. `switchMap` cancels the previous inner publisher when a new item arrives (useful for search-as-you-type).

---

**Q4: What is the difference between `publishOn` and `subscribeOn`?**
`publishOn(scheduler)` switches the thread for **all downstream operators** from that point. `subscribeOn(scheduler)` switches the thread **where the source subscription happens** — affects the whole upstream chain regardless of where it's placed. Use `subscribeOn` to wrap blocking source code; use `publishOn` for mid-chain thread hand-offs.

---

**Q5: When would you choose WebFlux over Spring MVC with virtual threads (Java 21)?**
Choose WebFlux for true streaming (SSE, WebSocket), when all dependencies are reactive, or when you need backpressure. Choose Spring MVC + virtual threads for CRUD with JDBC/Hibernate, simpler code, or when the team lacks reactive expertise — virtual threads eliminate the thread-per-request scaling problem for most CRUD workloads.

---

**Q6: What is R2DBC? How does it compare to JPA?**
R2DBC is a non-blocking database driver spec — repository methods return `Mono`/`Flux`. JPA/Hibernate is blocking (JDBC) but provides full ORM: lazy loading, relationship management, L2 cache, and mature tooling. R2DBC has none of those — no lazy loading, no relationships, manual joins. Only use R2DBC in a fully reactive stack.

---

**Q7: What is the difference between `Mono` and `Flux`?**
`Mono<T>` is 0 or 1 asynchronous item — use for single-item ops (findById, save). `Flux<T>` is 0 to N asynchronous items with backpressure — use for collections and streams. Both implement `Publisher<T>`. Convert with `mono.flux()`, `flux.next()`, or `flux.collectList()`.

---

**Q8: How do you handle errors in reactive chains?**
Error handling is declarative via operators: `onErrorReturn(value)` for a fallback value, `onErrorResume(fn)` to switch to a fallback publisher, `onErrorMap(fn)` to transform the exception type, `retry(n)` to resubscribe, `timeout(duration)` to error if no item arrives. Prefer `onErrorResume` over `onErrorContinue` — the latter affects upstream operators in surprising ways.

---

**Q9: What is a hot publisher vs cold publisher?**
**Cold:** starts producing when subscribed; each subscriber gets its own sequence from the beginning (e.g., `Flux.just()`, DB queries). **Hot:** emits regardless of subscribers; late subscribers miss earlier items (e.g., WebSocket messages, Kafka streams). Convert cold to hot with `flux.publish().connect()`, `flux.share()`, or `Sinks.Many`.

---

**Q10: How do you test WebFlux controllers?**
Use `@WebFluxTest(YourController.class)` for a sliced test — loads only the WebFlux layer, mock dependencies with `@MockBean`, `WebTestClient` is auto-configured. Use `@SpringBootTest(webEnvironment = RANDOM_PORT)` for full integration tests. For service/unit tests, use `StepVerifier` to verify reactive sequences step by step.

---

**Q11: What is Context in Project Reactor?**
`Context` is the reactive equivalent of `ThreadLocal`. Since the thread changes mid-chain, `ThreadLocal` breaks. Use `contextWrite()` to write data and `Mono.deferContextual()` to read it. Note: context flows **backwards** — you write downstream and read upstream. Spring Security's reactive support uses this internally.

---

**Q12: What happens if you call `block()` inside a WebFlux request handler?**
`IllegalStateException: block()/blockFirst()/blockLast() are blocking, not supported in thread reactor-http-nio-X`. Netty event loop threads cannot block — if they do, the app deadlocks. Fix: never call `block()` in a handler; wrap blocking code with `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`.

---

**Q13: What is the difference between `merge` and `concat`?**
`Flux.merge(flux1, flux2)` subscribes to all publishers concurrently — items interleave by timing. `Flux.concat(flux1, flux2)` subscribes sequentially — flux1 must complete before flux2 starts, order is guaranteed. Use `merge` for maximum throughput, `concat` for ordered results.

---

**Q14: How do you implement Server-Sent Events (SSE) with WebFlux?**
```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<StockPrice>> streamStockPrices() {
    return stockPriceService.getLivePrices()
        .map(price -> ServerSentEvent.<StockPrice>builder()
            .id(UUID.randomUUID().toString())
            .event("price-update")
            .data(price)
            .build());
}
```
`TEXT_EVENT_STREAM_VALUE` enables SSE encoding. The `Flux` stays open pushing events until it completes or the client disconnects.

---

## 10. Quick Reference — Comparison Tables

### Operator Cheat Sheet

| Goal | Operator |
|---|---|
| Transform 1-to-1 synchronously | `map` |
| Transform 1-to-many async (parallel) | `flatMap` |
| Transform 1-to-many async (sequential) | `concatMap` |
| Cancel previous on new input | `switchMap` |
| Filter items | `filter` |
| Take first N | `take(n)` |
| Skip first N | `skip(n)` |
| Merge publishers (concurrent) | `Flux.merge` |
| Combine publishers (sequential) | `Flux.concat` |
| Combine N publishers pairwise | `Mono.zip` |
| Fallback if empty | `switchIfEmpty` |
| Default value if empty | `defaultIfEmpty` |
| Aggregate to single value | `reduce` |
| Running aggregate | `scan` |
| Collect to List | `collectList` |
| Side effect | `doOnNext`, `doOnError` |
| Always execute (finally) | `doFinally` |
| Retry on error | `retry`, `retryWhen` |
| Fallback on error | `onErrorReturn`, `onErrorResume` |
| Transform exception | `onErrorMap` |
| Switch thread downstream | `publishOn` |
| Switch thread for source | `subscribeOn` |

### Common Pitfalls Summary

| Pitfall | Problem | Fix |
|---|---|---|
| `block()` in handler | Deadlocks event loop | Chain operators instead |
| Blocking JDBC in WebFlux | Starves event loop | Use R2DBC or offload to `boundedElastic()` |
| `ThreadLocal` in reactive | Thread changes mid-chain | Use `Context` |
| `Schedulers.parallel()` for blocking | Starves CPU pool | Use `Schedulers.boundedElastic()` |
| `new WebClient()` per request | New connection pool each time | Use singleton `@Bean` |
| `onErrorContinue` overuse | Silently hides errors | Use `onErrorResume` |
| Not subscribing | Nothing executes | Always subscribe or return from handler |
| `Mono.just(blockingCall())` | Blocking on assembly | Use `Mono.fromCallable(...)` |
| R2DBC without fully reactive stack | Complexity, no benefit | Use Spring MVC + JDBC instead |

---

*Last updated: 2026-06-18 | Covers Spring Boot 3.x, Project Reactor 3.x, Spring WebFlux, R2DBC, Java 17+*

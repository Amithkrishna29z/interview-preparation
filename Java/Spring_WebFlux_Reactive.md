# Spring WebFlux & Reactive Programming — Full Stack Java Developer Interview Guide

---

## 1. Reactive Programming Fundamentals

### 1.1 Why Reactive Programming?

#### The Problem: Traditional Blocking I/O

In a classic Spring MVC application backed by Tomcat, every HTTP request occupies one thread from the thread pool for the entire duration of the request — including all the time spent waiting for database queries, HTTP calls to external services, or file reads.

```
Request 1: [--------wait for DB--------][process][respond]   Thread 1 (occupied the whole time)
Request 2: [----wait for HTTP call------][process][respond]   Thread 2 (occupied the whole time)
Request 3: [waiting in queue...                           ]   No thread available — queued!
```

Consider this scenario:
- Tomcat default thread pool: **200 threads**
- Each request makes a DB call that takes **500ms**
- Maximum throughput: `200 threads / 0.5s = 400 req/s`
- But the CPU is doing nothing during those 500ms per thread — it's just waiting

If a downstream service slows down (latency increases from 50ms to 2000ms), your entire application grinds to a halt waiting for threads.

#### The Solution: Non-Blocking I/O

In the reactive model:
- A thread initiates an I/O operation and **immediately returns** to handle other work
- When the I/O completes, the thread is **notified** via a callback (or signal in Reactor's case)
- A handful of threads can serve thousands of concurrent I/O-bound requests

```
Request 1: [initiate DB call] -- thread free -- [notified, send response]
Request 2: [initiate HTTP call] -- thread free -- [notified, send response]
Request 3: [initiate DB call] -- thread free -- [notified, send response]
           ↑ All 3 started almost immediately on the same thread
```

#### Event Loop Model vs Thread-Per-Request Model

| Aspect | Thread-Per-Request (MVC/Tomcat) | Event Loop (WebFlux/Netty) |
|---|---|---|
| Concurrency unit | OS thread | Async callback / event |
| Threads needed | 1 per active request | Small fixed pool (usually = CPU cores) |
| Memory per unit | ~1MB stack per thread | Minimal |
| Blocking I/O effect | Thread hangs idle | Non-blocking — thread stays available |
| 10,000 concurrent slow I/O | 10,000 threads needed | ~8–16 threads needed |
| Programming model | Straightforward sequential | More complex, declarative |
| CPU-bound tasks | Fine | Use `Schedulers.parallel()` |

Netty's event loop: a small number of `NioEventLoopGroup` threads continuously `select()` on registered channels. When a channel has data ready, the registered callback fires on one of these threads. This is the same model used by Node.js, Nginx, and modern high-performance servers.

#### The Reactive Manifesto

The [Reactive Manifesto](https://www.reactivemanifesto.org/) defines four properties of reactive systems:

| Property | Meaning | How WebFlux addresses it |
|---|---|---|
| **Responsive** | System responds in timely manner | Non-blocking I/O keeps latency low under load |
| **Resilient** | System stays responsive in failure | Error handling operators, circuit breakers, fallbacks |
| **Elastic** | System stays responsive under varying load | Event loop scales without proportional thread growth |
| **Message Driven** | Components communicate via async messages | Publisher/Subscriber model — decoupled producers and consumers |

> **Common Pitfall:** The Reactive Manifesto is about system architecture, not just reactive programming libraries. You can build a reactive system with message queues (Kafka) and Spring MVC. WebFlux is one approach, not the only one.

---

## 2. Reactive Streams Specification

### 2.1 The Four Interfaces

The [Reactive Streams specification](https://www.reactive-streams.org/) defines a standard for asynchronous stream processing with non-blocking backpressure. It defines exactly four interfaces:

```java
// 1. Publisher: produces items
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}

// 2. Subscriber: consumes items
public interface Subscriber<T> {
    void onSubscribe(Subscription s);  // called first, once
    void onNext(T t);                  // called for each item
    void onError(Throwable t);         // called on failure (terminal)
    void onComplete();                 // called when done (terminal)
}

// 3. Subscription: link between Publisher and Subscriber
public interface Subscription {
    void request(long n);   // subscriber asks for n items
    void cancel();          // subscriber cancels the subscription
}

// 4. Processor: both Publisher and Subscriber
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
```

### 2.2 The Lifecycle

The flow: subscriber calls `subscribe()` → publisher sends a `Subscription` via `onSubscribe()` → subscriber calls `request(n)` (the PULL model) → publisher emits up to `n` items via `onNext()` → subscriber can `request()` more → publisher finishes with `onComplete()` (or `onError()`).

### 2.3 Backpressure — The Core Concept

Backpressure is the mechanism for a subscriber to tell the publisher "slow down, I can't handle this fast."

**Without backpressure:**
```
Producer: emit 1 million items/second
Consumer: can process 100 items/second
Result:   Buffer fills up → OutOfMemoryError
```

**With backpressure (pull model):**
```
Producer: ready to emit
Consumer: request(100)              → producer sends 100
Consumer: (processes 100 items)
Consumer: request(100)              → producer sends 100
Consumer: (processes 100 items)
...consumer drives the rate...
```

> **Interview Insight:** Reactive Streams uses a PULL model, not a PUSH model. The subscriber drives the rate by calling `request(N)`. This is what distinguishes it from simple callbacks or Futures.

### 2.4 Java 9 Flow API

Java 9 copied the same Reactive Streams interfaces into the JDK as `java.util.concurrent.Flow` (so `Flow.Publisher<T>` ≈ `org.reactivestreams.Publisher<T>`). Reactor bridges to it via `JdkFlowAdapter` — awareness is enough.

### 2.5 Contract Rules (Important for Interviews)

- `onSubscribe` is called exactly once before any other signal
- `onNext` is called at most `N` times where `N` is the total requested count
- After `onComplete` or `onError`, no further signals are sent
- `onError` and `onComplete` are mutually exclusive — only one terminal signal
- A Subscriber MUST call `request(N)` or it will receive nothing

---

## 3. Project Reactor — Deep Dive

Project Reactor is the reactive library used by Spring WebFlux. It implements the Reactive Streams specification and provides `Mono` and `Flux` as its core types.

### 3.1 Mono and Flux

```java
// ============================================================
// MONO — 0 or 1 asynchronous item
// ============================================================

// Common factory methods
Mono<String> just = Mono.just("hello");                    // always emits "hello"
Mono<String> empty = Mono.empty();                         // emits nothing, completes
Mono<String> error = Mono.error(new RuntimeException());   // immediately errors

// Lazy creation (runs on subscription)
Mono<String> fromCallable = Mono.fromCallable(() -> blockingDbCall());
Mono<String> fromFuture = Mono.fromFuture(CompletableFuture.supplyAsync(() -> "async result"));
// Also: Mono.defer(...), Mono.fromSupplier(...), Mono.never()

// ============================================================
// FLUX — 0 to N asynchronous items
// ============================================================

// Common factory methods
Flux<Integer> just = Flux.just(1, 2, 3, 4, 5);
Flux<Integer> range = Flux.range(1, 10);                      // 1, 2, 3...10
Flux<String> fromList = Flux.fromIterable(List.of("a", "b", "c"));
Flux<Long> interval = Flux.interval(Duration.ofSeconds(1));   // 0, 1, 2... every second
Flux<String> empty = Flux.empty();
Flux<String> error = Flux.error(new RuntimeException("failed"));
// Also: Flux.fromArray(...), Flux.fromStream(...)

// Programmatic creation (awareness): Flux.generate(...) for synchronous
// state-based emission; Flux.create(sink -> ...) to bridge callback-based APIs.
```

### 3.2 Subscribing — Terminal Operations

Nothing happens until you subscribe. Reactive chains are lazy — creating a `Flux` does not execute anything.

```java
// Full subscribe with all callbacks
flux.subscribe(
    item  -> System.out.println("Next: " + item),     // onNext
    err   -> System.err.println("Error: " + err),     // onError
    ()    -> System.out.println("Complete!")           // onComplete
);

// For manual backpressure control, subscribe with a custom BaseSubscriber and
// override hookOnSubscribe/hookOnNext to call request(n) yourself (advanced).

// block() — converts reactive to blocking (AVOID in production reactive code)
String result = mono.block();                           // blocks current thread
String withTimeout = mono.block(Duration.ofSeconds(5)); // with timeout

List<Integer> list = flux.collectList().block();        // collect all then block
```

> **Common Pitfall:** Calling `block()` inside a reactive chain will throw `IllegalStateException: block()/blockFirst()/blockLast() are blocking, which is not supported in thread reactor-http-nio-X`. Never call `block()` inside a WebFlux request handler.

### 3.3 Operators — Complete Guide

Operators transform, filter, and combine reactive streams. They are composable and lazy — each returns a new `Mono`/`Flux`.

#### Transformation Operators

```java
// map: synchronous 1-to-1 transformation
Flux<String> upper = Flux.just("a", "b", "c")
    .map(String::toUpperCase);  // A, B, C

Flux<Integer> lengths = Flux.just("hello", "world", "!")
    .map(String::length);       // 5, 5, 1

// flatMap: async 1-to-many (CONCURRENT — subscriptions happen eagerly)
// Each item becomes a new Publisher; results interleave
Flux<Order> orders = Flux.fromIterable(userIds)
    .flatMap(userId -> orderRepository.findByUserId(userId));
// All DB queries start at nearly the same time — parallel!

// concatMap: sequential flatMap (ORDERED — waits for each inner publisher)
Flux<Order> orderedOrders = Flux.fromIterable(userIds)
    .concatMap(userId -> orderRepository.findByUserId(userId));
// User 1 orders fully loaded before user 2 starts

// flatMap with concurrency limit
Flux<Result> limited = Flux.fromIterable(ids)
    .flatMap(id -> processAsync(id), 10);  // at most 10 concurrent inner subscriptions

// switchMap: like flatMap but cancels previous inner publisher when new item arrives
// Useful for "latest wins" scenarios (e.g., search-as-you-type)
Flux<SearchResult> searchResults = searchTerms
    .switchMap(term -> searchService.search(term));

// Also (awareness): cast(Class) for an unchecked cast, ofType(Class) to filter+cast by type.
```

#### Filtering Operators

```java
// filter: keep items matching predicate
Flux<Integer> evens = Flux.range(1, 10)
    .filter(n -> n % 2 == 0);  // 2, 4, 6, 8, 10

// take: keep first N items
Flux<Integer> first5 = Flux.range(1, 100).take(5);        // 1..5

// takeWhile: take items while predicate is true
Flux<Integer> belowFive = Flux.range(1, 10)
    .takeWhile(n -> n < 5);  // 1, 2, 3, 4

// skip: skip first N items
Flux<Integer> skip5 = Flux.range(1, 10).skip(5);          // 6..10

// distinct: remove duplicates
Flux<Integer> distinct = Flux.just(1, 2, 1, 3, 2).distinct();  // 1, 2, 3

// next: take the first item as a Mono
Mono<Integer> first = Flux.range(1, 10).next();           // Mono<1>

// Also (awareness): takeLast(n), takeUntil(pred), skipWhile(pred),
// distinctUntilChanged(), elementAt(i), single() (expect exactly 1, else error).
```

#### Aggregation Operators

```java
// reduce: fold to single value (emits final result only)
Mono<Integer> sum = Flux.range(1, 5)
    .reduce(0, Integer::sum);  // Mono<15>

// scan: running fold (emits each intermediate result)
Flux<Integer> runningSum = Flux.range(1, 5)
    .scan(0, Integer::sum);    // 0, 1, 3, 6, 10, 15

// count: count items
Mono<Long> count = Flux.range(1, 100).count();  // Mono<100>

// collectList: accumulate all items into a List
Mono<List<Integer>> list = Flux.range(1, 5).collectList();

// collectMap: accumulate into a Map
Mono<Map<Long, User>> userMap = userFlux.collectMap(User::getId);

// buffer: group items into lists
Flux<List<Integer>> buffers = Flux.range(1, 10).buffer(3);
// [1,2,3], [4,5,6], [7,8,9], [10]

// Also (awareness): collectMultimap, window(n) (Flux of Flux windows),
// groupBy(keyFn) (partition into GroupedFlux groups).
```

#### Combining Operators

```java
// zip: combine latest items from N publishers (all must emit)
Mono<String> combined = Mono.zip(
    fetchUser(userId),
    fetchOrder(orderId),
    (user, order) -> user.getName() + ": " + order.getTotal()
);

// zip with Flux
Flux<String> zipped = Flux.zip(
    Flux.just("A", "B", "C"),
    Flux.just(1, 2, 3),
    (letter, number) -> letter + number
);  // A1, B2, C3

// merge: subscribe to all concurrently, interleave outputs
Flux<String> merged = Flux.merge(
    Flux.just("a1", "a2").delayElements(Duration.ofMillis(100)),
    Flux.just("b1", "b2").delayElements(Duration.ofMillis(150))
);  // a1, b1, a2, b2 (interleaved by timing)

// concat: subscribe sequentially (wait for each to complete)
Flux<String> concatenated = Flux.concat(
    Flux.just("a1", "a2"),
    Flux.just("b1", "b2")
);  // a1, a2, b1, b2 (always in this order)

// Also (awareness): zipWith / mergeWith / concatWith (instance variants),
// startWith(items) to prepend, combineLatest, and firstWithSignal (fastest wins).
```

#### Side Effect Operators (no transformation)

```java
// doOnNext: side effect on each item (logging, metrics)
flux.doOnNext(item -> log.debug("Processing: {}", item))

// doOnError: side effect on error
flux.doOnError(err -> log.error("Pipeline failed", err))

// doOnComplete: side effect on completion
flux.doOnComplete(() -> log.info("Stream completed"))

// doFinally: always called on termination or cancellation (like finally block)
flux.doFinally(signalType -> closeResources())

// log: built-in operator that logs all signals
flux.log()

// Also (awareness): doOnSubscribe, doOnCancel, doOnRequest, doOnTerminate.
```

#### Fallback and Default Operators

```java
// switchIfEmpty: subscribe to fallback if source is empty
Mono<User> user = findById(id)
    .switchIfEmpty(Mono.error(new NotFoundException("User " + id + " not found")));

// defaultIfEmpty: return a default value if source is empty
Mono<User> withDefault = findById(id)
    .defaultIfEmpty(User.guestUser());

// Also (awareness): switchIfEmpty works on Flux too; Mono.or(other) falls back
// to another publisher when empty.
```

### 3.4 Error Handling

```java
// onErrorReturn: replace error with a fallback value
Mono<String> safe = riskyOperation()
    .onErrorReturn("default-value");

// onErrorResume: replace error with a fallback publisher
Mono<User> safe2 = findUser(id)
    .onErrorResume(NotFoundException.class, e -> Mono.just(User.guest()))
    .onErrorResume(e -> Mono.error(new ServiceException("User service down", e)));

// onErrorMap: transform exception type
Mono<User> mapped = findUser(id)
    .onErrorMap(DataAccessException.class, e -> new ServiceException("DB error", e));

// retry: resubscribe on error
Mono<String> withRetry = callExternalApi()
    .retry(3);  // retry up to 3 times on ANY error

// timeout: error if no item within duration
Mono<String> withTimeout = callExternalApi()
    .timeout(Duration.ofSeconds(5))
    .onErrorReturn("timed-out");

// Also (awareness):
//   onErrorReturn(ExType.class, value) for a specific exception type
//   onErrorContinue((err, item) -> ...) skips errored items in a Flux (use cautiously)
//   retryWhen(Retry.backoff(3, Duration.ofMillis(100)).maxBackoff(...).jitter(0.5)...)
//     for exponential backoff with jitter and a filter on which errors to retry
//   timeout(duration, fallbackPublisher) to fall back instead of erroring
```

> **Common Pitfall:** `onErrorContinue` can be confusing because it affects operators upstream in the chain, not just the one it's attached to. Prefer `onErrorResume` in most cases for explicit, predictable error handling.

### 3.5 Backpressure in Depth

```java
// Overflow strategies when publisher is faster than subscriber
Flux<Integer> fastProducer = Flux.range(1, 1_000_000);

fastProducer.onBackpressureBuffer(1000)        // buffer up to 1000, error if exceeded
fastProducer.onBackpressureBuffer()            // unbounded buffer (risk: OOM)
fastProducer.onBackpressureDrop()              // silently drop items when overwhelmed
fastProducer.onBackpressureDrop(i ->           // drop with callback
    log.warn("Dropped item: {}", i))
fastProducer.onBackpressureLatest()            // keep only the most recent item
fastProducer.onBackpressureError()             // signal OverflowException immediately

// limitRate: request items in chunks (automatic batching)
flux.limitRate(100)           // requests 100 at a time from upstream
flux.limitRate(100, 75)       // prefetch 100, refill when 75% consumed (75 items)
```

### 3.6 Schedulers — Thread Control

By default, Reactor operators run on whatever thread triggered the subscription. Schedulers allow you to control which thread pool executes which part of the pipeline.

```java
// publishOn: switch thread for ALL DOWNSTREAM operators from this point
Flux.range(1, 10)
    .map(i -> "item-" + i)          // runs on subscriber thread
    .publishOn(Schedulers.boundedElastic())
    .map(s -> s.toUpperCase())      // runs on boundedElastic thread
    .subscribe(System.out::println);

// subscribeOn: switch thread for the SOURCE PUBLISHER (where subscription happens)
// Affects the whole chain UPWARD from subscribeOn
Flux.fromCallable(() -> slowBlockingDbCall())  // runs on boundedElastic (due to subscribeOn)
    .subscribeOn(Schedulers.boundedElastic())  // position doesn't matter much
    .map(result -> transform(result))          // runs on boundedElastic
    .subscribe(System.out::println);

// Scheduler types — when to use each:
Schedulers.immediate()            // current thread (no switching — default)
Schedulers.single()               // single reusable background thread
Schedulers.parallel()             // fixed pool = CPU cores, for CPU-bound
Schedulers.boundedElastic()       // expandable pool (max 10x CPU cores), for blocking I/O
// Also (awareness): fromExecutorService(executor) and newBoundedElastic(...) for custom pools.
```

> **Interview Tip:** Always use `Schedulers.boundedElastic()` when wrapping legacy blocking code (JDBC, file I/O) in a reactive chain. NEVER block on a `Schedulers.parallel()` thread — those threads are shared for CPU-bound work and blocking them starves the entire reactive application.

### 3.7 Context — Reactive Equivalent of ThreadLocal

In traditional Java, `ThreadLocal` stores request-scoped data (user ID, trace ID) because the same thread handles the entire request. In reactive, the thread changes mid-chain — `ThreadLocal` breaks completely.

Reactor provides `Context` as the reactive replacement.

```java
// Write context downstream with contextWrite(...); read it upstream with deferContextual(...)
Mono<String> result = Mono.deferContextual(ctx -> {
        String traceId = ctx.getOrDefault("traceId", "unknown");
        return performWork(traceId);
    })
    .contextWrite(Context.of("traceId", "abc123", "userId", 42L));
```

> **Common Pitfall:** Context propagation is BACKWARDS (from `contextWrite` to upstream). You write context downstream and read it upstream. This is the opposite of what most developers expect. (Spring Security's reactive support uses this internally.)

### 3.8 Testing with StepVerifier

`StepVerifier` is Project Reactor's testing tool for verifying reactive sequences.

```java
// Basic verification
StepVerifier.create(Flux.just(1, 2, 3))
    .expectNext(1)
    .expectNext(2)
    .expectNext(3)
    .expectComplete()
    .verify();

// Verify with assertions
StepVerifier.create(userService.findByEmail("john@example.com"))
    .assertNext(user -> assertThat(user.getName()).isEqualTo("John"))
    .expectComplete()
    .verify();

// Verify error
StepVerifier.create(findUser(-1L))
    .expectError(NotFoundException.class)
    .verify();

// Verify with virtual time (for Flux.interval, retryWhen with delays, etc.)
StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofHours(1)).take(3))
    .expectSubscription()
    .thenAwait(Duration.ofHours(3))       // advance virtual clock by 3 hours
    .expectNext(0L, 1L, 2L)
    .expectComplete()
    .verify();
```

---

## 4. Spring WebFlux

### 4.1 Architecture Overview

Spring WebFlux is Spring's reactive web framework, introduced in Spring 5. It runs on **Netty** by default (non-blocking event loop) and can also run on Undertow or Jetty in non-blocking mode.

```
HTTP Request
      |
      v
Netty (NioEventLoopGroup — ~8 threads)
      |
      v
DispatcherHandler (WebFlux equivalent of DispatcherServlet)
      |
      v
HandlerMapping → finds matching controller method or router function
      |
      v
HandlerAdapter → invokes the handler
      |
      v
Controller method → returns Mono<ResponseEntity<T>> or Flux<T>
      |
      v
ResponseBodyResultHandler → serializes to JSON/XML via HttpMessageWriter
      |
      v
NettyServerHttpResponse → writes to channel (non-blocking)
```

### 4.2 Spring WebFlux vs Spring MVC — Detailed Comparison

| Aspect | Spring MVC | Spring WebFlux |
|---|---|---|
| I/O Model | Blocking (Servlet API) | Non-blocking (Reactive Streams) |
| Default Server | Tomcat (thread-per-request) | Netty (event loop) |
| Thread Model | 1 thread per request | Small pool (event loop) |
| Programming Model | Imperative (sequential code) | Functional/declarative (chain operators) |
| Return Types | `String`, `Object`, `ResponseEntity<T>` | `Mono<T>`, `Flux<T>`, `ResponseEntity<Mono<T>>` |
| Annotations | `@GetMapping`, `@PostMapping`, etc. | Same annotations + functional router |
| Database | JDBC (blocking) | R2DBC (non-blocking) |
| HTTP Client | `RestTemplate`, `RestClient` | `WebClient` |
| Security config | `@EnableWebSecurity` | `@EnableWebFluxSecurity` |
| Testing | `MockMvc` | `WebTestClient` |
| Streaming | Limited (DeferredResult) | First-class (Flux, SSE) |
| Debugging | Stack traces are clear | Stack traces are harder (async) |
| Learning curve | Low | High |
| Best for | Standard CRUD, simple apps | High concurrency, streaming, microservices |

**When to choose WebFlux:**
- Streaming large datasets to clients (Server-Sent Events, WebSocket)
- Very high concurrency with I/O-bound operations
- Microservices making many parallel downstream HTTP calls
- All dependencies support reactive (R2DBC, reactive MongoDB, reactive Redis)
- Real-time data pipelines

**When to STAY with Spring MVC:**
- Standard CRUD applications
- Team unfamiliar with reactive (productivity hit is real)
- Using JDBC — mixing blocking JDBC in a WebFlux pipeline is WORSE than just using MVC
- Java 21+ with virtual threads: Spring MVC + virtual threads can match WebFlux throughput for most CRUD workloads with simpler code
- Existing Hibernate/JPA — no reactive equivalent with full feature parity

> **Interview Tip:** Knowing when NOT to use WebFlux is as important as knowing when to use it. A common mistake is adopting WebFlux for a standard CRUD application with JDBC, which gives all the complexity with none of the performance benefits.

### 4.3 Annotated Controllers

WebFlux supports the same annotations as Spring MVC, just with reactive return types.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserRepository userRepository;
    private final OrderService orderService;

    public UserController(UserRepository userRepository, OrderService orderService) {
        this.userRepository = userRepository;
        this.orderService = orderService;
    }

    // GET single item — Mono<ResponseEntity> for proper 404 handling
    @GetMapping("/{id}")
    public Mono<ResponseEntity<User>> getUser(@PathVariable Long id) {
        return userRepository.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    // GET list — Flux streams items as they arrive
    @GetMapping
    public Flux<User> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return userRepository.findAll()
            .skip((long) page * size)
            .take(size);
    }

    // POST — accept reactive request body
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<User> createUser(@RequestBody @Valid Mono<CreateUserRequest> requestMono) {
        return requestMono
            .map(req -> User.builder()
                .name(req.name())
                .email(req.email())
                .build())
            .flatMap(userRepository::save);
    }

    // PUT — update with proper 404
    @PutMapping("/{id}")
    public Mono<ResponseEntity<User>> updateUser(
            @PathVariable Long id,
            @RequestBody @Valid Mono<UpdateUserRequest> requestMono) {
        return userRepository.findById(id)
            .flatMap(existingUser -> requestMono
                .map(req -> existingUser.withName(req.name()).withEmail(req.email())))
            .flatMap(userRepository::save)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    // DELETE
    @DeleteMapping("/{id}")
    public Mono<ResponseEntity<Void>> deleteUser(@PathVariable Long id) {
        return userRepository.findById(id)
            .flatMap(user -> userRepository.delete(user)
                .thenReturn(ResponseEntity.<Void>noContent().build()))
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    // Server-Sent Events — streaming
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<User>> streamUsers() {
        return userRepository.findAll()
            .delayElements(Duration.ofMillis(100))
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
            userRepository.findById(id)
                .switchIfEmpty(Mono.error(new NotFoundException("User not found"))),
            orderService.findByUserId(id).collectList()
        ).map(tuple -> new UserWithOrders(tuple.getT1(), tuple.getT2()));
    }

    // Reactive request params
    @GetMapping("/search")
    public Flux<User> searchUsers(@RequestParam String query) {
        return userRepository.findByNameContainingIgnoreCase(query)
            .take(50);  // limit results
    }
}
```

### 4.4 Functional Router API

An alternative to annotation-based controllers — useful for simple handlers and better for testing in isolation.

```java
// Router — defines routes
@Configuration
public class UserRouter {

    @Bean
    public RouterFunction<ServerResponse> userRoutes(UserHandler handler) {
        return RouterFunctions.route()
            .GET("/api/users/{id}", handler::getUser)
            .GET("/api/users", handler::getAllUsers)
            .POST("/api/users", handler::createUser)
            .PUT("/api/users/{id}", handler::updateUser)
            .DELETE("/api/users/{id}", handler::deleteUser)
            .GET("/api/users/stream",
                RequestPredicates.accept(MediaType.TEXT_EVENT_STREAM),
                handler::streamUsers)
            .build();
    }
}

// Handler — handles requests (no Spring annotations)
@Component
public class UserHandler {

    private final UserRepository userRepository;

    public Mono<ServerResponse> getUser(ServerRequest request) {
        Long id = Long.parseLong(request.pathVariable("id"));
        return userRepository.findById(id)
            .flatMap(user -> ServerResponse.ok()
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue(user))
            .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> getAllUsers(ServerRequest request) {
        Flux<User> users = userRepository.findAll();
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(users, User.class);
    }

    public Mono<ServerResponse> createUser(ServerRequest request) {
        return request.bodyToMono(CreateUserRequest.class)
            .flatMap(req -> userRepository.save(new User(req.name(), req.email())))
            .flatMap(saved -> ServerResponse.created(
                    URI.create("/api/users/" + saved.getId()))
                .bodyValue(saved));
    }
    // streamUsers, updateUser, deleteUser follow the same handler pattern.
}
```

### 4.5 WebClient — Reactive HTTP Client

`WebClient` is the reactive replacement for `RestTemplate`. It is non-blocking and returns `Mono`/`Flux`.

```java
// ============================================================
// Creating WebClient
// ============================================================

// Basic
WebClient client = WebClient.create("https://api.example.com");

// Builder pattern (recommended for production)
WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
    .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
    .codecs(configurer -> configurer
        .defaultCodecs()
        .maxInMemorySize(16 * 1024 * 1024))  // 16MB buffer limit
    .filter(logRequest())                    // custom filter
    .filter(retryFilter())                   // retry filter
    .build();

// ============================================================
// Making Requests
// ============================================================

// GET — single item
Mono<User> user = client.get()
    .uri("/users/{id}", userId)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, response ->
        response.bodyToMono(ErrorResponse.class)
            .flatMap(error -> Mono.error(new NotFoundException(error.getMessage()))))
    .onStatus(HttpStatusCode::is5xxServerError, response ->
        Mono.error(new ServiceException("Server error")))
    .bodyToMono(User.class);

// GET — list
Flux<Product> products = client.get()
    .uri("/products?category={cat}&limit={limit}", "electronics", 10)
    .retrieve()
    .bodyToFlux(Product.class);

// POST — with request body
Mono<Order> created = client.post()
    .uri("/orders")
    .contentType(MediaType.APPLICATION_JSON)
    .bodyValue(new CreateOrderRequest(userId, items))
    .retrieve()
    .bodyToMono(Order.class);

// PUT / DELETE follow the same pattern (.put()/.delete(), .bodyValue(...) as needed).
// Use .toEntity(Type.class) instead of .bodyToMono to also read response headers/status.
```

**Advanced (awareness):** `WebClient.builder()` accepts `.filter(...)` `ExchangeFilterFunction`s for cross-cutting concerns like logging, attaching auth tokens, or retry. For production you typically supply a custom Netty `HttpClient` via `clientConnector(...)` to configure connection/read/write timeouts and the connection pool.

> **Common Pitfall:** `WebClient.create()` creates a new HTTP client with a new connection pool each time. Always use a singleton `WebClient` bean (or a `WebClient.Builder` bean and build from it) to share the connection pool.

---

## 5. R2DBC — Reactive Relational Database

### 5.1 What is R2DBC?

R2DBC (Reactive Relational Database Connectivity) is a specification for non-blocking database drivers for relational databases. Unlike JDBC (which blocks the calling thread until the DB responds), R2DBC returns `Mono`/`Flux` immediately and processes results asynchronously.

### 5.2 Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>

<!-- PostgreSQL -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- MySQL -->
<dependency>
    <groupId>io.asyncer</groupId>
    <artifactId>r2dbc-mysql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- H2 for tests -->
<dependency>
    <groupId>io.r2dbc</groupId>
    <artifactId>r2dbc-h2</artifactId>
    <scope>test</scope>
</dependency>
```

```yaml
# application.yml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/mydb
    username: user
    password: pass
    pool:
      max-size: 20
      initial-size: 5
      max-idle-time: 30m
```

### 5.3 Entities and Repositories

```java
// Entity — NOTE: different annotations from JPA
@Table("users")
public class User {
    @Id
    private Long id;       // R2DBC handles auto-increment differently from JPA

    @Column("full_name")
    private String name;

    private String email;

    @Column("created_at")
    private LocalDateTime createdAt;

    // NO @Entity, NO @GeneratedValue
    // NO @OneToMany, @ManyToOne — R2DBC has no relationships

    // Use Lombok or generate constructors/getters
    public User() {}
    public User(String name, String email) {
        this.name = name;
        this.email = email;
        this.createdAt = LocalDateTime.now();
    }
    // getters/setters...
}

// Repository — extends ReactiveCrudRepository
public interface UserRepository extends ReactiveCrudRepository<User, Long> {
    // Derived query methods — just like JPA but return Mono/Flux
    Flux<User> findByActiveTrue();
    Mono<User> findByEmail(String email);
    Flux<User> findByNameContainingIgnoreCase(String name);
    Mono<Long> countByActiveTrue();
    Mono<Void> deleteByEmail(String email);

    // Custom query with @Query annotation
    @Query("SELECT * FROM users WHERE created_at > :since ORDER BY created_at DESC")
    Flux<User> findRecentUsers(@Param("since") LocalDateTime since);

    @Query("UPDATE users SET active = false WHERE last_login < :cutoff")
    Mono<Integer> deactivateInactiveUsers(@Param("cutoff") LocalDateTime cutoff);
}

// Reactive paging
public interface UserRepository extends ReactiveCrudRepository<User, Long>,
        ReactiveSortingRepository<User, Long> {

    Flux<User> findAllByOrderByNameAsc(Pageable pageable);
}
```

### 5.4 Custom R2DBC Queries with DatabaseClient

For anything derived queries can't express (joins, aggregations, generated keys), inject `DatabaseClient` and write SQL directly. Since R2DBC has no relationships, joins are manual.

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
        .all();  // .one() for a single row; .bind("name", value) to bind params
}
```

### 5.5 R2DBC vs JPA Comparison

| Feature | JPA / Hibernate | R2DBC |
|---|---|---|
| I/O Model | Blocking (JDBC) | Non-blocking |
| Lazy loading | Yes (proxy magic) | No |
| Eager loading | Yes (@Eager) | Manual via flatMap/zip |
| Relationships | @OneToMany, @ManyToOne, etc. | Manual joins with @Query |
| L2 Cache | Yes (EhCache, Caffeine) | No built-in |
| Entity tracking | Yes (dirty checking) | No |
| Migrations | Flyway/Liquibase (full support) | Flyway (workarounds needed) |
| Transactions | @Transactional | @Transactional (works reactively) |
| Query DSL | JPQL, Criteria API, QueryDSL | @Query, DatabaseClient |
| Schema complexity | Handles complex schemas | Best for simple schemas |
| Production maturity | Very mature | Maturing (not all DBs supported) |

> **Common Pitfall:** Developers sometimes adopt R2DBC hoping for easy Hibernate-style ORM with reactivity. R2DBC is much closer to raw JDBC in terms of features — you are responsible for mapping joins and managing relationships. Only use R2DBC if you truly need non-blocking DB access and are prepared for the manual work.

### 5.6 Reactive Transactions

`@Transactional` works with R2DBC, but the transaction is reactive — it commits when the returned `Mono`/`Flux` completes and rolls back if any step in the chain errors.

```java
@Service
public class OrderService {

    @Transactional
    public Mono<Order> createOrder(CreateOrderRequest request) {
        return userRepository.findById(request.getUserId())
            .switchIfEmpty(Mono.error(new NotFoundException("User not found")))
            .flatMap(user -> orderRepository.save(new Order(user.getId(), request.getProductId())));
        // If any step errors, the whole transaction rolls back
    }
}
```

---

## 6. WebFlux Security

### 6.1 Configuration

```java
@Configuration
@EnableWebFluxSecurity  // NOTE: NOT @EnableWebSecurity (that's for Spring MVC)
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
            .csrf(csrf -> csrf.disable())  // disable for stateless REST APIs
            .build();
    }
}
```

The key differences from MVC security: the config bean is a `SecurityWebFilterChain` built from `ServerHttpSecurity`, and you use `authorizeExchange`/`pathMatchers` instead of `authorizeHttpRequests`/`requestMatchers`. To map JWT claims to roles, provide a `ReactiveJwtAuthenticationConverter` bean; CORS is configured via `.cors(...)` with a `CorsConfigurationSource` — both work the same way as MVC, just with reactive types.

### 6.2 Accessing Security Context in Reactive Chain

```java
// In WebFlux, SecurityContextHolder doesn't work — it uses ThreadLocal
// Use ReactiveSecurityContextHolder instead

@GetMapping("/me")
public Mono<User> getCurrentUser() {
    return ReactiveSecurityContextHolder.getContext()
        .map(SecurityContext::getAuthentication)
        .map(Authentication::getName)
        .flatMap(userRepository::findByUsername);
}

// Using @AuthenticationPrincipal (easier)
@GetMapping("/me")
public Mono<User> getCurrentUser(@AuthenticationPrincipal Mono<JwtAuthenticationToken> tokenMono) {
    return tokenMono
        .map(token -> token.getTokenAttributes().get("sub").toString())
        .flatMap(userRepository::findByUsername);
}
```

---

## 7. WebFlux Testing

### 7.1 WebTestClient

```java
@WebFluxTest(UserController.class)  // loads only WebFlux layer
class UserControllerTest {

    @Autowired
    private WebTestClient webTestClient;

    @MockBean
    private UserRepository userRepository;

    @Test
    void shouldReturnUser_whenExists() {
        User mockUser = new User(1L, "John Doe", "john@example.com");
        when(userRepository.findById(1L)).thenReturn(Mono.just(mockUser));

        webTestClient.get()
            .uri("/api/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentType(MediaType.APPLICATION_JSON)
            .expectBody(User.class)
            .value(user -> {
                assertThat(user.getId()).isEqualTo(1L);
                assertThat(user.getName()).isEqualTo("John Doe");
            });
    }

    @Test
    void shouldReturn404_whenUserNotFound() {
        when(userRepository.findById(999L)).thenReturn(Mono.empty());

        webTestClient.get()
            .uri("/api/users/999")
            .exchange()
            .expectStatus().isNotFound();
    }
    // POST/streaming follow the same pattern, e.g. .post().bodyValue(req)
    // ...expectStatus().isCreated(), or .expectBodyList(User.class).hasSize(n).
}

// Integration test against a real server: @SpringBootTest(webEnvironment = RANDOM_PORT)
// auto-configures WebTestClient against the running app.

// ============================================================
// Unit testing service layer with StepVerifier
// ============================================================

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

### 8.1 Parallel Calls — Fetch and Combine

One of the most common use cases: fetch multiple resources concurrently and combine.

```java
// Fetch user and orders in parallel, combine with zip
@GetMapping("/{id}/full-profile")
public Mono<FullProfile> getFullProfile(@PathVariable Long id) {
    return Mono.zip(
        userService.findById(id),
        orderService.findByUserId(id).collectList(),
        reviewService.findByUserId(id).collectList()
    ).map(tuple -> FullProfile.builder()
        .user(tuple.getT1())
        .orders(tuple.getT2())
        .reviews(tuple.getT3())
        .build());
}

// Fan-out: call same service for multiple IDs
public Flux<ProductDetails> enrichProducts(List<Long> productIds) {
    return Flux.fromIterable(productIds)
        .flatMap(id -> productService.findById(id), 10)  // max 10 concurrent
        .onErrorContinue((err, id) -> log.warn("Failed to load product {}", id));
}
```

### 8.2 Circuit Breaker with Resilience4j

With the `resilience4j-reactor` dependency, wrap a reactive call with `transformDeferred(CircuitBreakerOperator.of(circuitBreaker))` (and similarly `RateLimiterOperator`) to add a circuit breaker, then handle the open state with `onErrorReturn(CallNotPermittedException.class, fallback)`.

```java
return webClient.post().uri("/payments").bodyValue(request).retrieve()
    .bodyToMono(PaymentResult.class)
    .transformDeferred(CircuitBreakerOperator.of(circuitBreaker))
    .onErrorReturn(CallNotPermittedException.class,
        PaymentResult.circuitOpen("Payment service unavailable"));
```

### 8.3 Hot vs Cold Publishers

```java
// ============================================================
// COLD publisher: each subscriber gets its own sequence
// ============================================================

// Cold: database query — each subscriber triggers a new query
Flux<User> coldQuery = userRepository.findAll();
// Subscriber 1 subscribes → triggers SELECT * FROM users
// Subscriber 2 subscribes → triggers another SELECT * FROM users

// Cold: Flux.just, Flux.range, etc. are cold
Flux<Integer> cold = Flux.range(1, 5);
cold.subscribe(i -> System.out.print(i + " "));  // 1 2 3 4 5
cold.subscribe(i -> System.out.print(i + " "));  // 1 2 3 4 5 (fresh sequence)

// ============================================================
// HOT publisher: emits regardless of subscribers, shared sequence
// ============================================================

// ConnectableFlux — hot publisher you control
ConnectableFlux<Integer> hot = Flux.range(1, 10)
    .doOnSubscribe(s -> System.out.println("New subscriber"))
    .publish();  // converts cold to hot, but doesn't start yet

hot.subscribe(i -> System.out.println("Sub1: " + i));  // registered, not yet receiving
hot.subscribe(i -> System.out.println("Sub2: " + i));  // registered, not yet receiving
hot.connect();  // NOW items start flowing to both subscribers simultaneously

// share() — auto-connect on first subscriber
Flux<StockPrice> liveStream = stockPriceService.getLivePrices()
    .share();  // shared: new subscribers see items from subscription point forward

// Sinks — programmatically push to a hot publisher from imperative code
Sinks.Many<String> sink = Sinks.many().multicast().onBackpressureBuffer();
Flux<String> hotFlux = sink.asFlux();
sink.tryEmitNext("event1");   // push items from anywhere
sink.tryEmitComplete();
```

### 8.4 Caching Reactive Results

The simplest cache is the `cache()` operator — the first subscriber fetches, later subscribers get the replayed result. `cache(Duration)` adds a TTL. For per-key caching, check a Caffeine cache first and fall back to the repository.

```java
Mono<Config> cachedConfig = configService.loadConfig().cache();           // cache forever
Mono<Config> timedCache  = configService.loadConfig().cache(Duration.ofMinutes(5));
```

### 8.5 Rate Limiting / Throttling

Throttle emissions with `delayElements(Duration)`, cap items per time window with `window(Duration).flatMap(w -> w.take(n))`, or use Resilience4j via `transformDeferred(RateLimiterOperator.of(rateLimiter))`.

### 8.6 Reactive Caching with `@Cacheable`

`@Cacheable` can wrap methods returning `Mono`/`Flux`, but it needs reactor-aware cache support (e.g. a reactive cache manager / Caffeine reactive support) to cache the emitted value rather than the publisher. Awareness that this requires extra configuration is enough.

---

## 9. Interview Questions & Answers

### Fundamentals

**Q1: What is the difference between Spring MVC and Spring WebFlux?**

Spring MVC uses the blocking Servlet API — each HTTP request occupies a thread from a fixed pool (default 200 in Tomcat) for the entire request duration. Spring WebFlux uses a non-blocking event loop (Project Reactor + Netty) — a small number of threads handle thousands of concurrent requests by never blocking.

Key differences:
- MVC: blocking I/O, thread-per-request, Tomcat/Jetty, `RestTemplate`/`RestClient`
- WebFlux: non-blocking I/O, event loop, Netty by default, `WebClient`
- Same annotation model (`@GetMapping`, `@RestController`) but different return types (`Mono<T>`, `Flux<T>`)
- WebFlux requires fully non-blocking dependencies (R2DBC not JDBC, reactive MongoDB not Spring Data MongoDB JPA)

---

**Q2: What is backpressure and why does it matter?**

Backpressure is the mechanism in Reactive Streams that allows a subscriber to control the rate at which a publisher sends data.

Without backpressure: a fast publisher sending 1 million events/second to a slow subscriber (100 events/second) will overflow the buffer and cause `OutOfMemoryError`.

With backpressure (Reactive Streams pull model): the subscriber calls `request(N)` to ask for exactly N items. The publisher sends no more than requested. The subscriber drives the rate.

This is fundamental because real systems have heterogeneous speeds — databases, networks, and CPUs all operate at different rates. Backpressure is what makes reactive systems resilient under load rather than just fast in ideal conditions.

---

**Q3: What is the difference between `flatMap` and `concatMap`?**

Both transform each item into a Publisher, then flatten the results.

- `flatMap`: subscribes to all inner publishers **concurrently/eagerly**. Results from different inner publishers interleave. Use when order doesn't matter and you want parallelism (e.g., fetch orders for 100 users concurrently).
- `concatMap`: subscribes to inner publishers **sequentially**. Each inner publisher must complete before the next starts. Results maintain order. Use when order matters or when sequential processing is required.

```java
// flatMap — all 100 DB queries start nearly simultaneously
Flux.fromIterable(userIds).flatMap(id -> db.findOrdersForUser(id))

// concatMap — user1 orders complete before user2 queries start
Flux.fromIterable(userIds).concatMap(id -> db.findOrdersForUser(id))
```

`switchMap` is a third variant: cancels the previous inner publisher when a new item arrives (useful for "latest wins" like search suggestions).

---

**Q4: What is the difference between `publishOn` and `subscribeOn`?**

Both switch the thread of execution, but at different points:

- `subscribeOn(scheduler)`: switches the thread **where the source subscription happens** — affects the entire upstream chain. Position in the chain doesn't matter; only the first `subscribeOn` takes effect. Use for wrapping blocking source code.
- `publishOn(scheduler)`: switches the thread for **all downstream operators** from that point in the chain. Multiple `publishOn` calls create thread hand-offs at each point.

```java
// subscribeOn: the Callable runs on boundedElastic
Mono.fromCallable(() -> blockingDbCall())
    .subscribeOn(Schedulers.boundedElastic())
    .map(result -> transform(result))  // also on boundedElastic

// publishOn: switch happens mid-chain
Flux.range(1, 10)
    .map(i -> i * 2)                    // runs on subscriber thread
    .publishOn(Schedulers.parallel())
    .map(i -> heavyCompute(i))          // runs on parallel pool
    .publishOn(Schedulers.boundedElastic())
    .flatMap(i -> saveToDb(i))          // runs on boundedElastic
```

---

**Q5: When would you choose WebFlux over Spring MVC with virtual threads (Java 21)?**

This is a nuanced question — both are valid in 2024+:

Choose **WebFlux** when:
- You need true **streaming** to clients (SSE, WebSocket, infinite Flux)
- All dependencies are reactive (R2DBC, reactive MongoDB, WebClient)
- You need **backpressure** control over the entire pipeline
- Architecturally required by your event-driven system design

Choose **Spring MVC + virtual threads** when:
- Standard CRUD with JDBC/Hibernate
- Team doesn't have reactive expertise
- Simpler code is the priority — virtual threads make blocking code scale similarly to reactive for most CRUD workloads
- You're using Spring Data JPA (no reactive equivalent with full feature parity)

The key insight: virtual threads eliminate the "thread-per-request is expensive" problem for blocking I/O — the JVM creates millions of cheap virtual threads. But they don't give you streaming, backpressure, or the functional composition model of reactive.

---

**Q6: What is R2DBC? How does it compare to JPA?**

R2DBC is a non-blocking database driver specification for relational databases. Repository methods return `Mono`/`Flux` — no thread is blocked waiting for database responses.

Compared to JPA:
- R2DBC: non-blocking, no lazy loading, no entity proxy, no L2 cache, manual joins required, simpler entity model
- JPA/Hibernate: blocking (JDBC), full ORM features, lazy loading, caching, relationship management, mature tooling

Use R2DBC only in **fully reactive applications** where all layers are non-blocking. Mixing R2DBC in a Spring MVC app or JDBC in a WebFlux app is counterproductive — you pay the reactive complexity cost without the benefits.

---

**Q7: What is the difference between `Mono` and `Flux`?**

`Mono<T>` represents an asynchronous sequence of **0 or 1 item**. Analogous to `CompletableFuture<Optional<T>>`. Use for single-item operations: find by ID, save, count.

`Flux<T>` represents an asynchronous sequence of **0 to N items**. Analogous to `Stream<T>` but asynchronous and with backpressure. Use for multi-item operations: find all, streaming, event streams.

Both implement `Publisher<T>` from the Reactive Streams specification. A `Mono` can be converted to `Flux` via `mono.flux()` and a `Flux` to `Mono` via `flux.next()` (first item), `flux.single()` (exactly one), or `flux.collectList()` (all items in a list).

---

**Q8: How do you handle errors in reactive chains?**

Unlike imperative try-catch (which requires blocking), reactive error handling is declarative:

| Operator | Behavior |
|---|---|
| `onErrorReturn(value)` | Replace error with a fallback value |
| `onErrorReturn(ExType.class, value)` | Replace specific error type with fallback |
| `onErrorResume(fn)` | Switch to a fallback publisher on error |
| `onErrorMap(fn)` | Transform the exception type |
| `onErrorContinue(fn)` | Skip errored items and continue (Flux) |
| `retry(n)` | Resubscribe up to n times on any error |
| `retryWhen(spec)` | Conditional retry with backoff |
| `timeout(duration)` | Error if no item within duration |

The ordering matters: operators are applied in sequence. `onErrorResume` for specific types should come before general catch-alls.

---

**Q9: What is a hot publisher vs cold publisher? Give real examples.**

**Cold publisher**: starts producing when subscribed; each subscriber gets its own independent sequence from the beginning.
- Examples: `Flux.just()`, `Flux.range()`, database queries, HTTP requests via WebClient
- Like a music file you play on demand — each listener starts from the beginning

**Hot publisher**: emits regardless of whether anyone is subscribed; subscribers only receive items emitted after they subscribe.
- Examples: `ConnectableFlux`, WebSocket messages, Kafka consumer streams, `Sinks.Many`
- Like a live radio station — you join and hear what's currently playing

Converting cold to hot:
- `flux.publish()` → returns `ConnectableFlux`, connect with `.connect()`
- `flux.share()` → auto-connects on first subscriber, ref-counts
- `Sinks.Many` → push items imperatively to multiple subscribers

---

**Q10: How do you test WebFlux controllers?**

Use `WebTestClient` — a non-blocking test client that verifies reactive HTTP interactions.

Two modes:
1. `@WebFluxTest(YourController.class)` — sliced test, loads only WebFlux layer, mock dependencies with `@MockBean`, `WebTestClient` auto-configured
2. `@SpringBootTest(webEnvironment = RANDOM_PORT)` — full integration test, `WebTestClient` configured against real running server

For service/unit tests, use `StepVerifier` from Project Reactor to verify reactive sequences step by step.

---

**Q11: What is Context in Project Reactor and when do you need it?**

`Context` is the reactive equivalent of `ThreadLocal`. In reactive code, the thread changes mid-chain — `ThreadLocal` breaks because the thread carrying the context may not be the thread processing downstream operators.

Use `Context` when you need to propagate request-scoped data (trace IDs, user IDs, correlation IDs) through a reactive chain.

Key characteristic: context flows **backwards** (upstream). You call `contextWrite()` downstream and read with `Mono.deferContextual()` upstream. Spring Security's reactive support uses this internally.

---

**Q12: What are the Reactive Streams specification rules you must follow?**

1. `onSubscribe` is called exactly once before any other signal
2. `onNext` is called at most the number of items requested (`request(N)`)
3. `onError` and `onComplete` are terminal — no signals can follow
4. Only one of `onError` or `onComplete` can be signalled
5. `Subscription.request(n)` where `n <= 0` signals `IllegalArgumentException`
6. A Subscriber must call `request(N)` to receive items — 0 items are sent if nothing is requested

---

**Q13: What happens if you call `block()` inside a WebFlux request handler?**

You get `IllegalStateException: block()/blockFirst()/blockLast() are blocking, which is not supported in thread reactor-http-nio-X`.

This is because `block()` parks the calling thread while waiting for the result. WebFlux's Netty event loop threads are not allowed to block — they must remain available to handle I/O events. If all event loop threads block, the entire application deadlocks.

The fix: never call `block()` in a reactive chain. Chain operators instead. If you must call blocking code, wrap it:

```java
// WRONG
Mono<User> bad = Mono.just(jdbcRepo.findById(id));  // calls blocking code directly on event loop

// CORRECT
Mono<User> good = Mono.fromCallable(() -> jdbcRepo.findById(id))
    .subscribeOn(Schedulers.boundedElastic());  // blocking call on bounded elastic pool
```

---

**Q14: What is the difference between `merge` and `concat`?**

- `Flux.merge(flux1, flux2)`: subscribes to all publishers **concurrently**. Items from all publishers interleave based on timing. Completion only when all publishers complete.
- `Flux.concat(flux1, flux2)`: subscribes **sequentially** — flux1 must complete before flux2 starts. Order is guaranteed.

Use `merge` when sources produce independently and you want maximum throughput. Use `concat` when you need ordered results or sequential processing.

---

**Q15: How do you implement Server-Sent Events (SSE) with WebFlux?**

```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<StockPrice>> streamStockPrices() {
    return stockPriceService.getLivePrices()
        .map(price -> ServerSentEvent.<StockPrice>builder()
            .id(UUID.randomUUID().toString())
            .event("price-update")
            .data(price)
            .comment("Live stock price")
            .build());
}
```

The `MediaType.TEXT_EVENT_STREAM_VALUE` content type tells Spring to use SSE encoding. The `Flux` stays open, pushing events to the client as they arrive. The HTTP connection remains open until the Flux completes or the client disconnects.

---

## 10. Quick Reference — Comparison Tables

### Operator Cheat Sheet

| Goal | Operator |
|---|---|
| Transform 1-to-1 synchronously | `map` |
| Transform 1-to-many asynchronously (parallel) | `flatMap` |
| Transform 1-to-many asynchronously (sequential) | `concatMap` |
| Transform 1-to-many (cancel previous on new input) | `switchMap` |
| Filter items | `filter` |
| Take first N | `take(n)` |
| Skip first N | `skip(n)` |
| Take while condition true | `takeWhile` |
| Merge multiple publishers (concurrent) | `Flux.merge` |
| Combine publishers (sequential) | `Flux.concat` |
| Combine latest items from N publishers | `Mono.zip` |
| Fallback if empty | `switchIfEmpty` |
| Default value if empty | `defaultIfEmpty` |
| Aggregate to single value | `reduce` |
| Running aggregate | `scan` |
| Collect to List | `collectList` |
| Side effect (no transformation) | `doOnNext`, `doOnError` |
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
| Blocking JDBC in WebFlux | Starves event loop threads | Use R2DBC, or offload to `boundedElastic()` |
| `ThreadLocal` in reactive | Thread changes mid-chain | Use `Context` |
| `Schedulers.parallel()` for blocking | Starves CPU pool | Use `Schedulers.boundedElastic()` |
| `new WebClient()` per request | Creates new connection pool | Use singleton `@Bean` |
| `onErrorContinue` overuse | Silently hides errors | Use `onErrorResume` instead |
| Not subscribing | Nothing executes | Always subscribe or return from handler |
| `Mono.just(blockingCall())` | Blocking call on assembly time | Use `Mono.fromCallable(...)` |
| Hot publisher misunderstanding | Late subscribers miss events | Use `share()` or `replay()` appropriately |
| R2DBC without fully reactive stack | Extra complexity, no benefit | Use Spring MVC + JDBC instead |

---

*Last updated: 2026-06-05 | Covers Spring Boot 3.x, Project Reactor 3.x, Spring WebFlux, R2DBC, Java 17+*

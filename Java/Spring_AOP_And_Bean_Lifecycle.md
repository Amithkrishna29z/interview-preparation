# Spring AOP and Bean Lifecycle — Complete Interview Guide

> **Start here.** Two things get asked in almost every interview: (1) the **order of the bean lifecycle** — when does Spring build a bean, inject dependencies, run init methods, and create proxies? and (2) **how proxies make `@Transactional` (and `@Async`, `@Cacheable`) work** — because once you understand the proxy idea, the "self-invocation" and "private method" gotchas make sense. Everything builds on: *instantiate → inject → initialize → wrap in proxy → use → destroy*.

---

## Part 1: Spring Bean Lifecycle

### The 12-Step Bean Lifecycle

```
1.  Instantiation            → Constructor called
2.  Populate Properties      → @Autowired fields and setter injections resolved
3.  BeanNameAware            → setBeanName(String name) called if implemented
4.  BeanFactoryAware         → setBeanFactory(BeanFactory) called if implemented
5.  ApplicationContextAware  → setApplicationContext(ctx) called if implemented
6.  BeanPostProcessor        → postProcessBeforeInitialization() called for ALL beans
7.  @PostConstruct           → method annotated with @PostConstruct runs
8.  InitializingBean         → afterPropertiesSet() runs if implemented
9.  @Bean(initMethod)        → custom init-method runs if specified
10. BeanPostProcessor        → postProcessAfterInitialization() called (PROXIES CREATED HERE)
11. Bean Ready               → bean placed in ApplicationContext and served
--- context closes ---
12. @PreDestroy              → cleanup method runs
    DisposableBean.destroy() → runs if implemented
    @Bean(destroyMethod)     → custom destroy-method runs
```

---

### Instantiation and Dependency Injection

```java
// Constructor injection (preferred)
@Service
public class OrderService {
    private final UserRepository userRepository;
    private final EmailService emailService;

    @Autowired  // optional in Spring 4.3+ if single constructor
    public OrderService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}

// Field injection (avoid — cannot inject in tests without Spring context)
@Service
public class OrderService {
    @Autowired private UserRepository userRepository;  // not recommended
}

// Setter injection (use when dependency is optional)
@Service
public class OrderService {
    private EmailService emailService;

    @Autowired
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}
```

**Why constructor injection is preferred:**
- Fields can be `final` (immutable)
- NPEs fail at startup, not runtime
- Easy to test without Spring (just pass mocks)
- Circular dependencies fail at startup (caught early)

---

### Initialization Methods (Execution Order)

```java
@Component
public class DataSourceBean implements InitializingBean {

    @PostConstruct          // Step 1: runs first
    public void postConstruct() { System.out.println("@PostConstruct"); }

    @Override
    public void afterPropertiesSet() {  // Step 2: runs second
        System.out.println("InitializingBean.afterPropertiesSet()");
    }
    // Step 3: @Bean(initMethod="customInit") runs last
}
```

**Use cases for init methods:** establishing DB connections, warming caches, validating configuration.

---

### Bean Scopes

```java
@Component
@Scope("singleton")   // default: one instance per ApplicationContext
public class SingletonBean { }

@Component
@Scope("prototype")   // new instance for every injection/getBean call
public class PrototypeBean { }

@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)  // web
public class RequestScopedBean { }
```

**Critical: Injecting prototype into singleton**

```java
// WRONG: prototype becomes singleton (only injected once at startup)
@Service
public class OrderService {
    @Autowired
    private PrototypeBean prototypeBean;  // same instance every time!
}

// Solution 1: ObjectProvider (recommended)
@Service
public class OrderService {
    @Autowired
    private ObjectProvider<PrototypeBean> prototypeBeanProvider;

    public void process() {
        PrototypeBean fresh = prototypeBeanProvider.getObject();  // new instance each time
    }
}

// Solution 2: @Lookup method injection
@Service
public abstract class OrderService {
    @Lookup
    public abstract PrototypeBean getPrototypeBean();  // Spring overrides this
}
```

**Important:** Spring does NOT call `@PreDestroy` or destroy methods for prototype-scoped beans.

---

### Circular Dependencies

```java
// Constructor injection circular dependency → FAILS AT STARTUP
@Service public class ServiceA { public ServiceA(ServiceB b) { } }
@Service public class ServiceB { public ServiceB(ServiceA a) { } }  // BeanCurrentlyInCreationException

// Setter/field injection → Spring resolves with 3-level cache (injects partially-constructed bean)

// Solution 1: @Lazy on one side
@Service
public class ServiceA {
    @Autowired
    public ServiceA(@Lazy ServiceB b) { }  // B is a lazy proxy, real B created on first use
}

// Solution 2: Refactor — extract shared logic into a third service (preferred)
```

---

### Destruction Lifecycle

```java
@Component
public class ResourceBean implements DisposableBean {

    @PreDestroy             // Step 1
    public void preDestroy() { System.out.println("@PreDestroy"); }

    @Override
    public void destroy() { // Step 2
        System.out.println("DisposableBean.destroy()");
    }
    // Step 3: @Bean(destroyMethod="customDestroy") runs last
}
```

Triggered by: `context.close()`, JVM shutdown hook (registered by SpringApplication), or eviction (prototype beans: never).

---

### @Configuration vs @Component (Full Mode vs Lite Mode)

```java
// Full Mode: @Configuration class is CGLIB-proxied
// Inter-bean calls go through proxy → singleton guarantee
@Configuration
public class AppConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB());  // same ServiceB instance returned
    }
    @Bean
    public ServiceB serviceB() { return new ServiceB(); }
}

// Lite Mode: @Component class is NOT proxied
// Inter-bean calls are regular Java calls → creates NEW instance each time
@Component
public class AppConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB());  // different ServiceB from Spring-managed one!
    }
    @Bean
    public ServiceB serviceB() { return new ServiceB(); }
}
```

**Interview Tip:** Use `@Configuration` when `@Bean` methods call each other. `@Component` is fine when they don't.

---

### ApplicationContext Events

```java
// Built-in events
@EventListener
public void onContextRefreshed(ContextRefreshedEvent event) { }

// Custom events
public class UserCreatedEvent {
    private final Long userId;
    public UserCreatedEvent(Long userId) { this.userId = userId; }
    public Long getUserId() { return userId; }
}

@Service
public class UserService {
    @Autowired private ApplicationEventPublisher publisher;

    public User createUser(CreateUserRequest req) {
        User user = userRepository.save(new User(req));
        publisher.publishEvent(new UserCreatedEvent(user.getId()));
        return user;
    }
}

@Component
public class UserCreatedHandler {
    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        emailService.sendWelcomeEmail(event.getUserId());
    }
}
```

Add `@Async` to run a listener on another thread, or use `@TransactionalEventListener(phase = AFTER_COMMIT)` to fire only after the surrounding transaction commits.

---

## Part 2: Spring AOP

AOP (Aspect-Oriented Programming) lets Spring add **cross-cutting concerns** — transactions, logging, security, caching — without cluttering your business code. It works through **proxies**: Spring quietly replaces your real bean with a proxy that runs extra logic before and after calling through to the real bean. This is how `@Transactional` and `@Async` work transparently — and understanding it explains the gotchas below.

### The Self-Invocation Problem — Critical Interview Topic

```java
@Service
public class OrderService {

    @Transactional
    public void processOrder(Long orderId) {
        this.validateStock(orderId);  // BYPASSES PROXY — @Transactional on validateStock is IGNORED
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateStock(Long orderId) { ... }
}
```

**Why:** `this` refers to the real object, not the proxy. Same problem affects `@Async`, `@Cacheable`, any AOP advice.

**Solutions:**

```java
// Solution 1: Inject self (simplest)
@Service
public class OrderService {
    @Autowired @Lazy
    private OrderService self;

    public void processOrder(Long orderId) {
        self.validateStock(orderId);  // goes through proxy
    }
}

// Solution 2: Refactor — move validateStock to a different bean (cleanest)
@Service
public class StockValidationService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateStock(Long orderId) { ... }
}
```

---

### How @Transactional Works Internally

```
1. Startup: TransactionInterceptor registered; @Transactional beans detected
2. Bean creation: BPP creates CGLIB proxy wrapping TransactionInterceptor
3. Runtime call: myService.save(entity)
   → CGLIB Proxy receives call
   → TransactionInterceptor: check propagation, get connection, set autoCommit=false
   → Call real myService.save(entity)
   → Success: commit; RuntimeException: rollback
   → Release connection back to pool
```

**Why @Transactional fails on:**
- **Private methods:** proxy can't override private (CGLIB subclass approach)
- **Final methods:** CGLIB can't override final
- **Self-invocation:** `this.method()` bypasses proxy

**Propagation levels:**
```
REQUIRED      // join existing TX or create new (default)
REQUIRES_NEW  // always create new TX, suspend existing
NESTED        // nested TX within existing (savepoint)
```

**Isolation:** `READ_UNCOMMITTED` < `READ_COMMITTED` (PostgreSQL default) < `REPEATABLE_READ` (MySQL default) < `SERIALIZABLE`

---

### @Async

```java
@Service
public class NotificationService {
    @Async  // runs on a separate thread — same proxy rules apply (no self-invocation)
    public CompletableFuture<Void> sendEmailAsync(String to) {
        emailClient.send(to);
        return CompletableFuture.completedFuture(null);
    }
}
// Enable with @EnableAsync. Define a ThreadPoolTaskExecutor @Bean to control thread pool.
```

---

## Interview Questions & Answers

**Q1: Walk me through the Spring bean lifecycle.**

Instantiation → DI (field/setter) → Aware interfaces → BPP.before → `@PostConstruct` → `afterPropertiesSet()` → initMethod → BPP.after (proxies created here) → Bean ready → (on close) `@PreDestroy` → `DisposableBean.destroy()` → destroyMethod.

**Q2: Why is constructor injection preferred over field injection?**

Fields can be `final` (immutable), failures are caught at startup not runtime, easy to test without Spring, and circular dependencies are detected early.

**Q3: How do you inject a prototype bean into a singleton?**

Use `ObjectProvider<T>` (call `getObject()` each time) or `@Lookup` method injection. Never use direct `@Autowired` — the prototype becomes effectively singleton.

**Q4: What is circular dependency? How is it resolved?**

Bean A needs Bean B and Bean B needs Bean A. Constructor injection fails with `BeanCurrentlyInCreationException` at startup. Setter/field injection is resolved via Spring's three-level bean cache (early exposure of partially-constructed beans). Best fix: `@Lazy` on one side or refactor to remove the cycle.

**Q5: What is Spring AOP and how does it work?**

Spring AOP adds cross-cutting concerns (logging, transactions, security) through proxies. At startup Spring wraps matching beans in a proxy. At runtime, method calls go to the proxy, which runs the extra logic (e.g., opening a transaction) before calling the real method.

**Q6: What is the self-invocation problem?**

When a bean calls its own method via `this.method()`, it bypasses the proxy. `@Transactional`, `@Async`, `@Cacheable` on the called method are silently ignored. Fix: inject self with `@Lazy`, use `ApplicationContext.getBean()`, or move the method to another bean.

**Q7: Why doesn't @Transactional work on private methods?**

The proxy can't intercept private methods (it works by subclassing/implementing interfaces), so the annotation is silently ignored. Make the method public and call it through the proxy.

**Q8: @Configuration full mode vs lite mode?**

`@Configuration` is CGLIB-proxied — inter-`@Bean` method calls return the same singleton. `@Component` with `@Bean` is not proxied — inter-`@Bean` calls are plain Java calls creating new instances. Use `@Configuration` when `@Bean` methods call each other.

---

## Common Pitfalls Table

| Pitfall | Cause | Fix |
|---------|-------|-----|
| @Transactional silently ignored | Self-invocation (`this.method()`) | Inject self with `@Lazy` |
| @Transactional on private method | Proxy can't override private | Make method public |
| @Async on private method | Same proxy limitation | Make method public |
| Prototype acts as singleton | Direct `@Autowired` injection | Use `ObjectProvider` or `@Lookup` |
| Circular dependency (constructor) | A→B, B→A in constructors | `@Lazy` on one, or refactor |
| @PostConstruct never called | Bean not managed by Spring | Ensure `@Component` / `@Bean` present |
| Destroy not called on prototype | Spring doesn't track prototypes | Handle cleanup manually |
| Checked exception doesn't rollback | Not a RuntimeException | `@Transactional(rollbackFor = X.class)` |

---

## Quick Reference Cheat Sheet

```
BEAN LIFECYCLE (order matters):
  1. Instantiate            → constructor runs
  2. Inject dependencies    → @Autowired fields/setters filled
  3. Aware callbacks        → setBeanName / setBeanFactory / setApplicationContext
  4. BPP.before             → postProcessBeforeInitialization
  5. @PostConstruct         → your setup logic
  6. InitializingBean       → afterPropertiesSet()
  7. initMethod             → @Bean(initMethod = "...")
  8. BPP.after              → postProcessAfterInitialization  ← PROXIES CREATED HERE
  9. Bean ready             → served from ApplicationContext
 --- on context close ---
 10. @PreDestroy → DisposableBean.destroy() → @Bean(destroyMethod)
 (prototype beans: destroy callbacks NOT called — you clean up)

INIT ORDER (all three present):  @PostConstruct → afterPropertiesSet() → initMethod
```

### Key Annotations

| Annotation | What it does |
|---|---|
| `@Component` / `@Service` / `@Repository` | Marks a Spring-managed bean |
| `@Autowired` | Injects a dependency (prefer on constructor) |
| `@Scope("singleton" / "prototype")` | One shared instance vs fresh each time |
| `@PostConstruct` / `@PreDestroy` | Setup / cleanup at the right lifecycle point |
| `@Lazy` | Delays creation; also breaks circular dependencies |
| `@Configuration` | Full-mode config — inter-`@Bean` calls return the same singleton |
| `@Transactional` | Wraps method in a DB transaction (via proxy) |
| `@Async` | Runs method on a separate thread (via proxy) |

### Proxy Gotchas (one-liners)

```
Self-invocation    → this.method() skips proxy → annotation ignored. Fix: inject self (@Lazy) or move to another bean.
Private method     → proxy can't override → @Transactional/@Async ignored. Fix: make it public.
Final method/class → CGLIB can't subclass → proxy fails. Fix: remove final.
Checked exception  → does NOT roll back by default. Fix: @Transactional(rollbackFor = X.class).
Prototype in singleton → injected once, acts singleton. Fix: ObjectProvider / @Lookup.
```

---

*Last Updated: 2026-06-18*

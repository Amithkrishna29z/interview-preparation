# Spring AOP and Bean Lifecycle — Complete Interview Guide

---

## Part 1: Spring Bean Lifecycle

### The 12-Step Bean Lifecycle

```
1.  Instantiation            → Constructor called (no-arg or @Autowired constructor)
2.  Populate Properties      → @Autowired fields and setter injections resolved
3.  BeanNameAware            → setBeanName(String name) called if implemented
4.  BeanFactoryAware         → setBeanFactory(BeanFactory) called if implemented
5.  ApplicationContextAware  → setApplicationContext(ctx) called if implemented
6.  BeanPostProcessor        → postProcessBeforeInitialization() called for ALL beans
7.  @PostConstruct           → method annotated with @PostConstruct runs
8.  InitializingBean         → afterPropertiesSet() runs if implemented
9.  @Bean(initMethod)        → custom init-method runs if specified
10. BeanPostProcessor        → postProcessAfterInitialization() called (PROXIES CREATED HERE)
11. Bean Ready               → bean is placed in ApplicationContext and served
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
- Dependencies are explicitly required (fail fast if missing)
- Fields can be `final` (immutable)
- Easy to test without Spring (just pass mocks)
- Circular dependencies fail at startup (caught early)

---

### Aware Interfaces

```java
@Component
public class MyBean implements BeanNameAware, ApplicationContextAware,
                                BeanFactoryAware, EnvironmentAware {

    private String beanName;
    private ApplicationContext context;

    @Override
    public void setBeanName(String name) {
        this.beanName = name;  // called after instantiation
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;  // gives access to the full context
    }

    @Override
    public void setBeanFactory(BeanFactory factory) {
        // access to BeanFactory (lower-level than ApplicationContext)
    }

    @Override
    public void setEnvironment(Environment environment) {
        // access to properties/profiles
    }
}
```

**Other Aware interfaces:** `ResourceLoaderAware`, `MessageSourceAware`, `ApplicationEventPublisherAware`

**Rule:** Avoid Aware interfaces in business logic — they couple your code to Spring. Use them only in infrastructure/framework code.

---

### BeanPostProcessor — The Most Important Lifecycle Extension Point

```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Called BEFORE @PostConstruct and InitializingBean
        // Return same bean or a wrapper
        System.out.println("Before init: " + beanName);
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Called AFTER @PostConstruct and InitializingBean
        // THIS IS WHERE SPRING CREATES AOP PROXIES
        System.out.println("After init: " + beanName);
        return bean;  // can return a proxy instead of the original bean!
    }
}
```

**What Spring uses BeanPostProcessor for internally:**
- `AnnotationAwareAspectJAutoProxyCreator` — creates AOP proxies for `@Transactional`, `@Async`, `@Cacheable`
- `AutowiredAnnotationBeanPostProcessor` — processes `@Autowired`
- `CommonAnnotationBeanPostProcessor` — processes `@PostConstruct`, `@PreDestroy`, `@Resource`
- `PersistenceAnnotationBeanPostProcessor` — processes `@PersistenceContext`

**Ordering multiple BeanPostProcessors:**
```java
@Component
@Order(1)  // lower number = runs first (highest priority)
public class FirstBPP implements BeanPostProcessor { ... }

@Component
@Order(2)
public class SecondBPP implements BeanPostProcessor { ... }
```

---

### BeanFactoryPostProcessor

Runs **before** any beans are instantiated — modifies bean definitions (metadata), not bean instances.

```java
@Component
public class CustomBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        // Modify BeanDefinitions BEFORE beans are created
        BeanDefinition bd = factory.getBeanDefinition("userService");
        bd.setScope(BeanDefinition.SCOPE_PROTOTYPE);  // change scope
    }
}
```

**Spring's own BeanFactoryPostProcessors:**
- `PropertySourcesPlaceholderConfigurer` — resolves `${property.name}` placeholders
- `ConfigurationClassPostProcessor` — processes `@Configuration`, `@ComponentScan`, `@Bean`, `@Import`

| | BeanPostProcessor | BeanFactoryPostProcessor |
|---|---|---|
| When | After instantiation | Before instantiation |
| Operates on | Bean instances | Bean definitions (metadata) |
| Use for | Wrapping/proxying beans | Modifying bean configuration |

---

### Initialization Methods (Execution Order)

```java
@Component
public class DataSourceBean implements InitializingBean {

    @PostConstruct          // Step 1: runs first
    public void postConstruct() {
        System.out.println("@PostConstruct");
    }

    @Override
    public void afterPropertiesSet() {  // Step 2: runs second
        System.out.println("InitializingBean.afterPropertiesSet()");
    }
    // Step 3: @Bean(initMethod="customInit") runs last
}

// In @Configuration
@Bean(initMethod = "customInit", destroyMethod = "customDestroy")
public DataSourceBean dataSourceBean() { return new DataSourceBean(); }
```

**Use cases for init methods:** establishing DB connections, warming caches, registering listeners, validating configuration.

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

@Component
@Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class SessionScopedBean { }
```

**Critical: Injecting prototype into singleton**

```java
// WRONG: prototype becomes singleton (only injected once at startup)
@Service
public class OrderService {
    @Autowired
    private PrototypeBean prototypeBean;  // same instance every time!
}

// Solution 1: @Lookup method injection
@Service
public abstract class OrderService {
    @Lookup
    public abstract PrototypeBean getPrototypeBean();  // Spring overrides this

    public void process() {
        PrototypeBean fresh = getPrototypeBean();  // new instance each time
    }
}

// Solution 2: ObjectProvider (recommended in Spring 4.3+)
@Service
public class OrderService {
    @Autowired
    private ObjectProvider<PrototypeBean> prototypeBeanProvider;

    public void process() {
        PrototypeBean fresh = prototypeBeanProvider.getObject();
    }
}

// Solution 3: ApplicationContext.getBean()
@Service
public class OrderService implements ApplicationContextAware {
    private ApplicationContext ctx;

    public void process() {
        PrototypeBean fresh = ctx.getBean(PrototypeBean.class);
    }
}
```

**Important:** Spring does NOT call `@PreDestroy` or destroy methods for prototype-scoped beans. The caller is responsible for cleanup.

---

### Circular Dependencies

```java
// Constructor injection circular dependency → FAILS AT STARTUP
@Service
public class ServiceA {
    public ServiceA(ServiceB b) { }  // A needs B
}
@Service
public class ServiceB {
    public ServiceB(ServiceA a) { }  // B needs A → BeanCurrentlyInCreationException
}

// Setter injection circular dependency → Spring resolves with 3-level cache
@Service
public class ServiceA {
    @Autowired private ServiceB b;  // Spring creates A first (partially), injects into B
}
@Service
public class ServiceB {
    @Autowired private ServiceA a;  // Spring injects partially-constructed A
}

// Solution 1: @Lazy on one side
@Service
public class ServiceA {
    @Autowired
    public ServiceA(@Lazy ServiceB b) { }  // B is a lazy proxy, real B created on first use
}

// Solution 2: Refactor — extract shared logic into a third service
// ServiceC holds shared logic; A and B both depend on C (no cycle)
```

---

### Destruction Lifecycle

```java
@Component
public class ResourceBean implements DisposableBean {

    @PreDestroy             // Step 1
    public void preDestroy() {
        System.out.println("@PreDestroy — releasing resources");
    }

    @Override
    public void destroy() { // Step 2
        System.out.println("DisposableBean.destroy()");
    }
    // Step 3: @Bean(destroyMethod="customDestroy") runs last
}
```

Destruction is triggered when:
- `context.close()` is called
- JVM shutdown hook fires (SpringApplication registers this automatically)
- Bean is evicted (prototype beans: never)

---

### @Configuration vs @Component (Lite Mode vs Full Mode)

```java
// Full Mode: @Configuration class is CGLIB-proxied
// Inter-bean method calls go through the proxy → singleton guarantee
@Configuration
public class AppConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB());  // calls serviceB() through proxy
        // same ServiceB instance returned (singleton)
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}

// Lite Mode: @Component class is NOT proxied
@Component
public class AppConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB());  // direct Java method call!
        // Creates a NEW ServiceB instance — different from the Spring-managed one!
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
```

**Interview Tip:** This is why `@Configuration` is needed when `@Bean` methods call other `@Bean` methods. `@Component` works fine when `@Bean` methods don't call each other.

---

### ApplicationContext Events

```java
// Built-in events
@EventListener
public void onContextRefreshed(ContextRefreshedEvent event) {
    // Fires when context is fully initialized (including restarts)
}

@EventListener
public void onContextClosed(ContextClosedEvent event) {
    // Fires when context is being closed
}

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

    // Async event listener
    @Async
    @EventListener
    public void handleUserCreatedAsync(UserCreatedEvent event) {
        analyticsService.track(event.getUserId());
    }

    // Only fires if transaction commits (not if rolled back)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleUserCreatedAfterCommit(UserCreatedEvent event) {
        auditService.recordCreation(event.getUserId());
    }
}
```

---

## Part 2: Spring AOP — Complete Deep Dive

### AOP Terminology

| Term | Definition | Analogy |
|------|-----------|---------|
| **Aspect** | Class containing cross-cutting logic | Security guard station |
| **Join Point** | Point in execution (Spring: always a method call) | Doorway in a building |
| **Pointcut** | Expression selecting which join points to intercept | "All doors on floor 3" |
| **Advice** | Action taken at a join point | What the guard does at the door |
| **Weaving** | Applying aspects to targets | Installing the guard |
| **Target Object** | The original bean being advised | The actual room |
| **AOP Proxy** | Wrapper around target (JDK proxy or CGLIB) | Guard standing in for the room |

**Cross-cutting concerns AOP solves:** logging, security checks, transaction management, caching, retry logic, auditing, rate limiting, performance monitoring.

---

### Spring AOP vs AspectJ

| Aspect | Spring AOP | AspectJ |
|--------|-----------|---------|
| Weaving time | Runtime (proxy-based) | Compile-time or load-time |
| Join points | Method execution only | Field access, constructor, method, cast, etc. |
| Requires AspectJ compiler | No | Yes (ajc) |
| Works on Spring beans | Yes | Any Java object |
| Performance | Proxy overhead per call | None at runtime |
| Self-invocation | Doesn't work | Works |
| Setup | Zero config needed | Needs special compiler/agent |

**When to use AspectJ over Spring AOP:**
- Need to intercept field access, constructor calls, static methods
- Need aspects on non-Spring objects (POJOs not managed by Spring)
- Self-invocation must be intercepted
- Performance-critical path where proxy overhead matters

---

### Proxy Mechanisms — How Spring AOP Works

```
ApplicationContext starts
        ↓
AnnotationAwareAspectJAutoProxyCreator (a BeanPostProcessor)
        ↓
postProcessAfterInitialization() called for each bean
        ↓
  Does bean match any pointcut?
    YES → Create Proxy (JDK or CGLIB) and return proxy
     NO → Return original bean
        ↓
Proxy is stored in context (not the original bean)
        ↓
At runtime: caller calls proxy method
        ↓
Proxy → runs advice chain → calls original bean method
```

#### JDK Dynamic Proxy
- Created when target bean **implements at least one interface**
- Proxy implements the same interfaces
- Implemented via `java.lang.reflect.Proxy` + `InvocationHandler`
- Cannot cast proxy to the concrete class

```java
// JDK dynamic proxy — manual example showing the mechanism
public class LoggingHandler implements InvocationHandler {
    private final Object target;

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before: " + method.getName());
        Object result = method.invoke(target, args);  // calls real method
        System.out.println("After: " + method.getName());
        return result;
    }
}
```

#### CGLIB Proxy
- Created when target bean has **no interface** (or `proxyTargetClass=true`)
- Generates a subclass of the target class at runtime
- **Cannot proxy `final` classes or `final` methods**
- Spring Boot 2.x+: **CGLIB is the default** even when interfaces exist
- Enabled globally: `@EnableAspectJAutoProxy(proxyTargetClass = true)`

**How to check which proxy type you have:**
```java
@Autowired OrderService orderService;
System.out.println(orderService.getClass().getName());
// JDK:  com.sun.proxy.$Proxy47
// CGLIB: com.example.OrderService$$EnhancerBySpringCGLIB$$abc123
```

---

### The Self-Invocation Problem — Critical Interview Topic

```java
@Service
public class OrderService {

    @Transactional
    public void processOrder(Long orderId) {
        // THIS WORKS: proxy intercepts this call
        prepareItems(orderId);

        // THIS DOES NOT WORK: self-invocation bypasses proxy
        this.validateStock(orderId);  // goes to real object, not proxy!
        // @Transactional(REQUIRES_NEW) on validateStock is IGNORED
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateStock(Long orderId) {
        // This annotation is silently ignored when called via this.
    }
}
```

**Why it happens:** When `processOrder` calls `this.validateStock`, `this` refers to the real object, not the proxy. The proxy is bypassed entirely.

**The same problem affects:** `@Async`, `@Cacheable`, `@Retry`, any AOP advice.

#### Solutions:

```java
// Solution 1: Inject self (simplest)
@Service
public class OrderService {
    @Autowired
    @Lazy  // prevents circular dependency during construction
    private OrderService self;

    @Transactional
    public void processOrder(Long orderId) {
        self.validateStock(orderId);  // goes through proxy
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateStock(Long orderId) { ... }
}

// Solution 2: ApplicationContext.getBean()
@Service
public class OrderService implements ApplicationContextAware {
    private ApplicationContext ctx;

    public void processOrder(Long orderId) {
        OrderService proxy = ctx.getBean(OrderService.class);
        proxy.validateStock(orderId);
    }
}

// Solution 3: Refactor — move validateStock to a different bean
@Service
public class StockValidationService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateStock(Long orderId) { ... }
}

// Solution 4: Use AspectJ weaving (compile-time, avoids proxy entirely)
// Add aspectj-maven-plugin and remove Spring AOP
```

---

### Advice Types — Complete Reference

#### @Before
```java
@Aspect
@Component
public class LoggingAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint jp) {
        System.out.println("Entering: " + jp.getSignature().toShortString());
        System.out.println("Args: " + Arrays.toString(jp.getArgs()));
    }
}
```

#### @AfterReturning
```java
@AfterReturning(
    pointcut = "execution(* com.example.service.*.*(..))",
    returning = "result"   // binds return value to parameter
)
public void logAfterReturning(JoinPoint jp, Object result) {
    System.out.println("Returned: " + result);
}
```

#### @AfterThrowing
```java
@AfterThrowing(
    pointcut = "execution(* com.example.service.*.*(..))",
    throwing = "ex"   // binds exception to parameter
)
public void logException(JoinPoint jp, Exception ex) {
    System.out.println("Exception in " + jp.getSignature() + ": " + ex.getMessage());
}
```

#### @After (finally)
```java
@After("execution(* com.example.service.*.*(..))")
public void logAfter(JoinPoint jp) {
    // Runs whether method completed normally OR threw exception
    System.out.println("Exiting: " + jp.getSignature().toShortString());
}
```

#### @Around (Most Powerful)
```java
@Around("execution(* com.example.service.*.*(..))")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();

    Object result;
    try {
        result = pjp.proceed();       // invoke actual method
        // result = pjp.proceed(newArgs); // invoke with modified args
        return result;                // can modify return value
    } catch (Exception ex) {
        log.error("Exception in {}", pjp.getSignature(), ex);
        throw ex;                     // re-throw or handle
    } finally {
        long elapsed = System.currentTimeMillis() - start;
        log.info("{} took {}ms", pjp.getSignature(), elapsed);
    }
}
```

**When to use which:**
- `@Before`: validation, logging entry, security checks
- `@AfterReturning`: logging result, modifying return value
- `@AfterThrowing`: error logging, exception translation
- `@After`: resource cleanup (like finally block)
- `@Around`: timing, retry, circuit breaker, caching, transaction management — anything needing control over method execution

---

### Pointcut Expressions — Complete Reference

```java
@Aspect
@Component
public class Pointcuts {

    // execution: matches method signatures
    @Pointcut("execution(public * com.example.service.*.*(..))")
    // execution(modifiers? returnType declaringType? methodName(params) throws?)
    public void serviceLayer() {}

    // within: all methods in matching type
    @Pointcut("within(com.example.service..*)")  // .. = any subpackage
    public void withinServicePackage() {}

    // @annotation: methods annotated with specific annotation
    @Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")
    public void transactionalMethods() {}

    // @within: all methods in type annotated with
    @Pointcut("@within(org.springframework.stereotype.Service)")
    public void inServiceAnnotatedType() {}

    // args: methods accepting specific argument types
    @Pointcut("args(String, ..)")  // first arg is String, rest any
    public void stringFirstArg() {}

    // @args: methods where first argument is annotated with
    @Pointcut("@args(com.example.Validated)")
    public void validatedArgs() {}

    // bean: Spring bean name pattern
    @Pointcut("bean(*Service)")   // all beans ending with "Service"
    public void beanNamed() {}

    // this: proxy type (what the proxy IS)
    @Pointcut("this(com.example.MyInterface)")
    public void proxyImplementsInterface() {}

    // target: target object type (what the proxy wraps)
    @Pointcut("target(com.example.MyInterface)")
    public void targetImplementsInterface() {}

    // Combining with &&, ||, !
    @Pointcut("serviceLayer() && !beanNamed()")
    public void serviceExcludingNamedBeans() {}
}
```

#### Accessing Annotations and Arguments in Advice
```java
// Bind annotation object to advice parameter
@Around("@annotation(transactional)")  // lowercase = parameter name
public Object aroundTransactional(ProceedingJoinPoint pjp, 
                                   Transactional transactional) throws Throwable {
    Propagation propagation = transactional.propagation();
    // ...
    return pjp.proceed();
}

// Bind argument value to advice parameter
@Before("execution(* *.*(..)) && args(userId, ..)")
public void beforeWithUserId(JoinPoint jp, Long userId) {
    log.info("Accessing data for user: {}", userId);
}
```

---

### Practical AOP Examples

#### 1. Performance Timing Aspect
```java
@Aspect
@Component
@Slf4j
public class PerformanceAspect {

    @Around("@within(org.springframework.stereotype.Service) || " +
            "@within(org.springframework.web.bind.annotation.RestController)")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        String method = pjp.getSignature().toShortString();
        try {
            return pjp.proceed();
        } finally {
            long elapsed = System.currentTimeMillis() - start;
            if (elapsed > 500) {
                log.warn("SLOW: {} took {}ms", method, elapsed);
            } else {
                log.debug("{} took {}ms", method, elapsed);
            }
        }
    }
}
```

#### 2. Retry Aspect with Exponential Backoff
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Retryable {
    int maxAttempts() default 3;
    Class<? extends Throwable>[] on() default {Exception.class};
    long backoffMs() default 100;
}

@Aspect
@Component
public class RetryAspect {

    @Around("@annotation(retryable)")
    public Object retry(ProceedingJoinPoint pjp, Retryable retryable) throws Throwable {
        int attempts = 0;
        Throwable lastException = null;
        long backoff = retryable.backoffMs();

        while (attempts < retryable.maxAttempts()) {
            try {
                return pjp.proceed();
            } catch (Throwable ex) {
                if (!isRetryable(ex, retryable.on())) throw ex;
                lastException = ex;
                attempts++;
                if (attempts < retryable.maxAttempts()) {
                    Thread.sleep(backoff);
                    backoff *= 2;  // exponential backoff
                }
            }
        }
        throw lastException;
    }

    private boolean isRetryable(Throwable ex, Class<? extends Throwable>[] retryOn) {
        return Arrays.stream(retryOn).anyMatch(c -> c.isInstance(ex));
    }
}

// Usage
@Service
public class PaymentService {
    @Retryable(maxAttempts = 3, on = {IOException.class}, backoffMs = 200)
    public PaymentResult charge(Payment payment) { ... }
}
```

#### 3. Audit Logging Aspect
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Audited {
    String action();
}

@Aspect
@Component
public class AuditAspect {

    @AfterReturning(
        pointcut = "@annotation(audited)",
        returning = "result"
    )
    public void auditAfterSuccess(JoinPoint jp, Audited audited, Object result) {
        String user = SecurityContextHolder.getContext().getAuthentication().getName();
        auditService.log(AuditEntry.builder()
            .action(audited.action())
            .user(user)
            .method(jp.getSignature().toShortString())
            .args(jp.getArgs())
            .result(result)
            .timestamp(Instant.now())
            .build());
    }

    @AfterThrowing(
        pointcut = "@annotation(audited)",
        throwing = "ex"
    )
    public void auditAfterFailure(JoinPoint jp, Audited audited, Exception ex) {
        auditService.logFailure(audited.action(), ex.getMessage());
    }
}
```

---

### How @Transactional Works Internally (AOP Under the Hood)

```
1. Spring startup:
   TransactionInterceptor (an AOP Advice) is registered
   AnnotationAwareAspectJAutoProxyCreator detects @Transactional beans

2. Bean creation:
   BeanPostProcessor.postProcessAfterInitialization() creates a CGLIB proxy
   The proxy wraps TransactionInterceptor around every @Transactional method

3. Runtime call: myService.save(entity)
   ↓
   CGLIB Proxy receives call
   ↓
   TransactionInterceptor.invoke() executes:
     - Check propagation: is there an active transaction?
     - Get connection from pool (DataSourceTransactionManager)
     - Set autoCommit = false
     - Call real myService.save(entity)
     - If success: connection.commit()
     - If RuntimeException: connection.rollback()
     - Release connection back to pool
```

**Why @Transactional fails on:**
- **Private methods:** proxy can't override private methods (CGLIB subclass approach)
- **Final methods:** CGLIB subclass can't override final methods
- **Self-invocation:** `this.method()` bypasses proxy — goes to real object

**Propagation levels:**
```java
REQUIRED          // join existing TX or create new (default)
REQUIRES_NEW      // always create new TX, suspend existing
NESTED            // nested TX within existing (savepoint)
SUPPORTS          // join if exists, no TX if not
NOT_SUPPORTED     // suspend existing TX, run without TX
MANDATORY         // must have existing TX, else throw
NEVER             // must NOT have TX, else throw
```

**Isolation levels:**
```java
READ_UNCOMMITTED  // dirty reads possible
READ_COMMITTED    // no dirty reads, non-repeatable reads possible (PostgreSQL default)
REPEATABLE_READ   // no non-repeatable reads, phantom reads possible (MySQL InnoDB default)
SERIALIZABLE      // full isolation, worst performance
```

---

### @Async Works the Same Way

```java
@Service
public class NotificationService {

    // @Async works via AOP proxy — same self-invocation problem applies
    @Async
    public CompletableFuture<Void> sendEmailAsync(String to) {
        // Runs in a separate thread (TaskExecutor)
        emailClient.send(to);
        return CompletableFuture.completedFuture(null);
    }
}

// Enable @Async
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean
    public TaskExecutor asyncTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

---

### Aspect Ordering

```java
@Aspect
@Order(1)  // Outermost — runs first entering, last exiting
@Component
public class SecurityAspect {
    @Around("serviceLayer()")
    public Object secure(ProceedingJoinPoint pjp) throws Throwable {
        checkPermission();  // runs first
        Object result = pjp.proceed();
        return result;
    }
}

@Aspect
@Order(2)  // Middle
@Component
public class LoggingAspect {
    @Around("serviceLayer()")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        log.info("Entering");
        Object result = pjp.proceed();
        log.info("Exiting");
        return result;
    }
}

@Aspect
@Order(3)  // Innermost — closest to actual method
@Component
public class MetricsAspect { ... }
```

**Execution order for @Order(1), @Order(2), @Order(3):**
```
Security.before → Logging.before → Metrics.before
→ ACTUAL METHOD
Metrics.after → Logging.after → Security.after
```

**Spring's internal aspect ordering:**
- `@Transactional`: `Integer.MAX_VALUE - 1` (very low priority = innermost proxy)
- `@Async`: `Integer.MAX_VALUE` (lowest priority)

---

## Interview Questions & Answers

**Q1: Walk me through the complete Spring bean lifecycle.**

Instantiation → DI (field/setter) → Aware interfaces → BeanPostProcessor.before → @PostConstruct → InitializingBean → initMethod → BeanPostProcessor.after (proxies created here) → Bean ready → (on close) @PreDestroy → DisposableBean → destroyMethod.

**Q2: What is BeanPostProcessor? What does Spring use it for?**

BeanPostProcessor intercepts bean creation to modify or wrap beans. Spring uses it to create AOP proxies (`AnnotationAwareAspectJAutoProxyCreator`), inject `@Autowired` dependencies (`AutowiredAnnotationBeanPostProcessor`), and call `@PostConstruct`/`@PreDestroy` (`CommonAnnotationBeanPostProcessor`). The proxy creation happens in `postProcessAfterInitialization`.

**Q3: What is the difference between BeanPostProcessor and BeanFactoryPostProcessor?**

BeanFactoryPostProcessor runs **before** any beans are created — it modifies bean definitions (metadata). BeanPostProcessor runs **after** each bean is instantiated — it modifies bean instances. `ConfigurationClassPostProcessor` is a BeanFactoryPostProcessor; `AutowiredAnnotationBeanPostProcessor` is a BeanPostProcessor.

**Q4: Why is constructor injection preferred over field injection?**

Fields can be final (immutable), NPEs fail at startup not runtime, easy to test without Spring (just call constructor with mocks), circular dependencies fail early at startup, and intent is explicit.

**Q5: How do you inject a prototype bean into a singleton?**

Three ways: `@Lookup` method injection (Spring overrides method), `ObjectProvider<T>` (call `getObject()` each time), or `ApplicationContext.getBean(Type.class)`. Never use direct `@Autowired` injection — prototype becomes effectively singleton.

**Q6: What is circular dependency? How is it resolved?**

When Bean A depends on Bean B and Bean B depends on Bean A. Constructor injection circular deps fail with `BeanCurrentlyInCreationException` (good — caught at startup). Setter/field injection can be resolved by Spring's three-level bean cache (early exposure of partially-constructed beans). Best fix: `@Lazy` on one side or refactor to remove the cycle.

**Q7: What is Spring AOP and how does it work?**

Spring AOP enables cross-cutting concerns (logging, transactions, security) through proxies. At startup, `AnnotationAwareAspectJAutoProxyCreator` wraps matching beans in a CGLIB or JDK dynamic proxy. At runtime, method calls go to the proxy, which runs the advice chain and then calls the real method.

**Q8: What is the difference between Spring AOP and AspectJ?**

Spring AOP uses runtime proxies and only intercepts method calls on Spring beans. AspectJ uses compile-time or load-time weaving and can intercept any join point (field access, constructors, static methods) on any Java object. Spring AOP is simpler to use; AspectJ is more powerful.

**Q9: What is the self-invocation problem?**

When a Spring bean calls another of its own methods via `this.method()`, it bypasses the proxy. The proxy only intercepts calls from outside the bean. So `@Transactional`, `@Async`, `@Cacheable` on the called method are silently ignored. Solutions: inject self with `@Lazy`, use `ApplicationContext.getBean()`, or refactor the method to a different bean.

**Q10: Why doesn't @Transactional work on private methods?**

Spring AOP creates proxies by subclassing (CGLIB) or implementing interfaces (JDK proxy). Private methods cannot be overridden by subclasses (CGLIB) and are not part of interfaces (JDK proxy), so the proxy cannot intercept them. The annotation is silently ignored.

**Q11: What is CGLIB proxy vs JDK dynamic proxy?**

JDK dynamic proxy: created when bean implements interface, uses `java.lang.reflect.Proxy`. Cannot proxy concrete class directly. CGLIB proxy: creates a runtime subclass of the target class. Can proxy concrete classes but cannot proxy final classes/methods. Spring Boot 2.x+ uses CGLIB by default even when interfaces exist.

**Q12: What is the difference between @Before, @After, and @Around?**

`@Before` runs before method execution but cannot prevent the call or change return value. `@After` runs after method execution (like finally), regardless of outcome. `@Around` wraps the entire method — calls `pjp.proceed()` to invoke the actual method and can control execution, modify args/return value, or prevent execution entirely.

**Q13: What is a pointcut expression? Give examples.**

Pointcut expressions select which join points to intercept. Examples: `execution(* com.example.service.*.*(..))` — all methods in service package; `@annotation(Transactional)` — methods with @Transactional; `within(com.example.service.*)` — all methods within service classes; `bean(*Service)` — all beans ending with "Service".

**Q14: How do you control the order of multiple aspects?**

Use `@Order(n)` on the aspect class. Lower number = higher priority = outermost proxy in the chain. The outermost advice runs first before the method and last after the method. Implement `Ordered` interface for programmatic ordering.

**Q15: What is @Configuration full mode vs lite mode?**

`@Configuration` class is CGLIB-proxied (full mode). Inter-`@Bean` method calls go through the proxy, ensuring singleton semantics. `@Component` class with `@Bean` is NOT proxied (lite mode) — inter-`@Bean` method calls are regular Java calls, creating new instances each time. Use `@Configuration` when `@Bean` methods call each other.

---

## Common Pitfalls Table

| Pitfall | Cause | Fix |
|---------|-------|-----|
| @Transactional silently ignored | Self-invocation (this.method()) | Inject self with @Lazy |
| @Transactional on private method | Proxy can't override private | Make method package-private or public |
| @Async on private method | Same proxy limitation | Make method public |
| Prototype acts as singleton | Direct @Autowired injection | Use ObjectProvider or @Lookup |
| Circular dependency (constructor) | A→B, B→A in constructors | @Lazy on one, or refactor |
| @PostConstruct never called | Bean not managed by Spring | Ensure @Component / @Bean present |
| Wrong proxy type | Expect interface proxy, get CGLIB | Check Spring Boot defaults |
| Aspect not applied | Bean not in component scan | Ensure @EnableAspectJAutoProxy |
| @EventListener fires twice | Multiple context init (refresh) | Filter by event type or use condition |
| Destroy not called on prototype | Spring doesn't track prototypes | Handle cleanup manually |
| @Transactional on new thread | TX bound to thread | Use @Transactional(propagation=REQUIRED) on called service |
| initMethod exception swallowed | Checked exception in init | Wrap in RuntimeException |
| CGLIB fails | final class or method | Remove final modifier |

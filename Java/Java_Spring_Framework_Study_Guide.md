# Java Spring Framework – Complete Study Guide

## Table of Contents
1. [What is Spring?](#1-what-is-spring)
2. [Inversion of Control (IoC) & Dependency Injection (DI)](#2-inversion-of-control-ioc--dependency-injection-di)
3. [Spring Bean Lifecycle](#3-spring-bean-lifecycle)
4. [Bean Scopes](#4-bean-scopes)
5. [Core Annotations](#5-core-annotations)
6. [Spring Boot Auto-Configuration](#6-spring-boot-auto-configuration)
7. [Spring MVC & REST](#7-spring-mvc--rest)
8. [Spring Data JPA](#8-spring-data-jpa)
9. [Spring Security](#9-spring-security)
10. [Spring AOP (Aspect-Oriented Programming)](#10-spring-aop)
11. [Spring Transactions](#11-spring-transactions)
12. [Exception Handling](#12-exception-handling)
13. [Configuration & Profiles](#13-configuration--profiles)
14. [Common Mistakes & Pitfalls](#14-common-mistakes--pitfalls)
15. [Interview Tips & Quick Revision](#15-interview-tips--quick-revision)

---

## 1. What is Spring?

Spring is a lightweight, open-source Java framework that provides Dependency Injection, AOP, data access abstractions, and Web MVC for building REST APIs.

### Spring vs Spring Boot

| Feature | Spring Framework | Spring Boot |
|---------|-----------------|-------------|
| Configuration | Manual XML / Java config | Auto-configured |
| Server | External (deploy WAR) | Embedded Tomcat/Jetty |
| Setup time | High boilerplate | Minimal, production-ready quickly |
| Starters | None | Curated dependency bundles |
| Actuator | Manual setup | Built-in `/actuator` endpoints |

---

## 2. Inversion of Control (IoC) & Dependency Injection (DI)

Instead of your class creating its own dependencies (`new SomeService()`), Spring creates and injects them. Control of object creation is *inverted* — from your code to the framework.

### Types of Dependency Injection

**Constructor Injection (Recommended)** — dependencies are immutable (`final`), testable without Spring, fails fast if missing.
```java
@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {  // @Autowired optional for single constructor
        this.paymentService = paymentService;
    }
}
```

**Setter Injection** — for optional dependencies. **Field Injection** (`@Autowired` directly on field) — avoid; hides dependencies and makes testing hard.

### ApplicationContext — The IoC Container

| Container Type | Usage |
|---------------|-------|
| `AnnotationConfigApplicationContext` | Java-based config (`@Configuration`) |
| `ClassPathXmlApplicationContext` | XML-based config (legacy) |
| `SpringApplication.run()` | Spring Boot (most common) |

---

## 3. Spring Bean Lifecycle

```
1. Instantiation      → Spring creates the bean via constructor
2. Property Injection → @Autowired / @Value fields are set
3. @PostConstruct     → Your custom init logic runs
4. Bean is READY      → Available for use
5. @PreDestroy        → Your custom cleanup logic runs (on shutdown)
```

```java
@Component
public class DatabaseConnectionPool {

    @PostConstruct
    public void init() { /* open connections, warm up cache */ }

    @PreDestroy
    public void cleanup() { /* close all connections */ }
}
```

---

## 4. Bean Scopes

| Scope | Instances | Lifespan | Use Case |
|-------|-----------|----------|----------|
| `singleton` (default) | One per container | App lifetime | Stateless services, repositories |
| `prototype` | New per injection | Until GC | Stateful beans |
| `request` | One per HTTP request | Single request | Web-layer data holders |
| `session` | One per HTTP session | User session | Shopping cart, user preferences |

**Common gotcha:** Injecting a `prototype` into a `singleton` — the prototype only gets created once. Use `@Lookup` or `ApplicationContext.getBean()` for a fresh instance each time.

---

## 5. Core Annotations

### Stereotype Annotations

| Annotation | Purpose | Layer |
|------------|---------|-------|
| `@Component` | Generic Spring-managed bean | Any |
| `@Service` | Business logic | Service |
| `@Repository` | Data access + exception translation | DAO |
| `@Controller` | MVC controller (returns views) | Web |
| `@RestController` | `@Controller` + `@ResponseBody` | REST API |

### Configuration & Injection

```java
@Configuration
public class AppConfig {
    @Bean                          // Method return value registered as a bean
    public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
}

@Autowired              // Inject by type
@Qualifier("emailNotifier")  // Pick specific implementation by name
@Primary                // Default when multiple implementations exist
@Value("${app.name}")   // Inject from properties file
```

### Multiple Implementations — Qualifier Example

```java
@Service("emailNotifier")
public class EmailNotificationService implements NotificationService { ... }

@Service
public class AlertService {
    @Autowired @Qualifier("emailNotifier")
    private NotificationService notifier;
}
```

---

## 6. Spring Boot Auto-Configuration

Spring Boot reads `AutoConfiguration.imports` and applies `@Conditional` checks to activate only relevant configurations based on the classpath.

```java
@SpringBootApplication  // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class MyApp {
    public static void main(String[] args) { SpringApplication.run(MyApp.class, args); }
}
```

**Key conditional annotations:**
```java
@ConditionalOnClass(DataSource.class)          // Only if DataSource is on classpath
@ConditionalOnMissingBean(DataSource.class)    // Only if no DataSource bean defined yet
@ConditionalOnProperty(name = "feature.enabled", havingValue = "true")
```

### application.properties Key Settings

```properties
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=postgres
spring.datasource.password=secret
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
logging.level.com.example=DEBUG
management.endpoints.web.exposure.include=health,info,metrics
```

---

## 7. Spring MVC & REST

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")                     // GET /api/users/1
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping                             // POST /api/users
    public ResponseEntity<UserDto> createUser(@RequestBody @Valid CreateUserRequest req) {
        UserDto created = userService.create(req);
        return ResponseEntity.created(URI.create("/api/users/" + created.getId())).body(created);
    }

    @DeleteMapping("/{id}")                  // DELETE /api/users/1
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

**Key parameter annotations:** `@PathVariable` (URI segment), `@RequestParam` (query string), `@RequestBody` (JSON body), `@RequestHeader` (HTTP header).

**Validation:** annotate the request class with `@NotBlank`, `@Email`, `@Min`, `@Size`; trigger with `@Valid` on the method parameter.

---

## 8. Spring Data JPA

JPA translates between Java objects and database tables — you define the shape in Java, JPA handles the SQL.

### Entity & Repository

```java
@Entity @Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    @Column(nullable = false) private String name;
    @Column(unique = true, nullable = false) private String email;
    @Enumerated(EnumType.STRING) private UserRole role;
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();
}

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);           // auto-generated from method name
    Page<User> findByRole(UserRole role, Pageable pageable);

    @Query("SELECT u FROM User u WHERE u.role = :role")
    List<User> findByRoleCustom(@Param("role") UserRole role);
}
```

### Relationships

```java
// Many-to-One (Order side)
@ManyToOne(fetch = FetchType.LAZY) @JoinColumn(name = "user_id")
private User user;

// Many-to-Many (User ↔ Role)
@ManyToMany @JoinTable(name = "user_roles",
    joinColumns = @JoinColumn(name = "user_id"),
    inverseJoinColumns = @JoinColumn(name = "role_id"))
private Set<Role> roles = new HashSet<>();
```

### Fetch Type & N+1

| Type | When loaded | Problem |
|------|------------|---------|
| `EAGER` | Immediately | N+1 queries, loads unused data |
| `LAZY` (preferred) | On access | `LazyInitializationException` outside transaction |

```java
// Fix N+1 with JOIN FETCH
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

---

## 9. Spring Security

```java
@Configuration @EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
}
```

### JWT Filter (key logic)

Extend `OncePerRequestFilter`, read the `Authorization: Bearer <token>` header, validate the token, and set authentication in `SecurityContextHolder`. If no token, pass the request down the chain unchanged.

### Method-Level Security

```java
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public List<Order> getOrdersForUser(Long userId) { ... }

@Secured("ROLE_ADMIN")
public void deleteOrder(Long orderId) { ... }
```

---

## 10. Spring AOP

AOP adds cross-cutting concerns (logging, security, auditing) by intercepting method calls without modifying the methods themselves.

### Key Concepts

| Term | Meaning |
|------|---------|
| **Aspect** | Class containing cross-cutting logic |
| **Advice** | Action to run (`@Before`, `@After`, `@Around`) |
| **Pointcut** | Expression matching which methods to intercept |
| **Join Point** | The actual method being intercepted |

```java
@Aspect @Component
public class LoggingAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint jp) { /* runs before method */ }

    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))", throwing = "ex")
    public void logException(JoinPoint jp, Exception ex) { /* runs on exception */ }

    // @Around is the most powerful — must call jp.proceed() to invoke the actual method
    @Around("@annotation(com.example.annotation.Timed)")
    public Object measureTime(ProceedingJoinPoint jp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = jp.proceed();
        System.out.println(jp.getSignature() + " took " + (System.currentTimeMillis() - start) + "ms");
        return result;
    }
}
```

---

## 11. Spring Transactions

A transaction ensures all operations succeed together or all roll back — no partial success.

```java
@Service
public class TransferService {

    @Transactional                          // REQUIRED: join existing or create new tx
    public void transferMoney(Long from, Long to, BigDecimal amount) {
        accountRepository.debit(from, amount);
        accountRepository.credit(to, amount);   // Both roll back if either fails
    }

    @Transactional(readOnly = true)         // Disables dirty checking — better performance
    public List<Account> getAllAccounts() { return accountRepository.findAll(); }
}
```

### Key Propagation Levels

| Propagation | Behavior |
|-------------|----------|
| `REQUIRED` (default) | Join existing or create new |
| `REQUIRES_NEW` | Always new tx — suspends existing (use for audit logs) |
| `NESTED` | Savepoint inside existing tx |
| `NOT_SUPPORTED` | Suspend existing tx, run without |

**Self-invocation pitfall:** Calling a `@Transactional` method from within the same class bypasses the Spring proxy — the annotation is ignored.

---

## 12. Exception Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest().body(new ErrorResponse("VALIDATION_FAILED", errors));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity.internalServerError()
            .body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred"));
    }
}

public record ErrorResponse(String code, String message) {}
```

---

## 13. Configuration & Profiles

```java
// Activate: spring.profiles.active=dev  (properties / JVM arg / env var)

@Configuration @Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.H2).build();
    }
}
```

**Prefer `@ConfigurationProperties` over scattered `@Value` annotations:**
```java
@ConfigurationProperties(prefix = "app.email")
@Component
public class EmailProperties {
    private String host;
    private int port = 587;
    private String username;
    private String password;
    // getters and setters
}
```

---

## 14. Common Mistakes & Pitfalls

| Pitfall | Fix |
|---------|-----|
| **Circular dependency** (A → B → A) | Redesign or use `@Lazy` on one dependency |
| **`@Transactional` on private method** | Spring AOP can't proxy private — make it public |
| **`LazyInitializationException`** | Use `@Transactional`, JOIN FETCH, or DTO projections |
| **N+1 queries** | Use `LEFT JOIN FETCH` in `@Query` |
| **Returning entity from controller** | Always return DTOs — hides DB structure, avoids infinite recursion |
| **Prototype in singleton** | Use `@Lookup` or `ApplicationContext.getBean()` for fresh instances |

---

## 15. Interview Tips & Quick Revision

### Most Asked Topics

| Topic | Key Points |
|-------|-----------|
| IoC / DI | Container manages beans; prefer constructor injection for immutability |
| Bean Lifecycle | Instantiation → `@PostConstruct` → ready → `@PreDestroy` |
| `@Transactional` | ACID; self-invocation pitfall; `readOnly` optimization; propagation |
| Spring Security | Filter chain; stateless JWT; `SecurityFilterChain` bean |
| JPA / Hibernate | LAZY vs EAGER; N+1; `@Transactional` on service layer |
| AOP | Cross-cutting concerns; `@Around` most powerful; proxy-based |
| Auto-configuration | `@Conditional*`; `AutoConfiguration.imports`; can be excluded |

### Quick Cheat Sheet — Annotations

```
@SpringBootApplication  = @Configuration + @EnableAutoConfiguration + @ComponentScan
@RestController         = @Controller + @ResponseBody
@Transactional          = begin + commit/rollback around method
@Cacheable              = cache result, skip method if cached
@Async                  = run in a separate thread
@Scheduled              = run on a schedule (cron or fixed rate)
@EnableMethodSecurity   = activate @PreAuthorize / @PostAuthorize
@ConfigurationProperties= bind properties file to typed Java class
```

### Architecture Flow (CRUD Request)

```
HTTP Request → DispatcherServlet → SecurityFilterChain
    → @RestController (@Valid, deserialize JSON)
    → @Service (@Transactional business logic)
    → @Repository / JpaRepository (SQL via Hibernate)
    → Database
    → Entity → DTO → JSON Response
```

### Interview One-Liners

- **Why constructor injection?** Explicit dependencies, `final` fields, testable without Spring.
- **What does `@Transactional` do?** Proxy opens a transaction before the method, commits or rolls back after.
- **`@Component` vs `@Service` vs `@Repository`?** Same for scanning; `@Repository` adds exception translation.
- **What is AOP weaving?** Linking aspects to target code at runtime via JDK dynamic proxies or CGLIB.
- **How does auto-configuration work?** Reads `AutoConfiguration.imports`, applies `@Conditional` checks per classpath.
- **What causes `LazyInitializationException`?** Accessing a lazy association after the Hibernate session closed.

---

*Last Updated: 2026-06-18*

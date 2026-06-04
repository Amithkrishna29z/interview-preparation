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

**Real-world analogy:** Spring is like a smart factory that assembles your application. Instead of you manually wiring components together, Spring reads a blueprint (your configuration), creates all the parts, and connects them.

Spring Framework is a lightweight, open-source Java framework that provides:
- **Dependency Injection (DI)** – manages object creation and wiring
- **Aspect-Oriented Programming (AOP)** – cross-cutting concerns (logging, security, transactions)
- **Data Access** – abstractions over JDBC, JPA, Hibernate
- **Web MVC** – building REST APIs and web apps
- **Testing** – first-class support for unit and integration tests

### Spring vs Spring Boot

| Feature | Spring Framework | Spring Boot |
|---------|-----------------|-------------|
| Configuration | Manual XML / Java config | Auto-configured |
| Server | External (deploy WAR) | Embedded Tomcat/Jetty |
| Setup time | High boilerplate | Minimal, production-ready in minutes |
| Starters | None | Curated dependency bundles |
| Actuator | Manual setup | Built-in `/actuator` endpoints |
| Best for | Fine-grained control | Rapid application development |

---

## 2. Inversion of Control (IoC) & Dependency Injection (DI)

**Concept:** Instead of your class creating its own dependencies (`new SomeService()`), Spring creates and injects them. Control of object creation is *inverted* — from your code to the framework.

**Real-world analogy:** You don't assemble your own car engine. The manufacturer (Spring) assembles it and hands it to you ready to use.

### 2.1 Types of Dependency Injection

#### Constructor Injection (Recommended)
```java
@Service
public class OrderService {
    private final PaymentService paymentService;

    // Spring detects single constructor — @Autowired is optional here
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```
**Why preferred:** Dependencies are immutable (`final`), class is testable without Spring, fails fast if dependency is missing.

#### Setter Injection (Optional dependencies)
```java
@Service
public class NotificationService {
    private EmailService emailService;

    @Autowired
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}
```

#### Field Injection (Avoid in production code)
```java
@Service
public class UserService {
    @Autowired  // Works but hides dependencies, hard to test
    private UserRepository userRepository;
}
```

### 2.2 ApplicationContext — The IoC Container

```java
// Spring's IoC container — holds and manages all beans
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
UserService userService = context.getBean(UserService.class);
```

| Container Type | Usage |
|---------------|-------|
| `AnnotationConfigApplicationContext` | Java-based config (`@Configuration`) |
| `ClassPathXmlApplicationContext` | XML-based config (legacy) |
| `SpringApplication.run()` | Spring Boot (most common) |

---

## 3. Spring Bean Lifecycle

**Real-world analogy:** A bean's lifecycle is like an employee's — hired (instantiated), trained (properties set), given an ID badge (initialized), works (in use), then retires (destroyed).

```
1. Instantiation        → Spring creates the bean via constructor
2. Property Injection   → @Autowired / @Value fields are set
3. BeanNameAware        → setBeanName() called (if implemented)
4. BeanFactoryAware     → setBeanFactory() called (if implemented)
5. @PostConstruct       → Your custom init logic runs
6. Bean is READY        → Available for use in the application
7. @PreDestroy          → Your custom cleanup logic runs (on shutdown)
8. Destruction          → Bean removed from context
```

```java
@Component
public class DatabaseConnectionPool {

    @PostConstruct
    public void init() {
        // Runs AFTER all properties are injected
        System.out.println("Initializing connection pool...");
        // Open connections, warm up cache, etc.
    }

    @PreDestroy
    public void cleanup() {
        // Runs BEFORE the bean is destroyed
        System.out.println("Closing all connections...");
    }
}
```

---

## 4. Bean Scopes

| Scope | Instances Created | Lifespan | Use Case |
|-------|------------------|----------|----------|
| `singleton` (default) | One per container | App lifetime | Stateless services, repositories |
| `prototype` | New one per request | Until GC | Stateful beans, heavy objects |
| `request` | One per HTTP request | Single request | Web-layer data holders |
| `session` | One per HTTP session | User session | Shopping cart, user preferences |
| `application` | One per ServletContext | App lifetime | App-wide shared state |

```java
@Component
@Scope("prototype")  // New instance every time it's injected
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();
}

@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    // Lives only for the duration of an HTTP request
}
```

**Common gotcha:** Injecting a `prototype` bean into a `singleton` bean — the prototype only gets created once! Use `@Lookup` or `ApplicationContext.getBean()` to get a fresh instance each time.

---

## 5. Core Annotations

### Stereotype Annotations

| Annotation | Purpose | Typical Layer |
|------------|---------|--------------|
| `@Component` | Generic Spring-managed bean | Any |
| `@Service` | Business logic | Service layer |
| `@Repository` | Data access + exception translation | DAO / Repository layer |
| `@Controller` | Web MVC controller (returns views) | Web layer |
| `@RestController` | `@Controller` + `@ResponseBody` | REST API layer |

### Configuration Annotations

```java
@Configuration          // Marks class as a source of bean definitions
public class AppConfig {

    @Bean               // Method return value registered as a bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}

@ComponentScan("com.example")  // Tells Spring where to look for @Component classes
@PropertySource("classpath:app.properties")  // Load custom property file
```

### Injection Annotations

```java
@Autowired              // Inject by type
@Qualifier("emailNotifier")  // When multiple implementations exist, pick by name
@Primary                // Mark one implementation as the default when multiple exist
@Value("${app.name}")   // Inject value from properties file
@Lazy                   // Initialize this bean only when first requested
```

### Multiple Implementations — Qualifier Example

```java
public interface NotificationService {
    void send(String message);
}

@Service("emailNotifier")
public class EmailNotificationService implements NotificationService { ... }

@Service("smsNotifier")
public class SmsNotificationService implements NotificationService { ... }

@Service
public class AlertService {
    @Autowired
    @Qualifier("emailNotifier")  // Explicitly choose which implementation
    private NotificationService notifier;
}
```

---

## 6. Spring Boot Auto-Configuration

**How it works:** Spring Boot scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` and conditionally activates configurations based on what's on the classpath and what you've already configured.

```java
// The magic annotation that enables everything
@SpringBootApplication
// = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

### Conditional Annotations

```java
@ConditionalOnClass(DataSource.class)     // Only if DataSource is on classpath
@ConditionalOnMissingBean(DataSource.class) // Only if no DataSource bean defined yet
@ConditionalOnProperty(name = "feature.enabled", havingValue = "true")
```

### application.properties Key Settings

```properties
# Server
server.port=8080
server.servlet.context-path=/api

# Database
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=postgres
spring.datasource.password=secret

# JPA
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Logging
logging.level.com.example=DEBUG
logging.level.org.hibernate.SQL=DEBUG

# Actuator
management.endpoints.web.exposure.include=health,info,metrics
```

---

## 7. Spring MVC & REST

### Request Mapping Annotations

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")                    // GET /api/users/1
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @GetMapping                             // GET /api/users?page=0&size=10
    public Page<UserDto> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return userService.findAll(PageRequest.of(page, size));
    }

    @PostMapping                            // POST /api/users
    public ResponseEntity<UserDto> createUser(
            @RequestBody @Valid CreateUserRequest request) {
        UserDto created = userService.create(request);
        URI location = URI.create("/api/users/" + created.getId());
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")                    // PUT /api/users/1
    public ResponseEntity<UserDto> updateUser(
            @PathVariable Long id,
            @RequestBody @Valid UpdateUserRequest request) {
        return ResponseEntity.ok(userService.update(id, request));
    }

    @DeleteMapping("/{id}")                 // DELETE /api/users/1
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Request Parameter Annotations

| Annotation | Example | Description |
|------------|---------|-------------|
| `@PathVariable` | `/users/{id}` | Extracts value from URI path |
| `@RequestParam` | `?page=0&size=10` | Query string parameter |
| `@RequestBody` | POST/PUT body | Deserializes JSON body to object |
| `@RequestHeader` | `Authorization: Bearer ...` | HTTP header value |
| `@CookieValue` | `sessionId=abc` | Cookie value |

### Validation with @Valid

```java
public class CreateUserRequest {
    @NotBlank(message = "Name is required")
    private String name;

    @Email(message = "Invalid email format")
    private String email;

    @Min(value = 18, message = "Must be at least 18")
    private int age;

    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;
}
```

---

## 8. Spring Data JPA

**Real-world analogy:** JPA is like a translator between Java objects and database tables. You define the shape in Java, and JPA handles the SQL.

### Entity Definition

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    @Enumerated(EnumType.STRING)
    private UserRole role;

    @CreatedDate
    private LocalDateTime createdAt;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();
}
```

### Repository Patterns

```java
// Basic CRUD — extend JpaRepository
public interface UserRepository extends JpaRepository<User, Long> {

    // Spring generates query from method name
    Optional<User> findByEmail(String email);
    List<User> findByRoleAndActiveTrue(UserRole role);
    boolean existsByEmail(String email);
    long countByRole(UserRole role);

    // Custom JPQL query
    @Query("SELECT u FROM User u WHERE u.createdAt > :since AND u.role = :role")
    List<User> findRecentByRole(@Param("since") LocalDateTime since,
                                @Param("role") UserRole role);

    // Native SQL query
    @Query(value = "SELECT * FROM users WHERE name ILIKE %:keyword%",
           nativeQuery = true)
    List<User> searchByName(@Param("keyword") String keyword);

    // Pagination
    Page<User> findByRole(UserRole role, Pageable pageable);
}
```

### Relationship Types

```java
// One-to-Many: One User has many Orders
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
}

// Many-to-Many: Users have many Roles, Roles have many Users
@Entity
public class User {
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
}
```

### Fetch Type — Critical Interview Topic

| Type | When loaded | Problem if misused |
|------|------------|-------------------|
| `EAGER` | Immediately with parent | N+1 problem, loads data you may not need |
| `LAZY` (preferred) | Only when accessed | `LazyInitializationException` outside transaction |

**N+1 Problem & Fix:**
```java
// BAD: 1 query for users + N queries for each user's orders
List<User> users = userRepository.findAll();  // N+1 if orders are EAGER

// GOOD: Single JOIN query
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

---

## 9. Spring Security

### Basic Security Configuration (Spring Boot 3.x)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())  // Disable for stateless REST APIs
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }
}
```

### JWT Authentication Filter

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {

        String header = request.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String token = header.substring(7);
        String username = jwtService.extractUsername(token);

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            if (jwtService.isTokenValid(token, userDetails)) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        chain.doFilter(request, response);
    }
}
```

### Method-Level Security

```java
@Configuration
@EnableMethodSecurity  // Required to activate method security
public class MethodSecurityConfig { }

@Service
public class OrderService {
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public List<Order> getOrdersForUser(Long userId) { ... }

    @PostAuthorize("returnObject.ownerId == authentication.principal.id")
    public Order getOrder(Long orderId) { ... }

    @Secured("ROLE_ADMIN")
    public void deleteOrder(Long orderId) { ... }
}
```

---

## 10. Spring AOP

**Real-world analogy:** AOP is like airport security — it intercepts passengers (method calls) at checkpoints (join points) without changing their ticket (the method itself). You add cross-cutting concerns (logging, security, auditing) in one place.

### Key Concepts

| Term | Meaning | Example |
|------|---------|---------|
| **Aspect** | Module that encapsulates cross-cutting concern | `LoggingAspect` |
| **Join Point** | Point in program execution | Method call/execution |
| **Advice** | Action taken at a join point | Log before/after method |
| **Pointcut** | Expression that matches join points | `execution(* com.example.service.*.*(..))` |
| **Weaving** | Linking aspects with target objects | Spring does this at runtime |

### Advice Types

```java
@Aspect
@Component
public class LoggingAspect {

    // Runs BEFORE the method
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("Calling: " + joinPoint.getSignature().getName());
    }

    // Runs AFTER method returns (not on exception)
    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))",
                    returning = "result")
    public void logAfter(JoinPoint joinPoint, Object result) {
        System.out.println("Returned: " + result);
    }

    // Runs if method throws an exception
    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))",
                   throwing = "ex")
    public void logException(JoinPoint joinPoint, Exception ex) {
        System.out.println("Exception in: " + joinPoint.getSignature().getName());
    }

    // Wraps entire method — most powerful, use sparingly
    @Around("@annotation(com.example.annotation.Timed)")
    public Object measureTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = joinPoint.proceed();  // Actually calls the method
        long elapsed = System.currentTimeMillis() - start;
        System.out.println(joinPoint.getSignature() + " took " + elapsed + "ms");
        return result;
    }
}
```

---

## 11. Spring Transactions

**Real-world analogy:** A transaction is like buying tickets at a counter — either you pay AND get all tickets, or neither happens. No partial success.

### @Transactional Deep Dive

```java
@Service
public class TransferService {

    // Default: REQUIRED — joins existing transaction or creates new one
    @Transactional
    public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
        accountRepository.debit(fromId, amount);   // If this fails...
        accountRepository.credit(toId, amount);    // ...this is also rolled back
    }

    // Read-only optimization — no dirty checking, better performance
    @Transactional(readOnly = true)
    public List<Account> getAllAccounts() {
        return accountRepository.findAll();
    }

    // Custom rollback — only roll back on specific exceptions
    @Transactional(rollbackFor = InsufficientFundsException.class)
    public void withdraw(Long accountId, BigDecimal amount) throws InsufficientFundsException {
        // ...
    }
}
```

### Propagation Levels

| Propagation | Behavior | When to use |
|-------------|----------|-------------|
| `REQUIRED` (default) | Join existing or create new | Standard operations |
| `REQUIRES_NEW` | Always creates new (suspends existing) | Audit logging — must save even if outer tx fails |
| `NESTED` | Savepoint within existing tx | Partial rollback inside a larger transaction |
| `SUPPORTS` | Joins if exists, no tx if not | Read operations that can work with or without tx |
| `NOT_SUPPORTED` | Suspends existing tx | Operations that must run without a transaction |
| `MANDATORY` | Must join existing, fails if none | Operations that always need a transaction caller |
| `NEVER` | Fails if a transaction exists | Operations that must never run in a transaction |

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAuditLog(String action) {
    // Saved in its own transaction — persists even if outer tx rolls back
    auditRepository.save(new AuditLog(action));
}
```

**Common Pitfall — Self-invocation doesn't go through the proxy:**
```java
@Service
public class OrderService {
    public void processOrder(Order order) {
        // BAD: @Transactional on applyDiscount() is IGNORED here
        // because 'this' bypasses the Spring AOP proxy
        applyDiscount(order);
    }

    @Transactional
    public void applyDiscount(Order order) { ... }
}
```

---

## 12. Exception Handling

### Global Exception Handler

```java
@RestControllerAdvice  // Applies to all @RestController classes
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.toList());
        return ResponseEntity
            .badRequest()
            .body(new ErrorResponse("VALIDATION_FAILED", errors.toString()));
    }

    @ExceptionHandler(Exception.class)  // Catch-all fallback
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity
            .internalServerError()
            .body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred"));
    }
}

public record ErrorResponse(String code, String message) {}
```

---

## 13. Configuration & Profiles

### Environment Profiles

```java
// Activate a profile
// In application.properties: spring.profiles.active=dev
// As JVM arg: -Dspring.profiles.active=prod
// As env var: SPRING_PROFILES_ACTIVE=prod

@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.H2).build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        // Real production database
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(jdbcUrl);
        return new HikariDataSource(config);
    }
}
```

### Externalized Configuration with @ConfigurationProperties

```java
// Better than scattered @Value annotations
@ConfigurationProperties(prefix = "app.email")
@Component
public class EmailProperties {
    private String host;
    private int port = 587;
    private String username;
    private String password;
    private boolean ssl = true;
    // getters and setters
}

// application.yml
// app:
//   email:
//     host: smtp.gmail.com
//     port: 587
//     username: myapp@gmail.com
//     password: secret
```

---

## 14. Common Mistakes & Pitfalls

### 1. Circular Dependency
```java
// ClassA depends on ClassB, ClassB depends on ClassA → Spring fails to start
// Fix: Redesign, or use @Lazy on one of the dependencies
@Service
public class ServiceA {
    public ServiceA(@Lazy ServiceB serviceB) { ... }
}
```

### 2. Transactional on Private Methods
```java
// @Transactional has NO EFFECT on private methods — Spring AOP can't proxy them
@Transactional  // IGNORED!
private void doSomethingInternal() { }

// Fix: make it public, or move to a different Spring bean
```

### 3. LazyInitializationException
```java
// FAILS: Accessing a LAZY collection outside a transaction
User user = userRepository.findById(1L).get();
// Transaction ends here after findById returns
user.getOrders().size();  // LazyInitializationException!

// Fix 1: Use @Transactional on the calling method
// Fix 2: Use JOIN FETCH in the query
// Fix 3: Use DTO projections instead of entities in the web layer
```

### 4. N+1 Query Problem
```java
// Loads 100 users, then fires 100 more queries for each user's orders
List<User> users = userRepository.findAll();
users.forEach(u -> System.out.println(u.getOrders().size())); // 100 extra queries!

// Fix: Fetch join
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

### 5. Exposing Entity Directly from Controller
```java
// BAD: Exposes internal DB structure, risk of infinite recursion with bidirectional relationships
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();  // Don't do this
}

// GOOD: Always use DTOs in the API layer
@GetMapping("/users/{id}")
public UserDto getUser(@PathVariable Long id) {
    return userMapper.toDto(userRepository.findById(id).orElseThrow());
}
```

---

## 15. Interview Tips & Quick Revision

### Most Asked Topics

| Topic | Key Points to Mention |
|-------|----------------------|
| IoC / DI | Container manages beans; prefer constructor injection for immutability |
| Bean Lifecycle | `@PostConstruct` → ready → `@PreDestroy`; scopes: singleton vs prototype |
| `@Transactional` | ACID guarantees; propagation; self-invocation pitfall; readOnly optimization |
| Spring Security | Filter chain; `SecurityFilterChain` bean; stateless JWT pattern |
| JPA / Hibernate | LAZY vs EAGER; N+1 problem; `@Transactional` on service layer |
| AOP | Cross-cutting concerns; `@Around` is most powerful; proxy-based |
| Auto-configuration | `@Conditional*`; `spring.factories`; can be excluded |

### Quick Cheat Sheet — Annotations

```
@SpringBootApplication  = @Configuration + @EnableAutoConfiguration + @ComponentScan
@RestController         = @Controller + @ResponseBody
@Transactional          = begin + commit/rollback transaction around method
@Cacheable              = cache method result, skip execution if cached
@Async                  = run method in a separate thread
@Scheduled              = run method on a schedule (cron or fixed rate)
@EnableMethodSecurity   = activate @PreAuthorize / @PostAuthorize
@ConfigurationProperties= bind properties file to a typed Java class
```

### Architecture Flow (CRUD Request)

```
HTTP Request
    ↓
DispatcherServlet (Front Controller)
    ↓
SecurityFilterChain (Authentication / Authorization)
    ↓
@RestController (Deserialize JSON → @Valid → call service)
    ↓
@Service (Business logic, @Transactional boundary)
    ↓
@Repository / JpaRepository (Query DB via JPA/Hibernate)
    ↓
Database
    ↓
Return: Entity → DTO → JSON Response
```

### Interview One-Liners

- **Why constructor injection?** Makes dependencies explicit, supports immutability (`final`), and makes the class testable without a Spring context.
- **What does `@Transactional` do exactly?** Spring wraps the method in a proxy that opens a transaction before the call and commits (or rolls back on exception) after.
- **Difference between `@Component`, `@Service`, `@Repository`?** Functionally equivalent for component scanning, but `@Repository` adds persistence exception translation and the others carry semantic meaning for developers.
- **What is AOP weaving?** The process of linking aspects with the application code. Spring uses runtime weaving via JDK dynamic proxies (interface-based) or CGLIB proxies (class-based).
- **How does Spring Boot auto-configure?** It reads `AutoConfiguration.imports`, then applies `@Conditional` checks to activate only relevant configurations based on classpath and existing beans.
- **What causes `LazyInitializationException`?** Accessing a lazy-loaded association after the Hibernate session (transaction) has closed. Fix with `@Transactional`, JOIN FETCH, or DTOs.

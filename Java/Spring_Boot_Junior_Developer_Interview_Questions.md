# Spring Boot Junior Developer Interview Questions & Answers

## Table of Contents
1. [Basic Concepts](#basic-concepts)
2. [Core Annotations](#core-annotations)
3. [Configuration & Properties](#configuration--properties)
4. [Dependency Injection](#dependency-injection)
5. [REST API Development](#rest-api-development)
6. [Data Access & JPA](#data-access--jpa)
7. [Testing](#testing)
8. [Spring Boot Features](#spring-boot-features)
9. [Error Handling](#error-handling)
10. [Security Basics](#security-basics)
11. [Spring vs Spring Boot](#spring-vs-spring-boot)
12. [Common Scenarios](#common-scenarios)

---

## Basic Concepts

### Q1: What is Spring Boot and why is it used?

**Answer:** Spring Boot is a Java framework built on Spring that creates stand-alone, production-grade applications with minimal setup. Key benefits:
- **Auto-configuration**: Configures Spring based on classpath dependencies
- **Embedded servers**: Includes Tomcat/Jetty — no WAR deployment needed
- **Starter dependencies**: Curated dependency bundles (e.g., `spring-boot-starter-web`)
- **Production-ready**: Built-in health checks and metrics via Actuator

### Q2: What are the key features of Spring Boot?

**Answer:**
- Auto-configuration, starter dependencies, embedded servers
- Spring Boot Actuator for monitoring
- No XML configuration — Java-based config only
- Spring Initializr for project generation

### Q3: What is a Spring Boot Starter?

**Answer:** A starter is a dependency descriptor that bundles related dependencies for a feature. Examples:
- `spring-boot-starter-web` — REST APIs
- `spring-boot-starter-data-jpa` — JPA + Hibernate
- `spring-boot-starter-security` — Spring Security
- `spring-boot-starter-test` — testing support

### Q4: What is the @SpringBootApplication annotation?

**Answer:** A composite annotation equivalent to `@Configuration + @EnableAutoConfiguration + @ComponentScan`. It marks the main class, enables auto-config, and triggers component scanning.

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

### Q5: What is Spring Boot Actuator?

**Answer:** Actuator adds production-ready monitoring endpoints to your app:
- `/actuator/health` — application health
- `/actuator/metrics` — application metrics
- `/actuator/env` — environment properties
- `/actuator/beans` — all registered beans

---

## Core Annotations

### Q6: What are the main Spring Boot annotations?

**Answer:**

| Category | Annotations |
|----------|-------------|
| Core | `@SpringBootApplication`, `@Configuration`, `@Bean`, `@ComponentScan` |
| Stereotype | `@Component`, `@Service`, `@Repository`, `@Controller` |
| REST | `@RestController`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping` |
| Request binding | `@PathVariable`, `@RequestParam`, `@RequestBody` |
| JPA | `@Entity`, `@Id`, `@GeneratedValue`, `@Column`, `@Table` |

### Q7: What is the difference between @Component, @Service, @Repository, and @Controller?

**Answer:** All four register a class as a Spring bean. The differences are semantic and layer-specific:
- `@Repository` additionally provides automatic exception translation (SQLExceptions → Spring's `DataAccessException`)
- `@Service` signals business logic; `@Controller` signals the presentation layer
- `@Component` is the generic fallback for anything that doesn't fit the others

### Q8: What is the difference between @Controller and @RestController?

**Answer:** `@RestController = @Controller + @ResponseBody`. A `@Controller` returns view names by default and requires `@ResponseBody` to return JSON. `@RestController` always serializes the return value to JSON/XML — use it for REST APIs.

```java
// @Controller needs @ResponseBody for JSON
@Controller
public class MyController {
    @GetMapping("/api/data")
    @ResponseBody
    public Data getData() { return data; }
}

// @RestController — JSON by default
@RestController
public class MyRestController {
    @GetMapping("/api/data")
    public Data getData() { return data; }
}
```

### Q9: What are @Autowired and @Qualifier?

**Answer:** `@Autowired` injects a dependency automatically. When multiple beans of the same type exist, `@Qualifier` specifies which one to use. Constructor injection is preferred.

```java
@Service
public class PaymentService {
    private final PaymentProcessor processor;

    @Autowired
    public PaymentService(@Qualifier("paypalProcessor") PaymentProcessor processor) {
        this.processor = processor;
    }
}
```

### Q10: What is @Value annotation?

**Answer:** Injects a single value from properties files, env vars, or SpEL expressions.

```java
@Component
public class AppConfig {
    @Value("${app.name}")
    private String appName;

    @Value("${app.version:1.0}")  // default if missing
    private String appVersion;
}
```

---

## Configuration & Properties

### Q11: How do you configure external properties in Spring Boot?

**Answer:** Four main ways:

```properties
# 1. application.properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
```

```yaml
# 2. application.yml
server:
  port: 8080
```

```java
// 3. @Value for individual values
@Value("${server.port}")
private int serverPort;

// 4. @ConfigurationProperties for grouped properties
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private int port;
    // getters and setters
}
```

```bash
# 5. Command-line override
java -jar app.jar --server.port=9090
```

### Q12: What is @ConfigurationProperties and how is it different from @Value?

**Answer:**

| | `@ConfigurationProperties` | `@Value` |
|---|---|---|
| Scope | Group of related properties | Single value |
| Nested properties | Yes | No |
| JSR-303 validation | Yes | No |
| SpEL support | No | Yes |
| Best for | Complex configs | Simple/individual values |

### Q13: What are Spring Boot profiles?

**Answer:** Profiles let you define environment-specific configuration (dev, test, prod).

```
application-dev.yml   # dev config
application-prod.yml  # prod config
```

Activate via property or command line:
```properties
spring.profiles.active=dev
```
```bash
java -jar app.jar --spring.profiles.active=prod
```

Use `@Profile` to conditionally register beans:
```java
@Profile("dev")
@Service
public class DevService { }
```

### Q14: What is the difference between application.properties and application.yml?

**Answer:** Both configure the same properties — choose one and stick to it. `.properties` uses flat key=value pairs; `.yml` uses indented hierarchy and is easier to read for nested configs. They are equivalent in Spring Boot.

---

## Dependency Injection

### Q15: What is Dependency Injection?

**Answer:** DI is a pattern where Spring creates and injects objects for you instead of you creating them with `new`. This gives loose coupling and makes testing easier. Three injection styles:

```java
// 1. Constructor (recommended — immutable, testable)
@Service
public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; }
}

// 2. Setter
@Autowired
public void setRepo(UserRepository repo) { this.repo = repo; }

// 3. Field (not recommended — hard to test)
@Autowired
private UserRepository repo;
```

### Q16: What is the difference between @Component and @Bean?

**Answer:** `@Component` is a class-level annotation — Spring detects and registers it automatically via component scan. `@Bean` is a method-level annotation in a `@Configuration` class — you manually control how the bean is built (useful for third-party classes you can't annotate).

```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create().url("jdbc:mysql://...").build();
    }
}
```

### Q17: What are the scopes of a Spring bean?

**Answer:**

| Scope | Instances | Notes |
|-------|-----------|-------|
| `singleton` (default) | One per container | Shared everywhere |
| `prototype` | New per request | Spring doesn't manage lifecycle after creation |
| `request` | One per HTTP request | Web apps only |
| `session` | One per HTTP session | Web apps only |

---

## REST API Development

### Q18: How do you create a RESTful API in Spring Boot?

**Answer:**

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping
    public ResponseEntity<List<User>> getAll() {
        return ResponseEntity.ok(userService.findAll());
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> getById(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<User> create(@Valid @RequestBody User user) {
        return ResponseEntity.status(HttpStatus.CREATED).body(userService.save(user));
    }

    @PutMapping("/{id}")
    public ResponseEntity<User> update(@PathVariable Long id, @Valid @RequestBody User user) {
        return ResponseEntity.ok(userService.update(id, user));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Q19: What is the difference between @PathVariable and @RequestParam?

**Answer:** `@PathVariable` extracts a value from the URI path (`/users/{id}`). `@RequestParam` extracts a value from the query string (`/users?page=1`).

```java
@GetMapping("/users/{id}")
public User getById(@PathVariable Long id) { ... }   // /api/users/123

@GetMapping("/users")
public List<User> getAll(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size) { ... }  // /api/users?page=1&size=10
```

### Q20: What is the difference between @RequestBody and @RequestParam?

**Answer:** `@RequestBody` deserializes the HTTP request body (JSON) into a Java object — used for POST/PUT with complex data. `@RequestParam` binds simple query/form values to method parameters.

### Q21: How do you validate request data in Spring Boot?

**Answer:** Add `spring-boot-starter-validation`, annotate your DTO, and use `@Valid` in the controller.

```java
public class UserDTO {
    @NotNull @Size(min = 2, max = 50)
    private String name;

    @NotBlank @Email
    private String email;

    @Min(18)
    private int age;
}

@PostMapping("/users")
public ResponseEntity<User> create(@Valid @RequestBody UserDTO dto) {
    return ResponseEntity.ok(userService.createUser(dto));
}
```

Handle validation errors with a `@ControllerAdvice` catching `MethodArgumentNotValidException` (see Error Handling section).

---

## Data Access & JPA

### Q22: What is Spring Data JPA?

**Answer:** Spring Data JPA simplifies database access by providing repository interfaces with built-in CRUD, query derivation from method names, pagination, and `@Query` for custom JPQL/SQL.

### Q23: What are the common Spring Data JPA repositories?

**Answer:** Use `JpaRepository` — it extends `CrudRepository` and `PagingAndSortingRepository`, giving you CRUD, pagination, sorting, and JPA-specific methods.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmail(String email);
    Page<User> findByLastName(String lastName, Pageable pageable);

    @Query("SELECT u FROM User u WHERE u.email = :email")
    User findByEmailCustom(@Param("email") String email);
}
```

### Q24: What are the common JPA annotations?

**Answer:**

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders;

    @ManyToMany
    @JoinTable(name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id"))
    private Set<Role> roles;
}
```

Key annotations: `@Entity`, `@Table`, `@Id`, `@GeneratedValue`, `@Column`, `@Transient`, `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`.

### Q25: What is @Transactional annotation?

**Answer:** Wraps a method in a database transaction (ACID). If any exception is thrown, all changes in that method are rolled back automatically.

```java
@Service
public class UserService {
    @Transactional
    public void createUserWithOrders(User user, List<Order> orders) {
        userRepository.save(user);
        orders.forEach(o -> { o.setUser(user); orderRepository.save(o); });
        // Any exception here rolls back both saves
    }
}
```

### Q26: What are the different types of Entity relationships?

**Answer:**

| Annotation | Example |
|------------|---------|
| `@OneToOne` | User ↔ UserProfile |
| `@OneToMany` / `@ManyToOne` | Department → Employees |
| `@ManyToMany` | Students ↔ Courses (needs `@JoinTable`) |

### Q27: What is the difference between lazy and eager loading?

**Answer:** **Lazy** (default for collections) loads related entities only when you access them — better performance. **Eager** loads them immediately with the parent — can cause N+1 problems with large datasets.

```java
@OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
private List<Order> orders;  // loaded only on getOrders()
```

---

## Testing

### Q28: How do you test a Spring Boot application?

**Answer:** Three main testing slices:

```java
// Integration test — full context
@SpringBootTest
class UserServiceTest {
    @Autowired private UserService userService;
    @MockBean  private UserRepository userRepository;

    @Test
    void testFindById() {
        when(userRepository.findById(1L)).thenReturn(Optional.of(new User("John", "john@example.com")));
        assertEquals("John", userService.findById(1L).getName());
    }
}

// Web layer only
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired private MockMvc mockMvc;
    @MockBean  private UserService userService;

    @Test
    void testGetUser() throws Exception {
        when(userService.findById(1L)).thenReturn(new User("John"));
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}

// Data layer only (uses in-memory DB)
@DataJpaTest
class UserRepositoryTest {
    @Autowired private TestEntityManager em;
    @Autowired private UserRepository repo;

    @Test
    void testFindByEmail() {
        em.persist(new User("John", "john@example.com"));
        assertEquals("John", repo.findByEmail("john@example.com").getName());
    }
}
```

### Q29: What is @MockBean?

**Answer:** Creates a Mockito mock and registers it as a Spring bean, replacing the real bean in the application context. Use it when testing one component in isolation from its dependencies.

### Q30: What is the difference between @SpringBootTest and @WebMvcTest?

**Answer:**

| | `@SpringBootTest` | `@WebMvcTest` |
|---|---|---|
| Context loaded | Full application | Web layer only |
| Speed | Slower | Faster |
| Use case | Integration tests | Controller unit tests |
| Dependencies | Real or mocked | Must mock services |

---

## Spring Boot Features

### Q31: What is Spring Boot DevTools?

**Answer:** A development-only dependency that enables automatic restart on file changes, LiveReload for browsers, and disables template caching. Add with `<optional>true</optional>` so it's excluded from production builds.

### Q32: What is a CommandLineRunner?

**Answer:** An interface whose `run()` method executes once after the application context is fully loaded. Useful for seeding data or running startup logic.

```java
@Component
@Order(1)
public class DataInitializer implements CommandLineRunner {
    @Autowired private UserRepository repo;

    @Override
    public void run(String... args) {
        if (repo.count() == 0) repo.save(new User("admin", "admin@example.com"));
    }
}
```

### Q33: How does Spring Boot auto-configuration work?

**Answer:** At startup, Spring Boot reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`, then uses conditional annotations to register beans only when conditions are met.

Key conditions:
- `@ConditionalOnClass` — class is on the classpath
- `@ConditionalOnMissingBean` — no existing bean of that type
- `@ConditionalOnProperty` — a property has a specific value

This is why adding `spring-boot-starter-data-jpa` to your POM automatically configures a `DataSource`, `EntityManagerFactory`, and transaction manager.

### Q34: What is Spring Boot Starter Parent?

**Answer:** A Maven parent POM that manages versions for all Spring Boot dependencies. Add it once and omit version tags for any managed dependency.

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>
```

---

## Error Handling

### Q35: How do you handle exceptions in Spring Boot?

**Answer:** Two main approaches:

```java
// Global handler (preferred)
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("USER_NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
            .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
        return ResponseEntity.badRequest().body(errors);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("INTERNAL_ERROR", "An error occurred"));
    }
}

// Annotate custom exceptions with a status
@ResponseStatus(HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}
```

### Q36: What is the difference between @ExceptionHandler and @ControllerAdvice?

**Answer:** `@ExceptionHandler` on a method inside a controller handles exceptions only from that controller. `@ControllerAdvice` is a class-level annotation that applies exception handlers globally across all controllers — always prefer `@ControllerAdvice` for centralised error handling.

---

## Security Basics

### Q37: How do you secure a Spring Boot application?

**Answer:** Add `spring-boot-starter-security` and define a `SecurityFilterChain` bean.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults());
        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        return new InMemoryUserDetailsManager(
            User.builder().username("user").password("{noop}password").roles("USER").build()
        );
    }
}
```

### Q38: What is JWT authentication in Spring Boot?

**Answer:** JWT (JSON Web Token) is stateless authentication for REST APIs — the token carries user info and is signed, so no server-side session is needed. A `JwtService` generates and validates tokens using a secret key.

```java
@Service
public class JwtService {
    private String secretKey = "your-secret-key";

    public String generateToken(UserDetails user) {
        return Jwts.builder()
            .subject(user.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + 86400000))
            .signWith(Keys.hmacShaKeyFor(secretKey.getBytes()))
            .compact();
    }

    public boolean validateToken(String token, UserDetails user) {
        return extractUsername(token).equals(user.getUsername()) && !isTokenExpired(token);
    }
}
```

---

## Spring vs Spring Boot

### Q39: What is the difference between Spring and Spring Boot?

**Answer:**

| Aspect | Spring Framework | Spring Boot |
|--------|-----------------|-------------|
| Configuration | Extensive XML or Java config | Auto-configuration with defaults |
| Dependencies | Manual management | Starter bundles |
| Server | External Tomcat/Jetty required | Embedded server included |
| Deployment | WAR file | Executable JAR |
| Learning curve | Steeper | Easier |
| Best for | Complex enterprise, fine-grained control | Microservices, modern apps, rapid dev |

### Q40: Can Spring Boot coexist with traditional Spring applications?

**Answer:** Yes. You can migrate incrementally — mix traditional XML/Java config with Spring Boot auto-configuration and adopt Boot features gradually.

---

## Common Scenarios

### Q41: How do you implement pagination in Spring Boot?

**Answer:**

```java
// Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findAll(Pageable pageable);
}

// Controller
@GetMapping
public Page<User> getUsers(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size,
    @RequestParam(defaultValue = "name") String sortBy) {

    return userRepository.findAll(PageRequest.of(page, size, Sort.by(sortBy)));
}
```

### Q42: How do you implement logging in Spring Boot?

**Answer:** Spring Boot uses SLF4J + Logback by default. Use `LoggerFactory` to get a logger.

```java
@Service
public class UserService {
    private static final Logger log = LoggerFactory.getLogger(UserService.class);

    public User createUser(User user) {
        log.info("Creating user: {}", user.getName());
        return userRepository.save(user);
    }
}
```

Configure log levels in `application.properties`:
```properties
logging.level.root=INFO
logging.level.com.example.demo=DEBUG
logging.file.name=application.log
```

---

## Quick Reference Summary

### Essential Annotations
- `@SpringBootApplication` — main class
- `@RestController` — REST API controller
- `@Autowired` — dependency injection
- `@Service`, `@Repository`, `@Component` — stereotypes
- `@Entity`, `@Id`, `@GeneratedValue` — JPA mapping
- `@Transactional` — transaction management
- `@Valid` — request validation
- `@ControllerAdvice` — global exception handling

### Common Starters
- `spring-boot-starter-web` — REST APIs
- `spring-boot-starter-data-jpa` — database
- `spring-boot-starter-security` — security
- `spring-boot-starter-test` — testing
- `spring-boot-starter-validation` — validation

### Configuration Files
- `application.properties` / `application.yml` — main config
- `application-{profile}.properties` — profile-specific config

### Testing Annotations
- `@SpringBootTest` — full integration tests
- `@WebMvcTest` — web layer tests
- `@DataJpaTest` — data layer tests
- `@MockBean` — mock a Spring bean

---

**Tips for Interview Success:**
1. Understand concepts, don't memorize — explain the *why*
2. Use code examples to back up your answers
3. Discuss trade-offs (e.g., lazy vs eager, constructor vs field injection)
4. Mention best practices (constructor injection, `@ControllerAdvice`, profiles)
5. Draw on hands-on experience when possible

**Last Updated:** 2026-06-18

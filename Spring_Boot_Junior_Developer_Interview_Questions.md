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

**Answer:** Spring Boot is an open-source Java-based framework built on top of the Spring Framework that makes it easy to create stand-alone, production-grade Spring applications. It's used because:

- **Simplifies Spring development**: Reduces boilerplate code and configuration
- **Auto-configuration**: Automatically configures Spring applications based on dependencies
- **Embedded servers**: Includes Tomcat, Jetty, or Undertow - no need to deploy WAR files
- **Starter dependencies**: Provides curated dependency bundles for common functionality
- **Production-ready**: Includes monitoring, metrics, and health checks out of the box
- **Faster development**: "Convention over configuration" approach speeds up development

### Q2: What are the key features of Spring Boot?

**Answer:**
- **Auto-configuration**: Automatic configuration based on classpath dependencies
- **Starter dependencies**: Simplified dependency management (e.g., spring-boot-starter-web)
- **Embedded servers**: Built-in Tomcat, Jetty, or Undertow
- **Spring Boot Actuator**: Production-ready features for monitoring and management
- **No XML configuration**: Uses Java-based configuration
- **Spring Boot CLI**: Command-line interface for rapid development
- **Spring Initializr**: Web-based tool for generating Spring Boot projects

### Q3: What is a Spring Boot Starter?

**Answer:** A Spring Boot Starter is a convenient dependency descriptor that aggregates a set of common dependencies for a specific functionality. Examples include:

- `spring-boot-starter-web`: For building web applications with RESTful support
- `spring-boot-starter-data-jpa`: For Spring Data JPA with Hibernate
- `spring-boot-starter-security`: For Spring Security integration
- `spring-boot-starter-test`: For testing Spring Boot applications
- `spring-boot-starter-validation`: For Java Bean Validation (JSR-380)

Starters eliminate the need to manually add and configure individual dependencies.

### Q4: What is the @SpringBootApplication annotation?

**Answer:** `@SpringBootApplication` is a composite annotation that combines three important annotations:

```java
@SpringBootApplication
// Equivalent to:
@Configuration + @EnableAutoConfiguration + @ComponentScan
```

- **@Configuration**: Indicates that the class can be used by the Spring IoC container as a source of bean definitions
- **@EnableAutoConfiguration**: Enables Spring Boot's auto-configuration mechanism
- **@ComponentScan**: Enables component scanning so that @Component, @Service, @Controller, etc. are automatically registered

### Q5: What is Spring Boot Actuator?

**Answer:** Spring Boot Actuator provides production-ready features that help you monitor and manage your application:

- **Health checks**: `/actuator/health` endpoint shows application health
- **Metrics**: `/actuator/metrics` provides application metrics
- **Environment info**: `/actuator/env` shows environment properties
- **HTTP tracing**: `/actuator/httptrace` shows HTTP request/response traces
- **Bean information**: `/actuator/beans` shows all registered beans
- **Custom endpoints**: You can create custom actuator endpoints

---

## Core Annotations

### Q6: What are the main Spring Boot annotations?

**Answer:**

**Core Annotations:**
- `@SpringBootApplication`: Marks main application class
- `@Configuration`: Indicates configuration class
- `@Bean`: Declares a bean
- `@ComponentScan`: Enables component scanning

**Stereotype Annotations:**
- `@Component`: Generic stereotype for any Spring-managed component
- `@Service`: Indicates service layer component
- `@Repository`: Indicates data access layer (DAO) component
- `@Controller`: Indicates presentation layer component

**REST API Annotations:**
- `@RestController`: Combines @Controller and @ResponseBody
- `@RequestMapping`: Maps HTTP requests to handler methods
- `@GetMapping, @PostMapping, @PutMapping, @DeleteMapping, @PatchMapping`: HTTP method-specific mappings
- `@PathVariable`: Binds URI template variables to method parameters
- `@RequestParam`: Binds request parameters to method parameters
- `@RequestBody`: Binds HTTP request body to method parameters
- `@ResponseBody`: Indicates return value should be written to HTTP response body

**Data Access Annotations:**
- `@Entity`: Maps a class to a database table
- `@Table`: Specifies the table name
- `@Id`: Specifies the primary key
- `@GeneratedValue`: Specifies generation strategy for primary key
- `@Column`: Specifies column details

### Q7: What is the difference between @Component, @Service, @Repository, and @Controller?

**Answer:** All are stereotype annotations used to mark classes as Spring beans:

- **@Component**: Generic stereotype for any Spring-managed component
- **@Service**: Specialized for service layer - indicates business logic
- **@Repository**: Specialized for data access layer - provides automatic exception translation
- **@Controller**: Specialized for presentation layer - typically returns views

**Key differences:**
- `@Repository` adds automatic exception translation (converts SQLExceptions to Spring's DataAccessException)
- All others are semantically the same but help with code organization and readability
- Used with `@ComponentScan` to automatically detect and register beans

### Q8: What is the difference between @Controller and @RestController?

**Answer:**

**@Controller:**
- Traditional Spring MVC controller
- Returns views (HTML pages) by default
- Requires `@ResponseBody` annotation to return JSON/XML
- Used for traditional web applications with server-side rendering

**@RestController:**
- Combination of @Controller and @ResponseBody
- Always returns data (JSON/XML) directly
- Used for RESTful APIs
- More concise for API development

```java
// @Controller example
@Controller
public class MyController {
    @GetMapping("/hello")
    public String hello() {
        return "hello"; // Returns view name
    }

    @GetMapping("/api/data")
    @ResponseBody
    public Data getData() {
        return data; // Returns JSON
    }
}

// @RestController example
@RestController
public class MyRestController {
    @GetMapping("/api/data")
    public Data getData() {
        return data; // Returns JSON automatically
    }
}
```

### Q9: What are @Autowired and @Qualifier?

**Answer:**

**@Autowired:**
- Automatically injects dependencies
- Can be used on constructors, fields, or setter methods
- Constructor injection is recommended

**@Qualifier:**
- Used when there are multiple beans of the same type
- Specifies which bean to inject

```java
@Service
public class UserService {
    private final UserRepository userRepository;

    @Autowired  // Constructor injection (recommended)
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}

// When multiple implementations exist
@Service
public class PaymentService {
    private final PaymentProcessor paymentProcessor;

    @Autowired
    public PaymentService(@Qualifier("paypalProcessor") PaymentProcessor processor) {
        this.paymentProcessor = processor;
    }
}
```

### Q10: What is @Value annotation?

**Answer:** `@Value` is used to inject values from property files, environment variables, or system properties into Spring beans:

```java
@Component
public class AppConfig {
    @Value("${app.name}")
    private String appName;

    @Value("${app.version:1.0}")  // Default value if property not found
    private String appVersion;

    @Value("#{systemProperties['user.home']}")
    private String userHome;

    @Value("${app.max.connections:10}")
    private int maxConnections;
}
```

---

## Configuration & Properties

### Q11: How do you configure external properties in Spring Boot?

**Answer:** Spring Boot provides multiple ways to configure external properties:

**1. application.properties / application.yml files:**
```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
```

```yaml
# application.yml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
```

**2. @Value annotation:**
```java
@Value("${server.port}")
private int serverPort;
```

**3. @ConfigurationProperties (Type-safe binding):**
```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private int port;
    // getters and setters
}
```

**4. Environment variables:**
- Spring Boot automatically maps environment variables to properties
- Format: SPRING_DATASOURCE_URL for spring.datasource.url

**5. Command-line arguments:**
```bash
java -jar app.jar --server.port=9090
```

### Q12: What is @ConfigurationProperties and how is it different from @Value?

**Answer:**

**@ConfigurationProperties:**
- Type-safe binding of properties to a POJO
- Supports nested properties
- Supports validation with JSR-303 annotations
- Better for groups of related properties
- More maintainable for complex configurations

**@Value:**
- SpEL (Spring Expression Language) support
- Better for simple, individual properties
- More flexible for expressions
- Simpler for single values

```java
// @ConfigurationProperties example
@ConfigurationProperties(prefix = "database")
public class DatabaseProperties {
    private String url;
    private String username;
    private String password;
    private int poolSize;
    // getters and setters
}

// @Value example
@Component
public class SimpleConfig {
    @Value("${app.name}")
    private String appName;
}
```

### Q13: What are Spring Boot profiles?

**Answer:** Profiles allow you to define different configurations for different environments (dev, test, prod):

**1. Create profile-specific configuration files:**
```
application-dev.yml    # Development environment
application-test.yml   # Testing environment
application-prod.yml   # Production environment
```

**2. Activate profiles:**
```properties
# application.properties
spring.profiles.active=dev
```

**3. Or use command-line:**
```bash
java -jar app.jar --spring.profiles.active=prod
```

**4. Or use @Profile annotation:**
```java
@Profile("dev")
@Service
public class DevService {
    // Development-specific service
}

@Profile("prod")
@Service
public class ProdService {
    // Production-specific service
}
```

### Q14: What is the difference between application.properties and application.yml?

**Answer:**

**application.properties:**
- Traditional key-value format
- Simpler for beginners
- More common in legacy projects
- Example: `server.port=8080`

**application.yml:**
- YAML format with hierarchy
- More readable for complex configurations
- Supports lists and maps better
- Example:
```yaml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/db
    username: root
    password: password
```

Both files are supported and can be used interchangeably, but only one should be used to avoid confusion.

---

## Dependency Injection

### Q15: What is Dependency Injection?

**Answer:** Dependency Injection (DI) is a design pattern where dependencies are provided to objects rather than created inside them. In Spring Boot:

- **IoC Container**: Spring manages object creation and lifecycle
- **Dependency Injection**: Spring injects dependencies into beans
- **Benefits**: Loose coupling, easier testing, better code organization

**Types of Dependency Injection:**

1. **Constructor Injection (Recommended):**
```java
@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

2. **Setter Injection:**
```java
@Service
public class UserService {
    private UserRepository userRepository;

    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

3. **Field Injection (Not recommended):**
```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
}
```

### Q16: What is the difference between @Component and @Bean?

**Answer:**

**@Component:**
- Class-level annotation
- Used for automatic component scanning
- Applied to classes that Spring should automatically detect and register as beans
- Spring automatically creates instances

**@Bean:**
- Method-level annotation
- Used in @Configuration classes
- Used for manual bean creation when you need more control
- Explicitly defines how to create and configure the bean

```java
// @Component example
@Component
public class UserService {
    // Spring automatically creates this bean
}

// @Bean example
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        // Custom bean creation logic
        return DataSourceBuilder.create()
            .url("jdbc:mysql://localhost:3306/mydb")
            .build();
    }
}
```

### Q17: What are the scopes of a Spring bean?

**Answer:** Spring beans have different scopes that define their lifecycle:

**1. Singleton (default):**
- Single instance per Spring IoC container
- All requests for the bean return the same instance

**2. Prototype:**
- New instance created each time the bean is requested
- Not managed by Spring after creation

**3. Request (web applications):**
- Single instance per HTTP request
- Valid only in web-aware Spring ApplicationContext

**4. Session (web applications):**
- Single instance per HTTP session
- Valid only in web-aware Spring ApplicationContext

```java
@Service
@Scope("singleton")  // Default
public class SingletonService {
}

@Component
@Scope("prototype")
public class PrototypeComponent {
}
```

---

## REST API Development

### Q18: How do you create a RESTful API in Spring Boot?

**Answer:** Creating a RESTful API involves:

**1. Create a REST controller:**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    // GET all users
    @GetMapping
    public ResponseEntity<List<User>> getAllUsers() {
        List<User> users = userService.findAll();
        return ResponseEntity.ok(users);
    }

    // GET user by ID
    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(user);
    }

    // POST - Create new user
    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        User createdUser = userService.save(user);
        return ResponseEntity.status(HttpStatus.CREATED).body(createdUser);
    }

    // PUT - Update user
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(
        @PathVariable Long id,
        @Valid @RequestBody User user) {
        User updatedUser = userService.update(id, user);
        return ResponseEntity.ok(updatedUser);
    }

    // DELETE - Delete user
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Q19: What is the difference between @PathVariable and @RequestParam?

**Answer:**

**@PathVariable:**
- Extracts values from the URI path
- Used for path parameters
- Example: `/api/users/{id}`

**@RequestParam:**
- Extracts values from the query string
- Used for query parameters
- Example: `/api/users?page=1&size=10`

```java
@GetMapping("/users/{id}")
public User getUserById(@PathVariable Long id) {
    // URL: /api/users/123
    return userService.findById(id);
}

@GetMapping("/users")
public List<User> getUsers(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size) {
    // URL: /api/users?page=1&size=10
    return userService.findAll(page, size);
}
```

### Q20: What is the difference between @RequestBody and @RequestParam?

**Answer:**

**@RequestBody:**
- Binds HTTP request body to a Java object
- Used for POST, PUT, PATCH requests
- Typically used for JSON data
- Handles complex objects

**@RequestParam:**
- Binds request parameters (query string or form data) to method parameters
- Used for simple types (String, int, etc.)
- Handles multiple parameters

```java
@PostMapping("/users")
public User createUser(@RequestBody User user) {
    // Request body: {"name": "John", "email": "john@example.com"}
    return userService.save(user);
}

@GetMapping("/search")
public List<User> searchUsers(
    @RequestParam String name,
    @RequestParam String email) {
    // URL: /api/users/search?name=John&email=john@example.com
    return userService.search(name, email);
}
```

### Q21: How do you validate request data in Spring Boot?

**Answer:** Spring Boot provides validation using JSR-380 (Java Bean Validation):

**1. Add validation dependency:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

**2. Add validation annotations to DTO:**
```java
public class UserDTO {
    @NotNull(message = "Name is required")
    @Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @Min(value = 18, message = "Age must be at least 18")
    private int age;
}
```

**3. Use @Valid in controller:**
```java
@PostMapping("/users")
public ResponseEntity<User> createUser(@Valid @RequestBody UserDTO userDTO) {
    // Validation will be performed automatically
    User user = userService.createUser(userDTO);
    return ResponseEntity.ok(user);
}
```

**4. Handle validation errors:**
```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationExceptions(
        MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach((error) -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        return ResponseEntity.badRequest().body(errors);
    }
}
```

---

## Data Access & JPA

### Q22: What is Spring Data JPA?

**Answer:** Spring Data JPA is a part of the larger Spring Data family that makes it easier to implement JPA-based repositories. It provides:

- **Repository interfaces**: Simplified data access with predefined methods
- **Query derivation**: Automatic query generation from method names
- **Pagination support**: Built-in pagination and sorting
- **Custom queries**: Support for JPQL and native SQL queries
- **Auditing**: Automatic tracking of created/modified dates and users

### Q23: What are the common Spring Data JPA repositories?

**Answer:**

**1. Repository:**
- Marker interface for all Spring Data repositories
- Basic CRUD operations

**2. CrudRepository:**
- Extends Repository
- Provides CRUD operations: save, findById, findAll, delete, etc.

**3. JpaRepository:**
- Extends CrudRepository and PagingAndSortingRepository
- Provides JPA-specific methods: flush, saveAndFlush, deleteInBatch, etc.
- Most commonly used for JPA applications

**4. PagingAndSortingRepository:**
- Provides pagination and sorting methods

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Custom query methods
    List<User> findByEmail(String email);

    List<User> findByAgeGreaterThan(int age);

    Page<User> findByLastName(String lastName, Pageable pageable);

    @Query("SELECT u FROM User u WHERE u.email = :email")
    User findByEmailCustom(@Param("email") String email);

    @Modifying
    @Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :date")
    void deactivateInactiveUsers(@Param("date") LocalDate date);
}
```

### Q24: What are the common JPA annotations?

**Answer:**

**Entity Mapping:**
- `@Entity`: Marks a class as JPA entity (maps to database table)
- `@Table`: Specifies table name and other table details
- `@Id`: Specifies primary key
- `@GeneratedValue`: Specifies primary key generation strategy
- `@Column`: Specifies column details (name, nullable, length, etc.)
- `@Transient`: Marks fields that should not be persisted

**Relationships:**
- `@OneToOne`: One-to-one relationship
- `@OneToMany`: One-to-many relationship
- `@ManyToOne`: Many-to-one relationship
- `@ManyToMany`: Many-to-many relationship

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
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles;
}
```

### Q25: What is @Transactional annotation?

**Answer:** `@Transactional` is used to define the scope of a single database transaction:

- **Atomicity**: All operations succeed or all fail
- **Consistency**: Database remains in consistent state
- **Isolation**: Transactions don't interfere with each other
- **Durability**: Committed changes persist even after system failure

```java
@Service
public class UserService {
    @Transactional
    public void createUserWithOrders(User user, List<Order> orders) {
        // All database operations here will be in a single transaction
        userRepository.save(user);
        for (Order order : orders) {
            order.setUser(user);
            orderRepository.save(order);
        }
        // If any exception occurs, all changes will be rolled back
    }
}
```

### Q26: What are the different types of Entity relationships?

**Answer:**

**1. One-to-One (@OneToOne):**
```java
@Entity
public class User {
    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "profile_id")
    private UserProfile profile;
}
```

**2. One-to-Many (@OneToMany):**
```java
@Entity
public class Department {
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
    private List<Employee> employees;
}
```

**3. Many-to-One (@ManyToOne):**
```java
@Entity
public class Employee {
    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;
}
```

**4. Many-to-Many (@ManyToMany):**
```java
@Entity
public class Student {
    @ManyToMany
    @JoinTable(
        name = "student_courses",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses;
}
```

### Q27: What is the difference between lazy and eager loading?

**Answer:**

**EAGER Loading:**
- Loads related entities immediately when the parent entity is loaded
- Can cause performance issues with large datasets
- Example: `@OneToMany(fetch = FetchType.EAGER)`

**Lazy Loading:**
- Loads related entities only when they are accessed
- Better performance, avoids unnecessary data loading
- Default for most relationships
- Example: `@OneToMany(fetch = FetchType.LAZY)`

```java
@Entity
public class User {
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;  // Loaded only when accessed
}

// Accessing lazy loaded collection
User user = userRepository.findById(userId);
List<Order> orders = user.getOrders();  // Loaded here
```

---

## Testing

### Q28: How do you test a Spring Boot application?

**Answer:** Spring Boot provides comprehensive testing support:

**1. Unit Tests:**
```java
@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService userService;

    @MockBean
    private UserRepository userRepository;

    @Test
    void testFindUserById() {
        User user = new User("John", "john@example.com");
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));

        User found = userService.findById(1L);

        assertEquals("John", found.getName());
        verify(userRepository, times(1)).findById(1L);
    }
}
```

**2. Web Layer Tests (@WebMvcTest):**
```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void testGetUserById() throws Exception {
        when(userService.findById(1L)).thenReturn(new User("John"));

        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}
```

**3. Data Layer Tests (@DataJpaTest):**
```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private UserRepository userRepository;

    @Test
    void testFindByEmail() {
        User user = new User("John", "john@example.com");
        entityManager.persist(user);

        User found = userRepository.findByEmail("john@example.com");

        assertEquals("John", found.getName());
    }
}
```

### Q29: What is @MockBean?

**Answer:** `@MockBean` is a Spring Boot test annotation used to mock Spring beans:

- Creates a mock of the specified class/interface
- Replaces the actual bean in the Spring application context
- Integrates Mockito with Spring Boot
- Used when you want to test one component without relying on its dependencies

```java
@SpringBootTest
class MyServiceTest {

    @MockBean
    private ExternalApiService externalApiService;

    @Autowired
    private MyService myService;

    @Test
    void testMyService() {
        // Mock the dependency
        when(externalApiService.getData()).thenReturn("mocked data");

        // Test the service
        String result = myService.processData();

        assertEquals("processed: mocked data", result);
        verify(externalApiService, times(1)).getData();
    }
}
```

### Q30: What is the difference between @SpringBootTest and @WebMvcTest?

**Answer:**

**@SpringBootTest:**
- Loads the full application context
- Creates a complete application with all beans
- Used for integration tests
- Slower but tests real interactions
- Good for end-to-end testing

**@WebMvcTest:**
- Loads only the web layer
- Creates only MVC-related beans
- Used for unit testing controllers
- Faster and more focused
- Requires mocking service dependencies

```java
// @SpringBootTest example
@SpringBootTest
class FullApplicationTest {
    // Tests entire application context
}

// @WebMvcTest example
@WebMvcTest(UserController.class)
class UserControllerTest {
    @MockBean
    private UserService userService;  // Must mock dependencies
    // Tests only the web layer
}
```

---

## Spring Boot Features

### Q31: What is Spring Boot DevTools?

**Answer:** Spring Boot DevTools provides development-time features to improve development experience:

- **Automatic restart**: Restarts the application when files change (faster than manual restart)
- **LiveReload**: Automatically refreshes the browser when resources change
- **Property defaults**: Development-friendly default properties
- **Remote debug support**: Remote debugging capabilities
- **Disabled template caching**: Templates reload on every request

**Usage:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
</dependency>
```

### Q32: What is a CommandLineRunner?

**Answer:** `CommandLineRunner` is an interface used to execute code at application startup:

- Runs after the application context is loaded
- Useful for initialization tasks
- Multiple CommandLineRunners can be ordered with @Order annotation

```java
@Component
@Order(1)
public class DataInitializer implements CommandLineRunner {

    @Autowired
    private UserRepository userRepository;

    @Override
    public void run(String... args) throws Exception {
        // Initialize data at startup
        if (userRepository.count() == 0) {
            userRepository.save(new User("admin", "admin@example.com"));
        }
    }
}
```

### Q33: How does Spring Boot auto-configuration work?

**Answer:** Spring Boot auto-configuration automatically configures your application based on the dependencies in the classpath:

**Process:**

1. **Classpath scanning**: Spring Boot scans the classpath for jar files and classes
2. **Configuration files**: Reads `META-INF/spring.factories` or newer `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
3. **Conditional creation**: Uses annotations like `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`
4. **Bean registration**: Creates beans only when conditions are met

**Common condition annotations:**
- `@ConditionalOnClass`: Class is on the classpath
- `@ConditionalOnMissingBean`: Bean is not already defined
- `@ConditionalOnProperty`: Property has a specific value
- `@ConditionalOnWebApplication`: Web application context

**Example:**
```java
@Configuration
@ConditionalOnClass(DataSource.class)
@ConditionalOnProperty(name = "database.enabled", havingValue = "true")
public class DatabaseAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource() {
        // Create DataSource only if DataSource class is available
        // and database.enabled property is true
        // and no DataSource bean already exists
        return new HikariDataSource();
    }
}
```

### Q34: What is Spring Boot Starter Parent?

**Answer:** `spring-boot-starter-parent` is a special starter that provides:

- **Default configuration**: Maven configuration defaults
- **Dependency management**: Manages versions of common dependencies
- **Plugin configuration**: Default plugin configurations
- **Resource filtering**: Automatic resource filtering

**Usage:**
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<!-- No need to specify versions for Spring Boot managed dependencies -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <!-- Version is managed by parent -->
    </dependency>
</dependencies>
```

---

## Error Handling

### Q35: How do you handle exceptions in Spring Boot?

**Answer:** Spring Boot provides multiple ways to handle exceptions:

**1. @ExceptionHandler (Controller-level):**
```java
@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException ex) {
        ErrorResponse error = new ErrorResponse("USER_NOT_FOUND", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

**2. @ControllerAdvice (Global exception handling):**
```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException ex) {
        ErrorResponse error = new ErrorResponse("USER_NOT_FOUND", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationExceptions(MethodArgumentNotValidException ex) {
        ErrorResponse error = new ErrorResponse("VALIDATION_ERROR", "Invalid input");
        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAllExceptions(Exception ex) {
        ErrorResponse error = new ErrorResponse("INTERNAL_ERROR", "An error occurred");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

**3. @ResponseStatus:**
```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

### Q36: What is the difference between @ExceptionHandler and @ControllerAdvice?

**Answer:**

**@ExceptionHandler:**
- Method-level annotation
- Handles exceptions in a specific controller
- Limited scope
- Good for controller-specific exception handling

**@ControllerAdvice:**
- Class-level annotation
- Handles exceptions globally across all controllers
- Wider scope
- Good for centralized exception handling
- Can handle multiple exception types

```java
// @ExceptionHandler example
@RestController
public class UserController {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity.notFound().build();
    }
}

// @ControllerAdvice example
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity.notFound().build();
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException ex) {
        return ResponseEntity.badRequest().build();
    }
}
```

---

## Security Basics

### Q37: How do you secure a Spring Boot application?

**Answer:** Spring Boot Security can be added with minimal configuration:

**1. Add dependency:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

**2. Basic security configuration:**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults())
            .formLogin(withDefaults());

        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.builder()
            .username("user")
            .password("{noop}password")
            .roles("USER")
            .build();

        UserDetails admin = User.builder()
            .username("admin")
            .password("{noop}admin")
            .roles("ADMIN", "USER")
            .build();

        return new InMemoryUserDetailsManager(user, admin);
    }
}
```

### Q38: What is JWT authentication in Spring Boot?

**Answer:** JWT (JSON Web Token) is a stateless authentication method commonly used in REST APIs:

**Key components:**
- **JWT Token**: Contains user information and is signed
- **No server-side session**: Stateless authentication
- **Self-contained**: Contains all needed information

**Basic implementation:**
```java
@Service
public class JwtService {

    private String secretKey = "your-secret-key";

    public String generateToken(UserDetails userDetails) {
        return Jwts.builder()
            .subject(userDetails.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + 86400000))
            .signWith(Keys.hmacShaKeyFor(secretKey.getBytes()))
            .compact();
    }

    public boolean validateToken(String token, UserDetails userDetails) {
        String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }
}
```

---

## Spring vs Spring Boot

### Q39: What is the difference between Spring and Spring Boot?

**Answer:**

| Aspect | Spring Framework | Spring Boot |
|--------|----------------|-------------|
| **Purpose** | Comprehensive framework for Java enterprise applications | Simplifies Spring application development |
| **Configuration** | Requires extensive XML or Java-based configuration | Auto-configuration with sensible defaults |
| **Setup** | Manual dependency management, more boilerplate code | Starter dependencies, minimal setup |
| **Server** | Requires external server setup (Tomcat, Jetty) | Embedded server included |
| **Deployment** | WAR deployment | Executable JAR, "just run" |
| **Development Speed** | Slower initial setup | Faster development, quicker time to market |
| **Learning Curve** | Steeper learning curve | Easier for beginners |
| **Best For** | Complex enterprise applications, fine-grained control | Microservices, quick prototypes, modern applications |

**When to use Spring Framework:**
- Need fine-grained control over configuration
- Legacy application requirements
- Specific non-standard setup needs
- Complex enterprise applications

**When to use Spring Boot:**
- Quick application development
- Microservices architecture
- Modern web applications
- Rapid prototyping

### Q40: Can Spring Boot coexist with traditional Spring applications?

**Answer:** Yes, Spring Boot can coexist with traditional Spring applications:

- You can gradually migrate to Spring Boot
- Can use Spring Boot features in existing Spring applications
- Can mix traditional configuration with Spring Boot auto-configuration
- Allows incremental adoption of Spring Boot features

---

## Common Scenarios

### Q41: How do you implement pagination in Spring Boot?

**Answer:** Spring Data JPA provides built-in pagination support:

**1. Repository with pagination:**
```java
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findByLastName(String lastName, Pageable pageable);

    Page<User> findAll(Pageable pageable);
}
```

**2. Controller with pagination:**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping
    public Page<User> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(defaultValue = "name") String sortBy) {

        Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy));
        return userRepository.findAll(pageable);
    }
}
```

### Q42: How do you implement logging in Spring Boot?

**Answer:** Spring Boot uses SLF4J with Logback as the default logging framework:

**1. Simple logging:**
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class UserService {
    private static final Logger logger = LoggerFactory.getLogger(UserService.class);

    public User createUser(User user) {
        logger.info("Creating user: {}", user.getName());
        logger.debug("User details: {}", user);
        logger.error("Error creating user", exception);
        return userRepository.save(user);
    }
}
```

**2. Configure logging in application.properties:**
```properties
logging.level.root=INFO
logging.level.com.example.demo=WARN
logging.file.name=application.log
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} - %msg%n
```

### Q43: How do you create a custom Spring Boot starter?

**Answer:** Creating a custom starter involves:

**1. Create starter project structure:**
```
my-spring-boot-starter/
├── pom.xml
└── src/
    └── main/
        ├── java/
        │   └── com/example/starter/
        │       ├── autoconfigure/
        │       │   └── MyAutoConfiguration.java
        │       └── properties/
        │           └── MyProperties.java
        └── resources/
            └── META-INF/
                └── spring/
                    └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

**2. Create auto-configuration class:**
```java
@Configuration
@EnableConfigurationProperties(MyProperties.class)
@ConditionalOnClass(MyService.class)
public class MyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties properties) {
        return new MyService(properties);
    }
}
```

**3. Create properties class:**
```java
@ConfigurationProperties(prefix = "my")
public class MyProperties {
    private String name;
    private boolean enabled;
    // getters and setters
}
```

---

## Quick Reference Summary

### Essential Annotations
- `@SpringBootApplication`: Main application class
- `@RestController`: REST API controller
- `@RequestMapping`: Request mapping
- `@Autowired`: Dependency injection
- `@Service`, `@Repository`, `@Component`: Component stereotypes
- `@Entity`: JPA entity
- `@Transactional`: Transaction management
- `@Valid`: Validation
- `@ControllerAdvice`: Global exception handling

### Common Dependencies
- `spring-boot-starter-web`: Web applications
- `spring-boot-starter-data-jpa`: Database access
- `spring-boot-starter-security`: Security
- `spring-boot-starter-test`: Testing
- `spring-boot-starter-validation`: Validation

### Configuration Files
- `application.properties` / `application.yml`: Main configuration
- `application-{profile}.properties`: Profile-specific configuration

### Testing Annotations
- `@SpringBootTest`: Integration tests
- `@WebMvcTest`: Web layer tests
- `@DataJpaTest`: Data layer tests
- `@MockBean`: Mock beans in tests

---

**Tips for Interview Success:**

1. **Understand, don't memorize**: Focus on understanding concepts rather than rote memorization
2. **Provide examples**: Use code examples to illustrate your answers
3. **Explain trade-offs**: Discuss when to use which approach
4. **Mention best practices**: Show awareness of industry best practices
5. **Stay current**: Be aware of recent Spring Boot features and updates
6. **Hands-on experience**: Mention your practical experience with Spring Boot

**Good luck with your interviews!**

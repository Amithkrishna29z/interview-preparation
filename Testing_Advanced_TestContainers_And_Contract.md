# Testing Advanced: TestContainers, Contract Testing & Integration Testing
## Full Stack Java Developer Interview Preparation

---

## Table of Contents
1. [The Testing Pyramid and Strategy](#1-the-testing-pyramid-and-strategy)
2. [TestContainers — Deep Dive](#2-testcontainers--deep-dive)
3. [Pact — Consumer-Driven Contract Testing](#3-pact--consumer-driven-contract-testing)
4. [WireMock — HTTP Service Virtualization](#4-wiremock--http-service-virtualization)
5. [REST Assured — API Integration Testing](#5-rest-assured--api-integration-testing)
6. [BDD with Cucumber](#6-bdd-with-cucumber)
7. [Spring Boot Testing Slices](#7-spring-boot-testing-slices)
8. [Test Data Management](#8-test-data-management)
9. [Testing Asynchronous Code](#9-testing-asynchronous-code)
10. [Interview Questions and Answers](#10-interview-questions-and-answers)

---

## 1. The Testing Pyramid and Strategy

### 1.1 The Classic Testing Pyramid

The testing pyramid, introduced by Mike Cohn, describes the ideal distribution of tests in a software system. It has three layers:

```
        /\
       /  \
      / E2E \          <-- Few, slow, expensive
     /________\
    /          \
   / Integration \     <-- Medium number, medium speed
  /______________\
 /                \
/   Unit Tests     \   <-- Many, fast, cheap
/___________________\
```

**Layer 1 — Unit Tests (Base, widest)**
- Test a single class or method in complete isolation
- All external dependencies (database, network, other classes) are mocked or stubbed
- Should run in milliseconds
- A large codebase might have thousands of unit tests
- Goal: verify business logic correctness in isolation
- Tools: JUnit 5, Mockito, AssertJ

**Layer 2 — Integration Tests (Middle)**
- Test multiple components together
- May involve a real database, real message broker, or multiple Spring beans wired together
- Slower than unit tests (seconds to minutes)
- Fewer in number than unit tests
- Goal: verify that components work together correctly
- Tools: Spring Boot Test, TestContainers, WireMock

**Layer 3 — End-to-End Tests (Top, narrowest)**
- Test the complete user journey across the entire system
- Browser, API gateway, multiple microservices, database all involved
- Very slow (minutes), very expensive to maintain
- Very few — only for the most critical user paths
- Goal: verify the system works from a user's perspective
- Tools: Selenium, Playwright, Cypress, REST Assured against a real environment

**The key insight:** The base should be wide (many unit tests), and each layer above narrows (fewer tests). Inverting this — having more E2E tests than unit tests — is called the "Testing Ice Cream Cone" anti-pattern, which leads to slow, flaky, expensive test suites.

---

### 1.2 The Testing Honeycomb (Modern Microservices Model)

Introduced by Spotify for microservices architectures, the honeycomb challenges the pyramid's emphasis on unit tests.

```
     [  E2E  ]           <-- very few

  [Integration] [Integration]    <-- most tests here

[Unit][Unit][Unit][Unit]         <-- some unit tests
```

**Why the honeycomb for microservices?**

In a microservices world, each service is small. Heavily unit-testing each tiny service with lots of mocks gives false confidence — the mocks may not accurately represent how the real dependency behaves.

The honeycomb says:
- **Integration tests are more valuable** than unit tests in microservices because services are small and the real risk is in interactions
- Unit tests still matter for complex business logic
- E2E tests are still few, and now you also have contract tests to replace some E2E coverage

**The layers in the honeycomb:**
1. **Implementation detail tests (unit):** Test complex algorithms, business logic in isolation
2. **Integration tests:** Test the service with real dependencies (using TestContainers)
3. **Contract tests:** Replace many E2E scenarios by testing interface agreements
4. **E2E / User Journey tests:** Very few, only for critical flows

---

### 1.3 Test Classification in Detail

| Test Type | Scope | Dependencies | Speed | Count |
|-----------|-------|-------------|-------|-------|
| Unit | Single class/method | All mocked | < 10ms | Thousands |
| Integration | Multiple layers | Real DB, etc. | 1–30s | Hundreds |
| Component | Whole microservice | External stubs | 5–60s | Tens |
| Contract | API agreement only | Mock server | < 5s | Tens-Hundreds |
| E2E | Full user journey | Full system | 1–10min | Handful |

**Unit Tests**
```java
// Pure unit test — no Spring context, no DB
class OrderServiceTest {
    @Mock
    OrderRepository orderRepository;  // Mocked — no real DB
    
    @InjectMocks
    OrderService orderService;
    
    @Test
    void shouldCalculateTotalCorrectly() {
        Order order = new Order();
        order.addItem(new OrderItem("Product A", 2, 50.0));
        order.addItem(new OrderItem("Product B", 1, 30.0));
        
        double total = orderService.calculateTotal(order);
        
        assertThat(total).isEqualTo(130.0);
    }
}
```

**Integration Tests**
```java
// Loads full Spring context + real PostgreSQL via TestContainers
@SpringBootTest
@Testcontainers
class OrderServiceIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @Autowired
    OrderService orderService;  // Real service with real DB behind it
    
    @Test
    void shouldPersistOrderAndReturnWithId() {
        Order order = orderService.createOrder(new CreateOrderRequest(1L, 2));
        assertThat(order.getId()).isNotNull();
    }
}
```

**Component Tests**
- Test the entire microservice (all its layers) but stub external services
- Use WireMock to stub payment gateway, inventory service, email service
- Use TestContainers for the service's own database

**Contract Tests**
- Consumer writes a contract saying "I expect this response shape"
- Provider verifies its actual implementation matches the contract
- Covered in the Pact section

---

### 1.4 Test Coverage

**What is test coverage?**
Test coverage (code coverage) measures what percentage of production code is executed by tests. Tools: JaCoCo (Java), Istanbul (JavaScript).

**Types of coverage:**
- **Line coverage:** % of lines executed
- **Branch coverage:** % of if/else branches taken (more meaningful)
- **Method coverage:** % of methods called
- **Class coverage:** % of classes instantiated

**The 80% guideline:**
- Commonly cited target: 80% line coverage
- This is a guideline, not a law — 80% in the wrong places is less valuable than 60% in the right places
- Focus coverage on: business logic, edge cases, error handling
- Don't chase coverage for: auto-generated code, DTOs, simple getters/setters, framework boilerplate

**JaCoCo Maven configuration:**
```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

**What NOT to measure coverage on:**
```java
// Exclude from coverage reporting
@Generated  // Mark auto-generated code
public class UserMapper { ... }
```

In JaCoCo config:
```xml
<excludes>
    <exclude>**/dto/**</exclude>
    <exclude>**/config/**</exclude>
    <exclude>**/*Application.class</exclude>
</excludes>
```

---

## 2. TestContainers — Deep Dive

### 2.1 What is TestContainers?

TestContainers is a Java library that provides lightweight, throwaway instances of common databases, message brokers, web browsers, and virtually anything else that can run in a Docker container — for use in tests.

**The core problem it solves:**

Before TestContainers, integration tests used in-memory alternatives:
- H2 instead of PostgreSQL
- Embedded Kafka instead of real Kafka
- Flapdoodle embedded MongoDB instead of real MongoDB

**Why in-memory databases are problematic:**
- H2 SQL dialect differs from PostgreSQL — some queries work in production but fail in H2 or vice versa
- PostgreSQL-specific features (JSON columns, array types, window functions, CTEs) do not work in H2
- In-memory brokers may not faithfully replicate the behavior of real Kafka in edge cases
- Tests give false confidence: "it passed in H2" does not mean it works in PostgreSQL

**Why TestContainers is better:**
- Tests run against the exact same database version as production
- All SQL features, constraints, stored procedures, triggers work correctly
- Container starts fresh each test run — clean state guaranteed
- Works in CI/CD — any environment with Docker can run these tests
- Supports hundreds of technologies via Docker images

**How it works internally:**
1. TestContainers uses the Docker daemon on the host (or a remote Docker daemon)
2. Before tests run, it pulls the specified Docker image and starts a container
3. It maps container ports to random host ports (to avoid conflicts)
4. After tests finish, a Ryuk container (the resource reaper) automatically stops and removes containers

---

### 2.2 Maven Dependencies

```xml
<properties>
    <testcontainers.version>1.19.3</testcontainers.version>
</properties>

<dependencyManagement>
    <dependencies>
        <!-- Import the TestContainers BOM for consistent versions -->
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers-bom</artifactId>
            <version>${testcontainers.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Core TestContainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- JUnit 5 integration -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- PostgreSQL module -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- Kafka module -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>kafka</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- MongoDB module -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>mongodb</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

### 2.3 PostgreSQL TestContainer with Spring Boot

```java
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {

    // static = shared container across ALL tests in this class (one container startup)
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("testuser")
        .withPassword("testpass")
        .withInitScript("sql/schema.sql");  // Optional: run SQL on startup

    // @DynamicPropertySource: override Spring Boot datasource properties
    // with the actual container's randomly assigned host/port
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.datasource.driver-class-name",
            () -> "org.postgresql.Driver");
    }

    @Autowired
    UserRepository userRepository;

    @BeforeEach
    void cleanUp() {
        userRepository.deleteAll();  // Clean state before each test
    }

    @Test
    void shouldSaveAndFindUser() {
        User user = new User(null, "John Doe", "john@example.com");
        User saved = userRepository.save(user);

        assertThat(saved.getId()).isNotNull();

        Optional<User> found = userRepository.findByEmail("john@example.com");
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John Doe");
    }

    @Test
    void shouldFindAllUsersOrderedByName() {
        userRepository.saveAll(List.of(
            new User(null, "Zebra User", "z@example.com"),
            new User(null, "Apple User", "a@example.com")
        ));

        List<User> users = userRepository.findAllByOrderByNameAsc();

        assertThat(users).hasSize(2);
        assertThat(users.get(0).getName()).isEqualTo("Apple User");
    }
}
```

**Key annotations explained:**

| Annotation | Purpose |
|-----------|---------|
| `@Testcontainers` | JUnit 5 extension that manages container lifecycle |
| `@Container` (static) | Container shared across all tests in the class — started once |
| `@Container` (instance) | New container for every test method — isolated but very slow |
| `@DynamicPropertySource` | Overrides Spring properties at test startup with container values |

**Why `static` @Container is preferred:**
- Starting a PostgreSQL container takes 3–8 seconds
- If you have 50 test methods, non-static means 50 container startups = 4+ minutes just in startup time
- Static container: 1 startup, all 50 tests share it — total startup ~5 seconds
- Trade-off: tests must not leave dirty data (use @BeforeEach cleanup or @Transactional)

---

### 2.4 Reusable Containers (withReuse)

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
    .withDatabaseName("testdb")
    .withUsername("test")
    .withPassword("test")
    .withReuse(true);  // Container persists between test runs (JVM restarts)
```

**How .withReuse(true) works:**
- TestContainers computes a hash of the container configuration
- On first run: starts the container and stores the hash
- On subsequent runs (e.g., running tests again in the same developer session): detects same hash, reuses the already-running container instead of starting a new one
- Massively speeds up local development

**Requirement:** Add `testcontainers.reuse.enable=true` to `~/.testcontainers.properties`

---

### 2.5 MySQL TestContainer

```java
@Container
static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
    .withDatabaseName("orders_test")
    .withUsername("orders_user")
    .withPassword("orders_pass")
    .withInitScript("db/init.sql")           // Run SQL on startup
    .withUrlParam("allowPublicKeyRetrieval", "true")  // MySQL 8 requirement
    .withUrlParam("useSSL", "false");

@DynamicPropertySource
static void mysqlProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", mysql::getJdbcUrl);
    registry.add("spring.datasource.username", mysql::getUsername);
    registry.add("spring.datasource.password", mysql::getPassword);
}
```

---

### 2.6 MongoDB TestContainer

```java
@SpringBootTest
@Testcontainers
class ProductRepositoryMongoTest {

    @Container
    static MongoDBContainer mongodb = new MongoDBContainer("mongo:6.0")
        .withExposedPorts(27017);

    @DynamicPropertySource
    static void mongoProperties(DynamicPropertyRegistry registry) {
        // getReplicaSetUrl returns mongodb://host:port/test
        registry.add("spring.data.mongodb.uri", mongodb::getReplicaSetUrl);
    }

    @Autowired
    ProductRepository productRepository;

    @Test
    void shouldSaveAndRetrieveProduct() {
        Product product = new Product(null, "Laptop", 999.99, "Electronics");
        Product saved = productRepository.save(product);

        assertThat(saved.getId()).isNotNull();

        List<Product> electronics = productRepository.findByCategory("Electronics");
        assertThat(electronics).hasSize(1);
        assertThat(electronics.get(0).getName()).isEqualTo("Laptop");
    }
}
```

---

### 2.7 Kafka TestContainer

```java
@SpringBootTest
@Testcontainers
class OrderEventKafkaTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
        registry.add("spring.kafka.consumer.auto-offset-reset", () -> "earliest");
    }

    @Autowired
    OrderEventProducer producer;

    @Autowired
    KafkaTemplate<String, String> kafkaTemplate;

    // Capture consumed messages
    @SpyBean
    OrderEventConsumer consumer;

    @Test
    void shouldProduceAndConsumeOrderEvent() throws InterruptedException {
        // Arrange
        OrderCreatedEvent event = new OrderCreatedEvent(1L, "PENDING", Instant.now());

        // Act
        producer.publishOrderCreated(event);

        // Assert — Kafka is async, must await
        // Option 1: Thread.sleep (unreliable, avoid in production tests)
        // Option 2: Awaitility (preferred)
        await()
            .atMost(Duration.ofSeconds(10))
            .untilAsserted(() ->
                verify(consumer, times(1)).handleOrderCreated(any(OrderCreatedEvent.class))
            );
    }

    @Test
    void shouldConsumeMessageDirectlyFromKafka() throws Exception {
        // Produce a raw message
        kafkaTemplate.send("orders", "key-1", "{\"orderId\":1,\"status\":\"CREATED\"}");

        // Wait for consumer to process it
        await()
            .atMost(Duration.ofSeconds(15))
            .untilAsserted(() ->
                verify(consumer, atLeastOnce()).handleOrderCreated(any())
            );
    }
}
```

**Kafka Consumer Configuration for Tests:**
```java
// application-test.properties (or in @DynamicPropertySource)
spring.kafka.consumer.group-id=test-group-${random.uuid}  // Unique group per run
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.enable-auto-commit=true
```

---

### 2.8 Redis TestContainer

```java
@SpringBootTest
@Testcontainers
class CacheIntegrationTest {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7.0-alpine")
        .withExposedPorts(6379)
        .waitingFor(Wait.forLogMessage(".*Ready to accept connections.*\\n", 1));

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port",
            () -> redis.getMappedPort(6379).toString());
    }

    @Autowired
    ProductCacheService cacheService;

    @Test
    void shouldCacheAndRetrieveProduct() {
        Product product = new Product(1L, "Laptop", 999.99);
        cacheService.put("product:1", product);

        Optional<Product> cached = cacheService.get("product:1");
        assertThat(cached).isPresent();
        assertThat(cached.get().getName()).isEqualTo("Laptop");
    }

    @Test
    void shouldExpireAfterTTL() throws InterruptedException {
        cacheService.putWithTTL("temp:key", "value", Duration.ofSeconds(1));
        Thread.sleep(1500);  // Wait for expiry

        Optional<String> expired = cacheService.get("temp:key");
        assertThat(expired).isEmpty();
    }
}
```

---

### 2.9 Multiple Containers with Docker Network

When your tests need multiple containers to communicate with each other (e.g., application container needs to talk to a database container):

```java
@SpringBootTest
@Testcontainers
class MultiContainerIntegrationTest {

    // Create a shared Docker network
    static Network network = Network.newNetwork();

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withNetwork(network)
        .withNetworkAliases("postgres-test");  // DNS alias within the network

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7.0")
        .withNetwork(network)
        .withNetworkAliases("redis-test")
        .withExposedPorts(6379);

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0"))
        .withNetwork(network);

    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port",
            () -> redis.getMappedPort(6379));
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
}
```

---

### 2.10 Wait Strategies

TestContainers supports various strategies for waiting until a container is ready:

```java
// Wait for a specific log message
.waitingFor(Wait.forLogMessage(".*database system is ready to accept connections.*", 1))

// Wait for an HTTP endpoint to return a specific status
.waitingFor(Wait.forHttp("/health")
    .forStatusCode(200)
    .withStartupTimeout(Duration.ofSeconds(60)))

// Wait for a specific port to be open (TCP)
.waitingFor(Wait.forListeningPort())

// Wait for a specific port to be open
.waitingFor(Wait.forListeningPorts(5432))

// Custom strategy
.waitingFor(new AbstractWaitStrategy() {
    @Override
    protected void waitUntilReady() {
        // custom check logic
    }
})
```

---

### 2.11 Spring Boot 3.1+ @ServiceConnection

Spring Boot 3.1 introduced `@ServiceConnection`, which eliminates the need for `@DynamicPropertySource`:

```java
@SpringBootTest
@Testcontainers
class ModernSpringBootTest {

    // Spring Boot 3.1+ automatically detects PostgreSQLContainer
    // and configures spring.datasource.* properties
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    // Same for Kafka
    @Container
    @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    // Same for Redis
    @Container
    @ServiceConnection
    static RedisContainer redis = new RedisContainer("redis:7.0");

    @Autowired
    UserRepository userRepository;

    @Test
    void contextLoads() {
        // Full Spring Boot context with real PostgreSQL, Kafka, Redis
        assertThat(userRepository).isNotNull();
    }
}
```

**Even simpler with spring-boot-testcontainers:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
```

---

### 2.12 TestContainers with @DataJpaTest

`@DataJpaTest` loads only the JPA layer (entities, repositories, JPA config), not the full Spring context. By default it uses an embedded H2 database. Override this to use TestContainers:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderRepositorySliceTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Autowired
    TestEntityManager entityManager;

    @Autowired
    OrderRepository orderRepository;

    @Test
    void shouldFindOrdersByStatus() {
        // Arrange
        entityManager.persistAndFlush(new Order(null, 1L, "PENDING", BigDecimal.TEN));
        entityManager.persistAndFlush(new Order(null, 2L, "COMPLETED", BigDecimal.valueOf(50)));
        entityManager.persistAndFlush(new Order(null, 3L, "PENDING", BigDecimal.valueOf(25)));

        // Act
        List<Order> pendingOrders = orderRepository.findByStatus("PENDING");

        // Assert
        assertThat(pendingOrders).hasSize(2);
        assertThat(pendingOrders).extracting(Order::getStatus)
            .containsOnly("PENDING");
    }

    @Test
    @Sql("/sql/orders-test-data.sql")  // Load test data from SQL file
    void shouldCalculateTotalRevenueByCustomer() {
        BigDecimal revenue = orderRepository.calculateTotalRevenueForCustomer(1L);
        assertThat(revenue).isEqualByComparingTo(BigDecimal.valueOf(150.0));
    }
}
```

**Why `Replace.NONE`?** By default `@DataJpaTest` replaces your datasource with H2. `Replace.NONE` tells it "don't replace my datasource" — allowing TestContainers to provide the real database.

---

### 2.13 Flyway + TestContainers

When you use Flyway for database migrations, they run automatically during the Spring Boot context startup — including in tests. TestContainers + Flyway means your integration tests always run against a freshly migrated, up-to-date schema:

```java
@SpringBootTest
@Testcontainers
class FlywayMigrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("myapp_test");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.flyway.enabled", () -> "true");
        // Flyway migration scripts in src/main/resources/db/migration
        // are automatically applied when the context starts
    }

    @Autowired
    DataSource dataSource;

    @Test
    void migrationsShouldRunSuccessfully() throws Exception {
        try (Connection conn = dataSource.getConnection()) {
            // Verify tables created by migrations exist
            ResultSet rs = conn.getMetaData().getTables(
                null, null, "users", new String[]{"TABLE"});
            assertThat(rs.next()).isTrue();
        }
    }
}
```

---

### 2.14 Singleton Container Pattern

For test suites with many test classes all needing the same database, starting a new container per class is wasteful. The Singleton pattern shares one container across the entire JVM:

```java
// AbstractIntegrationTest.java — all integration tests extend this
public abstract class AbstractIntegrationTest {

    static final PostgreSQLContainer<?> POSTGRES;

    static {
        POSTGRES = new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("integration_tests")
            .withUsername("test")
            .withPassword("test")
            .withReuse(true);  // Reuse across JVM restarts in local dev
        POSTGRES.start();  // Start once, shared forever
    }

    @DynamicPropertySource
    static void overrideProps(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }
}

// UserRepositoryTest.java
@SpringBootTest
class UserRepositoryTest extends AbstractIntegrationTest {
    // Uses the shared POSTGRES container
}

// OrderRepositoryTest.java
@SpringBootTest
class OrderRepositoryTest extends AbstractIntegrationTest {
    // Uses the same shared POSTGRES container — no second startup
}
```

---

### 2.15 Log Consumer (Debugging)

Capture container logs for debugging failed tests:

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
    .withLogConsumer(new Slf4jLogConsumer(LoggerFactory.getLogger("PostgreSQL")));
```

---

## 3. Pact — Consumer-Driven Contract Testing

### 3.1 The Problem Contract Testing Solves

In a microservices system, service A (the **consumer**) calls service B (the **provider**). How do you ensure they remain compatible as both evolve?

**Option 1: Integration tests against real services**
- Requires both services running simultaneously
- If service B changes its API, service A's tests break — but only when tested together
- Slow, brittle, hard to maintain in large systems

**Option 2: Contract testing**
- Consumer defines a "contract": "I will call this endpoint and expect this response shape"
- Contract is tested against each side independently
- Provider tests its implementation against the contract without needing the consumer running
- Fast, independent, gives early feedback

---

### 3.2 Consumer-Driven Contracts (CDC) Concept

The key insight in CDC is that the **consumer** drives the contract. The consumer knows what it actually needs — it writes the contract saying "I need these fields, in this format."

**Flow:**
```
1. Consumer team writes a Pact (contract test)
   - Defines: "when I call GET /products/1, I expect: id (integer), name (string), price (decimal)"
   
2. Consumer test runs locally, passes
   - Pact generates a pact file: consumer-provider.json
   
3. Pact file published to Pact Broker (or checked into git)

4. Provider team runs provider verification
   - Pact framework calls the real provider with the consumer's requests
   - Verifies responses match what consumer expected
   
5. If provider breaks the contract (removes a field, changes a type):
   - Provider verification fails
   - Provider team knows they broke a consumer
   - Fix required before deployment
```

---

### 3.3 Pact Maven Dependencies

```xml
<dependency>
    <groupId>au.com.dius.pact.consumer</groupId>
    <artifactId>junit5</artifactId>
    <version>4.6.7</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>au.com.dius.pact.provider</groupId>
    <artifactId>junit5spring</artifactId>
    <version>4.6.7</version>
    <scope>test</scope>
</dependency>
```

---

### 3.4 Consumer Test — Full Example

**Scenario:** OrderService (consumer) calls ProductService (provider) to get product details.

```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "ProductService", port = "9999")
class ProductServiceConsumerPactTest {

    // Define the pact: what the consumer expects
    @Pact(consumer = "OrderService")
    public RequestResponsePact getExistingProductById(PactDslWithProvider builder) {
        return builder
            .given("a product with ID 1 exists")  // Provider state
            .uponReceiving("a GET request for product 1")
                .path("/api/products/1")
                .method("GET")
                .headers(Map.of("Accept", "application/json"))
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                    .integerType("id", 1)             // Must be integer, example: 1
                    .stringType("name", "Laptop")      // Must be string, example: "Laptop"
                    .decimalType("price", 999.99)      // Must be decimal
                    .stringType("category", "Electronics")
                    .booleanType("available", true))
            .toPact();
    }

    @Pact(consumer = "OrderService")
    public RequestResponsePact getProductNotFound(PactDslWithProvider builder) {
        return builder
            .given("no product with ID 999 exists")
            .uponReceiving("a GET request for non-existent product")
                .path("/api/products/999")
                .method("GET")
            .willRespondWith()
                .status(404)
                .body(new PactDslJsonBody()
                    .stringType("message", "Product not found"))
            .toPact();
    }

    @Pact(consumer = "OrderService")
    public RequestResponsePact getProductsByCategory(PactDslWithProvider builder) {
        return builder
            .given("products in category Electronics exist")
            .uponReceiving("a GET request for products by category")
                .path("/api/products")
                .method("GET")
                .query("category=Electronics")
            .willRespondWith()
                .status(200)
                .body(new PactDslJsonArray()
                    .object()
                        .integerType("id")
                        .stringType("name")
                        .decimalType("price")
                    .closeObject())
            .toPact();
    }

    // Test 1: Consumer calls the mocked provider and verifies its own client code
    @Test
    @PactTestFor(pactMethod = "getExistingProductById")
    void shouldFetchProductById(MockServer mockServer) {
        // ProductClient is the HTTP client in OrderService that calls ProductService
        ProductClient client = new ProductClient(mockServer.getUrl());

        ProductDto product = client.getById(1L);

        assertThat(product.getId()).isEqualTo(1L);
        assertThat(product.getName()).isEqualTo("Laptop");
        assertThat(product.getPrice()).isEqualByComparingTo(BigDecimal.valueOf(999.99));
        assertThat(product.isAvailable()).isTrue();
    }

    @Test
    @PactTestFor(pactMethod = "getProductNotFound")
    void shouldThrowExceptionWhenProductNotFound(MockServer mockServer) {
        ProductClient client = new ProductClient(mockServer.getUrl());

        assertThatThrownBy(() -> client.getById(999L))
            .isInstanceOf(ProductNotFoundException.class)
            .hasMessageContaining("Product not found");
    }
}
```

**What happens when this test runs:**
1. Pact starts a mock server at port 9999
2. The mock server is configured with the expectations defined in `@Pact` methods
3. Your `ProductClient` calls the mock server (not the real ProductService)
4. Pact verifies that the client made the right call
5. Pact generates a file: `target/pacts/OrderService-ProductService.json`

**The generated pact file (JSON):**
```json
{
  "consumer": { "name": "OrderService" },
  "provider": { "name": "ProductService" },
  "interactions": [
    {
      "description": "a GET request for product 1",
      "providerStates": [
        { "name": "a product with ID 1 exists" }
      ],
      "request": {
        "method": "GET",
        "path": "/api/products/1"
      },
      "response": {
        "status": 200,
        "headers": { "Content-Type": "application/json" },
        "body": { "id": 1, "name": "Laptop", "price": 999.99 }
      }
    }
  ]
}
```

---

### 3.5 Provider Verification — Full Example

```java
@Provider("ProductService")
@PactFolder("pacts")  // Or @PactBroker for Pact Broker
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class ProductServiceProviderPactTest {

    @LocalServerPort
    private int port;

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    ProductRepository productRepository;

    @BeforeEach
    void setUp(PactVerificationContext context) {
        // Tell Pact where the real provider is running
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    // This is the test runner — DO NOT add assertions here
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.verifyInteraction();
    }

    // Provider states: set up the data needed for each interaction
    @State("a product with ID 1 exists")
    void productWithId1Exists() {
        productRepository.deleteAll();
        productRepository.save(new Product(1L, "Laptop", new BigDecimal("999.99"),
            "Electronics", true));
    }

    @State("no product with ID 999 exists")
    void noProductWithId999Exists() {
        productRepository.deleteById(999L);  // Ensure it doesn't exist
    }

    @State("products in category Electronics exist")
    void electronicsProductsExist() {
        productRepository.deleteAll();
        productRepository.saveAll(List.of(
            new Product(1L, "Laptop", new BigDecimal("999.99"), "Electronics", true),
            new Product(2L, "Mouse", new BigDecimal("29.99"), "Electronics", true)
        ));
    }
}
```

---

### 3.6 Pact Broker

The Pact Broker is a central service that:
- Stores pact files from consumers
- Serves pacts to providers for verification
- Records verification results
- Provides "Can I Deploy?" — checks if consumer and provider versions are compatible

**Running Pact Broker with Docker Compose:**
```yaml
version: '3'
services:
  pact-broker:
    image: pactfoundation/pact-broker:latest
    ports:
      - "9292:9292"
    environment:
      PACT_BROKER_DATABASE_ADAPTER: sqlite
      PACT_BROKER_DATABASE_NAME: /tmp/pact_broker.sqlite
```

**Publishing pacts to Pact Broker (Maven plugin):**
```xml
<plugin>
    <groupId>au.com.dius.pact.provider</groupId>
    <artifactId>maven</artifactId>
    <version>4.6.7</version>
    <configuration>
        <pactBrokerUrl>http://localhost:9292</pactBrokerUrl>
        <pactDirectory>target/pacts</pactDirectory>
        <projectVersion>${project.version}</projectVersion>
        <tags>
            <tag>main</tag>
        </tags>
    </configuration>
</plugin>
```

**Consumer test with Pact Broker:**
```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "ProductService")
class ProductConsumerPactBrokerTest {

    @Pact(consumer = "OrderService")
    public RequestResponsePact getProduct(PactDslWithProvider builder) { ... }

    @Test
    @PactTestFor(pactMethod = "getProduct")
    void test(MockServer mockServer) { ... }
}
// Run: mvn pact:publish to push pact to broker
```

**Provider test with Pact Broker:**
```java
@Provider("ProductService")
@PactBroker(
    url = "http://localhost:9292",
    consumerVersionSelectors = @ConsumerVersionSelector(branch = "main")
)
@SpringBootTest(webEnvironment = RANDOM_PORT)
class ProductProviderBrokerTest {
    // Same structure as before, but pacts come from broker
}
```

**Can I Deploy?**
```bash
# Check if OrderService version 1.2.0 is compatible with ProductService in production
pact-broker can-i-deploy \
  --pacticipant OrderService \
  --version 1.2.0 \
  --to-environment production
```

---

### 3.7 Message Contract Testing with Pact

For asynchronous messaging (Kafka, RabbitMQ):

**Consumer test (message consumer):**
```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "OrderService", providerType = ProviderType.ASYNCH)
class OrderEventConsumerPactTest {

    @Pact(consumer = "NotificationService")
    public MessagePact orderCreatedEvent(MessagePactBuilder builder) {
        return builder
            .given("an order was created")
            .expectsToReceive("an order created event")
            .withContent(new PactDslJsonBody()
                .integerType("orderId", 12345)
                .stringType("customerId", "CUST-001")
                .stringType("status", "CREATED")
                .decimalType("total", 199.99)
                .stringType("createdAt"))
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "orderCreatedEvent")
    void shouldHandleOrderCreatedEvent(List<Message> messages) {
        // Verify the consumer can handle the message
        OrderEventHandler handler = new OrderEventHandler();

        for (Message message : messages) {
            handler.handleOrderCreated(
                new String(message.contentsAsBytes())
            );
        }

        // Verify handler processed it correctly
        verify(notificationService).sendOrderConfirmation(any());
    }
}
```

---

## 4. WireMock — HTTP Service Virtualization

### 4.1 What is WireMock?

WireMock is a mock HTTP server. When your service under test calls an external HTTP API (payment gateway, email service, SMS provider, another microservice), WireMock intercepts those calls and returns predefined responses.

**Key use cases:**
- Stub external payment gateway APIs
- Simulate third-party API failures (timeouts, 500 errors)
- Test retry logic
- Test your HTTP client code in isolation
- Test how your service handles slow responses

**WireMock vs Pact:**
- **WireMock:** Provider of stubs (consumer-side only). No verification that the real provider matches the stub.
- **Pact:** Consumer defines contract, provider verifies it. Two-sided contract with verification.

---

### 4.2 Maven Dependencies

```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <version>3.3.1</version>
    <scope>test</scope>
</dependency>

<!-- OR for Spring Boot integration -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-wiremock</artifactId>
    <scope>test</scope>
</dependency>
```

---

### 4.3 WireMock Programmatic Setup

```java
@SpringBootTest
class PaymentServiceIntegrationTest {

    static WireMockServer wireMockServer;

    @BeforeAll
    static void startWireMock() {
        wireMockServer = new WireMockServer(WireMockConfiguration.options()
            .port(8089)
            .usingFilesUnderClasspath("wiremock"));  // Static stubs in classpath
        wireMockServer.start();
    }

    @AfterAll
    static void stopWireMock() {
        wireMockServer.stop();
    }

    @BeforeEach
    void resetStubs() {
        wireMockServer.resetAll();  // Clear stubs between tests
    }

    @Autowired
    PaymentGatewayClient paymentClient;  // Configured to call localhost:8089

    @Test
    void shouldProcessPaymentSuccessfully() {
        // Arrange: configure the stub
        wireMockServer.stubFor(
            post(urlEqualTo("/api/payments"))
                .withHeader("Content-Type", equalTo("application/json"))
                .withHeader("Authorization", matching("Bearer .*"))
                .withRequestBody(matchingJsonPath("$.amount",
                    equalTo("100.00")))
                .willReturn(aResponse()
                    .withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                        {
                            "transactionId": "TXN-12345",
                            "status": "SUCCESS",
                            "processedAt": "2024-01-15T10:30:00Z"
                        }
                        """))
        );

        // Act
        PaymentResult result = paymentClient.processPayment(
            new PaymentRequest("100.00", "USD", "card_token_abc"));

        // Assert — response
        assertThat(result.getTransactionId()).isEqualTo("TXN-12345");
        assertThat(result.getStatus()).isEqualTo("SUCCESS");

        // Assert — verify the client made the right request
        wireMockServer.verify(
            postRequestedFor(urlEqualTo("/api/payments"))
                .withHeader("Content-Type", equalTo("application/json"))
                .withRequestBody(matchingJsonPath("$.currency",
                    equalTo("USD")))
        );
    }

    @Test
    void shouldHandlePaymentGatewayTimeout() {
        wireMockServer.stubFor(
            post(urlEqualTo("/api/payments"))
                .willReturn(aResponse()
                    .withFixedDelay(5000)  // 5 second delay (trigger timeout)
                    .withStatus(200))
        );

        assertThatThrownBy(() ->
            paymentClient.processPayment(new PaymentRequest("50.00", "USD", "token")))
            .isInstanceOf(PaymentTimeoutException.class);
    }

    @Test
    void shouldHandlePaymentGateway500Error() {
        wireMockServer.stubFor(
            post(urlEqualTo("/api/payments"))
                .willReturn(aResponse()
                    .withStatus(500)
                    .withBody("{\"error\": \"Internal server error\"}"))
        );

        assertThatThrownBy(() ->
            paymentClient.processPayment(new PaymentRequest("50.00", "USD", "token")))
            .isInstanceOf(PaymentGatewayException.class)
            .hasMessageContaining("Gateway returned 500");
    }
}
```

---

### 4.4 WireMock Scenarios — Testing Retry Logic

Scenarios model stateful behavior: first call fails, second call succeeds (simulating transient failures):

```java
@Test
void shouldRetryOnTransientFailureAndSucceed() {
    // First call: server error
    wireMockServer.stubFor(
        post(urlEqualTo("/api/payments"))
            .inScenario("Payment Retry")
            .whenScenarioStateIs(STARTED)
            .willReturn(aResponse().withStatus(503))
            .willSetStateTo("First retry")
    );

    // Second call: another server error
    wireMockServer.stubFor(
        post(urlEqualTo("/api/payments"))
            .inScenario("Payment Retry")
            .whenScenarioStateIs("First retry")
            .willReturn(aResponse().withStatus(503))
            .willSetStateTo("Second retry")
    );

    // Third call: success
    wireMockServer.stubFor(
        post(urlEqualTo("/api/payments"))
            .inScenario("Payment Retry")
            .whenScenarioStateIs("Second retry")
            .willReturn(aResponse()
                .withStatus(200)
                .withBody("{\"transactionId\": \"TXN-OK\", \"status\": \"SUCCESS\"}"))
    );

    // PaymentClient has retry logic configured (max 3 retries)
    PaymentResult result = paymentClient.processPayment(
        new PaymentRequest("100.00", "USD", "token"));

    assertThat(result.getStatus()).isEqualTo("SUCCESS");

    // Verify exactly 3 calls were made (2 failures + 1 success)
    wireMockServer.verify(3,
        postRequestedFor(urlEqualTo("/api/payments")));
}
```

---

### 4.5 @AutoConfigureWireMock (Spring Boot)

Spring Cloud Contract provides an annotation-based approach:

```java
@SpringBootTest
@AutoConfigureWireMock(port = 0)  // port = 0 means random port
class ExternalApiTest {

    // WireMock stub files go in:
    // src/test/resources/__files/       (response body files)
    // src/test/resources/mappings/      (stub configuration JSON files)

    // Or use programmatic stubs with injected WireMockServer
    @Autowired
    WireMockServer wireMockServer;

    @Value("${wiremock.server.port}")  // Injected when port = 0
    int wireMockPort;
}
```

**Stub mapping file (src/test/resources/mappings/product-stub.json):**
```json
{
  "request": {
    "method": "GET",
    "url": "/api/products/1"
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "bodyFileName": "product-1.json"
  }
}
```

**Response body file (src/test/resources/__files/product-1.json):**
```json
{
  "id": 1,
  "name": "Laptop",
  "price": 999.99,
  "category": "Electronics"
}
```

---

### 4.6 WireMock Response Templating

Generate dynamic responses based on request content:

```java
wireMockServer.stubFor(
    get(urlPathMatching("/api/products/([0-9]+)"))
        .willReturn(aResponse()
            .withStatus(200)
            .withHeader("Content-Type", "application/json")
            .withTransformers("response-template")  // Enable Handlebars templating
            .withBody("""
                {
                    "id": {{request.pathSegments.[2]}},
                    "requestedAt": "{{now}}",
                    "requestId": "{{randomValue type='UUID'}}"
                }
                """))
);
```

---

## 5. REST Assured — API Integration Testing

### 5.1 What is REST Assured?

REST Assured is a Java DSL for testing and validating REST APIs. It provides a readable, BDD-style (given/when/then) syntax for making HTTP requests and asserting responses.

**Maven dependency:**
```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>5.4.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>json-path</artifactId>
    <version>5.4.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>json-schema-validator</artifactId>
    <version>5.4.0</version>
    <scope>test</scope>
</dependency>
```

---

### 5.2 Complete REST Assured Test Suite

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderApiRestAssuredTest {

    @LocalServerPort
    int port;

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Autowired
    OrderRepository orderRepository;

    @BeforeEach
    void setUp() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = port;
        RestAssured.basePath = "/api";
        orderRepository.deleteAll();
    }

    @Test
    void shouldCreateOrderAndReturn201() {
        String requestBody = """
            {
                "customerId": "CUST-001",
                "items": [
                    {"productId": 1, "quantity": 2, "price": 49.99},
                    {"productId": 2, "quantity": 1, "price": 19.99}
                ]
            }
            """;

        Integer orderId =
            given()
                .contentType(ContentType.JSON)
                .body(requestBody)
                .log().ifValidationFails()
            .when()
                .post("/orders")
            .then()
                .log().ifValidationFails()
                .statusCode(201)
                .header("Location", matchesPattern(".*/api/orders/\\d+"))
                .body("id", notNullValue())
                .body("customerId", equalTo("CUST-001"))
                .body("status", equalTo("PENDING"))
                .body("items", hasSize(2))
                .body("total", equalTo(119.97f))  // 2*49.99 + 1*19.99
            .extract()
                .path("id");

        assertThat(orderId).isNotNull();
    }

    @Test
    void shouldGetOrderById() {
        // Seed data
        Order order = orderRepository.save(
            new Order("CUST-001", "PENDING", BigDecimal.valueOf(100.00)));

        given()
        .when()
            .get("/orders/{id}", order.getId())
        .then()
            .statusCode(200)
            .body("id", equalTo(order.getId().intValue()))
            .body("customerId", equalTo("CUST-001"))
            .body("status", equalTo("PENDING"));
    }

    @Test
    void shouldReturn404ForNonExistentOrder() {
        given()
        .when()
            .get("/orders/999999")
        .then()
            .statusCode(404)
            .body("message", equalTo("Order not found"))
            .body("timestamp", notNullValue());
    }

    @Test
    void shouldReturn400ForInvalidRequest() {
        given()
            .contentType(ContentType.JSON)
            .body("{\"customerId\": \"\", \"items\": []}")  // Invalid: empty customerId
        .when()
            .post("/orders")
        .then()
            .statusCode(400)
            .body("errors", hasSize(greaterThan(0)))
            .body("errors[0].field", notNullValue());
    }

    @Test
    void shouldCancelOrder() {
        Order order = orderRepository.save(
            new Order("CUST-001", "PENDING", BigDecimal.valueOf(100.00)));

        given()
        .when()
            .put("/orders/{id}/cancel", order.getId())
        .then()
            .statusCode(200)
            .body("status", equalTo("CANCELLED"));
    }

    @Test
    void shouldGetAllOrdersForCustomer() {
        orderRepository.saveAll(List.of(
            new Order("CUST-001", "PENDING", BigDecimal.valueOf(50.00)),
            new Order("CUST-001", "COMPLETED", BigDecimal.valueOf(75.00)),
            new Order("CUST-002", "PENDING", BigDecimal.valueOf(25.00))
        ));

        given()
            .queryParam("customerId", "CUST-001")
        .when()
            .get("/orders")
        .then()
            .statusCode(200)
            .body("", hasSize(2))  // Only CUST-001's orders
            .body("customerId", everyItem(equalTo("CUST-001")));
    }

    @Test
    void shouldValidateResponseAgainstJsonSchema() {
        given()
        .when()
            .get("/orders")
        .then()
            .statusCode(200)
            .body(matchesJsonSchemaInClasspath("schemas/orders-response-schema.json"));
    }
}
```

---

### 5.3 REST Assured with Authentication

```java
@Test
void shouldRequireAuthenticationForProtectedEndpoint() {
    // No auth header — should get 401
    given()
    .when()
        .get("/admin/orders")
    .then()
        .statusCode(401);
}

@Test
void shouldAllowAccessWithValidBearerToken() {
    String token = obtainJwtToken("admin@example.com", "password");

    given()
        .header("Authorization", "Bearer " + token)
    .when()
        .get("/admin/orders")
    .then()
        .statusCode(200);
}

// OAuth2 with RestAssured
@Test
void shouldAccessWithOAuth2Token() {
    given()
        .auth().oauth2(jwtToken)
    .when()
        .get("/api/orders")
    .then()
        .statusCode(200);
}
```

---

### 5.4 REST Assured Request/Response Specs (Reuse)

```java
// Define reusable request spec
RequestSpecification authSpec = new RequestSpecBuilder()
    .setBaseUri("http://localhost")
    .setPort(port)
    .setBasePath("/api")
    .setContentType(ContentType.JSON)
    .addHeader("Authorization", "Bearer " + adminToken)
    .build();

// Define reusable response spec
ResponseSpecification successSpec = new ResponseSpecBuilder()
    .expectStatusCode(200)
    .expectContentType(ContentType.JSON)
    .build();

@Test
void shouldGetOrderUsingSpecs() {
    given()
        .spec(authSpec)
    .when()
        .get("/orders/1")
    .then()
        .spec(successSpec)
        .body("id", equalTo(1));
}
```

---

## 6. BDD with Cucumber

### 6.1 What is BDD?

Behavior-Driven Development (BDD) is a software development approach that encourages collaboration between developers, QA engineers, and business stakeholders. Tests are written in plain English using a format that everyone can understand.

**Key benefits:**
- Tests serve as living documentation
- Business stakeholders can read and write tests
- Forces thinking in terms of behavior, not implementation
- Gherkin syntax: Feature → Scenario → Given/When/Then

---

### 6.2 Maven Dependencies

```xml
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-java</artifactId>
    <version>7.15.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-spring</artifactId>
    <version>7.15.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-junit-platform-engine</artifactId>
    <version>7.15.0</version>
    <scope>test</scope>
</dependency>
```

---

### 6.3 Feature File (Gherkin)

```gherkin
# src/test/resources/features/order-management.feature
Feature: Order Management
  As a customer
  I want to manage my orders
  So that I can track and modify my purchases

  Background:
    Given I am authenticated as a customer with email "customer@example.com"
    And the product catalog has the following products:
      | id | name    | price | stock |
      | 1  | Laptop  | 999.99| 10    |
      | 2  | Mouse   | 29.99 | 50    |

  Scenario: Successfully create an order
    When I create an order with the following items:
      | productId | quantity |
      | 1         | 1        |
      | 2         | 2        |
    Then the order should be created successfully
    And the order total should be 1059.97
    And the order status should be "PENDING"
    And I should receive an order confirmation email

  Scenario: Attempt to order out-of-stock product
    Given product 1 has only 0 items in stock
    When I try to create an order for product 1 with quantity 1
    Then the order should fail with error "Product out of stock"

  Scenario: Cancel a pending order
    Given I have a pending order with ID "ORD-001"
    When I cancel order "ORD-001"
    Then the order status should be "CANCELLED"
    And the stock should be restored

  Scenario Outline: Apply discount by order total
    Given I have an order with total <total>
    When the discount is applied
    Then the discount amount should be <discount>

    Examples:
      | total  | discount |
      | 100.00 | 0.00     |
      | 500.00 | 25.00    |
      | 1000.00| 100.00   |
```

---

### 6.4 Spring Boot Configuration for Cucumber

```java
// CucumberSpringConfiguration.java — Required entry point for Cucumber + Spring
@CucumberContextConfiguration
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
public class CucumberSpringConfiguration {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
}
```

```java
// CucumberTestRunner.java — JUnit Platform runner
@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = PLUGIN_PROPERTY_NAME,
    value = "pretty, html:target/cucumber-reports/report.html")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME,
    value = "com.example.steps")
public class CucumberTestRunner { }
```

---

### 6.5 Step Definitions

```java
// OrderStepDefinitions.java
@Component
public class OrderStepDefinitions {

    @Autowired
    private OrderService orderService;

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private TestRestTemplate restTemplate;

    @LocalServerPort
    private int port;

    // Shared state between steps (use ThreadLocal in parallel execution)
    private ResponseEntity<?> lastResponse;
    private Long createdOrderId;
    private String authToken;

    @Given("I am authenticated as a customer with email {string}")
    public void authenticateAsCustomer(String email) {
        // Get JWT token for this user
        authToken = authService.getTokenForEmail(email);
    }

    @Given("the product catalog has the following products:")
    public void setupProducts(DataTable dataTable) {
        List<Map<String, String>> rows = dataTable.asMaps();
        for (Map<String, String> row : rows) {
            productRepository.save(new Product(
                Long.parseLong(row.get("id")),
                row.get("name"),
                new BigDecimal(row.get("price")),
                Integer.parseInt(row.get("stock"))
            ));
        }
    }

    @When("I create an order with the following items:")
    public void createOrderWithItems(DataTable dataTable) {
        List<Map<String, String>> rows = dataTable.asMaps();
        List<OrderItem> items = rows.stream()
            .map(row -> new OrderItem(
                Long.parseLong(row.get("productId")),
                Integer.parseInt(row.get("quantity"))))
            .collect(Collectors.toList());

        CreateOrderRequest request = new CreateOrderRequest(items);

        lastResponse = restTemplate.exchange(
            "http://localhost:" + port + "/api/orders",
            HttpMethod.POST,
            new HttpEntity<>(request, authHeaders()),
            OrderResponse.class);
    }

    @Then("the order should be created successfully")
    public void verifyOrderCreated() {
        assertThat(lastResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        OrderResponse order = (OrderResponse) lastResponse.getBody();
        createdOrderId = order.getId();
        assertThat(createdOrderId).isNotNull();
    }

    @Then("the order total should be {double}")
    public void verifyOrderTotal(double expectedTotal) {
        OrderResponse order = (OrderResponse) lastResponse.getBody();
        assertThat(order.getTotal().doubleValue())
            .isEqualByComparingTo(expectedTotal);
    }

    @Then("the order status should be {string}")
    public void verifyOrderStatus(String expectedStatus) {
        OrderResponse order = (OrderResponse) lastResponse.getBody();
        assertThat(order.getStatus()).isEqualTo(expectedStatus);
    }

    @Given("I have a pending order with ID {string}")
    public void createPendingOrder(String orderId) {
        // Create order in DB with specific ID or reference
        Order order = new Order("CUST-001", "PENDING", BigDecimal.valueOf(100.0));
        order.setExternalId(orderId);
        orderRepository.save(order);
    }

    @When("I cancel order {string}")
    public void cancelOrder(String orderId) {
        lastResponse = restTemplate.exchange(
            "http://localhost:" + port + "/api/orders/" + orderId + "/cancel",
            HttpMethod.PUT,
            new HttpEntity<>(authHeaders()),
            OrderResponse.class);
    }

    private HttpHeaders authHeaders() {
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(authToken);
        headers.setContentType(MediaType.APPLICATION_JSON);
        return headers;
    }
}
```

---

### 6.6 Cucumber Tags for Selective Execution

```gherkin
@smoke @critical
Scenario: Successfully create an order
  ...

@slow @e2e
Scenario: Complete order lifecycle
  ...
```

```java
// Run only @smoke tests
@ConfigurationParameter(key = FILTER_TAGS_PROPERTY_NAME, value = "@smoke")
public class SmokeTestRunner { }
```

---

## 7. Spring Boot Testing Slices

### 7.1 The Problem with @SpringBootTest

`@SpringBootTest` loads the entire Spring application context. This means all beans, all auto-configurations, all datasource connections — everything. For a unit or slice test, this is overkill and slow.

Spring Boot provides **test slices** — partial context loads that include only the components relevant to the layer being tested.

---

### 7.2 @WebMvcTest — Controller Layer Only

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    MockMvc mockMvc;

    @Autowired
    ObjectMapper objectMapper;

    // @MockBean creates a Mockito mock AND registers it as a Spring bean
    @MockBean
    OrderService orderService;

    @MockBean
    AuthenticationService authService;

    @Test
    void shouldReturn200WithOrderWhenFound() throws Exception {
        Order order = new Order(1L, "CUST-001", "PENDING", BigDecimal.valueOf(99.99));
        when(orderService.findById(1L)).thenReturn(order);

        mockMvc.perform(get("/api/orders/1")
                .header("Authorization", "Bearer valid-token")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.status").value("PENDING"))
            .andExpect(jsonPath("$.total").value(99.99))
            .andDo(print());
    }

    @Test
    void shouldReturn404WhenOrderNotFound() throws Exception {
        when(orderService.findById(999L))
            .thenThrow(new OrderNotFoundException("Order not found"));

        mockMvc.perform(get("/api/orders/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.message").value("Order not found"));
    }

    @Test
    void shouldReturn400WhenRequestBodyInvalid() throws Exception {
        String invalidBody = """
            {"customerId": "", "items": []}
            """;

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidBody))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isArray());
    }

    @Test
    void shouldReturn201WhenOrderCreated() throws Exception {
        CreateOrderRequest request = new CreateOrderRequest("CUST-001",
            List.of(new OrderItemDto(1L, 2)));
        Order created = new Order(42L, "CUST-001", "PENDING", BigDecimal.valueOf(199.98));

        when(orderService.createOrder(any())).thenReturn(created);

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", containsString("/api/orders/42")))
            .andExpect(jsonPath("$.id").value(42));
    }
}
```

**What @WebMvcTest loads:**
- `@Controller`, `@ControllerAdvice`, `@JsonComponent`, `Filter`
- `MockMvc` bean
- `WebSecurityConfigurer` beans

**What it does NOT load:**
- `@Service`, `@Repository`, `@Component` beans
- JPA repositories, datasource
- The full application context

---

### 7.3 @DataJpaTest — Repository Layer Only

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderRepositoryTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Autowired
    TestEntityManager entityManager;

    @Autowired
    OrderRepository orderRepository;

    @Test
    void shouldFindOrdersByCustomerIdAndStatus() {
        // Arrange using TestEntityManager
        entityManager.persistAndFlush(
            new Order(null, "CUST-001", "PENDING", BigDecimal.valueOf(100)));
        entityManager.persistAndFlush(
            new Order(null, "CUST-001", "COMPLETED", BigDecimal.valueOf(200)));
        entityManager.persistAndFlush(
            new Order(null, "CUST-002", "PENDING", BigDecimal.valueOf(50)));

        // Act
        List<Order> result = orderRepository
            .findByCustomerIdAndStatus("CUST-001", "PENDING");

        // Assert
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getCustomerId()).isEqualTo("CUST-001");
        assertThat(result.get(0).getStatus()).isEqualTo("PENDING");
    }

    @Test
    void shouldCalculateTotalRevenueForDateRange() {
        LocalDate start = LocalDate.of(2024, 1, 1);
        LocalDate end = LocalDate.of(2024, 1, 31);

        entityManager.persistAndFlush(
            new Order(null, "CUST-001", "COMPLETED", BigDecimal.valueOf(100),
                LocalDate.of(2024, 1, 15)));
        entityManager.persistAndFlush(
            new Order(null, "CUST-002", "COMPLETED", BigDecimal.valueOf(200),
                LocalDate.of(2024, 1, 20)));
        entityManager.persistAndFlush(
            new Order(null, "CUST-003", "COMPLETED", BigDecimal.valueOf(300),
                LocalDate.of(2024, 2, 1)));  // Out of range

        BigDecimal revenue = orderRepository
            .calculateRevenueForDateRange(start, end);

        assertThat(revenue).isEqualByComparingTo(BigDecimal.valueOf(300)); // 100 + 200
    }

    @Test
    @Transactional  // Rollback after each test
    void shouldUpdateOrderStatus() {
        Order order = entityManager.persistAndFlush(
            new Order(null, "CUST-001", "PENDING", BigDecimal.valueOf(100)));

        orderRepository.updateStatus(order.getId(), "COMPLETED");
        entityManager.refresh(order);  // Reload from DB

        assertThat(order.getStatus()).isEqualTo("COMPLETED");
    }
}
```

**What @DataJpaTest loads:**
- JPA-related components: `@Entity`, `@Repository`
- `TestEntityManager`
- Configures an embedded H2 by default (override with `Replace.NONE` + TestContainers)

---

### 7.4 @RestClientTest — HTTP Client Layer

```java
@RestClientTest(ProductServiceClient.class)
class ProductServiceClientTest {

    @Autowired
    ProductServiceClient client;

    @Autowired
    MockRestServiceServer server;

    @Test
    void shouldFetchProductById() throws Exception {
        server.expect(requestTo("/api/products/1"))
            .andExpect(method(HttpMethod.GET))
            .andExpect(header("Accept", "application/json"))
            .andRespond(withSuccess(
                """
                {"id": 1, "name": "Laptop", "price": 999.99}
                """,
                MediaType.APPLICATION_JSON));

        ProductDto product = client.getById(1L);

        assertThat(product.getName()).isEqualTo("Laptop");
        server.verify();
    }

    @Test
    void shouldThrowExceptionOnHttpError() {
        server.expect(requestTo("/api/products/999"))
            .andRespond(withStatus(HttpStatus.NOT_FOUND)
                .body("{\"message\": \"Not found\"}")
                .contentType(MediaType.APPLICATION_JSON));

        assertThatThrownBy(() -> client.getById(999L))
            .isInstanceOf(ProductNotFoundException.class);
    }
}
```

---

### 7.5 @SpringBootTest — Modes

```java
// MOCK (default): Creates a mock servlet environment, no actual HTTP server
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
class MockWebEnvTest {
    @Autowired MockMvc mockMvc;  // Use MockMvc
}

// RANDOM_PORT: Starts real HTTP server on a random port
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class RealServerTest {
    @LocalServerPort int port;  // Inject the random port
    @Autowired TestRestTemplate restTemplate;  // Or use REST Assured
}

// DEFINED_PORT: Starts real HTTP server on port from application.properties (default 8080)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
class DefinedPortTest {
    // Used when you need a predictable port (e.g., for WireMock config)
}

// NONE: Loads Spring context but does not start a servlet environment
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class ServiceLayerTest {
    // For testing services that don't need HTTP
    @Autowired OrderService orderService;
}
```

**When to use each slice:**

| Scenario | Recommended Approach |
|----------|---------------------|
| Test controller validation, HTTP mapping | `@WebMvcTest` |
| Test JPA queries, custom repository methods | `@DataJpaTest` + TestContainers |
| Test HTTP client code | `@RestClientTest` |
| Test security configuration | `@WebMvcTest` with security config |
| Test service layer (no HTTP, no DB) | Plain JUnit + Mockito |
| Full integration test (all layers) | `@SpringBootTest(RANDOM_PORT)` + TestContainers |
| Verify event publishing/consuming | `@SpringBootTest` + Kafka TestContainers |

---

## 8. Test Data Management

### 8.1 @Sql Annotation

Run SQL scripts before or after tests:

```java
@SpringBootTest
@Testcontainers
class OrderQueryTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Test
    @Sql("/sql/insert-test-orders.sql")    // Run before this test
    @Sql(scripts = "/sql/cleanup.sql",
         executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)  // Run after
    void shouldQueryOrdersCreatedInJanuary() {
        List<Order> orders = orderRepository.findByMonthCreated(1, 2024);
        assertThat(orders).hasSize(3);
    }
}
```

**SQL test data file (src/test/resources/sql/insert-test-orders.sql):**
```sql
INSERT INTO orders (id, customer_id, status, total, created_date)
VALUES
    (1, 'CUST-001', 'COMPLETED', 100.00, '2024-01-10'),
    (2, 'CUST-001', 'PENDING', 200.00, '2024-01-15'),
    (3, 'CUST-002', 'COMPLETED', 50.00, '2024-01-20');
```

---

### 8.2 Test Builder Pattern (Object Mother / Test Data Builder)

```java
// OrderTestBuilder.java — fluent builder for test data
public class OrderTestBuilder {
    private Long id = null;
    private String customerId = "CUST-DEFAULT";
    private String status = "PENDING";
    private BigDecimal total = BigDecimal.valueOf(100.00);
    private List<OrderItem> items = new ArrayList<>();
    private LocalDateTime createdAt = LocalDateTime.now();

    public static OrderTestBuilder anOrder() {
        return new OrderTestBuilder();
    }

    public OrderTestBuilder withId(Long id) {
        this.id = id;
        return this;
    }

    public OrderTestBuilder forCustomer(String customerId) {
        this.customerId = customerId;
        return this;
    }

    public OrderTestBuilder withStatus(String status) {
        this.status = status;
        return this;
    }

    public OrderTestBuilder withTotal(double total) {
        this.total = BigDecimal.valueOf(total);
        return this;
    }

    public OrderTestBuilder withItem(Long productId, int qty, double price) {
        this.items.add(new OrderItem(productId, qty, BigDecimal.valueOf(price)));
        return this;
    }

    public OrderTestBuilder pending() {
        this.status = "PENDING";
        return this;
    }

    public OrderTestBuilder completed() {
        this.status = "COMPLETED";
        return this;
    }

    public Order build() {
        return new Order(id, customerId, status, total, items, createdAt);
    }

    public Order buildAndSave(OrderRepository repo) {
        return repo.save(build());
    }
}

// Usage in tests:
Order order = anOrder()
    .forCustomer("CUST-001")
    .withItem(1L, 2, 49.99)
    .withItem(2L, 1, 19.99)
    .pending()
    .build();
```

---

### 8.3 @BeforeEach vs @Sql vs @Transactional for Cleanup

```java
// Option 1: @BeforeEach cleanup
@BeforeEach
void cleanDatabase() {
    orderRepository.deleteAll();
    customerRepository.deleteAll();
}

// Option 2: @Transactional rollback (fastest — no actual DB writes committed)
@SpringBootTest
@Transactional  // Each test runs in a transaction that is rolled back after
class OrderServiceTest {
    @Test
    void testSomething() {
        // Changes here are rolled back after the test
        // WARNING: Tests must be single-threaded
        // WARNING: Tests testing COMMIT behavior won't work
    }
}

// Option 3: @Sql cleanup script
@AfterEach
@Sql("/sql/cleanup-all.sql")
void cleanup() {}
```

---

### 8.4 ArchUnit — Architecture Fitness Tests

ArchUnit lets you write automated tests for your architecture rules:

```xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version>1.2.1</version>
    <scope>test</scope>
</dependency>
```

```java
@AnalyzeClasses(packages = "com.example.orderservice")
class ArchitectureTest {

    @ArchTest
    static final ArchRule servicesShouldNotDependOnControllers =
        noClasses()
            .that().resideInAPackage("..service..")
            .should().dependOnClassesThat()
            .resideInAPackage("..controller..");

    @ArchTest
    static final ArchRule repositoriesShouldOnlyBeUsedByServices =
        classes()
            .that().resideInAPackage("..repository..")
            .should().onlyBeAccessed()
            .byAnyPackage("..service..", "..repository..");

    @ArchTest
    static final ArchRule servicesShouldBeAnnotated =
        classes()
            .that().resideInAPackage("..service..")
            .and().haveSimpleNameEndingWith("Service")
            .should().beAnnotatedWith(Service.class);

    @ArchTest
    static final ArchRule noCircularDependencies =
        slices()
            .matching("com.example.orderservice.(*)..")
            .should().beFreeOfCycles();
}
```

---

## 9. Testing Asynchronous Code

### 9.1 Awaitility for Async Tests

Awaitility is a DSL for testing asynchronous operations. Never use `Thread.sleep()` in tests — it's unreliable and slow.

```xml
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <version>4.2.0</version>
    <scope>test</scope>
</dependency>
```

```java
import static org.awaitility.Awaitility.*;
import static java.util.concurrent.TimeUnit.*;

@SpringBootTest
@Testcontainers
class OrderEventProcessingTest {

    @Container
    @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    @Autowired
    OrderEventProducer producer;

    @Autowired
    OrderRepository orderRepository;

    @Test
    void shouldProcessOrderCreatedEventAndUpdateDatabase() {
        // Act: publish event
        producer.publishOrderCreated(new OrderCreatedEvent(42L, "CUST-001"));

        // Assert: wait until the consumer processes it and updates the DB
        await()
            .atMost(30, SECONDS)          // Maximum wait time
            .pollInterval(500, MILLISECONDS)  // Check every 500ms
            .untilAsserted(() -> {
                Optional<Order> order = orderRepository.findById(42L);
                assertThat(order).isPresent();
                assertThat(order.get().getStatus()).isEqualTo("PROCESSING");
            });
    }

    @Test
    void shouldSendEmailAfterOrderCompleted() {
        // Arrange
        Order order = createOrder();

        // Act: trigger async email sending
        orderService.completeOrder(order.getId());

        // Assert: check that email was eventually sent (async)
        await()
            .atMost(Duration.ofSeconds(10))
            .until(() -> emailRepository.findByOrderId(order.getId()).isPresent());

        Email email = emailRepository.findByOrderId(order.getId()).get();
        assertThat(email.getSubject()).contains("Order Confirmed");
    }
}
```

---

### 9.2 Testing @Async Methods

```java
@Service
public class ReportService {
    @Async
    public CompletableFuture<Report> generateReport(Long orderId) {
        // Long-running operation
        return CompletableFuture.completedFuture(new Report(orderId, "DONE"));
    }
}

@SpringBootTest
class ReportServiceTest {

    @Autowired
    ReportService reportService;

    @Test
    void shouldGenerateReportAsynchronously() throws Exception {
        CompletableFuture<Report> future = reportService.generateReport(1L);

        // Get with timeout
        Report report = future.get(5, TimeUnit.SECONDS);

        assertThat(report.getOrderId()).isEqualTo(1L);
        assertThat(report.getStatus()).isEqualTo("DONE");
    }

    @Test
    void shouldGenerateMultipleReportsConcurrently() throws Exception {
        List<CompletableFuture<Report>> futures = IntStream.rangeClosed(1, 5)
            .mapToObj(i -> reportService.generateReport((long) i))
            .collect(Collectors.toList());

        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .get(30, TimeUnit.SECONDS);

        List<Report> reports = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());

        assertThat(reports).hasSize(5);
    }
}
```

---

### 9.3 Testing Kafka Consumer with @EmbeddedKafka

When you don't need the full realism of a real Kafka (and TestContainers is too heavy for unit-ish tests):

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"orders"})
class OrderConsumerEmbeddedKafkaTest {

    @Autowired
    KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    OrderEventConsumer consumer;

    @Test
    void shouldConsumeOrderCreatedEvent() throws Exception {
        String message = "{\"orderId\": 1, \"status\": \"CREATED\"}";

        kafkaTemplate.send("orders", "key-1", message);

        await()
            .atMost(Duration.ofSeconds(10))
            .untilAsserted(() ->
                assertThat(consumer.getProcessedEvents())
                    .anyMatch(e -> e.getOrderId().equals(1L)));
    }
}
```

---

## 10. Interview Questions and Answers

### Section A: Testing Fundamentals

---

**Q1: What is the testing pyramid? Why does it matter?**

The testing pyramid is a model for structuring tests in three layers:
- **Base (wide): Unit tests** — fast, isolated, many. Test individual classes with all dependencies mocked.
- **Middle: Integration tests** — test multiple components together with real infrastructure.
- **Top (narrow): E2E tests** — test the complete user journey, slow and few.

It matters because it guides the right *proportion* of tests. Having mostly E2E tests (the "ice cream cone" anti-pattern) leads to slow, flaky, expensive test suites. Having mostly unit tests with good integration coverage gives fast feedback and confidence.

---

**Q2: What is the testing honeycomb, and when does it apply?**

The honeycomb is a model proposed by Spotify for **microservices**. It de-emphasizes unit tests in favor of integration tests, because in microservices:
- Each service is small, so there is less business logic to unit test
- The real risks are in service interactions and integrations
- Heavily mocked unit tests in small services can give false confidence

The honeycomb prioritizes: integration tests (test the service with real DB), contract tests (verify service agreements), and very few E2E tests. Unit tests are still used for complex algorithms.

---

**Q3: What is the difference between @Mock and @MockBean?**

| | `@Mock` (Mockito) | `@MockBean` (Spring Boot) |
|--|--|--|
| **Spring context** | No Spring context needed | Requires Spring context |
| **Replaces Spring bean** | No | Yes — replaces the real bean in context |
| **Usage** | Plain JUnit tests with `@ExtendWith(MockitoExtension.class)` | `@WebMvcTest`, `@SpringBootTest` |
| **Performance** | Fast — no context startup | Slow — needs context startup |

```java
// @Mock — no Spring context
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository orderRepository;  // Pure Mockito mock
    @InjectMocks OrderService orderService;
}

// @MockBean — Spring context, real bean replaced
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @MockBean OrderService orderService;  // Spring bean replaced with mock
    @Autowired MockMvc mockMvc;
}
```

---

**Q4: What is @DynamicPropertySource and why is it needed for TestContainers?**

`@DynamicPropertySource` is a Spring Boot annotation that allows overriding application properties at test startup time with values that are only known at runtime.

TestContainers starts a container on a **randomly assigned port** — you can't know the port at compile time. `@DynamicPropertySource` runs after the container starts but before the Spring context initializes, allowing you to inject the actual JDBC URL, port, and credentials into the Spring context.

```java
@DynamicPropertySource
static void configureProps(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgres::getJdbcUrl);
    registry.add("spring.datasource.username", postgres::getUsername);
    registry.add("spring.datasource.password", postgres::getPassword);
}
```

Spring Boot 3.1+ simplifies this further with `@ServiceConnection`.

---

**Q5: What is the difference between a static @Container and a non-static @Container?**

```java
// Static: ONE container for all test methods in the class
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
// Container starts once before the first test, stops after the last test

// Non-static (instance): NEW container for EACH test method
@Container
PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
// Container starts before every @Test method, stops after every @Test method
```

**Performance impact:** If a PostgreSQL container takes 5 seconds to start and you have 20 test methods:
- Static: 5 seconds total startup time
- Non-static: 100 seconds total startup time

**Trade-off:** Static containers require tests to clean up their own data (use `@BeforeEach` deleteAll, or `@Transactional`). Non-static containers give perfect test isolation at a massive performance cost.

**Rule of thumb:** Always use static unless you have a specific reason not to.

---

**Q6: Why is TestContainers better than H2 in-memory database for integration tests?**

| Concern | H2 | TestContainers (PostgreSQL) |
|---------|----|-----------------------------|
| SQL compatibility | H2 dialect — different from PostgreSQL | Exact PostgreSQL behavior |
| JSON column support | Limited | Full PostgreSQL JSON/JSONB |
| Array types | Not supported | Fully supported |
| Window functions | Partial | Full support |
| Custom functions | Not available | Works |
| Stored procedures | Limited | Works |
| False confidence | High (passes in H2, fails in prod) | Low (same as production) |
| CI/CD requirement | None (runs anywhere) | Docker required |
| Speed | Faster | Slightly slower (container start) |

**Bottom line:** H2 tests can pass while the same query fails on production PostgreSQL. TestContainers gives you genuine confidence because the test database behaves identically to production.

---

**Q7: What is @ServiceConnection in Spring Boot 3.1+?**

`@ServiceConnection` is a Spring Boot 3.1+ annotation that eliminates boilerplate `@DynamicPropertySource` code. Spring Boot automatically detects the container type (e.g., `PostgreSQLContainer`) and wires the appropriate properties (`spring.datasource.url`, etc.) without any manual configuration.

```java
// Before Spring Boot 3.1 (manual wiring needed):
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

@DynamicPropertySource
static void props(DynamicPropertyRegistry reg) {
    reg.add("spring.datasource.url", postgres::getJdbcUrl);
    reg.add("spring.datasource.username", postgres::getUsername);
    reg.add("spring.datasource.password", postgres::getPassword);
}

// After Spring Boot 3.1 (automatic wiring):
@Container
@ServiceConnection
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
// That's it. Spring Boot does the rest.
```

---

**Q8: What is consumer-driven contract testing?**

Consumer-driven contract testing (CDC) is a pattern where:
1. The **consumer** (service A, which calls service B) defines the contract — what it expects from service B's API
2. The contract is shared with the **provider** (service B)
3. The provider runs tests to verify its actual implementation satisfies the consumer's contract
4. If the provider breaks the contract, tests fail — catching integration issues without both services needing to run simultaneously

"Consumer-driven" means the consumer has the authority to define what it needs. The provider cannot break what consumers rely on without being notified.

---

**Q9: What is Pact? How does it work?**

Pact is the most widely used consumer-driven contract testing framework. It works as follows:

1. **Consumer side:** Write a Pact test that defines interactions (requests + expected responses) using `PactDslWithProvider`. The test runs against a Pact-provided mock server. If the test passes, Pact generates a pact file (JSON).

2. **Contract sharing:** The pact file is published to a Pact Broker (or stored in version control).

3. **Provider side:** Write a provider verification test. Pact reads the pact file, sends the defined requests to the real running provider, and compares responses against the expected contract. If the provider's response doesn't match, verification fails.

4. **Can I Deploy:** The Pact Broker can tell you if a new version of either service is safe to deploy to production, based on verification results.

---

**Q10: What is the Pact Broker?**

The Pact Broker is a central service for storing and sharing pact files between consumer and provider teams. Key features:
- Consumers publish pacts to it
- Providers read pacts from it and publish verification results
- Provides a dashboard showing which consumer-provider combinations are compatible
- **Can I Deploy** API: programmatically check if a version is safe to deploy
- Tracks versions and environments

It decouples consumer and provider teams — consumers publish once, providers verify independently.

---

**Q11: What is the difference between contract testing and integration testing?**

| | Contract Testing | Integration Testing |
|--|--|--|
| **Both services running?** | No — each side tests independently | Yes — both must run together |
| **Speed** | Fast (milliseconds–seconds) | Slow (seconds–minutes) |
| **Scope** | Only the API contract (request/response shape) | Full business logic and data flow |
| **Purpose** | Verify interface agreement | Verify the integrated behavior |
| **Tools** | Pact | TestContainers, WireMock, @SpringBootTest |
| **Feedback** | Early — catches breaking changes fast | Later — catches actual runtime bugs |

**They are complementary, not alternatives.** Use contract tests to catch API incompatibilities early and integration tests to verify correctness of the full interaction.

---

**Q12: What is WireMock? When would you use it?**

WireMock is a mock HTTP server that intercepts HTTP calls and returns configured stub responses. Use it when:
- Your service calls an external API (payment gateway, email service, SMS provider) and you cannot or don't want to call the real API in tests
- You want to test error handling: what happens when the external API returns 500, or times out
- You want to test retry logic (use WireMock scenarios to simulate first call fails, second succeeds)
- You want to verify that your code made the correct HTTP request (WireMock records and lets you verify calls)

**WireMock vs TestContainers:** TestContainers provides real services (real PostgreSQL, real Kafka). WireMock provides fake HTTP services. Use TestContainers for your own infrastructure, WireMock for third-party HTTP APIs.

---

**Q13: What is REST Assured?**

REST Assured is a Java DSL for testing REST APIs in a BDD (Given/When/Then) style. It provides:
- Fluent API for building HTTP requests
- Powerful response assertions (status code, JSON path, headers, body)
- JSON Schema validation
- Authentication helpers (Bearer, OAuth2, Basic)
- Request/response specification reuse

```java
given()
    .contentType(ContentType.JSON)
    .body(requestBody)
.when()
    .post("/api/orders")
.then()
    .statusCode(201)
    .body("status", equalTo("PENDING"));
```

It is used with `@SpringBootTest(webEnvironment = RANDOM_PORT)` for full-stack HTTP tests.

---

**Q14: What is the difference between @WebMvcTest and @SpringBootTest?**

| | `@WebMvcTest` | `@SpringBootTest` |
|--|--|--|
| **Context loaded** | Only web layer (controllers, filters, security) | Full application context |
| **Services/Repos** | Not loaded — must be `@MockBean` | Loaded (all beans) |
| **Database** | Not connected | Connected (if configured) |
| **Speed** | Fast | Slow |
| **Use case** | Test controller logic, request mapping, validation, HTTP responses | End-to-end integration tests |
| **MockMvc** | Auto-configured | Need `@AutoConfigureMockMvc` |

---

**Q15: What is @DataJpaTest?**

`@DataJpaTest` is a test slice for JPA repositories. It:
- Loads only JPA-related beans: `@Entity`, `@Repository`, JPA configuration
- Configures an **embedded H2 by default** (override with `@AutoConfigureTestDatabase(replace = NONE)`)
- Provides `TestEntityManager` for test data management
- Wraps each test in a transaction that is rolled back by default

Use it to test custom JPA queries, repository methods, entity relationships, and database-level constraints — without loading controllers, services, or the full Spring context.

---

**Q16: What is BDD? What is Cucumber?**

**BDD (Behavior-Driven Development)** is an approach where tests are written in plain English describing system behavior, making them readable by non-developers (QA, business analysts, product owners). Tests serve as living documentation.

**Cucumber** is a BDD framework that:
- Reads **feature files** written in **Gherkin** syntax (Given/When/Then)
- Maps each Gherkin step to a Java **step definition** method
- Runs as JUnit tests

Benefits:
- Business stakeholders can review test scenarios
- Tests describe *what* the system does, not *how*
- Scenarios serve as acceptance criteria

Drawbacks:
- Slower to write (need feature files + step definitions)
- Over-abstraction can make debugging hard
- Not suitable for unit tests (overkill)

---

**Q17: How do you test a Kafka consumer in Spring Boot?**

**Option 1: TestContainers Kafka (recommended for real-world behavior)**
```java
@Container
static KafkaContainer kafka = new KafkaContainer(
    DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

@DynamicPropertySource
static void kafkaProps(DynamicPropertyRegistry reg) {
    reg.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
}

@Test
void testConsumer() {
    kafkaTemplate.send("orders", message);
    await().atMost(30, SECONDS).untilAsserted(() ->
        verify(consumer).handleMessage(any()));
}
```

**Option 2: @EmbeddedKafka (faster, in-process)**
```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"orders"})
class ConsumerTest {
    @Autowired KafkaTemplate<String, String> template;
    // Send messages and assert using Awaitility
}
```

**Key points:**
- Kafka consumers are async — always use Awaitility, never `Thread.sleep()`
- Set `auto-offset-reset=earliest` so the consumer reads from the beginning of the topic
- Use a unique consumer group ID per test run to avoid offset interference: `${random.uuid}`

---

**Q18: What is the testing honeycomb vs. testing pyramid? Which applies to microservices?**

The **pyramid** (Cohn, 2009) was designed for monolithic applications:
- Many unit tests at the base
- Fewer integration tests in the middle
- Very few E2E tests at the top

The **honeycomb** (Spotify, for microservices):
- Unit tests still exist, but are fewer (services are small)
- Integration tests are the dominant layer (test the service with real infrastructure)
- Contract tests replace many E2E tests
- Very few E2E tests remain

For microservices, the honeycomb is more appropriate because mocking all the interactions between microservices in unit tests creates brittle, inaccurate tests. Integration tests with TestContainers give more confidence in a microservices context.

---

**Q19: How do you test asynchronous code in Spring Boot?**

Never use `Thread.sleep()` — it is slow and unreliable. Use **Awaitility**:

```java
await()
    .atMost(Duration.ofSeconds(30))
    .pollInterval(Duration.ofMillis(500))
    .untilAsserted(() -> {
        // Your assertion here — will be retried until it passes or timeout
        assertThat(repository.findById(1L)).isPresent();
    });
```

For `CompletableFuture`-based async operations:
```java
CompletableFuture<Result> future = service.processAsync(input);
Result result = future.get(5, TimeUnit.SECONDS);  // Block with timeout
assertThat(result).isNotNull();
```

---

**Q20: What is the Singleton container pattern in TestContainers?**

The Singleton pattern shares one container across multiple test classes in the same JVM run. Instead of each `@SpringBootTest` class starting its own container, all test classes use a single, shared container:

```java
// Abstract base class
public abstract class AbstractIntegrationTest {
    static final PostgreSQLContainer<?> POSTGRES;

    static {
        POSTGRES = new PostgreSQLContainer<>("postgres:15").withReuse(true);
        POSTGRES.start();  // Only starts if not already running
    }

    @DynamicPropertySource
    static void overrideProps(DynamicPropertyRegistry reg) {
        reg.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        // ...
    }
}
```

This is the most efficient approach for large test suites — one container startup for the entire test run.

---

**Q21: What wait strategies does TestContainers provide?**

TestContainers provides several strategies to determine when a container is ready:

- `Wait.forListeningPort()` — waits until the exposed port accepts TCP connections
- `Wait.forHttp("/health")` — waits until an HTTP endpoint returns a specific status code
- `Wait.forLogMessage(".*ready.*", 1)` — waits for a specific string in container logs
- `Wait.forHealthcheck()` — uses Docker's built-in HEALTHCHECK directive
- Custom strategies by extending `AbstractWaitStrategy`

Always configure a wait strategy to prevent tests from failing because the container wasn't ready when the first test ran.

---

**Q22: What is @Testcontainers and @Container?**

`@Testcontainers` is a JUnit 5 extension (`@ExtendWith(TestcontainersExtension.class)`) that:
- Scans the test class for `@Container`-annotated fields
- Starts containers before tests run
- Stops containers after tests finish
- Respects static vs instance scoping

`@Container` marks a field as a TestContainers container that should be managed by the extension.

Without these annotations, you can also manage containers manually (call `.start()` and `.stop()` explicitly).

---

**Q23: How do you configure Flyway migrations to run in tests with TestContainers?**

Flyway runs automatically when the Spring context starts, as long as:
1. `spring.flyway.enabled=true` (usually the default)
2. Migration scripts are in `src/main/resources/db/migration`
3. The datasource is configured (which TestContainers provides via `@DynamicPropertySource` or `@ServiceConnection`)

The container starts fresh → Spring context starts → Flyway detects an empty database → runs all migrations → tests run against a fully migrated schema.

For test-only data, you can use test-specific migration scripts:
```properties
spring.flyway.locations=classpath:db/migration,classpath:db/testdata
```

---

**Q24: What is provider state in Pact testing?**

A provider state is a precondition that the provider must set up before Pact verifies an interaction. For example, a consumer writes a contract that says "given product with ID 1 exists, when I call GET /products/1, I get a 200 response."

The "given product with ID 1 exists" is the provider state. On the provider side, a `@State` method sets up this data:

```java
@State("product with ID 1 exists")
void setUpProduct() {
    productRepository.save(new Product(1L, "Laptop", 999.99));
}
```

Provider states decouple the consumer's contract definition from the provider's data setup.

---

**Q25: How do you use @AutoConfigureTestDatabase(replace = NONE) and why?**

By default, `@DataJpaTest` replaces your configured datasource with an embedded H2 database. `replace = AutoConfigureTestDatabase.Replace.NONE` disables this replacement, allowing TestContainers to provide the real database:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class MyRepositoryTest {
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    // Now uses real PostgreSQL, not H2
}
```

This is essential when your repository uses PostgreSQL-specific features that H2 doesn't support.

---

**Q26: What is the difference between @SpyBean and @MockBean?**

| | `@MockBean` | `@SpyBean` |
|--|--|--|
| **Based on** | Complete mock — no real behavior | Wraps the real Spring bean |
| **Default behavior** | All methods return null/default | Calls the real method by default |
| **Override specific method** | Yes, with `when(...).thenReturn(...)` | Yes, with `doReturn(...).when(...)` |
| **Use case** | Replace a dependency entirely | Partially spy on a real bean (verify calls, or override one method) |

```java
// @SpyBean wraps the real bean — real methods are called unless overridden
@SpyBean
OrderEventConsumer consumer;

@Test
void testConsumerIsCalledWithKafkaMessage() {
    // consumer.handleMessage() will call the REAL implementation
    // but we can verify it was called:
    verify(consumer, times(1)).handleMessage(any());
}
```

---

**Q27: How do you test WireMock retry behavior?**

Use WireMock scenarios to simulate stateful behavior:

```java
// Set up a scenario where:
// - First call returns 503 (Service Unavailable)
// - Second call returns 200 (Success)
wireMock.stubFor(post(urlEqualTo("/api/pay"))
    .inScenario("Retry")
    .whenScenarioStateIs(STARTED)
    .willReturn(aResponse().withStatus(503))
    .willSetStateTo("retried"));

wireMock.stubFor(post(urlEqualTo("/api/pay"))
    .inScenario("Retry")
    .whenScenarioStateIs("retried")
    .willReturn(aResponse().withStatus(200).withBody("{\"status\":\"OK\"}")));

// If your client retries once on 503, this test verifies the retry succeeds
PaymentResult result = paymentClient.pay(request);
assertThat(result.getStatus()).isEqualTo("OK");
wireMock.verify(2, postRequestedFor(urlEqualTo("/api/pay")));
```

---

**Q28: What is REST Assured's JSON path syntax?**

REST Assured uses the JsonPath library. Key syntax:

```java
.body("id", equalTo(1))                        // Root field
.body("address.city", equalTo("London"))       // Nested field
.body("items", hasSize(3))                     // Array size
.body("items[0].productId", equalTo(1))        // Array element
.body("items.productId", hasItems(1, 2))       // Values in array
.body("items.findAll { it.qty > 1 }.name",     // Groovy expression
    hasItem("Laptop"))
```

---

**Q29: When should you NOT use TestContainers?**

- **Pure unit tests:** TestContainers adds startup overhead. For testing business logic with no DB interaction, use Mockito mocks.
- **Very fast CI pipelines where Docker is unavailable:** Some minimal CI setups don't have Docker (though most modern CI platforms do).
- **Testing mock behavior of a collaborator:** If you're testing how your code handles a dependency failing, WireMock is more appropriate than TestContainers.
- **Testing pure algorithms:** No external dependencies = no need for TestContainers.

---

**Q30: What is the TestContainers Ryuk container?**

Ryuk is a resource reaper container that TestContainers starts automatically in the background. Its job is to clean up containers, networks, and volumes that were created by TestContainers — even if the JVM crashes or the test process is killed unexpectedly.

When TestContainers starts, it also starts a Ryuk container. Your test containers register with Ryuk. When the JVM exits (or Ryuk detects the parent process died), it stops and removes all registered containers.

This ensures no dangling Docker containers are left running after tests finish.

---

**Q31: What is .withReuse(true) in TestContainers and what is required to use it?**

`.withReuse(true)` tells TestContainers to reuse an already-running container from a previous test run (within the same developer session) rather than starting a fresh one.

**Requirements:**
1. Add to `~/.testcontainers.properties`: `testcontainers.reuse.enable=true`
2. The container configuration hash must match between runs

**How it works:**
- First run: starts container, labels it with a config hash
- Subsequent runs: finds the existing container with the same hash, reuses it
- Container is never stopped automatically (developer must stop it manually)

This dramatically speeds up repeated local test runs.

---

**Q32: How do you write a component test for a microservice?**

A component test tests the entire microservice in isolation — all its internal layers (controller, service, repository, database) but with external dependencies stubbed.

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
class OrderServiceComponentTest {

    // Own database — real
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    // Own Kafka — real
    @Container
    @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

    // External services — stubbed with WireMock
    WireMockServer inventoryService;
    WireMockServer paymentService;

    @BeforeEach
    void setUp() {
        inventoryService = new WireMockServer(8091);
        paymentService = new WireMockServer(8092);
        inventoryService.start();
        paymentService.start();

        // Stub: inventory says product is in stock
        inventoryService.stubFor(get(urlEqualTo("/api/inventory/1"))
            .willReturn(aResponse().withStatus(200)
                .withBody("{\"productId\": 1, \"available\": true}")));
    }

    @Test
    void shouldCreateOrderAndPublishEvent() {
        // Full flow: HTTP request → Controller → Service → Repository → Kafka event
        given()
            .contentType(ContentType.JSON)
            .body("{\"customerId\": \"CUST-001\", \"productId\": 1, \"quantity\": 2}")
        .when()
            .post("/api/orders")
        .then()
            .statusCode(201)
            .body("status", equalTo("PENDING"));

        // Verify event was published to Kafka
        await().atMost(30, SECONDS).untilAsserted(() ->
            assertThat(consumeKafkaMessage("orders")).isNotNull());
    }
}
```

---

**Q33: How do you handle database cleanup between tests when using a shared container?**

**Option 1: deleteAll() in @BeforeEach**
```java
@BeforeEach
void cleanDatabase() {
    // Clean in reverse foreign key order
    orderItemRepository.deleteAll();
    orderRepository.deleteAll();
    customerRepository.deleteAll();
}
```

**Option 2: @Transactional rollback**
```java
@Transactional  // Each test rolls back, no persistent data
class MyTest { ... }
// Warning: doesn't test commit behavior; avoid for Kafka/event tests
```

**Option 3: @Sql cleanup**
```java
@Sql(scripts = "/sql/cleanup.sql",
     executionPhase = BEFORE_TEST_METHOD)
void myTest() { ... }
```

**Option 4: TRUNCATE with restart identity (PostgreSQL)**
```sql
-- cleanup.sql
TRUNCATE TABLE order_items, orders, customers RESTART IDENTITY CASCADE;
```

---

**Q34: What is the difference between @SpringBootTest with MOCK and RANDOM_PORT?**

| | `MOCK` (default) | `RANDOM_PORT` |
|--|--|--|
| **Server** | No real HTTP server — simulated servlet environment | Real HTTP server starts on random port |
| **Client** | `MockMvc` — calls controllers directly in-process | `TestRestTemplate`, REST Assured — real HTTP calls |
| **Speed** | Faster (no network stack) | Slightly slower (HTTP overhead) |
| **Realism** | Less realistic (no HTTP serialization) | More realistic (full HTTP lifecycle) |
| **Use for** | Testing Spring MVC layer | Full API integration tests |

Use `MOCK` with `MockMvc` when testing Spring MVC behavior (filters, interceptors, exception handlers). Use `RANDOM_PORT` when you need to test the actual HTTP protocol behavior (serialization, headers, redirects) or when using REST Assured.

---

**Q35: What is TestEntityManager in @DataJpaTest?**

`TestEntityManager` is a Spring Boot-provided wrapper around JPA's `EntityManager`, designed for test use. It provides convenient methods for test data setup:

```java
@Autowired
TestEntityManager entityManager;

// persistAndFlush: saves to DB and flushes immediately (forces DB write)
User user = entityManager.persistAndFlush(new User("John", "john@example.com"));

// find: retrieve entity by primary key
User found = entityManager.find(User.class, user.getId());

// refresh: reload entity state from DB (useful after update queries)
entityManager.refresh(user);

// persistFlushFind: persist, flush, clear first-level cache, then find
// Guarantees the entity is loaded fresh from DB
User fresh = entityManager.persistFlushFind(new User("Jane", "jane@test.com"));
```

`persistFlushFind` is especially useful when you want to verify that data was actually persisted to the database, not just cached in the first-level cache.

---

**Q36: How would you test a Spring @Scheduled method?**

Scheduled methods run automatically on a timer — in tests, you usually don't want to wait for the schedule. You test them by calling them directly:

```java
@Service
public class OrderCleanupService {
    @Scheduled(cron = "0 0 * * * *")  // Every hour
    public void cleanupExpiredOrders() {
        orderRepository.deleteExpired(LocalDateTime.now().minusDays(30));
    }
}

// In test — call directly, don't wait for the schedule
@SpringBootTest
@Testcontainers
class OrderCleanupServiceTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Autowired
    OrderCleanupService cleanupService;

    @Autowired
    OrderRepository orderRepository;

    @Test
    void shouldDeleteOrdersOlderThan30Days() {
        // Arrange: seed old and recent orders
        orderRepository.save(new Order(..., LocalDateTime.now().minusDays(31)));
        orderRepository.save(new Order(..., LocalDateTime.now().minusDays(5)));

        // Act: call the scheduled method directly
        cleanupService.cleanupExpiredOrders();

        // Assert
        assertThat(orderRepository.count()).isEqualTo(1);
    }
}
```

---

## Quick Reference Summary

### TestContainers Cheat Sheet

```java
// PostgreSQL
@Container
@ServiceConnection  // Spring Boot 3.1+
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

// Kafka
@Container
@ServiceConnection
static KafkaContainer kafka = new KafkaContainer(
    DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

// MongoDB
@Container
@ServiceConnection
static MongoDBContainer mongo = new MongoDBContainer("mongo:6.0");

// Redis (no module — use GenericContainer)
@Container
static GenericContainer<?> redis = new GenericContainer<>("redis:7.0")
    .withExposedPorts(6379)
    .waitingFor(Wait.forLogMessage(".*Ready to accept connections.*", 1));

// Shared container base class
public abstract class AbstractIntegrationTest {
    static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:15")
        .withReuse(true);
    static { POSTGRES.start(); }
}
```

### Testing Slice Quick Reference

| What to test | Slice to use | Loads |
|---|---|---|
| HTTP endpoints, request/response | `@WebMvcTest` | Controllers only |
| JPA repository queries | `@DataJpaTest` | JPA layer only |
| HTTP client code | `@RestClientTest` | HTTP client only |
| Service layer (no HTTP, no DB) | `@ExtendWith(MockitoExtension.class)` | Nothing (pure unit) |
| Full integration | `@SpringBootTest(RANDOM_PORT)` | Everything |

### Pact Workflow

```
Consumer writes pact test
    → pact file generated (target/pacts/)
        → publish to Pact Broker (mvn pact:publish)
            → Provider reads pact from Broker
                → Provider runs verification test
                    → Verification result published to Broker
                        → "Can I Deploy?" check in CI/CD pipeline
```

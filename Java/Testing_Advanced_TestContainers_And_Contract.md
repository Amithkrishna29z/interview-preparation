# Testing Advanced: TestContainers, Contract Testing & Integration Testing
## Full Stack Java Developer Interview Preparation

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

**Awareness (senior-level tooling):** JaCoCo is the standard Java coverage tool, wired in via the `jacoco-maven-plugin` (goals `prepare-agent` + `report`, with an optional `check` goal to fail the build below a threshold like 0.80). You can exclude packages (e.g. `**/dto/**`, `**/config/**`) from the report. A junior just needs to know coverage exists, what line vs. branch coverage means, and that 80% is a guideline, not a law.

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

For faster local dev, `.withReuse(true)` keeps a container running between test runs (JVM restarts) instead of starting a fresh one each time. TestContainers hashes the container config; matching hashes reuse the existing container.

```java
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
    .withReuse(true);  // Container persists between test runs
```

**Requirement:** add `testcontainers.reuse.enable=true` to `~/.testcontainers.properties`.

---

### 2.5 Other Backing Services (MySQL, MongoDB, Kafka, Redis, …)

Every backing service follows the **same pattern** as the PostgreSQL example above: pick the matching container class (or `GenericContainer` for ones without a dedicated module), start it with `@Container`, and wire its runtime host/port into Spring via `@DynamicPropertySource` (or `@ServiceConnection` on Spring Boot 3.1+).

```java
// MySQL
static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");
// MongoDB — wire mongodb::getReplicaSetUrl into spring.data.mongodb.uri
static MongoDBContainer mongo = new MongoDBContainer("mongo:6.0");
// Kafka — wire kafka::getBootstrapServers into spring.kafka.bootstrap-servers
static KafkaContainer kafka = new KafkaContainer(
    DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));
// Redis — no module, use GenericContainer
static GenericContainer<?> redis = new GenericContainer<>("redis:7.0-alpine")
    .withExposedPorts(6379);
```

TestContainers supports **hundreds** of technologies (Elasticsearch, RabbitMQ, LocalStack/AWS, etc.) — all "supported, similar pattern."

**Awareness — senior-level extras** (know they exist, no need to memorize):
- **Multiple containers on a shared `Network.newNetwork()`** with `withNetworkAliases(...)` so containers can talk to each other by DNS name.
- **Wait strategies** — `Wait.forListeningPort()`, `Wait.forHttp("/health").forStatusCode(200)`, `Wait.forLogMessage(".*ready.*", 1)` — to ensure a container is ready before tests hit it. Modules like PostgreSQL/Kafka configure sensible defaults already.
- For Kafka consumers in tests, set `auto-offset-reset=earliest` and a unique consumer `group-id` (e.g. `${random.uuid}`) so each run reads from the start without offset interference.

---

### 2.6 Spring Boot 3.1+ @ServiceConnection

Spring Boot 3.1 introduced `@ServiceConnection`, which eliminates the boilerplate `@DynamicPropertySource`. Spring detects the container type and wires the right properties automatically:

```java
@SpringBootTest
@Testcontainers
class ModernSpringBootTest {

    @Container
    @ServiceConnection  // auto-configures spring.datasource.* — no @DynamicPropertySource
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Autowired
    UserRepository userRepository;

    @Test
    void contextLoads() {
        assertThat(userRepository).isNotNull();
    }
}
```

The same `@ServiceConnection` works for Kafka, Redis, MongoDB, etc. Add the `spring-boot-testcontainers` dependency (`org.springframework.boot`) to enable it.

---

### 2.7 TestContainers with @DataJpaTest

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

### 2.8 Flyway + TestContainers

When you use Flyway, migrations in `src/main/resources/db/migration` run automatically as the Spring context starts — including in tests. Combined with TestContainers, every integration test runs against a fresh, fully migrated schema. You don't do anything special: start the container, point Spring at it (`@ServiceConnection`/`@DynamicPropertySource`), and Flyway migrates the empty container DB on startup.

---

### 2.9 Singleton Container Pattern (awareness)

For large suites where many test classes need the same database, starting a container per class is wasteful. The **Singleton pattern** starts one container in a `static {}` block of an abstract base class and has every test class extend it, so the whole JVM run shares a single container:

```java
public abstract class AbstractIntegrationTest {
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:15").withReuse(true);
    static { POSTGRES.start(); }  // Started once, shared by all subclasses
    // + @DynamicPropertySource wiring the datasource
}
```

### 2.10 Log Consumer (awareness)

To debug a failing container, stream its logs into your test logger with `.withLogConsumer(new Slf4jLogConsumer(LoggerFactory.getLogger("PostgreSQL")))`.

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

    // (Additional @Pact methods follow the same shape for 404 / list responses.)

    // Test: Consumer calls the mocked provider and verifies its own client code
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
}
```

**What happens when this test runs:**
1. Pact starts a mock server at port 9999
2. The mock server is configured with the expectations defined in `@Pact` methods
3. Your `ProductClient` calls the mock server (not the real ProductService)
4. Pact verifies that the client made the right call
5. Pact generates a JSON file: `target/pacts/OrderService-ProductService.json` — it records the consumer, provider, provider states, and each request/response interaction. This file is the contract the provider later verifies against.

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

    // Provider states: set up the data each interaction needs (one @State per "given")
    @State("a product with ID 1 exists")
    void productWithId1Exists() {
        productRepository.deleteAll();
        productRepository.save(new Product(1L, "Laptop", new BigDecimal("999.99"),
            "Electronics", true));
    }
    // ... one @State method per provider state declared by the consumer
}
```

---

### 3.6 Pact Broker (awareness)

The **Pact Broker** is a central service that stores pact files from consumers, serves them to providers for verification, records verification results, and answers **"Can I Deploy?"** (is consumer version X compatible with the provider currently in production?). You run it as a container (`pactfoundation/pact-broker`), publish consumer pacts to it (`mvn pact:publish`), and point the provider test at it with `@PactBroker(url = "...")` instead of `@PactFolder`. In CI, `pact-broker can-i-deploy --pacticipant ... --version ... --to-environment production` gates deployments. As a junior, just know the broker decouples the teams and enables the deploy check — the full pipeline setup is a senior/platform concern.

---

### 3.7 Message Contract Testing with Pact (awareness)

Pact also supports **asynchronous message contracts** (Kafka, RabbitMQ): the consumer declares the message body it expects to receive via `MessagePactBuilder` (with `providerType = ProviderType.ASYNCH`), and the provider verifies the messages it produces match. The pattern mirrors the HTTP flow above, just for message payloads instead of request/response.

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

### 4.4 WireMock Scenarios — Testing Retry Logic (awareness)

WireMock **scenarios** model stateful behavior so you can test retry logic: register multiple stubs for the same URL using `.inScenario("Retry").whenScenarioStateIs(STARTED)...willSetStateTo("retried")`, so the first call returns 503, the next returns 200. Then `wireMockServer.verify(N, postRequestedFor(...))` confirms your client retried the expected number of times. (See Q27 for a concrete snippet.)

---

### 4.5 @AutoConfigureWireMock (Spring Boot)

Spring Cloud Contract offers `@AutoConfigureWireMock(port = 0)` for annotation-based setup. Stubs can be programmatic (inject `WireMockServer`) or file-based: stub mappings in `src/test/resources/mappings/` reference response bodies in `src/test/resources/__files/`.

```java
@SpringBootTest
@AutoConfigureWireMock(port = 0)  // random port, injected as ${wiremock.server.port}
class ExternalApiTest {
    @Autowired WireMockServer wireMockServer;
}
```

### 4.6 WireMock Response Templating (awareness)

WireMock can generate **dynamic responses** from the request using Handlebars templating — enable with `.withTransformers("response-template")`, then reference helpers like `{{request.pathSegments.[2]}}`, `{{now}}`, or `{{randomValue type='UUID'}}` in the body.

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

    // (PUT /cancel, list-by-customer, etc. follow the same given/when/then shape.)

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

    // A @Given seeds data (note the DataTable maps to the Gherkin table)
    @Given("the product catalog has the following products:")
    public void setupProducts(DataTable dataTable) {
        for (Map<String, String> row : dataTable.asMaps()) {
            productRepository.save(new Product(
                Long.parseLong(row.get("id")), row.get("name"),
                new BigDecimal(row.get("price")), Integer.parseInt(row.get("stock"))));
        }
    }

    // A @When performs the action under test
    @When("I create an order with the following items:")
    public void createOrderWithItems(DataTable dataTable) {
        List<OrderItem> items = dataTable.asMaps().stream()
            .map(row -> new OrderItem(
                Long.parseLong(row.get("productId")),
                Integer.parseInt(row.get("quantity"))))
            .collect(Collectors.toList());

        lastResponse = restTemplate.postForEntity(
            "http://localhost:" + port + "/api/orders",
            new CreateOrderRequest(items), OrderResponse.class);
    }

    // A @Then asserts the outcome
    @Then("the order status should be {string}")
    public void verifyOrderStatus(String expectedStatus) {
        OrderResponse order = (OrderResponse) lastResponse.getBody();
        assertThat(order.getStatus()).isEqualTo(expectedStatus);
    }

    // ... remaining steps (auth, totals, cancel) follow the same Given/When/Then pattern
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
    void shouldReturn400WhenRequestBodyInvalid() throws Exception {
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"customerId": "", "items": []}
                    """))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isArray());
    }
    // (404-when-not-found and 201-when-created follow the same mock-and-perform shape.)
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

A fluent builder gives tests readable, default-filled objects so each test only sets the fields it cares about:

```java
// OrderTestBuilder.java — fluent builder for test data (sensible defaults)
public class OrderTestBuilder {
    private String customerId = "CUST-DEFAULT";
    private String status = "PENDING";
    private List<OrderItem> items = new ArrayList<>();

    public static OrderTestBuilder anOrder() { return new OrderTestBuilder(); }

    public OrderTestBuilder forCustomer(String c) { this.customerId = c; return this; }
    public OrderTestBuilder completed()          { this.status = "COMPLETED"; return this; }
    public OrderTestBuilder withItem(Long productId, int qty, double price) {
        this.items.add(new OrderItem(productId, qty, BigDecimal.valueOf(price)));
        return this;
    }
    public Order build() { return new Order(null, customerId, status, items); }
    public Order buildAndSave(OrderRepository repo) { return repo.save(build()); }
}

// Usage in tests — only specify what matters:
Order order = anOrder()
    .forCustomer("CUST-001")
    .withItem(1L, 2, 49.99)
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

### 8.4 ArchUnit — Architecture Fitness Tests (awareness)

ArchUnit (`com.tngtech.archunit:archunit-junit5`) lets you write JUnit tests that assert **architecture rules** — e.g. "services must not depend on controllers", "repositories are only accessed by services", "no circular package dependencies". Rules read fluently and fail the build when violated:

```java
@ArchTest
static final ArchRule servicesShouldNotDependOnControllers =
    noClasses().that().resideInAPackage("..service..")
        .should().dependOnClassesThat().resideInAPackage("..controller..");
```

Useful on larger teams to keep layering clean; a junior just needs to know such automated architecture checks exist.

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
}
```

`await()` polls your assertion until it passes or the timeout elapses — far more reliable than guessing a `Thread.sleep()` duration. Use `.until(...)` for a boolean condition or `.untilAsserted(...)` to retry assertions.

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
    // (For many concurrent futures, combine with CompletableFuture.allOf(...).get(timeout).)
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

**Q18: Testing honeycomb vs. pyramid — which applies to microservices?**

See Q1 and Q2: the **pyramid** (Cohn, monoliths) is unit-heavy; the **honeycomb** (Spotify, microservices) makes integration tests the dominant layer with contract tests replacing many E2E tests, because heavily mocked unit tests in small services give false confidence.

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

One container shared across all test classes in a JVM run (see section 2.9): an abstract base class starts the container once in a `static {}` block and every test class extends it. It's the most efficient option for large suites — a single container startup for the whole run.

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

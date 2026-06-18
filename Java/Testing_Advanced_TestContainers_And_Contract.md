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
11. [Quick Reference Summary](#quick-reference-summary)

---

## 1. The Testing Pyramid and Strategy

### 1.1 The Classic Testing Pyramid

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

| Layer | Scope | Speed | Tools |
|-------|-------|-------|-------|
| Unit | Single class/method, all dependencies mocked | < 10ms | JUnit 5, Mockito, AssertJ |
| Integration | Multiple components, real DB/broker | 1–30s | Spring Boot Test, TestContainers, WireMock |
| E2E | Full user journey | 1–10min | Selenium, Playwright, REST Assured |

The base should be wide (many unit tests) and each layer narrows. Inverting this — more E2E than unit tests — is the **"Testing Ice Cream Cone"** anti-pattern: slow, flaky, expensive.

---

### 1.2 The Testing Honeycomb (Modern Microservices Model)

Proposed by Spotify: in microservices each service is small, so heavily mocked unit tests give false confidence. The real risk is in service interactions.

**Honeycomb priorities:**
1. **Integration tests** — test the service with real dependencies (TestContainers)
2. **Contract tests** — verify service-to-service agreements (Pact)
3. **Unit tests** — still used for complex business logic
4. **E2E tests** — very few, only for critical flows

**Rule of thumb:** use the pyramid for monoliths, the honeycomb for microservices.

---

### 1.3 Test Classification

| Test Type | Scope | Dependencies | Speed | Count |
|-----------|-------|-------------|-------|-------|
| Unit | Single class/method | All mocked | < 10ms | Thousands |
| Integration | Multiple layers | Real DB, etc. | 1–30s | Hundreds |
| Component | Whole microservice | External stubs | 5–60s | Tens |
| Contract | API agreement only | Mock server | < 5s | Tens–Hundreds |
| E2E | Full user journey | Full system | 1–10min | Handful |

---

### 1.4 Test Coverage

**Types:** line coverage (% of lines executed), branch coverage (% of if/else branches taken — more meaningful), method coverage, class coverage.

**The 80% guideline:** 80% line coverage is a common target, not a law. Focus coverage on business logic, edge cases, and error handling. Don't chase coverage on auto-generated code, DTOs, or simple getters/setters. JaCoCo is the standard Java tool, wired in via `jacoco-maven-plugin`.

---

## 2. TestContainers — Deep Dive

### 2.1 What is TestContainers?

TestContainers is a Java library that provides lightweight, throwaway instances of databases, message brokers, and anything else that runs in Docker — for use in tests.

**The problem it solves:** Before TestContainers, integration tests used in-memory alternatives like H2 instead of PostgreSQL. H2's SQL dialect differs from PostgreSQL — some queries pass in H2 but fail in production (or vice versa). PostgreSQL-specific features (JSON columns, array types, window functions) don't work in H2.

**Why TestContainers is better:**
- Tests run against the exact same database version as production
- All SQL features, constraints, and stored procedures work correctly
- Container starts fresh each run — clean state guaranteed
- Works in any environment with Docker (including CI/CD)

**How it works:** TestContainers talks to the Docker daemon, pulls the specified image, starts the container, maps container ports to random host ports, and uses a Ryuk container to clean up automatically after tests finish.

---

### 2.2 Maven Dependencies

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers-bom</artifactId>
            <version>1.19.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- kafka, mongodb modules follow same pattern -->
</dependencies>
```

---

### 2.3 PostgreSQL TestContainer with Spring Boot

```java
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {

    // static = shared container across ALL tests in this class (started once)
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("testuser")
        .withPassword("testpass");

    // Overrides Spring datasource properties with the container's runtime values
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    UserRepository userRepository;

    @BeforeEach
    void cleanUp() {
        userRepository.deleteAll();
    }

    @Test
    void shouldSaveAndFindUser() {
        User saved = userRepository.save(new User(null, "John Doe", "john@example.com"));
        assertThat(saved.getId()).isNotNull();

        Optional<User> found = userRepository.findByEmail("john@example.com");
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John Doe");
    }
}
```

**Key annotations:**

| Annotation | Purpose |
|-----------|---------|
| `@Testcontainers` | JUnit 5 extension that manages container lifecycle |
| `@Container` (static) | One container shared across all tests in the class |
| `@Container` (instance) | New container per test method — very slow |
| `@DynamicPropertySource` | Overrides Spring properties at startup with container values |

**Why `static` is preferred:** A PostgreSQL container takes 3–8s to start. 50 test methods × non-static = 50 startups. Static = 1 startup. Trade-off: tests must not leave dirty data — use `@BeforeEach` cleanup or `@Transactional`.

---

### 2.4 Spring Boot 3.1+ @ServiceConnection

`@ServiceConnection` eliminates `@DynamicPropertySource` boilerplate. Spring auto-detects the container type and wires properties automatically.

```java
@SpringBootTest
@Testcontainers
class ModernSpringBootTest {

    @Container
    @ServiceConnection  // auto-configures spring.datasource.* — no @DynamicPropertySource needed
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Autowired
    UserRepository userRepository;
}
```

Works for Kafka, Redis, MongoDB, etc. Requires `org.springframework.boot:spring-boot-testcontainers` dependency.

---

### 2.5 Other Backing Services

Every backing service follows the same pattern — pick the matching container class, start with `@Container`, wire via `@ServiceConnection` or `@DynamicPropertySource`:

```java
// MySQL
static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");

// Kafka
static KafkaContainer kafka = new KafkaContainer(
    DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

// MongoDB
static MongoDBContainer mongo = new MongoDBContainer("mongo:6.0");

// Redis — no dedicated module, use GenericContainer
static GenericContainer<?> redis = new GenericContainer<>("redis:7.0-alpine")
    .withExposedPorts(6379);
```

---

### 2.6 @DataJpaTest with TestContainers

`@DataJpaTest` loads only the JPA layer. By default it uses H2 — override this to use a real database:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // don't replace with H2
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
        entityManager.persistAndFlush(new Order(null, 1L, "PENDING", BigDecimal.TEN));
        entityManager.persistAndFlush(new Order(null, 2L, "COMPLETED", BigDecimal.valueOf(50)));

        List<Order> pendingOrders = orderRepository.findByStatus("PENDING");

        assertThat(pendingOrders).hasSize(1);
    }
}
```

---

### 2.7 Other Patterns (awareness)

- **Flyway + TestContainers:** Flyway migrations run automatically when the Spring context starts. Point Spring at the container and Flyway migrates the empty DB before tests run — no special setup needed.
- **Singleton Container Pattern:** For large suites, start one container in a `static {}` block of an abstract base class and extend it across all test classes — one container for the whole JVM run.
- **`.withReuse(true)`:** Keeps the container running between test runs (local dev). Requires `testcontainers.reuse.enable=true` in `~/.testcontainers.properties`.

---

## 3. Pact — Consumer-Driven Contract Testing

### 3.1 The Problem Contract Testing Solves

In microservices, service A (the **consumer**) calls service B (the **provider**). If B changes its API, A breaks — but this is only discovered when both are deployed together. Contract testing catches this early, independently, without both services running.

**Consumer-Driven Contracts (CDC) flow:**
```
1. Consumer writes a pact test defining expected requests/responses
2. Pact generates a pact file (JSON contract)
3. Pact file published to Pact Broker (or checked into git)
4. Provider reads the pact file and verifies its implementation matches
5. If provider breaks the contract → verification fails → fix required before deploy
```

---

### 3.2 Maven Dependencies

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

### 3.3 Consumer Test

```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "ProductService", port = "9999")
class ProductServiceConsumerPactTest {

    @Pact(consumer = "OrderService")
    public RequestResponsePact getExistingProductById(PactDslWithProvider builder) {
        return builder
            .given("a product with ID 1 exists")             // provider state
            .uponReceiving("a GET request for product 1")
                .path("/api/products/1")
                .method("GET")
            .willRespondWith()
                .status(200)
                .body(new PactDslJsonBody()
                    .integerType("id", 1)
                    .stringType("name", "Laptop")
                    .decimalType("price", 999.99))
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "getExistingProductById")
    void shouldFetchProductById(MockServer mockServer) {
        ProductClient client = new ProductClient(mockServer.getUrl());
        ProductDto product = client.getById(1L);

        assertThat(product.getId()).isEqualTo(1L);
        assertThat(product.getName()).isEqualTo("Laptop");
    }
}
```

When the test runs, Pact starts a mock server at port 9999, your `ProductClient` calls it, and Pact generates `target/pacts/OrderService-ProductService.json` — the contract file.

---

### 3.4 Provider Verification

```java
@Provider("ProductService")
@PactFolder("pacts")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductServiceProviderPactTest {

    @LocalServerPort
    private int port;

    @Autowired
    ProductRepository productRepository;

    @BeforeEach
    void setUp(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.verifyInteraction();
    }

    // Set up data matching the consumer's "given" clause
    @State("a product with ID 1 exists")
    void productWithId1Exists() {
        productRepository.deleteAll();
        productRepository.save(new Product(1L, "Laptop", new BigDecimal("999.99"), "Electronics", true));
    }
}
```

---

### 3.5 Pact Broker (awareness)

The **Pact Broker** is a central service that stores pact files, records verification results, and answers **"Can I Deploy?"** — is this consumer/provider version safe to release? Consumers publish pacts with `mvn pact:publish`; providers point at the broker with `@PactBroker(url = "...")` instead of `@PactFolder`. As a junior, know that the broker decouples teams and enables automated deploy gates — setup is a senior/platform concern.

---

## 4. WireMock — HTTP Service Virtualization

### 4.1 What is WireMock?

WireMock is a mock HTTP server. When your service calls an external HTTP API (payment gateway, email service, another microservice), WireMock intercepts those calls and returns predefined responses.

**WireMock vs Pact:**
- **WireMock:** One-sided stub — you define what the mock returns, but there's no verification that the real provider matches the stub.
- **Pact:** Two-sided contract — consumer defines expectations, provider proves it satisfies them.

---

### 4.2 Maven Dependency

```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <version>3.3.1</version>
    <scope>test</scope>
</dependency>
```

---

### 4.3 WireMock Example

```java
@SpringBootTest
class PaymentServiceIntegrationTest {

    static WireMockServer wireMockServer;

    @BeforeAll
    static void startWireMock() {
        wireMockServer = new WireMockServer(WireMockConfiguration.options().port(8089));
        wireMockServer.start();
    }

    @AfterAll
    static void stopWireMock() { wireMockServer.stop(); }

    @BeforeEach
    void resetStubs() { wireMockServer.resetAll(); }

    @Autowired
    PaymentGatewayClient paymentClient;

    @Test
    void shouldProcessPaymentSuccessfully() {
        wireMockServer.stubFor(
            post(urlEqualTo("/api/payments"))
                .withHeader("Content-Type", equalTo("application/json"))
                .willReturn(aResponse()
                    .withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                        {"transactionId": "TXN-12345", "status": "SUCCESS"}
                        """))
        );

        PaymentResult result = paymentClient.processPayment(
            new PaymentRequest("100.00", "USD", "card_token_abc"));

        assertThat(result.getTransactionId()).isEqualTo("TXN-12345");
        assertThat(result.getStatus()).isEqualTo("SUCCESS");

        wireMockServer.verify(
            postRequestedFor(urlEqualTo("/api/payments"))
                .withHeader("Content-Type", equalTo("application/json"))
        );
    }

    @Test
    void shouldHandlePaymentGatewayTimeout() {
        wireMockServer.stubFor(
            post(urlEqualTo("/api/payments"))
                .willReturn(aResponse().withFixedDelay(5000).withStatus(200))
        );

        assertThatThrownBy(() ->
            paymentClient.processPayment(new PaymentRequest("50.00", "USD", "token")))
            .isInstanceOf(PaymentTimeoutException.class);
    }
}
```

---

### 4.4 Other WireMock Features (awareness)

- **Scenarios:** Simulate stateful behavior (e.g., first call returns 503, second returns 200) for retry testing using `.inScenario(...).whenScenarioStateIs(...).willSetStateTo(...)`.
- **`@AutoConfigureWireMock(port = 0)`:** Spring Cloud Contract annotation for annotation-based setup with random port.

---

## 5. REST Assured — API Integration Testing

### 5.1 What is REST Assured?

REST Assured is a Java DSL for testing REST APIs in a BDD (Given/When/Then) style. It provides a readable syntax for making HTTP requests and asserting responses.

**Maven dependency:**
```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>5.4.0</version>
    <scope>test</scope>
</dependency>
```

---

### 5.2 REST Assured Test Example

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
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"customerId": "CUST-001", "items": [{"productId": 1, "quantity": 2, "price": 49.99}]}
                """)
        .when()
            .post("/orders")
        .then()
            .statusCode(201)
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
            .body("message", equalTo("Order not found"));
    }

    @Test
    void shouldReturn400ForInvalidRequest() {
        given()
            .contentType(ContentType.JSON)
            .body("{\"customerId\": \"\", \"items\": []}")
        .when()
            .post("/orders")
        .then()
            .statusCode(400)
            .body("errors", hasSize(greaterThan(0)));
    }
}
```

---

### 5.3 Authentication and Spec Reuse

```java
// Authentication
given()
    .header("Authorization", "Bearer " + token)
.when()
    .get("/admin/orders")
.then()
    .statusCode(200);

// OAuth2
given()
    .auth().oauth2(jwtToken)
.when()
    .get("/api/orders")
.then()
    .statusCode(200);

// Reusable RequestSpecification
RequestSpecification authSpec = new RequestSpecBuilder()
    .setBaseUri("http://localhost")
    .setPort(port)
    .setBasePath("/api")
    .setContentType(ContentType.JSON)
    .addHeader("Authorization", "Bearer " + adminToken)
    .build();
```

---

## 6. BDD with Cucumber

### 6.1 What is BDD?

Behavior-Driven Development (BDD) encourages writing tests in plain English using Gherkin syntax (Feature → Scenario → Given/When/Then). Tests serve as living documentation that business stakeholders can read.

**Maven dependencies:**
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

### 6.2 Feature File (Gherkin)

```gherkin
# src/test/resources/features/order-management.feature
Feature: Order Management

  Background:
    Given I am authenticated as a customer with email "customer@example.com"
    And the product catalog has the following products:
      | id | name   | price  | stock |
      | 1  | Laptop | 999.99 | 10    |

  Scenario: Successfully create an order
    When I create an order with the following items:
      | productId | quantity |
      | 1         | 1        |
    Then the order should be created successfully
    And the order status should be "PENDING"

  Scenario: Attempt to order out-of-stock product
    Given product 1 has only 0 items in stock
    When I try to create an order for product 1 with quantity 1
    Then the order should fail with error "Product out of stock"

  Scenario Outline: Apply discount by order total
    Given I have an order with total <total>
    When the discount is applied
    Then the discount amount should be <discount>

    Examples:
      | total   | discount |
      | 100.00  | 0.00     |
      | 500.00  | 25.00    |
      | 1000.00 | 100.00   |
```

---

### 6.3 Spring Boot Configuration and Step Definitions

```java
// CucumberSpringConfiguration.java
@CucumberContextConfiguration
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
public class CucumberSpringConfiguration {
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
}

// CucumberTestRunner.java
@Suite
@IncludeEngines("cucumber")
@SelectClasspathResource("features")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME, value = "com.example.steps")
public class CucumberTestRunner { }
```

```java
// Step definitions
@Component
public class OrderStepDefinitions {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private TestRestTemplate restTemplate;

    @LocalServerPort
    private int port;

    private ResponseEntity<?> lastResponse;

    @Given("the product catalog has the following products:")
    public void setupProducts(DataTable dataTable) {
        for (Map<String, String> row : dataTable.asMaps()) {
            productRepository.save(new Product(
                Long.parseLong(row.get("id")), row.get("name"),
                new BigDecimal(row.get("price")), Integer.parseInt(row.get("stock"))));
        }
    }

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

    @Then("the order status should be {string}")
    public void verifyOrderStatus(String expectedStatus) {
        OrderResponse order = (OrderResponse) lastResponse.getBody();
        assertThat(order.getStatus()).isEqualTo(expectedStatus);
    }
}
```

**Cucumber tags for selective execution:**
```gherkin
@smoke
Scenario: Successfully create an order
```
```java
@ConfigurationParameter(key = FILTER_TAGS_PROPERTY_NAME, value = "@smoke")
public class SmokeTestRunner { }
```

---

## 7. Spring Boot Testing Slices

### 7.1 When to Use Each Slice

`@SpringBootTest` loads the entire Spring context — useful for full integration tests but overkill for testing a single layer. Use test slices for focused, faster tests.

| Scenario | Approach | Loads |
|----------|----------|-------|
| Controller validation, HTTP mapping | `@WebMvcTest` | Web layer only |
| JPA queries, repository methods | `@DataJpaTest` + TestContainers | JPA layer only |
| HTTP client code | `@RestClientTest` | HTTP client only |
| Service layer only | JUnit + Mockito | Nothing |
| Full integration (all layers) | `@SpringBootTest(RANDOM_PORT)` + TestContainers | Everything |

---

### 7.2 @WebMvcTest — Controller Layer

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    MockMvc mockMvc;

    @MockBean  // replaces the real Spring bean with a mock
    OrderService orderService;

    @Test
    void shouldReturn200WithOrderWhenFound() throws Exception {
        Order order = new Order(1L, "CUST-001", "PENDING", BigDecimal.valueOf(99.99));
        when(orderService.findById(1L)).thenReturn(order);

        mockMvc.perform(get("/api/orders/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.status").value("PENDING"));
    }

    @Test
    void shouldReturn400WhenRequestBodyInvalid() throws Exception {
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"customerId\": \"\", \"items\": []}"))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isArray());
    }
}
```

**Loads:** `@Controller`, `@ControllerAdvice`, `MockMvc`, security config.
**Does NOT load:** `@Service`, `@Repository`, datasource, full context.

---

### 7.3 @DataJpaTest — Repository Layer

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
        entityManager.persistAndFlush(new Order(null, "CUST-001", "PENDING", BigDecimal.valueOf(100)));
        entityManager.persistAndFlush(new Order(null, "CUST-001", "COMPLETED", BigDecimal.valueOf(200)));
        entityManager.persistAndFlush(new Order(null, "CUST-002", "PENDING", BigDecimal.valueOf(50)));

        List<Order> result = orderRepository.findByCustomerIdAndStatus("CUST-001", "PENDING");

        assertThat(result).hasSize(1);
        assertThat(result.get(0).getStatus()).isEqualTo("PENDING");
    }
}
```

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
            .andRespond(withSuccess(
                """{"id": 1, "name": "Laptop", "price": 999.99}""",
                MediaType.APPLICATION_JSON));

        ProductDto product = client.getById(1L);

        assertThat(product.getName()).isEqualTo("Laptop");
        server.verify();
    }
}
```

---

### 7.5 @SpringBootTest Web Environments

```java
// MOCK (default): no real HTTP server — use MockMvc
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)

// RANDOM_PORT: real HTTP server — use TestRestTemplate or REST Assured
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)

// NONE: loads Spring context but no servlet env — for service-only tests
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
```

---

## 8. Test Data Management

### 8.1 @Sql Annotation

```java
@Test
@Sql("/sql/insert-test-orders.sql")
@Sql(scripts = "/sql/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
void shouldQueryOrdersCreatedInJanuary() {
    List<Order> orders = orderRepository.findByMonthCreated(1, 2024);
    assertThat(orders).hasSize(3);
}
```

---

### 8.2 Test Builder Pattern

A fluent builder with sensible defaults so each test only sets what it cares about:

```java
public class OrderTestBuilder {
    private String customerId = "CUST-DEFAULT";
    private String status = "PENDING";
    private List<OrderItem> items = new ArrayList<>();

    public static OrderTestBuilder anOrder() { return new OrderTestBuilder(); }
    public OrderTestBuilder forCustomer(String c) { this.customerId = c; return this; }
    public OrderTestBuilder completed() { this.status = "COMPLETED"; return this; }
    public Order build() { return new Order(null, customerId, status, items); }
}

// Usage
Order order = anOrder().forCustomer("CUST-001").build();
```

---

### 8.3 Database Cleanup Strategies

```java
// Option 1: @BeforeEach cleanup
@BeforeEach
void cleanDatabase() {
    orderItemRepository.deleteAll();
    orderRepository.deleteAll();
}

// Option 2: @Transactional rollback (fastest — no actual writes committed)
@SpringBootTest
@Transactional  // each test rolls back — don't use when testing commit behavior
class OrderServiceTest { }

// Option 3: TRUNCATE via @Sql (PostgreSQL — fast, resets sequences)
// cleanup.sql: TRUNCATE TABLE order_items, orders RESTART IDENTITY CASCADE;
```

---

## 9. Testing Asynchronous Code

### 9.1 Awaitility

Never use `Thread.sleep()` in tests — use **Awaitility** to poll until an assertion passes or a timeout expires.

```xml
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <version>4.2.0</version>
    <scope>test</scope>
</dependency>
```

```java
@Test
void shouldProcessOrderCreatedEvent() {
    producer.publishOrderCreated(new OrderCreatedEvent(42L, "CUST-001"));

    await()
        .atMost(30, SECONDS)
        .pollInterval(500, MILLISECONDS)
        .untilAsserted(() -> {
            Optional<Order> order = orderRepository.findById(42L);
            assertThat(order).isPresent();
            assertThat(order.get().getStatus()).isEqualTo("PROCESSING");
        });
}
```

---

### 9.2 Testing @Async Methods

```java
@Test
void shouldGenerateReportAsynchronously() throws Exception {
    CompletableFuture<Report> future = reportService.generateReport(1L);
    Report report = future.get(5, TimeUnit.SECONDS);  // block with timeout
    assertThat(report.getStatus()).isEqualTo("DONE");
}
```

---

### 9.3 Testing Kafka with @EmbeddedKafka

For lighter-weight Kafka tests that don't need a real broker:

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"orders"})
class OrderConsumerEmbeddedKafkaTest {

    @Autowired
    KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    OrderEventConsumer consumer;

    @Test
    void shouldConsumeOrderCreatedEvent() {
        kafkaTemplate.send("orders", "key-1", "{\"orderId\": 1, \"status\": \"CREATED\"}");

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

**Q1: What is the testing pyramid? Why does it matter?**

Three layers: unit tests (fast, many, all mocked), integration tests (real infrastructure, fewer), E2E tests (full system, very few). It guides the right proportion — mostly unit tests, some integration, very few E2E. The "Ice Cream Cone" anti-pattern (more E2E than unit) leads to slow, flaky suites.

---

**Q2: What is the testing honeycomb, and when does it apply?**

Spotify's model for microservices. It prioritizes integration tests over unit tests because each service is small with little business logic — the real risk is in service interactions. Use pyramid for monoliths, honeycomb for microservices.

---

**Q3: What is the difference between @Mock and @MockBean?**

| | `@Mock` | `@MockBean` |
|--|--|--|
| Spring context | No | Yes |
| Replaces Spring bean | No | Yes |
| Usage | Pure JUnit + `@ExtendWith(MockitoExtension.class)` | `@WebMvcTest`, `@SpringBootTest` |
| Speed | Fast | Slower (context startup) |

---

**Q4: What is @DynamicPropertySource and why is it needed for TestContainers?**

TestContainers assigns random ports at runtime — you can't hardcode them. `@DynamicPropertySource` runs after the container starts but before Spring initializes, letting you inject the actual JDBC URL and port. Spring Boot 3.1+ replaces this with `@ServiceConnection`.

---

**Q5: What is the difference between a static and non-static @Container?**

Static: one container shared across all test methods (started once). Non-static: new container per test method. With 20 tests and a 5s startup, static = 5s total vs. non-static = 100s total. Always use static unless you need perfect per-test isolation. Clean up with `@BeforeEach` deleteAll or `@Transactional`.

---

**Q6: Why is TestContainers better than H2 for integration tests?**

H2's SQL dialect differs from PostgreSQL — tests pass in H2 but fail in production. PostgreSQL-specific features (JSON columns, arrays, window functions) don't work in H2. TestContainers runs the same DB version as production, giving genuine confidence. Trade-off: requires Docker; slightly slower to start.

---

**Q7: What is @ServiceConnection in Spring Boot 3.1+?**

Eliminates `@DynamicPropertySource` boilerplate. Add `@ServiceConnection` alongside `@Container` and Spring auto-detects the container type (PostgreSQLContainer, KafkaContainer, etc.) and wires the right properties automatically.

---

**Q8: What is consumer-driven contract testing?**

The **consumer** (the service that makes calls) defines the contract — what it expects from the provider's API. The **provider** verifies its implementation satisfies the contract. Neither service needs to run simultaneously. If the provider changes its API in a breaking way, contract tests fail before deployment.

---

**Q9: What is Pact? How does it work?**

Pact is the standard CDC framework. Consumer writes a test that defines interactions against a Pact mock server → test passes → pact JSON file generated → published to Pact Broker → provider reads the pact, sends the defined requests to the real provider, verifies responses match → "Can I Deploy?" check in CI gates releases.

---

**Q10: What is the Pact Broker?**

A central service that stores pact files, records verification results, and answers "Can I Deploy?" — is this version safe to release given all consumer contracts? Decouples consumer and provider teams; each team publishes/verifies independently.

---

**Q11: What is the difference between contract testing and integration testing?**

| | Contract Testing | Integration Testing |
|--|--|--|
| Both services running? | No — independent | Yes |
| Speed | Fast (ms–seconds) | Slow (seconds–minutes) |
| Scope | API shape only | Full business logic and data flow |
| Tools | Pact | TestContainers, WireMock, @SpringBootTest |

They are complementary: contract tests catch API incompatibilities early; integration tests verify correctness.

---

**Q12: What is WireMock? When would you use it?**

A mock HTTP server for stubbing third-party HTTP APIs (payment gateway, email service). Use it to test error handling (500s, timeouts), retry logic, and to verify your code makes the correct HTTP request. Distinct from TestContainers (real services for your own infra) — WireMock is for external HTTP services you can't control.

---

**Q13: What is REST Assured?**

A Java DSL for testing REST APIs using Given/When/Then syntax. Use it with `@SpringBootTest(RANDOM_PORT)` to make real HTTP calls, assert response codes, JSON paths, headers, and validate JSON schemas.

---

**Q14: What is the difference between @WebMvcTest and @SpringBootTest?**

| | `@WebMvcTest` | `@SpringBootTest` |
|--|--|--|
| Context | Web layer only | Full application context |
| Services/Repos | Not loaded — must `@MockBean` | Loaded |
| Speed | Fast | Slow |
| Use case | Controller logic, validation | Full integration tests |

---

**Q15: What is @DataJpaTest?**

A test slice that loads only JPA-related beans (`@Entity`, `@Repository`, JPA config). Provides `TestEntityManager`. Uses H2 by default — override with `@AutoConfigureTestDatabase(replace = NONE)` + TestContainers for PostgreSQL-specific queries.

---

**Q16: What is BDD? What is Cucumber?**

BDD writes tests in plain English describing behavior, readable by non-developers. Cucumber maps Gherkin feature files (Given/When/Then steps) to Java step definition methods. Good for acceptance tests and living documentation; overkill for unit tests.

---

**Q17: How do you test a Kafka consumer in Spring Boot?**

**Option 1 (recommended):** TestContainers Kafka with `@ServiceConnection` + Awaitility to poll until the consumer processes the message. Set `auto-offset-reset=earliest` and a unique `group-id` per test run.

**Option 2 (lighter):** `@EmbeddedKafka` for in-process Kafka, still use Awaitility — never `Thread.sleep()`.

---

**Q18: How do you test asynchronous code in Spring Boot?**

Use Awaitility — never `Thread.sleep()`. Poll with `await().atMost(...).untilAsserted(() -> { your assertion })`. For `CompletableFuture`, call `.get(timeout, SECONDS)` to block with a timeout.

---

**Q19: What is the Singleton container pattern?**

Start one container in a `static {}` block of an abstract base class; all test classes extend it. One container startup for the entire JVM run — most efficient for large suites with many test classes sharing the same database.

---

**Q20: How do you configure Flyway migrations in tests with TestContainers?**

No special setup needed. Start the container, point Spring at it via `@ServiceConnection` or `@DynamicPropertySource`, and Flyway automatically migrates the empty container DB when the Spring context starts — tests run against a fully migrated schema.

---

**Q21: What is provider state in Pact testing?**

A precondition declared by the consumer ("given product with ID 1 exists"). On the provider side, a `@State` method sets up the required data before Pact verifies that interaction. It decouples the consumer's expectations from the provider's data setup.

---

**Q22: What is the difference between @SpyBean and @MockBean?**

`@MockBean` replaces the bean entirely with a mock (all methods return null/default). `@SpyBean` wraps the real bean — real methods run unless you override specific ones. Use `@SpyBean` to verify a real bean was called, or to override just one method.

---

**Q23: How do you test WireMock retry behavior?**

Use WireMock scenarios: first stub returns 503 in `STARTED` state and transitions to `"retried"` state; second stub returns 200 in `"retried"` state. Then verify WireMock received the expected number of requests with `verify(2, postRequestedFor(...))`.

---

**Q24: What is the difference between @SpringBootTest MOCK and RANDOM_PORT?**

`MOCK` creates a simulated servlet environment — use `MockMvc`. `RANDOM_PORT` starts a real HTTP server — use `TestRestTemplate` or REST Assured. MOCK is faster; RANDOM_PORT is more realistic (tests full HTTP lifecycle including serialization).

---

**Q25: What is TestEntityManager in @DataJpaTest?**

A test-friendly wrapper around JPA `EntityManager`. Key methods:
- `persistAndFlush()` — saves and immediately flushes to DB
- `refresh()` — reloads entity from DB (after update queries)
- `persistFlushFind()` — persist, flush, clear first-level cache, reload from DB (guarantees fresh data)

---

**Q26: How would you test a Spring @Scheduled method?**

Don't wait for the schedule — call the method directly in your test. Seed the required data, invoke the scheduled method, assert the result. This avoids timing issues and keeps tests fast.

---

## Quick Reference Summary

### TestContainers Cheat Sheet

```java
// PostgreSQL (Spring Boot 3.1+)
@Container
@ServiceConnection
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

// Redis (no module)
@Container
static GenericContainer<?> redis = new GenericContainer<>("redis:7.0")
    .withExposedPorts(6379)
    .waitingFor(Wait.forLogMessage(".*Ready to accept connections.*", 1));

// Singleton base class
public abstract class AbstractIntegrationTest {
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:15").withReuse(true);
    static { POSTGRES.start(); }
}
```

### Testing Slice Quick Reference

| What to test | Slice | Loads |
|---|---|---|
| HTTP endpoints, request/response | `@WebMvcTest` | Controllers only |
| JPA queries | `@DataJpaTest` + TestContainers | JPA layer only |
| HTTP client code | `@RestClientTest` | HTTP client only |
| Service layer only | JUnit + Mockito | Nothing |
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

---

*Last Updated: 2026-06-18*

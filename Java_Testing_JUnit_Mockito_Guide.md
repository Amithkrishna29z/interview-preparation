# Java Testing — JUnit 5, Mockito & Spring Boot Test Guide

## Overview

Testing is asked in virtually every Java developer interview. You need to know JUnit 5 annotations, Mockito mocking, and Spring Boot's testing slice annotations. Interviewers often give you a service class and ask: "How would you write a unit test for this?"

---

## Table of Contents

1. [Testing Pyramid](#testing-pyramid)
2. [JUnit 5 Basics](#junit-5-basics)
3. [JUnit 5 Annotations](#junit-5-annotations)
4. [Assertions](#assertions)
5. [Parameterized Tests](#parameterized-tests)
6. [Mockito Basics](#mockito-basics)
7. [Mockito Advanced](#mockito-advanced)
8. [Spring Boot Test Slices](#spring-boot-test-slices)
9. [Integration Testing](#integration-testing)
10. [Test-Driven Development (TDD)](#test-driven-development-tdd)
11. [Common Interview Questions](#common-interview-questions)
12. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Testing Pyramid

```
           /\
          /  \          E2E Tests
         /    \         (few, slow, expensive)
        /──────\
       /        \       Integration Tests
      /          \      (some)
     /────────────\
    /              \    Unit Tests
   /                \   (many, fast, cheap)
  /──────────────────\
```

| Layer | What it Tests | Speed | Tools |
|---|---|---|---|
| **Unit** | Single class/method in isolation | Very fast | JUnit 5, Mockito |
| **Integration** | Multiple components together (DB, Spring context) | Medium | @SpringBootTest, @DataJpaTest |
| **E2E** | Full application via UI or HTTP | Slow | Selenium, Cypress, REST Assured |

---

## JUnit 5 Basics

JUnit 5 = JUnit Platform + JUnit Jupiter + JUnit Vintage

```xml
<!-- Maven dependency -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <!-- includes JUnit 5, Mockito, AssertJ, Hamcrest -->
</dependency>
```

### Basic Test Class

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    private Calculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new Calculator();
    }

    @Test
    void add_twoPositiveNumbers_returnsSum() {
        // Arrange
        int a = 3, b = 4;

        // Act
        int result = calculator.add(a, b);

        // Assert
        assertEquals(7, result);
    }

    @Test
    @DisplayName("Adding negative numbers returns correct result")
    void addNegativeNumbers() {
        assertEquals(-1, calculator.add(-3, 2));
    }

    @AfterEach
    void tearDown() {
        // cleanup after each test
    }
}
```

> **Test naming convention**: `methodName_scenario_expectedBehavior` — e.g., `save_validUser_returnsCreatedUser`

---

## JUnit 5 Annotations

```java
@Test                    // marks method as a test
@BeforeEach             // runs before EACH test method
@AfterEach              // runs after EACH test method
@BeforeAll              // runs ONCE before all tests (must be static)
@AfterAll               // runs ONCE after all tests (must be static)
@DisplayName("text")    // custom test name in reports
@Disabled("reason")     // skip this test
@Nested                 // nested test class (group related tests)
@Tag("fast")            // tag tests for selective running
@Timeout(5)             // fail if test exceeds 5 seconds
@RepeatedTest(3)        // run test 3 times
@Order(1)               // control execution order (with @TestMethodOrder)
```

### Lifecycle Annotations Example

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class UserServiceTest {

    @BeforeAll
    static void initAll() {
        System.out.println("Runs once before all tests");
    }

    @BeforeEach
    void init() {
        System.out.println("Runs before each test");
    }

    @Test
    @Order(1)
    @DisplayName("Create user successfully")
    void createUser() { ... }

    @Test
    @Order(2)
    @Disabled("Skipping until feature is ready")
    void futureFeatureTest() { ... }

    @AfterEach
    void tearDown() {
        System.out.println("Runs after each test");
    }

    @AfterAll
    static void tearDownAll() {
        System.out.println("Runs once after all tests");
    }
}
```

### @Nested Tests

```java
class UserServiceTest {

    @Nested
    @DisplayName("When user exists")
    class WhenUserExists {
        @BeforeEach void setUp() { /* create test user */ }

        @Test void findById_returnsUser() { ... }
        @Test void update_updatesUser() { ... }
        @Test void delete_deletesUser() { ... }
    }

    @Nested
    @DisplayName("When user does not exist")
    class WhenUserNotFound {
        @Test void findById_throwsException() { ... }
        @Test void update_throwsException() { ... }
    }
}
```

---

## Assertions

### JUnit 5 Assertions

```java
// Basic
assertEquals(expected, actual);
assertEquals(expected, actual, "custom message on failure");
assertNotEquals(unexpected, actual);

// Null checks
assertNull(value);
assertNotNull(value);

// Boolean
assertTrue(condition);
assertFalse(condition);

// Exceptions
assertThrows(IllegalArgumentException.class, () -> service.doWork(null));

// Verify message of exception
Exception ex = assertThrows(UserNotFoundException.class,
    () -> userService.findById(-1L));
assertEquals("User not found with id: -1", ex.getMessage());

// All assertions run even if some fail
assertAll(
    () -> assertEquals("Alice", user.getName()),
    () -> assertEquals("alice@example.com", user.getEmail()),
    () -> assertNotNull(user.getId())
);

// Arrays
assertArrayEquals(new int[]{1, 2, 3}, result);

// Same object reference
assertSame(expected, actual);
```

### AssertJ (fluent, more readable)

```java
import static org.assertj.core.api.Assertions.*;

// Fluent chaining
assertThat(result)
    .isNotNull()
    .isEqualTo(expected);

assertThat(user.getName()).isEqualTo("Alice");
assertThat(user.getAge()).isGreaterThan(18);
assertThat(list).hasSize(3).contains("Alice", "Bob");
assertThat(map).containsKey("name").containsEntry("name", "Alice");

// Exception assertion
assertThatThrownBy(() -> service.findById(-1L))
    .isInstanceOf(UserNotFoundException.class)
    .hasMessageContaining("not found");

// String assertions
assertThat(str).startsWith("Hello").endsWith("World").contains("lo");
```

---

## Parameterized Tests

Run the same test with multiple inputs.

```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3, 4, 5})
void isPositive(int number) {
    assertTrue(number > 0);
}

@ParameterizedTest
@ValueSource(strings = {"", " ", null})
void isBlank_shouldReturnTrue(@NullAndEmptySource String input) {
    assertTrue(StringUtils.isBlank(input));
}

@ParameterizedTest
@CsvSource({
    "Alice, 30, ACTIVE",
    "Bob,   25, INACTIVE",
    "Carol, 22, ACTIVE"
})
void createUser(String name, int age, String status) {
    User user = new User(name, age, Status.valueOf(status));
    assertNotNull(user);
}

@ParameterizedTest
@CsvFileSource(resources = "/test-data/users.csv", numLinesToSkip = 1)
void testFromCsvFile(String email, String expected) { ... }

@ParameterizedTest
@MethodSource("provideUsers")
void testWithMethodSource(String name, int age, boolean expected) {
    assertEquals(expected, userValidator.isValid(name, age));
}

static Stream<Arguments> provideUsers() {
    return Stream.of(
        Arguments.of("Alice", 25, true),
        Arguments.of("", 25, false),
        Arguments.of("Bob", -1, false)
    );
}
```

---

## Mockito Basics

Mockito creates **mock objects** that simulate dependencies, isolating the class under test.

```xml
<!-- included in spring-boot-starter-test -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <scope>test</scope>
</dependency>
```

### Setting Up Mocks

```java
@ExtendWith(MockitoExtension.class)  // JUnit 5 integration
class UserServiceTest {

    @Mock
    UserRepository userRepository;     // creates a mock

    @Mock
    EmailService emailService;

    @InjectMocks
    UserService userService;           // injects mocks into this class

    // Alternatively — manual setup:
    // UserRepository userRepository = Mockito.mock(UserRepository.class);
    // UserService userService = new UserService(userRepository);
}
```

### Stubbing with when().thenReturn()

```java
// Given
User mockUser = new User(1L, "Alice", "alice@example.com");
when(userRepository.findById(1L)).thenReturn(Optional.of(mockUser));
when(userRepository.findById(99L)).thenReturn(Optional.empty());
when(userRepository.existsByEmail(anyString())).thenReturn(false);
when(userRepository.save(any(User.class))).thenAnswer(inv -> inv.getArgument(0));

// Throw exception
when(userRepository.findById(-1L)).thenThrow(new IllegalArgumentException("Invalid id"));

// Multiple calls return different values
when(userRepository.count())
    .thenReturn(0L)    // first call
    .thenReturn(1L)    // second call
    .thenReturn(2L);   // third and subsequent calls

// Void method throws
doThrow(new RuntimeException("DB error")).when(userRepository).delete(any());
```

### Verifying Interactions

```java
// Verify method was called
verify(userRepository).findById(1L);
verify(userRepository, times(1)).save(any(User.class));
verify(emailService, times(1)).sendWelcomeEmail(anyString());

// Verify NOT called
verify(emailService, never()).sendWelcomeEmail(anyString());
verify(userRepository, times(0)).delete(any());

// Verify call count
verify(userRepository, atLeast(1)).findAll();
verify(userRepository, atMost(3)).findAll();

// Verify no more interactions
verifyNoMoreInteractions(userRepository);
verifyNoInteractions(emailService);

// Verify order
InOrder inOrder = inOrder(userRepository, emailService);
inOrder.verify(userRepository).save(any());
inOrder.verify(emailService).sendWelcomeEmail(anyString());
```

### Argument Matchers

```java
any()                        // any object (including null)
any(User.class)              // any User instance
anyString()                  // any String
anyInt(), anyLong()          // any int/long
anyList(), anyMap()          // any List/Map

eq("alice")                  // equals "alice"
isNull()                     // is null
isNotNull()                  // is not null

argThat(user -> user.getAge() > 18)  // custom predicate matcher

// Note: if you use a matcher for one arg, all args must be matchers
when(service.find(eq("Alice"), anyInt())).thenReturn(result); // correct
when(service.find("Alice", anyInt())).thenReturn(result);     // WRONG — mix
```

---

## Mockito Advanced

### ArgumentCaptor — capture arguments passed to mocks

```java
@Captor
ArgumentCaptor<User> userCaptor;

// Alternative: ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);

@Test
void createUser_savesCorrectData() {
    // Act
    userService.createUser("Alice", "alice@example.com");

    // Capture what was actually saved
    verify(userRepository).save(userCaptor.capture());
    User savedUser = userCaptor.getValue();

    assertEquals("Alice", savedUser.getName());
    assertEquals("alice@example.com", savedUser.getEmail());
    assertNotNull(savedUser.getCreatedAt());
}
```

### Spy — partial mocking

```java
// Spy wraps a REAL object — real methods called unless stubbed
@Spy
UserService userService = new UserService(userRepository);

// Only stub specific methods
doReturn("mocked").when(userService).heavyOperation();
// Other methods call real implementation
```

### Mock vs Spy

| | Mock | Spy |
|---|---|---|
| Wraps | Nothing (fake object) | Real object |
| Unstubbed methods | Return defaults (null, 0, false) | Call real implementation |
| Use when | Full isolation needed | Partial mocking of real class |

### BDDMockito (Given-When-Then style)

```java
import static org.mockito.BDDMockito.*;

// given
given(userRepository.findById(1L)).willReturn(Optional.of(mockUser));

// when
User result = userService.getUser(1L);

// then
then(userRepository).should().findById(1L);
assertThat(result).isEqualTo(mockUser);
```

---

## Spring Boot Test Slices

Spring Boot provides test slice annotations that load **only the relevant part** of the application context — much faster than `@SpringBootTest`.

### @WebMvcTest — Test Controllers Only

```java
@WebMvcTest(UserController.class)    // loads only MVC layer
class UserControllerTest {

    @Autowired
    MockMvc mockMvc;                 // HTTP test client

    @MockBean                        // mock the service (Spring-managed)
    UserService userService;

    @Autowired
    ObjectMapper objectMapper;

    @Test
    void getUser_returnsUser() throws Exception {
        // Given
        User user = new User(1L, "Alice", "alice@example.com");
        given(userService.getUser(1L)).willReturn(user);

        // When & Then
        mockMvc.perform(get("/api/users/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"))
            .andExpect(jsonPath("$.email").value("alice@example.com"));
    }

    @Test
    void createUser_validRequest_returns201() throws Exception {
        CreateUserRequest request = new CreateUserRequest("Alice", "alice@example.com");
        User created = new User(1L, "Alice", "alice@example.com");
        given(userService.createUser(any())).willReturn(created);

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1));
    }

    @Test
    void getUser_notFound_returns404() throws Exception {
        given(userService.getUser(99L)).willThrow(new UserNotFoundException(99L));

        mockMvc.perform(get("/api/users/99"))
            .andExpect(status().isNotFound());
    }
}
```

### @DataJpaTest — Test Repository Only

```java
@DataJpaTest  // loads only JPA layer — uses in-memory H2 DB by default
class UserRepositoryTest {

    @Autowired
    UserRepository userRepository;

    @Autowired
    TestEntityManager entityManager; // helper for test data setup

    @Test
    void findByEmail_existingEmail_returnsUser() {
        // Arrange — insert test data
        User user = new User("Alice", "alice@example.com");
        entityManager.persist(user);
        entityManager.flush();

        // Act
        Optional<User> found = userRepository.findByEmail("alice@example.com");

        // Assert
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Alice");
    }

    @Test
    void findByEmail_nonExistentEmail_returnsEmpty() {
        Optional<User> found = userRepository.findByEmail("nobody@example.com");
        assertThat(found).isEmpty();
    }
}
```

### @SpringBootTest — Full Integration Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserIntegrationTest {

    @Autowired
    TestRestTemplate restTemplate;  // HTTP client for full integration tests

    @Test
    void createAndFetchUser() {
        // Create
        CreateUserRequest req = new CreateUserRequest("Alice", "alice@example.com");
        ResponseEntity<User> created = restTemplate.postForEntity("/api/users", req, User.class);
        assertThat(created.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        // Fetch
        Long id = created.getBody().getId();
        ResponseEntity<User> fetched = restTemplate.getForEntity("/api/users/" + id, User.class);
        assertThat(fetched.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(fetched.getBody().getName()).isEqualTo("Alice");
    }
}
```

### Test Slice Comparison

| Annotation | Loads | Use For | Speed |
|---|---|---|---|
| `@WebMvcTest` | Controller + MVC config | Test HTTP endpoints | Fast |
| `@DataJpaTest` | JPA + H2 database | Test repositories | Fast |
| `@DataMongoTest` | MongoDB layer | Test Mongo repos | Fast |
| `@WebFluxTest` | WebFlux controllers | Test reactive endpoints | Fast |
| `@SpringBootTest` | Full application | End-to-end integration | Slow |

---

## Integration Testing

### Testing with Real Database (@DataJpaTest with real DB)

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // use real DB
@ActiveProfiles("test")
class UserRepositoryIntegrationTest {
    // uses test profile DB config (application-test.properties)
}
```

### Testcontainers (real DB in Docker for tests)

```java
@SpringBootTest
@Testcontainers
class UserIntegrationTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Test
    void testWithRealMysql() { ... }
}
```

---

## Test-Driven Development (TDD)

Write the test FIRST, then write the code to make it pass.

```
RED   → Write a failing test
GREEN → Write minimum code to make it pass
REFACTOR → Clean up code without breaking tests
```

```java
// TDD Example: implement a simple password validator

// Step 1: RED — write failing test
@Test
void validate_shortPassword_returnsFalse() {
    assertFalse(validator.validate("abc")); // fails: validator doesn't exist yet
}

// Step 2: GREEN — minimum code to pass
public boolean validate(String password) {
    return password.length() >= 8;
}

// Step 3: add more tests
@Test
void validate_noUppercase_returnsFalse() {
    assertFalse(validator.validate("password1")); // fails
}

// Step 4: extend implementation
public boolean validate(String password) {
    return password.length() >= 8
        && password.matches(".*[A-Z].*");
}

// Step 5: REFACTOR — clean up without breaking tests
```

---

## Common Interview Questions

### Q: What is the difference between `@Mock` and `@MockBean`?

- `@Mock` (Mockito): Creates a Mockito mock. Used in pure unit tests with `@ExtendWith(MockitoExtension.class)`. No Spring context.
- `@MockBean` (Spring Boot): Creates a Mockito mock AND registers it as a Spring bean, replacing any existing bean. Used in `@WebMvcTest` / `@SpringBootTest` where you need Spring context.

---

### Q: What is the difference between a mock and a spy?

- **Mock**: A completely fake object. All methods return defaults unless stubbed.
- **Spy**: Wraps a real object. Real methods are called unless explicitly stubbed. Use when you want to test a real class but mock one specific method.

---

### Q: What is the difference between `verify()` and `when()`?

- `when(...).thenReturn(...)`: **Stubbing** — defines what a mock returns when called. Set this up **before** calling the code under test.
- `verify(...)`: **Verification** — asserts that a mock method was called with specific arguments. Written **after** calling the code under test.

---

### Q: How do you test a method that throws an exception?

```java
// JUnit 5
assertThrows(UserNotFoundException.class, () -> userService.findById(-1L));

// With message check
Exception ex = assertThrows(UserNotFoundException.class,
    () -> userService.findById(-1L));
assertEquals("User not found: -1", ex.getMessage());

// AssertJ
assertThatThrownBy(() -> userService.findById(-1L))
    .isInstanceOf(UserNotFoundException.class)
    .hasMessage("User not found: -1");
```

---

### Q: What is `@InjectMocks`?

`@InjectMocks` tells Mockito to inject the `@Mock`-annotated fields into the class under test via constructor injection, setter injection, or field injection. The test subject becomes a real instance of the class with mocked dependencies.

---

### Q: What's the difference between `@SpringBootTest` and `@WebMvcTest`?

- `@SpringBootTest`: Loads the **full** Spring application context. All beans are created. Slow but tests realistic behavior. Used for integration tests.
- `@WebMvcTest`: Loads **only** the web layer (controllers, filters, security). Services/repositories must be mocked with `@MockBean`. Fast. Used for controller unit tests.

---

## Quick Reference Cheat Sheet

```
JUnit 5 Annotations:
  @Test, @BeforeEach, @AfterEach, @BeforeAll, @AfterAll
  @DisplayName, @Disabled, @Nested, @Tag, @Timeout
  @ParameterizedTest + @ValueSource / @CsvSource / @MethodSource

Assertions:
  assertEquals, assertNotNull, assertTrue, assertThrows
  assertAll (run all even if some fail)
  AssertJ: assertThat(x).isEqualTo(y).isNotNull()...

Mockito Setup:
  @ExtendWith(MockitoExtension.class) — enable Mockito in JUnit 5
  @Mock         — create mock
  @Spy          — wrap real object
  @InjectMocks  — inject mocks into subject under test
  @Captor       — capture arguments

Stubbing:
  when(mock.method(arg)).thenReturn(value)
  when(mock.method()).thenThrow(exception)
  doReturn / doThrow — for void methods and spies

Verification:
  verify(mock).method(arg)
  verify(mock, times(n))
  verify(mock, never())
  verify(mock, atLeast(n))

Argument Matchers:
  any(), anyString(), anyLong(), eq("value"), argThat(predicate)

Spring Boot Slices:
  @WebMvcTest       → controller tests (use @MockBean for services)
  @DataJpaTest      → repository tests (H2 in-memory DB)
  @SpringBootTest   → full integration test

MockMvc:
  perform(get("/url")).andExpect(status().isOk())
  perform(post("/url").content(json)).andExpect(status().isCreated())
  andExpect(jsonPath("$.field").value("expected"))

TDD: RED (failing test) → GREEN (minimum code) → REFACTOR
```

---

*Last Updated: 2026-06-04*

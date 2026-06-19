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
          /  \          E2E Tests (few, slow, expensive)
         /────\
        /      \        Integration Tests (some)
       /────────\
      /          \      Unit Tests (many, fast, cheap)
     /────────────\
```

| Layer | What it Tests | Tools |
|---|---|---|
| **Unit** | Single class/method in isolation | JUnit 5, Mockito |
| **Integration** | Multiple components together (DB, Spring context) | @SpringBootTest, @DataJpaTest |
| **E2E** | Full application via HTTP/UI | REST Assured, Selenium |

---

## JUnit 5 Basics

`spring-boot-starter-test` includes JUnit 5, Mockito, AssertJ, and Hamcrest.

```java
class CalculatorTest {

    private Calculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new Calculator();
    }

    @Test
    void add_twoPositiveNumbers_returnsSum() {
        // Arrange - Act - Assert
        int result = calculator.add(3, 4);
        assertEquals(7, result);
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

### @Nested Tests

Group related scenarios to keep tests readable:

```java
class UserServiceTest {

    @Nested
    @DisplayName("When user exists")
    class WhenUserExists {
        @Test void findById_returnsUser() { ... }
        @Test void delete_deletesUser() { ... }
    }

    @Nested
    @DisplayName("When user does not exist")
    class WhenUserNotFound {
        @Test void findById_throwsException() { ... }
    }
}
```

---

## Assertions

### JUnit 5 Assertions

```java
assertEquals(expected, actual);
assertNotEquals(unexpected, actual);
assertNull(value);
assertNotNull(value);
assertTrue(condition);
assertFalse(condition);

// Exception assertions
Exception ex = assertThrows(UserNotFoundException.class,
    () -> userService.findById(-1L));
assertEquals("User not found with id: -1", ex.getMessage());

// Run all assertions even if some fail
assertAll(
    () -> assertEquals("Alice", user.getName()),
    () -> assertEquals("alice@example.com", user.getEmail()),
    () -> assertNotNull(user.getId())
);
```

### AssertJ (fluent, more readable)

```java
import static org.assertj.core.api.Assertions.*;

assertThat(user.getName()).isEqualTo("Alice");
assertThat(user.getAge()).isGreaterThan(18);
assertThat(list).hasSize(3).contains("Alice", "Bob");

// Exception assertion
assertThatThrownBy(() -> service.findById(-1L))
    .isInstanceOf(UserNotFoundException.class)
    .hasMessageContaining("not found");
```

---

## Parameterized Tests

Run the same test with multiple inputs.

```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3})
void isPositive(int number) {
    assertTrue(number > 0);
}

@ParameterizedTest
@CsvSource({
    "Alice, 30, ACTIVE",
    "Bob,   25, INACTIVE"
})
void createUser(String name, int age, String status) {
    User user = new User(name, age, Status.valueOf(status));
    assertNotNull(user);
}

@ParameterizedTest
@MethodSource("provideUsers")
void testWithMethodSource(String name, int age, boolean expected) {
    assertEquals(expected, userValidator.isValid(name, age));
}

static Stream<Arguments> provideUsers() {
    return Stream.of(
        Arguments.of("Alice", 25, true),
        Arguments.of("", 25, false)
    );
}
```

---

## Mockito Basics

Mockito creates **mock objects** that simulate dependencies, isolating the class under test.

### Setting Up Mocks

```java
@ExtendWith(MockitoExtension.class)  // JUnit 5 integration
class UserServiceTest {

    @Mock
    UserRepository userRepository;   // fake object, returns defaults unless stubbed

    @Mock
    EmailService emailService;

    @InjectMocks
    UserService userService;         // real instance, mocks injected into it
}
```

### Stubbing with when().thenReturn()

```java
User mockUser = new User(1L, "Alice", "alice@example.com");
when(userRepository.findById(1L)).thenReturn(Optional.of(mockUser));
when(userRepository.findById(99L)).thenReturn(Optional.empty());
when(userRepository.existsByEmail(anyString())).thenReturn(false);

// Throw exception
when(userRepository.findById(-1L)).thenThrow(new IllegalArgumentException("Invalid id"));

// Void method throws
doThrow(new RuntimeException("DB error")).when(userRepository).delete(any());
```

### Verifying Interactions

```java
verify(userRepository).findById(1L);
verify(userRepository, times(1)).save(any(User.class));
verify(emailService, never()).sendWelcomeEmail(anyString());
verifyNoMoreInteractions(userRepository);

// Verify order
InOrder inOrder = inOrder(userRepository, emailService);
inOrder.verify(userRepository).save(any());
inOrder.verify(emailService).sendWelcomeEmail(anyString());
```

### Argument Matchers

```java
any(), any(User.class), anyString(), anyInt(), anyLong()
eq("alice")                          // exact value
argThat(user -> user.getAge() > 18)  // custom predicate

// Rule: if you use a matcher for one arg, ALL args must use matchers
when(service.find(eq("Alice"), anyInt())).thenReturn(result); // correct
when(service.find("Alice", anyInt())).thenReturn(result);     // WRONG — mix
```

---

## Mockito Advanced

### ArgumentCaptor — capture arguments passed to mocks

```java
@Captor
ArgumentCaptor<User> userCaptor;

@Test
void createUser_savesCorrectData() {
    userService.createUser("Alice", "alice@example.com");

    verify(userRepository).save(userCaptor.capture());
    User savedUser = userCaptor.getValue();

    assertEquals("Alice", savedUser.getName());
    assertNotNull(savedUser.getCreatedAt());
}
```

### Mock vs Spy

| | Mock | Spy |
|---|---|---|
| Wraps | Nothing (fake object) | Real object |
| Unstubbed methods | Return defaults (null, 0, false) | Call real implementation |
| Use when | Full isolation needed | Partial mocking of real class |

```java
// Spy: wraps a real object — real methods called unless stubbed
@Spy
UserService userService = new UserService(userRepository);
doReturn("mocked").when(userService).heavyOperation();
```

### BDDMockito (Given-When-Then style)

```java
import static org.mockito.BDDMockito.*;

given(userRepository.findById(1L)).willReturn(Optional.of(mockUser));
User result = userService.getUser(1L);
then(userRepository).should().findById(1L);
```

---

## Spring Boot Test Slices

Spring Boot provides test slice annotations that load **only the relevant part** of the context — much faster than `@SpringBootTest`.

### @WebMvcTest — Test Controllers Only

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean  UserService userService;   // Spring-managed mock
    @Autowired ObjectMapper objectMapper;

    @Test
    void getUser_returnsUser() throws Exception {
        given(userService.getUser(1L)).willReturn(new User(1L, "Alice", "alice@example.com"));

        mockMvc.perform(get("/api/users/1").contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"));
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

    @Autowired UserRepository userRepository;
    @Autowired TestEntityManager entityManager;

    @Test
    void findByEmail_existingEmail_returnsUser() {
        entityManager.persistAndFlush(new User("Alice", "alice@example.com"));

        Optional<User> found = userRepository.findByEmail("alice@example.com");

        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Alice");
    }
}
```

### @SpringBootTest — Full Integration Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserIntegrationTest {

    @Autowired TestRestTemplate restTemplate;

    @Test
    void createAndFetchUser() {
        CreateUserRequest req = new CreateUserRequest("Alice", "alice@example.com");
        ResponseEntity<User> created = restTemplate.postForEntity("/api/users", req, User.class);
        assertThat(created.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```

### Test Slice Comparison

| Annotation | Loads | Use For | Speed |
|---|---|---|---|
| `@WebMvcTest` | Controller + MVC config | Test HTTP endpoints | Fast |
| `@DataJpaTest` | JPA + H2 database | Test repositories | Fast |
| `@SpringBootTest` | Full application | End-to-end integration | Slow |

---

## Integration Testing

### Real Database

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@ActiveProfiles("test")
class UserRepositoryIntegrationTest {
    // uses application-test.properties DB config
}
```

---

## Test-Driven Development (TDD)

Write the test FIRST, then write the code to make it pass.

```
RED   → Write a failing test
GREEN → Write minimum code to make it pass
REFACTOR → Clean up without breaking tests
```

```java
// Step 1: RED — write failing test
@Test
void validate_shortPassword_returnsFalse() {
    assertFalse(validator.validate("abc"));
}

// Step 2: GREEN — minimum code to pass
public boolean validate(String password) {
    return password.length() >= 8;
}

// Step 3: add more tests, extend implementation, refactor
```

---

## Common Interview Questions

### Q: What is the difference between `@Mock` and `@MockBean`?

`@Mock` is plain Mockito — used in unit tests with `@ExtendWith(MockitoExtension.class)`, no Spring context involved. `@MockBean` is Spring Boot specific — it creates a Mockito mock and registers it as a Spring bean, replacing any existing bean. Use `@MockBean` inside `@WebMvcTest` or `@SpringBootTest`.

---

### Q: What is the difference between a mock and a spy?

A mock is a completely fake object — all methods return defaults (null, 0, false) unless stubbed. A spy wraps a real object and calls real methods unless you explicitly stub them. Use a spy when you want to test a real class but mock only one specific method.

---

### Q: What is the difference between `verify()` and `when()`?

`when(...).thenReturn(...)` is **stubbing** — it defines what a mock returns when called, set up before the code under test runs. `verify(...)` is **verification** — it asserts that a mock method was actually called with specific arguments, written after the code under test runs.

---

### Q: How do you test a method that throws an exception?

```java
// JUnit 5
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

`@InjectMocks` tells Mockito to inject the `@Mock`-annotated fields into the class under test via constructor, setter, or field injection. The result is a real instance of your class with all its dependencies replaced by mocks.

---

### Q: What's the difference between `@SpringBootTest` and `@WebMvcTest`?

`@SpringBootTest` loads the full application context — all beans, slow but realistic, used for integration tests. `@WebMvcTest` loads only the web layer (controllers, filters); services must be mocked with `@MockBean`. It's fast and focused on testing HTTP behavior.

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

*Last Updated: 2026-06-18*

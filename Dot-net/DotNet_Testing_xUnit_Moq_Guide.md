# .NET Testing with xUnit, Moq & FluentAssertions — A Guide for Java Developers

## Overview

If you already know **JUnit** and **Mockito**, you are 80% of the way to testing in .NET. The concepts are identical — arrange your data, run the code, assert the result, and mock the dependencies you don't want to hit. Only the *names* and a few *mechanics* change.

This guide maps everything you know from the Java testing world onto the modern .NET stack:

- **xUnit** — the test framework (your JUnit 5)
- **Moq** — the mocking library (your Mockito)
- **FluentAssertions** — readable assertions (your AssertJ)
- **WebApplicationFactory** — integration testing for ASP.NET Core (your `@SpringBootTest` + MockMvc)

Everything targets **.NET 8+** and modern C#. Every code block compares directly to the JUnit/Mockito equivalent so you can lean on what you already know.

---

## Table of Contents

- [Overview](#overview)
- [JUnit/Mockito → xUnit/Moq Mapping](#junitmockito--xunitmoq-mapping)
- [1. Why Test & The Testing Pyramid](#1-why-test--the-testing-pyramid)
- [2. The AAA Pattern (Arrange-Act-Assert)](#2-the-aaa-pattern-arrange-act-assert)
- [3. The Three .NET Frameworks (xUnit, NUnit, MSTest)](#3-the-three-net-frameworks-xunit-nunit-mstest)
- [4. Creating a Test Project & Running Tests](#4-creating-a-test-project--running-tests)
- [5. [Fact] vs [Theory] (Parameterized Tests)](#5-fact-vs-theory-parameterized-tests)
- [6. Assertions: Built-in vs FluentAssertions](#6-assertions-built-in-vs-fluentassertions)
- [7. Setup & Teardown (Fixtures)](#7-setup--teardown-fixtures)
- [8. Testing Exceptions](#8-testing-exceptions)
- [9. Testing Async Code](#9-testing-async-code)
- [10. Mocking with Moq](#10-mocking-with-moq)
- [11. Mocks vs Stubs vs Fakes](#11-mocks-vs-stubs-vs-fakes)
- [12. Integration Testing ASP.NET Core](#12-integration-testing-aspnet-core)
- [13. Test Data Builders](#13-test-data-builders)
- [14. Code Coverage & Naming Conventions](#14-code-coverage--naming-conventions)
- [15. Best Practices](#15-best-practices)
- [Common Interview Questions](#common-interview-questions)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## JUnit/Mockito → xUnit/Moq Mapping

This is the single most useful table in the guide. Bookmark it.

| Java (JUnit 5 / Mockito) | .NET (xUnit / Moq) | Notes |
|---|---|---|
| `@Test` | `[Fact]` | A single test with no parameters |
| `@ParameterizedTest` | `[Theory]` | A data-driven test |
| `@ValueSource` / `@CsvSource` | `[InlineData(...)]` | Inline data for a `[Theory]` |
| `@MethodSource` | `[MemberData(nameof(...))]` | Data from a property/method |
| custom `ArgumentsProvider` | `[ClassData(typeof(...))]` | Data from a class |
| `@BeforeEach` | **constructor** of the test class | Runs before *every* test |
| `@AfterEach` | `Dispose()` (implement `IDisposable`) | Runs after *every* test |
| `@BeforeAll` / `@AfterAll` | `IClassFixture<T>` | Shared setup once per class |
| (shared across many classes) | `ICollectionFixture<T>` | Shared once across a collection |
| `assertEquals(a, b)` | `Assert.Equal(a, b)` | Expected first, actual second |
| `assertTrue(x)` | `Assert.True(x)` | |
| `assertNull(x)` | `Assert.Null(x)` | |
| `assertThrows(Ex.class, () -> ...)` | `Assert.Throws<Ex>(() => ...)` | |
| AssertJ `assertThat(x).isEqualTo(y)` | FluentAssertions `x.Should().Be(y)` | Both fluent styles |
| `Mockito.mock(Foo.class)` | `new Mock<IFoo>()` | `.Object` gives the instance |
| `when(m.go()).thenReturn(x)` | `mock.Setup(m => m.Go()).Returns(x)` | |
| `when(m.go()).thenThrow(ex)` | `mock.Setup(m => m.Go()).Throws(ex)` | |
| `verify(m).go()` | `mock.Verify(m => m.Go(), Times.Once)` | |
| `verify(m, times(2))` | `mock.Verify(..., Times.Exactly(2))` | |
| `any()` / `anyString()` | `It.IsAny<T>()` | Argument matcher |
| `eq(5)` / `argThat(...)` | `It.Is<int>(v => v == 5)` | Conditional matcher |
| `@Mock` | manual `new Mock<T>()` (or AutoMocker) | No annotation magic by default |
| `@InjectMocks` | manual constructor injection (or `AutoMocker`) | Pass `mock.Object` into the SUT |
| `@SpringBootTest` | `WebApplicationFactory<Program>` | Boots the whole app |
| `MockMvc` | `factory.CreateClient()` (`HttpClient`) | Sends real HTTP to in-memory server |
| `@DataJpaTest` + H2 | EF Core `UseInMemoryDatabase` / SQLite | In-memory DB for repo tests |
| JaCoCo | coverlet (`--collect:"XPlat Code Coverage"`) | Coverage tool |
| Maven `mvn test` / Gradle `test` | `dotnet test` | CLI test runner |

---

## 1. Why Test & The Testing Pyramid

**Think of it like...** a smoke detector in your house. You hope it never goes off, but you'd never sleep soundly without it. Tests are smoke detectors for your code — they alarm the moment something breaks, *before* it reaches production.

Why we test:
- **Catch regressions** — change one thing, prove you didn't break ten others.
- **Document behavior** — a good test reads like a spec.
- **Enable refactoring** — you can rip out internals fearlessly if tests stay green.
- **Faster feedback** — milliseconds in a test vs. minutes clicking through the UI.

### The Testing Pyramid

```
        /\
       /e2e\          few   — slow, brittle, expensive (full app + browser/DB)
      /------\
     /  integ \       some  — medium speed (app + real DB, or HTTP)
    /----------\
   /    unit    \     many  — fast, isolated, cheap (one class, deps mocked)
  /--------------\
```

- **Unit tests** (base, most numerous): one class in isolation; dependencies mocked. Milliseconds. This is where Moq lives.
- **Integration tests** (middle): multiple components together — e.g. a controller calling a real service hitting an in-memory DB. This is `WebApplicationFactory`.
- **End-to-end tests** (top, fewest): the whole system, often through the UI. Slow and flaky — use sparingly.

Same pyramid you learned for Java. The shape is the lesson: lots of cheap unit tests at the bottom, few expensive e2e tests at the top.

---

## 2. The AAA Pattern (Arrange-Act-Assert)

**Think of it like...** a science experiment: set up your beakers (Arrange), pour them together (Act), record what happened (Assert). Java calls it the same thing — "Given-When-Then" is the BDD flavor of the exact same idea.

```csharp
[Fact]
public void Add_TwoPositiveNumbers_ReturnsSum()
{
    // Arrange — create the system under test (SUT) and inputs
    var calculator = new Calculator();      // like: var calc = new Calculator(); in JUnit
    int a = 2, b = 3;

    // Act — invoke the one method you're testing
    int result = calculator.Add(a, b);

    // Assert — verify the outcome
    Assert.Equal(5, result);                // JUnit: assertEquals(5, result);
}
```

The three comment blocks are a convention, not syntax — but writing them keeps tests honest. If your "Act" section has five lines, your test is probably doing too much.

---

## 3. The Three .NET Frameworks (xUnit, NUnit, MSTest)

**Think of it like...** JUnit vs. TestNG vs. Spock in Java — they all run tests, but with different idioms. In .NET the three are **xUnit**, **NUnit**, and **MSTest**.

| | xUnit | NUnit | MSTest |
|---|---|---|---|
| Test attribute | `[Fact]` / `[Theory]` | `[Test]` / `[TestCase]` | `[TestMethod]` / `[DataRow]` |
| Setup before each | constructor | `[SetUp]` | `[TestInitialize]` |
| Teardown after each | `IDisposable.Dispose()` | `[TearDown]` | `[TestCleanup]` |
| Closest to JUnit 5 | **Yes** | similar to JUnit 4 | Microsoft's own |
| Most common today | **Most popular** | Common, mature | Bundled with VS |

**Focus on xUnit.** It's the most modern, most widely used in new .NET projects, and the most JUnit-like in philosophy. A defining xUnit choice: it creates **a brand-new instance of the test class for every single test method** — meaning your tests can't accidentally share state through fields. That isolation is baked in (JUnit 5 does the same with its default `PER_METHOD` lifecycle). This is *why* xUnit uses the constructor for setup instead of a `[BeforeEach]` annotation — a new object means the constructor runs fresh each time.

The rest of this guide is xUnit only.

---

## 4. Creating a Test Project & Running Tests

**Think of it like...** `mvn archetype:generate` or a Gradle `test` source set — except it's one command.

```bash
# Create a new xUnit test project (like a Maven module with junit on the classpath)
dotnet new xunit -n MyApp.Tests

# Reference the project you want to test (like a <dependency> on your main module)
dotnet add MyApp.Tests reference MyApp/MyApp.csproj

# Add the mocking + assertion libraries (like adding mockito + assertj to pom.xml)
dotnet add MyApp.Tests package Moq
dotnet add MyApp.Tests package FluentAssertions
```

Running tests:

```bash
dotnet test                                  # run all tests  (≈ mvn test)
dotnet test --filter "FullyQualifiedName~Calculator"   # run a subset
dotnet test --logger "console;verbosity=detailed"      # verbose output
```

A fresh `dotnet new xunit` project gives you a `.csproj` already wired with the xUnit packages and a `UnitTest1.cs` stub — the equivalent of a generated `AppTest.java`.

---

## 5. [Fact] vs [Theory] (Parameterized Tests)

**Think of it like...** `@Test` vs `@ParameterizedTest`. A `[Fact]` is a single fixed test. A `[Theory]` runs the same body many times with different data.

### [Fact] — a single test

```csharp
[Fact]                                       // JUnit: @Test
public void IsEven_Four_ReturnsTrue()
{
    var result = NumberUtils.IsEven(4);
    result.Should().BeTrue();                // FluentAssertions
}
```

### [Theory] with [InlineData] — inline parameters

```csharp
[Theory]                                     // JUnit: @ParameterizedTest
[InlineData(2, true)]                        // JUnit: @CsvSource({"2,true", ...})
[InlineData(3, false)]
[InlineData(0, true)]
[InlineData(-4, true)]
public void IsEven_VariousInputs_ReturnsExpected(int input, bool expected)
{
    // Runs once PER [InlineData] line — each shows as a separate test in the runner
    NumberUtils.IsEven(input).Should().Be(expected);
}
```

### [MemberData] — data from a property/method

Use when inline literals aren't enough (objects, computed data). This is xUnit's `@MethodSource`.

```csharp
public class MathTests
{
    // A static property/method returning IEnumerable<object[]>; each object[] is one row of args
    public static IEnumerable<object[]> AddCases =>
        new List<object[]>
        {
            new object[] { 1, 2, 3 },
            new object[] { -1, 1, 0 },
            new object[] { 100, 200, 300 },
        };

    [Theory]
    [MemberData(nameof(AddCases))]           // JUnit: @MethodSource("addCases")
    public void Add_Cases_ReturnsSum(int a, int b, int expected)
    {
        new Calculator().Add(a, b).Should().Be(expected);
    }
}
```

### [ClassData] — data from a dedicated class

For reusable or complex data sets, put the data in its own class implementing `IEnumerable<object[]>`.

```csharp
public class AddTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { 1, 2, 3 };
        yield return new object[] { 5, 5, 10 };
    }
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(AddTestData))]             // JUnit: a custom ArgumentsProvider
public void Add_FromClassData_ReturnsSum(int a, int b, int expected)
{
    new Calculator().Add(a, b).Should().Be(expected);
}
```

**Rule of thumb:** `[InlineData]` for simple literals, `[MemberData]` for shared/computed data in the same class, `[ClassData]` for reusable data across many test classes.

---

## 6. Assertions: Built-in vs FluentAssertions

**Think of it like...** plain JUnit `assertEquals` vs. AssertJ's `assertThat(x).isEqualTo(y)`. .NET has the same split: built-in `Assert.*` (JUnit style) and **FluentAssertions** (AssertJ style).

### Built-in xUnit assertions

```csharp
Assert.Equal(5, result);                     // assertEquals(5, result)  — expected FIRST
Assert.NotEqual(5, result);                  // assertNotEquals
Assert.True(flag);                           // assertTrue
Assert.False(flag);                          // assertFalse
Assert.Null(obj);                            // assertNull
Assert.NotNull(obj);                         // assertNotNull
Assert.Contains("ell", "hello");             // string/collection contains
Assert.Empty(list);                          // assertTrue(list.isEmpty())
Assert.IsType<ArgumentException>(ex);        // assertEquals(class, ex.getClass())
Assert.Throws<InvalidOperationException>(() => svc.Run());  // assertThrows
```

> **Gotcha:** xUnit's `Assert.Equal(expected, actual)` puts **expected first** — same order as JUnit. Get it backwards and the failure message lies to you.

### FluentAssertions (recommended)

Reads like English, gives far better failure messages, and chains naturally.

```csharp
result.Should().Be(5);                       // AssertJ: assertThat(result).isEqualTo(5)
result.Should().NotBe(0);
flag.Should().BeTrue();
name.Should().StartWith("Jo").And.EndWith("hn").And.HaveLength(4);

person.Should().NotBeNull();
person.Name.Should().Be("Alice");

// Collections — extremely readable
numbers.Should().HaveCount(3)
       .And.Contain(2)
       .And.BeInAscendingOrder();

// Deep object equality without overriding equals/hashCode (huge win over Java)
actualOrder.Should().BeEquivalentTo(expectedOrder);   // compares property-by-property

// Exceptions, fluently
Action act = () => svc.Run();
act.Should().Throw<InvalidOperationException>()
   .WithMessage("*not allowed*");            // wildcard message match
```

**Why prefer FluentAssertions?** `Assert.True(person.Age > 18)` fails with a useless "Expected True but got False." `person.Age.Should().BeGreaterThan(18)` fails with "Expected person.Age to be greater than 18, but found 15." The second message tells you everything. Same reason Java teams reach for AssertJ over raw JUnit asserts.

> Note: FluentAssertions 8+ changed its license model (free for personal/open-source, paid for some commercial use). For a junior role just know it exists and is the de-facto fluent assertion library; some teams use the open `Shouldly` library as an alternative.

---

## 7. Setup & Teardown (Fixtures)

**Think of it like...** `@BeforeEach`/`@AfterEach`/`@BeforeAll`. xUnit maps each lifecycle to a C# language feature instead of an annotation.

### Per-test setup/teardown — constructor + IDisposable

Because xUnit makes a **new instance per test**, the constructor *is* your `@BeforeEach`, and `Dispose()` is your `@AfterEach`.

```csharp
public class OrderServiceTests : IDisposable
{
    private readonly OrderService _sut;          // fresh for every test — no shared state
    private readonly DbConnection _connection;

    public OrderServiceTests()                   // == @BeforeEach (runs before EACH test)
    {
        _connection = OpenTestConnection();
        _sut = new OrderService(_connection);
    }

    public void Dispose()                        // == @AfterEach (runs after EACH test)
    {
        _connection.Dispose();
    }

    [Fact]
    public void PlaceOrder_Valid_Succeeds() { /* uses fresh _sut */ }
}
```

### Shared setup once per class — IClassFixture<T>

This is xUnit's `@BeforeAll`/`@AfterAll`. Use it for expensive setup you want **created once and shared by all tests in the class** — e.g. spinning up a database or a web host.

```csharp
// 1. The fixture: built once, disposed once for the whole test class
public class DatabaseFixture : IDisposable
{
    public AppDbContext Db { get; }
    public DatabaseFixture()                     // runs ONCE before the first test (≈ @BeforeAll)
    {
        Db = BuildSeededInMemoryDb();
    }
    public void Dispose() => Db.Dispose();       // runs ONCE after the last test (≈ @AfterAll)
}

// 2. Implement IClassFixture<T> to receive it via constructor injection
public class ProductRepoTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;
    public ProductRepoTests(DatabaseFixture fixture) => _fixture = fixture;  // injected by xUnit

    [Fact]
    public void GetById_Existing_ReturnsProduct()
    {
        var p = new ProductRepo(_fixture.Db).GetById(1);
        p.Should().NotBeNull();
    }
}
```

### Shared across MANY test classes — ICollectionFixture<T>

When several test classes must share *the same* expensive resource (e.g. one Docker container or one web host for the whole suite), use a collection fixture.

```csharp
// Declare a collection that owns the fixture
[CollectionDefinition("Database collection")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture> { }

// Tag each test class into that collection — they now SHARE one DatabaseFixture instance
[Collection("Database collection")]
public class CustomerRepoTests
{
    private readonly DatabaseFixture _fixture;
    public CustomerRepoTests(DatabaseFixture fixture) => _fixture = fixture;
    // ...
}
```

| Scope | xUnit mechanism | JUnit equivalent |
|---|---|---|
| Before/after each test | constructor / `Dispose()` | `@BeforeEach` / `@AfterEach` |
| Once per test class | `IClassFixture<T>` | `@BeforeAll` / `@AfterAll` |
| Once across many classes | `ICollectionFixture<T>` | (no direct JUnit equivalent; ~extension) |

---

## 8. Testing Exceptions

**Think of it like...** `assertThrows(IllegalArgumentException.class, () -> ...)`.

### Built-in

```csharp
[Fact]
public void Withdraw_MoreThanBalance_Throws()
{
    var account = new Account(balance: 100);

    // Assert.Throws returns the caught exception so you can inspect it
    var ex = Assert.Throws<InvalidOperationException>(
        () => account.Withdraw(500));        // JUnit: assertThrows(...,() -> account.withdraw(500))

    ex.Message.Should().Contain("Insufficient funds");
}
```

### FluentAssertions (recommended — clearer)

```csharp
[Fact]
public void Withdraw_MoreThanBalance_Throws()
{
    var account = new Account(balance: 100);
    Action act = () => account.Withdraw(500);    // wrap the call in an Action delegate

    act.Should().Throw<InvalidOperationException>()
       .WithMessage("*Insufficient funds*");     // * = wildcard

    // Assert that NO exception is thrown:
    Action safe = () => account.Withdraw(50);
    safe.Should().NotThrow();
}
```

For async methods use `Assert.ThrowsAsync<T>` / `await act.Should().ThrowAsync<T>()` (note the `Async` suffix and the `await`).

---

## 9. Testing Async Code

**Think of it like...** testing a method returning `CompletableFuture` in Java — except in C# you simply make the **test method itself `async Task`** and `await` the call. No blocking, no `.get()`.

```csharp
[Fact]
public async Task GetUserAsync_ExistingId_ReturnsUser()   // return Task, not void
{
    // Arrange
    var service = new UserService(repo);

    // Act — just await like any other async call
    var user = await service.GetUserAsync(1);

    // Assert
    user.Should().NotBeNull();
    user!.Id.Should().Be(1);
}
```

Key rules:
- Test methods returning `Task` are awaited by xUnit — the test only passes once the task completes.
- **Never** make an async test `async void` — exceptions in `async void` escape the runner and may not fail the test. Always `async Task`.
- Testing an async exception:

```csharp
[Fact]
public async Task GetUserAsync_MissingId_ThrowsNotFound()
{
    Func<Task> act = async () => await service.GetUserAsync(999);  // wrap in Func<Task>
    await act.Should().ThrowAsync<NotFoundException>();            // note: await + ThrowAsync
}
```

---

## 10. Mocking with Moq

**Think of it like...** Mockito, almost line for line. The big mental shift: Moq uses **lambda expressions** (`x => x.Method(...)`) to point at the method, whereas Mockito wraps the actual call (`when(m.method())`).

You almost always mock **interfaces** (or virtual members) — same constraint as Mockito mocking concrete classes is possible but messy; prefer interfaces.

### Setup & Returns

```csharp
public interface IUserRepository
{
    User? GetById(int id);
    Task<User?> GetByIdAsync(int id);
    void Save(User user);
}

[Fact]
public void GetUser_ExistingId_ReturnsName()
{
    // Arrange
    var mockRepo = new Mock<IUserRepository>();          // Mockito: mock(IUserRepository.class)

    mockRepo.Setup(r => r.GetById(1))                    // Mockito: when(repo.getById(1))
            .Returns(new User { Id = 1, Name = "Alice" });//          .thenReturn(new User(...))

    var service = new UserService(mockRepo.Object);      // .Object = the real instance to inject
                                                         // (≈ Mockito's mock passed to @InjectMocks)
    // Act
    var name = service.GetUserName(1);

    // Assert
    name.Should().Be("Alice");
}
```

### Async returns — ReturnsAsync

```csharp
mockRepo.Setup(r => r.GetByIdAsync(1))
        .ReturnsAsync(new User { Id = 1, Name = "Bob" });  // Mockito: thenReturn(completedFuture(..))
```

### Argument matchers

```csharp
// Match ANY argument of the type
mockRepo.Setup(r => r.GetById(It.IsAny<int>()))            // Mockito: any(Integer.class) / anyInt()
        .Returns(new User { Name = "Default" });

// Match a CONDITION on the argument
mockRepo.Setup(r => r.GetById(It.Is<int>(id => id > 0)))   // Mockito: argThat(id -> id > 0)
        .Returns(new User());

// Match a specific value (just pass the literal — like Mockito's eq())
mockRepo.Setup(r => r.GetById(5)).Returns(new User());
```

> **Same Moq/Mockito gotcha:** if you use a matcher (`It.IsAny`) for *one* argument, you must use matchers for *all* arguments in that call. Don't mix `It.IsAny<int>()` with a raw literal in the same `Setup`.

### Verify — checking interactions

```csharp
[Fact]
public void Register_NewUser_SavesOnce()
{
    var mockRepo = new Mock<IUserRepository>();
    var service = new UserService(mockRepo.Object);

    service.Register(new User { Name = "Carol" });

    // Verify Save was called exactly once with any User
    mockRepo.Verify(r => r.Save(It.IsAny<User>()), Times.Once);   // Mockito: verify(repo).save(any())

    // Other Times options:
    // Times.Never  Times.Exactly(2)  Times.AtLeastOnce  Times.AtLeast(2)  Times.AtMost(3)

    // Verify a call with a SPECIFIC matched argument:
    mockRepo.Verify(r => r.Save(It.Is<User>(u => u.Name == "Carol")), Times.Once);
}
```

### Throwing from a mock

```csharp
mockRepo.Setup(r => r.GetById(It.IsAny<int>()))
        .Throws(new TimeoutException());                  // Mockito: thenThrow(new ...)
// async version:
mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<int>()))
        .ThrowsAsync(new TimeoutException());
```

### Callbacks — capture / react to arguments

```csharp
User? captured = null;
mockRepo.Setup(r => r.Save(It.IsAny<User>()))
        .Callback<User>(u => captured = u);               // Mockito: thenAnswer / ArgumentCaptor

service.Register(new User { Name = "Dave" });
captured!.Name.Should().Be("Dave");
```

### Mocking properties & sequences

```csharp
// Property getter (Mockito: when(mock.getName()).thenReturn("X"))
var mockConfig = new Mock<IConfig>();
mockConfig.Setup(c => c.MaxRetries).Returns(3);

// Auto-track property reads/writes like a real property
mockConfig.SetupProperty(c => c.Timeout, 30);

// Return different values on successive calls (Mockito: thenReturn(a).thenReturn(b))
mockRepo.SetupSequence(r => r.GetById(1))
        .Returns(new User { Name = "First" })
        .Returns(new User { Name = "Second" })
        .Throws(new InvalidOperationException());
```

### On `@InjectMocks`

Moq has **no `@InjectMocks` equivalent built in.** You inject manually by passing `.Object` into the constructor — which is honestly clearer. If a class has many dependencies, the community `Moq.AutoMock` package provides an `AutoMocker` that auto-creates and injects mocks, mimicking `@InjectMocks`:

```csharp
var mocker = new AutoMocker();                            // ≈ the @InjectMocks machinery
var service = mocker.CreateInstance<UserService>();       // all ctor deps auto-mocked
mocker.GetMock<IUserRepository>()
      .Setup(r => r.GetById(1)).Returns(new User());
```

---

## 11. Mocks vs Stubs vs Fakes

**Think of it like...** the same test-double vocabulary you learned with Mockito — the terms are identical across Java and .NET.

| Double | What it does | Use when |
|---|---|---|
| **Stub** | Returns canned answers; you don't care if/how it's called | You just need a dependency to *give back* data (`Setup().Returns()`) |
| **Mock** | A stub you also **assert calls on** | You care about the *interaction* (`Verify(...)`) — e.g. "did we send the email?" |
| **Fake** | A real, lightweight working implementation | A test double would be too fiddly; e.g. in-memory DB or in-memory list repo |
| **Spy** | Wraps a real object, records calls | Rarely needed; Moq does it via `mock.CallBase = true` |

In Moq, a `Mock<T>` with only `.Setup()` is being used as a **stub**; add `.Verify()` and it's a **mock**. Same object, different intent — exactly like Mockito.

### When to use a real in-memory dependency (a Fake)

Mocking a repository for *every* query gets painful. For data-access-heavy logic, prefer a **real but in-memory database** so you exercise real LINQ/EF behavior:

```csharp
// EF Core InMemory provider — quick, but NOT a real relational engine (no real SQL, no constraints)
var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())  // unique name = isolated per test
    .Options;

using var db = new AppDbContext(options);
db.Products.Add(new Product { Id = 1, Name = "Widget" });
db.SaveChanges();

var repo = new ProductRepo(db);
repo.GetById(1).Should().NotBeNull();
```

> **Tip:** `UseInMemoryDatabase` is fast but doesn't enforce relational rules (foreign keys, unique constraints). For tests that need *real* SQL behavior, use **SQLite in-memory** (`UseSqlite("DataSource=:memory:")`) — it's a genuine SQL engine and catches things the InMemory provider silently allows. This mirrors the Java choice between a naive fake repo and an H2 in-memory database for `@DataJpaTest`.

---

## 12. Integration Testing ASP.NET Core

**Think of it like...** `@SpringBootTest(webEnvironment = RANDOM_PORT)` combined with `MockMvc`. `WebApplicationFactory<Program>` boots your **entire ASP.NET Core app in memory** and hands you an `HttpClient` to hit it with real HTTP requests — no network, no port, fast.

You'll need the `Microsoft.AspNetCore.Mvc.Testing` package, and your `Program` class must be reachable from the test project (in .NET 8 minimal hosting, add `public partial class Program { }` at the bottom of `Program.cs`, or it's exposed automatically with the test SDK).

```csharp
// Inherit IClassFixture so the app is booted ONCE and shared across tests in this class
public class ProductsApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ProductsApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();    // an HttpClient wired to the in-memory TestServer
                                             // (≈ MockMvc, but issuing REAL HTTP)
    }

    [Fact]
    public async Task GetProducts_ReturnsOkAndJson()
    {
        // Act — a real GET request to the in-memory server
        var response = await _client.GetAsync("/api/products");

        // Assert status (≈ MockMvc andExpect(status().isOk()))
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        // Assert body deserialized from JSON
        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        products.Should().NotBeNullOrEmpty();
    }

    [Fact]
    public async Task CreateProduct_ValidPayload_Returns201()
    {
        var newProduct = new { Name = "Gadget", Price = 9.99m };

        var response = await _client.PostAsJsonAsync("/api/products", newProduct);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
    }
}
```

### Swapping real dependencies for test ones

The killer feature: override the app's DI container to replace, say, the real database with an in-memory one (like Spring's `@MockBean` or a `@TestConfiguration`).

```csharp
public class CustomFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Remove the real DbContext registration...
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor != null) services.Remove(descriptor);

            // ...and register an in-memory one for the test run (≈ Spring @MockBean / test profile)
            services.AddDbContext<AppDbContext>(opt =>
                opt.UseInMemoryDatabase("IntegrationTestDb"));
        });
    }
}
```

`TestServer` is the lower-level component `WebApplicationFactory` builds on; you rarely use it directly, but it's the in-memory HTTP server doing the work — the equivalent of Spring's mock servlet environment.

---

## 13. Test Data Builders

**Think of it like...** the Builder pattern you'd use in Java tests (or a `@TestConfiguration` factory). The problem: tests get cluttered constructing objects with 10 fields when only 1 matters. A builder gives sensible defaults and lets each test override just what it cares about.

```csharp
public class UserBuilder
{
    // Sensible defaults so most tests need to set nothing
    private string _name = "Default User";
    private int _age = 30;
    private bool _isActive = true;

    // Fluent "with" methods return `this` for chaining
    public UserBuilder WithName(string name) { _name = name; return this; }
    public UserBuilder WithAge(int age) { _age = age; return this; }
    public UserBuilder Inactive() { _isActive = false; return this; }

    public User Build() => new User { Name = _name, Age = _age, IsActive = _isActive };
}
```

Usage — the test screams its intent because only the *relevant* field is set:

```csharp
[Fact]
public void Discount_InactiveUser_IsZero()
{
    // Only "inactive" matters to this test; name/age are irrelevant noise, so default them
    var user = new UserBuilder().Inactive().Build();

    new PricingService().GetDiscount(user).Should().Be(0);
}
```

This keeps Arrange sections short and makes tests resilient — add a new required `User` field later and you fix it *once* in the builder, not in 50 tests. (Many .NET teams also use libraries like **Bogus** for realistic fake data or **AutoFixture** to auto-generate objects, mirroring Java's `EasyRandom`.)

---

## 14. Code Coverage & Naming Conventions

### Coverage with coverlet

**Think of it like...** JaCoCo. `coverlet` is the standard .NET coverage tool and ships with the xUnit template.

```bash
# Collect coverage during a test run (outputs a coverage.cobertura.xml)
dotnet test --collect:"XPlat Code Coverage"

# Turn that XML into a browsable HTML report (install the ReportGenerator global tool first)
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coveragereport" -reporttypes:Html
```

> **Caution (same as Java):** coverage measures lines *executed*, not behavior *verified*. 100% coverage with no meaningful assertions proves nothing. Aim for meaningful tests, not a vanity number. 70–80% on critical logic is healthier than 100% chasing trivial getters.

### Naming conventions

The most common .NET convention is **`MethodName_Scenario_ExpectedResult`** — descriptive enough that a failing test name tells you what broke without opening the file.

```csharp
Withdraw_AmountExceedsBalance_ThrowsInvalidOperation()
GetUser_IdNotFound_ReturnsNull()
IsEligible_AgeUnder18_ReturnsFalse()
Add_TwoNegatives_ReturnsNegativeSum()
```

This is the same `methodUnderTest_givenCondition_expectedBehavior` style recommended for JUnit. Keep it consistent across the codebase.

---

## 15. Best Practices

**Think of it like...** the FIRST principles you learned for JUnit — **F**ast, **I**solated, **R**epeatable, **S**elf-validating, **T**imely. They apply identically in .NET.

- **One assert-concept per test.** Not literally one `Assert`, but one *behavior*. If a test verifies "creates user AND sends email AND logs," split it — a failure should point at one thing.
- **No logic in tests.** No `if`/`for`/`while`/`try` in a test body. Branching means the test can take paths you never check. Use `[Theory]` data instead of loops.
- **Deterministic.** No `DateTime.Now`, `Guid.NewGuid()`, `Random`, or real network/time in assertions. Inject an `IClock`/`TimeProvider` (`.NET 8` has a built-in `TimeProvider`) and control it. A flaky test is worse than no test.
- **Fast & isolated.** Unit tests shouldn't touch disk, DB, or network. xUnit's new-instance-per-test already isolates state; don't reintroduce sharing via `static` fields.
- **Don't test private methods.** Test them *through* the public method that calls them. A private method that's hard to reach via the public API is a hint it should be its own class. Don't reach for reflection to test privates — same advice as Java.
- **Test behavior, not implementation.** Assert on outcomes (return values, interactions), not internal field values. Tests coupled to implementation break on every refactor and defeat the purpose.
- **Arrange the minimum.** Use builders/defaults so each test sets only what it needs.
- **Name clearly** (see section 14) and keep the AAA structure visible.
- **Prefer mocking interfaces.** Moq can only mock virtual/interface members — program to interfaces and your code becomes testable by design (same lesson as Mockito).

---

## Common Interview Questions

**1. What's the difference between `[Fact]` and `[Theory]`?**
`[Fact]` is a single test with no parameters (like JUnit's `@Test`). `[Theory]` is a parameterized/data-driven test (like `@ParameterizedTest`); it runs the same body once per data row supplied by `[InlineData]`, `[MemberData]`, or `[ClassData]`. Each row shows as a separate result in the runner.

**2. How does xUnit handle setup and teardown without `@BeforeEach`/`@AfterEach`?**
xUnit creates a **new instance of the test class per test method**, so the **constructor** acts as `@BeforeEach` and implementing **`IDisposable.Dispose()`** acts as `@AfterEach`. For once-per-class setup (like `@BeforeAll`), use **`IClassFixture<T>`**; for sharing across multiple classes, use **`ICollectionFixture<T>`**.

**3. Why does xUnit create a new test class instance for each test?**
To guarantee **test isolation** — tests can't accidentally share mutable state through instance fields, so order and side effects can't cause flaky failures. JUnit 5 defaults to the same per-method lifecycle.

**4. What's the difference between a mock, a stub, and a fake?**
A **stub** returns canned data and you don't assert on it (`Setup().Returns()`). A **mock** is a stub you also verify interactions on (`Verify()`) — used when the *call itself* is the behavior under test (e.g. "was the email sent?"). A **fake** is a real lightweight implementation (e.g. an in-memory DB). In Moq, the same `Mock<T>` is a stub or a mock depending on whether you call `Verify()`.

**5. How do you mock a method with Moq and verify it was called?**
`var m = new Mock<IRepo>(); m.Setup(r => r.Get(1)).Returns(user);` then inject `m.Object`. Afterward, `m.Verify(r => r.Get(1), Times.Once);`. This maps to Mockito's `when(...).thenReturn(...)` and `verify(..., times(1))`.

**6. What are `It.IsAny<T>()` and `It.Is<T>(...)`?**
Argument matchers. `It.IsAny<int>()` matches *any* int (Mockito's `any()`); `It.Is<int>(x => x > 0)` matches a *condition* (Mockito's `argThat(...)`). Gotcha: if you use a matcher for one argument in a call, you must use matchers for all arguments.

**7. How do you test asynchronous code in xUnit?**
Make the test method **`async Task`** (never `async void`) and `await` the call directly. Set up async mocks with `.ReturnsAsync(...)` / `.ThrowsAsync(...)`. Assert async exceptions with `Assert.ThrowsAsync<T>` or FluentAssertions' `await act.Should().ThrowAsync<T>()`.

**8. How do you test that a method throws an exception?**
Built-in: `Assert.Throws<TException>(() => sut.Method())` (returns the exception for further inspection). FluentAssertions: `Action act = () => sut.Method(); act.Should().Throw<TException>().WithMessage("*...*");`. Equivalent to JUnit's `assertThrows`.

**9. Why use FluentAssertions over built-in `Assert`?**
Readability and **far better failure messages**. `value.Should().BeGreaterThan(18)` reports the actual value and expectation in plain English, whereas `Assert.True(value > 18)` just says "Expected True, got False." It's the .NET analogue of preferring AssertJ over raw JUnit asserts.

**10. How do you write an integration test for an ASP.NET Core API?**
Use **`WebApplicationFactory<Program>`** (from `Microsoft.AspNetCore.Mvc.Testing`) to boot the whole app in memory, get an `HttpClient` via `CreateClient()`, and send real HTTP requests, asserting on status and JSON body. Override `ConfigureTestServices` to swap real dependencies (e.g. the database) for test ones. It's the .NET equivalent of `@SpringBootTest` + `MockMvc`.

**11. Should you test private methods? How?**
No — test them **indirectly through the public API** that uses them. If a private method is too complex to cover that way, that's a design smell suggesting it belongs in its own class. Avoid reflection-based private testing; it couples tests to implementation.

**12. What's a test data builder and why use it?**
A helper class with fluent `With...()` methods and sensible defaults that constructs test objects. It keeps Arrange sections short, lets each test set only the field it cares about, and means schema changes are fixed in one place instead of across many tests. It's the Builder pattern applied to test setup.

**13. Difference between EF Core InMemory and SQLite in-memory for tests?**
**EF Core InMemory** is fast but isn't a relational engine — it ignores SQL constraints, foreign keys, and unique indexes. **SQLite in-memory** is a real SQL engine, so it catches relational issues the InMemory provider silently allows. Use SQLite when you need real SQL behavior. Analogous to a naive fake repo vs. H2 in Java.

**14. What does good test isolation require, and what threatens it?**
Each test must run independently, in any order, with no shared mutable state. Threats: `static` fields, real `DateTime.Now`/`Random`/`Guid`, shared external resources, and order dependencies. Fix by injecting clocks/`TimeProvider`, using unique DB names per test, and relying on xUnit's per-test instances.

---

## Quick Reference Cheat Sheet

```text
TEST ATTRIBUTES
  [Fact]                         single test                       (JUnit @Test)
  [Theory] + [InlineData(...)]   parameterized, inline data        (@ParameterizedTest + @CsvSource)
  [Theory] + [MemberData(name)]  data from static property/method  (@MethodSource)
  [Theory] + [ClassData(type)]   data from a class                 (custom ArgumentsProvider)

LIFECYCLE
  constructor                    before each test                  (@BeforeEach)
  IDisposable.Dispose()          after each test                   (@AfterEach)
  IClassFixture<T>               once per test class               (@BeforeAll/@AfterAll)
  ICollectionFixture<T>          once across many classes          (—)
  (xUnit = NEW class instance per test → automatic isolation)

BUILT-IN ASSERTIONS
  Assert.Equal(expected, actual)          // expected FIRST
  Assert.True / False / Null / NotNull
  Assert.Contains / Empty / IsType<T>
  Assert.Throws<T>(() => ...)             // Assert.ThrowsAsync<T> for async

FLUENTASSERTIONS (preferred)
  result.Should().Be(5);
  result.Should().BeGreaterThan(0).And.BeLessThan(10);
  obj.Should().NotBeNull();
  list.Should().HaveCount(3).And.Contain(x).And.BeInAscendingOrder();
  actual.Should().BeEquivalentTo(expected);     // deep property equality
  Action a = () => sut.Run();
  a.Should().Throw<T>().WithMessage("*msg*");
  await act.Should().ThrowAsync<T>();

MOQ (mocking)
  var m = new Mock<IService>();
  m.Setup(x => x.Get(1)).Returns(value);          // when(...).thenReturn(...)
  m.Setup(x => x.GetAsync(1)).ReturnsAsync(value);
  m.Setup(x => x.Get(It.IsAny<int>())).Returns(v);// any()
  m.Setup(x => x.Get(It.Is<int>(i => i > 0)))...  // argThat(...)
  m.Setup(x => x.Get(1)).Throws(new Ex());        // thenThrow(...)
  m.Setup(x => x.Save(It.IsAny<T>()))
   .Callback<T>(t => captured = t);               // ArgumentCaptor / thenAnswer
  m.SetupSequence(...).Returns(a).Returns(b);     // thenReturn(a).thenReturn(b)
  m.SetupProperty(x => x.Prop, init);
  var sut = new Sut(m.Object);                    // .Object = the instance to inject
  m.Verify(x => x.Get(1), Times.Once);            // verify(..., times(1))
  // Times: Once Never Exactly(n) AtLeastOnce AtLeast(n) AtMost(n)

ASYNC TESTS
  public async Task Method() { await sut.Op(); }  // return Task, NEVER async void

INTEGRATION (ASP.NET Core)
  class T : IClassFixture<WebApplicationFactory<Program>>
  var client = factory.CreateClient();            // ≈ MockMvc (real HTTP, in-memory)
  await client.GetAsync("/api/x");
  await client.PostAsJsonAsync("/api/x", body);
  // override ConfigureTestServices to swap DB/deps (≈ @MockBean)

IN-MEMORY DB
  .UseInMemoryDatabase(Guid.NewGuid().ToString()) // fast, NOT real SQL
  .UseSqlite("DataSource=:memory:")               // real SQL engine

CLI
  dotnet new xunit -n MyApp.Tests
  dotnet add MyApp.Tests package Moq
  dotnet add MyApp.Tests package FluentAssertions
  dotnet test
  dotnet test --filter "FullyQualifiedName~Calculator"
  dotnet test --collect:"XPlat Code Coverage"     // coverlet (≈ JaCoCo)

NAMING:  MethodName_Scenario_ExpectedResult
PATTERN: Arrange → Act → Assert (Given → When → Then)
RULES:   one concept/test · no logic in tests · deterministic · fast · isolated · don't test privates
```

---

*Last Updated: 2026-06-16*

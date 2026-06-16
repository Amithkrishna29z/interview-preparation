# Software Engineering Principles Interview Questions

> 🎯 These questions appear in almost every junior developer interview. Easy explanations inside!

---

## Table of Contents
1. [SDLC (Software Development Life Cycle)](#sdlc)
2. [Agile Methodology](#agile-methodology)
3. [Scrum Framework](#scrum-framework)
4. [Kanban](#kanban)
5. [SOLID Principles](#solid-principles)
6. [Design Patterns](#design-patterns)
7. [Clean Code Principles](#clean-code-principles)
8. [Testing Principles](#testing-principles)
9. [Quick Revision Summary](#quick-revision-summary)

---

## SDLC

### Q1: What is SDLC and what are its phases?

**Easy Explanation:** SDLC is the step-by-step process used to plan, create, test, and deliver software — like a recipe for building software.

```
Phase 1: Planning
  → What are we building? How long? How much cost?
  → Define scope, timeline, budget, team

Phase 2: Requirements Analysis
  → What should the software DO?
  → Gather requirements from client/stakeholders
  → Create BRS (Business Requirements Specification)

Phase 3: System Design
  → HOW will we build it?
  → Architecture, database design, tech stack, UI wireframes

Phase 4: Implementation (Coding)
  → Developers write code
  → Follow design documents and coding standards

Phase 5: Testing
  → Does the software work correctly?
  → Unit tests, integration tests, UAT (User Acceptance Testing)

Phase 6: Deployment
  → Release to production (live environment)
  → CI/CD pipeline, monitoring

Phase 7: Maintenance
  → Fix bugs, add features, performance improvements
  → Ongoing support
```

**SDLC Models:**

| Model | Description | Best For |
|-------|-------------|---------|
| **Waterfall** | Sequential phases, no going back | Fixed requirements, simple projects |
| **Agile** | Iterative, flexible, delivery in sprints | Changing requirements, modern apps |
| **Spiral** | Risk-driven, iterative | Large, complex, risky projects |
| **V-Model** | Testing paired with each dev phase | Safety-critical systems |

---

### Q2: What is the difference between Waterfall and Agile?

**Easy Analogy:**
- **Waterfall** = Building a house. You finish all planning FIRST, then build. No changes mid-way.
- **Agile** = Renovating a house room by room. Deliver one room at a time, adjust based on feedback.

| Feature | Waterfall | Agile |
|---------|-----------|-------|
| **Delivery** | One big delivery at end | Small deliveries every 2-4 weeks |
| **Requirements** | Fixed upfront | Can change anytime |
| **Testing** | Only after development | Continuous throughout |
| **Customer** | Involved at start and end | Involved throughout |
| **Risk** | High (discovered late) | Low (discovered early) |
| **Documentation** | Heavy | Light (working software > docs) |
| **Team** | Specialists in silos | Cross-functional teams |
| **Best for** | Fixed, clear requirements | Evolving requirements |

---

## Agile Methodology

### Q3: What is Agile and what are its core values?

**Easy Explanation:** Agile is a **mindset** for software development that values flexibility, collaboration, and delivering working software frequently rather than following a strict plan.

**The 4 Agile Values** (from the Agile Manifesto):
```
1. Individuals and interactions  OVER  processes and tools
2. Working software              OVER  comprehensive documentation
3. Customer collaboration        OVER  contract negotiation
4. Responding to change          OVER  following a plan
```

> 💡 **Key Point:** The items on the LEFT are valued more. But the items on the RIGHT still matter!

**12 Agile Principles (Simplified):**
```
1. Deliver working software frequently (weeks, not months)
2. Welcome changing requirements, even late in development
3. Business people and developers work together daily
4. Build projects around motivated individuals
5. Face-to-face conversation is the best communication
6. Working software is the primary measure of progress
7. Sustainable development pace (no burnout)
8. Technical excellence and good design enhances agility
9. Simplicity — maximize work NOT done
10. Self-organizing teams produce best architectures
11. Reflect regularly and adjust behavior
12. Continuous attention to technical excellence
```

---

### Q4: What are the popular Agile frameworks?

| Framework | Best For | Key Feature |
|-----------|---------|------------|
| **Scrum** | Product development | Fixed sprints, defined roles |
| **Kanban** | Operations, support | Visual board, continuous flow |
| **XP (Extreme Programming)** | Tech-focused teams | TDD, pair programming, refactoring |
| **SAFe (Scaled Agile)** | Large enterprises | Multiple teams, synchronized |
| **Lean** | Waste elimination | Remove non-value adding activities |

---

## Scrum Framework

### Q5: What is Scrum? Explain its key components.

**Easy Explanation:** Scrum is the most popular Agile framework. Think of it like playing cricket:
- You have a **team** with specific roles
- You play in **fixed time periods** (sprints = innings)
- You have a **product backlog** (all tasks to do = full match plan)
- You have **daily standups** (team huddles)
- You **review and improve** after each sprint

---

### Q6: What are the 3 Scrum Roles?

```
1. PRODUCT OWNER (PO)
   → Represents the customer/business
   → Owns and prioritizes the Product Backlog
   → Decides WHAT to build and in what ORDER
   → Writes User Stories
   → "Voice of the customer"

2. SCRUM MASTER (SM)
   → Facilitates the Scrum process
   → Removes impediments/blockers for the team
   → Coaches team on Scrum practices
   → NOT a project manager! (doesn't give orders)
   → "Servant-leader"

3. DEVELOPMENT TEAM
   → Cross-functional (developers, testers, designers)
   → Self-organizing (decides HOW to build)
   → 3-9 people (small teams work best)
   → Collectively responsible for delivery
```

---

### Q7: What are the Scrum Artifacts?

**Easy Explanation:** Artifacts = important documents/lists in Scrum.

```
1. PRODUCT BACKLOG
   → Master list of ALL features/requirements for the product
   → Owned and maintained by Product Owner
   → Ordered by priority (most important at top)
   → Items are called "User Stories" or "PBIs"
   → Never complete — always evolving
   
   Example items:
   - As a user, I want to login with email and password
   - As an admin, I want to see all user reports
   - Fix the payment bug on checkout page

2. SPRINT BACKLOG
   → Subset of Product Backlog items chosen for THIS sprint
   → Team commits to completing these items
   → Created during Sprint Planning
   → Team OWNS this, not the PO

3. INCREMENT (Product Increment)
   → Working software delivered at end of each sprint
   → Must be "Done" (meets Definition of Done)
   → Potentially shippable to production
   → Each sprint adds to previous increments
```

---

### Q8: What are the Scrum Events (Ceremonies)?

```
1. SPRINT
   → Fixed time-box (usually 2 weeks, max 4 weeks)
   → Goal: Deliver a potentially shippable product increment
   → No scope changes during sprint
   → Same length throughout the project

2. SPRINT PLANNING (Start of sprint - max 8 hours for 1-month sprint)
   → WHAT: Team selects items from Product Backlog
   → HOW: Team plans how to complete selected items
   → Outputs: Sprint Goal + Sprint Backlog

3. DAILY SCRUM / DAILY STANDUP (Every day - max 15 min)
   → Same time, same place every day
   → Each team member answers:
     ✅ What did I do YESTERDAY?
     ✅ What will I do TODAY?
     ✅ Are there any BLOCKERS/IMPEDIMENTS?
   → NOT a status meeting for managers — it's for the team!

4. SPRINT REVIEW (End of sprint - max 4 hours)
   → Team DEMONSTRATES completed work to stakeholders
   → Product Owner accepts or rejects items
   → Stakeholders give feedback
   → Update Product Backlog based on feedback

5. SPRINT RETROSPECTIVE (After Sprint Review - max 3 hours)
   → Inspect HOW the team worked (process, not product)
   → Three questions:
     🟢 What went WELL?
     🔴 What could be IMPROVED?
     🎯 What will we COMMIT to improving next sprint?
   → Continuous process improvement
```

---

### Q9: What is a User Story?

**Easy Explanation:** A user story describes a feature from the user's perspective in simple, non-technical language.

**Format:**
```
As a [type of user],
I want [to do something],
So that [I get some benefit/value].
```

**Examples:**
```
As a customer,
I want to add items to my cart,
So that I can purchase multiple items at once.

As an admin,
I want to view all registered users,
So that I can manage the user database.

As a developer,
I want API documentation,
So that I can understand how to integrate the services.
```

**INVEST Criteria** (good user story checklist):
```
I — Independent   (can be developed independently)
N — Negotiable    (details can be discussed)
V — Valuable      (delivers value to user)
E — Estimable     (team can estimate effort)
S — Small         (can be done in one sprint)
T — Testable      (can be verified with tests)
```

**Acceptance Criteria:**
```
User Story: As a user, I want to reset my password.

Acceptance Criteria:
✅ User can enter registered email address
✅ System sends reset email within 2 minutes
✅ Reset link expires after 24 hours
✅ User can set new password (min 8 chars)
✅ Old password no longer works after reset
✅ User sees success message after reset
```

---

### Q10: What is Story Points and Sprint Velocity?

**Easy Explanation:**
- **Story Points** = Estimate of effort/complexity (not hours!)
- **Velocity** = How many story points a team completes per sprint (team speed)

```
Story Point Scale (Fibonacci): 1, 2, 3, 5, 8, 13, 21...
Why Fibonacci? Larger tasks = more uncertainty = larger gaps

Examples:
- Fix a typo in UI → 1 point (trivial)
- Add a new form field → 2 points (simple)
- Add email validation → 3 points (small effort)
- Build login API with JWT → 8 points (medium complexity)
- Build entire payment module → 21 points (large, complex)

Velocity Example:
Sprint 1: Completed 30 story points
Sprint 2: Completed 35 story points
Sprint 3: Completed 28 story points
Average Velocity: ~31 points/sprint

→ If Product Backlog = 150 points
→ Estimated sprints needed = 150/31 ≈ 5 sprints
→ At 2 weeks/sprint = ~10 weeks to complete
```

**Planning Poker:**
- Team estimates together (avoids anchoring bias)
- Each person shows card simultaneously
- Discuss differences, re-estimate if needed
- Builds shared understanding of the work

---

### Q11: What is Definition of Done (DoD)?

**Easy Explanation:** DoD is a shared checklist that defines what "finished" means for every task. If any item on the list is not done, the task is NOT done.

```
Example Definition of Done:
✅ Code is written and reviewed (code review passed)
✅ Unit tests written (80%+ coverage)
✅ Integration tests pass
✅ No critical bugs
✅ Code committed to Git repository
✅ Deployed to staging environment
✅ Product Owner has reviewed and accepted
✅ Documentation updated
✅ No merge conflicts

Why it matters:
- Prevents "90% done" syndrome
- Ensures quality standards are consistent
- Clear, shared understanding across team
```

---

## Kanban

### Q12: What is Kanban and how is it different from Scrum?

**Easy Explanation:** Kanban = visual board showing work flowing through stages. Like a to-do list on a board with columns.

**Kanban Board Example:**
```
┌──────────┬──────────────┬─────────────┬──────────┐
│ BACKLOG  │  IN PROGRESS │   TESTING   │   DONE   │
│          │   (WIP: 3)   │   (WIP: 2)  │          │
├──────────┼──────────────┼─────────────┼──────────┤
│ Task A   │   Task D     │   Task G    │ Task I   │
│ Task B   │   Task E     │   Task H    │ Task J   │
│ Task C   │   Task F     │             │ Task K   │
└──────────┴──────────────┴─────────────┴──────────┘
```

**Key Kanban Concepts:**
- **WIP Limit** (Work In Progress): Max tasks allowed in each column. Prevents multitasking overload.
- **Pull System**: Team pulls work when ready (not pushed by manager)
- **Continuous Flow**: No fixed sprints, work flows continuously

**Scrum vs Kanban:**

| | Scrum | Kanban |
|--|-------|--------|
| **Cadence** | Fixed sprints (2-4 weeks) | Continuous flow |
| **Roles** | PO, SM, Dev Team | No prescribed roles |
| **Changes** | No changes mid-sprint | Can change anytime |
| **Meetings** | Defined ceremonies | No required meetings |
| **Focus** | Deliver sprint goal | Optimize flow |
| **Best for** | Product development | Operations, support, maintenance |
| **Estimation** | Story points | Cycle time |

---

## SOLID Principles

### Q13: What are SOLID principles? (Very commonly asked!)

**Easy Explanation:** SOLID = 5 rules for writing clean, maintainable Java code.

---

**S — Single Responsibility Principle (SRP)**
> A class should have only ONE reason to change.

```java
// ❌ BAD: One class doing too many things
class UserManager {
    public void createUser(User user) { ... }
    public void sendWelcomeEmail(User user) { ... }  // Email logic here!
    public void saveToDatabase(User user) { ... }    // DB logic here!
    public String generateReport() { ... }           // Report logic here!
}

// ✅ GOOD: Each class has ONE job
class UserService {
    public void createUser(User user) { ... }
}
class EmailService {
    public void sendWelcomeEmail(User user) { ... }
}
class UserRepository {
    public void save(User user) { ... }
}
class ReportService {
    public String generateReport() { ... }
}
```

---

**O — Open/Closed Principle (OCP)**
> Open for EXTENSION, closed for MODIFICATION. Add new features without changing existing code.

```java
// ❌ BAD: Must modify existing code to add new payment type
class PaymentProcessor {
    public void process(String type, double amount) {
        if (type.equals("CREDIT_CARD")) { ... }
        else if (type.equals("PAYPAL")) { ... }
        // Must edit here to add new payment!
    }
}

// ✅ GOOD: Add new payment by creating new class (no editing existing)
interface PaymentMethod {
    void process(double amount);
}

class CreditCardPayment implements PaymentMethod {
    public void process(double amount) { /* credit card logic */ }
}
class PayPalPayment implements PaymentMethod {
    public void process(double amount) { /* paypal logic */ }
}
// Add new payment? Just create: class CryptoPayment implements PaymentMethod {}
```

---

**L — Liskov Substitution Principle (LSP)**
> Subclasses should be replaceable for their parent class without breaking anything.

```java
// ❌ BAD: Square breaks Rectangle behavior (LSP violation)
class Rectangle {
    protected int width, height;
    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int area() { return width * height; }
}

class Square extends Rectangle {
    public void setWidth(int w) {
        this.width = w;
        this.height = w; // Forces height = width (breaks parent contract!)
    }
}

// If code expects Rectangle but gets Square:
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(10);
System.out.println(r.area()); // Expected 50, gets 100! ❌

// ✅ GOOD: Use separate classes or interface
interface Shape {
    int area();
}
class Rectangle implements Shape { ... }
class Square implements Shape { ... }
```

---

**I — Interface Segregation Principle (ISP)**
> Don't force classes to implement methods they don't need. Make small, specific interfaces.

```java
// ❌ BAD: Fat interface forces unnecessary methods
interface Animal {
    void eat();
    void fly();   // Not all animals fly!
    void swim();  // Not all animals swim!
    void run();
}
class Dog implements Animal {
    public void eat() { ... }
    public void fly() { throw new UnsupportedOperationException(); } // Dogs can't fly!
    public void swim() { ... }
    public void run() { ... }
}

// ✅ GOOD: Small, specific interfaces
interface Eatable { void eat(); }
interface Flyable { void fly(); }
interface Swimmable { void swim(); }
interface Runnable { void run(); }

class Dog implements Eatable, Swimmable, Runnable { ... }  // Only what Dog needs
class Bird implements Eatable, Flyable, Runnable { ... }    // Only what Bird needs
```

---

**D — Dependency Inversion Principle (DIP)**
> Depend on ABSTRACTIONS (interfaces), not on CONCRETE implementations.

```java
// ❌ BAD: High-level class depends on low-level class directly
class UserService {
    private MySQLUserRepository repository = new MySQLUserRepository(); // Tightly coupled!
    // Can't easily switch to PostgreSQL or MongoDB
}

// ✅ GOOD: Depend on interface (abstraction)
interface UserRepository {
    User findById(Long id);
    void save(User user);
}

class MySQLUserRepository implements UserRepository { ... }
class MongoUserRepository implements UserRepository { ... }

class UserService {
    private UserRepository repository; // Depends on interface!

    // Inject via constructor (Spring @Autowired does this)
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
    // Can easily swap MySQL for MongoDB without changing UserService!
}
```

---

## Design Patterns

### Q14: What are Design Patterns? Name the most common ones.

**Easy Explanation:** Design patterns = proven solutions to common programming problems. Like cooking recipes — you don't need to invent from scratch.

**Categories:**

| Category | Purpose | Examples |
|----------|---------|---------|
| **Creational** | How objects are created | Singleton, Factory, Builder |
| **Structural** | How objects are composed | Adapter, Facade, Decorator |
| **Behavioral** | How objects communicate | Observer, Strategy, Template Method |

---

**Singleton Pattern** (Most asked!)
```java
// Only ONE instance of this class can exist
// Use case: Database connection, Configuration, Logger

public class DatabaseConnection {
    private static DatabaseConnection instance; // The one instance

    private DatabaseConnection() {} // Private constructor — no one can call new!

    public static DatabaseConnection getInstance() {
        if (instance == null) {
            instance = new DatabaseConnection();
        }
        return instance;
    }

    // Thread-safe version
    public static synchronized DatabaseConnection getInstanceSafe() {
        if (instance == null) {
            instance = new DatabaseConnection();
        }
        return instance;
    }
}

// Usage
DatabaseConnection conn1 = DatabaseConnection.getInstance();
DatabaseConnection conn2 = DatabaseConnection.getInstance();
System.out.println(conn1 == conn2); // true — same object!
```

---

**Factory Pattern**
```java
// Create objects without specifying the exact class
// Use case: Creating different notification types

interface Notification {
    void send(String message);
}

class EmailNotification implements Notification {
    public void send(String message) { System.out.println("Email: " + message); }
}

class SMSNotification implements Notification {
    public void send(String message) { System.out.println("SMS: " + message); }
}

// Factory — creates the right object based on type
class NotificationFactory {
    public static Notification create(String type) {
        return switch (type) {
            case "EMAIL" -> new EmailNotification();
            case "SMS"   -> new SMSNotification();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}

// Usage — caller doesn't care which class is created
Notification n = NotificationFactory.create("EMAIL");
n.send("Welcome to our app!"); // Email: Welcome to our app!
```

---

**Builder Pattern**
```java
// Build complex objects step by step
// Use case: Creating objects with many optional fields

public class User {
    private String name;     // required
    private String email;    // required
    private String phone;    // optional
    private String address;  // optional
    private int age;         // optional

    private User() {} // Private constructor

    // Builder class
    public static class Builder {
        private String name;
        private String email;
        private String phone;
        private String address;
        private int age;

        public Builder(String name, String email) { // required fields
            this.name = name;
            this.email = email;
        }
        public Builder phone(String phone) { this.phone = phone; return this; }
        public Builder address(String address) { this.address = address; return this; }
        public Builder age(int age) { this.age = age; return this; }

        public User build() {
            User user = new User();
            user.name = this.name;
            user.email = this.email;
            user.phone = this.phone;
            user.address = this.address;
            user.age = this.age;
            return user;
        }
    }
}

// Usage — readable, flexible object creation
User user = new User.Builder("Amith", "amith@gmail.com")
    .phone("9876543210")
    .age(25)
    .build();  // address is optional, not set
```

---

**Observer Pattern**
```java
// When one object changes, all dependents are notified automatically
// Use case: Event listeners, Spring ApplicationEvents

interface Observer {
    void update(String event);
}

class EventPublisher {
    private List<Observer> observers = new ArrayList<>();

    public void subscribe(Observer o) { observers.add(o); }
    public void unsubscribe(Observer o) { observers.remove(o); }

    public void publish(String event) {
        for (Observer o : observers) {
            o.update(event); // Notify all subscribers
        }
    }
}

// In Spring Boot context:
@Component
public class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public void placeOrder(Order order) {
        orderRepository.save(order);
        publisher.publishEvent(new OrderPlacedEvent(order)); // Notify listeners
    }
}

@Component
public class EmailListener {
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        emailService.sendConfirmation(event.getOrder());
    }
}
```

---

## Clean Code Principles

### Q15: What is Clean Code? (Robert C. Martin — "Uncle Bob")

```
1. MEANINGFUL NAMES
   ❌ int d;  String s;  void calc();
   ✅ int daysRemaining;  String userName;  void calculateTax();

2. SMALL FUNCTIONS
   ❌ One function that does 50 things
   ✅ Each function does ONE thing, fits on screen

3. DON'T REPEAT YOURSELF (DRY)
   ❌ Same logic copied in 5 places
   ✅ Extract to a shared method/utility

4. COMMENTS (only when necessary)
   ❌ // Get user by id
      User user = userRepo.findById(id);  // Comment explains obvious code
   ✅ // Using batch size of 1000 to avoid memory overflow on large datasets
      // (Good comment explains WHY, not WHAT)

5. ERROR HANDLING
   ❌ Return null when not found (causes NullPointerException)
   ✅ Throw meaningful exceptions or return Optional<>

6. NO MAGIC NUMBERS
   ❌ if (status == 2) { ... }
   ✅ if (status == UserStatus.ACTIVE) { ... }
      // Or: private static final int MAX_RETRY = 3;
```

---

## Testing Principles

### Q16: What is TDD (Test-Driven Development)?

**Easy Explanation:** TDD = Write the TEST before writing the actual CODE.

```
TDD Cycle (Red → Green → Refactor):

Step 1: RED   → Write a failing test (code doesn't exist yet)
Step 2: GREEN → Write minimum code to make test pass
Step 3: REFACTOR → Improve code quality (test still passes)
Repeat!

Example:
// Step 1: Write test FIRST
@Test
void shouldCalculateTaxCorrectly() {
    TaxCalculator calc = new TaxCalculator();
    assertEquals(18.0, calc.calculate(100.0, 0.18)); // This FAILS (class doesn't exist)
}

// Step 2: Write code to make test PASS
class TaxCalculator {
    double calculate(double amount, double rate) {
        return amount * rate;
    }
}

// Step 3: Refactor if needed (test still passes)
```

---

### Q17: What is the Testing Pyramid?

```
        /\
       /  \   E2E Tests (few, slow, expensive)
      /----\  → Test complete user flows in browser
     /      \ Integration Tests (some)
    /--------\ → Test multiple components together
   /          \ Unit Tests (many, fast, cheap)
  /____________\ → Test individual methods/classes

Unit Tests:   70% — Test single method/class in isolation
Integration:  20% — Test API endpoints, DB interactions
E2E Tests:    10% — Test full user flows (Selenium, Cypress)
```

**In Spring Boot:**
```java
// Unit Test (JUnit 5 + Mockito)
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;  // Mock the dependency

    @InjectMocks
    private UserService userService;  // Inject mocks into service

    @Test
    void shouldReturnUserWhenFound() {
        User user = new User(1L, "Amith", "amith@gmail.com");
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));

        User result = userService.findById(1L);

        assertNotNull(result);
        assertEquals("Amith", result.getName());
    }
}

// Integration Test (Spring Boot Test)
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldReturnUserList() throws Exception {
        mockMvc.perform(get("/api/users"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.length()").value(greaterThan(0)));
    }
}
```

---

## Quick Revision Summary

### 🔑 Scrum in One Table

| Item | Description |
|------|-------------|
| **Sprint** | 2-4 week iteration |
| **Product Backlog** | All features list (PO owns) |
| **Sprint Backlog** | This sprint's tasks (Team owns) |
| **Daily Standup** | 15 min, 3 questions, every day |
| **Sprint Review** | Demo to stakeholders |
| **Sprint Retro** | Improve the process |
| **Product Owner** | What to build |
| **Scrum Master** | Remove blockers |
| **Dev Team** | How to build it |

### 🔑 SOLID in One Line Each

| Principle | One Line |
|-----------|---------|
| **S**RP | One class = one job |
| **O**CP | Extend without modifying |
| **L**SP | Subclass must work where parent works |
| **I**SP | Small interfaces > fat interfaces |
| **D**IP | Depend on interfaces, not implementations |

### 🔑 Agile vs Scrum
- **Agile** = mindset/philosophy (the "what")
- **Scrum** = a specific framework that implements Agile (the "how")
- Scrum IS Agile, but Agile is NOT necessarily Scrum

### 📝 Common Interview Questions

**Q: What is Agile?**
> Iterative software development that delivers working software frequently and welcomes changing requirements.

**Q: What is the role of Scrum Master?**
> Facilitates the Scrum process, removes blockers, coaches the team. NOT a project manager.

**Q: What happens in Daily Standup?**
> 15-min meeting where each member answers: What did I do yesterday? What will I do today? Any blockers?

**Q: What is a Sprint?**
> A fixed time-box (1-4 weeks) where the team delivers a potentially shippable product increment.

**Q: What is Definition of Done?**
> A shared checklist that defines when a task is truly "complete" (code reviewed, tested, deployed, etc.)

**Q: Name the SOLID principles.**
> Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion.

**Q: What design pattern does Spring IoC use?**
> Dependency Inversion Principle + Factory Pattern (Spring creates and injects beans for you).

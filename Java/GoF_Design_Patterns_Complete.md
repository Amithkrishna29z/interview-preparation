# GoF Design Patterns — Java Interview Guide

All 23 Gang of Four patterns with Java code, real-world use cases, Spring usage, and interview Q&A.

> **Focus first on the 8 most-asked patterns** (bolded below): Singleton, Factory Method, Builder, Strategy, Observer, Decorator, Proxy, Template Method. The rest are "awareness" level for junior interviews.

---

## CREATIONAL PATTERNS

> How you create objects — hiding construction logic so it's flexible and clean.

### **1. Singleton**
One instance per JVM. All Spring beans are singletons by default.

> Like the President of a country — one at a time, everyone refers to the same person.

```java
// Best: Enum Singleton
public enum DatabaseConnection {
    INSTANCE;
    private final Connection conn;
    DatabaseConnection() { conn = createConnection(); }
    public Connection get() { return conn; }
}

// Thread-safe lazy: Bill Pugh (preferred)
public class ConfigManager {
    private ConfigManager() {}
    private static class Holder { static final ConfigManager INSTANCE = new ConfigManager(); }
    public static ConfigManager getInstance() { return Holder.INSTANCE; }
}
```
**Spring:** `@Scope("singleton")` (default). `@Bean` methods return the same instance.

---

### **2. Factory Method**
Define an interface for creating an object; subclasses decide which class to instantiate.

> Like ordering "a coffee" — you don't make it yourself; the café decides how.

```java
public interface Notification { void send(String message); }
public class EmailNotification implements Notification {
    public void send(String msg) { System.out.println("Email: " + msg); }
}

public abstract class NotificationService {
    public void notifyUser(String msg) { createNotification().send(msg); }
    protected abstract Notification createNotification(); // factory method
}

public class EmailNotificationService extends NotificationService {
    protected Notification createNotification() { return new EmailNotification(); }
}
```
**Spring:** `FactoryBean<T>`, `BeanFactory`, `@Bean` methods.

---

### 3. Abstract Factory
Create families of related objects without specifying concrete classes.

> Pick "Modern" furniture style — the factory gives you a matching chair, table, and sofa.

```java
public interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}
public class WindowsUIFactory implements UIFactory {
    public Button createButton() { return new WindowsButton(); }
    public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}
// Client uses UIFactory interface — doesn't know which platform it has
```
**vs Factory Method:** Abstract Factory creates a *family* of objects; Factory Method creates *one* type.  
**Spring:** DataSource abstractions, Spring Data repository factories.

---

### **4. Builder**
Construct complex objects step by step. Handles many optional parameters.

> Like building a custom burger — add each ingredient step by step, then `build()`.

```java
public class User {
    private final String name, email; // required
    private final String phone, address; // optional

    private User(Builder b) { this.name=b.name; this.email=b.email; this.phone=b.phone; this.address=b.address; }

    public static class Builder {
        private String name, email, phone, address;
        public Builder(String name, String email) { this.name=name; this.email=email; }
        public Builder phone(String p) { this.phone=p; return this; }
        public Builder address(String a) { this.address=a; return this; }
        public User build() { return new User(this); }
    }
}

User user = new User.Builder("John", "john@example.com").phone("555-1234").build();
```
**Spring:** `UriComponentsBuilder`, `ResponseEntity.ok().header(...).body(...)`. Lombok `@Builder` generates this automatically.

---

### 5. Prototype
*(Awareness)* Clone existing objects instead of creating from scratch. Useful when creation is expensive.

```java
public DocumentTemplate copy() { return new DocumentTemplate(this); } // copy constructor
```
**Spring:** `@Scope("prototype")`.

---

## STRUCTURAL PATTERNS

> How you assemble objects into bigger structures — wrapping, connecting, or simplifying them.

### 6. Adapter
Make incompatible interfaces work together.

> Like a travel plug adapter — sits between your plug and the foreign socket.

```java
public class LegacyPaymentProcessor {
    public void processPayment(double amount, String card) { /* ... */ }
}
public interface PaymentGateway { void charge(PaymentRequest request); }

public class LegacyPaymentAdapter implements PaymentGateway {
    private final LegacyPaymentProcessor legacy;
    public LegacyPaymentAdapter(LegacyPaymentProcessor l) { this.legacy = l; }
    public void charge(PaymentRequest req) {
        legacy.processPayment(req.getAmount(), req.getCardNumber());
    }
}
```
**Java:** `Arrays.asList()`, `InputStreamReader` (adapts InputStream → Reader).  
**Spring:** `HandlerAdapter` in Spring MVC, `JpaVendorAdapter`.

---

### 7. Bridge
*(Awareness)* Separate abstraction from implementation so both can vary independently.

> Like a universal remote (abstraction) + any TV brand (implementation).

```java
public abstract class Shape {
    protected Renderer renderer; // bridge to implementation
    public Shape(Renderer renderer) { this.renderer = renderer; }
}
```
**Java/Real-world:** JDBC (Java code = abstraction, driver = implementation), SLF4J.

---

### 8. Composite
*(Awareness)* Treat individual objects and compositions uniformly via a shared interface (tree structure).

> Like folders and files — asking either for its "size" just works.

```java
public interface FileSystemItem { long getSize(); }
// Directory.getSize() sums children; File.getSize() returns its own — same call works on both.
```
**Spring:** Spring Security FilterChain, UI component trees.

---

### **9. Decorator**
Add behavior dynamically without subclassing.

> Like getting dressed — jacket over shirt over person, each layer adds something.

```java
public interface Coffee { String getDescription(); double getCost(); }

public class SimpleCoffee implements Coffee {
    public String getDescription() { return "Coffee"; }
    public double getCost() { return 1.00; }
}

public abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    public CoffeeDecorator(Coffee c) { this.coffee = c; }
}

public class Milk extends CoffeeDecorator {
    public Milk(Coffee c) { super(c); }
    public String getDescription() { return coffee.getDescription() + ", Milk"; }
    public double getCost() { return coffee.getCost() + 0.25; }
}

Coffee c = new Milk(new SimpleCoffee()); // "Coffee, Milk = $1.25"
```
**Java:** `new BufferedReader(new FileReader("file.txt"))`.  
**Spring:** Spring Security filter chain, Servlet filter wrappers.

---

### 10. Facade
Simplified interface to a complex subsystem.

> Like a "Watch Movie" button — one press turns on TV, sound system, and cable box.

```java
public class HomeTheaterFacade {
    private DVDPlayer dvd; private Projector projector; private SoundSystem sound;

    public HomeTheaterFacade(DVDPlayer d, Projector p, SoundSystem s) { dvd=d; projector=p; sound=s; }

    public void watchMovie(String movie) {
        projector.on(); projector.wideScreenMode();
        sound.on(); sound.setVolume(10);
        dvd.on(); dvd.play(movie);
    }
    public void endMovie() { dvd.off(); sound.off(); projector.off(); }
}
```
**Spring:** `JdbcTemplate` (facade over JDBC boilerplate), Spring Data `Repository`, `RestTemplate`.

---

### 11. Flyweight
*(Awareness)* Share common (intrinsic) state among many objects to save memory; each holds only its unique (extrinsic) state.

> The letter "a" is stored once; only each position differs.

```java
TreeType type = TreeTypeFactory.get("Oak", "green", "rough"); // shared, reused for all Oaks
```
**Java:** String pool, `Integer.valueOf()` cache (-128 to 127).

---

### **12. Proxy**
Provide a surrogate that controls access to another object.

> Like a secretary who screens calls for a busy CEO — logs, delays, or blocks access.

```java
public interface Image { void display(); }

public class RealImage implements Image {
    private String filename;
    public RealImage(String f) { this.filename=f; loadFromDisk(); } // expensive
    private void loadFromDisk() { System.out.println("Loading: " + filename); }
    public void display() { System.out.println("Displaying: " + filename); }
}

// Virtual Proxy: defer creation until needed
public class ImageProxy implements Image {
    private String filename;
    private RealImage realImage;
    public ImageProxy(String f) { this.filename = f; }
    public void display() {
        if (realImage == null) realImage = new RealImage(filename); // lazy init
        realImage.display();
    }
}
```
**Proxy types:** Virtual (lazy loading), Protection (access control), Caching, Logging.  
**Spring:** `@Transactional`, `@Async`, `@Cacheable` all create proxies via BeanPostProcessor.

---

## BEHAVIORAL PATTERNS

> How objects communicate and divide responsibility.

### 13. Chain of Responsibility
*(Awareness)* Pass a request along a chain of handlers; each handles it or forwards it.

> Like support escalation: L1 → L2 → L3.

```java
public abstract class LogHandler {
    protected LogHandler next;
    public LogHandler setNext(LogHandler next) { this.next = next; return next; }
    public abstract void handle(String level, String message);
}
```
**Spring:** Spring Security `FilterChain`, Servlet `FilterChain`.

---

### 14. Command
*(Awareness)* Encapsulate a request as an object so it can be queued, logged, or undone.

> Like a restaurant order slip — holds the request as a "thing."

```java
public interface Command { void execute(); void undo(); }
// Invoker keeps a history stack; undo() pops and reverses the last command.
```
**Java:** `Runnable`, `Callable`. **Spring:** Spring Batch `Tasklet`, `@Async` tasks.

---

### 15. Interpreter
*(Awareness)* Define a grammar and interpret sentences in it.

> Like a calculator reading "3 + 5" — each rule is a class with `interpret()`.

```java
public interface Expression { int interpret(Map<String, Integer> context); }
// AddExpression.interpret() = left.interpret() + right.interpret()
```
**Spring:** SpEL, SQL parsing, regex engines.

---

### 16. Iterator
*(Awareness)* Sequential access to elements without exposing internal representation.

> Like a TV remote's "next channel" button.

```java
// Every Java for-each loop uses Iterator<T> internally
public Iterator<Integer> iterator() {
    return new Iterator<>() {
        int current = start;
        public boolean hasNext() { return current <= end; }
        public Integer next() { return current++; }
    };
}
```
**Java:** `Iterator<T>`, `Iterable<T>`, all Collection iterators.

---

### 17. Mediator
*(Awareness)* Reduce coupling by routing communication through a central mediator.

> Like an air-traffic controller — planes talk to the tower, not each other.

```java
public interface ChatMediator { void sendMessage(String message, User sender); }
// ChatRoom forwards each message to all other users — users never reference each other.
```
**Spring:** `ApplicationEventPublisher`, Spring MVC Controller, message brokers (Kafka, RabbitMQ).

---

### 18. Memento
*(Awareness)* Capture an object's state as a snapshot that can be restored later.

> Like a video-game save point.

```java
public Memento save() { return new Memento(content); }
public void restore(Memento m) { content = m.getState(); }
```
**Java/Spring:** `Serializable`, git commits, database transaction savepoints.

---

### **19. Observer**
Define a one-to-many dependency: when one object changes state, all dependents are notified.

> Like subscribing to a YouTube channel — all subscribers notified on upload.

```java
public interface StockObserver { void update(String stock, double price); }

public class StockMarket {
    private Map<String, Double> prices = new HashMap<>();
    private List<StockObserver> observers = new ArrayList<>();

    public void addObserver(StockObserver o) { observers.add(o); }
    public void removeObserver(StockObserver o) { observers.remove(o); }

    public void setPrice(String stock, double price) {
        prices.put(stock, price);
        observers.forEach(o -> o.update(stock, price));
    }
}

public class PriceAlertService implements StockObserver {
    private double threshold;
    public PriceAlertService(double t) { this.threshold = t; }
    public void update(String stock, double price) {
        if (price > threshold) System.out.println("ALERT: " + stock + " = " + price);
    }
}
```
**Spring:** `@EventListener` / `ApplicationEventPublisher`. Kafka/RabbitMQ are distributed observer implementations.

---

### 20. State
*(Awareness)* Let an object alter its behavior when its internal state changes.

> Like a traffic light — same object, different behavior per state.

```java
public interface OrderState { void confirm(Order o); void ship(Order o); }
// PendingState.confirm() does the work, then order.setState(new ConfirmedState()).
```
**vs Strategy:** State manages transitions and knows about other states; Strategy just swaps algorithms.  
**Spring:** `@EnableStateMachine`, `@State`, `@Transition`.

---

### **21. Strategy**
Define a family of algorithms, encapsulate each, and make them interchangeable.

> Like Google Maps routes — same goal, swap the algorithm: fastest, shortest, avoid-tolls.

```java
public interface SortStrategy { void sort(List<Integer> data); }

public class QuickSortStrategy implements SortStrategy {
    public void sort(List<Integer> data) { Collections.sort(data); }
}

public class DataProcessor {
    private SortStrategy strategy;
    public DataProcessor(SortStrategy s) { this.strategy = s; }
    public void setStrategy(SortStrategy s) { this.strategy = s; }
    public void process(List<Integer> data) { strategy.sort(data); }
}
```
**Java:** `Comparator<T>` is a strategy.  
**Spring:** `AuthenticationProvider`, `HandlerMapping`, `TransactionManager`.

---

### **22. Template Method**
Define algorithm skeleton in base class; subclasses fill in specific steps.

> Like a recipe template — fixed steps (prep, cook, serve), each dish fills in the details.

```java
public abstract class ReportGenerator {
    public final void generateReport() { // template method
        gatherData(); processData(); formatReport();
        if (shouldSendEmail()) sendEmail();
    }
    protected abstract void gatherData();
    protected abstract void processData();
    protected abstract void formatReport();
    protected boolean shouldSendEmail() { return false; } // hook
    protected void sendEmail() { System.out.println("Sending email..."); }
}

public class SalesReport extends ReportGenerator {
    protected void gatherData() { System.out.println("Fetching sales data"); }
    protected void processData() { System.out.println("Calculating totals"); }
    protected void formatReport() { System.out.println("Formatting as PDF"); }
    protected boolean shouldSendEmail() { return true; }
}
```
**Spring:** `JdbcTemplate` (you provide SQL + RowMapper; template handles connection/exceptions), `HttpServlet.service()` → `doGet()`/`doPost()`.  
**vs Strategy:** Template Method uses inheritance; Strategy uses composition.

---

### 23. Visitor
*(Awareness)* Separate an algorithm from the object structure it operates on — add new operations without modifying existing classes.

> Like a tax auditor visiting each business type — each element has `accept(visitor)`; visitor has `visitX()` per type.

```java
public interface Shape { void accept(ShapeVisitor visitor); }
public interface ShapeVisitor { void visitCircle(Circle c); void visitRectangle(Rectangle r); }
// Circle.accept() calls visitor.visitCircle(this) — double dispatch.
```
**Real-world:** Java compiler AST traversal, XML/JSON parsers, Spring's `BeanDefinitionVisitor`.

---

## Patterns Summary Table

| Pattern | Category | Problem Solved | Spring Example |
|---------|----------|----------------|----------------|
| Singleton | Creational | One shared instance | `@Bean` (default) |
| Factory Method | Creational | Delegate creation to subclass | `FactoryBean<T>` |
| Abstract Factory | Creational | Families of related objects | DataSource factories |
| Builder | Creational | Complex object with many params | `UriComponentsBuilder` |
| Prototype | Creational | Clone expensive objects | `@Scope("prototype")` |
| Adapter | Structural | Incompatible interfaces | `HandlerAdapter` |
| Bridge | Structural | Separate abstraction/impl | JDBC + drivers |
| Composite | Structural | Tree structures | SecurityFilterChain |
| Decorator | Structural | Add behavior dynamically | I/O streams, filters |
| Facade | Structural | Simplify complex subsystem | `JdbcTemplate` |
| Flyweight | Structural | Share common object state | String pool |
| Proxy | Structural | Control access, add behavior | `@Transactional`, `@Async` |
| Chain of Resp. | Behavioral | Pass request through handlers | Security FilterChain |
| Command | Behavioral | Encapsulate request as object | `Runnable`, Spring Batch |
| Interpreter | Behavioral | Interpret grammar/language | SpEL, regex |
| Iterator | Behavioral | Sequential element access | `Iterator<T>`, Streams |
| Mediator | Behavioral | Centralize communication | `ApplicationEventPublisher` |
| Memento | Behavioral | Capture/restore state | Savepoints, undo |
| Observer | Behavioral | Notify on state change | `@EventListener`, Kafka |
| State | Behavioral | Behavior by state | Spring State Machine |
| Strategy | Behavioral | Interchangeable algorithms | `Comparator`, auth providers |
| Template Method | Behavioral | Algorithm with variable steps | `JdbcTemplate`, Batch |
| Visitor | Behavioral | New op on existing structure | AST visitors |

---

## Interview Questions & Answers

**Q1: What are design patterns? Why use them?**
Reusable solutions to recurring design problems, distilled from experienced developers. They improve communication (shared vocabulary), reduce design time, and make code more maintainable. Drawback: overuse adds unnecessary complexity.

**Q2: What is the difference between Factory Method and Abstract Factory?**
Factory Method defines one method for creating one type of object; subclasses decide the concrete class. Abstract Factory provides an interface for creating *families* of related objects. Abstract Factory is often implemented using multiple Factory Methods.

**Q3: When would you use Builder instead of a constructor?**
When a class has many optional parameters (4+), when construction requires multiple steps, or when required vs optional fields need to be explicit. Builder avoids telescoping constructors.

**Q4: What is the difference between Decorator and Proxy?**
Decorator adds new behavior transparently (client knows it's decorated). Proxy controls access (lazy init, security, caching) — the client may not know it's talking to a proxy. Java I/O uses Decorator; Spring AOP uses Proxy.

**Q5: What is the difference between Strategy and Template Method?**
Strategy uses composition (inject the algorithm as a dependency; swap at runtime). Template Method uses inheritance (skeleton in base class, steps in subclasses; fixed at compile time). Strategy is more flexible; Template Method is simpler.

**Q6: What design patterns does Spring use internally?**
Proxy (`@Transactional`, `@Async`, `@Cacheable`), Template Method (`JdbcTemplate`), Observer (`@EventListener`), Factory Method (`BeanFactory`), Singleton (default bean scope), Decorator (Security filters), Chain of Responsibility (FilterChain), Facade (`JdbcTemplate`, Spring Data), Strategy (auth providers), Mediator (`ApplicationEventPublisher`).

**Q7: How does @Transactional use the Proxy pattern?**
At startup, Spring wraps each `@Transactional` bean in a CGLIB or JDK dynamic proxy. At runtime, the proxy intercepts the method call, starts a transaction, calls the real method, then commits or rolls back. The caller talks to the proxy, not the real object.

**Q8: How does JdbcTemplate use Template Method?**
`JdbcTemplate.query()` defines the fixed algorithm: get connection, create statement, execute SQL, handle ResultSet, release resources. You provide the variable parts: the SQL string and the `RowMapper` lambda.

**Q9: What is the Chain of Responsibility? Where is it in Spring Security?**
Each handler processes a request or passes it to the next. Spring Security's filter chain is exactly this: each `SecurityFilter` handles authentication, authorization, CSRF, etc., or passes to the next filter.

**Q10: When should you NOT use a design pattern?**
When it adds unnecessary complexity for a simple problem. YAGNI — don't abstract for hypothetical future needs. A simple if-else often beats a full Strategy for 2 cases; a direct constructor beats Builder for 1-2 parameters.

---

## Quick Reference Cheat Sheet

```
CREATIONAL — how objects are created
  Singleton         → one President per country (one shared instance)
  Factory Method    → order "a coffee"; café decides which barista makes it
  Abstract Factory  → pick a furniture STYLE; get a matching set (chair+table+sofa)
  Builder           → build a custom burger step by step, then build()
  Prototype         → photocopy a filled-in form and tweak it

STRUCTURAL — how objects are assembled
  Adapter           → travel plug adapter (incompatible shapes work together)
  Bridge            → TV + universal remote vary independently
  Composite         → folders contain files OR folders; treat them the same
  Decorator         → add a jacket, then a scarf (layers of behavior)
  Facade            → one "Watch Movie" button hides TV+sound+cable
  Flyweight         → the letter "a" stored once, reused everywhere
  Proxy             → secretary screens calls to a busy CEO

BEHAVIORAL — how objects talk and behave
  Chain of Resp.    → support escalation: L1 → L2 → L3
  Command           → restaurant order slip (queue/log/undo a request)
  Interpreter       → calculator reading "3 + 5"
  Iterator          → remote "next channel" button
  Mediator          → air-traffic controller (everyone talks to it, not each other)
  Memento           → video-game save point
  Observer          → YouTube subscription notifications
  State             → traffic light behaves differently per state
  Strategy          → Google Maps: fastest vs shortest vs avoid-tolls
  Template Method   → recipe template (fixed steps, fill the blanks)
  Visitor           → tax auditor visits each business type
```

**The 8 patterns interviews ask most — know these from memory:**  
`Singleton · Factory Method · Builder · Strategy · Observer · Decorator · Proxy · Template Method`

**Fast "which pattern?" decision guide:**

| If the problem is... | Reach for |
|---|---|
| Need exactly one shared instance | **Singleton** |
| Too many constructor arguments / optional fields | **Builder** |
| Creating objects but the exact type varies | **Factory Method** |
| Two incompatible interfaces must connect | **Adapter** |
| Add features without touching the original class | **Decorator** |
| Hide a messy subsystem behind one simple call | **Facade** |
| Control access / add behavior around a real object | **Proxy** |
| Swap an algorithm at runtime | **Strategy** |
| Notify many objects when one changes | **Observer** |
| Same steps, different details per subclass | **Template Method** |
| Behavior depends on a changing internal status | **State** |

**Most common confusions:**
- **Decorator vs Proxy** — Decorator *adds* behavior (you know it's wrapped); Proxy *controls access* (you may not know it's there).
- **Strategy vs State** — Strategy = *you* pick the algorithm; State = the object switches its own behavior as status changes.
- **Strategy vs Template Method** — Strategy uses *composition* (swap an object); Template Method uses *inheritance* (override steps).
- **Adapter vs Facade** — Adapter makes *incompatible* things fit (1-to-1); Facade *simplifies* something complex (1-to-many).
- **Observer vs Mediator** — Observer = subscribers listen to one subject; Mediator = everyone routes through a central hub.

**Patterns Spring uses (name these in interviews):**  
`@Bean` = Singleton · `@Transactional`/`@Async`/`@Cacheable` = Proxy · `JdbcTemplate` = Template Method + Facade · `@EventListener` = Observer · `Comparator` = Strategy · Security `FilterChain` = Chain of Responsibility · `BeanFactory` = Factory Method.

---

*Last Updated: 2026-06-18*

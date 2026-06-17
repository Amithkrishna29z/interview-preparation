# GoF Design Patterns — Complete Java Interview Guide

All 23 Gang of Four patterns with Java code, real-world use cases, Spring framework usage, and interview Q&A.

> **New to design patterns?** Don't try to memorize all 23. Patterns are just *named solutions to problems that keep coming up*. For each one below, read the **"Think of it like"** analogy first — once the everyday idea clicks, the code is easy. Focus on the **bolded common ones** (Singleton, Factory, Builder, Strategy, Observer, Decorator, Proxy, Template Method) — those are what 90% of interviews actually ask.

---

## CREATIONAL PATTERNS

> **The big idea:** These 5 patterns are all about *how you create objects* — instead of calling `new` everywhere, you hide the creation logic so it's flexible and clean.

### 1. Singleton
One instance per JVM. All Spring beans are singletons by default.

> **Think of it like:** the President of a country — there is only ever *one* at a time, and everyone refers to the same person. You don't create a new President each time you need one; you ask for the existing one.

```java
// Best approach: Enum Singleton (Josh Bloch)
public enum DatabaseConnection {
    INSTANCE;
    private final Connection conn;
    DatabaseConnection() { conn = createConnection(); }
    public Connection get() { return conn; }
}

// Thread-safe lazy: Static inner class (Bill Pugh)
public class ConfigManager {
    private ConfigManager() {}
    private static class Holder {
        static final ConfigManager INSTANCE = new ConfigManager();
    }
    public static ConfigManager getInstance() { return Holder.INSTANCE; }
}

// Double-checked locking (acceptable but less clean)
public class Logger {
    private static volatile Logger instance;
    private Logger() {}
    public static Logger getInstance() {
        if (instance == null) {
            synchronized (Logger.class) {
                if (instance == null) instance = new Logger();
            }
        }
        return instance;
    }
}
```
**Spring:** `@Scope("singleton")` (default). `@Bean` methods in `@Configuration` return the same instance.

---

### 2. Factory Method
Define an interface for creating an object; subclasses decide which class to instantiate.

> **Think of it like:** ordering "a coffee" at a café — you don't make it yourself, and you don't say *how* to make it. You just ask, and the café decides which barista/machine creates it. You depend on the order, not the construction details.

```java
public interface Notification {
    void send(String message);
}
public class EmailNotification implements Notification {
    public void send(String message) { System.out.println("Email: " + message); }
}
public class SMSNotification implements Notification {
    public void send(String message) { System.out.println("SMS: " + message); }
}

// Creator (abstract)
public abstract class NotificationService {
    public void notifyUser(String msg) {
        Notification n = createNotification();  // factory method
        n.send(msg);
    }
    protected abstract Notification createNotification();  // subclass decides
}

public class EmailNotificationService extends NotificationService {
    protected Notification createNotification() { return new EmailNotification(); }
}
```
**Spring:** `FactoryBean<T>`, `BeanFactory`. `@Bean` methods are factory methods.

---

### 3. Abstract Factory
Create families of related objects without specifying concrete classes.

> **Think of it like:** picking a furniture *style* — once you choose "Modern" vs "Victorian," the factory hands you a matching chair, table, AND sofa, all in that one style. Factory Method makes *one* product; Abstract Factory makes a *whole matching set*.

```java
// Abstract factory
public interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

public class WindowsUIFactory implements UIFactory {
    public Button createButton() { return new WindowsButton(); }
    public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}

public class MacUIFactory implements UIFactory {
    public Button createButton() { return new MacButton(); }
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}

// Client — doesn't know which factory it has
public class Application {
    private UIFactory factory;
    public Application(UIFactory factory) { this.factory = factory; }
    public void render() {
        Button btn = factory.createButton();
        Checkbox cb = factory.createCheckbox();
        btn.render(); cb.render();
    }
}
```
**Difference from Factory Method:** Abstract Factory creates a *family* of objects; Factory Method creates *one* type.
**Spring:** DataSource abstractions, Spring Data repository factories.

---

### 4. Builder
Construct complex objects step by step. Handles many optional parameters.

> **Think of it like:** building a custom burger — you add bun, then patty, then cheese, then sauce one step at a time, and finally say "done" (`build()`). Much clearer than a constructor with 8 arguments where you can't remember which is which.

```java
public class User {
    private final String name;       // required
    private final String email;      // required
    private final String phone;      // optional
    private final String address;    // optional

    private User(Builder b) {
        this.name = b.name; this.email = b.email;
        this.phone = b.phone; this.address = b.address;
    }

    public static class Builder {
        private String name, email, phone, address;

        public Builder(String name, String email) {
            this.name = name; this.email = email;
        }
        public Builder phone(String phone) { this.phone = phone; return this; }
        public Builder address(String address) { this.address = address; return this; }
        public User build() {
            if (name == null || name.isBlank()) throw new IllegalStateException("name required");
            return new User(this);
        }
    }
}

// Usage
User user = new User.Builder("John", "john@example.com")
    .phone("555-1234")
    .build();
```
**Spring:** `UriComponentsBuilder`, `ResponseEntity.ok().header(...).body(...)`, `MockMvcRequestBuilders`. Lombok `@Builder` generates this automatically.

---

### 5. Prototype
*(Awareness — rarely asked)* Clone existing objects instead of creating from scratch. Useful when creating an object is expensive — like photocopying a filled-in form and tweaking a few fields. Prefer a copy constructor over the broken `Object.clone()` API.

```java
public DocumentTemplate copy() { return new DocumentTemplate(this); }  // copy constructor
```
**Spring:** `@Scope("prototype")`.

---

## STRUCTURAL PATTERNS

> **The big idea:** These 7 patterns are about *how you assemble objects into bigger structures* — wrapping, connecting, or simplifying them so the pieces fit together cleanly.

### 6. Adapter
Make incompatible interfaces work together.

> **Think of it like:** a travel plug adapter — your charger has one plug shape, the foreign wall socket has another. The adapter sits in between so they work together, without changing either one.

```java
// Legacy system interface
public class LegacyPaymentProcessor {
    public void processPayment(double amount, String cardNumber) {
        System.out.println("Legacy: processing " + amount);
    }
}

// New interface expected by the system
public interface PaymentGateway {
    void charge(PaymentRequest request);
}

// Adapter wraps legacy in new interface
public class LegacyPaymentAdapter implements PaymentGateway {
    private final LegacyPaymentProcessor legacy;

    public LegacyPaymentAdapter(LegacyPaymentProcessor legacy) {
        this.legacy = legacy;
    }

    public void charge(PaymentRequest request) {
        // translate new interface to old
        legacy.processPayment(request.getAmount(), request.getCardNumber());
    }
}
```
**Java:** `Arrays.asList()`, `InputStreamReader` (adapts InputStream to Reader), `OutputStreamWriter`.
**Spring:** `HandlerAdapter` in Spring MVC, `JpaVendorAdapter`.

---

### 7. Bridge
*(Awareness — rarely asked)* Separate an abstraction from its implementation so both can vary independently — like a universal remote (abstraction) that works with any TV brand (implementation), avoiding a class for every combination. The abstraction holds a reference to the implementation interface.

```java
public abstract class Shape {
    protected Renderer renderer;  // bridge to implementation
    public Shape(Renderer renderer) { this.renderer = renderer; }
}
```
**Java/Real-world:** JDBC (Java code = abstraction, driver = implementation), SLF4J (logging facade + multiple backends).

---

### 8. Composite
*(Awareness — rarely asked)* Treat individual objects and compositions uniformly via a shared interface (tree structure) — like folders and files, where asking either for its "size" just works no matter how deeply nested. A `Directory` holds a list of `FileSystemItem` and both implement the same interface.

```java
public interface FileSystemItem { long getSize(); }
// Directory.getSize() sums children; File.getSize() returns its own size — same call works on both.
```
**Spring:** Spring Security FilterChain, UI component trees, expression trees.

---

### 9. Decorator
Add behavior dynamically without subclassing.

> **Think of it like:** getting dressed. You start with a person, add a jacket, then a scarf, then a hat. Each layer wraps the previous one and adds something — but the person underneath never changes. (The coffee example below: coffee → +milk → +sugar.)

```java
public interface Coffee {
    String getDescription();
    double getCost();
}

public class SimpleCoffee implements Coffee {
    public String getDescription() { return "Coffee"; }
    public double getCost() { return 1.00; }
}

// Base decorator
public abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    public CoffeeDecorator(Coffee coffee) { this.coffee = coffee; }
}

public class Milk extends CoffeeDecorator {
    public Milk(Coffee coffee) { super(coffee); }
    public String getDescription() { return coffee.getDescription() + ", Milk"; }
    public double getCost() { return coffee.getCost() + 0.25; }
}

public class Sugar extends CoffeeDecorator {
    public Sugar(Coffee coffee) { super(coffee); }
    public String getDescription() { return coffee.getDescription() + ", Sugar"; }
    public double getCost() { return coffee.getCost() + 0.10; }
}

// Usage
Coffee c = new Sugar(new Milk(new SimpleCoffee()));
System.out.println(c.getDescription() + " = $" + c.getCost());
// Coffee, Milk, Sugar = $1.35
```
**Java:** `java.io` streams — `new BufferedReader(new FileReader("file.txt"))`.
**Spring:** Spring Security's filter chain, `BeanDefinition` decoration, Servlet filter wrappers.

---

### 10. Facade
Simplified interface to a complex subsystem.

> **Think of it like:** the "Watch Movie" button on a universal remote. One press secretly turns on the TV, the sound system, AND the cable box. The facade hides all that complexity behind one simple call.

```java
// Complex subsystem
class DVDPlayer { void on(){} void play(String movie){} void off(){} }
class Projector { void on(){} void wideScreenMode(){} void off(){} }
class SoundSystem { void on(){} void setVolume(int v){} void off(){} }

// Facade
public class HomeTheaterFacade {
    private DVDPlayer dvd; private Projector projector; private SoundSystem sound;

    public HomeTheaterFacade(DVDPlayer d, Projector p, SoundSystem s) {
        this.dvd = d; this.projector = p; this.sound = s;
    }

    public void watchMovie(String movie) {
        projector.on(); projector.wideScreenMode();
        sound.on(); sound.setVolume(10);
        dvd.on(); dvd.play(movie);
    }

    public void endMovie() {
        dvd.off(); sound.off(); projector.off();
    }
}
```
**Java/Spring:** `SLF4J` (facade for logging implementations), `JdbcTemplate` (facade over JDBC boilerplate), Spring Data `Repository` interfaces, `RestTemplate`.

---

### 11. Flyweight
*(Awareness — rarely asked)* Share common (intrinsic) state among many fine-grained objects to save memory; each object holds only its unique (extrinsic) state. Like the letter "a" in a document: the font/shape is stored once and shared, only each position differs. A factory caches and reuses the shared part.

```java
TreeType type = TreeTypeFactory.get("Oak", "green", "rough");  // shared, reused for all Oaks
// 1,000,000 Tree objects (x, y) but only a few TreeType objects.
```
**Java:** `String` pool (`"hello" == "hello"` is true for literals), `Integer.valueOf()` cache (-128 to 127), `Character` cache.

---

### 12. Proxy
Provide a surrogate that controls access to another object.

> **Think of it like:** a secretary who screens calls for a busy CEO. You talk to the secretary (proxy), who decides whether, when, and how to reach the CEO (the real object) — maybe delaying it, logging it, or blocking it. This is *exactly* how Spring's `@Transactional` and `@Async` work under the hood.

```java
// Interface
public interface Image { void display(); }

// Real object (expensive to create)
public class RealImage implements Image {
    private String filename;
    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk();  // expensive!
    }
    private void loadFromDisk() { System.out.println("Loading: " + filename); }
    public void display() { System.out.println("Displaying: " + filename); }
}

// Virtual Proxy: defer expensive creation until needed
public class ImageProxy implements Image {
    private String filename;
    private RealImage realImage;  // null until first use

    public ImageProxy(String filename) { this.filename = filename; }

    public void display() {
        if (realImage == null) realImage = new RealImage(filename);  // lazy init
        realImage.display();
    }
}
```

**JDK Dynamic Proxy (how Spring AOP works internally):**
```java
Image proxy = (Image) Proxy.newProxyInstance(
    Image.class.getClassLoader(),
    new Class[]{Image.class},
    (proxyObj, method, args) -> {
        System.out.println("Before: " + method.getName());
        Object result = method.invoke(realImage, args);
        System.out.println("After: " + method.getName());
        return result;
    }
);
```
**Proxy types:** Virtual (lazy loading), Protection (access control), Caching (memoization), Remote (RMI), Logging.
**Spring:** `@Transactional`, `@Async`, `@Cacheable` all create proxies via BeanPostProcessor.

---

## BEHAVIORAL PATTERNS

> **The big idea:** These 11 patterns are about *how objects communicate and divide responsibility* — who calls whom, who decides what, and how behavior changes over time.

### 13. Chain of Responsibility
*(Awareness — rarely asked)* Pass a request along a chain of handlers; each handler either handles it or forwards it to the next. Like customer support escalation: L1 → L2 → L3. Each handler holds a `next` reference.

```java
public abstract class LogHandler {
    protected LogHandler next;
    public LogHandler setNext(LogHandler next) { this.next = next; return next; }
    public abstract void handle(String level, String message);  // handle or delegate to next
}
```
**Spring:** Spring Security `FilterChain`, Servlet `FilterChain`, Spring MVC `HandlerInterceptor`.

---

### 14. Command
*(Awareness — rarely asked)* Encapsulate a request as an object so it can be queued, logged, or undone — like a restaurant order slip that holds the request as a "thing." An invoker keeps a history stack of executed commands to support undo.

```java
public interface Command { void execute(); void undo(); }
// Invoker: execute() runs cmd and pushes to history; undo() pops and reverses the last command.
```
**Java:** `Runnable` and `Callable` are command interfaces. `java.awt.event.ActionListener`.
**Spring:** Spring Batch `Tasklet`, `@Async` tasks, message queue consumers.

---

### 15. Interpreter
*(Awareness — rarely asked)* Define a grammar and interpret sentences in it — like a calculator reading "3 + 5". Each grammar rule is a class with an `interpret()` method. Rarely written by hand; frameworks (SpEL, regex engines) do it for you.

```java
public interface Expression { int interpret(Map<String, Integer> context); }
// AddExpression.interpret() = left.interpret() + right.interpret() — composed into a tree.
```
**Spring:** SpEL (Spring Expression Language), SQL parsing, regex.

---

### 16. Iterator
*(Awareness — rarely asked)* Sequential access to aggregate elements without exposing internal representation — like a TV remote's "next channel" button. Implement `Iterable<T>` with `hasNext()`/`next()`; every Java `for (x : collection)` loop uses this.

```java
public Iterator<Integer> iterator() {
    return new Iterator<>() {
        int current = start;
        public boolean hasNext() { return current <= end; }
        public Integer next() { return current++; }
    };
}
```
**Java:** `Iterator<T>`, `Iterable<T>`, all Collection iterators, `Scanner`.
**External vs internal:** `iterator.next()` = external (caller controls); `forEach`, `stream()` = internal (collection controls).

---

### 17. Mediator
*(Awareness — rarely asked)* Reduce coupling by having components communicate through a central mediator instead of directly — like an air-traffic controller coordinating planes. Each component knows only the mediator, not the other components.

```java
public interface ChatMediator { void sendMessage(String message, User sender); }
// ChatRoom forwards each message to all other users — users never reference each other directly.
```
**Spring:** `ApplicationEventPublisher` mediates between publishers and listeners. Message brokers (Kafka, RabbitMQ) are distributed mediators. Spring MVC Controller mediates between View and Model.

---

### 18. Memento
*(Awareness — rarely asked)* Capture an object's state as a snapshot that can be restored later — like a video-game save point. The Originator creates/restores mementos; a Caretaker stores them in a history stack. The snapshot keeps its state private.

```java
public Memento save() { return new Memento(content); }   // Originator snapshots state
public void restore(Memento m) { content = m.getState(); } // and restores it later
```
**Spring/Java:** `Serializable` (memento via serialization), git commits, database transaction savepoints, game save/load.

---

### 19. Observer
Define a one-to-many dependency: when one object changes state, all dependents are notified.

> **Think of it like:** subscribing to a YouTube channel. When the channel uploads a video (state change), every subscriber gets notified automatically — the channel doesn't call each person individually, and subscribers can join or leave anytime.

```java
public interface StockObserver { void update(String stock, double price); }

public class StockMarket {
    private Map<String, Double> prices = new HashMap<>();
    private List<StockObserver> observers = new ArrayList<>();

    public void addObserver(StockObserver o) { observers.add(o); }
    public void removeObserver(StockObserver o) { observers.remove(o); }

    public void setPrice(String stock, double price) {
        prices.put(stock, price);
        observers.forEach(o -> o.update(stock, price));  // notify all
    }
}

public class PriceAlertService implements StockObserver {
    private double threshold;
    public PriceAlertService(double threshold) { this.threshold = threshold; }
    public void update(String stock, double price) {
        if (price > threshold) System.out.println("ALERT: " + stock + " = " + price);
    }
}
```
**Spring:** `@EventListener` / `ApplicationEventPublisher` is Observer pattern. Spring Data Domain Events (`@DomainEvents`). Kafka/RabbitMQ are distributed observer implementations.

---

### 20. State
*(Awareness — rarely asked)* Let an object alter its behavior when its internal state changes — like a traffic light where the same object behaves differently per state and each state knows the next one. The object delegates to a state object and swaps it on transitions.

```java
public interface OrderState { void confirm(Order o); void ship(Order o); }
// PendingState.confirm() does the work, then order.setState(new ConfirmedState()).
```
**Spring State Machine:** `@EnableStateMachine`, `@State`, `@Transition` annotations.
**vs Strategy:** State manages transitions and knows about other states; Strategy just switches algorithms.

---

### 21. Strategy
Define a family of algorithms, encapsulate each, and make them interchangeable.

> **Think of it like:** choosing a route in Google Maps. The goal is the same (get home), but you can swap the strategy on the fly: fastest, shortest, or avoid-tolls. Each strategy is interchangeable and the app doesn't care which you picked.

```java
public interface SortStrategy {
    void sort(List<Integer> data);
}

public class BubbleSortStrategy implements SortStrategy {
    public void sort(List<Integer> data) { /* bubble sort */ }
}
public class QuickSortStrategy implements SortStrategy {
    public void sort(List<Integer> data) { Collections.sort(data); }
}

public class DataProcessor {
    private SortStrategy strategy;
    public DataProcessor(SortStrategy strategy) { this.strategy = strategy; }
    public void setStrategy(SortStrategy s) { this.strategy = s; }
    public void process(List<Integer> data) {
        strategy.sort(data);
        System.out.println("Processed: " + data);
    }
}

// Runtime strategy switching
DataProcessor processor = new DataProcessor(new QuickSortStrategy());
processor.process(List.of(3, 1, 4, 1, 5));
processor.setStrategy(new BubbleSortStrategy());
```
**Java:** `Comparator<T>` is a strategy. `Comparable<T>` is another.
**Spring:** `AuthenticationProvider`, `HandlerMapping`, `ContentNegotiationStrategy`, `TransactionManager`.

---

### 22. Template Method
Define algorithm skeleton in base class; subclasses fill in specific steps.

> **Think of it like:** a recipe template. The steps are fixed and in order — prep, cook, serve — but each specific dish fills in its own details for each step. The base class owns the order; subclasses fill the blanks.

```java
// Abstract class defines the template
public abstract class ReportGenerator {

    // Template method — defines the algorithm skeleton
    public final void generateReport() {
        gatherData();
        processData();
        formatReport();
        if (shouldSendEmail()) sendEmail();  // hook (optional override)
    }

    protected abstract void gatherData();    // subclasses must implement
    protected abstract void processData();
    protected abstract void formatReport();

    // Hook — optional override point (has default behavior)
    protected boolean shouldSendEmail() { return false; }
    protected void sendEmail() { System.out.println("Sending email..."); }
}

public class SalesReport extends ReportGenerator {
    protected void gatherData() { System.out.println("Fetching sales data from DB"); }
    protected void processData() { System.out.println("Calculating totals"); }
    protected void formatReport() { System.out.println("Formatting as PDF"); }
    protected boolean shouldSendEmail() { return true; }  // override hook
}
```
**Spring:** `JdbcTemplate` (you provide SQL and RowMapper; template handles connection/exception), `AbstractList`, `HttpServlet.service()` → `doGet()`/`doPost()`. Spring Batch `AbstractItemReader`.
**vs Strategy:** Template Method uses inheritance; Strategy uses composition. Template Method fixes algorithm structure; Strategy swaps the whole algorithm.

---

### 23. Visitor
*(Awareness — rarely asked)* Separate an algorithm from the object structure it operates on, so you can add a new operation without modifying the classes — like a tax auditor (operation) visiting different business types. Each element has `accept(visitor)`; the visitor has a `visitX()` per type (double dispatch).

```java
public interface Shape { void accept(ShapeVisitor visitor); }       // element: accept()
public interface ShapeVisitor { void visitCircle(Circle c); void visitRectangle(Rectangle r); }
// Circle.accept() calls visitor.visitCircle(this) — a new visitor adds an op without touching shapes.
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
| Chain of Resp. | Behavioral | Pass request through handlers | Spring Security filter chain |
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
Design patterns are reusable solutions to commonly occurring design problems. They represent best practices distilled from experienced developers. They improve communication (shared vocabulary), reduce design time, and make code more maintainable. Drawback: overuse adds unnecessary complexity.

**Q2: What is the difference between Factory Method and Abstract Factory?**
Factory Method defines one method for creating one type of object; subclasses decide the concrete class. Abstract Factory provides an interface for creating *families* of related objects (multiple factory methods). Abstract Factory is often implemented using Factory Methods.

**Q3: When would you use Builder instead of a constructor?**
When a class has many optional parameters (≥4), when construction requires multiple steps, or when the same construction process can create different representations. Builder avoids telescoping constructors and makes required vs optional fields explicit.

**Q4: What is the difference between Decorator and Proxy?**
Decorator adds new behavior to an object transparently (client knows it's decorated). Proxy controls access to an object (lazy init, security, caching) — the client may not know it's talking to a proxy. Java I/O uses Decorator; Spring AOP uses Proxy.

**Q5: What is the difference between Strategy and Template Method?**
Strategy uses composition (inject the algorithm as a dependency; runtime swapping). Template Method uses inheritance (algorithm skeleton in base class, steps in subclasses; compile-time fixation). Strategy is more flexible; Template Method is simpler.

**Q6: What design patterns does Spring use internally?**
Proxy (`@Transactional`, `@Async`, `@Cacheable`), Template Method (`JdbcTemplate`), Observer (`@EventListener`), Factory Method (`BeanFactory`), Singleton (default bean scope), Decorator (Security filters), Chain of Responsibility (FilterChain), Facade (`JdbcTemplate`, Spring Data), Strategy (`Comparator`, auth providers), Mediator (`ApplicationEventPublisher`).

**Q7: How does @Transactional use the Proxy pattern?**
At startup, `AnnotationAwareAspectJAutoProxyCreator` wraps each `@Transactional` bean in a CGLIB or JDK dynamic proxy. At runtime, the proxy intercepts the method call, starts a transaction, calls the real method, and commits or rolls back. The caller talks to the proxy, not the real object.

**Q8: How does JdbcTemplate use Template Method?**
`JdbcTemplate.query()` defines the algorithm: get connection, create statement, execute SQL, handle ResultSet, release resources, handle exceptions. You provide the variable parts: the SQL string and the `RowMapper` lambda. The template handles all boilerplate.

**Q9: What is the Chain of Responsibility? Where is it used in Spring Security?**
Each handler processes a request or passes it to the next handler. Spring Security's filter chain is exactly this: each `SecurityFilter` either handles the request (authentication, authorization, CSRF check) or passes to the next filter. Each filter is independent; the chain is configured declaratively.

**Q10: When should you NOT use a design pattern?**
When it adds unnecessary complexity for a simple problem. YAGNI (You Aren't Gonna Need It) — don't add abstraction for hypothetical future needs. A simple if-else often beats a full Strategy implementation for 2 cases. A direct constructor beats Builder for 1-2 required parameters.

---

## Quick Reference Cheat Sheet

**One-line analogy per pattern (read this the night before an interview):**

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

**The 8 patterns interviews ask most** — make sure you can write these from memory:
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

**Most common confusions (one-liners):**
- **Decorator vs Proxy** → Decorator *adds* behavior (you know it's wrapped); Proxy *controls access* (you may not know it's there).
- **Strategy vs State** → Strategy = *you* pick the algorithm; State = the object switches its own behavior as its status changes.
- **Strategy vs Template Method** → Strategy uses *composition* (swap an object); Template Method uses *inheritance* (override steps).
- **Adapter vs Facade** → Adapter makes *incompatible* things fit (1-to-1); Facade *simplifies* something complex (1-to-many).
- **Observer vs Mediator** → Observer = subscribers listen to one subject; Mediator = everyone routes through a central hub.

**Patterns Spring uses (great to name in interviews):**
`@Bean` = Singleton · `@Transactional`/`@Async`/`@Cacheable` = Proxy · `JdbcTemplate` = Template Method + Facade · `@EventListener` = Observer · `Comparator` = Strategy · Security `FilterChain` = Chain of Responsibility + Composite · `BeanFactory` = Factory Method.

---

*Last Updated: 2026-06-06*

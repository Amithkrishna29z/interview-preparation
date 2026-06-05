# GoF Design Patterns — Complete Java Interview Guide

All 23 Gang of Four patterns with Java code, real-world use cases, Spring framework usage, and interview Q&A.

---

## CREATIONAL PATTERNS

### 1. Singleton
One instance per JVM. All Spring beans are singletons by default.

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
Clone existing objects instead of creating from scratch.

```java
public class DocumentTemplate implements Cloneable {
    private String title;
    private List<String> sections;

    // Deep copy constructor (preferred over Cloneable)
    public DocumentTemplate(DocumentTemplate other) {
        this.title = other.title;
        this.sections = new ArrayList<>(other.sections);  // deep copy
    }

    // Or static factory copy method
    public DocumentTemplate copy() {
        return new DocumentTemplate(this);
    }
}

DocumentTemplate invoice = templateRepository.findByName("Invoice");
DocumentTemplate newInvoice = invoice.copy();
newInvoice.setTitle("Invoice #1234");
```
**Java:** `Object.clone()` (avoid — broken API), prefer copy constructors.
**Spring:** `@Scope("prototype")`.

---

## STRUCTURAL PATTERNS

### 6. Adapter
Make incompatible interfaces work together.

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
Separate abstraction from implementation so both can vary independently.

```java
// Implementation interface
public interface Renderer {
    void renderCircle(float x, float y, float radius);
}
public class VectorRenderer implements Renderer {
    public void renderCircle(float x, float y, float radius) {
        System.out.printf("Drawing vector circle at %.1f,%.1f r=%.1f%n", x, y, radius);
    }
}
public class RasterRenderer implements Renderer {
    public void renderCircle(float x, float y, float radius) {
        System.out.printf("Drawing raster pixels for circle at %.1f,%.1f%n", x, y);
    }
}

// Abstraction
public abstract class Shape {
    protected Renderer renderer;  // bridge to implementation
    public Shape(Renderer renderer) { this.renderer = renderer; }
    public abstract void draw();
}

public class Circle extends Shape {
    private float x, y, radius;
    public Circle(float x, float y, float radius, Renderer renderer) {
        super(renderer); this.x = x; this.y = y; this.radius = radius;
    }
    public void draw() { renderer.renderCircle(x, y, radius); }
}
```
**Java/Real-world:** JDBC (Java code = abstraction, driver = implementation), SLF4J (logging facade + multiple backends).

---

### 8. Composite
Treat individual objects and compositions uniformly (tree structure).

```java
public interface FileSystemItem {
    String getName();
    long getSize();
    void print(String indent);
}

public class File implements FileSystemItem {
    private String name; private long size;
    public File(String name, long size) { this.name = name; this.size = size; }
    public String getName() { return name; }
    public long getSize() { return size; }
    public void print(String indent) { System.out.println(indent + name + " (" + size + "B)"); }
}

public class Directory implements FileSystemItem {
    private String name;
    private List<FileSystemItem> children = new ArrayList<>();

    public Directory(String name) { this.name = name; }
    public void add(FileSystemItem item) { children.add(item); }

    public String getName() { return name; }
    public long getSize() { return children.stream().mapToLong(FileSystemItem::getSize).sum(); }
    public void print(String indent) {
        System.out.println(indent + name + "/");
        children.forEach(c -> c.print(indent + "  "));
    }
}

// Client treats File and Directory identically
FileSystemItem root = new Directory("root");
((Directory)root).add(new File("readme.txt", 1024));
Directory src = new Directory("src");
src.add(new File("Main.java", 2048));
((Directory)root).add(src);
root.print("");
```
**Spring:** Spring Security FilterChain, UI component trees, expression trees.

---

### 9. Decorator
Add behavior dynamically without subclassing.

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
Share common state among many fine-grained objects to save memory.

```java
// Flyweight: intrinsic (shared) state
public class TreeType {
    private String name, color, texture;  // shared across all trees of this type
    public TreeType(String name, String color, String texture) {
        this.name = name; this.color = color; this.texture = texture;
    }
    public void draw(int x, int y) {
        System.out.printf("Drawing %s tree at (%d,%d)%n", name, x, y);
    }
}

// Factory manages the flyweight pool
public class TreeTypeFactory {
    private static Map<String, TreeType> cache = new HashMap<>();
    public static TreeType get(String name, String color, String texture) {
        String key = name + color + texture;
        return cache.computeIfAbsent(key, k -> new TreeType(name, color, texture));
    }
}

// Tree object stores extrinsic (unique per instance) state
public class Tree {
    private int x, y;  // extrinsic
    private TreeType type;  // shared flyweight

    public Tree(int x, int y, String typeName) {
        this.x = x; this.y = y;
        this.type = TreeTypeFactory.get(typeName, "green", "rough");
    }
    public void draw() { type.draw(x, y); }
}
// 1,000,000 trees but only a few TreeType objects
```
**Java:** `String` pool (`"hello" == "hello"` is true for literals), `Integer.valueOf()` cache (-128 to 127), `Character` cache.

---

### 12. Proxy
Provide a surrogate that controls access to another object.

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

### 13. Chain of Responsibility
Pass request along a chain of handlers; each decides to handle or pass forward.

```java
public abstract class LogHandler {
    protected LogHandler next;
    public LogHandler setNext(LogHandler next) { this.next = next; return next; }
    public abstract void handle(String level, String message);
}

public class ConsoleLogHandler extends LogHandler {
    public void handle(String level, String message) {
        if (level.equals("INFO")) System.out.println("Console: " + message);
        else if (next != null) next.handle(level, message);
    }
}

public class FileLogHandler extends LogHandler {
    public void handle(String level, String message) {
        if (level.equals("WARN")) System.out.println("File: " + message);
        else if (next != null) next.handle(level, message);
    }
}

public class EmailLogHandler extends LogHandler {
    public void handle(String level, String message) {
        if (level.equals("ERROR")) System.out.println("Email: " + message);
        else if (next != null) next.handle(level, message);
    }
}

// Build chain
LogHandler chain = new ConsoleLogHandler();
chain.setNext(new FileLogHandler()).setNext(new EmailLogHandler());
chain.handle("ERROR", "System down!");
```
**Spring:** Spring Security `FilterChain`, Servlet `FilterChain`, try-catch-rethrow chains, Spring MVC `HandlerInterceptor`.

---

### 14. Command
Encapsulate a request as an object (enables undo, queue, log).

```java
public interface Command { void execute(); void undo(); }

public class InsertTextCommand implements Command {
    private StringBuilder text;
    private String insertedText;
    private int position;

    public InsertTextCommand(StringBuilder text, String insert, int pos) {
        this.text = text; this.insertedText = insert; this.position = pos;
    }
    public void execute() { text.insert(position, insertedText); }
    public void undo() { text.delete(position, position + insertedText.length()); }
}

// Invoker — manages command history
public class TextEditor {
    private StringBuilder content = new StringBuilder();
    private Deque<Command> history = new ArrayDeque<>();

    public void execute(Command cmd) { cmd.execute(); history.push(cmd); }
    public void undo() { if (!history.isEmpty()) history.pop().undo(); }
}
```
**Java:** `Runnable` and `Callable` are command interfaces. `java.awt.event.ActionListener`.
**Spring:** Spring Batch `Tasklet`, `@Async` tasks, message queue consumers.

---

### 15. Interpreter
Define a grammar and interpret sentences (less commonly asked).

```java
// Expression interface
public interface Expression { int interpret(Map<String, Integer> context); }

public class NumberExpression implements Expression {
    private int number;
    public NumberExpression(int n) { this.number = n; }
    public int interpret(Map<String, Integer> context) { return number; }
}

public class AddExpression implements Expression {
    private Expression left, right;
    public AddExpression(Expression l, Expression r) { left = l; right = r; }
    public int interpret(Map<String, Integer> ctx) {
        return left.interpret(ctx) + right.interpret(ctx);
    }
}
// 3 + 5
Expression expr = new AddExpression(new NumberExpression(3), new NumberExpression(5));
System.out.println(expr.interpret(null));  // 8
```
**Spring:** SpEL (Spring Expression Language), SQL parsing, regex.

---

### 16. Iterator
Sequential access to aggregate elements without exposing internal representation.

```java
// Custom iterator
public class NumberRange implements Iterable<Integer> {
    private int start, end;
    public NumberRange(int start, int end) { this.start = start; this.end = end; }

    public Iterator<Integer> iterator() {
        return new Iterator<>() {
            int current = start;
            public boolean hasNext() { return current <= end; }
            public Integer next() {
                if (!hasNext()) throw new NoSuchElementException();
                return current++;
            }
        };
    }
}

// Usage — works with enhanced for-each
for (int n : new NumberRange(1, 5)) System.out.println(n);  // 1 2 3 4 5
```
**Java:** `Iterator<T>`, `Iterable<T>`, all Collection iterators, `Scanner`.
**External vs internal:** `iterator.next()` = external (caller controls); `forEach`, `stream()` = internal (collection controls).

---

### 17. Mediator
Reduce coupling by having components communicate through a central mediator.

```java
public interface ChatMediator {
    void sendMessage(String message, User sender);
    void addUser(User user);
}

public class ChatRoom implements ChatMediator {
    private List<User> users = new ArrayList<>();

    public void addUser(User user) { users.add(user); }

    public void sendMessage(String message, User sender) {
        users.stream()
            .filter(u -> u != sender)
            .forEach(u -> u.receive(message));
    }
}

public class User {
    private String name; private ChatMediator mediator;
    public User(String name, ChatMediator m) { this.name = name; this.mediator = m; }
    public void send(String msg) { mediator.sendMessage(name + ": " + msg, this); }
    public void receive(String msg) { System.out.println(name + " received: " + msg); }
}
```
**Spring:** `ApplicationEventPublisher` acts as mediator between publishers and listeners. Message brokers (Kafka, RabbitMQ) are distributed mediators. Spring MVC Controller mediates between View and Model.

---

### 18. Memento
Capture and restore object state.

```java
// Originator
public class TextEditor {
    private String content = "";

    public void type(String text) { content += text; }
    public String getContent() { return content; }

    public Memento save() { return new Memento(content); }
    public void restore(Memento m) { content = m.getState(); }

    // Memento — inner class keeps state private
    public static class Memento {
        private final String state;
        private Memento(String state) { this.state = state; }
        private String getState() { return state; }
    }
}

// Caretaker — manages history
public class History {
    private Deque<TextEditor.Memento> history = new ArrayDeque<>();
    public void push(TextEditor.Memento m) { history.push(m); }
    public TextEditor.Memento pop() { return history.pop(); }
}

TextEditor editor = new TextEditor();
History history = new History();
editor.type("Hello");
history.push(editor.save());
editor.type(" World");
System.out.println(editor.getContent());  // "Hello World"
editor.restore(history.pop());
System.out.println(editor.getContent());  // "Hello"
```
**Spring/Java:** `Serializable` (memento via serialization), git commits, database transaction savepoints, game save/load.

---

### 19. Observer
Define a one-to-many dependency: when one object changes state, all dependents are notified.

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
Allow an object to alter its behavior when its internal state changes.

```java
public interface OrderState {
    void confirm(Order order);
    void ship(Order order);
    void cancel(Order order);
}

public class PendingState implements OrderState {
    public void confirm(Order order) {
        System.out.println("Order confirmed");
        order.setState(new ConfirmedState());
    }
    public void ship(Order order) { throw new IllegalStateException("Must confirm first"); }
    public void cancel(Order order) {
        System.out.println("Order cancelled");
        order.setState(new CancelledState());
    }
}

public class ConfirmedState implements OrderState {
    public void confirm(Order order) { System.out.println("Already confirmed"); }
    public void ship(Order order) {
        System.out.println("Order shipped");
        order.setState(new ShippedState());
    }
    public void cancel(Order order) {
        System.out.println("Confirmed order cancelled");
        order.setState(new CancelledState());
    }
}

public class Order {
    private OrderState state = new PendingState();
    public void setState(OrderState s) { state = s; }
    public void confirm() { state.confirm(this); }
    public void ship() { state.ship(this); }
    public void cancel() { state.cancel(this); }
}
```
**Spring State Machine:** `@EnableStateMachine`, `@State`, `@Transition` annotations.
**vs Strategy:** State manages transitions and knows about other states; Strategy just switches algorithms.

---

### 21. Strategy
Define a family of algorithms, encapsulate each, and make them interchangeable.

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
Separate an algorithm from the object structure it operates on. Enables double dispatch.

```java
// Element interface
public interface Shape {
    void accept(ShapeVisitor visitor);
}

public class Circle implements Shape {
    public double radius;
    public Circle(double r) { radius = r; }
    public void accept(ShapeVisitor visitor) { visitor.visitCircle(this); }
}

public class Rectangle implements Shape {
    public double width, height;
    public Rectangle(double w, double h) { width = w; height = h; }
    public void accept(ShapeVisitor visitor) { visitor.visitRectangle(this); }
}

// Visitor interface
public interface ShapeVisitor {
    void visitCircle(Circle c);
    void visitRectangle(Rectangle r);
}

// Concrete visitor — add new operation without modifying shapes
public class AreaCalculator implements ShapeVisitor {
    private double totalArea = 0;
    public void visitCircle(Circle c) { totalArea += Math.PI * c.radius * c.radius; }
    public void visitRectangle(Rectangle r) { totalArea += r.width * r.height; }
    public double getTotalArea() { return totalArea; }
}

// Usage
List<Shape> shapes = List.of(new Circle(5), new Rectangle(3, 4));
AreaCalculator calc = new AreaCalculator();
shapes.forEach(s -> s.accept(calc));
System.out.println("Total area: " + calc.getTotalArea());
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

## Commonly Confused Patterns

### Adapter vs Facade vs Decorator
- **Adapter**: makes *incompatible* interfaces work together (1:1 mapping to existing interface)
- **Facade**: *simplifies* a complex subsystem with a new interface
- **Decorator**: *adds behavior* to an existing interface without changing it

### Strategy vs State vs Template Method
- **Strategy**: swap entire algorithm; client chooses strategy; strategies are unaware of each other
- **State**: behavior changes as object state changes; states know about other states and manage transitions
- **Template Method**: fixes algorithm skeleton in base class; subclasses fill variable steps; uses inheritance

### Observer vs Mediator
- **Observer**: 1-to-many, objects subscribe directly to a subject
- **Mediator**: M-to-M, objects communicate only through a central mediator; objects don't know about each other

### Command vs Strategy
- **Command**: encapsulates a *request* (what to do); supports undo, queuing, logging
- **Strategy**: encapsulates an *algorithm* (how to do it); supports runtime switching

### Proxy vs Decorator
- **Proxy**: controls *access* (lazy init, security, caching, remote); often hides the target from client
- **Decorator**: *adds behavior* transparently; client always knows they're using the enhanced version

---

## Interview Questions & Answers (30+)

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

**Q6: What is the Composite pattern? Give a real-world example.**
Composite treats individual objects and compositions uniformly via a common interface. Example: file system where both File and Directory implement FileSystemItem. Client code calls `getSize()` or `print()` without knowing if it's a file or directory. Spring Security FilterChain is a composite of filters.

**Q7: What design patterns does Spring use internally?**
Proxy (`@Transactional`, `@Async`, `@Cacheable`), Template Method (`JdbcTemplate`), Observer (`@EventListener`), Factory Method (`BeanFactory`), Singleton (default bean scope), Decorator (Security filters), Chain of Responsibility (FilterChain), Facade (`JdbcTemplate`, Spring Data), Strategy (`Comparator`, auth providers), Mediator (`ApplicationEventPublisher`).

**Q8: How does @Transactional use the Proxy pattern?**
At startup, `AnnotationAwareAspectJAutoProxyCreator` wraps each `@Transactional` bean in a CGLIB or JDK dynamic proxy. At runtime, the proxy intercepts the method call, starts a transaction, calls the real method, and commits or rolls back. The caller talks to the proxy, not the real object.

**Q9: How does JdbcTemplate use Template Method?**
`JdbcTemplate.query()` defines the algorithm: get connection, create statement, execute SQL, handle ResultSet, release resources, handle exceptions. You provide the variable parts: the SQL string and the `RowMapper` lambda. The template handles all boilerplate.

**Q10: What is the Flyweight pattern? Where is it used in Java?**
Flyweight shares common (intrinsic) state among many objects, with each object holding only unique (extrinsic) state. In Java: the String pool (literal strings are interned/shared), `Integer.valueOf()` caches -128 to 127, `Character` cache. Use when creating millions of similar objects.

**Q11: What is the Chain of Responsibility? Where is it used in Spring Security?**
Each handler processes a request or passes it to the next handler. Spring Security's filter chain is exactly this: each `SecurityFilter` either handles the request (authentication, authorization, CSRF check) or passes to the next filter. Each filter is independent; the chain is configured declaratively.

**Q12: What is the Command pattern? How does it support undo?**
Command encapsulates a request as an object with `execute()` and `undo()` methods. Undo is implemented by keeping a history stack of executed commands. Calling `undo()` pops the last command and executes its reverse operation.

**Q13: What is the difference between Observer and Mediator?**
Observer: subject maintains a list of observers and notifies them directly (1-to-many coupling). Mediator: components communicate only through the mediator; they don't know about each other (M-to-M decoupling). `ApplicationEventPublisher` is a mediator; a simple event bus/callback is an observer.

**Q14: What is the State pattern? How is it different from Strategy?**
State changes an object's behavior based on its current state; states know about and manage transitions to other states. Strategy switches the algorithm at the client's request; strategies are unaware of each other. State represents what the object IS; Strategy represents how the object does something.

**Q15: When should you NOT use a design pattern?**
When it adds unnecessary complexity for a simple problem. YAGNI (You Aren't Gonna Need It) — don't add abstraction for hypothetical future needs. A simple if-else often beats a full Strategy implementation for 2 cases. A direct constructor beats Builder for 1-2 required parameters.

**Q16: What is the Visitor pattern? Why is double dispatch needed?**
Visitor lets you add operations to an object hierarchy without modifying the classes. Double dispatch: when `shape.accept(visitor)` is called, Java dispatches based on the runtime type of `shape`. Inside `accept`, `visitor.visit(this)` dispatches based on the concrete type of `this` (Circle or Rectangle). Two runtime dispatch decisions = double dispatch.

**Q17: What is the difference between Iterator and Visitor?**
Iterator traverses a collection and accesses each element sequentially. Visitor traverses an object structure (often heterogeneous) and performs a specific operation on each element based on its type. Iterator is generic access; Visitor adds specific behavior per type.

**Q18: What is the Memento pattern? Where is it used?**
Memento captures an object's state as a snapshot that can be restored later — implementing undo functionality. The Originator creates/restores mementos; the Caretaker stores them. Used in: text editor undo, game save states, database transaction savepoints, git commits.

# 🚀 Full Stack Junior Java Developer — Interview Master Cheat Sheet

> ⚡ **Quick revision before your interview. Read this the night before!**

---

## 📚 Study Guide Index

| File | Topics Covered | Priority |
|------|---------------|---------|
| `Java_Developer_Interview_Questions.md` | Core Java, OOP, Collections, Exceptions, Multithreading, Streams, Java 8+ | ⭐⭐⭐ HIGH |
| `Java_Data_Types_Notes.md` | Primitives, Wrappers, Autoboxing, Type Casting, String, Arrays, var | ⭐⭐⭐ HIGH |
| `Spring_Boot_Junior_Developer_Interview_Questions.md` | Spring Boot, REST API, JPA, Testing, Annotations | ⭐⭐⭐ HIGH |
| `Spring_Security_JWT_Interview_Questions.md` | Spring Security, JWT, CORS, BCrypt | ⭐⭐⭐ HIGH |
| `REST_API_HTTP_Interview_Questions.md` | HTTP methods, Status codes, REST design | ⭐⭐⭐ HIGH |
| `Networking_Interview_Questions.md` | OSI Model, TCP/UDP, DNS, HTTPS, Ports, WebSockets, Load Balancing | ⭐⭐⭐ HIGH |
| `DSA_Interview_Questions.md` | Arrays, Linked List, Stack, Binary Search, Sorting | ⭐⭐⭐ HIGH |
| `Software_Engineering_Principles_Interview_Questions.md` | Agile, Scrum, Kanban, SOLID, Design Patterns, Clean Code, TDD | ⭐⭐⭐ HIGH |
| `Database_Concepts_Interview_Questions.md` | ACID, Normalization, SQL, Transactions | ⭐⭐ MEDIUM |
| `MySQL_Interview_Questions.md` | MySQL, Indexes, Joins, Optimization | ⭐⭐ MEDIUM |
| `Java_Collections_Framework_Internal_Working.md` | HashMap internal, ArrayList vs LinkedList | ⭐⭐ MEDIUM |
| `System_Design_Microservices_Interview_Questions.md` | Microservices, Caching, API Gateway | ⭐⭐ MEDIUM |
| `Git_Docker_DevOps_Interview_Questions.md` | Git commands, Docker, Maven | ⭐ LOWER |
| `MongoDB_Interview_Questions.md` | MongoDB CRUD, Aggregation, Indexing | ⭐ LOWER |
| `PostgreSQL_Interview_Questions.md` | PostgreSQL, JSONB, Advanced features | ⭐ LOWER |
| `JavaScript_Interview_Questions.md` | var/let/const, Closures, Prototypes, Promises, Event Loop, ES6+, DOM | ⭐⭐⭐ HIGH |
| `Kafka_RabbitMQ_Interview_Questions.md` | Kafka topics/partitions/offsets, consumer groups, delivery semantics, RabbitMQ exchanges, AMQP, messaging patterns | ⭐⭐ MEDIUM |
| `HR_Behavioral_Interview_Questions.md` | STAR method, HR questions, Tell me about yourself | ⭐⭐⭐ HIGH |
| `World_Class_Software_Engineer_Roadmap.md` | Full mastery roadmap: DSA, System Design, Architecture, DDD, Security, Performance, Leadership | MASTERY |

---

## ⚡ Java Core — Must Know Answers

### JDK vs JRE vs JVM
- **JVM** = Runs bytecode (platform-independent)
- **JRE** = JVM + Libraries (for running apps)
- **JDK** = JRE + Compiler + Tools (for developers)

### OOP 4 Pillars
- **Encapsulation** = Private fields + public getters/setters
- **Inheritance** = `extends` keyword, reuse code
- **Polymorphism** = Same method, different behavior (overloading = compile-time, overriding = runtime)
- **Abstraction** = Abstract class / Interface hides implementation

### Abstract Class vs Interface

| | Abstract Class | Interface |
|--|---------------|-----------|
| Methods | Abstract + concrete | Abstract by default (Java 8: default/static) |
| Variables | Instance variables | Only constants (static final) |
| Constructors | ✅ Yes | ❌ No |
| Inheritance | Single (`extends`) | Multiple (`implements`) |

### == vs equals()
- `==` → compares **references** (same object in memory?)
- `equals()` → compares **values** (same content?)

### final keyword
- `final variable` → can't reassign
- `final method` → can't override
- `final class` → can't extend (e.g., String, Integer)

### String, StringBuilder, StringBuffer
- **String** → Immutable, thread-safe, new object each concatenation
- **StringBuilder** → Mutable, NOT thread-safe, fast (use in single thread)
- **StringBuffer** → Mutable, thread-safe, slower than StringBuilder

---

## ⚡ Collections — Must Know Answers

### HashMap Internal Working
```
1. Call key.hashCode() → calculate hash
2. hash & (capacity-1) → find bucket index
3. If bucket empty → store new Node
4. If bucket not empty → chain (linked list)
5. If chain length ≥ 8 → convert to Red-Black tree
6. If size > capacity * 0.75 → resize (double capacity)
```

### ArrayList vs LinkedList

| | ArrayList | LinkedList |
|--|-----------|------------|
| Storage | Dynamic array | Doubly-linked nodes |
| get(i) | O(1) ✅ | O(n) ❌ |
| add/remove at end | O(1) ✅ | O(1) ✅ |
| add/remove at middle | O(n) ❌ | O(n) ❌ |
| Memory | Less | More (2 pointers per node) |

### HashMap vs TreeMap vs LinkedHashMap
- **HashMap** → No order, O(1) operations
- **LinkedHashMap** → Insertion order maintained, O(1)
- **TreeMap** → Sorted by key, O(log n)

---

## ⚡ Exception Handling — Must Know

```java
// Checked = must handle (IOException, SQLException)
// Unchecked = runtime exceptions (NullPointerException, ArrayIndexOutOfBoundsException)

try {
    // risky code
} catch (SpecificException e) {
    // handle specific
} catch (Exception e) {
    // handle general (always last)
} finally {
    // ALWAYS runs (cleanup resources)
}

// try-with-resources (auto-close resources)
try (FileReader fr = new FileReader("file.txt")) {
    // fr.close() called automatically
}
```

---

## ⚡ Java 8 Features — Must Know

### Lambda Expressions
```java
// Before Java 8
Runnable r = new Runnable() {
    public void run() { System.out.println("Hello"); }
};

// Java 8 Lambda
Runnable r = () -> System.out.println("Hello");

// With parameters
Comparator<String> comp = (a, b) -> a.compareTo(b);
```

### Streams API
```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "Anna");

// Filter → Map → Collect
List<String> result = names.stream()
    .filter(n -> n.startsWith("A"))    // ["Alice", "Anna"]
    .map(String::toUpperCase)           // ["ALICE", "ANNA"]
    .sorted()                           // ["ALICE", "ANNA"]
    .collect(Collectors.toList());

// Reduce
int sum = IntStream.rangeClosed(1, 10).sum(); // 55

// Group by
Map<Integer, List<String>> grouped = names.stream()
    .collect(Collectors.groupingBy(String::length));
```

### Optional
```java
Optional<User> user = userRepository.findById(id);

// Bad way
User u = user.get(); // throws exception if empty!

// Good way
User u = user.orElse(new User());          // return default
User u = user.orElseThrow(() -> new UserNotFoundException());
user.ifPresent(u -> System.out.println(u.getName()));
```

---

## ⚡ Spring Boot — Must Know Answers

### Key Annotations
```
@SpringBootApplication = @Configuration + @EnableAutoConfiguration + @ComponentScan
@RestController = @Controller + @ResponseBody
@Repository = @Component + exception translation
@Service = @Component (semantic marker for business logic)
@Transactional = all DB operations succeed or all rollback
@Valid = trigger bean validation
@PreAuthorize = method-level security
```

### REST Controller Template
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping                          // GET /api/users
    public List<User> getAll() { ... }

    @GetMapping("/{id}")                 // GET /api/users/1
    public User getById(@PathVariable Long id) { ... }

    @PostMapping                         // POST /api/users
    public ResponseEntity<User> create(@Valid @RequestBody UserDTO dto) {
        User created = service.create(dto);
        return ResponseEntity.status(201).body(created);
    }

    @PutMapping("/{id}")                 // PUT /api/users/1
    public User update(@PathVariable Long id, @RequestBody UserDTO dto) { ... }

    @DeleteMapping("/{id}")              // DELETE /api/users/1
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build(); // 204
    }
}
```

### JPA Entity Template
```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders;

    @CreationTimestamp
    private LocalDateTime createdAt;
}
```

---

## ⚡ HTTP Status Codes — Must Know

| Code | Meaning | Situation |
|------|---------|-----------|
| **200** | OK | GET/PUT success |
| **201** | Created | POST success |
| **204** | No Content | DELETE success |
| **400** | Bad Request | Invalid data sent |
| **401** | Unauthorized | Not logged in |
| **403** | Forbidden | No permission |
| **404** | Not Found | Resource doesn't exist |
| **409** | Conflict | Duplicate resource |
| **500** | Server Error | Bug in code |

---

## ⚡ JWT Flow — Must Know

```
1. POST /login {username, password}
2. Server validates → creates JWT (signed with secret key)
3. Client stores JWT
4. Client sends: GET /api/data + "Authorization: Bearer <token>"
5. Server: verify signature → extract username → check roles → respond
```

**JWT Structure:** `header.payload.signature` (base64 encoded, NOT encrypted)

---

## ⚡ SQL — Must Know

```sql
-- JOINS
INNER JOIN  = only matching rows in BOTH tables
LEFT JOIN   = ALL from left + matching from right (NULL if no match)
RIGHT JOIN  = ALL from right + matching from left
FULL JOIN   = ALL rows from BOTH tables

-- GROUP BY + HAVING
SELECT user_id, COUNT(*) as orders
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 5;  -- filter AFTER grouping (WHERE filters BEFORE)

-- Window Functions (important!)
SELECT name, salary,
    RANK() OVER (ORDER BY salary DESC) as rank
FROM employees;

-- Find duplicates
SELECT email, COUNT(*)
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

---

## ⚡ DSA Quick Solutions

```java
// Two Sum → HashMap
// Max Subarray → Kadane's (track currentSum, reset to 0 if negative)
// Valid Parentheses → Stack
// Reverse String → Two pointers (left++, right--)
// Find Cycle → Fast/Slow pointers (Floyd's)
// Binary Search → left + (right-left)/2 to avoid overflow
// Fibonacci → Iterative (not recursive — avoid O(2^n))
```

---

## ⚡ Networking — Must Know Answers

### OSI Model (7 Layers)
```
Memory trick: "Please Do Not Throw Sausage Pizza Away"
7 - Application   → HTTP, HTTPS, FTP, DNS, SMTP
6 - Presentation  → SSL/TLS, encryption, encoding
5 - Session       → Session management
4 - Transport     → TCP (reliable), UDP (fast)
3 - Network       → IP routing (routers)
2 - Data Link     → MAC addresses (switches)
1 - Physical      → Cables, Wi-Fi
```

### TCP vs UDP
| | TCP | UDP |
|--|-----|-----|
| Reliable? | ✅ Yes (ACK, retransmit) | ❌ No |
| Speed | Slower | ✅ Faster |
| Order | ✅ Guaranteed | ❌ No |
| Use | HTTP, SSH, Email, FTP | DNS, Streaming, Gaming, VoIP |

### TCP 3-Way Handshake
```
Client ──SYN──────────────→ Server   ("I want to connect")
Client ←──SYN-ACK───────── Server   ("OK, I'm ready")
Client ──ACK──────────────→ Server   ("Great, let's talk!")
[Connection Open — data can now flow]
```

### Must-Know Ports
```
22   = SSH          80   = HTTP
53   = DNS          443  = HTTPS
3306 = MySQL        5432 = PostgreSQL
6379 = Redis        8080 = Spring Boot (dev)
27017= MongoDB
```

### DNS Lookup Chain
```
Browser cache → OS cache → ISP Resolver → Root DNS → TLD (.com) → Authoritative DNS → IP returned
```

### HTTP vs HTTPS
```
HTTP  = Port 80, plain text, INSECURE
HTTPS = Port 443, TLS encrypted, SECURE
TLS Handshake: Certificate verification → session key exchange → encrypted channel
```

### WebSocket vs HTTP Polling
```
Short Polling:  Client asks every N seconds (wasteful)
Long Polling:   Client waits, server holds request open
WebSocket:      Persistent full-duplex connection (real-time chat, live data)
```

---

## ⚡ Git — Must Know Commands

```bash
git clone <url>           # Download repo
git status                # See changed files
git add .                 # Stage all
git commit -m "message"   # Save
git push origin main      # Upload
git pull origin main      # Download latest
git checkout -b feature   # Create + switch branch
git merge feature         # Merge branch
git stash                 # Hide WIP changes
git log --oneline         # Short history
```

---

## 🎯 Interview Day Checklist

### Before Interview
- [ ] Research the company (products, tech stack, culture)
- [ ] Review your projects — be ready to explain every line
- [ ] Prepare your "Tell me about yourself" (memorize it)
- [ ] Prepare 2-3 questions to ask the interviewer
- [ ] Test your setup (for online interviews — mic, camera, internet)

### During Technical Interview
- [ ] **Clarify** the question before coding
- [ ] **Talk through** your approach before writing code
- [ ] **Write clean code** with good variable names
- [ ] **Handle edge cases** (null, empty, negative values)
- [ ] **Mention time/space complexity** when done
- [ ] **Test your code** manually with examples

### Common Interview Rounds for Junior Java Developer
```
Round 1: Online Coding Test (30-60 min)
  → DSA problems (Easy to Medium level)
  → Java basics (output prediction questions)

Round 2: Technical Interview 1 (45-60 min)
  → Java Core + OOP + Collections
  → Spring Boot + JPA + REST API
  → Database (SQL queries, indexes, ACID)

Round 3: Technical Interview 2 (30-45 min)
  → Project discussion (deep dive into your projects)
  → Design questions (how would you build X?)
  → Problem-solving questions

Round 4: HR Interview (30 min)
  → Tell me about yourself
  → Behavioral questions (STAR method)
  → Salary negotiation
```

---

## 🔑 Top 10 Questions You WILL Be Asked

1. **"Tell me about yourself"** → Practice your 60-second pitch
2. **"How does HashMap work internally?"** → Array + linked list/tree, hash, bucket
3. **"What is the difference between abstract class and interface?"** → Key differences table
4. **"What is Spring Boot auto-configuration?"** → Reads classpath, configures beans automatically
5. **"What is @Transactional?"** → ACID properties, all DB ops succeed or all rollback
6. **"What is JWT and how does it work?"** → Header.Payload.Signature, stateless auth
7. **"What is SOLID?"** → Single Responsibility, Open/Closed, Liskov, Interface Segregation, Dependency Inversion
8. **"Explain OOP with real examples"** → Animal/Dog (inheritance), BankAccount (encapsulation)
9. **"What is the difference between == and equals()?"** → Reference vs value comparison
10. **"Where do you see yourself in 5 years?"** → Structured, ambitious, realistic answer

---

## 💪 Motivation

> *"Every expert was once a beginner. The only difference is they kept going."*

**Your preparation plan:**
- Study 2-3 topics per day
- Code at least 1 DSA problem daily on LeetCode
- Build/revisit your projects
- Mock interviews with friends or record yourself

**You've got this! All the best! 🚀**

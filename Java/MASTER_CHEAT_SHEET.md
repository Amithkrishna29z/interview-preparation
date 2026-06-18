# Full Stack Junior Java Developer — Interview Master Cheat Sheet

> Quick revision before your interview. Read this the night before!

---

## Study Guide Index

> Complete index — all guides in this repo, grouped by area. ⭐ = priority for a junior Java backend role.

### Core Java
| File | Topics Covered | Priority |
|------|---------------|---------|
| `Java_Developer_Interview_Questions.md` | Core Java, OOP, Collections, Exceptions, Multithreading, Streams, Java 8+ | ⭐⭐⭐ HIGH |
| `Java_Data_Types_Notes.md` | Primitives, Wrappers, Autoboxing, Type Casting, String, Arrays, var | ⭐⭐⭐ HIGH |
| `Java_Collections_Framework_Internal_Working.md` | HashMap internal, ArrayList vs LinkedList | ⭐⭐⭐ HIGH |
| `Java_Multithreading_Concurrency_Guide.md` | Threads, synchronization, Executors, locks, concurrent collections, deadlock | ⭐⭐⭐ HIGH |
| `Java_Streams_And_Lambdas_Guide.md` | Lambdas, functional interfaces, method refs, streams, collectors, Optional | ⭐⭐⭐ HIGH |
| `Java_Generics_And_Wildcards_Guide.md` | Generic classes/methods, bounds, wildcards, PECS, type erasure, invariance | ⭐⭐⭐ HIGH |
| `Java_Exception_Handling_Guide.md` | Throwable hierarchy, checked/unchecked, try/catch/finally, try-with-resources, custom exceptions | ⭐⭐⭐ HIGH |
| `Java_Equals_And_HashCode_Guide.md` | equals()/hashCode() contract, HashMap lookup, correct implementations, JPA caveat | ⭐⭐⭐ HIGH |
| `Java_Enums_Guide.md` | Enum fields/methods, EnumSet/EnumMap, enum singleton, @Enumerated STRING vs ORDINAL | ⭐⭐ MEDIUM |
| `JVM_Internals_Memory_Management_GC.md` | JVM architecture, class loading, stack vs heap, GC algorithms, tuning | ⭐⭐ MEDIUM |
| `Java_Modern_Features_9_to_21.md` | var, records, sealed classes, text blocks, virtual threads, pattern matching | ⭐⭐ MEDIUM |
| `Java_IO_And_NIO_Guide.md` | Byte vs char streams, buffering, NIO.2 Path/Files, channels/buffers, charsets | ⭐ LOWER |
| `Java_Reflection_And_Annotations_Guide.md` | Custom annotations, meta-annotations, reflection API, framework tie-ins | ⭐ LOWER |
| `Java_Serialization_Guide.md` | Serializable, serialVersionUID, transient, Jackson/JSON, DTO vs entity | ⭐ LOWER |
| `Java_Testing_JUnit_Mockito_Guide.md` | JUnit 5, Mockito, AssertJ, test slices, Testcontainers, TDD | ⭐⭐ MEDIUM |

### Spring & Persistence
| File | Topics Covered | Priority |
|------|---------------|---------|
| `Java_Spring_Framework_Study_Guide.md` | IoC/DI, bean lifecycle, scopes, MVC/REST, Data JPA, transactions, AOP | ⭐⭐⭐ HIGH |
| `Spring_Boot_Junior_Developer_Interview_Questions.md` | Spring Boot, REST API, JPA, Testing, Annotations | ⭐⭐⭐ HIGH |
| `JPA_Hibernate_Interview_Questions.md` | Entities, relationships, fetch types, N+1, cascades, transactions, caching | ⭐⭐⭐ HIGH |
| `Spring_Security_JWT_Interview_Questions.md` | Spring Security, JWT, CORS, BCrypt | ⭐⭐⭐ HIGH |
| `Spring_Bean_Validation_Guide.md` | @Valid/@Validated, built-in constraints, custom validators, error handling | ⭐⭐⭐ HIGH |
| `Tomcat_In_Spring_Boot_Guide.md` | Embedded vs external Tomcat, servlet container, request lifecycle, thread pool tuning, DispatcherServlet, alternative servers (Jetty/Undertow/Netty) | ⭐⭐ MEDIUM |
| `Spring_AOP_And_Bean_Lifecycle.md` | 12-step bean lifecycle, BeanPostProcessor, AOP, proxies, @Transactional internals | ⭐⭐ MEDIUM |
| `Spring_OAuth2_OIDC_Security.md` | OAuth2 grant types, OIDC, JWT validation, resource/client server, social login | ⭐⭐ MEDIUM |
| `Spring_WebFlux_Reactive.md` | Reactive streams, Mono/Flux, backpressure, schedulers, WebClient, R2DBC | ⭐ LOWER |

### Databases & Caching
| File | Topics Covered | Priority |
|------|---------------|---------|
| `Database_Concepts_Interview_Questions.md` | ACID, Normalization, SQL, Transactions, Indexing, Locks | ⭐⭐⭐ HIGH |
| `MySQL_Interview_Questions.md` | MySQL, Indexes, Joins, Optimization, Storage engines | ⭐⭐ MEDIUM |
| `SQL_Advanced_Window_Functions.md` | Window functions, ranking, CTEs, EXPLAIN, advanced queries | ⭐⭐ MEDIUM |
| `Database_Schema_Design_Patterns.md` | ERD, normalization, keys, soft delete, audit, multi-tenancy | ⭐⭐ MEDIUM |
| `Redis_Interview_Questions.md` | Data structures, caching patterns, expiry, persistence, pub/sub, cluster | ⭐⭐ MEDIUM |
| `PostgreSQL_Interview_Questions.md` | PostgreSQL, MVCC, JSONB, advanced features | ⭐ LOWER |
| `MongoDB_Interview_Questions.md` | MongoDB CRUD, Aggregation, Indexing, Sharding | ⭐ LOWER |

### Messaging
| File | Topics Covered | Priority |
|------|---------------|---------|
| `Kafka_Interview_Questions.md` | Topics/partitions/offsets, producers, consumers, delivery semantics, Spring Kafka | ⭐⭐ MEDIUM |
| `Kafka_RabbitMQ_Interview_Questions.md` | Kafka vs RabbitMQ, consumer groups, exchanges, AMQP, messaging patterns | ⭐⭐ MEDIUM |

### Architecture, System Design & Patterns
| File | Topics Covered | Priority |
|------|---------------|---------|
| `System_Design_Microservices_Interview_Questions.md` | Scalability, caching, microservices, API Gateway, circuit breaker | ⭐⭐ MEDIUM |
| `GoF_Design_Patterns_Complete.md` | All 23 Gang of Four patterns with analogies and Spring usage | ⭐⭐ MEDIUM |
| `Software_Engineering_Principles_Interview_Questions.md` | Agile, Scrum, Kanban, SOLID, Design Patterns, Clean Code, TDD | ⭐⭐ MEDIUM |
| `Clean_Architecture_Study_Guide.md` | 4 layers, dependency rule, ports & adapters, DDD, full Spring example | ⭐⭐ MEDIUM |
| `Architecture_Topics_For_Java_Developers.md` | DB scaling, caching, messaging, reliability, observability for Java devs | ⭐⭐ MEDIUM |
| `Distributed_Systems_Core_Concepts_Study_Guide.md` | Load balancing, CAP, eventual consistency, distributed locks, sharding | ⭐⭐ MEDIUM |
| `Microservices_Saga_CQRS_EventSourcing.md` | Saga, CQRS, event sourcing, outbox, compensating transactions | ⭐ LOWER |
| `Software_Architect_Study_Guide.md` | Architecture styles, CAP/BASE, cloud, security, ADRs | ⭐ LOWER |
| `Backend_Engineering_Mastery_Roadmap.md` | SQL/Postgres internals, Redis, Kafka, system design roadmap | MASTERY |
| `World_Class_Software_Engineer_Roadmap.md` | Full mastery roadmap: DSA, System Design, Architecture, DDD, Security, Leadership | MASTERY |

### Networking & APIs
| File | Topics Covered | Priority |
|------|---------------|---------|
| `REST_API_HTTP_Interview_Questions.md` | HTTP methods, status codes, REST design, versioning, pagination | ⭐⭐⭐ HIGH |
| `Networking_Interview_Questions.md` | OSI Model, TCP/UDP, DNS, HTTPS, Ports, WebSockets, Load Balancing | ⭐⭐⭐ HIGH |
| `Networking_Concepts_Cloud_DevOps.md` | Subnetting/CIDR, TLS, NGINX, VPC, Kubernetes networking, service mesh | ⭐⭐ MEDIUM |

### DevOps, Cloud & Tools
| File | Topics Covered | Priority |
|------|---------------|---------|
| `Git_Docker_DevOps_Interview_Questions.md` | Git commands, Docker, Maven, CI/CD | ⭐⭐ MEDIUM |
| `Docker_Concepts_Study_Guide.md` | Containers vs VMs, images, networking, volumes, Compose, security | ⭐⭐ MEDIUM |
| `DevOps_Core_Concepts.md` | DevOps culture, CI/CD, IaC, orchestration, monitoring, SRE | ⭐⭐ MEDIUM |
| `CI_CD_Pipelines_Deep_Dive.md` | GitHub Actions, Jenkins, GitLab CI, deployment strategies, quality gates | ⭐⭐ MEDIUM |
| `Kubernetes_Learning_Guide.md` | Hands-on K8s from zero: pods, deployments, services, config, probes, scaling, storage, Spring Boot deploy, debugging | ⭐⭐ MEDIUM |
| `Kubernetes_Interview_Questions.md` | K8s architecture, objects, services, scaling, probes, RBAC, Helm | ⭐⭐ MEDIUM |
| `Observability_Tracing_Metrics_Logging.md` | Logs/metrics/traces, OpenTelemetry, Actuator, Prometheus/Grafana, ELK | ⭐⭐ MEDIUM |
| `AWS_Cloud_Interview_Questions.md` | IAM, EC2, S3, VPC, RDS, Lambda, load balancing, CloudWatch | ⭐ LOWER |
| `TERRAFORM_AWS_STUDY_GUIDE.md` | Infrastructure as Code, Terraform HCL, AWS provisioning | ⭐ LOWER |
| `Vim_Nano_Editor_Study_Guide.md` | Vim & Nano editor commands and workflows | ⭐ LOWER |

### Data Structures & Algorithms
| File | Topics Covered | Priority |
|------|---------------|---------|
| `DSA_Interview_Questions.md` | Arrays, Linked List, Stack, Binary Search, Sorting, Trees, Graphs (BFS/DFS), Recursion/Backtracking | ⭐⭐⭐ HIGH |
| `DSA_Dynamic_Programming.md` | DP concept, memoization vs tabulation, classic 1D/2D problems | ⭐⭐ MEDIUM |

### Frontend / Web (for full-stack roles)
| File | Topics Covered | Priority |
|------|---------------|---------|
| `JavaScript_Interview_Questions.md` | var/let/const, Closures, Prototypes, Promises, Event Loop, ES6+, DOM | ⭐⭐ MEDIUM |
| `typescript.md` | Types, interfaces, generics, TypeScript fundamentals | ⭐⭐ MEDIUM |
| `React_Interview_Questions.md` | Components, hooks, state, props, lifecycle | ⭐⭐ MEDIUM |
| `NextJS_Interview_Questions.md` | SSR/SSG, routing, data fetching, Next.js features | ⭐ LOWER |
| `HTML_Interview_Questions.md` | Semantic HTML, forms, accessibility | ⭐ LOWER |
| `CSS_Interview_Study_Guide.md` | Box model, flexbox, grid, responsive design | ⭐ LOWER |

### Behavioral
| File | Topics Covered | Priority |
|------|---------------|---------|
| `HR_Behavioral_Interview_Questions.md` | STAR method, HR questions, Tell me about yourself | ⭐⭐⭐ HIGH |

---

## Java Core — Quick Reference

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
| Constructors | Yes | No |
| Inheritance | Single (`extends`) | Multiple (`implements`) |

### Key Points
- `==` compares references; `equals()` compares values
- `final variable` = can't reassign; `final method` = can't override; `final class` = can't extend
- **String** = immutable, thread-safe; **StringBuilder** = mutable, not thread-safe (fast); **StringBuffer** = mutable, thread-safe (slower)

---

## Collections — Quick Reference

### HashMap Internal Working
```
1. key.hashCode() → calculate hash
2. hash & (capacity-1) → find bucket index
3. Bucket empty → store new Node; not empty → chain (linked list)
4. Chain length >= 8 → convert to Red-Black tree
5. size > capacity * 0.75 → resize (double capacity)
```

### ArrayList vs LinkedList

| | ArrayList | LinkedList |
|--|-----------|------------|
| get(i) | O(1) | O(n) |
| add/remove at end | O(1) | O(1) |
| add/remove at middle | O(n) | O(n) |
| Memory | Less | More (2 pointers/node) |

- **HashMap** = no order, O(1); **LinkedHashMap** = insertion order, O(1); **TreeMap** = sorted by key, O(log n)

---

## Exception Handling — Quick Reference

```java
// Checked = must handle (IOException, SQLException)
// Unchecked = runtime (NullPointerException, ArrayIndexOutOfBoundsException)

try {
    // risky code
} catch (SpecificException e) {
    // handle specific first
} catch (Exception e) {
    // general last
} finally {
    // ALWAYS runs (cleanup)
}

// try-with-resources (auto-close)
try (FileReader fr = new FileReader("file.txt")) {
    // fr.close() called automatically
}
```

---

## Java 8 — Quick Reference

```java
// Lambda
Runnable r = () -> System.out.println("Hello");

// Stream: filter → map → collect
List<String> result = names.stream()
    .filter(n -> n.startsWith("A"))
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// Optional (safe usage)
user.orElse(new User());
user.orElseThrow(() -> new UserNotFoundException());
user.ifPresent(u -> System.out.println(u.getName()));
```

---

## Spring Boot — Quick Reference

### Key Annotations
```
@SpringBootApplication = @Configuration + @EnableAutoConfiguration + @ComponentScan
@RestController        = @Controller + @ResponseBody
@Transactional         = all DB ops succeed or all rollback
@Valid                 = trigger bean validation
```

### REST Controller Template
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @GetMapping             public List<User> getAll() { ... }
    @GetMapping("/{id}")    public User getById(@PathVariable Long id) { ... }
    @PostMapping            public ResponseEntity<User> create(@Valid @RequestBody UserDTO dto) { ... }
    @PutMapping("/{id}")    public User update(@PathVariable Long id, @RequestBody UserDTO dto) { ... }
    @DeleteMapping("/{id}") public ResponseEntity<Void> delete(@PathVariable Long id) { ... }
}
```

### JPA Entity Template
```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
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

## HTTP Status Codes — Must Know

| Code | Meaning | When |
|------|---------|------|
| 200 | OK | GET/PUT success |
| 201 | Created | POST success |
| 204 | No Content | DELETE success |
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Not logged in |
| 403 | Forbidden | No permission |
| 404 | Not Found | Resource missing |
| 409 | Conflict | Duplicate resource |
| 500 | Server Error | Bug in code |

---

## JWT Flow — Must Know

```
1. POST /login {username, password}
2. Server validates → creates JWT (signed with secret key)
3. Client sends: Authorization: Bearer <token>
4. Server: verify signature → extract username → check roles → respond
```

**Structure:** `header.payload.signature` (base64 encoded, NOT encrypted)

---

## SQL — Quick Reference

```sql
INNER JOIN = only matching rows in both tables
LEFT JOIN  = all from left + matching from right (NULL if no match)

-- GROUP BY + HAVING
SELECT user_id, COUNT(*) FROM orders
GROUP BY user_id HAVING COUNT(*) > 5;  -- HAVING filters after grouping

-- Window function
SELECT name, salary, RANK() OVER (ORDER BY salary DESC) FROM employees;
```

---

## DSA — Quick Patterns

```java
// Two Sum → HashMap
// Max Subarray → Kadane's (track currentSum, reset to 0 if negative)
// Valid Parentheses → Stack
// Reverse String → Two pointers
// Find Cycle → Fast/Slow pointers (Floyd's)
// Binary Search → left + (right-left)/2 to avoid overflow
```

---

## Networking — Quick Reference

### OSI Layers (mnemonic: "Please Do Not Throw Sausage Pizza Away")
```
7-Application → HTTP, HTTPS, DNS   4-Transport → TCP/UDP
3-Network     → IP routing          2-Data Link → MAC addresses
1-Physical    → Cables, Wi-Fi
```

- **TCP** = reliable, ordered, slower | **UDP** = fast, no guarantee
- **TCP handshake**: SYN → SYN-ACK → ACK
- **HTTPS** = port 443, TLS encrypted | **HTTP** = port 80, plain text
- **Key ports**: 22=SSH, 53=DNS, 3306=MySQL, 5432=PostgreSQL, 6379=Redis, 8080=Spring Boot

---

## Git — Must Know Commands

```bash
git checkout -b feature   # Create + switch branch
git add .                 # Stage all
git commit -m "message"   # Save
git push origin main      # Upload
git pull origin main      # Download latest
git merge feature         # Merge branch
git stash                 # Hide WIP changes
git log --oneline         # Short history
```

---

## Interview Day Checklist

### Before
- [ ] Research company tech stack
- [ ] Review your projects — explain every decision
- [ ] Prepare "Tell me about yourself" (60-second pitch)
- [ ] Prepare 2-3 questions to ask the interviewer

### During Technical Interview
- [ ] Clarify the question before coding
- [ ] Talk through your approach first
- [ ] Handle edge cases (null, empty, negative)
- [ ] Mention time/space complexity when done

### Typical Junior Java Interview Rounds
```
Round 1: Online Coding (DSA, Java basics)
Round 2: Technical — Java Core + Spring Boot + SQL
Round 3: Project deep-dive + design questions
Round 4: HR — behavioral (STAR) + salary
```

---

## Top 10 Questions You Will Be Asked

1. **"Tell me about yourself"** — 60-second pitch
2. **"How does HashMap work internally?"** — array + linked list/tree, hash, bucket
3. **"Abstract class vs interface?"** — see table above
4. **"What is Spring Boot auto-configuration?"** — reads classpath, configures beans automatically
5. **"What is @Transactional?"** — all DB ops succeed or all rollback
6. **"What is JWT and how does it work?"** — header.payload.signature, stateless auth
7. **"What is SOLID?"** — Single Responsibility, Open/Closed, Liskov, Interface Segregation, Dependency Inversion
8. **"Explain OOP with examples"** — Animal/Dog (inheritance), BankAccount (encapsulation)
9. **"== vs equals()?"** — reference vs value comparison
10. **"Where do you see yourself in 5 years?"** — structured, realistic, ambitious

---

*Last Updated: 2026-06-18*

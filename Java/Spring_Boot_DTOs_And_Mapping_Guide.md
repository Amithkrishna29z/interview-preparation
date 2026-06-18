# Spring Boot DTOs & Entity↔DTO Mapping Guide

## Overview

DTOs (Data Transfer Objects) are plain Java objects used to carry data between layers of an application — most commonly between the service layer and the HTTP API. Every junior Spring Boot developer touches DTOs daily: you receive one in a `@RequestBody`, convert it to an entity to save, then convert the saved entity back to a DTO for the response. This guide covers the why, the how (manual, MapStruct, ModelMapper), modern approaches with records and Lombok, and the common mistakes that trip up juniors.

---

## Table of Contents

1. [What is a DTO and Why Use One?](#what-is-a-dto-and-why-use-one)
2. [Entity vs DTO Comparison](#entity-vs-dto-comparison)
3. [Where Mapping Lives in a Layered App](#where-mapping-lives-in-a-layered-app)
4. [Manual Mapping — The Baseline](#manual-mapping--the-baseline)
5. [MapStruct — Compile-Time Code Generation](#mapstruct--compile-time-code-generation)
6. [ModelMapper — Runtime Reflection](#modelMapper--runtime-reflection)
7. [Java Records as DTOs](#java-records-as-dtos)
8. [Lombok for DTOs](#lombok-for-dtos)
9. [Validation on DTOs](#validation-on-dtos)
10. [Common Mistakes & Pitfalls](#common-mistakes--pitfalls)
11. [Common Interview Questions](#common-interview-questions)
12. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What is a DTO and Why Use One?

A **DTO** is a simple object whose only job is to carry data. It has no JPA annotations, no business logic, and no database knowledge.

**Why not just expose your JPA entity directly from the controller?**

| Problem | What Goes Wrong |
|---|---|
| **Over-exposure** | Entity may contain password hashes, internal flags, or audit fields you never want in the API response |
| **Over-posting** | Client sends `{"id": 1, "role": "ADMIN"}` and accidentally updates a field that should never change |
| **Lazy-loading / serialization** | Jackson tries to serialize a `LAZY` collection that isn't loaded → `LazyInitializationException` or infinite recursion |
| **Schema coupling** | Renaming a DB column forces an API change; DTOs decouple the two |
| **Security** | Exposing entities leaks your internal data model to the world |

**The rule of thumb:** entities live in the persistence layer; DTOs live at the API boundary.

---

## Entity vs DTO Comparison

| Aspect | Entity | DTO |
|---|---|---|
| Annotations | `@Entity`, `@Table`, `@Column`, `@Id` | None (plain POJO) |
| Purpose | Maps to a database row | Carries data over the wire |
| Lifecycle | Managed by JPA (4 states) | Created and discarded freely |
| Validation | DB-level constraints | Bean Validation (`@NotNull`, etc.) |
| Relationships | `@OneToMany`, `@ManyToOne`, etc. | Flat fields or nested DTOs |
| Lombok | Avoid `@Data` (see Pitfalls) | `@Data` or records are fine |

---

## Where Mapping Lives in a Layered App

```
Controller  ──receives──▶  Request DTO
                                │
                           map DTO → Entity
                                │
Service ──calls──▶  Repository.save(entity)
                                │
                           map Entity → Response DTO
                                │
Controller  ──returns──▶  Response DTO (JSON)
```

**Rule:** mapping belongs in the **service layer**, not the controller or repository.

- Controller should stay thin — it only calls the service and returns the result.
- Repository only knows about entities.
- Service owns the business logic and the mapping.

---

## Manual Mapping — The Baseline

Every junior should be able to map by hand before reaching for a library. A static mapper method is the clearest approach.

```java
// The JPA entity
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;
    private String firstName;
    private String lastName;
    private String email;
    private String passwordHash; // must NOT appear in response DTO
    // getters/setters omitted
}

// Request DTO — what the client sends on POST /users
public class CreateUserRequest {
    @NotBlank private String firstName;
    @NotBlank private String lastName;
    @Email    private String email;
    @NotBlank private String password;
    // getters/setters
}

// Response DTO — what the API returns
public class UserResponse {
    private Long   id;
    private String firstName;
    private String lastName;
    private String email;
    // getters/setters
}

// Manual mapper — a plain static helper class
public class UserMapper {

    // Entity → Response DTO  (used when returning data)
    public static UserResponse toResponse(User user) {
        UserResponse dto = new UserResponse();
        dto.setId(user.getId());
        dto.setFirstName(user.getFirstName());
        dto.setLastName(user.getLastName());
        dto.setEmail(user.getEmail());
        // passwordHash is intentionally omitted
        return dto;
    }

    // Request DTO → Entity  (used when creating/updating)
    public static User toEntity(CreateUserRequest req) {
        User user = new User();
        user.setFirstName(req.getFirstName());
        user.setLastName(req.getLastName());
        user.setEmail(req.getEmail());
        // password hashing happens in the service, not here
        return user;
    }
}
```

```java
// Service using the manual mapper
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;

    public UserResponse createUser(CreateUserRequest req) {
        User user = UserMapper.toEntity(req);
        user.setPasswordHash(hashPassword(req.getPassword()));
        User saved = userRepository.save(user);
        return UserMapper.toResponse(saved);    // map before returning
    }
}
```

**Pro:** zero dependencies, very readable.  
**Con:** tedious for large objects; easy to forget a new field.

---

## MapStruct — Compile-Time Code Generation

MapStruct generates the mapper implementation **at compile time** from an interface you define. No reflection at runtime — it produces plain Java code you can read and debug.

**Add dependency** (`pom.xml`):
```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
</dependency>
<plugin>
    <!-- mapstruct-processor must be in the maven-compiler-plugin annotationProcessorPaths -->
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <annotationProcessorPaths>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>1.5.5.Final</version>
            </path>
            <!-- add lombok-mapstruct-binding if you use Lombok too -->
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

**Define the mapper interface:**
```java
// componentModel = "spring" → MapStruct generates a @Component so you can @Autowired it
@Mapper(componentModel = "spring")
public interface UserMapper {

    // Same-name fields are mapped automatically
    // @Mapping handles renames or ignored fields
    @Mapping(target = "passwordHash", ignore = true)  // never copy to DTO
    UserResponse toResponse(User user);

    @Mapping(target = "id",           ignore = true)  // DB sets the id
    @Mapping(target = "passwordHash", ignore = true)  // set in service
    User toEntity(CreateUserRequest req);
}
```

MapStruct generates an implementation class like this (you never write it):
```java
// Generated by MapStruct at compile time — lives in target/generated-sources
@Component
public class UserMapperImpl implements UserMapper {
    @Override
    public UserResponse toResponse(User user) {
        if (user == null) return null;
        UserResponse response = new UserResponse();
        response.setId(user.getId());
        response.setFirstName(user.getFirstName());
        // ... etc.
        return response;
    }
}
```

**Using it in the service:**
```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    private final UserMapper userMapper;           // Spring injects the generated impl

    public UserResponse createUser(CreateUserRequest req) {
        User user = userMapper.toEntity(req);
        user.setPasswordHash(hashPassword(req.getPassword()));
        return userMapper.toResponse(userRepository.save(user));
    }
}
```

**Why MapStruct is preferred:**
- Errors at **compile time** (mismatched types, missing fields) — not at runtime
- Zero reflection = fast
- Generated code is readable and debuggable
- Works perfectly with Lombok (with correct annotation processor ordering)

---

## ModelMapper — Runtime Reflection

ModelMapper matches fields by name using reflection at runtime. Less configuration for simple cases, but:

```java
// One-time Spring Bean
@Bean
public ModelMapper modelMapper() { return new ModelMapper(); }

// Usage
UserResponse dto = modelMapper.map(user, UserResponse.class);
```

**When to use it:** quick prototypes or legacy codebases. For new projects, prefer MapStruct.

| | MapStruct | ModelMapper |
|---|---|---|
| Mapping time | Compile time | Runtime |
| Error detection | Compile-time | Runtime exception |
| Performance | Fast (plain code) | Slower (reflection) |
| Debuggability | High (readable generated code) | Low |
| Configuration | Interface + `@Mapping` | Fluent API or convention |
| Trend | Growing (standard choice) | Losing favor |

---

## Java Records as DTOs

Java 16+ records are ideal DTOs: immutable, compact, and serializable by Jackson with no extra config.

```java
// Request DTO as a record
public record CreateUserRequest(
    @NotBlank String firstName,
    @NotBlank String lastName,
    @Email    String email,
    @NotBlank String password
) {}

// Response DTO as a record — immutable, no setters needed
public record UserResponse(
    Long   id,
    String firstName,
    String lastName,
    String email
) {}
```

Records give you a canonical constructor, `equals`, `hashCode`, and `toString` for free. Jackson reads/writes them automatically in Spring Boot 2.5+.

**Note:** records cannot be JPA entities (JPA requires a no-arg constructor and mutable fields), so they are DTO-only.

---

## Lombok for DTOs

Lombok reduces boilerplate for class-based DTOs. Key annotations:

```java
@Data               // = @Getter + @Setter + @ToString + @EqualsAndHashCode + @RequiredArgsConstructor
@Builder            // enables: CreateUserRequest.builder().email("a@b.com").build()
@NoArgsConstructor  // required by Jackson for deserialization
@AllArgsConstructor // convenient for tests
public class CreateUserRequest {
    @NotBlank private String firstName;
    @NotBlank private String lastName;
    @Email    private String email;
    @NotBlank private String password;
}
```

**Common combo for DTOs:** `@Data + @NoArgsConstructor + @AllArgsConstructor` or `@Builder + @Getter + @NoArgsConstructor + @AllArgsConstructor`.

### IMPORTANT: Lombok on JPA Entities — Pitfalls

| Annotation | On DTO | On JPA Entity |
|---|---|---|
| `@Data` | Fine | **Avoid** — `equals`/`hashCode` use all fields including the `id`; breaks Hibernate's dirty-checking and entity equality in sets |
| `@ToString` | Fine | **Avoid** — can traverse LAZY relationships → `LazyInitializationException` |
| `@EqualsAndHashCode` | Fine | **Avoid** on entities unless you use business keys explicitly |
| `@Getter / @Setter` | Fine | Fine |
| `@Builder` | Fine | Fine (but add `@NoArgsConstructor + @AllArgsConstructor` alongside it) |
| `@RequiredArgsConstructor` | Fine | Preferred for constructor injection in service/component classes |

**On entities:** use `@Getter + @Setter` only, or write equals/hashCode based on a business key.

---

## Validation on DTOs

Put Bean Validation annotations on the **request DTO** fields, then trigger validation in the controller with `@Valid`.

```java
// DTO with constraints
public record CreateUserRequest(
    @NotBlank(message = "First name is required")
    String firstName,

    @Email(message = "Must be a valid email")
    @NotBlank
    String email,

    @Size(min = 8, message = "Password must be at least 8 characters")
    String password
) {}
```

```java
@RestController
@RequestMapping("/users")
@RequiredArgsConstructor
public class UserController {
    private final UserService userService;

    @PostMapping
    public ResponseEntity<UserResponse> create(
            @Valid @RequestBody CreateUserRequest req) {  // @Valid triggers validation
        return ResponseEntity.status(201).body(userService.createUser(req));
    }
}
```

If validation fails, Spring throws `MethodArgumentNotValidException` → returns `400 Bad Request` automatically. Handle it with `@ExceptionHandler` in a `@ControllerAdvice` class to return clean error messages.

See **Spring_Bean_Validation_Guide.md** for the full treatment.

---

## Common Mistakes & Pitfalls

**1. Exposing the entity directly from the controller**
```java
// BAD — leaks passwordHash, internal audit fields, triggers lazy loading
@GetMapping("/{id}")
public User getUser(@PathVariable Long id) { return userRepository.findById(id).get(); }

// GOOD
@GetMapping("/{id}")
public UserResponse getUser(@PathVariable Long id) { return userService.getUser(id); }
```

**2. Mapping in the wrong layer**
```java
// BAD — controller doing mapping logic
@PostMapping
public UserResponse create(@RequestBody CreateUserRequest req) {
    User u = new User(); u.setEmail(req.getEmail()); // mapping in controller
    return userRepository.save(u); // also returns entity — double mistake
}
// GOOD — controller delegates entirely to service
```

**3. Infinite recursion on bidirectional relationships**

If `User` has `List<Order> orders` and `Order` has `User user`, serializing `User` directly will recurse. Fix: map to a flat DTO that breaks the cycle, or use `@JsonIgnore`/`@JsonManagedReference` on the entity side.

**4. N+1 from mapping lazy fields**

Fetching a `List<User>` and then mapping each user's `user.getOrders()` triggers one extra query per user.  
Fix: use `JOIN FETCH` or `@EntityGraph` in the repository query so orders are loaded in one query.

**5. Forgetting `@NoArgsConstructor` on Lombok DTOs**

Jackson's default deserializer needs a no-arg constructor. `@Builder` alone removes the no-arg constructor — always pair it with `@NoArgsConstructor + @AllArgsConstructor`.

**6. Putting `@Valid` in the service instead of the controller**

`@Valid` on method parameters only works when called through a Spring proxy (i.e., via the controller). Putting it on a service method called internally does nothing without `@Validated` on the class.

---

## Common Interview Questions

**Q: What is a DTO and why not just return the entity?**  
A DTO is a plain object that carries only the data the client needs. Returning entities directly risks exposing sensitive fields (like password hashes), triggers lazy-loading exceptions during serialization, and couples your API contract to your database schema. DTOs solve all three problems.

**Q: What is the difference between MapStruct and ModelMapper?**  
MapStruct generates plain Java code at compile time — errors show up during the build and performance is fast. ModelMapper uses reflection at runtime, which is slower and fails at startup or runtime instead of compile time. MapStruct is the current industry standard for new projects.

**Q: Where should mapping happen in a layered Spring Boot app?**  
In the service layer. The controller receives a request DTO and returns a response DTO; it should not do any mapping itself. The repository deals only with entities. The service sits in between and owns both the business logic and the entity↔DTO conversion.

**Q: Why should you avoid `@Data` on a JPA entity?**  
`@Data` generates `equals` and `hashCode` using all fields, including the `id`. Before an entity is saved, `id` is null, so two unsaved entities are considered equal. It also generates `@ToString`, which can accidentally traverse lazy relationships and cause a `LazyInitializationException`. Use `@Getter + @Setter` on entities instead.

**Q: What happens if you serialize a JPA entity with a LAZY collection directly?**  
Jackson tries to access the collection while the Hibernate session may already be closed, causing a `LazyInitializationException`. Map to a DTO before the session closes (in the service layer, within the transaction) to avoid this.

**Q: Can you use Java records as DTOs?**  
Yes, and they're a great fit. Records are immutable, auto-generate equals/hashCode/toString, and Jackson handles them natively from Spring Boot 2.5+. They cannot be JPA entities (JPA requires a mutable no-arg constructor), so use them only as DTOs.

---

## Quick Reference Cheat Sheet

```
DTO = plain Java class/record — no JPA annotations, no business logic
Entity = JPA-managed class — DO NOT expose directly from controllers

Mapping flow:
  POST: @RequestBody RequestDTO → service maps → Entity → save → map → ResponseDTO → return
  GET:  repository returns Entity → service maps → ResponseDTO → return

Lombok on DTOs:   @Data + @NoArgsConstructor + @AllArgsConstructor  (or records)
Lombok on Entity: @Getter + @Setter ONLY — never @Data or @ToString

MapStruct setup:
  @Mapper(componentModel = "spring")          → Spring manages the mapper as a bean
  @Mapping(target = "field", ignore = true)   → skip a field
  @Mapping(source = "src", target = "dest")   → rename a field
  Inject with: private final UserMapper mapper;  (@RequiredArgsConstructor)

ModelMapper:  modelMapper.map(source, DestClass.class)   — prototypes only

Validation:   @Valid @RequestBody RequestDTO   → triggers Bean Validation on the DTO
              Failure → MethodArgumentNotValidException → 400 Bad Request

Common pitfalls:
  ✗ Returning entity from controller       → exposes internals, lazy-load crash
  ✗ Mapping in the controller              → wrong layer
  ✗ @ToString on entity with lazy fields  → LazyInitializationException
  ✗ No @NoArgsConstructor with @Builder   → Jackson deserialization failure
  ✗ Accessing lazy collection while mapping in service → N+1 or session closed
  ✓ Map entity→DTO inside @Transactional service method
  ✓ Use JOIN FETCH / @EntityGraph when you know you'll need related data
```

---

*Last Updated: 2026-06-18*

# Spring / Jakarta Bean Validation Interview Questions & Study Guide

## Overview

Bean Validation is how Spring Boot applications check that incoming data is valid **before** it reaches your business logic. It's one of the most practical topics in backend interviews because every real API has to reject bad input. Expect questions on built-in constraints, `@Valid` vs `@Validated`, handling validation errors cleanly, and writing custom validators.

---

## Table of Contents

1. [What is Bean Validation?](#what-is-bean-validation)
2. [Why Validate at the Boundary?](#why-validate-at-the-boundary)
3. [Setup & Dependency](#setup--dependency)
4. [Built-in Constraints](#built-in-constraints)
5. [@NotNull vs @NotBlank vs @NotEmpty](#notnull-vs-notblank-vs-notempty)
6. [@Valid vs @Validated](#valid-vs-validated)
7. [Validating @RequestBody DTOs](#validating-requestbody-dtos)
8. [Nested & Collection Validation](#nested--collection-validation)
9. [Validating @PathVariable & @RequestParam](#validating-pathvariable--requestparam)
10. [Handling Validation Errors](#handling-validation-errors)
11. [Custom Constraint Annotation](#custom-constraint-annotation)
12. [Validation Groups (Create vs Update)](#validation-groups-create-vs-update)
13. [Cross-Field Validation](#cross-field-validation)
14. [Common Mistakes & Pitfalls](#common-mistakes--pitfalls)
15. [Common Interview Questions](#common-interview-questions)
16. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What is Bean Validation?

Bean Validation is a Java **standard** (specification) for declaring validation rules with annotations. You put annotations like `@NotNull` or `@Email` directly on your fields, and a validation engine checks them for you.

Three layers:

```
Jakarta Bean Validation (JSR-380)  → the SPECIFICATION — defines annotations & interfaces
Hibernate Validator                → the IMPLEMENTATION — the engine that runs the checks
spring-boot-starter-validation     → bundles Hibernate Validator + wires it into Spring MVC
```

> **Interview Tip**: "JSR-380" is Bean Validation 2.0. The package moved from `javax.validation` (Spring Boot 2) to `jakarta.validation` (Spring Boot 3+).

---

## Why Validate at the Boundary?

**Never trust client input.** The "boundary" is the controller layer where external data first arrives. Validate there so bad data never reaches your business logic or database.

| Reason | Explanation |
|---|---|
| **Security** | Stops malformed/malicious input before it touches your logic or DB |
| **Data integrity** | Prevents null names or negative ages from being saved |
| **Clear errors** | Returns a clean `400 Bad Request` instead of a `500` crash |
| **Less boilerplate** | One annotation replaces many manual `if` checks |

---

## Setup & Dependency

In Spring Boot 3, validation is **not** included by default.

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

```groovy
// Gradle
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

> **Common Mistake**: Forgetting this dependency. Your `@NotNull` / `@Email` annotations will compile fine but **silently do nothing** at runtime. If validation "isn't firing," check this first.

Annotations come from `jakarta.validation.constraints`.

---

## Built-in Constraints

```java
public class UserRegistrationRequest {

    @NotBlank
    @Size(min = 2, max = 50)
    private String name;

    @NotBlank
    @Email
    private String email;

    @NotNull
    @Min(18) @Max(120)
    private Integer age;

    @NotBlank
    @Pattern(regexp = "^[A-Za-z0-9_]{3,20}$")
    private String username;

    @NotNull
    @Positive
    private BigDecimal salary;

    @Past
    private LocalDate birthDate;

    @Future
    private LocalDate appointmentDate;

    @DecimalMin("0.0") @DecimalMax("100.0")
    private BigDecimal discountPercent;
}
```

### Constraint Reference Table

| Annotation | What it checks | Works On |
|---|---|---|
| `@NotNull` | Value is not null | Any type |
| `@NotEmpty` | Not null AND size > 0 | String, Collection, Map, Array |
| `@NotBlank` | Not null AND has non-whitespace text | String only |
| `@Size(min, max)` | Length/size within range | String, Collection, Map, Array |
| `@Min(n)` / `@Max(n)` | Number within integer range | Integer types |
| `@DecimalMin` / `@DecimalMax` | Number within range (decimals) | BigDecimal, String, numbers |
| `@Positive` / `@PositiveOrZero` | Number > 0 / >= 0 | Numbers |
| `@Email` | Valid email format | String |
| `@Pattern(regexp)` | Matches a regular expression | String |
| `@Past` / `@Future` | Date/time in the past/future | LocalDate, LocalDateTime, etc. |
| `@Digits(integer, fraction)` | Limits digit count before/after decimal | Numbers, String |
| `@AssertTrue` / `@AssertFalse` | boolean must be true / false | Boolean |

> **Interview Tip**: Every constraint accepts a custom `message`, e.g. `@Min(value = 18, message = "Must be at least 18")`.

---

## @NotNull vs @NotBlank vs @NotEmpty

One of the **most common interview questions**. The differences are subtle but important.

| Annotation | Null fails? | Empty `""` fails? | Whitespace `"   "` fails? | Use On |
|---|---|---|---|---|
| `@NotNull` | Yes | No | No | Any type |
| `@NotEmpty` | Yes | Yes | No | String, Collection, Map, Array |
| `@NotBlank` | Yes | Yes | Yes | **String only** |

```java
@NotNull   private String a;   // a = null → FAIL;  a = "" → PASS;  a = "  " → PASS
@NotEmpty  private String b;   // b = null → FAIL;  b = "" → FAIL;  b = "  " → PASS
@NotBlank  private String c;   // c = null → FAIL;  c = "" → FAIL;  c = "  " → FAIL
```

**Simple rules:**
- **Strings** → use `@NotBlank` (strictest, catches null, `""`, and `"   "`).
- **Collections/Lists/Maps** → use `@NotEmpty`.
- **Numbers, booleans, dates** → use `@NotNull`.

> **Common Mistake**: Putting `@NotBlank` on an `Integer` — it only works on strings. Use `@NotNull` for required numbers.

---

## @Valid vs @Validated

| | `@Valid` | `@Validated` |
|---|---|---|
| Comes from | `jakarta.validation` (standard) | `org.springframework.validation` (Spring) |
| Supports validation **groups**? | No | **Yes** |
| Triggers **nested** validation? | **Yes** | No |
| Typical use | On `@RequestBody` params and nested fields | On a **class** for method-level / param validation |

```java
// @Valid — validate the request body and cascade into nested objects
@PostMapping("/users")
public User create(@Valid @RequestBody UserRequest request) { ... }

// @Validated — on the class: enables @PathVariable/@RequestParam validation and groups
@Validated
@RestController
public class UserController {

    @PostMapping("/users")
    public User create(@Validated(OnCreate.class) @RequestBody UserRequest req) { ... }
}
```

**Practical takeaway:** Use `@Valid` on `@RequestBody` and nested fields. Use `@Validated` on the **class** for `@PathVariable`/`@RequestParam` or validation groups.

---

## Validating @RequestBody DTOs

```java
// DTO with constraints
public class CreateProductRequest {
    @NotBlank(message = "Product name is required")
    private String name;

    @NotNull
    @Positive(message = "Price must be greater than zero")
    private BigDecimal price;

    @Min(value = 0, message = "Stock cannot be negative")
    private int stock;
}
```

```java
// Controller — @Valid before @RequestBody is the trigger
@PostMapping
public ResponseEntity<Product> create(@Valid @RequestBody CreateProductRequest request) {
    // If validation fails, Spring throws MethodArgumentNotValidException
    // and this method body NEVER runs
    Product saved = productService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(saved);
}
```

> **Interview Tip**: Without `@Valid`, Spring passes an invalid DTO straight into your method — the annotations on the DTO do nothing on their own.

---

## Nested & Collection Validation

By default, validation does **not** go into nested objects. Add `@Valid` on the nested field to cascade.

```java
public class OrderRequest {

    @NotBlank
    private String orderNumber;

    @NotNull
    @Valid                          // cascade into Address — without this, Address constraints are IGNORED
    private Address shippingAddress;

    @NotEmpty                       // list must not be null/empty
    @Valid                          // cascade into each OrderItem
    private List<OrderItem> items;
}

public class Address {
    @NotBlank private String street;
    @NotBlank private String city;
    @Pattern(regexp = "\\d{5}") private String zipCode;
}
```

> **Common Mistake**: Forgetting `@Valid` on nested fields. The outer DTO passes, but the broken nested object slips through unchecked.

---

## Validating @PathVariable & @RequestParam

Put constraints directly on parameters AND annotate the **class** with `@Validated`.

```java
@Validated                          // REQUIRED on the class for param validation
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getById(@PathVariable @Min(1) Long id) {
        return userService.findById(id);
    }

    @GetMapping
    public List<User> search(
            @RequestParam @NotBlank String keyword,
            @RequestParam @Min(1) @Max(100) int pageSize) {
        return userService.search(keyword, pageSize);
    }
}
```

> **Key difference**: Invalid `@RequestBody` throws `MethodArgumentNotValidException`. Invalid `@PathVariable`/`@RequestParam` throws `ConstraintViolationException`. Handle these with **two separate** exception handlers.

---

## Handling Validation Errors

Centralize error handling with `@RestControllerAdvice` to return structured JSON instead of Spring's default error blob.

### The two exceptions to handle

| Exception | Thrown when |
|---|---|
| `MethodArgumentNotValidException` | `@Valid @RequestBody` DTO fails |
| `ConstraintViolationException` | `@PathVariable` / `@RequestParam` constraint fails |

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleBodyValidation(MethodArgumentNotValidException ex) {
        List<ApiError.FieldError> fieldErrors = ex.getBindingResult()
                .getFieldErrors()
                .stream()
                .map(err -> new ApiError.FieldError(err.getField(), err.getDefaultMessage()))
                .toList();

        return new ApiError(HttpStatus.BAD_REQUEST.value(), "Validation failed",
                            LocalDateTime.now(), fieldErrors);
    }

    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleParamValidation(ConstraintViolationException ex) {
        List<ApiError.FieldError> fieldErrors = ex.getConstraintViolations()
                .stream()
                .map(v -> new ApiError.FieldError(v.getPropertyPath().toString(), v.getMessage()))
                .toList();

        return new ApiError(HttpStatus.BAD_REQUEST.value(), "Validation failed",
                            LocalDateTime.now(), fieldErrors);
    }
}
```

### Response the client receives

```json
{
  "status": 400,
  "message": "Validation failed",
  "timestamp": "2026-06-18T10:15:30",
  "errors": [
    { "field": "email", "message": "must be a valid email" },
    { "field": "age",   "message": "must be greater than or equal to 18" }
  ]
}
```

> **Interview Tip**: `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`. Catches exceptions across all controllers in one place.

---

## Custom Constraint Annotation

When built-in constraints aren't enough, create your own. Requires **two parts**: the annotation and the validator.

```java
// PART 1: The annotation
@Documented
@Constraint(validatedBy = StrongPasswordValidator.class)
@Target({ FIELD })
@Retention(RUNTIME)
public @interface StrongPassword {
    String message() default "Password must be 8+ chars with upper, lower, digit and symbol";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

```java
// PART 2: The validator
public class StrongPasswordValidator implements ConstraintValidator<StrongPassword, String> {

    private static final Pattern PATTERN = Pattern.compile(
        "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[^A-Za-z0-9]).{8,}$"
    );

    @Override
    public boolean isValid(String password, ConstraintValidatorContext context) {
        if (password == null) return true;  // let @NotNull handle nulls
        return PATTERN.matcher(password).matches();
    }
}
```

```java
// Usage
public class SignupRequest {
    @NotBlank
    @StrongPassword
    private String password;
}
```

> **Interview Tip**: Return `true` for null inside the validator and let `@NotNull`/`@NotBlank` handle the null case. Keeps each constraint focused on one job.

---

## Validation Groups (Create vs Update)

Let the **same DTO** apply different constraints in different situations (e.g., `id` must be null on create, required on update).

```java
// 1. Define marker interfaces (empty — just labels)
public interface OnCreate {}
public interface OnUpdate {}

// 2. Assign constraints to groups
public class UserRequest {
    @Null(groups = OnCreate.class)     // on CREATE, id must be null
    @NotNull(groups = OnUpdate.class)  // on UPDATE, id is required
    private Long id;

    @NotBlank(groups = {OnCreate.class, OnUpdate.class})
    private String name;
}

// 3. Select the group at the controller using @Validated
@PostMapping("/users")
public User create(@Validated(OnCreate.class) @RequestBody UserRequest req) { ... }

@PutMapping("/users/{id}")
public User update(@Validated(OnUpdate.class) @RequestBody UserRequest req) { ... }
```

> **Key point**: Groups **only work with `@Validated`**, not `@Valid`.

---

## Cross-Field Validation

For rules involving two or more fields (e.g., password = confirmPassword), use a **class-level constraint**.

```java
// 1. Annotation with @Target(TYPE)
@Constraint(validatedBy = PasswordMatchValidator.class)
@Target({ TYPE })
@Retention(RUNTIME)
public @interface PasswordMatch {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// 2. Validator receives the whole object
public class PasswordMatchValidator implements ConstraintValidator<PasswordMatch, SignupRequest> {
    @Override
    public boolean isValid(SignupRequest req, ConstraintValidatorContext ctx) {
        if (req.getPassword() == null) return true;
        return req.getPassword().equals(req.getConfirmPassword());
    }
}

// 3. Apply on the class
@PasswordMatch
public class SignupRequest {
    @NotBlank private String password;
    @NotBlank private String confirmPassword;
}
```

---

## Common Mistakes & Pitfalls

### 1. Forgetting `@Valid` on @RequestBody
```java
// WRONG — validation never runs
public User create(@RequestBody UserRequest req) { ... }
// RIGHT
public User create(@Valid @RequestBody UserRequest req) { ... }
```

### 2. Validation not firing on service methods
```java
@Service
@Validated  // REQUIRED for method-level validation
public class UserService {
    public void register(@Valid UserRequest req) { ... }
}
```

### 3. Using `@NotNull` when you need `@NotBlank`
```java
@NotNull private String name;   // BUG: "" and "   " PASS
@NotBlank private String name;  // CORRECT for required strings
```

### 4. Forgetting `@Valid` on nested objects
```java
@NotNull private Address address;       // BUG: Address fields NOT checked
@NotNull @Valid private Address address; // CORRECT: cascades into Address
```

### 5. Missing `spring-boot-starter-validation` dependency
Spring Boot 3 doesn't include it automatically — all annotations silently ignored.

### 6. Using groups with `@Valid`
```java
@Valid(OnCreate.class)      // COMPILE ERROR — @Valid takes no arguments
@Validated(OnCreate.class)  // CORRECT
```

### 7. `@NotBlank`/`@NotEmpty` on non-string types
```java
@NotBlank private Integer age;  // WRONG — strings only
@NotNull  private Integer age;  // CORRECT
```

---

## Common Interview Questions

### Q: What is Bean Validation and what is Hibernate Validator?

Bean Validation (JSR-380) is a **specification** defining annotations and interfaces for validation rules. Hibernate Validator is the **implementation** — the engine that runs the checks. In Spring Boot you add `spring-boot-starter-validation`, which bundles Hibernate Validator and wires it into Spring MVC.

---

### Q: What is the difference between `@NotNull`, `@NotEmpty`, and `@NotBlank`?

`@NotNull` only checks for null — empty/whitespace strings pass. `@NotEmpty` also rejects empty strings/collections, but a whitespace-only string still passes. `@NotBlank` is the strictest: null, `""`, and `"   "` all fail. Use `@NotBlank` for required strings, `@NotEmpty` for collections, `@NotNull` for numbers/dates/objects.

---

### Q: What is the difference between `@Valid` and `@Validated`?

`@Valid` (Jakarta standard) triggers validation and cascades into nested objects, but cannot specify groups. `@Validated` (Spring) enables method-level validation for `@PathVariable`/`@RequestParam` and supports groups, but doesn't cascade into nested fields. Use `@Valid` on `@RequestBody` and nested fields; use `@Validated` on the class for param validation or groups.

---

### Q: Why is `@Validated` needed on the class for `@PathVariable`/`@RequestParam`?

Constraints on simple parameters require Spring's **method-level validation via AOP proxy**. That proxy is only activated when `@Validated` is on the class. Without it, path/param constraints are silently ignored.

---

### Q: Which exceptions are thrown on validation failure?

`MethodArgumentNotValidException` — thrown when a `@Valid @RequestBody` DTO fails; read errors via `getBindingResult().getFieldErrors()`. `ConstraintViolationException` — thrown for `@PathVariable`/`@RequestParam` failures; read via `getConstraintViolations()`. They need separate `@ExceptionHandler` methods.

---

### Q: How do you return a clean validation error response?

Create a `@RestControllerAdvice` with `@ExceptionHandler` methods for both exception types. Extract field-level errors, map them into a structured `ApiError` DTO, and return with `400 Bad Request`.

---

### Q: How do you write a custom validation constraint?

Two parts: (1) an annotation with `@Constraint(validatedBy = ...)` plus required `message()`, `groups()`, and `payload()` members; (2) a class implementing `ConstraintValidator<Annotation, Type>` with logic in `isValid()`. Return `true` for null and let `@NotNull` handle it separately.

---

### Q: Does validation cascade into nested objects automatically?

No. You must put `@Valid` on the nested field to cascade. Without it, the outer object passes while a broken nested object slips through unchecked.

---

### Q: What are validation groups?

Groups let the same DTO apply different constraints in different scenarios (Create vs Update). Define empty marker interfaces, tag constraints with `groups = ...`, and select the group at the controller using `@Validated(Group.class)`. Groups only work with `@Validated`.

---

### Q: How do you validate that two fields match?

Use a **class-level constraint**: annotate with `@Target(TYPE)`, implement `ConstraintValidator` with the whole DTO as the target type, and compare fields inside `isValid()`. Apply the annotation on the class itself.

---

### Q: My validation annotations aren't firing. What's wrong?

Common causes: (1) missing `spring-boot-starter-validation`; (2) forgot `@Valid` on `@RequestBody`; (3) validating `@PathVariable`/`@RequestParam` without `@Validated` on the class; (4) forgot `@Valid` on a nested field; (5) self-invocation bypasses the AOP proxy on `@Validated` services.

---

### Q: Where should validation live — controller or service?

At the **controller boundary** — reject bad data as early as possible with a `400`. Keep the service layer focused on business logic. You can add `@Validated` on a service for defense-in-depth, but primary validation belongs at the edge.

---

## Quick Reference Cheat Sheet

```
THREE LAYERS:
  Jakarta Bean Validation         → specification (JSR-380), annotations
  Hibernate Validator             → implementation/engine
  spring-boot-starter-validation  → bundles engine + wires into Spring MVC

DEPENDENCY (Spring Boot 3 — NOT auto-included):
  spring-boot-starter-validation
  package: jakarta.validation.constraints

@NotNull vs @NotEmpty vs @NotBlank:
  null  ""    "  "
  @NotNull   FAIL  pass  pass   (any type)
  @NotEmpty  FAIL  FAIL  pass   (string/collection)
  @NotBlank  FAIL  FAIL  FAIL   (string only — strongest)
  → strings: @NotBlank | collections: @NotEmpty | numbers/dates: @NotNull

COMMON CONSTRAINTS:
  @Size(min,max)   → length/size range
  @Min / @Max      → integer bounds
  @DecimalMin/Max  → decimal bounds
  @Positive        → > 0
  @Email           → valid email format
  @Pattern(regexp) → matches regex
  @Past / @Future  → date in past / future

@Valid vs @Validated:
  @Valid     → Jakarta; cascades into nested; NO groups
  @Validated → Spring; enables param/method validation; SUPPORTS groups
  → @Valid on @RequestBody + nested fields
  → @Validated on the CLASS for @PathVariable/@RequestParam or groups

EXCEPTIONS:
  @Valid @RequestBody         → MethodArgumentNotValidException
  @PathVariable/@RequestParam → ConstraintViolationException (needs @Validated on class)

NESTED / COLLECTION:
  @Valid on nested field   → cascades into child object
  @Valid on List<X>        → cascades into every element
  @NotEmpty + @Valid       → list not empty AND each element valid

ERROR HANDLING:
  @RestControllerAdvice + @ExceptionHandler
    → handle MethodArgumentNotValidException (body failures)
    → handle ConstraintViolationException (param failures)
    → return structured ApiError DTO with 400

CUSTOM CONSTRAINT:
  1. @interface + @Constraint(validatedBy = XValidator.class)
     + message() / groups() / payload()
  2. class XValidator implements ConstraintValidator<X, Type>
     → isValid(): return true for null

GROUPS:
  interface OnCreate {} / OnUpdate {}
  @Null(groups=OnCreate) @NotNull(groups=OnUpdate) Long id;
  @Validated(OnCreate.class) @RequestBody ...
  → groups ONLY work with @Validated

CROSS-FIELD:
  @Constraint + @Target(TYPE)
  ConstraintValidator<Annotation, WholeDto>
  → compare fields inside isValid()

TOP PITFALLS:
  - forgot @Valid on @RequestBody → validation never runs
  - forgot @Validated on class → method/param validation ignored
  - @NotNull on a String allows "" and "   "
  - forgot @Valid on nested field → child not validated
  - missing starter-validation dependency → all annotations silent
```

---

*Last Updated: 2026-06-18*

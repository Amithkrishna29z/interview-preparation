# Spring / Jakarta Bean Validation Interview Questions & Study Guide

## Overview

Bean Validation is how Spring Boot applications check that incoming data is valid **before** it reaches your business logic. It's one of the most practical topics in backend interviews because every real API has to reject bad input (missing fields, invalid emails, out-of-range numbers). Expect questions on built-in constraints, `@Valid` vs `@Validated`, handling validation errors cleanly, and writing custom validators.

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

Bean Validation is a Java **standard** (specification) for declaring validation rules with annotations. You put annotations like `@NotNull` or `@Email` directly on your fields, and a validation engine checks them for you — you don't write `if (name == null) throw ...` everywhere.

**Think of it like a bouncer at a club door:**
- You (the developer) give the bouncer a **list of rules** ("must be 18+, must have ID, dress code required").
- The bouncer (the validator) checks every person (object) against those rules at the door (the boundary).
- Anyone who fails is turned away **before** they get inside (before your business logic runs).

There are three layers — just like JPA/Hibernate/Spring Data:

```
Jakarta Bean Validation (JSR-380 / Jakarta Validation)
  └── The SPECIFICATION — defines the rules and annotations
  └── Defines: @NotNull, @Size, @Email, the Validator interface, ConstraintValidator

Hibernate Validator
  └── The REFERENCE IMPLEMENTATION — the actual engine that runs the checks
  └── (Nothing to do with the Hibernate ORM — same company, different library)
  └── Adds extra annotations: @URL, @Range, @Length

spring-boot-starter-validation
  └── The Spring Boot starter that bundles Hibernate Validator
  └── Wires validation into Spring MVC (auto-validates @RequestBody when you add @Valid)
```

**Think of this like:** Jakarta Validation is the **rulebook** (a standard everyone agrees on), Hibernate Validator is the **referee** who actually enforces the rules, and `spring-boot-starter-validation` is the **stadium** that brings the referee onto the field for you.

> **Interview Tip**: "JSR-380" is just the formal name for **Bean Validation 2.0**. The package moved from `javax.validation` (old) to `jakarta.validation` (new) in Spring Boot 3+. If you see `javax.validation` in code, it's a Spring Boot 2 / older project.

---

## Why Validate at the Boundary?

**The golden rule: never trust client input.** Anything coming from outside your application (HTTP request body, query params, form data) could be malformed, missing, malicious, or just plain wrong. The "boundary" is the edge of your system — the controller layer where external data first arrives.

**Think of it like airport security:** you screen everyone **at the entrance**, not after they've already boarded the plane. If you let bad data flow deep into your service and database layers, the damage is harder to trace and undo.

Why validate early:

| Reason | Explanation |
|---|---|
| **Security** | Stops malformed/malicious input before it touches your logic or DB |
| **Data integrity** | Prevents garbage like `null` names or negative ages from being saved |
| **Clear errors** | You can return a clean `400 Bad Request` instead of a `500` crash deep in the code |
| **Less boilerplate** | One annotation replaces dozens of manual `if` checks |
| **Single source of truth** | Validation rules live on the DTO, visible to everyone reading the code |

> **Best Practice**: Validate at the controller (boundary) with annotations. Keep your service layer focused on business logic, not re-checking that `name != null`.

---

## Setup & Dependency

In Spring Boot 3, validation is **not** included by default — you must add the starter explicitly.

```xml
<!-- Maven: pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
    <!-- pulls in Hibernate Validator + Jakarta Validation API -->
</dependency>
```

```groovy
// Gradle: build.gradle
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

> **Common Mistake**: Forgetting this dependency. Your `@NotNull` / `@Email` annotations will compile fine but **silently do nothing** at runtime because there's no validation engine on the classpath. If validation "isn't firing," check this dependency first.

All constraint annotations come from the `jakarta.validation.constraints` package:

```java
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import jakarta.validation.constraints.Email;
// Hibernate Validator extras come from org.hibernate.validator.constraints
```

---

## Built-in Constraints

These are the constraints you'll use 95% of the time. Put them directly on DTO fields.

```java
public class UserRegistrationRequest {

    @NotBlank                                  // not null AND not just whitespace ("   " fails)
    @Size(min = 2, max = 50)                   // string length must be between 2 and 50 chars
    private String name;

    @NotBlank
    @Email                                     // must look like a valid email (a@b.com)
    private String email;

    @NotNull                                   // the field must be present (not null)
    @Min(18)                                   // numeric value must be >= 18
    @Max(120)                                  // numeric value must be <= 120
    private Integer age;

    @NotBlank
    @Pattern(regexp = "^[A-Za-z0-9_]{3,20}$")  // must match this regex (username rules)
    private String username;

    @NotNull
    @Positive                                  // must be > 0 (negative or zero fails)
    private BigDecimal salary;

    @Past                                      // date must be in the past (e.g. birth date)
    private LocalDate birthDate;

    @Future                                    // date must be in the future (e.g. appointment)
    private LocalDate appointmentDate;

    @Digits(integer = 6, fraction = 2)         // max 6 digits before decimal, 2 after (e.g. 123456.78)
    private BigDecimal accountBalance;

    @DecimalMin("0.0")                         // value must be >= 0.0 (works with decimals)
    @DecimalMax("100.0")                       // value must be <= 100.0
    private BigDecimal discountPercent;
}
```

### Constraint Reference Table

| Annotation | What it checks | Works On |
|---|---|---|
| `@NotNull` | Value is not null | Any type |
| `@NotEmpty` | Not null AND size/length > 0 | String, Collection, Map, Array |
| `@NotBlank` | Not null AND has non-whitespace text | String only |
| `@Size(min, max)` | Length/size within range | String, Collection, Map, Array |
| `@Min(n)` / `@Max(n)` | Number within integer range | Integer types |
| `@DecimalMin` / `@DecimalMax` | Number within range (decimals) | BigDecimal, String, numbers |
| `@Positive` / `@PositiveOrZero` | Number > 0 / >= 0 | Numbers |
| `@Negative` / `@NegativeOrZero` | Number < 0 / <= 0 | Numbers |
| `@Email` | Valid email format | String |
| `@Pattern(regexp)` | Matches a regular expression | String |
| `@Past` / `@PastOrPresent` | Date/time in the past | LocalDate, LocalDateTime, etc. |
| `@Future` / `@FutureOrPresent` | Date/time in the future | LocalDate, LocalDateTime, etc. |
| `@Digits(integer, fraction)` | Limits digit count before/after decimal | Numbers, String |
| `@AssertTrue` / `@AssertFalse` | boolean must be true / false | Boolean |

> **Interview Tip**: Every constraint accepts a custom `message`, e.g. `@Min(value = 18, message = "Must be at least 18")`. Without it, you get a default message like "must be greater than or equal to 18".

---

## @NotNull vs @NotBlank vs @NotEmpty

This is one of the **most common interview questions** — interviewers love it because juniors frequently mix them up. The differences are subtle but important.

| Annotation | Null fails? | Empty `""` fails? | Whitespace `"   "` fails? | Use On |
|---|---|---|---|---|
| `@NotNull` | Yes | No (`""` passes!) | No (`"   "` passes!) | Any type (numbers, objects, dates) |
| `@NotEmpty` | Yes | Yes | No (`"   "` passes!) | String, Collection, Map, Array |
| `@NotBlank` | Yes | Yes | Yes | **String only** |

```java
@NotNull   private String a;   // a = null → FAIL;  a = "" → PASS;  a = "  " → PASS
@NotEmpty  private String b;   // b = null → FAIL;  b = "" → FAIL;  b = "  " → PASS
@NotBlank  private String c;   // c = null → FAIL;  c = "" → FAIL;  c = "  " → FAIL (strongest)
```

**Think of it like checking if someone signed a form:**
- `@NotNull` = "Did they hand in the form at all?" (the form exists, even if blank)
- `@NotEmpty` = "Did they write *something*?" (at least one character, even a space)
- `@NotBlank` = "Did they write something *meaningful*?" (real text, not just spaces)

**Simple rules to remember:**
- For **Strings** → use `@NotBlank` almost always (it's the strictest, catches `null`, `""`, and `"   "`).
- For **Collections / Lists / Maps** → use `@NotEmpty` (`@NotBlank` doesn't work on collections).
- For **numbers, booleans, dates, and other objects** → use `@NotNull` (`@NotBlank`/`@NotEmpty` don't apply).

> **Common Mistake**: Putting `@NotEmpty` or `@NotBlank` on an `Integer` or `LocalDate` field. They only work on strings/collections — on a number you must use `@NotNull`. This often fails silently or throws a "constraint not applicable" error at startup.

---

## @Valid vs @Validated

These two look almost identical and constantly confuse juniors. Here's the clean breakdown.

| | `@Valid` | `@Validated` |
|---|---|---|
| Comes from | `jakarta.validation` (the standard) | `org.springframework.validation` (Spring's own) |
| Supports validation **groups**? | No | **Yes** (Create vs Update scenarios) |
| Triggers **nested** validation? | **Yes** (cascades into child objects) | No (not for nesting on fields) |
| Typical use | On `@RequestBody` method params and nested fields | On a **class** to enable method-level / param validation |

**Think of it like two different inspectors:**
- `@Valid` = a **thorough inspector** who also checks every room inside the house (nested objects), but only knows one set of rules.
- `@Validated` = a **specialized inspector** who can switch rulebooks depending on the situation (groups), but doesn't automatically go into nested rooms.

```java
// @Valid — on a method parameter: validate the request body and cascade into nested objects
@PostMapping("/users")
public User create(@Valid @RequestBody UserRequest request) { ... }

// @Validated — on the CLASS: enables validation of @PathVariable / @RequestParam,
// and lets you pick a validation group
@Validated
@RestController
public class UserController {

    @PostMapping("/users")
    public User create(@Validated(OnCreate.class) @RequestBody UserRequest req) { ... }
    //                  ^ uses the "OnCreate" validation group
}
```

**Practical takeaway for interviews:**
- Use `@Valid` on `@RequestBody` params and on nested object fields (it cascades).
- Use `@Validated` on the **class** when you need to validate `@PathVariable`/`@RequestParam`, or when you need **validation groups**.

---

## Validating @RequestBody DTOs

The most common use case: a client POSTs JSON, you map it to a DTO, and you want Spring to reject it automatically if any field is invalid.

```java
// 1. The DTO with constraints
public class CreateProductRequest {

    @NotBlank(message = "Product name is required")
    private String name;

    @NotNull
    @Positive(message = "Price must be greater than zero")
    private BigDecimal price;

    @Min(value = 0, message = "Stock cannot be negative")
    private int stock;

    // getters and setters...
}
```

```java
// 2. The controller — note @Valid before @RequestBody
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @PostMapping
    public ResponseEntity<Product> create(@Valid @RequestBody CreateProductRequest request) {
        // ^ @Valid tells Spring: validate this body BEFORE entering the method body
        // If validation fails, Spring throws MethodArgumentNotValidException
        // and this method body NEVER runs
        Product saved = productService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(saved);
    }
}
```

**What happens on invalid input:**
1. Client sends `{ "name": "", "price": -5 }`.
2. Spring binds the JSON to `CreateProductRequest`.
3. Spring runs the validator (because of `@Valid`).
4. `name` fails `@NotBlank`, `price` fails `@Positive`.
5. Spring throws `MethodArgumentNotValidException` → returns `400 Bad Request`.
6. Your controller method body **never executes**.

> **Interview Tip**: Without `@Valid`, Spring will happily pass an invalid DTO straight into your method — the annotations on the DTO do nothing on their own. `@Valid` is the trigger.

---

## Nested & Collection Validation

By default, validation does **not** automatically go into nested objects. You must put `@Valid` on the nested field to make it cascade.

```java
public class OrderRequest {

    @NotBlank
    private String orderNumber;

    @NotNull
    @Valid                          // <-- REQUIRED: cascade validation into the Address object
    private Address shippingAddress; // without @Valid here, Address's constraints are IGNORED

    @NotEmpty                        // the list itself must not be null/empty
    @Valid                          // <-- cascade validation into EACH OrderItem in the list
    private List<OrderItem> items;
}

public class Address {
    @NotBlank private String street;   // only checked if OrderRequest.shippingAddress has @Valid
    @NotBlank private String city;
    @Pattern(regexp = "\\d{5}") private String zipCode; // 5-digit zip
}

public class OrderItem {
    @NotBlank private String productId;
    @Min(1) private int quantity;      // each item must have quantity >= 1
}
```

**Think of it like luggage inspection at the airport:**
- `@Valid` on the outer object checks the **suitcase**.
- Without `@Valid` on a nested field, the inspector **does not open the bags inside** the suitcase.
- Adding `@Valid` to the nested field tells them: "open this bag and check its contents too."

**For collections:** `@Valid` on a `List<OrderItem>` cascades into **every element**. Combine it with `@NotEmpty` if the list itself must not be empty.

> **Common Mistake**: Forgetting `@Valid` on nested fields. The outer DTO passes validation, but the broken `Address` inside it slips through completely unchecked.

---

## Validating @PathVariable & @RequestParam

To validate individual method parameters (not a whole DTO), you put constraints **directly on the parameters** AND annotate the **class** with `@Validated`.

```java
@Validated                              // <-- REQUIRED on the class for param validation to work
@RestController
@RequestMapping("/api/users")
public class UserController {

    // Validate a path variable: id must be positive
    @GetMapping("/{id}")
    public User getById(@PathVariable @Min(1) Long id) {
        // GET /api/users/0  → fails @Min(1) → throws ConstraintViolationException
        return userService.findById(id);
    }

    // Validate query params
    @GetMapping
    public List<User> search(
            @RequestParam @NotBlank String keyword,        // ?keyword= must not be blank
            @RequestParam @Min(1) @Max(100) int pageSize) { // 1 <= pageSize <= 100
        return userService.search(keyword, pageSize);
    }
}
```

**Why is `@Validated` required here?** Constraints on `@RequestBody` are handled by Spring MVC's argument resolver. But constraints on **simple parameters** (`@PathVariable`, `@RequestParam`) require Spring's **method-level validation**, which is only switched on by `@Validated` on the class.

> **Key difference**: Invalid `@RequestBody` throws `MethodArgumentNotValidException`. Invalid `@PathVariable`/`@RequestParam` throws `ConstraintViolationException`. You handle these with **two different** exception handlers (covered next).

---

## Handling Validation Errors

By default, validation failures return an ugly Spring error blob. In a real API, you want a **clean, structured JSON error** so the frontend knows exactly which fields failed. You centralize this with `@RestControllerAdvice`.

### The two exceptions you must handle

| Exception | Thrown when |
|---|---|
| `MethodArgumentNotValidException` | `@Valid @RequestBody` DTO fails |
| `ConstraintViolationException` | `@PathVariable` / `@RequestParam` constraint fails (with `@Validated` on class) |

### Step 1: A structured error response DTO

```java
public class ApiError {
    private int status;                       // HTTP status code, e.g. 400
    private String message;                   // overall message, e.g. "Validation failed"
    private LocalDateTime timestamp;          // when the error happened
    private List<FieldError> errors;          // per-field details

    // getters, setters, constructor...

    public static class FieldError {
        private String field;                 // which field failed, e.g. "email"
        private String message;               // why it failed, e.g. "must be a valid email"
        // getters, setters, constructor...
    }
}
```

### Step 2: The global exception handler

```java
@RestControllerAdvice                          // applies to ALL controllers, returns JSON bodies
public class GlobalExceptionHandler {

    // Handles invalid @RequestBody DTOs
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)    // return 400
    public ApiError handleBodyValidation(MethodArgumentNotValidException ex) {

        // Each failed field constraint becomes a FieldError in our response
        List<ApiError.FieldError> fieldErrors = ex.getBindingResult()
                .getFieldErrors()              // all the fields that failed
                .stream()
                .map(err -> new ApiError.FieldError(
                        err.getField(),                 // e.g. "email"
                        err.getDefaultMessage()))       // e.g. "must be a valid email"
                .toList();

        ApiError error = new ApiError();
        error.setStatus(HttpStatus.BAD_REQUEST.value());
        error.setMessage("Validation failed");
        error.setTimestamp(LocalDateTime.now());
        error.setErrors(fieldErrors);
        return error;
    }

    // Handles invalid @PathVariable / @RequestParam
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError handleParamValidation(ConstraintViolationException ex) {

        List<ApiError.FieldError> fieldErrors = ex.getConstraintViolations() // all violations
                .stream()
                .map(v -> new ApiError.FieldError(
                        v.getPropertyPath().toString(), // e.g. "getById.id"
                        v.getMessage()))                // e.g. "must be greater than or equal to 1"
                .toList();

        ApiError error = new ApiError();
        error.setStatus(HttpStatus.BAD_REQUEST.value());
        error.setMessage("Validation failed");
        error.setTimestamp(LocalDateTime.now());
        error.setErrors(fieldErrors);
        return error;
    }
}
```

### The clean JSON the client now receives

```json
{
  "status": 400,
  "message": "Validation failed",
  "timestamp": "2026-06-11T10:15:30",
  "errors": [
    { "field": "email", "message": "must be a valid email" },
    { "field": "age",   "message": "must be greater than or equal to 18" }
  ]
}
```

**Think of it like a rejected job application with feedback:** instead of just stamping "REJECTED" (a generic 500 error), you hand back the form with **each problem circled** ("phone number missing," "date in wrong format") so the applicant can fix it and reapply.

> **Interview Tip**: `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`. It catches exceptions across **all** controllers in one place, so you don't repeat error-handling logic in every method.

---

## Custom Constraint Annotation

When built-in constraints aren't enough (e.g., a phone number format, a strong password rule), you create your own annotation. This is a **very common** senior-leaning interview question.

A custom constraint needs **two parts**:
1. The **annotation** (the label you put on fields).
2. The **validator** (the class that contains the actual checking logic).

### Worked example: @StrongPassword

```java
// PART 1: The annotation
@Documented
@Constraint(validatedBy = StrongPasswordValidator.class)  // links to the logic class
@Target({ FIELD })                          // can only be placed on fields
@Retention(RUNTIME)                         // must be available at runtime for the validator to read
public @interface StrongPassword {

    // The default error message if validation fails
    String message() default "Password must be 8+ chars with upper, lower, digit and symbol";

    // These two are REQUIRED boilerplate for every constraint annotation:
    Class<?>[] groups() default {};                       // for validation groups
    Class<? extends Payload>[] payload() default {};      // for metadata (rarely used)
}
```

```java
// PART 2: The validator (the actual checking logic)
public class StrongPasswordValidator
        implements ConstraintValidator<StrongPassword, String> {
    //          ConstraintValidator<TheAnnotation, TheTypeBeingValidated>

    private static final Pattern PATTERN = Pattern.compile(
        "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[^A-Za-z0-9]).{8,}$"
        // at least: 1 lowercase, 1 uppercase, 1 digit, 1 symbol, min length 8
    );

    @Override
    public boolean isValid(String password, ConstraintValidatorContext context) {
        // Return true = valid, false = invalid
        // Let @NotNull handle nulls separately — a null here should PASS this check
        if (password == null) {
            return true;   // don't fail on null; that's @NotNull's job
        }
        return PATTERN.matcher(password).matches();
    }
}
```

```java
// USAGE: now use it like any built-in constraint
public class SignupRequest {
    @NotBlank
    @StrongPassword              // your custom rule, reads as cleanly as @NotBlank
    private String password;
}
```

**Think of it like adding a custom rule to the bouncer's list:** the built-in rules cover age and dress code, but your club has a special rule ("members only"). You write that rule down (`@StrongPassword`) and teach the bouncer how to check it (`StrongPasswordValidator`).

> **Interview Tip**: A common best practice is to return `true` for `null` inside the validator and let a separate `@NotNull`/`@NotBlank` handle the null case. This keeps each constraint focused on one job (single responsibility).

---

## Validation Groups (Create vs Update)

Sometimes the **same DTO** needs different rules in different situations. Classic example: when **creating** a user the `id` must be null, but when **updating** the `id` is required. Validation groups let you toggle which constraints apply.

```java
// 1. Define marker interfaces (empty — just labels)
public interface OnCreate {}
public interface OnUpdate {}

// 2. Assign each constraint to a group
public class UserRequest {

    @Null(groups = OnCreate.class)      // on CREATE, id must be null (server assigns it)
    @NotNull(groups = OnUpdate.class)   // on UPDATE, id is required
    private Long id;

    @NotBlank(groups = {OnCreate.class, OnUpdate.class})  // name required in both cases
    private String name;
}

// 3. Pick the group at the controller using @Validated (NOT @Valid — groups need @Validated)
@RestController
public class UserController {

    @PostMapping("/users")
    public User create(@Validated(OnCreate.class) @RequestBody UserRequest req) { ... }
    //                  ^ only OnCreate-group constraints run

    @PutMapping("/users/{id}")
    public User update(@Validated(OnUpdate.class) @RequestBody UserRequest req) { ... }
    //                  ^ only OnUpdate-group constraints run
}
```

**Think of it like one exam with two answer keys:** the same test paper (DTO) is graded differently depending on whether the student is a beginner (Create) or advanced (Update).

> **Key point**: Groups **only work with `@Validated`**, not `@Valid`. `@Valid` has no way to specify a group.

---

## Cross-Field Validation

Some rules involve **two or more fields together** (e.g., "password and confirmPassword must match," or "endDate must be after startDate"). A single-field constraint can't see other fields, so you use a **class-level constraint**.

```java
// 1. A class-level annotation (note @Target TYPE, not FIELD)
@Constraint(validatedBy = PasswordMatchValidator.class)
@Target({ TYPE })                          // placed on the CLASS, not a field
@Retention(RUNTIME)
public @interface PasswordMatch {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// 2. The validator receives the WHOLE object, so it can compare fields
public class PasswordMatchValidator
        implements ConstraintValidator<PasswordMatch, SignupRequest> {

    @Override
    public boolean isValid(SignupRequest req, ConstraintValidatorContext ctx) {
        if (req.getPassword() == null) return true;        // let @NotNull handle nulls
        return req.getPassword().equals(req.getConfirmPassword()); // compare two fields
    }
}

// 3. Apply it on the class
@PasswordMatch                              // class-level: validates across fields
public class SignupRequest {
    @NotBlank private String password;
    @NotBlank private String confirmPassword;
}
```

**Think of it like checking that two signatures on a contract match** — you can't verify a match by looking at one signature alone; you need to see both at once.

---

## Common Mistakes & Pitfalls

These show up constantly in interviews and real code reviews.

### 1. Forgetting `@Valid` on the @RequestBody

```java
// WRONG — annotations on the DTO do nothing without @Valid
@PostMapping("/users")
public User create(@RequestBody UserRequest req) { ... }  // validation NEVER runs

// RIGHT
@PostMapping("/users")
public User create(@Valid @RequestBody UserRequest req) { ... }
```

### 2. Validation not firing on service methods

Validation on `@RequestBody` works because Spring MVC triggers it. But if you want validation on a **service method** parameter, you must annotate the **service class** with `@Validated`.

```java
@Service
@Validated                                  // <-- REQUIRED for method-level validation here
public class UserService {
    public void register(@Valid UserRequest req) {  // now this @Valid actually fires
        ...
    }
}
// Without @Validated on the class, @Valid on the parameter is ignored.
```

### 3. Confusing `@NotNull` with `@NotBlank`

```java
@NotNull private String name;   // BUG: name = "" or "   " PASSES — probably not what you want
@NotBlank private String name;  // CORRECT for required strings
```

### 4. Forgetting `@Valid` on nested objects

```java
@NotNull
private Address address;          // BUG: Address's own @NotBlank fields are NOT checked

@NotNull @Valid
private Address address;          // CORRECT: cascades into Address
```

### 5. Missing the `spring-boot-starter-validation` dependency

In Spring Boot 3 this isn't auto-included. No dependency = annotations silently ignored.

### 6. Using groups with `@Valid` instead of `@Validated`

```java
@Valid(OnCreate.class)  // COMPILE ERROR — @Valid takes no arguments
@Validated(OnCreate.class)  // CORRECT — groups require @Validated
```

### 7. Putting `@NotEmpty`/`@NotBlank` on non-string types

```java
@NotBlank private Integer age;  // WRONG — @NotBlank is for strings only
@NotNull private Integer age;   // CORRECT for a required number
```

---

## Common Interview Questions

### Q: What is Bean Validation and what is Hibernate Validator?

Bean Validation (JSR-380 / Jakarta Validation) is a **specification** — a set of annotations and interfaces for declaring validation rules. **Hibernate Validator** is the most popular **implementation** (the engine that actually runs the checks). Note it has nothing to do with the Hibernate ORM beyond sharing a vendor name. In Spring Boot you add `spring-boot-starter-validation`, which bundles Hibernate Validator and wires it into Spring MVC.

---

### Q: What is the difference between `@NotNull`, `@NotEmpty`, and `@NotBlank`?

- `@NotNull` — only checks the value isn't null. An empty or whitespace string passes. Works on any type.
- `@NotEmpty` — null fails AND size/length must be > 0. Works on strings and collections. Whitespace string still passes.
- `@NotBlank` — null, empty, **and** whitespace-only strings all fail. **Strings only** — it's the strictest.

Rule of thumb: `@NotBlank` for required strings, `@NotEmpty` for collections, `@NotNull` for everything else.

---

### Q: What is the difference between `@Valid` and `@Validated`?

- `@Valid` is from Jakarta (the standard). It triggers validation and **cascades into nested objects**, but cannot specify validation groups.
- `@Validated` is from Spring. It enables **method-level validation** (needed for `@PathVariable`/`@RequestParam`) and supports **validation groups**, but doesn't cascade into nested fields the way `@Valid` does.

Use `@Valid` on `@RequestBody` and nested fields. Use `@Validated` on the class for param validation or when you need groups.

---

### Q: Why is `@Validated` needed on the class to validate `@PathVariable` / `@RequestParam`?

Constraints on simple parameters require Spring's **method-level validation**, which is implemented via an AOP proxy. That proxy is only activated when the class is annotated with `@Validated`. Without it, the constraints on path variables and request params are silently ignored.

---

### Q: Which exceptions are thrown on validation failure, and how do they differ?

- `MethodArgumentNotValidException` — thrown when a `@Valid @RequestBody` DTO fails. You read failures via `getBindingResult().getFieldErrors()`.
- `ConstraintViolationException` — thrown when a `@PathVariable` or `@RequestParam` constraint fails (with `@Validated` on the class). You read failures via `getConstraintViolations()`.

They need two separate `@ExceptionHandler` methods because they expose errors through different APIs.

---

### Q: How do you return a clean, structured validation error response?

Create a `@RestControllerAdvice` class with `@ExceptionHandler` methods for `MethodArgumentNotValidException` and `ConstraintViolationException`. In each handler, extract the field-level errors, map them into a structured `ApiError` DTO (status, message, timestamp, list of field errors), and return it with a `400 Bad Request`. This centralizes error handling for all controllers.

---

### Q: How do you write a custom validation constraint?

Two parts: (1) an annotation marked with `@Constraint(validatedBy = ...)` plus the required `message()`, `groups()`, and `payload()` members; and (2) a class implementing `ConstraintValidator<TheAnnotation, TheType>` with the checking logic in `isValid()`. Best practice: return `true` for null and let `@NotNull` handle the null case separately.

---

### Q: Does validation automatically cascade into nested objects?

No. By default, validating an outer object does **not** validate its nested objects. You must put `@Valid` on the nested field (or on the collection) to cascade validation into it. Forgetting this is a common bug — the outer object passes while a broken nested object slips through.

---

### Q: What are validation groups and when do you use them?

Validation groups let the **same DTO** apply different constraints in different scenarios — the classic case is Create vs Update (e.g., `id` must be null on create but required on update). You define marker interfaces, tag constraints with `groups = ...`, and select the group at the controller using `@Validated(OnCreate.class)`. Groups only work with `@Validated`, not `@Valid`.

---

### Q: How do you validate that two fields match (e.g., password confirmation)?

A single-field constraint can't see other fields, so you use a **class-level constraint**: annotate the constraint with `@Target(TYPE)`, implement a `ConstraintValidator` whose target type is the whole DTO, and compare the fields inside `isValid()`. Apply the annotation on the class itself.

---

### Q: My validation annotations aren't firing. What could be wrong?

Common causes: (1) missing `spring-boot-starter-validation` dependency; (2) forgot `@Valid` on the `@RequestBody`; (3) trying to validate `@PathVariable`/`@RequestParam` without `@Validated` on the class; (4) calling a `@Validated` service method via self-invocation (bypasses the proxy); (5) forgot `@Valid` on a nested field so cascade didn't happen.

---

### Q: Where should validation live — controller or service?

At the **boundary** (controller), because you should never trust client input and you want to reject bad data as early as possible with a clean `400`. Keep the service layer focused on business logic. You *can* add `@Validated` on a service for defense-in-depth, but the primary validation belongs at the controller edge.

---

### Q: What's the difference between `@Min`/`@Max` and `@DecimalMin`/`@DecimalMax`?

`@Min` and `@Max` take a `long` and are for whole-number bounds on integer types. `@DecimalMin`/`@DecimalMax` take a `String` value and work with fractional/decimal numbers (`BigDecimal`, etc.), and let you control whether the boundary is inclusive. Use the Decimal versions for money and percentages.

---

## Quick Reference Cheat Sheet

```
THE THREE LAYERS:
  Jakarta Bean Validation  → the specification (JSR-380), the annotations
  Hibernate Validator      → the implementation/engine that runs the checks
  spring-boot-starter-validation → bundles the engine + wires it into Spring MVC

DEPENDENCY (Spring Boot 3 — NOT auto-included):
  spring-boot-starter-validation
  package: jakarta.validation.constraints

@NotNull  vs  @NotEmpty  vs  @NotBlank:
  null  ""    "  "
  ----  --    ----
  @NotNull   FAIL  pass  pass   (any type)
  @NotEmpty  FAIL  FAIL  pass   (string/collection)
  @NotBlank  FAIL  FAIL  FAIL   (string only — strongest)
  → strings: @NotBlank | collections: @NotEmpty | numbers/dates: @NotNull

COMMON CONSTRAINTS:
  @Size(min,max)   → length/size range (string, collection)
  @Min / @Max      → integer bounds
  @DecimalMin/Max  → decimal bounds (money, percentages)
  @Positive        → > 0
  @Email           → valid email format
  @Pattern(regexp) → matches a regex
  @Past / @Future  → date in past / future
  @Digits(int,frac)→ limit digits before/after decimal

@Valid  vs  @Validated:
  @Valid     → Jakarta; cascades into nested objects; NO groups
  @Validated → Spring; enables param + method validation; SUPPORTS groups
  → @Valid on @RequestBody + nested fields
  → @Validated on the CLASS for @PathVariable/@RequestParam or groups

VALIDATION TRIGGERS:
  @Valid @RequestBody DTO          → fails with MethodArgumentNotValidException
  @PathVariable/@RequestParam      → needs @Validated on class
                                   → fails with ConstraintViolationException

NESTED / COLLECTION:
  @Valid on a nested field         → cascades into the child object
  @Valid on List<X>                → cascades into every element
  @NotEmpty + @Valid on a list     → list not empty AND each element valid

ERROR HANDLING:
  @RestControllerAdvice + @ExceptionHandler
    → handle MethodArgumentNotValidException (body)
    → handle ConstraintViolationException (params)
    → return structured ApiError DTO with 400

CUSTOM CONSTRAINT:
  1. @interface + @Constraint(validatedBy = XValidator.class)
     + message() / groups() / payload()
  2. class XValidator implements ConstraintValidator<X, Type>
     → isValid() holds the logic; return true for null

GROUPS (Create vs Update):
  interface OnCreate {} / OnUpdate {}
  @Null(groups=OnCreate) @NotNull(groups=OnUpdate) Long id;
  @Validated(OnCreate.class) @RequestBody ...
  → groups ONLY work with @Validated

CROSS-FIELD (class-level):
  @Constraint + @Target(TYPE)
  ConstraintValidator<Annotation, WholeDto>
  → compare fields inside isValid()

TOP PITFALLS:
  - forgot @Valid on @RequestBody → validation never runs
  - forgot @Validated on service/class → method validation ignored
  - @NotNull on a String allows "" and "   "
  - forgot @Valid on nested field → child not validated
  - missing starter-validation dependency → all annotations silent
```

---

*Last Updated: 2026-06-11*

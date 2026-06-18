# Spring Boot Practical Features Guide

## Overview

This guide covers the practical Spring Boot features a junior developer implements day-to-day and is most likely to be asked to code live in an interview. These aren't theory questions — interviewers write a blank controller and say "add pagination" or "make this endpoint upload a file."

Covers: Pagination & Sorting, File Upload/Download, Scheduled Tasks, Async Methods, Caching, and Sending Email. For global exception handling with `@RestControllerAdvice`, see the **Java Exception Handling Guide**.

---

## Table of Contents

1. [Pagination & Sorting](#1-pagination--sorting)
2. [File Upload & Download](#2-file-upload--download)
3. [Scheduled Tasks](#3-scheduled-tasks)
4. [Asynchronous Methods](#4-asynchronous-methods)
5. [Caching](#5-caching)
6. [Sending Email](#6-sending-email)
7. [Common Interview Questions](#common-interview-questions)
8. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## 1. Pagination & Sorting

**Most asked live-coding feature.** Instead of returning all rows from a database table, you return one page at a time. Spring Data does the heavy lifting — you just pass a `Pageable` through.

### Repository

```java
// Extend JpaRepository — the Page<T> method is all you need
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Spring Data generates the SQL with LIMIT/OFFSET automatically
    Page<Product> findByCategory(String category, Pageable pageable);
}
```

### Service — mapping Entity to DTO

```java
// NEVER return Page<Product> (entity) directly to the client.
// Use .map() to convert each element inside the Page to a DTO.
public Page<ProductDTO> getProducts(String category, Pageable pageable) {
    return productRepository
            .findByCategory(category, pageable)
            .map(product -> new ProductDTO(product.getId(), product.getName(), product.getPrice()));
}
```

### Controller

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    // @PageableDefault sets fallback values when the client sends no params
    @GetMapping
    public Page<ProductDTO> list(
            @RequestParam String category,
            @PageableDefault(size = 20, sort = "name", direction = Sort.Direction.ASC)
            Pageable pageable) {

        return productService.getProducts(category, pageable);
    }
}
```

### Building a Pageable manually (e.g., in service tests)

```java
// PageRequest.of(pageNumber, pageSize, Sort)
Pageable pageable = PageRequest.of(0, 10, Sort.by("price").descending());

// Multi-column sort
Pageable pageable = PageRequest.of(0, 10,
        Sort.by(Sort.Order.desc("price"), Sort.Order.asc("name")));
```

### Query params the client sends

| Param | Example | Meaning |
|-------|---------|---------|
| `page` | `?page=0` | Zero-based page number |
| `size` | `?size=20` | Items per page |
| `sort` | `?sort=price,desc` | Sort field and direction |

### Response shape Spring returns

```json
{
  "content": [ { "id": 1, "name": "Widget", "price": 9.99 } ],
  "pageable": { "pageNumber": 0, "pageSize": 20 },
  "totalElements": 150,
  "totalPages": 8,
  "last": false,
  "first": true
}
```

### Common mistakes

- Returning `Page<Entity>` instead of `Page<DTO>` — exposes internal fields and can trigger lazy-load exceptions during serialization.
- Forgetting `@PageableDefault` — if the client omits `size`, Spring defaults to 20, which may be acceptable, but explicit is safer.
- Using `List<T>` in the repository instead of `Page<T>` — you lose `totalElements`/`totalPages` metadata.

---

## 2. File Upload & Download

### Enable multipart in `application.properties`

```properties
spring.servlet.multipart.enabled=true
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=10MB
```

### Upload endpoint

```java
@RestController
@RequestMapping("/api/files")
public class FileController {

    private static final String UPLOAD_DIR = "/uploads/";

    // consumes tells Spring this is a multipart/form-data request
    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file) {

        // 1. Validate — never trust the client
        if (file.isEmpty()) {
            return ResponseEntity.badRequest().body("File is empty");
        }

        String originalName = StringUtils.cleanPath(file.getOriginalFilename());
        String extension = FilenameUtils.getExtension(originalName);

        // 2. Whitelist allowed types
        List<String> allowed = List.of("jpg", "jpeg", "png", "pdf");
        if (!allowed.contains(extension.toLowerCase())) {
            return ResponseEntity.badRequest().body("File type not allowed");
        }

        // 3. Save with a unique name to avoid collisions
        String savedName = UUID.randomUUID() + "." + extension;
        Path target = Paths.get(UPLOAD_DIR + savedName);
        Files.copy(file.getInputStream(), target, StandardCopyOption.REPLACE_EXISTING);

        return ResponseEntity.ok("Uploaded: " + savedName);
    }
```

### Download endpoint

```java
    @GetMapping("/download/{filename}")
    public ResponseEntity<Resource> download(@PathVariable String filename) throws IOException {

        Path filePath = Paths.get(UPLOAD_DIR).resolve(filename).normalize();
        Resource resource = new UrlResource(filePath.toUri());

        if (!resource.exists() || !resource.isReadable()) {
            return ResponseEntity.notFound().build();
        }

        // Content-Disposition: attachment triggers browser download dialog
        String contentType = Files.probeContentType(filePath);
        if (contentType == null) contentType = "application/octet-stream";

        return ResponseEntity.ok()
                .contentType(MediaType.parseMediaType(contentType))
                .header(HttpHeaders.CONTENT_DISPOSITION,
                        "attachment; filename=\"" + resource.getFilename() + "\"")
                .body(resource);
    }
}
```

### Common mistakes

- No file size limit — an attacker uploads a 2 GB file and crashes the server.
- Saving with the original filename from the client — path traversal attack: `../../etc/passwd`.
- Not setting `Content-Disposition` — the browser renders the file inline instead of downloading it.

---

## 3. Scheduled Tasks

Think of `@Scheduled` as a built-in cron job inside your application — no separate scheduler process needed.

### Enable in your main class or config

```java
@SpringBootApplication
@EnableScheduling          // required — without this, @Scheduled is ignored
public class MyApp { ... }
```

### Three scheduling modes

```java
@Component
public class ReportScheduler {

    // fixedRate: run every N ms regardless of how long the previous run took
    @Scheduled(fixedRate = 60_000)
    public void syncInventory() {
        // runs every 60 seconds
    }

    // fixedDelay: wait N ms AFTER the previous run completes
    @Scheduled(fixedDelay = 30_000)
    public void cleanTempFiles() {
        // next run starts 30s after this one finishes
    }

    // cron: full cron expression — "second minute hour day month weekday"
    // Spring uses 6-field cron: 0 0 8 * * MON-FRI = 08:00 every weekday
    @Scheduled(cron = "0 0 8 * * MON-FRI")
    public void sendDailyReport() {
        // runs at 8:00 AM every weekday
    }
}
```

### Cron field order (Spring)

```
┌─ second (0-59)
│  ┌─ minute (0-59)
│  │  ┌─ hour (0-23)
│  │  │  ┌─ day of month (1-31)
│  │  │  │  ┌─ month (1-12 or JAN-DEC)
│  │  │  │  │  ┌─ day of week (0-7 or MON-SUN)
0  0  8  *  *  MON-FRI
```

### Common mistakes

- `@EnableScheduling` missing — methods are declared but never fire; no error is thrown.
- Using `fixedRate` for long-running jobs — concurrent executions pile up if the job takes longer than the interval. Use `fixedDelay` or `@Async` instead.
- No `@Component` on the class — Spring can't pick up the bean.

---

## 4. Asynchronous Methods

`@Async` moves a method off the request thread into a background thread pool, useful for fire-and-forget work (emails, notifications, audit logs).

### Enable in config

```java
@SpringBootApplication
@EnableAsync               // required
public class MyApp { ... }
```

### Async method returning a future

```java
@Service
public class NotificationService {

    // Void variant — fire and forget
    @Async
    public void sendPushNotification(Long userId) {
        // runs on a thread pool thread, not the HTTP thread
        pushClient.send(userId);
    }

    // CompletableFuture variant — caller can wait or chain
    @Async
    public CompletableFuture<String> generateReport(Long orderId) {
        String result = reportEngine.build(orderId); // expensive operation
        return CompletableFuture.completedFuture(result);
    }
}
```

### Custom thread pool (recommended for production)

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

### Common mistakes — the self-invocation trap

```java
@Service
public class OrderService {

    public void placeOrder(Order order) {
        saveOrder(order);
        sendConfirmation(order); // ← SELF-INVOCATION: @Async is IGNORED here
    }

    @Async
    public void sendConfirmation(Order order) { ... }
}
```

Spring AOP works through a proxy. Calling an `@Async` (or `@Transactional`) method from *within the same bean* bypasses the proxy entirely — the annotation has no effect. Fix: inject the bean into itself (`@Autowired ApplicationContext`) or move the async method to a separate bean.

### Common mistakes

- No `@EnableAsync` — methods run synchronously with no warning.
- Self-invocation — the `@Async` is silently skipped (same issue applies to `@Transactional`, `@Cacheable`).
- No thread pool config — Spring uses `SimpleAsyncTaskExecutor` by default, which creates a new thread per call (no pooling). Always define a `ThreadPoolTaskExecutor` bean.

---

## 5. Caching

Cache stores the result of an expensive call (DB query, external API). On repeat calls with the same arguments, the cached value is returned instantly.

### Enable caching

```java
@SpringBootApplication
@EnableCaching             // required
public class MyApp { ... }
```

### Core annotations

```java
@Service
public class ProductService {

    // @Cacheable — return cached result; execute method only on cache miss
    // "products" is the cache name; key defaults to method args
    @Cacheable(value = "products", key = "#id")
    public ProductDTO getProduct(Long id) {
        return productRepository.findById(id)
                .map(this::toDTO)
                .orElseThrow(() -> new ResourceNotFoundException("Product not found"));
    }

    // @CachePut — always runs the method AND updates the cache
    // Use this on update operations
    @CachePut(value = "products", key = "#dto.id")
    public ProductDTO updateProduct(ProductDTO dto) {
        Product saved = productRepository.save(toEntity(dto));
        return toDTO(saved);
    }

    // @CacheEvict — removes an entry (or all entries) from the cache
    // Use this on delete operations
    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }

    // allEntries = true — clears the entire "products" cache
    @CacheEvict(value = "products", allEntries = true)
    public void bulkImport(List<ProductDTO> products) { ... }
}
```

### Cache providers

| Provider | How to enable | Best for |
|----------|--------------|---------|
| Default (ConcurrentHashMap) | Nothing extra | Dev / single instance |
| Caffeine | `spring-boot-starter-cache` + caffeine dep + config | High-performance in-process |
| Redis | `spring-boot-starter-data-redis` + config | Multi-instance / distributed |

### When to cache

- Data that is read far more than it is written.
- Expensive DB queries or external API calls.
- Do NOT cache data that must be real-time accurate (stock prices, live inventory counts).

### Common mistakes

- Self-invocation — `@Cacheable` on a method called from within the same bean is bypassed (same proxy issue as `@Async`).
- Caching mutable entities — if the entity changes, the cache entry is stale until evicted.
- No eviction strategy — cache grows unbounded in memory. Always configure TTL when using Caffeine/Redis.

---

## 6. Sending Email

Spring wraps JavaMail into `JavaMailSender` so you don't deal with the raw SMTP protocol.

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

### SMTP configuration (`application.properties`)

```properties
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=your@gmail.com
spring.mail.password=${MAIL_PASSWORD}    # inject from env, never hardcode
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

### Email service

```java
@Service
@RequiredArgsConstructor
public class EmailService {

    private final JavaMailSender mailSender;

    public void sendWelcomeEmail(String to, String name) throws MessagingException {
        MimeMessage message = mailSender.createMimeMessage();

        // MimeMessageHelper handles multipart (HTML body, attachments)
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
        helper.setTo(to);
        helper.setSubject("Welcome to MyApp!");
        helper.setText("<h1>Hi " + name + ", welcome!</h1>", true); // true = HTML

        mailSender.send(message);
    }
}
```

### Common mistakes

- Hardcoding SMTP credentials in source code — use environment variables.
- Sending email synchronously on the request thread — use `@Async` so the HTTP response isn't delayed.
- Not handling `MessagingException` — let `@RestControllerAdvice` catch and map it to a clean error response.

---

## Common Interview Questions

**Q: How does Spring pagination work under the hood?**
Spring Data translates `Pageable` into SQL `LIMIT` and `OFFSET` clauses at the JPA/JDBC level. The `Page<T>` response includes both the content slice and metadata (`totalElements`, `totalPages`) so the frontend can render navigation. One extra `COUNT(*)` query is issued to calculate the total.

**Q: Why shouldn't you return `Page<Entity>` from a controller?**
Entities may expose sensitive fields, trigger lazy-loading during Jackson serialization (causing `LazyInitializationException`), and tightly couple your API contract to your database schema. Always map to a DTO using `Page.map()` before returning.

**Q: Why does `@Async` (or `@Cacheable`) sometimes not work?**
Both annotations are implemented via Spring AOP proxies. If you call the annotated method from *within the same class*, the call bypasses the proxy and the annotation is silently ignored. The fix is to call the method through an injected reference to the bean, not via `this`.

**Q: What is the difference between `fixedRate` and `fixedDelay`?**
`fixedRate` fires every N milliseconds measured from the *start* of the previous execution — if the job is slow, executions can overlap. `fixedDelay` waits N milliseconds after the *previous execution finishes*, so there is always a gap. For long-running jobs, `fixedDelay` is safer.

**Q: When should you use `@CachePut` vs `@Cacheable`?**
`@Cacheable` skips the method if the key is already cached — use it for reads. `@CachePut` always executes the method and updates the cache entry — use it on update operations so the cache stays consistent without requiring a separate evict-then-read cycle.

**Q: How do you prevent path traversal attacks in file uploads?**
Never use the filename provided by the client. Always generate a UUID-based filename for storage. If you must preserve the original name, run it through `StringUtils.cleanPath()` and strip any `../` sequences. Validate the file extension against a whitelist before saving.

---

## Quick Reference Cheat Sheet

### Pagination
```
Repository  : Page<T> findBy...(Pageable pageable)
Service     : page.map(entity -> new DTO(...))
Controller  : method arg Pageable pageable + @PageableDefault
Build       : PageRequest.of(page, size, Sort.by("field").descending())
Query params: ?page=0&size=20&sort=price,desc
```

### File Upload / Download
```
Upload  : @PostMapping(consumes = MULTIPART_FORM_DATA_VALUE)
          @RequestParam("file") MultipartFile file
Download: return ResponseEntity<Resource> with Content-Disposition: attachment
Validate: check empty, whitelist extension, UUID rename, size limit in props
```

### Scheduling
```
Enable : @EnableScheduling on @Configuration/@SpringBootApplication
Rate   : @Scheduled(fixedRate = 60_000)          // every 60s from start
Delay  : @Scheduled(fixedDelay = 30_000)         // 30s after finish
Cron   : @Scheduled(cron = "0 0 8 * * MON-FRI") // 6 fields: s m h d M wd
```

### Async
```
Enable : @EnableAsync
Method : @Async — returns void or CompletableFuture<T>
Config : define ThreadPoolTaskExecutor bean (avoids unbounded thread creation)
Gotcha : self-invocation bypasses proxy — must call through injected bean
```

### Caching
```
Enable     : @EnableCaching
Read       : @Cacheable(value = "cache", key = "#id")
Update     : @CachePut(value = "cache", key = "#dto.id")
Delete     : @CacheEvict(value = "cache", key = "#id")
Clear all  : @CacheEvict(value = "cache", allEntries = true)
Gotcha     : self-invocation + no TTL + caching mutable entities = stale data
```

### Email
```
Dep    : spring-boot-starter-mail
Config : spring.mail.host / port / username / password (use env var)
Send   : JavaMailSender.send(MimeMessage)
Helper : new MimeMessageHelper(message, true, "UTF-8") for HTML + attachments
Tip    : wrap send call in @Async so HTTP response is not blocked
```

---

*Last Updated: 2026-06-18*

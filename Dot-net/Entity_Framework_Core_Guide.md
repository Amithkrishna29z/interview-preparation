# Entity Framework Core — A Study Guide for JPA/Hibernate Developers

## Overview

**Entity Framework Core (EF Core)** is the modern, open-source, cross-platform **Object-Relational Mapper (ORM)** for .NET. If you come from the Java world, EF Core is to .NET what **JPA + Hibernate** is to Java: it lets you work with a relational database using C# objects instead of writing raw SQL, handles change tracking, generates SQL, manages relationships, and provides a migration system.

The mental model is almost identical:

- You define **entity classes** (POJOs in Java → POCOs "Plain Old CLR Objects" in C#).
- A **context object** (`EntityManager`/`Session` in JPA → `DbContext` in EF Core) acts as your unit of work and first-level cache.
- Changes are tracked automatically and flushed to the database in one shot (`em.flush()`/`tx.commit()` → `SaveChanges()`).
- You query with a high-level language (JPQL/Criteria API → **LINQ**) that is translated to SQL.
- Schema evolution is managed by **migrations** (Flyway/Liquibase → EF Core Migrations).

The biggest conceptual difference: **JPA is a specification with multiple implementations** (Hibernate, EclipseLink). **EF Core is both the specification AND the implementation** — there is one EF Core, maintained by Microsoft, with pluggable database **providers** (SQL Server, PostgreSQL, SQLite, etc.) sitting underneath.

This guide assumes **EF Core 8+** and maps every concept back to the JPA/Hibernate equivalent you already know.

---

## Table of Contents

- [Overview](#overview)
- [JPA/Hibernate → EF Core Mapping](#jpahibernate--ef-core-mapping)
- [1. What EF Core Is: ORM, Code-First vs Database-First](#1-what-ef-core-is-orm-code-first-vs-database-first)
- [2. DbContext and DbSet](#2-dbcontext-and-dbset)
- [3. Defining Entities: Conventions, Data Annotations, Fluent API](#3-defining-entities-conventions-data-annotations-fluent-api)
- [4. Keys, Identity, Composite Keys, Value Generation](#4-keys-identity-composite-keys-value-generation)
- [5. Relationships and Navigation Properties](#5-relationships-and-navigation-properties)
- [6. The Change Tracker](#6-the-change-tracker)
- [7. Querying with LINQ](#7-querying-with-linq)
- [8. Loading Related Data and the N+1 Problem](#8-loading-related-data-and-the-n1-problem)
- [9. AsNoTracking for Read-Only Queries](#9-asnotracking-for-read-only-queries)
- [10. IQueryable vs IEnumerable (Client vs Server Evaluation)](#10-iqueryable-vs-ienumerable-client-vs-server-evaluation)
- [11. CRUD and Bulk Operations](#11-crud-and-bulk-operations)
- [12. Migrations](#12-migrations)
- [13. Transactions and Optimistic Concurrency](#13-transactions-and-optimistic-concurrency)
- [14. Connection Strings, Providers, and Dependency Injection](#14-connection-strings-providers-and-dependency-injection)
- [15. DbContext Lifetime, Scoping, and Thread-Safety](#15-dbcontext-lifetime-scoping-and-thread-safety)
- [16. Raw SQL](#16-raw-sql)
- [17. Seeding Data](#17-seeding-data)
- [18. Worked Example: Blog/Post One-to-Many with Full CRUD](#18-worked-example-blogpost-one-to-many-with-full-crud)
- [Common Interview Questions](#common-interview-questions)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## JPA/Hibernate → EF Core Mapping

| JPA / Hibernate | EF Core | Notes |
|---|---|---|
| `@Entity` class | Plain class + `DbSet<T>` on the context | EF discovers entities via `DbSet` properties or `modelBuilder.Entity<T>()` |
| `EntityManager` (per-request) | `DbContext` | Unit of work + identity map + L1 cache |
| `SessionFactory` / `EntityManagerFactory` | `DbContextOptions` + DI registration | The "factory" is configuration registered once at startup |
| `@Id` | `[Key]` or `Id`/`<Type>Id` convention | EF auto-detects a property named `Id` or `ClassNameId` |
| `@GeneratedValue(strategy = IDENTITY)` | Identity column (default for `int`/`long` keys) | `ValueGeneratedOnAdd()` in Fluent API |
| `@Column(name=...)` | `[Column("...")]` or `.HasColumnName("...")` | Annotation **or** Fluent API |
| `@Table(name=...)` | `[Table("...")]` or `.ToTable("...")` | |
| `@Transient` | `[NotMapped]` or `.Ignore()` | Exclude property from mapping |
| `@Column(nullable=false)` | `[Required]` or `.IsRequired()` | Non-nullable C# types are required by default |
| `@OneToMany` / `@ManyToOne` | Navigation properties + FK convention | `List<Post> Posts` + `Blog Blog` / `int BlogId` |
| `@OneToOne` | Reference navigation both ways | Configure dependent's PK or FK |
| `@ManyToMany` | Collections both sides (auto join table) | EF Core 5+ creates the join table by convention |
| `@JoinColumn` | `.HasForeignKey(...)` | Fluent API for explicit FK |
| JPQL / Criteria API / HQL | **LINQ** (method or query syntax) | Translated to SQL by the provider |
| `@Repository` / Spring Data repo | `DbContext` + `DbSet<T>` (optionally a repository wrapper) | EF *is* the repository/unit-of-work; extra repo layer is optional |
| Flyway / Liquibase | **EF Core Migrations** (`dotnet ef`) | Code-generated, versioned, applied to DB |
| `FetchType.LAZY` / `FetchType.EAGER` | `Include()` (eager) / lazy-loading proxies / explicit `Load()` | EF defaults to **no loading** unless you `Include` |
| `@Transactional` | Implicit transaction per `SaveChanges()`; or `BeginTransaction()` | `SaveChanges()` wraps all changes in one transaction |
| Persistence context (L1 cache) | **Change Tracker** | Identity map + dirty checking |
| Dirty checking | Snapshot change detection | EF compares snapshots on `SaveChanges()` |
| `@Version` (optimistic locking) | `[Timestamp]` / `IsRowVersion()` / `IsConcurrencyToken()` | Throws `DbUpdateConcurrencyException` |
| `session.get()` / `em.find()` | `context.Set<T>().Find(id)` | Checks L1 cache first, then DB |
| `em.persist()` | `context.Add(entity)` | Marks as `Added` |
| `em.merge()` | `context.Update(entity)` | Marks graph as `Modified` |
| `em.remove()` | `context.Remove(entity)` | Marks as `Deleted` |
| `em.flush()` | `context.SaveChanges()` | Writes pending changes |
| `em.detach()` / `clear()` | `Entry(e).State = Detached` / `ChangeTracker.Clear()` | |
| Second-level cache (Ehcache) | No built-in L2 cache | Use a library (`EFCoreSecondLevelCacheInterceptor`) if needed |

---

## 1. What EF Core Is: ORM, Code-First vs Database-First

**Think of it like...** a universal translator between your C# object graph and SQL tables — the same job Hibernate does for your Java object graph. You speak "objects," it speaks "SQL," and it keeps both sides in sync.

An **ORM** removes the impedance mismatch between objects (which have references, inheritance, collections) and relational tables (which have rows, columns, foreign keys). Instead of `ResultSet rs = stmt.executeQuery(...)` and manual mapping, you get strongly-typed objects.

**Two workflows** (same as JPA):

- **Code-First** — You write C# classes, EF generates the database schema via **migrations**. This is the dominant modern approach, analogous to Hibernate's `hbm2ddl`/JPA entity-driven schema generation combined with Flyway-style versioning. You own the model in code.
- **Database-First** — You start from an existing database and **scaffold** entity classes from it using `dotnet ef dbcontext scaffold`. Analogous to using Hibernate reverse-engineering tools against a legacy schema.

```csharp
// Code-First: this class IS the source of truth for the "Blogs" table.
// Compare to a JPA @Entity-annotated POJO.
public class Blog
{
    public int Id { get; set; }          // Convention: becomes the primary key (identity column)
    public string Url { get; set; }       // Non-nullable string -> NOT NULL column
}
```

```bash
# Database-First: generate entities + DbContext from an existing DB
# (Like Hibernate's reverse-engineering / JPA scaffolding)
dotnet ef dbcontext scaffold "Server=.;Database=MyDb;Trusted_Connection=True;TrustServerCertificate=True" Microsoft.EntityFrameworkCore.SqlServer
```

---

## 2. DbContext and DbSet

**Think of it like...** `DbContext` is your `EntityManager`/`Session` — a short-lived unit of work that tracks everything you touch. `DbSet<T>` is the typed gateway to one table, similar to a JPA repository or `em.createQuery("from Entity")` rolled into one.

The `DbContext` is the heart of EF Core. It:
- Holds the **change tracker** (the persistence context / L1 cache).
- Exposes **`DbSet<T>`** properties — one per entity type, your query root and insertion point.
- Manages the database **connection** and **transactions**.
- Is configured via `DbContextOptions` (provider + connection string).

```csharp
using Microsoft.EntityFrameworkCore;

// Your unit of work. Compare to extending nothing in JPA but using EntityManager.
public class AppDbContext : DbContext
{
    // Constructor receives options (provider, connection string) via DI.
    // Compare to building an EntityManagerFactory from persistence.xml.
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    // Each DbSet<T> is a queryable collection mapped to a table.
    // Compare to a Spring Data JpaRepository<Blog, Integer>, but combined.
    public DbSet<Blog> Blogs => Set<Blog>();
    public DbSet<Post> Posts => Set<Post>();

    // Fluent API configuration goes here (see section 3).
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
    }
}
```

Key difference from `SessionFactory`: there is no separate long-lived "factory" object you create at runtime. The factory-equivalent is the **service registration** done once at startup (`AddDbContext`, section 14); each web request gets a fresh `DbContext` instance.

---

## 3. Defining Entities: Conventions, Data Annotations, Fluent API

**Think of it like...** three layers of configuration that mirror JPA's: convention (sensible defaults, like Hibernate's naming strategy), **Data Annotations** (`[Key]`, `[Required]` — exactly like JPA `@Id`, `@Column`), and the **Fluent API** (`OnModelCreating` — like a programmatic Hibernate `Configuration` or JPA `@Embeddable`/orm.xml, but type-safe and far more powerful).

EF Core resolves the model in this order of increasing specificity: **Convention → Data Annotations → Fluent API**. Fluent API always wins.

### Conventions (zero config)

```csharp
public class Product
{
    public int Id { get; set; }            // "Id" -> primary key by convention
    public string Name { get; set; }        // string property -> nvarchar(max) NOT NULL
    public decimal Price { get; set; }       // -> decimal column
    public string? Description { get; set; }  // nullable string -> NULL column
}
// EF infers: table "Products", PK "Id" (identity), columns by property name.
```

### Data Annotations (attribute-based, like JPA annotations)

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

[Table("tbl_products")]                       // == @Table(name="tbl_products")
public class Product
{
    [Key]                                     // == @Id
    public int ProductId { get; set; }

    [Required]                                // == @Column(nullable=false)
    [MaxLength(100)]                          // == @Column(length=100)
    [Column("product_name")]                  // == @Column(name="product_name")
    public string Name { get; set; } = "";

    [Column(TypeName = "decimal(18,2)")]      // == @Column(precision=18, scale=2)
    public decimal Price { get; set; }

    [NotMapped]                               // == @Transient
    public string DisplayLabel => $"{Name} (${Price})";
}
```

### Fluent API (`OnModelCreating`) — the powerful option

The Fluent API keeps mapping concerns out of your entity classes (cleaner POCOs) and can express things annotations cannot (composite keys, complex relationships, indexes, default values).

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Product>(entity =>
    {
        entity.ToTable("tbl_products");          // table mapping
        entity.HasKey(p => p.ProductId);          // primary key  (== @Id)

        entity.Property(p => p.Name)
              .HasColumnName("product_name")       // (== @Column(name=...))
              .HasMaxLength(100)                   // (== length=100)
              .IsRequired();                       // (== nullable=false)

        entity.Property(p => p.Price)
              .HasPrecision(18, 2);                // (== precision/scale)

        entity.HasIndex(p => p.Name).IsUnique();   // unique index (== @Column(unique=true))

        entity.Ignore(p => p.DisplayLabel);        // (== @Transient)
    });
}
```

A clean pattern for large models is to split each entity's Fluent config into an `IEntityTypeConfiguration<T>` class (like having one `*.hbm.xml` or mapping class per entity) and call `modelBuilder.ApplyConfigurationsFromAssembly(...)`.

---

## 4. Keys, Identity, Composite Keys, Value Generation

**Think of it like...** `[Key]` is `@Id`, an identity column is `@GeneratedValue(strategy = IDENTITY)`, and the Fluent `HasKey(...)` with multiple columns replaces JPA's `@IdClass`/`@EmbeddedId`.

```csharp
public class Order
{
    public int Id { get; set; }   // int/long named Id (or OrderId) -> identity PK by default.
                                   // EF emits: IDENTITY(1,1) in SQL Server (auto-increment).
                                   // == @GeneratedValue(strategy = GenerationType.IDENTITY)
}
```

**Value generation modes** (configured via Fluent API):

```csharp
// Generated on INSERT (database identity / sequence). Default for integer keys.
entity.Property(o => o.Id).ValueGeneratedOnAdd();

// Never generated by DB — you supply it (== @GeneratedValue absent, assigned key).
entity.Property(o => o.Id).ValueGeneratedNever();

// GUID/Guid keys: EF generates a sequential GUID client-side on Add.
public Guid Id { get; set; }   // == JPA UUID strategy

// Database-computed default (e.g., timestamps)
entity.Property(o => o.CreatedAt).HasDefaultValueSql("GETUTCDATE()");
```

**Composite keys** — only possible via Fluent API (no annotation equivalent):

```csharp
modelBuilder.Entity<OrderLine>()
    .HasKey(ol => new { ol.OrderId, ol.ProductId });  // == @IdClass / @EmbeddedId in JPA
```

---

## 5. Relationships and Navigation Properties

**Think of it like...** navigation properties ARE your `@OneToMany`/`@ManyToOne` collections and references — but EF infers most of the wiring from **conventions**, so you write far fewer annotations than JPA.

EF Core recognizes relationships by **navigation properties** + a **foreign key property** following naming conventions: `<NavigationName>Id` or `<PrincipalType>Id`.

### One-to-Many / Many-to-One

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Url { get; set; } = "";

    // Collection navigation: the "one" side. == @OneToMany(mappedBy="blog")
    public List<Post> Posts { get; set; } = new();
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = "";

    // Foreign key property (convention: <Nav>Id). == the FK column.
    public int BlogId { get; set; }

    // Reference navigation: the "many" side. == @ManyToOne @JoinColumn(name="BlogId")
    public Blog Blog { get; set; } = null!;
}
// By convention EF builds: Posts.BlogId FK -> Blogs.Id. No annotations needed.
```

Explicit Fluent config (when conventions aren't enough):

```csharp
modelBuilder.Entity<Post>()
    .HasOne(p => p.Blog)                 // many-to-one  (== @ManyToOne)
    .WithMany(b => b.Posts)              // one-to-many  (== @OneToMany(mappedBy))
    .HasForeignKey(p => p.BlogId)        // FK column    (== @JoinColumn)
    .OnDelete(DeleteBehavior.Cascade);   // == cascade = CascadeType / FK ON DELETE CASCADE
```

### One-to-One

```csharp
public class User { public int Id { get; set; } public UserProfile Profile { get; set; } = null!; }
public class UserProfile
{
    public int Id { get; set; }
    public int UserId { get; set; }      // FK that is also unique -> one-to-one
    public User User { get; set; } = null!;
}

modelBuilder.Entity<User>()
    .HasOne(u => u.Profile)              // == @OneToOne
    .WithOne(p => p.User)
    .HasForeignKey<UserProfile>(p => p.UserId);
```

### Many-to-Many (no join entity needed)

```csharp
public class Student { public int Id { get; set; } public List<Course> Courses { get; set; } = new(); }
public class Course  { public int Id { get; set; } public List<Student> Students { get; set; } = new(); }
// EF Core 5+ auto-creates the "CourseStudent" join table.
// == @ManyToMany @JoinTable in JPA, but with ZERO configuration.

// If you need extra columns on the join (e.g., Grade), define an explicit join entity
// (== mapping the @ManyToMany as two @OneToMany to an association entity).
```

---

## 6. The Change Tracker

**Think of it like...** the change tracker IS Hibernate's persistence context + dirty checking. Every entity you load or add is tracked; when you call `SaveChanges()`, EF diffs the current values against the snapshot it took (dirty checking) and emits only the necessary `INSERT`/`UPDATE`/`DELETE` statements.

Each tracked entity has an **`EntityState`**:

| EntityState | Meaning | JPA equivalent | SQL on SaveChanges |
|---|---|---|---|
| `Added` | New, not yet inserted | persisted (new) after `em.persist()` | `INSERT` |
| `Unchanged` | Loaded, untouched | managed, clean | none |
| `Modified` | Tracked + a property changed | managed, dirty | `UPDATE` |
| `Deleted` | Marked for removal | removed after `em.remove()` | `DELETE` |
| `Detached` | Not tracked | detached | none |

```csharp
var blog = context.Blogs.First();      // state: Unchanged (now tracked, like a managed entity)
blog.Url = "https://new-url.com";       // state automatically becomes Modified (dirty checking)

var newBlog = new Blog { Url = "..." };
context.Blogs.Add(newBlog);             // state: Added  (== em.persist())

context.Blogs.Remove(blog);             // state: Deleted (== em.remove())

// Inspect the tracker (great for debugging / interviews):
var state = context.Entry(blog).State;  // EntityState.Deleted

// One round trip, one transaction. == em.flush() / tx.commit().
int affected = context.SaveChanges();
// Async version (preferred in web apps to free the thread):
// int affected = await context.SaveChangesAsync();
```

Important nuance vs JPA: a **detached** entity (e.g., one deserialized from an HTTP request) is NOT tracked. Calling `context.Update(entity)` marks the whole graph as `Modified` (similar to `em.merge()`), generating an `UPDATE` for every column.

---

## 7. Querying with LINQ

**Think of it like...** LINQ is JPQL/Criteria API, but it is part of the C# language itself — strongly typed, compiler-checked, and refactor-safe. The provider translates your LINQ expression tree into SQL.

```csharp
// Method syntax (most common). Each call builds the SQL; nothing runs until enumeration.
var recentPosts = context.Posts
    .Where(p => p.BlogId == 1 && p.Title.Contains("EF"))  // == JPQL WHERE
    .OrderByDescending(p => p.PublishedOn)                  // == ORDER BY
    .Take(10)                                               // == setMaxResults(10) / LIMIT
    .ToList();                                              // executes the SQL NOW

// Find by primary key — checks the change tracker (L1 cache) FIRST, then DB.
// == em.find(Blog.class, 1) / session.get()
var blog = context.Blogs.Find(1);

// First vs Single (semantics matter):
var first  = context.Posts.First(p => p.BlogId == 1);          // first match, throws if none
var firstOr= context.Posts.FirstOrDefault(p => p.BlogId == 1); // null if none
var single = context.Posts.Single(p => p.Id == 5);            // exactly one, throws if 0 or >1

// Projection to a DTO — selects only needed columns (== JPQL constructor expression / SELECT NEW).
// This is also a performance win: less data, and it is implicitly no-tracking.
var dtos = context.Posts
    .Where(p => p.BlogId == 1)
    .Select(p => new PostDto { Id = p.Id, Title = p.Title })   // SELECT Id, Title only
    .ToList();

// Aggregates translate to SQL too:
int count   = context.Posts.Count(p => p.BlogId == 1);   // SELECT COUNT(*)
bool any    = context.Posts.Any(p => p.BlogId == 1);      // SELECT CASE WHEN EXISTS...
decimal max = context.Posts.Max(p => p.Views);
```

Async variants exist for all terminal operators: `ToListAsync()`, `FirstOrDefaultAsync()`, `CountAsync()`, `FindAsync()`, etc. Prefer them in web apps.

---

## 8. Loading Related Data and the N+1 Problem

**Think of it like...** EF's `Include` is JPA's `JOIN FETCH`/eager fetch, explicit `Load()` is `Hibernate.initialize()`, and lazy-loading proxies are JPA `FetchType.LAZY`. The crucial difference: **EF Core loads NO related data by default** — there is no implicit lazy loading unless you explicitly enable it.

### Eager loading — `Include` / `ThenInclude`

```csharp
var blogs = context.Blogs
    .Include(b => b.Posts)                 // JOIN/load Posts   (== JOIN FETCH b.posts)
        .ThenInclude(p => p.Comments)      // then load each Post's Comments (nested fetch)
    .ToList();
// One query (or split queries) brings back the whole graph. No N+1.
```

### Explicit loading — load on demand for already-tracked entities

```csharp
var blog = context.Blogs.First();
context.Entry(blog).Collection(b => b.Posts).Load();   // == Hibernate.initialize(blog.getPosts())
context.Entry(post).Reference(p => p.Blog).Load();      // load a single reference
```

### Lazy loading — opt-in proxies

```csharp
// Requires: Microsoft.EntityFrameworkCore.Proxies + .UseLazyLoadingProxies()
// AND navigation properties must be 'virtual' (so EF can subclass/proxy them).
public virtual List<Post> Posts { get; set; } = new();  // == FetchType.LAZY default in JPA
// Accessing blog.Posts then fires a SQL query transparently — and can trigger N+1.
```

### The N+1 Problem (a top interview topic)

```csharp
// BAD: 1 query for blogs, then 1 query PER blog for its posts = N+1 queries.
var blogs = context.Blogs.ToList();          // 1 query
foreach (var b in blogs)
    Console.WriteLine(b.Posts.Count);         // each access -> +1 query (if lazy loading on)

// GOOD: eager-load up front -> 1 (or 2 split) queries total.
var blogs2 = context.Blogs.Include(b => b.Posts).ToList();

// ALSO GOOD: project only what you need (no entities loaded at all).
var summary = context.Blogs
    .Select(b => new { b.Url, PostCount = b.Posts.Count })   // COUNT done in SQL
    .ToList();
```

**Why N+1 matters:** it turns one logical operation into hundreds of round trips, killing performance under load. The fix is identical in spirit to JPA: fetch joins (`Include`), DTO projections, or batch/split queries. EF Core's choice to **not lazy-load by default** is deliberate — it makes the N+1 trap visible rather than silent.

`AsSplitQuery()` can avoid the "cartesian explosion" when including multiple collections:

```csharp
var blogs = context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Authors)
    .AsSplitQuery()    // run one SQL per Include instead of one big JOIN
    .ToList();
```

---

## 9. AsNoTracking for Read-Only Queries

**Think of it like...** `AsNoTracking()` is the equivalent of a JPA read-only/stateless query — you tell EF "I'm only reading, don't put these in the persistence context." It skips snapshotting, so it is faster and uses less memory.

```csharp
// Tracked (default): EF stores a snapshot of every row for change detection.
var tracked = context.Posts.ToList();

// No-tracking: faster, lower memory, but the returned objects are NOT managed.
// Editing them and calling SaveChanges() does nothing. Use for pure reads / display.
var readOnly = context.Posts.AsNoTracking().ToList();   // == read-only / detached query

// Make it the default for a whole context if it is read-mostly:
context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
```

**Why it helps:** the change tracker's snapshotting and identity-map bookkeeping have real cost. For GET endpoints that never mutate data, `AsNoTracking()` can noticeably cut allocations and CPU. Note: **projections to DTOs are already non-tracked**, so you don't need `AsNoTracking()` with `Select(... new Dto ...)`.

---

## 10. IQueryable vs IEnumerable (Client vs Server Evaluation)

**Think of it like...** `IQueryable<T>` is a query *recipe* that gets translated to SQL and runs in the database (server evaluation), exactly like a JPQL/Criteria query. The moment you switch to `IEnumerable<T>`, you've pulled the data into memory and any further filtering runs in C# (client evaluation). Mixing them up is the classic "you accidentally loaded the whole table" bug.

```csharp
// IQueryable: composable, deferred, translated to SQL. NOTHING has hit the DB yet.
IQueryable<Post> query = context.Posts.Where(p => p.BlogId == 1);  // builds SQL expression tree
query = query.OrderBy(p => p.Title);                                // still just building SQL
var list = query.ToList();   // <-- SQL executes HERE: SELECT ... WHERE BlogId=1 ORDER BY Title

// IEnumerable: in-memory. The Where below runs in C#, NOT in SQL.
IEnumerable<Post> all = context.Posts.AsEnumerable();   // <-- SELECT * FROM Posts (whole table!)
var filtered = all.Where(p => p.BlogId == 1);            // filters in MEMORY after loading all rows. BAD.
```

**Rules to remember:**
- Keep the chain as `IQueryable` as long as possible so filtering/paging happens in SQL.
- `AsEnumerable()`, `ToList()`, `ToArray()`, or a `foreach` are the **execution boundary** — after them, you're in memory.
- Calling a method EF can't translate to SQL (e.g., a custom C# method inside `Where`) will throw in EF Core 3+, forcing you to be explicit instead of silently degrading to client evaluation.

```csharp
// Pitfall: pagination AFTER ToList loads everything first.
var bad  = context.Posts.ToList().Skip(20).Take(10);      // loads ALL rows, then pages in memory
var good = context.Posts.Skip(20).Take(10).ToList();       // OFFSET/FETCH in SQL — only 10 rows
```

This deferred-execution model is comparable to JPA's `TypedQuery` not running until `getResultList()`, but LINQ's composability makes the client/server boundary subtler — hence its prominence in interviews.

---

## 11. CRUD and Bulk Operations

**Think of it like...** `Add`/`Update`/`Remove` are `em.persist`/`em.merge`/`em.remove`, batched and flushed together by `SaveChanges()`. EF Core 7+ adds `ExecuteUpdate`/`ExecuteDelete` for set-based bulk operations that skip the change tracker entirely — like a JPQL bulk `UPDATE`/`DELETE`.

```csharp
// CREATE
context.Blogs.Add(new Blog { Url = "https://a.com" });         // == em.persist()
context.Blogs.AddRange(blog1, blog2, blog3);                    // batch insert
await context.SaveChangesAsync();

// READ
var blog = await context.Blogs.FindAsync(1);

// UPDATE (tracked): just mutate + save — dirty checking emits the UPDATE.
blog!.Url = "https://updated.com";
await context.SaveChangesAsync();

// UPDATE (detached, e.g., from an API request body):
context.Blogs.Update(detachedBlog);   // marks whole graph Modified (== em.merge())
await context.SaveChangesAsync();

// DELETE
context.Blogs.Remove(blog);                                    // == em.remove()
context.Blogs.RemoveRange(context.Blogs.Where(b => b.IsArchived));
await context.SaveChangesAsync();

// BULK set-based ops (EF Core 7+): single SQL statement, NO entities loaded/tracked.
// == JPQL "UPDATE Post p SET p.archived=true WHERE ..." — runs in the database directly.
await context.Posts
    .Where(p => p.PublishedOn < DateTime.UtcNow.AddYears(-5))
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.IsArchived, true));

await context.Posts
    .Where(p => p.IsSpam)
    .ExecuteDeleteAsync();   // single DELETE ... WHERE, no SaveChanges needed
```

Note: `ExecuteUpdate`/`ExecuteDelete` bypass the change tracker and run immediately (they are *not* deferred), so they don't participate in `SaveChanges()` batching and won't update already-tracked in-memory entities.

---

## 12. Migrations

**Think of it like...** EF Migrations are Flyway/Liquibase, but generated *from your C# model* instead of hand-written SQL/XML. Each migration is a versioned C# class with `Up()`/`Down()` methods; EF tracks applied migrations in a `__EFMigrationsHistory` table (just like Flyway's `flyway_schema_history`).

```bash
# Install the CLI tool once (global)
dotnet tool install --global dotnet-ef

# Create a migration from the current model diff (like writing a new Flyway script,
# but auto-generated by comparing your model to the last snapshot).
dotnet ef migrations add AddBlogTable

# Apply pending migrations to the database (== flyway migrate)
dotnet ef database update

# Roll back to a specific migration (== flyway undo / targeted migrate)
dotnet ef database update PreviousMigrationName

# Remove the last (unapplied) migration
dotnet ef migrations remove

# Generate an idempotent SQL script for production (no live DB connection needed)
dotnet ef migrations script --idempotent
```

A generated migration looks like:

```csharp
public partial class AddBlogTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)   // == Flyway "up" script
    {
        migrationBuilder.CreateTable(
            name: "Blogs",
            columns: table => new
            {
                Id  = table.Column<int>(nullable: false)
                           .Annotation("SqlServer:Identity", "1, 1"),   // auto-increment
                Url = table.Column<string>(nullable: false)
            },
            constraints: table => table.PrimaryKey("PK_Blogs", x => x.Id));
    }

    protected override void Down(MigrationBuilder migrationBuilder)  // == rollback script
        => migrationBuilder.DropTable(name: "Blogs");
}
```

**Production tip:** apply migrations explicitly (`dotnet ef database update` or a generated SQL script in CI/CD), not via `context.Database.Migrate()` at app startup in multi-instance deployments — concurrent instances can race, the same problem Flyway solves with locking.

---

## 13. Transactions and Optimistic Concurrency

**Think of it like...** `SaveChanges()` is already a single transaction (`@Transactional` around your unit of work). For multiple `SaveChanges()` or raw SQL in one atomic unit, use an explicit transaction. `[Timestamp]`/RowVersion is JPA's `@Version` — optimistic locking that throws when a concurrent update is detected.

### Transactions

```csharp
// Implicit: every SaveChanges() is wrapped in one transaction automatically.
context.SaveChanges();   // all tracked changes commit together, or all roll back.

// Explicit: span multiple SaveChanges / raw SQL atomically. == @Transactional method.
using var tx = await context.Database.BeginTransactionAsync();
try
{
    context.Add(blog);
    await context.SaveChangesAsync();

    await context.Database.ExecuteSqlRawAsync("UPDATE Stats SET BlogCount = BlogCount + 1");
    await context.SaveChangesAsync();

    await tx.CommitAsync();     // == tx.commit()
}
catch
{
    await tx.RollbackAsync();   // == tx.rollback()
    throw;
}
```

### Optimistic Concurrency (== `@Version`)

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";

    [Timestamp]                       // SQL Server rowversion; auto-incremented by the DB.
    public byte[]? RowVersion { get; set; }   // == @Version
}

// Fluent alternative (e.g., for PostgreSQL xmin, or a manual int version):
modelBuilder.Entity<Product>().Property(p => p.RowVersion).IsRowVersion();
// or:  .Property(p => p.Version).IsConcurrencyToken();   // any column as the version

// On SaveChanges, EF adds "WHERE Id=@id AND RowVersion=@original" to the UPDATE.
// If 0 rows match (someone else changed it first), EF throws:
try
{
    product.Name = "Updated";
    await context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)   // == OptimisticLockException
{
    // Reload, merge, or surface a conflict to the user.
}
```

---

## 14. Connection Strings, Providers, and Dependency Injection

**Think of it like...** the **provider** is your JDBC driver + Hibernate dialect rolled into one NuGet package, the connection string is your JDBC URL, and `AddDbContext` is registering the `EntityManagerFactory` with the container so each request gets a scoped `EntityManager`.

EF Core is database-agnostic; you pick a **provider** package:

| Database | Provider package | Hibernate dialect analogy |
|---|---|---|
| SQL Server | `Microsoft.EntityFrameworkCore.SqlServer` | `SQLServerDialect` |
| PostgreSQL | `Npgsql.EntityFrameworkCore.PostgreSQL` | `PostgreSQLDialect` |
| SQLite | `Microsoft.EntityFrameworkCore.Sqlite` | `SQLiteDialect` |
| MySQL | `Pomelo.EntityFrameworkCore.MySql` | `MySQLDialect` |
| In-Memory (tests) | `Microsoft.EntityFrameworkCore.InMemory` | H2 in-memory analogy |

```jsonc
// appsettings.json  (== persistence.xml / application.properties)
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=BloggingDb;Trusted_Connection=True;TrustServerCertificate=True"
  }
}
```

```csharp
// Program.cs — register DbContext with DI. Done ONCE at startup.
// This is the SessionFactory/EntityManagerFactory equivalent.
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(                                   // pick provider + dialect
        builder.Configuration.GetConnectionString("Default")));
// Swap provider by changing one line, e.g. options.UseNpgsql(...) or options.UseSqlite(...)

// Now any service/controller can inject AppDbContext via its constructor:
public class BlogService(AppDbContext db)   // primary constructor injection
{
    public Task<List<Blog>> GetAllAsync() => db.Blogs.AsNoTracking().ToListAsync();
}
```

`AddDbContext` registers the context with a **Scoped** lifetime by default (one instance per HTTP request) — exactly the right lifetime for a unit of work, mirroring a per-request `EntityManager`.

---

## 15. DbContext Lifetime, Scoping, and Thread-Safety

**Think of it like...** a `DbContext` is as short-lived and single-threaded as a JPA `EntityManager`. You would never share one `EntityManager` across threads or keep it alive for the whole app — same rules here. This is one of the most common real-world EF bugs.

Key facts:
- **`DbContext` is NOT thread-safe.** Using one instance from multiple threads concurrently (e.g., `Task.WhenAll` over the same context) throws `InvalidOperationException: A second operation was started on this context instance before a previous operation completed.`
- **Lifetime = Scoped** (per request) by default via `AddDbContext`. Don't make it a Singleton.
- A long-lived context accumulates tracked entities (memory leak) and serves stale data — like never closing an `EntityManager`.

```csharp
// WRONG: concurrent use of one context instance.
await Task.WhenAll(
    context.Blogs.ToListAsync(),
    context.Posts.ToListAsync());   // THROWS: second operation on same context

// RIGHT: await sequentially, OR use a separate context per parallel operation.
var blogs = await context.Blogs.ToListAsync();
var posts = await context.Posts.ToListAsync();

// For background services / parallel work, create scopes (each gets its own context):
using var scope = serviceProvider.CreateScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

// For factory-style creation (Blazor, console, parallel), register a pooled factory:
builder.Services.AddDbContextFactory<AppDbContext>(o => o.UseSqlServer(cs));
// then: using var db = factory.CreateDbContext();
```

---

## 16. Raw SQL

**Think of it like...** `FromSql` is JPA's `createNativeQuery(...)` that maps results back to entities, and `ExecuteSqlRaw` is a native `executeUpdate()`. Use parameterized interpolation to stay safe from SQL injection.

```csharp
// Query entities from raw SQL — results are tracked like any LINQ query.
// FromSql uses {0}/interpolation as PARAMETERS (safe). == createNativeQuery(..., Blog.class)
var blogs = context.Blogs
    .FromSql($"SELECT * FROM Blogs WHERE Url LIKE {pattern}")   // parameterized, injection-safe
    .Where(b => b.Id > 10)        // you can STILL compose LINQ on top (it runs as a subquery)
    .ToList();

// Scalar / non-entity SQL execution (INSERT/UPDATE/DELETE/DDL). == native executeUpdate()
int rows = context.Database.ExecuteSqlRaw(
    "UPDATE Blogs SET Url = {0} WHERE Id = {1}", newUrl, id);   // parameters, not concatenation
// Async: ExecuteSqlInterpolatedAsync($"...{var}...")

// Map raw SQL to a keyless DTO (== native query with a result-set mapping):
// configure with modelBuilder.Entity<BlogStat>().HasNoKey(); then:
var stats = context.Database.SqlQuery<int>($"SELECT COUNT(*) FROM Blogs").ToList();
```

**Never** build SQL with string concatenation of user input — always use `{param}` interpolation or positional `{0}` parameters, which EF turns into `DbParameter`s.

---

## 17. Seeding Data

**Think of it like...** seeding is Flyway/Liquibase reference-data scripts or a Hibernate `import.sql` — initial/static rows baked into the schema setup. EF supports model-level seeding (goes into migrations) and runtime seeding.

```csharp
// Model seeding via HasData — becomes INSERTs inside a migration.
// Requires explicit PK values. Good for static lookup/reference data.
modelBuilder.Entity<Blog>().HasData(
    new Blog { Id = 1, Url = "https://seed-a.com" },
    new Blog { Id = 2, Url = "https://seed-b.com" });

// Runtime seeding (dynamic data, dev/test setup) — like a CommandLineRunner in Spring Boot.
public static async Task SeedAsync(AppDbContext db)
{
    if (!await db.Blogs.AnyAsync())     // idempotent: only seed if empty
    {
        db.Blogs.Add(new Blog { Url = "https://first.com" });
        await db.SaveChangesAsync();
    }
}
```

EF Core 9 also adds a `UseSeeding`/`UseAsyncSeeding` hook configured on `DbContextOptions` for cleaner runtime seeding.

---

## 18. Worked Example: Blog/Post One-to-Many with Full CRUD

A complete, idiomatic example tying everything together.

```csharp
using Microsoft.EntityFrameworkCore;

// ---------- ENTITIES ----------
public class Blog
{
    public int Id { get; set; }                       // identity PK (== @Id @GeneratedValue)
    public string Url { get; set; } = "";              // NOT NULL
    public List<Post> Posts { get; set; } = new();     // one-to-many (== @OneToMany)
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public DateTime PublishedOn { get; set; }
    public int BlogId { get; set; }                    // FK column (convention)
    public Blog Blog { get; set; } = null!;            // many-to-one (== @ManyToOne)
}

// ---------- DTO (for read projections) ----------
public class PostDto { public int Id { get; set; } public string Title { get; set; } = ""; }

// ---------- DbContext ----------
public class BloggingContext : DbContext
{
    public BloggingContext(DbContextOptions<BloggingContext> options) : base(options) { }

    public DbSet<Blog> Blogs => Set<Blog>();
    public DbSet<Post> Posts => Set<Post>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Explicit relationship config (conventions would also work here).
        modelBuilder.Entity<Post>()
            .HasOne(p => p.Blog)                // == @ManyToOne
            .WithMany(b => b.Posts)             // == @OneToMany(mappedBy="blog")
            .HasForeignKey(p => p.BlogId)       // == @JoinColumn(name="BlogId")
            .OnDelete(DeleteBehavior.Cascade);  // deleting a Blog deletes its Posts

        modelBuilder.Entity<Blog>().Property(b => b.Url).IsRequired().HasMaxLength(500);

        // Seed reference data (lands in a migration).
        modelBuilder.Entity<Blog>().HasData(new Blog { Id = 1, Url = "https://dotnet.example" });
    }
}

// ---------- CRUD SERVICE ----------
public class BlogService(BloggingContext db)
{
    // CREATE: blog + child posts inserted together in one transaction.
    public async Task<int> CreateBlogWithPostsAsync(string url, params string[] titles)
    {
        var blog = new Blog
        {
            Url = url,
            Posts = titles.Select(t => new Post { Title = t, PublishedOn = DateTime.UtcNow }).ToList()
        };
        db.Blogs.Add(blog);                 // tracks blog AND posts as Added (cascade persist)
        await db.SaveChangesAsync();         // INSERT blog, then INSERT posts (FK auto-filled)
        return blog.Id;                      // PK populated after save (DB-generated identity)
    }

    // READ (eager): avoid N+1 by Include-ing posts up front.
    public async Task<Blog?> GetBlogWithPostsAsync(int id) =>
        await db.Blogs
            .Include(b => b.Posts)           // JOIN-load posts (== JOIN FETCH)
            .AsNoTracking()                  // read-only -> faster
            .FirstOrDefaultAsync(b => b.Id == id);

    // READ (projection): only the columns the UI needs, no tracking.
    public async Task<List<PostDto>> GetPostTitlesAsync(int blogId) =>
        await db.Posts
            .Where(p => p.BlogId == blogId)
            .OrderByDescending(p => p.PublishedOn)
            .Select(p => new PostDto { Id = p.Id, Title = p.Title })   // SELECT Id, Title
            .ToListAsync();

    // UPDATE (tracked): dirty checking emits a minimal UPDATE.
    public async Task UpdateBlogUrlAsync(int id, string newUrl)
    {
        var blog = await db.Blogs.FindAsync(id);   // tracked
        if (blog is null) return;
        blog.Url = newUrl;                          // state -> Modified
        await db.SaveChangesAsync();                 // UPDATE Blogs SET Url=... WHERE Id=...
    }

    // BULK UPDATE: set-based, no entities loaded (EF Core 7+).
    public Task ArchiveOldPostsAsync(int blogId) =>
        db.Posts
          .Where(p => p.BlogId == blogId && p.PublishedOn < DateTime.UtcNow.AddYears(-2))
          .ExecuteUpdateAsync(s => s.SetProperty(p => p.Title, p => "[ARCHIVED] " + p.Title));

    // DELETE: cascade removes child posts (configured above).
    public async Task DeleteBlogAsync(int id)
    {
        var blog = await db.Blogs.FindAsync(id);
        if (blog is null) return;
        db.Blogs.Remove(blog);                       // state -> Deleted
        await db.SaveChangesAsync();                  // DELETE (posts cascade)
    }
}
```

```csharp
// ---------- WIRING (Program.cs) ----------
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<BloggingContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddScoped<BlogService>();
var app = builder.Build();
```

```bash
# ---------- MIGRATE + RUN ----------
dotnet ef migrations add InitialCreate
dotnet ef database update
dotnet run
```

---

## Common Interview Questions

**1. What is EF Core and how does it compare to Hibernate?**
EF Core is Microsoft's ORM for .NET. Like Hibernate, it maps C# classes to tables, tracks changes, generates SQL, and manages relationships and migrations. Key differences: EF Core is a single product (not a spec with multiple implementations like JPA/Hibernate); it uses pluggable **providers** for different databases; LINQ is the query language (compiler-checked, part of C#) instead of JPQL; and EF Core does **not** lazy-load related data by default.

**2. What is a `DbContext` and how is it like/unlike an `EntityManager`?**
`DbContext` is the unit of work + identity map + change tracker, equivalent to a JPA `EntityManager`/Hibernate `Session`. It's short-lived (per request, Scoped), not thread-safe, and exposes `DbSet<T>` query roots. Unlike JPA, there's no separate runtime `SessionFactory`; configuration is registered once at startup via `AddDbContext`.

**3. Data Annotations vs Fluent API — when do you use each?**
Annotations (`[Key]`, `[Required]`, `[Column]`) are simple and live on the entity, like JPA annotations. The Fluent API (`OnModelCreating`) keeps entities as clean POCOs and is more powerful — it's required for composite keys, complex relationships, indexes, and default values. Fluent API overrides annotations, which override conventions.

**4. Explain the change tracker and EntityState.**
The change tracker is EF's persistence context. Each tracked entity has a state: `Added`, `Unchanged`, `Modified`, `Deleted`, or `Detached`. On `SaveChanges()`, EF performs **snapshot-based dirty checking** (like Hibernate) and emits `INSERT`/`UPDATE`/`DELETE` accordingly, all in one transaction. `AsNoTracking()` opts out for read-only queries.

**5. What is the N+1 problem and how do you avoid it in EF Core?**
Loading a list of parents and then accessing each parent's children triggers one query per parent (N+1 total). Avoid it with eager loading (`Include`/`ThenInclude`), DTO projections (`Select` that does the join/aggregate in SQL), or `AsSplitQuery()` for multiple collections. EF Core not lazy-loading by default makes the trap visible rather than silent.

**6. `Include` vs explicit loading vs lazy loading — what's the difference?**
`Include`/`ThenInclude` = eager loading via JOINs (like `JOIN FETCH`). Explicit loading = `Entry(e).Collection(...).Load()` on demand (like `Hibernate.initialize`). Lazy loading = opt-in proxies (`UseLazyLoadingProxies` + `virtual` navigations) that fire a query on first access (like JPA `FetchType.LAZY`), with N+1 risk.

**7. Explain `IQueryable` vs `IEnumerable`.**
`IQueryable<T>` is a deferred, composable query translated to SQL and executed in the database (server evaluation). `IEnumerable<T>` is in-memory; further LINQ runs in C# (client evaluation). Switching to `IEnumerable` (via `AsEnumerable`/`ToList`) too early loads more data than needed — e.g., paging or filtering after `ToList()` pulls the whole table first.

**8. What does `AsNoTracking()` do and why use it?**
It tells EF not to snapshot or track returned entities, skipping change-tracking overhead. It makes read-only queries faster and lighter on memory. Returned objects are detached, so edits won't be saved. Projections to DTOs are already non-tracked.

**9. How do migrations work, and how do they compare to Flyway/Liquibase?**
`dotnet ef migrations add <Name>` diffs your model against the last snapshot and generates a versioned C# migration with `Up()`/`Down()`. `dotnet ef database update` applies pending migrations, recording them in `__EFMigrationsHistory` (like Flyway's history table). Difference: EF migrations are generated from your code model, whereas Flyway/Liquibase scripts are typically hand-written.

**10. How does optimistic concurrency work in EF Core?**
Mark a property `[Timestamp]` (RowVersion) or `IsConcurrencyToken()`. EF adds the original value to the `WHERE` clause of `UPDATE`/`DELETE`. If a concurrent change means 0 rows match, EF throws `DbUpdateConcurrencyException` — the equivalent of JPA's `@Version`/`OptimisticLockException`. You then reload/merge/report the conflict.

**11. Is `DbContext` thread-safe? What lifetime should it have?**
No — `DbContext` is not thread-safe; concurrent operations on one instance throw `InvalidOperationException`. It should be **Scoped** (one per request), which `AddDbContext` does by default. Never make it a Singleton. For parallel/background work, create a new scope or use `AddDbContextFactory`.

**12. How do you do bulk updates/deletes efficiently?**
EF Core 7+ provides `ExecuteUpdate`/`ExecuteDelete`, which run a single set-based SQL statement without loading or tracking entities (like a JPQL bulk `UPDATE`/`DELETE`). They execute immediately and bypass the change tracker, so they don't batch with `SaveChanges()` or update in-memory tracked entities.

**13. `Find` vs `First` vs `Single`?**
`Find(id)` looks up by primary key and checks the change tracker (L1 cache) before hitting the DB (like `em.find`). `First`/`FirstOrDefault` returns the first match (throws/returns null if none). `Single`/`SingleOrDefault` asserts exactly one match (throws if zero or more than one).

**14. How do you map a many-to-many relationship?**
In EF Core 5+, just put a collection navigation on each side (`List<Course>` / `List<Student>`); EF auto-creates the join table. For extra columns on the join (e.g., a grade), define an explicit join entity with two one-to-many relationships — the same workaround as mapping a JPA `@ManyToMany` with attributes via an association entity.

**15. How do you run raw SQL safely?**
Use `FromSql`/`FromSqlInterpolated` to map results to entities, and `ExecuteSqlRaw`/`ExecuteSqlInterpolated` for commands. Always pass values as parameters (`{param}` interpolation or `{0}` positional), never string concatenation, to prevent SQL injection. `FromSql` results remain composable with LINQ.

---

## Quick Reference Cheat Sheet

```text
CORE OBJECTS
  DbContext            == EntityManager / Session (unit of work, change tracker, Scoped, NOT thread-safe)
  DbSet<T>             == query root + repository for one table
  DbContextOptions     == provider + connection string (registered via AddDbContext)

ENTITY MAPPING
  Convention           : Id / <Type>Id -> PK; property name -> column; non-null type -> NOT NULL
  Data Annotations     : [Key] [Required] [MaxLength] [Column] [Table] [NotMapped] [Timestamp]
  Fluent API           : OnModelCreating -> HasKey / Property / HasIndex / HasOne.WithMany (wins over all)

KEYS
  int/long Id          -> identity (auto-increment)        == @GeneratedValue(IDENTITY)
  Guid Id              -> sequential GUID client-side
  HasKey(e=>new{A,B})  -> composite key                    == @IdClass / @EmbeddedId
  ValueGeneratedNever  -> you assign the key

RELATIONSHIPS
  one-to-many   : List<Child> on parent + int ParentId + Parent nav on child
  many-to-one   : HasOne(c=>c.Parent).WithMany(p=>p.Children).HasForeignKey(c=>c.ParentId)
  one-to-one    : HasOne().WithOne().HasForeignKey<Dependent>(...)
  many-to-many  : collection on both sides -> auto join table (EF 5+)

CHANGE TRACKER STATES
  Added | Unchanged | Modified | Deleted | Detached
  Add()=persist  Update()=merge  Remove()=remove  SaveChanges()=flush/commit

QUERYING (LINQ)
  Where Select OrderBy Take Skip GroupBy Join
  Find(id)            : PK lookup, checks tracker first
  First/FirstOrDefault, Single/SingleOrDefault
  Count Any Sum Max   : translate to SQL aggregates
  *Async variants     : ToListAsync, FirstOrDefaultAsync, SaveChangesAsync (prefer in web apps)

LOADING RELATED DATA
  Eager   : .Include(b=>b.Posts).ThenInclude(p=>p.Comments)
  Explicit: context.Entry(blog).Collection(b=>b.Posts).Load()
  Lazy    : UseLazyLoadingProxies() + virtual nav (N+1 risk)
  Avoid N+1: Include, DTO projection, AsSplitQuery()

PERFORMANCE
  AsNoTracking()       : read-only, no snapshot (faster); projections already no-track
  IQueryable           : deferred, runs in SQL (server)  -- keep the chain queryable!
  IEnumerable/ToList   : execution boundary -> in-memory (client). Page/filter BEFORE ToList.

CRUD / BULK
  Add/AddRange  Update  Remove/RemoveRange  +  SaveChangesAsync()
  ExecuteUpdateAsync / ExecuteDeleteAsync   : set-based, no tracking, runs immediately (EF 7+)

MIGRATIONS (== Flyway/Liquibase)
  dotnet ef migrations add <Name>
  dotnet ef database update [TargetName]
  dotnet ef migrations remove
  dotnet ef migrations script --idempotent
  history table: __EFMigrationsHistory

TRANSACTIONS & CONCURRENCY
  SaveChanges()                          : one implicit transaction
  Database.BeginTransaction()/Commit/Rollback : explicit, multi-step
  [Timestamp] byte[] RowVersion          : optimistic lock (== @Version)
  catch DbUpdateConcurrencyException     : == OptimisticLockException

DI & PROVIDERS
  AddDbContext<T>(o => o.UseSqlServer(cs))   // Scoped lifetime by default
  Providers: UseSqlServer | UseNpgsql | UseSqlite | UseMySql (Pomelo) | UseInMemory
  AddDbContextFactory<T>()                   // for parallel/Blazor/console

RAW SQL
  context.Blogs.FromSql($"... {param}")             // entities, composable, parameterized
  context.Database.ExecuteSqlRaw("...", p0, p1)     // commands
  Always parameterize -> no SQL injection

SEEDING
  modelBuilder.Entity<T>().HasData(...)   // into migrations (static data, explicit PKs)
  runtime: if(!await db.Set.AnyAsync()) { ...; SaveChanges(); }   // idempotent
```

*Last Updated: 2026-06-16*

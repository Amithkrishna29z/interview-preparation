# C# LINQ Study Guide (for Java Devs Who Know Streams)

## Overview

**LINQ (Language Integrated Query)** is C#'s built-in querying language. If you know the **Java Streams API**, you already know 80% of LINQ — it's the same idea (declarative, chainable operations over a sequence) but baked directly into the C# language with extra syntax sugar.

The two pillars to understand:

1. **It's lazy (deferred).** Like Java streams, most LINQ operators build a *query*, not a *result*. Nothing runs until you iterate (`foreach`, `ToList()`, etc.).
2. **It targets two worlds.** **LINQ to Objects** runs in memory over `IEnumerable<T>` (exactly like Java streams over a `Collection`). **LINQ to Entities** (Entity Framework Core) translates your C# query into **SQL** that runs on the database — there is no Java Streams equivalent for this, and it's a top interview topic.

> **Think of it like...** Java Streams that Microsoft promoted from "a library" to "a first-class part of the language", then gave a second superpower: the same code can either run in memory *or* be translated into a SQL query.

This guide is junior-friendly. Every concept maps back to something you already do in Java.

---

## Table of Contents

- [Java Streams → LINQ Mapping](#java-streams--linq-mapping)
- [The Two Syntaxes: Method vs Query](#the-two-syntaxes-method-vs-query)
- [LINQ to Objects vs LINQ to Entities](#linq-to-objects-vs-linq-to-entities)
- [Filtering: Where, OfType](#filtering-where-oftype)
- [Projection: Select, SelectMany, Anonymous Types, DTOs](#projection-select-selectmany-anonymous-types-dtos)
- [Sorting: OrderBy, OrderByDescending, ThenBy](#sorting-orderby-orderbydescending-thenby)
- [Grouping: GroupBy, ToLookup](#grouping-groupby-tolookup)
- [Joining: Join, GroupJoin](#joining-join-groupjoin)
- [Aggregation: Count, Sum, Average, Min, Max, Aggregate](#aggregation-count-sum-average-min-max-aggregate)
- [Element Operators and Why ...OrDefault Matters](#element-operators-and-why-ordefault-matters)
- [Quantifiers: Any, All, Contains](#quantifiers-any-all-contains)
- [Partitioning: Take, Skip, TakeWhile, SkipWhile, Chunk](#partitioning-take-skip-takewhile-skipwhile-chunk)
- [Set Operations: Distinct, DistinctBy, Union, Intersect, Except](#set-operations-distinct-distinctby-union-intersect-except)
- [Conversion: ToList, ToArray, ToDictionary, ToHashSet](#conversion-tolist-toarray-todictionary-tohashset)
- [Deferred vs Immediate Execution](#deferred-vs-immediate-execution)
- [IEnumerable vs IQueryable](#ienumerable-vs-iqueryable)
- [Performance Pitfalls](#performance-pitfalls)
- [let, into, and Nested Queries](#let-into-and-nested-queries)
- [Common Interview Questions](#common-interview-questions)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Java Streams → LINQ Mapping

This is your Rosetta Stone. Bookmark it.

| Java Streams | LINQ (method syntax) | Notes |
|---|---|---|
| `stream.filter(p)` | `.Where(p)` | Keep elements matching predicate |
| `stream.map(f)` | `.Select(f)` | Transform each element |
| `stream.flatMap(f)` | `.SelectMany(f)` | Flatten nested sequences |
| `stream.collect(toList())` | `.ToList()` | Materialize to a `List<T>` |
| `stream.collect(toSet())` | `.ToHashSet()` | Materialize to a `HashSet<T>` |
| `stream.reduce(...)` | `.Aggregate(...)` | Fold into a single value |
| `stream.sorted()` | `.OrderBy(x => x)` | Ascending sort |
| `stream.sorted(cmp.reversed())` | `.OrderByDescending(...)` | Descending sort |
| `.thenComparing(...)` | `.ThenBy(...)` | Secondary sort key |
| `stream.distinct()` | `.Distinct()` | Remove duplicates |
| `stream.limit(n)` | `.Take(n)` | First `n` elements |
| `stream.skip(n)` | `.Skip(n)` | Drop first `n` elements |
| `stream.takeWhile(p)` | `.TakeWhile(p)` | Take until predicate fails |
| `stream.dropWhile(p)` | `.SkipWhile(p)` | Drop until predicate fails |
| `stream.anyMatch(p)` | `.Any(p)` | True if any match |
| `stream.allMatch(p)` | `.All(p)` | True if all match |
| `stream.noneMatch(p)` | `!.Any(p)` | True if none match |
| `stream.count()` | `.Count()` | Number of elements |
| `stream.findFirst()` | `.FirstOrDefault()` | First, or default if empty |
| `stream.findAny()` | `.FirstOrDefault()` | Same idea (LINQ is ordered) |
| `collect(groupingBy(f))` | `.GroupBy(f)` | Group by a key |
| `collect(toMap(k, v))` | `.ToDictionary(k, v)` | Build a key/value map |
| `mapToInt(...).sum()` | `.Sum(...)` | Sum a numeric projection |
| `.average()` | `.Average(...)` | Mean |
| `.min()/.max()` | `.Min(...)/.Max(...)` | Extremes |
| `IntStream.range(a,b)` | `Enumerable.Range(a, count)` | Generate a number range |
| `stream.peek(...)` | *(no direct equiv)* | Use a side-effecting `Select` sparingly |
| `Collectors.joining(",")` | `string.Join(",", seq)` | Join strings |

**Key naming difference:** Java verbs are imperative (`filter`, `map`); LINQ uses SQL-flavored names (`Where`, `Select`) because LINQ is modeled on SQL.

---

## The Two Syntaxes: Method vs Query

LINQ has **two interchangeable syntaxes** that compile to the exact same thing. Java only has the method one.

```csharp
int[] numbers = { 5, 3, 8, 1, 9, 2 };

// 1) METHOD (fluent) syntax — this is what looks like Java Streams.
var resultMethod = numbers
    .Where(n => n > 2)          // Java: .filter(n -> n > 2)
    .OrderBy(n => n)            // Java: .sorted()
    .Select(n => n * 10);       // Java: .map(n -> n * 10)

// 2) QUERY syntax — reads like SQL, no Java equivalent at all.
var resultQuery =
    from n in numbers           // 'from' is like SQL FROM / a foreach header
    where n > 2                 // where == .Where(...)
    orderby n                   // orderby == .OrderBy(...)
    select n * 10;              // select == .Select(...)

// Both produce the SAME query. The compiler rewrites query syntax into method calls.
```

**Think of it like...** two dialects for the same instructions. Query syntax is just sugar — under the hood the compiler turns `from/where/select` into `.Where().Select()`.

**Which to use?**
- **Method syntax**: most common, more operators available (e.g. `Count`, `ToList`, `Take` have no keyword). Use it 90% of the time.
- **Query syntax**: shines for **joins**, **grouping**, and **`let`** (intermediate variables), where it reads more cleanly.

Note `var` just means "let the compiler infer the type" (like Java's `var`). The type here is `IEnumerable<int>`.

---

## LINQ to Objects vs LINQ to Entities

| | LINQ to Objects | LINQ to Entities (EF Core) |
|---|---|---|
| Source type | `IEnumerable<T>` (in-memory list/array) | `IQueryable<T>` (a DB table via `DbSet<T>`) |
| Where it runs | In your app's RAM | **Translated to SQL**, runs on the database |
| Java analogy | Java Streams over a `List` | (no Java equivalent — like Streams that compile to SQL) |
| Lambda type | A real compiled delegate | An **expression tree** (a data structure describing the lambda, so it can become SQL) |

```csharp
// LINQ to Objects — runs in memory, just like Java streams.
List<Person> people = GetPeopleList();
var adults = people.Where(p => p.Age >= 18).ToList();   // filter happens in C#

// LINQ to Entities — looks identical, but EF Core turns it into:
//   SELECT * FROM People WHERE Age >= 18
var adultsFromDb = dbContext.People
    .Where(p => p.Age >= 18)   // NOT run in C#; becomes a SQL WHERE clause
    .ToList();                 // SQL executes here, rows come back
```

The big takeaway: **the same query code can run in two completely different places.** What you can write differs too (see [IEnumerable vs IQueryable](#ienumerable-vs-iqueryable)).

---

## Filtering: Where, OfType

**Think of it like...** `stream.filter()` — keep only what passes.

```csharp
var nums = new[] { 1, 2, 3, 4, 5, 6 };

// Where: keep elements matching a predicate.
var evens = nums.Where(n => n % 2 == 0);          // Java: nums.filter(n -> n % 2 == 0)

// Where with index (the second arg is the position) — Java streams can't do this directly.
var everyOther = nums.Where((n, index) => index % 2 == 0);

// OfType: filter a mixed collection by runtime type AND cast survivors.
object[] mixed = { 1, "hello", 2, "world", 3.5 };
var onlyStrings = mixed.OfType<string>();          // yields "hello", "world"
// Java equivalent: .filter(x -> x instanceof String).map(x -> (String) x)
```

`Where` keeps the same type; `OfType<T>` both filters and casts.

---

## Projection: Select, SelectMany, Anonymous Types, DTOs

**Think of it like...** `stream.map()` (Select) and `stream.flatMap()` (SelectMany).

```csharp
var people = new[]
{
    new Person("Ana",  new[] { "C#", "SQL" }),
    new Person("Ben",  new[] { "Java", "C#" })
};

// Select: transform each element (1-to-1).
var names = people.Select(p => p.Name);            // Java: .map(Person::getName)

// Select with index.
var numbered = people.Select((p, i) => $"{i}: {p.Name}");

// SelectMany: flatten nested collections (1-to-many → one flat sequence).
var allSkills = people.SelectMany(p => p.Skills);  // Java: .flatMap(p -> p.skills.stream())
// → "C#", "SQL", "Java", "C#"

// Anonymous types: build a throwaway shape on the fly (Java has no clean equivalent).
var summary = people.Select(p => new { p.Name, SkillCount = p.Skills.Length });
// 'summary' items have .Name and .SkillCount, compiler-generated type.

// Projecting to a DTO (the production pattern — return a defined class, not the entity).
var dtos = people.Select(p => new PersonDto
{
    Name = p.Name,
    SkillCount = p.Skills.Length
});

public record PersonDto { public string Name { get; init; } public int SkillCount { get; init; } }
public record Person(string Name, string[] Skills);
```

**Why project to DTOs in EF Core?** Selecting only the columns you need (`Select(p => new PersonDto {...})`) makes EF Core generate `SELECT Name, ...` instead of `SELECT *`, and avoids over-fetching. Big performance win and a common interview point.

---

## Sorting: OrderBy, OrderByDescending, ThenBy

**Think of it like...** `sorted(Comparator.comparing(...).thenComparing(...))`.

```csharp
var people = GetPeople();

// Single key ascending.
var byAge = people.OrderBy(p => p.Age);                  // Java: sorted(comparing(Person::getAge))

// Descending.
var byAgeDesc = people.OrderByDescending(p => p.Age);

// Multi-level sort: OrderBy first, then ThenBy for ties.
var sorted = people
    .OrderBy(p => p.LastName)          // primary key
    .ThenBy(p => p.FirstName)          // tie-breaker  (Java: .thenComparing)
    .ThenByDescending(p => p.Age);     // further tie-breaker, descending
```

**Gotcha:** You must use `ThenBy` for secondary keys — chaining a second `OrderBy` *re-sorts from scratch* and discards the first ordering. (`OrderBy` returns an `IOrderedEnumerable`, which `ThenBy` extends.)

`Reverse()` exists too but just flips the current order; it is not a sort.

---

## Grouping: GroupBy, ToLookup

**Think of it like...** `Collectors.groupingBy()`.

```csharp
var people = GetPeople();

// GroupBy: returns a sequence of IGrouping<TKey, TElement>.
// Each group has a .Key and is itself an IEnumerable of its members.
var byCity = people.GroupBy(p => p.City);

foreach (var group in byCity)
{
    Console.WriteLine($"{group.Key}: {group.Count()} people");  // group.Key is the city
}

// GroupBy with a result projection (group → summary object).
var cityStats = people
    .GroupBy(p => p.City)
    .Select(g => new { City = g.Key, Count = g.Count(), AvgAge = g.Average(x => x.Age) });

// Query syntax version (often clearer for grouping):
var bySalaryBand =
    from p in people
    group p by (p.Salary / 10000) into band   // 'into' names the group
    select new { Band = band.Key, People = band.ToList() };
```

**GroupBy vs ToLookup:**

```csharp
// GroupBy is DEFERRED (lazy) — like a stream, nothing runs until iterated.
var lazy = people.GroupBy(p => p.City);

// ToLookup is IMMEDIATE — executes now and returns a queryable, indexable structure.
ILookup<string, Person> lookup = people.ToLookup(p => p.City);
var londoners = lookup["London"];   // O(1) lookup; missing key returns EMPTY (not an exception)
```

**Think of `ToLookup` like** a `Map<K, List<V>>` you build eagerly with `groupingBy`, except missing keys safely return an empty sequence instead of `null`.

---

## Joining: Join, GroupJoin

**Think of it like...** SQL joins (Java Streams has no built-in join — you'd write a manual map+lookup).

```csharp
var customers = GetCustomers();   // each has Id, Name
var orders    = GetOrders();      // each has CustomerId, Amount

// Join = INNER JOIN. Matches on keys, produces one row per matching pair.
var customerOrders = customers.Join(
    orders,                       // the inner sequence
    c => c.Id,                    // outer key selector
    o => o.CustomerId,            // inner key selector
    (c, o) => new { c.Name, o.Amount });   // result selector for each matched pair

// Query syntax — much more readable for joins:
var joined =
    from c in customers
    join o in orders on c.Id equals o.CustomerId   // 'equals' is required (not '==')
    select new { c.Name, o.Amount };

// GroupJoin = LEFT-style: each outer element gets the GROUP of its matches (possibly empty).
var withOrders = customers.GroupJoin(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (c, matchingOrders) => new { c.Name, OrderCount = matchingOrders.Count() });

// LEFT OUTER JOIN: GroupJoin + SelectMany + DefaultIfEmpty.
var leftJoin =
    from c in customers
    join o in orders on c.Id equals o.CustomerId into custOrders
    from o in custOrders.DefaultIfEmpty()          // keeps customers with no orders
    select new { c.Name, Amount = o?.Amount ?? 0 };
```

In **EF Core** you rarely write `Join` manually — navigation properties (`customer.Orders`) and `Include()` handle relationships. But `Join` knowledge is still interview gold.

---

## Aggregation: Count, Sum, Average, Min, Max, Aggregate

**Think of it like...** the terminal reductions on a stream.

```csharp
var nums = new[] { 4, 8, 15, 16, 23, 42 };

int   count = nums.Count();                 // Java: .count()
int   total = nums.Sum();                   // Java: .sum()
double avg  = nums.Average();               // Java: .average().getAsDouble()
int   min   = nums.Min();                   // Java: .min().get()
int   max   = nums.Max();                   // Java: .max().get()

// Count with a predicate (filter + count in one).
int bigCount = nums.Count(n => n > 15);     // Java: .filter(n -> n > 15).count()

// Sum/Min/Max with a selector.
var people = GetPeople();
int totalAge = people.Sum(p => p.Age);
var oldest   = people.MaxBy(p => p.Age);    // returns the PERSON (not the age). Java: maxBy comparator

// Aggregate = reduce. Fold the sequence into one value.
int product = nums.Aggregate((acc, n) => acc * n);            // Java: reduce((a,b) -> a*b)

// Aggregate with a seed (initial accumulator) — like Java's 3-arg reduce.
string csv = nums.Aggregate("", (acc, n) => acc + n + ",");   // build a string
```

`MaxBy`/`MinBy` (.NET 6+) return the *element* with the max/min key — handy and clearer than `OrderByDescending().First()`.

---

## Element Operators and Why ...OrDefault Matters

**Think of it like...** `findFirst()` returning an `Optional`, except LINQ uses two method *variants* instead of `Optional`.

```csharp
var nums = new[] { 10, 20, 30 };

// First: returns first element. THROWS InvalidOperationException if empty.
int a = nums.First();                       // 10
int b = nums.First(n => n > 15);            // 20  (first matching predicate)

// FirstOrDefault: returns first, or default(T) if none. NO exception.
int c = nums.FirstOrDefault(n => n > 100);  // 0   (default for int)
Person? p = people.FirstOrDefault(x => x.Name == "Zoe");  // null for reference types

// Single: expects EXACTLY ONE match. Throws if zero OR more than one.
int d = nums.Single(n => n == 20);          // 20
// nums.Single(n => n > 15) would THROW — two matches (20 and 30).

// SingleOrDefault: zero → default; exactly one → it; >1 → still THROWS.
var maybe = people.SingleOrDefault(x => x.Id == 42);

int last = nums.Last();                      // 30 (throws if empty)
int third = nums.ElementAt(2);               // 30 (throws if out of range)
int safe  = nums.ElementAtOrDefault(99);     // 0  (no exception)
```

**Why `...OrDefault` matters (interview answer):**
- `First()`/`Single()` **throw** on empty/missing — use when absence is a bug.
- `FirstOrDefault()`/`SingleOrDefault()` **return `null`/default** — use when "not found" is normal. This is the LINQ stand-in for Java's `Optional.orElse(null)`.
- **`Single` vs `First`:** `Single` asserts uniqueness (it scans for a *second* match). Use `Single` for "find by primary key — there must be exactly one"; use `First` for "give me any one." Don't use `Single` on large unfiltered sets — it can't short-circuit.

---

## Quantifiers: Any, All, Contains

**Think of it like...** `anyMatch`, `allMatch`, and `contains`.

```csharp
var nums = new[] { 2, 4, 6, 8 };

bool hasAny   = nums.Any();                  // is the sequence non-empty? (cheap — checks one item)
bool anyOdd   = nums.Any(n => n % 2 == 1);   // Java: .anyMatch(n -> n % 2 == 1)  → false
bool allEven  = nums.All(n => n % 2 == 0);   // Java: .allMatch(...)              → true
bool hasSix   = nums.Contains(6);            // Java: list.contains(6)            → true
```

**Performance tip (very common interview catch):** use `list.Any()` instead of `list.Count() > 0`. `Count()` may walk the *entire* sequence; `Any()` stops at the first element. Same logic for `Any(predicate)` over `Where(predicate).Count() > 0`.

---

## Partitioning: Take, Skip, TakeWhile, SkipWhile, Chunk

**Think of it like...** `limit`, `skip`, `takeWhile`, `dropWhile`.

```csharp
var nums = new[] { 1, 2, 3, 4, 5, 6, 7, 8 };

var firstThree = nums.Take(3);          // 1,2,3      Java: .limit(3)
var afterThree = nums.Skip(3);          // 4,5,6,7,8  Java: .skip(3)

// Paging idiom: Skip then Take.
int page = 2, size = 3;
var pageItems = nums.Skip((page - 1) * size).Take(size);   // items 4,5,6

// Take a RANGE (.NET 6+) using the ^ "from end" and .. range syntax.
var middle = nums.Take(2..5);           // elements at index 2,3,4 → 3,4,5
var lastTwo = nums.TakeLast(2);         // 7,8
var skipLastTwo = nums.SkipLast(2);     // 1..6

// Conditional partitioning — stops at the first element that fails/passes.
var whileSmall = nums.TakeWhile(n => n < 4);   // 1,2,3        Java: .takeWhile
var dropSmall  = nums.SkipWhile(n => n < 4);   // 4,5,6,7,8    Java: .dropWhile

// Chunk: split into batches of N (.NET 6+). Great for batching DB writes.
var batches = nums.Chunk(3);            // [1,2,3], [4,5,6], [7,8]
```

---

## Set Operations: Distinct, DistinctBy, Union, Intersect, Except

**Think of it like...** `distinct()` plus Java `Set` algebra (`retainAll`, `addAll`, `removeAll`).

```csharp
var a = new[] { 1, 2, 2, 3, 4 };
var b = new[] { 3, 4, 5 };

var unique = a.Distinct();              // 1,2,3,4        Java: .distinct()

// DistinctBy (.NET 6+): dedupe by a key, keep the first of each. No Java Streams equivalent.
var people = GetPeople();
var onePerCity = people.DistinctBy(p => p.City);

var union     = a.Union(b);             // 1,2,3,4,5  (distinct union)   Java: addAll + dedupe
var intersect = a.Intersect(b);         // 3,4        (in both)          Java: retainAll
var except    = a.Except(b);            // 1,2        (in a, not in b)   Java: removeAll
var combined  = a.Concat(b);            // 1,2,2,3,4,3,4,5  (keeps dups) Java: Stream.concat
```

`Union`/`Intersect`/`Except` automatically remove duplicates; `Concat` does **not** — it's a raw append.

---

## Conversion: ToList, ToArray, ToDictionary, ToHashSet

**Think of it like...** the `collect(...)` terminal operations. **These force execution** (materialize the query now).

```csharp
var people = GetPeople();

List<Person>   list  = people.ToList();        // Java: collect(toList())
Person[]       array = people.ToArray();        // Java: toArray(Person[]::new)
HashSet<string> set  = people.Select(p => p.City).ToHashSet();  // collect(toSet())

// ToDictionary: build a key→value map. Java: collect(toMap(keyFn, valueFn)).
Dictionary<int, Person> byId = people.ToDictionary(p => p.Id);          // value defaults to element
Dictionary<int, string> idToName = people.ToDictionary(p => p.Id, p => p.Name);
// ⚠ Duplicate keys THROW (like toMap without a merge function). Ensure keys are unique.

// ToLookup when keys are NOT unique (one key → many values) — won't throw.
ILookup<string, Person> byCity = people.ToLookup(p => p.City);
```

Calling one of these is how you "leave LINQ" and get a concrete, reusable collection. It also fixes the [multiple-enumeration](#performance-pitfalls) problem.

---

## Deferred vs Immediate Execution

**This is the #1 LINQ interview topic.** Java streams are *single-use and lazy*; LINQ queries are *reusable but lazy* — and the reusability is a trap.

```csharp
var nums = new List<int> { 1, 2, 3 };

// Build a query — NOTHING runs yet. 'query' is a recipe, not a result.
var query = nums.Where(n => n > 1);    // deferred: no iteration happened

nums.Add(4);                           // mutate the SOURCE after defining the query

// Execution happens HERE, when we iterate. The query sees the CURRENT source.
foreach (var n in query)
    Console.WriteLine(n);              // prints 2, 3, AND 4  ← surprises juniors!
```

**Deferred (lazy) operators** build the pipeline: `Where`, `Select`, `OrderBy`, `Take`, `Skip`, `GroupBy`, `SelectMany`, etc.

**Immediate operators** run the pipeline right now and return a value/collection:
- Anything returning a single value: `Count`, `Sum`, `First`, `Any`, `Max`, `Aggregate`...
- Anything starting with `To`: `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`, `ToLookup`.

```csharp
// FORCE immediate execution to snapshot the result NOW.
var snapshot = nums.Where(n => n > 1).ToList();   // runs immediately, captures current state
nums.Add(99);                                     // does NOT affect 'snapshot'
```

**Vs Java:** A Java stream throws `IllegalStateException` if you reuse it. A LINQ query is *reusable* — but each iteration **re-executes from scratch** against the live source. Same laziness, different reuse rules.

---

## IEnumerable vs IQueryable

**The other top interview topic.** Both are "a lazy sequence", but they execute in fundamentally different places.

| | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| Executes | **In memory** (your C# process) | **Remotely** — EF Core translates to SQL |
| Lambda stored as | Compiled delegate (`Func<>`) | **Expression tree** (`Expression<Func<>>`) |
| Used by | LINQ to Objects | LINQ to Entities (databases) |
| Best for | Already-in-memory collections | Querying a DB efficiently |

```csharp
// IQueryable — EF Core inspects the expression tree and builds SQL.
IQueryable<Person> q = dbContext.People;
var adults = q.Where(p => p.Age >= 18)      // becomes: WHERE Age >= 18
              .Select(p => p.Name)          // becomes: SELECT Name
              .ToList();                    // SQL runs ONCE, returns just adult names

// THE CLASSIC BUG: calling AsEnumerable() / ToList() too early.
var bad = dbContext.People
    .AsEnumerable()                 // ← switches to IEnumerable HERE
    .Where(p => p.Age >= 18)        // now this filter runs IN C#, after loading EVERY row!
    .ToList();                      // SELECT * FROM People  (entire table pulled into memory)
```

**Interview soundbite:** *"`IQueryable` builds an expression tree so the provider (EF Core) can translate the whole query to SQL and filter on the database. `IEnumerable` runs the query in memory. Switching to `IEnumerable` too early (via `AsEnumerable`/`ToList`) drags the whole table into RAM before filtering."*

**Caveat:** `IQueryable` can only contain operations the provider can translate. Calling your own C# method inside an EF `Where` throws at runtime because it can't become SQL — that's when you intentionally `AsEnumerable()` *after* the DB-side filtering.

---

## Performance Pitfalls

### 1. Multiple enumeration

```csharp
var query = dbContext.People.Where(p => p.Age > 18);   // deferred

// BAD: each call re-runs the whole query (re-hits the DB!).
if (query.Any())                       // query #1
    Console.WriteLine(query.Count());  // query #2
var list = query.ToList();             // query #3

// GOOD: materialize ONCE, reuse the result.
var people = dbContext.People.Where(p => p.Age > 18).ToList();
if (people.Any()) Console.WriteLine(people.Count);     // in-memory, no extra DB calls
```

Tooling (ReSharper/Rider) literally warns "Possible multiple enumeration of IEnumerable."

### 2. N+1 queries in EF Core

```csharp
// BAD: 1 query for blogs + 1 query PER blog for its posts = N+1 round trips.
var blogs = dbContext.Blogs.ToList();
foreach (var blog in blogs)
    Console.WriteLine(blog.Posts.Count);   // lazy-loads Posts → a new query each loop

// GOOD: eager-load related data in ONE query with Include().
var blogs2 = dbContext.Blogs.Include(b => b.Posts).ToList();   // single JOIN query
```

### 3. Materializing too early / over-fetching

```csharp
// BAD: pulls full entities, then throws most data away.
var names = dbContext.People.ToList().Select(p => p.Name).ToList();   // SELECT * then map

// GOOD: project in the DB so only the Name column travels.
var names2 = dbContext.People.Select(p => p.Name).ToList();           // SELECT Name
```

**Rule of thumb:** keep the query as `IQueryable` (filter, project, page) for as long as possible, then call `ToList()` **once** at the very end.

---

## let, into, and Nested Queries

Query syntax has tools method syntax lacks cleanly.

```csharp
var people = GetPeople();

// 'let' introduces an intermediate computed variable (avoids recomputing).
var query =
    from p in people
    let fullName = $"{p.FirstName} {p.LastName}"   // computed once, reusable below
    let initials = $"{p.FirstName[0]}{p.LastName[0]}"
    where fullName.Length > 10
    orderby p.LastName
    select new { fullName, initials };
// In method syntax you'd need a Select to a temp anonymous type to mimic 'let'.

// 'into' pipes the result of one clause into a new query (continuation).
var grouped =
    from p in people
    group p by p.City into cityGroup        // 'into' names the grouped result
    where cityGroup.Count() > 1             // now query OVER the groups
    orderby cityGroup.Key
    select new { City = cityGroup.Key, Count = cityGroup.Count() };

// Nested query: a sub-query inside select (correlated).
var nested =
    from p in people
    select new
    {
        p.Name,
        Peers = (from q in people
                 where q.City == p.City && q.Name != p.Name
                 select q.Name).ToList()    // people in the same city
    };
```

**Think of `let` like** a local variable inside a stream pipeline — Java forces you to inline or use a helper; query syntax gives you a named slot.

---

## Common Interview Questions

**1. What is LINQ, and what does "language integrated" mean?**
A unified, declarative query syntax built into C#. You query in-memory collections, databases, XML, etc. with the *same operators*. "Integrated" = it's part of the language (keywords like `from`/`where`/`select`) and is type-checked by the compiler, not strings.

**2. Difference between deferred and immediate execution?**
Deferred operators (`Where`, `Select`, `OrderBy`) build a query but don't run until iterated (`foreach`, or an immediate operator). Immediate operators (`ToList`, `Count`, `First`, `Sum`) execute right away. Deferred queries re-execute every time you enumerate them, reflecting the current source state.

**3. `IEnumerable` vs `IQueryable` — when do you use each?**
`IEnumerable<T>` executes in memory (LINQ to Objects). `IQueryable<T>` stores the query as an expression tree so a provider like EF Core can translate it to SQL and execute on the database. Use `IQueryable` for DB queries (filter server-side); use `IEnumerable` for already-loaded data. Switching to `IEnumerable` too early pulls the whole table into memory.

**4. `First` vs `FirstOrDefault` vs `Single` vs `SingleOrDefault`?**
`First` returns the first match, throws if none. `FirstOrDefault` returns the first or `default(T)`/`null`. `Single` requires exactly one match (throws on zero *or* more than one). `SingleOrDefault` allows zero (returns default) but throws on more than one. Use `Single` to assert uniqueness (e.g., lookup by primary key); use `First` when you just want any.

**5. `Select` vs `SelectMany`?**
`Select` is 1-to-1 (`map`). `SelectMany` flattens nested sequences — 1-to-many merged into one flat sequence (`flatMap`). Use `SelectMany` when each element yields a collection you want flattened.

**6. Why is `.Any()` preferred over `.Count() > 0`?**
`Any()` short-circuits at the first element. `Count()` may enumerate the entire sequence (or run a `COUNT(*)` query). `Any()` is cheaper for checking existence.

**7. What is the N+1 problem in EF Core and how do you fix it?**
Loading a parent list (1 query) then accessing a navigation property per item (N queries) = N+1 round trips. Fix with eager loading: `.Include(x => x.Children)` to fetch related data in a single query.

**8. `GroupBy` vs `ToLookup`?**
Both group by key. `GroupBy` is deferred (lazy). `ToLookup` executes immediately and returns an indexable `ILookup<K,V>` where a missing key safely returns an empty sequence (no exception).

**9. What is "multiple enumeration" and why is it bad?**
Enumerating the same deferred query more than once re-runs the entire pipeline each time — repeated work, and repeated DB calls for `IQueryable`. Fix by materializing once with `ToList()`/`ToArray()`.

**10. Method syntax vs query syntax — is there a performance difference?**
No. Query syntax is compiled into the same method calls. Choose by readability: query syntax for joins/grouping/`let`; method syntax for everything else and operators with no keyword (`Count`, `Take`, `ToList`).

**11. What happens if `ToDictionary` hits a duplicate key?**
It throws `ArgumentException`. Ensure the key selector is unique, or use `ToLookup` (key → many values) or `GroupBy().ToDictionary(g => g.Key, g => g.ToList())`.

**12. How is LINQ deferred execution different from Java streams?**
Both are lazy. But a Java stream is **single-use** — reusing it throws. A LINQ query is **reusable**, and each enumeration re-executes against the (possibly changed) live source. So reuse is allowed but can be a subtle bug.

**13. What's an expression tree and why does `IQueryable` need one?**
An expression tree is a data structure representing code as data (the lambda's *structure*, not compiled IL). EF Core walks that tree to generate equivalent SQL. `IEnumerable` uses compiled delegates that can only run in memory, so they can't be translated to SQL.

---

## Quick Reference Cheat Sheet

```text
=== FILTER / PROJECT ===
.Where(p)                  filter            (Java filter)
.OfType<T>()               filter+cast by type
.Select(f)                 map               (Java map)
.SelectMany(f)             flatMap           (Java flatMap)
.Select((x,i) => ...)      map with index

=== SORT ===
.OrderBy(k) / .OrderByDescending(k)
.ThenBy(k) / .ThenByDescending(k)            secondary keys (NOT a 2nd OrderBy)
.Reverse()                 flip current order (not a sort)

=== GROUP / JOIN ===
.GroupBy(k)                deferred → IGrouping<K,T> (has .Key)
.ToLookup(k)               immediate → ILookup, missing key = empty
.Join(inner,ok,ik,res)     INNER JOIN
.GroupJoin(...)            LEFT-style (group of matches)
query: join b in bs on a.X equals b.Y

=== AGGREGATE ===
.Count() / .Count(p)       (use .Any() to test existence!)
.Sum() .Average() .Min() .Max()
.MaxBy(k) / .MinBy(k)      element with max/min key (.NET 6+)
.Aggregate(seed, fn)       reduce            (Java reduce)

=== ELEMENT (throwing vs safe) ===
.First() / .FirstOrDefault()      throws / null-or-default
.Single() / .SingleOrDefault()    exactly-one / allow-zero
.Last() .ElementAt(i) .ElementAtOrDefault(i)

=== QUANTIFIERS ===
.Any() .Any(p)             non-empty / any match   (Java anyMatch)
.All(p)                    all match               (Java allMatch)
.Contains(x)               membership

=== PARTITION ===
.Take(n) .Skip(n)          (Java limit / skip)
.TakeWhile(p) .SkipWhile(p)
.TakeLast(n) .SkipLast(n) .Chunk(n)
paging: .Skip((page-1)*size).Take(size)

=== SET ===
.Distinct() .DistinctBy(k)
.Union(b) .Intersect(b) .Except(b)   (dedupe)
.Concat(b)                            (keeps dups)

=== MATERIALIZE (immediate) ===
.ToList() .ToArray() .ToHashSet()
.ToDictionary(k[,v])       (dup key THROWS)

=== EXECUTION RULES ===
Deferred:  Where Select OrderBy Take Skip GroupBy SelectMany ...
Immediate: ToList ToArray Count Sum First Any Max Aggregate ...
IEnumerable -> runs in memory
IQueryable  -> EF Core translates to SQL (keep it IQueryable, ToList() ONCE at the end)

=== PITFALLS ===
- Multiple enumeration  -> materialize once with ToList()
- N+1 in EF Core        -> .Include(x => x.Children)
- Over-fetching         -> .Select(DTO) BEFORE ToList()
- .Count()>0            -> use .Any()
- AsEnumerable() early  -> pulls whole table into memory
```

*Last Updated: 2026-06-16*

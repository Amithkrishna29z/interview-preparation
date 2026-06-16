# C# Collections — A Study Guide for Java Developers

## Overview

If you know the Java Collections Framework, you already understand 90% of C# collections — the concepts map almost one-to-one. The biggest differences are **naming** (`List<T>` instead of `ArrayList`, `Dictionary` instead of `HashMap`), **interfaces** (`IEnumerable<T>` is the root, not `Iterable`), and a few **C#-specific patterns** like `TryGetValue` and `yield return`.

This guide is written for a junior developer who knows Java and wants a junior .NET/C# job. Every concept is mapped back to its Java equivalent, with a "Think of it like..." analogy and runnable, commented C# code (.NET 8+).

The single most important mental model: in Java, `Collection` is the root interface. In C#, **`IEnumerable<T>` is the root** — everything you can `foreach` over is an `IEnumerable<T>`. Build from there.

## Table of Contents

- [Java → C# Collections Mapping](#java--c-collections-mapping)
- [Arrays (Fixed Size)](#arrays-fixed-size)
- [List&lt;T&gt; (vs ArrayList)](#listt-vs-arraylist)
- [Dictionary&lt;TKey,TValue&gt; (vs HashMap)](#dictionarytkeytvalue-vs-hashmap)
- [HashSet&lt;T&gt; and SortedSet&lt;T&gt; (vs HashSet/TreeSet)](#hashsett-and-sortedsett-vs-hashsettreeset)
- [SortedDictionary and SortedList (vs TreeMap)](#sorteddictionary-and-sortedlist-vs-treemap)
- [Queue&lt;T&gt;, Stack&lt;T&gt;, LinkedList&lt;T&gt;](#queuet-stackt-linkedlistt)
- [The Interface Hierarchy](#the-interface-hierarchy)
- [Read-Only and Immutable Collections](#read-only-and-immutable-collections)
- [Concurrent Collections](#concurrent-collections)
- [Custom Equality and Ordering (IEqualityComparer / IComparer)](#custom-equality-and-ordering-iequalitycomparer--icomparer)
- [Iterating: foreach, Iterators, yield return](#iterating-foreach-iterators-yield-return)
- [Collection Initializers and Collection Expressions](#collection-initializers-and-collection-expressions)
- [Big-O and When to Use Which Collection](#big-o-and-when-to-use-which-collection)
- [Common Pitfalls](#common-pitfalls)
- [Common Interview Questions](#common-interview-questions)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

## Java → C# Collections Mapping

| Java | C# (.NET) | Notes |
|------|-----------|-------|
| `T[]` | `T[]` | Fixed-size array, same idea |
| `ArrayList<T>` / `List<T>` | `List<T>` | Resizable array-backed list |
| `HashMap<K,V>` | `Dictionary<TKey,TValue>` | Hash-based map, no order guarantee |
| `LinkedHashMap<K,V>` | *(no direct equivalent)* | Dictionary preserves insertion order in practice but **don't rely on it** |
| `TreeMap<K,V>` | `SortedDictionary<K,V>` / `SortedList<K,V>` | Sorted by key |
| `HashSet<T>` | `HashSet<T>` | Hash-based set |
| `LinkedHashSet<T>` | *(no direct equivalent)* | Use `HashSet` + separate order tracking if needed |
| `TreeSet<T>` | `SortedSet<T>` | Sorted set |
| `LinkedList<T>` | `LinkedList<T>` | Doubly-linked list |
| `ArrayDeque<T>` (as queue) | `Queue<T>` | FIFO |
| `ArrayDeque<T>` (as stack) | `Stack<T>` | LIFO |
| `PriorityQueue<T>` | `PriorityQueue<TElement,TPriority>` | Min-heap by default (.NET 6+) |
| `Iterable<T>` | `IEnumerable<T>` | **Root interface — the big one** |
| `Iterator<T>` | `IEnumerator<T>` | Cursor over a sequence |
| `Collection<T>` | `ICollection<T>` | Adds Count, Add, Remove |
| `List<T>` (interface) | `IList<T>` | Indexable |
| `Map<K,V>` | `IDictionary<TKey,TValue>` | Key-value interface |
| `Collections.unmodifiableList` | `IReadOnlyList<T>` / `ReadOnlyCollection<T>` | Read-only view |
| `List.of(...)` (immutable) | `ImmutableList<T>` | Truly immutable copy |
| `ConcurrentHashMap` | `ConcurrentDictionary<K,V>` | Thread-safe map |
| `ConcurrentLinkedQueue` | `ConcurrentQueue<T>` | Thread-safe FIFO |
| `BlockingQueue` | `BlockingCollection<T>` | Producer/consumer with blocking |
| `Comparator<T>` | `IComparer<T>` | Custom ordering |
| `equals()` + `hashCode()` | `IEqualityComparer<T>` (or override `Equals`/`GetHashCode`) | Custom equality |
| `stream()` | LINQ (`using System.Linq;`) | `.Where()`, `.Select()`, etc. |

> Generics note: Java erases generics at runtime; C# **reified generics** keep the type at runtime, so `List<int>` stores actual `int` values (no boxing) — faster and more memory-efficient than Java's `List<Integer>`.

## Arrays (Fixed Size)

**Think of it like...** a row of numbered lockers bolted to the wall. The count is fixed at install time; you can swap what's inside each locker but you can't add a locker.

```csharp
// Java: int[] nums = new int[5];  same syntax, different placement of []
int[] nums = new int[5];          // all elements default to 0
int[] primes = { 2, 3, 5, 7 };    // literal initializer
int[] more   = new int[] { 1, 2 };// explicit form

nums[0] = 42;                     // index access, identical to Java
Console.WriteLine(nums.Length);   // 5 — NOTE: .Length (property), Java uses .length (field)

// Arrays are fixed size: there is NO nums.Add(...). Need growth? Use List<T>.
```

### Multidimensional vs Jagged Arrays

This is one place C# differs from Java. C# has **true multidimensional arrays** (a single rectangular block) AND **jagged arrays** (arrays of arrays, which is all Java has).

```csharp
// MULTIDIMENSIONAL (rectangular) — single comma-separated index. Java has NO equivalent.
int[,] grid = new int[2, 3];      // 2 rows x 3 cols, one contiguous block
grid[0, 1] = 9;                   // single bracket, comma-separated
Console.WriteLine(grid.GetLength(0)); // 2 (rows)
Console.WriteLine(grid.GetLength(1)); // 3 (cols)

int[,] matrix = { { 1, 2, 3 }, { 4, 5, 6 } }; // initializer

// JAGGED (array of arrays) — this is exactly Java's int[][]
int[][] jagged = new int[2][];    // outer array of 2 references
jagged[0] = new int[] { 1, 2 };   // each row can have a different length
jagged[1] = new int[] { 3, 4, 5 };
Console.WriteLine(jagged[1][2]);  // 5 — double bracket like Java
```

> Rule of thumb: use `[,]` (multidimensional) when every row is the same length (a true matrix); use `[][]` (jagged) when rows vary. Jagged is often faster because each inner array is a normal 1-D array.

## List&lt;T&gt; (vs ArrayList)

**Think of it like...** an expandable shelf. Backed by an array under the hood; when it fills up it allocates a bigger array and copies everything over — exactly like Java's `ArrayList`.

`List<T>` is the workhorse. It is array-backed, gives O(1) index access, and grows automatically.

```csharp
using System.Collections.Generic;

var fruits = new List<string>();   // Java: new ArrayList<String>()
fruits.Add("apple");               // same as Java .add()
fruits.Add("banana");
fruits.Insert(0, "cherry");        // Java: .add(0, "cherry")

string first = fruits[0];          // INDEXER access — Java needs .get(0)
fruits[0] = "kiwi";                // assign by index — Java needs .set(0, "kiwi")

fruits.Remove("banana");           // remove by value, returns bool
fruits.RemoveAt(0);                // remove by index — Java: .remove(int)
Console.WriteLine(fruits.Count);   // .Count — Java uses .size()

bool has = fruits.Contains("kiwi");
int idx   = fruits.IndexOf("kiwi");

// Capacity vs Count: Count = elements you have; Capacity = slots allocated.
var nums = new List<int>(100);     // pre-size capacity to avoid re-allocations
Console.WriteLine(nums.Count);     // 0  (no elements yet)
Console.WriteLine(nums.Capacity);  // 100 (slots reserved) — Java ArrayList(100) is similar

// Bulk + LINQ (Java streams)
fruits.AddRange(new[] { "mango", "pear" });
var longOnes = fruits.Where(f => f.Length > 4).ToList(); // Java: stream().filter()...
```

> Key syntax wins vs Java: `list[i]` (indexer) instead of `get/set`, and `.Count` instead of `.size()`.

## Dictionary&lt;TKey,TValue&gt; (vs HashMap)

**Think of it like...** a coat-check counter: you hand over a ticket (key) and get back your coat (value). No guaranteed order of tickets.

`Dictionary<TKey,TValue>` is C#'s `HashMap`. The single most important thing to learn is the **`TryGetValue` pattern**, because the indexer throws if the key is missing.

```csharp
var ages = new Dictionary<string, int>(); // Java: new HashMap<String,Integer>()
ages["Alice"] = 30;                       // indexer ADD/UPDATE — Java: .put("Alice", 30)
ages["Bob"]   = 25;
ages.Add("Carol", 40);                    // .Add THROWS if key already exists (indexer doesn't)

// PITFALL: indexer read throws KeyNotFoundException if key is absent!
// int x = ages["Dave"];  //  THROWS — Java's HashMap.get() would return null instead

// THE IDIOMATIC PATTERN: TryGetValue (no exception, no double lookup)
if (ages.TryGetValue("Alice", out int aliceAge))
{
    Console.WriteLine(aliceAge);          // 30
}

// Safe membership + defaults
bool exists = ages.ContainsKey("Bob");    // Java: .containsKey()
int safe = ages.GetValueOrDefault("Zed", -1); // returns -1 if absent (Java: getOrDefault)

ages.Remove("Bob");                       // Java: .remove()
Console.WriteLine(ages.Count);            // .Count, not .size()

// Iteration yields KeyValuePair<K,V> (Java: Map.Entry<K,V>)
foreach (var kvp in ages)
{
    Console.WriteLine($"{kvp.Key} = {kvp.Value}");
}
foreach (var (name, age) in ages)         // tuple deconstruction (modern C#)
{
    Console.WriteLine($"{name} is {age}");
}
foreach (string key in ages.Keys) { }     // Java: .keySet()
foreach (int val in ages.Values) { }      // Java: .values()
```

> **Iteration order:** like Java's `HashMap`, `Dictionary` makes **no ordering guarantee**. It often looks insertion-ordered until you remove items, then the order can change. Never depend on it — use `SortedDictionary` if you need order.

## HashSet&lt;T&gt; and SortedSet&lt;T&gt; (vs HashSet/TreeSet)

**Think of it like...** a guest list. A name is either on it or not — duplicates are silently ignored. `SortedSet` is the same guest list but kept alphabetized.

```csharp
var seen = new HashSet<int>();     // Java: new HashSet<>()
bool added = seen.Add(5);          // returns true if newly added, false if duplicate
seen.Add(5);                       // ignored, returns false
bool has = seen.Contains(5);       // O(1)
seen.Remove(5);

// Set algebra (Java needs retainAll/addAll/removeAll — C# names are clearer)
var a = new HashSet<int> { 1, 2, 3 };
var b = new HashSet<int> { 2, 3, 4 };
a.UnionWith(b);                    // a becomes {1,2,3,4}   (Java: addAll)
a.IntersectWith(b);               // keep only common       (Java: retainAll)
a.ExceptWith(b);                  // remove b's elements     (Java: removeAll)
bool subset = a.IsSubsetOf(b);

// SortedSet — Java's TreeSet (kept in sorted order, backed by a red-black tree)
var sorted = new SortedSet<int> { 5, 1, 3, 1 };
foreach (int n in sorted) Console.Write(n); // 1 3 5  — sorted, duplicate dropped
Console.WriteLine(sorted.Min);     // 1  (Java TreeSet: .first())
Console.WriteLine(sorted.Max);     // 5  (Java TreeSet: .last())
```

## SortedDictionary and SortedList (vs TreeMap)

**Think of it like...** a dictionary book — entries are always kept in alphabetical (sorted) key order, so you can flip to any letter quickly.

Both keep keys **sorted**, mirroring Java's `TreeMap`. They differ in internals and performance trade-offs.

```csharp
// SortedDictionary<K,V> — red-black TREE. Fast inserts/removes: O(log n).
var sd = new SortedDictionary<string, int>();
sd["banana"] = 2;
sd["apple"]  = 1;
sd["cherry"] = 3;
foreach (var kvp in sd) Console.WriteLine(kvp.Key); // apple, banana, cherry (sorted)

// SortedList<K,V> — backed by two sorted ARRAYS. Less memory, FAST lookups by index,
// but SLOW (O(n)) inserts in the middle. Good for "build once, read many".
var sl = new SortedList<string, int>();
sl["banana"] = 2;
sl["apple"]  = 1;
Console.WriteLine(sl.Keys[0]);     // "apple" — can index into keys/values like an array!
```

| | `SortedDictionary` | `SortedList` |
|---|---|---|
| Backing | Red-black tree | Two arrays (keys, values) |
| Insert/Remove | O(log n) | O(n) |
| Index access | No | Yes (`.Keys[i]`) |
| Memory | More | Less |
| Use when | Many inserts/removes | Mostly reads, sorted, low memory |

## Queue&lt;T&gt;, Stack&lt;T&gt;, LinkedList&lt;T&gt;

**Think of it like...** a `Queue` is a line at a coffee shop (first in, first served, FIFO). A `Stack` is a pile of plates (take from the top, LIFO). A `LinkedList` is a chain where each link knows its neighbors.

```csharp
// QUEUE — FIFO. Java ArrayDeque/LinkedList as Queue.
var q = new Queue<string>();
q.Enqueue("a");                    // Java: .offer()/.add()
q.Enqueue("b");
string head = q.Peek();            // look without removing — "a"
string out1 = q.Dequeue();         // remove + return — "a"  (Java: .poll())
Console.WriteLine(q.Count);

// STACK — LIFO. Java's Deque (Java's old Stack class is discouraged).
var st = new Stack<int>();
st.Push(1);                        // Java: .push()
st.Push(2);
int top = st.Peek();               // 2, not removed
int popped = st.Pop();             // 2, removed

// LINKEDLIST — doubly-linked, O(1) add/remove at both ends and at a known node.
var ll = new LinkedList<int>();
ll.AddLast(2);                     // Java: .addLast()
ll.AddFirst(1);                    // Java: .addFirst()
LinkedListNode<int> node = ll.First; // expose nodes directly (Java hides them)
ll.AddAfter(node, 99);             // insert after a specific node — O(1)
ll.RemoveFirst();

// PRIORITYQUEUE (.NET 6+) — min-heap by priority. Java: PriorityQueue.
var pq = new PriorityQueue<string, int>();
pq.Enqueue("low",  5);             // element + explicit priority (lower = dequeued first)
pq.Enqueue("high", 1);
Console.WriteLine(pq.Dequeue());   // "high"
```

> Unlike Java where `ArrayDeque` serves as both queue and stack, C# gives you dedicated `Queue<T>` and `Stack<T>` classes — clearer intent.

## The Interface Hierarchy

**Think of it like...** a family tree of capabilities. Each level down adds more abilities. Code against the highest level you can — accept `IEnumerable<T>` in method parameters so callers can pass anything iterable.

In Java the root is `Iterable<T>`, then `Collection<T>`, then `List`/`Set`/etc. In C# the root is **`IEnumerable<T>`**.

```
IEnumerable<T>          // "can be foreach'd"      (Java: Iterable<T>) — only GetEnumerator()
   └── ICollection<T>   // adds Count, Add, Remove, Contains, Clear  (Java: Collection<T>)
          ├── IList<T>          // adds integer indexer this[int]    (Java: List<T>)
          │      └── List<T>, T[] (array implements IList<T>)
          └── ISet<T>           // set operations                     (Java: Set<T>)
                 └── HashSet<T>, SortedSet<T>

IDictionary<TKey,TValue>        // key/value, this[key]               (Java: Map<K,V>)
   └── Dictionary<K,V>, SortedDictionary<K,V>
```

```csharp
// Accept the WIDEST useful type. This method works on List, array, HashSet, query result...
int SumAll(IEnumerable<int> items)  // Java: takes Iterable<Integer>
{
    int total = 0;
    foreach (int x in items) total += x;
    return total;
}

SumAll(new[] { 1, 2, 3 });          // array
SumAll(new List<int> { 4, 5 });     // list
SumAll(new HashSet<int> { 6 });     // set — all fine, all are IEnumerable<int>
```

> Rule: **return concrete types** (`List<T>`) from your own methods so callers get full features, but **accept the most abstract type** (`IEnumerable<T>`) as parameters for flexibility.

## Read-Only and Immutable Collections

**Think of it like...** read-only is a window into a room — you can look but not touch (though someone else might rearrange the room). Immutable is a sealed photograph — nothing can ever change it.

There are two distinct ideas, and the interview difference matters:

```csharp
using System.Collections.ObjectModel;
using System.Collections.Immutable; // needs System.Collections.Immutable package/namespace

// 1) READ-ONLY VIEW — you can't modify it through this reference, but the
//    underlying list CAN still change. Like Java's Collections.unmodifiableList wrapper.
List<int> backing = new() { 1, 2, 3 };
IReadOnlyList<int> view = backing;  // expose without mutation methods
// view.Add(4);                     //  no Add method exists
backing.Add(4);                     //  the VIEW now reflects 4 too — it's a live window

// 2) IMMUTABLE — a genuinely frozen copy. Java: List.of(...) / List.copyOf(...).
ImmutableList<int> frozen = ImmutableList.Create(1, 2, 3);
ImmutableList<int> bigger = frozen.Add(4); // returns a NEW list; 'frozen' is unchanged
Console.WriteLine(frozen.Count);    // still 3
Console.WriteLine(bigger.Count);    // 4

// Common immutable types: ImmutableArray<T>, ImmutableDictionary<K,V>, ImmutableHashSet<T>
var dict = ImmutableDictionary<string,int>.Empty.Add("a", 1);

// Read-only dictionary view
IReadOnlyDictionary<string,int> roDict = new Dictionary<string,int> { ["x"] = 1 };
```

> Interview soundbite: **`IReadOnly*` = "you can't change it, but it might change"; `Immutable*` = "nobody can ever change it."**

## Concurrent Collections

**Think of it like...** a regular collection is a single-lane road — two threads crashing in is a wreck. Concurrent collections are roads built for multiple lanes of traffic at once.

Regular collections are **not thread-safe**. For multi-threaded access use `System.Collections.Concurrent`.

```csharp
using System.Collections.Concurrent;

// ConcurrentDictionary — Java's ConcurrentHashMap
var cd = new ConcurrentDictionary<string, int>();
cd["a"] = 1;
cd.TryAdd("b", 2);
// Atomic "compute if absent" — Java: computeIfAbsent
int v = cd.GetOrAdd("c", key => Compute(key));
// Atomic update — Java: merge / compute
cd.AddOrUpdate("a", 1, (key, old) => old + 1); // if absent add 1, else increment

// ConcurrentQueue — Java's ConcurrentLinkedQueue (thread-safe FIFO)
var cq = new ConcurrentQueue<int>();
cq.Enqueue(10);
if (cq.TryDequeue(out int item)) { /* got item safely */ }

// ConcurrentBag — unordered, optimized for "same thread adds and removes"
var bag = new ConcurrentBag<int>();
bag.Add(1);
bag.TryTake(out int taken);

// BlockingCollection — producer/consumer with blocking. Java: BlockingQueue.
var bc = new BlockingCollection<int>(boundedCapacity: 5); // bounded buffer
bc.Add(1);                          // blocks if full
int got = bc.Take();                // blocks if empty
bc.CompleteAdding();                // signal "no more items"

int Compute(string s) => s.Length;
```

> `ConcurrentDictionary` shines: `GetOrAdd` and `AddOrUpdate` are **atomic**, so you avoid the check-then-act race that plagues a plain `Dictionary` guarded by a lock.

## Custom Equality and Ordering (IEqualityComparer / IComparer)

**Think of it like...** `IEqualityComparer` answers "are these two the same?" (for hashing/sets/dictionaries). `IComparer` answers "which one comes first?" (for sorting). Java splits these the same way: `equals/hashCode` vs `Comparator`.

```csharp
// --- Ordering: IComparer<T>  (Java: Comparator<T>) ---
public class Person { public string Name = ""; public int Age; }

// Sort by age (Java: Comparator.comparingInt(Person::getAge))
var byAge = Comparer<Person>.Create((a, b) => a.Age.CompareTo(b.Age));
var people = new List<Person> { new(){Name="A",Age=30}, new(){Name="B",Age=20} };
people.Sort(byAge);                 // in-place sort, Java: list.sort(byAge)

// LINQ ordering (Java streams .sorted)
var sorted = people.OrderBy(p => p.Age).ThenBy(p => p.Name).ToList();

// --- Equality: IEqualityComparer<T>  (Java: equals + hashCode) ---
// Make a HashSet treat strings case-insensitively:
var ci = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
ci.Add("Hello");
Console.WriteLine(ci.Contains("HELLO")); // true

// Custom comparer for your own type used as a dictionary key:
public class PersonNameComparer : IEqualityComparer<Person>
{
    public bool Equals(Person? x, Person? y) => x?.Name == y?.Name;       // like equals()
    public int GetHashCode(Person p) => p.Name.GetHashCode();             // like hashCode()
}
var dict = new Dictionary<Person, int>(new PersonNameComparer());

// EASIEST modern way: use a record — value equality is auto-generated (like Java records).
public record PersonRec(string Name, int Age); // Equals/GetHashCode generated for you
```

> Just like Java, if you put a custom class in a `HashSet`/`Dictionary` you MUST give it proper equality — either override `Equals`/`GetHashCode`, supply an `IEqualityComparer`, or use a `record`. Otherwise it falls back to reference equality.

## Iterating: foreach, Iterators, yield return

**Think of it like...** `foreach` is a conveyor belt — items arrive one at a time and you don't know or care how many there are. `yield return` lets you BE the conveyor belt, handing out items lazily on demand.

```csharp
// foreach — Java's enhanced for loop. Works on any IEnumerable<T>.
var list = new List<int> { 1, 2, 3 };
foreach (int x in list) Console.WriteLine(x); // Java: for (int x : list)

// Under the hood foreach calls GetEnumerator() -> IEnumerator<T> (Java: iterator()/Iterator)
IEnumerator<int> e = list.GetEnumerator();
while (e.MoveNext())                 // Java: while (it.hasNext())
{
    int current = e.Current;         // Java: it.next()
}

// yield return — build a lazy sequence WITHOUT a backing collection.
// Java equivalent requires writing a whole Iterator class; C# does it in one method.
IEnumerable<int> Squares(int n)
{
    for (int i = 1; i <= n; i++)
        yield return i * i;          // pauses here, resumes on next iteration — LAZY
}

foreach (int sq in Squares(4)) Console.Write($"{sq} "); // 1 4 9 16

// Lazy means infinite sequences are fine if you stop early:
IEnumerable<int> Naturals()
{
    int i = 0;
    while (true) yield return i++;    // never-ending, but only runs as far as consumed
}
var firstFive = Naturals().Take(5).ToList(); // 0 1 2 3 4
```

> `yield return` is one of C#'s nicest features over Java. Combined with LINQ, it gives you lazy, composable pipelines with almost no boilerplate.

## Collection Initializers and Collection Expressions

**Think of it like...** shorthand for "fill this collection right after creating it" — and C# 12's `[...]` is the universal, type-agnostic version.

```csharp
// Classic collection initializer (since C# 3) — Java: List.of / Map.of (but mutable here)
var nums = new List<int> { 1, 2, 3 };
var map  = new Dictionary<string,int> { ["a"] = 1, ["b"] = 2 };
var map2 = new Dictionary<string,int> { { "a", 1 }, { "b", 2 } }; // alt syntax
var set  = new HashSet<int> { 1, 2, 2, 3 };       // {1,2,3}

// COLLECTION EXPRESSIONS (C# 12 / .NET 8) — the new universal [ ] syntax
int[] arr        = [1, 2, 3];                 // target type inferred from the variable
List<int> list   = [1, 2, 3];                 // same literal works for List
Span<int> span   = [1, 2, 3];                 // ...and many other collection types

// The spread operator ".." merges sequences (Java has no direct equivalent)
int[] head = [1, 2];
int[] tail = [4, 5];
int[] all  = [..head, 3, ..tail];             // [1, 2, 3, 4, 5]

// Great for method calls
void Print(IEnumerable<int> xs) { }
Print([10, 20, 30]);                          // no 'new List' noise
```

## Big-O and When to Use Which Collection

**Think of it like...** picking the right tool. A hammer (array) is great for fixed indexing; a label-maker (dictionary) is great for lookups by name. Match the collection to your access pattern.

| Collection | Access by index | Search (Contains) | Insert/Remove | Notes |
|---|---|---|---|---|
| `T[]` | O(1) | O(n) | n/a (fixed) | Fastest, smallest, fixed size |
| `List<T>` | O(1) | O(n) | O(1) end / O(n) middle | Default choice for sequences |
| `Dictionary<K,V>` | n/a | O(1) avg | O(1) avg | Key lookups |
| `HashSet<T>` | n/a | O(1) avg | O(1) avg | Uniqueness / membership |
| `SortedDictionary<K,V>` | n/a | O(log n) | O(log n) | Sorted keys, tree |
| `SortedList<K,V>` | O(1) by index | O(log n) | O(n) | Sorted, low memory, read-heavy |
| `SortedSet<T>` | n/a | O(log n) | O(log n) | Sorted uniqueness |
| `Queue<T>` | n/a | O(n) | O(1) enqueue/dequeue | FIFO |
| `Stack<T>` | n/a | O(n) | O(1) push/pop | LIFO |
| `LinkedList<T>` | O(n) | O(n) | O(1) at known node | Frequent middle insert/remove |

**Decision guide:**
- Need a resizable list of things, accessed by position? → `List<T>`
- Need fast lookup by a key? → `Dictionary<K,V>`
- Need to ensure no duplicates / test membership? → `HashSet<T>`
- Need keys/items kept sorted? → `SortedDictionary` / `SortedSet`
- FIFO processing? → `Queue<T>`. LIFO? → `Stack<T>`.
- Multi-threaded shared access? → `Concurrent*` collections.
- Default when unsure? → `List<T>` for sequences, `Dictionary<K,V>` for lookups.

## Common Pitfalls

```csharp
// PITFALL 1: Modifying a collection while iterating it.
// Same as Java's ConcurrentModificationException — C# throws InvalidOperationException.
var list = new List<int> { 1, 2, 3, 4 };
// foreach (int x in list) { if (x == 2) list.Remove(x); } //  THROWS

// FIX A: iterate a copy
foreach (int x in list.ToList()) { if (x == 2) list.Remove(x); }
// FIX B: loop backwards by index
for (int i = list.Count - 1; i >= 0; i--) { if (list[i] == 2) list.RemoveAt(i); }
// FIX C: RemoveAll with a predicate (cleanest) — Java: removeIf
list.RemoveAll(x => x == 2);

// PITFALL 2: Dictionary indexer read on a missing key throws KeyNotFoundException.
var d = new Dictionary<string,int> { ["a"] = 1 };
// int v = d["missing"]; //  THROWS (Java HashMap.get returns null instead)
int safe = d.TryGetValue("missing", out var val) ? val : 0;  // correct
int safe2 = d.GetValueOrDefault("missing", 0);               // also fine

// PITFALL 3: Dictionary.Add throws on duplicate key; indexer overwrites silently.
d["a"] = 99;     // overwrites, no error
// d.Add("a", 5); //  THROWS because "a" exists

// PITFALL 4: Custom class as a key without proper equality => reference equality only.
// Two "equal" objects won't be found. Use a record or override Equals/GetHashCode.

// PITFALL 5: Don't rely on Dictionary/HashSet iteration order — it's not guaranteed.

// PITFALL 6: LINQ queries are LAZY — they re-execute every time you enumerate.
var query = list.Where(x => x > 0);   // not run yet
list.Add(100);                        // query will see this when enumerated!
var snapshot = list.Where(x => x > 0).ToList(); // ToList() materializes NOW
```

## Common Interview Questions

**1. What's the C# equivalent of Java's `ArrayList` and `HashMap`?**
`List<T>` for `ArrayList`, and `Dictionary<TKey,TValue>` for `HashMap`. Both are generic, array/hash backed, and grow automatically. Naming is the main difference.

**2. What is the root collection interface in C#, and how does it differ from Java?**
`IEnumerable<T>` is the root (analogous to Java's `Iterable<T>`). It only requires `GetEnumerator()`. Everything you can `foreach` over is an `IEnumerable<T>`. `ICollection<T>`, `IList<T>`, and `IDictionary<K,V>` build on top of it.

**3. How do you safely read from a `Dictionary` when the key might not exist?**
Use `TryGetValue(key, out var value)` — it returns a `bool` and avoids the `KeyNotFoundException` the indexer throws. Alternatives: `ContainsKey` (extra lookup) or `GetValueOrDefault`. This contrasts with Java's `HashMap.get()` which returns `null` for a missing key.

**4. Difference between the indexer (`dict[key] = v`) and `dict.Add(key, v)`?**
The indexer **adds or overwrites** silently. `Add` **throws** if the key already exists. Use the indexer for upserts, `Add` when a duplicate should be an error.

**5. Multidimensional vs jagged arrays?**
`int[,]` is a true rectangular array — one contiguous block, every row the same length. `int[][]` (jagged) is an array of arrays where each row can have a different length — this is the only form Java has. Jagged is often faster; multidimensional is cleaner for true matrices.

**6. `IReadOnlyList<T>` vs `ImmutableList<T>`?**
`IReadOnlyList<T>` is a **read-only view** — you can't mutate through it, but the underlying collection can still change. `ImmutableList<T>` is **truly immutable** — every "modification" returns a new list and the original never changes. View vs frozen copy.

**7. How do you make a custom class work correctly as a `Dictionary` key or in a `HashSet`?**
Provide proper equality: override `Equals` and `GetHashCode`, pass an `IEqualityComparer<T>`, or (easiest) use a `record` which generates value equality. Without this, C# uses reference equality and "equal" objects won't match — same rule as Java's `equals`/`hashCode` contract.

**8. `IComparer<T>` vs `IEqualityComparer<T>`?**
`IComparer<T>` defines **ordering** (returns negative/zero/positive) for sorting — Java's `Comparator`. `IEqualityComparer<T>` defines **equality + hashing** for sets/dictionaries — Java's `equals`/`hashCode`. Different jobs.

**9. What is `yield return` and what's the Java equivalent?**
`yield return` builds a **lazy iterator** — the method pauses and resumes, producing items on demand without a backing collection. In Java you'd hand-write an `Iterator` class. It enables lazy/infinite sequences and pairs naturally with LINQ.

**10. Which concurrent collections does C# offer, and their Java analogs?**
`ConcurrentDictionary` (≈ `ConcurrentHashMap`), `ConcurrentQueue` (≈ `ConcurrentLinkedQueue`), `ConcurrentStack`, `ConcurrentBag`, and `BlockingCollection` (≈ `BlockingQueue`) for producer/consumer. `ConcurrentDictionary` provides atomic `GetOrAdd`/`AddOrUpdate`.

**11. What happens if you modify a `List` while iterating it with `foreach`?**
It throws `InvalidOperationException` (analogous to Java's `ConcurrentModificationException`). Fix by iterating a copy (`.ToList()`), looping backwards by index, or using `RemoveAll(predicate)`.

**12. `SortedDictionary` vs `SortedList` — when to use each?**
Both keep keys sorted (like `TreeMap`). `SortedDictionary` uses a red-black tree: O(log n) inserts/removes — good when the collection changes a lot. `SortedList` uses arrays: O(1) index access and low memory, but O(n) inserts — good for read-heavy, build-once scenarios.

**13. What are collection expressions in C# 12?**
The `[1, 2, 3]` syntax that works across array, `List<T>`, `Span<T>`, and more, with the target type inferred. The spread operator `..` merges sequences: `[..a, ..b]`. Reduces `new List<int> { ... }` boilerplate.

**14. Why prefer accepting `IEnumerable<T>` in method parameters?**
It's the most abstract sequence type, so callers can pass a `List`, array, `HashSet`, or a lazy LINQ result. Best practice: accept the widest type (`IEnumerable<T>`) for inputs, return concrete types (`List<T>`) so callers get full functionality.

## Quick Reference Cheat Sheet

```
JAVA → C# QUICK MAP
  ArrayList<T>        -> List<T>
  HashMap<K,V>        -> Dictionary<K,V>
  HashSet<T>          -> HashSet<T>
  TreeMap<K,V>        -> SortedDictionary<K,V> / SortedList<K,V>
  TreeSet<T>          -> SortedSet<T>
  LinkedList<T>       -> LinkedList<T>
  ArrayDeque (queue)  -> Queue<T>      ArrayDeque (stack) -> Stack<T>
  PriorityQueue<T>    -> PriorityQueue<TElem,TPrio>
  Iterable<T>         -> IEnumerable<T>     Iterator<T> -> IEnumerator<T>
  Comparator<T>       -> IComparer<T>
  equals/hashCode     -> IEqualityComparer<T> (or override / record)
  ConcurrentHashMap   -> ConcurrentDictionary<K,V>
  BlockingQueue       -> BlockingCollection<T>
  List.of(...)        -> ImmutableList.Create(...)
  stream()            -> LINQ (.Where/.Select/.OrderBy)

METHOD NAME DIFFERENCES
  .size()    -> .Count        .get(i)/.set(i,v) -> list[i] (indexer)
  .add(x)    -> .Add(x)       .put(k,v)         -> dict[k] = v
  .get(k)    -> dict[k] or TryGetValue          .containsKey(k) -> .ContainsKey(k)
  .offer()/.poll() (queue)    -> .Enqueue()/.Dequeue()
  .push()/.pop()  (stack)     -> .Push()/.Pop()
  array .length               -> array .Length (property)

DICTIONARY SAFE-READ (memorize this)
  if (dict.TryGetValue(key, out var value)) { use value; }
  var v = dict.GetValueOrDefault(key, fallback);

SETS
  Add returns bool (false if dup); UnionWith/IntersectWith/ExceptWith

PICK A COLLECTION
  sequence by index        -> List<T>
  lookup by key            -> Dictionary<K,V>
  unique / membership      -> HashSet<T>
  sorted keys / items      -> SortedDictionary / SortedSet
  FIFO -> Queue<T>     LIFO -> Stack<T>
  thread-safe              -> Concurrent* / BlockingCollection
  truly frozen             -> Immutable*

GOTCHAS
  - dict[missingKey] THROWS KeyNotFoundException (use TryGetValue)
  - dict.Add(existingKey) THROWS (indexer overwrites instead)
  - modifying a collection during foreach THROWS InvalidOperationException
  - Dictionary/HashSet have NO guaranteed iteration order
  - LINQ is lazy; call .ToList()/.ToArray() to materialize
  - custom key class needs Equals/GetHashCode (or use a record)

COLLECTION EXPRESSIONS (C# 12)
  int[] a = [1, 2, 3];     List<int> b = [..a, 4, 5];
```

*Last Updated: 2026-06-16*

# Java Collections Framework – Internal Working

> **How to use this guide (junior dev):** The single most asked question here is **"How does HashMap work internally?"** — start with [section 3.3](#33-hashmap-internal-structure-critical-for-interviews) and be able to explain it out loud. After that, the most common follow-ups are "ArrayList vs LinkedList" and "why must I override `equals()` and `hashCode()` together?". Everything else supports those three.

---

## Table of Contents

1. [Concept Explanation](#1-concept-explanation)
2. [Why It's Used in Real-World Applications](#2-why-its-used-in-real-world-applications)
3. [Internal Working (Deep Dive)](#3-internal-working-deep-dive)
   - [3.1 ArrayList](#31-arraylist-internal-structure)
   - [3.2 LinkedList](#32-linkedlist-internal-structure)
   - [3.3 HashMap ⭐ most asked](#33-hashmap-internal-structure-critical-for-interviews)
4. [Code Examples](#4-code-examples)
5. [Common Mistakes & Pitfalls](#5-common-mistakes--pitfalls)
6. [Interview Tips & Common Questions](#6-interview-tips)
7. [Short Revision Summary (Cheat Sheet)](#7-short-revision-summary)

---

## 1. Concept Explanation

The Java Collections Framework is a unified architecture for storing and manipulating groups of objects — a toolbox with different containers, each designed for specific use cases.

- **ArrayList** = Dynamic array (fast random access, can grow)
- **LinkedList** = Doubly-linked chain (fast insert/remove at ends)
- **HashMap** = Hash-based key-value store (O(1) lookup)
- **HashSet** = HashMap keys only (no duplicates)

## 2. Why It's Used in Real-World Applications

| Collection | Real-World Use Case | Why It's Chosen |
|------------|-------------------|----------------|
| ArrayList | User lists, product catalogs | Fast random access, memory-efficient |
| LinkedList | Queue systems, undo-redo | Fast insertions/deletions at ends |
| HashMap | Sessions, cache, configs | O(1) lookup time |
| HashSet | Unique IDs, permission checks | Automatic duplicate prevention |
| TreeSet | Sorted leaderboards, range queries | Maintains sorted order |
| PriorityQueue | Task schedulers, Dijkstra's | Always returns highest/lowest priority |

## 3. Internal Working (Deep Dive)

### 3.1 ArrayList Internal Structure

ArrayList is backed by an Object array that grows dynamically.

```java
// Growth strategy: new_capacity = old_capacity * 1.5
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5x
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

**Key details:**
- Initial capacity: 10 (unless specified at construction)
- Growth factor: 1.5x
- `add()` is O(1) amortized — occasional O(n) during resize
- Random access (`get(index)`) is O(1); removal is O(n) due to shifting

### 3.2 LinkedList Internal Structure

LinkedList uses a doubly-linked list — each node holds the data plus `prev` and `next` references.

```
null <- [prev|data|next] <-> [prev|data|next] <-> [prev|data|next] -> null
        first                                       last
```

**Key details:**
- No array resizing; each node is allocated individually
- `addFirst/addLast/removeFirst/removeLast` are O(1)
- `get(index)` requires traversal — O(n)

### 3.3 HashMap Internal Structure (CRITICAL for Interviews)

**Plain-English mental model:** Think of a wall of numbered mailboxes (buckets). On `put(key, value)`:
1. HashMap calls `key.hashCode()` to get a number.
2. It maps that number to a mailbox slot (array index).
3. The entry goes into that slot. If two keys land in the same slot (**collision**), they chain together as a linked list inside that mailbox.

On `get(key)`, the same math finds the mailbox instantly (O(1)), then `equals()` identifies the right entry within it. This is why **fast lookup needs a good `hashCode()`** (spreads keys) **and a correct `equals()`** (picks the right entry).

**The 3 steps:**

```java
// 1. Mix the hash to spread keys evenly
int hash = (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
// null keys always land in bucket 0; >>> 16 mixes high bits into low bits

// 2. Find bucket index (fast modulo — works because capacity is always power of 2)
int index = (n - 1) & hash;

// 3. Collision handling (Java 8+)
// bucket size < 8  → linked list  (O(n) within bucket)
// bucket size >= 8 → TreeNode     (O(log n) within bucket)
// tree size <= 6   → back to list
```

**Visual structure:**
```
table[0] -> null
table[1] -> Node(key1,val1) -> Node(key2,val2)  // collision chain
table[2] -> TreeNode(...)                        // treeified bucket
table[15]-> Node(key3,val3)
```

**Resize:** When `size > capacity * 0.75`, capacity doubles and all entries are rehashed.

- Default capacity: 16 (power of 2 for fast bit-wise index math)
- Default load factor: 0.75 (balance between space and collision rate)

## 4. Code Examples

### 4.1 ArrayList

```java
List<String> users = new ArrayList<>(100);  // pre-size when known

users.add("Alice");
users.add("Bob");

String first = users.get(0);       // O(1)
users.remove(0);                    // O(n) — shifts elements
boolean has = users.contains("Bob"); // O(n)
```

### 4.2 HashMap

```java
Map<String, Integer> scores = new HashMap<>();

scores.put("Alice", 95);
scores.put("Bob", 87);

Integer score = scores.get("Alice");                    // O(1) avg
Integer safe  = scores.getOrDefault("Unknown", 0);

for (Map.Entry<String, Integer> e : scores.entrySet()) {
    System.out.println(e.getKey() + ": " + e.getValue());
}

// Word-count pattern (Java 8)
wordCount.merge("hello", 1, Integer::sum);
```

### 4.3 HashSet — duplicate detection

```java
List<String> names = Arrays.asList("Alice", "Bob", "Alice");
Set<String> seen = new HashSet<>();
Set<String> duplicates = new HashSet<>();

for (String name : names) {
    if (!seen.add(name)) {   // add() returns false if already present
        duplicates.add(name);
    }
}
// duplicates → [Alice]
```

### 4.4 Custom Object as HashMap Key (CRITICAL)

Always override **both** `hashCode()` and `equals()` — using the same field(s).

```java
class User {
    private final String id;  // immutable — never change hash fields after insertion
    private String name;

    public User(String id, String name) { this.id = id; this.name = name; }

    @Override
    public int hashCode() { return Objects.hash(id); }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof User)) return false;
        return Objects.equals(id, ((User) obj).id);
    }
}

// Usage
Map<User, String> roles = new HashMap<>();
User u1 = new User("001", "Alice");
User u2 = new User("001", "Alice Updated");  // same id
roles.put(u1, "Admin");
roles.get(u2);  // → "Admin" (equals() matches on id)
```

## 5. Common Mistakes & Pitfalls

### Mistake 1: Wrong collection for the use case

```java
// ❌ BAD: LinkedList for random access — O(n) per get
List<Integer> numbers = new LinkedList<>();
numbers.get(5000);  // traverses half the list

// ✅ GOOD
List<Integer> numbers = new ArrayList<>();
numbers.get(5000);  // O(1)
```

### Mistake 2: Not pre-sizing when capacity is known

```java
// ❌ triggers multiple resizes
List<String> users = new ArrayList<>();

// ✅ single allocation
List<String> users = new ArrayList<>(10000);
```

### Mistake 3: Removing from list during for-each

```java
// ❌ ConcurrentModificationException
for (String name : names) {
    if (name.equals("B")) names.remove(name);
}

// ✅ Iterator.remove()
Iterator<String> it = names.iterator();
while (it.hasNext()) {
    if (it.next().equals("B")) it.remove();
}

// ✅ Or Java 8+
names.removeIf(name -> name.equals("B"));
```

### Mistake 4: Mutable HashMap key

```java
// ❌ Mutating a key after insertion loses the entry
key.setValue(2);   // hashCode changes
map.get(key);      // → null (wrong bucket now)

// ✅ Make hash fields final
private final int value;
```

### Mistake 5: Overriding only equals() without hashCode()

```java
// ❌ Two "equal" objects hash to different buckets → get() returns null
class User {
    @Override public boolean equals(Object obj) { ... }
    // missing hashCode() — inherits Object's identity hash
}

// ✅ Always override both, using the same fields
@Override public int hashCode() { return Objects.hash(id); }
```

## 6. Interview Tips

### What Interviewers Expect

**Time Complexity (memorise this table):**

```
ArrayList:   get(i)=O(1)  add()=O(1)*  add(i,e)=O(n)  remove=O(n)  contains=O(n)
LinkedList:  get(i)=O(n)  addFirst/Last=O(1)  removeFirst/Last=O(1)  contains=O(n)
HashMap:     get/put/containsKey/remove=O(1) average, O(n) worst
```
\* amortized

**When to use which:**
- ArrayList: frequent random access, infrequent modifications
- LinkedList: frequent add/remove at ends, queue/deque operations
- HashMap: key-value pairs, fast lookup
- HashSet: unique elements, membership testing
- TreeSet: sorted elements, range queries

**Common follow-up Q&A:**

| Question | Punchy Answer |
|----------|--------------|
| Why default capacity 16? | Power of 2 enables fast `(n-1) & hash` index calculation instead of `%`. |
| Why load factor 0.75? | Empirical sweet spot — lower wastes memory, higher causes too many collisions. |
| Same hashCode, not equal? | Collision — entries share a bucket, chained as linked list or tree. |
| Equal but different hashCode? | HashMap breaks — `get()` looks in the wrong bucket and returns null. |
| Why treeify at 8? | Protects against O(n) degradation from hash flooding; O(log n) tree takes over. |

## 7. Short Revision Summary

### ArrayList
- **Structure**: Dynamic Object array, grows 1.5x
- **Best for**: Random access, infrequent insertions/deletions
- **Time**: `get`=O(1), `add`=O(1) amortized, `remove`=O(n)

### LinkedList
- **Structure**: Doubly-linked nodes, no array
- **Best for**: Queue/deque, frequent add/remove at ends
- **Time**: `get`=O(n), `addFirst/Last`=O(1), `removeFirst/Last`=O(1)

### HashMap
- **Structure**: Array of buckets → linked list → tree (at 8 collisions)
- **Resize**: Double capacity when `size > capacity × 0.75`
- **Key contract**: Override both `equals()` and `hashCode()`; keys should be immutable
- **Time**: `get/put/remove`=O(1) average, O(n) worst

### HashSet
- **Structure**: HashMap internally (values ignored)
- **Time**: Same as HashMap — O(1) average

### Quick Comparison Table

| Collection | Add | Get | Remove | Contains | Best Use Case |
|------------|-----|-----|--------|----------|---------------|
| ArrayList  | O(1)* | O(1) | O(n) | O(n) | Random access |
| LinkedList | O(1)* | O(n) | O(1)* | O(n) | Queue/deque |
| HashMap    | O(1)* | O(1)* | O(1)* | O(1)* | Key-value lookup |
| HashSet    | O(1)* | — | O(1)* | O(1)* | Unique elements |

\* Amortized/average case

### Critical Points to Remember
1. `(n-1) & hash` for index — works because capacity is always a power of 2
2. `>>> 16` in hash function mixes high bits to reduce clustering
3. Treeify threshold = 8, untreeify = 6
4. Always override **both** `equals()` and `hashCode()` together
5. HashMap keys must be effectively immutable (never mutate hash fields after insertion)
6. Use `Iterator.remove()` or `removeIf()` when deleting during iteration
7. Pre-size collections when you know the expected element count

---

**Next Topics to Study:**
- OOP Principles (Encapsulation, Inheritance, Polymorphism, Abstraction)
- Java 8 Streams and Lambdas
- Multithreading Basics
- Spring Framework Fundamentals

---
*Last Updated: 2026-06-18*

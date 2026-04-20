# Java Collections Framework – Internal Working

## 1. Concept Explanation

The Java Collections Framework is a unified architecture for storing and manipulating groups of objects. Think of it as a toolbox with different types of containers, each designed for specific use cases.

**Real-world analogy:**
- **ArrayList** = Dynamic shopping list (fast to read, can grow)
- **LinkedList** = Train of connected cars (fast to insert/remove at ends)
- **HashMap** = Phone directory (quick lookup by name)
- **HashSet** = Unique ID collection (no duplicates)

## 2. Why It's Used in Real-World Applications

| Collection | Real-World Use Case | Why It's Chosen |
|------------|-------------------|----------------|
| ArrayList | User lists, product catalogs | Fast random access, memory-efficient |
| LinkedList | Queue systems, undo-redo operations | Fast insertions/deletions at ends |
| HashMap | User sessions, cache, configurations | O(1) lookup time |
| HashSet | Unique identifiers, permission checks | Automatic duplicate prevention |
| TreeSet | Sorted leaderboards, range queries | Maintains sorted order |
| PriorityQueue | Task schedulers, Dijkstra's algorithm | Always returns highest/lowest priority |

## 3. Internal Working (Deep Dive)

### 3.1 ArrayList Internal Structure

```java
// ArrayList is backed by an array that grows dynamically
public class ArrayList<E> {
    private static final int DEFAULT_CAPACITY = 10;
    private Object[] elementData;  // The backing array
    private int size;              // Number of elements

    // When you add an element:
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Check if array needs growth
        elementData[size++] = e;
        return true;
    }

    // Growth strategy: new_capacity = old_capacity * 1.5
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5x growth
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
}
```

**Key Internal Details:**
- Initial capacity: 10 (unless specified)
- Growth factor: 1.5x (old + old/2)
- `elementData.length` = actual array size
- `size` = number of elements in use
- Amortized O(1) for add (occasional O(n) during resize)

### 3.2 LinkedList Internal Structure

```java
// LinkedList uses a doubly-linked list
public class LinkedList<E> {
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }

    transient Node<E> first;  // Head node
    transient Node<E> last;   // Tail node
}
```

**Visual Representation:**
```
null <- [prev|data|next] <-> [prev|data|next] <-> [prev|data|next] -> null
        first                                       last
```

**Key Internal Details:**
- Each node has references to previous and next nodes
- No array resizing needed
- Memory overhead: 2 references + object overhead per element
- O(1) for add/remove at ends
- O(n) for access by index (must traverse)

### 3.3 HashMap Internal Structure (CRITICAL for Interviews)

```java
public class HashMap<K,V> {
    // Array of buckets (Node<K,V> is a linked list node)
    transient Node<K,V>[] table;

    // Default initial capacity = 16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

    // Load factor = 0.75 (when to resize)
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    // Threshold = capacity * load factor (resize when size > threshold)
    int threshold;

    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;  // Linked list for collisions

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }
}
```

**How HashMap Works:**

```java
// 1. Calculate hash
final int hash = (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);

// 2. Find bucket index
int index = (n - 1) & hash;  // n = table.length

// 3. Handle collisions (Java 8+)
// - If bucket size < 8: Use linked list
// - If bucket size >= 8: Convert to balanced tree (TREEIFY_THRESHOLD)
// - If tree size <= 6: Convert back to linked list (UNTREEIFY_THRESHOLD)
```

**Visual HashMap Structure:**
```
table array (index based on hash):
[0] -> null
[1] -> Node(key1, value1) -> Node(key2, value2)  // Collision
[2] -> TreeNode(...)  // Tree for many collisions
[3] -> null
...
[15] -> Node(key3, value3)
```

**Critical Hash Function:**
```java
static final int hash(Object key) {
    int h;
    // Right shift by 16 to mix higher bits into lower bits
    // This reduces collisions when table size is power of 2
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

**Resize Process:**
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int newCap = oldCap << 1;  // Double the capacity
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;

    // Rehash all entries
    for (int j = 0; j < oldCap; ++j) {
        Node<K,V> e;
        if ((e = oldTab[j]) != null) {
            oldTab[j] = null;
            if (e.next == null) {
                // Single node, just place in new bucket
                newTab[e.hash & (newCap - 1)] = e;
            }
            // Handle linked list or tree nodes...
        }
    }
    return newTab;
}
```

## 4. Code Examples

### 4.1 ArrayList Usage

```java
import java.util.*;

public class ArrayListExample {
    public static void main(String[] args) {
        // Create with initial capacity (good practice if you know size)
        List<String> users = new ArrayList<>(100);

        // Add elements
        users.add("Alice");
        users.add("Bob");
        users.add("Charlie");

        // Access by index - O(1)
        String firstUser = users.get(0);  // "Alice"

        // Remove by index - O(n) (shifts all elements)
        users.remove(0);

        // Remove by value - O(n)
        users.remove("Bob");

        // Check if contains - O(n)
        boolean hasAlice = users.contains("Alice");

        // Iterate
        for (String user : users) {
            System.out.println(user);
        }

        // Convert to array
        String[] userArray = users.toArray(new String[0]);
    }
}
```

### 4.2 HashMap Usage

```java
import java.util.*;

public class HashMapExample {
    public static void main(String[] args) {
        // Create with initial capacity and load factor
        Map<String, Integer> scores = new HashMap<>(16, 0.75f);

        // Put entries - O(1) average
        scores.put("Alice", 95);
        scores.put("Bob", 87);
        scores.put("Charlie", 92);

        // Get value - O(1) average
        Integer aliceScore = scores.get("Alice");  // 95

        // Get with default value
        Integer unknownScore = scores.getOrDefault("Unknown", 0);

        // Check if contains key - O(1)
        boolean hasBob = scores.containsKey("Bob");

        // Remove entry - O(1)
        scores.remove("Bob");

        // Iterate over entries
        for (Map.Entry<String, Integer> entry : scores.entrySet()) {
            System.out.println(entry.getKey() + ": " + entry.getValue());
        }

        // Iterate over keys
        for (String key : scores.keySet()) {
            System.out.println(key);
        }

        // Compute if absent (common pattern for counting)
        Map<String, Integer> wordCount = new HashMap<>();
        wordCount.computeIfAbsent("hello", k -> 0);
        wordCount.put("hello", wordCount.get("hello") + 1);

        // Better: merge (Java 8+)
        wordCount.merge("hello", 1, Integer::sum);
    }
}
```

### 4.3 HashSet Usage

```java
import java.util.*;

public class HashSetExample {
    public static void main(String[] args) {
        // Create set
        Set<String> uniqueNames = new HashSet<>();

        // Add elements (duplicates are ignored)
        uniqueNames.add("Alice");
        uniqueNames.add("Bob");
        uniqueNames.add("Alice");  // Won't be added again

        // Check if contains - O(1)
        boolean hasAlice = uniqueNames.contains("Alice");

        // Remove element - O(1)
        uniqueNames.remove("Bob");

        // Size
        int size = uniqueNames.size();

        // Common pattern: find duplicates in array
        List<String> names = Arrays.asList("Alice", "Bob", "Alice", "Charlie");
        Set<String> seen = new HashSet<>();
        Set<String> duplicates = new HashSet<>();

        for (String name : names) {
            if (!seen.add(name)) {  // add returns false if already present
                duplicates.add(name);
            }
        }

        System.out.println("Duplicates: " + duplicates);  // [Alice]
    }
}
```

### 4.4 Custom Object in HashMap (CRITICAL)

```java
import java.util.*;

// MUST implement equals() and hashCode() correctly
class User {
    private String id;
    private String name;

    public User(String id, String name) {
        this.id = id;
        this.name = name;
    }

    // IMPORTANT: Objects used as keys MUST implement hashCode()
    @Override
    public int hashCode() {
        return Objects.hash(id);  // Only use id for hashing
    }

    // IMPORTANT: equals() must be consistent with hashCode()
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        User user = (User) obj;
        return Objects.equals(id, user.id);
    }

    @Override
    public String toString() {
        return "User{id='" + id + "', name='" + name + "'}";
    }
}

public class CustomKeyExample {
    public static void main(String[] args) {
        Map<User, String> userRoles = new HashMap<>();

        User user1 = new User("001", "Alice");
        User user2 = new User("001", "Alice Updated");  // Same id, different name

        userRoles.put(user1, "Admin");

        // This will find the entry because equals() checks id
        String role = userRoles.get(user2);  // Returns "Admin"

        System.out.println("Role: " + role);  // Role: Admin
    }
}
```

## 5. Common Mistakes & Pitfalls

### Mistake 1: Using wrong collection for the use case

```java
// ❌ BAD: Using LinkedList for random access
List<Integer> numbers = new LinkedList<>();
for (int i = 0; i < 10000; i++) {
    numbers.get(i);  // O(n) for each access - VERY SLOW!
}

// ✅ GOOD: Use ArrayList for random access
List<Integer> numbers = new ArrayList<>();
for (int i = 0; i < 10000; i++) {
    numbers.get(i);  // O(1) - FAST!
}
```

### Mistake 2: Not initializing capacity when known

```java
// ❌ BAD: Multiple resizes
List<String> users = new ArrayList<>();
for (int i = 0; i < 10000; i++) {
    users.add("User" + i);  // Will resize multiple times
}

// ✅ GOOD: Initialize with known capacity
List<String> users = new ArrayList<>(10000);
for (int i = 0; i < 10000; i++) {
    users.add("User" + i);  // No resizing needed
}
```

### Mistake 3: Removing from list while iterating

```java
// ❌ BAD: ConcurrentModificationException
List<String> names = new ArrayList<>(Arrays.asList("A", "B", "C"));
for (String name : names) {
    if (name.equals("B")) {
        names.remove(name);  // Exception!
    }
}

// ✅ GOOD: Use Iterator
List<String> names = new ArrayList<>(Arrays.asList("A", "B", "C"));
Iterator<String> iterator = names.iterator();
while (iterator.hasNext()) {
    String name = iterator.next();
    if (name.equals("B")) {
        iterator.remove();  // Safe
    }
}

// ✅ ALSO GOOD: Use removeIf (Java 8+)
names.removeIf(name -> name.equals("B"));
```

### Mistake 4: Using HashMap with mutable keys

```java
// ❌ BAD: Mutable keys break HashMap
class BadKey {
    private int value;

    public BadKey(int value) {
        this.value = value;
    }

    public void setValue(int value) {
        this.value = value;
    }

    @Override
    public int hashCode() {
        return Objects.hash(value);
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        BadKey badKey = (BadKey) obj;
        return value == badKey.value;
    }
}

Map<BadKey, String> map = new HashMap<>();
BadKey key = new BadKey(1);
map.put(key, "Value");

key.setValue(2);  // MUTATING THE KEY!
map.get(key);     // Returns null - entry is lost!

// ✅ GOOD: Use immutable keys
class GoodKey {
    private final int value;

    public GoodKey(int value) {
        this.value = value;
    }

    // No setter - immutable
    public int getValue() {
        return value;
    }

    @Override
    public int hashCode() {
        return Objects.hash(value);
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        GoodKey goodKey = (GoodKey) obj;
        return value == goodKey.value;
    }
}
```

### Mistake 5: Not overriding both equals() and hashCode()

```java
// ❌ BAD: Only override equals()
class User {
    private String id;
    private String name;

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        User user = (User) obj;
        return Objects.equals(id, user.id);
    }
    // No hashCode() override!

    // Problem: Two equal users have different hash codes
    // This breaks HashMap's contract
}

Map<User, String> map = new HashMap<>();
User user1 = new User("001", "Alice");
User user2 = new User("001", "Alice");  // Equal to user1

map.put(user1, "Admin");
map.get(user2);  // Returns null! Different hash code = different bucket
```

### Mistake 6: Using == instead of equals() for string comparison

```java
// ❌ BAD: Using == for string comparison
Map<String, String> map = new HashMap<>();
map.put(new String("key"), "value");
map.get("key");  // Might return null if string interning doesn't happen

// ✅ GOOD: Always use equals()
String key = "key";
if (map.containsKey(key)) {  // Uses equals() internally
    String value = map.get(key);
}
```

## 6. Interview Tips

### What Interviewers Expect:

1. **HashMap Internal Working** (Most Common Question)
   - Know the array + linked list/tree structure
   - Explain hash function and bucket calculation
   - Understand collision handling (chaining)
   - Know about treeification (Java 8+)
   - Explain resize process

2. **Time Complexity Questions**
   ```
   ArrayList:
   - get(index): O(1)
   - add(): O(1) amortized (O(n) during resize)
   - add(index, element): O(n)
   - remove(index): O(n)
   - remove(object): O(n)
   - contains(object): O(n)

   LinkedList:
   - get(index): O(n)
   - add(): O(1) at end, O(n) at middle
   - addFirst/addLast: O(1)
   - remove(): O(1) at end, O(n) at middle
   - removeFirst/removeLast: O(1)

   HashMap:
   - get/put/containsKey/remove: O(1) average, O(n) worst case
   ```

3. **When to Use Which Collection**
   - ArrayList: Frequent random access, less frequent modifications
   - LinkedList: Frequent add/remove at ends, queue/deque operations
   - HashMap: Key-value pairs, fast lookup
   - HashSet: Unique elements, membership testing
   - TreeSet: Sorted elements, range queries

4. **Common Follow-up Questions**
   - "Why is HashMap's default capacity 16?"
     - Answer: Power of 2 allows fast bit-wise operations for index calculation
   - "Why is load factor 0.75?"
     - Answer: Balance between space and time. Lower = less collision but more space. Higher = more collision but less space.
   - "What happens if two objects have same hashCode but are not equal?"
     - Answer: They go in same bucket (collision), handled by chaining/tree
   - "What happens if two objects are equal but have different hashCode?"
     - Answer: HashMap breaks - can't find entry
   - "Why does Java 8 convert linked list to tree?"
     - Answer: O(n) linked list degrades to O(log n) tree for better performance with many collisions

5. **Code Writing Expectations**
   - Write clean, readable code
   - Handle null checks when appropriate
   - Use generics correctly
   - Follow Java naming conventions

## 7. Short Revision Summary

### ArrayList
- **Structure**: Dynamic array backing
- **Growth**: 1.5x when full
- **Best for**: Random access, infrequent modifications
- **Time**: get()=O(1), add()=O(1) amortized, remove()=O(n)

### LinkedList
- **Structure**: Doubly-linked list with prev/next pointers
- **Growth**: No resize needed
- **Best for**: Add/remove at ends, queue operations
- **Time**: get()=O(n), addFirst/Last()=O(1), removeFirst/Last()=O(1)

### HashMap
- **Structure**: Array of buckets (linked list/tree)
- **Collision**: Chaining (linked list), treeify at 8 elements
- **Resize**: Double capacity when size > capacity * 0.75
- **Key Contract**: MUST override equals() and hashCode()
- **Time**: get/put/containsKey/remove=O(1) average, O(n) worst

### HashSet
- **Structure**: HashMap internally (keys only)
- **Use case**: Unique elements, membership testing
- **Time**: Same as HashMap (O(1) average)

### Critical Points to Remember:
1. HashMap uses `hash & (n-1)` for index calculation (n = capacity)
2. Right shift (>>> 16) in hash function mixes higher bits
3. Load factor 0.75 = balance between space and time
4. Treeify threshold = 8, untreeify threshold = 6
5. Keys must be immutable or at least hash fields shouldn't change
6. Always override both equals() and hashCode() together
7. Use Iterator or removeIf() when removing during iteration
8. Initialize capacity when size is known
9. Choose collection based on your primary operation (read vs write)
10. Understand worst-case scenarios (e.g., all keys in same bucket)

### Quick Comparison Table:

| Collection | Add | Get | Remove | Contains | Best Use Case |
|------------|-----|-----|--------|----------|---------------|
| ArrayList | O(1)* | O(1) | O(n) | O(n) | Random access |
| LinkedList | O(1)* | O(n) | O(1)* | O(n) | Queue/deque |
| HashMap | O(1)* | O(1)* | O(1)* | O(1)* | Key-value lookup |
| HashSet | O(1)* | - | O(1)* | O(1)* | Unique elements |

\* Amortized/average case

---

**Next Topics to Study:**
- OOP Principles (Encapsulation, Inheritance, Polymorphism, Abstraction)
- Java 8 Streams and Lambdas
- Multithreading Basics
- Spring Framework Fundamentals

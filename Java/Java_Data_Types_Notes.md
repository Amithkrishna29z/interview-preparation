# ☕ Java Data Types — Complete Notes

> 🎯 Everything you need to know about Java data types for interviews!

---

## Table of Contents
1. [Overview of Java Data Types](#overview)
2. [Primitive Data Types](#primitive-data-types)
3. [Non-Primitive Data Types](#non-primitive-data-types)
4. [Wrapper Classes](#wrapper-classes)
5. [Type Casting](#type-casting)
6. [String in Java](#string-in-java)
7. [Arrays](#arrays)
8. [var — Local Variable Type Inference](#var)
9. [Common Interview Questions](#common-interview-questions)
10. [Quick Revision Summary](#quick-revision-summary)

---

## Overview

Java is a **statically typed** language — every variable must have a declared type.

```
Java Data Types
│
├── Primitive (8 types)
│   ├── Numeric Integer  → byte, short, int, long
│   ├── Numeric Decimal  → float, double
│   ├── Character        → char
│   └── Boolean          → boolean
│
└── Non-Primitive (Reference Types)
    ├── String
    ├── Arrays
    ├── Classes / Objects
    └── Interfaces
```

---

## Primitive Data Types

Java has exactly **8 primitive types**. They store values directly (not references).

### Complete Table

| Type | Size | Default | Min Value | Max Value | Example |
|------|------|---------|-----------|-----------|---------|
| `byte` | 1 byte (8 bits) | 0 | -128 | 127 | `byte b = 100;` |
| `short` | 2 bytes (16 bits) | 0 | -32,768 | 32,767 | `short s = 1000;` |
| `int` | 4 bytes (32 bits) | 0 | -2,147,483,648 | 2,147,483,647 | `int i = 100000;` |
| `long` | 8 bytes (64 bits) | 0L | -9.2 × 10^18 | 9.2 × 10^18 | `long l = 100L;` |
| `float` | 4 bytes (32 bits) | 0.0f | ~1.4e-45 | ~3.4e+38 | `float f = 3.14f;` |
| `double` | 8 bytes (64 bits) | 0.0d | ~4.9e-324 | ~1.8e+308 | `double d = 3.14;` |
| `char` | 2 bytes (16 bits) | '\u0000' | 0 | 65,535 | `char c = 'A';` |
| `boolean` | ~1 bit (JVM decides) | false | — | — | `boolean flag = true;` |

### Code Examples

```java
public class PrimitiveTypes {
    public static void main(String[] args) {

        // Integer types
        byte   b = 127;          // max byte value
        short  s = 32767;        // max short value
        int    i = 2147483647;   // max int value
        long   l = 9999999999L;  // must add 'L' suffix

        // Decimal types
        float  f = 3.14f;        // must add 'f' suffix
        double d = 3.14159265;   // default decimal type

        // Character
        char c1 = 'A';           // single quotes only
        char c2 = 65;            // ASCII value — also 'A'
        char c3 = '\n';          // escape character (newline)

        // Boolean
        boolean isActive = true;
        boolean isEmpty  = false;

        // Sizes
        System.out.println(Integer.MAX_VALUE);   // 2147483647
        System.out.println(Integer.MIN_VALUE);   // -2147483648
        System.out.println(Long.MAX_VALUE);      // 9223372036854775807
        System.out.println(Double.MAX_VALUE);    // 1.7976931348623157E308
    }
}
```

### Key Rules
```
✅ int    → default type for whole numbers
✅ double → default type for decimals
✅ long   → requires 'L' suffix:   long x = 100L;
✅ float  → requires 'f' suffix:   float x = 3.14f;
✅ char   → uses single quotes:    char c = 'A';
✅ String → uses double quotes:    String s = "Hello";
```

---

## Non-Primitive Data Types

Also called **Reference Types** — they store a **memory address (reference)** to the object, not the value itself.

```java
// Primitive: value stored directly in variable
int x = 10;   // x holds 10

// Non-Primitive: variable holds address of object in heap
String name = "Amith";   // name holds address → heap stores "Amith"
int[] arr    = {1, 2, 3}; // arr holds address → heap stores the array
```

### Primitive vs Non-Primitive

| Feature | Primitive | Non-Primitive |
|---------|-----------|---------------|
| Stores | Value directly | Memory reference |
| Memory | Stack | Heap (object) + Stack (reference) |
| Default value | 0 / false / '\u0000' | `null` |
| Size | Fixed (e.g. int = 4 bytes) | Dynamic |
| Methods | ❌ None | ✅ Has methods |
| Null possible? | ❌ No | ✅ Yes |
| Examples | `int, double, char` | `String, Array, Object` |

```java
// Non-primitive default is null
String name;         // name = null
int[]  arr;          // arr  = null

// Primitive default is zero/false
int count;           // count = 0
boolean flag;        // flag  = false
```

---

## Wrapper Classes

Every primitive has a corresponding **Wrapper Class** — an Object version of the primitive.

### Primitive → Wrapper Mapping

| Primitive | Wrapper Class |
|-----------|---------------|
| `byte` | `Byte` |
| `short` | `Short` |
| `int` | `Integer` |
| `long` | `Long` |
| `float` | `Float` |
| `double` | `Double` |
| `char` | `Character` |
| `boolean` | `Boolean` |

### Why Wrapper Classes?

```java
// 1. Collections can't store primitives — need wrapper
List<Integer> list = new ArrayList<>();   // ✅ Integer (not int)
list.add(10);                              // autoboxing: int → Integer

// 2. Useful utility methods
int max     = Integer.MAX_VALUE;          // 2147483647
int parsed  = Integer.parseInt("42");     // String → int
String str  = Integer.toString(42);       // int → String
int binary  = Integer.parseInt("1010", 2);// binary "1010" → 10
String bin  = Integer.toBinaryString(10); // 10 → "1010"
String hex  = Integer.toHexString(255);   // 255 → "ff"

// 3. Null handling (primitives can't be null)
Integer age = null;   // ✅ valid
// int age = null;    // ❌ compile error
```

### Autoboxing & Unboxing

```java
// Autoboxing: primitive → Wrapper (automatic)
int    primitive = 42;
Integer wrapper  = primitive;   // auto: int → Integer

// Unboxing: Wrapper → primitive (automatic)
Integer obj = Integer.valueOf(100);
int val     = obj;              // auto: Integer → int

// Common in collections
List<Integer> numbers = new ArrayList<>();
numbers.add(5);        // autoboxing   — int 5 → Integer
int first = numbers.get(0);  // unboxing — Integer → int
```

### ⚠️ Wrapper Comparison Pitfall (Important Interview Topic!)

```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b);       // ✅ true  (cached range: -128 to 127)

Integer c = 128;
Integer d = 128;
System.out.println(c == d);       // ❌ false (new objects outside cache range!)
System.out.println(c.equals(d));  // ✅ true  (always use equals() for Wrappers)

// Rule: ALWAYS use .equals() to compare Wrapper objects, never ==
```

**Why?** Java caches Integer values from **-128 to 127**. Outside this range, `new Integer()` creates a new object each time.

---

## Type Casting

### 1. Widening (Implicit) — Small → Large, Safe, Automatic

```
byte → short → int → long → float → double
char → int

No data loss. Java does it automatically.
```

```java
int    i = 100;
long   l = i;       // int → long  (automatic)
double d = l;       // long → double (automatic)

byte b = 10;
int  x = b;         // byte → int  (automatic)
```

### 2. Narrowing (Explicit) — Large → Small, Risk of Data Loss

```java
double d = 9.99;
int    i = (int) d;     // explicit cast required
System.out.println(i);  // 9 — decimal part lost!

long  l = 12345678901L;
int   i2 = (int) l;     // may lose data if value exceeds int range
System.out.println(i2); // -539222987 — overflow!

int   big = 130;
byte  b   = (byte) big;
System.out.println(b);  // -126 — overflow (byte max = 127)
```

### 3. String Conversions

```java
// int → String
int num = 42;
String s1 = String.valueOf(num);      // "42"
String s2 = Integer.toString(num);    // "42"
String s3 = "" + num;                 // "42" (concatenation trick)

// String → int
String str = "42";
int parsed = Integer.parseInt(str);   // 42
int parsed2 = Integer.valueOf(str);   // 42 (returns Integer, unboxed)

// double → String
double d = 3.14;
String sd = String.valueOf(d);        // "3.14"
String sd2 = Double.toString(d);      // "3.14"

// String → double
String ds = "3.14";
double dv = Double.parseDouble(ds);   // 3.14

// char → int
char c = 'A';
int ascii = (int) c;                  // 65

// int → char
int code = 66;
char letter = (char) code;            // 'B'
```

---

## String in Java

### String is Immutable

```java
String s = "Hello";
s.concat(" World");     // Creates NEW string — original s unchanged!
System.out.println(s);  // Still "Hello"

s = s.concat(" World"); // Must reassign to keep result
System.out.println(s);  // "Hello World"
```

### String Pool (Interning)

```java
String a = "Java";         // stored in String Pool
String b = "Java";         // reuses same object from pool
String c = new String("Java"); // creates NEW object on heap (outside pool)

System.out.println(a == b);       // ✅ true  (same pool reference)
System.out.println(a == c);       // ❌ false (different objects)
System.out.println(a.equals(c));  // ✅ true  (same content)
```

### String vs StringBuilder vs StringBuffer

| Feature | String | StringBuilder | StringBuffer |
|---------|--------|---------------|--------------|
| Mutable? | ❌ Immutable | ✅ Mutable | ✅ Mutable |
| Thread-safe? | ✅ Yes | ❌ No | ✅ Yes |
| Speed | Slow (new object each concat) | ✅ Fastest | Slower than SB |
| Use when | Fixed text | Single-thread concat | Multi-thread concat |

```java
// String concatenation in loop — BAD (creates new object each iteration)
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;   // ❌ 1000 new String objects created!
}

// StringBuilder — GOOD
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);  // ✅ modifies same object
}
String result = sb.toString();

// StringBuilder methods
StringBuilder sb2 = new StringBuilder("Hello");
sb2.append(" World");        // "Hello World"
sb2.insert(5, ",");          // "Hello, World"
sb2.delete(5, 6);            // "Hello World"
sb2.reverse();               // "dlroW olleH"
sb2.replace(0, 5, "Hi");     // "Hi olleH" → "Hi World"
System.out.println(sb2.length()); // length
```

### Common String Methods

```java
String s = "Hello, World!";

s.length()                // 13
s.charAt(0)               // 'H'
s.indexOf('o')            // 4
s.lastIndexOf('o')        // 8
s.substring(7)            // "World!"
s.substring(7, 12)        // "World"
s.toLowerCase()           // "hello, world!"
s.toUpperCase()           // "HELLO, WORLD!"
s.trim()                  // removes leading/trailing spaces
s.strip()                 // same as trim() but handles Unicode
s.replace('l', 'r')       // "Herro, Worrd!"
s.contains("World")       // true
s.startsWith("Hello")     // true
s.endsWith("!")           // true
s.isEmpty()               // false
s.isBlank()               // false (Java 11+)
s.split(", ")             // ["Hello", "World!"]
s.equals("Hello, World!") // true
s.equalsIgnoreCase("hello, world!") // true
String.join("-", "a","b","c")       // "a-b-c"
```

---

## Arrays

### Declaration & Initialization

```java
// Method 1: Declare then assign
int[] arr;
arr = new int[5];          // {0, 0, 0, 0, 0} (default int = 0)

// Method 2: Declare + size
int[] arr2 = new int[5];

// Method 3: Declare + initialize
int[] arr3 = {10, 20, 30, 40, 50};

// Method 4: new keyword with values
int[] arr4 = new int[]{10, 20, 30};

// String array
String[] names = {"Alice", "Bob", "Charlie"};

// Default values after new int[n]
// int[]     → 0
// double[]  → 0.0
// boolean[] → false
// String[]  → null
// char[]    → '\u0000'
```

### 2D Arrays

```java
// 3 rows, 4 columns
int[][] matrix = new int[3][4];

// Initialize with values
int[][] grid = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};

// Access element
System.out.println(grid[1][2]);   // 6 (row 1, col 2)

// Iterate 2D array
for (int row = 0; row < grid.length; row++) {
    for (int col = 0; col < grid[row].length; col++) {
        System.out.print(grid[row][col] + " ");
    }
    System.out.println();
}
```

### Array Utility Methods

```java
import java.util.Arrays;

int[] arr = {5, 3, 1, 4, 2};

// Sort
Arrays.sort(arr);                    // {1, 2, 3, 4, 5}

// Search (array must be sorted first)
int idx = Arrays.binarySearch(arr, 3); // returns index of 3

// Fill
Arrays.fill(arr, 0);                 // {0, 0, 0, 0, 0}

// Copy
int[] copy = Arrays.copyOf(arr, 3);          // copy first 3 elements
int[] range = Arrays.copyOfRange(arr, 1, 4); // copy index 1 to 3

// Convert to String (for printing)
System.out.println(Arrays.toString(arr));    // [1, 2, 3, 4, 5]

// Compare
int[] a = {1, 2, 3};
int[] b = {1, 2, 3};
System.out.println(Arrays.equals(a, b));     // true
System.out.println(a == b);                  // false (different objects)
```

---

## var

### Local Variable Type Inference (Java 10+)

```java
// Instead of writing the type explicitly
int    count  = 10;
String name   = "Amith";
double price  = 99.99;

// Use var — compiler infers the type
var count  = 10;       // inferred as int
var name   = "Amith";  // inferred as String
var price  = 99.99;    // inferred as double

// Works with complex types
var list = new ArrayList<String>();   // ArrayList<String>
var map  = new HashMap<String, Integer>(); // HashMap<String, Integer>

// Works in for-each loops
var numbers = List.of(1, 2, 3);
for (var n : numbers) {
    System.out.println(n);
}
```

### var Restrictions

```java
// ❌ Cannot be used for fields (class-level variables)
// ❌ Cannot be used for method parameters
// ❌ Cannot be used for return types
// ❌ Cannot be initialized to null (type can't be inferred)

var x;          // ❌ No initializer
var y = null;   // ❌ Can't infer type from null
```

---

## Common Interview Questions

### Q1: What is the difference between int and Integer?

```
int     → primitive, stores value directly, can't be null, no methods
Integer → Wrapper class, stores reference, can be null, has methods

int x = 10;          // 4 bytes on stack, value = 10
Integer y = 10;      // reference on stack → Integer object on heap

Use int    → performance-critical code (no overhead)
Use Integer → Collections (List, Map), nullable fields, utility methods
```

### Q2: Why is String immutable in Java?

```
1. Security — Class names, DB passwords, file paths won't be modified unexpectedly
2. Thread Safety — Multiple threads can safely share same String object
3. String Pool — Immutability enables caching in pool (saves memory)
4. HashCode Caching — String caches hashCode(); safe since value never changes
```

### Q3: What happens when you do `int x = Integer.MAX_VALUE + 1`?

```java
int x = Integer.MAX_VALUE + 1;
System.out.println(x);   // -2147483648 (Integer.MIN_VALUE)

// This is called INTEGER OVERFLOW
// Binary wraps around: 01111...1111 + 1 = 10000...0000 = MIN_VALUE

// Fix: use long
long y = (long) Integer.MAX_VALUE + 1;
System.out.println(y);   // 2147483648
```

### Q4: What is the default value of different data types?

```java
// In class fields (instance/static variables):
byte   → 0
short  → 0
int    → 0
long   → 0L
float  → 0.0f
double → 0.0d
char   → '\u0000' (null character)
boolean → false
Object/String/Array → null

// Local variables have NO default — must initialize before use!
int x;
System.out.println(x);  // ❌ Compile Error: variable might not have been initialized
```

### Q5: float vs double — which to use?

```
float  → 4 bytes, ~7 decimal digits precision
double → 8 bytes, ~15 decimal digits precision

Use double by default (it's Java's default for decimals)
Use float only when memory is extremely tight (e.g., large arrays of decimals)

⚠️ Floating-point imprecision:
double d = 0.1 + 0.2;
System.out.println(d);   // 0.30000000000000004 (not exactly 0.3!)

Fix: Use BigDecimal for financial calculations
BigDecimal result = new BigDecimal("0.1").add(new BigDecimal("0.2"));
System.out.println(result);   // 0.3 (exact!)
```

### Q6: What is the char data type in Java?

```java
char c = 'A';           // Single character, single quotes
char newline = '\n';    // Escape character
char tab     = '\t';
char unicode = '\u0041'; // Unicode for 'A'

// char is actually a 16-bit unsigned integer (0 to 65535)
// Can do arithmetic on chars
char ch = 'A';
System.out.println(ch + 1);           // 66 (int arithmetic)
System.out.println((char)(ch + 1));   // 'B' (cast back to char)

// Iterating over a String
String word = "Hello";
for (char letter : word.toCharArray()) {
    System.out.println(letter);
}
```

### Q7: What is type promotion in expressions?

```java
byte b1 = 10;
byte b2 = 20;
// byte b3 = b1 + b2;   // ❌ Compile error! Result is int
int  b3 = b1 + b2;      // ✅ Java promotes byte to int in arithmetic

// Rule: In expressions, byte/short/char are promoted to int
// If any operand is long → result is long
// If any operand is float → result is float
// If any operand is double → result is double

int   i = 10;
long  l = 20L;
float f = 30.0f;

long  r1 = i + l;   // int + long = long
float r2 = l + f;   // long + float = float
```

---

## Quick Revision Summary

### 🔑 8 Primitive Types

```
Integer family  → byte(1B) < short(2B) < int(4B) < long(8B)
Decimal family  → float(4B) < double(8B)
Character       → char (2B, 0 to 65535)
Boolean         → boolean (true/false)

Suffixes needed: long → L,  float → f
```

### 🔑 Key Rules

```
1. int is default for whole numbers
2. double is default for decimals
3. Always use .equals() for String/Wrapper comparison (not ==)
4. Integer cache: -128 to 127 (== works), outside range (== fails)
5. String is immutable — methods return new String
6. Use StringBuilder for string building in loops
7. Use BigDecimal for precise financial calculations
8. Local variables have no default — must initialize before use
9. Fields default to 0 / false / null
10. Widening is automatic; Narrowing requires explicit cast
```

### 🔑 Casting Quick Reference

```
Widening (automatic):  byte → short → int → long → float → double
Narrowing (explicit):  double → (int) → possible data loss

String ↔ int:
  int    → String:  String.valueOf(42)  or  "" + 42
  String → int:     Integer.parseInt("42")
```

### 🔑 Interview Tips

1. **"Why is String immutable?"** → Security, Thread Safety, String Pool, HashCode caching
2. **"int vs Integer"** → primitive vs object, null, methods, boxing overhead
3. **"== vs equals() for Integer"** → cache range -128 to 127
4. **"What is autoboxing?"** → automatic int ↔ Integer conversion
5. **"float vs double"** → prefer double; use BigDecimal for money
6. **"Overflow"** → MAX_VALUE + 1 wraps to MIN_VALUE

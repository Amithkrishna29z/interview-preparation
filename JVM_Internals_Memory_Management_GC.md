# JVM Internals, Memory Management & Garbage Collection — Study Guide

## Overview

The JVM (Java Virtual Machine) is the engine that actually *runs* your Java program. Understanding how it loads classes, where it stores your objects, and how it cleans up memory is one of the biggest differentiators in a Java backend interview — it separates someone who "writes Java" from someone who "understands Java".

This guide covers the modern HotSpot JVM (Java 8 through 21). The most important historical change to remember: **PermGen was removed in Java 8 and replaced by Metaspace** (which lives in native memory, not the heap).

**Think of the JVM like a universal translator at the United Nations:**
You write your speech in one language (Java). You don't care what language each listener speaks (Windows, Linux, Mac, ARM, x86). The translator (JVM) takes your one speech and instantly converts it into whatever each listener understands. You **write once**, and it **runs anywhere** there's a translator.

---

## Table of Contents

1. [Overview & The Journey of Java Code](#overview--the-journey-of-java-code)
2. [JVM Architecture](#jvm-architecture)
3. [Class Loading](#class-loading)
4. [Runtime Memory Areas](#runtime-memory-areas)
5. [Stack vs Heap](#stack-vs-heap)
6. [Heap Structure & Object Lifecycle](#heap-structure--object-lifecycle)
7. [Garbage Collection](#garbage-collection)
8. [GC Algorithms](#gc-algorithms)
9. [Memory Leaks in Java](#memory-leaks-in-java)
10. [Errors: OutOfMemoryError & StackOverflowError](#errors-outofmemoryerror--stackoverflowerror)
11. [JVM Tuning Flags](#jvm-tuning-flags)
12. [Monitoring & Troubleshooting Tools](#monitoring--troubleshooting-tools)
13. [Common Interview Questions](#common-interview-questions)
14. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Overview & The Journey of Java Code

The famous Java slogan is **"Write Once, Run Anywhere" (WORA)**. Here's how that magic actually works.

When you compile a C++ program, the compiler turns it directly into machine code for *your* CPU — and that file only runs on that kind of machine. Java does something cleverer: it compiles to an **intermediate language called bytecode**, which any JVM on any platform can understand.

```
┌──────────────┐   javac    ┌──────────────┐   loaded by   ┌──────────────────────┐
│  Hello.java  │ ─────────► │  Hello.class │ ────────────► │        JVM           │
│ (source code,│  (compile) │  (bytecode,  │               │  interprets +        │
│  human text) │            │  platform-   │               │  JIT-compiles to     │
│              │            │  independent)│               │  NATIVE machine code │
└──────────────┘            └──────────────┘               └──────────┬───────────┘
                                                                      │
                                                                      ▼
                                                            ┌──────────────────────┐
                                                            │  CPU runs native     │
                                                            │  instructions        │
                                                            │ (Windows/Linux/Mac)  │
                                                            └──────────────────────┘
```

Step by step:

1. **`.java`** — You write human-readable source code.
2. **`javac`** — The Java compiler converts your source into **bytecode** (a `.class` file). Bytecode is NOT machine code — it's a set of instructions only the JVM understands.
3. **`.class` bytecode** — This file is identical no matter which OS you ship it to. The same `.class` runs on Windows, Linux, and Mac.
4. **JVM** — When you run the program, the JVM **interprets** the bytecode (executes it line by line) and, for "hot" code that runs often, the **JIT compiler** converts it to fast **native machine code**.
5. **Native** — The CPU finally executes real machine instructions.

**Key terms:**
- **JDK** (Java Development Kit) = JRE + developer tools (`javac`, `jdb`, etc.). You need this to *build*.
- **JRE** (Java Runtime Environment) = JVM + core libraries. You need this to *run*.
- **JVM** (Java Virtual Machine) = the actual engine that executes bytecode.

```
JDK  ⊃  JRE  ⊃  JVM
(build)  (run)  (execute)
```

> **Why bytecode and not direct machine code?** Because then Oracle/OpenJDK only has to write *one* JVM per platform, and your single compiled program runs on all of them. The platform-specific part is the JVM, not your code.

---

## JVM Architecture

The JVM has **three big subsystems** that work together:

1. **ClassLoader Subsystem** — finds, loads, and prepares your `.class` files.
2. **Runtime Data Areas** — the memory regions where everything lives (heap, stacks, etc.).
3. **Execution Engine** — actually runs the bytecode (Interpreter + JIT Compiler + Garbage Collector).

```
                        ┌───────────────────────────────────────────────┐
        Hello.class ──► │            1. CLASSLOADER SUBSYSTEM             │
        (bytecode)      │   Loading → Linking → Initialization            │
                        └───────────────────────┬───────────────────────┘
                                                 │ loaded classes
                                                 ▼
        ┌─────────────────────────────────────────────────────────────────────────┐
        │                       2. RUNTIME DATA AREAS (Memory)                      │
        │                                                                           │
        │   ┌──────────── SHARED across all threads ────────────┐                  │
        │   │   HEAP (objects, instances)   │   METASPACE        │                  │
        │   │                               │   (class metadata) │                  │
        │   └───────────────────────────────┴────────────────────┘                 │
        │                                                                           │
        │   ┌──────────── PER-THREAD (one set per thread) ──────────────────────┐   │
        │   │   JVM Stack  │  PC Register  │  Native Method Stack               │   │
        │   └────────────────────────────────────────────────────────────────────┘ │
        └─────────────────────────────────────────┬─────────────────────────────────┘
                                                   │
                                                   ▼
                        ┌───────────────────────────────────────────────┐
                        │            3. EXECUTION ENGINE                  │
                        │   ┌───────────┐ ┌──────────┐ ┌──────────────┐  │
                        │   │Interpreter│ │   JIT    │ │  Garbage     │  │
                        │   │(line-by-  │ │ Compiler │ │  Collector   │  │
                        │   │ line)     │ │(hot code)│ │  (cleanup)   │  │
                        │   └───────────┘ └──────────┘ └──────────────┘  │
                        └───────────────────────────────────────────────┘
```

**The Execution Engine's three helpers:**
- **Interpreter** — Reads and executes bytecode one instruction at a time. Starts fast but is slow for repeated code.
- **JIT (Just-In-Time) Compiler** — Watches for "hot spots" (methods/loops run many times) and compiles them to native code so they run much faster. This is why the JVM is called **HotSpot**.
- **Garbage Collector (GC)** — Automatically frees memory used by objects nobody needs anymore.

---

## Class Loading

Before any class can be used, it must be loaded into memory. This happens in **3 phases**.

**Think of class loading like a new employee's first day:**
- **Loading** = HR finds your file and brings it into the building.
- **Linking** = They verify your documents are real (verify), set up your default desk and badge (prepare), and connect you to the teams you'll work with (resolve).
- **Initialization** = You actually start working — your `static` setup runs.

### The 3 Phases

```
┌────────────┐      ┌──────────────────────────────────┐      ┌──────────────────┐
│  LOADING   │ ───► │             LINKING              │ ───► │ INITIALIZATION   │
│ read .class│      │  Verify → Prepare → Resolve      │      │ run static blocks│
│ into memory│      │                                  │      │ & static = values│
└────────────┘      └──────────────────────────────────┘      └──────────────────┘
```

1. **Loading** — The ClassLoader reads the `.class` bytecode and creates a `Class` object in memory (stored in Metaspace).
2. **Linking** — three sub-steps:
   - **Verify** — Checks the bytecode is valid and safe (no corrupted or malicious instructions). This is a key security feature.
   - **Prepare** — Allocates memory for `static` fields and sets them to **default values** (0, `false`, `null`) — not your assigned values yet.
   - **Resolve** — Replaces symbolic references (names like `"java/lang/String"`) with actual direct memory references.
3. **Initialization** — Runs `static` initializer blocks and assigns the **real** values to static fields. This is when `static int x = 5;` actually becomes 5.

```java
public class Config {
    static int count = 10;            // PREPARE sets count = 0; INITIALIZATION sets count = 10
    static { System.out.println("init"); }  // runs during INITIALIZATION only
}
```

### The ClassLoader Hierarchy

There are three main classloaders, arranged as parent → child:

```
┌────────────────────────────────────────────────────┐
│  Bootstrap ClassLoader  (written in C/C++, no parent)│  loads core JDK: java.lang.*, java.util.*
└──────────────────────────┬─────────────────────────┘   (rt.jar / java.base module)
                           │ parent of
                           ▼
┌────────────────────────────────────────────────────┐
│  Platform ClassLoader   (was "Extension" pre-Java 9) │  loads JDK extension/platform modules
└──────────────────────────┬─────────────────────────┘
                           │ parent of
                           ▼
┌────────────────────────────────────────────────────┐
│  Application ClassLoader (a.k.a. System ClassLoader) │  loads YOUR classes from the classpath
└────────────────────────────────────────────────────┘
```

> **Note:** Before Java 9 the middle one was called the **Extension ClassLoader**. From Java 9 onwards (with the module system) it's the **Platform ClassLoader**. Interviewers may say either name.

### The Parent Delegation Model

When a class needs loading, the request goes **UP** to the parent first. Each classloader asks its parent "can you load this?" before trying itself.

```
Application CL gets request for "java.lang.String"
   │  "Hey parent, can YOU load this?"
   ▼
Platform CL
   │  "Hey parent, can YOU load this?"
   ▼
Bootstrap CL  → "Yes! java.lang.String is a core class. I'll load it." ✅
```

The request only falls back down to the child if every parent says "no, not mine".

**Think of it like asking your boss before doing something:**
You ask your manager, who asks the director, who asks the CEO. If the CEO can handle it, they do. Only if nobody above you can handle it do *you* take care of it yourself.

**Why does parent delegation exist?**
1. **Security** — Nobody can replace core classes. If an attacker ships a malicious `java.lang.String`, the Application ClassLoader delegates up, and the Bootstrap ClassLoader loads the *real* one first. The fake never gets a chance.
2. **No duplicate core classes** — `java.lang.Object` is loaded exactly once by the Bootstrap loader, so there's a single consistent definition everywhere.

---

## Runtime Memory Areas

When the JVM runs, it divides memory into several regions. The key distinction interviewers test: **which areas are shared across all threads, and which are private per-thread.**

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          JVM MEMORY (one process)                          │
│                                                                            │
│   ════════════ SHARED across ALL threads ════════════                      │
│   ┌────────────────────────────┐   ┌──────────────────────────────────┐   │
│   │            HEAP             │   │           METASPACE              │   │
│   │  All objects & instances   │   │  Class metadata, method info,    │   │
│   │  (everything you 'new')    │   │  static vars. OFF-HEAP (native)  │   │
│   │  GC works here             │   │  since Java 8 (replaced PermGen) │   │
│   └────────────────────────────┘   └──────────────────────────────────┘   │
│                                                                            │
│   ════════════ PER-THREAD (each thread gets its own) ════════════          │
│   ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────────┐  │
│   │  JVM Stack   │  │  PC Register │  │     Native Method Stack        │  │
│   │ method calls,│  │ address of   │  │ for native (C/C++) methods     │  │
│   │ local vars,  │  │ current      │  │ called via JNI                 │  │
│   │ frames (LIFO)│  │ instruction  │  │                                │  │
│   └──────────────┘  └──────────────┘  └────────────────────────────────┘  │
│   (Thread 1 has its own set; Thread 2 has another set; etc.)               │
└──────────────────────────────────────────────────────────────────────────┘
```

### Each area explained

| Area | Shared or Per-Thread | What it stores | Lifetime |
|---|---|---|---|
| **Heap** | **Shared** | All objects & arrays (everything created with `new`), instance variables | Whole application; cleaned by GC |
| **JVM Stack** | **Per-thread** | Stack frames: method calls, local variables, partial results, references to heap objects | Frame lives only during the method call |
| **Metaspace** | **Shared** | Class metadata, method bytecode, static variables, runtime constant pool. Lives in **native memory** (off-heap) since Java 8 | Until classes are unloaded |
| **PC Register** | **Per-thread** | The address of the bytecode instruction the thread is currently executing | Lifetime of the thread |
| **Native Method Stack** | **Per-thread** | Used when Java calls native C/C++ methods (via JNI) | Lifetime of the thread |

> **Memorize:** Heap and Metaspace are **shared**. Stack, PC Register, and Native Method Stack are **per-thread**. This is a guaranteed interview question.

> **Metaspace vs PermGen:** Before Java 8, class metadata lived in **PermGen**, a fixed-size region *inside the heap*. It caused frequent `OutOfMemoryError: PermGen space`. Java 8 replaced it with **Metaspace**, which lives in **native memory** and grows automatically by default. This is why you rarely hit Metaspace OOM unless you set a limit or leak classloaders.

---

## Stack vs Heap

This is the **single most-asked JVM interview question.** Get this rock solid.

**Think of it like this:**
- **Stack = a stack of plates** in a cafeteria. You can only add or remove from the **top** (Last-In-First-Out). It's small, super fast, and automatically managed. Each thread carries its own stack of plates.
- **Heap = a giant shared warehouse.** Anyone (any thread) can store big items here. It's large but slower to manage, and a cleanup crew (GC) periodically removes items nobody can reach anymore.

### Comparison Table

| Aspect | Stack | Heap |
|---|---|---|
| **What's stored** | Local variables, method call frames, references to heap objects, primitives declared locally | Objects, instances, arrays, instance variables (everything `new`'d) |
| **Scope** | Per-thread (each thread has its own) | Shared across all threads |
| **Lifetime** | Lives only during the method call; freed automatically when method returns | Lives until no longer reachable; freed by Garbage Collector |
| **Access speed** | Very fast (just move the stack pointer) | Slower (allocation + GC overhead) |
| **Memory management** | Automatic (LIFO push/pop) | Garbage Collector |
| **Thread safety** | Inherently thread-safe (each thread has its own) | NOT thread-safe (shared — needs synchronization) |
| **Size** | Small (default ~512KB–1MB per thread) | Large (can be GBs) |
| **Error when full** | `StackOverflowError` | `OutOfMemoryError: Java heap space` |
| **Controlled by flag** | `-Xss` | `-Xms` / `-Xmx` |

### Walk through a tiny example

```java
public void run() {
    int x = 10;                       // x is a PRIMITIVE local → lives on the STACK
    Person p = new Person("Alice");   // 'p' (the reference) is on the STACK;
                                      // the actual Person OBJECT is on the HEAP
    int[] nums = {1, 2, 3};           // 'nums' reference on STACK; the ARRAY on the HEAP
}
```

What memory looks like during `run()`:

```
        STACK (this thread)                    HEAP (shared)
   ┌──────────────────────────┐         ┌──────────────────────────────┐
   │ run() frame:             │         │  Person object               │
   │   x      = 10            │         │    name = "Alice"  ───────┐   │
   │   p      = ref ──────────┼────────►│                           │   │
   │   nums   = ref ──────────┼───┐     │  String "Alice"  ◄────────┘   │
   └──────────────────────────┘   │     │                              │
                                  └────►│  int[] array  [1, 2, 3]      │
                                        └──────────────────────────────┘
```

Key insight:
- `x` (a primitive) lives entirely on the stack.
- `p` and `nums` are **references** (like an address) on the stack, but the **objects they point to** live on the heap.
- When `run()` returns, the whole stack frame (including `x`, `p`, `nums`) is popped and gone. But the `Person` object remains on the heap until GC notices nothing points to it anymore.

> **Interview gold:** "Where do local primitives live? Where do objects live?" → Local primitives and references live on the **stack**; the objects themselves always live on the **heap**.

---

## Heap Structure & Object Lifecycle

The heap isn't one big blob — it's divided into **generations** based on a powerful observation called the **Weak Generational Hypothesis**:

> **Most objects die young.** The vast majority of objects (loop variables, temporary strings, request objects) become garbage almost immediately. A small minority (caches, config, long-lived service objects) survive a long time.

So the JVM optimizes by separating short-lived from long-lived objects.

```
┌──────────────────────────────────────── HEAP ──────────────────────────────────────────┐
│                                                                                          │
│   ┌──────────────── YOUNG GENERATION ────────────────┐   ┌──────── OLD GENERATION ────┐ │
│   │                                                   │   │       (Tenured)            │ │
│   │  ┌──────────────┐  ┌────────┐  ┌────────┐         │   │  long-lived objects that   │ │
│   │  │     EDEN     │  │  S0    │  │  S1    │         │   │  survived many GC cycles   │ │
│   │  │ new objects  │  │Survivor│  │Survivor│         │   │  (caches, singletons,      │ │
│   │  │ born here    │  │ space  │  │ space  │         │   │   session data)            │ │
│   │  └──────────────┘  └────────┘  └────────┘         │   │                            │ │
│   │  (Minor GC cleans here — fast & frequent)         │   │ (Major/Full GC — slow,     │ │
│   │                                                   │   │  infrequent)               │ │
│   └───────────────────────────────────────────────────┘   └────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### The life of an object

```
1. new Object()           → allocated in EDEN
        │
2. Eden fills up          → Minor GC runs
        │                    survivors copied to S0 (or S1), dead objects discarded
        ▼
3. Survives more Minor GCs → bounces between S0 ↔ S1, "age" counter increments each time
        │
4. age reaches threshold  → PROMOTED to Old Generation (default threshold ~15, see -XX:MaxTenuringThreshold)
        │
5. Lives in Old Gen       → only cleaned by Major/Full GC (rare, but more expensive)
```

**Why two survivor spaces (S0 and S1)?**
At any moment, **one is always empty.** During a Minor GC, live objects from Eden + the *occupied* survivor space are all copied into the *empty* survivor space. Then the roles flip. This "copying collector" approach automatically compacts memory (no fragmentation) and is very fast.

**Think of the heap like a hospital:**
- **Eden** = the emergency intake ward. Everyone arrives here. Most are treated and discharged quickly (die young).
- **Survivor spaces (S0/S1)** = the observation rooms. Patients who need more monitoring get moved here, possibly several times.
- **Old Generation** = long-term care. Patients who've been around a long time are clearly staying — move them somewhere stable so we stop re-checking them constantly.

This is the whole point: GC checks Eden constantly (cheap, lots of garbage), but rarely bothers the Old Gen (expensive, mostly live objects).

---

## Garbage Collection

### What it is and why Java has it

In languages like C/C++, you must manually `free()` every piece of memory you allocate. Forget to free → **memory leak**. Free twice or free too early → **crash / dangling pointer**. This is a huge source of bugs.

Java automates this with the **Garbage Collector**: it automatically finds objects nobody needs anymore and reclaims their memory. You never call `free()` — you just stop referencing an object, and eventually the GC cleans it up.

**Think of GC like an office cleaner who follows one rule:**
"I only throw away things that nobody can still reach." If there's any chain of references leading to an object — from a variable in use, a static field, etc. — the cleaner leaves it alone. The moment *nothing* can reach it, it's trash.

### Reachability & GC Roots

An object is **alive** if it is **reachable** — meaning there's a chain of references from a **GC Root** to it. If no GC Root can reach an object, it's garbage.

**GC Roots** are the "anchors" that are always considered alive:
- **Local variables** on any thread's stack (currently executing methods)
- **Static variables** of loaded classes
- **Active threads**
- **JNI references** (objects referenced from native code)

```
GC ROOTS                          HEAP OBJECTS
┌──────────────┐
│ Stack local  │ ──────► [ A ] ──────► [ B ]        ← A and B are REACHABLE (alive)
├──────────────┤
│ Static field │ ──────► [ C ]                      ← C is REACHABLE (alive)
└──────────────┘
                          [ D ] ──────► [ E ]        ← D and E: NOTHING points to them
                                                       → UNREACHABLE → garbage, will be collected
```

Notice that **D points to E** — they reference each other. But since no GC Root reaches them, *both* are garbage. This is why Java's reachability-based GC handles **circular references** correctly (unlike simple reference counting).

### The Core Algorithm: Mark → Sweep → Compact

```
1. MARK     — Start from GC Roots, follow every reference, and mark all reachable objects as "alive".
              ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐
              │ A ✓ │───►│ B ✓ │    │ C   │───►│ D   │   (✓ = marked alive; C,D not reachable)
              └─────┘    └─────┘    └─────┘    └─────┘

2. SWEEP    — Reclaim memory of all UNMARKED (unreachable) objects.
              ┌─────┐    ┌─────┐    ┌ ─ ─ ┐    ┌ ─ ─ ┐
              │ A ✓ │    │ B ✓ │    │ free│    │ free│   (C, D memory reclaimed → holes/fragmentation)
              └─────┘    └─────┘    └ ─ ─ ┘    └ ─ ─ ┘

3. COMPACT  — Slide surviving objects together to remove gaps (defragmentation).
              ┌─────┐┌─────┐
              │ A ✓ ││ B ✓ │ [────── one big contiguous free block ──────]
              └─────┘└─────┘
```

- **Mark** finds what's alive.
- **Sweep** deletes what's dead.
- **Compact** removes the resulting "holes" so future allocations are fast (no fragmentation). Not every collector compacts, but generational young-gen collection effectively does via copying.

### Minor GC vs Major/Full GC

| | Minor GC | Major / Full GC |
|---|---|---|
| **Where** | Young Generation (Eden + Survivors) | Old Generation (Major); whole heap incl. Metaspace (Full) |
| **Frequency** | Often | Rare |
| **Speed** | Fast (little memory, lots of dead objects) | Slow (large area, mostly live objects) |
| **Pause impact** | Short stop-the-world pause | Longer stop-the-world pause |

### Stop-the-World (STW)

Most GC phases require **pausing all application threads** so the GC can safely walk the object graph without things changing under it. This is called a **stop-the-world pause**.

**Think of it like a librarian doing inventory:** to count books accurately, they briefly lock the doors so nobody moves books around while counting. Short pauses are fine; long ones (a Full GC on a huge heap) freeze your app and cause latency spikes. Reducing STW pause time is the main goal of modern collectors like G1, ZGC, and Shenandoah.

---

## GC Algorithms

Different collectors make different trade-offs between **throughput** (total work done), **latency** (pause times), and **footprint** (memory used).

| Collector | How it works | Trade-off / When to pick | Flag |
|---|---|---|---|
| **Serial GC** | Single thread does all GC work; stops the world fully | Tiny heaps, single-CPU, small apps/containers | `-XX:+UseSerialGC` |
| **Parallel GC** (Throughput) | Multiple threads do GC in parallel; still stop-the-world | Maximize total throughput when occasional longer pauses are OK (batch jobs) | `-XX:+UseParallelGC` |
| **CMS** (Concurrent Mark-Sweep) | Does most marking concurrently with the app to reduce pauses; no compaction (fragments) | **Deprecated in Java 9, removed in Java 14.** Don't pick it — use G1 | `-XX:+UseConcMarkSweepGC` |
| **G1 GC** (Garbage-First) | Splits heap into many equal **regions**; collects regions with the most garbage first; mostly concurrent; predictable pause targets | **Default since Java 9.** Balanced latency + throughput for most server apps | `-XX:+UseG1GC` |
| **ZGC** | Concurrent, region-based; sub-millisecond pauses regardless of heap size | Very large heaps (TBs) needing ultra-low latency | `-XX:+UseZGC` |
| **Shenandoah** | Concurrent compaction; pause times independent of heap size | Low-latency apps on large heaps (Red Hat OpenJDK) | `-XX:+UseShenandoahGC` |

> **One-liner summary:** Serial = tiny apps. Parallel = max throughput, pauses OK. CMS = obsolete. **G1 = the modern default, good all-rounder.** ZGC / Shenandoah = huge heaps + you can't tolerate pauses.

---

## Memory Leaks in Java

**Yes — Java can absolutely have memory leaks**, even with a garbage collector. A leak happens when objects you no longer need are still **reachable** from a GC Root, so the GC won't collect them. They pile up until you hit `OutOfMemoryError`.

**Think of it like:** the office cleaner only throws away unreachable trash. If you accidentally keep a "reference rope" tied to a pile of junk, the cleaner sees it's "still in use" and leaves it forever — even though you'll never look at it again.

### Common causes

| Cause | Why it leaks |
|---|---|
| **Static collections that grow forever** | A `static Map`/`List` lives for the whole app. If you keep `add()`-ing and never remove, it grows without bound. Static = always reachable from a GC Root. |
| **Unclosed resources** | Streams, connections, sockets not closed (use try-with-resources). Holds native + heap memory. |
| **Listeners / callbacks not deregistered** | You register a listener but never remove it. The event source keeps a reference, keeping the (often large) listener alive. |
| **ThreadLocal misuse** | In thread pools, threads are reused and live forever. A `ThreadLocal` value never removed (`remove()`) stays tied to that long-lived thread → leak. |
| **Caches without eviction** | A cache that only ever grows. Use bounded caches with eviction (e.g., Caffeine, LRU, `WeakHashMap`). |
| **Inner / anonymous classes** | A non-static inner class holds an implicit reference to its outer object, keeping it alive longer than expected. |

### How to spot them

```
Symptom: heap usage trends UP over time and never comes back down,
         Full GCs become more frequent, eventually OutOfMemoryError.

Steps:
  1. -verbose:gc / GC logs  → see if memory keeps climbing after Full GC
  2. jmap -histo <pid>      → which class has a growing instance count?
  3. Take a heap dump (jmap -dump or -XX:+HeapDumpOnOutOfMemoryError)
  4. Open in Eclipse MAT / VisualVM → run "Dominator Tree" / leak suspect report
  5. Trace the reference chain back to the GC Root that's wrongly holding the objects
```

---

## Errors: OutOfMemoryError & StackOverflowError

These are two different errors in two different memory areas. Don't confuse them.

### OutOfMemoryError (heap / metaspace ran out)

`OutOfMemoryError` is thrown when the JVM cannot allocate memory and GC can't free enough. Common variants:

| Message | Cause | Fix |
|---|---|---|
| `Java heap space` | Too many live objects (or a leak) for the heap size | Increase `-Xmx`; fix the leak; reduce object retention |
| `GC overhead limit exceeded` | JVM spent >98% of time in GC reclaiming <2% of heap — basically thrashing | Same as above; the heap is effectively full of live objects |
| `Metaspace` | Too many classes loaded (e.g., classloader leak, dynamic class generation) | Increase `-XX:MaxMetaspaceSize`; fix classloader leaks |
| `Unable to create new native thread` | OS hit thread limit (each thread reserves stack memory) | Create fewer threads; reduce `-Xss`; raise OS limits |

### StackOverflowError (stack ran out)

`StackOverflowError` happens when a thread's **stack** is exhausted — almost always from **too-deep or infinite recursion**. Each method call pushes a new frame; if they never return, frames pile up until the stack is full.

```java
public int countDown(int n) {
    return countDown(n - 1);   // BUG: no base case! recurses forever
}                              // each call pushes a new frame → StackOverflowError
```

```
STACK fills with frames that never pop:
  ┌──────────────────┐
  │ countDown(-9998) │  ← keeps pushing...
  │ countDown(-9999) │
  │ ...              │
  │ countDown(2)     │
  │ countDown(1)     │
  │ countDown(0)     │  ← never returns → BOOM: StackOverflowError
  └──────────────────┘
```

**Fix:** Add a proper base case to recursion, or convert deep recursion into iteration (a loop). If legitimately deep recursion is needed, increase stack size with `-Xss` (but that's a band-aid).

> **One-line distinction:** `StackOverflowError` = stack full, usually bad recursion. `OutOfMemoryError` = heap (or metaspace) full, usually too many live objects or a leak.

---

## JVM Tuning Flags

You pass these on the command line: `java -Xmx512m -XX:+UseG1GC MyApp`.

| Flag | What it does |
|---|---|
| `-Xms<size>` | **Initial** heap size (e.g., `-Xms256m`). Setting `-Xms = -Xmx` avoids resize pauses. |
| `-Xmx<size>` | **Maximum** heap size (e.g., `-Xmx2g`). The single most important flag. |
| `-Xss<size>` | **Stack size per thread** (e.g., `-Xss1m`). Bigger = deeper recursion allowed, but more memory per thread. |
| `-XX:+UseG1GC` | Use the G1 garbage collector (default since Java 9; explicit for older versions). |
| `-XX:+UseZGC` | Use the ZGC low-latency collector. |
| `-XX:MaxMetaspaceSize=<size>` | Cap Metaspace (off-heap class metadata) so a classloader leak can't eat all native memory. |
| `-XX:+HeapDumpOnOutOfMemoryError` | Automatically write a heap dump file when OOM occurs — invaluable for diagnosing leaks. |
| `-XX:HeapDumpPath=<path>` | Where to write that heap dump. |
| `-verbose:gc` / `-Xlog:gc*` | Print GC activity to the log (Java 9+ uses `-Xlog:gc*`) so you can see pauses and memory trends. |
| `-XX:MaxGCPauseMillis=<ms>` | Target maximum GC pause (a hint to G1). |

Example production startup:

```bash
java -Xms2g -Xmx2g \              # fixed 2GB heap, no resizing
     -XX:+UseG1GC \               # G1 collector
     -XX:MaxGCPauseMillis=200 \   # aim for <200ms pauses
     -XX:+HeapDumpOnOutOfMemoryError \   # dump on OOM for post-mortem
     -XX:HeapDumpPath=/var/log/app/ \
     -Xlog:gc*:file=/var/log/app/gc.log \  # log GC activity
     -jar myapp.jar
```

---

## Monitoring & Troubleshooting Tools

These ship with the JDK (in the `bin/` folder). Know what each is for.

| Tool | What it does / when to use |
|---|---|
| **`jps`** | Lists running Java processes and their PIDs. Your starting point — find the PID first. |
| **`jstat`** | Live GC and memory statistics for a running JVM (e.g., `jstat -gcutil <pid> 1000` every second). Use to watch heap/GC behavior over time. |
| **`jmap`** | Inspect or **dump the heap** (`jmap -dump:format=b,file=heap.hprof <pid>`) and view histograms (`jmap -histo <pid>`). Use to investigate leaks. |
| **`jstack`** | Take a **thread dump** — shows what every thread is doing. Use to diagnose deadlocks, hangs, and high CPU. |
| **`jconsole`** | GUI to monitor heap, threads, classes, and CPU live via JMX. Good for a quick visual check. |
| **`VisualVM`** | Richer GUI: live monitoring + heap dump analysis + CPU/memory profiling. Great all-rounder. |
| **`jcmd`** | Swiss-army-knife: send many diagnostic commands to a JVM (GC, dumps, flags) from one tool. |
| **Eclipse MAT** | Best tool for **reading a heap dump** — finds leak suspects and shows the reference chain (dominator tree) holding objects alive. |

**Typical workflow for a memory problem:**
```
jps              → find the PID
jstat -gcutil    → confirm heap is filling / GC thrashing
jmap -histo      → which class is growing?
jmap -dump       → capture a heap dump (.hprof)
Eclipse MAT      → open dump, find leak suspect & the GC Root holding it
```

---

## Common Interview Questions

### Q: What is the difference between the stack and the heap?

- **Stack**: per-thread, stores method frames, local variables, and references. LIFO, very fast, automatically freed when a method returns. Throws `StackOverflowError` when full.
- **Heap**: shared across all threads, stores all objects and arrays. Managed by the Garbage Collector. Throws `OutOfMemoryError` when full.

Local primitives and references live on the stack; the objects themselves always live on the heap.

---

### Q: What is garbage collection and how does the JVM know what to collect?

GC is the automatic process of reclaiming memory from objects that are no longer needed, so you never manually `free()`. The JVM determines what to collect using **reachability**: starting from **GC Roots**, it marks every object reachable through a chain of references as alive. Anything **not** reachable is garbage and gets collected.

---

### Q: What are GC Roots?

GC Roots are the starting "anchor" references that are always considered alive: local variables on thread stacks, static variables of loaded classes, active threads, and JNI references. An object survives GC only if it is reachable from at least one GC Root.

---

### Q: What is the difference between Minor GC and Major/Full GC?

- **Minor GC** cleans the **Young Generation** (Eden + Survivor spaces). It's frequent and fast because most young objects are already dead.
- **Major GC** cleans the **Old Generation**; a **Full GC** cleans the whole heap (and Metaspace). These are rarer but slower and cause longer stop-the-world pauses.

---

### Q: Why does the JVM use generational garbage collection?

Because of the **Weak Generational Hypothesis: most objects die young.** By separating short-lived (Young) from long-lived (Old) objects, the GC can scan the small, garbage-heavy Young Generation frequently and cheaply, while rarely touching the Old Generation. This makes the common case very fast.

---

### Q: What is Metaspace and how is it different from PermGen?

Metaspace stores class metadata (class definitions, method bytecode, static info). **PermGen** (pre-Java 8) was a fixed-size region *inside the heap* and frequently caused `OutOfMemoryError: PermGen space`. **Java 8 replaced it with Metaspace**, which lives in **native (off-heap) memory** and grows automatically by default — so OOM there is now much rarer (usually only from classloader leaks or an explicit `-XX:MaxMetaspaceSize` limit).

---

### Q: Can Java have memory leaks if it has a garbage collector?

Yes. The GC only collects **unreachable** objects. If you unintentionally keep objects reachable from a GC Root — e.g., an ever-growing static collection, an unbounded cache, unregistered listeners, or `ThreadLocal`s never cleared in a thread pool — the GC won't collect them, and memory climbs until OOM.

---

### Q: What's the difference between StackOverflowError and OutOfMemoryError?

- **`StackOverflowError`**: a thread's **stack** is exhausted, almost always due to too-deep or infinite **recursion** (each call adds a frame that never pops).
- **`OutOfMemoryError`**: the **heap** (or Metaspace) is full and GC can't free enough — usually too many live objects or a memory leak.

---

### Q: What is the JIT compiler?

The Just-In-Time compiler is part of the Execution Engine. The JVM starts by **interpreting** bytecode line by line (fast startup, slow execution). The JIT watches for **hot** code (methods/loops run many times) and compiles it to optimized **native machine code**, dramatically speeding up the parts that matter. This is why HotSpot JVM is named "HotSpot".

---

### Q: What is the Parent Delegation model in class loading?

When a class needs to be loaded, the classloader first **delegates the request up to its parent** (Application → Platform → Bootstrap) before trying to load it itself. It exists for **security** (nobody can replace core classes like `java.lang.String` with a fake) and to **avoid duplicate** core class definitions (each core class is loaded exactly once).

---

### Q: Which garbage collector is the default, and what makes it good?

**G1 (Garbage-First)** has been the default since **Java 9**. It divides the heap into many equal-sized **regions** and prioritizes collecting the regions with the most garbage first. It does most work concurrently and lets you set a **target pause time**, giving a good balance of low latency and high throughput for typical server applications.

---

### Q: What is a "stop-the-world" pause?

It's when the GC **pauses all application threads** so it can safely walk the object graph without it changing mid-collection. Short pauses are unnoticeable; long ones (e.g., a Full GC on a huge heap) freeze the app and cause latency spikes. Modern collectors (G1, ZGC, Shenandoah) aim to minimize STW time by doing more work concurrently.

---

### Q: What are the three subsystems of the JVM?

1. **ClassLoader Subsystem** — loads, links, and initializes classes.
2. **Runtime Data Areas** — memory regions (Heap, Stacks, Metaspace, PC Register, Native Method Stack).
3. **Execution Engine** — runs bytecode via the Interpreter + JIT compiler, with the Garbage Collector managing memory.

---

### Q: Are static variables stored on the heap or somewhere else?

Class-level metadata and the static fields' "slots" are associated with **Metaspace** (the class's runtime data, off-heap since Java 8). The actual **objects** that static reference variables point to still live on the **heap**. Static fields are themselves GC Roots, which is why static collections are a classic leak source.

---

## Quick Reference Cheat Sheet

```
JOURNEY OF CODE:
  .java  --javac-->  .class (bytecode)  --JVM-->  interpret + JIT  -->  native machine code
  JDK ⊃ JRE ⊃ JVM   (build) (run) (execute)

JVM ARCHITECTURE (3 parts):
  1. ClassLoader Subsystem  → Loading → Linking → Initialization
  2. Runtime Data Areas     → memory regions
  3. Execution Engine       → Interpreter + JIT + Garbage Collector
```

```
CLASS LOADING:
  Phases : Loading → Linking (Verify, Prepare, Resolve) → Initialization
  Loaders: Bootstrap → Platform(Extension) → Application
  Parent Delegation: ask parent first (security + no duplicate core classes)
```

```
MEMORY AREAS — SHARED vs PER-THREAD:
  SHARED      → Heap (objects), Metaspace (class metadata, off-heap since Java 8)
  PER-THREAD  → JVM Stack, PC Register, Native Method Stack
```

```
STACK vs HEAP:
                STACK                         HEAP
  stores   local vars, frames, refs      objects, arrays, instance vars
  scope    per-thread                    shared
  speed    very fast (LIFO)              slower (GC managed)
  thread   inherently safe               NOT safe (needs sync)
  error    StackOverflowError            OutOfMemoryError
  flag     -Xss                          -Xms / -Xmx
```

```
HEAP GENERATIONS:
  Young Gen = Eden + S0 + S1   → new objects; cleaned by Minor GC (fast, frequent)
  Old Gen   = Tenured          → survivors; cleaned by Major/Full GC (slow, rare)
  Rule: "most objects die young" (Weak Generational Hypothesis)
  Lifecycle: Eden → Survivor (age++) → promoted to Old Gen at threshold
```

```
GARBAGE COLLECTION:
  Alive = reachable from a GC Root (stack locals, static vars, threads, JNI)
  Algorithm: MARK (find alive) → SWEEP (delete dead) → COMPACT (defragment)
  Minor GC = Young | Major GC = Old | Full GC = whole heap
  Stop-the-world = pause all app threads during collection
```

```
GC ALGORITHMS (one-liners):
  Serial      → tiny heaps / single CPU
  Parallel    → max throughput, longer pauses OK (batch)
  CMS         → DEPRECATED (Java 9) / REMOVED (Java 14) — don't use
  G1          → DEFAULT since Java 9; region-based; balanced
  ZGC         → huge heaps, sub-ms pauses
  Shenandoah  → low-latency concurrent compaction, large heaps
```

```
ERRORS:
  StackOverflowError       → stack exhausted (deep/infinite recursion) → add base case / use loop
  OOM: Java heap space     → heap full / leak → raise -Xmx, fix leak
  OOM: GC overhead limit   → GC thrashing (heap effectively full)
  OOM: Metaspace           → too many classes / classloader leak
```

```
KEY TUNING FLAGS:
  -Xms / -Xmx                       initial / max heap (set equal to avoid resize)
  -Xss                              stack size per thread
  -XX:+UseG1GC                      G1 collector
  -XX:MaxMetaspaceSize              cap class-metadata memory
  -XX:+HeapDumpOnOutOfMemoryError   dump heap on OOM (for leak analysis)
  -verbose:gc / -Xlog:gc*           log GC activity
```

```
COMMON LEAK CAUSES:
  - static collections that grow forever
  - unclosed resources (streams, connections) → use try-with-resources
  - listeners/callbacks never deregistered
  - ThreadLocal never remove()'d in pooled threads
  - caches without eviction (use bounded/LRU/Caffeine)

TOOLS WORKFLOW:
  jps → jstat -gcutil → jmap -histo → jmap -dump → Eclipse MAT (find leak suspect)
  jstack → thread dump (deadlocks/hangs)   |   jconsole / VisualVM → live monitoring
```

---

*Last Updated: 2026-06-06*

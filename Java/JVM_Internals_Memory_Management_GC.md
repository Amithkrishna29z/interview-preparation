# JVM Internals, Memory Management & Garbage Collection — Study Guide

## Overview

The JVM (Java Virtual Machine) is the engine that runs your Java program. Understanding how it loads classes, where it stores objects, and how it cleans up memory is a key differentiator in a Java backend interview.

This guide covers the modern HotSpot JVM (Java 8–21). The most important historical change: **PermGen was removed in Java 8 and replaced by Metaspace** (native memory, not the heap).

> 📎 **Companion resource:** [NotebookLM notebook — JVM Internals & Memory Management](https://notebooklm.google.com/notebook/e0f80d4c-83df-4674-9058-56a63d1b7830/artifact/cbabc1fb-57bc-421b-8932-b1ad569fe7fd?utm_content=&utm_smc=nlm_web_share_google_oo_art_share_2_)

---

## Table of Contents

1. [The Journey of Java Code](#the-journey-of-java-code)
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

## The Journey of Java Code

Java's slogan is **"Write Once, Run Anywhere" (WORA)**. It compiles to **bytecode** (a `.class` file), not machine code — so the same file runs on any OS that has a JVM.

```
.java  --javac-->  .class (bytecode)  --JVM-->  interpret + JIT  -->  native code
```

1. **`.java`** — Human-readable source.
2. **`javac`** — Compiles to **bytecode** (platform-independent `.class` file).
3. **JVM** — Interprets bytecode; **JIT compiler** converts hot code to native machine code.
4. **Native** — CPU executes real instructions.

**Key terms:**

```
JDK ⊃ JRE ⊃ JVM
(build) (run) (execute)
```

- **JDK** = JRE + developer tools (`javac`, `jdb`). Needed to *build*.
- **JRE** = JVM + core libraries. Needed to *run*.
- **JVM** = engine that executes bytecode.

---

## JVM Architecture

Three big subsystems:

1. **ClassLoader Subsystem** — finds, loads, and prepares `.class` files.
2. **Runtime Data Areas** — memory regions (heap, stacks, etc.).
3. **Execution Engine** — runs bytecode: Interpreter + JIT Compiler + Garbage Collector.

**Execution Engine's three helpers:**
- **Interpreter** — Executes bytecode one instruction at a time. Fast startup, slow for repeated code.
- **JIT Compiler** — Compiles "hot" methods/loops to native code for speed. This is why the JVM is called **HotSpot**.
- **Garbage Collector** — Automatically frees memory from unreachable objects.

---

## Class Loading

Classes are loaded in **3 phases**:

```
LOADING  →  LINKING (Verify → Prepare → Resolve)  →  INITIALIZATION
```

1. **Loading** — ClassLoader reads the `.class` file; creates a `Class` object in Metaspace.
2. **Linking**:
   - **Verify** — Checks bytecode is valid and safe (security feature).
   - **Prepare** — Allocates memory for `static` fields; sets them to **default values** (0, `false`, `null`).
   - **Resolve** — Replaces symbolic names with direct memory references.
3. **Initialization** — Runs `static` blocks and assigns **real** values to static fields.

```java
public class Config {
    static int count = 10; // PREPARE: count = 0; INITIALIZATION: count = 10
    static { System.out.println("init"); } // runs during INITIALIZATION
}
```

### The ClassLoader Hierarchy

```
Bootstrap ClassLoader  (loads core JDK: java.lang.*, java.util.*)
        ↓ parent of
Platform ClassLoader   (loads JDK extension/platform modules; was "Extension" pre-Java 9)
        ↓ parent of
Application ClassLoader  (loads YOUR classes from classpath)
```

### The Parent Delegation Model

When a class needs loading, the request goes **UP** to the parent first. Each classloader asks its parent before trying itself — falling back to the child only if every parent says "not mine."

**Why it exists:**
1. **Security** — Nobody can replace core classes (e.g., a fake `java.lang.String`). Bootstrap always loads the real one.
2. **No duplicates** — `java.lang.Object` is loaded exactly once.

---

## Runtime Memory Areas

**Key distinction: which areas are shared, which are per-thread.**

| Area | Shared or Per-Thread | What it stores | Error when full |
|---|---|---|---|
| **Heap** | **Shared** | All objects & arrays (everything `new`'d), instance variables | `OutOfMemoryError: Java heap space` |
| **JVM Stack** | **Per-thread** | Stack frames: method calls, local variables, references | `StackOverflowError` |
| **Metaspace** | **Shared** | Class metadata, static vars. **Native memory** since Java 8 (replaced PermGen) | `OutOfMemoryError: Metaspace` |
| **PC Register** | **Per-thread** | Address of the current bytecode instruction | — |
| **Native Method Stack** | **Per-thread** | Used for native C/C++ methods via JNI | — |

> **Memorize:** Heap and Metaspace are **shared**. Stack, PC Register, and Native Method Stack are **per-thread**.

> **PermGen → Metaspace:** PermGen was a fixed-size region *inside* the heap (Java 7 and earlier) that frequently caused OOM. Java 8 replaced it with Metaspace in **native memory**, which grows automatically by default.

---

## Stack vs Heap

**The most-asked JVM interview question.**

| Aspect | Stack | Heap |
|---|---|---|
| **What's stored** | Local variables, method frames, references to heap objects, local primitives | Objects, arrays, instance variables |
| **Scope** | Per-thread | Shared across all threads |
| **Lifetime** | Freed automatically when method returns | Freed by Garbage Collector |
| **Speed** | Very fast | Slower (GC overhead) |
| **Thread safety** | Inherently safe (private per thread) | Not thread-safe (needs synchronization) |
| **Error when full** | `StackOverflowError` | `OutOfMemoryError` |
| **JVM flag** | `-Xss` | `-Xms` / `-Xmx` |

```java
public void run() {
    int x = 10;                      // primitive → lives on STACK
    Person p = new Person("Alice");  // 'p' reference on STACK; Person object on HEAP
    int[] nums = {1, 2, 3};          // 'nums' reference on STACK; array on HEAP
}
```

> **Interview gold:** Local primitives and references live on the **stack**; the objects they point to always live on the **heap**. When `run()` returns, the stack frame is gone — but the heap objects remain until GC collects them.

---

## Heap Structure & Object Lifecycle

The heap is divided into **generations** based on the **Weak Generational Hypothesis**: most objects die young (loop vars, temporaries). A small minority (caches, config) survive long.

```
HEAP
├── YOUNG GENERATION
│   ├── Eden      ← new objects born here
│   ├── Survivor 0 (S0)
│   └── Survivor 1 (S1)   ← Minor GC: fast, frequent
└── OLD GENERATION (Tenured)
    └── long-lived objects   ← Major/Full GC: slow, rare
```

**Object lifecycle:**

1. `new Object()` → allocated in **Eden**.
2. Eden fills → **Minor GC**: survivors copied to S0/S1; dead objects discarded.
3. Survives more Minor GCs → bounces S0 ↔ S1, age counter increments.
4. Age hits threshold (~15, `-XX:MaxTenuringThreshold`) → **promoted to Old Generation**.
5. Old Gen cleaned only by Major/Full GC.

**Why two survivor spaces?** One is always empty. During Minor GC, live objects from Eden + the occupied survivor space are all copied into the empty one, then roles flip. This automatically compacts memory (no fragmentation).

---

## Garbage Collection

### Reachability & GC Roots

An object is **alive** if reachable from a **GC Root**. If nothing can reach it, it's garbage.

**GC Roots** (always alive):
- Local variables on any thread's stack
- Static variables of loaded classes
- Active threads
- JNI references

```
GC ROOTS                  HEAP
Stack local  ──► [ A ] ──► [ B ]     ← A, B: REACHABLE (alive)
Static field ──► [ C ]               ← C: REACHABLE (alive)

                 [ D ] ──► [ E ]     ← D, E: nothing points to them → GARBAGE
```

D and E reference each other, but since no GC Root reaches them, both are collected. This is why Java's GC handles **circular references** correctly (unlike reference counting).

### Mark → Sweep → Compact

1. **Mark** — Follow all references from GC Roots; mark reachable objects alive.
2. **Sweep** — Reclaim memory of unmarked (unreachable) objects.
3. **Compact** — Slide survivors together to remove fragmentation.

### Minor GC vs Major/Full GC

| | Minor GC | Major / Full GC |
|---|---|---|
| **Where** | Young Generation | Old Generation / whole heap |
| **Frequency** | Often | Rare |
| **Speed** | Fast | Slow |
| **Pause** | Short stop-the-world | Longer stop-the-world |

### Stop-the-World (STW)

Most GC phases **pause all application threads** so GC can safely walk the object graph. Short pauses are fine; long ones (Full GC on a huge heap) cause latency spikes. Reducing STW pause time is the main goal of modern collectors (G1, ZGC, Shenandoah).

---

## GC Algorithms

| Collector | How it works | When to pick | Flag |
|---|---|---|---|
| **Serial GC** | Single-thread, full STW | Tiny heaps, single-CPU | `-XX:+UseSerialGC` |
| **Parallel GC** | Multi-thread, still full STW | Max throughput, pauses OK (batch jobs) | `-XX:+UseParallelGC` |
| **CMS** | Mostly concurrent; no compaction | **Removed in Java 14. Don't use.** | — |
| **G1 GC** | Heap split into regions; collects most-garbage regions first; concurrent | **Default since Java 9.** Balanced latency + throughput | `-XX:+UseG1GC` |
| **ZGC** | Concurrent; sub-millisecond pauses | Very large heaps, ultra-low latency | `-XX:+UseZGC` |
| **Shenandoah** | Concurrent compaction; pauses independent of heap size | Large heaps, low latency (Red Hat) | `-XX:+UseShenandoahGC` |

> Serial = tiny apps. Parallel = max throughput. **G1 = modern default.** ZGC/Shenandoah = huge heaps + can't tolerate pauses.

---

## Memory Leaks in Java

Yes — Java can have memory leaks. A leak happens when objects you no longer need are still **reachable** from a GC Root, so GC won't collect them. They accumulate until `OutOfMemoryError`.

| Cause | Why it leaks |
|---|---|
| **Static collections that grow forever** | `static Map`/`List` is always reachable; never-removed entries pile up. |
| **Unclosed resources** | Streams/connections not closed — use try-with-resources. |
| **Listeners not deregistered** | Event source keeps a reference to the listener, keeping it alive. |
| **ThreadLocal misuse** | Thread pool threads live forever; `ThreadLocal` values not `remove()`'d stay attached. |
| **Caches without eviction** | Cache that only grows — use bounded caches (Caffeine, `WeakHashMap`). |

**Diagnosis workflow:** heap usage trends up over time → `jps` (find PID) → `jstat -gcutil` (confirm heap filling) → `jmap -histo` (which class is growing?) → `jmap -dump` (capture `.hprof`) → Eclipse MAT (find GC Root holding the objects).

---

## Errors: OutOfMemoryError & StackOverflowError

| Error | Area | Cause | Fix |
|---|---|---|---|
| `OutOfMemoryError: Java heap space` | Heap | Too many live objects or a leak | Increase `-Xmx`; fix the leak |
| `OutOfMemoryError: GC overhead limit exceeded` | Heap | JVM spending >98% time in GC | Same as above |
| `OutOfMemoryError: Metaspace` | Metaspace | Too many classes / classloader leak | Increase `-XX:MaxMetaspaceSize`; fix leaks |
| `OutOfMemoryError: Unable to create new native thread` | Native | OS hit thread limit | Fewer threads, reduce `-Xss` |
| `StackOverflowError` | Stack | Too-deep or infinite recursion | Add base case; convert to iteration |

```java
public int countDown(int n) {
    return countDown(n - 1); // no base case → StackOverflowError
}
```

> `StackOverflowError` = stack full (bad recursion). `OutOfMemoryError` = heap/metaspace full (too many live objects or a leak).

---

## JVM Tuning Flags

```bash
java -Xms2g -Xmx2g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/app/ \
     -Xlog:gc*:file=/var/log/app/gc.log \
     -jar myapp.jar
```

| Flag | What it does |
|---|---|
| `-Xms<size>` | Initial heap size. Set equal to `-Xmx` to avoid resize pauses. |
| `-Xmx<size>` | Maximum heap size. The most important flag. |
| `-Xss<size>` | Stack size per thread. |
| `-XX:+UseG1GC` | Use G1 collector (default Java 9+). |
| `-XX:+UseZGC` | Use ZGC low-latency collector. |
| `-XX:MaxMetaspaceSize=<size>` | Cap Metaspace to prevent classloader leaks from eating native memory. |
| `-XX:+HeapDumpOnOutOfMemoryError` | Auto-write heap dump on OOM — essential for leak diagnosis. |
| `-Xlog:gc*` | Print GC activity to log (Java 9+). |
| `-XX:MaxGCPauseMillis=<ms>` | Target max pause time (hint to G1). |

---

## Monitoring & Troubleshooting Tools

| Tool | What it does |
|---|---|
| **`jps`** | Lists running Java processes and PIDs. Start here. |
| **`jstat`** | Live GC/memory stats (`jstat -gcutil <pid> 1000`). Watch heap/GC over time. |
| **`jmap`** | Dump heap (`jmap -dump:format=b,file=heap.hprof <pid>`) or histogram (`jmap -histo <pid>`). |
| **`jstack`** | Thread dump — diagnose deadlocks and hangs. |
| **`jconsole`** | GUI: live heap, threads, classes, CPU via JMX. |
| **`VisualVM`** | Richer GUI: live monitoring + heap dump analysis + profiling. |
| **`jcmd`** | Swiss-army-knife: send GC/dump/flag commands to a running JVM. |
| **Eclipse MAT** | Best tool for reading heap dumps — finds leak suspects and dominator tree. |

---

## Common Interview Questions

### Q: What is the difference between the stack and the heap?

Stack is per-thread; stores method frames, local variables, and references; freed automatically when a method returns; throws `StackOverflowError` when full. Heap is shared across threads; stores all objects and arrays; managed by GC; throws `OutOfMemoryError` when full. Local primitives and references live on the stack; the objects they point to live on the heap.

---

### Q: What is garbage collection and how does the JVM know what to collect?

GC automatically reclaims memory from objects that are no longer needed. The JVM uses **reachability**: starting from GC Roots, it marks every object reachable through a reference chain as alive. Anything not reachable is collected.

---

### Q: What are GC Roots?

The "anchor" references always considered alive: local variables on thread stacks, static variables, active threads, and JNI references. An object survives GC only if reachable from at least one GC Root.

---

### Q: What is the difference between Minor GC and Major/Full GC?

Minor GC cleans the Young Generation (Eden + Survivors) — frequent and fast because most young objects are already dead. Major GC cleans the Old Generation; Full GC cleans the whole heap. Both are rarer and slower, causing longer stop-the-world pauses.

---

### Q: Why does the JVM use generational garbage collection?

Because of the **Weak Generational Hypothesis: most objects die young.** Separating short-lived (Young) from long-lived (Old) objects lets GC scan the small, garbage-heavy Young Generation cheaply and frequently, while rarely touching the Old Generation.

---

### Q: What is Metaspace and how is it different from PermGen?

Metaspace stores class metadata (definitions, method bytecode, static info). PermGen (pre-Java 8) was a fixed-size region *inside* the heap and often caused `OutOfMemoryError: PermGen space`. Java 8 replaced it with Metaspace in **native (off-heap) memory** that grows automatically — so OOM there is now rare.

---

### Q: Can Java have memory leaks if it has a garbage collector?

Yes. GC only collects **unreachable** objects. If you accidentally keep objects reachable — via a growing static collection, unbounded cache, unregistered listeners, or uncleaned `ThreadLocal`s in a thread pool — GC won't collect them, and memory grows until OOM.

---

### Q: What's the difference between StackOverflowError and OutOfMemoryError?

`StackOverflowError`: the stack is exhausted — almost always infinite or too-deep recursion (each call adds a frame that never pops). `OutOfMemoryError`: the heap or Metaspace is full and GC can't free enough — too many live objects or a memory leak.

---

### Q: What is the JIT compiler?

The Just-In-Time compiler watches for **hot** code (methods/loops run many times) and compiles it to optimized **native machine code**, dramatically speeding up the parts that matter most. The JVM starts by interpreting bytecode for fast startup, then JIT-compiles hot paths for speed. This is why the JVM is called HotSpot.

---

### Q: What is the Parent Delegation model in class loading?

When a class needs loading, the classloader **delegates up to its parent** (Application → Platform → Bootstrap) before trying itself. This ensures security (no fake `java.lang.String` can replace the real one) and avoids duplicate core class definitions.

---

### Q: Which garbage collector is the default, and what makes it good?

**G1 (Garbage-First)** has been the default since **Java 9**. It divides the heap into equal-sized regions and prioritizes collecting regions with the most garbage first. It does most work concurrently with the app and supports a **target pause time** goal, giving a good balance of low latency and throughput for typical server applications.

---

### Q: Are static variables stored on the heap or Metaspace?

Static field slots live with class metadata in **Metaspace** (off-heap since Java 8), but the actual **objects** that static reference variables point to live on the **heap**. Static fields are themselves GC Roots — which is why static collections are a classic leak source.

---

## Quick Reference Cheat Sheet

```
JOURNEY OF CODE:
  .java --javac--> .class (bytecode) --JVM--> interpret + JIT --> native code
  JDK ⊃ JRE ⊃ JVM   |   JVM = ClassLoader + Runtime Data Areas + Execution Engine

MEMORY AREAS — SHARED vs PER-THREAD:
  SHARED      → Heap (objects), Metaspace (class metadata, off-heap since Java 8)
  PER-THREAD  → JVM Stack, PC Register, Native Method Stack

STACK vs HEAP:
  Stack → per-thread; local vars, frames, refs; LIFO/fast; StackOverflowError; -Xss
  Heap  → shared; objects/arrays; GC-managed; OutOfMemoryError; -Xms/-Xmx

HEAP GENERATIONS:
  Young (Eden+S0+S1) → Minor GC (fast, frequent) | Old (Tenured) → Major/Full GC (slow, rare)
  "Most objects die young." Eden → Survivor (age++) → promoted to Old Gen at threshold

GARBAGE COLLECTION:
  Alive = reachable from a GC Root (stack locals, static vars, threads, JNI)
  MARK (find alive) → SWEEP (delete dead) → COMPACT (defragment)
  GC ALGOS: Serial=tiny | Parallel=throughput | G1=default(Java 9) | ZGC/Shenandoah=huge heaps

ERRORS:
  StackOverflowError    → deep/infinite recursion → add base case / use loop
  OOM: Java heap space  → heap full / leak → raise -Xmx, fix leak
  OOM: Metaspace        → too many classes / classloader leak

KEY FLAGS: -Xms/-Xmx (heap), -Xss (stack), -XX:+UseG1GC,
           -XX:+HeapDumpOnOutOfMemoryError, -Xlog:gc*

LEAK CAUSES: static collections, unclosed resources, underegistered listeners,
             ThreadLocal not remove()'d in pools, unbounded caches
TOOLS: jps → jstat -gcutil → jmap -histo → jmap -dump → Eclipse MAT
```

---

*Last Updated: 2026-06-18*

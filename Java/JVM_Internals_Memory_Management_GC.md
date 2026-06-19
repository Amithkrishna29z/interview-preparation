# JVM Internals, Memory Management & Garbage Collection — Study Guide

## Overview

The JVM (Java Virtual Machine) is the engine that runs your Java program. Understanding how it loads classes, where it stores objects, and how it cleans up memory is a key differentiator in a Java backend interview.

This guide covers the modern HotSpot JVM (Java 8–21). The most important historical change: **PermGen was removed in Java 8 and replaced by Metaspace** (native memory, not the heap).

> 📎 **Companion resource:** [NotebookLM notebook — JVM Internals & Memory Management](https://notebooklm.google.com/notebook/e0f80d4c-83df-4674-9058-56a63d1b7830/artifact/cbabc1fb-57bc-421b-8932-b1ad569fe7fd?utm_content=&utm_smc=nlm_web_share_google_oo_art_share_2_)

---

## Table of Contents

1. [The Journey of Java Code](#the-journey-of-java-code)
2. [JVM Architecture](#jvm-architecture)
3. [Runtime Memory Areas](#runtime-memory-areas)
4. [Stack vs Heap](#stack-vs-heap)
5. [Garbage Collection](#garbage-collection)
6. [Memory Leaks in Java](#memory-leaks-in-java)
7. [Errors: OutOfMemoryError & StackOverflowError](#errors-outofmemoryerror--stackoverflowerror)
8. [Common Interview Questions](#common-interview-questions)
9. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

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

## Memory Leaks in Java

Yes — Java can have memory leaks. A leak happens when objects you no longer need are still **reachable** from a GC Root, so GC won't collect them. They accumulate until `OutOfMemoryError`.

| Cause | Why it leaks |
|---|---|
| **Static collections that grow forever** | `static Map`/`List` is always reachable; never-removed entries pile up. |
| **Unclosed resources** | Streams/connections not closed — use try-with-resources. |
| **Listeners not deregistered** | Event source keeps a reference to the listener, keeping it alive. |
| **ThreadLocal misuse** | Thread pool threads live forever; `ThreadLocal` values not `remove()`'d stay attached. |
| **Caches without eviction** | Cache that only grows — use bounded caches (Caffeine, `WeakHashMap`). |

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

GARBAGE COLLECTION:
  Alive = reachable from a GC Root (stack locals, static vars, threads, JNI)
  MARK (find alive) → SWEEP (delete dead) → COMPACT (defragment)

ERRORS:
  StackOverflowError    → deep/infinite recursion → add base case / use loop
  OOM: Java heap space  → heap full / leak → raise -Xmx, fix leak
  OOM: Metaspace        → too many classes / classloader leak

LEAK CAUSES: static collections, unclosed resources, underegistered listeners,
             ThreadLocal not remove()'d in pools, unbounded caches
```

---

*Last Updated: 2026-06-18*

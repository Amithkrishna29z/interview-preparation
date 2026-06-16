# .NET CLR, Memory & Garbage Collection — A Study Guide for Java Developers

## Overview

If you already understand the **JVM**, its **heap**, and **garbage collection**, you are roughly 80% of the way to understanding the **.NET runtime**. The concepts map almost one-to-one — only the names and a few design decisions differ.

This guide explains the **CLR (Common Language Runtime)**, how C# turns into running code, how memory is laid out (stack vs heap, value vs reference types), how the **generational garbage collector** works, and how to clean up resources deterministically (`IDisposable`/`using`) versus non-deterministically (finalizers). Throughout, every concept is mapped back to its Java equivalent so you can lean on what you already know.

Target runtime: **modern .NET (8+)** — the unified, cross-platform, open-source runtime (the successor to both .NET Framework and .NET Core).

---

## Table of Contents

- [JVM → CLR Mapping](#jvm--clr-mapping)
- [The CLR: What It Is](#the-clr-what-it-is)
- [IL/MSIL, JIT, Tiered Compilation, ReadyToRun, Native AOT](#ilmsil-jit-tiered-compilation-readytorun-native-aot)
- [Assemblies, Metadata, the BCL](#assemblies-metadata-the-bcl)
- [Managed vs Unmanaged Code & Memory](#managed-vs-unmanaged-code--memory)
- [Stack vs Heap, Value Types vs Reference Types](#stack-vs-heap-value-types-vs-reference-types)
- [Memory Layout: Object Header & Type Pointer](#memory-layout-object-header--type-pointer)
- [Boxing and Unboxing](#boxing-and-unboxing)
- [The Garbage Collector: Generations & the LOH](#the-garbage-collector-generations--the-loh)
- [How GC Works: Mark, Sweep, Compact, Roots](#how-gc-works-mark-sweep-compact-roots)
- [GC Modes: Workstation vs Server, Background GC](#gc-modes-workstation-vs-server-background-gc)
- [IDisposable, Dispose, using — Deterministic Cleanup](#idisposable-dispose-using--deterministic-cleanup)
- [Finalizers: Why They Are Costly](#finalizers-why-they-are-costly)
- [WeakReference and Common Memory Leaks](#weakreference-and-common-memory-leaks)
- [Span&lt;T&gt;, Memory&lt;T&gt;, stackalloc — Low-Allocation Code](#spant-memoryt-stackalloc--low-allocation-code)
- [Diagnostics Tools](#diagnostics-tools)
- [GC.Collect — and Why Not to Call It](#gccollect--and-why-not-to-call-it)
- [Common Interview Questions](#common-interview-questions)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## JVM → CLR Mapping

This is the single most useful table in the guide. Bookmark it.

| Java / JVM concept | .NET / CLR equivalent | Notes |
|---|---|---|
| JVM (Java Virtual Machine) | **CLR** (Common Language Runtime) | The managed execution engine |
| Bytecode (`.class`) | **IL / MSIL / CIL** (Intermediate Language) | The portable instruction set |
| `javac` (compiler) | **Roslyn** (C# compiler, `csc`) | Source → IL |
| HotSpot JIT (C1/C2) | **RyuJIT** | IL → native machine code at runtime |
| Tiered compilation (C1→C2) | **Tiered Compilation** (Tier 0 → Tier 1) | Same idea: fast first, optimized later |
| AOT (GraalVM native-image) | **Native AOT** | Compile to a standalone native binary |
| JAR / class loader | **Assembly** (`.dll`/`.exe`) + assembly loading | Unit of deployment + metadata |
| Classpath / module path | Assembly references / `AssemblyLoadContext` | How types are found and loaded |
| Java Standard Library (JDK) | **BCL** (Base Class Library) | `System.*` namespaces |
| Primitives (`int`, `double`) | **Value types** (`struct`, `int`, `double`) | C# value types are richer (custom structs) |
| Objects / reference types | **Reference types** (`class`) | Live on the heap |
| Autoboxing (`int`→`Integer`) | **Boxing** (`int`→`object`) | Same cost: heap allocation |
| Young / Old generation | **Gen 0, Gen 1, Gen 2** | .NET has 3 generations, not 2 |
| Eden + Survivor spaces | Gen 0 + Gen 1 (promotion path) | Conceptually similar nursery |
| Humongous objects (G1) | **Large Object Heap (LOH)** | Objects ≥ 85,000 bytes |
| Mark-and-sweep + compaction | Mark-and-sweep + compaction | Same core algorithm |
| Stop-the-world pause | Stop-the-world (blocking) GC | Background GC reduces these |
| G1 / ZGC / Shenandoah | **Background (concurrent) GC**, Server GC | Different knobs, similar goals |
| `-Xmx` / `-Xms` (heap sizing) | **`runtimeconfig.json`** / env vars (`DOTNET_GCHeapHardLimit`) | No fixed `-Xmx`; GC self-tunes |
| `-XX:+UseG1GC` flags | `ServerGarbageCollection`, `ConcurrentGarbageCollection` settings | Configured in project / runtimeconfig |
| `jstat`, VisualVM, JFR | **`dotnet-counters`**, **`dotnet-trace`**, **`dotnet-dump`** | Live metrics, tracing, dumps |
| `System.gc()` | **`GC.Collect()`** | Both: a hint you should almost never use |
| `finalize()` (deprecated) | **Finalizer** `~ClassName()` | Non-deterministic, costly, avoid |
| `try-with-resources` / `AutoCloseable` | **`using`** / `IDisposable` | Deterministic cleanup |
| `WeakReference<T>` | **`WeakReference`** / `WeakReference<T>` | Same concept |
| `Runtime.totalMemory()` | **`GC.GetTotalMemory()`** | Bytes the managed heap thinks it uses |
| JNI (native interop) | **P/Invoke** / unmanaged interop | Calling native code |

---

## The CLR: What It Is

The **CLR** is .NET's virtual machine. Like the JVM, it provides:

- A **JIT compiler** (IL → native code)
- A **garbage collector** (automatic memory management)
- A **type system** (the CTS — Common Type System)
- **Exception handling**, **security**, **threading**, and **interop** services

**Think of it like...** the JVM with a different badge. You write C# (like Java), it compiles to IL (like bytecode), and the CLR runs it on any platform with a runtime installed (like "write once, run anywhere").

A key difference: the JVM was designed around one language (Java) and others (Kotlin, Scala) joined later. The CLR was **designed from day one to be multi-language** — C#, F#, and VB.NET all compile to the same IL and interoperate seamlessly.

---

## IL/MSIL, JIT, Tiered Compilation, ReadyToRun, Native AOT

### IL / MSIL

When you compile C#, you do **not** get machine code. You get **IL** (Intermediate Language, also called MSIL or CIL) — a stack-based, CPU-independent instruction set stored inside the assembly.

```csharp
// This C#...
int Add(int a, int b) => a + b;

// ...compiles to IL roughly like this (analogous to JVM bytecode):
// ldarg.0      // load first argument
// ldarg.1      // load second argument
// add          // add them
// ret          // return
```

**Think of it like...** Java bytecode. `ildasm`/ILSpy is the .NET equivalent of `javap -c`.

### JIT compilation (RyuJIT)

At runtime, **RyuJIT** compiles each method's IL into native machine code **the first time it is called**, then caches it. This is exactly how HotSpot works.

### Tiered Compilation

Modern .NET uses **Tiered Compilation**, mirroring HotSpot's C1/C2 tiers:

- **Tier 0**: compile fast, minimal optimization → quick startup.
- **Tier 1**: for "hot" methods that run often, recompile with full optimization in the background.

```
First calls  → Tier 0 (quick, unoptimized)
Hot methods  → Tier 1 (re-JIT, fully optimized)  // like HotSpot promoting to C2
```

### ReadyToRun (R2R)

**ReadyToRun** is **ahead-of-time precompilation** baked into the assembly. The DLL ships with native code already generated, so the JIT does less work at startup. (Roughly analogous to a JVM AppCDS / tiered warm-up shortcut.) The runtime can still re-JIT to Tier 1 later.

### Native AOT

**Native AOT** compiles your whole app to a **single self-contained native executable** — no JIT, no IL at runtime, no separate runtime install needed. Fast startup, low memory, small footprint.

**Think of it like...** GraalVM `native-image` for Java. The tradeoff is the same: you lose runtime reflection/dynamic-code flexibility in exchange for speed and size. Great for CLI tools, microservices, and serverless.

```
JIT (default)   : IL shipped, compiled on demand. Flexible, slower startup.
ReadyToRun      : IL + precompiled native. Faster startup, larger DLL.
Native AOT      : pure native binary, no JIT. Fastest startup, least flexible.
```

---

## Assemblies, Metadata, the BCL

An **assembly** is the unit of deployment and versioning in .NET — a `.dll` (library) or `.exe` (application).

An assembly contains:
- **IL code** (the compiled methods)
- **Metadata** — a complete, self-describing manifest of every type, method, field, and attribute (this is why .NET reflection is so rich and you rarely need extra config files)
- **The manifest** — name, version, culture, and referenced assemblies

**Think of it like...** a JAR file, but with built-in, structured metadata instead of relying on a `MANIFEST.MF` plus reflection over `.class` files. Assembly loading is the CLR's class-loading mechanism; `AssemblyLoadContext` is roughly the analog of a Java `ClassLoader` (used for plugins, isolation, hot-reload scenarios).

The **BCL (Base Class Library)** is the standard library — `System`, `System.Collections.Generic`, `System.IO`, `System.Text`, `System.Linq`, etc. It is the JDK of .NET.

---

## Managed vs Unmanaged Code & Memory

- **Managed code**: runs under the CLR. Memory is allocated on the **managed heap** and reclaimed automatically by the GC. This is 99% of the C# you will write. (Equivalent to ordinary Java objects on the JVM heap.)
- **Unmanaged code/memory**: native resources the GC does **not** know about — OS file handles, sockets, database connections, native memory (`Marshal.AllocHGlobal`), GDI handles, etc.

**Think of it like...** managed memory is the JVM heap (auto-cleaned). Unmanaged resources are like the native file descriptor behind a Java `FileInputStream` — the GC frees the wrapper object, but **you** are responsible for releasing the underlying OS handle promptly. That responsibility is exactly what `IDisposable` exists for.

---

## Stack vs Heap, Value Types vs Reference Types

This is one of the most asked junior .NET questions. The crucial idea: in C#, **you control whether a type is a value type or a reference type**, which is richer than Java's "primitives are special, everything else is an object."

- **Value types** (`struct`, and all the built-ins: `int`, `double`, `bool`, `char`, `DateTime`, `Guid`, enums, tuples): hold their data **directly**.
- **Reference types** (`class`, `interface`, `string`, arrays, delegates): the variable holds a **reference** to data on the heap.

```csharp
struct Point { public int X, Y; }        // value type
class Person { public string Name; }      // reference type

void Demo()
{
    int n = 5;                 // value: the 5 lives directly in n (on the stack)
    Point p = new Point();     // value: p's data is inline on the stack
    Person person = new Person();
    // 'person' is a reference (on the stack) pointing to a Person object (on the heap)
}
```

```java
// Java equivalent
int n = 5;                     // primitive, on the stack
// Java has no 'struct' — there is no Point-on-the-stack option (until Project Valhalla)
Person person = new Person();  // reference on stack, object on heap
```

### The important nuance

People say "value types live on the stack, reference types on the heap" — but that is a **simplification**. The accurate rule is about **where the variable lives**:

- A value type stored in a **local variable** → on the **stack**.
- A value type that is a **field of a class** → lives **inline inside that class's heap object**.
- A value type inside an array → lives **inline in the array on the heap**.
- A value type that gets **boxed** → copied to the heap.

```csharp
class Container { public Point P; }  // Point lives INSIDE the Container object on the heap

Point[] pts = new Point[1000];       // 1000 Points stored inline in one heap array (cache-friendly!)
```

**Think of it like...** value types are passed and copied **by value** (like Java primitives), reference types are passed **by reference-value** (like Java objects — you copy the pointer, not the object). The big extra power vs Java: a `struct[]` is a flat block of data with no per-element object headers, which is why structs matter for performance.

---

## Memory Layout: Object Header & Type Pointer

Every **reference-type** object on the managed heap has a small header (on 64-bit):

```
+------------------------+
|  Sync Block Index      |  8 bytes  -> used for locking, hash code, etc.
+------------------------+
|  Type Pointer (MT*)    |  8 bytes  -> points to the MethodTable (the type's metadata)
+------------------------+
|  Instance fields...    |  the actual data
+------------------------+
```

- The **Type Pointer** points to the **MethodTable** — the runtime description of the type (its methods, base type, etc.). This is how virtual dispatch and `GetType()` work.
- The **Sync Block Index** backs `lock`, `GetHashCode()`, and so on.

**Think of it like...** the JVM's object header (mark word + klass pointer). The type pointer ≈ the klass pointer; the sync block index ≈ the mark word used for monitors/identity hash. Because of this overhead (~16 bytes minimum per object), allocating millions of tiny objects is costly — another reason structs and `Span<T>` exist.

---

## Boxing and Unboxing

**Boxing** wraps a value type in a heap object so it can be treated as `object` (or an interface). **Unboxing** extracts it back. Both have a cost.

```csharp
int x = 42;
object boxed = x;       // BOXING: allocates a heap object, copies 42 into it
int y = (int)boxed;     // UNBOXING: copies the value back out (and type-checks)
```

```java
// Java's autoboxing is the same idea:
int x = 42;
Object boxed = x;       // autobox int -> Integer on the heap
int y = (int) boxed;
```

Why it matters: boxing causes **hidden heap allocations** and GC pressure. The classic culprits are non-generic collections (avoid `ArrayList`, use `List<int>`) and `string.Format`/`object`-based APIs in hot loops. Generics in C# are **reified** (real at runtime), so `List<int>` stores `int`s with **no boxing** — unlike Java's erased generics, where `List<Integer>` always boxes.

---

## The Garbage Collector: Generations & the LOH

The .NET GC is a **generational, mark-and-compact, tracing collector** — the same family as the JVM's. The core insight is identical: **most objects die young**, so collect the young region often and cheaply.

.NET has **three generations** plus a special heap:

| Region | Holds | Java analog |
|---|---|---|
| **Gen 0** | Brand-new, short-lived objects | Eden / Young |
| **Gen 1** | Survived one GC — a buffer between young and old | Survivor space |
| **Gen 2** | Long-lived objects (survived multiple GCs) | Old / Tenured |
| **LOH** (Large Object Heap) | Objects **≥ 85,000 bytes** | Humongous objects (G1) |

How promotion works:
- New objects go to **Gen 0**.
- A **Gen 0 collection** runs frequently and is very fast. Survivors are promoted to **Gen 1**.
- Gen 1 survivors are promoted to **Gen 2**.
- **Gen 2 collections** are "full" collections — they scan everything and are the most expensive.

**Think of it like...** Eden → Survivor → Old. Gen 0 GCs are like minor/young GCs (cheap, frequent); Gen 2 GCs are like full/major GCs (expensive, rare). The LOH is collected only during Gen 2 GCs.

### The Large Object Heap (LOH)

Objects of **85,000 bytes or more** (e.g., big arrays, large buffers) go straight to the **LOH**.

- The LOH is collected as part of **Gen 2** only.
- Historically it was **not compacted** by default (to avoid copying huge blocks), which can cause **fragmentation**. You can force compaction occasionally via `GCSettings.LargeObjectHeapCompactionMode`, but it is expensive.

**Think of it like...** G1's humongous regions — large objects bypass the nursery and are handled specially because copying them is too costly.

---

## How GC Works: Mark, Sweep, Compact, Roots

The GC runs a **tracing** algorithm:

1. **Find the roots** — starting points that are definitely "alive":
   - Local variables and method arguments on thread stacks
   - **Static** fields
   - CPU registers
   - GC handles (e.g., pinned objects, `GCHandle`)
2. **Mark** — walk the object graph from the roots, marking every reachable object.
3. **Sweep** — anything **not** marked is garbage and is reclaimed.
4. **Compact** — move the survivors together to remove gaps, leaving one contiguous block of free space. This makes the next allocation a simple **pointer bump** (extremely fast).

```
Before:  [A][garbage][B][garbage][garbage][C]
Mark:     A, B, C reachable from roots
Compact: [A][B][C][............ free .............]
                      ^ allocation pointer
```

**Think of it like...** the JVM's mark-compact collectors. Because the managed heap is compacted, .NET allocation is essentially incrementing a pointer — just like the JVM's bump-the-pointer allocation in Eden. (The LOH is the exception: not compacted by default.)

A subtle but important consequence: because objects **move** during compaction, you must **pin** an object (`fixed` / `GCHandle.Alloc(..., Pinned)`) before passing its address to unmanaged code — otherwise the GC could relocate it under your feet.

---

## GC Modes: Workstation vs Server, Background GC

.NET ships **two GC flavors** plus a concurrency mode. You choose them in `runtimeconfig.json` or the project file — there is no zoo of `-XX` flags like the JVM.

### Workstation GC (default for desktop/client apps)
- Optimized for **low latency** and responsiveness.
- Fewer heaps, lower memory overhead.
- Default for client apps.

### Server GC (default for ASP.NET Core / server apps)
- Creates **one managed heap and one GC thread per CPU core**, so collections run in parallel.
- Optimized for **throughput** on multi-core machines under heavy allocation.
- Higher memory usage, but scales much better for web servers.

### Background (Concurrent) GC
- For **Gen 2** collections, the GC does most of its work on a **background thread** while your app keeps running, minimizing stop-the-world pauses. Enabled by default.

**Think of it like...** Workstation vs Server is roughly "pause-optimized vs throughput-optimized." Background GC plays the same role as the JVM's concurrent collectors (**G1**, **ZGC**, **Shenandoah**) — doing collection work concurrently to shrink pause times. The big philosophical difference: the JVM exposes many pluggable collectors and tuning flags; .NET gives you a small set of well-tuned modes and self-manages the rest.

```jsonc
// runtimeconfig.json (this is your "-Xmx / -XX flags" equivalent)
{
  "runtimeOptions": {
    "configProperties": {
      "System.GC.Server": true,            // Server GC
      "System.GC.Concurrent": true,        // Background GC
      "System.GC.HeapHardLimit": 0         // optional hard cap (like -Xmx)
    }
  }
}
```

---

## IDisposable, Dispose, using — Deterministic Cleanup

The GC handles **managed memory** automatically, but it does **not** promptly release **unmanaged resources** (file handles, sockets, DB connections). For those you need **deterministic** cleanup — release exactly when you are done, not "eventually."

That contract is **`IDisposable`** with its single method **`Dispose()`**.

```csharp
public class FileLogger : IDisposable
{
    private readonly StreamWriter _writer;          // wraps an OS file handle (unmanaged)

    public FileLogger(string path) => _writer = new StreamWriter(path);

    public void Write(string msg) => _writer.WriteLine(msg);

    public void Dispose() => _writer.Dispose();     // release the handle NOW
}
```

The **`using`** statement guarantees `Dispose()` is called when the scope exits — even on exceptions:

```csharp
// Classic 'using' block — scoped
using (var logger = new FileLogger("log.txt"))
{
    logger.Write("hello");
}   // logger.Dispose() called here, guaranteed (like a finally block)

// Modern 'using declaration' (C# 8+) — disposes at end of enclosing scope
using var logger2 = new FileLogger("log2.txt");
logger2.Write("hi");
// logger2.Dispose() called automatically at end of the method
```

**Think of it like...** Java's **try-with-resources** and the **`AutoCloseable`** interface:

```java
// Exact Java analog
try (var logger = new FileLogger("log.txt")) {  // AutoCloseable
    logger.write("hello");
}                                                // close() called, guaranteed
```

`IDisposable.Dispose()` ≈ `AutoCloseable.close()`; the `using` statement ≈ `try-with-resources`. For async cleanup, .NET also has **`IAsyncDisposable`** with `await using` — handy for things like async DB connections.

---

## Finalizers: Why They Are Costly

A **finalizer** (`~ClassName()`) is a last-resort safety net the GC calls before reclaiming an object **if** you forgot to `Dispose()` an unmanaged resource. It is the C# analog of Java's deprecated `finalize()`.

```csharp
public class NativeBuffer : IDisposable
{
    private IntPtr _handle = Marshal.AllocHGlobal(1024);  // unmanaged memory
    private bool _disposed;

    // Finalizer: runs ONLY if Dispose() was never called. Safety net, not the plan.
    ~NativeBuffer() => Dispose(false);

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);   // we cleaned up; tell the GC to skip the finalizer
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing) { /* dispose managed IDisposable fields here */ }
        if (_handle != IntPtr.Zero)   // free unmanaged memory either way
        {
            Marshal.FreeHGlobal(_handle);
            _handle = IntPtr.Zero;
        }
        _disposed = true;
    }
}
```

### Why finalizers are expensive

1. An object **with a finalizer cannot be collected in one pass**. The GC first puts it on a **finalization queue**, a dedicated **finalizer thread** runs the finalizer, and only the **next** GC actually frees the object. So it survives **at least one extra GC** and gets **promoted** to an older generation — exactly the objects you don't want lingering.
2. Finalizers run on a single thread, **non-deterministically**, in **no guaranteed order**, possibly **never** (e.g., at abrupt process exit).

That is why `GC.SuppressFinalize(this)` is called inside `Dispose()`: if you cleaned up properly, you tell the GC **not** to bother running the finalizer, restoring the cheap one-pass collection.

**Think of it like...** Java's `finalize()` — deprecated for the very same reasons (non-deterministic, slow, unreliable). The modern guidance is identical in both worlds: **prefer `Dispose`/`close` (deterministic); use a finalizer only as a backstop for raw unmanaged handles.** In fact, modern C# usually wraps native handles in **`SafeHandle`**, which provides the finalizer for you so you rarely write `~ClassName()` yourself.

---

## WeakReference and Common Memory Leaks

A **`WeakReference`** lets you reference an object **without keeping it alive** — if nothing else references it, the GC may collect it.

```csharp
var weak = new WeakReference<Person>(person);

if (weak.TryGetTarget(out var p))   // got it — still alive
    Console.WriteLine(p.Name);
else                                // collected already
    Console.WriteLine("gone");
```

**Think of it like...** Java's `WeakReference<T>`. Useful for caches and observer patterns where you don't want the cache to prevent collection.

### Yes, .NET *can* leak memory

GC frees **unreachable** objects — but if something keeps a reference, the object stays alive forever. Common culprits (same as the JVM):

1. **Event handlers (the #1 .NET leak).** Subscribing to an event creates a strong reference **from the publisher to the subscriber**. If the publisher outlives the subscriber and you never unsubscribe, the subscriber can't be collected.
   ```csharp
   publisher.SomeEvent += handler;    // publisher now holds a ref to 'this'
   // ...
   publisher.SomeEvent -= handler;    // MUST unsubscribe, or you leak
   ```
2. **Static references.** A `static` field roots an object for the lifetime of the process (it's a GC root). Static caches that only ever grow are a classic leak.
3. **Captured closures.** A lambda captures the variables it uses, keeping them (and their containing object) alive. A long-lived delegate capturing a heavy object pins that object.
4. **Long-lived collections** you add to but never remove from (caches, lists, dictionaries).
5. **Undisposed `IDisposable`s** leak unmanaged resources (handles), even if the managed memory is eventually collected.

**Think of it like...** the JVM equivalents: listener leaks, `static` map caches, and inner classes capturing `this`. The mental model — "reachable = retained" — is identical.

---

## Span&lt;T&gt;, Memory&lt;T&gt;, stackalloc — Low-Allocation Code

For high-performance, low-allocation code, modern .NET gives you tools to work with memory **without allocating new objects on the heap**.

- **`Span<T>`**: a type-safe, bounds-checked **window over existing memory** (an array, a slice, stack memory, or unmanaged memory). Slicing a span allocates **nothing**. It is a `ref struct`, so it lives on the stack and can't escape to the heap.
- **`Memory<T>`**: like `Span<T>` but **can** be stored on the heap and used across `async`/`await` (spans can't, because they're stack-only).
- **`stackalloc`**: allocate a small buffer **on the stack**, bypassing the GC entirely.

```csharp
// Slice without copying or allocating:
int[] data = { 1, 2, 3, 4, 5 };
Span<int> middle = data.AsSpan(1, 3);   // view over {2,3,4} — zero allocation
middle[0] = 99;                          // writes through to data[1]

// Stack buffer — no heap, no GC pressure:
Span<byte> buffer = stackalloc byte[256]; // 256 bytes on the stack
```

**Think of it like...** `Span<T>` is conceptually similar to a **`ByteBuffer` slice** or `Arrays.asList`-style view, but zero-cost and stack-bound. There is no direct Java equivalent to `stackalloc`; the closest modern analog is the JVM's work on value types/off-heap buffers. You won't write much of this as a junior, but knowing it exists (and *why* — avoiding GC pressure in hot paths) is a great interview signal.

---

## Diagnostics Tools

These are the cross-platform CLI tools you install with `dotnet tool install -g`. They are the .NET answer to `jstat`, VisualVM, and JFR.

| Tool | What it does | Java analog |
|---|---|---|
| **`dotnet-counters`** | Live, low-overhead metrics: GC heap size, Gen 0/1/2 collection counts, alloc rate, CPU, working set | `jstat`, VisualVM monitor |
| **`dotnet-trace`** | Capture detailed event traces (GC events, JIT, exceptions) for offline analysis | JFR (Java Flight Recorder) |
| **`dotnet-dump`** | Capture and analyze a **process memory dump** (find what's on the heap, who roots a leak) | `jmap` + Eclipse MAT |
| **`dotnet-gcdump`** | Lightweight GC heap snapshot for analyzing object retention | Heap dump |
| **`GC.GetTotalMemory(false)`** | Programmatic: approx bytes currently allocated on the managed heap | `Runtime.totalMemory()` |

```bash
# Watch GC and memory live for a running process (PID 1234):
dotnet-counters monitor --process-id 1234 System.Runtime

# Collect a trace, then a dump:
dotnet-trace collect --process-id 1234
dotnet-dump collect --process-id 1234
```

```csharp
long before = GC.GetTotalMemory(forceFullCollection: false); // bytes on managed heap
// ... do work ...
long after  = GC.GetTotalMemory(false);
Console.WriteLine($"Allocated ~{after - before} bytes");

// Per-generation collection counts (great for spotting GC pressure):
Console.WriteLine($"Gen0: {GC.CollectionCount(0)}, " +
                  $"Gen1: {GC.CollectionCount(1)}, " +
                  $"Gen2: {GC.CollectionCount(2)}");
```

---

## GC.Collect — and Why Not to Call It

`GC.Collect()` forces a garbage collection. **In normal application code, do not call it.**

```csharp
GC.Collect();                          // forces a collection — almost always a mistake
GC.WaitForPendingFinalizers();         // sometimes paired with the above
```

Why it's harmful:
- The GC is **self-tuning** and has far more runtime information than you do about *when* to collect.
- Forcing a collection can **promote** young objects to Gen 2 prematurely (because a forced Gen 2 GC touches everything), making future collections **more** expensive — the opposite of what you wanted.
- It introduces unnecessary **stop-the-world pauses**, hurting latency.

**Think of it like...** calling `System.gc()` in Java — universally discouraged for the same reasons; the JVM even lets you disable it. The legitimate exceptions are rare: micro-benchmarking, or right after freeing a huge batch of LOH objects in a tool. If you think you need `GC.Collect()`, you almost certainly have a real problem (a leak or excessive allocation) to fix instead.

---

## Common Interview Questions

**1. What is the CLR, and how does it relate to the JVM?**
The CLR (Common Language Runtime) is .NET's managed execution engine — the equivalent of the JVM. It JIT-compiles IL to native code, manages memory via a generational garbage collector, and provides the type system, exceptions, and threading. The main difference is that the CLR was designed to be multi-language from the start (C#, F#, VB all compile to the same IL).

**2. What is IL/MSIL? How does C# get executed?**
The C# compiler (Roslyn) compiles source to IL (Intermediate Language) stored in an assembly — analogous to Java bytecode in a `.class`. At runtime, RyuJIT compiles IL to native machine code on first use (JIT). Tiered Compilation compiles fast first (Tier 0) then re-optimizes hot methods (Tier 1), like HotSpot's C1/C2.

**3. Explain JIT vs ReadyToRun vs Native AOT.**
JIT compiles IL on demand at runtime (flexible, slower startup). ReadyToRun ships precompiled native code inside the assembly for faster startup. Native AOT compiles the whole app to a standalone native binary with no JIT — fastest startup and smallest footprint, like GraalVM native-image, at the cost of runtime dynamism (limited reflection).

**4. What's the difference between value types and reference types?**
Value types (`struct`, `int`, `bool`, `DateTime`, enums) hold their data directly and are copied by value. Reference types (`class`, `string`, arrays) hold a reference to a heap object and are copied by reference-value. The "stack vs heap" rule is a simplification — what really matters is *where the variable lives*: a struct field inside a class lives inline on the heap with that object; a struct local lives on the stack. C# lets you define your own value types, which Java can't (yet).

**5. Explain the .NET generational garbage collector.**
It's a generational, mark-and-compact tracing collector. New objects start in Gen 0 (collected often and cheaply). Survivors are promoted Gen 0 → Gen 1 → Gen 2. Gen 2 collections are full and expensive. Objects ≥ 85,000 bytes go to the Large Object Heap. It maps to the JVM's Young/Old generations (Gen 0/1 ≈ young, Gen 2 ≈ old).

**6. What is the Large Object Heap and why does it exist?**
The LOH holds objects ≥ 85,000 bytes (e.g., big arrays). They're segregated because compacting large blocks is expensive — historically the LOH wasn't compacted, which can cause fragmentation. The LOH is collected only during Gen 2 GCs. It's analogous to G1's humongous regions.

**7. Walk through how a GC collection works.**
The GC finds roots (stack locals, static fields, registers, GC handles), marks every object reachable from them, sweeps away unmarked objects, and compacts survivors so free space is contiguous (making allocation a fast pointer bump). Because objects move, you must pin them before passing addresses to native code.

**8. Workstation vs Server GC?**
Workstation GC is tuned for low latency on client apps (fewer heaps, less memory). Server GC creates a heap and GC thread per CPU core for parallel collection and high throughput — the default for ASP.NET Core. Background (concurrent) GC does most Gen 2 work on a background thread to reduce pauses. Conceptually similar to choosing pause-optimized vs throughput-optimized JVM collectors (G1/ZGC).

**9. IDisposable/using vs finalizers — when do you use each?**
`IDisposable.Dispose()` (via `using`) is deterministic cleanup for unmanaged resources — the exact analog of Java's `AutoCloseable.close()` and try-with-resources. A finalizer (`~ClassName`) is a non-deterministic safety net the GC runs only if you forgot to dispose. Prefer `Dispose`; use a finalizer only as a backstop for raw native handles (and call `GC.SuppressFinalize(this)` in `Dispose`).

**10. Why are finalizers expensive?**
An object with a finalizer can't be collected in one pass: it goes on a finalization queue, a separate finalizer thread runs the finalizer, and only the *next* GC frees it. So it survives an extra collection and gets promoted to an older generation. Finalizers run non-deterministically, in no fixed order, and may never run at all. That's why `GC.SuppressFinalize` matters and why Java deprecated `finalize()`.

**11. Can .NET have memory leaks if it has a GC?**
Yes. The GC only frees *unreachable* objects. Anything still referenced stays alive. The top causes: event handlers you never unsubscribe (publisher holds the subscriber), static fields/caches that only grow, captured closures keeping objects alive, and undisposed `IDisposable`s leaking unmanaged handles. Same failure modes as the JVM.

**12. What is boxing and why does it matter?**
Boxing wraps a value type in a heap object to treat it as `object`; unboxing extracts it. Both allocate/copy and add GC pressure. Avoid it in hot loops; use generic collections (`List<int>`, not `ArrayList`). Unlike Java's erased generics (which box `Integer`), C# generics are reified, so `List<int>` stores ints without boxing.

**13. What does `GC.Collect()` do and should you call it?**
It forces a collection. Avoid it in normal code — the GC self-tunes, and forcing collections can prematurely promote objects to Gen 2 and add stop-the-world pauses, making things worse. It's the equivalent of `System.gc()` in Java: a code smell pointing at a real allocation/leak problem to fix instead.

**14. What is `Span<T>` and why is it useful?**
`Span<T>` is a stack-only, bounds-checked view over existing memory (arrays, slices, stack buffers). Slicing allocates nothing, so it's used in high-performance, low-allocation code to avoid copies and GC pressure. `Memory<T>` is the heap-storable, async-friendly version; `stackalloc` allocates a small buffer on the stack outside the GC.

---

## Quick Reference Cheat Sheet

```text
=========================  .NET CLR / MEMORY / GC CHEAT SHEET  =========================

RUNTIME PIPELINE
  C# --(Roslyn)--> IL/MSIL --(RyuJIT)--> native code
  Tiered Compilation: Tier 0 (fast)  ->  Tier 1 (optimized, hot methods)
  ReadyToRun = precompiled native in the DLL (faster startup)
  Native AOT = whole app -> standalone native binary, no JIT (like GraalVM)

JVM -> CLR QUICK MAP
  JVM=CLR  bytecode=IL  HotSpot JIT=RyuJIT  JAR=Assembly(.dll)  JDK=BCL
  finalize()=~ClassName  AutoCloseable/try-with-resources=IDisposable/using
  System.gc()=GC.Collect()  jstat/VisualVM=dotnet-counters/dotnet-trace
  Young/Old = Gen0/Gen1 / Gen2 ;  Humongous = LOH ;  -Xmx = runtimeconfig.json

VALUE vs REFERENCE
  Value types:  struct, int, double, bool, char, DateTime, Guid, enum  -> copied by value
  Reference types: class, string, array, delegate, interface          -> heap, copied by ref
  Rule: "where does the VARIABLE live?" struct field in a class -> inline on heap

OBJECT LAYOUT (64-bit ref type)
  [ sync block index 8B ][ type pointer (MethodTable) 8B ][ fields... ]

GARBAGE COLLECTOR
  Generations:  Gen 0 (new, cheap/frequent) -> Gen 1 (buffer) -> Gen 2 (old, expensive)
  LOH: objects >= 85,000 bytes; collected with Gen 2; not compacted by default
  Algorithm: find ROOTS (stack, statics, registers, handles) -> MARK -> SWEEP -> COMPACT
  Allocation after compaction = pointer bump (fast)

GC MODES (set in runtimeconfig.json)
  Workstation GC : low latency, client apps (default desktop)
  Server GC      : 1 heap + 1 GC thread per core, throughput (default ASP.NET Core)
  Background GC  : Gen 2 work on background thread -> shorter pauses (on by default)

DETERMINISTIC CLEANUP
  using (var x = new Resource()) { ... }   // Dispose() guaranteed (== try-with-resources)
  using var x = new Resource();            // C# 8+ scoped disposal
  await using ...                          // IAsyncDisposable
  Dispose pattern: Dispose() -> GC.SuppressFinalize(this); finalizer = backstop only

FINALIZERS (~ClassName)  -- avoid; costly
  Object survives an extra GC, runs on finalizer thread, non-deterministic order/timing
  Prefer SafeHandle for native handles; call GC.SuppressFinalize when disposed

BOXING
  int x = 42; object o = x;   // boxing = heap alloc + copy ; avoid in hot loops
  Use generics (List<int>), NOT ArrayList. C# generics are reified (no boxing).

LEAKS (GC frees only UNREACHABLE objects)
  #1 event handlers not unsubscribed (-=) ; static caches ; captured closures ;
  growing collections ; undisposed IDisposable (leaks native handles)

LOW-ALLOC
  Span<T>     : stack-only view over memory, zero-alloc slicing
  Memory<T>   : heap-storable / async-friendly span
  stackalloc  : small buffer on the stack, no GC

DIAGNOSTICS
  dotnet-counters monitor --process-id <pid> System.Runtime   # live GC/mem metrics
  dotnet-trace collect / dotnet-dump collect / dotnet-gcdump  # traces & dumps
  GC.GetTotalMemory(false)        # approx managed heap bytes
  GC.CollectionCount(0|1|2)       # per-gen collection counts

DON'T
  GC.Collect()  // self-tuning GC; forcing it promotes objects + adds pauses (== System.gc())
=======================================================================================
```

*Last Updated: 2026-06-16*

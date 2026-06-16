# C# Async/Await & Concurrency — A Java Developer's Guide

## Overview

If you come from Java, you already understand threads, `ExecutorService`, `Future`, and `CompletableFuture`. C# concurrency covers the **same ideas** but the headline feature — `async`/`await` — is a *language-level* construct, not a library bolted on top. It looks like synchronous code, reads top-to-bottom, but doesn't block threads while waiting for I/O.

The single most important mental model: **`await` does not start a thread and does not block a thread.** It's closer to Java's `CompletableFuture.thenApply(...)` callback chaining — except the compiler writes the callbacks for you so the code *looks* sequential. This guide maps every concept back to something you already know in Java, and `async`/`await` is the part interviewers drill hardest, so it gets the most attention.

Target runtime: **.NET 8+** (modern C#).

---

## Table of Contents

- [Java → C# Concurrency Mapping](#java--c-concurrency-mapping)
- [1. Why Async Matters (I/O-bound vs CPU-bound)](#1-why-async-matters-io-bound-vs-cpu-bound)
- [2. Task and Task&lt;T&gt; — The Unit of Async Work](#2-task-and-taskt--the-unit-of-async-work)
- [3. async / await — How They Actually Work](#3-async--await--how-they-actually-work)
- [4. Return Types: Task, Task&lt;T&gt;, ValueTask, async void](#4-return-types-task-taskt-valuetask-async-void)
- [5. Task.Run and the ThreadPool (CPU-bound work)](#5-taskrun-and-the-threadpool-cpu-bound-work)
- [6. Awaiting Multiple Tasks: WhenAll, WhenAny](#6-awaiting-multiple-tasks-whenall-whenany)
- [7. Cancellation with CancellationToken](#7-cancellation-with-cancellationtoken)
- [8. Exception Handling in Async](#8-exception-handling-in-async)
- [9. Deadlocks: .Result/.Wait(), ConfigureAwait, SynchronizationContext](#9-deadlocks-resultwait-configureawait-synchronizationcontext)
- [10. IAsyncEnumerable and await foreach](#10-iasyncenumerable-and-await-foreach)
- [11. Lower-Level Threading Primitives](#11-lower-level-threading-primitives)
- [12. Data Parallelism: Parallel.For / PLINQ](#12-data-parallelism-parallelfor--plinq)
- [13. Thread-Safety, Race Conditions, volatile, Memory Model](#13-thread-safety-race-conditions-volatile-memory-model)
- [14. Best Practices](#14-best-practices)
- [Common Interview Questions](#common-interview-questions)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Java → C# Concurrency Mapping

| Concept | Java | C# (.NET 8+) |
|---|---|---|
| OS thread | `Thread` | `Thread` |
| Unit of work (no return) | `Runnable` | `Action` / delegate / `Task` |
| Unit of work (returns value) | `Callable<T>` | `Func<T>` / `Task<T>` |
| Thread pool / executor | `ExecutorService` (`Executors.newFixedThreadPool`) | `ThreadPool` / `Task.Run` |
| Future result handle | `Future<T>` | `Task<T>` |
| Composable async + chaining | `CompletableFuture<T>` | `Task<T>` with `async`/`await` |
| `.thenApply` / `.thenCompose` | `cf.thenApply(...)` | `await` then continue (compiler-generated) |
| Wait for all | `CompletableFuture.allOf(...)` | `Task.WhenAll(...)` |
| Wait for first | `CompletableFuture.anyOf(...)` | `Task.WhenAny(...)` |
| Block & get result | `future.get()` | `task.Result` / `task.Wait()` (**avoid**) |
| Cancellation | `Future.cancel()` / interrupt flag | `CancellationToken` + `CancellationTokenSource` |
| Mutual exclusion | `synchronized` block/method | `lock (obj) { }` (sugar for `Monitor`) |
| Lower-level lock | `ReentrantLock` | `Monitor.Enter/Exit`, `lock` |
| Visibility keyword | `volatile` | `volatile` |
| Atomic counter | `AtomicInteger` | `Interlocked.Increment/Add/Exchange` |
| Atomic reference CAS | `AtomicReference.compareAndSet` | `Interlocked.CompareExchange` |
| Latch (wait for N) | `CountDownLatch` | `CountdownEvent` (or `Task.WhenAll`) |
| Counting semaphore | `Semaphore` | `SemaphoreSlim` (async-friendly!) |
| Cross-process lock | `FileLock` / named sync | `Mutex` (named) |
| One-time signal | `CompletableFuture<Void>` | `TaskCompletionSource` |
| Concurrent map | `ConcurrentHashMap` | `ConcurrentDictionary<K,V>` |
| Blocking queue | `BlockingQueue` / `ArrayBlockingQueue` | `BlockingCollection<T>` / `Channel<T>` |
| Thread-safe list (copy) | `CopyOnWriteArrayList` | `ImmutableList<T>` / `ConcurrentBag<T>` |
| Parallel stream | `list.parallelStream()` | `Parallel.ForEach` / PLINQ (`.AsParallel()`) |
| Async stream | `Flow.Publisher` (reactive) | `IAsyncEnumerable<T>` + `await foreach` |
| Thread-local | `ThreadLocal<T>` | `ThreadLocal<T>` / `AsyncLocal<T>` |

---

## 1. Why Async Matters (I/O-bound vs CPU-bound)

**Think of it like...** a waiter at a restaurant. The *synchronous/thread-per-request* approach is hiring one waiter per table who stands frozen at the kitchen window waiting for food. The *async* approach is one waiter who takes an order, hands it to the kitchen, and immediately goes to serve another table. The waiter (thread) is never idle waiting — they only do work when there's work to do.

There are two kinds of work, and they need opposite treatment:

- **I/O-bound** — waiting on a network call, database, disk, or web API. The CPU does *nothing* during the wait. → Use `async`/`await`. No thread is held during the wait.
- **CPU-bound** — hashing, image processing, big calculations. The CPU is busy the whole time. → Use `Task.Run` to push it onto a background thread.

```csharp
// I/O-bound: the right way. While the HTTP call is in flight,
// NO thread is blocked. The thread returns to the pool to do other work.
public async Task<string> GetPageAsync(string url)
{
    using var client = new HttpClient();
    string html = await client.GetStringAsync(url); // waits WITHOUT holding a thread
    return html;
}

// CPU-bound: offload to a pool thread so the caller (e.g. UI) stays responsive.
public Task<long> ComputeAsync(int[] data)
{
    return Task.Run(() => HeavyCalculation(data)); // genuinely uses a thread
}
```

**Java contrast:** Classic Java servlets are *thread-per-request* — each blocked DB call ties up a whole thread. With 200 threads you cap at ~200 concurrent in-flight requests. C# async (and Java's newer virtual threads / `CompletableFuture`) frees the thread during the wait, so a handful of threads can serve thousands of concurrent I/O operations.

---

## 2. Task and Task&lt;T&gt; — The Unit of Async Work

**Think of it like...** `Future<T>` / `CompletableFuture<T>` in Java. A `Task` is a *promise* of work that will complete in the future. `Task` = "will finish, no value" (like `Future<Void>`/`Runnable`). `Task<T>` = "will finish and produce a T" (like `Future<T>`/`Callable<T>`).

```csharp
Task plainTask = Task.Delay(1000);          // like CompletableFuture<Void>
Task<int> valueTask = Task.FromResult(42);  // like CompletableFuture.completedFuture(42)

// A Task has state you can inspect (rarely needed when using await):
bool done   = plainTask.IsCompleted;        // similar to future.isDone()
bool faulted= plainTask.IsFaulted;          // completed with exception
bool cancel = plainTask.IsCanceled;         // was cancelled
```

You almost never inspect this state manually — you `await` instead, just as you'd prefer `cf.thenApply` over polling `future.isDone()`.

---

## 3. async / await — How They Actually Work

**Think of it like...** `CompletableFuture` callback chaining, but written by the compiler so it *looks* like normal sequential code. When you write `await`, the compiler chops your method into pieces and wires up "continuations" — exactly what `.thenApply(...)` does, just invisible.

### The key truths (interviewers love these):

1. **`async` does NOT create a thread.** It only enables `await` inside the method and changes the return type wrapping.
2. **`await` does NOT block a thread.** It *suspends* the method, returns the thread to the pool, and resumes later when the awaited task completes.
3. The compiler rewrites the method into a **state machine** — each `await` is a resume point.

```csharp
public async Task<int> GetUserAgeAsync(int id)
{
    Console.WriteLine("Before await");      // runs on caller's thread
    User user = await _db.GetUserAsync(id); // SUSPEND here. Thread is freed.
                                            // When DB completes, method RESUMES.
    Console.WriteLine("After await");       // may run on a different thread
    return user.Age;
}
```

What the compiler effectively builds (conceptually — you never write this):

```text
state 0: run code up to first await; hook a continuation onto the DB task; RETURN.
         (the thread is now free to do anything else)
state 1: DB task done -> resume here -> compute Age -> complete the returned Task.
```

**Java mental map:** `await x` is `x.thenCompose(result -> { ...rest of method... })`. The difference is purely syntactic — C# lets you keep the linear shape and `try/catch`/`using`/`for` all work normally across the `await`.

### "Async all the way" — don't break the chain

```csharp
// GOOD: await propagates up the call stack
public async Task<int> A() => await B();
public async Task<int> B() => await C();

// BAD: blocking in the middle re-introduces thread blocking + deadlock risk
public int A() => B().Result;   // <-- blocks a thread; see section 9
```

---

## 4. Return Types: Task, Task&lt;T&gt;, ValueTask, async void

**Think of it like...** choosing a `Future` flavor in Java — but with one footgun (`async void`) that has no Java equivalent and that you must learn to avoid.

```csharp
// Returns NO value -> Task (caller can await + catch exceptions)
public async Task SaveAsync(Order o) { await _db.InsertAsync(o); }

// Returns a value -> Task<T>
public async Task<decimal> GetTotalAsync(int id) { return await _db.SumAsync(id); }

// ValueTask<T>: micro-optimization to avoid allocating a Task object when
// the result is often already available (e.g. cache hit). Use sparingly.
public ValueTask<int> GetCachedAsync(int key)
{
    if (_cache.TryGetValue(key, out var v))
        return new ValueTask<int>(v);            // no heap allocation
    return new ValueTask<int>(LoadAsync(key));   // falls back to a real Task
}

// async void: AVOID. Caller cannot await it and cannot catch its exceptions
// (they crash the process). Only legitimate use: event handlers.
private async void Button_Click(object s, EventArgs e) // OK here only
{
    await DoWorkAsync();
}
```

**Why `async void` is dangerous:** an unhandled exception in an `async Task` surfaces when the caller `await`s it (catchable). An exception in `async void` has nowhere to go — it's raised on the `SynchronizationContext` and typically crashes the app. There is no Java equivalent; in Java a `CompletableFuture` always carries the exception. **Rule: every async method returns `Task`/`Task<T>` unless it is an event handler.**

`ValueTask` caveats: don't `await` it twice, don't store it, don't `WhenAll` over it. When unsure, return `Task`.

---

## 5. Task.Run and the ThreadPool (CPU-bound work)

**Think of it like...** `executorService.submit(callable)` against a shared pool. `Task.Run` queues work onto the .NET **ThreadPool** — a managed, auto-sized pool of worker threads (Java's `ForkJoinPool.commonPool()` is the closest analog).

```csharp
// Push CPU-heavy work off the current thread (e.g. off a UI thread).
int[] data = LoadData();
long sum = await Task.Run(() => data.Sum(x => Fib(x))); // runs on a pool thread

// Java equivalent:
// CompletableFuture.supplyAsync(() -> heavyCalc())  // commonPool
```

**Critical rule:** `Task.Run` is for **CPU-bound** work only. Do **not** wrap I/O in it:

```csharp
// BAD: wastes a pool thread that just sits blocked on I/O
await Task.Run(() => client.GetStringAsync(url).Result);

// GOOD: native async I/O holds no thread during the wait
await client.GetStringAsync(url);
```

The ThreadPool also backs `await` continuations, timers, and `Parallel.For`. You rarely touch raw `ThreadPool.QueueUserWorkItem` — prefer `Task.Run`.

---

## 6. Awaiting Multiple Tasks: WhenAll, WhenAny

**Think of it like...** `CompletableFuture.allOf(...)` (wait for all) and `anyOf(...)` (wait for first). This is how you run things **in parallel** instead of sequentially.

```csharp
// SEQUENTIAL (slow): total time = t1 + t2 + t3
var a = await GetAsync("a");
var b = await GetAsync("b");
var c = await GetAsync("c");

// PARALLEL (fast): kick them ALL off first, THEN await. Total ≈ max(t1,t2,t3).
Task<string> ta = GetAsync("a");   // started
Task<string> tb = GetAsync("b");   // started
Task<string> tc = GetAsync("c");   // started
string[] results = await Task.WhenAll(ta, tb, tc); // like allOf + join

// WhenAny: react to whichever finishes first (e.g. fastest mirror, or timeout).
Task<string> winner = await Task.WhenAny(ta, tb, tc);
string fastest = await winner; // unwrap the completed task's value
```

A common timeout pattern with `WhenAny`:

```csharp
var work = LongRunningAsync();
var timeout = Task.Delay(TimeSpan.FromSeconds(5));
if (await Task.WhenAny(work, timeout) == timeout)
    throw new TimeoutException();
var result = await work; // safe: it finished first
```

**Note on exceptions with `WhenAll`:** if multiple tasks fail, `await Task.WhenAll(...)` re-throws **only the first** exception. To see them all, inspect `task.Exception` (an `AggregateException`). See section 8.

---

## 7. Cancellation with CancellationToken

**Think of it like...** a cooperative version of Java's `Thread.interrupt()` + interrupt flag, or `Future.cancel(true)`. A `CancellationToken` is a signal that flows *into* your async methods; the work checks it and bows out gracefully.

```csharp
// 1. Create a source (the "controller") and pass its token down the call chain.
using var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(10)); // auto-cancel, like a timeout

try
{
    await DownloadAsync(url, cts.Token);
}
catch (OperationCanceledException) // thrown when cancellation is observed
{
    Console.WriteLine("Cancelled.");
}

// 2. Honor the token inside your method.
public async Task DownloadAsync(string url, CancellationToken ct)
{
    while (MoreData())
    {
        ct.ThrowIfCancellationRequested();        // cooperative check (like interrupt flag)
        var chunk = await ReadChunkAsync(ct);     // pass it to inner async calls too
    }
}
```

**Key points:**
- Cancellation is **cooperative** — nothing is force-killed. Your code (or the BCL methods you call) must observe the token, exactly like checking `Thread.interrupted()` in Java.
- The convention is `CancellationToken` is the **last parameter** of an async method.
- `cts.CancelAfter(...)` gives you a clean timeout. `cts.Cancel()` cancels on demand (e.g. user clicks Stop).

---

## 8. Exception Handling in Async

**Think of it like...** `try/catch` works normally — but you must catch at the `await`, because that's where a faulted task re-throws (similar to `future.get()` re-throwing the cause wrapped in `ExecutionException`).

```csharp
public async Task<int> GetAsync()
{
    try
    {
        return await CallApiAsync(); // exception surfaces HERE when awaited
    }
    catch (HttpRequestException ex)  // caught like normal synchronous code
    {
        _log.Error(ex);
        return -1;
    }
}
```

**Where exceptions surface:**
- An exception inside an `async Task` is **captured** and stored in the returned `Task`. It re-throws when you `await` (or touch `.Result`).
- If you never await/observe the task, the exception is effectively swallowed (use analyzers to catch this).
- `async void` exceptions cannot be caught by the caller — they crash the app (see section 4).

**AggregateException:** When you block with `.Result`/`.Wait()`, a failure is wrapped in `AggregateException` (mirrors Java's `ExecutionException` wrapping). `await` is smarter — it **unwraps** and throws the original exception. This is one more reason to prefer `await`.

```csharp
// Multiple failures: await throws only the FIRST; inspect Exception for all.
Task all = Task.WhenAll(t1, t2, t3);
try { await all; }
catch
{
    foreach (var ex in all.Exception!.InnerExceptions) // ALL failures
        _log.Error(ex);
}
```

---

## 9. Deadlocks: .Result/.Wait(), ConfigureAwait, SynchronizationContext

**Think of it like...** calling `future.get()` on the same single thread that's supposed to complete the future — you wait forever. The classic .NET deadlock is the async version of that self-block.

### The classic deadlock

```csharp
// In a UI app or classic ASP.NET (which have a SynchronizationContext):
public string GetData()
{
    // BAD: blocks the UI thread waiting for the task...
    return GetDataAsync().Result;
}

public async Task<string> GetDataAsync()
{
    await Task.Delay(100); // wants to RESUME on the captured UI thread...
    return "done";         // ...but the UI thread is BLOCKED on .Result. DEADLOCK.
}
```

**What's happening:** By default, `await` captures the current **SynchronizationContext** (e.g. the single UI thread, or the legacy ASP.NET request context) and tries to *resume on it*. If you block that very thread with `.Result`/`.Wait()`, the continuation can never run. Classic circular wait.

### Fixes (in order of preference)

```csharp
// 1. BEST: async all the way. Never block.
public async Task<string> GetData() => await GetDataAsync();

// 2. In library code, opt out of capturing the context so it resumes on ANY
//    pool thread. Removes the deadlock and is faster.
await Task.Delay(100).ConfigureAwait(false);
```

**`ConfigureAwait(false)`:**
- Means "I don't need to resume on the original context; any thread is fine."
- **Use it in library / non-UI code** to avoid deadlocks and reduce overhead.
- **Don't use it** in code that must touch the UI after the await (it needs the UI thread).
- Good news: **modern ASP.NET Core has NO SynchronizationContext**, so the deadlock above does *not* happen there — but `.Result`/`.Wait()` is still bad (thread-pool starvation). Java has no direct analog to `SynchronizationContext`; it's a .NET concept tying continuations to a specific thread/context.

**Rule of thumb:** never use `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` to bridge async-to-sync. Make the caller async instead.

---

## 10. IAsyncEnumerable and await foreach

**Think of it like...** a `Stream`/`Iterator` whose `hasNext()`/`next()` are themselves async — useful for paging an API or streaming DB rows without loading everything into memory. Java's closest equivalent is a reactive `Flow.Publisher`, but `IAsyncEnumerable` keeps the simple `for`-loop shape.

```csharp
// Producer: 'yield return' inside an async iterator. Each item is awaited lazily.
public async IAsyncEnumerable<int> ReadRowsAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int page = 0; ; page++)
    {
        var batch = await FetchPageAsync(page, ct); // async per page
        if (batch.Count == 0) yield break;
        foreach (var row in batch)
            yield return row;                       // streamed one at a time
    }
}

// Consumer: await foreach pulls items as they become available.
await foreach (var row in ReadRowsAsync(ct))
{
    Process(row);
}
```

This streams results — you start processing the first rows before the last page is even fetched, and memory stays flat.

---

## 11. Lower-Level Threading Primitives

When you genuinely need raw threads or locks, here are the building blocks. **Think of it like...** the `java.util.concurrent` toolbox.

### Thread (raw OS thread)

```csharp
var t = new Thread(() => Console.WriteLine("hi")); // like new Thread(runnable)
t.Start();
t.Join();                                          // like thread.join()
// Prefer Task.Run over raw Thread for most work — pooling + composition.
```

### lock / Monitor (mutual exclusion)

```csharp
private readonly object _gate = new(); // dedicated lock object (never lock 'this' or a string)

public void Add(int x)
{
    lock (_gate)            // ≈ Java synchronized(_gate) { } ; sugar over Monitor.Enter/Exit
    {
        _total += x;        // critical section — one thread at a time
    }
}
```

> Note: you **cannot** `await` inside a `lock`. For async mutual exclusion use `SemaphoreSlim`.

### Interlocked (lock-free atomics)

```csharp
private int _counter;
Interlocked.Increment(ref _counter);          // ≈ AtomicInteger.incrementAndGet()
Interlocked.Add(ref _counter, 5);             // ≈ atomicInt.addAndGet(5)
int old = Interlocked.Exchange(ref _counter, 0);        // ≈ getAndSet(0)
Interlocked.CompareExchange(ref _counter, 10, 9);       // CAS: if==9 set 10 ≈ compareAndSet
```

### SemaphoreSlim (async-friendly semaphore — also great as an async lock)

```csharp
private readonly SemaphoreSlim _sem = new(1, 1); // 1 permit = mutex; (initial, max)

public async Task UseResourceAsync()
{
    await _sem.WaitAsync();   // async acquire — does NOT block a thread (unlike Java Semaphore.acquire)
    try { await DoWorkAsync(); }
    finally { _sem.Release(); } // ALWAYS release in finally
}
// Use SemaphoreSlim(n) to throttle concurrency to n, e.g. max 5 parallel downloads.
```

### Mutex (cross-process named lock)

```csharp
using var mutex = new Mutex(false, "Global\\MyAppSingleInstance");
if (!mutex.WaitOne(0)) return; // another process holds it — exit
// Heavier than lock; only for cross-process coordination (single-instance apps).
```

### CountdownEvent (wait for N signals — like CountDownLatch)

```csharp
var latch = new CountdownEvent(3);   // ≈ new CountDownLatch(3)
// worker: latch.Signal();           // ≈ latch.countDown();
latch.Wait();                        // ≈ latch.await();
// In async code, prefer Task.WhenAll over CountdownEvent.
```

### TaskCompletionSource (manually completable task — like CompletableFuture you complete yourself)

```csharp
var tcs = new TaskCompletionSource<int>(); // ≈ new CompletableFuture<Integer>()
// elsewhere: tcs.SetResult(42);            // ≈ cf.complete(42)
//            tcs.SetException(ex);          // ≈ cf.completeExceptionally(ex)
int v = await tcs.Task;                     // await the eventual result
```

---

## 12. Data Parallelism: Parallel.For / PLINQ

**Think of it like...** Java's parallel streams (`list.parallelStream()`). This is for **CPU-bound, data-parallel** work — splitting a big collection across cores. It is *not* for I/O.

```csharp
// Parallel.For / Parallel.ForEach: run loop iterations across CPU cores.
Parallel.For(0, items.Length, i =>
{
    results[i] = Compute(items[i]);   // each iteration independent — no shared mutation!
});

Parallel.ForEach(urls, url => Process(url)); // ≈ urls.parallelStream().forEach(...)

// PLINQ: parallel LINQ. Add .AsParallel() to a query.
var heavy = numbers
    .AsParallel()                     // ≈ .parallelStream()
    .Where(n => IsPrime(n))
    .Select(n => n * n)
    .ToArray();
```

**.NET 6+ async batching** — for I/O-bound parallelism with a concurrency cap, prefer this over `Parallel.ForEach`:

```csharp
await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 5 }, // throttle to 5 at once
    async (url, ct) => await DownloadAsync(url, ct));   // async body, no thread waste
```

**Gotcha:** inside `Parallel.For`/PLINQ, never mutate shared state without synchronization — same race-condition rules as Java parallel streams.

---

## 13. Thread-Safety, Race Conditions, volatile, Memory Model

**Think of it like...** the exact same hazards as Java's memory model — non-atomic compound operations and stale cached values.

### Race condition (compound operation isn't atomic)

```csharp
private int _count;
public void Bad() => _count++;   // read-modify-write: NOT atomic. Races under threads.
public void Good() => Interlocked.Increment(ref _count); // atomic, lock-free
```

### volatile (visibility, not atomicity)

```csharp
private volatile bool _stop;     // ≈ Java volatile: guarantees other threads SEE the latest write
public void Stop() => _stop = true;
public void Loop() { while (!_stop) { /* ... */ } } // reliably sees the flag flip
```

`volatile` guarantees **visibility/ordering**, *not* atomicity of compound ops. For counters use `Interlocked`; for invariants across multiple fields use `lock`. The C# memory model, like Java's, allows reordering and per-thread caching of fields unless you use `volatile`, `lock`, or `Interlocked`.

### Concurrent collections (prefer these over manual locks)

```csharp
var map = new ConcurrentDictionary<string, int>();        // ≈ ConcurrentHashMap
map.AddOrUpdate("k", 1, (key, old) => old + 1);           // atomic upsert
map.GetOrAdd("k", _ => Compute());                        // ≈ computeIfAbsent

var bag = new ConcurrentQueue<int>();                     // thread-safe FIFO
// BlockingCollection<T> ≈ BlockingQueue;  Channel<T> = modern async producer/consumer
```

---

## 14. Best Practices

- **Async all the way.** If a method awaits, it returns `Task`/`Task<T>`, and its callers `await` it too. Don't block the chain with `.Result`/`.Wait()`.
- **Name async methods with the `Async` suffix** — `GetUserAsync`, `SaveAsync`. Convention across the whole BCL.
- **Don't "async over sync".** Don't wrap synchronous CPU code in `Task.Run` *inside a library* and call it async — that just hides a thread. Expose sync as sync; let the caller decide to `Task.Run`.
- **Don't "sync over async".** Never block on async with `.Result`/`.Wait()`/`.GetAwaiter().GetResult()`.
- **Avoid `async void`** except event handlers.
- **Pass `CancellationToken`** through your async APIs (last parameter).
- **Use `ConfigureAwait(false)`** in library code; not needed in ASP.NET Core app code.
- **Prefer `Interlocked` / concurrent collections** over manual `lock` where they fit.
- **`Task.Run` is for CPU-bound work**, not I/O.
- **Start tasks then `WhenAll`** to parallelize independent I/O.

---

## Common Interview Questions

**1. Does `await` create or block a thread?**
No. `async` doesn't create a thread; `await` doesn't block one. `await` suspends the method and returns the current thread to the pool. When the awaited operation completes, a continuation resumes the method (possibly on a different thread). It's compiler-generated callback chaining — conceptually like `CompletableFuture.thenCompose`.

**2. Difference between `Task` and `Task<T>`?**
`Task` represents async work with no result (like `Future<Void>`/`Runnable`). `Task<T>` represents async work producing a `T` (like `Future<T>`/`Callable<T>`). You `await` both; only `Task<T>` yields a value.

**3. What is `async void` and why avoid it?**
`async void` can't be awaited and its exceptions can't be caught by the caller — they propagate to the SynchronizationContext and typically crash the app. The only acceptable use is event handlers (whose signature must be `void`). Otherwise always return `Task`/`Task<T>`.

**4. I/O-bound vs CPU-bound — how do you handle each?**
I/O-bound (network/DB/disk): use native `async`/`await` — no thread is held during the wait. CPU-bound (calculations): offload with `Task.Run` so you don't block the caller (e.g. UI thread). Never wrap I/O in `Task.Run` — that wastes a pool thread sitting blocked.

**5. Explain the classic async deadlock.**
In a context with a SynchronizationContext (UI, classic ASP.NET), calling `.Result`/`.Wait()` blocks the context thread. The awaited continuation wants to resume *on that same thread*, but it's blocked — circular wait, deadlock. Fixes: async all the way, or `ConfigureAwait(false)` in library code. ASP.NET Core has no SynchronizationContext, so this specific deadlock doesn't occur there (but blocking is still bad).

**6. What does `ConfigureAwait(false)` do and when do you use it?**
It tells `await` not to capture/resume on the original SynchronizationContext — resume on any pool thread instead. Use it in library code to avoid deadlocks and reduce overhead. Don't use it where you must touch the UI after the await. Not necessary in ASP.NET Core.

**7. `Task.WhenAll` vs `Task.WhenAny`?**
`WhenAll` completes when *all* tasks complete (like `CompletableFuture.allOf`) — use to parallelize independent work. `WhenAny` completes when the *first* finishes (like `anyOf`) — use for timeouts or "fastest wins". To parallelize, start all tasks first, then `await Task.WhenAll(...)`.

**8. How does cancellation work in .NET?**
Cooperatively, via `CancellationToken`. A `CancellationTokenSource` produces the token; you pass it down and the work calls `ThrowIfCancellationRequested()` or passes it to inner async calls. Cancellation throws `OperationCanceledException`. Nothing is force-killed — like checking Java's interrupt flag.

**9. Where do exceptions surface in async code, and what's `AggregateException`?**
An exception in an `async Task` is captured in the task and re-thrown when you `await` it — so normal `try/catch` around the `await` works. `await` unwraps to the original exception. Blocking with `.Result`/`.Wait()` instead throws an `AggregateException` wrapper (like Java's `ExecutionException`). `Task.WhenAll` stores all failures in `task.Exception.InnerExceptions` but `await` re-throws only the first.

**10. `Task.Run` vs `Parallel.ForEach` vs PLINQ vs async/await — when each?**
`async`/`await`: I/O-bound concurrency. `Task.Run`: offload a single CPU-bound job. `Parallel.For/ForEach` and PLINQ (`.AsParallel()`): CPU-bound *data* parallelism across cores (like parallel streams). For throttled async I/O over a collection, use `Parallel.ForEachAsync`.

**11. `lock` vs `Interlocked` vs `SemaphoreSlim`?**
`lock` (sugar over `Monitor`) = mutual exclusion for a critical section, like `synchronized`; you cannot `await` inside it. `Interlocked` = lock-free atomic ops on a single variable (like `AtomicInteger`), fastest for counters/CAS. `SemaphoreSlim` = counting semaphore that supports `WaitAsync` — use it as an *async-compatible* lock or to cap concurrency.

**12. `volatile` — what does it guarantee?**
Visibility and ordering: a `volatile` write is seen promptly by other threads (no stale cached value), same as Java's `volatile`. It does **not** make compound operations (`x++`) atomic — use `Interlocked` or `lock` for that.

**13. What is `ValueTask` and when would you use it?**
A struct-based alternative to `Task` that avoids a heap allocation when the result is frequently already available (e.g. cache hits). Use only in hot paths; never await it twice, store it, or pass it to `WhenAll`. When in doubt, return `Task`.

**14. What is `IAsyncEnumerable<T>`?**
An asynchronous stream consumed with `await foreach`, where producing each element can itself be async (`yield return` in an `async` iterator). Ideal for paging APIs or streaming DB rows lazily with low memory — like an async `Iterator`/`Stream`.

**15. Why prefer `Task.WhenAll` over a loop of `await`s?**
Sequential `await`s run one after another (total = sum of durations). Starting all tasks first and then `await Task.WhenAll` runs them concurrently (total ≈ the slowest one), the standard way to parallelize independent I/O.

---

## Quick Reference Cheat Sheet

```text
ASYNC/AWAIT FUNDAMENTALS
  async             enables await; does NOT create a thread; wraps return in Task
  await             SUSPENDS method, frees thread; resumes on completion (no blocking)
  Task              async work, no result          (Java Future<Void> / CompletableFuture<Void>)
  Task<T>           async work returning T          (Java Future<T> / CompletableFuture<T>)
  ValueTask<T>      alloc-free Task for hot paths   (await once; don't store/reuse)
  async void        AVOID — event handlers only (uncatchable exceptions = crash)

RETURN TYPE RULE
  returns value -> Task<T>   |   returns nothing -> Task   |   event handler -> async void

I/O vs CPU
  I/O-bound  -> await native async API     (NO thread held)
  CPU-bound  -> await Task.Run(() => ...)  (uses a pool thread)
  NEVER wrap I/O in Task.Run.

PARALLELISM
  Task.WhenAll(t1,t2,t3)   wait for ALL    (Java allOf) — start tasks FIRST, then await
  Task.WhenAny(t1,t2)      wait for FIRST  (Java anyOf) — timeouts / fastest-wins
  Parallel.For / ForEach   CPU data parallelism (Java parallelStream)
  Parallel.ForEachAsync    async I/O w/ MaxDegreeOfParallelism throttle
  list.AsParallel()...     PLINQ parallel query

CANCELLATION
  var cts = new CancellationTokenSource();  cts.CancelAfter(ts);  cts.Cancel();
  ct.ThrowIfCancellationRequested();        // cooperative check
  catch (OperationCanceledException)        // observed cancellation
  Convention: CancellationToken is the LAST parameter.

EXCEPTIONS
  await -> re-throws original exception (try/catch as normal)
  .Result/.Wait() -> wraps in AggregateException (Java ExecutionException)
  WhenAll -> await throws first; task.Exception.InnerExceptions = all

DEADLOCK AVOIDANCE
  NEVER .Result / .Wait() / .GetAwaiter().GetResult() to bridge async->sync
  Async all the way.  In libraries: await x.ConfigureAwait(false);
  ASP.NET Core has NO SynchronizationContext (that deadlock won't occur there).

THREADING PRIMITIVES
  Thread                 raw OS thread (prefer Task.Run)
  lock(obj){ }           mutual exclusion (Java synchronized); NO await inside
  Monitor.Enter/Exit     what lock compiles to
  Interlocked.Increment  atomic counter (Java AtomicInteger)
  Interlocked.CompareExchange   CAS (Java compareAndSet)
  SemaphoreSlim(1,1)     async lock; WaitAsync()/Release() (Java Semaphore, async)
  Mutex (named)          cross-process lock
  CountdownEvent         wait for N signals (Java CountDownLatch)
  TaskCompletionSource   manually completable Task (Java CompletableFuture you complete)
  volatile               visibility only, NOT atomic (Java volatile)

CONCURRENT COLLECTIONS
  ConcurrentDictionary<K,V>   Java ConcurrentHashMap (AddOrUpdate / GetOrAdd)
  ConcurrentQueue/Bag/Stack   thread-safe collections
  BlockingCollection<T>       Java BlockingQueue
  Channel<T>                  modern async producer/consumer

ASYNC STREAMS
  async IAsyncEnumerable<T> ... yield return ...   // async producer
  await foreach (var x in stream) { ... }          // async consumer

BEST PRACTICES
  + Async all the way        + Name *Async       + Pass CancellationToken
  + ConfigureAwait(false) in libs                 + Task.Run = CPU only
  - No async void (except events)  - No .Result/.Wait()  - Don't async-over-sync
```

---

*Last Updated: 2026-06-16*

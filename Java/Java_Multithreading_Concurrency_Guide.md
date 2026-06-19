# Java Multithreading & Concurrency Interview Guide

## Overview

Concurrency is a heavily tested Java interview topic. This guide covers thread lifecycle, synchronization, the `java.util.concurrent` package, and common concurrency pitfalls — at a junior interview level.

---

## Table of Contents

1. [Thread Basics](#thread-basics)
2. [Thread Lifecycle](#thread-lifecycle)
3. [Creating Threads](#creating-threads)
4. [Synchronization](#synchronization)
5. [volatile Keyword](#volatile-keyword)
6. [Atomic Classes & CAS](#atomic-classes--cas)
7. [java.util.concurrent Package](#javautilconcurrent-package)
8. [ExecutorService & Thread Pools](#executorservice--thread-pools)
9. [Locks](#locks)
10. [Concurrent Collections](#concurrent-collections)
11. [Deadlock, Livelock, Starvation](#deadlock-livelock-starvation)
12. [Common Interview Questions](#common-interview-questions)
13. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Thread Basics

```
Process → independent program with its own memory space
Thread  → lightweight unit within a process; shares heap, has its own stack
```

### Why Concurrency?
- **CPU utilization** — while one thread waits for I/O, another runs
- **Responsiveness** — UI thread stays live while background threads work
- **Performance** — parallel execution on multi-core CPUs

### Concurrency vs Parallelism
```
Concurrency  → multiple tasks make progress by interleaving (any number of cores)
Parallelism  → multiple tasks execute at the exact same time (requires multi-core)
```

---

## Thread Lifecycle

```
              start()
NEW ─────────────────────► RUNNABLE
                               │
                           RUNNING
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
    TIMED_WAITING          WAITING            BLOCKED
    (sleep, join        (wait(), join()    (waiting for
     with timeout)       without timeout)   monitor lock)
            └──────────────────┴──────────────────┘
                               │
                           TERMINATED
```

| State | Cause |
|---|---|
| `NEW` | Thread created, `start()` not yet called |
| `RUNNABLE` | Ready to run or currently running |
| `BLOCKED` | Waiting to acquire a monitor lock |
| `WAITING` | `wait()`, `join()`, `LockSupport.park()` — no timeout |
| `TIMED_WAITING` | `sleep()`, `wait(timeout)`, `join(timeout)` |
| `TERMINATED` | `run()` finished or exception thrown |

---

## Creating Threads

### Method 1: Extend Thread
```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running: " + Thread.currentThread().getName());
    }
}
new MyThread().start(); // never call run() directly
```

### Method 2: Implement Runnable (preferred)
```java
Runnable task = () -> System.out.println("Running: " + Thread.currentThread().getName());
new Thread(task).start();
```

### Method 3: Callable + Future (returns a result)
```java
Callable<Integer> task = () -> { Thread.sleep(1000); return 42; };
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(task);
Integer result = future.get(); // blocks until result is ready
executor.shutdown();
```

> **Interview Tip**: `Runnable` cannot return a result or throw checked exceptions. `Callable` can do both — use it with `ExecutorService` when you need a return value.

---

## Synchronization

### The Problem: Race Condition
```java
// NOT thread-safe — count++ is 3 steps: read → modify → write
class Counter {
    private int count = 0;
    public void increment() { count++; }
}
```

### synchronized Method
```java
class Counter {
    private int count = 0;
    public synchronized void increment() { count++; }      // locks "this"
    public synchronized int getCount()   { return count; }
}
```

### synchronized Block (finer-grained)
```java
private final Object lock = new Object();
public void increment() {
    synchronized (lock) { count++; } // only this block is locked
}
```

### Key Rules
- Instance `synchronized` method → locks `this`
- Static `synchronized` method → locks `ClassName.class`
- `synchronized(obj)` block → locks `obj`

---

## volatile Keyword

`volatile` guarantees **visibility**: writes are immediately visible to all threads (no stale CPU-cache reads). It does NOT guarantee atomicity.

```java
class Flag {
    private volatile boolean running = true;

    public void stop() { running = false; }
    public void run() {
        while (running) { /* always reads from main memory */ }
    }
}
```

### volatile vs synchronized

| | `volatile` | `synchronized` |
|---|---|---|
| Visibility | Yes | Yes |
| Atomicity | No | Yes |
| Mutual Exclusion | No | Yes |
| Performance | Faster | Slower |
| Use Case | Simple flags/status | Compound operations |

> **Interview Tip**: `count++` is NOT safe with `volatile` — it's still 3 operations. Use `AtomicInteger` or `synchronized` for compound operations.

---

## Atomic Classes & CAS

The `java.util.concurrent.atomic` classes (`AtomicInteger`, `AtomicLong`, `AtomicReference`, etc.) give you lock-free, thread-safe updates for compound operations like `count++`. They are built on **CAS (Compare-And-Swap)**.

### What is CAS (Compare-And-Swap)?

CAS is a single hardware-level (atomic) CPU instruction that updates a value **only if it hasn't changed** since you read it. It takes three arguments:

```
CAS(memoryLocation, expectedValue, newValue)
```

- Read the current value at the location.
- **If** it still equals `expectedValue` → write `newValue` and return `true` (success).
- **Else** (another thread changed it) → do nothing and return `false` (retry).

Because the compare and the swap happen as one indivisible CPU operation, no other thread can sneak in between them. This is the foundation of "lock-free" concurrency.

### The retry loop (optimistic locking)

`AtomicInteger.incrementAndGet()` is essentially a CAS in a loop. It keeps retrying until the swap succeeds:

```java
// Conceptual implementation of getAndIncrement()
public int getAndIncrement() {
    int current;
    do {
        current = get();                       // 1. read current value
    } while (!compareAndSet(current, current + 1)); // 2. swap if unchanged, else retry
    return current;
}
```

This is **optimistic locking**: assume no conflict, and only retry on the rare occasion another thread won the race.

### Example

```java
AtomicInteger counter = new AtomicInteger(0);

counter.incrementAndGet();           // atomic ++counter
counter.getAndAdd(5);                // atomic counter += 5
counter.compareAndSet(6, 100);       // set to 100 only if currently 6

AtomicReference<String> ref = new AtomicReference<>("A");
ref.compareAndSet("A", "B");         // swap reference atomically
```

### CAS vs locks

| | CAS (lock-free) | `synchronized` / locks |
|---|---|---|
| Blocking | Non-blocking (busy-retry) | Blocks waiting threads |
| Best under | Low–moderate contention | High contention |
| Overhead | No context switching | Thread parking/waking |
| Deadlock risk | None | Possible |

### The ABA problem

CAS only checks that the value **equals** the expected value — not that it never changed. If a value goes `A → B → A`, CAS still succeeds even though it was modified in between. This matters for things like lock-free stacks where a reused node looks "unchanged."

**Fix:** use `AtomicStampedReference`, which pairs the value with a version stamp so `A(v1) → B → A(v2)` is detected as a change.

> **Interview Tip**: CAS powers `AtomicInteger`, `ConcurrentHashMap` writes, and `ConcurrentLinkedQueue`. Strength: no locks, no blocking. Weakness: under **high contention** the retry loop wastes CPU (many failed attempts), and it's vulnerable to the **ABA problem**.

---

## java.util.concurrent Package

Introduced in Java 5 — high-level concurrency utilities replacing manual `synchronized` code.

```
java.util.concurrent
  ├── Executors / ExecutorService  — thread pool management
  ├── CompletableFuture            — async pipeline
  ├── Locks: ReentrantLock, ReadWriteLock
  ├── Atomic classes               — lock-free thread-safe operations
  ├── Concurrent collections       — thread-safe data structures
  ├── Synchronizers: CountDownLatch, CyclicBarrier, Semaphore
  └── BlockingQueues: LinkedBlockingQueue, ArrayBlockingQueue
```

---

## ExecutorService & Thread Pools

A thread pool reuses threads, avoiding the cost of creating/destroying one per task.

### Common Pool Types
```java
ExecutorService fixed    = Executors.newFixedThreadPool(4);      // N threads, tasks queue when all busy
ExecutorService single   = Executors.newSingleThreadExecutor();  // sequential execution
ExecutorService cached   = Executors.newCachedThreadPool();      // grows unbounded — use carefully
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);
scheduled.scheduleAtFixedRate(task, 0, 1, TimeUnit.MINUTES);
```

### ThreadPoolExecutor (fine-grained control)
```java
ExecutorService executor = new ThreadPoolExecutor(
    2,                               // corePoolSize
    10,                              // maximumPoolSize
    60L, TimeUnit.SECONDS,           // idle thread timeout
    new ArrayBlockingQueue<>(100),   // task queue
    new ThreadPoolExecutor.CallerRunsPolicy() // rejection policy
);
```

### Rejection Policies
| Policy | Behavior |
|---|---|
| `AbortPolicy` (default) | Throw `RejectedExecutionException` |
| `CallerRunsPolicy` | Caller thread runs the task (back-pressure) |
| `DiscardPolicy` | Silently drop the task |
| `DiscardOldestPolicy` | Drop oldest queued task, retry |

### Submitting & Shutting Down
```java
Future<String> f = executor.submit(() -> computeResult());
String result    = f.get();                        // blocks
String result    = f.get(5, TimeUnit.SECONDS);     // blocks with timeout

List<Future<String>> all  = executor.invokeAll(callables);  // wait for all
String first              = executor.invokeAny(callables);  // first to finish

executor.shutdown();                               // finish existing tasks, no new ones
executor.awaitTermination(60, TimeUnit.SECONDS);
```

---

## Locks

### ReentrantLock
```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // critical section
} finally {
    lock.unlock(); // always in finally
}

// Try without blocking
if (lock.tryLock(5, TimeUnit.SECONDS)) {
    try { ... } finally { lock.unlock(); }
}
```

### ReentrantReadWriteLock
Multiple readers simultaneously; writers need exclusive access. Best for read-heavy workloads.

```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
// reads: rwLock.readLock().lock() / unlock()
// writes: rwLock.writeLock().lock() / unlock()
```

### synchronized vs ReentrantLock

| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| Syntax | Simple | Verbose (lock/unlock) |
| Interruptible | No | Yes (`lockInterruptibly()`) |
| Trylock | No | Yes |
| Fairness | No | Yes (`new ReentrantLock(true)`) |
| Multiple conditions | No | Yes |

---

## Concurrent Collections

### ConcurrentHashMap
Thread-safe map using CAS + node-level locking (Java 8+). Concurrent reads + fine-grained writes.

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.putIfAbsent("key", 1);                         // atomic check-then-put
map.compute("key", (k, v) -> v == null ? 1 : v+1); // atomic update
map.computeIfAbsent("key", k -> expensiveCompute(k));
```

### BlockingQueue (producer-consumer)
```java
BlockingQueue<String> queue = new LinkedBlockingQueue<>(100);
queue.put("task");   // blocks if full
String t = queue.take(); // blocks if empty
```

### Other Collections

| Class | Description |
|---|---|
| `CopyOnWriteArrayList` | Thread-safe list; writes copy the array (read-heavy) |
| `ConcurrentLinkedQueue` | Non-blocking FIFO using CAS |
| `PriorityBlockingQueue` | Blocking queue with priority ordering |
| `ConcurrentSkipListMap` | Sorted concurrent map (like TreeMap) |

---

## Deadlock, Livelock, Starvation

### Deadlock
Thread A holds Lock 1, waits for Lock 2. Thread B holds Lock 2, waits for Lock 1. Neither can proceed.

```java
// Deadlock: t1 locks lock1 then lock2; t2 locks lock2 then lock1 (reversed order)
```

**Prevention**:
1. **Lock ordering** — always acquire locks in the same global order
2. **tryLock with timeout** — avoids blocking forever
3. **Avoid nested locks** where possible

### Livelock
Threads are active but keep reacting to each other, making no progress (like two people in a corridor stepping the same direction).

### Starvation
A thread is perpetually denied CPU time by higher-priority threads. Fix: `new ReentrantLock(true)` (fair/FIFO lock).

---

## Common Interview Questions

### Q: What is the difference between `wait()` and `sleep()`?

| | `wait()` | `sleep()` |
|---|---|---|
| Class | `Object` | `Thread` |
| Releases lock | Yes | No |
| Wake up | `notify()` / `notifyAll()` | After timeout |
| Requires `synchronized` | Yes | No |

---

### Q: What is the difference between `notify()` and `notifyAll()`?

`notify()` wakes one arbitrary waiting thread; `notifyAll()` wakes all of them (they compete for the lock). Prefer `notifyAll()` unless you are certain only one waiting thread can proceed — `notify()` can cause missed signals.

---

### Q: What is a daemon thread?

A background thread that is automatically killed when all non-daemon threads finish. Set before `start()`: `t.setDaemon(true)`. Used for GC, logging, heartbeats.

---

### Q: How does `ConcurrentHashMap` differ from `HashMap` and `Hashtable`?

`HashMap` is not thread-safe. `Hashtable` is thread-safe but uses a single lock on every operation — poor concurrency. `ConcurrentHashMap` uses CAS + node-level locking, allowing concurrent reads and fine-grained writes with much better throughput.

---

### Q: What is the difference between `Runnable` and `Callable`?

`Runnable` returns void and cannot throw checked exceptions. `Callable<V>` returns a value and can throw checked exceptions. Use `Callable` with `ExecutorService` when you need a result or need to propagate exceptions.

---

### Q: What is a ThreadLocal?

Gives each thread its own independent copy of a variable — threads never share it. Common use: `SimpleDateFormat` (not thread-safe), per-request context in web apps. Always call `remove()` when done in thread-pool threads to prevent memory leaks.

```java
ThreadLocal<SimpleDateFormat> fmt = ThreadLocal.withInitial(
    () -> new SimpleDateFormat("yyyy-MM-dd"));
fmt.get().format(new Date());
fmt.remove(); // prevent leak in pooled threads
```

---

## Quick Reference Cheat Sheet

```
Thread states: NEW → RUNNABLE ↔ BLOCKED/WAITING/TIMED_WAITING → TERMINATED

synchronized  → intrinsic lock; visibility + atomicity
volatile      → visibility only; NOT atomic for compound ops
AtomicInteger → lock-free CAS; atomic compound ops

Thread creation:
  extends Thread         → simple, inflexible
  implements Runnable    → preferred (no return)
  implements Callable    → use with ExecutorService (return + exceptions)

Thread Pools:
  newFixedThreadPool(n)  → fixed n threads
  newCachedThreadPool()  → unbounded (use carefully)
  new ThreadPoolExecutor(core, max, ttl, queue, rejectPolicy)

Locks:
  ReentrantLock          → tryLock, interruptible, fair mode
  ReentrantReadWriteLock → many readers OR one writer
  synchronized           → simpler, good for most cases

Deadlock prevention:
  1. Consistent lock ordering
  2. tryLock with timeout
  3. Avoid nested locks

Collections:
  ConcurrentHashMap    → thread-safe map, fine-grained locking
  CopyOnWriteArrayList → read-heavy lists
  BlockingQueue        → producer-consumer (put/take block)

wait() vs sleep():
  wait()  → releases lock, needs notify() to wake
  sleep() → holds lock, wakes after timeout
```

---

*Last Updated: 2026-06-18*

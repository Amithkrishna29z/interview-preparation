# Java Multithreading & Concurrency Interview Guide

## Overview

Concurrency is one of the most heavily tested Java interview topics for mid-to-senior roles. This guide covers thread lifecycle, synchronization, the `java.util.concurrent` package, CompletableFuture, and common concurrency patterns and pitfalls.

---

## Table of Contents

1. [Thread Basics](#thread-basics)
2. [Thread Lifecycle](#thread-lifecycle)
3. [Creating Threads](#creating-threads)
4. [Synchronization](#synchronization)
5. [volatile Keyword](#volatile-keyword)
6. [java.util.concurrent Package](#javautilconcurrent-package)
7. [ExecutorService & ThreadPools](#executorservice--thread-pools)
8. [CompletableFuture](#completablefuture)
9. [Locks (ReentrantLock, ReadWriteLock)](#locks)
10. [Atomic Classes](#atomic-classes)
11. [Concurrent Collections](#concurrent-collections)
12. [Deadlock, Livelock, Starvation](#deadlock-livelock-starvation)
13. [Java Memory Model](#java-memory-model)
14. [Common Interview Questions](#common-interview-questions)
15. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Thread Basics

```
Process → independent program with its own memory space
Thread  → lightweight unit of execution within a process
          shares heap memory with other threads in the same process
          has its own stack, registers, and program counter
```

### Why Concurrency?

- **Better CPU utilization** — while one thread waits for I/O, another executes
- **Responsiveness** — UI thread stays responsive while background threads work
- **Performance** — parallel execution on multi-core CPUs

### Concurrency vs Parallelism

```
Concurrency  → multiple tasks make progress by interleaving (single or multi-core)
Parallelism  → multiple tasks execute simultaneously (requires multi-core)
```

---

## Thread Lifecycle

```
              start()
NEW ─────────────────────► RUNNABLE
                               │  ▲
                   scheduler   │  │ scheduled
                   preempts    ▼  │ again
                            RUNNING
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
    TIMED_WAITING          WAITING            BLOCKED
    (sleep, join        (wait(), join()    (waiting for
     with timeout)       without timeout)   monitor lock)
            │                  │                  │
            └──────────────────┴──────────────────┘
                               │
                           TERMINATED
                         (run() returns
                          or exception)
```

| State | Cause |
|---|---|
| `NEW` | Thread created but `start()` not called |
| `RUNNABLE` | Ready to run or running |
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
MyThread t = new MyThread();
t.start(); // starts new thread; never call run() directly
```

### Method 2: Implement Runnable (preferred)

```java
Runnable task = () -> System.out.println("Running: " + Thread.currentThread().getName());
Thread t = new Thread(task);
t.start();
```

### Method 3: Callable + Future (returns a result)

```java
Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 42;
};
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(task);
Integer result = future.get(); // blocks until result is ready
executor.shutdown();
```

> **Interview Tip**: `Runnable` cannot return a result or throw checked exceptions. `Callable` can do both. Use `Callable` with `ExecutorService` for tasks that return results.

---

## Synchronization

### The Problem: Race Condition

```java
// NOT thread-safe — race condition on counter
class Counter {
    private int count = 0;

    public void increment() {
        count++; // NOT atomic: read → modify → write (3 steps)
    }
}
// Two threads calling increment() simultaneously can lose updates
```

### synchronized Method

```java
class Counter {
    private int count = 0;

    // Lock is acquired on "this" object
    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }
}
```

### synchronized Block (finer-grained)

```java
class Counter {
    private int count = 0;
    private final Object lock = new Object(); // explicit lock object

    public void increment() {
        synchronized (lock) { // only this block is synchronized, not whole method
            count++;
        }
        // code here runs without lock
    }
}
```

### static synchronized

```java
// Lock is acquired on the Class object (shared across all instances)
public static synchronized void classLevelMethod() { ... }
```

### Intrinsic Lock (Monitor Lock)

Every Java object has an implicit lock (monitor). `synchronized` acquires this lock:
- Method `synchronized` → locks `this`
- Static `synchronized` → locks `ClassName.class`
- Block `synchronized(obj)` → locks `obj`

---

## volatile Keyword

`volatile` ensures:
1. **Visibility** — writes to a volatile variable are immediately visible to all threads (no CPU cache stale reads)
2. **No instruction reordering** around the volatile variable

```java
class Flag {
    private volatile boolean running = true; // without volatile, other threads may see stale value

    public void stop() { running = false; }

    public void run() {
        while (running) { // always reads fresh value from main memory
            // do work
        }
    }
}
```

### volatile vs synchronized

| | `volatile` | `synchronized` |
|---|---|---|
| Visibility | Yes | Yes |
| Atomicity | No (only single read/write) | Yes (block is atomic) |
| Mutual Exclusion | No | Yes |
| Performance | Faster | Slower |
| Use Case | Simple flags, status | Compound operations, critical sections |

> **Interview Tip**: `volatile` makes a single variable thread-safe but `count++` is NOT atomic even with `volatile` (it's 3 operations). Use `AtomicInteger` or `synchronized` for compound operations.

---

## java.util.concurrent Package

Introduced in Java 5 — provides high-level concurrency utilities that replace manual `synchronized` code.

```
java.util.concurrent
  ├── Executors / ExecutorService  — thread pool management
  ├── CompletableFuture            — async pipeline
  ├── Locks: ReentrantLock, ReadWriteLock, StampedLock
  ├── Atomic classes               — lock-free thread-safe operations
  ├── Concurrent collections       — thread-safe data structures
  ├── Synchronizers: CountDownLatch, CyclicBarrier, Semaphore, Phaser
  └── BlockingQueues: LinkedBlockingQueue, ArrayBlockingQueue, PriorityBlockingQueue
```

---

## ExecutorService & Thread Pools

A thread pool reuses a fixed number of threads, avoiding the overhead of creating/destroying threads for every task.

### Creating Thread Pools

```java
// Fixed pool — N threads, tasks queue if all busy
ExecutorService fixed = Executors.newFixedThreadPool(4);

// Single thread — tasks execute sequentially
ExecutorService single = Executors.newSingleThreadExecutor();

// Cached pool — creates new threads as needed, reuses idle ones (no upper limit!)
ExecutorService cached = Executors.newCachedThreadPool();

// Scheduled — for delayed/periodic tasks
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);
scheduled.schedule(task, 5, TimeUnit.SECONDS);        // one-time delay
scheduled.scheduleAtFixedRate(task, 0, 1, TimeUnit.MINUTES); // periodic
```

### ThreadPoolExecutor (low-level control)

```java
ExecutorService executor = new ThreadPoolExecutor(
    2,                          // corePoolSize: threads always alive
    10,                         // maximumPoolSize: max threads under load
    60L, TimeUnit.SECONDS,      // keepAliveTime: idle thread timeout
    new ArrayBlockingQueue<>(100), // work queue: holds pending tasks
    new ThreadPoolExecutor.CallerRunsPolicy() // rejection policy
);
```

### Rejection Policies

| Policy | Behavior |
|---|---|
| `AbortPolicy` (default) | Throw `RejectedExecutionException` |
| `CallerRunsPolicy` | Caller thread runs the task itself (back-pressure) |
| `DiscardPolicy` | Silently discard the task |
| `DiscardOldestPolicy` | Discard oldest queued task, retry submission |

### Submitting Tasks

```java
// submit Runnable (no result)
Future<?> future = executor.submit(() -> doWork());

// submit Callable (has result)
Future<String> future = executor.submit(() -> computeResult());
String result = future.get();             // blocks
String result = future.get(5, TimeUnit.SECONDS); // blocks with timeout

// invokeAll — submit many, wait for all
List<Future<String>> futures = executor.invokeAll(callables);

// invokeAny — submit many, return first result
String first = executor.invokeAny(callables);

// Shutdown (always do this)
executor.shutdown();                      // no new tasks, finish existing
executor.awaitTermination(60, TimeUnit.SECONDS);
executor.shutdownNow();                   // interrupt running tasks
```

---

## CompletableFuture

Introduced in Java 8 — enables asynchronous, non-blocking, pipeline-style programming.

```java
// Async computation
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    Thread.sleep(1000);
    return "Hello";
});

// Chain transformations
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> fetchUser(userId))        // async: fetch user
    .thenApply(user -> user.getName())           // sync transform
    .thenApply(String::toUpperCase)              // sync transform
    .thenCompose(name -> fetchOrders(name))      // async chain (flatMap)
    .exceptionally(ex -> "Default");             // handle error

// Combine two futures
CompletableFuture<String> combined = CompletableFuture
    .supplyAsync(() -> "Hello")
    .thenCombine(
        CompletableFuture.supplyAsync(() -> " World"),
        (s1, s2) -> s1 + s2
    ); // "Hello World"

// Wait for all
CompletableFuture.allOf(future1, future2, future3)
    .thenRun(() -> System.out.println("All done"));

// First to complete
CompletableFuture.anyOf(future1, future2, future3)
    .thenAccept(result -> System.out.println("First: " + result));

// Manual completion
CompletableFuture<String> manual = new CompletableFuture<>();
manual.complete("done");        // resolve
manual.completeExceptionally(new RuntimeException("failed")); // reject

// Error handling
future
    .thenApply(val -> process(val))
    .handle((result, ex) -> {   // handle both success and failure
        if (ex != null) return "error: " + ex.getMessage();
        return result;
    });
```

### thenApply vs thenCompose vs thenAccept

| Method | Input | Returns | Use For |
|---|---|---|---|
| `thenApply(fn)` | Result | `CF<U>` | Sync transform (like map) |
| `thenCompose(fn)` | Result | `CF<U>` | Async chain (like flatMap) |
| `thenAccept(consumer)` | Result | `CF<Void>` | Consume result, no return |
| `thenRun(runnable)` | Nothing | `CF<Void>` | Run after completion |

---

## Locks

### ReentrantLock (explicit lock)

```java
ReentrantLock lock = new ReentrantLock();

// Basic usage
lock.lock();
try {
    // critical section
} finally {
    lock.unlock(); // ALWAYS in finally
}

// Try to acquire without blocking
if (lock.tryLock()) {
    try { ... } finally { lock.unlock(); }
}

// Try with timeout
if (lock.tryLock(5, TimeUnit.SECONDS)) {
    try { ... } finally { lock.unlock(); }
}

// Condition variables (like wait/notify but more flexible)
Condition condition = lock.newCondition();
lock.lock();
try {
    while (!conditionMet) condition.await();  // like wait()
    // do work
    condition.signalAll();                    // like notifyAll()
} finally {
    lock.unlock();
}
```

### ReentrantReadWriteLock

Multiple readers can read simultaneously; writers need exclusive access.

```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock  = rwLock.readLock();
Lock writeLock = rwLock.writeLock();

public String read() {
    readLock.lock();
    try { return data; }
    finally { readLock.unlock(); }
}

public void write(String newData) {
    writeLock.lock();
    try { data = newData; }
    finally { writeLock.unlock(); }
}
```

> **Use when**: Many reads, few writes (e.g., caching, config). `ReadWriteLock` dramatically improves throughput for read-heavy workloads.

### synchronized vs ReentrantLock

| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| Syntax | Simple | Verbose (lock/unlock) |
| Interruptible | No | Yes (`lockInterruptibly()`) |
| Trylock | No | Yes (`tryLock()`) |
| Fairness | No | Yes (new ReentrantLock(true)) |
| Condition variables | `wait()/notify()` | Multiple `Condition` objects |
| Performance | Good | Better under high contention |

---

## Atomic Classes

Lock-free thread-safe operations using CPU-level Compare-And-Swap (CAS) instructions.

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();   // atomic ++
counter.decrementAndGet();   // atomic --
counter.addAndGet(5);        // atomic += 5
counter.getAndAdd(5);        // returns old value, then +=5
counter.compareAndSet(10, 20); // if value==10, set to 20 (CAS)

AtomicLong longCounter = new AtomicLong();
AtomicBoolean flag = new AtomicBoolean(false);
AtomicReference<String> ref = new AtomicReference<>("initial");
ref.compareAndSet("initial", "updated");

// Java 8+ adders — better performance under high contention
LongAdder adder = new LongAdder();
adder.increment();
adder.add(5);
adder.sum(); // approximate (exact after all writes complete)
```

> **CAS (Compare-And-Swap)**: Atomic hardware instruction — "set value to new only if current equals expected". No locks needed. Under high contention, `LongAdder` outperforms `AtomicLong`.

---

## Concurrent Collections

### ConcurrentHashMap

Thread-safe HashMap. Uses **segment locking** (Java 7) / **CAS + node locking** (Java 8+) — allows concurrent reads and fine-grained writes without blocking the entire map.

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("key", 1);
map.get("key");
map.putIfAbsent("key", 2);          // atomic check-then-put
map.computeIfAbsent("key", k -> expensiveCompute(k)); // compute only if absent
map.compute("key", (k, v) -> v == null ? 1 : v + 1); // atomic update

// forEach with parallelism threshold
map.forEach(2, (k, v) -> process(k, v)); // parallel if size > 2
```

> **vs synchronized HashMap**: `Collections.synchronizedMap()` locks the entire map per operation. `ConcurrentHashMap` allows concurrent reads + fine-grained writes.

### BlockingQueue

```java
BlockingQueue<String> queue = new LinkedBlockingQueue<>(100);

// Producer thread
queue.put("task");    // blocks if queue is full
queue.offer("task", 5, TimeUnit.SECONDS); // waits up to 5s

// Consumer thread
String task = queue.take();            // blocks if queue is empty
String task = queue.poll(5, TimeUnit.SECONDS); // waits up to 5s
```

### Other Concurrent Collections

| Class | Description |
|---|---|
| `CopyOnWriteArrayList` | Thread-safe list; writes copy entire array (read-heavy) |
| `ConcurrentLinkedQueue` | Non-blocking FIFO queue using CAS |
| `LinkedBlockingDeque` | Double-ended blocking queue |
| `PriorityBlockingQueue` | Blocking queue with priority ordering |
| `ConcurrentSkipListMap` | Sorted concurrent map (like TreeMap) |

---

## Deadlock, Livelock, Starvation

### Deadlock

Thread A holds Lock 1, waits for Lock 2.
Thread B holds Lock 2, waits for Lock 1.
Neither can proceed.

```java
// Classic deadlock
Object lock1 = new Object(), lock2 = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lock1) {
        Thread.sleep(50);
        synchronized (lock2) { /* work */ }
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lock2) {
        synchronized (lock1) { /* work */ } // reversed order!
    }
});
```

**Prevention**:
1. **Lock ordering** — always acquire locks in the same global order
2. **tryLock with timeout** — `lock.tryLock(timeout)` avoids blocking forever
3. **Avoid nested locks** where possible

### Livelock

Threads are active but keep responding to each other's state, making no progress (like two people in a hallway constantly stepping aside in the same direction).

### Starvation

A thread is perpetually denied CPU time because higher-priority threads keep running. Fix with `ReentrantLock(true)` (fair lock — FIFO ordering).

---

## Java Memory Model

### Happens-Before Relationship

JMM defines when writes by one thread are **guaranteed to be visible** to another:
1. Thread start: writes before `t.start()` are visible in new thread
2. Thread join: writes in thread t are visible after `t.join()` returns
3. Synchronized: unlock of monitor → happens-before → next lock of same monitor
4. Volatile: write to volatile → happens-before → subsequent read of same variable

### Memory Visibility Problem

```java
// Without volatile or sync — y change may NEVER be visible to Thread 2
boolean flag = false;       // not volatile!

// Thread 1
flag = true;

// Thread 2 — may loop forever because it caches flag in CPU register
while (!flag) { /* spin */ }
```

---

## Common Interview Questions

### Q: What is the difference between `wait()` and `sleep()`?

| | `wait()` | `sleep()` |
|---|---|---|
| Class | `Object` | `Thread` |
| Releases lock | Yes — releases monitor | No — holds lock |
| Wake up | `notify()` / `notifyAll()` | After timeout |
| Must be in `synchronized` | Yes | No |
| Use case | Thread coordination | Pause execution |

---

### Q: What is the difference between `notify()` and `notifyAll()`?

- `notify()`: wakes ONE waiting thread (undefined which one)
- `notifyAll()`: wakes ALL waiting threads (they compete for the lock)

Use `notifyAll()` unless you are sure only one waiting thread can proceed — `notify()` can cause missed signals.

---

### Q: What is a daemon thread?

A daemon thread runs in the background and is automatically killed when all non-daemon (user) threads finish. Used for background services (GC, logging, heartbeats).

```java
Thread t = new Thread(() -> { while(true) { /* background work */ } });
t.setDaemon(true); // must be set before start()
t.start();
```

---

### Q: How does `ConcurrentHashMap` differ from `HashMap` and `Hashtable`?

- `HashMap`: Not thread-safe; concurrent access causes data corruption
- `Hashtable`: Thread-safe but uses a single lock — very poor performance under concurrency
- `ConcurrentHashMap`: Thread-safe with fine-grained locking; concurrent reads, segmented writes — much better performance

---

### Q: What is the difference between `Runnable` and `Callable`?

| | `Runnable` | `Callable<V>` |
|---|---|---|
| Return value | void | V (generic) |
| Checked exceptions | Cannot throw | Can throw |
| Use with | Thread, ExecutorService | ExecutorService only |
| Introduced | Java 1.0 | Java 5 |

---

### Q: Explain CompletableFuture vs Future.

| | `Future` | `CompletableFuture` |
|---|---|---|
| Blocking | `get()` always blocks | Non-blocking pipeline with `thenApply` |
| Chaining | Not supported | Yes — `thenApply`, `thenCompose` |
| Manual completion | No | Yes — `complete()`, `completeExceptionally()` |
| Combining | Not supported | `allOf`, `anyOf`, `thenCombine` |
| Error handling | `try/catch` on `get()` | `exceptionally`, `handle` |

---

### Q: What is a ThreadLocal?

Provides a **per-thread variable** — each thread has its own independent copy of the value.

```java
ThreadLocal<SimpleDateFormat> dateFormat = ThreadLocal.withInitial(
    () -> new SimpleDateFormat("yyyy-MM-dd")
);

// Each thread gets its own SimpleDateFormat (not thread-safe otherwise)
String formatted = dateFormat.get().format(new Date());

// Always remove in thread pool threads to prevent memory leaks
dateFormat.remove();
```

---

## Quick Reference Cheat Sheet

```
Thread states: NEW → RUNNABLE ↔ BLOCKED/WAITING/TIMED_WAITING → TERMINATED

synchronized  → intrinsic lock on object; visibility + atomicity
volatile      → visibility only; no atomicity for compound ops
AtomicInteger → lock-free CAS; atomic compound ops (incrementAndGet)

Thread creation:
  extends Thread         → simple, inflexible
  implements Runnable    → preferred (no return)
  implements Callable    → use with ExecutorService (has return + exception)

Thread Pools:
  newFixedThreadPool(n)  → fixed n threads, bounded
  newCachedThreadPool()  → unbounded (use carefully)
  new ThreadPoolExecutor(core, max, ttl, queue, rejectPolicy)

CompletableFuture:
  supplyAsync  → start async
  thenApply    → transform (sync)
  thenCompose  → chain async (flatMap)
  thenCombine  → combine two futures
  allOf / anyOf → wait for all / first

Locks:
  ReentrantLock          → explicit lock, tryLock, interruptible
  ReentrantReadWriteLock → multiple readers OR one writer
  synchronized           → simpler, good for most cases

Deadlock prevention:
  1. Consistent lock ordering
  2. tryLock with timeout
  3. Avoid nested locks

Collections:
  ConcurrentHashMap → thread-safe map, fine-grained locking
  CopyOnWriteArrayList → read-heavy lists
  BlockingQueue → producer-consumer (put/take block)

wait() vs sleep():
  wait() → releases lock, needs notify() to wake
  sleep() → holds lock, wakes after timeout
```

---

*Last Updated: 2026-06-04*

# Java I/O and NIO Study Guide

## Overview

Almost every backend application reads and writes data: log files, config files, uploaded documents, network responses, CSV imports, and more. Java gives you two toolkits:

- **Classic I/O (`java.io`)** — the original streams-based API (since Java 1.0). Simple, blocking, stream-oriented.
- **NIO / NIO.2 (`java.nio`)** — the modern API (NIO since Java 1.4, NIO.2 since Java 7). Buffer-oriented, channel-based, can be non-blocking, and gives a much nicer file API (`Path`, `Files`).

For junior backend interviews you are expected to: read/write files correctly, use try-with-resources so nothing leaks, know why buffering matters, always specify a charset (UTF-8), and explain the difference between blocking classic I/O and non-blocking NIO at a high level.

---

## Table of Contents

1. [What Is a Stream?](#what-is-a-stream)
2. [Byte Streams vs Character Streams](#byte-streams-vs-character-streams)
3. [Buffered Streams & Why Buffering Matters](#buffered-streams--why-buffering-matters)
4. [Reading & Writing Files the Classic Way](#reading--writing-files-the-classic-way)
5. [try-with-resources (Always Close Your Streams)](#try-with-resources-always-close-your-streams)
6. [The Decorator Pattern in java.io](#the-decorator-pattern-in-javaio)
7. [Modern File I/O with NIO.2 (Path, Paths, Files)](#modern-file-io-with-nio2-path-paths-files)
8. [Reading Large Files: Files.lines() vs readAllLines()](#reading-large-files-fileslines-vs-readalllines)
9. [Classic I/O vs NIO: Buffers, Channels, Selectors](#classic-io-vs-nio-buffers-channels-selectors)
10. [Character Encoding / Charset Gotcha](#character-encoding--charset-gotcha)
11. [Serialization (Brief Mention)](#serialization-brief-mention)
12. [Common Mistakes](#common-mistakes)
13. [Common Interview Questions](#common-interview-questions)
14. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What Is a Stream?

A **stream** in `java.io` is a sequence of data flowing from a **source** to a **destination**, one piece at a time. It is NOT the same as a Java 8 `Stream` (`java.util.stream`) — that is a totally different thing.

- **Input stream** = data flows INTO your program (reading).
- **Output stream** = data flows OUT of your program (writing).

```
READING (input):    File/Network  ──stream──►  Your Program
WRITING (output):   Your Program  ──stream──►  File/Network
```

Streams are **sequential** (start to end) and **one-directional** (reads OR writes, not both).

---

## Byte Streams vs Character Streams

Java has **two parallel families** of streams.

| Family | Base Classes | Works With | Use For |
|---|---|---|---|
| **Byte streams** | `InputStream` / `OutputStream` | Raw bytes (8-bit) | Images, PDFs, ZIPs, any binary file |
| **Character streams** | `Reader` / `Writer` | Characters (charset-aware) | Text files, CSV, JSON, logs |

Under the hood, a file is always bytes. Character streams add a **charset** that groups bytes into characters — e.g., "é" is 2 bytes in UTF-8; a character stream handles that for you.

> **Rule of thumb**: Text → `Reader`/`Writer`. Binary → `InputStream`/`OutputStream`.

### Byte stream example (binary copy)

```java
try (InputStream in = new FileInputStream("photo.jpg");
     OutputStream out = new FileOutputStream("copy.jpg")) {
    byte[] buffer = new byte[8192];
    int bytesRead;
    while ((bytesRead = in.read(buffer)) != -1) {
        out.write(buffer, 0, bytesRead);  // write only the bytes actually read
    }
}
```

### Character stream example (reading text)

```java
try (Reader reader = new FileReader("notes.txt", StandardCharsets.UTF_8)) {
    int ch;
    while ((ch = reader.read()) != -1) {
        System.out.print((char) ch);
    }
}
```

### The bridge: InputStreamReader / OutputStreamWriter

When you have a byte stream (e.g., a network socket) but the data is text:

```java
InputStream byteStream = socket.getInputStream();
Reader charStream = new InputStreamReader(byteStream, StandardCharsets.UTF_8);
```

---

## Buffered Streams & Why Buffering Matters

Reading one byte at a time from disk is slow — each `read()` may trigger a separate OS call. A **buffered stream** reads a large chunk (e.g., 8 KB) into memory at once and serves your small `read()` calls from there.

Think of it like grocery shopping: without a buffer, you drive to the store for every single egg. With a buffer, you buy a whole carton once.

```java
// Wrap in BufferedReader for speed + readLine() convenience
try (BufferedReader br = new BufferedReader(new FileReader("big.txt", StandardCharsets.UTF_8))) {
    String line;
    while ((line = br.readLine()) != null) {  // readLine() only exists on BufferedReader
        System.out.println(line);
    }
}
```

For output, `BufferedWriter` collects writes and flushes in batches. `close()` automatically flushes; call `flush()` manually if you need data on disk sooner.

> **Interview point**: Buffering reduces system calls. The two key extras `BufferedReader` adds are `readLine()` and large-chunk reads.

---

## Reading & Writing Files the Classic Way

### Writing a text file

```java
try (BufferedWriter writer = new BufferedWriter(
        new FileWriter("report.txt", StandardCharsets.UTF_8))) {
    writer.write("Order summary");
    writer.newLine();           // portable line break
    writer.write("Total: $42.00");
}
```

### Reading a text file line-by-line

```java
try (BufferedReader reader = new BufferedReader(
        new FileReader("report.txt", StandardCharsets.UTF_8))) {
    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line);
    }
}
```

### Appending to an existing file

```java
// 'true' = append mode (do not overwrite)
try (FileWriter fw = new FileWriter("audit.log", StandardCharsets.UTF_8, true)) {
    fw.write("2026-06-11 user logged in\n");
}
```

> Note: `FileReader`/`FileWriter` charset constructors require **Java 11+**. On older Java use `new InputStreamReader(new FileInputStream(file), StandardCharsets.UTF_8)`.

---

## try-with-resources (Always Close Your Streams)

Streams hold OS file handles. Not closing them leaks handles — eventually the OS crashes your app with "Too many open files." Buffered data may also never flush.

### The modern way (Java 7+)

```java
try (BufferedReader br = new BufferedReader(new FileReader("data.txt", StandardCharsets.UTF_8))) {
    String line = br.readLine();
    System.out.println(line);
}   // br.close() called automatically — even if an exception was thrown
```

Multiple resources are closed in **reverse order** of declaration:

```java
try (InputStream in = new FileInputStream("a.bin");       // closed SECOND
     OutputStream out = new FileOutputStream("b.bin")) {  // closed FIRST
    in.transferTo(out);   // Java 9+: copies all bytes in one call
}
```

> **Interview tip**: Any class implementing `AutoCloseable` (all I/O streams do) works in try-with-resources. This is the single most important I/O habit.

---

## The Decorator Pattern in java.io

`java.io` is the textbook example of the **Decorator design pattern**: wrap a basic stream in another stream that adds one feature, and stack as many wrappers as you need.

```java
// Read a GZIP-compressed UTF-8 text file, line by line, buffered — all composed:
BufferedReader br = new BufferedReader(
    new InputStreamReader(
        new BufferedInputStream(
            new GZIPInputStream(
                new FileInputStream("data.gz"))),
        StandardCharsets.UTF_8));
```

> A stream whose **constructor takes another stream** is a decorator (e.g., `new BufferedReader(reader)`, `new GZIPInputStream(in)`). Each layer adds exactly one capability.

---

## Modern File I/O with NIO.2 (Path, Paths, Files)

Since **Java 7**, `java.nio.file` gives a cleaner, more powerful file API. Prefer this over `File` and raw streams in new code.

| Type | Role |
|---|---|
| `Path` | Represents a location (doesn't open it) |
| `Path.of` / `Paths.get` | Factory to create `Path` objects |
| `Files` | Static utility that does the work (read, write, copy, delete) |

### Creating a Path

```java
Path path = Path.of("data", "reports", "june.txt");  // portable path separator
```

### Common `Files` operations

```java
Path file = Path.of("notes.txt");

// Reading
String content  = Files.readString(file, StandardCharsets.UTF_8);       // whole file → String (Java 11+)
List<String> lines = Files.readAllLines(file, StandardCharsets.UTF_8);  // whole file → List<String>
byte[] bytes    = Files.readAllBytes(file);                              // whole file → byte[]

// Writing
Files.writeString(file, "hello world", StandardCharsets.UTF_8);          // overwrite
Files.writeString(file, "extra\n", StandardCharsets.UTF_8, StandardOpenOption.APPEND);

// Copy / Move / Delete
Files.copy(Path.of("a.txt"), Path.of("b.txt"), StandardCopyOption.REPLACE_EXISTING);
Files.move(Path.of("b.txt"), Path.of("c.txt"), StandardCopyOption.REPLACE_EXISTING);
Files.delete(Path.of("c.txt"));          // throws if not found
Files.deleteIfExists(Path.of("c.txt"));  // no exception if missing

// Existence / Metadata
boolean exists = Files.exists(file);
long size      = Files.size(file);
Files.createDirectories(Path.of("a", "b", "c"));  // creates all missing parents
```

### Walking a directory tree

```java
try (Stream<Path> paths = Files.walk(Path.of("project"))) {  // must be closed — holds a handle
    paths.filter(Files::isRegularFile)
         .filter(p -> p.toString().endsWith(".java"))
         .forEach(System.out::println);
}
```

> **Why NIO.2 over `File`?** `File.delete()` returned a boolean (easy to ignore failures). `Files` throws descriptive exceptions, supports atomic moves, symbolic links, and directory walking.

---

## Reading Large Files: Files.lines() vs readAllLines()

| Method | Loads | Memory | Use When |
|---|---|---|---|
| `Files.readAllLines()` | Whole file into a `List` | High (entire file) | File is small; you need all lines at once |
| `Files.lines()` | One line at a time (lazy `Stream`) | Low (constant) | File could be large; process sequentially |
| `BufferedReader.readLine()` | One line at a time | Low (constant) | Classic-I/O style or fine control |

```java
// DANGER: 5 GB file → OutOfMemoryError
List<String> lines = Files.readAllLines(Path.of("huge.log"), StandardCharsets.UTF_8);

// SAFE: constant memory regardless of file size
try (Stream<String> lines = Files.lines(Path.of("huge.log"), StandardCharsets.UTF_8)) {
    lines.filter(l -> l.contains("ERROR"))
         .forEach(this::handleError);
}
```

> **Key gotcha**: `Files.lines()` holds an **open file handle** — always use try-with-resources. `readAllLines()` closes the file before returning, so no try-with-resources needed for the stream itself.

---

## Classic I/O vs NIO: Buffers, Channels, Selectors

- **Classic I/O (`java.io`)** is **stream-oriented** and **blocking** — thread waits for data, one thread per connection.
- **NIO (`java.nio`)** is **buffer-oriented** and can be **non-blocking** — data flows through **channels** into **buffers**, one thread can manage many connections via a **selector**.

Think of it like a restaurant: classic I/O is one waiter per table (the waiter stands idle while you decide). NIO is one waiter for many tables — they check each table and only serve the ones that are ready.

### Comparison table

| Aspect | Classic I/O | NIO |
|---|---|---|
| Orientation | Stream-oriented | Buffer-oriented |
| Blocking | Always blocking | Can be non-blocking |
| Core abstractions | `InputStream`, `OutputStream`, `Reader`, `Writer` | `Channel`, `Buffer`, `Selector` |
| Direction | One-directional | Channels are bidirectional |
| Threads for many connections | One per connection | One thread can handle many |
| Best for | Simple file I/O, small/medium workloads | High-concurrency servers |

### Buffer basics

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);

channel.read(buffer);   // channel writes INTO the buffer

buffer.flip();          // switch: write mode → read mode (sets limit=position, position=0)

while (buffer.hasRemaining()) {
    byte b = buffer.get();
}

buffer.clear();         // reset to fill again
```

> **The classic NIO bug**: forgetting `flip()` before reading. You must call it to switch from "filling the buffer" to "consuming the buffer."

### Channel basics

```java
try (FileChannel channel = FileChannel.open(Path.of("data.txt"), StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    while (channel.read(buffer) != -1) {
        buffer.flip();
        // ... consume buffer ...
        buffer.clear();
    }
}
```

### Selector concept (non-blocking)

A `Selector` lets **one thread monitor many channels** and react only to the ones that are ready. This is the foundation of high-performance servers (Netty, Spring WebFlux).

```
                ┌──────────┐
                │ Selector │  (one thread: "which channels are ready?")
                └────┬─────┘
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   Channel A     Channel B     Channel C
   (ready)       (waiting)     (ready)
   → handle A                  → handle C
```

> **Junior takeaway**: You rarely write raw `Selector` code — frameworks do it. But you should explain: NIO allows **non-blocking I/O** where **one thread serves many connections** via **channels**, **buffers**, and a **selector**, which scales far better than blocking one-thread-per-connection I/O.

---

## Character Encoding / Charset Gotcha

A **charset** maps characters to bytes (e.g., UTF-8, Windows-1252). If you write with one charset and read with another, you get garbled text ("mojibake": `Ã©` instead of `é`).

### The trap

```java
// BAD — uses platform default charset, which varies per machine
Reader r = new FileReader("data.txt");
byte[] b = "héllo".getBytes();
```

### The fix

```java
// GOOD — explicit UTF-8, same behavior everywhere
Reader r = new FileReader("data.txt", StandardCharsets.UTF_8);
byte[] b = "héllo".getBytes(StandardCharsets.UTF_8);
List<String> lines = Files.readAllLines(Path.of("data.txt"), StandardCharsets.UTF_8);
```

> **Golden rule**: Always pass `StandardCharsets.UTF_8`. Use the `StandardCharsets` constants (compile-time safe), not the string `"UTF-8"` (which can throw `UnsupportedEncodingException`). Java 18+ made UTF-8 the default (JEP 400), but don't rely on it.

---

## Serialization (Brief Mention)

**Serialization** converts a Java object into bytes (for saving to a file or sending over a network). **Deserialization** rebuilds it.

```java
// Serialize
try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("user.ser"))) {
    out.writeObject(user);   // user must implement Serializable
}

// Deserialize
try (ObjectInputStream in = new ObjectInputStream(new FileInputStream("user.ser"))) {
    User user = (User) in.readObject();
}
```

For `serialVersionUID`, `transient` fields, and security risks of deserializing untrusted data, see the dedicated **Serialization** material in `Java_Developer_Interview_Questions.md`. For this guide, just know it's another use of byte streams and modern apps usually prefer JSON (Jackson) over Java's built-in serialization.

---

## Common Mistakes

| Mistake | Why It's Bad | Fix |
|---|---|---|
| Not closing streams | Leaks OS file handles → crash | Use try-with-resources |
| Relying on default charset | Corrupts non-ASCII text on different machines | Always pass `StandardCharsets.UTF_8` |
| `Files.readAllLines()` on a huge file | Loads entire file → `OutOfMemoryError` | Use `Files.lines()` |
| Not wrapping `Files.lines()` / `Files.walk()` in try-with-resources | Leaks open file handle | Always try-with-resources |
| No buffering for many small reads/writes | Slow — one disk trip per tiny operation | Wrap in `BufferedReader`/`BufferedWriter` |
| Using character streams for binary data | Charset decoding corrupts bytes | Use byte streams for binary |
| Forgetting `flip()` in NIO | Buffer reads garbage / nothing | Call `buffer.flip()` before reading |
| Writing whole buffer instead of bytes read | Corrupts data with stale bytes | Use `out.write(buffer, 0, bytesRead)` |

---

## Common Interview Questions

### Q: What is the difference between byte streams and character streams?

Byte streams (`InputStream`/`OutputStream`) handle raw 8-bit bytes for binary data (images, ZIPs). Character streams (`Reader`/`Writer`) are charset-aware and handle text. A file is always bytes underneath; character streams decode those bytes into characters using a charset. Rule: text → character streams, binary → byte streams.

---

### Q: Why use a BufferedReader instead of a plain FileReader?

Two reasons: **performance** and **convenience**. `BufferedReader` reads large chunks into memory, reducing slow disk/system calls. It also adds `readLine()`, which doesn't exist on plain `Reader`.

---

### Q: What does try-with-resources do and why is it important for I/O?

It automatically calls `close()` on any `AutoCloseable` resource when the block exits — normally or via exception. This prevents file-handle leaks and ensures buffered data is flushed. It replaced the verbose and error-prone `finally { stream.close(); }` pattern.

---

### Q: What is the decorator pattern and how does java.io use it?

The decorator pattern wraps an object to add behavior without subclassing. `java.io` uses it heavily: `new BufferedReader(new InputStreamReader(new FileInputStream(file), UTF_8))` stacks file reading + charset decoding + buffering. You recognize a decorator because its constructor takes another stream as an argument.

---

### Q: What is the difference between classic I/O and NIO?

Classic I/O is **stream-oriented** and **blocking** — one thread per connection. NIO is **buffer-oriented** and can be **non-blocking** — data flows through **channels** into **buffers**, and one thread manages many connections via a **selector**. Classic I/O is simpler for files; NIO scales for high-concurrency servers.

---

### Q: What is a Channel, a Buffer, and a Selector in NIO?

- **Buffer**: fixed-size memory block for reading/writing data. Call `flip()` to switch from fill mode to consume mode.
- **Channel**: two-way connection to a file or socket that always works through a buffer (e.g., `FileChannel`, `SocketChannel`).
- **Selector**: lets one thread monitor many channels and act only on ready ones, enabling one thread to serve thousands of connections.

---

### Q: What is the difference between Files.lines() and Files.readAllLines()?

`readAllLines()` loads the **entire file** into a `List<String>` — fine for small files, `OutOfMemoryError` on large ones. `Files.lines()` returns a **lazy Stream** reading one line at a time, keeping memory constant. `Files.lines()` holds an open file handle so it must be closed with try-with-resources.

---

### Q: Why should you always specify a charset when reading/writing text?

The platform default charset varies between machines. Code that works on your laptop can corrupt non-ASCII characters in production. Always pass `StandardCharsets.UTF_8` explicitly and use the `StandardCharsets` constants rather than the `"UTF-8"` string.

---

### Q: What is the difference between Path and File?

`File` (old `java.io`) has clumsy error handling — `delete()` returns a boolean, easy to ignore. `Path` (Java 7+, `java.nio.file`) paired with the `Files` utility throws descriptive exceptions, supports atomic moves, symbolic links, and directory walking. Prefer `Path` + `Files` in new code.

---

### Q: How do you copy a file in Java?

Modern way: `Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING)`. Classic way: open `FileInputStream` and `FileOutputStream` in try-with-resources, loop reading into a `byte[]`, write `out.write(buffer, 0, bytesRead)` until `read()` returns -1. Java 9+ also has `in.transferTo(out)`.

---

### Q: When would you use a byte stream vs a character stream for a network socket carrying text?

A socket gives you raw bytes (`InputStream`). Since the payload is text, bridge it to a character stream using `InputStreamReader` with an explicit charset (UTF-8), typically wrapped in a `BufferedReader`. For binary payloads, keep working with the byte stream directly.

---

### Q: Is the java.io Stream the same as a Java 8 Stream?

No — completely different. A `java.io` stream (`InputStream`/`Reader`) is an I/O channel for bytes or characters. A `java.util.stream.Stream` (Java 8) is a pipeline for processing collections with `map`, `filter`, `collect`. The word "stream" is all they share. Note: `Files.lines()` returns a Java 8 `Stream<String>`.

---

## Quick Reference Cheat Sheet

```
TWO STREAM FAMILIES
  Byte streams      → InputStream / OutputStream   → binary (images, PDFs, ZIPs)
  Character streams → Reader / Writer              → text (charset-aware)
  Bridge bytes→chars → InputStreamReader / OutputStreamWriter (pass UTF-8!)

BUFFERING (wrap for speed + readLine())
  BufferedReader / BufferedWriter          (character)
  BufferedInputStream / BufferedOutputStream (byte)
  Why: batches I/O → fewer slow disk/system calls

CLASSIC FILE READ (text, line by line)
  try (BufferedReader br = new BufferedReader(
          new FileReader("f.txt", StandardCharsets.UTF_8))) {
      String line;
      while ((line = br.readLine()) != null) { ... }
  }

try-with-resources
  → auto-closes any AutoCloseable on block exit (normal OR exception)
  → multiple resources closed in REVERSE order
  → THE habit that prevents file-handle leaks

DECORATOR PATTERN
  Wrap streams to stack features:
  new BufferedReader(new InputStreamReader(new FileInputStream(f), UTF_8))
  A stream whose constructor takes another stream = a decorator.

NIO.2 (java.nio.file) — prefer for everyday file work
  Path   = the address (location only)
  Path.of / Paths.get = make a Path
  Files  = the courier (does the work)
    Files.readString / readAllLines / readAllBytes
    Files.writeString / write
    Files.copy / move / delete / deleteIfExists
    Files.exists / isDirectory / size / createDirectories
    Files.walk(path)   → recursive Stream<Path> (close it!)

LARGE FILES
  readAllLines() → whole file in RAM (small files only)
  Files.lines()  → lazy Stream, constant memory (any size; close it!)

CLASSIC I/O vs NIO
  Classic I/O → stream-oriented, BLOCKING, 1 thread/connection, simple
  NIO         → buffer-oriented, NON-BLOCKING option, channels+buffers+selector,
                1 thread serves many connections, scales for servers
  Buffer   → memory block; call flip() to switch write→read mode
  Channel  → two-way connection (file/socket), works through a buffer
  Selector → 1 thread monitors many channels, handles only ready ones

CHARSET GOTCHA (most common bug)
  ALWAYS pass StandardCharsets.UTF_8 — never rely on platform default.
  Default charset varies per machine → corrupts non-ASCII text (mojibake).

SERIALIZATION
  ObjectOutputStream.writeObject / ObjectInputStream.readObject
  Object must implement Serializable. See Java Developer guide for depth.

TOP MISTAKES
  ✗ not closing streams        → leaks; use try-with-resources
  ✗ default charset            → corruption; use UTF_8
  ✗ readAllLines on huge file  → OOM; use Files.lines()
  ✗ no buffering               → slow; wrap in Buffered*
  ✗ char streams for binary    → corruption; use byte streams
  ✗ forgetting buffer.flip()   → NIO reads garbage
```

---

*Last Updated: 2026-06-18*

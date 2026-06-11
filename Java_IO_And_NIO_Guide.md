# Java I/O and NIO Study Guide

## Overview

Almost every backend application reads and writes data: log files, config files, uploaded documents, network responses, CSV imports, and more. Java gives you two big toolkits for this:

- **Classic I/O (`java.io`)** — the original streams-based API (since Java 1.0). Simple, blocking, stream-oriented.
- **NIO / NIO.2 (`java.nio`)** — the modern API (NIO since Java 1.4, NIO.2 since Java 7). Buffer-oriented, channel-based, can be non-blocking, and gives you a much nicer file API (`Path`, `Files`).

For junior backend interviews you are mostly expected to: read/write files correctly, use try-with-resources so nothing leaks, know why buffering matters, always specify a charset (UTF-8), and explain the difference between blocking classic I/O and non-blocking NIO at a high level.

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

A **stream** in `java.io` is a sequence of data flowing from a **source** to a **destination**, one piece at a time. It is NOT the same as a Java 8 `Stream` (from `java.util.stream`) — that is a totally different thing. Here we mean an **I/O stream**.

**Think of it like a garden hose:** water flows through the hose one drop at a time from a tap (the source) to your garden (the destination). You don't get all the water at once — it streams through continuously. An I/O stream is the same: bytes (or characters) flow through it one chunk at a time.

There are two directions:

- **Input stream** = data flows INTO your program (reading). Example: reading a file's contents.
- **Output stream** = data flows OUT of your program (writing). Example: writing a file.

```
READING (input):    File/Network  ──stream──►  Your Program
WRITING (output):   Your Program  ──stream──►  File/Network
```

The key property of a stream is that it is **sequential**: you read/write from start to end, you don't randomly jump around (for random access you'd use `RandomAccessFile` or NIO channels). Streams are also **one-directional** — an input stream only reads, an output stream only writes.

---

## Byte Streams vs Character Streams

This is the first big fork in `java.io`. Java has **two parallel families** of streams.

| Family | Base Classes | Works With | Use For |
|---|---|---|---|
| **Byte streams** | `InputStream` / `OutputStream` | Raw bytes (8-bit) | Images, PDFs, ZIPs, audio, any binary file |
| **Character streams** | `Reader` / `Writer` | Characters (text, charset-aware) | Text files, CSV, JSON, logs, config |

**Think of it like moving house:**
- **Byte streams** are like moving raw boxes — you don't care what's inside, you just move the physical box (bytes) exactly as-is. Perfect for a photo or a ZIP file where every byte must stay identical.
- **Character streams** are like a translator who reads the labels — they understand text and encoding (e.g., "this is UTF-8 text"), so they correctly turn bytes into readable characters and back.

### Byte stream example (binary-safe copy)

```java
// Copy a binary file (image) byte-for-byte. We use BYTE streams because images are binary.
try (InputStream in = new FileInputStream("photo.jpg");      // reads raw bytes from the file
     OutputStream out = new FileOutputStream("copy.jpg")) {  // writes raw bytes to the new file

    byte[] buffer = new byte[8192];   // a small in-memory chunk (8 KB) we reuse for each read
    int bytesRead;                    // how many bytes were actually read this round
    while ((bytesRead = in.read(buffer)) != -1) {  // read() returns -1 when the file is fully consumed
        out.write(buffer, 0, bytesRead);           // write ONLY the bytes we actually read (not the whole array)
    }
}
```

### Character stream example (reading text)

```java
// Read a TEXT file. We use CHARACTER streams because the file is human-readable text.
try (Reader reader = new FileReader("notes.txt", StandardCharsets.UTF_8)) {  // decodes bytes -> chars using UTF-8
    int ch;                              // one character at a time (as an int code)
    while ((ch = reader.read()) != -1) { // read() returns -1 at end of file
        System.out.print((char) ch);     // cast the int code back to a printable char
    }
}
```

### Why the difference matters

Under the hood, a file on disk is ALWAYS just bytes. The difference is **interpretation**:

- A **byte stream** gives you those bytes raw. If you read a UTF-8 text file with a byte stream, the letter "é" (which is 2 bytes in UTF-8) comes through as 2 separate bytes — you'd have to decode it yourself.
- A **character stream** has a **charset** built in. It groups those bytes correctly and hands you the actual character "é". This is why text should use `Reader`/`Writer`.

> **Rule of thumb**: Text data → character streams (`Reader`/`Writer`). Binary data (anything not meant to be read by humans) → byte streams (`InputStream`/`OutputStream`).

### The bridge: InputStreamReader / OutputStreamWriter

Sometimes you have a byte stream (e.g., from a network socket) but the data is text. The "bridge" classes convert between the two:

```java
// InputStreamReader bridges a byte stream INTO a character stream, applying a charset.
InputStream byteStream = socket.getInputStream();             // we have raw bytes from the network
Reader charStream = new InputStreamReader(byteStream, StandardCharsets.UTF_8); // now we read characters
// Always pass the charset explicitly! (see the Charset Gotcha section)
```

---

## Buffered Streams & Why Buffering Matters

Reading one byte (or one character) at a time directly from disk is **slow** because each `read()` may trigger a separate trip to the operating system / hardware. Buffering fixes this.

**Think of it like grocery shopping:** Without a buffer, every time you need one egg you drive to the store, buy one egg, and drive home — then repeat for the next egg. With a buffer, you drive once, buy a whole carton (a chunk), and take eggs from the carton at home. Far fewer trips.

A **buffered stream** reads a big chunk (e.g., 8 KB) from disk into an in-memory array ONCE, then serves your small `read()` calls from that array. It only goes back to disk when the chunk is empty.

```java
// SLOW — every readLine ultimately pokes the disk character by character without a buffer.
Reader raw = new FileReader("big.txt", StandardCharsets.UTF_8);

// FAST — wrap it in a BufferedReader. Now reads happen in big chunks behind the scenes.
try (BufferedReader br = new BufferedReader(new FileReader("big.txt", StandardCharsets.UTF_8))) {
    String line;                         // holds one line of text at a time
    while ((line = br.readLine()) != null) {  // readLine() reads a whole line; returns null at end of file
        System.out.println(line);        // process the line
    }
    // readLine() is a HUGE convenience — it only exists on BufferedReader, not on plain Reader.
}
```

For output, `BufferedWriter` / `BufferedOutputStream` collect your small writes and flush them to disk in big batches:

```java
try (BufferedWriter bw = new BufferedWriter(new FileWriter("out.txt", StandardCharsets.UTF_8))) {
    bw.write("Hello");   // goes into the in-memory buffer, NOT to disk yet
    bw.newLine();        // writes a platform-correct line separator (\n or \r\n)
    bw.write("World");   // still buffered
    // When the try block ends, close() automatically flushes the buffer to disk.
    // If you need data on disk sooner, call bw.flush() manually.
}
```

> **Interview point**: Buffering reduces the number of system calls / disk operations. The data is identical; you just batch the I/O. This can make file processing many times faster. The two extra features `BufferedReader` gives you are `readLine()` and large-chunk reads.

---

## Reading & Writing Files the Classic Way

Here is the "classic `java.io`" toolkit you'll use most for text files.

### Writing a text file

```java
// FileWriter writes characters. true = append mode; omit (or false) = overwrite.
try (BufferedWriter writer = new BufferedWriter(
        new FileWriter("report.txt", StandardCharsets.UTF_8))) {  // create/overwrite report.txt as UTF-8
    writer.write("Order summary");   // write a line of text into the buffer
    writer.newLine();                // line break (portable across OSes)
    writer.write("Total: $42.00");   // another line
}   // try-with-resources closes + flushes automatically
```

### Reading a text file line-by-line

```java
try (BufferedReader reader = new BufferedReader(
        new FileReader("report.txt", StandardCharsets.UTF_8))) {  // open report.txt as UTF-8
    String line;
    while ((line = reader.readLine()) != null) {  // read until end of file (null)
        System.out.println(line);                 // do something with each line
    }
}
```

### Appending to an existing file

```java
// The 'true' flag tells FileWriter to APPEND instead of overwriting the file.
try (FileWriter fw = new FileWriter("audit.log", StandardCharsets.UTF_8, true)) {
    fw.write("2026-06-11 user logged in\n");  // new content is added to the END of the file
}
```

> Note: `FileReader`/`FileWriter` only gained the charset-accepting constructors in **Java 11**. On older Java you must use `new InputStreamReader(new FileInputStream(file), StandardCharsets.UTF_8)` to control the charset. For NIO.2 (`Files`), charset has always been a parameter.

---

## try-with-resources (Always Close Your Streams)

Streams hold operating-system resources (file handles). If you don't close them, you **leak** — eventually the OS runs out of file handles and your app crashes with "Too many open files." You also risk losing buffered data that was never flushed.

**Think of it like a library book:** if you borrow a book (open a stream) and never return it (close it), eventually the library runs out of copies for everyone else. try-with-resources is an automatic "return it the moment you're done" system.

### The old, error-prone way (don't do this)

```java
BufferedReader br = null;
try {
    br = new BufferedReader(new FileReader("data.txt", StandardCharsets.UTF_8));
    // ... use br ...
} finally {
    if (br != null) br.close();  // easy to forget; close() itself can throw; verbose
}
```

### The modern way: try-with-resources (Java 7+)

```java
// Any resource declared in the parentheses is AUTO-CLOSED when the block exits,
// whether it exits normally OR by throwing an exception.
try (BufferedReader br = new BufferedReader(new FileReader("data.txt", StandardCharsets.UTF_8))) {
    String line = br.readLine();  // use the resource
    System.out.println(line);
}   // br.close() is called automatically right here — even if readLine() threw
```

You can declare **multiple** resources; they are closed in **reverse order** of declaration:

```java
try (InputStream in = new FileInputStream("a.bin");      // closed SECOND
     OutputStream out = new FileOutputStream("b.bin")) {  // closed FIRST
    in.transferTo(out);   // Java 9+: copies all bytes from in to out in one call
}
```

> **Interview tip**: Any object that implements `AutoCloseable` (which all I/O streams do) can go inside try-with-resources. This is the single most important habit for I/O code — it guarantees no resource leaks.

---

## The Decorator Pattern in java.io

`java.io` is the textbook real-world example of the **Decorator design pattern**. You "wrap" a basic stream in another stream that adds a feature, and you can keep wrapping to stack features.

**Think of it like getting dressed in layers:** your body is the base (raw stream). You add a shirt (buffering), then a jacket (a specific data format), then a raincoat (compression). Each layer adds a capability but the thing underneath still does its core job.

```java
// Start with a raw byte stream from a file (the base "component")
InputStream fileBytes = new FileInputStream("data.gz");

// Decorator 1: decompress GZIP on the fly
InputStream unzipped = new GZIPInputStream(fileBytes);

// Decorator 2: add buffering for speed
InputStream buffered = new BufferedInputStream(unzipped);

// Decorator 3: bridge bytes -> characters with a charset
Reader reader = new InputStreamReader(buffered, StandardCharsets.UTF_8);

// Decorator 4: add readLine() convenience
BufferedReader br = new BufferedReader(reader);
// Now: read a GZIP-compressed UTF-8 text file, line by line, buffered — all from composed layers!
```

**Why this design?** Instead of having one giant class with every possible combination (BufferedGzipUtf8FileReader, etc.), Java gives you small single-purpose classes you compose like LEGO. Each wrapper takes another stream in its constructor and adds exactly one capability.

> You will recognize the pattern by the constructors: a stream class that **takes another stream as a constructor argument** is a decorator (e.g., `new BufferedReader(reader)`, `new GZIPInputStream(in)`).

---

## Modern File I/O with NIO.2 (Path, Paths, Files)

Since **Java 7**, the `java.nio.file` package gives a much cleaner, more powerful file API. For everyday file work in modern code, **prefer this over `File` and raw streams**.

The three key players:

| Type | Role | Analogy |
|---|---|---|
| `Path` | Represents a file/directory location (does not open it) | A street address written on paper |
| `Paths` / `Path.of` | Factory to create `Path` objects | The person who writes the address |
| `Files` | Static utility methods that DO things (read, write, copy, delete) | The delivery driver who actually visits the address |

**Think of it like ordering a package:** a `Path` is just the address (it doesn't move anything). `Files` is the courier that actually goes to that address and reads, writes, copies, or deletes.

### Creating a Path

```java
Path path = Path.of("data", "reports", "june.txt");   // builds "data/reports/june.txt" portably
// Path.of (Java 11+) is the modern way. Older code uses Paths.get("data", "reports", "june.txt").
// Note: it joins with the correct OS separator (/ on Linux, \ on Windows) for you.
```

### Common `Files` operations

```java
Path file = Path.of("notes.txt");

// --- READING ---
String content = Files.readString(file, StandardCharsets.UTF_8);   // entire file -> one String (Java 11+)
List<String> lines = Files.readAllLines(file, StandardCharsets.UTF_8); // entire file -> List<String> (one per line)
byte[] bytes = Files.readAllBytes(file);                            // entire file -> byte[] (binary)

// --- WRITING ---
Files.writeString(file, "hello world", StandardCharsets.UTF_8);     // write a String (overwrites) (Java 11+)
Files.write(file, lines, StandardCharsets.UTF_8);                   // write a List<String> as lines
Files.write(file, bytes);                                           // write raw bytes

// Append instead of overwrite:
Files.writeString(file, "extra\n", StandardCharsets.UTF_8, StandardOpenOption.APPEND);

// --- COPY / MOVE / DELETE ---
Files.copy(Path.of("a.txt"), Path.of("b.txt"), StandardCopyOption.REPLACE_EXISTING); // copy, overwrite if exists
Files.move(Path.of("b.txt"), Path.of("c.txt"), StandardCopyOption.REPLACE_EXISTING); // move/rename
Files.delete(Path.of("c.txt"));        // delete; throws if the file does not exist
Files.deleteIfExists(Path.of("c.txt")); // delete only if present; no exception if it's missing

// --- EXISTENCE / METADATA ---
boolean exists = Files.exists(file);       // does it exist?
boolean isDir  = Files.isDirectory(file);  // is it a directory?
long size      = Files.size(file);         // size in bytes

// --- DIRECTORIES ---
Files.createDirectories(Path.of("a", "b", "c"));  // creates a/b/c including any missing parent folders
```

### Walking a directory tree

```java
// Files.walk returns a lazy Stream<Path> of EVERY file/folder under the start path (recursively).
try (Stream<Path> paths = Files.walk(Path.of("project"))) {   // try-with-resources: the stream holds an open handle!
    paths.filter(Files::isRegularFile)        // keep only files (skip directories)
         .filter(p -> p.toString().endsWith(".java"))  // keep only .java files
         .forEach(System.out::println);       // print each matching file path
}   // closing the stream releases the directory handle
```

> **Why NIO.2 over the old `File` class?** The old `java.io.File` had clumsy error handling (`delete()` returned a boolean instead of throwing — easy to ignore a failure), no symbolic-link support, and no charset-aware read/write helpers. `Files` throws descriptive exceptions, supports atomic moves, file attributes, directory walking, and more.

---

## Reading Large Files: Files.lines() vs readAllLines()

This is a very common interview gotcha. Both read a text file line by line — but their **memory behavior is completely different**.

**Think of it like watching a movie:** `readAllLines()` is **downloading the whole movie** before you watch — you need disk/RAM space for the entire thing. `Files.lines()` is **streaming the movie** — it pulls in a bit at a time, so you can watch a huge film without storing it all at once.

### readAllLines() — loads the ENTIRE file into memory

```java
// Returns a List<String> with EVERY line in memory at once.
List<String> lines = Files.readAllLines(Path.of("huge.log"), StandardCharsets.UTF_8);
for (String line : lines) {
    process(line);
}
// DANGER: if huge.log is 5 GB, this tries to hold ~5 GB of Strings in RAM -> OutOfMemoryError.
```

### Files.lines() — streams line by line (lazy, low memory)

```java
// Returns a lazy Stream<String>. Lines are read one at a time as you consume them.
try (Stream<String> lines = Files.lines(Path.of("huge.log"), StandardCharsets.UTF_8)) {  // MUST be closed!
    lines.filter(l -> l.contains("ERROR"))   // process each line on the fly
         .forEach(this::handleError);         // only ONE line is in memory at any moment
}
// Safe for files of ANY size — memory stays roughly constant.
```

| Method | Loads | Memory | Use When |
|---|---|---|---|
| `Files.readAllLines()` | Whole file into a `List` | High (entire file) | File is small and you need random access to all lines |
| `Files.lines()` | One line at a time (lazy `Stream`) | Low (constant) | File could be large; you process sequentially |
| `BufferedReader.readLine()` | One line at a time (manual loop) | Low (constant) | You want classic-I/O style or fine control |

> **Key gotcha**: `Files.lines()` returns a `Stream` that holds an **open file handle**, so it MUST be used inside try-with-resources. Forgetting that leaks the handle. `readAllLines()` reads everything immediately and closes the file before returning, so it needs no try-with-resources.

---

## Classic I/O vs NIO: Buffers, Channels, Selectors

This is the conceptual heart of the topic. Interviewers love the comparison.

### The mental model

- **Classic I/O (`java.io`)** is **stream-oriented** and **blocking**. You read bytes one direction at a time, and the thread **waits** (is blocked) until data is available.
- **NIO (`java.nio`)** is **buffer-oriented** and can be **non-blocking**. You read/write into reusable **buffers** through **channels**, and a single thread can manage many connections using a **selector**.

**Think of it like a restaurant:**
- **Classic I/O (blocking)** = one waiter per table. The waiter stands at your table doing nothing while you decide your order (the thread is blocked, idle, but tied up). 100 tables need 100 waiters.
- **NIO (non-blocking)** = one waiter for many tables. He checks each table: "Ready to order? No? I'll come back." He only spends effort on tables that actually need him. One waiter (thread) serves many tables (connections).

### Comparison table

| Aspect | Classic I/O (`java.io`) | NIO (`java.nio`) |
|---|---|---|
| Orientation | Stream-oriented (read/write sequentially) | Buffer-oriented (data goes into a `Buffer`) |
| Blocking | Blocking — thread waits for data | Can be non-blocking — thread isn't stuck |
| Core abstractions | `InputStream`, `OutputStream`, `Reader`, `Writer` | `Channel`, `Buffer`, `Selector` |
| Direction | One-directional (a stream reads OR writes) | Channels are bidirectional (read AND write) |
| Threads for many connections | One thread per connection | One thread can handle many connections |
| Best for | Simple file I/O, small/medium workloads | High-concurrency servers, thousands of connections |
| Ease of use | Simpler, more readable | More complex to code by hand |

### Buffer basics

A `Buffer` is a fixed-size block of memory you read into and write out of. The key idea is its **position/limit** cursor and the `flip()` operation that switches between writing-into-the-buffer and reading-out-of-it.

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);  // a 1 KB memory block to hold data

int bytesRead = channel.read(buffer);  // channel WRITES data INTO the buffer (position moves forward)

buffer.flip();   // switch modes: now we want to READ what we just wrote (sets limit=position, position=0)

while (buffer.hasRemaining()) {   // while there is still data to consume
    byte b = buffer.get();        // read one byte out of the buffer (position moves forward)
}

buffer.clear();  // reset the buffer so we can fill it again on the next read
```

> The single most common interview note about `Buffer`: you call **`flip()`** to switch from "writing into the buffer" mode to "reading from the buffer" mode. Forgetting `flip()` is the classic NIO bug.

### Channel basics

A `Channel` is a connection to something you can read from / write to (a file, a socket). Unlike a stream, a channel is **two-way** and always works **through a buffer**.

```java
// Read a file using a FileChannel + ByteBuffer (NIO style).
try (FileChannel channel = FileChannel.open(Path.of("data.txt"), StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocate(1024);   // our reusable memory block
    while (channel.read(buffer) != -1) {  // read a chunk from the file into the buffer; -1 = end of file
        buffer.flip();                    // switch to read-from-buffer mode
        // ... consume buffer ...
        buffer.clear();                   // reset for the next chunk
    }
}
```

### Selector concept (non-blocking, high level)

A `Selector` lets **one thread monitor many channels** at once and react only to the ones that are "ready" (have data to read, or are ready to write). This is the foundation of high-performance servers (Netty, the Spring WebFlux server, Node-style event loops).

```
                ┌──────────┐
                │ Selector │  (one thread asks: "which channels are ready?")
                └────┬─────┘
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   Channel A     Channel B     Channel C
   (ready)       (waiting)     (ready)
   → handle A                  → handle C   (skip B until it's ready)
```

The thread calls `selector.select()`, which returns only the channels that need attention. This is how a single thread can serve thousands of network connections without one-thread-per-connection overhead.

> **Junior-level takeaway**: You rarely write raw `Selector` code by hand — frameworks (Netty, Tomcat NIO connector, WebFlux) do it for you. But you should be able to explain: NIO allows **non-blocking** I/O where **one thread serves many connections**, using **channels**, **buffers**, and a **selector**, which scales far better than classic blocking one-thread-per-connection I/O.

---

## Character Encoding / Charset Gotcha

This is the bug that bites everyone eventually. A **charset** (character encoding) is the rulebook that maps characters to bytes and back (e.g., UTF-8, UTF-16, ISO-8859-1).

**Think of it like a secret code:** if I encode a message with one codebook (UTF-8) and you decode it with a different codebook (Windows-1252), you get garbled nonsense — the famous "mojibake" like `Ã©` instead of `é`.

### The trap: relying on the default charset

```java
// BAD — no charset specified. Uses the platform DEFAULT charset.
Reader r = new FileReader("data.txt");           // default charset (pre-Java 18: OS-dependent!)
String s = new String(bytes);                    // default charset again
byte[] b = "héllo".getBytes();                   // default charset again
```

The problem: the **default charset differs between machines**. Your laptop might default to UTF-8, but a Windows server might default to Windows-1252, and a CI box to something else. Code that works on your machine produces corrupted text in production. (Note: Java 18+ made UTF-8 the default via JEP 400, but you still cannot rely on the Java version everywhere — be explicit.)

### The fix: ALWAYS specify the charset (use UTF-8)

```java
// GOOD — explicit UTF-8 everywhere. Same behavior on every machine.
Reader r = new FileReader("data.txt", StandardCharsets.UTF_8);
String s = new String(bytes, StandardCharsets.UTF_8);
byte[] b = "héllo".getBytes(StandardCharsets.UTF_8);
List<String> lines = Files.readAllLines(Path.of("data.txt"), StandardCharsets.UTF_8);
```

> **Golden rule for interviews**: "Always specify the charset explicitly, and use `StandardCharsets.UTF_8`. Never rely on the platform default charset — it varies between environments and silently corrupts non-ASCII text." Use the `StandardCharsets` constants (compile-time safe) rather than the string `"UTF-8"` (which can throw `UnsupportedEncodingException`).

---

## Serialization (Brief Mention)

**Serialization** is converting a Java object into a stream of bytes (so it can be saved to a file or sent over a network), and **deserialization** is rebuilding the object from those bytes. In `java.io` this is done with `ObjectOutputStream` / `ObjectInputStream` and the `Serializable` marker interface.

```java
// Write an object to a file (serialize)
try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("user.ser"))) {
    out.writeObject(user);   // 'user' must implement Serializable
}

// Read it back (deserialize)
try (ObjectInputStream in = new ObjectInputStream(new FileInputStream("user.ser"))) {
    User user = (User) in.readObject();   // cast back to the original type
}
```

Serialization is a topic on its own (`serialVersionUID`, `transient` fields, security risks of deserializing untrusted data, and why modern apps usually prefer JSON via Jackson over Java's built-in serialization). For the full treatment — `serialVersionUID`, `transient`, custom `writeObject`/`readObject`, and deserialization vulnerabilities — see the dedicated **Serialization** material in the Java Developer guide (`Java_Developer_Interview_Questions.md`). For this guide, just know that it's another use of byte streams.

---

## Common Mistakes

| Mistake | Why It's Bad | Fix |
|---|---|---|
| Not closing streams | Leaks OS file handles → "Too many open files" crash; buffered data may never flush | Use try-with-resources |
| Relying on default charset | Corrupts non-ASCII text on different machines | Always pass `StandardCharsets.UTF_8` |
| `Files.readAllLines()` on a huge file | Loads entire file into RAM → `OutOfMemoryError` | Use `Files.lines()` (streamed) |
| Not wrapping `Files.lines()` / `Files.walk()` in try-with-resources | These streams hold an open file handle → leak | Always try-with-resources |
| No buffering for many small reads/writes | Slow — one disk trip per tiny read | Wrap in `BufferedReader`/`BufferedWriter` |
| Using character streams for binary data | Charset decoding corrupts binary bytes | Use byte streams (`InputStream`/`OutputStream`) for binary |
| Forgetting `flip()` in NIO | Buffer position is wrong → reads garbage / nothing | Call `buffer.flip()` before reading the buffer |
| Ignoring the return of `in.read(buffer)` | Writing the whole array instead of bytes actually read corrupts data | Use `out.write(buffer, 0, bytesRead)` |
| Forgetting `flush()` before reading back in same run | Buffered data not yet on disk | Call `flush()` or let close() handle it |

---

## Common Interview Questions

### Q: What is the difference between byte streams and character streams?

Byte streams (`InputStream`/`OutputStream`) handle raw 8-bit bytes and are used for binary data (images, PDFs, ZIPs). Character streams (`Reader`/`Writer`) handle characters and are charset-aware, used for text. Internally a file is always bytes; character streams add a charset to correctly decode bytes into characters and back. Rule: text → character streams, binary → byte streams.

---

### Q: Why use a BufferedReader instead of a plain FileReader?

Two reasons: **performance** and **convenience**. A `BufferedReader` reads a large chunk into memory at once, drastically reducing the number of slow disk/system calls compared to reading char-by-char. It also adds the `readLine()` method, which makes line-by-line reading trivial. You wrap a `FileReader` inside a `BufferedReader`.

---

### Q: What does try-with-resources do and why is it important for I/O?

It automatically calls `close()` on any resource declared in its parentheses when the block exits — normally or via an exception. This guarantees streams are closed, preventing file-handle leaks ("Too many open files") and ensuring buffered data is flushed. Any class implementing `AutoCloseable` works. It replaced the verbose and error-prone manual `finally { stream.close(); }` pattern.

---

### Q: What is the decorator pattern and how does java.io use it?

The decorator pattern wraps an object to add behavior without changing its class. `java.io` uses it heavily: you wrap a base stream in feature-adding wrappers. For example, `new BufferedReader(new InputStreamReader(new FileInputStream(file), UTF_8))` stacks file reading + charset decoding + buffering + `readLine()`. You recognize decorators because their constructor takes another stream as an argument.

---

### Q: What is the difference between classic I/O and NIO?

Classic I/O (`java.io`) is **stream-oriented** and **blocking** — the thread waits for data, typically one thread per connection. NIO (`java.nio`) is **buffer-oriented** and can be **non-blocking** — data flows through **channels** into **buffers**, and a single thread can manage many connections via a **selector**. Classic I/O is simpler for files; NIO scales better for high-concurrency network servers.

---

### Q: What is a Channel, a Buffer, and a Selector in NIO?

- **Buffer**: a fixed-size memory block you read data into and write data out of (e.g., `ByteBuffer`). You call `flip()` to switch from writing-in to reading-out.
- **Channel**: a two-way connection to a file or socket that always reads/writes through a buffer (e.g., `FileChannel`, `SocketChannel`).
- **Selector**: lets one thread monitor many channels and act only on the ones that are ready, enabling non-blocking I/O where one thread serves thousands of connections.

---

### Q: What is the difference between Files.lines() and Files.readAllLines()?

`readAllLines()` loads the **entire file** into a `List<String>` in memory — fine for small files, but causes `OutOfMemoryError` on huge ones. `Files.lines()` returns a **lazy Stream** that reads one line at a time, keeping memory roughly constant regardless of file size — ideal for large files. `Files.lines()` holds an open file handle, so it must be used inside try-with-resources.

---

### Q: Why should you always specify a charset when reading/writing text?

Because the **platform default charset varies** between machines (UTF-8 on one, Windows-1252 on another). Code that works on your laptop can corrupt non-ASCII characters (mojibake) in production. Always pass `StandardCharsets.UTF_8` explicitly. Use the `StandardCharsets` constants rather than the `"UTF-8"` string, since the latter can throw `UnsupportedEncodingException`.

---

### Q: What is the difference between Path and File?

`File` is the old `java.io` class for file paths with clumsy boolean-returning operations (e.g., `delete()` returns false on failure, easy to ignore). `Path` (from `java.nio.file`, Java 7+) is the modern replacement, used with the `Files` utility class. `Files` throws descriptive exceptions, supports atomic moves, symbolic links, file attributes, directory walking, and charset-aware read/write. Prefer `Path` + `Files` in new code.

---

### Q: How do you copy a file in Java?

Modern way (NIO.2): `Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING)`. Classic way: open a `FileInputStream` and `FileOutputStream` in try-with-resources and loop, reading into a `byte[]` buffer and writing `out.write(buffer, 0, bytesRead)` until `read()` returns -1. Since Java 9 you can also use `in.transferTo(out)`.

---

### Q: What happens if you don't close a stream?

You leak an operating-system file handle. The OS has a limited number of file descriptors, so leaking them eventually causes "Too many open files" errors and crashes. For output streams, buffered data that wasn't flushed may be lost. try-with-resources prevents this by always closing.

---

### Q: When would you use a byte stream vs a character stream for a network socket carrying text?

The socket gives you raw bytes (`InputStream`). Since the payload is text, you bridge it into a character stream using `InputStreamReader` (and usually wrap that in a `BufferedReader`), passing an explicit charset like UTF-8. This decodes the bytes into characters correctly. For binary payloads, you'd keep working with the byte stream directly.

---

### Q: Is the java.io Stream the same as a Java 8 Stream?

No — completely different. A `java.io` stream (`InputStream`/`Reader`) is an I/O channel for reading/writing bytes or characters to a source/destination. A `java.util.stream.Stream` (Java 8) is a pipeline for processing collections of data with operations like `map`, `filter`, `collect`. The shared word "stream" is the only thing in common. Note `Files.lines()` returns the Java 8 `Stream<String>` type.

---

## Quick Reference Cheat Sheet

```
TWO STREAM FAMILIES
  Byte streams      → InputStream / OutputStream   → binary (images, PDFs, ZIPs)
  Character streams → Reader / Writer              → text (charset-aware)
  Bridge bytes→chars → InputStreamReader / OutputStreamWriter (pass UTF-8!)

BUFFERING (wrap for speed + readLine())
  BufferedReader  / BufferedWriter        (character)
  BufferedInputStream / BufferedOutputStream (byte)
  Why: batches I/O into big chunks → fewer slow disk/system calls

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
  Paths/Path.of = make a Path
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
  Buffer  → memory block; call flip() to switch write→read mode
  Channel → two-way connection (file/socket), works through a buffer
  Selector→ 1 thread monitors many channels, handles only ready ones

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

*Last Updated: 2026-06-11*

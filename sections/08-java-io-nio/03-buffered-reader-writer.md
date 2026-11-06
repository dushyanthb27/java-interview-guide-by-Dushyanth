# Q3: BufferedReader and BufferedWriter Usage

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟢 Easy

---

## Question
How do `BufferedReader` and `BufferedWriter` work under the hood? Why is buffering essential for I/O performance, and what are the architectural patterns used in stream chaining?

---

## Answer

Buffered streams wrap unbuffered streams and maintain an internal memory buffer (defaulting to 8KB / 8192 units). Instead of performing a costly system call (context switch from user space to kernel space) for every single byte or character read/written from disk or network, data is transferred in large, efficient blocks.

---

## The Buffering Concept

```
Without Buffering:
  App ──(1 byte)──► OS Syscall ──► Disk / Network (per byte/char) ──► Extremely Slow

With Buffering (Default 8KB Buffer):
  App ──(1 char)──► Internal Memory Buffer (8192 slots) 
                          │ (Flushes / Fills once buffer is exhausted)
                          ▼
                    OS Syscall ──► Disk / Network (per 8KB block) ──► High Throughput
```

```java
// ❌ Anti-pattern: Unbuffered — one OS syscall per byte read
try (FileInputStream fis = new FileInputStream("large.dat")) {
    int b;
    while ((b = fis.read()) != -1) {  // Triggers OS syscall for EVERY byte!
        // process byte
    }
}

// ✅ Best practice: Buffered — one OS syscall per 8KB chunk
try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream("large.dat"))) {
    int b;
    while ((b = bis.read()) != -1) {  // Reads from memory buffer, hits disk every 8KB
        // process byte
    }
}
```

---

## BufferedReader

### Basic Usage

```java
// ✅ Read a text file line by line
try (BufferedReader br = new BufferedReader(
        new FileReader("data.txt", StandardCharsets.UTF_8))) {
    String line;
    while ((line = br.readLine()) != null) {  // readLine() is BufferedReader-specific
        System.out.println(line);
    }
}

// ✅ Modern alternative: Files.newBufferedReader() returning Stream<String>
try (BufferedReader br = Files.newBufferedReader(
        Path.of("data.txt"), StandardCharsets.UTF_8)) {
    br.lines()  // returns Stream<String>
      .filter(line -> !line.isBlank())
      .forEach(System.out::println);
}
```

### Custom Buffer Size

```java
// Default buffer size: 8192 characters (16 KB in memory for chars)
BufferedReader br1 = new BufferedReader(new FileReader("file.txt"));

// ✅ Custom buffer size for very large files or high-throughput log processing
BufferedReader br2 = new BufferedReader(new FileReader("huge.txt"), 65536); // 64KB buffer

// ✅ Smaller buffer for memory-constrained environments processing many small files
BufferedReader br3 = new BufferedReader(new FileReader("small.txt"), 1024); // 1KB buffer
```

### Key Methods

```java
try (BufferedReader br = new BufferedReader(
        new FileReader("data.txt", StandardCharsets.UTF_8))) {

    // readLine() — reads one line (excludes line terminator \n, \r\n), returns null at EOF
    String line = br.readLine();

    // read() — reads a single character (returns int 0-65535, or -1 at EOF)
    int ch = br.read();

    // read(char[], off, len) — reads chars into array portion
    char[] buffer = new char[256];
    int charsRead = br.read(buffer, 0, buffer.length);

    // ready() — checks if stream is guaranteed ready to be read without blocking
    boolean canRead = br.ready();

    // skip(n) — skips n characters in the stream
    long skipped = br.skip(100);

    // mark(readAheadLimit) + reset() — allows re-reading within buffer capacity
    br.mark(512);       // mark current position with read-ahead limit
    br.readLine();      // read ahead
    br.reset();         // rewind stream back to marked position

    // lines() — returns sequential Stream<String> (Java 8+)
    Stream<String> lineStream = br.lines();
}
```

---

## BufferedWriter

### Basic Usage

```java
// ✅ Write text to a file with explicit charset
try (BufferedWriter bw = new BufferedWriter(
        new FileWriter("output.txt", StandardCharsets.UTF_8))) {
    bw.write("First line");
    bw.newLine();           // writes platform-independent line separator
    bw.write("Second line");
    bw.newLine();
}

// ✅ Modern alternative: Files.newBufferedWriter()
try (BufferedWriter bw = Files.newBufferedWriter(
        Path.of("output.txt"), StandardCharsets.UTF_8,
        StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {
    bw.write("Hello, NIO!");
    bw.newLine();
}
```

### Key Methods

```java
try (BufferedWriter bw = new BufferedWriter(
        new FileWriter("log.txt", StandardCharsets.UTF_8))) {

    // write(String) — writes a string to buffer
    bw.write("Log entry");

    // write(String, off, len) — writes a substring
    bw.write("Hello, World!", 0, 5);  // writes "Hello"

    // write(int) — writes a single character
    bw.write('A');

    // write(char[], off, len) — writes a char array portion
    char[] data = {'H', 'i', '!'};
    bw.write(data, 0, data.length);

    // newLine() — writes platform-specific line separator (\r\n on Windows, \n on Unix/macOS)
    bw.newLine();

    // flush() — forces buffered content to be written to underlying stream and OS
    bw.flush();
}
```

---

## BufferedInputStream / BufferedOutputStream

```java
// ✅ High-performance buffered byte stream file copy
public static void copyFile(Path source, Path target) throws IOException {
    try (BufferedInputStream bis = new BufferedInputStream(
                new FileInputStream(source.toFile()));
         BufferedOutputStream bos = new BufferedOutputStream(
                new FileOutputStream(target.toFile()))) {

        byte[] buffer = new byte[8192];
        int bytesRead;
        while ((bytesRead = bis.read(buffer)) != -1) {
            bos.write(buffer, 0, bytesRead);
        }
    } // auto-flushes and closes underlying streams via try-with-resources
}
```

---

## Decorator Pattern & Stream Chaining

Buffered streams are classic examples of the **Decorator Pattern**. They wrap an underlying stream to add buffering and enhanced capabilities without modifying the underlying class hierarchy.

```java
// ✅ Chaining streams: raw byte stream → buffered byte stream → data stream (typed primitives)
DataInputStream dis = new DataInputStream(      
    new BufferedInputStream(                     
        new FileInputStream("data.bin")          
    )
);

// ✅ Chaining writers: raw char stream → buffered char stream → print writer (formatting)
PrintWriter pw = new PrintWriter(               
    new BufferedWriter(                          
        new FileWriter("log.txt")               
    )
);

// ✅ Bridge chain: byte stream → charset decoder bridge → buffered character stream
BufferedReader br = new BufferedReader(          
    new InputStreamReader(                       
        new FileInputStream("data.txt"),         
        StandardCharsets.UTF_8                   
    )
);

// ✅ Standard console input reader chain
BufferedReader console = new BufferedReader(
    new InputStreamReader(System.in, StandardCharsets.UTF_8)
);
String userInput = console.readLine();
```

### Decorator Chain Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  PrintWriter (Formatting: println, printf, auto-flush)          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  BufferedWriter (8KB character buffer, newLine support)   │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  OutputStreamWriter (Char ──► Byte character encoder)│  │  │
│  │  │  ┌───────────────────────────────────────────────┐  │  │  │
│  │  │  │  FileOutputStream (Raw OS byte stream)        │  │  │  │
│  │  │  └───────────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Performance Benchmark Comparison

```java
public class BufferingBenchmark {

    public static void main(String[] args) throws IOException {
        Path file = Path.of("test-1mb.txt");

        // Generate 1MB test file
        byte[] data = new byte[1_000_000];
        Arrays.fill(data, (byte) 'A');
        Files.write(file, data);

        // ❌ Unbuffered read: ~800ms for 1MB (1M individual OS context switches)
        long start = System.nanoTime();
        try (FileInputStream fis = new FileInputStream(file.toFile())) {
            while (fis.read() != -1) {}
        }
        System.out.printf("Unbuffered: %d ms%n",
            (System.nanoTime() - start) / 1_000_000);

        // ✅ Buffered read: ~5ms for 1MB (~122 block reads)
        start = System.nanoTime();
        try (BufferedInputStream bis = new BufferedInputStream(
                new FileInputStream(file.toFile()))) {
            while (bis.read() != -1) {}
        }
        System.out.printf("Buffered:   %d ms%n",
            (System.nanoTime() - start) / 1_000_000);

        Files.delete(file);
    }
}
```

---

## Common Anti-Patterns

```java
// ❌ Double buffering — redundant and wastes memory (wrapping a BufferedReader inside another BufferedReader)
BufferedReader br = new BufferedReader(
    new BufferedReader(new FileReader("file.txt")));

// ❌ Forgetting to handle unflushed data when bypassing try-with-resources
BufferedWriter bw = new BufferedWriter(new FileWriter("out.txt"));
bw.write("data");
// If exception occurs before close() without try-with-resources, data in the 8KB buffer is lost!

// ❌ Not using try-with-resources on decorator chains
BufferedReader br = new BufferedReader(new FileReader("file.txt"));
String line = br.readLine();
// Forgot br.close() — causes persistent file handle leaks in the OS process table

// ✅ Correct approach: Always use try-with-resources
try (BufferedReader br2 = new BufferedReader(new FileReader("file.txt", StandardCharsets.UTF_8))) {
    String line2 = br2.readLine();
}
```

---

## ⚠️ Gotchas

- **Data Loss on BufferedWriter Without Flush/Close**: `BufferedWriter` accumulates written data in its internal memory buffer. If an application crashes, terminates abruptly, or fails to call `close()` or `flush()`, any data currently residing inside the buffer (less than 8KB) is never written to disk or network, resulting in truncated output files.
- **`mark()` and `reset()` Buffer Limitations**: `BufferedReader.mark(int readAheadLimit)` relies on its internal buffer. If the number of characters read between `mark()` and `reset()` exceeds `readAheadLimit` (or exceeds the total capacity of the internal buffer), `mark` is invalidated and calling `reset()` throws an `IOException` (`Stream reset failed`).
- **Memory Overhead of Excessive Buffer Sizes**: While setting a 1MB buffer (`new BufferedReader(reader, 1024 * 1024)`) can optimize high-throughput reads, allocating oversized buffers across thousands of concurrent network connections or file workers in a multi-threaded server causes severe heap memory pressure and frequent Garbage Collection (GC) pauses.
- **Blocking Behavior of `readLine()` on Network Sockets**: When `BufferedReader.readLine()` is called on a socket input stream, the thread blocks indefinitely until a newline character (`\n` or `\r\n`) or EOF is received. If a malicious or hanging client sends data without a newline terminator, the server thread hangs forever, risking thread-pool exhaustion.
- **Hidden Double Buffering in Framework Wrappers**: Wrapping an already-buffered stream (such as passing a `BufferedInputStream` into another `BufferedInputStream` or chaining multiple buffering decorators) creates redundant buffer allocations, wastes heap memory, and adds unnecessary pointer indirection overhead during every read operation.

---

## Key Takeaways

1. **Buffering minimizes OS system calls** — internal buffers (default 8KB) reduce expensive user/kernel mode switches
2. **`readLine()`** is exclusively available on `BufferedReader` and `LineNumberReader`
3. **`newLine()`** writes platform-agnostic line delimiters (`\r\n` on Windows, `\n` on Unix)
4. **Decorator pattern** enables flexible, modular stream composition (raw → buffered → formatted)
5. **Always use try-with-resources** to guarantee proper closure and flushing of all decorators in the chain

---

[← Previous: Q2 - File I/O](./02-file-io.md) | [Next: Q4 - Serialization →](./q04-serialization.md)

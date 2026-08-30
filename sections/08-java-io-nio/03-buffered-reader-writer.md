# Q3: BufferedReader and BufferedWriter Usage

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟢 Easy

---

## Question
How do BufferedReader and BufferedWriter work? Why is buffering important for I/O performance?

---

## Answer

Buffered streams wrap unbuffered streams and maintain an internal buffer, reducing the number of native I/O calls. Instead of reading/writing one byte or character at a time, data is transferred in larger chunks.

---

## The Buffering Concept

```
Without Buffering:
  App → OS call → Disk (per byte/char) → very slow

With Buffering (default 8KB):
  App → Internal Buffer (8192 bytes) → OS call → Disk (per chunk) → much faster
```

```java
// ❌ Anti-pattern: Unbuffered — one OS call per byte
try (FileInputStream fis = new FileInputStream("large.dat")) {
    int b;
    while ((b = fis.read()) != -1) {  // OS call for EVERY byte!
        // process byte
    }
}

// ✅ Best practice: Buffered — one OS call per 8KB chunk
try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream("large.dat"))) {
    int b;
    while ((b = bis.read()) != -1) {  // reads from internal buffer
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

// ✅ Modern alternative: Files.newBufferedReader()
try (BufferedReader br = Files.newBufferedReader(
        Path.of("data.txt"), StandardCharsets.UTF_8)) {
    br.lines()  // returns Stream<String>
      .filter(line -> !line.isBlank())
      .forEach(System.out::println);
}
```

### Custom Buffer Size

```java
// Default buffer size: 8192 characters (8KB)
BufferedReader br = new BufferedReader(new FileReader("file.txt"));

// ✅ Custom buffer size for very large files
BufferedReader br = new BufferedReader(new FileReader("huge.txt"), 65536); // 64KB buffer

// ✅ Smaller buffer for many small files
BufferedReader br = new BufferedReader(new FileReader("small.txt"), 1024); // 1KB buffer
```

### Key Methods

```java
try (BufferedReader br = new BufferedReader(
        new FileReader("data.txt", StandardCharsets.UTF_8))) {

    // readLine() — reads one line (excludes line terminator), null at EOF
    String line = br.readLine();

    // read() — reads a single character, -1 at EOF
    int ch = br.read();

    // read(char[], off, len) — reads chars into array
    char[] buffer = new char[256];
    int charsRead = br.read(buffer, 0, buffer.length);

    // ready() — checks if stream is ready to be read
    boolean canRead = br.ready();

    // skip(n) — skips n characters
    long skipped = br.skip(100);

    // mark(readAheadLimit) + reset() — for re-reading
    br.mark(512);       // mark current position
    br.readLine();      // read ahead
    br.reset();         // go back to marked position

    // lines() — returns Stream<String> (Java 8+)
    Stream<String> lineStream = br.lines();
}
```

---

## BufferedWriter

### Basic Usage

```java
// ✅ Write text to a file
try (BufferedWriter bw = new BufferedWriter(
        new FileWriter("output.txt", StandardCharsets.UTF_8))) {
    bw.write("First line");
    bw.newLine();           // platform-independent line separator
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

    // write(String) — writes a string
    bw.write("Log entry");

    // write(String, off, len) — writes a substring
    bw.write("Hello, World!", 0, 5);  // writes "Hello"

    // write(int) — writes a single character
    bw.write('A');

    // write(char[], off, len) — writes a char array portion
    char[] data = {'H', 'i', '!'};
    bw.write(data, 0, data.length);

    // newLine() — writes platform-specific line separator
    bw.newLine();  // "\r\n" on Windows, "\n" on Unix

    // flush() — forces buffered data to be written
    bw.flush();
}
```

---

## BufferedInputStream / BufferedOutputStream

```java
// ✅ Buffered byte stream copy
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
    } // auto-flush and close
}
```

---

## Decorator Pattern (Stream Chaining)

Buffered streams use the **Decorator Pattern** — wrapping streams to add functionality.

```java
// ✅ Chaining streams: raw → buffered → data
DataInputStream dis = new DataInputStream(      // adds typed reads
    new BufferedInputStream(                     // adds buffering
        new FileInputStream("data.bin")          // raw byte stream
    )
);

// ✅ Chaining writers: raw → buffered → print
PrintWriter pw = new PrintWriter(               // adds println/printf
    new BufferedWriter(                          // adds buffering
        new FileWriter("log.txt")               // raw char stream
    )
);

// ✅ Bridge chain: byte stream → char stream → buffered
BufferedReader br = new BufferedReader(          // adds buffering + readLine
    new InputStreamReader(                       // byte-to-char bridge
        new FileInputStream("data.txt"),         // raw byte stream
        StandardCharsets.UTF_8                   // explicit encoding
    )
);

// ✅ Console input reading
BufferedReader console = new BufferedReader(
    new InputStreamReader(System.in, StandardCharsets.UTF_8)
);
String userInput = console.readLine();
```

### Decorator Chain Diagram

```
┌──────────────────────────────────────────────────┐
│  PrintWriter (formatting: println, printf)       │
│  ┌────────────────────────────────────────────┐  │
│  │  BufferedWriter (8KB buffer, newLine)       │  │
│  │  ┌──────────────────────────────────────┐  │  │
│  │  │  OutputStreamWriter (char → byte)     │  │  │
│  │  │  ┌────────────────────────────────┐  │  │  │
│  │  │  │  FileOutputStream (raw bytes)   │  │  │  │
│  │  │  └────────────────────────────────┘  │  │  │
│  │  └──────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

---

## Performance Comparison

```java
public class BufferingBenchmark {

    public static void main(String[] args) throws IOException {
        Path file = Path.of("test-1mb.txt");

        // Generate test file
        byte[] data = new byte[1_000_000];
        Arrays.fill(data, (byte) 'A');
        Files.write(file, data);

        // ❌ Unbuffered read: ~800ms for 1MB
        long start = System.nanoTime();
        try (FileInputStream fis = new FileInputStream(file.toFile())) {
            while (fis.read() != -1) {}
        }
        System.out.printf("Unbuffered: %d ms%n",
            (System.nanoTime() - start) / 1_000_000);

        // ✅ Buffered read: ~5ms for 1MB
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
// ❌ Double buffering — unnecessary
BufferedReader br = new BufferedReader(
    new BufferedReader(new FileReader("file.txt")));

// ❌ Forgetting to flush before close (auto-close handles this)
BufferedWriter bw = new BufferedWriter(new FileWriter("out.txt"));
bw.write("data");
// bw.flush();  — not needed if using try-with-resources

// ❌ Not using try-with-resources
BufferedReader br = new BufferedReader(new FileReader("file.txt"));
String line = br.readLine();
// forgot br.close() — resource leak!

// ✅ Always use try-with-resources
try (BufferedReader br2 = new BufferedReader(new FileReader("file.txt"))) {
    String line2 = br2.readLine();
}
```

---

## Key Takeaways

1. **Buffering reduces OS calls** — internal buffer (default 8KB) dramatically improves performance
2. **`readLine()`** is only available on `BufferedReader`, not basic `Reader`
3. **`newLine()`** writes platform-independent line separators
4. **Decorator pattern** enables flexible stream chaining (raw → buffered → formatted)
5. **`Files.newBufferedReader()`** / **`Files.newBufferedWriter()`** are the modern preferred constructors
6. **Always use try-with-resources** to prevent resource leaks

---

[← Previous: Q2 - File I/O](./q02-file-io.md) | [Next: Q4 - Serialization →](./q04-serialization.md)

# Q1: Byte Streams vs Character Streams

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟢 Easy

---

## Question
What is the fundamental architectural difference between Java Byte Streams and Character Streams? When should you choose one over the other, and what happens under the hood when bridging them?

---

## Answer

Java provides two fundamental, parallel stream hierarchies for I/O operations to handle different data modalities:
- **Byte Streams** (`InputStream` / `OutputStream`) — designed for handling raw binary data (8-bit bytes). They perform no interpretation, encoding, or transformation of the underlying data.
- **Character Streams** (`Reader` / `Writer`) — designed for handling 16-bit Unicode character text data. They natively process characters and automatically manage character set decoding/encoding.

---

## Byte Stream Hierarchy

```
InputStream (abstract)                  OutputStream (abstract)
├── FileInputStream                     ├── FileOutputStream
├── ByteArrayInputStream                ├── ByteArrayOutputStream
├── BufferedInputStream                 ├── BufferedOutputStream
├── DataInputStream                     ├── DataOutputStream
├── ObjectInputStream                   ├── ObjectOutputStream
└── FilterInputStream                   └── FilterOutputStream
```

### Reading Bytes

```java
// ✅ Read a binary file byte by byte
try (InputStream is = new FileInputStream("image.png")) {
    int byteData;
    while ((byteData = is.read()) != -1) {
        // process each byte (0–255, -1 represents EOF)
    }
}

// ✅ Read into a byte array for better performance
try (InputStream is = new FileInputStream("image.png")) {
    byte[] buffer = new byte[4096];
    int bytesRead;
    while ((bytesRead = is.read(buffer)) != -1) {
        // process buffer[0..bytesRead-1]
    }
}

// ✅ Java 9+: Read all bytes at once (convenience method)
byte[] allBytes = Files.readAllBytes(Path.of("image.png"));

// ✅ Java 11+: InputStream.readAllBytes() instance method
try (InputStream is = new FileInputStream("data.bin")) {
    byte[] data = is.readAllBytes();
}
```

### Writing Bytes

```java
// ✅ Write bytes to a file
try (OutputStream os = new FileOutputStream("output.bin")) {
    os.write(65);                      // single byte (casts int to 8-bit byte)
    os.write(new byte[]{66, 67, 68}); // byte array chunk
}
```

---

## Character Stream Hierarchy

```
Reader (abstract)                       Writer (abstract)
├── FileReader                          ├── FileWriter
├── BufferedReader                      ├── BufferedWriter
├── InputStreamReader                   ├── OutputStreamWriter
├── StringReader                        ├── StringWriter
├── CharArrayReader                     ├── CharArrayWriter
└── FilterReader                        └── FilterWriter
```

### Reading Characters

```java
// ✅ Read a text file character by character
try (Reader reader = new FileReader("notes.txt", StandardCharsets.UTF_8)) {
    int charData;
    while ((charData = reader.read()) != -1) {
        char ch = (char) charData;
        // process character (0–65535 BMP code point or surrogate pair)
    }
}

// ✅ Read into a char array with explicit charset
try (Reader reader = new FileReader("notes.txt", StandardCharsets.UTF_8)) {
    char[] buffer = new char[1024];
    int charsRead;
    while ((charsRead = reader.read(buffer)) != -1) {
        String chunk = new String(buffer, 0, charsRead);
    }
}
```

### Writing Characters

```java
// ✅ Write text to a file with explicit charset
try (Writer writer = new FileWriter("output.txt", StandardCharsets.UTF_8)) {
    writer.write("Hello, World!\n");
    writer.write("Java I/O is powerful.");
}
```

---

## Bridge Classes: InputStreamReader & OutputStreamWriter

These adapter classes bridge byte streams and character streams. They wrap an underlying `InputStream` or `OutputStream` and translate byte sequences into Unicode characters (and vice versa) according to a specified `Charset`.

```java
// ✅ Read bytes as characters with explicit encoding
try (Reader reader = new InputStreamReader(
        new FileInputStream("data.txt"), StandardCharsets.UTF_8)) {
    // reads underlying bytes → decodes variable-width sequences to chars using UTF-8
    int ch;
    while ((ch = reader.read()) != -1) {
        System.out.print((char) ch);
    }
}

// ✅ Write characters as bytes with explicit encoding
try (Writer writer = new OutputStreamWriter(
        new FileOutputStream("output.txt"), StandardCharsets.ISO_8859_1)) {
    writer.write("Encoded text");
}

// ❌ Anti-pattern: Relying on default platform charset (non-portable)
Reader reader = new InputStreamReader(new FileInputStream("data.txt"));

// ✅ Best practice: Always specify charset explicitly
Reader reader = new InputStreamReader(
    new FileInputStream("data.txt"), StandardCharsets.UTF_8);
```

---

## Character Encoding & Internals

Java's `char` primitive represents a 16-bit UTF-16 code unit. Multilingual text (such as emojis or supplementary code points outside the Basic Multilingual Plane) requires surrogate pairs (two `char` values). 

```java
// ✅ Common charsets
StandardCharsets.UTF_8        // Variable-width (1 to 4 bytes per character)
StandardCharsets.UTF_16       // 2 or 4 bytes, includes BOM option
StandardCharsets.US_ASCII     // 7-bit ASCII (0-127)
StandardCharsets.ISO_8859_1   // 8-bit Latin-1 (1-to-1 byte-to-char mapping)

// ✅ Converting between bytes and strings safely
String text = "Hello, 世界! 🚀";
byte[] utf8Bytes = text.getBytes(StandardCharsets.UTF_8);
String decoded = new String(utf8Bytes, StandardCharsets.UTF_8);

// ❌ Anti-pattern: Using String.getBytes() without charset
byte[] bytes = text.getBytes(); // uses platform default — breaks cross-platform deployments!
```

---

## When to Use Each

| Use Case | Stream Type | Example Classes / Utilities |
|----------|------------|-----------------------------|
| Images, audio, video files | Byte streams | `FileInputStream`, `FileOutputStream`, `Files.readAllBytes()` |
| Binary network protocols / RPC | Byte streams | `DataInputStream`, `DataOutputStream`, `Socket.getInputStream()` |
| Text files (JSON, XML, Java source) | Character streams | `BufferedReader`, `Files.readString()` |
| Console input/output | Character streams | `BufferedReader` + `InputStreamReader(System.in)` |
| Network text protocols (HTTP body) | Bridge classes | `InputStreamReader` wrapping socket `InputStream` |
| Object serialization / RMI | Byte streams | `ObjectInputStream`, `ObjectOutputStream` |

---

## Common Implementations Summary

```java
// ✅ Byte stream: Copy a binary file efficiently
public static void copyBinary(Path source, Path target) throws IOException {
    try (InputStream in = new BufferedInputStream(new FileInputStream(source.toFile()));
         OutputStream out = new BufferedOutputStream(new FileOutputStream(target.toFile()))) {
        byte[] buffer = new byte[8192];
        int len;
        while ((len = in.read(buffer)) != -1) {
            out.write(buffer, 0, len);
        }
    }
}

// ✅ Character stream: Read text file line by line
public static List<String> readLines(Path path) throws IOException {
    List<String> lines = new ArrayList<>();
    try (BufferedReader br = new BufferedReader(
            new FileReader(path.toFile(), StandardCharsets.UTF_8))) {
        String line;
        while ((line = br.readLine()) != null) {
            lines.add(line);
        }
    }
    return lines;
}

// ✅ Modern high-level approach (Java 11+): Use Files utility
String content = Files.readString(Path.of("file.txt"), StandardCharsets.UTF_8);
List<String> lines = Files.readAllLines(Path.of("file.txt"), StandardCharsets.UTF_8);
```

---

## ⚠️ Gotchas

- **Silent Data Corruption via Platform Default Charset**: Instantiating `FileReader`, `FileWriter`, `InputStreamReader`, or calling `String.getBytes()` without an explicit `Charset` falls back to `Charset.defaultCharset()`. This varies between macOS/Linux (often UTF-8) and Windows (often Cp1252), causing silent character corruption or mojibake in production deployments.
- **Surrogate Pair Truncation in Character Streams**: Reading text character-by-character via `Reader.read()` treats `char` as a 16-bit unit. Supplementary Unicode characters (emojis, rare ideographs) span two `char` code units (surrogate pairs). Splitting a character stream arbitrarily in buffers or casting raw ints can corrupt surrogate pairs.
- **Resource Leaks in Stream Chains**: When chaining decorators (e.g., `new BufferedReader(new InputStreamReader(new FileInputStream(file)))`), closing the outermost wrapper (`BufferedReader`) automatically closes the underlying streams. However, if an exception occurs during construction *between* nested instantiations (e.g., `FileInputStream` succeeds but `InputStreamReader` constructor throws an `IOException`), earlier streams are leaked unless explicitly guarded or managed via multi-resource try-with-resources.
- **EOF Handling Mismatch (`-1` as `int`)**: Both `InputStream.read()` and `Reader.read()` return an `int` instead of `byte` or `char`. This is required so that byte value `255` (`0xFF`) does not falsely signal End-Of-File (which is represented by `-1`). Casting the return value directly to `byte` or `char` before checking against `-1` results in an infinite loop on byte `0xFF`.
- **Unhandled Partial Reads (`read(byte[])`)**: The `read(byte[] b)` and `read(char[] c)` methods do not guarantee that the buffer is completely filled. They return the number of bytes/chars actually read (which may be less than the buffer length). Assuming the entire buffer is populated on every read operation leads to processing stale or uninitialized data.

---

## Key Takeaways

1. **Byte streams** (`InputStream`/`OutputStream`) handle raw binary data — use for images, audio, serialization
2. **Character streams** (`Reader`/`Writer`) handle text with encoding — use for text files, console I/O
3. **Always specify `StandardCharsets.UTF_8`** explicitly to avoid platform-dependent behavior
4. **Bridge classes** (`InputStreamReader`/`OutputStreamWriter`) convert between byte and character streams
5. **Modern Java** (`Files.readString`, `Files.readAllBytes`) simplifies common I/O tasks

---

[Next: Q2 - File I/O →](./02-file-io.md)

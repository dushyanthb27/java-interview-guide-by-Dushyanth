# Q1: Byte Streams vs Character Streams

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟢 Easy

---

## Question
What is the difference between byte streams and character streams in Java? When should you use each?

---

## Answer

Java provides two fundamental stream hierarchies for I/O operations:
- **Byte Streams** — handle raw binary data (8-bit bytes)
- **Character Streams** — handle Unicode text data (16-bit chars)

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
        // process each byte (0–255)
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

// ✅ Java 9+: Read all bytes at once
byte[] allBytes = Files.readAllBytes(Path.of("image.png"));

// ✅ Java 11+: InputStream.readAllBytes()
try (InputStream is = new FileInputStream("data.bin")) {
    byte[] data = is.readAllBytes();
}
```

### Writing Bytes

```java
// ✅ Write bytes to a file
try (OutputStream os = new FileOutputStream("output.bin")) {
    os.write(65);                      // single byte
    os.write(new byte[]{66, 67, 68}); // byte array
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
└── PrintReader                         └── PrintWriter
```

### Reading Characters

```java
// ✅ Read a text file character by character
try (Reader reader = new FileReader("notes.txt")) {
    int charData;
    while ((charData = reader.read()) != -1) {
        char ch = (char) charData;
        // process character
    }
}

// ✅ Read into a char array
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
// ✅ Write text to a file
try (Writer writer = new FileWriter("output.txt", StandardCharsets.UTF_8)) {
    writer.write("Hello, World!\n");
    writer.write("Java I/O is powerful.");
}
```

---

## Bridge Classes: InputStreamReader & OutputStreamWriter

These classes bridge byte streams and character streams, handling character encoding.

```java
// ✅ Read bytes as characters with explicit encoding
try (Reader reader = new InputStreamReader(
        new FileInputStream("data.txt"), StandardCharsets.UTF_8)) {
    // reads bytes → decodes to chars using UTF-8
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

// ❌ Anti-pattern: Relying on default charset (platform-dependent)
Reader reader = new InputStreamReader(new FileInputStream("data.txt"));

// ✅ Best practice: Always specify charset explicitly
Reader reader = new InputStreamReader(
    new FileInputStream("data.txt"), StandardCharsets.UTF_8);
```

---

## Character Encoding

```java
// ✅ Common charsets
StandardCharsets.UTF_8        // Most common, variable-width
StandardCharsets.UTF_16       // Java internal representation
StandardCharsets.US_ASCII     // 7-bit ASCII
StandardCharsets.ISO_8859_1   // Latin-1

// ✅ Converting between bytes and strings
String text = "Hello, 世界!";
byte[] utf8Bytes = text.getBytes(StandardCharsets.UTF_8);
String decoded = new String(utf8Bytes, StandardCharsets.UTF_8);

// ❌ Anti-pattern: Using getBytes() without charset
byte[] bytes = text.getBytes(); // uses platform default — non-portable!
```

---

## When to Use Each

| Use Case | Stream Type | Example Classes |
|----------|------------|-----------------|
| Images, audio, video | Byte streams | `FileInputStream`, `FileOutputStream` |
| Binary protocols | Byte streams | `DataInputStream`, `DataOutputStream` |
| Text files (UTF-8, ASCII) | Character streams | `FileReader`, `FileWriter` |
| Console input/output | Character streams | `BufferedReader` + `InputStreamReader` |
| Network text protocols | Bridge classes | `InputStreamReader` wrapping socket stream |
| Serialized objects | Byte streams | `ObjectInputStream`, `ObjectOutputStream` |

---

## Common Implementations Summary

```java
// ✅ Byte stream: Copy a binary file
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

// ✅ Modern approach (Java 11+): Use Files utility
String content = Files.readString(Path.of("file.txt"), StandardCharsets.UTF_8);
List<String> lines = Files.readAllLines(Path.of("file.txt"), StandardCharsets.UTF_8);
```

---

## Key Takeaways

1. **Byte streams** (`InputStream`/`OutputStream`) handle raw binary data — use for images, audio, serialization
2. **Character streams** (`Reader`/`Writer`) handle text with encoding — use for text files, console I/O
3. **Always specify `StandardCharsets.UTF_8`** explicitly to avoid platform-dependent behavior
4. **Bridge classes** (`InputStreamReader`/`OutputStreamWriter`) convert between byte and character streams
5. **Modern Java** (`Files.readString`, `Files.readAllBytes`) simplifies common I/O tasks

---

[Next: Q2 - File I/O →](./q02-file-io.md)

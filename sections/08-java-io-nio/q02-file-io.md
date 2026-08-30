# Q2: How Does File I/O Work in Java?

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟢 Easy

---

## Question
How does file I/O work in Java? Compare traditional `java.io.File` with the modern `java.nio.file` approach.

---

## Answer

Java provides two APIs for file operations:
- **Legacy**: `java.io.File` + `FileInputStream`/`FileOutputStream`
- **Modern (NIO.2)**: `java.nio.file.Path` + `java.nio.file.Files` (preferred since Java 7+)

---

## Legacy File I/O (java.io)

### FileInputStream / FileOutputStream

```java
// ✅ Reading bytes from a file
try (FileInputStream fis = new FileInputStream("data.bin")) {
    byte[] buffer = new byte[1024];
    int bytesRead;
    while ((bytesRead = fis.read(buffer)) != -1) {
        // process buffer
    }
}

// ✅ Writing bytes to a file
try (FileOutputStream fos = new FileOutputStream("output.bin")) {
    fos.write("Hello".getBytes(StandardCharsets.UTF_8));
}

// ✅ Appending to a file
try (FileOutputStream fos = new FileOutputStream("log.txt", true)) {
    fos.write("New log entry\n".getBytes(StandardCharsets.UTF_8));
}
```

### FileReader / FileWriter

```java
// ✅ Reading text from a file (Java 11+ constructor with charset)
try (FileReader fr = new FileReader("notes.txt", StandardCharsets.UTF_8)) {
    int ch;
    while ((ch = fr.read()) != -1) {
        System.out.print((char) ch);
    }
}

// ✅ Writing text to a file
try (FileWriter fw = new FileWriter("output.txt", StandardCharsets.UTF_8)) {
    fw.write("Line 1\n");
    fw.write("Line 2\n");
}

// ❌ Anti-pattern: FileReader without explicit charset (pre-Java 11)
FileReader fr = new FileReader("data.txt"); // uses platform default encoding
```

---

## Modern File I/O (java.nio.file) — Preferred

### Path vs File

```java
// Legacy java.io.File
File file = new File("data/config.json");

// Modern java.nio.file.Path (preferred)
Path path = Path.of("data", "config.json");        // Java 11+
Path path2 = Paths.get("data", "config.json");      // Java 7+

// Converting between them
Path fromFile = file.toPath();
File fromPath = path.toFile();
```

### Reading Files with Files Utility

```java
// ✅ Read entire file as a String (Java 11+)
String content = Files.readString(Path.of("config.json"));

// ✅ Read with explicit charset
String content = Files.readString(Path.of("config.json"), StandardCharsets.UTF_8);

// ✅ Read all lines into a List
List<String> lines = Files.readAllLines(Path.of("data.csv"));

// ✅ Read all bytes
byte[] bytes = Files.readAllBytes(Path.of("image.png"));

// ✅ Lazy line stream (memory-efficient for large files)
try (Stream<String> lineStream = Files.lines(Path.of("huge-log.txt"))) {
    lineStream
        .filter(line -> line.contains("ERROR"))
        .forEach(System.out::println);
}

// ✅ BufferedReader via Files
try (BufferedReader br = Files.newBufferedReader(Path.of("data.txt"), StandardCharsets.UTF_8)) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}
```

### Writing Files with Files Utility

```java
// ✅ Write a String to a file (Java 11+)
Files.writeString(Path.of("output.txt"), "Hello, World!");

// ✅ Write with options
Files.writeString(
    Path.of("output.txt"),
    "Appended text",
    StandardCharsets.UTF_8,
    StandardOpenOption.CREATE,
    StandardOpenOption.APPEND
);

// ✅ Write lines to a file
List<String> lines = List.of("Line 1", "Line 2", "Line 3");
Files.write(Path.of("output.txt"), lines, StandardCharsets.UTF_8);

// ✅ Write bytes
byte[] data = {0x48, 0x65, 0x6C, 0x6C, 0x6F};
Files.write(Path.of("binary.dat"), data);

// ✅ BufferedWriter via Files
try (BufferedWriter bw = Files.newBufferedWriter(
        Path.of("log.txt"), StandardCharsets.UTF_8,
        StandardOpenOption.CREATE, StandardOpenOption.APPEND)) {
    bw.write("New log entry");
    bw.newLine();
}
```

---

## File Operations

### Creating Files and Directories

```java
// ✅ Create a new file
Path newFile = Files.createFile(Path.of("newfile.txt"));

// ✅ Create a directory
Path dir = Files.createDirectory(Path.of("logs"));

// ✅ Create nested directories (like mkdir -p)
Path dirs = Files.createDirectories(Path.of("data/cache/temp"));

// ✅ Create a temporary file
Path tempFile = Files.createTempFile("prefix-", ".tmp");

// ✅ Create a temporary directory
Path tempDir = Files.createTempDirectory("myapp-");
```

### Copying, Moving, and Deleting

```java
// ✅ Copy a file
Files.copy(
    Path.of("source.txt"),
    Path.of("dest.txt"),
    StandardCopyOption.REPLACE_EXISTING
);

// ✅ Copy from InputStream to file
try (InputStream is = new URL("https://example.com/data").openStream()) {
    Files.copy(is, Path.of("downloaded.dat"), StandardCopyOption.REPLACE_EXISTING);
}

// ✅ Move/rename a file
Files.move(
    Path.of("old-name.txt"),
    Path.of("new-name.txt"),
    StandardCopyOption.ATOMIC_MOVE
);

// ✅ Delete a file
Files.delete(Path.of("unwanted.txt"));          // throws if missing
Files.deleteIfExists(Path.of("maybe.txt"));      // returns boolean

// ❌ Anti-pattern: Not handling FileAlreadyExistsException on copy
Files.copy(source, target); // throws if target exists
```

### Checking File Properties

```java
Path path = Path.of("config.json");

boolean exists      = Files.exists(path);
boolean notExists   = Files.notExists(path);
boolean isFile      = Files.isRegularFile(path);
boolean isDir       = Files.isDirectory(path);
boolean isReadable  = Files.isReadable(path);
boolean isWritable  = Files.isWritable(path);
boolean isExec      = Files.isExecutable(path);
boolean isHidden    = Files.isHidden(path);
boolean isSame      = Files.isSameFile(path1, path2);
long    size        = Files.size(path);
```

---

## Directory Traversal

```java
// ✅ List directory contents (non-recursive)
try (DirectoryStream<Path> stream = Files.newDirectoryStream(
        Path.of("src"), "*.java")) {
    for (Path entry : stream) {
        System.out.println(entry.getFileName());
    }
}

// ✅ Walk a file tree recursively
try (Stream<Path> walk = Files.walk(Path.of("project"))) {
    walk.filter(Files::isRegularFile)
        .filter(p -> p.toString().endsWith(".java"))
        .forEach(System.out::println);
}

// ✅ Find files matching a condition
try (Stream<Path> found = Files.find(
        Path.of("project"), 10,
        (path, attrs) -> attrs.isRegularFile() && path.toString().endsWith(".log"))) {
    found.forEach(System.out::println);
}

// ✅ List immediate children
try (Stream<Path> list = Files.list(Path.of("src"))) {
    list.forEach(System.out::println);
}
```

---

## Complete Example: File Utility Class

```java
public final class FileUtils {

    private FileUtils() {}

    // ✅ Read JSON config file
    public static String readConfig(String fileName) throws IOException {
        Path configPath = Path.of("config", fileName);
        if (!Files.exists(configPath)) {
            throw new FileNotFoundException("Config not found: " + configPath);
        }
        return Files.readString(configPath, StandardCharsets.UTF_8);
    }

    // ✅ Safe file write with backup
    public static void safeWrite(Path target, String content) throws IOException {
        Path backup = target.resolveSibling(target.getFileName() + ".bak");
        if (Files.exists(target)) {
            Files.copy(target, backup, StandardCopyOption.REPLACE_EXISTING);
        }
        Files.writeString(target, content, StandardCharsets.UTF_8,
            StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
    }

    // ✅ Recursively delete a directory
    public static void deleteRecursively(Path directory) throws IOException {
        if (!Files.exists(directory)) return;
        try (Stream<Path> walk = Files.walk(directory)) {
            walk.sorted(Comparator.reverseOrder())
                .forEach(path -> {
                    try { Files.delete(path); }
                    catch (IOException e) { throw new UncheckedIOException(e); }
                });
        }
    }
}
```

---

## Key Takeaways

1. **Prefer `java.nio.file.Path`** and `Files` over legacy `java.io.File`
2. **`Files.readString()`** and **`Files.writeString()`** (Java 11+) are the simplest way to handle text files
3. **`Files.lines()`** provides lazy, memory-efficient reading for large files
4. **Always use try-with-resources** for streams and readers to prevent resource leaks
5. **Specify `StandardCharsets.UTF_8`** explicitly when reading/writing text

---

[← Previous: Q1 - Byte vs Character Streams](./q01-byte-vs-character-streams.md) | [Next: Q3 - BufferedReader and BufferedWriter →](./q03-buffered-reader-writer.md)

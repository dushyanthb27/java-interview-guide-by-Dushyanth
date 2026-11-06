# Q2: How Does File I/O Work in Java?

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟢 Easy

---

## Question
How does file I/O work in Java? Compare traditional `java.io.File` with the modern `java.nio.file` (NIO.2) approach, and analyze why NIO.2 is superior for enterprise applications.

---

## Answer

Java provides two APIs for file operations:
- **Legacy (`java.io`)**: Built around the `File` class and blocking stream hierarchies. Operations are synchronous, lack detailed error reporting, and have cumbersome path manipulation.
- **Modern NIO.2 (`java.nio.file`)**: Introduced in Java 7 (`Path`, `Files`, `FileSystem`), offering comprehensive file metadata, asynchronous/symbolic link support, atomic file operations, rich exception reporting, and streamlined stream/string factories.

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
// Legacy java.io.File (represents file or directory path, often throws opaque security/IO exceptions)
File file = new File("data/config.json");

// Modern java.nio.file.Path (interface representing a hierarchical path, immutable and thread-safe)
Path path = Path.of("data", "config.json");        // Java 11+
Path path2 = Paths.get("data", "config.json");      // Java 7+

// Converting between legacy and modern types
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

// ✅ BufferedReader via Files utility
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

// ✅ BufferedWriter via Files utility
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
// ✅ Create a new file (throws FileAlreadyExistsException if it exists)
Path newFile = Files.createFile(Path.of("newfile.txt"));

// ✅ Create a directory
Path dir = Files.createDirectory(Path.of("logs"));

// ✅ Create nested directories (equivalent to mkdir -p)
Path dirs = Files.createDirectories(Path.of("data/cache/temp"));

// ✅ Create a secure temporary file
Path tempFile = Files.createTempFile("prefix-", ".tmp");

// ✅ Create a temporary directory
Path tempDir = Files.createTempDirectory("myapp-");
```

### Copying, Moving, and Deleting

```java
// ✅ Copy a file with overwrite option
Files.copy(
    Path.of("source.txt"),
    Path.of("dest.txt"),
    StandardCopyOption.REPLACE_EXISTING
);

// ✅ Copy from InputStream to file
try (InputStream is = URI.create("https://example.com/data").toURL().openStream()) {
    Files.copy(is, Path.of("downloaded.dat"), StandardCopyOption.REPLACE_EXISTING);
}

// ✅ Move/rename a file atomically (guaranteed atomic operation on supported file systems)
Files.move(
    Path.of("old-name.txt"),
    Path.of("new-name.txt"),
    StandardCopyOption.ATOMIC_MOVE
);

// ✅ Delete a file (throws NoSuchFileException if missing)
Files.delete(Path.of("unwanted.txt"));

// ✅ Delete if exists (returns boolean indicating deletion status)
boolean deleted = Files.deleteIfExists(Path.of("maybe.txt"));
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
// ✅ List directory contents (non-recursive, lazy DirectoryStream)
try (DirectoryStream<Path> stream = Files.newDirectoryStream(
        Path.of("src"), "*.java")) {
    for (Path entry : stream) {
        System.out.println(entry.getFileName());
    }
}

// ✅ Walk a file tree recursively (returns Stream<Path>, must be closed)
try (Stream<Path> walk = Files.walk(Path.of("project"))) {
    walk.filter(Files::isRegularFile)
        .filter(p -> p.toString().endsWith(".java"))
        .forEach(System.out::println);
}

// ✅ Find files matching a predicate with depth limit
try (Stream<Path> found = Files.find(
        Path.of("project"), 10,
        (path, attrs) -> attrs.isRegularFile() && path.toString().endsWith(".log"))) {
    found.forEach(System.out::println);
}

// ✅ List immediate children (non-recursive Stream<Path>)
try (Stream<Path> list = Files.list(Path.of("src"))) {
    list.forEach(System.out::println);
}
```

---

## Complete Example: Robust File Utility Class

```java
public final class FileUtils {

    private FileUtils() {}

    // ✅ Read JSON config file with validation
    public static String readConfig(String fileName) throws IOException {
        Path configPath = Path.of("config", fileName);
        if (!Files.exists(configPath)) {
            throw new FileNotFoundException("Config not found: " + configPath);
        }
        return Files.readString(configPath, StandardCharsets.UTF_8);
    }

    // ✅ Safe file write with atomic backup rotation
    public static void safeWrite(Path target, String content) throws IOException {
        Path backup = target.resolveSibling(target.getFileName() + ".bak");
        if (Files.exists(target)) {
        	// Copy existing target to backup atomically or with replace
            Files.copy(target, backup, StandardCopyOption.REPLACE_EXISTING);
        }
        Files.writeString(target, content, StandardCharsets.UTF_8,
            StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
    }

    // ✅ Recursively delete a directory safely (post-order traversal)
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

## ⚠️ Gotchas

- **Resource Leaks from Unclosed File Streams / Streams**: Methods like `Files.lines()`, `Files.walk()`, `Files.list()`, and `Files.find()` return a `Stream<Path>` that wraps underlying native directory streams. Failing to wrap them in a try-with-resources block or calling `close()` causes open file descriptor leaks at the OS level, eventually leading to `Too many open files` errors.
- **Race Conditions in `Files.exists()` Checks**: Checking `if (Files.exists(path))` followed by `Files.createFile(path)` is non-atomic. Between the existence check and creation, another thread or process may create the file, resulting in a `FileAlreadyExistsException`. Use `StandardOpenOption.CREATE_NEW` or handle exceptions defensively instead of pre-checking.
- **Incomplete Exception Context in Legacy `java.io.File`**: Legacy `java.io.File` methods return `false` or `null` on failure (e.g., `file.delete()` or `file.mkdir()`) rather than throwing descriptive exceptions (`NoSuchFileException`, `AccessDeniedException`, `FileSystemException`). NIO.2 `Files` methods provide specific subclassed `IOException` instances containing exact OS error codes and path diagnostics.
- **To-Path Conversion Quirks on Absolute/Relative Paths**: Converting between `java.io.File` and `java.nio.file.Path` via `file.toPath()` or `path.toFile()` works seamlessly, but `Path.toAbsolutePath()` resolves against the current working directory at runtime, which can cause subtle bugs in multi-threaded environments or unit tests that modify the working directory.
- **Symbolic Link Traversal Traps**: When using `Files.walk()` or `Files.find()`, default options do not follow symbolic links. However, if configured with `FileVisitOption.FOLLOW_LINKS`, circular symbolic links (symlinks pointing to parent directories) cause infinite recursion and `StackOverflowError` or `OutOfMemoryError`.

---

## Key Takeaways

1. **Prefer `java.nio.file.Path`** and `Files` over legacy `java.io.File` for all modern applications
2. **`Files.readString()`** and **`Files.writeString()`** (Java 11+) provide the cleanest APIs for text file I/O
3. **`Files.lines()`** and `Files.walk()` return streams that **must be closed** via try-with-resources to prevent file descriptor leaks
4. **`StandardCopyOption.ATOMIC_MOVE`** guarantees zero-downtime file replacement on supported operating systems
5. **Always specify `StandardCharsets.UTF_8`** explicitly when reading/writing text files

---

[← Previous: Q1 - Byte vs Character Streams](./01-byte-vs-character-streams.md) | [Next: Q3 - BufferedReader and BufferedWriter →](./03-buffered-reader-writer.md)

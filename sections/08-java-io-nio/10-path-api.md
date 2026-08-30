# Q10: Path API and Files Utility

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟡 Medium

---

## Question
How does the `Path` API work in Java NIO? What operations does the `Files` utility class provide?

---

## Answer

The `Path` interface (Java 7+, `java.nio.file`) represents a file system path and is the modern replacement for `java.io.File`. The `Files` utility class provides static methods for common file operations — reading, writing, copying, moving, deleting, traversing, and querying attributes.

---

## Creating Paths

```java
// ✅ Path.of() — preferred since Java 11
Path path1 = Path.of("src", "main", "java", "App.java");
Path path2 = Path.of("/home/user/documents/report.pdf");
Path path3 = Path.of("C:\\Users\\admin\\config.json");

// ✅ Paths.get() — equivalent, available since Java 7
Path path4 = Paths.get("src", "main", "resources");

// ✅ From URI
Path path5 = Path.of(URI.create("file:///home/user/data.txt"));

// ✅ From File (interop with legacy API)
File legacyFile = new File("data.txt");
Path fromFile = legacyFile.toPath();
File backToFile = fromFile.toFile();  // reverse conversion
```

---

## Path Operations

### Extracting Components

```java
Path path = Path.of("/home/user/documents/report.pdf");

path.getFileName();     // report.pdf
path.getParent();       // /home/user/documents
path.getRoot();         // /
path.getNameCount();    // 4 (home, user, documents, report.pdf)
path.getName(0);        // home
path.getName(2);        // documents
path.subpath(1, 3);     // user/documents
path.isAbsolute();      // true

// ✅ Convert to absolute path
Path relative = Path.of("config.json");
Path absolute = relative.toAbsolutePath();
// e.g., /home/user/project/config.json

// ✅ Normalize — removes redundant elements
Path messy = Path.of("/home/user/../user/./documents/report.pdf");
Path clean = messy.normalize();
// /home/user/documents/report.pdf

// ✅ Real path — resolves symlinks and normalizes
Path real = path.toRealPath();  // throws if path doesn't exist
```

### Resolving Paths

```java
// ✅ resolve() — append a child path to a parent
Path base = Path.of("/home/user");
Path resolved = base.resolve("documents/report.pdf");
// /home/user/documents/report.pdf

// If argument is absolute, base is ignored
Path abs = base.resolve("/etc/config");
// /etc/config

// ✅ resolveSibling() — resolve against parent directory
Path file = Path.of("/home/user/docs/old.txt");
Path sibling = file.resolveSibling("new.txt");
// /home/user/docs/new.txt
```

### Relativizing Paths

```java
// ✅ relativize() — compute relative path between two paths
Path base = Path.of("/home/user");
Path target = Path.of("/home/user/documents/report.pdf");

Path relative = base.relativize(target);
// documents/report.pdf

Path reverse = target.relativize(base);
// ../..

// ❌ Cannot relativize between absolute and relative paths
Path abs = Path.of("/home");
Path rel = Path.of("data");
// abs.relativize(rel);  // throws IllegalArgumentException
```

---

## Files Utility: File Operations

### Existence and Type Checks

```java
Path path = Path.of("config.json");

boolean exists      = Files.exists(path);
boolean notExists   = Files.notExists(path);
boolean isFile      = Files.isRegularFile(path);
boolean isDir       = Files.isDirectory(path);
boolean isSymlink   = Files.isSymbolicLink(path);
boolean isReadable  = Files.isReadable(path);
boolean isWritable  = Files.isWritable(path);
boolean isExecutable = Files.isExecutable(path);
boolean isHidden    = Files.isHidden(path);
boolean isSame      = Files.isSameFile(path1, path2);
```

### Copy, Move, Delete

```java
// ✅ Copy
Files.copy(source, target);
Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);
Files.copy(source, target, StandardCopyOption.COPY_ATTRIBUTES); // preserve timestamps

// ✅ Copy from/to streams
Files.copy(inputStream, targetPath, StandardCopyOption.REPLACE_EXISTING);
Files.copy(sourcePath, outputStream);

// ✅ Move (rename)
Files.move(source, target);
Files.move(source, target, StandardCopyOption.ATOMIC_MOVE);     // atomic rename

// ✅ Delete
Files.delete(path);                // throws NoSuchFileException if missing
Files.deleteIfExists(path);        // returns false if missing

// ✅ Create
Files.createFile(Path.of("new.txt"));
Files.createDirectory(Path.of("newdir"));
Files.createDirectories(Path.of("a/b/c/d"));       // mkdir -p
Path temp = Files.createTempFile("prefix-", ".tmp");
Path tempDir = Files.createTempDirectory("myapp-");
```

---

## File Attributes

```java
Path path = Path.of("document.pdf");

// ✅ Basic attributes
long size = Files.size(path);
FileTime lastModified = Files.getLastModifiedTime(path);
Files.setLastModifiedTime(path, FileTime.fromMillis(System.currentTimeMillis()));

// ✅ Read all basic attributes at once (single system call)
BasicFileAttributes attrs = Files.readAttributes(path, BasicFileAttributes.class);
attrs.size();
attrs.creationTime();
attrs.lastModifiedTime();
attrs.lastAccessTime();
attrs.isRegularFile();
attrs.isDirectory();
attrs.isSymbolicLink();

// ✅ POSIX attributes (Unix/Linux/macOS)
PosixFileAttributes posixAttrs = Files.readAttributes(path, PosixFileAttributes.class);
posixAttrs.owner();
posixAttrs.group();
posixAttrs.permissions();  // Set<PosixFilePermission>

// ✅ Set POSIX permissions
Set<PosixFilePermission> perms = PosixFilePermissions.fromString("rwxr-x---");
Files.setPosixFilePermissions(path, perms);

// ✅ File owner
UserPrincipal owner = Files.getOwner(path);
```

---

## Symbolic Links

```java
// ✅ Create a symbolic link
Path link = Path.of("link-to-config");
Path target = Path.of("/etc/myapp/config.json");
Files.createSymbolicLink(link, target);

// ✅ Read link target
Path linkTarget = Files.readSymbolicLink(link);

// ✅ Check if a path is a symbolic link
boolean isSymlink = Files.isSymbolicLink(link);

// ✅ Follow vs not follow symlinks
Files.exists(link);                                      // follows symlink
Files.exists(link, LinkOption.NOFOLLOW_LINKS);           // checks the link itself
Files.readAttributes(link, BasicFileAttributes.class,
    LinkOption.NOFOLLOW_LINKS);                           // attributes of the link
```

---

## Directory Traversal

```java
// ✅ List immediate children
try (Stream<Path> list = Files.list(Path.of("src"))) {
    list.forEach(System.out::println);
}

// ✅ Walk directory tree (recursive, depth-first)
try (Stream<Path> walk = Files.walk(Path.of("project"))) {
    List<Path> javaFiles = walk
        .filter(Files::isRegularFile)
        .filter(p -> p.toString().endsWith(".java"))
        .collect(Collectors.toList());
}

// ✅ Walk with max depth
try (Stream<Path> walk = Files.walk(Path.of("project"), 3)) {
    walk.forEach(System.out::println);
}

// ✅ Find files matching a BiPredicate
try (Stream<Path> found = Files.find(Path.of("project"), Integer.MAX_VALUE,
        (path, attrs) -> attrs.isRegularFile() && attrs.size() > 1_000_000)) {
    found.forEach(p -> System.out.println("Large file: " + p));
}

// ✅ DirectoryStream with glob pattern
try (DirectoryStream<Path> stream = Files.newDirectoryStream(
        Path.of("logs"), "*.log")) {
    for (Path entry : stream) {
        System.out.println(entry);
    }
}
```

---

## Practical Utility: Project File Manager

```java
public final class ProjectFiles {

    private ProjectFiles() {}

    // ✅ Find all Java source files
    public static List<Path> findJavaFiles(Path projectRoot) throws IOException {
        try (Stream<Path> walk = Files.walk(projectRoot)) {
            return walk
                .filter(Files::isRegularFile)
                .filter(p -> p.toString().endsWith(".java"))
                .sorted()
                .collect(Collectors.toList());
        }
    }

    // ✅ Calculate total directory size
    public static long directorySize(Path dir) throws IOException {
        try (Stream<Path> walk = Files.walk(dir)) {
            return walk
                .filter(Files::isRegularFile)
                .mapToLong(p -> {
                    try { return Files.size(p); }
                    catch (IOException e) { return 0L; }
                })
                .sum();
        }
    }

    // ✅ Safe recursive delete
    public static void deleteRecursively(Path dir) throws IOException {
        if (!Files.exists(dir)) return;
        try (Stream<Path> walk = Files.walk(dir)) {
            walk.sorted(Comparator.reverseOrder())
                .forEach(p -> {
                    try { Files.delete(p); }
                    catch (IOException e) { throw new UncheckedIOException(e); }
                });
        }
    }

    // ✅ Copy directory tree
    public static void copyDirectory(Path source, Path target) throws IOException {
        try (Stream<Path> walk = Files.walk(source)) {
            walk.forEach(src -> {
                Path dest = target.resolve(source.relativize(src));
                try {
                    if (Files.isDirectory(src)) {
                        Files.createDirectories(dest);
                    } else {
                        Files.copy(src, dest, StandardCopyOption.REPLACE_EXISTING);
                    }
                } catch (IOException e) {
                    throw new UncheckedIOException(e);
                }
            });
        }
    }
}
```

---

## Key Takeaways

1. **`Path.of()`** (Java 11+) is the preferred way to create paths — replaces `Paths.get()`
2. **`resolve()`** appends a child; **`relativize()`** computes the relative path between two paths
3. **`Files`** utility provides one-line methods for copy, move, delete, read, write, and traversal
4. **`Files.walk()`** and **`Files.find()`** return lazy `Stream<Path>` — always use try-with-resources
5. **`BasicFileAttributes`** reads all attributes in a single system call — efficient for multiple checks
6. **Always prefer `java.nio.file.Path`** over `java.io.File` in modern Java code

---

[← Previous: Q9 - Memory-Mapped Files](./q09-memory-mapped-files.md) | [Next: Q11 - File Watching →](./q11-file-watching.md)

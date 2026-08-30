# Q6: Design a Simple File System

## Question
Design and implement a simple in-memory File System in Java. Use the Composite pattern for the directory tree. Support file operations (create, delete, read, write, search), path resolution, and permissions.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Create files and directories at a given path
- Read and write file contents
- Delete files and directories (recursive)
- List directory contents
- Search for files by name (glob or substring)
- Path resolution (absolute paths like `/home/user/file.txt`)
- Basic permission model (read, write, execute)

**Non-Functional Requirements:**
- Thread-safe for concurrent file operations
- Efficient path lookup
- Clear separation between files and directories (Composite pattern)

### API Design

```java
public interface FileSystem {
    void mkdir(String path);
    void createFile(String path, String content);
    String readFile(String path);
    void writeFile(String path, String content);
    void delete(String path);
    List<String> list(String path);
    List<String> search(String directory, String namePattern);
    boolean exists(String path);
    FileInfo getInfo(String path);
}

public class FileInfo {
    private final String name;
    private final boolean isDirectory;
    private final long size;
    private final long createdAt;
    private final long modifiedAt;
    private final Set<Permission> permissions;
    // constructor, getters ...
}

public enum Permission { READ, WRITE, EXECUTE }
```

### Implementation

#### Composite Pattern: FileNode Hierarchy

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public abstract class FileNode {
    protected final String name;
    protected final long createdAt;
    protected long modifiedAt;
    protected final Set<Permission> permissions;

    public FileNode(String name) {
        this.name = name;
        this.createdAt = System.currentTimeMillis();
        this.modifiedAt = this.createdAt;
        this.permissions = EnumSet.of(Permission.READ, Permission.WRITE);
    }

    public String getName() { return name; }
    public long getCreatedAt() { return createdAt; }
    public long getModifiedAt() { return modifiedAt; }
    public Set<Permission> getPermissions() { return Collections.unmodifiableSet(permissions); }
    public void setPermissions(Set<Permission> perms) { this.permissions.clear(); this.permissions.addAll(perms); }

    public abstract boolean isDirectory();
    public abstract long getSize();
}

public class File extends FileNode {
    private String content;
    private final ReadWriteLock lock = new ReentrantReadWriteLock();

    public File(String name, String content) {
        super(name);
        this.content = content != null ? content : "";
    }

    @Override
    public boolean isDirectory() { return false; }

    @Override
    public long getSize() { return content.length(); }

    public String read() {
        lock.readLock().lock();
        try {
            return content;
        } finally {
            lock.readLock().unlock();
        }
    }

    public void write(String content) {
        lock.writeLock().lock();
        try {
            this.content = content;
            this.modifiedAt = System.currentTimeMillis();
        } finally {
            lock.writeLock().unlock();
        }
    }
}

public class Directory extends FileNode {
    private final ConcurrentHashMap<String, FileNode> children = new ConcurrentHashMap<>();

    public Directory(String name) {
        super(name);
    }

    @Override
    public boolean isDirectory() { return true; }

    @Override
    public long getSize() {
        return children.values().stream().mapToLong(FileNode::getSize).sum();
    }

    public void addChild(FileNode node) {
        children.put(node.getName(), node);
        this.modifiedAt = System.currentTimeMillis();
    }

    public FileNode getChild(String name) {
        return children.get(name);
    }

    public FileNode removeChild(String name) {
        this.modifiedAt = System.currentTimeMillis();
        return children.remove(name);
    }

    public Collection<FileNode> getChildren() {
        return Collections.unmodifiableCollection(children.values());
    }

    public boolean hasChild(String name) {
        return children.containsKey(name);
    }
}
```

#### FileSystem Implementation

```java
import java.util.*;
import java.util.stream.Collectors;

public class InMemoryFileSystem implements FileSystem {

    private final Directory root;

    public InMemoryFileSystem() {
        this.root = new Directory("/");
    }

    @Override
    public void mkdir(String path) {
        validatePath(path);
        String[] parts = splitPath(path);
        Directory current = root;

        for (String part : parts) {
            FileNode child = current.getChild(part);
            if (child == null) {
                Directory newDir = new Directory(part);
                current.addChild(newDir);
                current = newDir;
            } else if (child.isDirectory()) {
                current = (Directory) child;
            } else {
                throw new IllegalArgumentException("Path conflict: '" + part + "' is a file, not a directory");
            }
        }
    }

    @Override
    public void createFile(String path, String content) {
        validatePath(path);
        String[] parts = splitPath(path);
        Directory parent = navigateToParent(parts);
        String fileName = parts[parts.length - 1];

        if (parent.hasChild(fileName)) {
            throw new IllegalArgumentException("File already exists: " + path);
        }

        parent.addChild(new File(fileName, content));
    }

    @Override
    public String readFile(String path) {
        FileNode node = resolve(path);
        if (node == null || node.isDirectory()) {
            throw new IllegalArgumentException("Not a file: " + path);
        }
        if (!node.getPermissions().contains(Permission.READ)) {
            throw new SecurityException("No read permission: " + path);
        }
        return ((File) node).read();
    }

    @Override
    public void writeFile(String path, String content) {
        FileNode node = resolve(path);
        if (node == null) {
            // Auto-create
            createFile(path, content);
            return;
        }
        if (node.isDirectory()) {
            throw new IllegalArgumentException("Not a file: " + path);
        }
        if (!node.getPermissions().contains(Permission.WRITE)) {
            throw new SecurityException("No write permission: " + path);
        }
        ((File) node).write(content);
    }

    @Override
    public void delete(String path) {
        if ("/".equals(path)) throw new IllegalArgumentException("Cannot delete root");

        String[] parts = splitPath(path);
        Directory parent = navigateToParent(parts);
        String name = parts[parts.length - 1];

        FileNode removed = parent.removeChild(name);
        if (removed == null) {
            throw new IllegalArgumentException("Path not found: " + path);
        }
        // Recursive cleanup is implicit — children become unreferenced and GC'd
    }

    @Override
    public List<String> list(String path) {
        FileNode node = resolve(path);
        if (node == null || !node.isDirectory()) {
            throw new IllegalArgumentException("Not a directory: " + path);
        }
        return ((Directory) node).getChildren().stream()
            .map(child -> child.getName() + (child.isDirectory() ? "/" : ""))
            .sorted()
            .collect(Collectors.toList());
    }

    @Override
    public List<String> search(String directory, String namePattern) {
        FileNode node = resolve(directory);
        if (node == null || !node.isDirectory()) {
            throw new IllegalArgumentException("Not a directory: " + directory);
        }

        List<String> results = new ArrayList<>();
        searchRecursive((Directory) node, directory, namePattern, results);
        return results;
    }

    private void searchRecursive(Directory dir, String currentPath, String pattern, List<String> results) {
        for (FileNode child : dir.getChildren()) {
            String childPath = currentPath.endsWith("/")
                ? currentPath + child.getName()
                : currentPath + "/" + child.getName();

            if (matchesPattern(child.getName(), pattern)) {
                results.add(childPath);
            }

            if (child.isDirectory()) {
                searchRecursive((Directory) child, childPath, pattern, results);
            }
        }
    }

    private boolean matchesPattern(String name, String pattern) {
        // Simple glob: * matches any sequence, ? matches single char
        String regex = pattern
            .replace(".", "\\.")
            .replace("*", ".*")
            .replace("?", ".");
        return name.matches(regex);
    }

    @Override
    public boolean exists(String path) {
        return resolve(path) != null;
    }

    @Override
    public FileInfo getInfo(String path) {
        FileNode node = resolve(path);
        if (node == null) throw new IllegalArgumentException("Path not found: " + path);
        return new FileInfo(node.getName(), node.isDirectory(), node.getSize(),
                            node.getCreatedAt(), node.getModifiedAt(), node.getPermissions());
    }

    // --- Path utilities ---

    private FileNode resolve(String path) {
        if ("/".equals(path)) return root;
        String[] parts = splitPath(path);
        FileNode current = root;

        for (String part : parts) {
            if (!current.isDirectory()) return null;
            current = ((Directory) current).getChild(part);
            if (current == null) return null;
        }
        return current;
    }

    private Directory navigateToParent(String[] parts) {
        Directory current = root;
        for (int i = 0; i < parts.length - 1; i++) {
            FileNode child = current.getChild(parts[i]);
            if (child == null || !child.isDirectory()) {
                throw new IllegalArgumentException("Parent directory not found: " + parts[i]);
            }
            current = (Directory) child;
        }
        return current;
    }

    private String[] splitPath(String path) {
        return Arrays.stream(path.split("/"))
            .filter(s -> !s.isEmpty())
            .toArray(String[]::new);
    }

    private void validatePath(String path) {
        if (path == null || path.isEmpty() || !path.startsWith("/")) {
            throw new IllegalArgumentException("Path must be absolute (start with /): " + path);
        }
    }
}
```

### Concurrency Considerations

| Concern | Solution |
|---------|----------|
| Directory child mutations | `ConcurrentHashMap` in each `Directory` |
| File content reads/writes | `ReadWriteLock` per `File` — allows concurrent reads |
| Path traversal during mutation | Each node independently thread-safe |
| Directory listing consistency | Snapshot via `unmodifiableCollection` |
| Recursive delete | Children become unreferenced; no dangling pointers |

### Performance Analysis

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `resolve(path)` | O(D) | D = depth of path |
| `mkdir` | O(D) | Creates intermediate directories |
| `createFile` | O(D) | Navigate to parent + O(1) add |
| `readFile` / `writeFile` | O(D) | Path resolution + O(1) read/write |
| `list` | O(D + C) | D = depth, C = children count |
| `search` | O(N) | N = total nodes in subtree |
| `delete` | O(D) | Path resolution + O(1) remove |

### Testing Strategy

```java
public class FileSystemTest {

    @Test
    void testCreateAndReadFile() {
        InMemoryFileSystem fs = new InMemoryFileSystem();
        fs.mkdir("/home/user");
        fs.createFile("/home/user/hello.txt", "Hello, World!");

        assertEquals("Hello, World!", fs.readFile("/home/user/hello.txt"));
    }

    @Test
    void testDirectoryListing() {
        InMemoryFileSystem fs = new InMemoryFileSystem();
        fs.mkdir("/docs");
        fs.createFile("/docs/a.txt", "A");
        fs.createFile("/docs/b.txt", "B");
        fs.mkdir("/docs/sub");

        List<String> listing = fs.list("/docs");
        assertEquals(3, listing.size());
        assertTrue(listing.contains("sub/"));
        assertTrue(listing.contains("a.txt"));
    }

    @Test
    void testSearchByPattern() {
        InMemoryFileSystem fs = new InMemoryFileSystem();
        fs.mkdir("/project/src");
        fs.createFile("/project/src/Main.java", "class Main {}");
        fs.createFile("/project/src/Test.java", "class Test {}");
        fs.createFile("/project/README.md", "# README");

        List<String> javaFiles = fs.search("/project", "*.java");
        assertEquals(2, javaFiles.size());
    }

    @Test
    void testDeleteDirectory() {
        InMemoryFileSystem fs = new InMemoryFileSystem();
        fs.mkdir("/tmp/cache");
        fs.createFile("/tmp/cache/data.bin", "binary data");

        fs.delete("/tmp/cache");
        assertFalse(fs.exists("/tmp/cache"));
        assertFalse(fs.exists("/tmp/cache/data.bin"));
    }

    @Test
    void testPermissions() {
        InMemoryFileSystem fs = new InMemoryFileSystem();
        fs.createFile("/secret.txt", "top secret");

        FileInfo info = fs.getInfo("/secret.txt");
        assertTrue(info.getPermissions().contains(Permission.READ));
    }

    @Test
    void testConcurrentFileAccess() throws Exception {
        InMemoryFileSystem fs = new InMemoryFileSystem();
        fs.createFile("/shared.txt", "initial");

        ExecutorService exec = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 100; i++) {
            final int val = i;
            exec.submit(() -> fs.writeFile("/shared.txt", "value-" + val));
        }
        exec.shutdown();
        exec.awaitTermination(5, TimeUnit.SECONDS);

        String content = fs.readFile("/shared.txt");
        assertTrue(content.startsWith("value-"));
    }
}
```

---

## Interviewer Insight
Interviewers want to see correct application of the Composite pattern (File and Directory as tree nodes). They look for clean path resolution, recursive operations, and thread safety at the node level. Discuss how a real file system handles inodes, block allocation, and journaling — even though the in-memory version abstracts these away.

## Follow-up Questions
1. How would you add symbolic links (symlinks) to this file system?
2. How would you persist this file system to disk efficiently?
3. How would you implement file locking for exclusive access?

## Related Topics
- [OOP Concepts](../02-oop-concepts/README.md)
- [Collections Framework](../03-collections-framework/README.md)
- [Java I/O & NIO](../08-java-io-nio/README.md)

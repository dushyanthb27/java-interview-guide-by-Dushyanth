# Q4 🟡 What Is Try-With-Resources?

**Question:** What is try-with-resources in Java? How does it work and how does it compare to traditional try-finally?

---

## Answer

### The AutoCloseable Interface

```java
// Any class implementing AutoCloseable can be used with try-with-resources
public interface AutoCloseable {
    void close() throws Exception;
}

// Closeable extends AutoCloseable (for I/O classes)
public interface Closeable extends AutoCloseable {
    void close() throws IOException;
}
```

---

### Basic Syntax

```java
// ✅ Try-with-resources — resource automatically closed
try (FileReader reader = new FileReader("data.txt")) {
    int ch;
    while ((ch = reader.read()) != -1) {
        System.out.print((char) ch);
    }
}
// reader.close() is called automatically here
// Even if an exception is thrown inside the try block
```

---

### Traditional Try-Finally vs Try-With-Resources

```java
// ❌ Traditional approach — verbose and error-prone
BufferedReader br = null;
try {
    br = new BufferedReader(new FileReader("data.txt"));
    String line = br.readLine();
    System.out.println(line);
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (br != null) {
        try {
            br.close();     // close() can also throw!
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

// ✅ Try-with-resources — clean and safe
try (BufferedReader br = new BufferedReader(new FileReader("data.txt"))) {
    String line = br.readLine();
    System.out.println(line);
} catch (IOException e) {
    e.printStackTrace();
}
// br is automatically closed — even on exception
```

> **🎯 Interviewer Insight:** The traditional approach has three problems. First, verbose boilerplate. Second, resource variable must be declared outside try. Third, close() exceptions can mask the original exception. Try-with-resources solves all three.

---

### Multiple Resources — Closed in Reverse Order

```java
// ✅ Multiple resources separated by semicolons
try (
    FileInputStream fis = new FileInputStream("input.bin");
    BufferedInputStream bis = new BufferedInputStream(fis);
    FileOutputStream fos = new FileOutputStream("output.bin");
    BufferedOutputStream bos = new BufferedOutputStream(fos)
) {
    byte[] buffer = new byte[1024];
    int bytesRead;
    while ((bytesRead = bis.read(buffer)) != -1) {
        bos.write(buffer, 0, bytesRead);
    }
}
// Closing order: bos → fos → bis → fis (REVERSE of declaration)
```

#### Demonstrating Reverse Close Order

```java
public class ReverseCloseDemo {

    static class Resource implements AutoCloseable {
        private final String name;

        Resource(String name) {
            this.name = name;
            System.out.println("Opening: " + name);
        }

        @Override
        public void close() {
            System.out.println("Closing: " + name);
        }
    }

    public static void main(String[] args) {
        try (
            Resource a = new Resource("A");
            Resource b = new Resource("B");
            Resource c = new Resource("C")
        ) {
            System.out.println("Using resources");
        }
    }
}
// Output:
// Opening: A
// Opening: B
// Opening: C
// Using resources
// Closing: C    ← reverse order
// Closing: B
// Closing: A
```

> **🎯 Interviewer Insight:** Resources close in **reverse declaration order** (LIFO). This matters when resources depend on each other. For example, `BufferedInputStream` wraps `FileInputStream` — the buffer must close first to flush, then the underlying stream.

---

### Effectively Final Variables (Java 9+)

```java
// Java 7-8: resource must be declared inside try()
BufferedReader br = new BufferedReader(new FileReader("data.txt"));
try (BufferedReader wrapper = br) {   // ❌ Ugly workaround
    // use wrapper
}

// ✅ Java 9+: effectively final variables can be used directly
BufferedReader br = new BufferedReader(new FileReader("data.txt"));
try (br) {   // br is effectively final — OK in Java 9+
    String line = br.readLine();
    System.out.println(line);
}

// ✅ Multiple effectively final resources (Java 9+)
Connection conn = dataSource.getConnection();
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users");
ResultSet rs = ps.executeQuery();
try (conn; ps; rs) {
    while (rs.next()) {
        System.out.println(rs.getString("name"));
    }
}
```

---

### Creating Custom AutoCloseable

```java
// ✅ Custom resource with try-with-resources support
public class DatabaseConnectionPool implements AutoCloseable {

    private final List<Connection> connections = new ArrayList<>();
    private boolean closed = false;

    public DatabaseConnectionPool(int size) throws SQLException {
        for (int i = 0; i < size; i++) {
            connections.add(DriverManager.getConnection("jdbc:h2:mem:test"));
        }
        System.out.println("Pool created with " + size + " connections");
    }

    public Connection getConnection() {
        if (closed) {
            throw new IllegalStateException("Pool is closed");
        }
        return connections.remove(0);
    }

    @Override
    public void close() {
        closed = true;
        for (Connection conn : connections) {
            try {
                conn.close();
            } catch (SQLException e) {
                // Log but continue closing others
                System.err.println("Error closing connection: " + e.getMessage());
            }
        }
        connections.clear();
        System.out.println("Pool closed");
    }
}

// Usage:
try (DatabaseConnectionPool pool = new DatabaseConnectionPool(5)) {
    Connection conn = pool.getConnection();
    // work with connection
}
// Pool is automatically closed
```

> **🏭 Production Consideration:** When implementing `AutoCloseable`, make `close()` idempotent (safe to call multiple times). Never let it throw — log and swallow cleanup exceptions to avoid masking the primary exception.

---

### Resource Leak Prevention

```java
// ❌ Resource leak — exception during construction of second resource
try {
    InputStream is = new FileInputStream("input.txt");      // Opens successfully
    OutputStream os = new FileOutputStream("/invalid/path"); // Throws!
    // is is NEVER closed!
} catch (IOException e) {
    e.printStackTrace();
}

// ✅ Try-with-resources prevents leaks
try (
    InputStream is = new FileInputStream("input.txt");       // Opens successfully
    OutputStream os = new FileOutputStream("/invalid/path")  // Throws!
) {
    // ...
}
// If os construction fails, is.close() is still called automatically

// ❌ Nested constructors can also leak
try (BufferedInputStream bis = new BufferedInputStream(
        new FileInputStream("input.txt"))) {
    // If BufferedInputStream constructor throws (very unlikely),
    // FileInputStream is leaked because it was never assigned to a variable
}

// ✅ Safest: declare each resource separately
try (
    FileInputStream fis = new FileInputStream("input.txt");
    BufferedInputStream bis = new BufferedInputStream(fis)
) {
    // Both are tracked and will be closed
}
```

> **⚠️ Common Mistake:** Nesting resource construction (`new BufferedInputStream(new FileInputStream(...))`). If the outer constructor throws, the inner resource is leaked. 

**Always declare each resource separately** in the try-with-resources header.

---

### Try-With-Resources and Suppressed Exceptions

```java
// When both try body and close() throw, close()'s exception is SUPPRESSED
public class FailingResource implements AutoCloseable {
    @Override
    public void close() {
        throw new RuntimeException("Close failed");
    }
}

try (FailingResource res = new FailingResource()) {
    throw new RuntimeException("Body failed");
} catch (RuntimeException e) {
    System.out.println("Primary: " + e.getMessage());      // "Body failed"
    System.out.println("Suppressed: " +
        e.getSuppressed()[0].getMessage());                 // "Close failed"
}
// The try body exception is PRIMARY
// The close() exception is SUPPRESSED (not lost!)
```

> **🎯 Interviewer Insight:** This is a key difference from try-finally. In try-finally, an exception in `finally` **replaces** the original. 

In try-with-resources, the close() exception becomes a **suppressed exception** — accessible via `getSuppressed()`. The primary exception is never lost.

---

### Common Use Cases

```java
// ✅ File I/O
try (var reader = new BufferedReader(new FileReader("data.csv"))) {
    reader.lines().forEach(System.out::println);
}

// ✅ JDBC
try (
    Connection conn = DriverManager.getConnection(url);
    PreparedStatement ps = conn.prepareStatement("SELECT * FROM users");
    ResultSet rs = ps.executeQuery()
) {
    while (rs.next()) {
        System.out.println(rs.getString("name"));
    }
}

// ✅ Locks (with wrapper)
public class CloseableLock implements AutoCloseable {
    private final Lock lock;

    public CloseableLock(Lock lock) {
        this.lock = lock;
        lock.lock();
    }

    @Override
    public void close() {
        lock.unlock();
    }
}

try (var ignored = new CloseableLock(myLock)) {
    // Critical section — lock auto-released
}

// ✅ HTTP connections (Java 11+)
try (var client = HttpClient.newHttpClient()) {
    // Note: HttpClient does not implement AutoCloseable until Java 21
}
```

---

### Anti-Patterns

```java
// ❌ Using try-with-resources with non-closeable resource
try (String s = "hello") { }  // Compile error: String not AutoCloseable

// ❌ Reassigning the resource variable
try (InputStream is = new FileInputStream("a.txt")) {
    is = new FileInputStream("b.txt"); // Compile error: resource variable is final
}

// ❌ Returning the resource from inside try-with-resources
public InputStream getStream() {
    try (InputStream is = new FileInputStream("data.txt")) {
        return is; // BAD! Stream is closed when try block exits
    }
}

// ✅ If you need to return a resource, don't use try-with-resources
public InputStream getStream() throws IOException {
    return new FileInputStream("data.txt"); // Caller is responsible for closing
}
```

> **⚠️ Common Mistake:** Returning a resource from inside try-with-resources. The resource is closed when the try block exits, so the caller gets a closed stream. 

If you need to return a resource, manage its lifecycle manually (or use a pool).

---

## 🎯 Interviewer Insight

Interviewers test these specific areas:

1. **Reverse close order** — LIFO, matters for dependent resources
2. **Suppressed exceptions** — primary vs suppressed, `getSuppressed()`
3. **Resource leak prevention** — construction failures, nested constructors
4. **Java 9 effectively final** — cleaner syntax for pre-existing resources
5. **Custom AutoCloseable** — idempotent close(), don't throw from close()
6. **Returning resources** — don't return from inside try-with-resources

---

## ⚡ Follow-up Questions

> **⚡ Follow-up:** What happens if an exception is thrown in the try block AND in the close() method?

> **⚡ Follow-up:** Why are resources closed in reverse order?

> **⚡ Follow-up:** How does try-with-resources handle exceptions during resource construction?

> **⚡ Follow-up:** Can you use try-with-resources with a resource that doesn't implement AutoCloseable?

> **⚡ Follow-up:** What changed in Java 9 for try-with-resources?

---

## 📚 Related Topics

- [Try-Catch-Finally](./03-try-catch-finally.md)
- [Custom Exceptions →](./05-custom-exceptions.md)
- [Suppressed Exceptions →](./09-suppressed-exceptions.md)
- [Exception Best Practices →](./06-exception-best-practices.md)

\n
### ⚠️ Gotchas

- **Memory Leaks & Object Reference Retention**: Unintentional object retention via static references, unclosed resource handles, or long-lived caches preventing Garbage Collection.
- **Concurrency & Thread Safety**: Assuming thread safety on standard collections (`ArrayList`, `HashMap`) or primitives, leading to race conditions and data corruption.
- **Exception Masking & Swallowing**: Catching generic `Exception` or `Throwable` without proper logging, rethrowing, or handling suppressed exceptions in try-with-resources.
- **Performance Antipatterns**: Excessive object allocation in tight loops, improper stream boxing/unboxing overhead, or inefficient synchronization blocks causing thread contention.
\n---\n

[← Back to Section Index](./README.md) · [Previous: Try-Catch-Finally ←](./03-try-catch-finally.md) · [Next: Custom Exceptions →](./05-custom-exceptions.md)

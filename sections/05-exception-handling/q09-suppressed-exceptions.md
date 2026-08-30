# Q9: What Are Suppressed Exceptions?

[Home](../../README.md) > [Exception Handling](./README.md) > Suppressed Exceptions

---

## Difficulty: 🟠 Hard

## Question
> What are suppressed exceptions in Java? How does try-with-resources handle them, and how do they differ from the pre-Java 7 behavior?

---

## Answer

### The Problem Before Java 7 (Lost Exceptions)

```java
// ❌ Java 6 behavior: close() exception REPLACES the try body exception
public void java6Problem() throws Exception {
    FileInputStream fis = null;
    try {
        fis = new FileInputStream("data.txt");
        processStream(fis);     // throws IOException("Read failed")
    } finally {
        if (fis != null) {
            fis.close();         // throws IOException("Close failed")
        }
    }
    // Only "Close failed" propagates!
    // "Read failed" is COMPLETELY LOST - impossible to debug
}
```

---

### Java 7+ Solution: Suppressed Exceptions

With try-with-resources, the **try body exception is primary** and any **close() exceptions are suppressed** (attached to the primary exception, not lost).

```java
// ✅ Java 7+: Both exceptions are preserved
public class SuppressedExceptionDemo {

    static class FailingResource implements AutoCloseable {
        @Override
        public void close() {
            throw new RuntimeException("Close failed");
        }
    }

    public static void main(String[] args) {
        try (FailingResource res = new FailingResource()) {
            throw new RuntimeException("Body failed");  // PRIMARY exception
        } catch (RuntimeException e) {
            System.out.println("Primary: " + e.getMessage());
            // Output: Primary: Body failed

            Throwable[] suppressed = e.getSuppressed();
            for (Throwable s : suppressed) {
                System.out.println("Suppressed: " + s.getMessage());
                // Output: Suppressed: Close failed
            }
        }
    }
}
```

---

### Multiple Suppressed Exceptions

```java
// When multiple resources fail to close, ALL close exceptions are suppressed

public class MultiSuppressedDemo {

    static class Resource implements AutoCloseable {
        private final String name;

        Resource(String name) { this.name = name; }

        @Override
        public void close() {
            throw new RuntimeException(name + " close failed");
        }
    }

    public static void main(String[] args) {
        try (
            Resource a = new Resource("A");
            Resource b = new Resource("B");
            Resource c = new Resource("C")
        ) {
            throw new RuntimeException("Body failed");
        } catch (RuntimeException e) {
            System.out.println("Primary: " + e.getMessage());

            // Resources close in REVERSE order: C, B, A
            for (Throwable suppressed : e.getSuppressed()) {
                System.out.println("Suppressed: " + suppressed.getMessage());
            }
        }
    }
}
// Output:
// Primary: Body failed
// Suppressed: C close failed
// Suppressed: B close failed
// Suppressed: A close failed
```

---

### When Only close() Throws (No Body Exception)

```java
// If the try body does NOT throw, the close() exception becomes the PRIMARY
public class CloseOnlyThrowsDemo {

    static class FailingResource implements AutoCloseable {
        @Override
        public void close() {
            throw new RuntimeException("Close failed");
        }
    }

    public static void main(String[] args) {
        try (FailingResource res = new FailingResource()) {
            System.out.println("Body succeeds");
            // No exception here
        } catch (RuntimeException e) {
            System.out.println("Caught: " + e.getMessage());
            // Output: Caught: Close failed
            // close() exception is now the primary (not suppressed)
        }
    }
}
```

---

### addSuppressed() and getSuppressed()

```java
// ✅ You can manually add suppressed exceptions
public void processMultipleFiles(List<String> files) {
    Exception primaryException = null;

    for (String file : files) {
        try {
            processFile(file);
        } catch (Exception e) {
            if (primaryException == null) {
                primaryException = e;          // First exception is primary
            } else {
                primaryException.addSuppressed(e); // Others are suppressed
            }
        }
    }

    if (primaryException != null) {
        throw new BatchProcessingException(
            "Failed to process " + files.size() + " files", primaryException);
    }
}

// Usage:
try {
    processMultipleFiles(List.of("a.txt", "b.txt", "c.txt"));
} catch (BatchProcessingException e) {
    System.out.println("Primary: " + e.getCause().getMessage());
    for (Throwable suppressed : e.getCause().getSuppressed()) {
        System.out.println("Also failed: " + suppressed.getMessage());
    }
}
```

---

### Custom AutoCloseable That Throws

```java
public class TransactionManager implements AutoCloseable {

    private final Connection connection;
    private boolean committed = false;

    public TransactionManager(Connection connection) throws SQLException {
        this.connection = connection;
        connection.setAutoCommit(false);
        System.out.println("Transaction started");
    }

    public void commit() throws SQLException {
        connection.commit();
        committed = true;
        System.out.println("Transaction committed");
    }

    @Override
    public void close() throws SQLException {
        try {
            if (!committed) {
                System.out.println("Rolling back (not committed)");
                connection.rollback();
            }
        } finally {
            connection.setAutoCommit(true);
        }
    }
}

// ✅ Usage with try-with-resources
try (TransactionManager tx = new TransactionManager(conn)) {
    insertOrder(conn, order);
    insertOrderItems(conn, order.getItems());
    tx.commit();
}
// If insertOrderItems throws:
//   1. The exception from insertOrderItems is PRIMARY
//   2. close() is called -> rollback occurs
//   3. If rollback also fails, that exception is SUPPRESSED
```

---

### How It Looks in the Stack Trace

```
java.lang.RuntimeException: Body failed
    at com.example.Demo.main(Demo.java:15)
    Suppressed: java.lang.RuntimeException: Close C failed
        at com.example.Resource.close(Resource.java:8)
        at com.example.Demo.main(Demo.java:14)
    Suppressed: java.lang.RuntimeException: Close B failed
        at com.example.Resource.close(Resource.java:8)
        at com.example.Demo.main(Demo.java:14)
    Suppressed: java.lang.RuntimeException: Close A failed
        at com.example.Resource.close(Resource.java:8)
        at com.example.Demo.main(Demo.java:14)
```

---

### Comparison: Java 6 vs Java 7+

```java
// ===== Java 6: Manual try-finally (exception lost) =====
Resource r = new Resource();
try {
    r.doWork();         // throws ExceptionA
} finally {
    r.close();          // throws ExceptionB
}
// Result: Only ExceptionB propagates. ExceptionA is LOST.

// ===== Java 6: Workaround to preserve both =====
Resource r = new Resource();
Exception primary = null;
try {
    r.doWork();
} catch (Exception e) {
    primary = e;
    throw e;
} finally {
    if (primary != null) {
        try {
            r.close();
        } catch (Exception e) {
            // Log it, but can't attach it to primary in Java 6
            System.err.println("Close also failed: " + e);
        }
    } else {
        r.close();
    }
}

// ===== Java 7+: Try-with-resources (both preserved automatically) =====
try (Resource r = new Resource()) {
    r.doWork();         // throws ExceptionA (PRIMARY)
}                        // close() throws ExceptionB (SUPPRESSED)
// ExceptionA propagates with ExceptionB attached as suppressed
```

---

### Disabling Suppressed Exceptions

```java
// Throwable has a protected constructor to disable suppression
// Used internally by the JVM for optimization

public class LightweightException extends Exception {

    // writableStackTrace=false and enableSuppression=false
    public LightweightException(String message) {
        super(message, null, false, false);
        // No stack trace, no suppressed exceptions
        // Useful for control flow exceptions (though generally discouraged)
    }
}
```

---

### Anti-Patterns

```java
// ❌ Ignoring suppressed exceptions in logging
try {
    processWithResources();
} catch (Exception e) {
    log.error("Failed: " + e.getMessage()); // Suppressed exceptions not logged!
}

// ✅ Log suppressed exceptions too
try {
    processWithResources();
} catch (Exception e) {
    log.error("Failed: " + e.getMessage(), e); // Stack trace includes suppressed
    // Or explicitly:
    for (Throwable suppressed : e.getSuppressed()) {
        log.warn("Suppressed: " + suppressed.getMessage(), suppressed);
    }
}

// ❌ Using return in finally (swallows ALL exceptions including suppressed)
try (Resource r = new Resource()) {
    throw new Exception("Important error");
} finally {
    return defaultValue; // EVERYTHING is lost - don't do this!
}

// ❌ Catching and re-throwing in finally (replaces suppression mechanism)
try (Resource r = new Resource()) {
    r.doWork();
} catch (Exception e) {
    // Let try-with-resources handle suppression automatically
    throw e;
}
```

---

## Interview Tips
- Explain the problem that suppressed exceptions solve (Java 6 lost exceptions)
- Know that try body exception is **primary** and close() exception is **suppressed**
- Mention `getSuppressed()` and `addSuppressed()` methods
- Explain that if only close() throws, its exception becomes the primary (not suppressed)
- Know that suppressed exceptions are visible in stack traces under the `Suppressed:` label

---

[<< Previous: Q8 - Exception Chaining](./q08-exception-chaining.md) | [Next: Q10 - Exceptions in Streams >>](./q10-exceptions-in-streams.md)

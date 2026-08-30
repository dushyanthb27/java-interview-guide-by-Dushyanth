# Q4: What Is Try-With-Resources?

[Home](../../README.md) > [Exception Handling](./README.md) > Try-With-Resources

---

## Difficulty: 🟡 Medium

## Question
> What is try-with-resources in Java? How does it work and how does it compare to traditional try-finally?

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
// ✅ Try-with-resources - resource automatically closed
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
// ❌ Traditional approach - verbose and error-prone
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

// ✅ Try-with-resources - clean and safe
try (BufferedReader br = new BufferedReader(new FileReader("data.txt"))) {
    String line = br.readLine();
    System.out.println(line);
} catch (IOException e) {
    e.printStackTrace();
}
// br is automatically closed - even on exception
```

---

### Multiple Resources (Closed in Reverse Order)

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
// Closing order: bos -> fos -> bis -> fis (REVERSE of declaration)
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
// Closing: C    <-- reverse order
// Closing: B
// Closing: A
```

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
try (br) {   // br is effectively final - OK in Java 9+
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

---

### Resource Leak Prevention

```java
// ❌ Resource leak - exception during construction of second resource
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
    // Critical section - lock auto-released
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

---

## Interview Tips
- Know that resources are closed in **reverse** declaration order
- Explain the suppressed exceptions mechanism (body exception is primary)
- Mention Java 9's effectively final variable enhancement
- Emphasize resource leak prevention during construction failures
- Never return from inside try-with-resources if the return value IS the resource

---

[<< Previous: Q3 - Try-Catch-Finally](./q03-try-catch-finally.md) | [Next: Q5 - Custom Exceptions >>](./q05-custom-exceptions.md)

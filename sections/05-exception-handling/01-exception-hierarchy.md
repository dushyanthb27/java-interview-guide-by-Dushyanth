# Q1: Explain the Java Exception Hierarchy

[Home](../../README.md) > [Exception Handling](./README.md) > Exception Hierarchy

---

## Difficulty: 🟢 Easy

## Question
> Explain the Java exception class hierarchy. What are the main branches, and how are they classified?

---

## Answer

### The Complete Hierarchy Diagram

```
                        java.lang.Object
                              |
                      java.lang.Throwable
                       /              \
               java.lang.Error    java.lang.Exception
                  /     \              /            \
        OutOfMemory  StackOverflow  RuntimeException  IOException
        Error        Error            /      \          /       \
                                NullPointer  ClassCast FileNot   Socket
                                Exception    Exception Found     Exception
                                                       Exception
```

### Core Classes Explained

#### `Throwable` - The Root

```java
// Throwable is the superclass of ALL errors and exceptions
public class Throwable implements Serializable {
    private String message;
    private Throwable cause;
    private StackTraceElement[] stackTrace;
    private List<Throwable> suppressedExceptions;
}
```

#### `Error` - JVM-Level Problems (Do NOT Catch)

```java
// Errors indicate serious problems that applications should NOT catch
// Examples:
OutOfMemoryError        // JVM ran out of heap space
StackOverflowError      // Infinite recursion
NoClassDefFoundError    // Class missing at runtime
VirtualMachineError     // JVM is broken or out of resources
```

#### `Exception` - Application-Level Problems

```java
// Checked Exceptions (must handle or declare)
IOException
SQLException
ClassNotFoundException
InterruptedException
ReflectiveOperationException

// Unchecked Exceptions (RuntimeException subclasses)
NullPointerException
IllegalArgumentException
IllegalStateException
IndexOutOfBoundsException
ClassCastException
UnsupportedOperationException
ArithmeticException
ConcurrentModificationException
```

---

### Classification Summary

| Category | Superclass | Compile-Time Check | Typical Cause |
|----------|-----------|-------------------|---------------|
| **Error** | `Error` | No | JVM / system failure |
| **Checked Exception** | `Exception` (not `RuntimeException`) | Yes | Recoverable external condition |
| **Unchecked Exception** | `RuntimeException` | No | Programming bug |

---

### Demonstrating the Hierarchy

```java
public class ExceptionHierarchyDemo {

    public static void main(String[] args) {

        // ✅ Checking hierarchy at runtime
        System.out.println(new IOException() instanceof Exception);          // true
        System.out.println(new IOException() instanceof Throwable);          // true
        System.out.println(new RuntimeException() instanceof Exception);     // true
        System.out.println(new OutOfMemoryError() instanceof Error);         // true
        System.out.println(new OutOfMemoryError() instanceof Exception);     // false

        // ✅ instanceof checks for catch block matching
        try {
            throw new FileNotFoundException("config.yml");
        } catch (IOException e) {
            // FileNotFoundException IS-A IOException, so this catches it
            System.out.println("Caught: " + e.getClass().getSimpleName());
        }
    }
}
```

---

### Key Exception Classes You Must Know

#### Common Unchecked Exceptions (RuntimeException subclasses)

```java
// NullPointerException - accessing member on null reference
String s = null;
s.length(); // throws NullPointerException

// IllegalArgumentException - invalid method argument
public void setAge(int age) {
    if (age < 0) throw new IllegalArgumentException("Age cannot be negative: " + age);
    this.age = age;
}

// IllegalStateException - method called at wrong time
public void start() {
    if (running) throw new IllegalStateException("Already running");
    running = true;
}

// IndexOutOfBoundsException - invalid index
List<String> list = List.of("a", "b");
list.get(5); // throws IndexOutOfBoundsException

// ClassCastException - invalid cast
Object obj = "hello";
Integer num = (Integer) obj; // throws ClassCastException

// UnsupportedOperationException - operation not supported
List<String> immutable = List.of("a", "b");
immutable.add("c"); // throws UnsupportedOperationException

// ArithmeticException - illegal arithmetic operation
int result = 10 / 0; // throws ArithmeticException
```

#### Common Checked Exceptions

```java
// IOException - I/O operation failure
Files.readString(Path.of("missing.txt")); // throws IOException

// ClassNotFoundException - class not found by name
Class.forName("com.example.NonExistent"); // throws ClassNotFoundException

// InterruptedException - thread interrupted
Thread.sleep(1000); // throws InterruptedException

// SQLException - database access error
connection.executeQuery("INVALID SQL"); // throws SQLException
```

---

### Hierarchy Traversal with getCause()

```java
public class HierarchyTraversalDemo {

    public static void printExceptionChain(Throwable t) {
        Throwable current = t;
        int depth = 0;

        while (current != null) {
            String indent = "  ".repeat(depth);
            System.out.printf("%s[%s] %s: %s%n",
                indent,
                getCategory(current),
                current.getClass().getSimpleName(),
                current.getMessage());
            current = current.getCause();
            depth++;
        }
    }

    private static String getCategory(Throwable t) {
        if (t instanceof Error) return "ERROR";
        if (t instanceof RuntimeException) return "UNCHECKED";
        if (t instanceof Exception) return "CHECKED";
        return "THROWABLE";
    }

    public static void main(String[] args) {
        Exception chain = new RuntimeException("Service failed",
            new IOException("Connection refused",
                new IllegalArgumentException("Invalid port: -1")));

        printExceptionChain(chain);
        // [UNCHECKED] RuntimeException: Service failed
        //   [CHECKED] IOException: Connection refused
        //     [UNCHECKED] IllegalArgumentException: Invalid port: -1
    }
}
```

---

### Anti-Patterns

```java
// ❌ Catching Throwable - too broad, catches Errors too
try {
    riskyOperation();
} catch (Throwable t) {
    // This catches OutOfMemoryError, StackOverflowError, etc.
    log.error("Something went wrong", t);
}

// ❌ Catching Exception when you mean a specific type
try {
    readFile();
} catch (Exception e) {
    // Catches NullPointerException, ClassCastException, etc.
    System.out.println("File error");
}

// ✅ Catch specific exception types
try {
    readFile();
} catch (FileNotFoundException e) {
    System.out.println("File not found: " + e.getMessage());
} catch (IOException e) {
    System.out.println("I/O error: " + e.getMessage());
}
```

---

## Interview Tips
- Draw the hierarchy tree: `Throwable` -> `Error` + `Exception` -> `RuntimeException`
- Emphasize that `Error` should NOT be caught in application code
- Know at least 5 common `RuntimeException` subclasses by heart
- Mention that `Throwable` implements `Serializable` (relevant for RMI / distributed systems)
- Understand that catch blocks match via `instanceof` (polymorphic)

---

[Next: Q2 - Checked vs Unchecked Exceptions >>](./q02-checked-vs-unchecked.md)

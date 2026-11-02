# Q7 🟡 Error vs Exception — What's the Difference?

**Question:** What is the difference between `Error` and `Exception` in Java? When should you catch each, and why?

---

## Answer

### Key Differences

| Feature | `Error` | `Exception` |
|---------|---------|-------------|
| **Package** | `java.lang.Error` | `java.lang.Exception` |
| **Represents** | JVM/system-level failure | Application-level problem |
| **Recoverable?** | Generally ❌ NO | Generally ✅ YES |
| **Should catch?** | Almost **NEVER** | YES (when appropriate) |
| **Who throws?** | JVM / system | Application / libraries |
| **Examples** | `OutOfMemoryError`, `StackOverflowError` | `IOException`, `NullPointerException` |

---

### Common Error Classes

```java
// ===== OutOfMemoryError =====
// JVM has exhausted available heap memory
public void outOfMemoryDemo() {
    List<byte[]> memoryHog = new ArrayList<>();
    while (true) {
        memoryHog.add(new byte[1024 * 1024]); // 1 MB chunks
        // Eventually throws OutOfMemoryError: Java heap space
    }
}

// ===== StackOverflowError =====
// Call stack exceeded (usually infinite recursion)
public int infiniteRecursion(int n) {
    return infiniteRecursion(n + 1); // No base case!
    // throws StackOverflowError
}

// ===== NoClassDefFoundError =====
// Class was available at compile time but missing at runtime
// Example: a dependency JAR removed from classpath after compilation

// ===== ClassFormatError / VerifyError =====
// Corrupted or incompatible class file
// Example: compiled with Java 17 but running on Java 11

// ===== ExceptionInInitializerError =====
// Static initializer block throws an exception
public class BadInit {
    static {
        int result = 10 / 0; // Causes ExceptionInInitializerError
    }
}

// ===== AssertionError =====
// An assertion failed (java -ea)
assert value > 0 : "Value must be positive"; // throws AssertionError if false
```

---

### Why You Should NOT Catch Errors

```java
// ❌ Catching OutOfMemoryError — the JVM is in an unstable state
try {
    byte[] hugeArray = new byte[Integer.MAX_VALUE];
} catch (OutOfMemoryError e) {
    // DANGEROUS! The JVM may not have enough memory to:
    // - Create this catch block's stack frame
    // - Allocate the log message string
    // - Perform ANY operation reliably
    log.error("Out of memory", e);
}

// ❌ Catching StackOverflowError — stack is corrupted
try {
    deepRecursion();
} catch (StackOverflowError e) {
    // The stack is exhausted — calling any method could fail
    // Even this catch block might not execute reliably
    System.out.println("Stack overflow");
}

// ❌ Catching Error — too broad, catches all JVM problems
try {
    someOperation();
} catch (Error e) {
    // Catches OutOfMemoryError, StackOverflowError,
    // NoClassDefFoundError, InternalError, etc.
    log.error("Error occurred", e);
}
```

> **🎯 Interviewer Insight:** This is a classic trap question. "Have you ever caught `Error`?" The correct answer: **"Never in application code."** Catching `Error` masks JVM failures that should crash the process so you can debug them. The only exceptions are application servers (Tomcat, Jetty) and monitoring agents that need to stay alive.

---

### When OutOfMemoryError "Might" Be Recoverable (Not Really)

```java
// Some argue OOME is recoverable in specific scenarios:
public byte[] tryAllocate(int size) {
    try {
        return new byte[size];
    } catch (OutOfMemoryError e) {
        // Free some cached data and retry?
        cache.clear();
        System.gc(); // Suggest GC (no guarantee)
        return new byte[size]; // Might work... or might not
    }
}

// ❌ Problems with this approach:
// 1. OOME can occur in ANY thread at ANY time
// 2. The catch block itself needs memory to execute
// 3. The JVM state is unpredictable after OOME
// 4. Other threads may have already corrupted data

// ✅ Better approach: prevent OOME with proper limits
public byte[] safeAllocate(int size) {
    if (size > MAX_ALLOCATION_SIZE) {
        throw new IllegalArgumentException("Requested size too large: " + size);
    }
    return new byte[size];
}
```

> **🏭 Production Consideration:** Don't try to "recover" from `OutOfMemoryError`. The JVM is in an undefined state. Instead, **prevent** it: set appropriate `-Xmx`, use bounded collections, stream large datasets, and fix memory leaks.

---

### Legitimate Cases for Catching Error

```java
// ✅ Application server / container shutdown hooks
// Servers like Tomcat catch Errors to attempt graceful shutdown
public class AppServer {
    public void handleRequest(Request req) {
        try {
            processRequest(req);
        } catch (Exception e) {
            sendErrorResponse(req, 500);
        } catch (Error e) {
            // Log the error, attempt graceful shutdown
            log.fatal("JVM Error — shutting down", e);
            initiateShutdown();
            throw e; // Always rethrow Errors!
        }
    }
}

// ✅ Unit testing — verifying assertion errors
@Test
void testPrecondition() {
    assertThrows(AssertionError.class, () -> {
        new Account(-100); // Should trigger assertion
    });
}

// ✅ Framework-level code that must be resilient
// (logging frameworks, monitoring agents)
public class MonitoringAgent {
    public void recordMetric(String name, double value) {
        try {
            metricsStore.record(name, value);
        } catch (Throwable t) {
            // Monitoring should NEVER crash the application
            // Swallowing is acceptable here
            fallbackLog(name, value);
        }
    }
}
```

> **🎯 Interviewer Insight:** The only legitimate places to catch `Error` are: (1) application servers doing graceful shutdown, (2) monitoring/logging agents that must never crash the host app, (3) unit tests verifying assertions. In all cases, **rethrow the Error** after logging.

---

### Error vs Exception in Practice

```java
public class ErrorVsExceptionDemo {

    // ===== Exception scenarios (application handles these) =====

    public User findUser(String id) {
        User user = userRepository.findById(id);
        if (user == null) {
            throw new UserNotFoundException(id); // Application-level problem
        }
        return user;
    }

    public Config readConfig(String path) throws IOException {
        return objectMapper.readValue(new File(path), Config.class); // I/O problem
    }

    // ===== Error scenarios (JVM-level, not much you can do) =====

    // StackOverflowError — fix the recursion, don't catch it
    public int factorial(int n) {
        if (n <= 1) return 1;           // ✅ Base case prevents StackOverflowError
        return n * factorial(n - 1);
    }

    // OutOfMemoryError — tune JVM settings, don't catch it
    // -Xmx512m -Xms256m (set appropriate heap sizes)

    // NoClassDefFoundError — fix the classpath, don't catch it
    // Check Maven/Gradle dependencies
}
```

---

### How to Handle Errors Properly — Prevent, Don't Catch

```java
// ✅ Prevent rather than catch

// 1. OutOfMemoryError — use bounded collections and streaming
// ❌
List<Record> allRecords = database.fetchAll(); // Millions of records!

// ✅
database.streamAll()
    .filter(Record::isActive)
    .forEach(this::processRecord);

// 2. StackOverflowError — use iteration instead of deep recursion
// ❌
public int fibonacci(int n) {
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// ✅
public int fibonacci(int n) {
    int a = 0, b = 1;
    for (int i = 2; i <= n; i++) {
        int temp = a + b;
        a = b;
        b = temp;
    }
    return b;
}

// 3. NoClassDefFoundError — verify dependencies at startup
public class StartupChecker {
    public static void verifyDependencies() {
        try {
            Class.forName("com.fasterxml.jackson.databind.ObjectMapper");
            Class.forName("org.apache.commons.lang3.StringUtils");
        } catch (ClassNotFoundException e) {
            System.err.println("Missing dependency: " + e.getMessage());
            System.exit(1); // Fail fast at startup
        }
    }
}
```

> **🏭 Production Consideration:** The best way to "handle" Errors is to prevent them. Stream large datasets instead of loading into memory. Use iteration instead of deep recursion. Verify dependencies at startup. Set appropriate JVM flags (`-Xmx`, `-Xms`, `-XX:MaxMetaspaceSize`).

---

### Quick Reference

```
Throwable
├── Error (DON'T catch — JVM problems)
│   ├── OutOfMemoryError      → Tune -Xmx, fix memory leaks
│   ├── StackOverflowError    → Fix recursion, use iteration
│   ├── NoClassDefFoundError  → Fix classpath/dependencies
│   ├── ExceptionInInitializerError → Fix static initializers
│   └── VirtualMachineError   → JVM is broken
│
└── Exception (DO catch — application problems)
    ├── Checked (IOException, SQLException, ...)
    │   → Must handle or declare with throws
    └── RuntimeException (NPE, IAE, ...)
        → Optional handling, indicates bugs
```

---

## 🎯 Interviewer Insight

Interviewers test these specific areas:

1. **Clear distinction** — `Error` = JVM/system, `Exception` = application
2. **Never catch Error** — in application code, period
3. **Top 5 Error subclasses** — know causes: OOME, SOE, NoClassDefFoundError, ExceptionInInitializerError, AssertionError
4. **Prevention over catching** — stream, iterate, verify dependencies
5. **Legitimate exceptions** — app servers, monitoring agents, unit tests (and always rethrow)

---

## ⚡ Follow-up Questions

> **⚡ Follow-up:** Why is catching `OutOfMemoryError` dangerous even if you just log it?

> **⚡ Follow-up:** What's the difference between `NoClassDefFoundError` and `ClassNotFoundException`?

> **⚡ Follow-up:** When would an application server catch `Error`?

> **⚡ Follow-up:** How do you prevent `StackOverflowError` in recursive algorithms?

> **⚡ Follow-up:** What is `ExceptionInInitializerError` and what causes it?

---

## 📚 Related Topics

- [Exception Best Practices](./06-exception-best-practices.md)
- [Exception Chaining →](./08-exception-chaining.md)
- [Exception Performance →](./11-exception-performance.md)

---

[← Back to Section Index](./README.md) · [Previous: Exception Best Practices ←](./06-exception-best-practices.md) · [Next: Exception Chaining →](./08-exception-chaining.md)

# Q19: What is ThreadLocal and When to Use It?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What is `ThreadLocal`? What are its use cases and pitfalls?

---

## Answer

### What is ThreadLocal?

`ThreadLocal` provides **per-thread isolated storage**. Each thread has its own independent copy of the variable.

```
┌──────────────────────────────────────────────────┐
│              ThreadLocal<SimpleDateFormat>        │
│                                                  │
│  Thread A ──► its own SimpleDateFormat instance   │
│  Thread B ──► its own SimpleDateFormat instance   │
│  Thread C ──► its own SimpleDateFormat instance   │
│                                                  │
│  No sharing, no synchronization needed!          │
└──────────────────────────────────────────────────┘
```

### Basic Usage

```java
public class ThreadLocalBasics {
    // Each thread gets its own copy
    private static final ThreadLocal<SimpleDateFormat> dateFormatter =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    public static String formatDate(Date date) {
        // Thread-safe! Each thread uses its own formatter
        return dateFormatter.get().format(date);
    }

    public static void main(String[] args) throws InterruptedException {
        Date now = new Date();

        Runnable task = () -> {
            String formatted = formatDate(now);
            System.out.println(Thread.currentThread().getName() + ": " + formatted);
        };

        Thread t1 = new Thread(task, "Thread-1");
        Thread t2 = new Thread(task, "Thread-2");
        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

### Why ThreadLocal? The SimpleDateFormat Problem

```java
public class WhyThreadLocal {

    // ❌ Not thread-safe! SimpleDateFormat is NOT thread-safe
    private static final SimpleDateFormat SHARED_FORMAT =
        new SimpleDateFormat("yyyy-MM-dd");

    // ❌ Fix 1: synchronized (but creates contention)
    public static synchronized String formatSync(Date date) {
        return SHARED_FORMAT.format(date);
    }

    // ❌ Fix 2: new instance every time (garbage creation)
    public static String formatNew(Date date) {
        return new SimpleDateFormat("yyyy-MM-dd").format(date);
    }

    // ✅ Fix 3: ThreadLocal (best of both worlds)
    private static final ThreadLocal<SimpleDateFormat> FORMAT =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

    public static String formatThreadLocal(Date date) {
        return FORMAT.get().format(date); // Reuses per-thread instance
    }

    // ✅ Fix 4: Java 8+ DateTimeFormatter (thread-safe by design)
    private static final DateTimeFormatter FORMATTER =
        DateTimeFormatter.ofPattern("yyyy-MM-dd");
    // No ThreadLocal needed! DateTimeFormatter is immutable & thread-safe
}
```

### Common Use Cases

#### 1. User Context / Request Context

```java
public class UserContext {
    private static final ThreadLocal<User> currentUser = new ThreadLocal<>();

    public static void setCurrentUser(User user) {
        currentUser.set(user);
    }

    public static User getCurrentUser() {
        return currentUser.get();
    }

    public static void clear() {
        currentUser.remove(); // ✅ Always clean up!
    }

    record User(String name, String role) {}
}

// In a web filter:
public class AuthFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        try {
            User user = authenticate(request);
            UserContext.setCurrentUser(user);
            chain.doFilter(request, response);
        } finally {
            UserContext.clear(); // ✅ CRITICAL! Thread is reused in pool
        }
    }
}

// Deep in business logic (no need to pass User everywhere):
public class OrderService {
    public void placeOrder(OrderRequest request) {
        User user = UserContext.getCurrentUser(); // Access without parameter passing
        System.out.println("Order placed by: " + user.name());
    }
}
```

#### 2. Database Connection per Thread

```java
public class ConnectionManager {
    private static final ThreadLocal<Connection> connectionHolder =
        ThreadLocal.withInitial(() -> {
            try {
                return DriverManager.getConnection("jdbc:mysql://localhost/mydb");
            } catch (SQLException e) {
                throw new RuntimeException("Failed to create connection", e);
            }
        });

    public static Connection getConnection() {
        return connectionHolder.get();
    }

    public static void closeConnection() {
        Connection conn = connectionHolder.get();
        if (conn != null) {
            try {
                conn.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
            connectionHolder.remove();
        }
    }
}
```

#### 3. Transaction Context

```java
public class TransactionManager {
    private static final ThreadLocal<Transaction> currentTx = new ThreadLocal<>();

    public static void begin() {
        currentTx.set(new Transaction());
    }

    public static Transaction current() {
        Transaction tx = currentTx.get();
        if (tx == null) throw new IllegalStateException("No active transaction");
        return tx;
    }

    public static void commit() {
        try {
            current().commit();
        } finally {
            currentTx.remove();
        }
    }

    public static void rollback() {
        try {
            current().rollback();
        } finally {
            currentTx.remove();
        }
    }

    static class Transaction {
        void commit() { System.out.println("Committed"); }
        void rollback() { System.out.println("Rolled back"); }
    }
}
```

### InheritableThreadLocal

```java
public class InheritableThreadLocalDemo {
    // Values are inherited by child threads
    private static final InheritableThreadLocal<String> requestId =
        new InheritableThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {
        requestId.set("REQ-12345");

        Thread child = new Thread(() -> {
            // Child thread inherits the value!
            System.out.println("Child sees: " + requestId.get()); // REQ-12345
        });
        child.start();
        child.join();

        // ⚠️ WARNING: Does NOT work with thread pools!
        // Thread pool threads are created once, not per-task
        // The inherited value would be from the FIRST parent, not the current one
    }
}
```

### Memory Leak with Thread Pools

```java
public class ThreadLocalLeakDemo {
    /*
     * HOW THE LEAK HAPPENS:
     *
     * Thread pool thread → ThreadLocalMap → Entry(WeakRef→key, StrongRef→value)
     *
     * If you don't call remove():
     * 1. ThreadLocal key gets GC'd (weak reference)
     * 2. But the VALUE stays in the map (strong reference)
     * 3. Thread lives forever (pool) → map lives forever → value lives forever
     * 4. MEMORY LEAK!
     *
     * Even worse with web apps:
     * Value may reference classloader → entire WAR's classes leak!
     */

    // ❌ Leak-prone
    private static final ThreadLocal<byte[]> leaky = new ThreadLocal<>();

    public void handleRequestBad() {
        leaky.set(new byte[1024 * 1024]); // 1MB per thread, never freed!
        processRequest();
        // Missing leaky.remove()!
    }

    // ✅ Leak-safe
    public void handleRequestGood() {
        try {
            leaky.set(new byte[1024 * 1024]);
            processRequest();
        } finally {
            leaky.remove(); // ✅ ALWAYS clean up in finally
        }
    }

    private void processRequest() { /* ... */ }
}
```

### ThreadLocal Under the Hood

```
Thread object:
┌─────────────────────────────────┐
│ Thread                          │
│  ├── name                       │
│  ├── priority                   │
│  └── threadLocals: ThreadLocalMap│
│       ├── Entry[0]: key=TL1 → value1  │
│       ├── Entry[1]: key=TL2 → value2  │
│       └── Entry[2]: key=TL3 → value3  │
└─────────────────────────────────┘

- Each Thread has a ThreadLocalMap
- ThreadLocal.get() → currentThread.threadLocals.get(this)
- Keys are WeakReferences to ThreadLocal objects
- Values are strong references (→ leak risk!)
```

### Virtual Threads and ThreadLocal (Java 21)

```java
// ⚠️ ThreadLocal works with virtual threads BUT:
// - Virtual threads are cheap and plentiful (millions)
// - ThreadLocal per virtual thread = memory explosion!
// - Use ScopedValue (preview) instead for virtual threads

// ❌ Bad with virtual threads
ThreadLocal<ExpensiveObject> tl = new ThreadLocal<>();
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        executor.submit(() -> {
            tl.set(new ExpensiveObject()); // 1M instances!
        });
    }
}

// ✅ Better: ScopedValue (Java 21 preview)
// ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();
// ScopedValue.where(CURRENT_USER, user).run(() -> { ... });
```

---

## Anti-Patterns

❌ **Not calling `remove()` in thread pool environments:**
```java
threadLocal.set(value);
// process...
// ❌ Forgot threadLocal.remove()! → Memory leak
```

❌ **Using InheritableThreadLocal with thread pools**

❌ **Excessive use of ThreadLocal as a "global variable":**
```java
// ❌ Using ThreadLocal to avoid passing parameters = code smell
// Consider dependency injection instead
```

## Best Practices

✅ **Always** call `remove()` in a finally block  
✅ Use `ThreadLocal.withInitial()` for lazy initialization  
✅ Prefer thread-safe alternatives (e.g., `DateTimeFormatter` over `SimpleDateFormat`)  
✅ Use ThreadLocal for request-scoped data in web applications  
✅ Be cautious with virtual threads (consider `ScopedValue`)  
✅ Document ThreadLocal usage and lifecycle clearly

---

[← Fork/Join Framework](./q18-fork-join-framework.md) | [Next: Thread Safety Patterns →](./q20-thread-safety-patterns.md)

# Q20: What Are Common Thread Safety Patterns?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What are the common patterns for achieving thread safety in Java?

---

## Answer

### Thread Safety Strategies Overview

```
┌───────────────────────────────────────────────┐
│           Thread Safety Strategies            │
├───────────────────────────────────────────────┤
│                                               │
│  1. Immutability    - No mutable state        │
│  2. Confinement     - Don't share state       │
│  3. Synchronization - Guard shared state      │
│  4. Atomic classes  - Lock-free operations    │
│  5. Safe publication- Make objects visible     │
│                                               │
│  Safest ──────────────────────── Most complex │
│  Immutability ◄─────────► Synchronization     │
└───────────────────────────────────────────────┘
```

### Pattern 1: Immutability

The simplest and safest approach — immutable objects are inherently thread-safe.

```java
// ✅ Immutable class - thread-safe by design
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;
        this.currency = currency;
    }

    public BigDecimal getAmount() { return amount; }
    public Currency getCurrency() { return currency; }

    // Operations return NEW instances
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    // No setters! No way to modify state after construction
}

// ✅ Java Records (Java 16+) - immutable by default
public record Point(double x, double y) {
    public double distanceTo(Point other) {
        return Math.sqrt(Math.pow(x - other.x, 2) + Math.pow(y - other.y, 2));
    }
}

// ✅ Unmodifiable collections
List<String> immutableList = List.of("a", "b", "c");
Map<String, Integer> immutableMap = Map.of("key", 1);
```

### Pattern 2: Thread Confinement

If data is never shared between threads, no synchronization is needed.

```java
// Stack confinement - local variables are thread-confined
public class StackConfinement {
    public int processData(List<Integer> input) {
        // Local variables live on the thread's stack - inherently safe
        List<Integer> localCopy = new ArrayList<>(input); // Defensive copy
        int sum = 0;
        for (int val : localCopy) {
            sum += val; // Only this thread accesses 'sum' and 'localCopy'
        }
        return sum;
    }
}

// ThreadLocal confinement
public class ThreadConfinement {
    private static final ThreadLocal<Connection> conn =
        ThreadLocal.withInitial(() -> createConnection());

    public void doWork() {
        Connection myConn = conn.get(); // Each thread has its own connection
        // No synchronization needed!
    }

    private static Connection createConnection() {
        return null; // Placeholder
    }
}
```

### Pattern 3: Synchronization

```java
// Guarded state pattern - all access to shared state is synchronized
public class GuardedCounter {
    // Document the locking policy!
    // @GuardedBy("this")
    private int count = 0;
    // @GuardedBy("this")
    private int max = 0;

    public synchronized void increment() {
        count++;
        max = Math.max(max, count);
    }

    public synchronized void decrement() {
        count--;
    }

    // ALL access must be synchronized - including reads!
    public synchronized int getCount() { return count; }
    public synchronized int getMax() { return max; }
}

// Private lock pattern - prevents external interference
public class PrivateLock {
    private final Object lock = new Object();
    private final List<String> items = new ArrayList<>();

    public void add(String item) {
        synchronized (lock) {
            items.add(item);
        }
    }

    public List<String> getSnapshot() {
        synchronized (lock) {
            return new ArrayList<>(items); // Defensive copy under lock
        }
    }
}
```

### Pattern 4: Thread-Safe Singleton Patterns

```java
// ✅ Approach 1: Enum Singleton (recommended by Joshua Bloch)
public enum DatabaseConnection {
    INSTANCE;

    private Connection connection;

    DatabaseConnection() {
        // Initialization
        connection = createConnection();
    }

    public Connection getConnection() {
        return connection;
    }

    private Connection createConnection() { return null; }
}

// ✅ Approach 2: Initialization-on-Demand Holder (lazy, thread-safe)
public class Singleton {
    private Singleton() {}

    // Inner class is not loaded until getInstance() is called
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
        // JVM guarantees class loading is thread-safe!
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE; // Lazy, thread-safe, no synchronization!
    }
}

// ✅ Approach 3: Double-checked locking (volatile required)
public class DCLSingleton {
    private static volatile DCLSingleton instance;

    private DCLSingleton() {}

    public static DCLSingleton getInstance() {
        if (instance == null) {                // First check (no lock)
            synchronized (DCLSingleton.class) {
                if (instance == null) {        // Second check (with lock)
                    instance = new DCLSingleton();
                }
            }
        }
        return instance;
    }
}

// ❌ BROKEN: Eager initialization (wastes resources if never used)
// ❌ BROKEN: synchronized getInstance() (unnecessary contention after init)
```

### Pattern 5: Safe Publication

```java
public class SafePublication {

    // ❌ UNSAFE publication - other threads may see partially constructed object
    public static Holder holder;

    // ✅ Safe publication techniques:

    // 1. Static initializer (class loading is thread-safe)
    public static final Holder STATIC_HOLDER = new Holder(42);

    // 2. Volatile field
    public static volatile Holder volatileHolder;

    // 3. AtomicReference
    public static final AtomicReference<Holder> atomicHolder = new AtomicReference<>();

    // 4. Synchronized block
    private static Holder syncHolder;
    public static synchronized void setHolder(Holder h) { syncHolder = h; }
    public static synchronized Holder getHolder() { return syncHolder; }

    // 5. Final field (safe publication of 'this' in constructor)
    public static class Container {
        private final Holder holder;

        public Container(Holder holder) {
            this.holder = holder;
            // After constructor completes, 'holder' is safely published
        }

        public Holder getHolder() { return holder; }
    }

    static class Holder {
        private final int value;
        Holder(int value) { this.value = value; }
    }
}
```

### Pattern 6: Copy-on-Write

```java
public class CopyOnWritePattern {
    // Reads are lock-free, writes create a new copy
    private volatile List<String> listeners = List.of();

    public void addListener(String listener) {
        synchronized (this) {
            List<String> newList = new ArrayList<>(listeners);
            newList.add(listener);
            listeners = Collections.unmodifiableList(newList);
        }
    }

    public void notifyListeners(String event) {
        // No lock needed! 'listeners' reference is volatile
        List<String> snapshot = listeners;
        for (String listener : snapshot) {
            // Safe iteration - snapshot won't change
            System.out.println(listener + ": " + event);
        }
    }

    // Or use the built-in:
    private final CopyOnWriteArrayList<String> builtIn = new CopyOnWriteArrayList<>();
}
```

### Pattern 7: Atomic Field Updaters

```java
// For classes with many instances where AtomicInteger would waste memory
public class AtomicFieldUpdaterPattern {

    private static final AtomicIntegerFieldUpdater<AtomicFieldUpdaterPattern> STATUS_UPDATER =
        AtomicIntegerFieldUpdater.newUpdater(AtomicFieldUpdaterPattern.class, "status");

    private static final AtomicReferenceFieldUpdater<AtomicFieldUpdaterPattern, String> NAME_UPDATER =
        AtomicReferenceFieldUpdater.newUpdater(
            AtomicFieldUpdaterPattern.class, String.class, "name");

    volatile int status;      // Must be volatile for field updaters
    volatile String name;

    public boolean activate() {
        return STATUS_UPDATER.compareAndSet(this, 0, 1);
    }

    public void setName(String newName) {
        NAME_UPDATER.set(this, newName);
    }
}
```

### Pattern Summary

```
┌─────────────────────┬───────────────┬──────────────┬──────────────┐
│ Pattern             │ Performance   │ Complexity   │ When to Use  │
├─────────────────────┼───────────────┼──────────────┼──────────────┤
│ Immutability        │ ⭐⭐⭐       │ ⭐           │ Always first │
│ Stack confinement   │ ⭐⭐⭐       │ ⭐           │ Local vars   │
│ ThreadLocal         │ ⭐⭐⭐       │ ⭐⭐         │ Per-thread   │
│ volatile            │ ⭐⭐⭐       │ ⭐⭐         │ Simple flags │
│ Atomic classes      │ ⭐⭐⭐       │ ⭐⭐         │ Counters     │
│ synchronized        │ ⭐⭐         │ ⭐⭐         │ Compound ops │
│ ReentrantLock       │ ⭐⭐         │ ⭐⭐⭐       │ Advanced     │
│ ConcurrentMap       │ ⭐⭐⭐       │ ⭐           │ Shared maps  │
│ Copy-on-write       │ ⭐⭐ (reads) │ ⭐⭐         │ Read-heavy   │
└─────────────────────┴───────────────┴──────────────┴──────────────┘
```

---

## Anti-Patterns

❌ **Mutable singletons without synchronization**  
❌ **Leaking `this` reference during construction:**
```java
public class BadConstruction {
    public BadConstruction(EventSource source) {
        source.register(this); // ❌ 'this' escapes before construction completes!
    }
}
```

❌ **Publishing objects without proper visibility guarantees**  
❌ **Over-synchronization (locking everything "just in case")**

## Best Practices

✅ Prefer immutability as the first line of defense  
✅ Use `enum` for singletons  
✅ Use the holder pattern for lazy thread-safe initialization  
✅ Document your locking policy (`@GuardedBy`)  
✅ Use `private final` lock objects  
✅ Choose the simplest pattern that provides the required safety

---

[← ThreadLocal](./q19-thread-local.md) | [Next: Semaphore & CountDownLatch →](./q21-semaphore-countdownlatch.md)

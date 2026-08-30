# Q8: What is the Volatile Keyword?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What does the `volatile` keyword do in Java? How does it differ from `synchronized`? When is it sufficient?

---

## Answer

### The Visibility Problem

Without `volatile`, threads may cache variables in CPU registers or L1/L2 cache:

```
┌─────────────┐    ┌─────────────┐
│   Thread A  │    │   Thread B  │
│             │    │             │
│ CPU Cache   │    │ CPU Cache   │
│ flag = true │    │ flag = false│  ← Stale!
└──────┬──────┘    └──────┬──────┘
       │                  │
       └────────┬─────────┘
                │
        ┌───────▼───────┐
        │  Main Memory  │
        │  flag = true  │
        └───────────────┘
```

### What Volatile Does

The `volatile` keyword guarantees:
1. **Visibility** - Writes are immediately flushed to main memory; reads always fetch from main memory
2. **Ordering** - Prevents instruction reordering around volatile accesses (happens-before)

```java
public class VolatileDemo {
    // ❌ Without volatile - thread may never see the update
    // private boolean running = true;

    // ✅ With volatile - guaranteed visibility
    private volatile boolean running = true;

    public void start() {
        new Thread(() -> {
            int count = 0;
            while (running) {  // Reads from main memory every time
                count++;
            }
            System.out.println("Stopped after " + count + " iterations");
        }).start();
    }

    public void stop() {
        running = false; // Write is immediately visible to all threads
    }

    public static void main(String[] args) throws InterruptedException {
        VolatileDemo demo = new VolatileDemo();
        demo.start();
        Thread.sleep(1000);
        demo.stop(); // Without volatile, the other thread might spin forever!
    }
}
```

### Volatile vs Synchronized

```
                ┌───────────────┬─────────────┬──────────────┐
                │   Feature     │  volatile   │ synchronized │
                ├───────────────┼─────────────┼──────────────┤
                │ Visibility    │     ✅      │      ✅      │
                │ Atomicity     │     ❌      │      ✅      │
                │ Mutual excl.  │     ❌      │      ✅      │
                │ Blocking      │     No      │     Yes      │
                │ Performance   │   Faster    │    Slower    │
                │ Scope         │  Variables  │ Blocks/Meth  │
                └───────────────┴─────────────┴──────────────┘
```

### When Volatile is Sufficient

**Rule:** Volatile is sufficient when writes don't depend on the current value.

```java
// ✅ SAFE: Simple flag (write doesn't depend on current value)
private volatile boolean shutdown = false;

public void shutdown() { shutdown = true; }  // Independent write
public boolean isShutdown() { return shutdown; }

// ✅ SAFE: Publishing an immutable object
private volatile Config config;

public void updateConfig(Config newConfig) {
    config = newConfig; // Atomic reference assignment
}

// ❌ UNSAFE: Compound operation (read-modify-write)
private volatile int count = 0;

public void increment() {
    count++; // NOT ATOMIC! Read + increment + write
    // Thread A reads 5, Thread B reads 5
    // Both write 6 → lost update!
}
```

### Volatile and Atomicity

```java
public class VolatileNotAtomic {
    private volatile int count = 0;

    // ❌ This is NOT thread-safe even with volatile
    public void increment() {
        count++; // This is actually: temp = count; temp = temp + 1; count = temp;
    }

    // ✅ Fix 1: Use synchronized
    public synchronized void incrementSafe() {
        count++;
    }

    // ✅ Fix 2: Use AtomicInteger (better performance)
    private final AtomicInteger atomicCount = new AtomicInteger(0);
    public void incrementAtomic() {
        atomicCount.incrementAndGet();
    }
}
```

### Happens-Before with Volatile

```java
public class VolatileHappensBefore {
    private int a = 0;
    private int b = 0;
    private volatile boolean flag = false;

    // Thread A
    public void writer() {
        a = 1;          // (1) Non-volatile write
        b = 2;          // (2) Non-volatile write
        flag = true;    // (3) Volatile write
        // The volatile write at (3) establishes happens-before:
        // Writes at (1) and (2) are GUARANTEED to be visible
        // to any thread that reads flag == true
    }

    // Thread B
    public void reader() {
        if (flag) {     // (4) Volatile read
            // If we see flag == true, we are GUARANTEED to see:
            // a == 1 and b == 2
            System.out.println(a); // Always 1 (never 0)
            System.out.println(b); // Always 2 (never 0)
        }
    }
}
```

### Volatile with Arrays and Objects

```java
public class VolatileReferences {

    // volatile on an array reference - ONLY the reference is volatile
    private volatile int[] array = new int[10];

    public void updateArray() {
        // ❌ Array element writes are NOT volatile
        array[0] = 42; // NOT guaranteed visible to other threads!

        // ✅ Replace the entire array reference
        int[] newArray = array.clone();
        newArray[0] = 42;
        array = newArray; // Volatile write of reference → all elements visible
    }

    // volatile on an object reference - same issue
    private volatile Map<String, String> config = new HashMap<>();

    public void updateConfig() {
        // ❌ Mutation of the map is NOT volatile
        config.put("key", "value"); // NOT safe!

        // ✅ Replace with new map
        Map<String, String> newConfig = new HashMap<>(config);
        newConfig.put("key", "value");
        config = newConfig; // Safe - volatile write publishes all changes
    }
}
```

### Volatile for Safe Publication

```java
public class SafePublication {
    // The "publication" pattern - making an object safely visible to other threads

    // ❌ UNSAFE without volatile
    // private Holder holder;

    // ✅ SAFE with volatile
    private volatile Holder holder;

    public void initialize() {
        holder = new Holder(42); // Volatile write ensures:
        // 1. Holder is fully constructed before publication
        // 2. All threads see a fully initialized Holder
    }

    static class Holder {
        private final int value;
        Holder(int value) { this.value = value; }
        int getValue() { return value; }
    }
}
```

### Practical Example: Lazy Initialization

```java
public class LazyInit {

    // ✅ Thread-safe lazy initialization with volatile
    private volatile ExpensiveResource resource;

    public ExpensiveResource getResource() {
        ExpensiveResource local = resource; // Single volatile read
        if (local == null) {
            synchronized (this) {
                local = resource;
                if (local == null) {
                    local = new ExpensiveResource();
                    resource = local; // Volatile write
                }
            }
        }
        return local;
    }

    static class ExpensiveResource {
        // Heavy initialization
        ExpensiveResource() {
            // Load configuration, connect to services, etc.
        }
    }
}
```

### Performance Comparison

```java
public class PerformanceBenchmark {
    private int normalVar = 0;
    private volatile int volatileVar = 0;
    private final AtomicInteger atomicVar = new AtomicInteger(0);

    // Approximate relative costs (single-threaded):
    // Normal variable read/write:    ~1x (baseline)
    // Volatile variable read/write:  ~5-10x (memory barrier)
    // Synchronized block:            ~25-50x (lock acquisition)
    // AtomicInteger CAS:             ~10-20x (CAS instruction)

    // Note: These are rough estimates and vary by hardware and contention level
}
```

---

## Anti-Patterns

❌ **Using volatile for compound operations:**
```java
volatile int count = 0;
count++; // ❌ Not atomic! Use AtomicInteger instead
```

❌ **Assuming volatile makes array elements visible:**
```java
volatile int[] arr = new int[5];
arr[0] = 10; // ❌ Element writes are NOT volatile
```

❌ **Using volatile when mutual exclusion is needed:**
```java
volatile Map<String, String> map = new HashMap<>();
map.put("key", "val"); // ❌ HashMap is not thread-safe
```

## Best Practices

✅ Use `volatile` for simple flags and status indicators  
✅ Use `volatile` for safe publication of immutable objects  
✅ Use `AtomicInteger/Long` for counters (not `volatile int++`)  
✅ Use `synchronized` or `Lock` when multiple variables must be updated atomically  
✅ Prefer `volatile` over `synchronized` for single-variable visibility  
✅ Use the "local variable" trick for multiple reads of a volatile field:

```java
volatile Config config;
public void process() {
    Config local = config; // Single volatile read
    local.getA();          // Non-volatile reads from here
    local.getB();
}
```

---

[← Synchronized Keyword](./q07-synchronized-keyword.md) | [Next: Atomic Classes →](./q09-atomic-classes.md)

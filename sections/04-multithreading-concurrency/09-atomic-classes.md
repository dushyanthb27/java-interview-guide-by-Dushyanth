# Q9: How Do Atomic Classes Work?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
How do Atomic classes in `java.util.concurrent.atomic` work? What is CAS (Compare-And-Swap)?

---

## Answer

### The Problem: Thread-Safe Operations Without Locks

```java
// ❌ Not thread-safe
private int count = 0;
public void increment() { count++; } // Read-modify-write is NOT atomic

// ✅ Thread-safe with synchronized (but has overhead)
public synchronized void incrementSync() { count++; }

// ✅ Thread-safe with AtomicInteger (lock-free, faster)
private final AtomicInteger atomicCount = new AtomicInteger(0);
public void incrementAtomic() { atomicCount.incrementAndGet(); }
```

### Compare-And-Swap (CAS) Operation

```
CAS(memoryLocation, expectedValue, newValue)

Step 1: Read current value from memory
Step 2: Compare current value with expected value
Step 3: If equal → update to new value (atomically)
        If not equal → retry (spin)

┌─────────────────────────────────────────────┐
│  Thread A:                                  │
│    1. Read count = 5  (expected = 5)        │
│    2. CAS(count, 5, 6) → SUCCESS (count=6) │
│                                             │
│  Thread B:                                  │
│    1. Read count = 5  (expected = 5)        │
│    2. CAS(count, 5, 6) → FAIL (count is 6) │
│    3. Retry: Read count = 6 (expected = 6)  │
│    4. CAS(count, 6, 7) → SUCCESS (count=7) │
└─────────────────────────────────────────────┘
```

CAS is a **CPU-level atomic instruction** (e.g., `CMPXCHG` on x86) — no locks needed.

### AtomicInteger

```java
public class AtomicIntegerDemo {
    private final AtomicInteger counter = new AtomicInteger(0);

    public void demonstrate() {
        // Basic operations
        counter.set(10);                        // Atomic set
        int value = counter.get();              // Atomic get

        // Atomic increment/decrement
        int newVal = counter.incrementAndGet(); // ++counter (returns 11)
        int oldVal = counter.getAndIncrement(); // counter++ (returns 11, now 12)
        counter.decrementAndGet();              // --counter (returns 11)
        counter.getAndDecrement();              // counter-- (returns 11, now 10)

        // Atomic add
        counter.addAndGet(5);                   // counter += 5 (returns 15)
        counter.getAndAdd(3);                   // returns 15, counter = 18

        // CAS
        boolean success = counter.compareAndSet(18, 20); // if (counter == 18) counter = 20
        System.out.println("CAS success: " + success);   // true

        // Atomic update with function (Java 8+)
        counter.updateAndGet(x -> x * 2);               // counter = counter * 2
        counter.getAndUpdate(x -> x + 10);               // returns old, counter += 10

        // Accumulate (combine with another value)
        counter.accumulateAndGet(5, Integer::max);       // counter = max(counter, 5)
    }
}
```

### AtomicLong and AtomicBoolean

```java
public class OtherAtomicsDemo {

    // AtomicLong - same API as AtomicInteger but for long values
    private final AtomicLong requestCount = new AtomicLong(0);

    public void recordRequest() {
        requestCount.incrementAndGet();
    }

    // AtomicBoolean - for thread-safe flags
    private final AtomicBoolean initialized = new AtomicBoolean(false);

    public void initOnce() {
        // Guaranteed only one thread runs initialization
        if (initialized.compareAndSet(false, true)) {
            performExpensiveInit();
            System.out.println("Initialized by: " + Thread.currentThread().getName());
        } else {
            System.out.println("Already initialized, skipping");
        }
    }

    private void performExpensiveInit() { /* heavy work */ }
}
```

### AtomicReference

```java
public class AtomicReferenceDemo {
    // Atomic operations on object references
    private final AtomicReference<String> currentUser = new AtomicReference<>("anonymous");

    public void login(String username) {
        String prev = currentUser.getAndSet(username);
        System.out.println("Switched from " + prev + " to " + username);
    }

    // Thread-safe immutable object update
    record Config(String host, int port, boolean ssl) {}

    private final AtomicReference<Config> config = new AtomicReference<>(
        new Config("localhost", 8080, false)
    );

    public void enableSsl() {
        config.updateAndGet(c -> new Config(c.host(), c.port(), true));
    }

    // Lock-free stack using AtomicReference
    static class LockFreeStack<T> {
        private final AtomicReference<Node<T>> top = new AtomicReference<>();

        private static class Node<T> {
            final T value;
            final Node<T> next;
            Node(T value, Node<T> next) {
                this.value = value;
                this.next = next;
            }
        }

        public void push(T value) {
            Node<T> newHead;
            Node<T> oldHead;
            do {
                oldHead = top.get();
                newHead = new Node<>(value, oldHead);
            } while (!top.compareAndSet(oldHead, newHead));
        }

        public T pop() {
            Node<T> oldHead;
            Node<T> newHead;
            do {
                oldHead = top.get();
                if (oldHead == null) return null;
                newHead = oldHead.next;
            } while (!top.compareAndSet(oldHead, newHead));
            return oldHead.value;
        }
    }
}
```

### The ABA Problem

```
The ABA Problem with CAS:
1. Thread A reads value = A
2. Thread B changes value: A → B → A
3. Thread A does CAS(A, A, C) → SUCCESS (but value was modified!)

This is a problem when the identity of the value matters,
not just its current state (e.g., linked list node reuse).
```

### AtomicStampedReference (Solves ABA)

```java
public class ABADemo {
    // AtomicStampedReference tracks a "stamp" (version number) alongside the reference
    private final AtomicStampedReference<String> ref =
        new AtomicStampedReference<>("initial", 0);

    public void update() {
        int[] stampHolder = new int[1];
        String current = ref.get(stampHolder);   // Get value AND stamp
        int currentStamp = stampHolder[0];

        // CAS checks both value AND stamp
        boolean success = ref.compareAndSet(
            current,           // expected reference
            "updated",         // new reference
            currentStamp,      // expected stamp
            currentStamp + 1   // new stamp
        );
        System.out.println("Update success: " + success);
    }

    // AtomicMarkableReference - uses boolean instead of int stamp
    private final AtomicMarkableReference<String> markable =
        new AtomicMarkableReference<>("data", false);

    public void markForDeletion() {
        markable.compareAndSet("data", "data", false, true);
    }
}
```

### LongAdder and LongAccumulator (Java 8+)

For high-contention counters, `LongAdder` outperforms `AtomicLong`:

```java
public class LongAdderDemo {
    // AtomicLong: all threads CAS on same memory location → contention
    private final AtomicLong atomicLong = new AtomicLong(0);

    // LongAdder: distributes updates across cells → less contention
    private final LongAdder longAdder = new LongAdder();

    public void benchmarkComparison() {
        // LongAdder usage
        longAdder.increment();           // Equivalent to add(1)
        longAdder.add(10);
        longAdder.decrement();

        long sum = longAdder.sum();      // Get current value
        long sumThenReset = longAdder.sumThenReset(); // Atomic sum + reset

        // LongAccumulator: custom accumulation function
        LongAccumulator maxAccumulator = new LongAccumulator(Long::max, Long.MIN_VALUE);
        maxAccumulator.accumulate(42);
        maxAccumulator.accumulate(99);
        maxAccumulator.accumulate(7);
        System.out.println("Max: " + maxAccumulator.get()); // 99

        // DoubleAdder / DoubleAccumulator for floating-point
        DoubleAdder doubleAdder = new DoubleAdder();
        doubleAdder.add(3.14);
    }
}
```

```
LongAdder internals:
┌─────────────────────────────────────────┐
│  Thread 1 ──► Cell[0]: 5               │
│  Thread 2 ──► Cell[1]: 3               │
│  Thread 3 ──► Cell[2]: 7               │
│  Thread 4 ──► Cell[0]: 2 (hash clash)  │
│                                         │
│  sum() = Cell[0] + Cell[1] + Cell[2]   │
│        = (5+2) + 3 + 7 = 17            │
└─────────────────────────────────────────┘
Use LongAdder when writes dominate reads (counters, stats).
Use AtomicLong when you need an exact current value (sequence generators).
```

### AtomicIntegerFieldUpdater

Update fields atomically without wrapping them in Atomic classes:

```java
public class FieldUpdaterDemo {
    // Useful when you have many instances and don't want AtomicInteger per instance
    private static final AtomicIntegerFieldUpdater<FieldUpdaterDemo> COUNT_UPDATER =
        AtomicIntegerFieldUpdater.newUpdater(FieldUpdaterDemo.class, "count");

    // Field must be volatile, non-static, non-private, non-final
    volatile int count = 0;

    public void increment() {
        COUNT_UPDATER.incrementAndGet(this);
    }

    public int getCount() {
        return count; // volatile read is sufficient
    }
}
```

### Practical Example: Thread-Safe Metrics

```java
public class MetricsCollector {
    private final LongAdder requestCount = new LongAdder();
    private final LongAdder errorCount = new LongAdder();
    private final LongAdder totalLatencyMs = new LongAdder();
    private final LongAccumulator maxLatencyMs = new LongAccumulator(Long::max, 0);

    public void recordRequest(long latencyMs, boolean success) {
        requestCount.increment();
        totalLatencyMs.add(latencyMs);
        maxLatencyMs.accumulate(latencyMs);
        if (!success) {
            errorCount.increment();
        }
    }

    public String getSnapshot() {
        long requests = requestCount.sum();
        long errors = errorCount.sum();
        long totalLatency = totalLatencyMs.sum();
        long maxLatency = maxLatencyMs.get();
        double avgLatency = requests > 0 ? (double) totalLatency / requests : 0;
        double errorRate = requests > 0 ? (double) errors / requests * 100 : 0;

        return String.format(
            "Requests: %d | Errors: %d (%.1f%%) | Avg Latency: %.1fms | Max: %dms",
            requests, errors, errorRate, avgLatency, maxLatency
        );
    }
}
```

---

## Anti-Patterns

❌ **Using AtomicLong under high write contention:**
```java
AtomicLong counter = new AtomicLong(); // ❌ High contention → use LongAdder
```

❌ **Assuming multiple atomic operations are collectively atomic:**
```java
AtomicInteger a = new AtomicInteger(0);
AtomicInteger b = new AtomicInteger(0);
a.incrementAndGet();
b.incrementAndGet();
// ❌ Not atomic together! Another thread can see a=1, b=0
```

❌ **Ignoring the ABA problem in lock-free data structures**

## Best Practices

✅ Use `AtomicInteger/Long` for simple counters with low-moderate contention  
✅ Use `LongAdder` for high-contention write-heavy counters  
✅ Use `AtomicReference` for lock-free object publication  
✅ Use `AtomicStampedReference` when ABA matters  
✅ Use field updaters when memory footprint matters (many instances)  
✅ Prefer atomic classes over `synchronized` for single-variable atomicity

---

[← Volatile Keyword](./q08-volatile-keyword.md) | [Next: ReentrantLock →](./q10-reentrant-lock.md)

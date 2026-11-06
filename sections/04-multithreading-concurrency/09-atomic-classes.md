# Q9: How Do Atomic Classes Work?

[← Back to Multithreading & Concurrency](./README.md)

---

## 📋 Question

How do atomic classes in `java.util.concurrent.atomic` work under the hood? What is Compare-And-Swap (CAS), how does it avoid locks, what is the **ABA problem**, and how do modern Java features (`LongAdder`, `VarHandle`) enhance atomic performance?

## Difficulty: 🟡 Intermediate

---

## 💡 Answer

### The Problem: Lock-Free Thread Safety

Traditional thread-safe operations rely on pessimistic locking (`synchronized` or `ReentrantLock`), which blocks contending threads and causes costly OS context switches. Atomic classes provide **lock-free, optimistic concurrency** using hardware-level atomic instructions.

```java
// ❌ Not thread-safe (read-modify-write race condition)
private int count = 0;
public void increment() { count++; }

// ✅ Thread-safe with lock-free atomic CAS
private final AtomicInteger atomicCount = new AtomicInteger(0);
public void incrementAtomic() { atomicCount.incrementAndGet(); }
```

### Compare-And-Swap (CAS) Mechanics

CAS is an atomic CPU instruction (e.g., `CMPXCHG` on x86, `LDREX/STREX` on ARM) supported directly by the memory bus. It takes three operands:
1. **Memory location ($V$)**
2. **Expected old value ($A$)**
3. **New value ($B$)**

```
CAS(V, Expected A, New B)
- If value at memory location V equals A, atomically replace V with B and return true.
- If value at V does not equal A, do nothing and return false (the caller loops/spins and retries).
```

```
┌────────────────────────────────────────────────────────┐
│ Thread A:                                              │
│   1. Read count = 5                                    │
│   2. CAS(count, 5, 6) → SUCCESS (count is now 6)       │
│                                                        │
│ Thread B:                                              │
│   1. Read count = 5 (stale expected value)             │
│   2. CAS(count, 5, 6) → FAIL (memory value is 6 != 5)  │
│   3. Reload new value (6), compute 7, CAS(count, 6, 7) │
└────────────────────────────────────────────────────────┘
```

### Core Atomic Categories

```java
public class AtomicsDemo {
    // 1. Primitive Atomics (AtomicInteger, AtomicLong, AtomicBoolean)
    private final AtomicInteger counter = new AtomicInteger(0);
    private final AtomicBoolean initialized = new AtomicBoolean(false);

    public void demonstratePrimitives() {
        counter.incrementAndGet();                       // ++counter
        counter.updateAndGet(x -> x * 2);                // Functional update
        counter.accumulateAndGet(5, Integer::max);       // max(current, 5)

        // Compare-And-Set idiom
        if (initialized.compareAndSet(false, true)) {
            // Perform one-time initialization safely without locks
        }
    }

    // 2. Reference Atomics (AtomicReference, AtomicStampedReference)
    private final AtomicReference<String> currentConfig = new AtomicReference<>("v1");

    public void updateConfig(String oldCfg, String newCfg) {
        currentConfig.compareAndSet(oldCfg, newCfg);
    }
}
```

### The ABA Problem & `AtomicStampedReference`

**The ABA Dilemma:**
1. Thread 1 reads value `A` from memory location $V$.
2. Thread 1 gets preempted.
3. Thread 2 changes $V$ from `A` to `B`, and then changes it back from `B` to `A`.
4. Thread 1 wakes up, executes CAS($V$, `A`, `C`). Because the value is still `A`, the CAS **succeeds**, even though intermediate changes occurred behind its back.

**The Solution (`AtomicStampedReference`):**
`AtomicStampedReference` pairs the reference with an integer **stamp** (version counter). Every modification increments the stamp, ensuring CAS fails if the stamp has changed even if the object reference matches.

```java
public class ABAPreventionDemo {
    // Pairs value with an integer version stamp
    private final AtomicStampedReference<String> stackTop = 
        new AtomicStampedReference<>(null, 0);

    public void push(String item) {
        int[] stampHolder = new int[1];
        String currentRef;
        int currentStamp;
        do {
            currentRef = stackTop.getReference();
            currentStamp = stampHolder[0];
            stackTop.get(stampHolder);
        } while (!stackTop.compareAndSet(currentRef, item, currentStamp, currentStamp + 1));
    }
}
```

### High-Contention Scalability: `LongAdder` vs `AtomicLong`

Under heavy concurrent contention (e.g., thousands of threads incrementing a metrics counter simultaneously), CAS retry loops (`while(!cas(...))`) cause massive CPU cache invalidation traffic and latency spikes.

- **`AtomicLong`:** Single shared memory location. High contention = severe CAS spin overhead.
- **`LongAdder` (Java 8+):** Maintains a base value and an array of variables (`Cell[]`). Threads hash to distinct cells, reducing cache contention. Values are summed only upon calling `sum()`. Ideal for high-throughput counters and statistics.

```java
public class MetricsCollector {
    // Highly scalable under high contention
    private final LongAdder requestCounter = new LongAdder();

    public void recordRequest() {
        requestCounter.increment(); // Lock-free striping across cells
    }

    long getTotal() {
        return requestCounter.sum();
    }
}
```

### Modern Java: `VarHandle` (Java 9+)

Introduced in Java 9, `VarHandle` replaces and generalizes `sun.misc.Unsafe` features, offering standard, type-safe, and safe reflection-free access to fields and array elements with atomic, acquire-release, and volatile semantics.

```java
public class VarHandleDemo {
    private int score = 0;
    private static final VarHandle SCORE_HANDLE;

    static {
        try {
            MethodHandles.Lookup l = MethodHandles.lookup();
            SCORE_HANDLE = l.findVarHandle(VarHandleDemo.class, "score", int.class);
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    public boolean updateScore(int expected, int newValue) {
        // Atomic CAS via VarHandle
        return SCORE_HANDLE.compareAndSet(this, expected, newValue);
    }
}
```

---

## ⚠️ Gotchas & Pitfalls

1. **CAS Starvation (Livelock):** Under extreme contention, a thread executing a CAS retry loop can spin indefinitely while other threads continually win the CAS race.
2. **Ignoring ABA in Lock-Free Data Structures:** Using plain `AtomicReference` in lock-free linked lists or stacks where nodes are recycled leads to silent memory corruption and infinite loops.
3. **Misusing `LongAdder` for Exact Instantaneous Reads:** `LongAdder.sum()` is **not** atomically snapshotting all cells simultaneously if concurrent updates occur during summation. Do not use it when exact sequential consistency is mandatory.
4. **False Sharing:** Placing multiple atomic variables adjacent in an object layout causes them to share the same CPU cache line (typically 64 bytes), leading to cross-core cache invalidation storms. (Resolved in Java 8+ via `@Contended`).

---

## ✅ Best Practices

- Use `AtomicInteger`/`AtomicLong` for low-to-moderate contention counters and flags.
- Switch to `LongAdder` or `DoubleAdder` when write contention is extremely high (e.g., metrics, hit counters).
- Use `AtomicStampedReference` or `AtomicMarkableReference` whenever object graph mutations or lock-free node recycling are implemented.
- Prefer `VarHandle` over `sun.misc.Unsafe` when implementing custom high-performance lock-free data structures in modern Java.

---

## 🎯 Interviewer Insight

Interviewers expect you to explain both the mechanics of CAS and its limitations. Strong candidates describe the **ABA problem** and how `AtomicStampedReference` solves it, contrast `AtomicLong` with `LongAdder` under high thread contention, and demonstrate familiarity with low-level primitives like `VarHandle`.

## ⚡ Follow-up Questions

1. How does the CPU implement CAS at the hardware bus level (e.g., cache locking vs bus locking)?
2. What is the ABA problem, and in what specific data structures does it cause silent corruption?
3. How does `LongAdder` achieve superior throughput compared to `AtomicLong` under heavy contention?
4. What is **false sharing**, and how does the `@Contended` annotation mitigate it?
5. How do `VarHandle` memory access modes (volatile, acquire, release, plain) differ from traditional volatile fields?

## 📚 Related Topics

- [Volatile Keyword](./08-volatile-keyword.md)
- [Reentrant Lock](./10-reentrant-lock.md)
- [Concurrent Collections](./17-concurrent-collections.md)

---

[← Volatile Keyword](./08-volatile-keyword.md) | [Next: Reentrant Lock →](./10-reentrant-lock.md)

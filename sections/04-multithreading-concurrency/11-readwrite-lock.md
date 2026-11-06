# Q11: How Does ReadWriteLock Work?

[← Back to Multithreading & Concurrency](./README.md)

---

## 📋 Question

What is `ReadWriteLock`? How does `ReentrantReadWriteLock` work, when should you use it, how do lock downgrading and reader/writer starvation policies operate, and what is `StampedLock`?

## Difficulty: 🟡 Intermediate

---

## 💡 Answer

### The Readers-Writers Problem

Standard mutual exclusion locks (`synchronized` or `ReentrantLock`) allow only one thread at a time into a critical section, even if the operation is purely read-only and thread-safe. When read operations vastly outnumber write operations, exclusive locking creates severe throughput bottlenecks.

`ReadWriteLock` decouples read access from write access:
- **Multiple readers** can hold the read lock concurrently.
- **Writers** require exclusive access (no other readers or writers allowed).

```
Exclusive Lock (synchronized):      ReadWriteLock:
  Reader A ──LOCK──read──UNLOCK──     Reader A ──RLOCK──read──RUNLOCK──
  Reader B ──WAIT...──LOCK──        Reader B ──RLOCK──read──RUNLOCK── (Concurrent!)
  Writer X ──WAIT...──LOCK──        Writer X ──WAIT...──WLOCK──write──
```

### `ReentrantReadWriteLock` Architecture

```java
public class ThreadSafeCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    public V get(K key) {
        readLock.lock(); // Shared read lock
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }

    public V put(K key, V value) {
        writeLock.lock(); // Exclusive write lock
        try {
            return cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}
```

#### Lock Compatibility Matrix

| Current \ Requesting | Read Lock (Shared) | Write Lock (Exclusive) |
|----------------------|--------------------|------------------------|
| **Read Lock** | ✅ Allowed | ❌ Blocked |
| **Write Lock** | ❌ Blocked | ❌ Blocked |

### Fairness Policies & Writer Starvation

`ReentrantReadWriteLock` supports two fairness settings (`new ReentrantReadWriteLock(true)`):
- **Unfair (Default):** Higher throughput. Readers can jump ahead of waiting writers if read locks are active, which can cause **writer starvation** if read requests arrive continuously.
- **Fair:** Queued threads acquire locks in FIFO order. If a writer is waiting, subsequent incoming readers are queued behind it, preventing writer starvation at the expense of throughput.

### Lock Downgrading (Write → Read)

`ReentrantReadWriteLock` supports **lock downgrading**: a thread holding the write lock can acquire the read lock, and then release the write lock, safely transitioning from exclusive writer to shared reader without losing data visibility.

```java
public class LockDowngradeDemo {
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    private Object data = "Initial";
    private volatile boolean changed = false;

    public void updateAndRead() {
        writeLock.lock(); // 1. Acquire write lock
        try {
            if (changed) {
                data = "Updated Data";
                changed = false;
            }
            readLock.lock(); // 2. Downgrade by acquiring read lock while holding write lock
        } finally {
            writeLock.unlock(); // 3. Release write lock; retain read lock
        }

        try {
            System.out.println("Reading safely under shared read lock: " + data);
        } finally {
            readLock.unlock(); // 4. Release read lock
        }
    }
}
```
*Note:* Lock upgrading (Read → Write) is **not** supported and will deadlock if attempted directly.

### Modern Alternative: `StampedLock` (Java 8+)

Introduced in Java 8, `StampedLock` controls read/write access with return **stamps**. Unlike `ReentrantReadWriteLock`, it introduces **Optimistic Locking**:
- Reads do not acquire any lock or modify state; instead, they read data and validate whether a write occurred in the interim using a stamp validation check (`validate(stamp)`).
- If validation fails, the reader falls back to acquiring a standard shared read lock.

```java
public class Point {
    private double x, y;
    private final StampedLock lock = new StampedLock();

    // Optimistic Read - extremely fast, lock-free
    public double distanceToOrigin() {
        long stamp = lock.tryOptimisticRead(); // Non-blocking stamp
        double currentX = x;
        double currentY = y;

        // Check if any writer intervened during our read
        if (!lock.validate(stamp)) {
            stamp = lock.readLock(); // Fallback to traditional read lock
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return Math.hypot(currentX, currentY);
    }

    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock(); // Exclusive write lock
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

---

## ⚠️ Gotchas & Pitfalls

1. **Writer Starvation in Unfair Mode:** Under heavy read concurrency, an unfair `ReentrantReadWriteLock` can starve waiting writers indefinitely.
2. **Attempting Lock Upgrading:** Trying to acquire a write lock while holding a read lock will block forever (deadlock). You must fully release the read lock before acquiring a write lock.
3. **`StampedLock` Reentrancy Limitation:** `StampedLock` **is not reentrant**. If a thread holding a write lock attempts to acquire another write lock, it will deadlock.
4. **Incorrect Optimistic Read Fallback:** Forgetting to check `lock.validate(stamp)` or omitting the fallback read lock results in reading inconsistent, half-mutated multi-variable states.

---

## ✅ Best Practices

- Use `ReentrantReadWriteLock` when read operations dominate write operations (e.g., 90%+ reads) and critical sections are long or complex.
- Prefer `StampedLock` for read-heavy coordinate or numerical state objects where optimistic reads eliminate lock acquisition overhead entirely.
- Remember that `StampedLock` does not support reentrancy or conditions—use `ReentrantReadWriteLock` if those features are required.

---

## 🎯 Interviewer Insight

Interviewers use `ReadWriteLock` and `StampedLock` to test your ability to scale high-concurrency read scenarios. Strong candidates contrast shared vs. exclusive lock matrices, explain writer starvation risks, demonstrate lock downgrading, and explain how `StampedLock` optimistic reads bypass locking overhead through stamp validation.

## ⚡ Follow-up Questions

1. What is the difference between reader and writer starvation in `ReentrantReadWriteLock`, and how does fair mode resolve it?
2. Why does `ReentrantReadWriteLock` support lock downgrading but prohibit lock upgrading?
3. How does `StampedLock` optimistic reading work, and when does it outperform standard read locks?
4. Why is `StampedLock` not reentrant?
5. What happens if a thread holding a `StampedLock` optimistic read is garbage collected or blocked?

## 📚 Related Topics

- [Reentrant Lock](./10-reentrant-lock.md)
- [Synchronized Keyword](./07-synchronized-keyword.md)
- [Concurrent Collections](./17-concurrent-collections.md)

---

[← Reentrant Lock](./10-reentrant-lock.md) | [Next: Condition Variables →](./12-condition-variables.md)

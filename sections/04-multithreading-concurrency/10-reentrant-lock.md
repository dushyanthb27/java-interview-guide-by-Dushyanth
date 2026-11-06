# Q10: Explain ReentrantLock and Its Advantages

[← Back to Multithreading & Concurrency](./README.md)

---

## 📋 Question

How does `ReentrantLock` work? What advantages does it have over `synchronized`? How do AQS (AbstractQueuedSynchronizer), fair vs unfair locking modes, interruptible lock acquisition, and condition variables operate under the hood?

## Difficulty: 🟡 Intermediate

---

## 💡 Answer

### `ReentrantLock` vs `synchronized`

While `synchronized` is built into the JVM monitor mechanism, `ReentrantLock` is a flexible, API-driven locking implementation provided in `java.util.concurrent.locks`.

| Feature | `synchronized` | `ReentrantLock` |
|---------|---------------|-----------------|
| **Acquisition & Release** | Implicit (block entry/exit) | Explicit (`lock()` / `unlock()`) |
| **Try-Lock (Non-blocking)** | ❌ No | ✅ `tryLock()` |
| **Timed Lock Acquisition** | ❌ No | ✅ `tryLock(timeout, unit)` |
| **Interruptible Acquisition**| ❌ No | ✅ `lockInterruptibly()` |
| **Fairness Configuration** | ❌ Unfair only | ✅ Fair or Unfair constructor (`new ReentrantLock(true)`) |
| **Condition Variables** | 1 per object (`wait`/`notify`) | Multiple per lock (`newCondition()`) |
| **Inspection APIs** | ❌ Limited | ✅ `hasQueuedThreads()`, `getHoldCount()`, etc. |

### Basic Usage Pattern

```java
public class ReentrantLockBasics {
    private final ReentrantLock lock = new ReentrantLock();
    private int balance = 1000;

    public void withdraw(int amount) {
        lock.lock(); // Explicit acquisition
        try {
            if (balance >= amount) {
                balance -= amount;
            }
        } finally {
            lock.unlock(); // MUST be in finally to avoid permanent lock leaks!
        }
    }
}
```

### Advanced Features

#### 1. Non-Blocking and Timed Lock Acquisition (`tryLock`)
Allows threads to attempt lock acquisition without blocking indefinitely, preventing deadlocks when acquiring multiple locks.

```java
public boolean transferFunds(Account target, int amount) throws InterruptedException {
    long timeoutNanos = TimeUnit.SECONDS.toNanos(3);
    if (this.lock.tryLock()) {
        try {
            if (target.lock.tryLock(timeoutNanos, TimeUnit.NANOSECONDS)) {
                try {
                    this.balance -= amount;
                    target.balance += amount;
                    return true;
                } finally {
                    target.lock.unlock();
                }
            }
        } finally {
            this.lock.unlock();
        }
    }
    return false; // Could not acquire both locks atomically
}
```

#### 2. Interruptible Lock Acquisition (`lockInterruptibly`)
Unlike `synchronized`, where a blocked thread cannot be interrupted while waiting for a lock, `lockInterruptibly()` allows a waiting thread to respond to `Thread.interrupt()`.

```java
public void performCancellableWork() throws InterruptedException {
    lock.lockInterruptibly(); // Throws InterruptedException if interrupted while waiting
    try {
        // Critical section
    } finally {
        lock.unlock();
    }
}
```

### Under the Hood: AbstractQueuedSynchronizer (AQS)

`ReentrantLock` is built on top of **`AbstractQueuedSynchronizer` (AQS)**, a framework for building locks and synchronization primitives using:
1. An atomic integer state variable (`getState()`, `setState()`, `compareAndSetState()`) representing lock acquisition status/hold count.
2. A **FIFO CLH (Craig, Landin, and Hagersten) wait queue** of queued threads waiting for the lock.
3. Lock parking/unparking via `LockSupport.park(thread)` and `LockSupport.unpark(thread)`.

```
AQS FIFO Queue:
[Head] ◄─── [Thread B (Queued)] ◄─── [Thread C (Queued)] ◄─── [Tail]
```

### Fair vs. Unfair Locking

`ReentrantLock` supports two fairness policies:
- **Unfair (Default):** A newly arriving thread can "barges" and grab the lock immediately if it becomes available, bypassing threads waiting in the AQS queue. **Advantage:** Higher throughput because it minimizes thread context switching. **Risk:** Potential thread starvation.
- **Fair (`new ReentrantLock(true)`):** Threads acquire the lock in strict FIFO order from the AQS queue. When a lock is released, it is handed to the longest-waiting thread. **Advantage:** No starvation. **Disadvantage:** Significantly lower throughput due to massive thread suspension and context-switching overhead.

---

## ⚠️ Gotchas & Pitfalls

1. **Forgetting the `try-finally` Unlock Pattern:** Placing `lock.unlock()` outside the `try` block means that if an exception occurs during the critical section, the lock is never released, causing permanent deadlock for all other threads.
2. **Reentrancy Deadlocks with Conditions:** Because `ReentrantLock` is reentrant, a thread must call `unlock()` as many times as it called `lock()` to fully release the lock. Releasing only once while holding multiple reentrant acquisitions leaves the lock locked.
3. **Assuming Fair Locks Eliminate All Livelocks:** While fair locks prevent thread starvation in queue order, they do not guarantee freedom from higher-level application deadlocks or livelocks.
4. **Performance Degradation with Fair Locks:** Enabling fairness (`fair=true`) introduces heavy context-switching overhead; do not enable it unless strict FIFO ordering is explicitly required.

---

## ✅ Best Practices

- Always write `lock.lock(); try { ... } finally { lock.unlock(); }` immediately on consecutive lines without any intervening code that could throw an exception before the `try` block.
- Stick to the default **unfair lock** unless strict fairness guarantees are mandated by business requirements.
- Use `tryLock()` with randomized backoff intervals as a robust deadlock-avoidance strategy when acquiring multiple locks.

---

## 🎯 Interviewer Insight

Interviewers frequently probe beneath the surface of `ReentrantLock` API features to test your knowledge of **AQS (AbstractQueuedSynchronizer)**, fairness trade-offs, and comparison against `synchronized`. Strong candidates explain how AQS uses a volatile state variable and a FIFO CLH wait queue, contrast fair vs. unfair throughput implications, and cite practical use cases for `tryLock` and `lockInterruptibly`.

## ⚡ Follow-up Questions

1. How does `AbstractQueuedSynchronizer` (AQS) manage the FIFO wait queue and thread parking/unparking?
2. What are the performance trade-offs between fair and unfair locking in `ReentrantLock`?
3. How does `lockInterruptibly()` differ from standard lock acquisition in `synchronized`?
4. Can a `ReentrantLock` be used across different methods or objects?
5. How does AQS implement reentrancy for `ReentrantLock`?

## 📚 Related Topics

- [Synchronized Keyword](./07-synchronized-keyword.md)
- [ReadWriteLock](./11-readwrite-lock.md)
- [Condition Variables](./12-condition-variables.md)

---

[← Atomic Classes](./09-atomic-classes.md) | [Next: ReadWriteLock →](./11-readwrite-lock.md)

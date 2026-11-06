# Q12: What Are Condition Variables?

[← Back to Multithreading & Concurrency](./README.md)

---

## 📋 Question

What are `Condition` variables in Java? How do they differ from traditional `wait()`/`notify()` mechanisms on object monitors? How do you implement advanced synchronization patterns (like a bounded buffer) using multiple condition queues?

## Difficulty: 🟡 Intermediate

---

## 💡 Answer

### The `Condition` Interface

A `Condition` variable (associated with a `Lock` via `lock.newCondition()`) provides a sophisticated replacement for `Object.wait()`, `notify()`, and `notifyAll()`. 

While intrinsic locks have only **a single implicit wait-set per object monitor**, `ReentrantLock` allows you to create **multiple distinct condition wait-sets** per lock. This solves the classic "wakeup ambiguity" problem where calling `notifyAll()` wakes up every waiting thread (both producers and consumers) even when only items or space became available.

```
Object Monitor (wait/notify):        Condition (await/signal):
┌─────────────────────────┐         ┌─────────────────────────┐
│ synchronized(lock)      │         │ lock.lock()             │
│                         │         │                         │
│ ONE shared wait-set     │         │ Condition notFull       │
│ ┌─────────────────────┐ │         │ ┌─────────────────────┐ │
│ │ wait() / notifyAll()│ │         │ │ await() / signal()  │ │
│ └─────────────────────┘ │         │ └─────────────────────┘ │
│                         │         │                         │
│                         │         │ Condition notEmpty      │
│                         │         │ ┌─────────────────────┐ │
│                         │         │ │ await() / signal()  │ │
│                         │         │ └─────────────────────┘ │
└─────────────────────────┘         └─────────────────────────┘
```

### `Condition` vs. `wait()` / `notify()`

| Feature | `Object.wait()` / `notify()` | `Condition.await()` / `signal()` |
|---------|------------------------------|----------------------------------|
| **Associated Lock** | Implicit object monitor (`synchronized`) | Explicit `Lock` (`ReentrantLock`) |
| **Wait-Sets per Lock** | Exactly 1 | Multiple (independent condition queues) |
| **Method Names** | `wait()`, `notify()`, `notifyAll()` | `await()`, `signal()`, `signalAll()` |
| **Timed Waits** | `wait(long millis)` | `await(long time, TimeUnit unit)`, `awaitNanos()`, `awaitUntil()` |
| **Interruptible Waits** | `wait()` throws `InterruptedException` | `await()` (throws) or `awaitUninterruptibly()` |
| **Precision** | Milliseconds | Nanoseconds |

### Classic Producer-Consumer Bounded Buffer Implementation

Using multiple condition variables (`notFull` and `notEmpty`) enables precise wakeups: producers only wake producers or consumers when space is freed, and consumers only wake producers when items are consumed.

```java
public class BoundedBuffer<T> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition(); // Producers wait here
    private final Condition notEmpty = lock.newCondition(); // Consumers wait here

    private final Object[] items;
    private int putptr, takeptr, count;

    public BoundedBuffer(int capacity) {
        items = new Object[capacity];
    }

    public void put(T x) throws InterruptedException {
        lock.lock();
        try {
            // Always check condition in a while loop to guard against spurious wakeups
            while (count == items.length) {
                notFull.await(); // Releases lock atomically and suspends thread
            }
            items[putptr] = x;
            if (++putptr == items.length) putptr = 0;
            ++count;
            
            notEmpty.signal(); // Wake up exactly ONE waiting consumer
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await(); // Releases lock and waits for items
            }
            T x = (T) items[takeptr];
            items[takeptr] = null;
            if (++takeptr == items.length) takeptr = 0;
            --count;

            notFull.signal(); // Wake up exactly ONE waiting producer
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```

### Key Behavioral Rules

1. **Lock Ownership:** A thread **must** hold the associated `ReentrantLock` before calling `await()`, `signal()`, or `signalAll()`. Failing to do so throws an `IllegalMonitorStateException`.
2. **Atomic Release:** Calling `await()` atomically releases the underlying lock and suspends the current thread, placing it in the condition queue. When the thread is signaled and wakes up, it automatically re-acquires the lock before returning from `await()`.
3. **The `while` Loop Guard:** Always wrap `await()` in a `while` check evaluating the predicate. This guards against **spurious wakeups** (where a thread wakes up without an explicit signal) and race conditions where state changes between notification and thread resumption.

---

## ⚠️ Gotchas & Pitfalls

1. **Forgetting the `while` Loop Guard:** Using an `if` statement instead of a `while` loop around `await()` leads to silent data corruption or IndexOutOfBounds exceptions when spurious wakeups occur or multiple threads contend for state.
2. **Calling Condition Methods Without Holding the Lock:** Invoking `await()` or `signal()` without holding the parent `ReentrantLock` throws `IllegalMonitorStateException`.
3. **Deadlocks via Incorrect Signaling (`signal()` vs `signalAll()`):** Using `signal()` when multiple threads are waiting for different predicates can cause missed signals and permanent thread hangs. Use `signalAll()` when multiple waiting threads might qualify for resumption.
4. **InterruptedException Handling:** When `await()` throws `InterruptedException`, the thread has automatically re-acquired the lock upon waking. You must ensure the lock is correctly released in a `finally` block, or that the interruption is properly propagated.

---

## ✅ Best Practices

- Always pair `await()` with a `while(!predicate)` loop check.
- Use multiple `Condition` instances per lock (`notFull`, `notEmpty`) to separate distinct waiting semantics and avoid unnecessary `signalAll()` broadcasts.
- Always execute lock acquisition, condition waiting, and state modification within strict `try-finally` blocks.

---

## 🎯 Interviewer Insight

Interviewers use condition variables to evaluate your mastery of advanced thread coordination. Strong candidates explain why multiple condition queues surpass intrinsic `wait`/`notify` (avoiding thundering herds and wakeup ambiguity), write robust bounded buffer implementations with `while` loop predicates, and demonstrate precise lock ownership requirements.

## ⚡ Follow-up Questions

1. Why must `await()` always be wrapped in a `while` loop rather than an `if` statement?
2. What happens to the lock when a thread invokes `condition.await()`?
3. How do multiple `Condition` instances attached to a single `ReentrantLock` improve producer-consumer efficiency?
4. What is the difference between `signal()` and `signalAll()`, and when can using `signal()` incorrectly cause a deadlock?
5. How does `awaitNanos()` assist in implementing timeout-based waiting patterns?

## 📚 Related Topics

- [Reentrant Lock](./10-reentrant-lock.md)
- [Synchronized Keyword](./07-synchronized-keyword.md)
- [Thread Communication](./14-thread-communication.md)

---

[← ReadWriteLock](./11-readwrite-lock.md) | [Next: Deadlock & Livelock →](./13-deadlock.md)

# Q10: Explain ReentrantLock and Its Advantages

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
How does `ReentrantLock` work? What advantages does it have over `synchronized`?

---

## Answer

### ReentrantLock vs Synchronized

```
┌────────────────────────┬──────────────────┬──────────────────┐
│ Feature                │  synchronized    │  ReentrantLock   │
├────────────────────────┼──────────────────┼──────────────────┤
│ Lock acquisition       │  Automatic       │  Explicit        │
│ Lock release           │  Automatic       │  Manual (finally)│
│ Try lock (non-block)   │  ❌              │  ✅ tryLock()    │
│ Timed lock             │  ❌              │  ✅ tryLock(t)   │
│ Interruptible lock     │  ❌              │  ✅ lockInterr.. │
│ Fair ordering          │  ❌              │  ✅ fair=true    │
│ Multiple conditions    │  ❌ (1 per obj)  │  ✅ newCondition │
│ Lock across methods    │  ❌              │  ✅              │
│ Performance            │  Optimized(JDK6+)│  Comparable      │
│ Syntax                 │  Simpler         │  More verbose    │
└────────────────────────┴──────────────────┴──────────────────┘
```

### Basic Usage

```java
public class ReentrantLockBasics {
    private final ReentrantLock lock = new ReentrantLock();
    private int balance = 1000;

    // ✅ Always use try-finally pattern
    public void withdraw(int amount) {
        lock.lock();
        try {
            if (balance >= amount) {
                balance -= amount;
                System.out.println("Withdrew " + amount + ", balance: " + balance);
            } else {
                System.out.println("Insufficient funds");
            }
        } finally {
            lock.unlock(); // MUST unlock in finally!
        }
    }

    public int getBalance() {
        lock.lock();
        try {
            return balance;
        } finally {
            lock.unlock();
        }
    }
}
```

### tryLock - Non-Blocking Acquisition

```java
public class TryLockDemo {
    private final ReentrantLock lock = new ReentrantLock();
    private final List<String> sharedList = new ArrayList<>();

    // Non-blocking: returns immediately if lock unavailable
    public boolean tryAdd(String item) {
        if (lock.tryLock()) {
            try {
                sharedList.add(item);
                return true;
            } finally {
                lock.unlock();
            }
        }
        System.out.println("Lock busy, skipping operation");
        return false;
    }

    // Timed: waits up to specified time
    public boolean tryAddWithTimeout(String item) throws InterruptedException {
        if (lock.tryLock(2, TimeUnit.SECONDS)) {
            try {
                sharedList.add(item);
                return true;
            } finally {
                lock.unlock();
            }
        }
        System.out.println("Could not acquire lock within 2 seconds");
        return false;
    }

    // ✅ Deadlock avoidance with tryLock
    public static void transferSafe(TryLockDemo from, TryLockDemo to, String item) {
        boolean fromLocked = false;
        boolean toLocked = false;
        try {
            fromLocked = from.lock.tryLock(1, TimeUnit.SECONDS);
            toLocked = to.lock.tryLock(1, TimeUnit.SECONDS);

            if (fromLocked && toLocked) {
                from.sharedList.remove(item);
                to.sharedList.add(item);
                System.out.println("Transfer successful");
            } else {
                System.out.println("Could not acquire both locks - retry later");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (fromLocked) from.lock.unlock();
            if (toLocked) to.lock.unlock();
        }
    }
}
```

### Fair vs Unfair Locks

```java
public class FairLockDemo {
    // Unfair lock (default) - better throughput
    private final ReentrantLock unfairLock = new ReentrantLock(false);

    // Fair lock - FIFO ordering, prevents starvation
    private final ReentrantLock fairLock = new ReentrantLock(true);

    /*
     * Unfair lock:
     *   Thread A releases lock
     *   Thread C (just arriving) can "barge" ahead of waiting Thread B
     *   Higher throughput but possible starvation
     *
     * Fair lock:
     *   Thread A releases lock
     *   Thread B (longest waiting) always gets it next
     *   Lower throughput but guaranteed fairness
     *
     * Performance: Fair locks are ~2-10x slower under contention
     */

    public void fairOperation() {
        fairLock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " acquired fair lock");
        } finally {
            fairLock.unlock();
        }
    }
}
```

### Interruptible Lock Acquisition

```java
public class InterruptibleLockDemo {
    private final ReentrantLock lock = new ReentrantLock();

    // ❌ synchronized blocks cannot be interrupted while waiting for lock
    // public synchronized void blocking() { ... }

    // ✅ ReentrantLock supports interruption
    public void interruptibleOperation() throws InterruptedException {
        lock.lockInterruptibly(); // Throws InterruptedException if interrupted while waiting
        try {
            System.out.println("Performing operation...");
            Thread.sleep(5000);
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        InterruptibleLockDemo demo = new InterruptibleLockDemo();

        // Thread 1 holds the lock
        Thread t1 = new Thread(() -> {
            demo.lock.lock();
            try {
                Thread.sleep(10000); // Hold lock for 10 seconds
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                demo.lock.unlock();
            }
        });

        // Thread 2 tries to acquire interruptibly
        Thread t2 = new Thread(() -> {
            try {
                demo.interruptibleOperation();
            } catch (InterruptedException e) {
                System.out.println("T2 was interrupted while waiting for lock!");
            }
        });

        t1.start();
        Thread.sleep(100); // Let t1 acquire lock
        t2.start();
        Thread.sleep(1000);
        t2.interrupt(); // Interrupt t2 while it waits for the lock
        // T2 will throw InterruptedException and respond immediately!
    }
}
```

### Reentrancy

```java
public class ReentrancyDemo {
    private final ReentrantLock lock = new ReentrantLock();

    public void outer() {
        lock.lock(); // Hold count = 1
        try {
            System.out.println("In outer, hold count: " + lock.getHoldCount());
            inner(); // Same thread acquires again
        } finally {
            lock.unlock(); // Hold count = 0
        }
    }

    public void inner() {
        lock.lock(); // Hold count = 2 (reentrant)
        try {
            System.out.println("In inner, hold count: " + lock.getHoldCount());
        } finally {
            lock.unlock(); // Hold count = 1
        }
    }
}
```

### Condition Objects

```java
public class BoundedBuffer<T> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    private final Object[] items;
    private int count = 0, putIndex = 0, takeIndex = 0;

    public BoundedBuffer(int capacity) {
        items = new Object[capacity];
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await(); // Wait until buffer is not full
            }
            items[putIndex] = item;
            putIndex = (putIndex + 1) % items.length;
            count++;
            notEmpty.signal(); // Signal waiting consumers
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await(); // Wait until buffer is not empty
            }
            T item = (T) items[takeIndex];
            takeIndex = (takeIndex + 1) % items.length;
            count--;
            notFull.signal(); // Signal waiting producers
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

### Lock Diagnostic Methods

```java
public class LockDiagnostics {
    private final ReentrantLock lock = new ReentrantLock();

    public void diagnostics() {
        lock.lock();
        try {
            System.out.println("Is locked: " + lock.isLocked());
            System.out.println("Is held by current thread: " + lock.isHeldByCurrentThread());
            System.out.println("Hold count: " + lock.getHoldCount());
            System.out.println("Has queued threads: " + lock.hasQueuedThreads());
            System.out.println("Queue length: " + lock.getQueueLength());
            System.out.println("Is fair: " + lock.isFair());
        } finally {
            lock.unlock();
        }
    }
}
```

---

## Anti-Patterns

❌ **Forgetting to unlock in finally:**
```java
lock.lock();
doWork(); // If this throws, lock is NEVER released → deadlock!
lock.unlock();
```

❌ **Locking and unlocking in different methods without careful design:**
```java
public void start() { lock.lock(); }   // Who calls unlock?
public void end() { lock.unlock(); }   // ❌ Error-prone, hard to reason about
```

❌ **Using fair locks without need:**
```java
new ReentrantLock(true); // ❌ Significant performance overhead if fairness isn't required
```

## Best Practices

✅ **Always** use try-finally pattern with explicit lock/unlock  
✅ Use `tryLock()` to avoid deadlocks in multi-lock scenarios  
✅ Use `lockInterruptibly()` for cancellable operations  
✅ Use `Condition` objects for multiple wait conditions  
✅ Prefer `synchronized` for simple cases; use `ReentrantLock` for advanced features  
✅ Use fair locks only when starvation is a real concern

---

[← Atomic Classes](./q09-atomic-classes.md) | [Next: ReadWriteLock →](./q11-readwrite-lock.md)

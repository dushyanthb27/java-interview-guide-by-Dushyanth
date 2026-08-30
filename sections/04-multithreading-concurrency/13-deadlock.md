# Q13: What is Deadlock and How to Prevent It?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What is deadlock? What are the four necessary conditions? How do you detect and prevent it?

---

## Answer

### What is Deadlock?

Deadlock occurs when two or more threads are **blocked forever**, each waiting for a resource held by the other.

```
┌──────────────┐                    ┌──────────────┐
│   Thread A   │                    │   Thread B   │
│              │   holds Lock 1     │              │
│  Lock 1 ████ │◄───────────────────│  wants Lock 1│
│              │                    │              │
│  wants Lock 2│───────────────────►│  Lock 2 ████ │
│              │   holds Lock 2     │              │
└──────────────┘                    └──────────────┘
         Both threads wait forever! ☠️
```

### Four Necessary Conditions (Coffman Conditions)

**ALL four must be present simultaneously for deadlock:**

| # | Condition | Description |
|---|-----------|-------------|
| 1 | **Mutual Exclusion** | Resource can only be held by one thread at a time |
| 2 | **Hold and Wait** | Thread holds one resource while waiting for another |
| 3 | **No Preemption** | Resources cannot be forcibly taken from a thread |
| 4 | **Circular Wait** | Thread A waits for B, B waits for A (or longer chain) |

### Deadlock Example

```java
public class DeadlockExample {
    private static final Object lockA = new Object();
    private static final Object lockB = new Object();

    public static void main(String[] args) {
        Thread thread1 = new Thread(() -> {
            synchronized (lockA) {
                System.out.println("Thread 1: Holding lockA");
                try { Thread.sleep(100); } catch (InterruptedException e) {}

                System.out.println("Thread 1: Waiting for lockB...");
                synchronized (lockB) { // ☠️ DEADLOCK!
                    System.out.println("Thread 1: Holding lockA & lockB");
                }
            }
        }, "Thread-1");

        Thread thread2 = new Thread(() -> {
            synchronized (lockB) {
                System.out.println("Thread 2: Holding lockB");
                try { Thread.sleep(100); } catch (InterruptedException e) {}

                System.out.println("Thread 2: Waiting for lockA...");
                synchronized (lockA) { // ☠️ DEADLOCK!
                    System.out.println("Thread 2: Holding lockB & lockA");
                }
            }
        }, "Thread-2");

        thread1.start();
        thread2.start();
    }
}
```

### Deadlock Detection

#### Using jstack / Thread Dumps

```bash
# Get thread dump
jstack <pid>

# Output:
# Found one Java-level deadlock:
# =============================
# "Thread-1":
#   waiting to lock monitor 0x00007f...  (object 0x000000076...)
#   which is held by "Thread-2"
# "Thread-2":
#   waiting to lock monitor 0x00007f...  (object 0x000000076...)
#   which is held by "Thread-1"
```

#### Programmatic Detection

```java
public class DeadlockDetector {
    public static void detectDeadlocks() {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();

        // Check for deadlocks
        long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();
        // Also: findMonitorDeadlockedThreads() for synchronized-only deadlocks

        if (deadlockedThreads != null) {
            System.out.println("☠️ DEADLOCK DETECTED!");
            ThreadInfo[] threadInfos = threadMXBean.getThreadInfo(deadlockedThreads, true, true);

            for (ThreadInfo info : threadInfos) {
                System.out.println("Thread: " + info.getThreadName());
                System.out.println("  State: " + info.getThreadState());
                System.out.println("  Waiting for: " + info.getLockName());
                System.out.println("  Held by: " + info.getLockOwnerName());

                for (StackTraceElement ste : info.getStackTrace()) {
                    System.out.println("    at " + ste);
                }
            }
        } else {
            System.out.println("✅ No deadlocks detected");
        }
    }

    // Periodic deadlock monitoring
    public static void startMonitoring() {
        ScheduledExecutorService monitor = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "Deadlock-Monitor");
            t.setDaemon(true);
            return t;
        });

        monitor.scheduleAtFixedRate(DeadlockDetector::detectDeadlocks,
            10, 10, TimeUnit.SECONDS);
    }
}
```

### Prevention Strategies

#### Strategy 1: Consistent Lock Ordering

```java
// ❌ Inconsistent ordering → deadlock risk
public void transferBad(Account from, Account to, double amount) {
    synchronized (from) {
        synchronized (to) { // Different threads may lock in opposite order!
            from.debit(amount);
            to.credit(amount);
        }
    }
}

// ✅ Consistent ordering → no circular wait
public void transferSafe(Account from, Account to, double amount) {
    // Always lock the account with the smaller ID first
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;

    synchronized (first) {
        synchronized (second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}

// Alternative: Use System.identityHashCode for ordering
public void transferByHash(Account from, Account to, double amount) {
    int fromHash = System.identityHashCode(from);
    int toHash = System.identityHashCode(to);

    Object first, second;
    if (fromHash < toHash) {
        first = from; second = to;
    } else if (fromHash > toHash) {
        first = to; second = from;
    } else {
        // Hash collision: use a tie-breaking lock
        synchronized (Account.class) {
            synchronized (from) {
                synchronized (to) {
                    from.debit(amount);
                    to.credit(amount);
                }
            }
        }
        return;
    }

    synchronized (first) {
        synchronized (second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

#### Strategy 2: Timeout-Based Lock Acquisition

```java
public class TimeoutLocking {
    private final ReentrantLock lockA = new ReentrantLock();
    private final ReentrantLock lockB = new ReentrantLock();

    // ✅ tryLock with timeout - prevents indefinite blocking
    public boolean transferWithTimeout() throws InterruptedException {
        boolean gotA = false;
        boolean gotB = false;

        try {
            gotA = lockA.tryLock(1, TimeUnit.SECONDS);
            gotB = lockB.tryLock(1, TimeUnit.SECONDS);

            if (gotA && gotB) {
                // Perform transfer
                return true;
            }
            // Could not acquire both locks
            return false;
        } finally {
            if (gotB) lockB.unlock();
            if (gotA) lockA.unlock();
        }
    }

    // With retry and backoff
    public boolean transferWithRetry(int maxRetries) throws InterruptedException {
        for (int i = 0; i < maxRetries; i++) {
            if (transferWithTimeout()) {
                return true;
            }
            // Random backoff to reduce contention
            Thread.sleep(ThreadLocalRandom.current().nextInt(10, 100));
        }
        return false;
    }
}
```

#### Strategy 3: Single Lock (Coarse-Grained)

```java
// ✅ One lock for everything - no deadlock possible
// ❌ But reduces parallelism
public class SingleLockApproach {
    private final Object globalLock = new Object();

    public void operationA() {
        synchronized (globalLock) {
            // All operations use the same lock
        }
    }

    public void operationB() {
        synchronized (globalLock) {
            // No deadlock possible, but no parallelism either
        }
    }
}
```

### Livelock

Threads are not blocked but keep retrying and making no progress:

```java
public class LivelockExample {
    static class Diner {
        private final String name;
        private boolean isHungry = true;

        Diner(String name) { this.name = name; }

        // Polite diners keep yielding to each other → livelock!
        public void eatWith(Diner partner, Object spoon) {
            while (isHungry) {
                synchronized (spoon) {
                    if (partner.isHungry) {
                        System.out.println(name + ": You go first, " + partner.name);
                        // Both keep yielding → nobody eats!
                        continue;
                    }
                    isHungry = false;
                    System.out.println(name + ": I'm eating!");
                }
            }
        }
    }

    // ✅ Fix: Add randomized backoff
    // Thread.sleep(ThreadLocalRandom.current().nextInt(10));
}
```

### Starvation

A thread never gets CPU time because other higher-priority threads monopolize resources:

```java
// ❌ Starvation risk with unfair locks under high contention
ReentrantLock unfairLock = new ReentrantLock(false); // Default

// ✅ Fair lock prevents starvation (FIFO ordering)
ReentrantLock fairLock = new ReentrantLock(true);
```

### Quick Reference: Deadlock vs Livelock vs Starvation

```
┌──────────────┬─────────────┬─────────────┬─────────────┐
│              │  Deadlock   │  Livelock   │ Starvation  │
├──────────────┼─────────────┼─────────────┼─────────────┤
│ Threads      │  Blocked    │  Active     │  Active     │
│ Progress     │  None       │  None       │  Others yes │
│ Resource     │  Held       │  Released   │  Not given  │
│ Fix          │  Lock order │  Backoff    │  Fairness   │
│              │  Timeout    │  Random     │  Priority   │
└──────────────┴─────────────┴─────────────┴─────────────┘
```

---

## Anti-Patterns

❌ **Acquiring multiple locks in inconsistent order across the codebase**  
❌ **Holding a lock while calling external/unknown code (callbacks, listeners)**  
❌ **Using `synchronized` on public objects (external code can deadlock you)**  
❌ **Nesting too many locks (increases deadlock surface)**

## Best Practices

✅ Establish and document a global lock ordering  
✅ Use `tryLock()` with timeouts for multi-lock scenarios  
✅ Keep lock scope as small as possible  
✅ Use higher-level concurrency utilities (`ConcurrentHashMap`, `BlockingQueue`)  
✅ Set up periodic deadlock detection in production  
✅ Prefer immutable objects (no locks needed!)

---

[← Condition Variables](./q12-condition-variables.md) | [Next: Thread Communication →](./q14-thread-communication.md)

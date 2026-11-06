# Q2: Explain the Thread Lifecycle and States

[← Back to Multithreading & Concurrency](./README.md)

---

## Question

What are the 6 states in a Java thread's lifecycle (`Thread.State`)? How do threads transition between them, and how do modern Java concurrency features and diagnostic tools interact with thread states?

---

## Answer

### Thread States (`java.lang.Thread.State`)

Java defines **6 thread states** in the `Thread.State` enum:

```
              start()
  ┌─────┐  ─────────►  ┌──────────┐
  │ NEW │               │ RUNNABLE │◄──────────────────────┐
  └─────┘               └────┬─────┘                       │
                              │                             │
              ┌───────────────┼──────────────┐              │
              │               │              │              │
              ▼               ▼              ▼              │
        ┌──────────┐   ┌──────────┐   ┌─────────────┐      │
        │ BLOCKED  │   │ WAITING  │   │TIMED_WAITING│      │
        │          │   │          │   │             │      │
        │ waiting  │   │ wait()   │   │ sleep(ms)   │      │
        │ for      │   │ join()   │   │ wait(ms)    │      │
        │ monitor  │   │ park()   │   │ join(ms)    │      │
        │ lock     │   │          │   │ parkNanos() │      │
        └────┬─────┘   └────┬─────┘   └──────┬──────┘      │
             │               │                │             │
             │  lock acquired│  notify/       │  timeout/   │
             │               │  unpark        │  notify     │
             └───────────────┴────────────────┴─────────────┘
                              │
                              ▼
                       ┌──────────────┐
                       │  TERMINATED  │
                       │  (run() ends │
                       │  or exception)│
                       └──────────────┘
```

### Detailed Breakdown of Each State

```java
public class ThreadStateDemo {
    private static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        // 1. NEW - Created via 'new Thread()', but start() not yet called
        Thread thread = new Thread(() -> {});
        System.out.println("After creation: " + thread.getState()); // NEW

        // 2. RUNNABLE - Executing on CPU or waiting in OS ready queue
        Thread runnableThread = new Thread(() -> {
            for (long i = 0; i < 1_000_000L; i++) { /* busy work */ }
        });
        runnableThread.start();
        System.out.println("After start: " + runnableThread.getState()); // RUNNABLE

        // 3. BLOCKED - Awaiting entry to a synchronized monitor block/method
        demonstrateBlocked();

        // 4. WAITING - Indefinitely waiting via Object.wait(), Thread.join(), or LockSupport.park()
        demonstrateWaiting();

        // 5. TIMED_WAITING - Waiting with a timeout (sleep, wait(ms), join(ms), parkNanos)
        demonstrateTimedWaiting();

        // 6. TERMINATED - Execution completed normally or threw an uncaught exception
        runnableThread.join();
        System.out.println("After join: " + runnableThread.getState()); // TERMINATED
    }

    private static void demonstrateBlocked() throws InterruptedException {
        Thread holder = new Thread(() -> {
            synchronized (lock) {
                try { Thread.sleep(2000); } catch (InterruptedException e) {}
            }
        }, "Lock-Holder");

        Thread waiter = new Thread(() -> {
            synchronized (lock) { // BLOCKED here until holder exits block
                System.out.println("Acquired lock!");
            }
        }, "Lock-Waiter");

        holder.start();
        Thread.sleep(100);
        waiter.start();
        Thread.sleep(100);

        System.out.println("Waiter state: " + waiter.getState()); // BLOCKED
        holder.join();
        waiter.join();
    }

    private static void demonstrateWaiting() throws InterruptedException {
        Thread waitingThread = new Thread(() -> {
            synchronized (lock) {
                try {
                    lock.wait(); // Enters WAITING state indefinitely
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "Waiting-Thread");

        waitingThread.start();
        Thread.sleep(100);
        System.out.println("Waiting state: " + waitingThread.getState()); // WAITING

        synchronized (lock) {
            lock.notify();
        }
        waitingThread.join();
    }

    private static void demonstrateTimedWaiting() throws InterruptedException {
        Thread sleepingThread = new Thread(() -> {
            try {
                Thread.sleep(2000); // TIMED_WAITING
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Sleeping-Thread");

        sleepingThread.start();
        Thread.sleep(100);
        System.out.println("Sleeping state: " + sleepingThread.getState()); // TIMED_WAITING
        sleepingThread.join();
    }
}
```

### State Transition Matrix

| From State | To State | Trigger / Cause |
|------------|----------|-----------------|
| `NEW` | `RUNNABLE` | Calling `Thread.start()` |
| `RUNNABLE` | `BLOCKED` | Attempting to enter `synchronized` block/method held by another thread |
| `RUNNABLE` | `WAITING` | Calling `Object.wait()`, `Thread.join()`, or `LockSupport.park()` |
| `RUNNABLE` | `TIMED_WAITING` | Calling `Thread.sleep()`, `Object.wait(ms)`, `Thread.join(ms)`, or `LockSupport.parkNanos()` |
| `RUNNABLE` | `TERMINATED` | Normal return from `run()` or uncaught exception termination |
| `BLOCKED` | `RUNNABLE` | Releasing the monitor lock to the waiting thread |
| `WAITING` | `RUNNABLE` | `Object.notify()`, `notifyAll()`, `LockSupport.unpark()`, or joined thread termination |
| `TIMED_WAITING` | `RUNNABLE` | Timeout expiration, notification, or joined thread termination |

### Virtual Threads and Lifecycle Considerations

In Java 21+, virtual threads (`Thread.ofVirtual()`) decouple Java thread identity from OS threads:
- Virtual threads also pass through `Thread.State` values, but blocking operations (like I/O or `LockSupport.park()`) unmount the virtual thread from its carrier platform thread, leaving the carrier free to run other virtual threads.
- Thread dump tooling (`jcmd <pid> Thread.print`) in Java 21 formats virtual threads hierarchical to task structure or `StructuredTaskScope`.

---

## ⚠️ Gotchas & Common Pitfalls

1. **Busy-Wait Polling**: Polling `thread.getState()` in a tight loop to wait for completion burns 100% CPU. Always use `thread.join()` or concurrency coordination primitives.
2. **BLOCKED vs WAITING Confusion**: A thread waiting on a `ReentrantLock` or `CountDownLatch` is in **`WAITING`** (via `LockSupport.park()`), **NOT** `BLOCKED`. `BLOCKED` is strictly reserved for intrinsic `synchronized` monitor entries.
3. **RUNNABLE Does Not Mean Running**: A `RUNNABLE` thread may be executing instructions on a CPU core **or** sitting in the OS ready queue waiting for CPU scheduler slices.
4. **Deprecated Unsafe Methods**: Using `Thread.stop()` or `Thread.suspend()` corrupts internal monitor states and leaves shared objects half-mutated.
5. **Swallowing Thread Death**: Uncaught exceptions in custom threads terminate the thread silently unless an `UncaughtExceptionHandler` is registered.

---

## Interviewer Expectations & Deep Mental Models

- **Thread Dumps**: Demonstrate fluency in reading thread dumps (`jstack`, `jcmd Thread.print`, or `ThreadMXBean.dumpAllThreads()`) to spot deadlocks (`BLOCKED` waiting for monitor owned by another thread).
- **Monitor Entry Queue**: Explain how the JVM's ObjectMonitor internal VM structure manages the EntryList (for `BLOCKED` threads) and WaitSet (for `WAITING` threads via `wait()`).

---

[← Thread Basics](./01-thread-basics.md) | [Next: Creating Threads →](./03-creating-threads.md)

# Q2: Explain the Thread Lifecycle and States

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What are the different states in a thread's lifecycle? How do threads transition between states?

---

## Answer

### Thread States (java.lang.Thread.State)

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
             │  lock acquired │  notify/      │  timeout/   │
             │               │  unpark       │  notify     │
             └───────────────┴────────────────┴─────────────┘
                              │
                              ▼
                       ┌──────────────┐
                       │  TERMINATED  │
                       │  (run() ends │
                       │  or exception)│
                       └──────────────┘
```

### Each State Explained

```java
public class ThreadStateDemo {
    private static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        // 1. NEW - Thread created but not yet started
        Thread thread = new Thread(() -> {});
        System.out.println("After creation: " + thread.getState()); // NEW

        // 2. RUNNABLE - Thread is executing or ready to execute
        Thread runnableThread = new Thread(() -> {
            // Running on CPU or in the OS ready queue
            for (int i = 0; i < 1_000_000; i++) { /* busy work */ }
        });
        runnableThread.start();
        System.out.println("After start: " + runnableThread.getState()); // RUNNABLE

        // 3. BLOCKED - Waiting to acquire a monitor lock
        demonstrateBlocked();

        // 4. WAITING - Waiting indefinitely for another thread
        demonstrateWaiting();

        // 5. TIMED_WAITING - Waiting with a timeout
        demonstrateTimedWaiting();

        // 6. TERMINATED - Thread has completed execution
        runnableThread.join();
        System.out.println("After join: " + runnableThread.getState()); // TERMINATED
    }

    private static void demonstrateBlocked() throws InterruptedException {
        Object lock = new Object();

        Thread holder = new Thread(() -> {
            synchronized (lock) {
                try { Thread.sleep(2000); } catch (InterruptedException e) {}
            }
        }, "Lock-Holder");

        Thread waiter = new Thread(() -> {
            synchronized (lock) { // Will be BLOCKED until holder releases
                System.out.println("Acquired lock!");
            }
        }, "Lock-Waiter");

        holder.start();
        Thread.sleep(100); // Let holder acquire lock
        waiter.start();
        Thread.sleep(100); // Let waiter attempt to acquire

        System.out.println("Waiter state: " + waiter.getState()); // BLOCKED

        holder.join();
        waiter.join();
    }

    private static void demonstrateWaiting() throws InterruptedException {
        Object monitor = new Object();

        Thread waitingThread = new Thread(() -> {
            synchronized (monitor) {
                try {
                    monitor.wait(); // WAITING - no timeout
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }, "Waiting-Thread");

        waitingThread.start();
        Thread.sleep(100);

        System.out.println("Waiting state: " + waitingThread.getState()); // WAITING

        synchronized (monitor) {
            monitor.notify(); // Wake up the waiting thread
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

### State Transition Table

| From State | To State | Trigger |
|-----------|----------|---------|
| NEW | RUNNABLE | `thread.start()` |
| RUNNABLE | BLOCKED | Attempting to enter `synchronized` block held by another thread |
| RUNNABLE | WAITING | `Object.wait()`, `Thread.join()`, `LockSupport.park()` |
| RUNNABLE | TIMED_WAITING | `Thread.sleep(ms)`, `Object.wait(ms)`, `Thread.join(ms)`, `LockSupport.parkNanos()` |
| RUNNABLE | TERMINATED | `run()` completes normally or throws uncaught exception |
| BLOCKED | RUNNABLE | Monitor lock becomes available |
| WAITING | RUNNABLE | `notify()`, `notifyAll()`, `LockSupport.unpark()`, or joined thread terminates |
| TIMED_WAITING | RUNNABLE | Timeout expires, `notify()`, or joined thread terminates |

### Thread.getState() in Practice

```java
public class ThreadStateMonitor {

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            try {
                // Simulate work phases
                Thread.sleep(500);          // TIMED_WAITING
                synchronized (ThreadStateMonitor.class) {
                    ThreadStateMonitor.class.wait(500); // TIMED_WAITING
                }
                for (int i = 0; i < 10_000_000; i++) {} // RUNNABLE
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Worker");

        // Monitor thread state changes
        Thread monitor = new Thread(() -> {
            Thread.State previousState = null;
            while (worker.getState() != Thread.State.TERMINATED) {
                Thread.State currentState = worker.getState();
                if (currentState != previousState) {
                    System.out.println("Worker state changed: " +
                        previousState + " -> " + currentState);
                    previousState = currentState;
                }
            }
            System.out.println("Worker state changed: " +
                previousState + " -> TERMINATED");
        }, "Monitor");

        System.out.println("Before start: " + worker.getState());
        monitor.start();
        worker.start();

        worker.join();
        monitor.join();
    }
}
```

### Thread Dump Analysis

Thread dumps show the state of all threads - invaluable for debugging:

```
"main" #1 prio=5 os_prio=0 tid=0x00007f... nid=0x1 RUNNABLE
   at com.example.App.processData(App.java:42)
   at com.example.App.main(App.java:15)

"Worker-1" #12 prio=5 os_prio=0 tid=0x00007f... nid=0xc BLOCKED
   waiting for monitor entry [0x00007f...]
   at com.example.SharedResource.update(SharedResource.java:28)
   - waiting to lock <0x000000076...> (a java.lang.Object)
   - locked by "Worker-2" #13

"Worker-2" #13 prio=5 os_prio=0 tid=0x00007f... nid=0xd BLOCKED
   waiting for monitor entry [0x00007f...]
   at com.example.SharedResource.read(SharedResource.java:35)
   - waiting to lock <0x000000076...> (a java.lang.Object)
   - locked by "Worker-1" #12

Found one Java-level deadlock:
=============================
"Worker-1": waiting to lock <0x000000076ab...>
  which is held by "Worker-2"
"Worker-2": waiting to lock <0x000000076cd...>
  which is held by "Worker-1"
```

**How to capture a thread dump:**
```bash
# Method 1: jstack
jstack <pid>

# Method 2: kill signal (Linux/Mac)
kill -3 <pid>

# Method 3: jcmd
jcmd <pid> Thread.print

# Method 4: Programmatic
ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(true, true);
```

```java
// Programmatic thread dump
public class ThreadDumpUtil {
    public static void printThreadDump() {
        ThreadMXBean bean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] infos = bean.dumpAllThreads(true, true);

        for (ThreadInfo info : infos) {
            System.out.println("Thread: " + info.getThreadName());
            System.out.println("  State: " + info.getThreadState());
            System.out.println("  Blocked count: " + info.getBlockedCount());
            System.out.println("  Waited count: " + info.getWaitedCount());

            if (info.getLockName() != null) {
                System.out.println("  Waiting on: " + info.getLockName());
            }

            StackTraceElement[] stack = info.getStackTrace();
            for (StackTraceElement element : stack) {
                System.out.println("    at " + element);
            }
            System.out.println();
        }

        // Detect deadlocks
        long[] deadlockedThreads = bean.findDeadlockedThreads();
        if (deadlockedThreads != null) {
            System.out.println("DEADLOCK DETECTED!");
            ThreadInfo[] deadlockInfos = bean.getThreadInfo(deadlockedThreads);
            for (ThreadInfo info : deadlockInfos) {
                System.out.println("  Deadlocked thread: " + info.getThreadName());
            }
        }
    }
}
```

### Important Notes

> **RUNNABLE does not mean "running":** A RUNNABLE thread can be either actively executing on a CPU core or sitting in the OS ready queue waiting for CPU time. Java does not distinguish between the two.

> **BLOCKED only applies to `synchronized`:** If a thread waits for a `ReentrantLock`, it enters **WAITING** (via `LockSupport.park()`), NOT BLOCKED. Only intrinsic monitor locks cause the BLOCKED state.

---

## Anti-Patterns

❌ **Polling thread state in a tight loop:**
```java
while (thread.getState() != Thread.State.TERMINATED) {
    // Burns CPU! Use thread.join() instead
}
```

❌ **Using Thread.stop() or Thread.suspend() (deprecated):**
```java
thread.stop();    // ❌ Unsafe - can leave objects in inconsistent state
thread.suspend(); // ❌ Deadlock-prone
```

## Best Practices

✅ Use `thread.join()` to wait for thread completion  
✅ Use `Thread.getState()` for monitoring/debugging only, not for control flow  
✅ Use thread dumps (`jstack`, `jcmd`) to diagnose production issues  
✅ Use cooperative interruption (`thread.interrupt()`) to stop threads  
✅ Understand that BLOCKED is for monitors, WAITING is for explicit waits

---

[← Thread Basics](./q01-thread-basics.md) | [Next: Creating Threads →](./q03-creating-threads.md)

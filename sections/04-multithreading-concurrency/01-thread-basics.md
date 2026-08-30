# Q1: What is a Thread and Why Do We Need Multithreading?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What is a thread? How does it differ from a process? Why do we need multithreading in Java?

---

## Answer

### Thread vs Process

```
┌─────────────────────────────────────────────────────────┐
│                      PROCESS                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Thread 1 │  │ Thread 2 │  │ Thread 3 │              │
│  │          │  │          │  │          │              │
│  │ Stack    │  │ Stack    │  │ Stack    │  ◄─ Own      │
│  │ PC       │  │ PC       │  │ PC       │    Stack     │
│  │ Locals   │  │ Locals   │  │ Locals   │              │
│  └──────────┘  └──────────┘  └──────────┘              │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              SHARED MEMORY                      │    │
│  │  Heap  │  Code  │  Static Data  │  Open Files   │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

| Feature | Process | Thread |
|---------|---------|--------|
| Memory | Separate address space | Shared heap, own stack |
| Creation cost | Heavy (OS-level) | Lighter (shares process resources) |
| Communication | IPC (pipes, sockets) | Shared memory (direct) |
| Context switch | Expensive | Less expensive |
| Crash impact | Isolated | Can crash entire process |

### Why Multithreading?

**1. Responsiveness** - Keep UI responsive while doing background work:
```java
public class ResponsiveApp {
    public static void main(String[] args) {
        // ❌ Single-threaded: UI freezes during long task
        // longRunningTask();
        // updateUI();

        // ✅ Multi-threaded: UI stays responsive
        Thread backgroundTask = new Thread(() -> {
            byte[] data = downloadFile("https://example.com/large-file.zip");
            Platform.runLater(() -> updateUI(data)); // update UI on main thread
        });
        backgroundTask.start();
    }
}
```

**2. Resource Sharing & Efficiency** - Threads share process memory:
```java
public class SharedCounter {
    private int count = 0; // Shared across threads

    public synchronized void increment() {
        count++;
    }
}
```

**3. Scalability** - Leverage multi-core processors:
```java
public class ParallelProcessing {
    public static void main(String[] args) {
        int cores = Runtime.getRuntime().availableProcessors();
        System.out.println("Available cores: " + cores);

        // Process data in parallel across available cores
        List<Integer> numbers = IntStream.rangeClosed(1, 1_000_000)
            .boxed()
            .collect(Collectors.toList());

        long sum = numbers.parallelStream()
            .mapToLong(Integer::longValue)
            .sum();
    }
}
```

**4. Simplified I/O Modeling** - Handle multiple clients concurrently:
```java
public class SimpleServer {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8080);

        while (true) {
            Socket client = serverSocket.accept();
            // Handle each client in its own thread
            new Thread(() -> handleClient(client)).start();
        }
    }

    private static void handleClient(Socket client) {
        // Process client request independently
    }
}
```

### Thread Scheduling

The JVM delegates thread scheduling to the OS. Java supports **preemptive scheduling**:

```java
public class ThreadSchedulingDemo {
    public static void main(String[] args) {
        Thread highPriority = new Thread(() -> {
            System.out.println("High priority task running");
        });
        highPriority.setPriority(Thread.MAX_PRIORITY); // 10

        Thread lowPriority = new Thread(() -> {
            System.out.println("Low priority task running");
        });
        lowPriority.setPriority(Thread.MIN_PRIORITY);  // 1

        // Priority is a HINT to the scheduler, not a guarantee
        highPriority.start();
        lowPriority.start();
    }
}
```

> ⚠️ **Thread priority is platform-dependent** and should NOT be relied upon for correctness.

### Context Switching Cost

```
Thread A running ──► Save state A ──► Load state B ──► Thread B running
                     (registers,       (registers,
                      PC, stack)        PC, stack)

Cost: ~1-10 microseconds per switch
```

**Factors increasing context switch cost:**
- Cache invalidation (L1/L2/L3 cache misses)
- TLB (Translation Lookaside Buffer) flushes
- Pipeline flushes in CPU

```java
// ❌ Too many threads = excessive context switching
ExecutorService bad = Executors.newFixedThreadPool(10_000);

// ✅ Match thread count to workload
int optimalThreads = Runtime.getRuntime().availableProcessors();
ExecutorService good = Executors.newFixedThreadPool(optimalThreads);

// For I/O-bound tasks, more threads may help:
// optimalThreads = cores * (1 + waitTime/computeTime)
```

### User Threads vs Daemon Threads

```java
public class DaemonThreadDemo {
    public static void main(String[] args) {
        // User thread - JVM waits for it to finish
        Thread userThread = new Thread(() -> {
            System.out.println("User thread: " + Thread.currentThread().isDaemon()); // false
        });

        // Daemon thread - JVM does NOT wait for it
        Thread daemonThread = new Thread(() -> {
            while (true) {
                System.out.println("Daemon running...");
                try { Thread.sleep(1000); } catch (InterruptedException e) { break; }
            }
        });
        daemonThread.setDaemon(true); // Must be set BEFORE start()

        userThread.start();
        daemonThread.start();

        // JVM exits when all user threads finish,
        // daemon threads are abruptly terminated
    }
}
```

| Feature | User Thread | Daemon Thread |
|---------|------------|---------------|
| JVM shutdown | JVM waits for completion | Killed on JVM exit |
| Use case | Application logic | Background services (GC, logging) |
| Default | Yes | No |
| Set before | N/A | Must call `setDaemon(true)` before `start()` |

### Basic Thread Example

```java
public class ThreadBasics {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(() -> {
            String name = Thread.currentThread().getName();
            System.out.println("[" + name + "] Started");

            for (int i = 0; i < 5; i++) {
                System.out.println("[" + name + "] Count: " + i);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    System.out.println("[" + name + "] Interrupted!");
                    Thread.currentThread().interrupt(); // Restore flag
                    return;
                }
            }

            System.out.println("[" + name + "] Finished");
        }, "Worker-1");

        System.out.println("Thread state before start: " + thread.getState()); // NEW
        thread.start();
        System.out.println("Thread state after start: " + thread.getState());  // RUNNABLE

        thread.join(); // Wait for thread to finish
        System.out.println("Thread state after join: " + thread.getState());   // TERMINATED
    }
}
```

---

## Anti-Patterns

❌ **Creating unbounded threads:**
```java
// Never do this in production
while (true) {
    new Thread(() -> handleRequest()).start(); // OutOfMemoryError!
}
```

❌ **Relying on thread priority for correctness**

❌ **Ignoring InterruptedException:**
```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    // ❌ NEVER swallow this silently!
}
```

## Best Practices

✅ Use thread pools instead of creating threads manually  
✅ Always restore the interrupt flag when catching `InterruptedException`  
✅ Name your threads for easier debugging  
✅ Use daemon threads only for non-critical background tasks  
✅ Size thread pools based on workload type (CPU-bound vs I/O-bound)

---

[Next: Thread Lifecycle →](./q02-thread-lifecycle.md)

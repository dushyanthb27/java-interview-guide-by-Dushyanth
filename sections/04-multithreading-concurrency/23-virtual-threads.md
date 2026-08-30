# Q23: What Are Virtual Threads (Project Loom)?

## 📋 Question
What are virtual threads in Java 21? How do they differ from platform threads? When should you use them?

## Difficulty: 🟠 Intermediate

---

## 💡 Answer

### Platform Threads vs Virtual Threads

```
Platform Threads (traditional):
┌─────────────────────────────────────────────┐
│ JVM                                         │
│  Thread A ──── 1:1 ──── OS Thread A (1MB)   │
│  Thread B ──── 1:1 ──── OS Thread B (1MB)   │
│  Thread C ──── 1:1 ──── OS Thread C (1MB)   │
│                                             │
│  Max ~2,000-10,000 threads (limited by OS)  │
└─────────────────────────────────────────────┘

Virtual Threads (Java 21):
┌─────────────────────────────────────────────┐
│ JVM                                         │
│  VThread 1 ─┐                               │
│  VThread 2 ─┤── M:N ── Carrier Thread 1     │
│  VThread 3 ─┘            (OS Thread)        │
│  VThread 4 ─┐                               │
│  VThread 5 ─┤── M:N ── Carrier Thread 2     │
│  VThread 6 ─┘            (OS Thread)        │
│     ...                     ...             │
│  Millions of virtual threads, few OS threads│
└─────────────────────────────────────────────┘
```

| Feature | Platform Thread | Virtual Thread |
|---------|----------------|----------------|
| Memory | ~1MB stack | ~1KB initial |
| Creation cost | Expensive (OS) | Cheap (JVM) |
| Count limit | ~2K-10K | Millions |
| Scheduling | OS scheduler | JVM scheduler |
| Blocking cost | Wastes OS thread | JVM unmounts (cheap) |
| Best for | CPU-bound work | I/O-bound work |
| Since | JDK 1.0 | JDK 21 (GA) |

### Creating Virtual Threads

```java
public class VirtualThreadCreation {
    public static void main(String[] args) throws Exception {

        // 1. Thread.ofVirtual()
        Thread vThread = Thread.ofVirtual()
            .name("my-virtual-thread")
            .start(() -> {
                System.out.println("Running on: " + Thread.currentThread());
                System.out.println("Is virtual: " + Thread.currentThread().isVirtual());
            });
        vThread.join();

        // 2. Thread.startVirtualThread() (shortcut)
        Thread vThread2 = Thread.startVirtualThread(() -> {
            System.out.println("Quick virtual thread");
        });
        vThread2.join();

        // 3. Virtual thread factory
        ThreadFactory factory = Thread.ofVirtual()
            .name("worker-", 0) // worker-0, worker-1, worker-2...
            .factory();

        Thread t = factory.newThread(() -> System.out.println("From factory"));
        t.start();
        t.join();

        // 4. ExecutorService with virtual threads (RECOMMENDED)
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 100_000; i++) {
                executor.submit(() -> {
                    // Each task gets its own virtual thread
                    Thread.sleep(Duration.ofMillis(100));
                    return "done";
                });
            }
        } // Auto-closes and awaits all tasks
    }
}
```

### Thread-Per-Request Model

```java
// ✅ Virtual threads make thread-per-request viable again!
public class VirtualThreadServer {
    public static void main(String[] args) throws IOException {
        try (ServerSocket server = new ServerSocket(8080);
             ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {

            while (true) {
                Socket client = server.accept();
                executor.submit(() -> handleRequest(client));
                // Each request gets its own virtual thread
                // Can handle millions of concurrent connections!
            }
        }
    }

    static void handleRequest(Socket client) {
        try (client;
             var reader = new BufferedReader(new InputStreamReader(client.getInputStream()));
             var writer = new PrintWriter(client.getOutputStream(), true)) {

            String request = reader.readLine();

            // I/O blocking is CHEAP with virtual threads
            String dbResult = queryDatabase(request);     // Blocks - virtual thread unmounts
            String apiResult = callExternalAPI(request);  // Blocks - virtual thread unmounts

            writer.println("Response: " + dbResult + " " + apiResult);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    static String queryDatabase(String query) throws IOException {
        Thread.sleep(Duration.ofMillis(50)); // Simulated I/O
        return "db-result";
    }

    static String callExternalAPI(String data) throws IOException {
        Thread.sleep(Duration.ofMillis(100)); // Simulated I/O
        return "api-result";
    }
}
```

### How Virtual Threads Work: Mounting/Unmounting

```
Virtual Thread lifecycle:

1. VThread running on Carrier Thread (mounted)
   [VThread-1] ──mounted──► [Carrier-1 (OS Thread)]

2. VThread hits blocking I/O (socket read, sleep, etc.)
   [VThread-1] ──UNMOUNTED── [Carrier-1 is FREE]
   VThread-1 state saved to heap (continuation)

3. Carrier picks up another virtual thread
   [VThread-2] ──mounted──► [Carrier-1]

4. I/O completes, VThread-1 remounts (maybe different carrier)
   [VThread-1] ──mounted──► [Carrier-3]

Key insight: Blocking I/O no longer wastes OS threads!
```

### Pinning Issues

Virtual threads get **pinned** to their carrier thread in certain situations:

```java
public class PinningDemo {

    // ❌ synchronized blocks PIN the virtual thread to carrier
    private final Object lock = new Object();

    public void pinnedOperation() {
        synchronized (lock) {
            // Virtual thread CANNOT unmount during this block!
            // If I/O happens here, the carrier thread is wasted
            Thread.sleep(Duration.ofMillis(100)); // Blocks the carrier!
        }
    }

    // ✅ Fix: Use ReentrantLock instead
    private final ReentrantLock reentrantLock = new ReentrantLock();

    public void unpinnedOperation() {
        reentrantLock.lock();
        try {
            // Virtual thread CAN unmount here
            Thread.sleep(Duration.ofMillis(100)); // Carrier is freed!
        } finally {
            reentrantLock.unlock();
        }
    }

    // ❌ Also causes pinning:
    // - Native methods (JNI)
    // - Foreign function calls (Panama)
}
```

> **JVM flag to detect pinning:** `-Djdk.tracePinnedThreads=full`

### Migration Guide

```java
// BEFORE: Platform thread pool
ExecutorService executor = Executors.newFixedThreadPool(200);
// Limited to 200 concurrent requests

// AFTER: Virtual thread executor
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
// Unlimited concurrent requests (bounded by I/O, not threads)

// BEFORE: Manual thread creation
new Thread(task).start(); // Expensive

// AFTER: Virtual thread
Thread.startVirtualThread(task); // Cheap

// Spring Boot 3.2+: Enable virtual threads
// application.properties:
// spring.threads.virtual.enabled=true
```

### What NOT to Do with Virtual Threads

```java
public class VirtualThreadAntiPatterns {

    // ❌ Don't pool virtual threads - they're cheap to create!
    ExecutorService bad = Executors.newFixedThreadPool(100);
    // Virtual threads are NOT meant to be pooled

    // ✅ Create as many as you need
    ExecutorService good = Executors.newVirtualThreadPerTaskExecutor();

    // ❌ Don't use for CPU-bound work
    void cpuBound() {
        Thread.startVirtualThread(() -> {
            // Heavy computation doesn't benefit from virtual threads
            // Platform threads are better here
            fibonacci(45);
        });
    }

    // ❌ Don't use ThreadLocal with virtual threads (memory concern)
    ThreadLocal<byte[]> heavyTL = new ThreadLocal<>();
    // With millions of virtual threads, this uses massive memory!

    // ❌ Don't use synchronized for guarding I/O in virtual threads
    synchronized void syncIO() {
        // read from socket... (PINNED!)
    }

    long fibonacci(int n) {
        if (n <= 1) return n;
        return fibonacci(n - 1) + fibonacci(n - 2);
    }
}
```

### Decision Guide

```
┌───────────────────────────────────────────────┐
│ Should I use Virtual Threads?                 │
│                                               │
│ ✅ YES if:                                    │
│   • I/O-bound workload (DB, HTTP, files)      │
│   • High concurrency (>1000 concurrent tasks) │
│   • Thread-per-request server                 │
│   • Many blocking operations                  │
│                                               │
│ ❌ NO if:                                     │
│   • CPU-bound work (use platform threads)     │
│   • Need synchronized blocks with I/O inside  │
│   • Using heavy ThreadLocal per task          │
│   • Framework doesn't support it yet          │
└───────────────────────────────────────────────┘
```

---

## Anti-Patterns

❌ **Pooling virtual threads** - defeats the purpose  
❌ **Using `synchronized` with I/O inside** - causes pinning  
❌ **Using virtual threads for CPU-bound work** - no benefit  
❌ **Heavy ThreadLocal usage** - memory explosion with millions of threads

## Best Practices

✅ Use `Executors.newVirtualThreadPerTaskExecutor()` as the primary pattern  
✅ Replace `synchronized` with `ReentrantLock` for I/O-guarding code  
✅ Use `-Djdk.tracePinnedThreads=full` to detect pinning  
✅ Keep virtual threads for I/O-bound work; use platform threads for CPU-bound  
✅ Don't pool virtual threads; create them freely  
✅ Consider `ScopedValue` instead of `ThreadLocal`

---

## 🎯 Interviewer Insight
Interviewers ask about virtual threads to assess your knowledge of modern Java. They want you to understand the key difference: virtual threads are **cheap to create** and **don't waste OS threads** when blocking. Strong candidates can explain mounting/unmounting, pinning issues, and when to use virtual threads vs platform threads.

## ⚡ Follow-up Questions
1. What is the difference between virtual threads and platform threads?
2. When should you use virtual threads vs traditional thread pools?
3. What causes pinning in virtual threads and how do you avoid it?

## 📚 Related Topics
- [Thread Pools](./05-thread-pools.md)
- [Executor Framework](./06-executor-framework.md)
- [CompletableFuture](./15-completable-future.md)

# Q18: Virtual Threads and Project Loom

## 📋 Question
What are virtual threads in Java? Explain their creation, the continuation-based implementation, when they shine, pinning issues, structured concurrency, and migration from platform threads.

## Difficulty: 🔴 Staff/Principal

---

## 💡 Answer

**Virtual threads** (Java 19 preview, Java 21 final — JEP 444) are **lightweight, JVM-managed threads** that decouple the Java thread abstraction from OS-level threads. While platform threads map 1:1 to OS threads (limiting concurrency to thousands), virtual threads are managed by the JVM runtime and can scale to **millions** of concurrent threads — enabling a "thread-per-request" model for I/O-bound applications without the complexity of async/reactive programming.

### Creating Virtual Threads

```java
package com.interview.java21;

import java.util.concurrent.*;
import java.util.stream.*;
import java.time.*;

public class VirtualThreadCreationDemo {

    public static void main(String[] args) throws Exception {

        // ═══════════════════════════════════════════════════
        // Method 1: Thread.ofVirtual() builder
        // ═══════════════════════════════════════════════════
        Thread vt1 = Thread.ofVirtual()
                .name("my-virtual-thread")
                .start(() -> {
                    System.out.println("Running on: " + Thread.currentThread());
                    System.out.println("Is virtual: " + Thread.currentThread().isVirtual());
                });
        vt1.join();

        // ═══════════════════════════════════════════════════
        // Method 2: Thread.startVirtualThread() — quickest way
        // ═══════════════════════════════════════════════════
        Thread vt2 = Thread.startVirtualThread(() -> {
            System.out.println("Quick virtual thread running!");
        });
        vt2.join();

        // ═══════════════════════════════════════════════════
        // Method 3: Virtual thread executor — THE recommended approach
        // ═══════════════════════════════════════════════════
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            // Each submitted task gets its OWN virtual thread
            var futures = IntStream.range(0, 10_000)
                    .mapToObj(i -> executor.submit(() -> {
                        Thread.sleep(Duration.ofSeconds(1));  // Simulates I/O
                        return "Task-" + i;
                    }))
                    .toList();

            // All 10,000 tasks run concurrently on a few OS threads!
            System.out.println("Completed: " + futures.size());
        }

        // ═══════════════════════════════════════════════════
        // Method 4: Thread factory for custom executors
        // ═══════════════════════════════════════════════════
        ThreadFactory factory = Thread.ofVirtual()
                .name("worker-", 0)  // worker-0, worker-1, worker-2...
                .factory();
        try (var executor = Executors.newThreadPerTaskExecutor(factory)) {
            executor.submit(() -> System.out.println("Named virtual thread"));
        }
    }
}
```

### Platform Threads vs Virtual Threads

| Feature | Platform Threads | Virtual Threads |
|---------|-----------------|-----------------|
| **Mapped to** | 1:1 to OS thread | Many-to-few carrier threads |
| **Stack size** | ~1 MB default | Grows/shrinks dynamically (few KB) |
| **Max concurrent** | Thousands (OS limit) | **Millions** |
| **Creation cost** | ~1 ms (OS call) | ~1 µs (JVM-internal) |
| **Scheduling** | OS scheduler | JVM scheduler (ForkJoinPool) |
| **Best for** | CPU-bound work | **I/O-bound work** |
| **Thread pool needed?** | Yes (costly to create) | No (cheap to create per task) |
| **Blocking I/O** | Wastes OS thread | Unmounts from carrier (free) |
| **ThreadLocal** | Works (careful with pooling) | Works but avoid heavy usage |

### Continuation-Based Implementation

```java
package com.interview.java21;

import java.net.http.*;
import java.net.URI;
import java.util.concurrent.*;

public class ContinuationDemo {

    // ═══════════════════════════════════════════════════
    // HOW IT WORKS UNDER THE HOOD:
    //
    // Virtual Thread        Carrier (Platform) Thread
    // ─────────────        ────────────────────────────
    //  [Task Code]          [ForkJoinPool worker-1]
    //       │                        │
    //  Runs on carrier ──────────────│
    //       │                        │
    //  Hits blocking I/O             │
    //  (e.g., socket.read)           │
    //       │                        │
    //  UNMOUNTS from carrier ────────│ Carrier is FREE
    //  (stores continuation          │ to run other
    //   on heap)                     │ virtual threads!
    //       │                        │
    //  I/O completes                 │
    //       │                        │
    //  REMOUNTS on a carrier ────────│ (might be different
    //  (resumes continuation)        │  carrier thread!)
    //       │                        │
    //  Continues execution           │
    // ═══════════════════════════════════════════════════

    public static void main(String[] args) throws Exception {
        // Demonstration: 100,000 concurrent HTTP requests
        // Each virtual thread blocks on I/O but only uses carrier during CPU work
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            var client = HttpClient.newHttpClient();

            var futures = java.util.stream.IntStream.range(0, 100)
                    .mapToObj(i -> executor.submit(() -> {
                        var request = HttpRequest.newBuilder()
                                .uri(URI.create("https://httpbin.org/delay/1"))
                                .build();
                        var response = client.send(request,
                                HttpResponse.BodyHandlers.ofString());
                        return "Task-" + i + ": " + response.statusCode();
                    }))
                    .toList();

            for (var future : futures) {
                try {
                    System.out.println(future.get(30, TimeUnit.SECONDS));
                } catch (Exception e) {
                    System.out.println("Error: " + e.getMessage());
                }
            }
        }
    }
}
```

### When Virtual Threads Shine (and When They Don't)

```java
package com.interview.java21;

import java.util.concurrent.*;

public class WhenToUseDemo {

    // ═══════════════════════════════════════════════════
    // ✅ IDEAL: I/O-bound, high-concurrency scenarios
    // ═══════════════════════════════════════════════════
    // - Web servers handling thousands of concurrent requests
    // - Microservices making downstream API calls
    // - Database query execution with connection pooling
    // - File I/O operations
    // - Message queue consumers

    // ✅ Perfect use case: fan-out to multiple services
    static void idealScenario() throws Exception {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            Future<String> userFuture = executor.submit(() -> fetchUser());
            Future<String> ordersFuture = executor.submit(() -> fetchOrders());
            Future<String> recommendFuture = executor.submit(() -> fetchRecommendations());

            // All three run concurrently on virtual threads
            // Each blocks on I/O independently
            String user = userFuture.get();
            String orders = ordersFuture.get();
            String recommendations = recommendFuture.get();
        }
    }

    // ═══════════════════════════════════════════════════
    // ❌ NOT IDEAL: CPU-bound work
    // ═══════════════════════════════════════════════════
    // - Mathematical computations
    // - Image/video processing
    // - Sorting/searching large datasets
    // - Machine learning inference
    // Virtual threads add no benefit for CPU-bound tasks — they still
    // need carrier threads, and there are only as many carriers as CPUs

    // ❌ Anti-pattern: CPU-bound work on virtual threads
    static void badScenario() throws Exception {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            // These all compete for the same carrier threads (CPU cores)
            // No better than using a fixed thread pool
            for (int i = 0; i < 1000; i++) {
                executor.submit(() -> {
                    // CPU-intensive computation — stays mounted on carrier
                    long sum = 0;
                    for (long j = 0; j < 1_000_000_000L; j++) sum += j;
                    return sum;
                });
            }
        }
    }

    static String fetchUser() throws Exception {
        Thread.sleep(100); return "User";
    }
    static String fetchOrders() throws Exception {
        Thread.sleep(150); return "Orders";
    }
    static String fetchRecommendations() throws Exception {
        Thread.sleep(200); return "Recommendations";
    }
}
```

### Pinning — When Virtual Threads Can't Unmount

```java
package com.interview.java21;

import java.util.concurrent.locks.ReentrantLock;

public class PinningDemo {

    // ═══════════════════════════════════════════════════
    // PINNING: Virtual thread is stuck on carrier thread
    // Happens in two cases:
    // 1. Inside a synchronized block/method
    // 2. During native method/foreign function calls
    // ═══════════════════════════════════════════════════

    private static final Object lock = new Object();
    private static final ReentrantLock reentrantLock = new ReentrantLock();

    // ❌ PINNING: synchronized blocks pin the virtual thread to its carrier
    static void pinnedExample() {
        synchronized (lock) {
            try {
                Thread.sleep(1000);  // Virtual thread CANNOT unmount!
                // Carrier thread is blocked — defeats the purpose
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    // ✅ SOLUTION: Use ReentrantLock instead of synchronized
    static void unpinnedExample() {
        reentrantLock.lock();
        try {
            Thread.sleep(1000);  // Virtual thread CAN unmount
            // Carrier thread is freed for other virtual threads
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            reentrantLock.unlock();
        }
    }

    // ✅ Detect pinning at runtime
    // JVM flag: -Djdk.tracePinnedThreads=full
    // or: -Djdk.tracePinnedThreads=short

    public static void main(String[] args) {
        Thread.startVirtualThread(() -> unpinnedExample());
    }
}
```

### Structured Concurrency (Preview — JEP 453)

```java
package com.interview.java21;

import java.util.concurrent.*;

public class StructuredConcurrencyDemo {

    record UserProfile(String name, String orders, String recommendations) {}

    // ═══════════════════════════════════════════════════
    // Structured Concurrency — child tasks are bound to parent scope
    // If parent fails/cancels, all children are cancelled
    // ═══════════════════════════════════════════════════

    // ✅ Using StructuredTaskScope (Preview in Java 21)
    static UserProfile fetchProfile(String userId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            // Fork subtasks — each runs on its own virtual thread
            Subtask<String> user = scope.fork(() -> fetchUser(userId));
            Subtask<String> orders = scope.fork(() -> fetchOrders(userId));
            Subtask<String> recs = scope.fork(() -> fetchRecommendations(userId));

            scope.join();               // Wait for ALL subtasks
            scope.throwIfFailed();      // Propagate any exception

            // All subtasks completed successfully
            return new UserProfile(user.get(), orders.get(), recs.get());
        }
        // When scope closes, all subtasks are guaranteed done/cancelled
    }

    // ✅ ShutdownOnSuccess — returns first successful result, cancels rest
    static String fetchFromFastestMirror(String resource) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
            scope.fork(() -> fetchFromMirror("US", resource));
            scope.fork(() -> fetchFromMirror("EU", resource));
            scope.fork(() -> fetchFromMirror("ASIA", resource));

            scope.join();
            return scope.result();  // Returns first successful result
        }
    }

    static String fetchUser(String id) throws Exception {
        Thread.sleep(100); return "User-" + id;
    }
    static String fetchOrders(String id) throws Exception {
        Thread.sleep(200); return "Orders-" + id;
    }
    static String fetchRecommendations(String id) throws Exception {
        Thread.sleep(150); return "Recs-" + id;
    }
    static String fetchFromMirror(String region, String resource) throws Exception {
        Thread.sleep((long) (Math.random() * 500));
        return region + ":" + resource;
    }

    public static void main(String[] args) throws Exception {
        var profile = fetchProfile("alice");
        System.out.println(profile);
    }
}
```

### Migration from Platform Threads

```java
package com.interview.java21;

import java.util.concurrent.*;

public class MigrationGuideDemo {

    // ═══════════════════════════════════════════════════
    // STEP 1: Replace fixed thread pools with virtual thread executor
    // ═══════════════════════════════════════════════════

    // ❌ Before: Fixed thread pool (limits concurrency)
    static final ExecutorService oldPool = Executors.newFixedThreadPool(200);

    // ✅ After: Virtual thread per task (unlimited concurrency)
    static final ExecutorService newPool = Executors.newVirtualThreadPerTaskExecutor();

    // ═══════════════════════════════════════════════════
    // STEP 2: Replace synchronized with ReentrantLock
    // ═══════════════════════════════════════════════════

    // ❌ Before: synchronized pins virtual threads
    // synchronized (sharedResource) { doBlockingWork(); }

    // ✅ After: ReentrantLock allows unmounting
    // lock.lock();
    // try { doBlockingWork(); } finally { lock.unlock(); }

    // ═══════════════════════════════════════════════════
    // STEP 3: Avoid thread-local caching (memory per VT adds up)
    // ═══════════════════════════════════════════════════

    // ❌ Before: ThreadLocal with heavy objects (fine for pooled threads)
    // static final ThreadLocal<ExpensiveObject> cache = ThreadLocal.withInitial(...);
    // With 1M virtual threads, this creates 1M ExpensiveObjects!

    // ✅ After: Use scoped values (Java 21 preview) or shared cache
    // static final ScopedValue<String> CURRENT_USER = ScopedValue.newInstance();

    // ═══════════════════════════════════════════════════
    // STEP 4: Don't pool virtual threads!
    // ═══════════════════════════════════════════════════

    // ❌ Bad: Pooling virtual threads (they're cheap — just create new ones)
    // ExecutorService pool = Executors.newFixedThreadPool(100,
    //         Thread.ofVirtual().factory());  // ❌ Defeats the purpose!

    // ✅ Good: One virtual thread per task, no pooling
    // ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

    public static void main(String[] args) throws Exception {
        // ✅ Migration is often a ONE-LINE change for executor-based code
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            var future = executor.submit(() -> {
                Thread.sleep(100);
                return "Hello from virtual thread!";
            });
            System.out.println(future.get());
            System.out.println("Is virtual: " + Thread.currentThread().isVirtual());
        }
    }
}
```

### Migration Checklist

| Action | Priority | Details |
|--------|----------|---------|
| Replace `newFixedThreadPool` with `newVirtualThreadPerTaskExecutor` | High | One-line change |
| Replace `synchronized` with `ReentrantLock` | High | Prevents pinning |
| Audit `ThreadLocal` usage | Medium | Avoid heavy per-thread state |
| Remove thread pool sizing tuning | Medium | No longer needed |
| Test for pinning | Medium | Use `-Djdk.tracePinnedThreads=short` |
| Audit native/JNI calls | Low | May cause pinning |
| Consider structured concurrency | Low | Preview feature in Java 21 |

### ⚠️ Common Mistake
**Pooling virtual threads**: Virtual threads are so cheap (~1 µs to create, few KB of stack) that pooling them is an anti-pattern. Unlike platform threads (where pooling avoids expensive OS thread creation), virtual threads should be created **on-demand, one per task**, and discarded after use. Using `Executors.newFixedThreadPool(n, virtualThreadFactory)` defeats the entire purpose of virtual threads — it artificially limits concurrency just as platform thread pools do.

### ⚡ Performance Tip
Virtual threads excel when the workload is **I/O-bound with high concurrency**. A server handling 100,000 concurrent connections, each waiting on database or API responses, can use 100,000 virtual threads — all multiplexed onto a handful of carrier threads (typically `availableProcessors()` count). The key metric is **Little's Law**: `concurrency = throughput × latency`. If each request takes 100ms and you want 10,000 req/s throughput, you need 1,000 concurrent threads — trivial for virtual threads, near the limit for platform threads.

### 🔧 JVM Internals
Virtual threads are implemented using **continuations** — a JVM-level mechanism that saves and restores a thread's execution state (stack frames, program counter, local variables). When a virtual thread blocks on I/O, the JVM captures its continuation (moves the stack from the carrier thread to the heap), unmounts it from the carrier, and schedules another virtual thread on the freed carrier. When the I/O completes, the continuation is resumed on any available carrier. The carrier thread pool is a `ForkJoinPool` with `availableProcessors()` threads by default (configurable via `jdk.virtualThreadScheduler.parallelism`). The mounting/unmounting is transparent to application code — existing blocking APIs (`Thread.sleep`, `Socket.read`, `Lock.lock`) automatically yield the carrier.

---

## 🎯 Interviewer Insight
This is a **Staff/Principal level** question. Interviewers expect you to explain the **problem** virtual threads solve (thread-per-request at scale without async complexity), the **mechanism** (continuations, mounting/unmounting), the **limitations** (pinning with `synchronized`, not for CPU-bound work), and the **migration path** from traditional thread pools. Mentioning structured concurrency and `ScopedValue` (replacing `ThreadLocal`) shows awareness of the full Project Loom ecosystem. The analogy "virtual threads are to platform threads what virtual memory is to physical memory" is a powerful talking point.

## ⚡ Follow-up Questions
1. Why can't `synchronized` blocks work well with virtual threads?
2. How many carrier threads does the JVM use by default?
3. How do virtual threads compare to Kotlin coroutines or Go goroutines?

## 📚 Related Topics
- [CompletableFuture](./q11-completable-future-detail.md)
- [Lambda Expressions](./q01-lambda-expressions.md)
- [Stream API Operations](./q04-stream-api-operations.md)

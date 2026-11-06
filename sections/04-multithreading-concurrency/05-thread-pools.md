# Q5: How Do Thread Pools Work?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question

What are thread pools and why are they necessary? Explain `ThreadPoolExecutor` architecture, parameters, queue types, rejection policies, work-stealing pools, and virtual thread executors (`Executors.newVirtualThreadPerTaskExecutor()`).

---

## Answer

### Why Thread Pools?

```
WITHOUT Thread Pool:                    WITH Thread Pool:
┌──────────┐                           ┌──────────┐
│ Request 1│──► new Thread()           │ Request 1│──┐
│ Request 2│──► new Thread()           │ Request 2│──┤    ┌─────────┐
│ Request 3│──► new Thread()           │ Request 3│──┼───►│  Pool   │
│    ...   │──► new Thread()           │    ...   │──┤    │ (fixed  │
│ Request N│──► new Thread()           │ Request N│──┘    │ threads)│
└──────────┘                           └──────────┘       └─────────┘
❌ Unbounded thread creation            ✅ Thread reuse & resource control
❌ OutOfMemoryError risk                ✅ Lower latency (no thread creation overhead)
❌ High creation/destruction cost       ✅ Throttling under heavy load
```

### `ThreadPoolExecutor` - The Core Engine

```java
public ThreadPoolExecutor(
    int corePoolSize,                 // Minimum threads kept alive in pool
    int maximumPoolSize,              // Maximum threads allowed in pool
    long keepAliveTime,               // Idle timeout for non-core (or all) threads
    TimeUnit unit,                    // Time unit for keepAliveTime
    BlockingQueue<Runnable> workQueue,// Queue holding pending tasks
    ThreadFactory threadFactory,      // Factory for custom naming / daemon config
    RejectedExecutionHandler handler  // Saturation policy when pool & queue are full
)
```

#### Task Submission Flow Algorithm

```
                                 ┌───────────────────────┐
                                 │   Submit new task      │
                                 └───────────┬───────────┘
                                             │
                              ┌──────────────▼──────────────┐
                              │ Active threads < coreSize?   │
                              └──────┬───────────────┬──────┘
                                  YES│               │NO
                              ┌──────▼──────┐  ┌─────▼───────────┐
                              │ Create new   │  │ Queue has space? │
                              │ core thread  │  └──┬──────────┬───┘
                              └─────────────┘   YES│          │NO
                                           ┌──────▼─────┐ ┌──▼──────────────┐
                                           │ Add to      │ │ threads < max?   │
                                           │ work queue  │ └──┬──────────┬───┘
                                           └────────────┘  YES│          │NO
                                                  ┌───────▼──────┐ ┌──▼───────────┐
                                                  │ Create new    │ │ Reject task   │
                                                  │ non-core      │ │ (handler)     │
                                                  │ thread        │ └──────────────┘
                                                  └──────────────┘
```

### Standard Factory Methods & Built-in Pools

1. **Fixed Thread Pool**: `Executors.newFixedThreadPool(n)`
   - `corePoolSize == maximumPoolSize == n`
   - Queue: Unbounded `LinkedBlockingQueue`
   - Best for: Predictable CPU-bound workloads.

2. **Cached Thread Pool**: `Executors.newCachedThreadPool()`
   - `corePoolSize = 0`, `maximumPoolSize = Integer.MAX_VALUE`, `keepAliveTime = 60s`
   - Queue: `SynchronousQueue` (zero-capacity handoff)
   - ⚠️ **Danger**: Can spawn unlimited threads under bursty traffic, risking OOM.

3. **Single Thread Executor**: `Executors.newSingleThreadExecutor()`
   - `corePoolSize == maximumPoolSize == 1`
   - Best for: Enforcing strict sequential task ordering.

4. **Work-Stealing Pool (Java 8+)**: `Executors.newWorkStealingPool()`
   - Backed by `ForkJoinPool`, where each thread maintains its own deque and steals tasks from idle/overloaded queues.

5. **Virtual Thread Per-Task Executor (Java 21+)**: `Executors.newVirtualThreadPerTaskExecutor()`
   - Spawns a new virtual thread for every submitted task. No traditional pool sizing needed.

### Custom Production Configuration Pattern

```java
public class ProductionThreadPool {
    public static ExecutorService createOptimalPool() {
        int cores = Runtime.getRuntime().availableProcessors();
        int coreSize = cores;
        int maxSize = cores * 2;
        long keepAlive = 60L;

        // ✅ Always use a BOUNDED queue in production to prevent OOM
        BlockingQueue<Runnable> queue = new ArrayBlockingQueue<>(1000);

        ThreadFactory factory = r -> {
            Thread t = new Thread(r, "app-worker-" + Thread.currentThread().getId());
            t.setDaemon(false);
            t.setUncaughtExceptionHandler((thread, ex) ->
                System.err.println("Thread " + thread.getName() + " threw: " + ex.getMessage()));
            return t;
        };

        // CallerRunsPolicy: backpressure mechanism — submitting thread executes task if pool saturated
        RejectedExecutionHandler rejectionHandler = new ThreadPoolExecutor.CallerRunsPolicy();

        return new ThreadPoolExecutor(
            coreSize, maxSize, keepAlive, TimeUnit.SECONDS,
            queue, factory, rejectionHandler
        );
    }
}
```

### Rejection Policies

1. **`AbortPolicy` (Default)**: Throws `RejectedExecutionException`.
2. **`CallerRunsPolicy`**: Executes task in the caller's thread, providing natural rate-limiting backpressure.
3. **`DiscardPolicy`**: Silently discards the rejected task.
4. **`DiscardOldestPolicy`**: Drops the oldest unhandled task in the work queue and retries submission.

---

## ⚠️ Gotchas & Common Pitfalls

1. **Unbounded Queue Hazard**: Using `Executors.newFixedThreadPool(...)` uses `LinkedBlockingQueue` with capacity `Integer.MAX_VALUE`, allowing memory exhaustion if task generation exceeds consumption rate.
2. **Cached Pool Explosion**: Using `Executors.newCachedThreadPool()` under high load spawns thousands of OS threads, causing native OOM crashes.
3. **Deadlock via Task Dependencies**: If tasks submitted to a fixed thread pool wait on results of other tasks submitted to the **same** pool, the pool can deadlock if all threads are blocked waiting.
4. **Forgetting Pool Shutdown**: Failing to call `shutdown()` or using try-with-resources (Java 19+) keeps non-daemon threads active, preventing application exit.

---

## Interviewer Expectations & Deep Mental Models

- **Queue Sizing & Backpressure**: Be ready to explain how queue capacity and rejection policies govern system stability under traffic spikes.
- **CPU vs I/O Sizing**: Formula for I/O-bound pools: $\text{Threads} = \text{Cores} \times (1 + \frac{\text{Wait Time}}{\text{Compute Time}})$.
- **Virtual Threads vs Thread Pools**: Explain that with Java 21 Virtual Threads, traditional pooling is discouraged for I/O workloads; instead, create a virtual thread per task (`Executors.newVirtualThreadPerTaskExecutor()`).

---

[← Runnable vs Callable](./04-runnable-vs-callable.md) | [Next: Executor Framework →](./06-executor-framework.md)

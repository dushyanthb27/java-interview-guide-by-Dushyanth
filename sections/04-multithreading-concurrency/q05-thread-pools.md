# Q5: How Do Thread Pools Work?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What are thread pools? What types are available and how do you configure them?

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
❌ Unbounded thread creation            ✅ Thread reuse
❌ OutOfMemoryError risk                ✅ Controlled resources
❌ High creation/destruction cost       ✅ Better performance
```

### ThreadPoolExecutor - The Core

```java
public ThreadPoolExecutor(
    int corePoolSize,        // Minimum threads kept alive
    int maximumPoolSize,     // Maximum threads in pool
    long keepAliveTime,      // Idle time before non-core threads die
    TimeUnit unit,           // Time unit for keepAliveTime
    BlockingQueue<Runnable> workQueue,  // Queue for pending tasks
    ThreadFactory threadFactory,        // How to create threads
    RejectedExecutionHandler handler   // What to do when saturated
)
```

```
Task Submission Flow:
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

### Types of Thread Pools

#### 1. Fixed Thread Pool
```java
// Creates a pool with exactly N threads
ExecutorService fixedPool = Executors.newFixedThreadPool(4);
// Equivalent to:
// new ThreadPoolExecutor(4, 4, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>())

// Best for: CPU-bound tasks with predictable load
int cores = Runtime.getRuntime().availableProcessors();
ExecutorService cpuPool = Executors.newFixedThreadPool(cores);
```

#### 2. Cached Thread Pool
```java
// Creates threads as needed, reuses idle threads
ExecutorService cachedPool = Executors.newCachedThreadPool();
// Equivalent to:
// new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue<>())

// ⚠️ WARNING: Can create unlimited threads!
// Best for: Many short-lived I/O tasks
```

#### 3. Single Thread Executor
```java
// Guarantees sequential execution
ExecutorService singlePool = Executors.newSingleThreadExecutor();
// Equivalent to a FixedThreadPool of size 1

// Best for: Sequential task ordering, event loops
```

#### 4. Scheduled Thread Pool
```java
ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(2);

// One-time delayed execution
scheduledPool.schedule(() -> System.out.println("Delayed!"),
    5, TimeUnit.SECONDS);

// Fixed rate - starts every N units (overlap possible)
scheduledPool.scheduleAtFixedRate(() -> System.out.println("Fixed rate"),
    0, 1, TimeUnit.SECONDS);

// Fixed delay - N units gap between end and next start
scheduledPool.scheduleWithFixedDelay(() -> System.out.println("Fixed delay"),
    0, 1, TimeUnit.SECONDS);
```

#### 5. Work-Stealing Pool (Java 8+)
```java
// Uses ForkJoinPool with parallelism = available processors
ExecutorService stealingPool = Executors.newWorkStealingPool();
ExecutorService stealingPool4 = Executors.newWorkStealingPool(4);

// Best for: Tasks that spawn subtasks, recursive algorithms
```

### Custom ThreadPoolExecutor Configuration

```java
public class CustomThreadPool {
    public static ExecutorService createOptimalPool() {
        int coreSize = Runtime.getRuntime().availableProcessors();
        int maxSize = coreSize * 2;
        long keepAlive = 60L;

        // Bounded queue prevents memory issues
        BlockingQueue<Runnable> queue = new ArrayBlockingQueue<>(1000);

        ThreadFactory factory = new ThreadFactory() {
            private final AtomicInteger counter = new AtomicInteger(0);
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r, "app-worker-" + counter.incrementAndGet());
                t.setDaemon(false);
                t.setUncaughtExceptionHandler((thread, ex) ->
                    System.err.println(thread.getName() + " error: " + ex.getMessage()));
                return t;
            }
        };

        // CallerRunsPolicy: submitting thread runs the task when pool is saturated
        return new ThreadPoolExecutor(
            coreSize, maxSize, keepAlive, TimeUnit.SECONDS,
            queue, factory, new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}
```

### Rejection Policies

```java
// When pool is saturated (max threads reached AND queue is full):

// 1. AbortPolicy (DEFAULT) - throws RejectedExecutionException
new ThreadPoolExecutor.AbortPolicy();

// 2. CallerRunsPolicy - submitting thread runs the task
new ThreadPoolExecutor.CallerRunsPolicy();

// 3. DiscardPolicy - silently drops the task
new ThreadPoolExecutor.DiscardPolicy();

// 4. DiscardOldestPolicy - drops oldest queued task, retries submit
new ThreadPoolExecutor.DiscardOldestPolicy();

// 5. Custom policy
RejectedExecutionHandler customHandler = (runnable, executor) -> {
    System.err.println("Task rejected! Queue size: " + executor.getQueue().size());
    // Log, persist to database, or push to a backup queue
};
```

### Monitoring a Thread Pool

```java
public class ThreadPoolMonitor {
    public static void printStats(ThreadPoolExecutor pool) {
        System.out.println("=== Thread Pool Stats ===");
        System.out.println("Pool Size:        " + pool.getPoolSize());
        System.out.println("Core Pool Size:   " + pool.getCorePoolSize());
        System.out.println("Max Pool Size:    " + pool.getMaximumPoolSize());
        System.out.println("Active Threads:   " + pool.getActiveCount());
        System.out.println("Queued Tasks:     " + pool.getQueue().size());
        System.out.println("Completed Tasks:  " + pool.getCompletedTaskCount());
        System.out.println("Total Tasks:      " + pool.getTaskCount());
        System.out.println("Is Shutdown:      " + pool.isShutdown());
        System.out.println("Is Terminated:    " + pool.isTerminated());
    }

    public static void main(String[] args) throws InterruptedException {
        ThreadPoolExecutor pool = (ThreadPoolExecutor) Executors.newFixedThreadPool(4);

        for (int i = 0; i < 20; i++) {
            pool.submit(() -> {
                try { Thread.sleep(500); } catch (InterruptedException e) {}
            });
        }

        // Monitor every second
        ScheduledExecutorService monitor = Executors.newSingleThreadScheduledExecutor();
        monitor.scheduleAtFixedRate(() -> printStats(pool), 0, 1, TimeUnit.SECONDS);

        pool.shutdown();
        pool.awaitTermination(10, TimeUnit.SECONDS);
        monitor.shutdown();
    }
}
```

### Pool Sizing Guidelines

```java
public class PoolSizingGuide {

    // CPU-bound tasks: threads ≈ number of cores
    static int cpuBoundPoolSize() {
        return Runtime.getRuntime().availableProcessors();
    }

    // I/O-bound tasks: threads = cores * (1 + waitTime/computeTime)
    static int ioBoundPoolSize(double waitTime, double computeTime) {
        int cores = Runtime.getRuntime().availableProcessors();
        return (int) (cores * (1 + waitTime / computeTime));
    }

    // Example: DB calls with 200ms wait, 20ms compute
    // Pool size = 8 * (1 + 200/20) = 8 * 11 = 88 threads
}
```

---

## Anti-Patterns

❌ **Using `Executors.newCachedThreadPool()` for unbounded workloads:**
```java
// Can create thousands of threads and crash the JVM
ExecutorService pool = Executors.newCachedThreadPool();
for (int i = 0; i < 100_000; i++) {
    pool.submit(() -> Thread.sleep(10000)); // ❌ OOM!
}
```

❌ **Using unbounded `LinkedBlockingQueue` with limited threads:**
```java
// Queue grows without limit → memory exhaustion
new ThreadPoolExecutor(4, 4, 0, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>()); // ❌ Unbounded queue
```

❌ **Not shutting down thread pools:**
```java
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.submit(task);
// ❌ JVM won't exit - pool threads are still alive
```

## Best Practices

✅ Use bounded queues with `ThreadPoolExecutor` for production code  
✅ Always configure a rejection policy  
✅ Use custom `ThreadFactory` for naming and daemon configuration  
✅ Size pools based on workload type (CPU-bound vs I/O-bound)  
✅ Monitor pool metrics in production  
✅ Always call `shutdown()` in a finally block or shutdown hook

```java
// ✅ Proper shutdown pattern
ExecutorService executor = Executors.newFixedThreadPool(4);
try {
    // submit tasks
} finally {
    executor.shutdown();
    if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
        executor.shutdownNow();
    }
}
```

---

[← Runnable vs Callable](./q04-runnable-vs-callable.md) | [Next: Executor Framework →](./q06-executor-framework.md)

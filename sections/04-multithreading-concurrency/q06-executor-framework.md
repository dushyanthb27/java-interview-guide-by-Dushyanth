# Q6: Explain the Executor Framework

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What is the Executor framework in Java? Explain `ExecutorService`, `ScheduledExecutorService`, and the `Executors` factory class.

---

## Answer

### Executor Framework Hierarchy

```
                    ┌──────────┐
                    │ Executor │  (interface)
                    │          │
                    │execute(r)│
                    └────┬─────┘
                         │
                ┌────────▼─────────┐
                │ ExecutorService   │  (interface)
                │                  │
                │ submit()         │
                │ shutdown()       │
                │ invokeAll()      │
                │ invokeAny()      │
                └────────┬─────────┘
                         │
          ┌──────────────┼──────────────────┐
          │              │                  │
┌─────────▼──────┐  ┌───▼──────────────┐  ┌▼──────────────────┐
│AbstractExecutor │  │ScheduledExecutor │  │ForkJoinPool       │
│Service         │  │Service           │  │                   │
└────────┬───────┘  │                  │  │ work-stealing     │
         │          │ schedule()       │  └───────────────────┘
┌────────▼───────┐  │ scheduleAtFixed  │
│ThreadPool      │  │  Rate()         │
│Executor        │  │ scheduleWith     │
│                │  │  FixedDelay()   │
└────────────────┘  └──────────────────┘
```

### Executor Interface (Base)

```java
// Simplest abstraction - just execute a task
public interface Executor {
    void execute(Runnable command);
}

// Example: Custom executor
Executor synchronousExecutor = Runnable::run; // Runs in caller's thread
synchronousExecutor.execute(() -> System.out.println("Same thread!"));
```

### ExecutorService - The Workhorse

```java
public class ExecutorServiceDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(4);

        // ─── execute() vs submit() ───

        // execute(): fire-and-forget, no return value
        executor.execute(() -> System.out.println("execute(): no return"));

        // submit(Runnable): returns Future<?> (result is null on success)
        Future<?> f1 = executor.submit(() -> System.out.println("submit Runnable"));
        f1.get(); // null

        // submit(Callable): returns Future<V>
        Future<String> f2 = executor.submit(() -> "Hello from Callable");
        System.out.println(f2.get()); // "Hello from Callable"

        // submit(Runnable, result): returns Future<V> with predefined result
        Future<Integer> f3 = executor.submit(() -> System.out.println("with result"), 42);
        System.out.println(f3.get()); // 42

        executor.shutdown();
    }
}
```

### invokeAll vs invokeAny

```java
public class InvokeDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(3);

        List<Callable<String>> tasks = List.of(
            () -> { Thread.sleep(2000); return "Slow Service"; },
            () -> { Thread.sleep(500);  return "Fast Service"; },
            () -> { Thread.sleep(1000); return "Medium Service"; }
        );

        // ─── invokeAll: Wait for ALL tasks to complete ───
        List<Future<String>> allResults = executor.invokeAll(tasks);
        // Blocks until ALL tasks complete
        for (Future<String> future : allResults) {
            System.out.println("invokeAll result: " + future.get());
        }

        // invokeAll with timeout
        List<Future<String>> timedResults = executor.invokeAll(tasks, 1, TimeUnit.SECONDS);
        for (Future<String> future : timedResults) {
            if (future.isCancelled()) {
                System.out.println("Task was cancelled (timed out)");
            } else {
                System.out.println("Completed: " + future.get());
            }
        }

        // ─── invokeAny: Return FIRST successful result ───
        String firstResult = executor.invokeAny(tasks);
        // Blocks until first task completes, cancels the rest
        System.out.println("invokeAny result: " + firstResult); // "Fast Service"

        // invokeAny with timeout
        String timedFirst = executor.invokeAny(tasks, 3, TimeUnit.SECONDS);
        System.out.println("invokeAny timed: " + timedFirst);

        executor.shutdown();
    }
}
```

### Shutdown Strategies

```java
public class ShutdownDemo {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            executor.submit(() -> {
                try {
                    Thread.sleep(1000);
                    System.out.println("Task " + taskId + " completed");
                } catch (InterruptedException e) {
                    System.out.println("Task " + taskId + " interrupted");
                    Thread.currentThread().interrupt();
                }
            });
        }

        // ─── shutdown(): Graceful shutdown ───
        // - No new tasks accepted
        // - Already submitted tasks will complete
        executor.shutdown();

        // Check if terminated
        boolean terminated = executor.awaitTermination(30, TimeUnit.SECONDS);
        System.out.println("All tasks done: " + terminated);
    }

    static void aggressiveShutdown(ExecutorService executor) throws InterruptedException {
        // ─── shutdownNow(): Aggressive shutdown ───
        // - No new tasks accepted
        // - Running tasks are interrupted
        // - Returns list of tasks that were waiting in queue
        List<Runnable> pending = executor.shutdownNow();
        System.out.println("Pending tasks: " + pending.size());
    }

    // ✅ Recommended shutdown pattern
    static void bestPracticeShutdown(ExecutorService executor) {
        executor.shutdown(); // Disable new tasks
        try {
            // Wait for existing tasks to finish
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                executor.shutdownNow(); // Force shutdown
                // Wait for tasks to respond to cancellation
                if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                    System.err.println("Executor did not terminate!");
                }
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

### ScheduledExecutorService

```java
public class ScheduledExecutorDemo {
    public static void main(String[] args) throws Exception {
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

        // 1. One-time delayed execution
        ScheduledFuture<String> delayed = scheduler.schedule(
            () -> "Delayed result",
            3, TimeUnit.SECONDS
        );
        System.out.println("Will get in 3s: " + delayed.get());

        // 2. Fixed-rate: Every N seconds from start of each execution
        //    If execution takes longer than period, next starts immediately
        //    |--task--|--task--|--task--|
        //    0        1        2        3  (seconds)
        ScheduledFuture<?> fixedRate = scheduler.scheduleAtFixedRate(
            () -> {
                System.out.println("Fixed rate: " + LocalTime.now());
            },
            0,       // initial delay
            1,       // period
            TimeUnit.SECONDS
        );

        // 3. Fixed-delay: N seconds gap between end of one and start of next
        //    |--task--|  gap  |--task--|  gap  |--task--|
        ScheduledFuture<?> fixedDelay = scheduler.scheduleWithFixedDelay(
            () -> {
                System.out.println("Fixed delay: " + LocalTime.now());
                try { Thread.sleep(500); } catch (InterruptedException e) {}
            },
            0,       // initial delay
            1,       // delay between executions
            TimeUnit.SECONDS
        );

        // Let it run for 5 seconds
        Thread.sleep(5000);

        // Cancel scheduled tasks
        fixedRate.cancel(false);
        fixedDelay.cancel(false);

        scheduler.shutdown();
    }
}
```

### Executors Factory Class Summary

```java
public class ExecutorsFactoryGuide {

    // ✅ For predictable, bounded concurrency
    ExecutorService fixed = Executors.newFixedThreadPool(
        Runtime.getRuntime().availableProcessors()
    );

    // ⚠️ For many short-lived tasks (can grow unbounded)
    ExecutorService cached = Executors.newCachedThreadPool();

    // ✅ For sequential task execution
    ExecutorService single = Executors.newSingleThreadExecutor();

    // ✅ For scheduling periodic/delayed tasks
    ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);

    // ✅ Single-threaded scheduler
    ScheduledExecutorService singleScheduled = Executors.newSingleThreadScheduledExecutor();

    // ✅ For recursive/divide-and-conquer tasks (Java 8+)
    ExecutorService workStealing = Executors.newWorkStealingPool();

    // ✅ Virtual thread executor (Java 21+)
    // ExecutorService virtual = Executors.newVirtualThreadPerTaskExecutor();
}
```

### AutoCloseable ExecutorService (Java 19+)

```java
// Java 19+: ExecutorService extends AutoCloseable
public class AutoCloseableExecutor {
    public static void main(String[] args) {
        // ✅ try-with-resources automatically shuts down the executor
        try (ExecutorService executor = Executors.newFixedThreadPool(4)) {
            Future<String> result = executor.submit(() -> "Auto-closed!");
            System.out.println(result.get());
        } catch (Exception e) {
            e.printStackTrace();
        }
        // executor.shutdown() called automatically at end of try block
    }
}
```

### Practical Example: Task Orchestration

```java
public class TaskOrchestration {
    private final ExecutorService executor = Executors.newFixedThreadPool(8);

    public Map<String, Object> aggregateData(long userId) throws Exception {
        List<Callable<Map.Entry<String, Object>>> tasks = List.of(
            () -> Map.entry("profile", fetchProfile(userId)),
            () -> Map.entry("orders", fetchOrders(userId)),
            () -> Map.entry("notifications", fetchNotifications(userId)),
            () -> Map.entry("recommendations", fetchRecommendations(userId))
        );

        // Execute all in parallel, collect results
        Map<String, Object> result = new HashMap<>();
        List<Future<Map.Entry<String, Object>>> futures = executor.invokeAll(
            tasks, 5, TimeUnit.SECONDS
        );

        for (Future<Map.Entry<String, Object>> future : futures) {
            if (!future.isCancelled()) {
                Map.Entry<String, Object> entry = future.get();
                result.put(entry.getKey(), entry.getValue());
            }
        }

        return result;
    }

    private Object fetchProfile(long userId) throws InterruptedException {
        Thread.sleep(200);
        return Map.of("name", "Alice", "id", userId);
    }

    private Object fetchOrders(long userId) throws InterruptedException {
        Thread.sleep(500);
        return List.of("ORD-001", "ORD-002");
    }

    private Object fetchNotifications(long userId) throws InterruptedException {
        Thread.sleep(100);
        return List.of("Welcome!", "New offer");
    }

    private Object fetchRecommendations(long userId) throws InterruptedException {
        Thread.sleep(300);
        return List.of("Product-A", "Product-B");
    }
}
```

---

## Anti-Patterns

❌ **Using `Executors.newCachedThreadPool()` without understanding the risk:**
```java
// Unbounded thread creation - dangerous for bursty workloads
ExecutorService pool = Executors.newCachedThreadPool();
```

❌ **Submitting tasks after shutdown:**
```java
executor.shutdown();
executor.submit(task); // ❌ RejectedExecutionException
```

❌ **Not handling exceptions from submitted tasks:**
```java
executor.execute(() -> { throw new RuntimeException("Lost!"); }); // ❌ Exception lost
```

## Best Practices

✅ Use `ThreadPoolExecutor` directly in production for full control  
✅ Always shut down executor services in a finally block  
✅ Use `invokeAny()` for first-wins scenarios (failover, caching)  
✅ Use `invokeAll()` when you need all results (aggregation)  
✅ Add shutdown hooks for graceful termination:

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    executor.shutdown();
    try {
        executor.awaitTermination(10, TimeUnit.SECONDS);
    } catch (InterruptedException e) {
        executor.shutdownNow();
    }
}));
```

---

[← Thread Pools](./q05-thread-pools.md) | [Next: Synchronized Keyword →](./q07-synchronized-keyword.md)

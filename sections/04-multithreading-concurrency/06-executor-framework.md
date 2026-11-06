# Q6: Explain the Executor Framework

[← Back to Multithreading & Concurrency](./README.md)

---

## Question

What is the Executor framework in Java? Explain `ExecutorService`, `ScheduledExecutorService`, bulk execution methods (`invokeAll`, `invokeAny`), graceful shutdown patterns, and modern `AutoCloseable` support (Java 19+).

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
┌────────▼───────┘  │ scheduleAtFixed  │
│ThreadPool      │  │  Rate()         │
│Executor        │  │ scheduleWith     │
│                │  │  FixedDelay()   │
└────────────────┘  └──────────────────┘
```

### `Executor` and `ExecutorService`

```java
public class ExecutorServiceWorkflow {
    public static void main(String[] args) throws Exception {
        // AutoCloseable ExecutorService (Java 19+)
        try (ExecutorService executor = Executors.newFixedThreadPool(4)) {
            // 1. execute(Runnable) - fire and forget
            executor.execute(() -> System.out.println("Executing Runnable"));

            // 2. submit(Callable) - returns Future<V>
            Future<String> future = executor.submit(() -> {
                Thread.sleep(200);
                return "Result from Callable";
            });

            System.out.println(future.get());
        } // Automatically calls shutdown() and awaits termination
    }
}
```

### `invokeAll` vs `invokeAny`

```java
public class BulkExecutionDemo {
    public static void main(String[] args) throws Exception {
        try (ExecutorService executor = Executors.newFixedThreadPool(3)) {
            List<Callable<String>> tasks = List.of(
                () -> { Thread.sleep(1000); return "Task A"; },
                () -> { Thread.sleep(300);  return "Task B (Fast)"; },
                () -> { Thread.sleep(600);  return "Task C"; }
            );

            // 1. invokeAny: Returns result of the FIRST successfully completed task,
            //    canceling all remaining tasks.
            String fastest = executor.invokeAny(tasks);
            System.out.println("Fastest result: " + fastest); // Task B (Fast)

            // 2. invokeAll: Executes all tasks and returns list of Futures once ALL complete
            List<Future<String>> results = executor.invokeAll(tasks);
            for (Future<String> f : results) {
                System.out.println("invokeAll item: " + f.get());
            }
        }
    }
}
```

### `ScheduledExecutorService`

```java
public class SchedulerDemo {
    public static void main(String[] args) throws Exception {
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

        // 1. Delayed one-off execution
        scheduler.schedule(() -> System.out.println("Delayed task fired"), 2, TimeUnit.SECONDS);

        // 2. Fixed-rate execution (starts every N seconds regardless of duration)
        ScheduledFuture<?> fixedRateTask = scheduler.scheduleAtFixedRate(
            () -> System.out.println("Fixed rate tick: " + LocalTime.now()),
            0, 1, TimeUnit.SECONDS
        );

        // 3. Fixed-delay execution (N seconds gap between task completion and next start)
        ScheduledFuture<?> fixedDelayTask = scheduler.scheduleWithFixedDelay(
            () -> {
                System.out.println("Fixed delay tick starting");
                try { Thread.sleep(400); } catch (InterruptedException e) {}
            },
            0, 1, TimeUnit.SECONDS
        );

        Thread.sleep(3000);
        fixedRateTask.cancel(true);
        fixedDelayTask.cancel(true);
        scheduler.shutdown();
    }
}
```

### Graceful Shutdown Best Practice

```java
public class ShutdownBestPractices {
    public static void shutdownGracefully(ExecutorService executor) {
        executor.shutdown(); // Stop accepting new tasks
        try {
            // Wait a while for existing tasks to terminate
            if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                executor.shutdownNow(); // Force cancel currently executing tasks
                // Wait again for tasks to respond to interruption
                if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                    System.err.println("Pool did not terminate");
                }
            }
        } catch (InterruptedException ie) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

---

## ⚠️ Gotchas & Common Pitfalls

1. **Forgetting Shutdown**: Leaving `ExecutorService` instances unclosed keeps worker threads alive, preventing JVM process termination.
2. **Infinite Blocking in `invokeAll`**: Without a timeout overload (`executor.invokeAll(tasks, timeout, unit)`), `invokeAll` blocks indefinitely until every task finishes.
3. **Scheduled Thread Pool Exception Swallowing**: If a scheduled task throws an unchecked exception and it is not caught inside the task, **subsequent periodic executions of that task are suppressed silently**.
4. **Shared Scheduler Bottleneck**: Using a single-threaded scheduled executor (`newSingleThreadScheduledExecutor`) for long-running periodic tasks starves other scheduled tasks.

---

## Interviewer Expectations & Deep Mental Models

- **Executor Abstraction Benefits**: Decouples task submission from execution mechanics (threading policy, queuing strategy, thread reuse).
- **AutoCloseable (Java 19+)**: Mention how modern try-with-resources replaces boilerplate shutdown code safely.

---

[← Thread Pools](./05-thread-pools.md) | [Next: Synchronized Keyword →](./07-synchronized-keyword.md)

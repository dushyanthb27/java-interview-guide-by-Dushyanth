# Q4: Runnable vs Callable - What's the Difference?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question

What is the difference between `Runnable` and `Callable`? How do `Future`, `FutureTask`, and modern asynchronous alternatives (`CompletableFuture`, `StructuredTaskScope`) relate to them?

---

## Answer

### Interface Definitions

```java
// Runnable - since JDK 1.0 (FunctionalInterface)
@FunctionalInterface
public interface Runnable {
    void run(); // Returns void, cannot throw checked exceptions
}

// Callable - since JDK 1.5 (FunctionalInterface)
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception; // Returns generic value V, can throw checked exceptions
}
```

### Key Differences

| Feature | Runnable | Callable |
|---------|----------|----------|
| Return Type | `void` | Generic type `V` |
| Checked Exceptions | Cannot throw checked exceptions | Can throw any `Exception` |
| SAM Method | `run()` | `call()` |
| Introduced | JDK 1.0 | JDK 1.5 (`java.util.concurrent`) |
| Execution Context | `Thread`, `ExecutorService.execute()`, `submit()` | `ExecutorService.submit()`, `FutureTask` |

### Runnable Example

```java
public class RunnableDemo {
    public static void main(String[] args) {
        Runnable task = () -> {
            System.out.println("Processing in thread: " + Thread.currentThread().getName());
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt(); // Restore interrupted flag
            }
        };

        // Execution via Thread
        new Thread(task, "Runnable-Worker").start();

        // Execution via ExecutorService
        try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
            executor.execute(task); // Fire and forget
            Future<?> future = executor.submit(task); // Returns Future<?> (get() returns null)
        }
    }
}
```

### Callable Example

```java
public class CallableDemo {
    public static void main(String[] args) throws Exception {
        Callable<Double> priceTask = () -> {
            System.out.println("Calculating price in: " + Thread.currentThread().getName());
            Thread.sleep(1000);
            return 299.99; // Returns result
        };

        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            Future<Double> future = executor.submit(priceTask);

            System.out.println("Doing other work while task executes...");

            // Blocking get() retrieves result
            Double price = future.get();
            System.out.println("Calculated Price: $" + price);
        }
    }
}
```

### The `Future` Interface & Methods

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit) throws InterruptedException,
                                              ExecutionException,
                                              TimeoutException;
}
```

```java
public class FutureHandlingDemo {
    public static void main(String[] args) {
        try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
            Future<String> future = executor.submit(() -> {
                Thread.sleep(2000);
                return "Task Completed Successfully";
            });

            try {
                // Always use timeout in production to prevent indefinite hangs
                String result = future.get(3, TimeUnit.SECONDS);
                System.out.println(result);
            } catch (TimeoutException e) {
                System.err.println("Task timed out!");
                future.cancel(true); // Attempt cancellation with interrupt
            } catch (ExecutionException e) {
                // Encapsulates exception thrown inside call()
                System.err.println("Task failed: " + e.getCause().getMessage());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

### `FutureTask`: The Bridge

`FutureTask` implements both `Runnable` and `Future<V>`, serving as a concrete adapter:

```java
public class FutureTaskBridgeDemo {
    public static void main(String[] args) throws Exception {
        FutureTask<Integer> futureTask = new FutureTask<>(() -> {
            Thread.sleep(500);
            return 42;
        });

        // FutureTask implements Runnable, so it runs directly in a Thread
        Thread thread = new Thread(futureTask, "FutureTask-Thread");
        thread.start();

        // FutureTask implements Future, so we retrieve results easily
        Integer result = futureTask.get();
        System.out.println("FutureTask result: " + result);
    }
}
```

---

## ⚠️ Gotchas & Common Pitfalls

1. **Swallowing Exceptions with `execute()`**: Submitting tasks via `ExecutorService.execute(Runnable)` or throwing exceptions inside `Runnable` results in uncaught exceptions being lost or swallowed unless an `UncaughtExceptionHandler` is configured. `Callable` wraps exceptions in `ExecutionException`.
2. **Indefinite Blocking on `Future.get()`**: Calling `future.get()` without a timeout can block indefinitely if the underlying task hangs or deadlocks.
3. **Cancellation Misunderstanding**: Calling `future.cancel(true)` only interrupts the thread if it is responsive to interruption (e.g. handles `InterruptedException` or checks `Thread.currentThread().isInterrupted()`).
4. **Not Shutting Down Executor**: Failing to shut down executor services leaves worker threads active, preventing JVM termination.

---

## Interviewer Expectations & Deep Mental Models

- **Exception Transparency**: Explain why `Callable` is superior for computations that throw business or checked exceptions, as `ExecutionException` preserves the causal exception chain via `getCause()`.
- **Modern Evolution**: Be ready to contrast traditional `Future.get()` polling with non-blocking callback chaining (`CompletableFuture`) and scoped hierarchical task graphs (`StructuredTaskScope`).

---

[← Creating Threads](./03-creating-threads.md) | [Next: Thread Pools →](./05-thread-pools.md)

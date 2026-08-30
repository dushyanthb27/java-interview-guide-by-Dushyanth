# Q4: Runnable vs Callable - What's the Difference?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What is the difference between `Runnable` and `Callable`? How do `Future` and `FutureTask` relate to them?

---

## Answer

### Interface Definitions

```java
// Runnable - since JDK 1.0
@FunctionalInterface
public interface Runnable {
    void run(); // No return value, no checked exceptions
}

// Callable - since JDK 1.5
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception; // Returns a value, can throw checked exceptions
}
```

### Key Differences

| Feature | Runnable | Callable |
|---------|----------|----------|
| Return value | `void` | Generic type `V` |
| Checked exceptions | Cannot throw | Can throw `Exception` |
| Method name | `run()` | `call()` |
| Since | JDK 1.0 | JDK 1.5 |
| Used with | `Thread`, `ExecutorService.execute()` | `ExecutorService.submit()` |
| Functional interface | Yes | Yes |

### Runnable Example

```java
public class RunnableDemo {
    public static void main(String[] args) {
        Runnable task = () -> {
            System.out.println("Processing in: " + Thread.currentThread().getName());
            // Cannot return a value
            // Cannot throw checked exceptions
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                // Must handle checked exceptions internally
            }
        };

        // Use with Thread
        new Thread(task, "Worker").start();

        // Use with ExecutorService
        ExecutorService executor = Executors.newSingleThreadExecutor();
        executor.execute(task);        // void - fire and forget
        Future<?> future = executor.submit(task); // Returns Future<?>
        executor.shutdown();
    }
}
```

### Callable Example

```java
public class CallableDemo {
    public static void main(String[] args) throws Exception {
        Callable<Double> priceCalculator = () -> {
            System.out.println("Calculating price...");
            Thread.sleep(2000); // Can throw checked exceptions directly
            return 149.99;
        };

        ExecutorService executor = Executors.newFixedThreadPool(2);
        Future<Double> priceFuture = executor.submit(priceCalculator);

        System.out.println("Doing other work while price is calculated...");

        // Block and get the result
        Double price = priceFuture.get(); // Throws ExecutionException if call() threw
        System.out.println("Price: $" + price);

        executor.shutdown();
    }
}
```

### The Future Interface

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
public class FutureDemo {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(3);

        // Submit multiple tasks
        Future<String> future1 = executor.submit(() -> {
            Thread.sleep(1000);
            return "Task 1 result";
        });

        Future<String> future2 = executor.submit(() -> {
            Thread.sleep(2000);
            return "Task 2 result";
        });

        Future<String> future3 = executor.submit(() -> {
            throw new RuntimeException("Task 3 failed!");
        });

        try {
            // get() blocks until result is available
            System.out.println(future1.get());

            // get() with timeout
            System.out.println(future2.get(3, TimeUnit.SECONDS));

            // get() wraps exceptions in ExecutionException
            System.out.println(future3.get());
        } catch (ExecutionException e) {
            System.out.println("Task failed: " + e.getCause().getMessage());
        } catch (TimeoutException e) {
            System.out.println("Task timed out!");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // Cancellation
        Future<String> longTask = executor.submit(() -> {
            Thread.sleep(60000);
            return "Done";
        });

        boolean cancelled = longTask.cancel(true); // Interrupt if running
        System.out.println("Cancelled: " + cancelled);
        System.out.println("Is cancelled: " + longTask.isCancelled());
        System.out.println("Is done: " + longTask.isDone()); // true (cancelled counts as done)

        executor.shutdown();
    }
}
```

### FutureTask: Bridge Between Runnable and Callable

```java
public class FutureTaskDemo {
    public static void main(String[] args) throws Exception {
        // FutureTask wraps a Callable
        FutureTask<Integer> futureTask = new FutureTask<>(() -> {
            System.out.println("Computing...");
            Thread.sleep(1000);
            return 100;
        });

        // FutureTask implements Runnable, so it can be used with Thread
        Thread thread = new Thread(futureTask, "FutureTask-Thread");
        thread.start();

        // FutureTask also implements Future, so we can get the result
        System.out.println("Result: " + futureTask.get());

        // FutureTask from Runnable + result
        FutureTask<String> fromRunnable = new FutureTask<>(
            () -> System.out.println("Running"),
            "Default Result" // Returned when task completes
        );
        new Thread(fromRunnable).start();
        System.out.println("Runnable result: " + fromRunnable.get()); // "Default Result"
    }
}
```

### Practical Example: Parallel Service Calls

```java
public class ParallelServiceCalls {

    record UserProfile(String name, int age) {}
    record OrderHistory(List<String> orders) {}
    record Recommendations(List<String> items) {}

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(3);

        long userId = 12345L;

        // Submit parallel calls
        Future<UserProfile> profileFuture = executor.submit(() -> fetchProfile(userId));
        Future<OrderHistory> ordersFuture = executor.submit(() -> fetchOrders(userId));
        Future<Recommendations> recoFuture = executor.submit(() -> fetchRecommendations(userId));

        // Gather results (with timeouts)
        try {
            UserProfile profile = profileFuture.get(2, TimeUnit.SECONDS);
            OrderHistory orders = ordersFuture.get(2, TimeUnit.SECONDS);
            Recommendations recs = recoFuture.get(2, TimeUnit.SECONDS);

            System.out.println("Profile: " + profile);
            System.out.println("Orders: " + orders);
            System.out.println("Recommendations: " + recs);
        } catch (TimeoutException e) {
            System.out.println("Service call timed out");
            profileFuture.cancel(true);
            ordersFuture.cancel(true);
            recoFuture.cancel(true);
        }

        executor.shutdown();
    }

    static UserProfile fetchProfile(long id) throws InterruptedException {
        Thread.sleep(500); // Simulate API call
        return new UserProfile("Alice", 30);
    }

    static OrderHistory fetchOrders(long id) throws InterruptedException {
        Thread.sleep(800);
        return new OrderHistory(List.of("Order-1", "Order-2"));
    }

    static Recommendations fetchRecommendations(long id) throws InterruptedException {
        Thread.sleep(300);
        return new Recommendations(List.of("Item-A", "Item-B"));
    }
}
```

### Converting Between Runnable and Callable

```java
public class ConversionDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newSingleThreadExecutor();

        // Runnable → Callable using Executors.callable()
        Runnable runnable = () -> System.out.println("I'm a Runnable");
        Callable<Object> callable = Executors.callable(runnable);
        Callable<String> callableWithResult = Executors.callable(runnable, "Done");

        Future<String> result = executor.submit(callableWithResult);
        System.out.println(result.get()); // "Done"

        // Callable can be adapted to Runnable via FutureTask
        Callable<Integer> myCallable = () -> 42;
        FutureTask<Integer> futureTask = new FutureTask<>(myCallable);
        // FutureTask implements Runnable
        executor.execute(futureTask);
        System.out.println("Callable as Runnable result: " + futureTask.get());

        executor.shutdown();
    }
}
```

### Exception Handling Comparison

```java
public class ExceptionHandlingComparison {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newSingleThreadExecutor();

        // ❌ Runnable: exceptions are swallowed silently (unless using submit)
        executor.execute(() -> {
            throw new RuntimeException("Runnable exception");
            // This exception goes to UncaughtExceptionHandler
            // or is simply lost
        });

        // ✅ Callable: exceptions are captured in Future
        Future<String> future = executor.submit(() -> {
            throw new Exception("Callable exception");
        });

        try {
            future.get(); // Throws ExecutionException wrapping the original
        } catch (ExecutionException e) {
            System.out.println("Caught: " + e.getCause().getMessage());
            // "Caught: Callable exception"
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        executor.shutdown();
    }
}
```

---

## Anti-Patterns

❌ **Ignoring the Future returned by submit():**
```java
executor.submit(task); // ❌ Exception is silently swallowed!
```

❌ **Calling get() without timeout on untrusted tasks:**
```java
future.get(); // ❌ Can block forever if the task hangs
future.get(5, TimeUnit.SECONDS); // ✅ Always use timeout
```

❌ **Not shutting down ExecutorService:**
```java
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(task);
// ❌ JVM won't exit because non-daemon threads in pool are still alive
```

## Best Practices

✅ Use `Callable` when you need a return value or must throw checked exceptions  
✅ Always handle `ExecutionException` when calling `Future.get()`  
✅ Use `get(timeout, unit)` to avoid indefinite blocking  
✅ Cancel futures you no longer need to free resources  
✅ Prefer `CompletableFuture` for modern async programming with chaining

---

[← Creating Threads](./q03-creating-threads.md) | [Next: Thread Pools →](./q05-thread-pools.md)

# Q3: What Are the Different Ways to Create Threads?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What are the different ways to create and run threads in Java? Which approach is preferred and why?

---

## Answer

### Approach 1: Extending Thread Class

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running in: " + getName());
    }

    public static void main(String[] args) {
        MyThread thread = new MyThread();
        thread.setName("Custom-Thread");
        thread.start(); // Calls run() in a new thread
    }
}
```

### Approach 2: Implementing Runnable Interface

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }

    public static void main(String[] args) {
        Thread thread = new Thread(new MyRunnable(), "Runnable-Thread");
        thread.start();
    }
}
```

### Approach 3: Implementing Callable Interface

```java
public class MyCallable implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        System.out.println("Computing in: " + Thread.currentThread().getName());
        Thread.sleep(1000);
        return 42;
    }

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<Integer> future = executor.submit(new MyCallable());

        System.out.println("Result: " + future.get()); // Blocks until result is ready
        executor.shutdown();
    }
}
```

### Approach 4: Lambda-Based Threads (Java 8+)

```java
public class LambdaThreads {
    public static void main(String[] args) {
        // Lambda with Runnable
        Thread thread1 = new Thread(() -> {
            System.out.println("Lambda Runnable: " + Thread.currentThread().getName());
        }, "Lambda-Thread");
        thread1.start();

        // Lambda with ExecutorService
        ExecutorService executor = Executors.newFixedThreadPool(2);

        // Runnable lambda
        executor.execute(() -> System.out.println("Execute: " + Thread.currentThread().getName()));

        // Callable lambda
        Future<String> future = executor.submit(() -> {
            return "Hello from " + Thread.currentThread().getName();
        });

        try {
            System.out.println(future.get());
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }

        executor.shutdown();
    }
}
```

### Approach 5: Using FutureTask

```java
public class FutureTaskDemo {
    public static void main(String[] args) throws Exception {
        FutureTask<String> futureTask = new FutureTask<>(() -> {
            Thread.sleep(1000);
            return "Task completed!";
        });

        // Can be used with a raw Thread
        Thread thread = new Thread(futureTask, "FutureTask-Thread");
        thread.start();

        // Or with an ExecutorService
        // executor.execute(futureTask);

        System.out.println("Doing other work...");
        System.out.println("Result: " + futureTask.get()); // Blocks until done
    }
}
```

### Approach 6: CompletableFuture (Java 8+)

```java
public class CompletableFutureCreation {
    public static void main(String[] args) {
        // Runs asynchronously in ForkJoinPool.commonPool()
        CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
            System.out.println("Running in: " + Thread.currentThread().getName());
            return "Async result";
        });

        // With custom executor
        ExecutorService executor = Executors.newFixedThreadPool(4);
        CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> {
            return "Custom executor result";
        }, executor);

        cf.thenAccept(result -> System.out.println("Got: " + result));

        executor.shutdown();
    }
}
```

### Comparison Table

```
┌──────────────────┬─────────┬───────────┬───────────┬────────────────┐
│ Feature          │ Thread  │ Runnable  │ Callable  │ CompleteFuture │
├──────────────────┼─────────┼───────────┼───────────┼────────────────┤
│ Return value     │  No     │  No       │  Yes      │  Yes           │
│ Throw exception  │  No     │  No       │  Yes      │  Yes           │
│ Extend other     │  No     │  Yes      │  Yes      │  Yes           │
│  class           │         │           │           │                │
│ Lambda support   │  No     │  Yes      │  Yes      │  Yes           │
│ Chaining support │  No     │  No       │  No       │  Yes           │
│ Since            │ JDK 1.0 │ JDK 1.0   │ JDK 1.5   │  JDK 1.8      │
└──────────────────┴─────────┴───────────┴───────────┴────────────────┘
```

### Why Runnable is Preferred Over Thread

```java
// ❌ Extending Thread - wastes the single inheritance slot
public class TaskThread extends Thread {
    // Cannot extend any other class!
    @Override
    public void run() { /* task logic */ }
}

// ✅ Implementing Runnable - flexible composition
public class Task implements Runnable, Serializable { // Can implement multiple interfaces
    @Override
    public void run() { /* task logic */ }
}

// ✅ Separation of concerns
Runnable task = () -> System.out.println("Task logic");

// Same task, different execution strategies:
new Thread(task).start();                         // Raw thread
executor.execute(task);                           // Thread pool
scheduledExecutor.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS); // Scheduled
```

### Practical Example: All Approaches Combined

```java
public class ThreadCreationShowcase {
    public static void main(String[] args) throws Exception {
        System.out.println("Main thread: " + Thread.currentThread().getName());

        // 1. Thread subclass
        new Thread("Approach-1") {
            @Override
            public void run() {
                System.out.println("[" + getName() + "] Thread subclass");
            }
        }.start();

        // 2. Runnable
        new Thread(() -> System.out.println("[Approach-2] Runnable lambda")).start();

        // 3. Callable with ExecutorService
        ExecutorService pool = Executors.newCachedThreadPool();
        Future<Double> priceFuture = pool.submit(() -> {
            Thread.sleep(500);
            return 99.99;
        });

        // 4. CompletableFuture chain
        CompletableFuture.supplyAsync(() -> "Hello", pool)
            .thenApply(s -> s + " World")
            .thenAccept(s -> System.out.println("[Approach-4] " + s));

        // 5. Thread factory (custom naming & configuration)
        ThreadFactory factory = r -> {
            Thread t = new Thread(r);
            t.setName("Custom-" + t.getId());
            t.setDaemon(true);
            t.setUncaughtExceptionHandler((thread, ex) ->
                System.err.println(thread.getName() + " failed: " + ex.getMessage()));
            return t;
        };
        factory.newThread(() -> System.out.println("[Approach-5] Custom factory")).start();

        System.out.println("Price: " + priceFuture.get());
        pool.shutdown();
    }
}
```

### Thread Factory Pattern

```java
public class NamedThreadFactory implements ThreadFactory {
    private final AtomicInteger counter = new AtomicInteger(0);
    private final String prefix;
    private final boolean daemon;

    public NamedThreadFactory(String prefix, boolean daemon) {
        this.prefix = prefix;
        this.daemon = daemon;
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread thread = new Thread(r, prefix + "-" + counter.incrementAndGet());
        thread.setDaemon(daemon);
        thread.setUncaughtExceptionHandler((t, e) -> {
            System.err.println("Thread " + t.getName() + " failed with: " + e.getMessage());
        });
        return thread;
    }
}

// Usage
ExecutorService executor = Executors.newFixedThreadPool(4,
    new NamedThreadFactory("worker", false));
```

---

## Anti-Patterns

❌ **Calling `run()` instead of `start()`:**
```java
Thread thread = new Thread(() -> System.out.println("Task"));
thread.run();   // ❌ Runs in the CURRENT thread, no new thread is created!
thread.start(); // ✅ Starts a new thread
```

❌ **Calling `start()` twice:**
```java
thread.start();
thread.start(); // ❌ Throws IllegalThreadStateException!
```

❌ **Creating threads manually in a loop:**
```java
for (Request req : requests) {
    new Thread(() -> process(req)).start(); // ❌ Unbounded thread creation
}
```

## Best Practices

✅ Prefer `Runnable` / `Callable` over extending `Thread`  
✅ Use `ExecutorService` for thread management  
✅ Use `CompletableFuture` for async operations with chaining  
✅ Use `ThreadFactory` for consistent thread configuration  
✅ Always name your threads for debugging

---

[← Thread Lifecycle](./q02-thread-lifecycle.md) | [Next: Runnable vs Callable →](./q04-runnable-vs-callable.md)

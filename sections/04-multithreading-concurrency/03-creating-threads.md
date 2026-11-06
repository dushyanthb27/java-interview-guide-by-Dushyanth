# Q3: What Are the Different Ways to Create Threads?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question

What are the different ways to create and run threads in Java, including modern Java 21+ Virtual Threads (`Thread.ofVirtual()`) and Structured Concurrency (`StructuredTaskScope`)? Which approach is preferred and why?

---

## Answer

### Approach 1: Extending `Thread` Class

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running in platform thread: " + getName());
    }

    public static void main(String[] args) {
        MyThread thread = new MyThread();
        thread.setName("Custom-Platform-Thread");
        thread.start(); // Spawns OS-backed platform thread
    }
}
```

### Approach 2: Implementing `Runnable` Interface

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

### Approach 3: Implementing `Callable` Interface

```java
public class MyCallable implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        System.out.println("Computing in executor pool: " + Thread.currentThread().getName());
        Thread.sleep(500);
        return 42;
    }

    public static void main(String[] args) throws Exception {
        try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
            Future<Integer> future = executor.submit(new MyCallable());
            System.out.println("Result: " + future.get());
        }
    }
}
```

### Approach 4: Lambda-Based Threads (Java 8+)

```java
public class LambdaThreads {
    public static void main(String[] args) {
        Thread thread = new Thread(() -> {
            System.out.println("Lambda Runnable executing on: " + Thread.currentThread().getName());
        }, "Lambda-Worker");
        thread.start();
    }
}
```

### Approach 5: Modern Virtual Threads (Java 21+)

Java 21 introduces **Virtual Threads** (`Thread.ofVirtual()`), which are lightweight threads managed by the JVM rather than the OS kernel. Millions can run concurrently.

```java
public class VirtualThreadCreation {
    public static void main(String[] args) throws InterruptedException {
        // 1. Unstarted virtual thread builder
        Thread vThread = Thread.ofVirtual().unstarted(() -> {
            System.out.println("Running virtual thread: " + Thread.currentThread());
        });
        vThread.start();
        vThread.join();

        // 2. Direct start factory
        Thread.ofVirtual().name("worker-vthread").start(() -> {
            System.out.println("Quick virtual thread execution");
        });

        // 3. Virtual thread per task ExecutorService (Java 21+)
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            executor.submit(() -> {
                System.out.println("Submitted to virtual thread executor");
            });
        } // Auto-closes and awaits termination
    }
}
```

### Approach 6: Structured Concurrency (`StructuredTaskScope` - Java 21+)

```java
public class StructuredConcurrencyDemo {
    record UserProfile(String name) {}
    record OrderHistory(List<String> orders) {}
    public record UserDashboard(UserProfile profile, OrderHistory orders) {}

    public UserDashboard fetchDashboard(long userId) throws Exception {
        // StructuredTaskScope ensures subtasks complete within scope boundaries
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            Supplier<UserProfile> profileTask = scope.fork(() -> fetchProfile(userId));
            Supplier<OrderHistory> ordersTask = scope.fork(() -> fetchOrders(userId));

            scope.join();           // Join all subtasks
            scope.throwIfFailed();  // Propagate exception if any subtask failed

            return new UserDashboard(profileTask.get(), ordersTask.get());
        }
    }

    private UserProfile fetchProfile(long id) { return new UserProfile("Alice"); }
    private OrderHistory fetchOrders(long id) { return new OrderHistory(List.of("ORD-1")); }
}
```

### Comparison Matrix

| Approach | Thread Type | Return Value | Checked Exceptions | Best Use Case |
|----------|-------------|--------------|-------------------|---------------|
| Extending `Thread` | Platform | `void` | No | ❌ Anti-pattern (wastes single inheritance slot) |
| Implementing `Runnable` | Platform | `void` | No | Legacy task definition |
| Implementing `Callable` | Platform / Pool | Generic `V` | Yes (`throws Exception`) | Tasks requiring result & checked exceptions |
| Virtual Threads (`Thread.ofVirtual`) | Virtual | `void` / via Future | Optional | High-throughput I/O bound servers |
| `StructuredTaskScope` | Virtual tasks | Aggregated | Yes | Concurrent hierarchical task trees |

---

## ⚠️ Gotchas & Common Pitfalls

1. **Calling `run()` Instead of `start()`**: Executing `thread.run()` runs the task synchronously on the current thread, creating **no new thread** whatsoever.
2. **Starting a Thread Twice**: Invoking `thread.start()` a second time throws `IllegalThreadStateException`.
3. **Pinning Virtual Threads**: Executing synchronized blocks, native methods, or foreign function calls inside virtual threads can "pin" the virtual thread to its carrier OS thread, blocking carrier scaling. (Use `ReentrantLock` instead of `synchronized` in high-throughput virtual thread code).
4. **Unbounded Platform Thread Creation**: Spawning raw platform threads (`new Thread(...)`) without an executor pool risks `OutOfMemoryError` under high traffic.
5. **Forgetting AutoCloseable Shutdown**: Failing to close `ExecutorService` or `StructuredTaskScope` leaves background non-daemon threads active or causes resource leaks.

---

## Interviewer Expectations & Deep Mental Models

- **Why `Runnable` over `Thread`**: Emphasize composition over inheritance, separation of task definition from execution mechanics, and task reusability across thread pools and virtual executors.
- **Virtual Thread Architecture**: Understand how the JVM schedules virtual threads onto a pool of carrier platform threads (default pool size = number of available processors) using work-stealing `ForkJoinPool`.

---

[← Thread Lifecycle](./02-thread-lifecycle.md) | [Next: Runnable vs Callable →](./04-runnable-vs-callable.md)

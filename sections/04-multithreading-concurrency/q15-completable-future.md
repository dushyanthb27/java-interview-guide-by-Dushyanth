# Q15: Explain CompletableFuture in Depth

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
How does `CompletableFuture` work? Explain creation, chaining, combining, and exception handling.

---

## Answer

### Why CompletableFuture?

```
Future (JDK 5):                    CompletableFuture (JDK 8):
- Blocking get()                   - Non-blocking callbacks
- No chaining                      - Full chaining pipeline
- No combining                     - Combine multiple futures
- No exception handling pipeline   - Declarative error handling
- Manual completion impossible     - Manual completion possible
```

### Creation

```java
public class CompletableFutureCreation {
    public static void main(String[] args) throws Exception {

        // 1. supplyAsync - with return value (uses ForkJoinPool.commonPool())
        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
            return "Hello from " + Thread.currentThread().getName();
        });

        // 2. runAsync - no return value (Void)
        CompletableFuture<Void> cf2 = CompletableFuture.runAsync(() -> {
            System.out.println("Running in " + Thread.currentThread().getName());
        });

        // 3. With custom executor
        ExecutorService executor = Executors.newFixedThreadPool(4);
        CompletableFuture<String> cf3 = CompletableFuture.supplyAsync(
            () -> "Custom executor result",
            executor
        );

        // 4. Already completed
        CompletableFuture<String> completed = CompletableFuture.completedFuture("Done!");

        // 5. Failed future (Java 9+)
        CompletableFuture<String> failed = CompletableFuture.failedFuture(
            new RuntimeException("Oops!")
        );

        // 6. Manual completion
        CompletableFuture<String> manual = new CompletableFuture<>();
        // Later: manual.complete("result") or manual.completeExceptionally(ex)

        executor.shutdown();
    }
}
```

### Chaining: Transform Results

```java
public class ChainingDemo {
    public static void main(String[] args) throws Exception {

        // thenApply: Transform result (like map) - same thread or FJP
        CompletableFuture<Integer> lengthFuture =
            CompletableFuture.supplyAsync(() -> "Hello World")
                .thenApply(String::length);        // 11
        System.out.println("Length: " + lengthFuture.get());

        // thenAccept: Consume result (no return value)
        CompletableFuture.supplyAsync(() -> "Hello")
            .thenAccept(s -> System.out.println("Got: " + s));

        // thenRun: Run action after completion (no access to result)
        CompletableFuture.supplyAsync(() -> "Hello")
            .thenRun(() -> System.out.println("Done!"));

        // thenCompose: FlatMap - chain dependent async operations
        CompletableFuture<String> composed =
            CompletableFuture.supplyAsync(() -> 42)
                .thenCompose(id -> fetchUserName(id));  // Returns CompletableFuture
        System.out.println("User: " + composed.get());

        // Multi-step pipeline
        CompletableFuture.supplyAsync(() -> "raw data")
            .thenApply(data -> data.trim())
            .thenApply(data -> data.toUpperCase())
            .thenApply(data -> "Processed: " + data)
            .thenAccept(System.out::println);

        Thread.sleep(500); // Wait for async completion
    }

    static CompletableFuture<String> fetchUserName(int userId) {
        return CompletableFuture.supplyAsync(() -> "User-" + userId);
    }
}
```

### thenApply vs thenCompose

```java
// thenApply: Function returns a value → CompletableFuture<T>
CompletableFuture<String> result1 = future.thenApply(x -> x.toString());
// Input: CompletableFuture<Integer>
// Output: CompletableFuture<String>

// thenCompose: Function returns a CompletableFuture → flattened
CompletableFuture<String> result2 = future.thenCompose(x -> asyncToString(x));
// Input: CompletableFuture<Integer>
// Output: CompletableFuture<String>  (NOT CompletableFuture<CompletableFuture<String>>)

// Analogy:
// thenApply  = map()     in streams
// thenCompose = flatMap() in streams
```

### Combining Multiple Futures

```java
public class CombiningDemo {
    public static void main(String[] args) throws Exception {

        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> "Hello");
        CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> "World");

        // thenCombine: Combine results of TWO futures
        CompletableFuture<String> combined = cf1.thenCombine(cf2,
            (s1, s2) -> s1 + " " + s2);
        System.out.println(combined.get()); // "Hello World"

        // allOf: Wait for ALL futures to complete
        CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> fetchFromA());
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> fetchFromB());
        CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> fetchFromC());

        CompletableFuture<Void> allDone = CompletableFuture.allOf(f1, f2, f3);

        // Get all results after allOf completes
        CompletableFuture<List<String>> allResults = allDone.thenApply(v ->
            List.of(f1.join(), f2.join(), f3.join())
        );
        System.out.println("All: " + allResults.get());

        // anyOf: Get FIRST completed future
        CompletableFuture<Object> firstDone = CompletableFuture.anyOf(f1, f2, f3);
        System.out.println("First: " + firstDone.get());

        // Pattern: collect allOf results with stream
        List<CompletableFuture<String>> futures = List.of(f1, f2, f3);
        CompletableFuture<List<String>> allWithStream = CompletableFuture
            .allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList()));
    }

    static String fetchFromA() { sleep(100); return "A"; }
    static String fetchFromB() { sleep(200); return "B"; }
    static String fetchFromC() { sleep(150); return "C"; }
    static void sleep(long ms) { try { Thread.sleep(ms); } catch (InterruptedException e) {} }
}
```

### Exception Handling

```java
public class ExceptionHandlingDemo {
    public static void main(String[] args) throws Exception {

        // exceptionally: Recover from exceptions (catch block equivalent)
        CompletableFuture<String> recovered =
            CompletableFuture.supplyAsync(() -> {
                if (true) throw new RuntimeException("API Error");
                return "data";
            })
            .exceptionally(ex -> {
                System.out.println("Error: " + ex.getMessage());
                return "default value"; // Fallback
            });
        System.out.println(recovered.get()); // "default value"

        // handle: Process both success AND failure
        CompletableFuture<String> handled =
            CompletableFuture.supplyAsync(() -> {
                if (Math.random() > 0.5) throw new RuntimeException("Random fail");
                return "success";
            })
            .handle((result, ex) -> {
                if (ex != null) {
                    return "Handled error: " + ex.getMessage();
                }
                return "Result: " + result;
            });

        // whenComplete: Side-effect on completion (doesn't transform result)
        CompletableFuture.supplyAsync(() -> "data")
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    System.err.println("Failed: " + ex.getMessage());
                } else {
                    System.out.println("Success: " + result);
                }
            });

        // Exception propagation in chains
        CompletableFuture.supplyAsync(() -> "start")
            .thenApply(s -> { throw new RuntimeException("Step 2 failed"); })
            .thenApply(s -> s + " step 3")  // SKIPPED (exception propagates)
            .thenApply(s -> s + " step 4")  // SKIPPED
            .exceptionally(ex -> "Recovered at the end");
    }
}
```

### Async Variants

```java
// Every chaining method has 3 variants:
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> "Hello");

// 1. thenApply     - runs in same thread as previous stage (or caller)
cf.thenApply(s -> s.toUpperCase());

// 2. thenApplyAsync - runs in ForkJoinPool.commonPool()
cf.thenApplyAsync(s -> s.toUpperCase());

// 3. thenApplyAsync(fn, executor) - runs in specified executor
cf.thenApplyAsync(s -> s.toUpperCase(), myExecutor);
```

### Timeout Support (Java 9+)

```java
public class TimeoutDemo {
    public static void main(String[] args) throws Exception {

        // orTimeout: Completes exceptionally if not done within timeout
        CompletableFuture<String> withTimeout = CompletableFuture
            .supplyAsync(() -> {
                try { Thread.sleep(5000); } catch (InterruptedException e) {}
                return "slow result";
            })
            .orTimeout(2, TimeUnit.SECONDS)
            .exceptionally(ex -> "Timed out: " + ex.getMessage());

        // completeOnTimeout: Completes with default value if not done
        CompletableFuture<String> withDefault = CompletableFuture
            .supplyAsync(() -> {
                try { Thread.sleep(5000); } catch (InterruptedException e) {}
                return "slow result";
            })
            .completeOnTimeout("default value", 2, TimeUnit.SECONDS);

        System.out.println(withDefault.get()); // "default value"
    }
}
```

### Real-World Example: Service Aggregation

```java
public class ServiceAggregator {
    private final ExecutorService executor = Executors.newFixedThreadPool(8);

    public CompletableFuture<UserDashboard> buildDashboard(long userId) {
        CompletableFuture<UserProfile> profileFuture =
            CompletableFuture.supplyAsync(() -> fetchProfile(userId), executor);

        CompletableFuture<List<Order>> ordersFuture =
            CompletableFuture.supplyAsync(() -> fetchOrders(userId), executor);

        CompletableFuture<List<Notification>> notifFuture =
            CompletableFuture.supplyAsync(() -> fetchNotifications(userId), executor)
                .completeOnTimeout(List.of(), 2, TimeUnit.SECONDS); // Graceful timeout

        return profileFuture.thenCombine(ordersFuture, (profile, orders) ->
                new PartialDashboard(profile, orders))
            .thenCombine(notifFuture, (partial, notifications) ->
                new UserDashboard(partial.profile(), partial.orders(), notifications))
            .exceptionally(ex -> {
                log.error("Dashboard build failed for user " + userId, ex);
                return UserDashboard.empty();
            });
    }

    record UserProfile(String name) {}
    record Order(String id) {}
    record Notification(String message) {}
    record PartialDashboard(UserProfile profile, List<Order> orders) {}
    record UserDashboard(UserProfile profile, List<Order> orders, List<Notification> notifs) {
        static UserDashboard empty() { return new UserDashboard(null, List.of(), List.of()); }
    }

    UserProfile fetchProfile(long id) { return new UserProfile("User-" + id); }
    List<Order> fetchOrders(long id) { return List.of(new Order("ORD-1")); }
    List<Notification> fetchNotifications(long id) { return List.of(new Notification("Hi!")); }
}
```

---

## Anti-Patterns

❌ **Blocking with `get()` on the common ForkJoinPool:**
```java
CompletableFuture.supplyAsync(() -> ...)
    .thenApply(x -> anotherFuture.get()); // ❌ Blocks a common pool thread
```

❌ **Not handling exceptions in async chains:**
```java
CompletableFuture.supplyAsync(() -> riskyOperation()); // ❌ Exception is silently swallowed
```

❌ **Using `join()` in production without timeout considerations**

## Best Practices

✅ Use custom executors for I/O-bound operations (not the common pool)  
✅ Always add `exceptionally()` or `handle()` for error recovery  
✅ Use `orTimeout()` / `completeOnTimeout()` (Java 9+) for resilience  
✅ Use `thenCompose()` (not `thenApply()`) when the function returns a `CompletableFuture`  
✅ Use `allOf()` + stream pattern to collect multiple future results  
✅ Prefer `join()` over `get()` (throws unchecked CompletionException)

---

[← Thread Communication](./q14-thread-communication.md) | [Next: Parallel Streams →](./q16-parallel-streams.md)

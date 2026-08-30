# Q11: CompletableFuture Advanced Patterns

## 📋 Question
Explain advanced CompletableFuture composition patterns. Cover thenApply vs thenCompose, combining futures, exception handling, timeouts, and real-world async pipeline design.

## Difficulty: 🟠 Senior

---

## 💡 Answer

`CompletableFuture<T>` (Java 8) is a **composable, non-blocking future** that supports functional-style chaining of async operations. Unlike the basic `Future`, it enables building complex async pipelines with transformation, combination, and robust error handling — all without blocking threads.

### Core Composition: thenApply vs thenCompose

```java
package com.interview.java8;

import java.util.concurrent.*;

public class CompletableFutureCompositionDemo {

    public static void main(String[] args) throws Exception {

        // ═══════════════════════════════════════════════════
        // thenApply — synchronous transformation (like Stream.map)
        // Takes Function<T, R>, returns CompletableFuture<R>
        // ═══════════════════════════════════════════════════
        CompletableFuture<String> greeting = CompletableFuture
                .supplyAsync(() -> "Hello")
                .thenApply(s -> s + ", World")     // Synchronous transform
                .thenApply(String::toUpperCase);   // Chain transforms

        System.out.println(greeting.get());  // HELLO, WORLD

        // ═══════════════════════════════════════════════════
        // thenCompose — async chaining (like Stream.flatMap)
        // Takes Function<T, CompletableFuture<R>>
        // Flattens the result (avoids CompletableFuture<CompletableFuture<R>>)
        // ═══════════════════════════════════════════════════
        CompletableFuture<String> userData = CompletableFuture
                .supplyAsync(() -> getUserId())
                .thenCompose(id -> fetchUserAsync(id))   // Returns CF<User>
                .thenCompose(user -> fetchOrdersAsync(user)); // Returns CF<String>

        System.out.println(userData.get());

        // ❌ thenApply would give CompletableFuture<CompletableFuture<User>>
        // ✅ thenCompose flattens to CompletableFuture<User>

        // ═══════════════════════════════════════════════════
        // thenCombine — combines TWO independent futures
        // ═══════════════════════════════════════════════════
        CompletableFuture<Double> priceFuture = CompletableFuture
                .supplyAsync(() -> getPrice("AAPL"));
        CompletableFuture<Double> exchangeRate = CompletableFuture
                .supplyAsync(() -> getExchangeRate("USD", "INR"));

        // Both run in parallel, combined when both complete
        CompletableFuture<String> converted = priceFuture.thenCombine(
                exchangeRate,
                (price, rate) -> String.format("₹%.2f", price * rate)
        );
        System.out.println("Price in INR: " + converted.get());
    }

    // Simulated async service methods
    static String getUserId() { return "user-123"; }
    static CompletableFuture<String> fetchUserAsync(String id) {
        return CompletableFuture.supplyAsync(() -> "Alice");
    }
    static CompletableFuture<String> fetchOrdersAsync(String user) {
        return CompletableFuture.supplyAsync(() -> "Orders for " + user);
    }
    static double getPrice(String ticker) { return 150.0; }
    static double getExchangeRate(String from, String to) { return 83.5; }
}
```

### Combining Multiple Futures: allOf and anyOf

```java
package com.interview.java8;

import java.util.*;
import java.util.concurrent.*;
import java.util.stream.*;

public class CombiningFuturesDemo {

    public static void main(String[] args) throws Exception {

        // ═══════════════════════════════════════════════════
        // allOf — waits for ALL futures to complete (returns Void)
        // ═══════════════════════════════════════════════════
        List<String> services = List.of("UserService", "OrderService", "PaymentService");

        List<CompletableFuture<String>> futures = services.stream()
                .map(svc -> CompletableFuture.supplyAsync(() -> callService(svc)))
                .collect(Collectors.toList());

        // Wait for all and collect results
        CompletableFuture<List<String>> allResults = CompletableFuture
                .allOf(futures.toArray(CompletableFuture[]::new))
                .thenApply(v -> futures.stream()
                        .map(CompletableFuture::join)  // Safe — all completed
                        .collect(Collectors.toList()));

        System.out.println("All results: " + allResults.get());

        // ═══════════════════════════════════════════════════
        // anyOf — completes when FIRST future completes
        // ═══════════════════════════════════════════════════
        CompletableFuture<Object> fastest = CompletableFuture.anyOf(
                CompletableFuture.supplyAsync(() -> { sleep(200); return "Server-A"; }),
                CompletableFuture.supplyAsync(() -> { sleep(100); return "Server-B"; }),
                CompletableFuture.supplyAsync(() -> { sleep(300); return "Server-C"; })
        );
        System.out.println("Fastest: " + fastest.get());  // Server-B
    }

    static String callService(String name) {
        sleep(100);
        return name + ": OK";
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

### Exception Handling

```java
package com.interview.java8;

import java.util.concurrent.*;

public class ExceptionHandlingDemo {

    public static void main(String[] args) throws Exception {

        // ═══════════════════════════════════════════════════
        // exceptionally — recover from exceptions (like catch)
        // ═══════════════════════════════════════════════════
        CompletableFuture<String> withRecovery = CompletableFuture
                .supplyAsync(() -> {
                    if (true) throw new RuntimeException("Service down!");
                    return "Success";
                })
                .exceptionally(ex -> {
                    System.out.println("Error: " + ex.getMessage());
                    return "Fallback value";  // Recovery value
                });
        System.out.println(withRecovery.get());  // Fallback value

        // ═══════════════════════════════════════════════════
        // handle — processes BOTH success and failure
        // ═══════════════════════════════════════════════════
        CompletableFuture<String> handled = CompletableFuture
                .supplyAsync(() -> riskyOperation())
                .handle((result, ex) -> {
                    if (ex != null) {
                        return "Error: " + ex.getMessage();
                    }
                    return "Success: " + result;
                });
        System.out.println(handled.get());

        // ═══════════════════════════════════════════════════
        // whenComplete — observe result/error WITHOUT transforming
        // ═══════════════════════════════════════════════════
        CompletableFuture<String> observed = CompletableFuture
                .supplyAsync(() -> "Result")
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        System.err.println("Failed: " + ex.getMessage());
                    } else {
                        System.out.println("Completed: " + result);
                    }
                });
        // Returns the ORIGINAL result (unlike handle which can transform)

        // ═══════════════════════════════════════════════════
        // Exception handling comparison
        // ═══════════════════════════════════════════════════
        // exceptionally(fn)    → catches error, provides fallback VALUE
        // handle(bifn)         → handles both success/error, TRANSFORMS result
        // whenComplete(bifn)   → observes both, does NOT transform (side-effect only)
    }

    static String riskyOperation() {
        return "OK";
    }
}
```

### Timeout Handling (Java 9+)

```java
package com.interview.java8;

import java.util.concurrent.*;

public class TimeoutDemo {

    public static void main(String[] args) throws Exception {

        // ═══════════════════════════════════════════════════
        // orTimeout — fails with TimeoutException if not completed in time
        // ═══════════════════════════════════════════════════
        CompletableFuture<String> withTimeout = CompletableFuture
                .supplyAsync(() -> {
                    sleep(5000);  // Slow operation
                    return "Done";
                })
                .orTimeout(2, TimeUnit.SECONDS)  // Java 9+
                .exceptionally(ex -> "Timed out: " + ex.getMessage());

        System.out.println(withTimeout.get());  // Timed out: ...

        // ═══════════════════════════════════════════════════
        // completeOnTimeout — provides fallback value on timeout
        // ═══════════════════════════════════════════════════
        CompletableFuture<String> withFallback = CompletableFuture
                .supplyAsync(() -> {
                    sleep(5000);
                    return "Real result";
                })
                .completeOnTimeout("Cached result", 2, TimeUnit.SECONDS);  // Java 9+

        System.out.println(withFallback.get());  // Cached result
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

### Custom Executors and Async Variants

```java
package com.interview.java8;

import java.util.concurrent.*;

public class CustomExecutorDemo {

    public static void main(String[] args) throws Exception {
        ExecutorService customPool = Executors.newFixedThreadPool(4);

        try {
            // ✅ supplyAsync with custom executor
            CompletableFuture<String> result = CompletableFuture
                    .supplyAsync(() -> heavyComputation(), customPool)
                    .thenApplyAsync(s -> transform(s), customPool)    // Async variant
                    .thenApply(s -> s.toUpperCase());                  // Sync variant

            // Naming convention:
            // thenApply    → runs on COMPLETING thread (or caller if already done)
            // thenApplyAsync → runs on ForkJoinPool.commonPool()
            // thenApplyAsync(fn, executor) → runs on specified executor

            System.out.println(result.get());
        } finally {
            customPool.shutdown();
        }
    }

    // ✅ Real-world async pipeline
    static CompletableFuture<String> processOrder(String orderId, ExecutorService pool) {
        return CompletableFuture
                .supplyAsync(() -> validateOrder(orderId), pool)
                .thenComposeAsync(order -> checkInventory(order), pool)
                .thenComposeAsync(order -> processPayment(order), pool)
                .thenApplyAsync(order -> sendConfirmation(order), pool)
                .orTimeout(30, TimeUnit.SECONDS)
                .exceptionally(ex -> "Order failed: " + ex.getMessage());
    }

    static String heavyComputation() { return "computed"; }
    static String transform(String s) { return "transformed-" + s; }
    static String validateOrder(String id) { return "validated-" + id; }
    static CompletableFuture<String> checkInventory(String o) {
        return CompletableFuture.completedFuture("inventory-" + o);
    }
    static CompletableFuture<String> processPayment(String o) {
        return CompletableFuture.completedFuture("paid-" + o);
    }
    static String sendConfirmation(String o) { return "confirmed-" + o; }
}
```

### Method Comparison Table

| Method | Input | Async? | Error Handling | Returns |
|--------|-------|--------|---------------|---------|
| `thenApply` | `Function<T,R>` | No | Propagates | `CF<R>` |
| `thenCompose` | `Function<T,CF<R>>` | No | Propagates | `CF<R>` |
| `thenCombine` | `CF<U>, BiFunction<T,U,R>` | No | Propagates | `CF<R>` |
| `thenAccept` | `Consumer<T>` | No | Propagates | `CF<Void>` |
| `thenRun` | `Runnable` | No | Propagates | `CF<Void>` |
| `exceptionally` | `Function<Throwable,T>` | — | Recovers | `CF<T>` |
| `handle` | `BiFunction<T,Throwable,R>` | No | Transforms | `CF<R>` |
| `whenComplete` | `BiConsumer<T,Throwable>` | No | Observes | `CF<T>` |
| `allOf` | `CF<?>...` | — | Propagates first | `CF<Void>` |
| `anyOf` | `CF<?>...` | — | Returns first | `CF<Object>` |

### ⚠️ Common Mistake
**Using `thenApply` when you need `thenCompose`**: If your transformation function returns a `CompletableFuture`, using `thenApply` gives you `CompletableFuture<CompletableFuture<T>>` — a nested future. Use `thenCompose` to flatten it, just like `flatMap` for streams. Also, **beware of `get()` without timeout** — it blocks indefinitely if the future never completes. Always use `get(timeout, unit)` or `join()` (which throws unchecked exceptions).

### ⚡ Performance Tip
The default executor for `supplyAsync` and `*Async` variants is `ForkJoinPool.commonPool()`, which is shared across the entire JVM. For I/O-bound operations (HTTP calls, database queries), use a **dedicated thread pool** to avoid starving CPU-bound tasks. Also note: `thenApply` (non-async) runs on the completing thread, which might be the caller's thread if the future is already completed — this can cause unexpected blocking. Use `thenApplyAsync` if the transformation is non-trivial.

### 🔧 JVM Internals
Internally, `CompletableFuture` maintains a **Treiber stack** (lock-free concurrent stack) of dependent actions. When a future completes, it pops and executes all registered actions. The `*Async` variants submit actions to an executor rather than running them inline. `join()` vs `get()`: both block, but `join()` wraps exceptions in `CompletionException` (unchecked), while `get()` wraps in `ExecutionException` (checked). For parallel streams and `CompletableFuture`, the `ForkJoinPool.commonPool()` parallelism defaults to `Runtime.getRuntime().availableProcessors() - 1`.

---

## 🎯 Interviewer Insight
Interviewers expect you to distinguish `thenApply` vs `thenCompose` (the most common question) and explain the analogy to `map` vs `flatMap`. A strong answer demonstrates real-world usage patterns: fan-out/fan-in with `allOf`, timeout handling, and proper exception recovery. Knowing the difference between `handle`, `exceptionally`, and `whenComplete` shows mastery of the error-handling model.

## ⚡ Follow-up Questions
1. What thread does `thenApply` (non-async) execute on?
2. How do you implement retry logic with CompletableFuture?
3. How does CompletableFuture compare to reactive streams (Project Reactor)?

## 📚 Related Topics
- [Lambda Expressions](./q01-lambda-expressions.md)
- [Stream API Operations](./q04-stream-api-operations.md)
- [Virtual Threads](./q18-virtual-threads-detail.md)

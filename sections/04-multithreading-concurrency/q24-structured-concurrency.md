# Q24: What is Structured Concurrency?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What is Structured Concurrency in Java? How does `StructuredTaskScope` work and how does it improve over unstructured approaches?

---

## Answer

### The Problem with Unstructured Concurrency

```
Unstructured (CompletableFuture):
┌─────────────────────────────────────────────────┐
│ main() {                                        │
│   cf1 = supplyAsync(taskA);  ──► runs somewhere │
│   cf2 = supplyAsync(taskB);  ──► runs somewhere │
│                                                 │
│   // If taskA fails:                            │
│   // - taskB keeps running (wasted work!)       │
│   // - No clean cancellation                    │
│   // - Error handling is manual                 │
│   // - Thread dump doesn't show parent-child    │
│ }                                               │
└─────────────────────────────────────────────────┘

Structured (StructuredTaskScope):
┌─────────────────────────────────────────────────┐
│ try (var scope = new StructuredTaskScope()) {    │
│   ┌──────────────────────────────────────┐      │
│   │ subtask1 = scope.fork(taskA);        │      │
│   │ subtask2 = scope.fork(taskB);        │      │
│   │                                      │      │
│   │ scope.join();                        │      │
│   │ // ALL subtasks complete or cancel   │      │
│   │ // before scope exits               │      │
│   └──────────────────────────────────────┘      │
│   // Clean boundary: nothing leaks out          │
│ }                                               │
└─────────────────────────────────────────────────┘
```

### Key Principles

1. **Task lifetime is bounded by its scope** - subtasks can't outlive the parent
2. **Cancellation propagates** - if one fails, siblings are cancelled
3. **Errors propagate** - exceptions from subtasks surface at the scope
4. **Observability** - thread dumps show parent-child relationships

### StructuredTaskScope (Java 21 Preview)

```java
// Note: This is a preview feature in Java 21
// Run with: --enable-preview

import java.util.concurrent.StructuredTaskScope;

public class StructuredConcurrencyDemo {

    record UserProfile(String name) {}
    record OrderHistory(List<String> orders) {}
    record UserPage(UserProfile profile, OrderHistory orders) {}

    // ✅ Structured concurrency
    public UserPage fetchUserPage(long userId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

            // Fork subtasks (each runs in its own virtual thread)
            StructuredTaskScope.Subtask<UserProfile> profileTask =
                scope.fork(() -> fetchProfile(userId));

            StructuredTaskScope.Subtask<OrderHistory> ordersTask =
                scope.fork(() -> fetchOrders(userId));

            // Wait for both to complete (or one to fail)
            scope.join();

            // Propagate any exceptions
            scope.throwIfFailed();

            // Both succeeded - combine results
            return new UserPage(profileTask.get(), ordersTask.get());
        }
        // Scope closes here - all subtasks guaranteed to be done
    }

    UserProfile fetchProfile(long id) throws InterruptedException {
        Thread.sleep(500);
        return new UserProfile("Alice");
    }

    OrderHistory fetchOrders(long id) throws InterruptedException {
        Thread.sleep(800);
        return new OrderHistory(List.of("ORD-001", "ORD-002"));
    }
}
```

### ShutdownOnFailure

If **any** subtask fails, all other subtasks are cancelled:

```java
public class ShutdownOnFailureDemo {

    public String fetchWithFailover() throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

            var profileTask = scope.fork(() -> fetchProfile(1));
            var ordersTask = scope.fork(() -> {
                throw new RuntimeException("Database error!");
            });
            var recsTask = scope.fork(() -> fetchRecommendations(1));

            scope.join();

            // This will throw because ordersTask failed
            scope.throwIfFailed();
            // The exception from ordersTask is thrown here
            // profileTask and recsTask were cancelled automatically!

            return "unreachable";
        }
    }

    /*
     * What happens:
     * 1. All three tasks are forked
     * 2. ordersTask throws RuntimeException
     * 3. ShutdownOnFailure cancels profileTask and recsTask
     * 4. scope.join() returns (all tasks are done or cancelled)
     * 5. scope.throwIfFailed() throws ExecutionException wrapping the RuntimeException
     */

    String fetchProfile(long id) throws InterruptedException {
        Thread.sleep(500);
        return "profile";
    }

    String fetchRecommendations(long id) throws InterruptedException {
        Thread.sleep(300);
        return "recommendations";
    }
}
```

### ShutdownOnSuccess

Returns the **first successful** result, cancelling the rest:

```java
public class ShutdownOnSuccessDemo {

    // First-wins pattern (e.g., query multiple replicas)
    public String queryFastestReplica() throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {

            scope.fork(() -> queryReplica("replica-1", 200));
            scope.fork(() -> queryReplica("replica-2", 100));
            scope.fork(() -> queryReplica("replica-3", 300));

            scope.join();

            // Returns the first successful result
            // Other subtasks are cancelled
            return scope.result();
        }
    }

    String queryReplica(String name, long latencyMs) throws InterruptedException {
        Thread.sleep(latencyMs);
        return "Result from " + name;
    }

    /*
     * What happens:
     * 1. All three tasks start in parallel
     * 2. replica-2 completes first (100ms)
     * 3. ShutdownOnSuccess cancels replica-1 and replica-3
     * 4. scope.result() returns "Result from replica-2"
     */
}
```

### Comparison: CompletableFuture vs Structured Concurrency

```java
// ❌ Unstructured: CompletableFuture
public UserPage fetchUnstructured(long userId) {
    CompletableFuture<UserProfile> profileFuture =
        CompletableFuture.supplyAsync(() -> fetchProfile(userId));

    CompletableFuture<OrderHistory> ordersFuture =
        CompletableFuture.supplyAsync(() -> fetchOrders(userId));

    // Problems:
    // 1. If profileFuture fails, ordersFuture keeps running
    // 2. If this method throws before get(), futures leak
    // 3. No parent-child relationship in thread dumps
    // 4. Cancellation is manual and error-prone

    try {
        return new UserPage(profileFuture.get(), ordersFuture.get());
    } catch (Exception e) {
        // Must manually cancel the other future
        profileFuture.cancel(true);
        ordersFuture.cancel(true);
        throw new RuntimeException(e);
    }
}

// ✅ Structured: StructuredTaskScope
public UserPage fetchStructured(long userId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        var profile = scope.fork(() -> fetchProfile(userId));
        var orders = scope.fork(() -> fetchOrders(userId));

        scope.join();
        scope.throwIfFailed();

        return new UserPage(profile.get(), orders.get());
    }
    // Automatic cleanup, cancellation, and error handling!
}
```

### Nested Scopes

```java
public class NestedScopes {

    public FullReport generateReport(long userId) throws Exception {
        try (var outerScope = new StructuredTaskScope.ShutdownOnFailure()) {

            var summaryTask = outerScope.fork(() -> fetchSummary(userId));
            var detailsTask = outerScope.fork(() -> fetchDetails(userId));

            outerScope.join();
            outerScope.throwIfFailed();

            return new FullReport(summaryTask.get(), detailsTask.get());
        }
    }

    // This method also uses structured concurrency internally
    private Details fetchDetails(long userId) throws Exception {
        try (var innerScope = new StructuredTaskScope.ShutdownOnFailure()) {

            var ordersTask = innerScope.fork(() -> fetchOrders(userId));
            var paymentsTask = innerScope.fork(() -> fetchPayments(userId));

            innerScope.join();
            innerScope.throwIfFailed();

            return new Details(ordersTask.get(), paymentsTask.get());
        }
        // Inner scope completes before outer scope continues
        // Clean, hierarchical structure!
    }

    record FullReport(String summary, Details details) {}
    record Details(String orders, String payments) {}

    String fetchSummary(long id) throws InterruptedException {
        Thread.sleep(200); return "summary";
    }
    String fetchOrders(long id) throws InterruptedException {
        Thread.sleep(300); return "orders";
    }
    String fetchPayments(long id) throws InterruptedException {
        Thread.sleep(400); return "payments";
    }
}
```

### Benefits Over Unstructured Concurrency

```
┌─────────────────────┬──────────────────────┬──────────────────────┐
│ Aspect              │ Unstructured (CF)    │ Structured (STS)     │
├─────────────────────┼──────────────────────┼──────────────────────┤
│ Lifetime management │ Manual               │ Automatic (scope)    │
│ Cancellation        │ Manual per future    │ Automatic propagation│
│ Error handling      │ Scattered            │ Centralized (join)   │
│ Thread leaks        │ Possible             │ Impossible           │
│ Observability       │ No hierarchy         │ Parent-child visible │
│ Code clarity        │ Callback chains      │ Sequential-looking   │
│ Resource cleanup    │ Try-catch-finally    │ Try-with-resources   │
└─────────────────────┴──────────────────────┴──────────────────────┘
```

### Timeout Support

```java
public UserPage fetchWithTimeout(long userId) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        var profile = scope.fork(() -> fetchProfile(userId));
        var orders = scope.fork(() -> fetchOrders(userId));

        // Join with timeout
        scope.joinUntil(Instant.now().plusSeconds(5));
        scope.throwIfFailed();

        return new UserPage(profile.get(), orders.get());
    }
    // If timeout expires, all subtasks are cancelled
}
```

---

## Anti-Patterns

❌ **Using StructuredTaskScope for CPU-bound work (use Fork/Join instead)**  
❌ **Ignoring the preview status - API may change between Java versions**  
❌ **Accessing subtask results before `join()`:**
```java
var task = scope.fork(() -> compute());
task.get(); // ❌ Must call scope.join() first!
```

❌ **Mixing structured and unstructured concurrency in the same workflow**

## Best Practices

✅ Use `ShutdownOnFailure` when ALL subtasks must succeed  
✅ Use `ShutdownOnSuccess` for first-wins / failover patterns  
✅ Always call `scope.join()` before accessing results  
✅ Use with virtual threads for maximum benefit  
✅ Nest scopes for hierarchical concurrent workflows  
✅ Set timeouts with `joinUntil()` for resilience  
✅ Combine with `ScopedValue` instead of `ThreadLocal`

---

[← Virtual Threads](./q23-virtual-threads.md) | [Next: Reactive Basics →](./q25-reactive-basics.md)

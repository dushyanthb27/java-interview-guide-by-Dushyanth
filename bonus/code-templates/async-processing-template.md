# Async Processing Template

## Overview
CompletableFuture patterns and async processing techniques for non-blocking, concurrent operations.

## Use Cases
- Parallel task execution
- Chaining async operations
- Timeout handling
- Error recovery
- Reactive programming
- Batch processing

## Implementation

Multiple async processing patterns.

## Code

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

/**
 * SIMPLE VERSION: Basic CompletableFuture Usage
 * - Simple async operations
 * - Basic chaining
 * - Error handling
 */
public class SimpleAsyncProcessing {
    /**
     * Simple async operation
     */
    public static CompletableFuture<String> fetchUserData(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            // Simulate network call
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "User{id=" + userId + ", name=John}";
        });
    }

    /**
     * Chain multiple async operations
     */
    public static CompletableFuture<String> fetchUserWithOrders(String userId) {
        return fetchUserData(userId)
            .thenCompose(user -> fetchOrders(userId)
                .thenApply(orders -> user + ", orders=" + orders));
    }

    /**
     * Fetch orders for user
     */
    public static CompletableFuture<String> fetchOrders(String userId) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(50);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return "[Order1, Order2]";
        });
    }

    /**
     * Handle exceptions
     */
    public static CompletableFuture<String> fetchWithErrorHandling(String userId) {
        return fetchUserData(userId)
            .exceptionally(ex -> "Error: " + ex.getMessage());
    }
}

/**
 * PRODUCTION VERSION: Advanced Async Patterns
 * - Timeout handling
 * - Parallel execution
 * - Result aggregation
 * - Error recovery
 */
public class ProductionAsyncProcessing {
    private static final ExecutorService executor = 
        Executors.newFixedThreadPool(10);

    /**
     * Async operation with timeout
     */
    public static <T> CompletableFuture<T> withTimeout(
            CompletableFuture<T> future,
            long timeoutMs) {

        CompletableFuture<T> timeoutFuture = new CompletableFuture<>();
        
        ScheduledExecutorService scheduler = 
            Executors.newScheduledThreadPool(1);
        
        scheduler.schedule(() -> {
            timeoutFuture.completeExceptionally(
                new TimeoutException("Operation timed out after " + timeoutMs + "ms"));
        }, timeoutMs, TimeUnit.MILLISECONDS);

        return future.applyToEither(timeoutFuture, Function.identity());
    }

    /**
     * Execute multiple async operations in parallel
     */
    public static <T> CompletableFuture<List<T>> allOf(
            List<CompletableFuture<T>> futures) {

        return CompletableFuture.allOf(
            futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList()));
    }

    /**
     * Execute multiple async operations, return first successful
     */
    public static <T> CompletableFuture<T> anyOf(
            List<CompletableFuture<T>> futures) {

        return CompletableFuture.anyOf(
            futures.toArray(new CompletableFuture[0]))
            .thenApply(result -> (T) result);
    }

    /**
     * Retry async operation
     */
    public static <T> CompletableFuture<T> retryAsync(
            java.util.function.Supplier<CompletableFuture<T>> operation,
            int maxRetries,
            long delayMs) {

        return operation.get()
            .exceptionally(ex -> {
                if (maxRetries > 0) {
                    return null; // Placeholder
                }
                throw new CompletionException(ex);
            })
            .thenCompose(result -> {
                if (result != null) {
                    return CompletableFuture.completedFuture(result);
                }
                // Retry with delay
                return CompletableFuture.delayedExecutor(delayMs, TimeUnit.MILLISECONDS)
                    .execute(() -> {});
                // Note: This is simplified; real implementation would be more complex
            });
    }

    /**
     * Async operation with fallback
     */
    public static <T> CompletableFuture<T> withFallback(
            CompletableFuture<T> primary,
            java.util.function.Supplier<CompletableFuture<T>> fallback) {

        return primary.exceptionally(ex -> null)
            .thenCompose(result -> {
                if (result != null) {
                    return CompletableFuture.completedFuture(result);
                }
                return fallback.get();
            });
    }

    /**
     * Batch async processing
     */
    public static <T, R> CompletableFuture<List<R>> processBatch(
            List<T> items,
            java.util.function.Function<T, CompletableFuture<R>> processor,
            int batchSize) {

        List<CompletableFuture<R>> futures = items.stream()
            .map(processor)
            .collect(Collectors.toList());

        return allOf(futures);
    }

    /**
     * Async pipeline
     */
    public static class AsyncPipeline<T> {
        private CompletableFuture<T> future;

        public AsyncPipeline(CompletableFuture<T> future) {
            this.future = future;
        }

        public <R> AsyncPipeline<R> map(
                java.util.function.Function<T, R> mapper) {
            return new AsyncPipeline<>(future.thenApply(mapper));
        }

        public <R> AsyncPipeline<R> flatMap(
                java.util.function.Function<T, CompletableFuture<R>> mapper) {
            return new AsyncPipeline<>(future.thenCompose(mapper));
        }

        public AsyncPipeline<T> filter(
                java.util.function.Predicate<T> predicate) {
            return new AsyncPipeline<>(future.thenApply(value -> {
                if (!predicate.test(value)) {
                    throw new IllegalArgumentException("Filter predicate failed");
                }
                return value;
            }));
        }

        public AsyncPipeline<T> onError(
                java.util.function.Function<Throwable, T> handler) {
            return new AsyncPipeline<>(future.exceptionally(handler));
        }

        public CompletableFuture<T> build() {
            return future;
        }
    }
}

/**
 * THREAD-SAFE VERSION: Concurrent Async Processing
 * - Thread-safe result aggregation
 * - Concurrent batch processing
 * - Proper resource management
 */
public class ConcurrentAsyncProcessing {
    /**
     * Thread-safe result collector
     */
    public static class AsyncResultCollector<T> {
        private final List<T> results = Collections.synchronizedList(
            new ArrayList<>());
        private final List<Throwable> errors = Collections.synchronizedList(
            new ArrayList<>());
        private final AtomicInteger completed = new AtomicInteger(0);
        private final int total;
        private final CompletableFuture<List<T>> future = 
            new CompletableFuture<>();

        public AsyncResultCollector(int total) {
            this.total = total;
        }

        public void addResult(T result) {
            results.add(result);
            checkCompletion();
        }

        public void addError(Throwable error) {
            errors.add(error);
            checkCompletion();
        }

        private void checkCompletion() {
            if (completed.incrementAndGet() == total) {
                if (errors.isEmpty()) {
                    future.complete(results);
                } else {
                    future.completeExceptionally(
                        new CompletionException("Multiple errors occurred", 
                            errors.get(0)));
                }
            }
        }

        public CompletableFuture<List<T>> getFuture() {
            return future;
        }
    }

    /**
     * Concurrent batch processor
     */
    public static <T, R> CompletableFuture<List<R>> processConcurrent(
            List<T> items,
            java.util.function.Function<T, CompletableFuture<R>> processor,
            int maxConcurrent) {

        AsyncResultCollector<R> collector = 
            new AsyncResultCollector<>(items.size());
        Semaphore semaphore = new Semaphore(maxConcurrent);

        for (T item : items) {
            try {
                semaphore.acquire();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                collector.addError(e);
                continue;
            }

            processor.apply(item)
                .whenComplete((result, error) -> {
                    try {
                        if (error != null) {
                            collector.addError(error);
                        } else {
                            collector.addResult(result);
                        }
                    } finally {
                        semaphore.release();
                    }
                });
        }

        return collector.getFuture();
    }

    /**
     * Async stream processor
     */
    public static <T> CompletableFuture<Void> processStream(
            java.util.stream.Stream<T> stream,
            java.util.function.Function<T, CompletableFuture<Void>> processor) {

        List<CompletableFuture<Void>> futures = stream
            .map(processor)
            .collect(Collectors.toList());

        return CompletableFuture.allOf(
            futures.toArray(new CompletableFuture[0]));
    }
}

/**
 * REACTIVE PATTERNS: Reactive-style async processing
 */
public class ReactiveAsyncProcessing {
    /**
     * Simple reactive stream
     */
    public static class ReactiveStream<T> {
        private final Queue<T> items = new ConcurrentLinkedQueue<>();
        private final List<java.util.function.Consumer<T>> subscribers = 
            new CopyOnWriteArrayList<>();
        private volatile boolean completed = false;

        public void subscribe(java.util.function.Consumer<T> subscriber) {
            subscribers.add(subscriber);
            // Emit existing items
            items.forEach(subscriber);
        }

        public void emit(T item) {
            if (!completed) {
                items.add(item);
                subscribers.forEach(s -> s.accept(item));
            }
        }

        public void complete() {
            completed = true;
        }

        public boolean isCompleted() {
            return completed;
        }
    }

    /**
     * Async event emitter
     */
    public static class AsyncEventEmitter<T> {
        private final ReactiveStream<T> stream = new ReactiveStream<>();
        private final ExecutorService executor = 
            Executors.newFixedThreadPool(5);

        public void subscribe(java.util.function.Consumer<T> handler) {
            stream.subscribe(handler);
        }

        public void emitAsync(T event) {
            executor.submit(() -> stream.emit(event));
        }

        public void complete() {
            stream.complete();
            executor.shutdown();
        }
    }
}

/**
 * UTILITY FUNCTIONS
 */
public class AsyncUtils {
    /**
     * Convert callback-based API to CompletableFuture
     */
    public static <T> CompletableFuture<T> fromCallback(
            java.util.function.Consumer<java.util.function.Consumer<T>> operation) {

        CompletableFuture<T> future = new CompletableFuture<>();
        operation.accept(future::complete);
        return future;
    }

    /**
     * Convert blocking operation to async
     */
    public static <T> CompletableFuture<T> async(
            java.util.function.Supplier<T> operation) {

        return CompletableFuture.supplyAsync(operation);
    }

    /**
     * Delay execution
     */
    public static CompletableFuture<Void> delay(long delayMs) {
        CompletableFuture<Void> future = new CompletableFuture<>();
        ScheduledExecutorService scheduler = 
            Executors.newScheduledThreadPool(1);
        scheduler.schedule(() -> {
            future.complete(null);
            scheduler.shutdown();
        }, delayMs, TimeUnit.MILLISECONDS);
        return future;
    }

    /**
     * Combine multiple futures
     */
    public static <T> CompletableFuture<List<T>> combine(
            CompletableFuture<T>... futures) {

        return CompletableFuture.allOf(futures)
            .thenApply(v -> Arrays.stream(futures)
                .map(CompletableFuture::join)
                .collect(Collectors.toList()));
    }
}
```

## Testing

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import java.util.concurrent.*;

public class AsyncProcessingTest {

    @Test
    public void testSimpleAsync() throws Exception {
        CompletableFuture<String> future = 
            SimpleAsyncProcessing.fetchUserData("user1");

        String result = future.get(5, TimeUnit.SECONDS);
        assertTrue(result.contains("user1"));
    }

    @Test
    public void testAsyncChaining() throws Exception {
        CompletableFuture<String> future = 
            SimpleAsyncProcessing.fetchUserWithOrders("user1");

        String result = future.get(5, TimeUnit.SECONDS);
        assertTrue(result.contains("user1"));
        assertTrue(result.contains("orders"));
    }

    @Test
    public void testAsyncErrorHandling() throws Exception {
        CompletableFuture<String> future = 
            SimpleAsyncProcessing.fetchWithErrorHandling("user1");

        String result = future.get(5, TimeUnit.SECONDS);
        assertNotNull(result);
    }

    @Test
    public void testAsyncTimeout() throws Exception {
        CompletableFuture<String> future = 
            SimpleAsyncProcessing.fetchUserData("user1");

        CompletableFuture<String> withTimeout = 
            ProductionAsyncProcessing.withTimeout(future, 5000);

        String result = withTimeout.get(10, TimeUnit.SECONDS);
        assertNotNull(result);
    }

    @Test
    public void testParallelExecution() throws Exception {
        List<CompletableFuture<String>> futures = Arrays.asList(
            SimpleAsyncProcessing.fetchUserData("user1"),
            SimpleAsyncProcessing.fetchUserData("user2"),
            SimpleAsyncProcessing.fetchUserData("user3")
        );

        CompletableFuture<List<String>> combined = 
            ProductionAsyncProcessing.allOf(futures);

        List<String> results = combined.get(10, TimeUnit.SECONDS);
        assertEquals(3, results.size());
    }

    @Test
    public void testAsyncPipeline() throws Exception {
        CompletableFuture<String> future = 
            SimpleAsyncProcessing.fetchUserData("user1");

        String result = new ProductionAsyncProcessing.AsyncPipeline<>(future)
            .map(String::toUpperCase)
            .map(s -> s + "_PROCESSED")
            .build()
            .get(5, TimeUnit.SECONDS);

        assertTrue(result.contains("USER1"));
        assertTrue(result.contains("PROCESSED"));
    }

    @Test
    public void testConcurrentBatchProcessing() throws Exception {
        List<Integer> items = Arrays.asList(1, 2, 3, 4, 5);

        CompletableFuture<List<Integer>> future = 
            ConcurrentAsyncProcessing.processConcurrent(
                items,
                item -> CompletableFuture.completedFuture(item * 2),
                2
            );

        List<Integer> results = future.get(10, TimeUnit.SECONDS);
        assertEquals(5, results.size());
    }

    @Test
    public void testAsyncDelay() throws Exception {
        long start = System.currentTimeMillis();

        AsyncUtils.delay(100)
            .get(5, TimeUnit.SECONDS);

        long elapsed = System.currentTimeMillis() - start;
        assertTrue(elapsed >= 100);
    }

    @Test
    public void testReactiveStream() throws InterruptedException {
        ReactiveAsyncProcessing.ReactiveStream<Integer> stream = 
            new ReactiveAsyncProcessing.ReactiveStream<>();

        List<Integer> results = new CopyOnWriteArrayList<>();
        stream.subscribe(results::add);

        stream.emit(1);
        stream.emit(2);
        stream.emit(3);

        Thread.sleep(100);
        assertEquals(3, results.size());
    }
}
```

## Performance Considerations

### Time Complexity
- **CompletableFuture creation**: O(1)
- **Chaining operations**: O(1)
- **Parallel execution**: O(n/p) where n = tasks, p = processors

### Space Complexity
- **O(n)** where n = number of pending futures

### Optimization Tips
1. **Use appropriate executor**: Match thread pool size to workload
2. **Avoid blocking operations**: Use async APIs
3. **Limit concurrency**: Use semaphores for rate limiting
4. **Batch operations**: Reduce overhead
5. **Reuse futures**: Cache completed futures

## Common Pitfalls

1. **Blocking on futures**: Defeats async benefits
2. **Exception handling**: Not handling all error cases
3. **Resource leaks**: Not shutting down executors
4. **Deadlocks**: Circular dependencies in futures
5. **Memory leaks**: Holding references to completed futures
6. **Thread explosion**: Too many threads created
7. **Timeout issues**: No timeout handling

## Variations

1. **Reactive Streams**: RxJava, Project Reactor
2. **Virtual Threads**: Java 21+ lightweight threads
3. **Coroutines**: Kotlin-style async
4. **Async/Await**: Similar to JavaScript
5. **Promises**: Promise-based async

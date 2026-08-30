# How to Optimize Concurrent Code?

[Home](../../README.md) > [Performance Optimization](./README.md) > Concurrency Optimization

---

## Overview

Concurrency optimization aims to maximize parallelism while minimizing contention overhead. Key strategies include reducing lock contention, using lock-free algorithms, proper thread pool sizing, and avoiding false sharing.

---

## 1. Lock Contention Reduction

```java
// BAD: Single coarse-grained lock - high contention
public class CoarseGrainedCounter {
    private final Map<String, Long> counters = new HashMap<>();

    public synchronized void increment(String key) {    // ALL threads block here
        counters.merge(key, 1L, Long::sum);
    }

    public synchronized long get(String key) {           // Readers block too!
        return counters.getOrDefault(key, 0L);
    }
}

// BETTER: Fine-grained locking with ConcurrentHashMap
public class FineGrainedCounter {
    private final ConcurrentHashMap<String, LongAdder> counters =
        new ConcurrentHashMap<>();

    public void increment(String key) {
        counters.computeIfAbsent(key, k -> new LongAdder()).increment();
        // No global lock; ConcurrentHashMap uses segment-level locking
    }

    public long get(String key) {
        LongAdder adder = counters.get(key);
        return adder != null ? adder.sum() : 0L;
    }
}

// BEST: Lock-free with LongAdder (for high-contention counters)
public class HighThroughputCounter {
    private final LongAdder counter = new LongAdder();

    public void increment() {
        counter.increment(); // each thread updates its own cell
    }

    public long get() {
        return counter.sum(); // aggregates all cells
    }
}
```

### Lock Contention Strategies

| Strategy                | Technique                        | Use Case                |
|------------------------|----------------------------------|--------------------------|
| Reduce lock scope       | Lock only critical section       | General                  |
| Lock splitting          | Multiple locks for resources     | Independent data         |
| Lock striping           | Array of locks (e.g., CHM)       | Hash-based structures    |
| Read-write locks        | `ReentrantReadWriteLock`         | Read-heavy workloads     |
| Lock-free algorithms    | CAS operations                   | Counters, queues         |
| Lock elimination        | Thread-local data                | No sharing needed        |

---

## 2. Lock-Free Algorithms

```java
import java.util.concurrent.atomic.*;

// Lock-free stack using CAS
public class LockFreeStack<T> {
    private final AtomicReference<Node<T>> top = new AtomicReference<>();

    private static class Node<T> {
        final T value;
        final Node<T> next;

        Node(T value, Node<T> next) {
            this.value = value;
            this.next = next;
        }
    }

    public void push(T value) {
        Node<T> newHead;
        Node<T> oldHead;
        do {
            oldHead = top.get();
            newHead = new Node<>(value, oldHead);
        } while (!top.compareAndSet(oldHead, newHead)); // CAS retry loop
    }

    public T pop() {
        Node<T> oldHead;
        Node<T> newHead;
        do {
            oldHead = top.get();
            if (oldHead == null) return null;
            newHead = oldHead.next;
        } while (!top.compareAndSet(oldHead, newHead));
        return oldHead.value;
    }
}

// AtomicInteger vs synchronized comparison
public class AtomicVsSynchronized {

    // Synchronized: thread acquires monitor, others block
    private int syncCounter = 0;
    public synchronized void syncIncrement() {
        syncCounter++;
    }

    // Atomic: lock-free CAS, no blocking
    private final AtomicInteger atomicCounter = new AtomicInteger(0);
    public void atomicIncrement() {
        atomicCounter.incrementAndGet();
    }

    // LongAdder: best for high contention (striped cells)
    private final LongAdder adderCounter = new LongAdder();
    public void adderIncrement() {
        adderCounter.increment();
    }
}
```

### Performance Comparison (8 threads, 10M increments each)

```
synchronized counter:  ~3200 ms  (high contention on monitor)
AtomicLong:            ~1800 ms  (CAS contention)
LongAdder:             ~350 ms   (cell striping, minimal contention)
```

---

## 3. Thread Pool Sizing

### CPU-Bound Tasks

```java
// Formula: threads = N (number of CPU cores)
int cpuCores = Runtime.getRuntime().availableProcessors();

ExecutorService cpuPool = Executors.newFixedThreadPool(cpuCores);

// Example: Parallel computation
List<Future<Double>> futures = new ArrayList<>();
for (int i = 0; i < 100; i++) {
    final int task = i;
    futures.add(cpuPool.submit(() -> computeIntensive(task)));
}
```

### I/O-Bound Tasks

```java
// Formula: threads = N * (1 + W/C)
// N = CPU cores, W = wait time, C = compute time
// If W/C = 10 (waiting 10x more than computing): threads = N * 11

int cpuCores = Runtime.getRuntime().availableProcessors();
double waitToComputeRatio = 10.0; // e.g., DB calls, HTTP requests
int ioPoolSize = (int) (cpuCores * (1 + waitToComputeRatio));

ExecutorService ioPool = Executors.newFixedThreadPool(ioPoolSize);

// Example: Parallel API calls
List<Future<Response>> responses = new ArrayList<>();
for (String url : urls) {
    responses.add(ioPool.submit(() -> httpClient.get(url)));
}
```

### Thread Pool Sizing Table

| Workload Type | Formula              | Example (8 cores) | Pool Size |
|---------------|----------------------|--------------------|-----------|
| CPU-bound     | N                    | 8                  | 8         |
| Mixed (W/C=1) | N * 2               | 8 * 2              | 16        |
| I/O-bound     | N * (1 + W/C)        | 8 * (1 + 10)       | 88        |
| Virtual threads| Unlimited (Java 21) | N/A                | unbounded |

```java
// Java 21: Virtual threads for I/O-bound workloads
// No need to size thread pools manually!
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> results = urls.stream()
        .map(url -> executor.submit(() -> fetchUrl(url)))
        .toList();

    for (Future<String> result : results) {
        System.out.println(result.get());
    }
}
```

---

## 4. False Sharing

False sharing occurs when threads modify variables that reside on the same CPU cache line (typically 64 bytes), causing cache invalidation.

```java
// BAD: False sharing - counters share a cache line
public class FalseSharingDemo {
    private volatile long counter1; // offset 0
    private volatile long counter2; // offset 8 (same cache line!)

    // Thread 1 modifying counter1 invalidates Thread 2's cache line
    // even though they access different variables
}

// GOOD: Padding to separate cache lines
public class NoPaddingFalseSharing {
    private volatile long counter1;
    private long p1, p2, p3, p4, p5, p6, p7; // 56 bytes padding
    private volatile long counter2; // now on a different cache line
}
```

---

## 5. @Contended Annotation

Java 8+ provides `@Contended` to automatically add padding to prevent false sharing.

```java
import jdk.internal.vm.annotation.Contended;

// Requires: --add-opens java.base/jdk.internal.vm.annotation=ALL-UNNAMED
// and: -XX:-RestrictContended

public class ContendedExample {

    @Contended  // JVM adds padding around this field
    private volatile long counter1;

    @Contended
    private volatile long counter2;
}

// Grouping related fields (same group shares cache line)
public class ContendedGroups {

    @Contended("group1")
    private volatile long readCounter;

    @Contended("group1")
    private volatile long readTimestamp;  // same cache line as readCounter

    @Contended("group2")
    private volatile long writeCounter;  // different cache line
}
```

### False Sharing Benchmark Results

```
Without padding (false sharing):    ~4500 ms  (cache thrashing)
With manual padding:                ~800 ms
With @Contended:                    ~800 ms
Single-threaded baseline:           ~400 ms
```

---

## 6. Async Processing

```java
// CompletableFuture for non-blocking async operations
public class AsyncService {
    private final ExecutorService executor =
        Executors.newFixedThreadPool(
            Runtime.getRuntime().availableProcessors());

    public CompletableFuture<OrderResponse> processOrder(Order order) {
        return CompletableFuture
            .supplyAsync(() -> validateOrder(order), executor)
            .thenApplyAsync(valid -> calculatePrice(valid), executor)
            .thenApplyAsync(priced -> applyDiscount(priced), executor)
            .thenComposeAsync(discounted -> saveToDatabase(discounted), executor)
            .thenApplyAsync(saved -> buildResponse(saved), executor)
            .exceptionally(ex -> buildErrorResponse(ex));
    }

    // Fan-out: parallel independent operations
    public CompletableFuture<UserProfile> getUserProfile(String userId) {
        CompletableFuture<User> userFuture =
            CompletableFuture.supplyAsync(() -> userService.getUser(userId));
        CompletableFuture<List<Order>> ordersFuture =
            CompletableFuture.supplyAsync(() -> orderService.getOrders(userId));
        CompletableFuture<Preferences> prefsFuture =
            CompletableFuture.supplyAsync(() -> prefService.getPrefs(userId));

        // All three run in parallel, combine when all complete
        return CompletableFuture.allOf(userFuture, ordersFuture, prefsFuture)
            .thenApply(v -> new UserProfile(
                userFuture.join(),
                ordersFuture.join(),
                prefsFuture.join()
            ));
    }
}
```

---

## 7. Concurrent Data Structure Selection

```java
// Choose the right concurrent collection
// Read-heavy, rare writes
CopyOnWriteArrayList<String> listeners = new CopyOnWriteArrayList<>();

// High-throughput map operations
ConcurrentHashMap<String, Data> cache = new ConcurrentHashMap<>();

// Producer-consumer pattern
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);

// Work-stealing for parallel tasks
ForkJoinPool forkJoinPool = new ForkJoinPool(
    Runtime.getRuntime().availableProcessors());

// Non-blocking queue (lock-free)
ConcurrentLinkedQueue<Event> events = new ConcurrentLinkedQueue<>();
```

| Collection                 | Lock Type     | Best For                        |
|---------------------------|---------------|---------------------------------|
| `ConcurrentHashMap`       | Lock striping | General concurrent map          |
| `CopyOnWriteArrayList`    | Copy on write | Read-heavy, rare modifications  |
| `LinkedBlockingQueue`     | Two-lock      | Bounded producer-consumer       |
| `ConcurrentLinkedQueue`   | Lock-free     | Unbounded non-blocking queue    |
| `ConcurrentSkipListMap`   | Lock-free     | Sorted concurrent map           |

---

## 8. Reducing Context Switch Overhead

```java
// Reduce thread count to minimize context switches
// Use work-stealing pool for uneven workloads
ForkJoinPool pool = new ForkJoinPool(
    Runtime.getRuntime().availableProcessors(),
    ForkJoinPool.defaultForkJoinWorkerThreadFactory,
    null,
    true  // asyncMode for event-style tasks
);

// Batch processing to amortize synchronization cost
public class BatchProcessor {
    private final BlockingQueue<Event> queue = new LinkedBlockingQueue<>();
    private final int BATCH_SIZE = 100;

    public void processLoop() {
        List<Event> batch = new ArrayList<>(BATCH_SIZE);
        while (!Thread.currentThread().isInterrupted()) {
            batch.clear();
            queue.drainTo(batch, BATCH_SIZE); // single lock acquisition
            if (batch.isEmpty()) {
                try {
                    batch.add(queue.take()); // block if empty
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
            processBatch(batch); // process all at once
        }
    }
}
```

---

## Interview Tips

- `LongAdder` outperforms `AtomicLong` under high contention by orders of magnitude
- Thread pool size: CPU-bound = N cores, I/O-bound = N * (1 + W/C)
- Virtual threads (Java 21) eliminate manual pool sizing for I/O-bound work
- False sharing is a hidden performance killer; use `@Contended` or manual padding
- Prefer `ConcurrentHashMap` over `Collections.synchronizedMap()`
- `CompletableFuture` enables non-blocking async pipelines
- Always measure contention with profiling tools before optimizing locks

---

[<< Previous: Memory Management](./q03-memory-management-perf.md) | [Next: Collection Performance >>](./q05-collection-performance.md)

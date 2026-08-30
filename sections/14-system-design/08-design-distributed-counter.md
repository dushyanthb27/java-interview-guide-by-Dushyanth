# Q8: Design a Distributed Counter

## Question
Design and implement a Distributed Counter in Java. Compare `AtomicLong`, `LongAdder`, and CAS-based approaches. Implement sharded counters for high contention and discuss eventual consistency.

## Difficulty: Hard

---

## Solution

### Requirements

**Functional Requirements:**
- Increment and decrement a counter
- Get the current count
- Support multiple named counters
- Support batch increment (add N)
- Reset counter

**Non-Functional Requirements:**
- Thread-safe under extreme contention (thousands of concurrent writers)
- Minimize cache-line contention across CPU cores
- Support eventual consistency for distributed reads
- Scalable to millions of increments per second

### API Design

```java
public interface DistributedCounter {
    void increment(String counterName);
    void decrement(String counterName);
    void add(String counterName, long delta);
    long get(String counterName);
    void reset(String counterName);
}
```

### Implementation 1: AtomicLong (Baseline)

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

public class AtomicLongCounter implements DistributedCounter {

    private final ConcurrentHashMap<String, AtomicLong> counters = new ConcurrentHashMap<>();

    private AtomicLong getOrCreate(String name) {
        return counters.computeIfAbsent(name, k -> new AtomicLong(0));
    }

    @Override
    public void increment(String name) {
        getOrCreate(name).incrementAndGet();
    }

    @Override
    public void decrement(String name) {
        getOrCreate(name).decrementAndGet();
    }

    @Override
    public void add(String name, long delta) {
        getOrCreate(name).addAndGet(delta);
    }

    @Override
    public long get(String name) {
        AtomicLong counter = counters.get(name);
        return counter != null ? counter.get() : 0;
    }

    @Override
    public void reset(String name) {
        AtomicLong counter = counters.get(name);
        if (counter != null) counter.set(0);
    }
}
```

**Problem:** Under high contention, `AtomicLong.incrementAndGet()` uses CAS (compare-and-swap). When many threads compete on the same memory location, CAS retries spike — causing CPU waste and degraded throughput.

### Implementation 2: LongAdder (JDK 8+)

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.LongAdder;

public class LongAdderCounter implements DistributedCounter {

    private final ConcurrentHashMap<String, LongAdder> counters = new ConcurrentHashMap<>();

    private LongAdder getOrCreate(String name) {
        return counters.computeIfAbsent(name, k -> new LongAdder());
    }

    @Override
    public void increment(String name) {
        getOrCreate(name).increment();
    }

    @Override
    public void decrement(String name) {
        getOrCreate(name).decrement();
    }

    @Override
    public void add(String name, long delta) {
        getOrCreate(name).add(delta);
    }

    @Override
    public long get(String name) {
        LongAdder adder = counters.get(name);
        return adder != null ? adder.sum() : 0;
    }

    @Override
    public void reset(String name) {
        LongAdder adder = counters.get(name);
        if (adder != null) adder.reset();
    }
}
```

**How `LongAdder` works internally:**
- Maintains a `base` value + array of `Cell` objects (one per contended thread)
- Each thread increments its own `Cell` — avoiding CAS contention
- `sum()` aggregates `base + all cells` — eventual consistency
- 5-10x faster than `AtomicLong` under high contention

### Implementation 3: Custom Sharded Counter (Manual LongAdder)

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

public class ShardedCounter implements DistributedCounter {

    private final int shardCount;
    private final ConcurrentHashMap<String, AtomicLong[]> counters = new ConcurrentHashMap<>();

    public ShardedCounter(int shardCount) {
        // Use power of 2 for fast modulo
        this.shardCount = Integer.highestOneBit(shardCount - 1) << 1;
    }

    public ShardedCounter() {
        this(Runtime.getRuntime().availableProcessors() * 4);
    }

    private AtomicLong[] getOrCreate(String name) {
        return counters.computeIfAbsent(name, k -> {
            AtomicLong[] shards = new AtomicLong[shardCount];
            for (int i = 0; i < shardCount; i++) {
                shards[i] = new AtomicLong(0);
            }
            return shards;
        });
    }

    private int shardIndex() {
        // Use thread ID to spread across shards — reduces contention
        long threadId = Thread.currentThread().getId();
        return (int) (threadId & (shardCount - 1)); // fast modulo for power of 2
    }

    @Override
    public void increment(String name) {
        getOrCreate(name)[shardIndex()].incrementAndGet();
    }

    @Override
    public void decrement(String name) {
        getOrCreate(name)[shardIndex()].decrementAndGet();
    }

    @Override
    public void add(String name, long delta) {
        getOrCreate(name)[shardIndex()].addAndGet(delta);
    }

    @Override
    public long get(String name) {
        AtomicLong[] shards = counters.get(name);
        if (shards == null) return 0;

        long sum = 0;
        for (AtomicLong shard : shards) {
            sum += shard.get();
        }
        return sum;
    }

    @Override
    public void reset(String name) {
        AtomicLong[] shards = counters.get(name);
        if (shards != null) {
            for (AtomicLong shard : shards) {
                shard.set(0);
            }
        }
    }
}
```

### Implementation 4: CAS-Based Counter with Backoff

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

public class CASWithBackoffCounter implements DistributedCounter {

    private final ConcurrentHashMap<String, AtomicLong> counters = new ConcurrentHashMap<>();

    private AtomicLong getOrCreate(String name) {
        return counters.computeIfAbsent(name, k -> new AtomicLong(0));
    }

    @Override
    public void increment(String name) {
        add(name, 1);
    }

    @Override
    public void decrement(String name) {
        add(name, -1);
    }

    @Override
    public void add(String name, long delta) {
        AtomicLong counter = getOrCreate(name);
        int backoff = 1;

        while (true) {
            long current = counter.get();
            long next = current + delta;
            if (counter.compareAndSet(current, next)) {
                return; // success
            }

            // Exponential backoff on CAS failure
            if (backoff < 1024) {
                for (int i = 0; i < backoff; i++) {
                    Thread.onSpinWait(); // JDK 9+ CPU hint
                }
                backoff <<= 1;
            } else {
                Thread.yield(); // yield to other threads after many retries
            }
        }
    }

    @Override
    public long get(String name) {
        AtomicLong counter = counters.get(name);
        return counter != null ? counter.get() : 0;
    }

    @Override
    public void reset(String name) {
        AtomicLong counter = counters.get(name);
        if (counter != null) counter.set(0);
    }
}
```

### Approach Comparison

| Approach | Write Throughput | Read Accuracy | Memory | Best For |
|----------|-----------------|---------------|--------|----------|
| `AtomicLong` | Low under contention | Exact | O(1)/counter | Low-moderate contention |
| `LongAdder` | Very high | Eventual (sum) | O(cores)/counter | High write contention |
| Sharded Counter | Very high | Eventual (sum) | O(shards)/counter | Custom shard control |
| CAS + Backoff | Medium-high | Exact | O(1)/counter | Moderate contention |

### Cache Line Contention Explained

```
┌──────────────────────────────────────────────────────┐
│  AtomicLong under contention:                        │
│                                                      │
│  Core 0 ──CAS──┐                                    │
│  Core 1 ──CAS──┤──► [Single cache line] ◄── bounce  │
│  Core 2 ──CAS──┤    between L1 caches               │
│  Core 3 ──CAS──┘                                    │
│                                                      │
│  LongAdder / Sharded Counter:                        │
│                                                      │
│  Core 0 ──► [Cell 0]  (own cache line)               │
│  Core 1 ──► [Cell 1]  (own cache line)               │
│  Core 2 ──► [Cell 2]  (own cache line)               │
│  Core 3 ──► [Cell 3]  (own cache line)               │
│                                                      │
│  sum() = Cell[0] + Cell[1] + Cell[2] + Cell[3]      │
└──────────────────────────────────────────────────────┘
```

### Concurrency Considerations

| Concern | Solution |
|---------|----------|
| Write-write contention | Shard writes across multiple cells / cache lines |
| CAS retry storms | Exponential backoff + `Thread.onSpinWait()` |
| Read consistency | `get()` on sharded counters is eventually consistent |
| Memory padding | JDK `@Contended` annotation prevents false sharing |
| Counter registry | `ConcurrentHashMap.computeIfAbsent` for lazy creation |

### Performance Analysis

| Operation | AtomicLong | LongAdder | Sharded |
|-----------|-----------|-----------|---------|
| `increment()` | O(1) + CAS retries | O(1) amortized | O(1) |
| `get()` | O(1) exact | O(cells) sum | O(shards) sum |
| Memory/counter | 8 bytes | ~8 * cores bytes | ~8 * shards bytes |
| Contention @ 16 threads | ~400M ops/s | ~2B ops/s | ~1.8B ops/s |

### Testing Strategy

```java
public class DistributedCounterTest {

    private void testImplementation(DistributedCounter counter) throws Exception {
        int threadCount = 16;
        int incrementsPerThread = 100_000;
        ExecutorService exec = Executors.newFixedThreadPool(threadCount);
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch doneLatch = new CountDownLatch(threadCount);

        for (int t = 0; t < threadCount; t++) {
            exec.submit(() -> {
                try {
                    startLatch.await(); // all threads start at once
                    for (int i = 0; i < incrementsPerThread; i++) {
                        counter.increment("hits");
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    doneLatch.countDown();
                }
            });
        }

        startLatch.countDown();
        doneLatch.await();
        exec.shutdown();

        long expected = (long) threadCount * incrementsPerThread;
        assertEquals(expected, counter.get("hits"));
    }

    @Test
    void testAtomicLong() throws Exception {
        testImplementation(new AtomicLongCounter());
    }

    @Test
    void testLongAdder() throws Exception {
        testImplementation(new LongAdderCounter());
    }

    @Test
    void testShardedCounter() throws Exception {
        testImplementation(new ShardedCounter());
    }

    @Test
    void testCASBackoff() throws Exception {
        testImplementation(new CASWithBackoffCounter());
    }

    @Test
    void testMultipleCounters() {
        ShardedCounter counter = new ShardedCounter();
        counter.add("page-views", 100);
        counter.add("api-calls", 50);
        counter.increment("errors");

        assertEquals(100, counter.get("page-views"));
        assertEquals(50, counter.get("api-calls"));
        assertEquals(1, counter.get("errors"));
    }

    @Test
    void benchmarkComparison() throws Exception {
        int threads = Runtime.getRuntime().availableProcessors();
        int ops = 1_000_000;

        long atomicTime = benchmark(new AtomicLongCounter(), threads, ops);
        long adderTime = benchmark(new LongAdderCounter(), threads, ops);
        long shardedTime = benchmark(new ShardedCounter(), threads, ops);

        System.out.println("AtomicLong: " + atomicTime + "ms");
        System.out.println("LongAdder:  " + adderTime + "ms");
        System.out.println("Sharded:    " + shardedTime + "ms");

        // LongAdder and Sharded should be faster under contention
        assertTrue(adderTime <= atomicTime * 2); // rough sanity check
    }

    private long benchmark(DistributedCounter counter, int threads, int ops) throws Exception {
        ExecutorService exec = Executors.newFixedThreadPool(threads);
        long start = System.currentTimeMillis();

        CountDownLatch latch = new CountDownLatch(threads);
        for (int t = 0; t < threads; t++) {
            exec.submit(() -> {
                for (int i = 0; i < ops; i++) counter.increment("bench");
                latch.countDown();
            });
        }
        latch.await();
        exec.shutdown();
        return System.currentTimeMillis() - start;
    }
}
```

---

## Interviewer Insight
This question tests deep understanding of CPU-level concurrency: cache-line bouncing, CAS retry storms, and the `LongAdder` optimization strategy. Interviewers want to see that you know *why* `AtomicLong` is slow under contention and *how* `LongAdder` solves it by striping across cells. For distributed systems, discuss how this pattern maps to sharded counters in databases (like Cassandra counters or DynamoDB atomic counters).

## Follow-up Questions
1. How does `@Contended` annotation prevent false sharing in `LongAdder`'s `Cell` class?
2. How would you implement a distributed counter across multiple JVMs using Redis or Cassandra?
3. What is the trade-off between strong consistency and high availability in distributed counters (CAP theorem)?

## Related Topics
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md)
- [JVM Internals](../07-jvm-internals/README.md)
- [Performance Optimization](../12-performance-optimization/README.md)

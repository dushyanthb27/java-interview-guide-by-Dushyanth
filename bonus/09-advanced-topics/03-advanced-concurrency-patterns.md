# Advanced Concurrency Patterns Deep-Dive

## 📋 Question
Explain advanced concurrency patterns including StampedLock, LongAdder, custom synchronization, and the differences between various lock implementations. When should you use each? Provide performance comparisons and real-world use cases.

## Difficulty: 🔴 Expert

---

## 💡 Answer

Beyond basic `synchronized` and `ReentrantLock`, Java provides specialized concurrency utilities for high-performance scenarios. Understanding these advanced patterns is crucial for building scalable, low-latency systems.

### StampedLock: Optimistic Locking

`StampedLock` (Java 8) provides a lock with three modes: **write**, **read**, and **optimistic read**. It's designed for scenarios where reads vastly outnumber writes.

```java
import java.util.concurrent.locks.*;

public class StampedLockDemo {
    private final StampedLock lock = new StampedLock();
    private double x, y;
    
    // ✅ Write lock — exclusive
    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
    
    // ✅ Optimistic read — non-blocking
    public double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();
        double currentX = x, currentY = y;
        
        if (!lock.validate(stamp)) {
            // Optimistic read failed — acquire read lock
            stamp = lock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
    
    // ✅ Upgrade from optimistic to read lock
    public void moveIfAtOrigin(double deltaX, double deltaY) {
        long stamp = lock.tryOptimisticRead();
        try {
            while (x == 0.0 && y == 0.0) {
                long ws = lock.tryConvertToWriteLock(stamp);
                if (ws != 0L) {
                    stamp = ws;
                    x += deltaX;
                    y += deltaY;
                    return;
                } else {
                    lock.unlockRead(stamp);
                    stamp = lock.writeLock();
                }
            }
        } finally {
            lock.unlock(stamp);
        }
    }
}
```

**When to use StampedLock:**
- Read-heavy workloads (reads >> writes)
- When you can tolerate occasional retries
- When you need to avoid read lock contention

**When NOT to use StampedLock:**
- When you need reentrant locking (it's not reentrant)
- When you need condition variables (use ReentrantLock instead)
- When write contention is high

### LongAdder: High-Performance Counter

`LongAdder` (Java 8) is a high-performance alternative to `AtomicLong` for scenarios with high contention. It maintains a set of variables to reduce contention, summing them only when needed.

```java
import java.util.concurrent.atomic.*;

public class LongAdderDemo {
    // ❌ AtomicLong — single variable, high contention
    private final AtomicLong atomicCounter = new AtomicLong();
    
    // ✅ LongAdder — multiple variables, low contention
    private final LongAdder adderCounter = new LongAdder();
    
    public void incrementAtomic() {
        atomicCounter.incrementAndGet();  // CAS loop under contention
    }
    
    public void incrementAdder() {
        adderCounter.increment();  // Thread-local increment, no CAS
    }
    
    public long getAtomicCount() {
        return atomicCounter.get();
    }
    
    public long getAdderCount() {
        return adderCounter.sum();  // Sums all cells (slightly expensive)
    }
}
```

**Performance comparison:**

```java
public class CounterBenchmark {
    private static final int THREADS = 32;
    private static final int ITERATIONS = 10_000_000;
    
    public static void main(String[] args) throws InterruptedException {
        benchmarkAtomicLong();
        benchmarkLongAdder();
    }
    
    private static void benchmarkAtomicLong() throws InterruptedException {
        AtomicLong counter = new AtomicLong();
        ExecutorService executor = Executors.newFixedThreadPool(THREADS);
        
        long start = System.nanoTime();
        for (int i = 0; i < THREADS; i++) {
            executor.submit(() -> {
                for (int j = 0; j < ITERATIONS; j++) {
                    counter.incrementAndGet();
                }
            });
        }
        executor.shutdown();
        executor.awaitTermination(1, TimeUnit.MINUTES);
        
        long duration = System.nanoTime() - start;
        System.out.println("AtomicLong: " + duration / 1_000_000 + "ms, count=" + counter.get());
    }
    
    private static void benchmarkLongAdder() throws InterruptedException {
        LongAdder counter = new LongAdder();
        ExecutorService executor = Executors.newFixedThreadPool(THREADS);
        
        long start = System.nanoTime();
        for (int i = 0; i < THREADS; i++) {
            executor.submit(() -> {
                for (int j = 0; j < ITERATIONS; j++) {
                    counter.increment();
                }
            });
        }
        executor.shutdown();
        executor.awaitTermination(1, TimeUnit.MINUTES);
        
        long duration = System.nanoTime() - start;
        System.out.println("LongAdder: " + duration / 1_000_000 + "ms, count=" + counter.sum());
    }
}

// Typical results (32 threads, 10M iterations each):
// AtomicLong: ~8000ms
// LongAdder:  ~2000ms (4x faster!)
```

**When to use LongAdder:**
- High-frequency statistics (counters, histograms)
- When you only need the sum occasionally
- When write contention is very high

**When to use AtomicLong:**
- When you need exact current value frequently
- When you need compare-and-swap operations
- When contention is low

### Custom Synchronization: AbstractQueuedSynchronizer

For building custom synchronization constructs, `AbstractQueuedSynchronizer` (AQS) provides the foundation for locks, semaphores, and other synchronizers.

```java
import java.util.concurrent.locks.*;

// Custom lock with timeout and interrupt support
public class CustomLock implements Lock {
    private final Sync sync = new Sync();
    
    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int acquires) {
            assert acquires == 1;
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        
        @Override
        protected boolean tryRelease(int releases) {
            assert releases == 1;
            if (getState() == 0) {
                throw new IllegalMonitorStateException();
            }
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        
        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }
        
        Condition newCondition() {
            return new ConditionObject();
        }
    }
    
    @Override
    public void lock() {
        sync.acquire(1);
    }
    
    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
    
    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }
    
    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }
    
    @Override
    public void unlock() {
        sync.release(1);
    }
    
    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}
```

**Custom semaphore with AQS:**

```java
public class WeightedSemaphore {
    private final Sync sync;
    
    public WeightedSemaphore(int permits) {
        this.sync = new Sync(permits);
    }
    
    public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }
    
    public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }
    
    private static class Sync extends AbstractQueuedSynchronizer {
        Sync(int permits) {
            setState(permits);
        }
        
        @Override
        protected int tryAcquireShared(int acquires) {
            while (true) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 || compareAndSetState(available, remaining)) {
                    return remaining;
                }
            }
        }
        
        @Override
        protected boolean tryReleaseShared(int releases) {
            while (true) {
                int current = getState();
                int newCount = current + releases;
                if (newCount < current) {  // Overflow
                    throw new Error("Maximum permit count exceeded");
                }
                if (compareAndSetState(current, newCount)) {
                    return true;
                }
            }
        }
    }
}
```

### Lock Comparison Matrix

| Lock Type | Reentrant | Fair | Condition | Performance | Best For |
|-----------|-----------|------|-----------|-------------|----------|
| `synchronized` | Yes | No (biased) | Yes | Good (biased) | Simple cases |
| `ReentrantLock` | Yes | Optional | Yes | Good | Complex locking |
| `StampedLock` | No | No | No | Excellent (read-heavy) | Read-mostly data |
| `ReadWriteLock` | Yes | Optional | Yes | Good | Read-heavy workloads |
| `LongAdder` | N/A | N/A | N/A | Excellent (counters) | High-contention counters |

### Advanced Pattern: Striped Locks

Striped locks reduce contention by using multiple locks to protect different segments of data.

```java
import com.google.common.util.concurrent.Striped;
import java.util.concurrent.locks.Lock;

public class StripedLockDemo {
    // Guava Striped — automatically hashes keys to locks
    private final Striped<Lock> locks = Striped.lock(16);  // 16 stripes
    private final Map<String, String> data = new ConcurrentHashMap<>();
    
    public void put(String key, String value) {
        Lock lock = locks.get(key);
        lock.lock();
        try {
            data.put(key, value);
        } finally {
            lock.unlock();
        }
    }
    
    public String get(String key) {
        Lock lock = locks.get(key);
        lock.lock();
        try {
            return data.get(key);
        } finally {
            lock.unlock();
        }
    }
    
    // Different keys can be accessed concurrently if they hash to different locks
}
```

### Advanced Pattern: Lock Striping with LongAdder

```java
public class ConcurrentCounterMap<K> {
    private final ConcurrentHashMap<K, LongAdder> counters = new ConcurrentHashMap<>();
    
    public void increment(K key) {
        counters.computeIfAbsent(key, k -> new LongAdder()).increment();
    }
    
    public long get(K key) {
        LongAdder adder = counters.get(key);
        return adder == null ? 0 : adder.sum();
    }
    
    public Map<K, Long> getAll() {
        Map<K, Long> result = new HashMap<>();
        counters.forEach((key, adder) -> result.put(key, adder.sum()));
        return result;
    }
}
```

### Code Example: Complete Advanced Concurrency Demo

```java
package com.interview.advanced.concurrency;

import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.concurrent.locks.*;
import java.util.*;

// === STAMPEDLOCK: OPTIMISTIC LOCKING ===
class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();
    
    // Mutative operation — exclusive lock
    public void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }
    
    // Read-only — optimistic read
    public double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead();
        double currentX = x, currentY = y;
        if (!sl.validate(stamp)) {
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
    
    // Read with possible upgrade to write
    public void moveIfAtOrigin(double newX, double newY) {
        long stamp = sl.readLock();
        try {
            while (x == 0.0 && y == 0.0) {
                long ws = sl.tryConvertToWriteLock(stamp);
                if (ws != 0L) {
                    stamp = ws;
                    x = newX;
                    y = newY;
                    return;
                } else {
                    sl.unlockRead(stamp);
                    stamp = sl.writeLock();
                }
            }
        } finally {
            sl.unlock(stamp);
        }
    }
}

// === LONGADDER: HIGH-PERFORMANCE COUNTER ===
class StatisticsCollector {
    // ✅ LongAdder for high-frequency metrics
    private final LongAdder requestCount = new LongAdder();
    private final LongAdder errorCount = new LongAdder();
    private final LongAdder totalLatency = new LongAdder();
    
    public void recordRequest(long latencyMs, boolean isError) {
        requestCount.increment();
        totalLatency.add(latencyMs);
        if (isError) {
            errorCount.increment();
        }
    }
    
    public StatisticsSnapshot getSnapshot() {
        return new StatisticsSnapshot(
            requestCount.sum(),
            errorCount.sum(),
            totalLatency.sum()
        );
    }
    
    public static class StatisticsSnapshot {
        public final long requestCount;
        public final long errorCount;
        public final long totalLatency;
        
        public StatisticsSnapshot(long requestCount, long errorCount, long totalLatency) {
            this.requestCount = requestCount;
            this.errorCount = errorCount;
            this.totalLatency = totalLatency;
        }
        
        public double getErrorRate() {
            return requestCount == 0 ? 0 : (double) errorCount / requestCount;
        }
        
        public double getAverageLatency() {
            return requestCount == 0 ? 0 : (double) totalLatency / requestCount;
        }
    }
}

// === CUSTOM BOUNDED SEMAPHORE ===
class BoundedSemaphore {
    private final Semaphore semaphore;
    private final int maxPermits;
    
    public BoundedSemaphore(int permits) {
        this.maxPermits = permits;
        this.semaphore = new Semaphore(permits);
    }
    
    public void acquire() throws InterruptedException {
        semaphore.acquire();
    }
    
    public void release() {
        if (semaphore.availablePermits() >= maxPermits) {
            throw new IllegalStateException("Cannot release beyond max permits");
        }
        semaphore.release();
    }
    
    public int availablePermits() {
        return semaphore.availablePermits();
    }
}

// === PHASER: ADVANCED BARRIER ===
class PhaserDemo {
    public static void main(String[] args) throws InterruptedException {
        int parties = 3;
        Phaser phaser = new Phaser(parties);
        
        for (int i = 0; i < parties; i++) {
            new Thread(new Worker(phaser, i)).start();
        }
    }
    
    static class Worker implements Runnable {
        private final Phaser phaser;
        private final int id;
        
        Worker(Phaser phaser, int id) {
            this.phaser = phaser;
            this.id = id;
        }
        
        @Override
        public void run() {
            System.out.println("Worker " + id + " starting phase 0");
            phaser.arriveAndAwaitAdvance();  // Wait for all
            
            System.out.println("Worker " + id + " starting phase 1");
            phaser.arriveAndAwaitAdvance();  // Wait for all
            
            System.out.println("Worker " + id + " done");
            phaser.arriveAndDeregister();   // Remove from phaser
        }
    }
}

// === LOCK STRIPING ===
class StripedCache<K, V> {
    private static final int STRIPES = 16;
    private final Lock[] locks = new Lock[STRIPES];
    private final Map<K, V>[] caches;
    
    @SuppressWarnings("unchecked")
    public StripedCache() {
        for (int i = 0; i < STRIPES; i++) {
            locks[i] = new ReentrantLock();
        }
        caches = new Map[STRIPES];
        for (int i = 0; i < STRIPES; i++) {
            caches[i] = new HashMap<>();
        }
    }
    
    private int getStripe(K key) {
        return Math.abs(key.hashCode()) % STRIPES;
    }
    
    public V get(K key) {
        int stripe = getStripe(key);
        locks[stripe].lock();
        try {
            return caches[stripe].get(key);
        } finally {
            locks[stripe].unlock();
        }
    }
    
    public void put(K key, V value) {
        int stripe = getStripe(key);
        locks[stripe].lock();
        try {
            caches[stripe].put(key, value);
        } finally {
            locks[stripe].unlock();
        }
    }
}

// === BENCHMARK ===
class ConcurrencyBenchmark {
    private static final int THREADS = 32;
    private static final int OPERATIONS = 1_000_000;
    
    public static void main(String[] args) throws InterruptedException {
        benchmarkAtomicLong();
        benchmarkLongAdder();
        benchmarkStampedLock();
    }
    
    private static void benchmarkAtomicLong() throws InterruptedException {
        AtomicLong counter = new AtomicLong();
        ExecutorService executor = Executors.newFixedThreadPool(THREADS);
        
        long start = System.nanoTime();
        for (int i = 0; i < THREADS; i++) {
            executor.submit(() -> {
                for (int j = 0; j < OPERATIONS; j++) {
                    counter.incrementAndGet();
                }
            });
        }
        executor.shutdown();
        executor.awaitTermination(1, TimeUnit.MINUTES);
        
        System.out.printf("AtomicLong: %dms, count=%d%n",
            (System.nanoTime() - start) / 1_000_000, counter.get());
    }
    
    private static void benchmarkLongAdder() throws InterruptedException {
        LongAdder counter = new LongAdder();
        ExecutorService executor = Executors.newFixedThreadPool(THREADS);
        
        long start = System.nanoTime();
        for (int i = 0; i < THREADS; i++) {
            executor.submit(() -> {
                for (int j = 0; j < OPERATIONS; j++) {
                    counter.increment();
                }
            });
        }
        executor.shutdown();
        executor.awaitTermination(1, TimeUnit.MINUTES);
        
        System.out.printf("LongAdder: %dms, count=%d%n",
            (System.nanoTime() - start) / 1_000_000, counter.sum());
    }
    
    private static void benchmarkStampedLock() throws InterruptedException {
        Point point = new Point();
        ExecutorService executor = Executors.newFixedThreadPool(THREADS);
        
        long start = System.nanoTime();
        List<Future<?>> futures = new ArrayList<>();
        
        // 90% reads, 10% writes
        for (int i = 0; i < THREADS; i++) {
            if (i < THREADS * 0.9) {
                futures.add(executor.submit(() -> {
                    for (int j = 0; j < OPERATIONS; j++) {
                        point.distanceFromOrigin();
                    }
                }));
            } else {
                futures.add(executor.submit(() -> {
                    for (int j = 0; j < OPERATIONS; j++) {
                        point.move(1.0, 1.0);
                    }
                }));
            }
        }
        
        for (Future<?> f : futures) f.get();
        executor.shutdown();
        
        System.out.printf("StampedLock: %dms%n",
            (System.nanoTime() - start) / 1_000_000);
    }
}
```

### ⚠️ Common Mistakes

**1. Using StampedLock reentrantly**

```java
// ❌ StampedLock is NOT reentrant
private final StampedLock lock = new StampedLock();

public void methodA() {
    long stamp = lock.writeLock();
    try {
        methodB();  // Deadlock! StampedLock not reentrant
    } finally {
        lock.unlockWrite(stamp);
    }
}

public void methodB() {
    long stamp = lock.writeLock();  // Will block forever
    try {
        // ...
    } finally {
        lock.unlockWrite(stamp);
    }
}

// ✅ Use ReentrantLock for reentrant scenarios
```

**2. Forgetting to validate optimistic reads**

```java
// ❌ Not validating — race condition
public double badRead() {
    long stamp = lock.tryOptimisticRead();
    double value = x;  // Might be stale!
    return value;
}

// ✅ Always validate
public double goodRead() {
    long stamp = lock.tryOptimisticRead();
    double value = x;
    if (!lock.validate(stamp)) {
        stamp = lock.readLock();
        try {
            value = x;
        } finally {
            lock.unlockRead(stamp);
        }
    }
    return value;
}
```

**3. Using LongAdder when you need exact values**

```java
// ❌ LongAdder.sum() is expensive and not atomic
LongAdder counter = new LongAdder();
counter.increment();
counter.increment();
long value1 = counter.sum();  // Sums all cells
long value2 = counter.sum();  // Sums again — might be different if concurrent

// ✅ Use AtomicLong for exact, frequently-read values
```

### ⚡ Performance Tip

**StampedLock** can be 10-100x faster than `ReentrantLock` for read-heavy workloads due to optimistic reads. However, it's not reentrant and doesn't support condition variables. Profile your application to determine if the complexity is worth the performance gain.

**LongAdder** scales linearly with CPU cores for increment operations, but `sum()` is O(n) where n is the number of cells. Use it when you rarely need the total value.

### 🔧 JVM Internals

`LongAdder` uses **striping** similar to `ConcurrentHashMap`. It maintains an array of `Cell` objects, each with its own `volatile` value. When incrementing, a thread picks a cell based on its CPU core (via `ThreadLocalRandom.getProbe()`), reducing contention. The `sum()` operation traverses all cells and adds their values.

`StampedLock` uses **optimistic locking** by maintaining a version stamp. The `validate(stamp)` method checks if the stamp is still valid (no write occurred). This avoids acquiring locks for reads that don't conflict with writes.

---

## 🎯 Interviewer Insight

Advanced concurrency questions test your ability to choose the right tool for the job. Interviewers want to see that you understand:
- When to use `StampedLock` vs `ReentrantLock` (read-heavy vs general purpose)
- Why `LongAdder` outperforms `AtomicLong` under high contention
- How to build custom synchronizers with AQS
- The trade-offs between different lock implementations

Strong candidates can explain the internal mechanics of these classes and provide real-world examples from production systems.

## ⚡ Follow-up Questions

1. How does `LongAdder` achieve better performance than `AtomicLong` under high contention?
2. What are the dangers of using `StampedLock` incorrectly?
3. How would you implement a custom lock with timeout support using AQS?

## 📚 Related Topics

- [Thread Safety Patterns](../../sections/04-multithreading-concurrency/q20-thread-safety-patterns.md)
- [Reentrant Lock](../../sections/04-multithreading-concurrency/q10-reentrant-lock.md)
- [Virtual Threads](../../sections/04-multithreading-concurrency/q23-virtual-threads.md)

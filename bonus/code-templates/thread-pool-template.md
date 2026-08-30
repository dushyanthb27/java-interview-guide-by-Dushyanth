# Thread Pool Template

## Overview
A custom thread pool implementation that manages a fixed number of worker threads executing tasks from a queue. This is fundamental to understanding concurrent programming and is a common interview question.

## Use Cases
- Managing a fixed set of worker threads
- Preventing thread explosion in high-concurrency scenarios
- Decoupling task submission from execution
- Implementing backpressure with bounded queues
- Building async task processing systems

## Implementation

### Simple Version
Basic thread pool with fixed workers and unbounded queue.

### Production Version
Thread pool with bounded queue, rejection policies, and lifecycle management.

### Thread-Safe Version
Fully concurrent implementation with proper synchronization and shutdown handling.

## Code

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.ReentrantLock;

/**
 * SIMPLE VERSION: Basic Thread Pool
 * - Fixed number of worker threads
 * - Unbounded queue
 * - Simple task execution
 */
public class SimpleThreadPool {
    private final BlockingQueue<Runnable> taskQueue;
    private final List<Thread> workers;
    private volatile boolean shutdown = false;

    public SimpleThreadPool(int numThreads) {
        this.taskQueue = new LinkedBlockingQueue<>();
        this.workers = new ArrayList<>();

        // Create and start worker threads
        for (int i = 0; i < numThreads; i++) {
            Thread worker = new Thread(new Worker());
            worker.setName("PoolWorker-" + i);
            worker.start();
            workers.add(worker);
        }
    }

    /**
     * Submit a task to the thread pool
     * Blocks if queue is full (unbounded, so won't happen)
     */
    public void submit(Runnable task) throws InterruptedException {
        if (shutdown) {
            throw new IllegalStateException("Thread pool is shutdown");
        }
        taskQueue.put(task);
    }

    /**
     * Worker thread that continuously processes tasks from the queue
     */
    private class Worker implements Runnable {
        @Override
        public void run() {
            while (!shutdown) {
                try {
                    Runnable task = taskQueue.take(); // Blocks until task available
                    task.run();
                } catch (InterruptedException e) {
                    // Check if shutdown was called
                    if (shutdown) {
                        break;
                    }
                    Thread.currentThread().interrupt();
                }
            }
        }
    }

    /**
     * Shutdown the thread pool
     * Waits for all tasks to complete
     */
    public void shutdown() throws InterruptedException {
        shutdown = true;
        for (Thread worker : workers) {
            worker.join(); // Wait for worker to finish
        }
    }
}

/**
 * PRODUCTION VERSION: Advanced Thread Pool
 * - Bounded queue with rejection policy
 * - Configurable thread count
 * - Proper lifecycle management
 * - Task timeout support
 * - Statistics tracking
 */
public class ProductionThreadPool {
    private final BlockingQueue<Runnable> taskQueue;
    private final List<Thread> workers;
    private final ReentrantLock lock = new ReentrantLock();
    private volatile boolean shutdown = false;
    private final RejectionPolicy rejectionPolicy;
    private final AtomicInteger submittedTasks = new AtomicInteger(0);
    private final AtomicInteger completedTasks = new AtomicInteger(0);

    public enum RejectionPolicy {
        ABORT,           // Throw exception
        DISCARD,         // Silently discard
        CALLER_RUNS,     // Execute in caller thread
        DISCARD_OLDEST   // Remove oldest task and retry
    }

    public ProductionThreadPool(int numThreads, int queueSize, RejectionPolicy policy) {
        this.taskQueue = new LinkedBlockingQueue<>(queueSize);
        this.workers = new ArrayList<>();
        this.rejectionPolicy = policy;

        // Create worker threads
        for (int i = 0; i < numThreads; i++) {
            Thread worker = new Thread(new Worker());
            worker.setName("PoolWorker-" + i);
            worker.setUncaughtExceptionHandler((t, e) -> {
                System.err.println("Worker thread " + t.getName() + " crashed: " + e.getMessage());
                e.printStackTrace();
            });
            worker.start();
            workers.add(worker);
        }
    }

    /**
     * Submit task with rejection policy handling
     */
    public void submit(Runnable task) {
        if (shutdown) {
            throw new IllegalStateException("Thread pool is shutdown");
        }

        submittedTasks.incrementAndGet();

        // Try to add to queue
        if (!taskQueue.offer(task)) {
            handleRejection(task);
        }
    }

    /**
     * Handle rejected tasks based on policy
     */
    private void handleRejection(Runnable task) {
        switch (rejectionPolicy) {
            case ABORT:
                throw new RejectedExecutionException("Task queue is full");

            case DISCARD:
                System.out.println("Task discarded: queue is full");
                break;

            case CALLER_RUNS:
                System.out.println("Executing task in caller thread");
                task.run();
                break;

            case DISCARD_OLDEST:
                taskQueue.poll(); // Remove oldest
                taskQueue.offer(task); // Add new task
                break;
        }
    }

    /**
     * Worker implementation with error handling
     */
    private class Worker implements Runnable {
        @Override
        public void run() {
            while (!shutdown) {
                Runnable task = null;
                try {
                    // Wait for task with timeout
                    task = taskQueue.poll(1, TimeUnit.SECONDS);

                    if (task != null) {
                        try {
                            task.run();
                            completedTasks.incrementAndGet();
                        } catch (Exception e) {
                            System.err.println("Task execution failed: " + e.getMessage());
                            e.printStackTrace();
                        }
                    }
                } catch (InterruptedException e) {
                    if (shutdown) {
                        break;
                    }
                    Thread.currentThread().interrupt();
                }
            }
        }
    }

    /**
     * Graceful shutdown with timeout
     */
    public void shutdown(long timeoutSeconds) throws InterruptedException {
        lock.lock();
        try {
            shutdown = true;
        } finally {
            lock.unlock();
        }

        // Wait for workers to finish
        long endTime = System.currentTimeMillis() + (timeoutSeconds * 1000);
        for (Thread worker : workers) {
            long remainingTime = endTime - System.currentTimeMillis();
            if (remainingTime > 0) {
                worker.join(remainingTime);
            }
        }

        // Force shutdown remaining threads
        for (Thread worker : workers) {
            if (worker.isAlive()) {
                worker.interrupt();
            }
        }
    }

    /**
     * Get pool statistics
     */
    public Map<String, Integer> getStats() {
        Map<String, Integer> stats = new HashMap<>();
        stats.put("submitted", submittedTasks.get());
        stats.put("completed", completedTasks.get());
        stats.put("queued", taskQueue.size());
        stats.put("active_workers", (int) workers.stream().filter(Thread::isAlive).count());
        return stats;
    }
}

/**
 * THREAD-SAFE VERSION: Enterprise-Grade Thread Pool
 * - Implements ExecutorService interface
 * - Proper Future handling
 * - Thread-safe statistics
 * - Comprehensive lifecycle management
 */
public class EnterpriseThreadPool implements ExecutorService {
    private final BlockingQueue<Runnable> taskQueue;
    private final List<WorkerThread> workers;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull = lock.newCondition();
    private volatile boolean shutdown = false;
    private volatile boolean shutdownNow = false;
    private final AtomicInteger activeCount = new AtomicInteger(0);

    public EnterpriseThreadPool(int coreThreads, int maxThreads, int queueSize) {
        this.taskQueue = new LinkedBlockingQueue<>(queueSize);
        this.workers = Collections.synchronizedList(new ArrayList<>());

        // Create core threads
        for (int i = 0; i < coreThreads; i++) {
            addWorker();
        }
    }

    /**
     * Add a new worker thread
     */
    private void addWorker() {
        WorkerThread worker = new WorkerThread();
        worker.start();
        workers.add(worker);
    }

    @Override
    public void execute(Runnable command) {
        if (shutdown) {
            throw new RejectedExecutionException("Executor is shutdown");
        }

        try {
            // Try non-blocking offer first
            if (!taskQueue.offer(command)) {
                // Queue full, try blocking with timeout
                if (!taskQueue.offer(command, 5, TimeUnit.SECONDS)) {
                    throw new RejectedExecutionException("Task queue is full");
                }
            }
            lock.lock();
            try {
                notEmpty.signal();
            } finally {
                lock.unlock();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RejectedExecutionException("Interrupted while submitting task", e);
        }
    }

    @Override
    public void shutdown() {
        lock.lock();
        try {
            shutdown = true;
            notEmpty.signalAll();
        } finally {
            lock.unlock();
        }
    }

    @Override
    public List<Runnable> shutdownNow() {
        lock.lock();
        try {
            shutdownNow = true;
            shutdown = true;
            notEmpty.signalAll();
            // Interrupt all workers
            workers.forEach(Thread::interrupt);
        } finally {
            lock.unlock();
        }
        return new ArrayList<>(taskQueue);
    }

    @Override
    public boolean isShutdown() {
        return shutdown;
    }

    @Override
    public boolean isTerminated() {
        return shutdown && taskQueue.isEmpty() && activeCount.get() == 0;
    }

    @Override
    public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
        long deadline = System.nanoTime() + unit.toNanos(timeout);
        lock.lock();
        try {
            while (!isTerminated()) {
                long remaining = deadline - System.nanoTime();
                if (remaining <= 0) {
                    return false;
                }
                notEmpty.awaitNanos(remaining);
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        FutureTask<T> futureTask = new FutureTask<>(task);
        execute(futureTask);
        return futureTask;
    }

    @Override
    public <T> Future<T> submit(Runnable task, T result) {
        FutureTask<T> futureTask = new FutureTask<>(task, result);
        execute(futureTask);
        return futureTask;
    }

    @Override
    public Future<?> submit(Runnable task) {
        FutureTask<Void> futureTask = new FutureTask<>(task, null);
        execute(futureTask);
        return futureTask;
    }

    @Override
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) {
        throw new UnsupportedOperationException();
    }

    @Override
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, 
                                         long timeout, TimeUnit unit) {
        throw new UnsupportedOperationException();
    }

    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks) {
        throw new UnsupportedOperationException();
    }

    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks, 
                          long timeout, TimeUnit unit) {
        throw new UnsupportedOperationException();
    }

    /**
     * Worker thread implementation
     */
    private class WorkerThread extends Thread {
        @Override
        public void run() {
            while (!shutdownNow) {
                Runnable task = null;
                try {
                    lock.lock();
                    try {
                        // Wait for task
                        while (taskQueue.isEmpty() && !shutdown && !shutdownNow) {
                            notEmpty.await();
                        }
                        if (!shutdown && !shutdownNow) {
                            task = taskQueue.poll();
                        }
                    } finally {
                        lock.unlock();
                    }

                    if (task != null) {
                        activeCount.incrementAndGet();
                        try {
                            task.run();
                        } finally {
                            activeCount.decrementAndGet();
                        }
                    }
                } catch (InterruptedException e) {
                    if (!shutdown) {
                        Thread.currentThread().interrupt();
                    }
                }
            }
        }
    }
}
```

## Testing

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadPoolTest {

    @Test
    public void testSimpleThreadPool() throws InterruptedException {
        SimpleThreadPool pool = new SimpleThreadPool(2);
        AtomicInteger counter = new AtomicInteger(0);

        // Submit 10 tasks
        for (int i = 0; i < 10; i++) {
            pool.submit(() -> {
                counter.incrementAndGet();
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        // Wait for completion
        Thread.sleep(500);
        pool.shutdown();

        assertEquals(10, counter.get());
    }

    @Test
    public void testProductionThreadPoolRejection() {
        ProductionThreadPool pool = new ProductionThreadPool(
            2, 5, ProductionThreadPool.RejectionPolicy.ABORT
        );

        // Fill the queue
        for (int i = 0; i < 7; i++) {
            try {
                pool.submit(() -> {
                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                });
            } catch (RejectedExecutionException e) {
                assertTrue(i >= 7); // Should reject after queue is full
            }
        }
    }

    @Test
    public void testEnterpriseThreadPoolFutures() throws Exception {
        EnterpriseThreadPool pool = new EnterpriseThreadPool(2, 4, 10);

        Future<Integer> future = pool.submit(() -> {
            Thread.sleep(50);
            return 42;
        });

        assertEquals(42, future.get(1, TimeUnit.SECONDS));
        pool.shutdown();
    }

    @Test
    public void testThreadPoolShutdown() throws InterruptedException {
        ProductionThreadPool pool = new ProductionThreadPool(
            2, 10, ProductionThreadPool.RejectionPolicy.DISCARD
        );

        AtomicInteger completed = new AtomicInteger(0);
        for (int i = 0; i < 5; i++) {
            pool.submit(() -> {
                completed.incrementAndGet();
                try {
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        pool.shutdown(5);
        assertTrue(pool.getStats().get("completed") > 0);
    }

    @Test
    public void testConcurrentSubmission() throws InterruptedException {
        ProductionThreadPool pool = new ProductionThreadPool(
            4, 100, ProductionThreadPool.RejectionPolicy.CALLER_RUNS
        );

        AtomicInteger counter = new AtomicInteger(0);
        int taskCount = 100;

        // Submit tasks from multiple threads
        Thread[] submitters = new Thread[4];
        for (int i = 0; i < 4; i++) {
            submitters[i] = new Thread(() -> {
                for (int j = 0; j < 25; j++) {
                    pool.submit(counter::incrementAndGet);
                }
            });
            submitters[i].start();
        }

        for (Thread t : submitters) {
            t.join();
        }

        Thread.sleep(500);
        pool.shutdown(5);

        assertEquals(taskCount, counter.get());
    }
}
```

## Performance Considerations

### Time Complexity
- **submit()**: O(1) average case, O(n) worst case (queue full, rejection policy)
- **task execution**: O(1) per task (depends on task implementation)

### Space Complexity
- **O(n)** where n = queue size + number of worker threads

### Optimization Tips
1. **Queue Size**: Balance between memory and backpressure
2. **Thread Count**: CPU-bound: num_cores, I/O-bound: num_cores * 2-4
3. **Rejection Policy**: Choose based on use case (abort for critical, discard for non-critical)
4. **Task Batching**: Group small tasks to reduce overhead

## Common Pitfalls

1. **Not handling InterruptedException**: Can cause thread pool to hang
2. **Unbounded queues**: Can lead to OutOfMemoryError
3. **No rejection policy**: Tasks silently dropped or threads crash
4. **Not shutting down properly**: Resource leaks
5. **Ignoring task exceptions**: Silent failures in production
6. **Too many threads**: Context switching overhead
7. **Too few threads**: Underutilization and latency

## Variations

1. **ScheduledThreadPool**: For periodic/delayed tasks
2. **ForkJoinPool**: For divide-and-conquer tasks
3. **VirtualThreadPool**: Java 21+ lightweight threads
4. **WorkStealingPool**: Better load balancing
5. **CustomThreadFactory**: Control thread naming and properties

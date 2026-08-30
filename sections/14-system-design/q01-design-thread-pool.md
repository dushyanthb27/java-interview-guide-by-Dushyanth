# Q1: Design a Thread Pool

## Question
Design and implement a custom Thread Pool in Java. Discuss the data structures, concurrency considerations, and optimization strategies.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Submit `Runnable` and `Callable` tasks for asynchronous execution
- Support core pool size and maximum pool size
- Graceful and immediate shutdown mechanisms
- Configurable rejection policies when the pool is saturated
- Custom thread factory support

**Non-Functional Requirements:**
- Thread-safe task submission and execution
- Minimal lock contention under high load
- Clean resource cleanup on shutdown
- Fair scheduling of queued tasks

### API Design

```java
public interface CustomExecutor {
    void execute(Runnable task);
    <T> Future<T> submit(Callable<T> task);
    void shutdown();          // graceful
    void shutdownNow();       // immediate
    boolean isShutdown();
    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
}

@FunctionalInterface
public interface RejectionPolicy {
    void reject(Runnable task, CustomThreadPool pool);
}

@FunctionalInterface
public interface CustomThreadFactory {
    Thread newThread(Runnable r);
}
```

### Implementation

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.ReentrantLock;
import java.util.HashSet;
import java.util.List;
import java.util.ArrayList;

public class CustomThreadPool implements CustomExecutor {

    private final int corePoolSize;
    private final int maxPoolSize;
    private final long keepAliveTime;
    private final TimeUnit timeUnit;
    private final BlockingQueue<Runnable> workQueue;
    private final RejectionPolicy rejectionPolicy;
    private final CustomThreadFactory threadFactory;

    private final HashSet<Worker> workers = new HashSet<>();
    private final ReentrantLock mainLock = new ReentrantLock();
    private final AtomicInteger workerCount = new AtomicInteger(0);
    private volatile boolean isShutdown = false;
    private final CountDownLatch terminationLatch = new CountDownLatch(1);

    // --- Built-in rejection policies ---
    public static final RejectionPolicy ABORT_POLICY = (task, pool) -> {
        throw new RejectedExecutionException("Task rejected: pool is saturated");
    };

    public static final RejectionPolicy CALLER_RUNS_POLICY = (task, pool) -> {
        if (!pool.isShutdown()) {
            task.run(); // execute in caller's thread
        }
    };

    public static final RejectionPolicy DISCARD_POLICY = (task, pool) -> {
        // silently discard
    };

    // --- Default thread factory ---
    public static final CustomThreadFactory DEFAULT_THREAD_FACTORY = new CustomThreadFactory() {
        private final AtomicInteger counter = new AtomicInteger(1);
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "custom-pool-thread-" + counter.getAndIncrement());
            t.setDaemon(false);
            return t;
        }
    };

    public CustomThreadPool(int corePoolSize, int maxPoolSize,
                            long keepAliveTime, TimeUnit timeUnit,
                            BlockingQueue<Runnable> workQueue,
                            RejectionPolicy rejectionPolicy,
                            CustomThreadFactory threadFactory) {
        if (corePoolSize < 0 || maxPoolSize <= 0 || maxPoolSize < corePoolSize || keepAliveTime < 0)
            throw new IllegalArgumentException("Invalid pool configuration");

        this.corePoolSize = corePoolSize;
        this.maxPoolSize = maxPoolSize;
        this.keepAliveTime = keepAliveTime;
        this.timeUnit = timeUnit;
        this.workQueue = workQueue;
        this.rejectionPolicy = rejectionPolicy;
        this.threadFactory = threadFactory;
    }

    // Convenience constructor
    public CustomThreadPool(int corePoolSize, int maxPoolSize) {
        this(corePoolSize, maxPoolSize, 60L, TimeUnit.SECONDS,
             new LinkedBlockingQueue<>(128), ABORT_POLICY, DEFAULT_THREAD_FACTORY);
    }

    @Override
    public void execute(Runnable task) {
        if (task == null) throw new NullPointerException("Task cannot be null");
        if (isShutdown) throw new RejectedExecutionException("Pool is shut down");

        int currentCount = workerCount.get();

        // Step 1: If below core size, add a core worker
        if (currentCount < corePoolSize) {
            if (addWorker(task, true)) return;
        }

        // Step 2: Try to enqueue the task
        if (workQueue.offer(task)) {
            // Double-check shutdown after enqueue
            if (isShutdown && workQueue.remove(task)) {
                rejectionPolicy.reject(task, this);
            }
            return;
        }

        // Step 3: Queue is full — try to add a non-core worker up to maxPoolSize
        if (!addWorker(task, false)) {
            rejectionPolicy.reject(task, this);
        }
    }

    @Override
    public <T> Future<T> submit(Callable<T> callable) {
        if (callable == null) throw new NullPointerException();
        FutureTask<T> futureTask = new FutureTask<>(callable);
        execute(futureTask);
        return futureTask;
    }

    private boolean addWorker(Runnable firstTask, boolean core) {
        mainLock.lock();
        try {
            int count = workerCount.get();
            int limit = core ? corePoolSize : maxPoolSize;
            if (count >= limit || isShutdown) return false;

            Worker worker = new Worker(firstTask);
            workers.add(worker);
            workerCount.incrementAndGet();
            worker.thread.start();
            return true;
        } finally {
            mainLock.unlock();
        }
    }

    private void removeWorker(Worker worker) {
        mainLock.lock();
        try {
            workers.remove(worker);
            int remaining = workerCount.decrementAndGet();
            if (remaining == 0 && isShutdown) {
                terminationLatch.countDown();
            }
        } finally {
            mainLock.unlock();
        }
    }

    @Override
    public void shutdown() {
        mainLock.lock();
        try {
            isShutdown = true;
            // Don't interrupt workers — let them finish current + queued tasks
        } finally {
            mainLock.unlock();
        }
        // If no workers exist, signal termination immediately
        if (workerCount.get() == 0) {
            terminationLatch.countDown();
        }
    }

    @Override
    public void shutdownNow() {
        mainLock.lock();
        try {
            isShutdown = true;
            for (Worker w : workers) {
                w.thread.interrupt();
            }
            workQueue.clear();
        } finally {
            mainLock.unlock();
        }
    }

    @Override
    public boolean isShutdown() {
        return isShutdown;
    }

    @Override
    public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
        return terminationLatch.await(timeout, unit);
    }

    // ----- Inner Worker class -----
    private class Worker implements Runnable {
        final Thread thread;
        Runnable firstTask;

        Worker(Runnable firstTask) {
            this.firstTask = firstTask;
            this.thread = threadFactory.newThread(this);
        }

        @Override
        public void run() {
            Runnable task = firstTask;
            firstTask = null;
            try {
                while (task != null || (task = getTask()) != null) {
                    try {
                        task.run();
                    } catch (RuntimeException ex) {
                        // Log and continue — don't kill the worker
                        System.err.println("Task threw exception: " + ex.getMessage());
                    } finally {
                        task = null;
                    }
                }
            } finally {
                removeWorker(this);
            }
        }

        private Runnable getTask() {
            try {
                if (isShutdown && workQueue.isEmpty()) return null;

                int count = workerCount.get();
                boolean allowTimeout = count > corePoolSize;

                if (allowTimeout) {
                    Runnable task = workQueue.poll(keepAliveTime, timeUnit);
                    if (task == null) return null; // timed out — let non-core worker die
                    return task;
                } else {
                    if (isShutdown) {
                        return workQueue.poll(); // drain remaining, don't block
                    }
                    return workQueue.take(); // core worker blocks indefinitely
                }
            } catch (InterruptedException e) {
                return null; // shutdown interrupted us
            }
        }
    }
}
```

### Concurrency Considerations

| Concern | Solution |
|---------|----------|
| Worker set mutation | `ReentrantLock` guards `workers` set |
| Task queue access | `BlockingQueue` provides thread-safe enqueue/dequeue |
| Worker count tracking | `AtomicInteger` for lock-free reads |
| Shutdown visibility | `volatile boolean isShutdown` |
| Termination waiting | `CountDownLatch` signals when all workers exit |
| Task exception isolation | Each task runs in try/catch — one failure doesn't kill the worker |

### Performance Analysis

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `execute()` | O(1) amortized | Queue offer is O(1) for linked queues |
| `submit()` | O(1) amortized | Wraps in `FutureTask`, then calls `execute` |
| `shutdown()` | O(1) | Sets flag only |
| `shutdownNow()` | O(W) | Interrupts W workers |

**Bottlenecks:**
- `mainLock` is held during worker creation/removal (short critical section)
- `BlockingQueue` contention under very high throughput — consider `LinkedTransferQueue`

### Testing Strategy

```java
public class ThreadPoolTest {

    @Test
    void testBasicExecution() throws Exception {
        CustomThreadPool pool = new CustomThreadPool(2, 4);
        AtomicInteger counter = new AtomicInteger(0);

        for (int i = 0; i < 10; i++) {
            pool.execute(counter::incrementAndGet);
        }

        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);
        assertEquals(10, counter.get());
    }

    @Test
    void testSubmitCallable() throws Exception {
        CustomThreadPool pool = new CustomThreadPool(2, 4);
        Future<String> future = pool.submit(() -> "hello");
        assertEquals("hello", future.get(2, TimeUnit.SECONDS));
        pool.shutdown();
    }

    @Test
    void testRejectionPolicy() {
        CustomThreadPool pool = new CustomThreadPool(1, 1, 60, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(1), CustomThreadPool.ABORT_POLICY,
            CustomThreadPool.DEFAULT_THREAD_FACTORY);

        // Fill the pool and queue
        pool.execute(() -> { try { Thread.sleep(5000); } catch (Exception e) {} });
        pool.execute(() -> {}); // fills the queue

        assertThrows(RejectedExecutionException.class, () -> pool.execute(() -> {}));
        pool.shutdownNow();
    }

    @Test
    void testGracefulShutdown() throws Exception {
        CustomThreadPool pool = new CustomThreadPool(2, 2);
        AtomicInteger counter = new AtomicInteger(0);

        for (int i = 0; i < 5; i++) {
            pool.execute(() -> {
                try { Thread.sleep(100); } catch (Exception e) {}
                counter.incrementAndGet();
            });
        }

        pool.shutdown();
        assertTrue(pool.awaitTermination(10, TimeUnit.SECONDS));
        assertEquals(5, counter.get());
    }
}
```

---

## Interviewer Insight
Interviewers want to see that you understand the lifecycle of threads in a pool: creation, reuse, keep-alive timeout, and shutdown. They look for proper use of `BlockingQueue`, awareness of rejection policies, and the difference between core and max pool size. Bonus points for comparing your design to `ThreadPoolExecutor` internals.

## Follow-up Questions
1. How does `ThreadPoolExecutor` decide when to create a new thread vs. enqueue a task?
2. What happens if a task throws an unchecked exception — does the worker thread die?
3. How would you implement a scheduled thread pool (like `ScheduledExecutorService`)?

## Related Topics
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md)
- [Performance Optimization](../12-performance-optimization/README.md)

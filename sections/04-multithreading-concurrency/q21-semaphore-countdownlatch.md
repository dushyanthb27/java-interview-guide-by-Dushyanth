# Q21: Explain Semaphore and CountDownLatch

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
How do `Semaphore` and `CountDownLatch` work? What are their use cases?

---

## Answer

### Semaphore

A `Semaphore` controls access to a shared resource by maintaining a set of **permits**.

```
Semaphore(3):   [■ ■ ■]  ← 3 permits available

Thread A: acquire() → [■ ■ □]  (took 1 permit, 2 left)
Thread B: acquire() → [■ □ □]  (took 1 permit, 1 left)
Thread C: acquire() → [□ □ □]  (took 1 permit, 0 left)
Thread D: acquire() → BLOCKED   (no permits available)

Thread A: release() → [■ □ □]  (returned 1 permit)
Thread D: unblocked → [□ □ □]  (took the permit)
```

```java
public class SemaphoreDemo {
    // Limit concurrent access to 3 threads
    private final Semaphore semaphore = new Semaphore(3);

    public void accessResource() {
        try {
            semaphore.acquire(); // Blocks until a permit is available
            try {
                System.out.println(Thread.currentThread().getName() +
                    " acquired permit. Available: " + semaphore.availablePermits());
                Thread.sleep(2000); // Simulate work
            } finally {
                semaphore.release(); // Always release in finally!
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) {
        SemaphoreDemo demo = new SemaphoreDemo();

        // 10 threads competing for 3 permits
        for (int i = 0; i < 10; i++) {
            new Thread(demo::accessResource, "Thread-" + i).start();
        }
    }
}
```

### Semaphore: Connection Pool Example

```java
public class ConnectionPool {
    private final Semaphore semaphore;
    private final BlockingQueue<Connection> pool;

    public ConnectionPool(int poolSize) {
        this.semaphore = new Semaphore(poolSize);
        this.pool = new ArrayBlockingQueue<>(poolSize);

        for (int i = 0; i < poolSize; i++) {
            pool.add(createConnection(i));
        }
    }

    public Connection borrowConnection() throws InterruptedException {
        semaphore.acquire(); // Wait for available connection
        return pool.poll();  // Take from pool
    }

    public Connection borrowConnection(long timeout, TimeUnit unit)
            throws InterruptedException {
        if (semaphore.tryAcquire(timeout, unit)) {
            return pool.poll();
        }
        throw new RuntimeException("Connection pool timeout");
    }

    public void returnConnection(Connection conn) {
        pool.offer(conn);
        semaphore.release(); // Signal that a connection is available
    }

    private Connection createConnection(int id) {
        return null; // Placeholder
    }

    public static void main(String[] args) throws InterruptedException {
        ConnectionPool pool = new ConnectionPool(5);

        ExecutorService executor = Executors.newFixedThreadPool(20);
        for (int i = 0; i < 20; i++) {
            executor.submit(() -> {
                try {
                    Connection conn = pool.borrowConnection();
                    Thread.sleep(1000); // Use connection
                    pool.returnConnection(conn);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        executor.shutdown();
    }
}
```

### Semaphore: Fair vs Unfair

```java
// Unfair (default) - better throughput
Semaphore unfair = new Semaphore(5, false);

// Fair - FIFO ordering, prevents starvation
Semaphore fair = new Semaphore(5, true);

// tryAcquire: Non-blocking attempt
if (semaphore.tryAcquire()) {
    try {
        // Use resource
    } finally {
        semaphore.release();
    }
} else {
    System.out.println("Resource busy, trying alternative...");
}

// tryAcquire with timeout
if (semaphore.tryAcquire(2, TimeUnit.SECONDS)) {
    // Got permit within 2 seconds
}

// Acquire multiple permits
semaphore.acquire(3); // Acquire 3 permits at once
semaphore.release(3); // Release 3 permits
```

### Binary Semaphore (Mutex)

```java
// Semaphore(1) acts as a mutual exclusion lock (mutex)
public class BinarySemaphore {
    private final Semaphore mutex = new Semaphore(1);
    private int sharedData = 0;

    public void criticalSection() throws InterruptedException {
        mutex.acquire();
        try {
            sharedData++;
        } finally {
            mutex.release();
        }
    }

    // Note: Unlike ReentrantLock, Semaphore is NOT reentrant
    // The same thread calling acquire() twice will deadlock with Semaphore(1)!
}
```

---

### CountDownLatch

A `CountDownLatch` allows one or more threads to wait until a set of operations completes. The count can only go down, never up.

```
CountDownLatch(3):  count = 3

Thread A: countDown() → count = 2
Thread B: countDown() → count = 1
Thread C: countDown() → count = 0

Main:     await()    → UNBLOCKED! (count reached 0)

Note: One-time use only! Cannot be reset.
```

```java
public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        int workerCount = 5;
        CountDownLatch latch = new CountDownLatch(workerCount);

        for (int i = 0; i < workerCount; i++) {
            final int workerId = i;
            new Thread(() -> {
                try {
                    System.out.println("Worker " + workerId + " starting...");
                    Thread.sleep(ThreadLocalRandom.current().nextInt(1000, 3000));
                    System.out.println("Worker " + workerId + " done!");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown(); // Decrement count
                }
            }).start();
        }

        System.out.println("Main thread waiting for workers...");
        latch.await(); // Block until count reaches 0
        System.out.println("All workers completed!");
    }
}
```

### CountDownLatch: Service Startup Coordination

```java
public class ServiceStartup {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch readyLatch = new CountDownLatch(3);

        // Start services in parallel
        new Thread(() -> {
            System.out.println("Database initializing...");
            try { Thread.sleep(2000); } catch (InterruptedException e) {}
            System.out.println("Database ready!");
            readyLatch.countDown();
        }, "DB-Init").start();

        new Thread(() -> {
            System.out.println("Cache warming...");
            try { Thread.sleep(1500); } catch (InterruptedException e) {}
            System.out.println("Cache ready!");
            readyLatch.countDown();
        }, "Cache-Init").start();

        new Thread(() -> {
            System.out.println("Message queue connecting...");
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
            System.out.println("Message queue ready!");
            readyLatch.countDown();
        }, "MQ-Init").start();

        // Wait for all services to be ready
        System.out.println("Waiting for all services...");
        boolean ready = readyLatch.await(10, TimeUnit.SECONDS);

        if (ready) {
            System.out.println("All services ready! Starting application...");
        } else {
            System.err.println("Timeout! Not all services started.");
            System.exit(1);
        }
    }
}
```

### CountDownLatch: Coordinated Start (Starting Gun)

```java
public class CoordinatedStart {
    public static void main(String[] args) throws InterruptedException {
        int runners = 5;
        CountDownLatch startGun = new CountDownLatch(1);    // Wait for "go!"
        CountDownLatch finished = new CountDownLatch(runners); // Wait for all to finish

        for (int i = 0; i < runners; i++) {
            final int runnerId = i;
            new Thread(() -> {
                try {
                    System.out.println("Runner " + runnerId + " ready!");
                    startGun.await(); // All runners wait at the start line

                    // Race!
                    long time = ThreadLocalRandom.current().nextInt(100, 500);
                    Thread.sleep(time);
                    System.out.println("Runner " + runnerId + " finished in " + time + "ms");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    finished.countDown();
                }
            }).start();
        }

        Thread.sleep(1000); // Prepare...
        System.out.println("GO!");
        startGun.countDown(); // Release all runners simultaneously!

        finished.await(); // Wait for all to finish
        System.out.println("Race over!");
    }
}
```

### Comparison: Semaphore vs CountDownLatch

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ Feature          │ Semaphore            │ CountDownLatch       │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Purpose          │ Limit concurrency    │ Wait for events      │
│ Count direction  │ Up AND down          │ Down only            │
│ Reusable         │ ✅ Yes              │ ❌ One-time          │
│ Thread ownership │ No (any can release) │ No (any can count)   │
│ Zero means       │ All permits in use   │ All events completed │
│ Blocking method  │ acquire()            │ await()              │
│ Release method   │ release()            │ countDown()          │
└──────────────────┴──────────────────────┴──────────────────────┘
```

---

## Anti-Patterns

❌ **Not releasing semaphore in finally block:**
```java
semaphore.acquire();
doWork(); // If this throws, permit is lost forever!
semaphore.release();
```

❌ **Trying to reuse CountDownLatch:**
```java
CountDownLatch latch = new CountDownLatch(1);
latch.countDown();
latch.await(); // Returns immediately
// Cannot reset! Use CyclicBarrier for reusable barriers
```

❌ **Mismatched acquire/release counts in Semaphore**

## Best Practices

✅ Always release semaphore permits in a finally block  
✅ Use `CountDownLatch` for one-time coordination events  
✅ Use `await(timeout, unit)` to prevent indefinite blocking  
✅ Use `Semaphore` for resource pooling and rate limiting  
✅ For reusable barriers, use `CyclicBarrier` instead of `CountDownLatch`

---

[← Thread Safety Patterns](./q20-thread-safety-patterns.md) | [Next: CyclicBarrier & Phaser →](./q22-cyclic-barrier-phaser.md)

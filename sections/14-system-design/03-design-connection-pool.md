# Q3: Design a Connection Pool

## Question
Design and implement a Connection Pool in Java. Discuss the data structures, concurrency considerations, and optimization strategies.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Acquire a connection from the pool (blocking if none available)
- Release a connection back to the pool
- Support a configurable minimum and maximum pool size
- Connection validation before handing out
- Timeout support when waiting for a connection
- Health checking / idle connection eviction

**Non-Functional Requirements:**
- Thread-safe acquire and release
- Efficient resource utilization — avoid creating unnecessary connections
- Graceful shutdown with cleanup of all connections
- Leak detection for unreturned connections

### API Design

```java
public interface ConnectionPool<T> {
    T acquire() throws InterruptedException;
    T acquire(long timeout, TimeUnit unit) throws InterruptedException, TimeoutException;
    void release(T connection);
    void shutdown();
    int getActiveCount();
    int getIdleCount();
    int getTotalCount();
}

public interface ConnectionFactory<T> {
    T create() throws Exception;
    boolean validate(T connection);
    void destroy(T connection);
}
```

### Implementation

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.ReentrantLock;

public class GenericConnectionPool<T> implements ConnectionPool<T> {

    private final int minSize;
    private final int maxSize;
    private final ConnectionFactory<T> factory;
    private final BlockingQueue<T> idleConnections;
    private final ConcurrentHashMap<T, Long> activeConnections; // connection -> checkout time
    private final AtomicInteger totalCreated = new AtomicInteger(0);
    private final ReentrantLock createLock = new ReentrantLock();
    private volatile boolean shutdown = false;

    private final long maxIdleTimeMillis;
    private final long leakDetectionThresholdMillis;
    private final ScheduledExecutorService maintenanceExecutor;

    public GenericConnectionPool(int minSize, int maxSize,
                                  ConnectionFactory<T> factory,
                                  long maxIdleTimeMillis,
                                  long leakDetectionThresholdMillis) {
        this.minSize = minSize;
        this.maxSize = maxSize;
        this.factory = factory;
        this.maxIdleTimeMillis = maxIdleTimeMillis;
        this.leakDetectionThresholdMillis = leakDetectionThresholdMillis;
        this.idleConnections = new LinkedBlockingQueue<>(maxSize);
        this.activeConnections = new ConcurrentHashMap<>();

        // Pre-populate minimum connections
        initializePool();

        // Schedule maintenance tasks
        maintenanceExecutor = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "pool-maintenance");
            t.setDaemon(true);
            return t;
        });
        maintenanceExecutor.scheduleAtFixedRate(this::performMaintenance, 30, 30, TimeUnit.SECONDS);
    }

    private void initializePool() {
        for (int i = 0; i < minSize; i++) {
            try {
                T conn = factory.create();
                idleConnections.offer(conn);
                totalCreated.incrementAndGet();
            } catch (Exception e) {
                System.err.println("Failed to initialize connection: " + e.getMessage());
            }
        }
    }

    @Override
    public T acquire() throws InterruptedException {
        try {
            return acquire(Long.MAX_VALUE, TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            throw new RuntimeException("Unexpected timeout", e); // should never happen
        }
    }

    @Override
    public T acquire(long timeout, TimeUnit unit) throws InterruptedException, TimeoutException {
        if (shutdown) throw new IllegalStateException("Pool is shut down");

        long deadlineNanos = System.nanoTime() + unit.toNanos(timeout);

        while (true) {
            // 1. Try to get an idle connection
            T conn = idleConnections.poll();

            if (conn != null) {
                // Validate before returning
                if (factory.validate(conn)) {
                    activeConnections.put(conn, System.currentTimeMillis());
                    return conn;
                } else {
                    // Connection is stale — destroy and try again
                    destroyConnection(conn);
                    continue;
                }
            }

            // 2. No idle connection — try to create a new one
            if (totalCreated.get() < maxSize) {
                createLock.lock();
                try {
                    if (totalCreated.get() < maxSize) {
                        try {
                            conn = factory.create();
                            totalCreated.incrementAndGet();
                            activeConnections.put(conn, System.currentTimeMillis());
                            return conn;
                        } catch (Exception e) {
                            throw new RuntimeException("Failed to create connection", e);
                        }
                    }
                } finally {
                    createLock.unlock();
                }
            }

            // 3. Pool is full — wait for a released connection
            long remainingNanos = deadlineNanos - System.nanoTime();
            if (remainingNanos <= 0) {
                throw new TimeoutException("Timed out waiting for a connection. " +
                    "Active: " + getActiveCount() + ", Idle: " + getIdleCount());
            }

            conn = idleConnections.poll(Math.min(remainingNanos, TimeUnit.SECONDS.toNanos(1)),
                                         TimeUnit.NANOSECONDS);
            if (conn != null) {
                if (factory.validate(conn)) {
                    activeConnections.put(conn, System.currentTimeMillis());
                    return conn;
                } else {
                    destroyConnection(conn);
                }
            }
        }
    }

    @Override
    public void release(T connection) {
        if (connection == null) return;
        if (shutdown) {
            destroyConnection(connection);
            return;
        }

        activeConnections.remove(connection);

        if (factory.validate(connection)) {
            if (!idleConnections.offer(connection)) {
                // Pool is full — destroy the extra connection
                destroyConnection(connection);
            }
        } else {
            destroyConnection(connection);
        }
    }

    @Override
    public void shutdown() {
        shutdown = true;
        maintenanceExecutor.shutdown();

        // Destroy all idle connections
        T conn;
        while ((conn = idleConnections.poll()) != null) {
            destroyConnection(conn);
        }

        // Active connections will be destroyed when returned via release()
        System.out.println("Pool shutdown initiated. Active connections: " + getActiveCount());
    }

    @Override
    public int getActiveCount() {
        return activeConnections.size();
    }

    @Override
    public int getIdleCount() {
        return idleConnections.size();
    }

    @Override
    public int getTotalCount() {
        return totalCreated.get();
    }

    private void destroyConnection(T conn) {
        try {
            factory.destroy(conn);
        } catch (Exception e) {
            System.err.println("Error destroying connection: " + e.getMessage());
        }
        totalCreated.decrementAndGet();
    }

    // --- Maintenance: idle eviction + leak detection ---

    private void performMaintenance() {
        if (shutdown) return;

        // Evict idle connections beyond minSize
        int idleCount = idleConnections.size();
        int toEvict = idleCount - minSize;
        for (int i = 0; i < toEvict; i++) {
            T conn = idleConnections.poll();
            if (conn != null) {
                destroyConnection(conn);
            }
        }

        // Detect leaked connections
        long now = System.currentTimeMillis();
        activeConnections.forEach((conn, checkoutTime) -> {
            long heldMillis = now - checkoutTime;
            if (heldMillis > leakDetectionThresholdMillis) {
                System.err.println("LEAK DETECTED: Connection held for " +
                    heldMillis + "ms (threshold: " + leakDetectionThresholdMillis + "ms)");
            }
        });

        // Replenish to minSize
        while (totalCreated.get() < minSize) {
            try {
                T conn = factory.create();
                totalCreated.incrementAndGet();
                if (!idleConnections.offer(conn)) {
                    destroyConnection(conn);
                }
            } catch (Exception e) {
                break;
            }
        }
    }
}
```

#### Example Connection Factory (JDBC)

```java
import java.sql.Connection;
import java.sql.DriverManager;

public class JdbcConnectionFactory implements ConnectionFactory<Connection> {

    private final String url;
    private final String username;
    private final String password;

    public JdbcConnectionFactory(String url, String username, String password) {
        this.url = url;
        this.username = username;
        this.password = password;
    }

    @Override
    public Connection create() throws Exception {
        return DriverManager.getConnection(url, username, password);
    }

    @Override
    public boolean validate(Connection connection) {
        try {
            return connection != null && !connection.isClosed() && connection.isValid(1);
        } catch (Exception e) {
            return false;
        }
    }

    @Override
    public void destroy(Connection connection) {
        try {
            if (connection != null && !connection.isClosed()) {
                connection.close();
            }
        } catch (Exception e) {
            // log and ignore
        }
    }
}
```

### Concurrency Considerations

| Concern | Solution |
|---------|----------|
| Idle queue access | `BlockingQueue` — thread-safe blocking deque |
| Active tracking | `ConcurrentHashMap` — lock-free reads |
| Connection creation race | `ReentrantLock` + double-check on `totalCreated < maxSize` |
| Pool exhaustion | `poll(timeout)` on the blocking queue |
| Shutdown safety | `volatile boolean` flag checked on acquire/release |
| Leak detection | Background thread checks checkout timestamps |

### Performance Analysis

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `acquire()` (idle available) | O(1) | Queue poll |
| `acquire()` (create new) | O(1) + connection setup time | Lock is short |
| `acquire()` (wait) | Blocks up to timeout | Uses BlockingQueue |
| `release()` | O(1) | Queue offer + map remove |

**Comparison with HikariCP:**
- HikariCP uses `ConcurrentBag` instead of `BlockingQueue` — optimizes for thread-local reuse
- HikariCP tracks connections via a state machine (STATE_NOT_IN_USE, STATE_IN_USE)
- HikariCP avoids `synchronized` blocks entirely, using CAS operations

### Testing Strategy

```java
public class ConnectionPoolTest {

    private ConnectionFactory<String> mockFactory() {
        AtomicInteger counter = new AtomicInteger(0);
        return new ConnectionFactory<>() {
            public String create() { return "conn-" + counter.incrementAndGet(); }
            public boolean validate(String c) { return c != null; }
            public void destroy(String c) { /* no-op */ }
        };
    }

    @Test
    void testAcquireAndRelease() throws Exception {
        var pool = new GenericConnectionPool<>(2, 5, mockFactory(), 60000, 300000);
        String c1 = pool.acquire();
        assertNotNull(c1);
        assertEquals(1, pool.getActiveCount());

        pool.release(c1);
        assertEquals(0, pool.getActiveCount());
        assertEquals(1, pool.getIdleCount());
        pool.shutdown();
    }

    @Test
    void testPoolExhaustion() throws Exception {
        var pool = new GenericConnectionPool<>(1, 2, mockFactory(), 60000, 300000);
        String c1 = pool.acquire();
        String c2 = pool.acquire();

        assertThrows(TimeoutException.class,
            () -> pool.acquire(500, TimeUnit.MILLISECONDS));

        pool.release(c1);
        pool.release(c2);
        pool.shutdown();
    }

    @Test
    void testConcurrentAccess() throws Exception {
        var pool = new GenericConnectionPool<>(5, 10, mockFactory(), 60000, 300000);
        ExecutorService exec = Executors.newFixedThreadPool(20);
        AtomicInteger successCount = new AtomicInteger(0);

        for (int i = 0; i < 100; i++) {
            exec.submit(() -> {
                try {
                    String conn = pool.acquire(2, TimeUnit.SECONDS);
                    Thread.sleep(50);
                    pool.release(conn);
                    successCount.incrementAndGet();
                } catch (Exception e) { /* timeout */ }
            });
        }

        exec.shutdown();
        exec.awaitTermination(30, TimeUnit.SECONDS);
        assertTrue(successCount.get() > 90); // most should succeed
        pool.shutdown();
    }
}
```

---

## Interviewer Insight
Interviewers look for understanding of resource lifecycle management — creation, validation, reuse, and cleanup. Key discussion areas include how to handle pool exhaustion (block vs. reject), leak detection strategies, and idle connection eviction. Comparing your design to production pools like HikariCP shows depth of knowledge.

## Follow-up Questions
1. How does HikariCP achieve such high performance compared to other pools?
2. How would you implement connection warmup to avoid cold-start latency?
3. How would you handle a database failover — should the pool detect it and recreate connections?

## Related Topics
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md)
- [Performance Optimization](../12-performance-optimization/README.md)

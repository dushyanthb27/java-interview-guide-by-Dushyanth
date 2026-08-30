# Q8: When Should You Use Object Pooling?

[← Back to Memory Management](./README.md)

---

## Overview

Object pooling is a design pattern where a set of initialized objects is kept ready for use rather than created and destroyed on demand. In modern JVMs, pooling is **rarely beneficial** for ordinary Java objects, but it remains essential for **expensive resources** like database connections and threads.

---

## When Pooling Helps ✅

### 1. Database Connections

Creating a TCP connection, performing authentication, and setting up a session is expensive (~5-50ms per connection).

```java
// ❌ Anti-pattern: creating a new connection per request
public User findUser(int id) throws SQLException {
    Connection conn = DriverManager.getConnection(url, user, pass);   // ~20ms
    try {
        PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
        ps.setInt(1, id);
        return mapUser(ps.executeQuery());
    } finally {
        conn.close();   // Connection destroyed
    }
}
```

```java
// ✅ Using HikariCP connection pool
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("user");
        config.setPassword("pass");
        config.setMaximumPoolSize(10);         // Max 10 connections
        config.setMinimumIdle(5);              // Keep 5 warm
        config.setConnectionTimeout(30_000);   // 30s to get a connection
        config.setIdleTimeout(600_000);        // 10min idle before eviction
        config.setMaxLifetime(1_800_000);      // 30min max lifetime
        return new HikariDataSource(config);
    }
}
```

### 2. Thread Pools

Thread creation involves OS-level resource allocation.

```java
// ❌ Anti-pattern: creating a thread per task
public void handleRequest(Request req) {
    new Thread(() -> process(req)).start();   // OS thread creation ~1ms + memory
}
```

```java
// ✅ Using thread pool
ExecutorService pool = Executors.newFixedThreadPool(
        Runtime.getRuntime().availableProcessors()
);

public void handleRequest(Request req) {
    pool.submit(() -> process(req));   // Reuses existing thread
}

// ✅ Even better: virtual threads (Java 21+)
ExecutorService virtualPool = Executors.newVirtualThreadPerTaskExecutor();
```

### 3. Expensive-to-Create Objects

```java
// ✅ Pooling makes sense for heavyweight objects
// - SSL contexts / sessions
// - XML/JSON parsers (SAX/DOM parsers)
// - Compiled regex patterns (though these are usually cached differently)
// - Graphics contexts

// Example: reusable byte buffers for I/O
public class BufferPool {

    private final Queue<ByteBuffer> pool = new ConcurrentLinkedQueue<>();
    private final int bufferSize;

    public BufferPool(int bufferSize, int initialCount) {
        this.bufferSize = bufferSize;
        for (int i = 0; i < initialCount; i++) {
            pool.offer(ByteBuffer.allocateDirect(bufferSize));
        }
    }

    public ByteBuffer acquire() {
        ByteBuffer buffer = pool.poll();
        if (buffer == null) {
            buffer = ByteBuffer.allocateDirect(bufferSize);   // Grow pool
        }
        buffer.clear();   // Reset for reuse
        return buffer;
    }

    public void release(ByteBuffer buffer) {
        if (buffer.capacity() == bufferSize) {
            pool.offer(buffer);   // Return to pool
        }
    }
}
```

---

## When Pooling Hurts ❌

### 1. Lightweight Objects

```java
// ❌ Anti-pattern: pooling simple objects
public class PointPool {
    private final Queue<Point> pool = new ConcurrentLinkedQueue<>();

    public Point acquire(int x, int y) {
        Point p = pool.poll();
        if (p == null) {
            return new Point(x, y);
        }
        p.setX(x);
        p.setY(y);
        return p;
    }

    public void release(Point p) {
        pool.offer(p);
    }
}
// Problems:
// 1. Pool overhead (synchronization) exceeds allocation cost
// 2. New objects are allocated in Eden — nearly free
// 3. Short-lived objects are collected in minor GC — nearly free
// 4. Mutable state bugs from forgetting to reset fields
```

```java
// ✅ Just create new instances — JVM is optimized for this
Point p = new Point(x, y);   // ~10 nanoseconds
// GC handles the rest efficiently
```

### 2. Immutable or Value-Like Objects

```java
// ❌ Don't pool Strings, Integers, records, etc.
// The JVM already caches Integer (-128 to 127), String literals, etc.

// ✅ Use records/value objects — trivial to allocate and GC
record Coordinate(double lat, double lon) {}
Coordinate c = new Coordinate(40.7128, -74.0060);
```

---

## Commons Pool Library (Apache)

For custom pooling scenarios, Apache Commons Pool provides a robust framework:

```java
import org.apache.commons.pool2.BasePooledObjectFactory;
import org.apache.commons.pool2.PooledObject;
import org.apache.commons.pool2.impl.DefaultPooledObject;
import org.apache.commons.pool2.impl.GenericObjectPool;
import org.apache.commons.pool2.impl.GenericObjectPoolConfig;

// Step 1: Define a factory
public class ExpensiveResourceFactory
        extends BasePooledObjectFactory<ExpensiveResource> {

    @Override
    public ExpensiveResource create() {
        System.out.println("Creating expensive resource...");
        return new ExpensiveResource();   // Simulated expensive creation
    }

    @Override
    public PooledObject<ExpensiveResource> wrap(ExpensiveResource resource) {
        return new DefaultPooledObject<>(resource);
    }

    @Override
    public void passivateObject(PooledObject<ExpensiveResource> pooled) {
        pooled.getObject().reset();   // Clean state before returning to pool
    }

    @Override
    public boolean validateObject(PooledObject<ExpensiveResource> pooled) {
        return pooled.getObject().isValid();   // Health check
    }

    @Override
    public void destroyObject(PooledObject<ExpensiveResource> pooled) {
        pooled.getObject().dispose();   // Release underlying resources
    }
}
```

```java
// Step 2: Configure and create the pool
GenericObjectPoolConfig<ExpensiveResource> config = new GenericObjectPoolConfig<>();
config.setMaxTotal(10);               // Max objects in pool
config.setMaxIdle(5);                 // Max idle objects
config.setMinIdle(2);                 // Min idle objects (pre-warmed)
config.setTestOnBorrow(true);         // Validate before giving to client
config.setTestOnReturn(true);         // Validate on return
config.setMaxWaitMillis(5000);        // Max wait for available object

GenericObjectPool<ExpensiveResource> pool =
        new GenericObjectPool<>(new ExpensiveResourceFactory(), config);
```

```java
// Step 3: Use the pool
public void processWork() throws Exception {
    ExpensiveResource resource = null;
    try {
        resource = pool.borrowObject();    // Borrow from pool
        resource.doWork();
    } finally {
        if (resource != null) {
            pool.returnObject(resource);   // Return to pool
        }
    }
}
```

---

## Pool Sizing Guidelines

| Parameter        | Too Small                              | Too Large                            |
|------------------|----------------------------------------|--------------------------------------|
| **Max size**     | Thread contention, timeouts            | Wasted memory, resource exhaustion   |
| **Min idle**     | Cold start latency                     | Idle resources consuming memory      |
| **Max wait**     | Fast failure but poor UX               | Thread starvation if pool exhausted  |

### Connection pool sizing formula (HikariCP recommendation)

```
pool_size = (core_count * 2) + effective_spindle_count

# For SSD-backed PostgreSQL on an 8-core machine:
# pool_size = (8 * 2) + 1 = 17
```

---

## HikariCP Connection Pool Internals

```
                    ┌─────────────────┐
                    │   Application   │
                    │   Thread Pool   │
                    └────────┬────────┘
                             │  borrowConnection()
                    ┌────────▼────────┐
                    │    HikariCP     │
                    │  ┌───────────┐  │
                    │  │ ConcurrentBag│ ← Lock-free collection
                    │  │  ┌─────┐  │  │
                    │  │  │Conn1│  │  │  ← threadLocal first (no contention)
                    │  │  │Conn2│  │  │  ← then shared list (CAS-based)
                    │  │  │Conn3│  │  │
                    │  │  └─────┘  │  │
                    │  └───────────┘  │
                    │                 │
                    │  HouseKeeper    │  ← Scheduled task: evicts idle, replaces dead
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Database (TCP) │
                    └─────────────────┘
```

### HikariCP key design choices:
- **ConcurrentBag**: lock-free, thread-local affinity for borrowed connections
- **No `Statement` caching**: delegates to JDBC driver (more efficient)
- **Bytecode generation**: uses Javassist to create fast proxy classes
- **Connection validation**: `isValid()` or a lightweight test query

---

## Simple Pool Implementation

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.function.Supplier;

public class SimpleObjectPool<T> {

    private final BlockingQueue<T> pool;
    private final Supplier<T> factory;

    public SimpleObjectPool(Supplier<T> factory, int size) {
        this.factory = factory;
        this.pool = new LinkedBlockingQueue<>(size);

        // Pre-populate
        for (int i = 0; i < size; i++) {
            pool.offer(factory.get());
        }
    }

    public T borrow() throws InterruptedException {
        T obj = pool.poll();
        return (obj != null) ? obj : factory.get();   // Create if pool is empty
    }

    public void release(T obj) {
        if (!pool.offer(obj)) {
            // Pool is full — discard the object
        }
    }

    public int available() {
        return pool.size();
    }
}
```

---

## Decision Framework

```
Should I pool this object?

  Is the object expensive to create?  ──── No ───▶ Don't pool ❌
        │
       Yes
        │
  Is the object reused frequently?  ──── No ───▶ Don't pool ❌
        │
       Yes
        │
  Can the object be safely reset?   ──── No ───▶ Don't pool ❌
        │
       Yes
        │
  Does pooling reduce GC pressure  ──── No ───▶ Don't pool ❌
  more than synchronization costs?
        │
       Yes
        ▼
    Pool it ✅
```

---

## Best Practices

```java
// ✅ Pool database connections (HikariCP, DBCP2)
// ✅ Pool threads (ExecutorService, ForkJoinPool)
// ✅ Pool direct ByteBuffers (native memory allocation is expensive)
// ✅ Use try-finally to always return objects to the pool
// ✅ Validate objects before handing them out
// ✅ Set max pool sizes to prevent resource exhaustion

// ❌ Don't pool plain Java objects (Points, DTOs, etc.)
// ❌ Don't pool immutable objects — they're GC-friendly
// ❌ Don't forget to reset object state before returning to pool
// ❌ Don't create unbounded pools — they can exhaust memory
// ❌ Don't share pool objects across threads without synchronization
```

---

## Interview Tips
- Modern JVMs allocate objects in ~10ns and collect short-lived objects in minor GC — pooling lightweight objects is counterproductive
- Connection pools (HikariCP) and thread pools (ExecutorService) are the two most important pools in Java
- HikariCP uses `ConcurrentBag` with thread-local affinity for near-zero contention
- Pool sizing is usually `(cores * 2) + spindles` for database connections
- Object pooling adds complexity (reset logic, validation, leak detection) — use only when justified

---

[← Q7: Reference Queues](./q07-reference-queues.md) | [Next: Q9 - String Pool →](./q09-string-pool.md)

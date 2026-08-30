# Q3: Design a Connection Pool

## 📋 Question
Design a database connection pool. Discuss the data structures, connection lifecycle management, and performance optimization strategies.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Borrow and return connections
- Support connection validation
- Handle connection timeouts
- Support multiple databases
- Connection lifecycle management

**Non-Functional Requirements:**
- Thread-safe operations
- Minimal connection creation overhead
- Proper resource cleanup
- Configurable pool size

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Application Layer                    │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                 Connection Pool Manager                 │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Idle Connections (Stack/Queue)                   │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐                       │  │
│  │  │ Conn│ │ Conn│ │ Conn│                       │  │
│  │  └─────┘ └─────┘ └─────┘                       │  │
│  └──────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Active Connections (Set)                        │  │
│  │  ┌─────┐ ┌─────┐                                 │  │
│  │  │ Conn│ │ Conn│                                 │  │
│  │  └─────┘ └─────┘                                 │  │
│  └──────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                  Database Server                        │
└─────────────────────────────────────────────────────────┘
```

### Implementation

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class ConnectionPool {
    private final BlockingQueue<PooledConnection> idleConnections;
    private final AtomicInteger activeConnections = new AtomicInteger(0);
    private final int maxPoolSize;
    private final String url;
    private final String username;
    private final String password;
    private final long connectionTimeout;
    private final TimeUnit timeUnit;

    public ConnectionPool(String url, String username, String password, 
                          int maxPoolSize, long connectionTimeout, TimeUnit timeUnit) {
        this.url = url;
        this.username = username;
        this.password = password;
        this.maxPoolSize = maxPoolSize;
        this.connectionTimeout = connectionTimeout;
        this.timeUnit = timeUnit;
        this.idleConnections = new LinkedBlockingQueue<>(maxPoolSize);
    }

    public Connection getConnection() throws SQLException {
        PooledConnection pooledConn = idleConnections.poll();
        
        if (pooledConn != null) {
            if (validateConnection(pooledConn)) {
                activeConnections.incrementAndGet();
                return new PooledConnectionWrapper(pooledConn, this);
            } else {
                // Invalid connection, close and create new
                closeConnection(pooledConn);
            }
        }
        
        // Create new connection if under limit
        if (activeConnections.get() < maxPoolSize) {
            Connection newConn = createNewConnection();
            PooledConnection newPooledConn = new PooledConnection(newConn);
            activeConnections.incrementAndGet();
            return new PooledConnectionWrapper(newPooledConn, this);
        }
        
        // Wait for available connection
        try {
            pooledConn = idleConnections.poll(connectionTimeout, timeUnit);
            if (pooledConn != null && validateConnection(pooledConn)) {
                activeConnections.incrementAndGet();
                return new PooledConnectionWrapper(pooledConn, this);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new SQLException("Interrupted while waiting for connection");
        }
        
        throw new SQLException("Connection timeout - no available connections");
    }

    public void releaseConnection(PooledConnection pooledConn) {
        if (validateConnection(pooledConn)) {
            idleConnections.offer(pooledConn);
        } else {
            closeConnection(pooledConn);
        }
        activeConnections.decrementAndGet();
    }

    private Connection createNewConnection() throws SQLException {
        return DriverManager.getConnection(url, username, password);
    }

    private boolean validateConnection(PooledConnection pooledConn) {
        try {
            return pooledConn.getConnection().isValid(5); // 5 second timeout
        } catch (SQLException e) {
            return false;
        }
    }

    private void closeConnection(PooledConnection pooledConn) {
        try {
            pooledConn.getConnection().close();
        } catch (SQLException e) {
            // Log error
        }
    }

    public int getActiveConnectionCount() {
        return activeConnections.get();
    }

    public int getIdleConnectionCount() {
        return idleConnections.size();
    }

    public void shutdown() {
        for (PooledConnection conn : idleConnections) {
            closeConnection(conn);
        }
        idleConnections.clear();
    }

    static class PooledConnection {
        private final Connection connection;
        private long lastUsedTime;

        PooledConnection(Connection connection) {
            this.connection = connection;
            this.lastUsedTime = System.currentTimeMillis();
        }

        Connection getConnection() {
            return connection;
        }

        long getLastUsedTime() {
            return lastUsedTime;
        }

        void updateLastUsedTime() {
            this.lastUsedTime = System.currentTimeMillis();
        }
    }

    static class PooledConnectionWrapper extends PooledConnection {
        private final ConnectionPool pool;
        private boolean closed = false;

        PooledConnectionWrapper(PooledConnection pooledConn, ConnectionPool pool) {
            super(pooledConn.getConnection());
            this.pool = pool;
        }

        @Override
        public void close() throws SQLException {
            if (!closed) {
                closed = true;
                pool.releaseConnection(this);
            }
        }
    }
}
```

### Connection Validation

```java
public class ConnectionValidator {
    private final long validationInterval;
    private final TimeUnit timeUnit;

    public ConnectionValidator(long validationInterval, TimeUnit timeUnit) {
        this.validationInterval = validationInterval;
        this.timeUnit = timeUnit;
    }

    public boolean validate(Connection connection) {
        try {
            if (connection == null || connection.isClosed()) {
                return false;
            }
            
            // Execute simple query to test connection
            try (Statement stmt = connection.createStatement()) {
                stmt.execute("SELECT 1");
            }
            return true;
        } catch (SQLException e) {
            return false;
        }
    }

    public boolean isValidWithTimeout(Connection connection, long timeout) {
        try {
            return connection.isValid((int) timeout);
        } catch (SQLException e) {
            return false;
        }
    }
}
```

### Connection Lifecycle

```
Connection Lifecycle:
┌─────────────┐
│   Created   │ ← Establish connection to database
└──────┬──────┘
       │
       ▼
┌─────────────┐
│    Idle     │ ← Available in pool, not in use
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Active    │ ← Borrowed by application, in use
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Validated  │ ← Check if still valid before reuse
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Closed    │ ← Connection closed, removed from pool
└─────────────┘
```

### Performance Optimization

| Optimization | Description | Impact |
|--------------|-------------|--------|
| Connection validation | Test connection before reuse | Prevents stale connections |
| Idle connection timeout | Close idle connections after timeout | Frees resources |
| Prepared statement cache | Reuse prepared statements | Reduces parsing overhead |
| Connection testing | Simple query to test connection | Ensures connection health |

### Configuration Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| maxPoolSize | Maximum connections in pool | 10 |
| minIdle | Minimum idle connections to maintain | 2 |
| connectionTimeout | Time to wait for connection | 30s |
| idleTimeout | Time before idle connection is closed | 10min |
| validationQuery | Query to test connection | SELECT 1 |
| validationInterval | Time between connection validations | 30s |

---

## Interviewer Insight
Interviewers want to see that you understand connection pool design: managing idle vs active connections, validation, thread safety, and resource cleanup. Strong candidates discuss connection lifecycle, validation strategies, and performance optimization techniques.

## Follow-up Questions
1. How do you handle database connection failures?
2. What is the difference between connection pooling and connection sharing?
3. How do you optimize connection pool performance?

## 📚 Related Topics
- [Thread Pool Design](./01-design-thread-pool.md)
- [Database Patterns](../../bonus/06-database-patterns/README.md)
- [Performance Optimization](../12-performance-optimization/README.md)

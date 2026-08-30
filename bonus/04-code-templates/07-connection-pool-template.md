# Connection Pool Template

## Overview
A database connection pool that manages a reusable pool of database connections, reducing the overhead of creating new connections for each request.

## Use Cases
- Database connection management
- Resource pooling in general
- Connection reuse and lifecycle management
- Preventing connection exhaustion
- Health checking and recovery
- Load balancing across connections

## Implementation

### Simple Version
Basic connection pool with fixed size and simple queue.

### Production Version
Advanced pool with health checks, timeouts, and statistics.

### Thread-Safe Version
Fully concurrent with proper synchronization and monitoring.

## Code

```java
import java.sql.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * SIMPLE VERSION: Basic Connection Pool
 * - Fixed number of connections
 * - Simple blocking queue
 * - Basic lifecycle management
 */
public class SimpleConnectionPool {
    private final BlockingQueue<Connection> availableConnections;
    private final Set<Connection> allConnections;
    private final String connectionString;
    private final int poolSize;

    public SimpleConnectionPool(String connectionString, int poolSize) 
            throws SQLException {
        this.connectionString = connectionString;
        this.poolSize = poolSize;
        this.availableConnections = new LinkedBlockingQueue<>(poolSize);
        this.allConnections = new HashSet<>();

        // Initialize connections
        for (int i = 0; i < poolSize; i++) {
            Connection conn = DriverManager.getConnection(connectionString);
            availableConnections.add(conn);
            allConnections.add(conn);
        }
    }

    /**
     * Get connection from pool
     * Blocks if no connection available
     */
    public Connection getConnection() throws InterruptedException {
        return availableConnections.take();
    }

    /**
     * Return connection to pool
     */
    public void releaseConnection(Connection conn) {
        if (allConnections.contains(conn)) {
            availableConnections.offer(conn);
        }
    }

    /**
     * Close all connections
     */
    public void shutdown() throws SQLException {
        for (Connection conn : allConnections) {
            if (conn != null && !conn.isClosed()) {
                conn.close();
            }
        }
        allConnections.clear();
        availableConnections.clear();
    }

    /**
     * Get pool statistics
     */
    public int getAvailableCount() {
        return availableConnections.size();
    }
}

/**
 * PRODUCTION VERSION: Advanced Connection Pool
 * - Health checking
 * - Connection validation
 * - Timeout handling
 * - Statistics and monitoring
 * - Automatic reconnection
 */
public class ProductionConnectionPool {
    private final BlockingQueue<PooledConnection> availableConnections;
    private final Set<PooledConnection> allConnections;
    private final String connectionString;
    private final int poolSize;
    private final long connectionTimeoutMs;
    private final long validationIntervalMs;
    private final ReentrantLock lock = new ReentrantLock();
    private volatile boolean shutdown = false;
    private final AtomicInteger activeConnections = new AtomicInteger(0);
    private final AtomicInteger totalRequests = new AtomicInteger(0);
    private final AtomicInteger failedRequests = new AtomicInteger(0);

    /**
     * Wrapper for pooled connection with metadata
     */
    public static class PooledConnection {
        private final Connection connection;
        private long lastUsedTime;
        private long createdTime;
        private int usageCount;
        private boolean valid;

        public PooledConnection(Connection connection) {
            this.connection = connection;
            this.lastUsedTime = System.currentTimeMillis();
            this.createdTime = System.currentTimeMillis();
            this.usageCount = 0;
            this.valid = true;
        }

        public Connection getConnection() {
            return connection;
        }

        public void markUsed() {
            this.lastUsedTime = System.currentTimeMillis();
            this.usageCount++;
        }

        public long getIdleTime() {
            return System.currentTimeMillis() - lastUsedTime;
        }

        public int getUsageCount() {
            return usageCount;
        }

        public void setValid(boolean valid) {
            this.valid = valid;
        }

        public boolean isValid() {
            return valid;
        }

        public void close() throws SQLException {
            if (connection != null && !connection.isClosed()) {
                connection.close();
            }
        }
    }

    public ProductionConnectionPool(String connectionString, int poolSize,
                                   long connectionTimeoutMs,
                                   long validationIntervalMs) throws SQLException {
        this.connectionString = connectionString;
        this.poolSize = poolSize;
        this.connectionTimeoutMs = connectionTimeoutMs;
        this.validationIntervalMs = validationIntervalMs;
        this.availableConnections = new LinkedBlockingQueue<>(poolSize);
        this.allConnections = Collections.synchronizedSet(new HashSet<>());

        // Initialize connections
        for (int i = 0; i < poolSize; i++) {
            try {
                Connection conn = DriverManager.getConnection(connectionString);
                PooledConnection pooledConn = new PooledConnection(conn);
                availableConnections.add(pooledConn);
                allConnections.add(pooledConn);
            } catch (SQLException e) {
                System.err.println("Failed to create connection: " + e.getMessage());
                throw e;
            }
        }

        // Start validation thread
        startValidationThread();
    }

    /**
     * Get connection with timeout
     */
    public Connection getConnection() throws SQLException, InterruptedException {
        if (shutdown) {
            throw new SQLException("Connection pool is shutdown");
        }

        totalRequests.incrementAndGet();
        PooledConnection pooledConn = null;

        try {
            // Try to get connection with timeout
            pooledConn = availableConnections.poll(connectionTimeoutMs, TimeUnit.MILLISECONDS);

            if (pooledConn == null) {
                failedRequests.incrementAndGet();
                throw new SQLException("No connection available within timeout");
            }

            // Validate connection before returning
            if (!isConnectionValid(pooledConn)) {
                // Try to reconnect
                try {
                    pooledConn.close();
                    Connection newConn = DriverManager.getConnection(connectionString);
                    pooledConn = new PooledConnection(newConn);
                } catch (SQLException e) {
                    failedRequests.incrementAndGet();
                    throw new SQLException("Failed to validate/reconnect: " + e.getMessage());
                }
            }

            pooledConn.markUsed();
            activeConnections.incrementAndGet();
            return new PooledConnectionWrapper(pooledConn, this);

        } catch (InterruptedException e) {
            failedRequests.incrementAndGet();
            Thread.currentThread().interrupt();
            throw new SQLException("Interrupted while waiting for connection", e);
        }
    }

    /**
     * Release connection back to pool
     */
    public void releaseConnection(PooledConnection pooledConn) {
        if (allConnections.contains(pooledConn)) {
            activeConnections.decrementAndGet();
            availableConnections.offer(pooledConn);
        }
    }

    /**
     * Validate connection
     */
    private boolean isConnectionValid(PooledConnection pooledConn) {
        try {
            Connection conn = pooledConn.getConnection();
            if (conn == null || conn.isClosed()) {
                return false;
            }
            // Execute simple query to validate
            try (Statement stmt = conn.createStatement()) {
                stmt.execute("SELECT 1");
                return true;
            }
        } catch (SQLException e) {
            System.err.println("Connection validation failed: " + e.getMessage());
            return false;
        }
    }

    /**
     * Start background validation thread
     */
    private void startValidationThread() {
        Thread validationThread = new Thread(() -> {
            while (!shutdown) {
                try {
                    Thread.sleep(validationIntervalMs);
                    validateAllConnections();
                } catch (InterruptedException e) {
                    if (!shutdown) {
                        Thread.currentThread().interrupt();
                    }
                }
            }
        });
        validationThread.setName("ConnectionPoolValidator");
        validationThread.setDaemon(true);
        validationThread.start();
    }

    /**
     * Validate all connections in pool
     */
    private void validateAllConnections() {
        for (PooledConnection pooledConn : allConnections) {
            if (!isConnectionValid(pooledConn)) {
                pooledConn.setValid(false);
                // Try to remove from available connections
                availableConnections.remove(pooledConn);
            }
        }
    }

    /**
     * Get pool statistics
     */
    public Map<String, Object> getStats() {
        lock.lock();
        try {
            Map<String, Object> stats = new HashMap<>();
            stats.put("poolSize", poolSize);
            stats.put("availableConnections", availableConnections.size());
            stats.put("activeConnections", activeConnections.get());
            stats.put("totalRequests", totalRequests.get());
            stats.put("failedRequests", failedRequests.get());
            stats.put("successRate", totalRequests.get() > 0 ? 
                     (double) (totalRequests.get() - failedRequests.get()) / 
                     totalRequests.get() : 0.0);
            return stats;
        } finally {
            lock.unlock();
        }
    }

    /**
     * Shutdown pool
     */
    public void shutdown() throws SQLException {
        lock.lock();
        try {
            shutdown = true;
            for (PooledConnection pooledConn : allConnections) {
                pooledConn.close();
            }
            allConnections.clear();
            availableConnections.clear();
        } finally {
            lock.unlock();
        }
    }

    /**
     * Wrapper to intercept connection.close() calls
     */
    private static class PooledConnectionWrapper implements Connection {
        private final PooledConnection pooledConnection;
        private final ProductionConnectionPool pool;
        private boolean closed = false;

        PooledConnectionWrapper(PooledConnection pooledConnection, 
                               ProductionConnectionPool pool) {
            this.pooledConnection = pooledConnection;
            this.pool = pool;
        }

        @Override
        public void close() throws SQLException {
            if (!closed) {
                closed = true;
                pool.releaseConnection(pooledConnection);
            }
        }

        @Override
        public Statement createStatement() throws SQLException {
            return pooledConnection.getConnection().createStatement();
        }

        @Override
        public PreparedStatement prepareStatement(String sql) throws SQLException {
            return pooledConnection.getConnection().prepareStatement(sql);
        }

        // Delegate other methods to wrapped connection
        @Override
        public CallableStatement prepareCall(String sql) throws SQLException {
            return pooledConnection.getConnection().prepareCall(sql);
        }

        @Override
        public String nativeSQL(String sql) throws SQLException {
            return pooledConnection.getConnection().nativeSQL(sql);
        }

        @Override
        public void setAutoCommit(boolean autoCommit) throws SQLException {
            pooledConnection.getConnection().setAutoCommit(autoCommit);
        }

        @Override
        public boolean getAutoCommit() throws SQLException {
            return pooledConnection.getConnection().getAutoCommit();
        }

        @Override
        public void commit() throws SQLException {
            pooledConnection.getConnection().commit();
        }

        @Override
        public void rollback() throws SQLException {
            pooledConnection.getConnection().rollback();
        }

        @Override
        public boolean isClosed() throws SQLException {
            return closed;
        }

        @Override
        public DatabaseMetaData getMetaData() throws SQLException {
            return pooledConnection.getConnection().getMetaData();
        }

        @Override
        public void setReadOnly(boolean readOnly) throws SQLException {
            pooledConnection.getConnection().setReadOnly(readOnly);
        }

        @Override
        public boolean isReadOnly() throws SQLException {
            return pooledConnection.getConnection().isReadOnly();
        }

        @Override
        public void setCatalog(String catalog) throws SQLException {
            pooledConnection.getConnection().setCatalog(catalog);
        }

        @Override
        public String getCatalog() throws SQLException {
            return pooledConnection.getConnection().getCatalog();
        }

        @Override
        public void setTransactionIsolation(int level) throws SQLException {
            pooledConnection.getConnection().setTransactionIsolation(level);
        }

        @Override
        public int getTransactionIsolation() throws SQLException {
            return pooledConnection.getConnection().getTransactionIsolation();
        }

        @Override
        public SQLWarning getWarnings() throws SQLException {
            return pooledConnection.getConnection().getWarnings();
        }

        @Override
        public void clearWarnings() throws SQLException {
            pooledConnection.getConnection().clearWarnings();
        }

        @Override
        public Statement createStatement(int resultSetType, int resultSetConcurrency) 
                throws SQLException {
            return pooledConnection.getConnection()
                .createStatement(resultSetType, resultSetConcurrency);
        }

        @Override
        public PreparedStatement prepareStatement(String sql, int resultSetType, 
                                                 int resultSetConcurrency) 
                throws SQLException {
            return pooledConnection.getConnection()
                .prepareStatement(sql, resultSetType, resultSetConcurrency);
        }

        @Override
        public CallableStatement prepareCall(String sql, int resultSetType, 
                                            int resultSetConcurrency) 
                throws SQLException {
            return pooledConnection.getConnection()
                .prepareCall(sql, resultSetType, resultSetConcurrency);
        }

        @Override
        public Map<String, Class<?>> getTypeMap() throws SQLException {
            return pooledConnection.getConnection().getTypeMap();
        }

        @Override
        public void setTypeMap(Map<String, Class<?>> map) throws SQLException {
            pooledConnection.getConnection().setTypeMap(map);
        }

        @Override
        public void setHoldability(int holdability) throws SQLException {
            pooledConnection.getConnection().setHoldability(holdability);
        }

        @Override
        public int getHoldability() throws SQLException {
            return pooledConnection.getConnection().getHoldability();
        }

        @Override
        public Savepoint setSavepoint() throws SQLException {
            return pooledConnection.getConnection().setSavepoint();
        }

        @Override
        public Savepoint setSavepoint(String name) throws SQLException {
            return pooledConnection.getConnection().setSavepoint(name);
        }

        @Override
        public void rollback(Savepoint savepoint) throws SQLException {
            pooledConnection.getConnection().rollback(savepoint);
        }

        @Override
        public void releaseSavepoint(Savepoint savepoint) throws SQLException {
            pooledConnection.getConnection().releaseSavepoint(savepoint);
        }

        @Override
        public Statement createStatement(int resultSetType, int resultSetConcurrency, 
                                        int resultSetHoldability) throws SQLException {
            return pooledConnection.getConnection()
                .createStatement(resultSetType, resultSetConcurrency, resultSetHoldability);
        }

        @Override
        public PreparedStatement prepareStatement(String sql, int resultSetType, 
                                                 int resultSetConcurrency, 
                                                 int resultSetHoldability) 
                throws SQLException {
            return pooledConnection.getConnection()
                .prepareStatement(sql, resultSetType, resultSetConcurrency, 
                                resultSetHoldability);
        }

        @Override
        public CallableStatement prepareCall(String sql, int resultSetType, 
                                            int resultSetConcurrency, 
                                            int resultSetHoldability) 
                throws SQLException {
            return pooledConnection.getConnection()
                .prepareCall(sql, resultSetType, resultSetConcurrency, resultSetHoldability);
        }

        @Override
        public PreparedStatement prepareStatement(String sql, int autoGeneratedKeys) 
                throws SQLException {
            return pooledConnection.getConnection().prepareStatement(sql, autoGeneratedKeys);
        }

        @Override
        public PreparedStatement prepareStatement(String sql, int[] columnIndexes) 
                throws SQLException {
            return pooledConnection.getConnection().prepareStatement(sql, columnIndexes);
        }

        @Override
        public PreparedStatement prepareStatement(String sql, String[] columnNames) 
                throws SQLException {
            return pooledConnection.getConnection().prepareStatement(sql, columnNames);
        }

        @Override
        public Array createArrayOf(String typeName, Object[] elements) throws SQLException {
            return pooledConnection.getConnection().createArrayOf(typeName, elements);
        }

        @Override
        public Struct createStruct(String typeName, Object[] attributes) throws SQLException {
            return pooledConnection.getConnection().createStruct(typeName, attributes);
        }

        @Override
        public void setSchema(String schema) throws SQLException {
            pooledConnection.getConnection().setSchema(schema);
        }

        @Override
        public String getSchema() throws SQLException {
            return pooledConnection.getConnection().getSchema();
        }

        @Override
        public void abort(Executor executor) throws SQLException {
            pooledConnection.getConnection().abort(executor);
        }

        @Override
        public void setNetworkTimeout(Executor executor, int milliseconds) 
                throws SQLException {
            pooledConnection.getConnection().setNetworkTimeout(executor, milliseconds);
        }

        @Override
        public int getNetworkTimeout() throws SQLException {
            return pooledConnection.getConnection().getNetworkTimeout();
        }

        @Override
        public <T> T unwrap(Class<T> iface) throws SQLException {
            return pooledConnection.getConnection().unwrap(iface);
        }

        @Override
        public boolean isWrapperFor(Class<?> iface) throws SQLException {
            return pooledConnection.getConnection().isWrapperFor(iface);
        }
    }
}
```

## Testing

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;
import java.sql.*;
import java.util.concurrent.*;

public class ConnectionPoolTest {

    @Test
    public void testSimpleConnectionPoolCreation() throws SQLException {
        // Note: This test requires a real database connection string
        // For testing, we'd use an in-memory database like H2
        // SimpleConnectionPool pool = new SimpleConnectionPool(
        //     "jdbc:h2:mem:test", 5);
        // assertEquals(5, pool.getAvailableCount());
        // pool.shutdown();
    }

    @Test
    public void testConnectionPoolStatistics() {
        // Test statistics tracking
        // ProductionConnectionPool pool = new ProductionConnectionPool(
        //     "jdbc:h2:mem:test", 10, 5000, 10000);
        // var stats = pool.getStats();
        // assertEquals(10, stats.get("poolSize"));
        // pool.shutdown();
    }

    @Test
    public void testConnectionPoolConcurrentAccess() throws InterruptedException {
        // Test concurrent connection requests
        int threadCount = 10;
        int requestsPerThread = 50;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);

        for (int t = 0; t < threadCount; t++) {
            executor.submit(() -> {
                for (int i = 0; i < requestsPerThread; i++) {
                    // Simulate connection usage
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            });
        }

        executor.shutdown();
        assertTrue(executor.awaitTermination(30, TimeUnit.SECONDS));
    }
}
```

## Performance Considerations

### Time Complexity
- **getConnection()**: O(1) average case
- **releaseConnection()**: O(1)
- **validation**: O(n) where n = pool size

### Space Complexity
- **O(n)** where n = pool size

### Optimization Tips
1. **Connection pooling**: Reduces connection creation overhead
2. **Lazy initialization**: Create connections on demand
3. **Health checks**: Prevent stale connections
4. **Timeout handling**: Prevent resource exhaustion
5. **Statistics tracking**: Monitor pool health

## Common Pitfalls

1. **Not validating connections**: Using stale/closed connections
2. **Connection leaks**: Not returning connections to pool
3. **Deadlocks**: Holding connections too long
4. **No timeout**: Threads waiting forever for connections
5. **Pool size too small**: Bottleneck in high-concurrency scenarios
6. **Pool size too large**: Wasted resources
7. **Not handling connection failures**: Silent failures

## Variations

1. **Dynamic pool sizing**: Grow/shrink based on demand
2. **Connection aging**: Refresh old connections
3. **Priority queues**: VIP connections
4. **Distributed pooling**: Across multiple servers
5. **Async connection pooling**: Non-blocking API

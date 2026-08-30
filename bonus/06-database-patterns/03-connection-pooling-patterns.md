# Connection Pooling Patterns

## Overview

Connection pooling is a fundamental pattern for managing database connections efficiently. Instead of creating a new connection for each database operation (which is expensive), a pool maintains a set of reusable connections that applications can borrow and return.

## Why Connection Pooling Matters

Creating a new database connection is a resource-intensive operation:

1. **Network Overhead** - TCP handshake, authentication
2. **Database Resources** - Session creation, memory allocation
3. **Time Cost** - Typically 10-100ms per connection
4. **Resource Limits** - Databases have max connection limits

Without pooling, a high-traffic application would:
- Exhaust database connection limits
- Suffer from high latency
- Waste resources on connection setup
- Potentially crash under load

With pooling, connections are reused, reducing overhead by **10-100x**.

## Connection Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                     Connection Pool                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Idle Connections (Available for Use)                     │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐         │  │
│  │  │ C1  │ │ C2  │ │ C3  │ │ C4  │ │ C5  │ │ C6  │ ...    │  │
│  │  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘         │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Active Connections (In Use by Application)              │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐                                 │  │
│  │  │ C7  │ │ C8  │ │ C9  │ ...                             │  │
│  │  └─────┘ └─────┘ └─────┘                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

Application Flow:
1. Request Connection → Pool assigns idle connection
2. Execute Query → Connection used for DB operations
3. Return Connection → Connection validated and returned to idle pool
4. Validation → Check connection health before reuse
```

## HikariCP - The Gold Standard

HikariCP is the fastest, most lightweight connection pool for Java. It's the default in Spring Boot 2.x+.

### Key Features

- **Extreme Performance** - ~25x faster than DBCP, 5x faster than Tomcat JDBC
- **Small Footprint** - ~130KB JAR
- **Smart Features** - Connection validation, leak detection, metrics
- **Zero Dependencies** - No external libraries required

### Spring Boot Configuration

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: myuser
    password: mypassword
    driver-class-name: org.postgresql.Driver
    hikari:
      # Core pool settings
      maximum-pool-size: 20              # Max connections in pool
      minimum-idle: 5                    # Min idle connections ready
      connection-timeout: 30000          # ms to wait for connection (30s)
      idle-timeout: 600000               # ms before idle connection closed (10min)
      max-lifetime: 1800000              # ms before connection closed (30min)
      
      # Connection validation
      validation-timeout: 5000           # ms to validate connection (5s)
      connection-test-query: SELECT 1    # Simple validation query
      # OR use JDBC4 validation (preferred)
      connection-init-sql: SET TIME ZONE 'UTC'
      
      # Leak detection
      leak-detection-threshold: 60000    # Detect leaks after 60s
      
      # Performance tuning
      pool-name: MyAppHikariPool
      allow-pool-suspension: false
      read-only: false
      register-mbeans: true              # Enable JMX monitoring
      
      # Connection properties
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
        useServerPrepStmts: true
        useLocalSessionState: true
        rewriteBatchedStatements: true
        cacheResultSetMetadata: true
        cacheServerConfiguration: true
        elideSetAutoCommits: true
        maintainTimeStats: false
```

### Programmatic Configuration

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import javax.sql.DataSource;

@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        
        // Basic connection settings
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("myuser");
        config.setPassword("mypassword");
        config.setDriverClassName("org.postgresql.Driver");
        
        // Pool sizing
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        
        // Timeouts
        config.setConnectionTimeout(30000);      // 30 seconds
        config.setIdleTimeout(600000);           // 10 minutes
        config.setMaxLifetime(1800000);          // 30 minutes
        config.setValidationTimeout(5000);       // 5 seconds
        
        // Connection validation
        config.setConnectionTestQuery("SELECT 1");
        // OR for JDBC4 drivers:
        config.setConnectionTestQuery(null);
        
        // Leak detection
        config.setLeakDetectionThreshold(60000); // 60 seconds
        
        // Pool name for monitoring
        config.setPoolName("MyAppHikariPool");
        
        // JMX for monitoring
        config.setRegisterMbeans(true);
        
        // Connection properties
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");
        config.addDataSourceProperty("useLocalSessionState", "true");
        config.addDataSourceProperty("rewriteBatchedStatements", "true");
        config.addDataSourceProperty("cacheResultSetMetadata", "true");
        config.addDataSourceProperty("cacheServerConfiguration", "true");
        config.addDataSourceProperty("elideSetAutoCommits", "true");
        config.addDataSourceProperty("maintainTimeStats", "false");
        
        return new HikariDataSource(config);
    }
}
```

### Dynamic Pool Sizing

```java
@Service
public class DynamicPoolService {

    @Autowired
    private HikariDataSource dataSource;

    // Adjust pool size based on load
    @Scheduled(fixedRate = 60000) // Every minute
    public void adjustPoolSize() {
        HikariPoolMXBean poolProxy = dataSource.getHikariPoolMXBean();
        
        int activeConnections = poolProxy.getActiveConnections();
        int idleConnections = poolProxy.getIdleConnections();
        int totalConnections = poolProxy.getTotalConnections();
        int waitingThreads = poolProxy.getThreadsAwaitingConnection();
        
        // Scale up if threads are waiting
        if (waitingThreads > 5 && totalConnections < 50) {
            dataSource.setMaximumPoolSize(Math.min(totalConnections + 5, 50));
            log.info("Scaled up pool to {}", dataSource.getMaximumPoolSize());
        }
        
        // Scale down if idle connections are high
        if (idleConnections > 15 && totalConnections > 10) {
            dataSource.setMaximumPoolSize(Math.max(totalConnections - 5, 10));
            log.info("Scaled down pool to {}", dataSource.getMaximumPoolSize());
        }
    }
}
```

## Apache Druid - Feature-Rich Pool

Druid is a production-ready database connection pool with extensive monitoring capabilities.

### Key Features

- **Monitoring** - Built-in metrics, SQL logging, slow query detection
- **Security** - SQL injection detection, password encryption
- **Extensibility** - Filter chain for custom logic
- **High Performance** - Competitive with HikariCP

### Spring Boot Configuration

```yaml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/mydb?useSSL=false
    username: myuser
    password: mypassword
    druid:
      # Pool settings
      initial-size: 5
      min-idle: 5
      max-active: 20
      max-wait: 60000
      
      # Validation
      validation-query: SELECT 1
      test-while-idle: true
      test-on-borrow: false
      test-on-return: false
      
      # Timeouts
      time-between-eviction-runs-millis: 60000
      min-evictable-idle-time-millis: 300000
      max-evictable-idle-time-millis: 900000
      
      # Filters
      filters: stat,wall,slf4j
      
      # Stat view (monitoring console)
      stat-view-servlet:
        enabled: true
        url-pattern: /druid/*
        reset-enable: false
        login-username: admin
        login-password: admin123
        
      # Web stat filter
      web-stat-filter:
        enabled: true
        url-pattern: /*
        exclusions: "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*"
        
      # Wall filter (SQL injection protection)
      filter:
        wall:
          enabled: true
          config:
            multi-statement-allow: false
```

### Programmatic Configuration

```java
@Configuration
public class DruidConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.druid")
    public DataSource druidDataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        
        // Basic settings
        dataSource.setUrl("jdbc:mysql://localhost:3306/mydb");
        dataSource.setUsername("myuser");
        dataSource.setPassword("mypassword");
        dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        
        // Pool sizing
        dataSource.setInitialSize(5);
        dataSource.setMinIdle(5);
        dataSource.setMaxActive(20);
        dataSource.setMaxWait(60000);
        
        // Validation
        dataSource.setValidationQuery("SELECT 1");
        dataSource.setTestWhileIdle(true);
        dataSource.setTestOnBorrow(false);
        dataSource.setTestOnReturn(false);
        
        // Eviction
        dataSource.setTimeBetweenEvictionRunsMillis(60000);
        dataSource.setMinEvictableIdleTimeMillis(300000);
        dataSource.setMaxEvictableIdleTimeMillis(900000);
        
        // Filters
        try {
            dataSource.setFilters("stat,wall,slf4j");
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
        
        // Connection properties
        dataSource.setConnectionProperties("druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000");
        
        return dataSource;
    }

    @Bean
    public ServletRegistrationBean<StatViewServlet> druidStatViewServlet() {
        ServletRegistrationBean<StatViewServlet> registrationBean = 
            new ServletRegistrationBean<>(new StatViewServlet(), "/druid/*");
        registrationBean.addInitParameter("loginUsername", "admin");
        registrationBean.addInitParameter("loginPassword", "admin123");
        registrationBean.addInitParameter("resetEnable", "false");
        return registrationBean;
    }

    @Bean
    public FilterRegistrationBean<WebStatFilter> druidWebStatFilter() {
        FilterRegistrationBean<WebStatFilter> filterRegistrationBean = 
            new FilterRegistrationBean<>(new WebStatFilter());
        filterRegistrationBean.addUrlPatterns("/*");
        filterRegistrationBean.addInitParameter("exclusions", 
            "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*");
        return filterRegistrationBean;
    }
}
```

## Custom Connection Pool

Sometimes you need a custom pool for specific requirements.

### Basic Custom Pool Implementation

```java
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class CustomConnectionPool implements DataSource {

    private final BlockingQueue<PooledConnection> idleConnections;
    private final AtomicInteger activeConnections = new AtomicInteger(0);
    private final AtomicInteger totalConnections = new AtomicInteger(0);
    private final String url;
    private final String username;
    private final String password;
    private final int maxPoolSize;
    private final long connectionTimeout;

    public CustomConnectionPool(String url, String username, String password, 
                                 int maxPoolSize, long connectionTimeout) {
        this.url = url;
        this.username = username;
        this.password = password;
        this.maxPoolSize = maxPoolSize;
        this.connectionTimeout = connectionTimeout;
        this.idleConnections = new LinkedBlockingQueue<>(maxPoolSize);
    }

    @Override
    public Connection getConnection() throws SQLException {
        return getConnection(username, password);
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        // Try to get idle connection
        PooledConnection pooled = idleConnections.poll();
        
        if (pooled != null) {
            if (validateConnection(pooled)) {
                activeConnections.incrementAndGet();
                return pooled;
            } else {
                // Connection is stale, close it
                closeConnection(pooled);
                totalConnections.decrementAndGet();
            }
        }
        
        // No idle connection available, try to create new one
        if (totalConnections.get() < maxPoolSize) {
            if (totalConnections.incrementAndGet() <= maxPoolSize) {
                Connection conn = createNewConnection();
                activeConnections.incrementAndGet();
                return new PooledConnection(conn, this);
            } else {
                totalConnections.decrementAndGet(); // Race condition
            }
        }
        
        // Wait for connection to become available
        try {
            pooled = idleConnections.poll(connectionTimeout, TimeUnit.MILLISECONDS);
            if (pooled == null) {
                throw new SQLException("Connection timeout after " + connectionTimeout + "ms");
            }
            if (!validateConnection(pooled)) {
                closeConnection(pooled);
                totalConnections.decrementAndGet();
                return getConnection(username, password); // Retry
            }
            activeConnections.incrementAndGet();
            return pooled;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new SQLException("Interrupted while waiting for connection", e);
        }
    }

    private Connection createNewConnection() throws SQLException {
        return DriverManager.getConnection(url, username, password);
    }

    private boolean validateConnection(PooledConnection pooled) {
        try {
            return pooled.isValid(2); // 2 second timeout
        } catch (SQLException e) {
            return false;
        }
    }

    private void closeConnection(PooledConnection pooled) {
        try {
            pooled.getDelegate().close();
        } catch (SQLException e) {
            // Log error
        }
    }

    void returnConnection(PooledConnection pooled) {
        activeConnections.decrementAndGet();
        if (validateConnection(pooled)) {
            idleConnections.offer(pooled);
        } else {
            closeConnection(pooled);
            totalConnections.decrementAndGet();
        }
    }

    // Pool statistics
    public int getActiveConnections() {
        return activeConnections.get();
    }

    public int getIdleConnections() {
        return idleConnections.size();
    }

    public int getTotalConnections() {
        return totalConnections.get();
    }

    // Other DataSource methods...
    @Override
    public PrintWriter getLogWriter() throws SQLException { return null; }
    @Override
    public void setLogWriter(PrintWriter out) throws SQLException {}
    @Override
    public void setLoginTimeout(int seconds) throws SQLException {}
    @Override
    public int getLoginTimeout() throws SQLException { return 0; }
    @Override
    public java.util.logging.Logger getParentLogger() throws SQLFeatureNotSupportedException { 
        return null; 
    }
    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException { return null; }
    @Override
    public boolean isWrapperFor(Class<?> iface) throws SQLException { return false; }

    // Inner class for pooled connection
    private static class PooledConnection implements Connection {
        private final Connection delegate;
        private final CustomConnectionPool pool;
        private boolean closed = false;

        PooledConnection(Connection delegate, CustomConnectionPool pool) {
            this.delegate = delegate;
            this.pool = pool;
        }

        Connection getDelegate() {
            return delegate;
        }

        @Override
        public void close() throws SQLException {
            if (!closed) {
                closed = true;
                pool.returnConnection(this);
            }
        }

        @Override
        public boolean isClosed() throws SQLException {
            return closed || delegate.isClosed();
        }

        // Delegate all other methods to underlying connection
        @Override
        public Statement createStatement() throws SQLException {
            checkNotClosed();
            return delegate.createStatement();
        }

        @Override
        public PreparedStatement prepareStatement(String sql) throws SQLException {
            checkNotClosed();
            return delegate.prepareStatement(sql);
        }

        // ... implement all other Connection methods by delegating ...
        
        private void checkNotClosed() throws SQLException {
            if (closed) {
                throw new SQLException("Connection is closed");
            }
        }
    }
}
```

## Pool Sizing Strategies

### Formula-Based Sizing

```java
// Common formula: connections = ((core_count * 2) + effective_spindle_count)
// For CPU-bound: connections = core_count * 2
// For I/O-bound: connections = core_count * 2 + spindle_count

public class PoolSizeCalculator {

    public static int calculateOptimalPoolSize() {
        int cores = Runtime.getRuntime().availableProcessors();
        
        // CPU-bound application
        int cpuBound = cores * 2;
        
        // I/O-bound application (with waiting)
        int ioBound = cores * (1 + waitTime / computeTime);
        
        // Conservative estimate
        return Math.max(cpuBound, ioBound);
    }

    // For example: 8 cores, 80% wait time
    // ioBound = 8 * (1 + 0.8 / 0.2) = 8 * 5 = 40 connections
}
```

### Dynamic Sizing Based on Metrics

```java
@Service
public class AdaptivePoolSizing {

    @Autowired
    private HikariDataSource dataSource;

    @Autowired
    private MeterRegistry meterRegistry;

    @Scheduled(fixedRate = 30000) // Every 30 seconds
    public void adjustPoolSize() {
        HikariPoolMXBean pool = dataSource.getHikariPoolMXBean();
        
        // Get metrics
        double activeConnections = pool.getActiveConnections();
        double idleConnections = pool.getIdleConnections();
        double waitingThreads = pool.getThreadsAwaitingConnection();
        double totalConnections = pool.getTotalConnections();
        
        // Calculate utilization
        double utilization = activeConnections / totalConnections;
        
        // Get application metrics
        double requestRate = meterRegistry.counter("http.requests").count();
        double avgResponseTime = meterRegistry.timer("http.requests").mean(TimeUnit.MILLISECONDS);
        
        // Scaling decisions
        int currentMax = dataSource.getMaximumPoolSize();
        int newMax = currentMax;
        
        // Scale up if high utilization and threads waiting
        if (utilization > 0.8 && waitingThreads > 2 && currentMax < 100) {
            newMax = Math.min(currentMax + 5, 100);
            log.info("Scaling up: {} -> {} (utilization: {}, waiting: {})", 
                     currentMax, newMax, utilization, waitingThreads);
        }
        
        // Scale down if low utilization
        if (utilization < 0.3 && idleConnections > 10 && currentMax > 10) {
            newMax = Math.max(currentMax - 2, 10);
            log.info("Scaling down: {} -> {} (utilization: {}, idle: {})", 
                     currentMax, newMax, utilization, idleConnections);
        }
        
        if (newMax != currentMax) {
            dataSource.setMaximumPoolSize(newMax);
        }
    }
}
```

## Connection Validation Strategies

### JDBC4 Validation (Preferred)

```java
// HikariCP with JDBC4 validation
config.setConnectionTestQuery(null); // Disable simple query
config.setValidationTimeout(5000);

// This uses Connection.isValid() which is efficient
// Most modern drivers support this
```

### Custom Query Validation

```java
// For older drivers or specific validation needs
config.setConnectionTestQuery("SELECT 1");

// PostgreSQL-specific
config.setConnectionTestQuery("SELECT 1");

// MySQL-specific
config.setConnectionTestQuery("SELECT 1");

// Oracle-specific
config.setConnectionTestQuery("SELECT 1 FROM DUAL");
```

### Validation on Borrow vs Idle

```java
// Validate when borrowing connection (safer, slower)
config.setConnectionTestQuery("SELECT 1");
config.setTestOnBorrow(true);

// Validate while idle (faster, less safe)
config.setTestWhileIdle(true);
config.setTimeBetweenEvictionRunsMillis(60000);

// Best practice: testWhileIdle + short validation timeout
config.setTestWhileIdle(true);
config.setValidationTimeout(1000); // 1 second
```

## Leak Detection and Prevention

### HikariCP Leak Detection

```java
// Enable leak detection
config.setLeakDetectionThreshold(60000); // 60 seconds

// This will log a warning if connection is held longer than threshold
// Useful for finding connection leaks in code
```

### Programmatic Leak Detection

```java
@Aspect
@Component
public class ConnectionLeakDetector {

    @Around("@annotation(org.springframework.transaction.annotation.Transactional)")
    public Object detectLongTransactions(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return joinPoint.proceed();
        } finally {
            long duration = System.currentTimeMillis() - start;
            if (duration > 5000) { // 5 seconds
                log.warn("Long transaction detected: {} took {}ms", 
                         joinPoint.getSignature(), duration);
            }
        }
    }
}
```

## Performance Comparison

### Benchmark Results (Approximate)

| Pool | Connections/sec | Memory | Startup Time |
|------|-----------------|--------|--------------|
| HikariCP | 1,000,000+ | 130KB | ~50ms |
| Tomcat JDBC | 500,000 | 400KB | ~100ms |
| DBCP2 | 300,000 | 350KB | ~150ms |
| Druid | 400,000 | 2MB | ~200ms |

### When to Use Which Pool

- **HikariCP** - Default choice, best performance
- **Druid** - Need monitoring, SQL logging, security features
- **Tomcat JDBC** - Running in Tomcat, need compatibility
- **DBCP2** - Legacy applications, need stability

## Common Pitfalls

### 1. Pool Too Small

```java
// ❌ Bad: Pool too small for load
config.setMaximumPoolSize(5); // Only 5 connections for high traffic

// Result: Connection timeouts, poor performance
```

### 2. Pool Too Large

```java
// ❌ Bad: Pool too large
config.setMaximumPoolSize(1000); // Too many for database

// Result: Database overwhelmed, context switching overhead
```

### 3. No Connection Validation

```java
// ❌ Bad: No validation
config.setTestWhileIdle(false);
config.setConnectionTestQuery(null);

// Result: Stale connections cause errors
```

### 4. Long Max Lifetime

```java
// ❌ Bad: Connections live too long
config.setMaxLifetime(18000000); // 5 hours

// Result: Stale connections, memory leaks
```

### 5. Not Closing Connections

```java
// ❌ Bad: Not closing connection
public User getUser(Long id) {
    Connection conn = dataSource.getConnection();
    // Forgot to close!
}

// Result: Connection leak, pool exhaustion
```

## Best Practices

1. **Use HikariCP** as default pool
2. **Size pool appropriately** based on load testing
3. **Enable validation** with testWhileIdle
4. **Set reasonable timeouts** (30s connection, 30min max lifetime)
5. **Monitor pool metrics** via JMX or Micrometer
6. **Enable leak detection** during development
7. **Use connection properties** for driver optimization
8. **Test under load** to validate pool configuration

## Interview Questions

1. Why is connection pooling necessary?
2. How do you size a connection pool?
3. What's the difference between HikariCP and Druid?
4. How do you detect connection leaks?
5. What happens when the pool is exhausted?
6. How does connection validation work?
7. When would you use a custom connection pool?

## Related Topics

- [Transaction Management](./transaction-management.md)
- [ORM Patterns](./orm-patterns.md)
- [Query Optimization](./query-optimization.md)

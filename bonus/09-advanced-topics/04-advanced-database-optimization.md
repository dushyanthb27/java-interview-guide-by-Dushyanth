# Advanced Database Optimization Deep-Dive

## 📋 Question
Explain advanced database optimization techniques including connection pooling strategies, query optimization, indexing strategies, and ORM performance tuning. How do you identify and fix N+1 query problems? Provide real-world examples and best practices.

## Difficulty: 🔴 Expert

---

## 💡 Answer

Database performance is often the bottleneck in Java applications. Advanced optimization requires understanding connection pooling, query execution plans, indexing strategies, and ORM internals.

### Connection Pooling Strategies

Connection pooling reduces the overhead of establishing database connections by reusing existing connections.

**HikariCP Configuration:**

```java
import com.zaxxer.hikari.*;
import javax.sql.DataSource;

@Configuration
public class DataSourceConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("user");
        config.setPassword("password");
        
        // Pool sizing
        config.setMaximumPoolSize(20);  // Based on formula: (cores * 2) + effective_spindle_count
        config.setMinimumIdle(5);      // Minimum idle connections
        
        // Connection timeout
        config.setConnectionTimeout(30000);  // 30 seconds
        config.setIdleTimeout(600000);      // 10 minutes
        config.setMaxLifetime(1800000);      // 30 minutes
        
        // Performance tuning
        config.setConnectionTestQuery("SELECT 1");
        config.setValidationTimeout(5000);
        config.setLeakDetectionThreshold(60000);  // Detect connection leaks
        
        return new HikariDataSource(config);
    }
}
```

**Connection Pool Sizing Formula:**

```
Optimal pool size = ((core_count * 2) + effective_spindle_count)

Where:
- core_count = number of CPU cores
- effective_spindle_count = number of disk spindles (for I/O bound workloads)

Example: 8-core machine with SSD (high I/O):
Pool size = (8 * 2) + 1 = 17 connections
```

**Monitoring Connection Pool:**

```java
@Component
public class ConnectionPoolMonitor {
    
    @Autowired
    private DataSource dataSource;
    
    @Scheduled(fixedRate = 60000)  // Every minute
    public void monitorPool() {
        if (dataSource instanceof HikariDataSource) {
            HikariPoolMXBean pool = ((HikariDataSource) dataSource).getHikariPoolMXBean();
            
            System.out.println("=== Connection Pool Stats ===");
            System.out.println("Active connections: " + pool.getActiveConnections());
            System.out.println("Idle connections: " + pool.getIdleConnections());
            System.out.println("Total connections: " + pool.getTotalConnections());
            System.out.println("Threads awaiting: " + pool.getThreadsAwaitingConnection());
            System.out.println("Total connections: " + pool.getTotalConnections());
        }
    }
}
```

### Query Optimization

**Explain Plan Analysis:**

```sql
-- PostgreSQL explain plan
EXPLAIN ANALYZE
SELECT u.id, u.name, o.id as order_id, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
  AND o.status = 'completed';

-- Output shows:
-- - Seq Scan vs Index Scan
-- - Cost estimates
-- - Actual execution time
-- - Rows processed
```

**Index Usage Strategies:**

```sql
-- B-tree index (default, good for equality and range queries)
CREATE INDEX idx_users_email ON users(email);

-- Composite index (order matters!)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
-- Good for: WHERE user_id = ? AND status = ?
-- Bad for: WHERE status = ? (user_id not used)

-- Partial index (smaller, faster)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- Covering index (includes extra columns to avoid table lookup)
CREATE INDEX idx_orders_covering ON orders(user_id, status) INCLUDE (total, created_at);

-- Hash index (equality only, faster than B-tree for exact matches)
CREATE INDEX idx_hash_email ON users USING HASH(email);

-- GIN index (for array/jsonb columns)
CREATE INDEX idx_users_tags ON users USING GIN(tags);
```

**N+1 Query Problem:**

```java
// ❌ N+1 problem: Executes 1 query for users + N queries for orders
@Entity
public class User {
    @Id
    private Long id;
    private String name;
    
    @OneToMany(mappedBy = "user")
    private List<Order> orders;  // Lazy loaded
}

// This triggers N+1 queries
List<User> users = userRepository.findAll();
for (User user : users) {
    // Each access triggers a SELECT for orders
    System.out.println(user.getOrders().size());
}

// ✅ Solution 1: JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();

// ✅ Solution 2: @EntityGraph
@EntityGraph(attributePaths = {"orders"})
List<User> findAll();

// ✅ Solution 3: Batch fetching
@BatchSize(size = 100)
@OneToMany(mappedBy = "user")
private List<Order> orders;

// ✅ Solution 4: DTO projection (no lazy loading)
public interface UserOrderSummary {
    Long getId();
    String getName();
    Long getOrderCount();
}

@Query("SELECT u.id as id, u.name as name, COUNT(o) as orderCount " +
       "FROM User u LEFT JOIN u.orders o GROUP BY u.id")
List<UserOrderSummary> findSummaries();
```

### Hibernate Performance Tuning

**Batch Insert Optimization:**

```java
// ❌ Bad: Inserts one by one
for (int i = 0; i < 10000; i++) {
    User user = new User("user-" + i);
    entityManager.persist(user);
}

// ✅ Good: Batch insert with JDBC batch size
@Configuration
public class HibernateConfig {
    
    @Bean
    public JpaProperties jpaProperties() {
        Map<String, String> properties = new HashMap<>();
        properties.put("hibernate.jdbc.batch_size", "50");
        properties.put("hibernate.order_inserts", "true");
        properties.put("hibernate.order_updates", "true");
        properties.put("hibernate.jdbc.batch_versioned_data", "true");
        return new JpaProperties(properties);
    }
}

// ✅ Even better: StatelessSession for bulk operations
StatelessSession session = entityManagerFactory.unwrap(SessionFactory.class)
    .openStatelessSession();
Transaction tx = session.beginTransaction();

for (int i = 0; i < 10000; i++) {
    User user = new User("user-" + i);
    session.insert(user);
    
    if (i % 50 == 0) {
        session.flush();  // Flush every 50 records
        session.clear();   // Clear to avoid memory issues
    }
}

tx.commit();
session.close();
```

**Second-Level Cache:**

```java
// Enable second-level cache with Ehcache
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CachingProvider cachingProvider = Caching.getCachingProvider();
        CacheManager cacheManager = cachingProvider.getCacheManager();
        
        return cacheManager;
    }
}

@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class User {
    @Id
    private Long id;
    private String name;
    
    @Cacheable
    @org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}

// Query cache
@QueryHints({
    @QueryHint(name = "org.hibernate.cacheable", value = "true"),
    @QueryHint(name = "org.hibernate.cacheRegion", value = "userQueries")
})
@Query("SELECT u FROM User u WHERE u.name = :name")
User findByName(@Param("name") String name);
```

### Database-Specific Optimizations

**PostgreSQL Optimizations:**

```sql
-- Enable parallel query
SET max_parallel_workers_per_gather = 4;
SET max_parallel_workers = 8;
SET max_worker_processes = 16;

-- Table partitioning (for large tables)
CREATE TABLE orders (
    id BIGSERIAL,
    user_id BIGINT,
    created_at TIMESTAMP,
    status VARCHAR(50)
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Materialized view for complex aggregations
CREATE MATERIALIZED VIEW user_order_stats AS
SELECT user_id, 
       COUNT(*) as total_orders,
       SUM(total) as total_spent,
       AVG(total) as avg_order_value
FROM orders
GROUP BY user_id;

-- Refresh materialized view
REFRESH MATERIALIZED VIEW user_order_stats;

-- Concurrent refresh (doesn't block reads)
REFRESH MATERIALIZED VIEW CONCURRENTLY user_order_stats;
```

**MySQL Optimizations:**

```sql
-- InnoDB buffer pool sizing (should be 70-80% of RAM for dedicated DB server)
SET GLOBAL innodb_buffer_pool_size = 8589934592;  -- 8GB

-- Enable query cache (MySQL 5.7 and earlier)
SET GLOBAL query_cache_size = 67108864;  -- 64MB
SET GLOBAL query_cache_type = ON;

-- Optimize JOIN operations
-- Ensure JOIN columns have same data types and are indexed

-- Use EXPLAIN to analyze queries
EXPLAIN SELECT * FROM users u JOIN orders o ON u.id = o.user_id;
```

### Code Example: Complete Database Optimization Demo

```java
package com.interview.advanced.database;

import org.springframework.boot.*;
import org.springframework.boot.autoconfigure.*;
import org.springframework.context.annotation.*;
import org.springframework.beans.factory.annotation.*;
import org.springframework.stereotype.*;
import org.springframework.transaction.annotation.*;
import org.springframework.data.jpa.repository.*;
import org.springframework.cache.annotation.*;
import com.zaxxer.hikari.*;
import javax.persistence.*;
import java.util.*;

// === ENTITY DEFINITIONS ===
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = org.hibernate.annotations.CacheConcurrencyStrategy.READ_WRITE)
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String email;
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    @org.hibernate.annotations.BatchSize(size = 100)
    private List<Order> orders = new ArrayList<>();
    
    // Getters and setters
}

@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String status;
    private Double total;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    
    // Getters and setters
}

// === REPOSITORIES ===
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    // ✅ JOIN FETCH to avoid N+1
    @Query("SELECT u FROM User u JOIN FETCH u.orders")
    List<User> findAllWithOrders();
    
    // ✅ EntityGraph for selective loading
    @EntityGraph(attributePaths = {"orders"})
    List<User> findAll();
    
    // ✅ DTO projection for read-only queries
    @Query("SELECT u.id as id, u.name as name, COUNT(o) as orderCount " +
           "FROM User u LEFT JOIN u.orders o GROUP BY u.id")
    List<UserSummary> findSummaries();
}

interface UserSummary {
    Long getId();
    String getName();
    Long getOrderCount();
}

// === SERVICE ===
@Service
@Transactional
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EntityManager entityManager;
    
    // ✅ Batch insert
    public void batchInsertUsers(List<User> users) {
        for (int i = 0; i < users.size(); i++) {
            entityManager.persist(users.get(i));
            
            if (i % 50 == 0) {
                entityManager.flush();
                entityManager.clear();
            }
        }
    }
    
    // ✅ Avoid N+1 with JOIN FETCH
    public List<User> getUsersWithOrders() {
        return userRepository.findAllWithOrders();
    }
    
    // ✅ Use DTO projection for reporting
    public List<UserSummary> getUserSummaries() {
        return userRepository.findSummaries();
    }
    
    // ✅ Cached read operation
    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    // ✅ Cache eviction on update
    @CacheEvict(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }
}

// === DATA SOURCE CONFIG ===
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("user");
        config.setPassword("password");
        
        // Pool sizing based on formula
        config.setMaximumPoolSize(17);  // (8 cores * 2) + 1
        config.setMinimumIdle(5);
        
        // Timeouts
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        
        // Leak detection
        config.setLeakDetectionThreshold(60000);
        
        return new HikariDataSource(config);
    }
    
    @Bean
    public JpaProperties jpaProperties() {
        Map<String, String> properties = new HashMap<>();
        properties.put("hibernate.jdbc.batch_size", "50");
        properties.put("hibernate.order_inserts", "true");
        properties.put("hibernate.order_updates", "true");
        properties.put("hibernate.generate_statistics", "true");
        properties.put("hibernate.jdbc.batch_versioned_data", "true");
        return new JpaProperties(properties);
    }
}

// === CONNECTION POOL MONITOR ===
@Component
public class PoolMonitor {
    
    @Autowired
    private DataSource dataSource;
    
    @Scheduled(fixedRate = 60000)
    public void monitor() {
        if (dataSource instanceof HikariDataSource) {
            HikariPoolMXBean pool = ((HikariDataSource) dataSource).getHikariPoolMXBean();
            System.out.printf("Pool: active=%d, idle=%d, waiting=%d%n",
                pool.getActiveConnections(),
                pool.getIdleConnections(),
                pool.getThreadsAwaitingConnection());
        }
    }
}
```

### ⚠️ Common Mistakes

**1. Oversizing connection pool**

```java
// ❌ Too many connections — database overwhelmed
config.setMaximumPoolSize(200);

// ✅ Use formula: (cores * 2) + spindle_count
config.setMaximumPoolSize(17);
```

**2. N+1 query problem**

```java
// ❌ Triggers N+1 queries
List<User> users = userRepository.findAll();
users.forEach(u -> System.out.println(u.getOrders().size()));

// ✅ Use JOIN FETCH or EntityGraph
List<User> users = userRepository.findAllWithOrders();
```

**3. Not using indexes properly**

```sql
-- ❌ Index on wrong column order
CREATE INDEX idx_bad ON orders(status, user_id);
-- Bad for: WHERE user_id = ?

-- ✅ Correct order
CREATE INDEX idx_good ON orders(user_id, status);
```

### ⚡ Performance Tip

**Monitor slow queries** and optimize them first. Use database-specific tools:
- PostgreSQL: `pg_stat_statements` extension
- MySQL: `slow_query_log`
- Oracle: AWR reports

**Use connection pooling** with appropriate sizing. Oversized pools cause database contention; undersized pools cause application bottlenecks.

### 🔧 Database Internals

Connection pools maintain a cache of physical database connections. When a connection is requested, the pool provides an existing connection or creates a new one if none are available and the pool hasn't reached maximum size. HikariCP uses a concurrent bag data structure for minimal contention.

Hibernate's session acts as a first-level cache, tracking entity state. The second-level cache is optional and shared across sessions. Query caching stores result sets of queries.

---

## 🎯 Interviewer Insight

Database optimization questions test your understanding of the full stack. Interviewers want to see that you:
- Understand connection pool sizing and configuration
- Can identify and fix N+1 query problems
- Know indexing strategies and when to use different index types
- Understand Hibernate caching and batch operations
- Can analyze query execution plans

Strong candidates can explain the formula for connection pool sizing and demonstrate how to use JOIN FETCH to avoid N+1 problems.

## ⚡ Follow-up Questions

1. How do you determine the optimal connection pool size?
2. What is the difference between first-level and second-level cache in Hibernate?
3. Why does the order of columns in a composite index matter?

## 📚 Related Topics

- [Hibernate/JPA](../../sections/13-framework-integration/q03-hibernate-jpa.md)
- [Performance Optimization](../../sections/12-performance-optimization/README.md)
- [JDBC](../../sections/08-java-io-nio/README.md)

# JDBC and Database Optimization

[Home](../../README.md) > [Performance Optimization](./README.md) > JDBC Optimization

---

## Overview

Database operations are often the dominant bottleneck in enterprise Java applications. Optimizing JDBC involves connection management, query optimization, batch processing, and proper fetch tuning to minimize round-trips and resource usage.

---

## 1. PreparedStatement Caching

```java
// BAD: Creating new Statement each time (no caching, SQL injection risk)
public User findUser(Connection conn, String name) throws SQLException {
    Statement stmt = conn.createStatement();
    ResultSet rs = stmt.executeQuery(
        "SELECT * FROM users WHERE name = '" + name + "'"); // SQL injection!
    // Statement is parsed + planned on each call
}

// GOOD: PreparedStatement (server-side caching, safe)
public User findUser(Connection conn, String name) throws SQLException {
    try (PreparedStatement ps = conn.prepareStatement(
            "SELECT * FROM users WHERE name = ?")) {
        ps.setString(1, name);
        try (ResultSet rs = ps.executeQuery()) {
            if (rs.next()) {
                return mapUser(rs);
            }
        }
    }
    return null;
}

// BEST: PreparedStatement with connection pool caching
// HikariCP caches PreparedStatements per connection
// Configure in HikariCP:
HikariConfig config = new HikariConfig();
config.addDataSourceProperty("cachePrepStmts", "true");
config.addDataSourceProperty("prepStmtCacheSize", "250");
config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
```

### PreparedStatement Benefits

```
1. SQL injection prevention (parameterized queries)
2. Server-side query plan caching (parse once, execute many)
3. Client-side statement caching (pool reuses PreparedStatement objects)
4. Binary protocol (some drivers send params in binary, not text)

Performance: 2-5x faster than Statement for repeated queries
```

---

## 2. Batch Operations

```java
// BAD: Individual inserts (one round-trip per insert)
public void insertUsers(Connection conn, List<User> users) throws SQLException {
    try (PreparedStatement ps = conn.prepareStatement(
            "INSERT INTO users (name, email) VALUES (?, ?)")) {
        for (User user : users) {
            ps.setString(1, user.getName());
            ps.setString(2, user.getEmail());
            ps.executeUpdate();  // network round-trip each time!
        }
    }
}
// 10,000 users = 10,000 round-trips

// GOOD: Batch insert (one round-trip for many inserts)
public void insertUsersBatch(Connection conn, List<User> users)
        throws SQLException {
    conn.setAutoCommit(false);
    try (PreparedStatement ps = conn.prepareStatement(
            "INSERT INTO users (name, email) VALUES (?, ?)")) {

        int batchSize = 0;
        for (User user : users) {
            ps.setString(1, user.getName());
            ps.setString(2, user.getEmail());
            ps.addBatch();

            if (++batchSize % 1000 == 0) { // flush every 1000 rows
                ps.executeBatch();
                batchSize = 0;
            }
        }
        ps.executeBatch(); // flush remaining
        conn.commit();
    } catch (SQLException e) {
        conn.rollback();
        throw e;
    }
}
// 10,000 users = 10 round-trips (batch of 1000 each)
```

### Batch Performance

```
Inserting 100,000 rows:
  Individual inserts:  ~45 seconds  (100K round-trips)
  Batch (size 100):    ~3 seconds   (1,000 round-trips)
  Batch (size 1000):   ~1.5 seconds (100 round-trips)
  Batch (size 5000):   ~1.2 seconds (20 round-trips)
  
  Diminishing returns after batch size ~1000-5000
```

---

## 3. Connection Pooling (HikariCP)

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

public class DataSourceConfig {

    public static HikariDataSource createDataSource() {
        HikariConfig config = new HikariConfig();

        // Connection settings
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("user");
        config.setPassword("password");

        // Pool sizing (critical for performance)
        config.setMaximumPoolSize(10);   // max connections
        config.setMinimumIdle(5);        // min idle connections
        config.setIdleTimeout(300_000);  // 5 minutes idle before eviction
        config.setMaxLifetime(1_800_000);// 30 minutes max connection lifetime
        config.setConnectionTimeout(30_000); // 30 seconds to get connection

        // PreparedStatement caching
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");

        // Performance settings
        config.addDataSourceProperty("useServerPrepStmts", "true");
        config.addDataSourceProperty("rewriteBatchedStatements", "true");

        return new HikariDataSource(config);
    }
}
```

### Pool Sizing Formula

```
Pool Size = CPU_cores * 2 + effective_spindle_count

Example:
  4-core server with SSD (1 spindle equivalent):
  Pool Size = 4 * 2 + 1 = 9 (round to 10)

Common mistake: Too many connections!
  50 connections = 50 threads contending for CPU + disk
  10 connections often outperforms 50 (less context switching)
```

### Connection Pool Comparison

| Feature              | HikariCP | Tomcat Pool | C3P0    | DBCP2   |
|----------------------|----------|-------------|---------|---------|
| Connection acquire   | ~1 μs    | ~5 μs       | ~10 μs  | ~8 μs   |
| Overhead per conn    | Minimal  | Moderate    | High    | Moderate|
| Spring Boot default  | Yes      | No          | No      | No      |
| Statement caching    | Yes      | Yes         | Yes     | Yes     |

---

## 4. Fetch Size Tuning

```java
// Default fetch size is typically 10 rows - too small for large result sets
// Each fetch = one network round-trip to database

// BAD: Default fetch size
try (PreparedStatement ps = conn.prepareStatement(
        "SELECT * FROM large_table")) {
    try (ResultSet rs = ps.executeQuery()) { // fetches 10 rows at a time
        while (rs.next()) { process(rs); }
    }
}

// GOOD: Tuned fetch size
try (PreparedStatement ps = conn.prepareStatement(
        "SELECT * FROM large_table")) {
    ps.setFetchSize(500);  // fetch 500 rows per round-trip
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) { process(rs); }
    }
}

// STREAMING: Very large result sets (avoid OutOfMemoryError)
try (PreparedStatement ps = conn.prepareStatement(
        "SELECT * FROM huge_table",
        ResultSet.TYPE_FORWARD_ONLY,
        ResultSet.CONCUR_READ_ONLY)) {
    ps.setFetchSize(Integer.MIN_VALUE); // MySQL streaming mode
    // PostgreSQL: conn.setAutoCommit(false); ps.setFetchSize(100);
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) { process(rs); }
    }
}
```

### Fetch Size Impact

```
Reading 100,000 rows:
  Fetch size 10:    ~8.5 seconds   (10,000 round-trips)
  Fetch size 100:   ~1.2 seconds   (1,000 round-trips)
  Fetch size 500:   ~0.6 seconds   (200 round-trips)
  Fetch size 1000:  ~0.5 seconds   (100 round-trips)
  Fetch size 5000:  ~0.45 seconds  (20 round-trips, more memory)
```

---

## 5. Lazy Loading

```java
// Eager loading: Load everything upfront (may waste resources)
public class EagerUserDao {
    public User findById(long id) throws SQLException {
        User user = loadUser(id);
        user.setOrders(loadAllOrders(id));      // loaded even if not needed
        user.setPreferences(loadPreferences(id)); // loaded even if not needed
        return user;
    }
}

// Lazy loading: Load on demand
public class LazyUser {
    private final long id;
    private final UserDao dao;
    private List<Order> orders;  // loaded lazily

    public LazyUser(long id, UserDao dao) {
        this.id = id;
        this.dao = dao;
    }

    public List<Order> getOrders() {
        if (orders == null) {
            orders = dao.loadOrders(id);  // loaded only when accessed
        }
        return orders;
    }
}
```

---

## 6. N+1 Query Problem

```java
// BAD: N+1 queries (1 query for users + N queries for orders)
public List<UserWithOrders> loadAll() throws SQLException {
    List<User> users = userDao.findAll();  // 1 query: SELECT * FROM users

    List<UserWithOrders> result = new ArrayList<>();
    for (User user : users) {
        // N queries: SELECT * FROM orders WHERE user_id = ?
        List<Order> orders = orderDao.findByUserId(user.getId());
        result.add(new UserWithOrders(user, orders));
    }
    return result;
}
// 1000 users = 1001 queries!

// GOOD: JOIN query (1 query total)
public List<UserWithOrders> loadAllOptimized() throws SQLException {
    String sql = """
        SELECT u.id, u.name, o.id as order_id, o.total
        FROM users u
        LEFT JOIN orders o ON u.id = o.user_id
        ORDER BY u.id
        """;

    Map<Long, UserWithOrders> map = new LinkedHashMap<>();
    try (PreparedStatement ps = conn.prepareStatement(sql);
         ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            long userId = rs.getLong("id");
            map.computeIfAbsent(userId, id ->
                new UserWithOrders(new User(id, rs.getString("name"))))
                .addOrder(new Order(rs.getLong("order_id"), rs.getDouble("total")));
        }
    }
    return new ArrayList<>(map.values());
}

// ALTERNATIVE: Batch IN clause
public List<UserWithOrders> loadWithBatchIn(List<Long> userIds) throws SQLException {
    // Load users: 1 query
    List<User> users = userDao.findByIds(userIds);

    // Load all orders in one batch: 1 query with IN clause
    String inClause = userIds.stream()
        .map(String::valueOf)
        .collect(Collectors.joining(",", "(", ")"));
    String sql = "SELECT * FROM orders WHERE user_id IN " + inClause;
    // Total: 2 queries instead of N+1
}
```

---

## 7. Index Optimization

```sql
-- Identify missing indexes with EXPLAIN
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'john@example.com';
-- Sequential Scan on users  (cost=0.00..1234.00 rows=1 width=128) <- BAD

-- Add index
CREATE INDEX idx_users_email ON users(email);

-- After index:
-- Index Scan using idx_users_email on users  (cost=0.28..8.29 rows=1) <- GOOD
```

```java
// Query patterns that benefit from indexes
// 1. WHERE clause columns
"SELECT * FROM users WHERE email = ?"        // index on email

// 2. JOIN columns
"SELECT * FROM orders JOIN users ON orders.user_id = users.id"
// index on orders.user_id

// 3. ORDER BY columns
"SELECT * FROM products ORDER BY price DESC" // index on price

// 4. Covering index (all selected columns in index)
"SELECT name, email FROM users WHERE status = 'ACTIVE'"
// CREATE INDEX idx_covering ON users(status) INCLUDE (name, email)
```

---

## 8. Read Replicas

```java
// Route reads to replicas, writes to primary
public class ReadWriteDataSource {
    private final DataSource primary;     // for writes
    private final DataSource replica;     // for reads

    public ReadWriteDataSource(DataSource primary, DataSource replica) {
        this.primary = primary;
        this.replica = replica;
    }

    public Connection getReadConnection() throws SQLException {
        return replica.getConnection();
    }

    public Connection getWriteConnection() throws SQLException {
        return primary.getConnection();
    }
}

// Spring Boot: Use @Transactional(readOnly = true) to route to replica
@Service
public class UserService {

    @Transactional(readOnly = true) // can be routed to read replica
    public List<User> findAll() {
        return userRepository.findAll();
    }

    @Transactional // must go to primary
    public User save(User user) {
        return userRepository.save(user);
    }
}
```

---

## 9. JDBC Optimization Checklist

| Optimization             | Impact  | Effort | Priority |
|--------------------------|---------|--------|----------|
| Connection pooling       | High    | Low    | Must-do  |
| PreparedStatement cache  | Medium  | Low    | Must-do  |
| Batch operations         | High    | Medium | High     |
| Fix N+1 queries          | High    | Medium | High     |
| Fetch size tuning        | Medium  | Low    | Medium   |
| Add missing indexes      | High    | Low    | High     |
| Read replicas            | High    | High   | For scale|
| Query optimization       | High    | Medium | Ongoing  |

---

## Interview Tips

- Connection pool size: `CPU_cores * 2 + spindle_count` (typically 10-20)
- HikariCP is the fastest connection pool and Spring Boot's default
- PreparedStatements prevent SQL injection AND enable query plan caching
- Batch inserts can be 30-50x faster than individual inserts
- Always tune `fetchSize` for large result sets (default of 10 is too small)
- N+1 is the most common performance bug; fix with JOINs or batch loading
- `@Transactional(readOnly = true)` enables routing to read replicas

---

[<< Previous: I/O Optimization](./q07-io-optimization.md) | [Next: Caching Strategies >>](./q09-caching-strategies.md)

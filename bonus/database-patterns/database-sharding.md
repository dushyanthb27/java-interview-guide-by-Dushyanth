# Database Sharding

## Overview

Database sharding is a horizontal scaling technique that splits a large database into smaller, more manageable pieces called shards. Each shard is a separate database that holds a subset of the data. This guide covers sharding strategies, implementation patterns, and best practices for Java applications.

## Why Shard?

### When to Shard

```java
// Signs you need sharding:
// 1. Single database can't handle load (CPU, memory, I/O)
// 2. Database size exceeds hardware capacity (>1TB)
// 3. Write performance degraded by locking contention
// 4. Backup/restore takes too long
// 5. Replication lag too high for read replicas

// Before sharding:
// Single database: 10M users, 100M orders, 1TB data
// Write throughput: 5,000 ops/sec
// Read latency: 200ms average

// After sharding:
// 10 shards: 1M users per shard, 10M orders per shard, 100GB per shard
// Write throughput: 50,000 ops/sec (10x)
// Read latency: 50ms average (4x)
```

## Sharding Strategies

### Horizontal Sharding (Row-Level)

```sql
-- Shard by user_id
-- Shard 0: user_id % 10 = 0
-- Shard 1: user_id % 10 = 1
-- ...
-- Shard 9: user_id % 10 = 9

-- Shard 0
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100)
);
-- Contains users with id % 10 = 0

-- Shard 1
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100)
);
-- Contains users with id % 10 = 1
```

### Vertical Sharding (Column-Level)

```sql
-- Split tables by functionality

-- Shard 1: User profile data
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    created_at TIMESTAMP
);

-- Shard 2: User activity data
CREATE TABLE user_activities (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    activity_type VARCHAR(50),
    activity_data JSONB,
    created_at TIMESTAMP
);

-- Shard 3: User transactions
CREATE TABLE user_transactions (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    amount DECIMAL(10, 2),
    transaction_date TIMESTAMP
);
```

## Sharding Key Selection

### Choosing a Sharding Key

```java
// ✅ Good sharding keys:
// - User ID (even distribution, natural partition)
// - Tenant ID (multi-tenant applications)
// - Geographic region (data locality)
// - Time range (time-series data)

// ❌ Bad sharding keys:
// - Status (low cardinality, uneven distribution)
// - Boolean (only 2 values)
// - Sequential ID (hotspot on recent data)
// - Random UUID (no locality, poor performance)
```

### Hash-Based Sharding

```java
@Service
public class HashShardingService {

    private final List<DataSource> shardDataSources;
    private final int shardCount;

    public HashShardingService(List<DataSource> dataSources) {
        this.shardDataSources = dataSources;
        this.shardCount = dataSources.size();
    }

    private int getShardNumber(Long userId) {
        // Consistent hash
        return (int) (userId % shardCount);
    }

    public User getUser(Long userId) {
        int shardNum = getShardNumber(userId);
        DataSource shard = shardDataSources.get(shardNum);
        
        try (Connection conn = shard.getConnection();
             PreparedStatement stmt = conn.prepareStatement(
                 "SELECT * FROM users WHERE id = ?")) {
            stmt.setLong(1, userId);
            ResultSet rs = stmt.executeQuery();
            if (rs.next()) {
                return mapToUser(rs);
            }
            return null;
        } catch (SQLException e) {
            throw new RuntimeException("Failed to get user", e);
        }
    }

    public void saveUser(User user) {
        int shardNum = getShardNumber(user.getId());
        DataSource shard = shardDataSources.get(shardNum);
        
        try (Connection conn = shard.getConnection();
             PreparedStatement stmt = conn.prepareStatement(
                 "INSERT INTO users (id, username, email) VALUES (?, ?, ?)")) {
            stmt.setLong(1, user.getId());
            stmt.setString(2, user.getUsername());
            stmt.setString(3, user.getEmail());
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("Failed to save user", e);
        }
    }
}
```

### Range-Based Sharding

```java
@Service
public class RangeShardingService {

    private final List<ShardRange> shardRanges;

    public RangeShardingService() {
        this.shardRanges = Arrays.asList(
            new ShardRange(0, 1000000, dataSource1),
            new ShardRange(1000001, 2000000, dataSource2),
            new ShardRange(2000001, 3000000, dataSource3),
            new ShardRange(3000001, Long.MAX_VALUE, dataSource4)
        );
    }

    private DataSource getShard(Long userId) {
        for (ShardRange range : shardRanges) {
            if (userId >= range.getMin() && userId <= range.getMax()) {
                return range.getDataSource();
            }
        }
        throw new IllegalArgumentException("User ID out of range");
    }

    public User getUser(Long userId) {
        DataSource shard = getShard(userId);
        // Query from appropriate shard
    }
}

class ShardRange {
    private final long min;
    private final long max;
    private final DataSource dataSource;
    // constructor, getters
}
```

### Directory-Based Sharding

```java
@Service
public class DirectoryShardingService {

    @Autowired
    private JdbcTemplate directoryJdbcTemplate;

    @Autowired
    private Map<String, DataSource> shardDataSources;

    public DataSource getShardForUser(Long userId) {
        // Query directory to find shard
        String shardName = directoryJdbcTemplate.queryForObject(
            "SELECT shard_name FROM user_shard_mapping WHERE user_id = ?",
            String.class,
            userId
        );
        return shardDataSources.get(shardName);
    }

    public void assignUserToShard(Long userId, String shardName) {
        directoryJdbcTemplate.update(
            "INSERT INTO user_shard_mapping (user_id, shard_name) VALUES (?, ?)",
            userId, shardName
        );
    }
}
```

## Sharding Implementation

### Dynamic DataSource Routing

```java
public class ShardRoutingDataSource extends AbstractRoutingDataSource {

    private final Map<Object, Object> targetDataSources;

    public ShardRoutingDataSource(Map<String, DataSource> dataSources) {
        this.targetDataSources = new HashMap<>(dataSources);
        setTargetDataSources(this.targetDataSources);
        setDefaultTargetDataSource(dataSources.values().iterator().next());
    }

    @Override
    protected Object determineCurrentLookupKey() {
        return ShardContextHolder.getShardKey();
    }
}

public class ShardContextHolder {
    private static final ThreadLocal<String> shardKey = new ThreadLocal<>();

    public static void setShardKey(String key) {
        shardKey.set(key);
    }

    public static String getShardKey() {
        return shardKey.get();
    }

    public static void clear() {
        shardKey.remove();
    }
}

@Configuration
public class ShardingConfig {

    @Bean
    public DataSource shardRoutingDataSource() {
        Map<String, DataSource> dataSources = new HashMap<>();
        dataSources.put("shard0", createDataSource("jdbc:postgresql://localhost:5432/shard0"));
        dataSources.put("shard1", createDataSource("jdbc:postgresql://localhost:5432/shard1"));
        dataSources.put("shard2", createDataSource("jdbc:postgresql://localhost:5432/shard2"));
        
        return new ShardRoutingDataSource(dataSources);
    }

    private DataSource createDataSource(String url) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername("user");
        config.setPassword("password");
        return new HikariDataSource(config);
    }
}
```

### Sharding with Hibernate

```java
@Entity
@Table(name = "users")
@ShardingKey("userId")
public class User {
    @Id
    private Long id;
    
    private String username;
    private String email;
}

@Repository
public class ShardedUserRepository {

    @PersistenceContext
    private EntityManager entityManager;

    @Autowired
    private ShardRoutingDataSource routingDataSource;

    public User findById(Long userId) {
        String shardKey = calculateShardKey(userId);
        ShardContextHolder.setShardKey(shardKey);
        
        try {
            return entityManager.find(User.class, userId);
        } finally {
            ShardContextHolder.clear();
        }
    }

    public void save(User user) {
        String shardKey = calculateShardKey(user.getId());
        ShardContextHolder.setShardKey(shardKey);
        
        try {
            entityManager.persist(user);
        } finally {
            ShardContextHolder.clear();
        }
    }

    private String calculateShardKey(Long userId) {
        return "shard" + (userId % 3);
    }
}
```

## Cross-Shard Queries

### Distributed Transactions

```java
@Service
public class CrossShardTransactionService {

    @Autowired
    private List<DataSource> shardDataSources;

    @Transactional
    public void transferMoney(Long fromUserId, Long toUserId, BigDecimal amount) {
        // Get shards
        DataSource fromShard = getShard(fromUserId);
        DataSource toShard = getShard(toUserId);
        
        if (fromShard.equals(toShard)) {
            // Same shard - simple transaction
            transferWithinShard(fromShard, fromUserId, toUserId, amount);
        } else {
            // Different shards - distributed transaction
            transferAcrossShards(fromShard, toShard, fromUserId, toUserId, amount);
        }
    }

    private void transferAcrossShards(DataSource fromShard, DataSource toShard,
                                       Long fromUserId, Long toUserId, BigDecimal amount) {
        // Use Saga pattern for distributed transaction
        try {
            // Step 1: Deduct from source
            deductFromAccount(fromShard, fromUserId, amount);
            
            // Step 2: Add to destination
            addToAccount(toShard, toUserId, amount);
            
        } catch (Exception e) {
            // Compensating transaction
            addToAccount(fromShard, fromUserId, amount);
            throw e;
        }
    }
}
```

### Scatter-Gather Queries

```java
@Service
public class ScatterGatherService {

    @Autowired
    private List<DataSource> shardDataSources;

    @Async
    public CompletableFuture<List<User>> getAllUsersAsync() {
        List<CompletableFuture<List<User>>> futures = shardDataSources.stream()
            .map(shard -> CompletableFuture.supplyAsync(() -> getUsersFromShard(shard)))
            .collect(Collectors.toList());
        
        CompletableFuture<Void> allFutures = CompletableFuture.allOf(
            futures.toArray(new CompletableFuture[0])
        );
        
        return allFutures.thenApply(v -> 
            futures.stream()
                .flatMap(future -> future.join().stream())
                .collect(Collectors.toList())
        );
    }

    private List<User> getUsersFromShard(DataSource shard) {
        try (Connection conn = shard.getConnection();
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {
            
            List<User> users = new ArrayList<>();
            while (rs.next()) {
                users.add(mapToUser(rs));
            }
            return users;
        } catch (SQLException e) {
            throw new RuntimeException("Failed to query shard", e);
        }
    }
}
```

## Rebalancing Shards

### Resharding Strategy

```java
@Service
public class ReshardingService {

    @Autowired
    private List<DataSource> oldShards;

    @Autowired
    private List<DataSource> newShards;

    public void reshard(int oldShardCount, int newShardCount) {
        for (int oldShardNum = 0; oldShardNum < oldShardCount; oldShardNum++) {
            DataSource oldShard = oldShards.get(oldShardNum);
            
            // Get all users from old shard
            List<User> users = getAllUsersFromShard(oldShard);
            
            for (User user : users) {
                // Calculate new shard
                int newShardNum = (int) (user.getId() % newShardCount);
                DataSource newShard = newShards.get(newShardNum);
                
                // Copy to new shard
                copyUserToShard(user, newShard);
            }
            
            // Verify data
            verifyResharding(oldShard, newShards);
            
            // Switch traffic to new shards
            // (gradual cutover)
        }
    }

    private void copyUserToShard(User user, DataSource newShard) {
        try (Connection conn = newShard.getConnection();
             PreparedStatement stmt = conn.prepareStatement(
                 "INSERT INTO users (id, username, email) VALUES (?, ?, ?) " +
                 "ON CONFLICT (id) DO UPDATE SET username = EXCLUDED.username, email = EXCLUDED.email")) {
            stmt.setLong(1, user.getId());
            stmt.setString(2, user.getUsername());
            stmt.setString(3, user.getEmail());
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new RuntimeException("Failed to copy user", e);
        }
    }
}
```

## Sharding Frameworks

### ShardingSphere

```yaml
# application.yml
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1,ds2
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: org.postgresql.Driver
        jdbc-url: jdbc:postgresql://localhost:5432/shard0
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: org.postgresql.Driver
        jdbc-url: jdbc:postgresql://localhost:5432/shard1
      ds2:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: org.postgresql.Driver
        jdbc-url: jdbc:postgresql://localhost:5432/shard2
    
    rules:
      sharding:
        tables:
          users:
            actual-data-nodes: ds$->{0..2}.users
            table-strategy:
              standard:
                sharding-column: id
                sharding-algorithm-name: user-inline
        sharding-algorithms:
          user-inline:
            type: INLINE
            props:
              algorithm-expression: ds$->{id % 3}
```

### Vitess

```java
// Vitess provides MySQL sharding with automatic routing
// Configuration through VSchema

// vschema.json
{
  "tables": {
    "users": {
      "column_vindexes": [
        {
          "column": "id",
          "name": "hash"
        }
      ]
    }
  },
  "vindexes": {
    "hash": {
      "type": "hash"
    }
  }
}
```

## Common Pitfalls

### 1. Wrong Sharding Key

```java
// ❌ Bad: Sharding by status (low cardinality)
// Shard 0: ACTIVE users (90% of data)
// Shard 1: INACTIVE users (10% of data)
// Uneven distribution, hotspot on shard 0

// ✅ Good: Sharding by user_id (high cardinality)
// Even distribution across all shards
```

### 2. Cross-Shard Joins

```sql
-- ❌ Bad: Join across shards
SELECT u.*, o.* 
FROM users u 
JOIN orders o ON u.id = o.user_id;
-- Users and orders on different shards - very slow

-- ✅ Good: Denormalize or query separately
-- Option 1: Denormalize user data in orders table
SELECT o.* 
FROM orders o 
WHERE o.user_id = ?;

-- Option 2: Query separately and join in application
```

### 3. Hotspot on Recent Data

```java
// ❌ Bad: Sequential ID sharding
// Recent IDs all go to same shard
// Shard 0: IDs 1-1000 (old, cold)
// Shard 1: IDs 1001-2000 (old, cold)
// Shard 2: IDs 2001-3000 (new, hot) - overloaded

// ✅ Good: Hash-based sharding
// Even distribution regardless of ID sequence
int shard = userId % shardCount;
```

### 4. No Rebalancing Strategy

```java
// ❌ Problem: Data grows unevenly over time
// No plan to rebalance shards
// Performance degrades

// ✅ Solution: Plan for resharding
// - Add new shards
// - Redistribute data
// - Gradual cutover
// - Monitor during process
```

### 5. Complex Cross-Shard Transactions

```java
// ❌ Bad: 2PC across many shards
// High latency, complex failure handling

// ✅ Good: Saga pattern
// - Break into local transactions
// - Implement compensating actions
// - Eventual consistency
```

## Best Practices

1. **Choose sharding key carefully** - High cardinality, even distribution
2. **Minimize cross-shard queries** - Denormalize when needed
3. **Plan for rebalancing** - Data will grow unevenly
4. **Use consistent hashing** - Minimize data movement on resharding
5. **Implement monitoring** - Track shard health and performance
6. **Handle failures gracefully** - Shard failures shouldn't crash app
7. **Document sharding strategy** - Team must understand architecture
8. **Test at scale** - Validate with production-like data
9. **Consider managed solutions** - Cloud database sharding services
10. **Plan for growth** - Sharding is complex, consider alternatives first

## When NOT to Shard

```java
// Don't shard if:
// 1. Database can be optimized with indexing
// 2. Read replicas can handle read load
// 3. Database size < 500GB
// 4. Write throughput < 10,000 ops/sec
// 5. Team lacks sharding expertise
// 6. Application doesn't need horizontal scale

// Consider alternatives:
// - Better indexing
// - Read replicas
// - Caching
// - Database optimization
// - Hardware upgrade
```

## Performance Impact

| Metric | Single DB | 10 Shards | Improvement |
|--------|-----------|------------|-------------|
| Write Throughput | 5,000 ops/sec | 50,000 ops/sec | 10x |
| Read Latency | 200ms | 50ms | 4x |
| Storage | 1TB per node | 100GB per node | 10x less |
| Backup Time | 4 hours | 30 minutes | 8x faster |
| Complexity | Low | Very High | - |

## Interview Questions

1. What is the difference between horizontal and vertical sharding?
2. How do you choose a sharding key?
3. What are the challenges of cross-shard queries?
4. How do you handle distributed transactions across shards?
5. What is consistent hashing and why is it useful?
6. How do you rebalance shards?
7. When should you shard vs use read replicas?
8. What are the common pitfalls of sharding?

## Related Topics

- [Read Replicas Patterns](./read-replicas-patterns.md)
- [Caching Patterns](./caching-patterns.md)
- [Data Migration Patterns](./data-migration-patterns.md)

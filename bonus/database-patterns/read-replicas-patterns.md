# Read Replicas Patterns

## Overview

Read replicas are copies of the primary database that serve read traffic. They enable horizontal scaling of read operations, improve performance for read-heavy workloads, and provide high availability. This guide covers read replica patterns, consistency models, and implementation strategies for Java applications.

## Why Read Replicas?

### Benefits

```java
// Without read replicas:
// Single database handles all reads and writes
// Write throughput: 5,000 ops/sec
// Read throughput: 5,000 ops/sec
// Total throughput: 5,000 ops/sec
// Response time: 200ms average

// With read replicas:
// Primary handles writes
// 5 replicas handle reads
// Write throughput: 5,000 ops/sec
// Read throughput: 25,000 ops/sec (5 replicas)
// Total throughput: 30,000 ops/sec (6x improvement)
// Response time: 50ms average (4x improvement)
```

### Use Cases

- **Read-heavy applications** - More reads than writes
- **Geographic distribution** - Serve users from local replicas
- **Reporting/analytics** - Run heavy queries on replicas
- **High availability** - Failover to replica if primary fails
- **Load balancing** - Distribute read load across replicas

## Consistency Models

### Strong Consistency

```java
// Reads from primary only
// Guarantees latest data
// No replication lag

@Service
public class StrongConsistencyService {

    @Autowired
    private JdbcTemplate primaryJdbcTemplate;

    public User getUser(Long userId) {
        // Always read from primary
        return primaryJdbcTemplate.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new BeanPropertyRowMapper<>(User.class),
            userId
        );
    }

    public void updateUser(User user) {
        primaryJdbcTemplate.update(
            "UPDATE users SET username = ?, email = ? WHERE id = ?",
            user.getUsername(), user.getEmail(), user.getId()
        );
    }
}
```

### Eventual Consistency

```java
// Reads from replicas
// Accepts stale data
// Better performance

@Service
public class EventualConsistencyService {

    @Autowired
    private JdbcTemplate primaryJdbcTemplate;

    @Autowired
    private List<JdbcTemplate> replicaJdbcTemplates;

    private final Random random = new Random();

    public User getUser(Long userId) {
        // Load balance across replicas
        JdbcTemplate replica = getRandomReplica();
        return replica.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new BeanPropertyRowMapper<>(User.class),
            userId
        );
    }

    private JdbcTemplate getRandomReplica() {
        int index = random.nextInt(replicaJdbcTemplates.size());
        return replicaJdbcTemplates.get(index);
    }
}
```

### Read-After-Write Consistency

```java
// Read from primary immediately after write
// Read from replicas after delay

@Service
public class ReadAfterWriteService {

    @Autowired
    private JdbcTemplate primaryJdbcTemplate;

    @Autowired
    private List<JdbcTemplate> replicaJdbcTemplates;

    @Autowired
    private Cache<Long, User> recentWritesCache;

    public void updateUser(User user) {
        primaryJdbcTemplate.update(
            "UPDATE users SET username = ?, email = ? WHERE id = ?",
            user.getUsername(), user.getEmail(), user.getId()
        );
        
        // Cache recent writes
        recentWritesCache.put(user.getId(), user);
    }

    public User getUser(Long userId) {
        // Check cache first (recent writes)
        User cached = recentWritesCache.getIfPresent(userId);
        if (cached != null) {
            return cached;
        }
        
        // Read from replica
        JdbcTemplate replica = getRandomReplica();
        return replica.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new BeanPropertyRowMapper<>(User.class),
            userId
        );
    }
}
```

## Routing Strategies

### Round-Robin Routing

```java
@Service
public class RoundRobinRoutingService {

    @Autowired
    private List<JdbcTemplate> replicaJdbcTemplates;

    private final AtomicInteger counter = new AtomicInteger(0);

    public User getUser(Long userId) {
        int index = counter.getAndIncrement() % replicaJdbcTemplates.size();
        JdbcTemplate replica = replicaJdbcTemplates.get(index);
        
        return replica.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new BeanPropertyRowMapper<>(User.class),
            userId
        );
    }
}
```

### Weighted Round-Robin

```java
@Service
public class WeightedRoutingService {

    private final List<Replica> replicas;

    public WeightedRoutingService() {
        this.replicas = Arrays.asList(
            new Replica(replica1, 3),  // 30% of traffic
            new Replica(replica2, 2),  // 20% of traffic
            new Replica(replica3, 5)   // 50% of traffic
        );
    }

    public User getUser(Long userId) {
        Replica replica = selectReplicaByWeight();
        return replica.getJdbcTemplate().queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new BeanPropertyRowMapper<>(User.class),
            userId
        );
    }

    private Replica selectReplicaByWeight() {
        int totalWeight = replicas.stream().mapToInt(Replica::getWeight).sum();
        int random = ThreadLocalRandom.current().nextInt(totalWeight);
        
        int currentWeight = 0;
        for (Replica replica : replicas) {
            currentWeight += replica.getWeight();
            if (random < currentWeight) {
                return replica;
            }
        }
        return replicas.get(replicas.size() - 1);
    }

    static class Replica {
        private final JdbcTemplate jdbcTemplate;
        private final int weight;
        // constructor, getters
    }
}
```

### Least Connections Routing

```java
@Service
public class LeastConnectionsRoutingService {

    private final List<ReplicaConnection> replicas;

    public User getUser(Long userId) {
        ReplicaConnection replica = selectLeastConnectedReplica();
        replica.incrementConnections();
        
        try {
            return replica.getJdbcTemplate().queryForObject(
                "SELECT * FROM users WHERE id = ?",
                new BeanPropertyRowMapper<>(User.class),
                userId
            );
        } finally {
            replica.decrementConnections();
        }
    }

    private ReplicaConnection selectLeastConnectedReplica() {
        return replicas.stream()
            .min(Comparator.comparingInt(ReplicaConnection::getConnections))
            .orElse(replicas.get(0));
    }

    static class ReplicaConnection {
        private final JdbcTemplate jdbcTemplate;
        private final AtomicInteger connections = new AtomicInteger(0);
        // constructor, getters, increment, decrement
    }
}
```

### Geographic Routing

```java
@Service
public class GeographicRoutingService {

    @Autowired
    private Map<String, JdbcTemplate> regionalReplicas;

    public User getUser(Long userId, String userRegion) {
        // Route to nearest replica
        JdbcTemplate replica = regionalReplicas.getOrDefault(
            userRegion,
            regionalReplicas.get("default")
        );
        
        return replica.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new BeanPropertyRowMapper<>(User.class),
            userId
        );
    }
}
```

## Spring Boot Configuration

### Dynamic DataSource Routing

```java
public class ReadWriteDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceContextHolder.getDataSourceType();
    }
}

public class DataSourceContextHolder {
    private static final ThreadLocal<DataSourceType> context = new ThreadLocal<>();

    public static void setDataSourceType(DataSourceType type) {
        context.set(type);
    }

    public static DataSourceType getDataSourceType() {
        return context.get();
    }

    public static void clear() {
        context.remove();
    }

    public enum DataSourceType {
        MASTER, SLAVE
    }
}

@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource routingDataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DataSourceType.MASTER, masterDataSource());
        targetDataSources.put(DataSourceType.SLAVE, slaveDataSource());

        ReadWriteDataSource routingDataSource = new ReadWriteDataSource();
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(masterDataSource());
        return routingDataSource;
    }

    @Bean
    public DataSource masterDataSource() {
        return createDataSource("jdbc:postgresql://primary:5432/mydb");
    }

    @Bean
    public DataSource slaveDataSource() {
        return createDataSource("jdbc:postgresql://replica:5432/mydb");
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

### AOP for Automatic Routing

```java
@Aspect
@Component
public class DataSourceRoutingAspect {

    @Before("@annotation(org.springframework.transaction.annotation.Transactional)")
    public void setWriteDataSource() {
        DataSourceContextHolder.setDataSourceType(DataSourceType.MASTER);
    }

    @Before("@annotation(com.example.annotation.ReadOnly)")
    public void setReadDataSource() {
        DataSourceContextHolder.setDataSourceType(DataSourceType.SLAVE);
    }

    @After("@annotation(org.springframework.transaction.annotation.Transactional)")
    public void clearDataSource() {
        DataSourceContextHolder.clear();
    }
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ReadOnly {
}
```

### Multiple Replicas with Load Balancing

```java
@Configuration
public class MultiReplicaConfig {

    @Bean
    public DataSource routingDataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DataSourceType.MASTER, masterDataSource());
        targetDataSources.put(DataSourceType.SLAVE, loadBalancedSlaveDataSource());

        ReadWriteDataSource routingDataSource = new ReadWriteDataSource();
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(masterDataSource());
        return routingDataSource;
    }

    @Bean
    public DataSource loadBalancedSlaveDataSource() {
        List<DataSource> slaves = Arrays.asList(
            slaveDataSource1(),
            slaveDataSource2(),
            slaveDataSource3()
        );
        return new LoadBalancedDataSource(slaves);
    }

    private DataSource createDataSource(String url) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername("user");
        config.setPassword("password");
        return new HikariDataSource(config);
    }
}

public class LoadBalancedDataSource implements DataSource {

    private final List<DataSource> slaves;
    private final AtomicInteger counter = new AtomicInteger(0);

    public LoadBalancedDataSource(List<DataSource> slaves) {
        this.slaves = slaves;
    }

    @Override
    public Connection getConnection() throws SQLException {
        int index = counter.getAndIncrement() % slaves.size();
        return slaves.get(index).getConnection();
    }

    // Implement other DataSource methods by delegating to first slave
    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return slaves.get(0).getConnection(username, password);
    }

    // ... other DataSource methods
}
```

## Replication Lag Handling

### Monitoring Replication Lag

```java
@Service
public class ReplicationLagMonitor {

    @Autowired
    private JdbcTemplate masterJdbcTemplate;

    @Autowired
    private List<JdbcTemplate> replicaJdbcTemplates;

    @Scheduled(fixedRate = 10000) // Every 10 seconds
    public void monitorReplicationLag() {
        // Get master position
        String masterPosition = masterJdbcTemplate.queryForObject(
            "SHOW MASTER STATUS",
            (rs, rowNum) -> rs.getString("File") + ":" + rs.getString("Position")
        );

        // Check each replica
        for (int i = 0; i < replicaJdbcTemplates.size(); i++) {
            JdbcTemplate replica = replicaJdbcTemplates.get(i);
            
            String replicaPosition = replica.queryForObject(
                "SHOW SLAVE STATUS",
                (rs, rowNum) -> rs.getString("Master_Log_File") + ":" + rs.getString("Exec_Master_Log_Pos")
            );

            long lag = calculateLag(masterPosition, replicaPosition);
            
            if (lag > 5000) { // More than 5 seconds
                log.warn("Replica {} has replication lag: {}ms", i, lag);
                // Alert or take action
            }
        }
    }

    private long calculateLag(String masterPos, String replicaPos) {
        // Simplified calculation
        // In production, use more sophisticated logic
        return 0;
    }
}
```

### Lag-Aware Routing

```java
@Service
public class LagAwareRoutingService {

    @Autowired
    private List<ReplicaHealth> replicas;

    public User getUser(Long userId) {
        // Select replica with acceptable lag
        ReplicaHealth selected = replicas.stream()
            .filter(replica -> replica.getLag() < 1000) // Less than 1 second
            .min(Comparator.comparingLong(ReplicaHealth::getLag))
            .orElse(replicas.get(0)); // Fallback to first replica

        return selected.getJdbcTemplate().queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new BeanPropertyRowMapper<>(User.class),
            userId
        );
    }
}

class ReplicaHealth {
    private final JdbcTemplate jdbcTemplate;
    private volatile long lag;
    // constructor, getters, setters
}
```

### Fallback to Primary

```java
@Service
public class FallbackService {

    @Autowired
    private JdbcTemplate primaryJdbcTemplate;

    @Autowired
    private List<JdbcTemplate> replicaJdbcTemplates;

    public User getUser(Long userId) {
        // Try replicas first
        for (JdbcTemplate replica : replicaJdbcTemplates) {
            try {
                return replica.queryForObject(
                    "SELECT * FROM users WHERE id = ?",
                    new BeanPropertyRowMapper<>(User.class),
                    userId
                );
            } catch (DataAccessException e) {
                log.warn("Replica query failed, trying next", e);
            }
        }

        // Fallback to primary
        log.info("All replicas failed, falling back to primary");
        return primaryJdbcTemplate.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new BeanPropertyRowMapper<>(User.class),
            userId
        );
    }
}
```

## Failover Strategies

### Manual Failover

```java
@Service
public class ManualFailoverService {

    @Autowired
    private JdbcTemplate primaryJdbcTemplate;

    @Autowired
    private JdbcTemplate replicaJdbcTemplate;

    public void promoteReplicaToPrimary() {
        // Step 1: Stop writes to primary
        // (Application-level stop)

        // Step 2: Ensure replica is fully synced
        waitForReplicaSync();

        // Step 3: Promote replica to primary
        replicaJdbcTemplate.execute("STOP SLAVE");
        replicaJdbcTemplate.execute("RESET SLAVE");

        // Step 4: Update application configuration
        // (Point primary datasource to new primary)

        // Step 5: Reconfigure old primary as replica
        primaryJdbcTemplate.execute("CHANGE MASTER TO ...");
        primaryJdbcTemplate.execute("START SLAVE");
    }

    private void waitForReplicaSync() {
        // Wait until Seconds_Behind_Master = 0
        // Implementation depends on database
    }
}
```

### Automatic Failover with Orchestrator

```java
// Use tools like Orchestrator (MySQL), Patroni (PostgreSQL)
// These tools handle automatic failover

// Configuration example (Patroni for PostgreSQL)
@Configuration
public class PatroniConfig {

    @Bean
    public DataSource patroniAwareDataSource() {
        // Patroni provides virtual IP that points to current primary
        return createDataSource("jdbc:postgresql://patroni-vip:5432/mydb");
    }
}
```

## Common Pitfalls

### 1. Writes to Replica

```java
// ❌ Bad: Writing to replica
@Transactional
public void updateUser(User user) {
    // Transaction routed to replica
    // Write fails or causes replication issues
    userRepository.save(user);
}

// ✅ Good: Ensure writes go to primary
@Transactional
@Write // Custom annotation to force primary
public void updateUser(User user) {
    userRepository.save(user);
}
```

### 2. Ignoring Replication Lag

```java
// ❌ Bad: Read immediately after write
public void updateUserAndRead(Long userId) {
    userRepository.updateUser(userId, "newname");
    // Might read old data from replica
    User user = userRepository.findById(userId);
}

// ✅ Good: Read from primary after write
public void updateUserAndRead(Long userId) {
    userRepository.updateUser(userId, "newname");
    User user = userRepository.findByIdFromPrimary(userId);
}
```

### 3. Uneven Load Distribution

```java
// ❌ Bad: All reads go to one replica
public User getUser(Long userId) {
    return replica1JdbcTemplate.queryForObject(...);
}

// ✅ Good: Load balance across replicas
public User getUser(Long userId) {
    JdbcTemplate replica = selectReplicaByLoad();
    return replica.queryForObject(...);
}
```

### 4. No Monitoring

```java
// ❌ Bad: No monitoring of replica health
// Replica fails silently, application suffers

// ✅ Good: Monitor replica health and lag
@Scheduled(fixedRate = 10000)
public void monitorReplicas() {
    checkReplicaHealth();
    checkReplicationLag();
    alertOnFailure();
}
```

### 5. Complex Cross-Replica Queries

```sql
-- ❌ Bad: Query across replicas
-- Each query goes to different replica
-- Inconsistent results

-- ✅ Good: Query same replica for consistency
-- Or use primary for consistency
```

## Best Practices

1. **Route writes to primary** - Never write to replicas
2. **Route reads to replicas** - Scale read capacity
3. **Monitor replication lag** - Alert on high lag
4. **Handle replica failures** - Fallback to primary
5. **Use connection pooling** - For each replica
6. **Implement health checks** - Detect replica failures
7. **Consider consistency requirements** - Choose appropriate model
8. **Test failover procedures** - Regular drills
9. **Document failover process** - Team readiness
10. **Use managed services** - Cloud database replication

## Performance Impact

| Metric | Single DB | 1 Primary + 5 Replicas | Improvement |
|--------|-----------|----------------------|-------------|
| Read Throughput | 5,000 ops/sec | 25,000 ops/sec | 5x |
| Write Throughput | 5,000 ops/sec | 5,000 ops/sec | Same |
| Read Latency | 200ms | 50ms | 4x |
| Write Latency | 200ms | 200ms | Same |
| Availability | 99.9% | 99.99% | 10x |
| Cost | 1x | 6x | 6x |

## Consistency vs Performance Trade-off

| Consistency Model | Read Latency | Staleness | Use Case |
|-------------------|--------------|-----------|----------|
| Strong | High (primary) | None | Financial data |
| Read-After-Write | Medium | Minimal | User profiles |
| Eventual | Low (replica) | Lag | Analytics, feeds |
| Bounded Staleness | Low | < N seconds | Social media |

## Interview Questions

1. What is the difference between strong and eventual consistency?
2. How do you handle replication lag?
3. What are the different routing strategies for read replicas?
4. How do you implement automatic failover?
5. What are the trade-offs of using read replicas?
6. How do you monitor replica health?
7. When would you choose read replicas vs sharding?
8. How do you ensure writes go to primary?

## Related Topics

- [Database Sharding](./database-sharding.md)
- [Caching Patterns](./caching-patterns.md)
- [Transaction Management](./transaction-management.md)

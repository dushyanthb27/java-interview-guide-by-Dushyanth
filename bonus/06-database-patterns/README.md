# Database Patterns for Java Developers

## Overview

This comprehensive guide covers essential database patterns and best practices for Java developers. Understanding these patterns is crucial for building scalable, performant, and maintainable applications that interact with relational databases effectively.

## What You'll Learn

This section covers:

1. **Connection Pooling Patterns** - HikariCP, Druid, and custom pool implementations
2. **Transaction Management** - Transaction patterns, isolation levels, and propagation
3. **ORM Patterns** - JPA/Hibernate patterns, lazy loading, and caching strategies
4. **Query Optimization** - SQL optimization techniques and execution plans
5. **Indexing Strategies** - Database indexing patterns and trade-offs
6. **Data Migration Patterns** - Schema migration strategies and versioning
7. **Caching Patterns** - Application-level caching strategies (Redis, Caffeine, etc.)
8. **Database Sharding** - Sharding and partitioning strategies for scale
9. **Read Replicas Patterns** - Read replica patterns and consistency models

## Why Database Patterns Matter

Database interactions are often the bottleneck in application performance. According to various studies:

- **70-90%** of application performance issues are database-related
- **60-80%** of application time is spent waiting for database queries
- Poor database design can cost **10-100x** in performance degradation

Mastering these patterns helps you:

- **Improve Performance** - Reduce query latency and increase throughput
- **Ensure Consistency** - Maintain data integrity across distributed systems
- **Scale Effectively** - Handle increased load through proper architecture
- **Reduce Costs** - Optimize resource utilization and database licensing
- **Prevent Bugs** - Avoid common pitfalls like connection leaks and deadlocks

## Target Audience

This guide is designed for:

- **Java Developers** - Spring Boot, JPA/Hibernate, JDBC users
- **Backend Engineers** - Building REST APIs and microservices
- **Technical Leads** - Making architectural decisions
- **Interview Candidates** - Preparing for senior/lead positions
- **Performance Engineers** - Optimizing database interactions

## Prerequisites

Before diving into these patterns, you should have:

- Basic understanding of SQL and relational databases
- Experience with Java and Spring Framework
- Knowledge of JDBC and JPA/Hibernate basics
- Understanding of ACID properties and transactions

## Quick Reference

### Connection Pooling

```java
// HikariCP - Recommended for Spring Boot 2.x+
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
```

### Transaction Management

```java
@Transactional(isolation = Isolation.READ_COMMITTED,
               propagation = Propagation.REQUIRED,
               timeout = 30,
               readOnly = false)
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    // Transactional business logic
}
```

### ORM Patterns

```java
@Entity
@Table(name = "users")
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @OneToMany(fetch = FetchType.LAZY, mappedBy = "user")
    private List<Order> orders;
}
```

### Query Optimization

```sql
-- Bad: N+1 query problem
SELECT * FROM users WHERE id = ?;
SELECT * FROM orders WHERE user_id = ?; -- Repeated N times

-- Good: JOIN with fetch
SELECT u.*, o.* 
FROM users u 
LEFT JOIN orders o ON u.id = o.user_id 
WHERE u.id = ?;
```

### Indexing Strategies

```sql
-- Single column index
CREATE INDEX idx_user_email ON users(email);

-- Composite index (order matters!)
CREATE INDEX idx_user_status_created ON users(status, created_at);

-- Covering index (includes all query columns)
CREATE INDEX idx_order_covering ON orders(user_id, status, total);
```

## Performance Impact Summary

| Pattern | Performance Impact | Complexity | Use Case |
|---------|-------------------|------------|----------|
| Connection Pooling | 10-100x faster connection acquisition | Low | All applications |
| Read Replicas | 2-10x read throughput | Medium | Read-heavy workloads |
| Caching | 10-1000x faster reads | Medium | Frequently accessed data |
| Query Optimization | 2-100x faster queries | High | Slow queries |
| Proper Indexing | 10-1000x faster lookups | Medium | Large datasets |
| Sharding | Linear scalability | Very High | Massive scale |

## Common Pitfalls to Avoid

### 1. Connection Leaks

```java
// ❌ Bad: Connection not closed
public User getUser(Long id) {
    Connection conn = dataSource.getConnection();
    Statement stmt = conn.createStatement();
    ResultSet rs = stmt.executeQuery("SELECT * FROM users WHERE id = " + id);
    // Connection leak if exception occurs!
}

// ✅ Good: Try-with-resources
public User getUser(Long id) {
    try (Connection conn = dataSource.getConnection();
         Statement stmt = conn.createStatement();
         ResultSet rs = stmt.executeQuery("SELECT * FROM users WHERE id = " + id)) {
        // Auto-closed
    }
}
```

### 2. N+1 Query Problem

```java
// ❌ Bad: N+1 queries
List<User> users = userRepository.findAll();
for (User user : users) {
    List<Order> orders = user.getOrders(); // Triggers query per user
}

// ✅ Good: Entity graph or JOIN fetch
@EntityGraph(attributePaths = {"orders"})
List<User> findAllWithOrders();
```

### 3. Over-fetching Data

```java
// ❌ Bad: Fetching all columns
@Query("SELECT u FROM User u")
List<User> findAll();

// ✅ Good: Projection for specific fields
@Query("SELECT new com.dto.UserDTO(u.id, u.name, u.email) FROM User u")
List<UserDTO> findAllDTO();
```

### 4. Missing Indexes

```sql
-- ❌ Bad: Full table scan
SELECT * FROM orders WHERE user_id = 12345;

-- ✅ Good: Index scan
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### 5. Long-running Transactions

```java
// ❌ Bad: Transaction holds locks too long
@Transactional
public void processBatch(List<Long> userIds) {
    for (Long id : userIds) {
        // External API call inside transaction!
        externalService.notify(id);
        userRepository.updateStatus(id, "PROCESSED");
    }
}

// ✅ Good: Short transactions
public void processBatch(List<Long> userIds) {
    for (Long id : userIds) {
        transactionTemplate.execute(status -> {
            userRepository.updateStatus(id, "PROCESSED");
            return null;
        });
        externalService.notify(id); // Outside transaction
    }
}
```

## Interview Preparation

When asked about database patterns in interviews, focus on:

1. **Trade-offs** - Every pattern has pros and cons
2. **Context** - When to use which pattern
3. **Experience** - Real-world examples and lessons learned
4. **Metrics** - How you measure success
5. **Failures** - What went wrong and how you fixed it

### Common Interview Questions

- How do you optimize slow database queries?
- Explain the N+1 problem and how to solve it
- What are the different transaction isolation levels?
- When would you use read replicas vs sharding?
- How do you handle database schema migrations?
- What's the difference between optimistic and pessimistic locking?
- How do you design indexes for a complex query?

## Learning Path

Recommended order for studying these patterns:

1. **Start Here** - Connection Pooling (foundational)
2. **Transaction Management** - Essential for data consistency
3. **ORM Patterns** - Daily development patterns
4. **Query Optimization** - Performance tuning
5. **Indexing Strategies** - Database performance
6. **Caching Patterns** - Application-level optimization
7. **Data Migration Patterns** - Schema evolution
8. **Read Replicas** - Scaling reads
9. **Database Sharding** - Advanced scaling

## Tools and Technologies

### Connection Pools
- **HikariCP** - Fastest, Spring Boot default
- **Apache DBCP2** - Legacy, robust
- **Tomcat JDBC** - Good for Tomcat deployments
- **Vibur DBCP** - Advanced monitoring

### ORM Frameworks
- **Hibernate** - Most popular JPA implementation
- **EclipseLink** - Reference JPA implementation
- **MyBatis** - SQL-centric ORM
- **jOOQ** - Type-safe SQL builder

### Caching Solutions
- **Redis** - Distributed cache
- **Caffeine** - High-performance local cache
- **Ehcache** - Mature local cache
- **Hazelcast** - Distributed data grid

### Database Monitoring
- **Prometheus + Grafana** - Metrics and visualization
- **Percona PMM** - Database monitoring
- **Datadog** - APM and monitoring
- **New Relic** - Performance monitoring

## Additional Resources

### Books
- "High Performance MySQL" by Baron Schwartz
- "SQL Performance Explained" by Markus Winand
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "Java Persistence with Hibernate" by Christian Bauer

### Online Resources
- PostgreSQL Documentation
- MySQL Reference Manual
- Spring Data JPA Documentation
- Hibernate User Guide

### Blogs
- Percona Database Blog
- Vitess Blog (sharding)
- Hibernate Blog
- Spring Blog

## Contributing

This guide is part of the Java Interview Guide project. If you find errors or want to add content, please contribute!

## License

This guide follows the same license as the parent project.

---

**Next:** [Connection Pooling Patterns](./connection-pooling-patterns.md)

# Query Optimization

## Overview

Query optimization is the process of improving database query performance. Poorly optimized queries can cause significant performance issues, leading to slow application response times and increased database load. This guide covers techniques for analyzing and optimizing SQL queries in Java applications.

## Query Performance Analysis

### Measuring Query Performance

```java
@Repository
public class PerformanceAuditRepository {

    @PersistenceContext
    private EntityManager entityManager;

    public List<Order> findOrdersWithMetrics(Long userId) {
        long startTime = System.currentTimeMillis();
        
        Query query = entityManager.createQuery(
            "SELECT o FROM Order o WHERE o.user.id = :userId", Order.class);
        query.setParameter("userId", userId);
        List<Order> orders = query.getResultList();
        
        long duration = System.currentTimeMillis() - startTime;
        log.info("Query executed in {}ms, returned {} rows", duration, orders.size());
        
        return orders;
    }
}
```

### Using EXPLAIN to Analyze Queries

```sql
-- PostgreSQL EXPLAIN
EXPLAIN ANALYZE 
SELECT o.id, o.total, o.order_date 
FROM orders o 
WHERE o.user_id = 12345 
ORDER BY o.order_date DESC 
LIMIT 10;

-- Output shows:
-- - Query plan (Seq Scan vs Index Scan)
-- - Execution time
-- - Rows processed
-- - Memory usage

-- MySQL EXPLAIN
EXPLAIN 
SELECT o.id, o.total, o.order_date 
FROM orders o 
WHERE o.user_id = 12345 
ORDER BY o.order_date DESC 
LIMIT 10;

-- Output shows:
-- - id: Query identifier
-- - select_type: Query type (SIMPLE, PRIMARY, SUBQUERY, etc.)
-- - table: Table accessed
-- - type: Access type (ALL, index, range, ref, etc.)
-- - possible_keys: Indexes that could be used
-- - key: Index actually used
-- - rows: Estimated rows examined
-- - Extra: Additional information (Using where, Using filesort, etc.)
```

## Common Performance Issues

### 1. SELECT * Anti-Pattern

```sql
-- ❌ Bad: Selects all columns
SELECT * FROM orders WHERE user_id = 12345;

-- Problems:
-- - Fetches unnecessary columns
-- - Increases network transfer
-- - Prevents index-only scans
-- - Wastes memory

-- ✅ Good: Select only needed columns
SELECT id, total, order_date 
FROM orders 
WHERE user_id = 12345;
```

### 2. Missing WHERE Clause

```sql
-- ❌ Bad: Full table scan
SELECT * FROM orders;

-- ✅ Good: Add WHERE clause with indexed column
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

### 3. Functions on Indexed Columns

```sql
-- ❌ Bad: Function prevents index usage
SELECT * FROM orders 
WHERE DATE(order_date) = '2024-01-15';

-- ✅ Good: Use range on indexed column
SELECT * FROM orders 
WHERE order_date >= '2024-01-15' AND order_date < '2024-01-16';

-- ✅ Or use function-based index
CREATE INDEX idx_orders_date ON orders(DATE(order_date));
```

### 4. Leading Wildcard in LIKE

```sql
-- ❌ Bad: Leading wildcard prevents index usage
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- ✅ Good: Trailing wildcard uses index
SELECT * FROM users WHERE email LIKE 'john%';

-- ✅ Alternative: Use full-text search
SELECT * FROM users 
WHERE to_tsvector('english', email) @@ to_tsquery('gmail');
```

### 5. OR Conditions on Different Columns

```sql
-- ❌ Bad: OR on different columns often prevents index usage
SELECT * FROM orders 
WHERE user_id = 12345 OR status = 'PENDING';

-- ✅ Good: Use UNION ALL
SELECT * FROM orders WHERE user_id = 12345
UNION ALL
SELECT * FROM orders WHERE status = 'PENDING' AND user_id != 12345;

-- ✅ Or use composite index
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

## Index Usage Optimization

### Index Scan vs Sequential Scan

```sql
-- Sequential scan (bad for large tables)
-- Reads entire table row by row
EXPLAIN SELECT * FROM orders WHERE user_id = 12345;
-- Output: Seq Scan on orders

-- Index scan (good for selective queries)
-- Uses index to find matching rows
EXPLAIN SELECT * FROM orders WHERE user_id = 12345;
-- Output: Index Scan using idx_orders_user_id
```

### Covering Indexes

```sql
-- Covering index includes all columns needed by query
-- Avoids table access (index-only scan)

CREATE INDEX idx_orders_covering 
ON orders(user_id, status, total, order_date);

-- Query uses only index, no table access needed
SELECT status, total, order_date 
FROM orders 
WHERE user_id = 12345;
```

### Partial Indexes

```sql
-- Index only a subset of rows
-- Smaller index, faster maintenance

CREATE INDEX idx_orders_active 
ON orders(user_id, order_date) 
WHERE status = 'ACTIVE';

-- Only active orders indexed
SELECT * FROM orders 
WHERE user_id = 12345 AND status = 'ACTIVE';
```

## JOIN Optimization

### JOIN Order Matters

```sql
-- ❌ Bad: Large table first
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.status = 'ACTIVE';

-- ✅ Good: Filter first, then join
SELECT * FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.status = 'ACTIVE';
```

### Use Appropriate JOIN Types

```sql
-- INNER JOIN: Only matching rows
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: All from left, matching from right
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- Choose based on business logic
-- INNER JOIN is generally faster
```

### Avoid Subqueries in WHERE Clause

```sql
-- ❌ Bad: Subquery in WHERE
SELECT * FROM orders 
WHERE user_id IN (SELECT id FROM users WHERE status = 'ACTIVE');

-- ✅ Good: JOIN
SELECT o.* 
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.status = 'ACTIVE';

-- ✅ Or EXISTS (often faster for subqueries)
SELECT * FROM orders o
WHERE EXISTS (
    SELECT 1 FROM users u 
    WHERE u.id = o.user_id AND u.status = 'ACTIVE'
);
```

## Pagination Optimization

### OFFSET Performance Problem

```sql
-- ❌ Bad: OFFSET becomes slow for large offsets
SELECT * FROM orders 
ORDER BY created_at DESC 
OFFSET 10000 LIMIT 10;

-- Database must fetch and discard 10000 rows
-- Performance degrades linearly with offset
```

### Keyset Pagination (Cursor-based)

```sql
-- ✅ Good: Use WHERE with last seen value
SELECT * FROM orders 
WHERE created_at < '2024-01-15 10:30:00'
ORDER BY created_at DESC 
LIMIT 10;

-- Uses index on created_at
-- Consistent performance regardless of page
```

### Java Implementation

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    // Keyset pagination
    @Query("SELECT o FROM Order o " +
           "WHERE o.createdAt < :lastCreatedAt " +
           "ORDER BY o.createdAt DESC")
    List<Order> findNextPage(
        @Param("lastCreatedAt") LocalDateTime lastCreatedAt,
        Pageable pageable
    );
}

@Service
public class OrderService {
    
    public Page<OrderDTO> getOrdersPage(LocalDateTime lastSeen, int size) {
        Pageable pageable = PageRequest.of(0, size);
        List<Order> orders;
        
        if (lastSeen == null) {
            orders = orderRepository.findAllByOrderByCreatedAtDesc(pageable);
        } else {
            orders = orderRepository.findNextPage(lastSeen, pageable);
        }
        
        return new PageImpl<>(
            orders.stream().map(OrderDTO::fromEntity).collect(Collectors.toList()),
            pageable,
            orders.size()
        );
    }
}
```

## Batch Operations

### Batch Inserts

```java
// ❌ Bad: Individual inserts
for (Order order : orders) {
    orderRepository.save(order); // N separate SQL statements
}

// ✅ Good: Batch insert
@Transactional
public void batchInsert(List<Order> orders) {
    for (int i = 0; i < orders.size(); i++) {
        orderRepository.save(orders.get(i));
        
        // Flush and clear every 50 records
        if (i % 50 == 0) {
            entityManager.flush();
            entityManager.clear();
        }
    }
}

// ✅ Even better: Use JDBC batch
@Modifying
@Query("INSERT INTO Order (user_id, total, status) VALUES (:userId, :total, :status)")
void batchInsert(@Param("userId") Long userId, 
                 @Param("total") BigDecimal total,
                 @Param("status") String status);
```

### Batch Updates

```java
// ❌ Bad: Individual updates
for (Order order : orders) {
    order.setStatus("PROCESSED");
    orderRepository.save(order);
}

// ✅ Good: Single bulk update
@Modifying
@Query("UPDATE Order o SET o.status = 'PROCESSED' WHERE o.id IN :ids")
void bulkUpdateStatus(@Param("ids") List<Long> ids);
```

## Query Hints

### Force Index Usage

```java
// PostgreSQL hint
@Query(value = "SELECT * FROM orders " +
       "WHERE user_id = :userId " +
       "ORDER BY created_at DESC " +
       "LIMIT 10",
       nativeQuery = true)
List<Order> findWithIndexHint(@Param("userId") Long userId);

// MySQL hint
@Query(value = "SELECT * FROM orders USE INDEX (idx_user_created) " +
       "WHERE user_id = :userId " +
       "ORDER BY created_at DESC " +
       "LIMIT 10",
       nativeQuery = true)
List<Order> findWithIndexHint(@Param("userId") Long userId);
```

### Set Fetch Size

```java
@Repository
public class LargeResultSetRepository {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    public void processLargeResultSet() {
        Query query = entityManager.createQuery(
            "SELECT o FROM Order o", Order.class);
        
        // Set fetch size for streaming
        query.setHint("org.hibernate.fetchSize", 1000);
        query.setHint("org.hibernate.readOnly", true);
        
        try (ScrollableResults results = query.unwrap(
                org.hibernate.query.Query.class).scroll()) {
            while (results.next()) {
                Order order = (Order) results.get()[0];
                // Process order
            }
        }
    }
}
```

## Materialized Views

### Creating Materialized Views

```sql
-- PostgreSQL materialized view
CREATE MATERIALIZED VIEW order_summary AS
SELECT 
    user_id,
    COUNT(*) as order_count,
    SUM(total) as total_spent,
    MAX(order_date) as last_order_date
FROM orders
GROUP BY user_id;

-- Refresh materialized view
REFRESH MATERIALIZED VIEW order_summary;

-- Query materialized view (fast)
SELECT * FROM order_summary WHERE user_id = 12345;
```

### Java Integration

```java
@Entity
@Table(name = "order_summary")
@Immutable
public class OrderSummary {
    @Id
    @Column(name = "user_id")
    private Long userId;
    
    @Column(name = "order_count")
    private Long orderCount;
    
    @Column(name = "total_spent")
    private BigDecimal totalSpent;
    
    @Column(name = "last_order_date")
    private LocalDateTime lastOrderDate;
}

@Repository
public interface OrderSummaryRepository extends JpaRepository<OrderSummary, Long> {
    // Queries against materialized view
}
```

## Query Plan Caching

### Parameterized Queries

```java
// ❌ Bad: String concatenation - no plan caching
public List<Order> findByUserId(Long userId) {
    String sql = "SELECT * FROM orders WHERE user_id = " + userId;
    return entityManager.createNativeQuery(sql).getResultList();
}

// ✅ Good: Parameterized query - plan cached
public List<Order> findByUserId(Long userId) {
    Query query = entityManager.createQuery(
        "SELECT o FROM Order o WHERE o.userId = :userId", Order.class);
    query.setParameter("userId", userId);
    return query.getResultList();
}
```

## Stored Procedures

### Calling Stored Procedures

```sql
-- PostgreSQL stored procedure
CREATE OR REPLACE FUNCTION get_user_orders(p_user_id BIGINT)
RETURNS TABLE (
    id BIGINT,
    total NUMERIC,
    order_date TIMESTAMP
) AS $$
BEGIN
    RETURN QUERY
    SELECT o.id, o.total, o.order_date
    FROM orders o
    WHERE o.user_id = p_user_id
    ORDER BY o.order_date DESC;
END;
$$ LANGUAGE plpgsql;
```

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @Query(value = "SELECT * FROM get_user_orders(:userId)", 
           nativeQuery = true)
    List<Order> findUserOrdersStoredProcedure(@Param("userId") Long userId);
}
```

## Performance Monitoring

### Slow Query Logging

```yaml
# application.yml - PostgreSQL
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb?options=-c%20log_min_duration_statement=1000
# Logs queries taking > 1 second

# MySQL
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?logger=Slf4JLogger&profileSQL=true
```

### Custom Query Logging

```java
@Aspect
@Component
public class QueryPerformanceAspect {
    
    @Around("execution(* org.springframework.data.jpa.repository.JpaRepository.*(..))")
    public Object logQueryPerformance(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        long startTime = System.currentTimeMillis();
        
        try {
            Object result = joinPoint.proceed();
            long duration = System.currentTimeMillis() - startTime;
            
            if (duration > 1000) { // Log slow queries (> 1s)
                log.warn("Slow query: {} took {}ms", methodName, duration);
            }
            
            return result;
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            log.error("Query failed: {} after {}ms", methodName, duration, e);
            throw e;
        }
    }
}
```

## Common Pitfalls

### 1. Not Using Prepared Statements

```java
// ❌ Bad: SQL injection risk, no plan caching
String sql = "SELECT * FROM users WHERE name = '" + name + "'";
Query query = entityManager.createNativeQuery(sql);

// ✅ Good: Prepared statement, plan cached
Query query = entityManager.createQuery(
    "SELECT u FROM User u WHERE u.name = :name");
query.setParameter("name", name);
```

### 2. Fetching Too Much Data

```java
// ❌ Bad: Fetches entire table
List<Order> orders = orderRepository.findAll();

// ✅ Good: Pagination
Page<Order> orders = orderRepository.findAll(PageRequest.of(0, 50));
```

### 3. Ignoring Database Statistics

```sql
-- Update statistics for better query plans
ANALYZE orders; -- PostgreSQL
ANALYZE TABLE orders; -- MySQL
```

### 4. Not Using Connection Pooling

```yaml
# ✅ Always use connection pooling
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
```

## Best Practices

1. **Always use parameterized queries** - Security and performance
2. **Select only needed columns** - Reduce data transfer
3. **Use appropriate indexes** - Cover frequently queried columns
4. **Avoid functions on indexed columns** - Prevents index usage
5. **Use JOIN instead of subqueries** - Generally faster
6. **Implement pagination** - Avoid large result sets
7. **Batch operations** - Reduce round trips
8. **Monitor slow queries** - Identify performance issues
9. **Update statistics regularly** - Help query optimizer
10. **Use EXPLAIN ANALYZE** - Understand query plans

## Performance Checklist

- [ ] Are all queries using parameterized statements?
- [ ] Are indexes created on frequently filtered columns?
- [ ] Is SELECT * avoided in production?
- [ ] Are functions not used on indexed columns in WHERE?
- [ ] Is pagination implemented for large result sets?
- [ ] Are batch operations used for bulk inserts/updates?
- [ ] Is connection pooling configured?
- [ ] Are slow queries being monitored?
- [ ] Are database statistics up to date?
- [ ] Are materialized views used for complex aggregations?

## Interview Questions

1. How do you identify slow queries?
2. What is the difference between index scan and sequential scan?
3. How do you optimize pagination for large datasets?
4. What is a covering index and when would you use it?
5. How do you handle the N+1 query problem?
6. What is keyset pagination and why is it better than OFFSET?
7. How do you optimize batch inserts?
8. What are materialized views and when would you use them?

## Related Topics

- [Indexing Strategies](./indexing-strategies.md)
- [ORM Patterns](./orm-patterns.md)
- [Connection Pooling](./connection-pooling-patterns.md)

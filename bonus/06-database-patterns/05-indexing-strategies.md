# Indexing Strategies

## Overview

Database indexes are data structures that improve the speed of data retrieval operations on database tables at the cost of additional writes and storage space. Understanding indexing strategies is crucial for optimizing database performance.

## How Indexes Work

### B-Tree Index Structure

```
B-Tree Index Structure (Simplified):

                    [Root Node]
                   /            \
          [Internal]         [Internal]
          /    |    \         /    |    \
    [Leaf] [Leaf] [Leaf]  [Leaf] [Leaf] [Leaf]
     |      |      |       |      |      |
   Data   Data   Data    Data   Data   Data

Each node contains sorted keys and pointers
Leaf nodes contain actual data or pointers to data
Tree is balanced - all leaf nodes at same depth
```

### Index Lookup Process

```sql
-- Without index: Sequential scan
SELECT * FROM users WHERE email = 'john@example.com';
-- Scans entire table: O(n)

-- With index on email: B-tree lookup
SELECT * FROM users WHERE email = 'john@example.com';
-- B-tree lookup: O(log n)
-- For 1M rows: ~20 index reads vs 1M table reads
```

## Types of Indexes

### B-Tree Index (Default)

```sql
-- Standard B-tree index
CREATE INDEX idx_users_email ON users(email);

-- Best for:
-- - Equality queries (=, IN)
-- - Range queries (>, <, BETWEEN)
-- - ORDER BY on indexed column
-- - Most common use cases

-- Example usage:
SELECT * FROM users WHERE email = 'john@example.com';
SELECT * FROM users WHERE email > 'a@example.com' AND email < 'z@example.com';
SELECT * FROM users ORDER BY email;
```

### Hash Index

```sql
-- Hash index (PostgreSQL)
CREATE INDEX idx_users_email_hash ON users USING HASH(email);

-- Best for:
-- - Equality queries only (=, IN)
-- - No range queries
-- - Faster than B-tree for equality

-- Example usage:
SELECT * FROM users WHERE email = 'john@example.com'; -- Fast
SELECT * FROM users WHERE email > 'a@example.com'; -- Won't use index
```

### GIN Index (Generalized Inverted Index)

```sql
-- GIN index for array columns
CREATE INDEX idx_users_tags ON users USING GIN(tags);

-- Best for:
-- - Array columns
-- - JSON/JSONB columns
-- - Full-text search

-- Example usage:
SELECT * FROM users WHERE 'developer' = ANY(tags);
SELECT * FROM users WHERE tags @> ARRAY['developer', 'java'];
```

### GiST Index (Generalized Search Tree)

```sql
-- GiST index for geometric data
CREATE INDEX idx_locations ON locations USING GIST(coordinates);

-- Best for:
-- - Geometric data
-- - Full-text search
-- - Range types

-- Example usage:
SELECT * FROM locations 
WHERE ST_DWithin(coordinates, ST_MakePoint(-73.9857, 40.7484), 1000);
```

### BRIN Index (Block Range Index)

```sql
-- BRIN index for very large tables
CREATE INDEX idx_orders_created ON orders USING BRIN(created_at);

-- Best for:
-- - Very large tables (billions of rows)
-- - Naturally ordered data (timestamps, IDs)
-- - Low storage overhead

-- Example usage:
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

## Index Design Patterns

### Single Column Index

```sql
-- Simple single column index
CREATE INDEX idx_users_email ON users(email);

-- Use when:
-- - Column frequently used in WHERE clause
-- - Column has high cardinality (many distinct values)
-- - Queries filter on single column

-- Example:
SELECT * FROM users WHERE email = 'john@example.com';
```

### Composite Index

```sql
-- Composite (multi-column) index
CREATE INDEX idx_users_status_created ON users(status, created_at);

-- Order matters! Most selective column first
-- Used for queries that filter on both columns

-- Example queries that use this index:
SELECT * FROM users WHERE status = 'ACTIVE' AND created_at > '2024-01-01';
SELECT * FROM users WHERE status = 'ACTIVE' ORDER BY created_at;
SELECT * FROM users WHERE status = 'ACTIVE';

-- These queries WON'T use the index efficiently:
SELECT * FROM users WHERE created_at > '2024-01-01'; -- Skips first column
SELECT * FROM users WHERE created_at > '2024-01-01' AND status = 'ACTIVE'; -- Wrong order
```

### Covering Index

```sql
-- Covering index includes all columns needed by query
CREATE INDEX idx_orders_covering ON orders(user_id, status, total);

-- Query uses only index, no table access needed
SELECT status, total FROM orders WHERE user_id = 12345;

-- Benefits:
-- - Faster (no table lookup)
-- - Less I/O
-- - Called "index-only scan"

-- Check with EXPLAIN:
EXPLAIN SELECT status, total FROM orders WHERE user_id = 12345;
-- Should show "Index Only Scan"
```

### Partial Index

```sql
-- Index only a subset of rows
CREATE INDEX idx_orders_active ON orders(user_id, created_at) 
WHERE status = 'ACTIVE';

-- Benefits:
-- - Smaller index (faster)
-- - Less storage
-- - Faster index maintenance

-- Example:
SELECT * FROM orders WHERE user_id = 12345 AND status = 'ACTIVE';
-- Uses partial index

SELECT * FROM orders WHERE user_id = 12345 AND status = 'PENDING';
-- Does NOT use partial index
```

### Unique Index

```sql
-- Enforces uniqueness
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Prevents duplicate emails
INSERT INTO users (email) VALUES ('john@example.com'); -- Success
INSERT INTO users (email) VALUES ('john@example.com'); -- Error!

-- Also creates index for queries
SELECT * FROM users WHERE email = 'john@example.com'; -- Uses index
```

### Expression Index (Functional Index)

```sql
-- Index on expression/function result
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- Useful for case-insensitive searches
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';

-- Date extraction
CREATE INDEX idx_orders_month ON orders(EXTRACT(MONTH FROM created_at));
SELECT * FROM orders WHERE EXTRACT(MONTH FROM created_at) = 1;
```

## Index Selection Strategy

### Cardinality Considerations

```sql
-- High cardinality (many distinct values) - Good for indexing
-- Example: email, user_id, phone_number
CREATE INDEX idx_users_email ON users(email); -- Excellent

-- Low cardinality (few distinct values) - Poor for indexing
-- Example: status (ACTIVE, INACTIVE), gender (M, F)
CREATE INDEX idx_users_status ON users(status); -- Limited benefit

-- For low cardinality, consider:
-- - Bitmap index (data warehouse)
-- - Partial index (filter common values)
-- - Don't index at all
```

### Selectivity Calculation

```sql
-- Selectivity = distinct_values / total_rows
-- Higher selectivity = better index effectiveness

-- Calculate selectivity:
SELECT 
    COUNT(DISTINCT email) as distinct_emails,
    COUNT(*) as total_rows,
    COUNT(DISTINCT email)::float / COUNT(*) as selectivity
FROM users;

-- Results:
-- - email: 0.95 (95% distinct) - Excellent for index
-- - status: 0.05 (5% distinct) - Poor for index
```

### Index Usage Heuristics

```sql
-- Index columns used in WHERE clauses
CREATE INDEX idx_orders_user ON orders(user_id);

-- Index columns used in JOIN conditions
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_id ON users(id);

-- Index columns used in ORDER BY
CREATE INDEX idx_orders_created ON orders(created_at);

-- Index columns used in GROUP BY
CREATE INDEX idx_orders_status ON orders(status);

-- DON'T index:
-- - Columns with low selectivity
-- - Columns frequently updated (write overhead)
-- - Very small tables (table scan is faster)
-- - Columns rarely queried
```

## Index Maintenance

### Rebuilding Indexes

```sql
-- PostgreSQL
REINDEX INDEX idx_users_email;
REINDEX TABLE users; -- Rebuild all indexes on table

-- MySQL
ANALYZE TABLE users; -- Update statistics
OPTIMIZE TABLE users; -- Rebuild table and indexes

-- When to rebuild:
-- - After bulk deletes
-- - After bulk updates
-- - Index fragmentation high
-- - Performance degraded
```

### Monitoring Index Usage

```sql
-- PostgreSQL - Check unused indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as index_scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY schemaname, tablename, indexname;

-- MySQL - Check index usage
SHOW INDEX FROM users;
SELECT * FROM sys.schema_unused_indexes;
```

### Index Size Monitoring

```sql
-- PostgreSQL - Check index sizes
SELECT 
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexname::text)) as size
FROM pg_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexname::text) DESC;

-- MySQL - Check index sizes
SELECT 
    table_name,
    index_name,
    ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) as size_mb
FROM mysql.innodb_index_stats
WHERE database_name = 'mydb'
AND stat_name = 'size'
ORDER BY stat_value DESC;
```

## Index Performance Trade-offs

### Write Performance Impact

```sql
-- Indexes slow down INSERT/UPDATE/DELETE
-- Each index must be updated for every write

-- Without index:
INSERT INTO users (email, name) VALUES ('john@example.com', 'John');
-- Fast: 1 write to table

-- With 5 indexes:
INSERT INTO users (email, name) VALUES ('john@example.com', 'John');
-- Slower: 1 write to table + 5 writes to indexes

-- Trade-off:
-- - More indexes = faster reads, slower writes
-- - Fewer indexes = slower reads, faster writes
-- - Balance based on read/write ratio
```

### Storage Overhead

```sql
-- Indexes consume disk space
-- Typically 10-50% of table size per index

-- Check index vs table size:
SELECT 
    pg_size_pretty(pg_relation_size('users')) as table_size,
    pg_size_pretty(pg_total_relation_size('users') - pg_relation_size('users')) as indexes_size;
```

## Java Integration

### JPA Index Annotations

```java
@Entity
@Table(name = "users",
    indexes = {
        @Index(name = "idx_users_email", columnList = "email"),
        @Index(name = "idx_users_status_created", columnList = "status, created_at"),
        @Index(name = "idx_users_name", columnList = "name", unique = false)
    }
)
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @Column(nullable = false)
    private String status;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column
    private String name;
}
```

### Programmatic Index Creation

```java
@Service
public class IndexManagementService {

    @PersistenceContext
    private EntityManager entityManager;

    public void createIndexIfNotExists(String tableName, String indexName, String columns) {
        String checkSql = String.format(
            "SELECT COUNT(*) FROM information_schema.statistics " +
            "WHERE table_schema = DATABASE() AND table_name = '%s' AND index_name = '%s'",
            tableName, indexName
        );
        
        Query checkQuery = entityManager.createNativeQuery(checkSql);
        BigInteger count = (BigInteger) checkQuery.getSingleResult();
        
        if (count.compareTo(BigInteger.ZERO) == 0) {
            String createSql = String.format(
                "CREATE INDEX %s ON %s (%s)",
                indexName, tableName, columns
            );
            entityManager.createNativeQuery(createSql).executeUpdate();
            log.info("Created index {} on table {}", indexName, tableName);
        }
    }
}
```

## Common Pitfalls

### 1. Over-Indexing

```sql
-- ❌ Bad: Too many indexes
CREATE INDEX idx1 ON users(email);
CREATE INDEX idx2 ON users(name);
CREATE INDEX idx3 ON users(status);
CREATE INDEX idx4 ON users(created_at);
CREATE INDEX idx5 ON users(updated_at);
CREATE INDEX idx6 ON users(email, name);
CREATE INDEX idx7 ON users(email, status);
-- ... 20+ indexes on one table

-- Problems:
-- - Slow writes (all indexes updated)
-- - High storage cost
-- - Confusing for optimizer
-- - Maintenance overhead
```

### 2. Indexing Low-Cardinality Columns

```sql
-- ❌ Bad: Indexing boolean column
CREATE INDEX idx_users_active ON users(is_active);
-- Only 2 distinct values (true/false)
-- Index rarely used, high overhead

-- ✅ Better: Partial index for rare value
CREATE INDEX idx_users_inactive ON users(id) WHERE is_active = false;
-- Only index inactive users (small subset)
```

### 3. Wrong Column Order in Composite Index

```sql
-- ❌ Bad: Wrong order
CREATE INDEX idx_orders_status_user ON orders(status, user_id);

-- Query:
SELECT * FROM orders WHERE user_id = 12345;
-- Won't use index efficiently (skips first column)

-- ✅ Good: Correct order
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
-- Query uses index
```

### 4. Not Considering Query Patterns

```sql
-- ❌ Bad: Index doesn't match query pattern
CREATE INDEX idx_orders_created ON orders(created_at);

-- Query:
SELECT * FROM orders WHERE user_id = 12345 ORDER BY created_at;
-- Index not used for filtering, only for sorting

-- ✅ Good: Composite index matches query
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);
-- Index used for both filtering and sorting
```

### 5. Ignoring Index Maintenance

```sql
-- ❌ Bad: Never rebuilding indexes
-- Indexes become fragmented over time
-- Performance degrades

-- ✅ Good: Regular maintenance
-- Schedule periodic index rebuilds
-- Monitor index usage and size
```

## Best Practices

1. **Index frequently queried columns** - WHERE, JOIN, ORDER BY
2. **Use composite indexes wisely** - Order matters
3. **Consider selectivity** - High cardinality columns first
4. **Use covering indexes** - Include all query columns
5. **Use partial indexes** - For common filter conditions
6. **Monitor index usage** - Remove unused indexes
7. **Balance read/write performance** - Don't over-index
8. **Rebuild indexes periodically** - Reduce fragmentation
9. **Test index changes** - Measure performance impact
10. **Document index purpose** - Why each index exists

## Index Design Checklist

- [ ] Does the column have high selectivity?
- [ ] Is the column frequently used in WHERE clauses?
- [ ] Is the column frequently used in JOIN conditions?
- [ ] Is the column frequently used in ORDER BY?
- [ ] Is the column frequently used in GROUP BY?
- [ ] Will the index benefit read performance more than it hurts write performance?
- [ ] Is the index size acceptable?
- [ ] Is the index order correct for query patterns?
- [ ] Can a covering index be used?
- [ ] Can a partial index be used?

## Performance Impact Summary

| Index Type | Read Speed | Write Speed | Storage | Use Case |
|------------|------------|--------------|---------|----------|
| B-Tree | Fast | Slower | Medium | General purpose |
| Hash | Very Fast (equality) | Slower | Low | Equality only |
| GIN | Fast (arrays) | Slower | High | Arrays, JSON |
| GiST | Fast (spatial) | Slower | High | Geometric data |
| BRIN | Medium | Fast | Very Low | Large ordered data |

## Interview Questions

1. How does a B-tree index work?
2. What is the difference between a single-column and composite index?
3. What is a covering index and when would you use it?
4. How do you decide which columns to index?
5. What is the impact of indexes on write performance?
6. What is a partial index and when would you use it?
7. How do you monitor index usage?
8. What is the difference between B-tree and hash indexes?

## Related Topics

- [Query Optimization](./query-optimization.md)
- [ORM Patterns](./orm-patterns.md)
- [Database Sharding](./database-sharding.md)

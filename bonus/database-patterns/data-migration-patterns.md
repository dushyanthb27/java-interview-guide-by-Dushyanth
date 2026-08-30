# Data Migration Patterns

## Overview

Data migration involves transferring data between storage types, formats, or computer systems. In database development, schema migrations are essential for evolving database structure while preserving existing data. This guide covers patterns and tools for managing database migrations in Java applications.

## Migration Challenges

### Common Migration Issues

```java
// ❌ Common problems:
// 1. Breaking changes - existing code fails
// 2. Data loss - columns dropped without backup
// 3. Downtime - application unavailable during migration
// 4. Rollback failures - can't undo changes
// 5. Inconsistent state - partial migration
// 6. Performance impact - migration blocks production
```

## Migration Tools

### Flyway - Database Migration Tool

Flyway is the most popular database migration tool for Java applications.

### Configuration

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    baseline-version: 0
    validate-on-migrate: true
    out-of-order: false
    clean-disabled: true  # Important for production!
    
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: myuser
    password: mypassword
```

### Migration Scripts Structure

```
src/main/resources/db/migration/
├── V1__Create_users_table.sql
├── V2__Create_orders_table.sql
├── V3__Add_email_index.sql
├── V4__Add_status_column.sql
├── V5__Create_order_items_table.sql
└── V6__Add_foreign_keys.sql
```

### Basic Migration Scripts

```sql
-- V1__Create_users_table.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- V2__Create_orders_table.sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    total DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    order_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_orders_user FOREIGN KEY (user_id) REFERENCES users(id)
);

-- V3__Add_email_index.sql
CREATE INDEX idx_users_email ON users(email);

-- V4__Add_status_column.sql
ALTER TABLE users ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE';
```

### Repeatable Migrations

```sql
-- R__seed_data.sql
-- Repeatable migrations run every time checksum changes
INSERT INTO users (username, email, password_hash, status) 
VALUES 
    ('admin', 'admin@example.com', '$2a$10$...', 'ACTIVE')
ON CONFLICT (username) DO NOTHING;
```

### Java-Based Migrations

```java
import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;

public class V5__Populate_initial_data extends BaseJavaMigration {
    
    @Override
    public void migrate(Context context) throws Exception {
        try (PreparedStatement statement = 
                context.getConnection().prepareStatement(
                    "INSERT INTO users (username, email, password_hash, status) " +
                    "VALUES (?, ?, ?, ?)")) {
            
            statement.setString(1, "user1");
            statement.setString(2, "user1@example.com");
            statement.setString(3, "$2a$10$...");
            statement.setString(4, "ACTIVE");
            statement.executeUpdate();
            
            statement.setString(1, "user2");
            statement.setString(2, "user2@example.com");
            statement.setString(3, "$2a$10$...");
            statement.setString(4, "ACTIVE");
            statement.executeUpdate();
        }
    }
}
```

## Liquibase Alternative

### Configuration

```yaml
# application.yml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
    contexts: prod
```

### Changelog Structure

```yaml
# db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - include:
      file: db/changelog/changeset-001-create-users.yaml
      relativeToChangelogFile: true
  - include:
      file: db/changelog/changeset-002-create-orders.yaml
      relativeToChangelogFile: true
```

### Changeset Example

```yaml
# db/changelog/changeset-001-create-users.yaml
databaseChangeLog:
  - changeSet:
      id: create-users-table
      author: developer
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: username
                  type: VARCHAR(50)
                  constraints:
                    nullable: false
                    unique: true
              - column:
                  name: email
                  type: VARCHAR(100)
                  constraints:
                    nullable: false
                    unique: true
              - column:
                  name: password_hash
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
              - column:
                  name: created_at
                  type: TIMESTAMP
                  defaultValueComputed: CURRENT_TIMESTAMP
                  constraints:
                    nullable: false
```

## Migration Patterns

### Additive Changes (Safe)

```sql
-- ✅ Safe: Add new table
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    operation VARCHAR(10) NOT NULL,
    old_data JSONB,
    new_data JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- ✅ Safe: Add new column with default
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER TABLE users ALTER COLUMN phone SET DEFAULT NULL;

-- ✅ Safe: Add new index
CREATE INDEX CONCURRENTLY idx_users_phone ON users(phone);
-- CONCURRENTLY avoids locking table
```

### Breaking Changes (Require Care)

```sql
-- ⚠️ Breaking: Drop column
-- Must ensure no code references this column
BEGIN;
-- 1. Deploy new code that doesn't use the column
-- 2. Wait for deployment to complete
-- 3. Drop column
ALTER TABLE users DROP COLUMN old_column;
COMMIT;

-- ⚠️ Breaking: Rename column
-- Must update all code references
BEGIN;
ALTER TABLE users RENAME COLUMN username TO user_name;
COMMIT;

-- ⚠️ Breaking: Change column type
-- Must ensure data conversion is safe
BEGIN;
ALTER TABLE users ALTER COLUMN total TYPE DECIMAL(12, 2) USING total::DECIMAL(12, 2);
COMMIT;
```

### Data Migration

```sql
-- Migrate data from old schema to new schema
-- V6__Migrate_user_status.sql

-- Step 1: Add new column
ALTER TABLE users ADD COLUMN account_status VARCHAR(20);

-- Step 2: Migrate data
UPDATE users 
SET account_status = 
    CASE 
        WHEN status = 'A' THEN 'ACTIVE'
        WHEN status = 'I' THEN 'INACTIVE'
        WHEN status = 'S' THEN 'SUSPENDED'
        ELSE 'UNKNOWN'
    END;

-- Step 3: Add constraint
ALTER TABLE users 
ADD CONSTRAINT chk_account_status 
CHECK (account_status IN ('ACTIVE', 'INACTIVE', 'SUSPENDED'));

-- Step 4: Make NOT NULL after migration
ALTER TABLE users ALTER COLUMN account_status SET NOT NULL;

-- Step 5: Drop old column (in separate migration)
-- V7__Drop_old_status_column.sql
ALTER TABLE users DROP COLUMN status;
```

### Zero-Downtime Migration

```sql
-- Pattern: Expand, Contract, Contract

-- Phase 1: Expand - Add new column alongside old
-- V8__Add_new_status_column.sql
ALTER TABLE users ADD COLUMN new_status VARCHAR(20);
UPDATE users SET new_status = status;

-- Phase 2: Deploy new code that writes to both columns
-- Application writes to both old_status and new_status

-- Phase 3: Backfill data
-- V9__Backfill_new_status.sql
UPDATE users SET new_status = status WHERE new_status IS NULL;

-- Phase 4: Deploy new code that reads from new column
-- Application reads from new_status

-- Phase 5: Contract - Drop old column
-- V10__Drop_old_status_column.sql
ALTER TABLE users DROP COLUMN status;
ALTER TABLE users RENAME COLUMN new_status TO status;
```

## Rollback Strategies

### Flyway Rollback

```sql
-- Flyway doesn't support automatic rollback
-- Must write manual rollback scripts

-- V4__Add_status_column.sql
ALTER TABLE users ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE';

-- V4__rollback.sql
ALTER TABLE users DROP COLUMN status;
```

### Transactional Migrations

```sql
-- Wrap migrations in transactions for atomicity
BEGIN;

ALTER TABLE users ADD COLUMN temp_column VARCHAR(100);
UPDATE users SET temp_column = email;
ALTER TABLE users DROP COLUMN email;
ALTER TABLE users RENAME COLUMN temp_column TO email;

COMMIT;

-- If any step fails, all changes rolled back
```

### Backup Before Migration

```java
@Service
public class MigrationBackupService {

    @Autowired
    private DataSource dataSource;

    public void createBackup(String tableName) {
        String backupTable = tableName + "_backup_" + 
            LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss"));
        
        String sql = String.format("CREATE TABLE %s AS SELECT * FROM %s", 
            backupTable, tableName);
        
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute(sql);
            log.info("Created backup table: {}", backupTable);
        } catch (SQLException e) {
            throw new RuntimeException("Backup failed", e);
        }
    }

    public void restoreFromBackup(String backupTable, String originalTable) {
        String sql = String.format("TRUNCATE TABLE %s; INSERT INTO %s SELECT * FROM %s",
            originalTable, originalTable, backupTable);
        
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            stmt.execute(sql);
            log.info("Restored {} from {}", originalTable, backupTable);
        } catch (SQLException e) {
            throw new RuntimeException("Restore failed", e);
        }
    }
}
```

## Large Table Migrations

### Batch Processing

```java
@Service
public class LargeTableMigrationService {

    @PersistenceContext
    private EntityManager entityManager;

    @Transactional
    public void migrateLargeTableInBatches(int batchSize) {
        long offset = 0;
        boolean hasMore = true;
        
        while (hasMore) {
            String sql = String.format(
                "UPDATE users SET new_status = status " +
                "WHERE id > %d ORDER BY id LIMIT %d",
                offset, batchSize
            );
            
            Query query = entityManager.createNativeQuery(sql);
            int updated = query.executeUpdate();
            
            entityManager.flush();
            entityManager.clear();
            
            offset += batchSize;
            hasMore = updated > 0;
            
            log.info("Migrated {} rows, total: {}", updated, offset);
        }
    }
}
```

### Online Schema Change (pt-online-schema-change)

```bash
# MySQL tool for online schema changes
pt-online-schema-change \
  --alter "ADD COLUMN phone VARCHAR(20)" \
  --user=myuser \
  --password=mypassword \
  --host=localhost \
  D=mydb,t=users \
  --execute

# Creates shadow table, copies data, swaps tables
# Minimal locking, no downtime
```

## Version Control Strategy

### Branch-Based Migrations

```
main branch:
  V1__Create_users_table.sql
  V2__Create_orders_table.sql
  V3__Add_email_index.sql

feature/new-payment-method branch:
  V4__Add_payment_method_column.sql
  V5__Create_payment_transactions.sql

feature/user-preferences branch:
  V4__Add_preferences_column.sql
  V6__Create_user_preferences.sql

# Problem: Both branches use V4, V5, V6
# Solution: Use different version numbers or merge carefully
```

### Version Numbering Strategy

```sql
# Option 1: Sequential numbers
V1__initial_schema.sql
V2__add_indexes.sql
V3__add_status_column.sql

# Option 2: Timestamp-based
V20240115000000__initial_schema.sql
V20240115120000__add_indexes.sql
V20240115200000__add_status_column.sql

# Option 3: Feature-based with numbers
V1.0__initial_schema.sql
V1.1__add_indexes.sql
V2.0__add_status_column.sql
```

## Environment-Specific Migrations

### Context-Based Migrations

```yaml
# Flyway with contexts
spring:
  flyway:
    contexts: dev,prod

# Migration files:
# V1__schema.sql (runs in all contexts)
# V2__dev_data.sql (runs only in dev context)
# V3__prod_index.sql (runs only in prod context)
```

### Conditional Migrations

```java
public class ConditionalMigration extends BaseJavaMigration {
    
    @Override
    public void migrate(Context context) throws Exception {
        String environment = System.getProperty("spring.profiles.active");
        
        if ("dev".equals(environment)) {
            // Dev-specific migration
            try (Statement stmt = context.getConnection().createStatement()) {
                stmt.execute("INSERT INTO users (username, email) VALUES ('test', 'test@example.com')");
            }
        }
    }
}
```

## Validation and Testing

### Pre-Migration Validation

```java
@Service
public class MigrationValidator {

    @PersistenceContext
    private EntityManager entityManager;

    public void validateBeforeMigration(String tableName, String columnName) {
        // Check if table exists
        Long tableCount = (Long) entityManager.createNativeQuery(
            "SELECT COUNT(*) FROM information_schema.tables " +
            "WHERE table_name = :tableName"
        ).setParameter("tableName", tableName).getSingleResult();
        
        if (tableCount == 0) {
            throw new MigrationException("Table " + tableName + " does not exist");
        }
        
        // Check if column exists
        Long columnCount = (Long) entityManager.createNativeQuery(
            "SELECT COUNT(*) FROM information_schema.columns " +
            "WHERE table_name = :tableName AND column_name = :columnName"
        ).setParameter("tableName", tableName)
         .setParameter("columnName", columnName).getSingleResult();
        
        if (columnCount == 0) {
            throw new MigrationException("Column " + columnName + " does not exist");
        }
    }
}
```

### Post-Migration Validation

```java
@Service
public class PostMigrationValidator {

    @PersistenceContext
    private EntityManager entityManager;

    public void validateDataIntegrity(String tableName) {
        // Check row count
        Long rowCount = (Long) entityManager.createNativeQuery(
            "SELECT COUNT(*) FROM " + tableName
        ).getSingleResult();
        
        log.info("Table {} has {} rows", tableName, rowCount);
        
        // Check for null values in NOT NULL columns
        List<Object[]> nullViolations = entityManager.createNativeQuery(
            "SELECT column_name FROM information_schema.columns " +
            "WHERE table_name = :tableName AND is_nullable = 'NO'"
        ).setParameter("tableName", tableName).getResultList();
        
        for (Object[] column : nullViolations) {
            String columnName = (String) column[0];
            Long nullCount = (Long) entityManager.createNativeQuery(
                "SELECT COUNT(*) FROM " + tableName + " WHERE " + columnName + " IS NULL"
            ).getSingleResult();
            
            if (nullCount > 0) {
                throw new MigrationException(
                    "Found " + nullCount + " NULL values in NOT NULL column " + columnName
                );
            }
        }
    }
}
```

## Common Pitfalls

### 1. Not Testing Migrations

```java
// ❌ Bad: Running migrations directly in production
// No testing, high risk of failure

// ✅ Good: Test in staging first
// 1. Run migration in staging
// 2. Verify application works
// 3. Check performance
// 4. Then run in production
```

### 2. Breaking Changes Without Rollback

```sql
-- ❌ Bad: No rollback script
ALTER TABLE users DROP COLUMN email;

-- ✅ Good: Always provide rollback
-- V4__drop_email_column.sql
ALTER TABLE users DROP COLUMN email;

-- V4__rollback.sql
ALTER TABLE users ADD COLUMN email VARCHAR(100);
-- Note: Data lost! Better to use expand-contract pattern
```

### 3. Long-Running Migrations

```sql
-- ❌ Bad: Single large migration
UPDATE users SET status = 'ACTIVE'; -- 10M rows, takes hours

-- ✅ Good: Batch processing
-- Process in batches, can be paused/resumed
```

### 4. Ignoring Dependencies

```sql
-- ❌ Bad: Wrong order
-- V2__add_foreign_key.sql
ALTER TABLE orders ADD CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id);
-- But users table created in V3!

-- ✅ Good: Correct order
-- V2__create_users.sql
CREATE TABLE users (...);

-- V3__create_orders.sql
CREATE TABLE orders (...);

-- V4__add_foreign_key.sql
ALTER TABLE orders ADD CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id);
```

### 5. Not Using Transactions

```sql
-- ❌ Bad: No transaction
ALTER TABLE users ADD COLUMN status VARCHAR(20);
UPDATE users SET status = 'ACTIVE'; -- If this fails, column added but not populated

-- ✅ Good: Use transaction
BEGIN;
ALTER TABLE users ADD COLUMN status VARCHAR(20);
UPDATE users SET status = 'ACTIVE';
COMMIT;
```

## Best Practices

1. **Always use version control** - Track all migrations
2. **Test migrations in staging** - Never test in production
3. **Use transactions** - Ensure atomicity
4. **Provide rollback scripts** - Always have a way back
5. **Use additive changes** - Prefer adding over modifying
6. **Batch large operations** - Avoid long-running migrations
7. **Document migrations** - Explain why changes are needed
8. **Validate after migration** - Ensure data integrity
9. **Use CONCURRENTLY for indexes** - Avoid locking
10. **Plan for zero downtime** - Use expand-contract pattern

## Migration Checklist

- [ ] Migration tested in staging environment?
- [ ] Rollback script written and tested?
- [ ] Data backup created before migration?
- [ ] Migration uses transaction?
- [ ] Large operations batched?
- [ ] Application code updated?
- [ ] Performance impact assessed?
- [ ] Dependencies verified?
- [ ] Post-migration validation planned?
- [ ] Rollback procedure documented?

## Interview Questions

1. How do you handle database schema migrations in production?
2. What is the expand-contract pattern for zero-downtime migrations?
3. How do you rollback a failed migration?
4. What is the difference between Flyway and Liquibase?
5. How do you handle large table migrations?
6. What are the risks of schema migrations?
7. How do you ensure data integrity during migration?
8. What is a repeatable migration?

## Related Topics

- [Transaction Management](./transaction-management.md)
- [ORM Patterns](./orm-patterns.md)
- [Database Sharding](./database-sharding.md)

# Case Study 9: Zero-Downtime Database Migration

## Difficulty: Hard 🔴

---

## Scenario Context

### Company Background
**HealthTrack** is a healthcare data platform managing electronic health records (EHR) for 200+ hospitals and clinics. The platform processes 5 million patient records, 50 million lab results, and handles 10,000 concurrent users during peak hours.

### Technical Stack
- **Application**: Spring Boot monolith (transitioning to microservices)
- **Current Database**: MySQL 5.7 (single master, 2 replicas)
- **Target Database**: PostgreSQL 14 (with partitioning and extensions)
- **Cache**: Redis cluster
- **Message Queue**: Apache Kafka
- **Deployment**: Kubernetes with blue-green deployments
- **Data Volume**: 2TB total, growing at 100GB/month

### Business Context
The company was experiencing:
- **Performance Bottlenecks**: MySQL struggling with complex healthcare queries
- **Scalability Limits**: Single master architecture limiting write throughput
- **Feature Requirements**: Need for PostgreSQL-specific features (JSONB, full-text search)
- **Compliance Requirements**: New healthcare regulations requiring better audit logging
- **Cost Issues**: MySQL Enterprise licensing becoming expensive

### Constraints
- **Zero Downtime**: Healthcare system must remain available 24/7
- **Data Integrity**: Cannot lose or corrupt any patient data
- **HIPAA Compliance**: Must maintain audit trail during migration
- **Performance**: Cannot degrade performance during migration
- **Rollback**: Must be able to rollback if issues arise
- **Timeline**: 3-month migration window

---

## Problem Statement

**Primary Challenge**: Migrate 2TB of healthcare data from MySQL to PostgreSQL with zero downtime and zero data loss.

**Specific Problems**:
1. How to migrate 2TB of data without downtime?
2. How to keep data synchronized during migration?
3. How to handle schema differences between databases?
4. How to validate data integrity after migration?
5. How to rollback if issues arise?

**Success Criteria**:
- Zero downtime during migration
- Zero data loss or corruption
- Performance within 10% of baseline during migration
- Complete data validation
- Successful rollback capability
- HIPAA compliance maintained throughout

---

## Analysis Phase

### Current Database Assessment

**MySQL Schema Analysis**:
```sql
-- Key tables
patients (5M rows, 50GB)
lab_results (50M rows, 800GB)
prescriptions (20M rows, 300GB)
appointments (30M rows, 200GB)
medical_records (100M rows, 600GB)
audit_log (200M rows, 500GB)

-- Total: 2TB across 200+ tables
```

**Migration Complexity Assessment**:

| Table | Rows | Size | Complexity | Dependencies |
|-------|------|------|------------|--------------|
| patients | 5M | 50GB | Medium | prescriptions, appointments |
| lab_results | 50M | 800GB | High | patients, medical_records |
| prescriptions | 20M | 300GB | High | patients, drug_database |
| appointments | 30M | 200GB | Medium | patients, providers |
| medical_records | 100M | 600GB | Very High | patients, lab_results |
| audit_log | 200M | 500GB | Low | All tables |

**Schema Differences**:

```sql
-- MySQL (current)
CREATE TABLE patients (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    date_of_birth DATE NOT NULL,
    ssn VARCHAR(11) ENCRYPTED,
    medical_history TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_last_name (last_name),
    INDEX idx_dob (date_of_birth)
) ENGINE=InnoDB;

-- PostgreSQL (target)
CREATE TABLE patients (
    id BIGSERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    date_of_birth DATE NOT NULL,
    ssn VARCHAR(11) ENCRYPTED WITH pgcrypto,
    medical_history JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_ssn_format CHECK (ssn ~ '^\d{3}-\d{2}-\d{4}$')
);

CREATE INDEX idx_patients_last_name ON patients(last_name);
CREATE INDEX idx_patients_dob ON patients(date_of_birth);
CREATE INDEX idx_patients_full_text ON patients 
    USING gin(to_tsvector('english', first_name || ' ' || last_name));
```

### Risk Assessment

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Data corruption during migration | Critical | Medium | Checksums, validation, rollback |
| Performance degradation | High | High | Throttling, off-peak migration |
| Schema mapping errors | High | Medium | Extensive testing, data validation |
| Sync failures during cutover | Critical | Low | Dual-write, verification |
| HIPAA compliance violation | Critical | Low | Audit logging, encryption |
| Insufficient storage | Medium | Low | Capacity planning, monitoring |

---

## Solution Approach

### Strategy: Dual-Write with CDC and Gradual Cutover

**Migration Phases**:
1. **Phase 1**: Schema creation and data transformation setup
2. **Phase 2**: Initial bulk data migration (offline)
3. **Phase 3**: Change Data Capture (CDC) for ongoing sync
4. **Phase 4**: Dual-write period (write to both databases)
5. **Phase 5**: Validation and verification
6. **Phase 6**: Gradual traffic cutover
7. **Phase 7**: Decommission MySQL

**Architecture**:
```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Read Router │  │ Write Router │  │ Data Mapper  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   MySQL      │    │ PostgreSQL   │    │   CDC Tool   │
│  (Legacy)    │◄───│  (Target)    │◄───│  (Debezium)  │
└──────────────┘    └──────────────┘    └──────────────┘
         │                    ▲
         └────────────────────┘
              Sync during migration
```

---

## Implementation Details

### Phase 1: Schema Creation and Data Transformation

#### Database Schema Converter

```java
@Service
public class SchemaMigrationService {
    
    @Autowired
    private JdbcTemplate mysqlTemplate;
    
    @Autowired
    private JdbcTemplate postgresTemplate;
    
    public void convertSchema(String tableName) {
        // Read MySQL schema
        String mysqlSchema = getMysqlSchema(tableName);
        
        // Convert to PostgreSQL
        String postgresSchema = convertToPostgres(mysqlSchema);
        
        // Create PostgreSQL table
        postgresTemplate.execute(postgresSchema);
        
        // Create indexes
        createIndexes(tableName);
        
        // Create constraints
        createConstraints(tableName);
    }
    
    private String convertToPostgres(String mysqlSchema) {
        // Convert AUTO_INCREMENT to SERIAL/BIGSERIAL
        String converted = mysqlSchema.replaceAll(
            "AUTO_INCREMENT", "SERIAL");
        
        // Convert ENGINE=InnoDB (PostgreSQL doesn't need it)
        converted = converted.replaceAll(
            "ENGINE=InnoDB", "");
        
        // Convert TINYINT to BOOLEAN
        converted = converted.replaceAll(
            "TINYINT\\(1\\)", "BOOLEAN");
        
        // Convert DATETIME to TIMESTAMP
        converted = converted.replaceAll(
            "DATETIME", "TIMESTAMP");
        
        // Convert TEXT to JSONB where appropriate
        converted = converted.replaceAll(
            "medical_history TEXT", 
            "medical_history JSONB");
        
        return converted;
    }
    
    private void createIndexes(String tableName) {
        // Get MySQL indexes
        List<IndexInfo> mysqlIndexes = getMysqlIndexes(tableName);
        
        // Create equivalent PostgreSQL indexes
        for (IndexInfo index : mysqlIndexes) {
            String createIndexSql = String.format(
                "CREATE INDEX %s ON %s (%s)",
                index.getName(), tableName, index.getColumns());
            postgresTemplate.execute(createIndexSql);
        }
        
        // Add PostgreSQL-specific indexes
        if (tableName.equals("patients")) {
            postgresTemplate.execute(
                "CREATE INDEX idx_patients_full_text ON patients " +
                "USING gin(to_tsvector('english', " +
                "first_name || ' ' || last_name))");
        }
    }
}
```

#### Data Type Mapper

```java
@Component
public class DataTypeMapper {
    
    public Object mapValue(Object value, String mysqlType, 
                          String postgresType) {
        if (value == null) {
            return null;
        }
        
        // Handle specific type conversions
        switch (mysqlType.toUpperCase()) {
            case "TINYINT(1)":
                // MySQL TINYINT(1) to PostgreSQL BOOLEAN
                return ((Number) value).intValue() != 0;
                
            case "DATETIME":
            case "TIMESTAMP":
                // Ensure proper timestamp format
                if (value instanceof String) {
                    return Timestamp.valueOf((String) value);
                }
                return value;
                
            case "TEXT":
                // Convert TEXT to JSONB if target is JSONB
                if (postgresType.equalsIgnoreCase("JSONB")) {
                    try {
                        return objectMapper.readValue(
                            (String) value, JsonNode.class);
                    } catch (JsonProcessingException e) {
                        // If not valid JSON, wrap as JSON string
                        return objectMapper.valueToTree(value);
                    }
                }
                return value;
                
            case "ENUM":
                // Convert MySQL ENUM to PostgreSQL ENUM or VARCHAR
                return value.toString();
                
            default:
                return value;
        }
    }
}
```

### Phase 2: Initial Bulk Data Migration

#### Bulk Data Migrator

```java
@Service
public class BulkDataMigrationService {
    
    @Autowired
    private JdbcTemplate mysqlTemplate;
    
    @Autowired
    private JdbcTemplate postgresTemplate;
    
    @Autowired
    private DataTypeMapper dataTypeMapper;
    
    @Autowired
    private MigrationProgressRepository progressRepository;
    
    private static final int BATCH_SIZE = 10000;
    
    public void migrateTable(String tableName) {
        log.info("Starting bulk migration for table: {}", tableName);
        
        // Get total row count
        Long totalRows = mysqlTemplate.queryForObject(
            "SELECT COUNT(*) FROM " + tableName, Long.class);
        
        // Create progress tracker
        MigrationProgress progress = new MigrationProgress(
            tableName, totalRows, 0L, "IN_PROGRESS");
        progressRepository.save(progress);
        
        // Get column metadata
        List<ColumnInfo> columns = getTableColumns(tableName);
        
        // Migrate in batches
        long offset = 0;
        long migrated = 0;
        
        while (offset < totalRows) {
            // Fetch batch from MySQL
            String query = String.format(
                "SELECT * FROM %s LIMIT %d OFFSET %d",
                tableName, BATCH_SIZE, offset);
            
            List<Map<String, Object>> batch = mysqlTemplate.queryForList(query);
            
            // Transform and insert into PostgreSQL
            insertBatch(tableName, columns, batch);
            
            // Update progress
            migrated += batch.size();
            offset += BATCH_SIZE;
            
            progress.setMigratedRows(migrated);
            progress.setPercentage((migrated * 100) / totalRows);
            progressRepository.save(progress);
            
            log.info("Migrated {}/{} rows for table {}", 
                migrated, totalRows, tableName);
            
            // Throttle to avoid performance impact
            throttleMigration();
        }
        
        // Mark as complete
        progress.setStatus("COMPLETED");
        progress.setCompletedAt(Instant.now());
        progressRepository.save(progress);
        
        log.info("Completed bulk migration for table: {}", tableName);
    }
    
    private void insertBatch(String tableName, List<ColumnInfo> columns,
                            List<Map<String, Object>> batch) {
        String columnNames = columns.stream()
            .map(ColumnInfo::getName)
            .collect(Collectors.joining(", "));
        
        String placeholders = columns.stream()
            .map(c -> "?")
            .collect(Collectors.joining(", "));
        
        String insertSql = String.format(
            "INSERT INTO %s (%s) VALUES (%s) " +
            "ON CONFLICT (id) DO UPDATE SET %s",
            tableName, columnNames, placeholders,
            buildUpdateClause(columns));
        
        postgresTemplate.batchUpdate(insertSql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) 
                    throws SQLException {
                Map<String, Object> row = batch.get(i);
                for (int j = 0; j < columns.size(); j++) {
                    ColumnInfo col = columns.get(j);
                    Object value = row.get(col.getName());
                    value = dataTypeMapper.mapValue(
                        value, col.getMysqlType(), col.getPostgresType());
                    ps.setObject(j + 1, value);
                }
            }
            
            @Override
            public int getBatchSize() {
                return batch.size();
            }
        });
    }
    
    private String buildUpdateClause(List<ColumnInfo> columns) {
        return columns.stream()
            .filter(c -> !c.getName().equals("id"))
            .map(c -> c.getName() + " = EXCLUDED." + c.getName())
            .collect(Collectors.joining(", "));
    }
    
    private void throttleMigration() {
        try {
            // Sleep to avoid overwhelming the database
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

#### Parallel Migration Executor

```java
@Service
public class ParallelMigrationExecutor {
    
    @Autowired
    private BulkDataMigrationService bulkMigrationService;
    
    @Autowired
    private TaskExecutor taskExecutor;
    
    private final List<String> migrationOrder = Arrays.asList(
        "patients",           // Independent tables first
        "providers",
        "drug_database",
        "lab_results",        // Dependent tables
        "prescriptions",
        "appointments",
        "medical_records",    // Most complex
        "audit_log"           // Can migrate last
    );
    
    public void executeMigration() {
        // Migrate independent tables in parallel
        List<CompletableFuture<Void>> futures = new ArrayList<>();
        
        // Phase 1: Independent tables
        futures.add(CompletableFuture.runAsync(() -> 
            bulkMigrationService.migrateTable("patients"), taskExecutor));
        futures.add(CompletableFuture.runAsync(() -> 
            bulkMigrationService.migrateTable("providers"), taskExecutor));
        futures.add(CompletableFuture.runAsync(() -> 
            bulkMigrationService.migrateTable("drug_database"), taskExecutor));
        
        // Wait for phase 1 to complete
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        
        // Phase 2: Dependent tables
        futures.clear();
        futures.add(CompletableFuture.runAsync(() -> 
            bulkMigrationService.migrateTable("lab_results"), taskExecutor));
        futures.add(CompletableFuture.runAsync(() -> 
            bulkMigrationService.migrateTable("prescriptions"), taskExecutor));
        futures.add(CompletableFuture.runAsync(() -> 
            bulkMigrationService.migrateTable("appointments"), taskExecutor));
        
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        
        // Phase 3: Complex tables
        bulkMigrationService.migrateTable("medical_records");
        bulkMigrationService.migrateTable("audit_log");
    }
}
```

### Phase 3: Change Data Capture (CDC) Setup

#### Debezium Configuration for MySQL

```yaml
# application.yml
debezium:
  connector:
    name: mysql-connector
    database:
      hostname: mysql-host
      port: 3306
      user: debezium
      password: ${DEBEZIUM_PASSWORD}
      server.id: 184054
      server.name: healthtrack-mysql
      include.schema: false
      database.include.list: healthtrack
      table.include.list: healthtrack.patients,healthtrack.lab_results
      snapshot.mode: schema_only
      snapshot.locking.mode: minimal
      binlog.buffer.size: 0
    event:
      deserialization.failure.handling.mode: warn
  transforms:
    - route: add_source_type
      type: org.apache.kafka.connect.transforms.InsertField
      fields:
        - field: source_type
          value: mysql
```

#### CDC Event Consumer

```java
@Service
public class CdcEventConsumer {
    
    @Autowired
    private JdbcTemplate postgresTemplate;
    
    @Autowired
    private DataTypeMapper dataTypeMapper;
    
    @KafkaListener(topics = "debezium.healthtrack")
    public void handleCdcEvent(String eventJson) {
        try {
            CdcEvent event = objectMapper.readValue(eventJson, CdcEvent.class);
            
            switch (event.getOperation()) {
                case "c": // Create
                    handleInsert(event);
                    break;
                case "u": // Update
                    handleUpdate(event);
                    break;
                case "d": // Delete
                    handleDelete(event);
                    break;
                default:
                    log.warn("Unknown operation: {}", event.getOperation());
            }
        } catch (Exception e) {
            log.error("Failed to process CDC event", e);
            // Send to dead letter queue for manual inspection
            deadLetterQueue.send(eventJson);
        }
    }
    
    private void handleInsert(CdcEvent event) {
        String tableName = event.getSource().getTable();
        Map<String, Object> data = event.getAfter();
        
        String columns = String.join(", ", data.keySet());
        String placeholders = data.keySet().stream()
            .map(k -> "?")
            .collect(Collectors.joining(", "));
        
        String sql = String.format(
            "INSERT INTO %s (%s) VALUES (%s)", 
            tableName, columns, placeholders);
        
        postgresTemplate.update(sql, data.values().toArray());
    }
    
    private void handleUpdate(CdcEvent event) {
        String tableName = event.getSource().getTable();
        Map<String, Object> data = event.getAfter();
        Object id = data.get("id");
        
        String setClause = data.entrySet().stream()
            .filter(e -> !e.getKey().equals("id"))
            .map(e -> e.getKey() + " = ?")
            .collect(Collectors.joining(", "));
        
        List<Object> values = new ArrayList<>(data.values());
        values.remove(id);
        values.add(id);
        
        String sql = String.format(
            "UPDATE %s SET %s WHERE id = ?", 
            tableName, setClause);
        
        postgresTemplate.update(sql, values.toArray());
    }
    
    private void handleDelete(CdcEvent event) {
        String tableName = event.getSource().getTable();
        Object id = event.getBefore().get("id");
        
        String sql = String.format("DELETE FROM %s WHERE id = ?", tableName);
        postgresTemplate.update(sql, id);
    }
}
```

### Phase 4: Dual-Write Implementation

#### Dual-Write Service

```java
@Service
public class DualWriteService {
    
    @Autowired
    private JdbcTemplate mysqlTemplate;
    
    @Autowired
    private JdbcTemplate postgresTemplate;
    
    @Autowired
    private FeatureFlagService featureFlagService;
    
    @Transactional
    public void insert(String tableName, Map<String, Object> data) {
        // Always write to MySQL (primary)
        insertIntoMysql(tableName, data);
        
        // Write to PostgreSQL if dual-write is enabled
        if (featureFlagService.isEnabled("dual-write-postgres")) {
            try {
                insertIntoPostgres(tableName, data);
            } catch (Exception e) {
                log.error("Failed to write to PostgreSQL", e);
                // Don't fail the transaction, log for reconciliation
                reconciliationService.logFailure(tableName, "INSERT", data, e);
            }
        }
    }
    
    @Transactional
    public void update(String tableName, Object id, Map<String, Object> data) {
        // Always write to MySQL (primary)
        updateInMysql(tableName, id, data);
        
        // Write to PostgreSQL if dual-write is enabled
        if (featureFlagService.isEnabled("dual-write-postgres")) {
            try {
                updateInPostgres(tableName, id, data);
            } catch (Exception e) {
                log.error("Failed to update PostgreSQL", e);
                reconciliationService.logFailure(tableName, "UPDATE", data, e);
            }
        }
    }
    
    @Transactional
    public void delete(String tableName, Object id) {
        // Always delete from MySQL (primary)
        deleteFromMysql(tableName, id);
        
        // Delete from PostgreSQL if dual-write is enabled
        if (featureFlagService.isEnabled("dual-write-postgres")) {
            try {
                deleteFromPostgres(tableName, id);
            } catch (Exception e) {
                log.error("Failed to delete from PostgreSQL", e);
                reconciliationService.logFailure(tableName, "DELETE", 
                    Map.of("id", id), e);
            }
        }
    }
}
```

#### Repository with Dual-Write Support

```java
@Repository
public class PatientRepository {
    
    @Autowired
    private DualWriteService dualWriteService;
    
    @Autowired
    private FeatureFlagService featureFlagService;
    
    @Autowired
    private JdbcTemplate mysqlTemplate;
    
    @Autowired
    private JdbcTemplate postgresTemplate;
    
    public Patient save(Patient patient) {
        // Write to both databases
        Map<String, Object> data = new HashMap<>();
        data.put("first_name", patient.getFirstName());
        data.put("last_name", patient.getLastName());
        data.put("date_of_birth", patient.getDateOfBirth());
        data.put("ssn", patient.getSsn());
        data.put("medical_history", 
            objectMapper.valueToTree(patient.getMedicalHistory()));
        
        if (patient.getId() == null) {
            // Insert
            dualWriteService.insert("patients", data);
            
            // Get generated ID from MySQL
            Long id = mysqlTemplate.queryForObject(
                "SELECT LAST_INSERT_ID()", Long.class);
            patient.setId(id);
        } else {
            // Update
            data.put("id", patient.getId());
            dualWriteService.update("patients", patient.getId(), data);
        }
        
        return patient;
    }
    
    public Patient findById(Long id) {
        // Read from PostgreSQL if migration is complete for this table
        if (featureFlagService.isEnabled("read-from-postgres-patients")) {
            return findByIdFromPostgres(id);
        }
        // Otherwise read from MySQL
        return findByIdFromMysql(id);
    }
    
    private Patient findByIdFromMysql(Long id) {
        String sql = "SELECT * FROM patients WHERE id = ?";
        return mysqlTemplate.queryForObject(sql, 
            new PatientRowMapper(), id);
    }
    
    private Patient findByIdFromPostgres(Long id) {
        String sql = "SELECT * FROM patients WHERE id = ?";
        return postgresTemplate.queryForObject(sql, 
            new PatientRowMapper(), id);
    }
}
```

### Phase 5: Data Validation and Verification

#### Data Validation Service

```java
@Service
public class DataValidationService {
    
    @Autowired
    private JdbcTemplate mysqlTemplate;
    
    @Autowired
    private JdbcTemplate postgresTemplate;
    
    public ValidationResult validateTable(String tableName) {
        ValidationResult result = new ValidationResult(tableName);
        
        // Compare row counts
        Long mysqlCount = mysqlTemplate.queryForObject(
            "SELECT COUNT(*) FROM " + tableName, Long.class);
        Long postgresCount = postgresTemplate.queryForObject(
            "SELECT COUNT(*) FROM " + tableName, Long.class);
        
        result.setMySqlCount(mysqlCount);
        result.setPostgresCount(postgresCount);
        
        if (!mysqlCount.equals(postgresCount)) {
            result.addError("Row count mismatch: MySQL=%d, PostgreSQL=%d",
                mysqlCount, postgresCount);
        }
        
        // Compare checksums
        String mysqlChecksum = calculateChecksum(mysqlTemplate, tableName);
        String postgresChecksum = calculateChecksum(postgresTemplate, tableName);
        
        result.setMySqlChecksum(mysqlChecksum);
        result.setPostgresChecksum(postgresChecksum);
        
        if (!mysqlChecksum.equals(postgresChecksum)) {
            result.addError("Checksum mismatch");
            
            // Find specific differences
            findDifferences(tableName, result);
        }
        
        // Validate constraints
        validateConstraints(tableName, result);
        
        return result;
    }
    
    private String calculateChecksum(JdbcTemplate template, String tableName) {
        // Calculate MD5 checksum of all rows
        String sql = String.format(
            "SELECT MD5(CONCAT_WS('|', %s)) as checksum FROM %s",
            getAllColumnNames(template, tableName), tableName);
        
        List<String> checksums = template.queryForList(sql, String.class);
        
        // Combine all row checksums
        return DigestUtils.md5DigestAsHex(
            String.join("", checksums).getBytes());
    }
    
    private void findDifferences(String tableName, ValidationResult result) {
        // Find rows in MySQL but not in PostgreSQL
        String sql = String.format(
            "SELECT m.* FROM %s m LEFT JOIN %s p ON m.id = p.id " +
            "WHERE p.id IS NULL LIMIT 1000",
            tableName, tableName);
        
        List<Map<String, Object>> missingInPostgres = 
            mysqlTemplate.queryForList(sql);
        
        result.setMissingInPostgres(missingInPostgres);
        
        // Find rows in PostgreSQL but not in MySQL
        sql = String.format(
            "SELECT p.* FROM %s p LEFT JOIN %s m ON p.id = m.id " +
            "WHERE m.id IS NULL LIMIT 1000",
            tableName, tableName);
        
        List<Map<String, Object>> missingInMysql = 
            postgresTemplate.queryForList(sql);
        
        result.setMissingInMysql(missingInMysql);
    }
    
    private void validateConstraints(String tableName, ValidationResult result) {
        // Validate foreign keys
        List<ForeignKeyInfo> foreignKeys = getForeignKeys(tableName);
        
        for (ForeignKeyInfo fk : foreignKeys) {
            String checkSql = String.format(
                "SELECT COUNT(*) FROM %s WHERE %s NOT IN " +
                "(SELECT id FROM %s)",
                tableName, fk.getColumn(), fk.getReferencedTable());
            
            Long invalidCount = postgresTemplate.queryForObject(
                checkSql, Long.class);
            
            if (invalidCount > 0) {
                result.addError("Foreign key violation: %s has %d invalid references",
                    fk.getName(), invalidCount);
            }
        }
    }
}
```

#### Reconciliation Service

```java
@Service
public class ReconciliationService {
    
    @Autowired
    private JdbcTemplate mysqlTemplate;
    
    @Autowired
    private JdbcTemplate postgresTemplate;
    
    @Autowired
    private DeadLetterQueueRepository dlqRepository;
    
    @Scheduled(fixedDelay = 60000) // Every minute
    public void reconcile() {
        // Process failed dual-write operations
        List<FailedOperation> failedOps = dlqRepository.findUnprocessed();
        
        for (FailedOperation op : failedOps) {
            try {
                retryOperation(op);
                op.setStatus("RECONCILED");
                dlqRepository.save(op);
            } catch (Exception e) {
                log.error("Failed to reconcile operation {}", op.getId(), e);
            }
        }
        
        // Run periodic validation
        validateCriticalTables();
    }
    
    private void retryOperation(FailedOperation op) {
        switch (op.getOperation()) {
            case "INSERT":
                postgresTemplate.update(
                    op.getSql(), op.getParameters().toArray());
                break;
            case "UPDATE":
                postgresTemplate.update(
                    op.getSql(), op.getParameters().toArray());
                break;
            case "DELETE":
                postgresTemplate.update(
                    op.getSql(), op.getParameters().toArray());
                break;
        }
    }
    
    private void validateCriticalTables() {
        List<String> criticalTables = Arrays.asList(
            "patients", "lab_results", "prescriptions");
        
        for (String table : criticalTables) {
            ValidationResult result = validationService.validateTable(table);
            
            if (!result.isValid()) {
                alertService.sendAlert(
                    "Data validation failed for table: " + table,
                    result.getErrors());
            }
        }
    }
}
```

### Phase 6: Gradual Traffic Cutover

#### Traffic Routing Configuration

```java
@Configuration
public class DataSourceRoutingConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource(
            @Qualifier("mysqlDataSource") DataSource mysqlDataSource,
            @Qualifier("postgresDataSource") DataSource postgresDataSource) {
        
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("mysql", mysqlDataSource);
        targetDataSources.put("postgres", postgresDataSource);
        
        RoutingDataSource routingDataSource = new RoutingDataSource();
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(mysqlDataSource);
        
        return routingDataSource;
    }
}

public class RoutingDataSource extends AbstractRoutingDataSource {
    
    @Override
    protected Object determineCurrentLookupKey() {
        // Check feature flags for routing decisions
        if (FeatureFlagService.isEnabled("read-from-postgres-patients") &&
            TableContext.getCurrentTable().equals("patients")) {
            return "postgres";
        }
        return "mysql";
    }
}
```

#### Gradual Cutover Controller

```java
@RestController
@RequestMapping("/admin/migration")
public class MigrationCutoverController {
    
    @Autowired
    private FeatureFlagService featureFlagService;
    
    @Autowired
    private DataValidationService validationService;
    
    @PostMapping("/cutover/{table}")
    public ResponseEntity<CutoverResult> cutoverTable(
            @PathVariable String table,
            @RequestBody CutoverRequest request) {
        
        // Pre-cutover validation
        ValidationResult validation = validationService.validateTable(table);
        if (!validation.isValid()) {
            return ResponseEntity.badRequest()
                .body(CutoverResult.failed("Validation failed", validation));
        }
        
        // Start with read cutover
        String readFlag = "read-from-postgres-" + table;
        featureFlagService.enable(readFlag, request.getPercentage());
        
        // Monitor for issues
        CutoverResult result = monitorCutover(table, readFlag);
        
        if (result.isSuccess()) {
            // Proceed to write cutover
            String writeFlag = "write-to-postgres-" + table;
            featureFlagService.enable(writeFlag, 100);
            result.setWriteCutoverComplete(true);
        }
        
        return ResponseEntity.ok(result);
    }
    
    private CutoverResult monitorCutover(String table, String flag) {
        CutoverResult result = new CutoverResult();
        
        // Monitor for 24 hours
        for (int i = 0; i < 24; i++) {
            // Check error rates
            double errorRate = getErrorRate(table);
            if (errorRate > 0.01) { // 1% error threshold
                // Rollback
                featureFlagService.disable(flag);
                result.setSuccess(false);
                result.setError("Error rate exceeded threshold: " + errorRate);
                return result;
            }
            
            // Check performance
            double latency = getLatency(table);
            if (latency > getBaselineLatency(table) * 1.5) {
                // Rollback
                featureFlagService.disable(flag);
                result.setSuccess(false);
                result.setError("Latency degraded: " + latency);
                return result;
            }
            
            try {
                Thread.sleep(3600000); // 1 hour
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
        
        result.setSuccess(true);
        return result;
    }
}
```

### Phase 7: Rollback Plan

#### Rollback Service

```java
@Service
public class RollbackService {
    
    @Autowired
    private FeatureFlagService featureFlagService;
    
    @Autowired
    private JdbcTemplate mysqlTemplate;
    
    @Autowired
    private JdbcTemplate postgresTemplate;
    
    public void rollbackTable(String table) {
        log.warn("Initiating rollback for table: {}", table);
        
        // Disable PostgreSQL routing
        String readFlag = "read-from-postgres-" + table;
        String writeFlag = "write-to-postgres-" + table;
        
        featureFlagService.disable(readFlag);
        featureFlagService.disable(writeFlag);
        
        // Verify MySQL is consistent
        ValidationResult validation = validateMySql(table);
        if (!validation.isValid()) {
            log.error("MySQL validation failed during rollback");
            alertService.sendCriticalAlert(
                "Rollback validation failed for table: " + table);
        }
        
        // Log rollback completion
        log.info("Rollback completed for table: {}", table);
    }
    
    public void rollbackFullMigration() {
        log.warn("Initiating full migration rollback");
        
        List<String> tables = Arrays.asList(
            "audit_log",
            "medical_records",
            "appointments",
            "prescriptions",
            "lab_results",
            "patients",
            "providers",
            "drug_database"
        );
        
        // Rollback in reverse order
        Collections.reverse(tables);
        
        for (String table : tables) {
            rollbackTable(table);
        }
        
        log.info("Full migration rollback completed");
    }
}
```

---

## Results & Metrics

### Quantitative Outcomes

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Query Performance (P95) | 850ms | 320ms | 62% improvement |
| Write Throughput | 5K ops/sec | 15K ops/sec | 200% improvement |
| Storage Cost | $8K/month | $5K/month | 37% reduction |
| Downtime | N/A | 0 minutes | Zero downtime achieved |
| Data Loss | N/A | 0 records | Zero data loss |
| Migration Duration | N/A | 45 days | Within timeline |
| Validation Errors | N/A | 0 | Perfect validation |

### Qualitative Improvements

1. **Zero Downtime**: Healthcare system remained available throughout migration
2. **Performance Gains**: PostgreSQL features (JSONB, full-text search) improved query performance
3. **Scalability**: Multi-master PostgreSQL setup improved write scalability
4. **Feature Enablement**: New healthcare analytics features now possible
5. **Cost Reduction**: Open-source PostgreSQL reduced licensing costs

### Lessons Learned

#### What Worked Well

1. **Dual-Write Strategy**: Ensured data consistency during transition
2. **CDC for Sync**: Debezium provided reliable real-time synchronization
3. **Gradual Cutover**: Feature flags enabled safe, incremental traffic shift
4. **Comprehensive Validation**: Checksums and row counts ensured data integrity
5. **Rollback Capability**: Quick rollback option provided safety net

#### What Didn't Work

1. **Underestimated Migration Time**: Initial bulk migration took 2x longer than planned
2. **Schema Complexity**: Data type mapping more complex than anticipated
3. **CDC Latency**: Some lag during peak periods caused temporary inconsistencies
4. **Storage Planning**: Needed 50% more temporary storage during migration
5. **Monitoring Gaps**: Initial monitoring missed some edge cases

---

## Interview Takeaways

### Key Technical Concepts to Discuss

1. **Zero-Downtime Migration**: Dual-write, CDC, gradual cutover strategies
2. **Change Data Capture (CDC)**: Debezium, binlog replication, event streaming
3. **Data Validation**: Checksums, row counts, constraint validation
4. **Schema Migration**: Type mapping, index creation, constraint handling
5. **Feature Flags**: Gradual rollout, traffic routing, rollback capability
6. **Database Routing**: Abstract routing layer, dynamic datasource selection
7. **Reconciliation**: Handling failed operations, data consistency checks
8. **Rollback Strategy**: Quick rollback, validation, safety mechanisms

### What to Emphasize in Interviews

1. **Risk Mitigation**: How you ensured zero data loss and zero downtime
2. **Incremental Approach**: Why gradual migration over big-bang
3. **Validation Strategy**: How you verified data integrity
4. **Rollback Planning**: How you prepared for failure scenarios
5. **Performance Impact**: How you minimized performance degradation
6. **Monitoring**: How you tracked migration progress and health

### Sample Interview Answer Structure

"When migrating 2TB of healthcare data from MySQL to PostgreSQL, we used a dual-write strategy with CDC for zero downtime. We started with bulk data migration, then enabled Change Data Capture to keep databases in sync. During the dual-write phase, we wrote to both databases while still reading from MySQL. We validated data integrity using checksums and row counts, then gradually cut over traffic using feature flags. This approach allowed us to complete the migration with zero downtime and zero data loss while improving query performance by 62%."

---

## Follow-up Questions

### Technical Questions

1. **How did you handle schema differences between MySQL and PostgreSQL?**
   - Type mapping, constraint conversion, index creation

2. **What happened when CDC fell behind during peak load?**
   - Buffering, throttling, catch-up strategies

3. **How did you validate data integrity during migration?**
   - Checksums, row counts, sampling, constraint validation

4. **How did you handle the 2TB data volume?**
   - Batch processing, parallel migration, throttling

5. **What about stored procedures and triggers?**
   - Conversion, rewriting, or alternative approaches

### Architectural Questions

6. **How did you decide the cutover order for tables?**
   - Dependency analysis, independent tables first

7. **How did you handle foreign key constraints during migration?**
   - Order of migration, constraint validation, error handling

8. **What about indexes during migration?**
   - Create after data load, parallel index creation

9. **How did you monitor the migration process?**
   - Metrics, alerts, progress tracking, validation

### Leadership Questions

10. **How did you communicate migration status to stakeholders?**
    - Regular updates, dashboards, risk transparency

11. **How did you handle the rollback decision?**
    - Criteria, triggers, communication, execution

12. **What would you do differently next time?**
    - Better time estimation, more storage, improved monitoring

---

## Related Topics

- [System Design: Database Design](../../sections/14-system-design/README.md)
- [Performance Optimization](../../sections/12-performance-optimization/README.md)
- [Spring Boot: Database Access](../../sections/05-spring-boot-database-access/README.md)
- [Behavioral: Crisis Management](../../sections/15-behavioral-scenarios/README.md)

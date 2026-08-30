# Case Study 2: Production Performance Optimization

## Difficulty: Medium 🟠

---

## Scenario Context

### Company Background
**HealthTrack** is a healthcare data analytics platform processing patient records, lab results, and clinical data. The platform serves 500 hospitals and processes 10 million records daily.

### Technical Stack
- **Application**: Spring Boot 2.7 with Spring Data JPA
- **Database**: PostgreSQL 13 (2TB, 50 million rows in main table)
- **Cache**: Redis (single instance, not clustered)
- **Message Queue**: RabbitMQ
- **Deployment**: AWS EC2 (m5.xlarge instances)
- **Monitoring**: New Relic (basic metrics only)

### Performance Issues
The platform was experiencing:
- **Slow API Responses**: P95 latency of 8.5 seconds for patient search
- **Database CPU**: 95% CPU utilization during peak hours
- **Memory Issues**: Frequent GC pauses causing request timeouts
- **Cache Misses**: 85% cache miss rate on frequently accessed data
- **Timeout Errors**: 15% of requests timing out after 30 seconds

### Business Impact
- **User Complaints**: Doctors experiencing delays during patient consultations
- **Lost Revenue**: 3 hospitals canceled contracts due to performance issues
- **SLA Breaches**: Missing 99.5% availability SLA (currently at 97.2%)
- **Team Stress**: On-call team overwhelmed with performance incidents

### Constraints
- **Data Privacy**: Cannot export patient data for external analysis
- **Regulatory**: Must maintain audit logs for all data access
- **Budget**: Limited budget for infrastructure upgrades
- **Timeline**: 3 months to achieve 50% latency improvement
- **Team**: 2 senior developers, 3 junior developers

---

## Problem Statement

**Primary Challenge**: Optimize a healthcare data analytics platform to reduce P95 latency from 8.5 seconds to under 4 seconds while maintaining data privacy and regulatory compliance.

**Specific Problems**:
1. Why is the patient search API so slow?
2. What's causing the high database CPU utilization?
3. Why is the cache miss rate so high?
4. How can we reduce GC pauses without increasing heap size?
5. What's the root cause of the 15% timeout rate?

**Success Criteria**:
- P95 latency under 4 seconds for patient search
- Database CPU under 70% during peak hours
- Cache hit rate above 80%
- GC pause frequency reduced by 60%
- Timeout rate under 2%
- Maintain all regulatory compliance requirements

---

## Analysis Phase

### Performance Profiling

#### Step 1: Identify Hotspots

Using **YourKit Profiler** and **APM tools**, we identified the bottlenecks:

```
Top 5 Hot Methods by CPU Time:
1. PatientRepository.findByCriteria() - 42% of CPU time
2. PatientService.enrichPatientData() - 28% of CPU time
3. AuditLogService.logAccess() - 15% of CPU time
4. CacheManager.get() - 8% of CPU time
5. ObjectMapper.writeValueAsString() - 7% of CPU time
```

#### Step 2: Database Query Analysis

Using **pg_stat_statements**:

```sql
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;
```

**Findings**:

| Query | Calls | Total Time | Mean Time | Rows |
|-------|-------|------------|-----------|------|
| SELECT * FROM patients WHERE ... | 2.5M | 450h | 648ms | 125M |
| SELECT * FROM lab_results WHERE patient_id = ? | 5.2M | 320h | 221ms | 260M |
| INSERT INTO audit_logs ... | 10M | 180h | 65ms | 10M |
| SELECT * FROM medications WHERE patient_id = ? | 3.1M | 120h | 139ms | 93M |

**Key Issues**:
- Patient search query fetching all columns (SELECT *)
- Missing indexes on frequently filtered columns
- N+1 query problem in patient data enrichment
- Audit log inserts causing table bloat

#### Step 3: Cache Analysis

Redis cache analysis revealed:

```
Cache Statistics:
- Total Keys: 50,000
- Hit Rate: 15%
- Evictions: 1.2M/day
- Memory Usage: 95% (near capacity)
- Key Distribution: 80% of keys accessed < 5 times
```

**Root Causes**:
- Cache keys not following consistent patterns
- No TTL strategy causing memory pressure
- Caching entire objects instead of frequently accessed fields
- Cache warming strategy missing

#### Step 4: JVM Analysis

Using **Java Flight Recorder**:

```
JVM Metrics:
- Heap Size: 4GB
- Heap Usage: 85% average, 98% peak
- GC Pauses: 12 pauses/hour, avg 450ms, max 2.1s
- Young Generation: 512MB (too small)
- Old Generation: 3.5GB (too large)
- GC Algorithm: G1GC (not tuned)
```

**Issues**:
- Young generation too small, causing frequent promotions
- G1GC not tuned for low-latency requirements
- High object allocation rate in hot paths

### Root Cause Summary

| Issue | Root Cause | Impact |
|-------|------------|--------|
| Slow patient search | Missing indexes, SELECT *, N+1 queries | 42% CPU time |
| High DB CPU | Inefficient queries, no connection pooling tuning | 95% CPU utilization |
| Low cache hit rate | Poor key design, no TTL, over-caching | 85% miss rate |
| GC pauses | Improper heap sizing, high allocation rate | 15% timeout rate |
| Audit logging overhead | Synchronous logging, table bloat | 15% CPU time |

---

## Solution Approach

### Optimization Strategy

We prioritized optimizations using the **Pareto Principle (80/20 rule)**:

1. **Database Optimization** (Weeks 1-4) - Highest impact
2. **Caching Strategy** (Weeks 5-7) - Medium effort, high impact
3. **JVM Tuning** (Weeks 8-9) - Medium effort, medium impact
4. **Code Optimization** (Weeks 10-12) - Lower impact but necessary

### Architecture Improvements

```
Before:
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │───▶│  App Server │───▶│ PostgreSQL  │
└─────────────┘    └─────────────┘    └─────────────┘
                       │
                       ▼
                  ┌─────────┐
                  │  Redis  │
                  └─────────┘

After:
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │───▶│  App Server │───▶│ PostgreSQL  │
└─────────────┘    └─────────────┘    └─────────────┘
                       │                   │
                       ▼                   ▼
                  ┌─────────┐      ┌──────────┐
                  │ Redis   │      │ Read     │
                  │ Cluster │      │ Replica  │
                  └─────────┘      └──────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Async Audit Log │
              │   (Kafka)       │
              └─────────────────┘
```

---

## Implementation Details

### Phase 1: Database Optimization (Weeks 1-4)

#### Step 1: Query Optimization

**Before** (Slow query):
```java
@Repository
public interface PatientRepository extends JpaRepository<Patient, Long> {
    
    @Query("SELECT p FROM Patient p WHERE " +
           "(:name IS NULL OR p.name LIKE %:name%) AND " +
           "(:dob IS NULL OR p.dateOfBirth = :dob) AND " +
           "(:mrn IS NULL OR p.medicalRecordNumber = :mrn)")
    List<Patient> findByCriteria(@Param("name") String name,
                                  @Param("dob") LocalDate dob,
                                  @Param("mrn") String mrn);
}
```

**After** (Optimized):
```java
@Repository
public interface PatientRepository extends JpaRepository<Patient, Long> {
    
    @Query("SELECT NEW com.healthtrack.dto.PatientSearchResult(" +
           "p.id, p.name, p.dateOfBirth, p.medicalRecordNumber) " +
           "FROM Patient p WHERE " +
           "(:name IS NULL OR p.name LIKE CONCAT('%', :name, '%')) AND " +
           "(:dob IS NULL OR p.dateOfBirth = :dob) AND " +
           "(:mrn IS NULL OR p.medicalRecordNumber = :mrn)")
    Page<PatientSearchResult> findByCriteria(
        @Param("name") String name,
        @Param("dob") LocalDate dob,
        @Param("mrn") String mrn,
        Pageable pageable);
}
```

**DTO Projection**:
```java
public class PatientSearchResult {
    private Long id;
    private String name;
    private LocalDate dateOfBirth;
    private String medicalRecordNumber;
    
    // Constructor for JPQL
    public PatientSearchResult(Long id, String name, 
                               LocalDate dob, String mrn) {
        this.id = id;
        this.name = name;
        this.dateOfBirth = dob;
        this.medicalRecordNumber = mrn;
    }
    
    // Getters only - immutable
}
```

#### Step 2: Index Optimization

```sql
-- Composite index for search criteria
CREATE INDEX idx_patient_search 
ON patients (medical_record_number, date_of_birth, name)
WHERE deleted_at IS NULL;

-- Partial index for active patients
CREATE INDEX idx_patient_active 
ON patients (id, name, date_of_birth, medical_record_number)
WHERE status = 'ACTIVE';

-- Covering index for lab results
CREATE INDEX idx_lab_results_patient_covering 
ON lab_results (patient_id, test_date, result_value)
INCLUDE (test_name, unit, reference_range);

-- Index for audit log cleanup
CREATE INDEX idx_audit_logs_created 
ON audit_logs (created_at);
```

#### Step 3: Connection Pool Tuning

```yaml
# application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 10
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      connection-test-query: SELECT 1
      pool-name: HealthTrackHikariCP
```

**Monitoring Connection Pool**:
```java
@Component
public class ConnectionPoolMonitor {
    
    @Autowired
    private HikariDataSource dataSource;
    
    @Scheduled(fixedRate = 60000)
    public void logPoolMetrics() {
        HikariPoolMXBean poolProxy = dataSource.getHikariPoolMXBean();
        
        Metrics.gauge("hikari.active.connections", 
            poolProxy.getActiveConnections());
        Metrics.gauge("hikari.idle.connections", 
            poolProxy.getIdleConnections());
        Metrics.gauge("hikari.threads.awaiting.connection", 
            poolProxy.getThreadsAwaitingConnection());
        
        if (poolProxy.getActiveConnections() == poolProxy.getTotalConnections()) {
            log.warn("Connection pool exhausted! Active: {}, Total: {}", 
                poolProxy.getActiveConnections(), 
                poolProxy.getTotalConnections());
        }
    }
}
```

#### Step 4: N+1 Query Fix

**Before** (N+1 problem):
```java
@Service
public class PatientService {
    
    public PatientDetail getPatientDetail(Long patientId) {
        Patient patient = patientRepository.findById(patientId)
            .orElseThrow();
        
        // N+1: Each call triggers a separate query
        List<LabResult> labResults = labResultRepository
            .findByPatientId(patientId);
        
        List<Medication> medications = medicationRepository
            .findByPatientId(patientId);
        
        List<Allergy> allergies = allergyRepository
            .findByPatientId(patientId);
        
        return new PatientDetail(patient, labResults, medications, allergies);
    }
}
```

**After** (Entity Graph):
```java
@Repository
public interface PatientRepository extends JpaRepository<Patient, Long> {
    
    @EntityGraph(attributePaths = {"labResults", "medications", "allergies"})
    Optional<Patient> findByIdWithDetails(Long id);
}

@Service
public class PatientService {
    
    @Cacheable(value = "patient-details", key = "#patientId")
    public PatientDetail getPatientDetail(Long patientId) {
        Patient patient = patientRepository.findByIdWithDetails(patientId)
            .orElseThrow();
        
        return PatientDetail.fromEntity(patient);
    }
}
```

**Entity Graph Definition**:
```java
@NamedEntityGraph(
    name = "Patient.withDetails",
    attributeNodes = {
        @NamedAttributeNode("labResults"),
        @NamedAttributeNode("medications"),
        @NamedAttributeNode("allergies")
    }
)
@Entity
public class Patient {
    // ...
}
```

### Phase 2: Caching Strategy (Weeks 5-7)

#### Step 1: Redis Cluster Configuration

```yaml
# application.yml
spring:
  redis:
    cluster:
      nodes:
        - redis-node-1:6379
        - redis-node-2:6379
        - redis-node-3:6379
        - redis-node-4:6379
        - redis-node-5:6379
        - redis-node-6:6379
      max-redirects: 3
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
    timeout: 5000ms
```

#### Step 2: Multi-Level Caching

```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .disableCachingNullValues()
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer())
            );
        
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        
        // Patient cache - longer TTL, lower eviction
        cacheConfigurations.put("patients", config
            .entryTtl(Duration.ofHours(2))
            .prefixCacheNameWith("patient:"));
        
        // Lab results - medium TTL
        cacheConfigurations.put("lab-results", config
            .entryTtl(Duration.ofMinutes(45))
            .prefixCacheNameWith("lab:"));
        
        // Search results - short TTL
        cacheConfigurations.put("search-results", config
            .entryTtl(Duration.ofMinutes(10))
            .prefixCacheNameWith("search:"));
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withInitialCacheConfigurations(cacheConfigurations)
            .transactionAware()
            .build();
    }
}
```

#### Step 3: Cache Warming Strategy

```java
@Component
public class CacheWarmer {
    
    @Autowired
    private PatientRepository patientRepository;
    
    @Autowired
    private CacheManager cacheManager;
    
    @Scheduled(cron = "0 0 2 * * ?") // 2 AM daily
    public void warmPatientCache() {
        log.info("Starting cache warming...");
        
        // Warm frequently accessed patients
        List<Long> frequentPatientIds = patientRepository
            .findFrequentlyAccessedPatients(1000);
        
        frequentPatientIds.parallelStream().forEach(patientId -> {
            try {
                Patient patient = patientRepository.findByIdWithDetails(patientId)
                    .orElse(null);
                if (patient != null) {
                    Cache cache = cacheManager.getCache("patients");
                    if (cache != null) {
                        cache.put(patientId, patient);
                    }
                }
            } catch (Exception e) {
                log.error("Failed to warm cache for patient {}", patientId, e);
            }
        });
        
        log.info("Cache warming completed. Warmed {} patients", 
            frequentPatientIds.size());
    }
}
```

#### Step 4: Cache Aside Pattern with Refresh

```java
@Service
public class PatientService {
    
    @Autowired
    private PatientRepository patientRepository;
    
    @Autowired
    private CacheManager cacheManager;
    
    @Cacheable(value = "patients", key = "#id")
    public Patient getPatient(Long id) {
        return patientRepository.findById(id)
            .orElseThrow(() -> new PatientNotFoundException(id));
    }
    
    @CacheEvict(value = "patients", key = "#patient.id")
    public Patient updatePatient(Patient patient) {
        return patientRepository.save(patient);
    }
    
    @Caching(evict = {
        @CacheEvict(value = "patients", key = "#id"),
        @CacheEvict(value = "search-results", allEntries = true)
    })
    public void deletePatient(Long id) {
        patientRepository.deleteById(id);
    }
    
    // Refresh cache asynchronously
    @Async
    public void refreshPatientCache(Long id) {
        Cache cache = cacheManager.getCache("patients");
        if (cache != null) {
            cache.evict(id);
            getPatient(id); // This will repopulate the cache
        }
    }
}
```

### Phase 3: JVM Tuning (Weeks 8-9)

#### Step 1: G1GC Tuning

```bash
JAVA_OPTS="
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
-XX:G1HeapRegionSize=16m
-XX:G1ReservePercent=15
-XX:G1MixedGCCountTarget=8
-XX:G1OldCSetRegionThreshold=10
-XX:MaxTenuringThreshold=15
-XX:G1MixedGCLiveThreshold=85
-XX:G1HeapWastePercent=5
-Xms6g
-Xmx6g
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
-XX:+AlwaysPreTouch
-XX:+UseStringDeduplication
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintGCTimeStamps
-Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=5,filesize=10m
"
```

#### Step 2: Object Allocation Reduction

**Before** (High allocation):
```java
@Service
public class PatientSearchService {
    
    public List<Patient> searchPatients(SearchCriteria criteria) {
        // Creates new list and filters in memory
        List<Patient> allPatients = patientRepository.findAll();
        return allPatients.stream()
            .filter(p -> matchesCriteria(p, criteria))
            .collect(Collectors.toList());
    }
}
```

**After** (Database-level filtering):
```java
@Service
public class PatientSearchService {
    
    public Page<PatientSearchResult> searchPatients(
            SearchCriteria criteria, Pageable pageable) {
        // Let database do the filtering
        return patientRepository.findByCriteria(
            criteria.getName(),
            criteria.getDateOfBirth(),
            criteria.getMedicalRecordNumber(),
            pageable
        );
    }
}
```

#### Step 3: String Optimization

```java
@Component
public class PatientFormatter {
    
    // Use StringBuilder instead of string concatenation in loops
    public String formatPatientList(List<Patient> patients) {
        StringBuilder sb = new StringBuilder(patients.size() * 100);
        
        for (Patient patient : patients) {
            sb.append(patient.getId())
              .append(",")
              .append(escapeCsv(patient.getName()))
              .append(",")
              .append(patient.getDateOfBirth())
              .append("\n");
        }
        
        return sb.toString();
    }
    
    // Reuse formatter instances
    private static final DateTimeFormatter DATE_FORMATTER = 
        DateTimeFormatter.ofPattern("yyyy-MM-dd");
    
    public String formatDate(LocalDate date) {
        return date.format(DATE_FORMATTER);
    }
}
```

### Phase 4: Async Audit Logging (Weeks 10-12)

#### Step 1: Kafka-based Async Logging

```java
@Configuration
@EnableKafka
public class KafkaConfig {
    
    @Bean
    public ProducerFactory<String, AuditEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
            StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
            JsonSerializer.class);
        config.put(ProducerConfig.ACKS_CONFIG, "1");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        
        return new DefaultKafkaProducerFactory<>(config);
    }
    
    @Bean
    public KafkaTemplate<String, AuditEvent> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

#### Step 2: Audit Event Publisher

```java
@Service
public class AuditEventPublisher {
    
    @Autowired
    private KafkaTemplate<String, AuditEvent> kafkaTemplate;
    
    @Async
    public void publishAuditEvent(AuditEvent event) {
        try {
            kafkaTemplate.send("audit-logs", event.getUserId(), event)
                .addCallback(
                    result -> log.debug("Audit event published: {}", event.getId()),
                    failure -> log.error("Failed to publish audit event: {}", 
                        event.getId(), failure)
                );
        } catch (Exception e) {
            log.error("Error publishing audit event", e);
            // Fallback to database if Kafka is down
            auditLogRepository.save(AuditLog.fromEvent(event));
        }
    }
}
```

#### Step 3: Audit Log Consumer

```java
@Service
public class AuditLogConsumer {
    
    @Autowired
    private AuditLogRepository auditLogRepository;
    
    @KafkaListener(
        topics = "audit-logs",
        groupId = "audit-log-consumer",
        concurrency = "3"
    )
    public void consumeAuditEvent(AuditEvent event) {
        try {
            AuditLog auditLog = AuditLog.fromEvent(event);
            auditLogRepository.save(auditLog);
        } catch (Exception e) {
            log.error("Failed to save audit log: {}", event.getId(), e);
            // Send to dead letter queue
            deadLetterQueue.send(event);
        }
    }
}
```

---

## Results & Metrics

### Quantitative Outcomes

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| P95 Latency (Patient Search) | 8.5s | 2.8s | 67% reduction |
| P99 Latency (Patient Search) | 15.2s | 5.1s | 66% reduction |
| Database CPU (Peak) | 95% | 58% | 39% reduction |
| Cache Hit Rate | 15% | 87% | 480% improvement |
| GC Pause Frequency | 12/hour | 3/hour | 75% reduction |
| GC Pause Duration (Avg) | 450ms | 120ms | 73% reduction |
| Timeout Rate | 15% | 1.2% | 92% reduction |
| Query Time (Avg) | 648ms | 89ms | 86% reduction |
| Throughput | 100 req/s | 350 req/s | 250% increase |

### Qualitative Improvements

1. **User Experience**: Doctors reported significantly faster patient lookups
2. **Team Morale**: On-call incidents reduced from 20/week to 3/week
3. **Contract Renewals**: All 3 at-risk hospitals renewed contracts
4. **SLA Compliance**: Achieved 99.7% availability (exceeded 99.5% target)
5. **Scalability**: System can now handle 3x current load

### Lessons Learned

#### What Worked Well

1. **Profiling-First Approach**: Metrics-driven optimization prevented wasted effort
2. **Incremental Improvements**: Small wins built momentum and confidence
3. **Database-First Strategy**: Query optimization had highest ROI
4. **Multi-Level Caching**: Reduced database load significantly
5. **Async Logging**: Removed synchronous bottleneck from critical path

#### What Didn't Work

1. **Premature Caching**: Caching without understanding access patterns was ineffective
2. **Over-Optimization**: Spent time optimizing rarely-used endpoints
3. **Insufficient Monitoring**: Needed more granular metrics earlier
4. **Lack of Load Testing**: Some optimizations broke under production load
5. **Ignoring Network Latency**: Focused on app performance, missed network issues

---

## Interview Takeaways

### Key Technical Concepts to Discuss

1. **Database Query Optimization**: Indexes, query plans, N+1 problems
2. **Connection Pooling**: HikariCP tuning, pool sizing
3. **Caching Strategies**: Multi-level caching, cache warming, TTL strategies
4. **JVM Tuning**: G1GC, heap sizing, GC pause optimization
5. **Asynchronous Processing**: Kafka, event-driven architecture
6. **Performance Profiling**: APM tools, profilers, metrics
7. **Load Testing**: Simulating production traffic
8. **Monitoring**: Application performance monitoring, alerting

### What to Emphasize in Interviews

1. **Metrics-Driven Approach**: How you identified bottlenecks using data
2. **Prioritization**: How you decided what to optimize first (80/20 rule)
3. **Trade-off Analysis**: Why you chose specific optimizations
4. **Measurement**: How you validated improvements
5. **Iterative Process**: How you refined solutions based on feedback
6. **Team Collaboration**: How you worked with DBAs, DevOps, etc.

### Sample Interview Answer Structure

"When optimizing our healthcare platform, I started by profiling the application using YourKit and APM tools. I found that 42% of CPU time was spent in patient search queries. I analyzed the database using pg_stat_statements and identified missing indexes and N+1 query problems. I optimized the queries, added composite indexes, and implemented entity graphs to fix N+1 issues. This reduced query time from 648ms to 89ms. I also implemented a multi-level caching strategy with Redis, which increased cache hit rate from 15% to 87%. Finally, I tuned the JVM for G1GC and moved audit logging to Kafka. Overall, we reduced P95 latency by 67% and met our SLA targets."

---

## Follow-up Questions

### Technical Questions

1. **How did you determine which indexes to add?**
   - Query analysis, execution plans, access patterns

2. **What would you do if cache hit rate was still low after optimization?**
   - Analyze access patterns, adjust TTL, consider different caching strategies

3. **How did you handle cache invalidation?**
   - Write-through, write-back, cache aside patterns

4. **What monitoring did you implement?**
   - APM, database metrics, JVM metrics, custom metrics

5. **How did you test optimizations before production?**
   - Load testing, canary deployments, feature flags

### Architectural Questions

6. **When would you use Redis vs. another cache?**
   - Use cases, data structures, persistence requirements

7. **How did you handle cache consistency?**
   - TTL, invalidation strategies, eventual consistency

8. **What about cache stampede prevention?**
   - Locking, probabilistic early expiration, refresh-ahead

9. **How did you size the connection pool?**
   - Based on query time, concurrency requirements, database capacity

### Leadership Questions

10. **How did you prioritize optimizations?**
    - Impact vs. effort analysis, business value

11. **How did you communicate progress to stakeholders?**
    - Metrics, demos, regular updates

12. **What would you do differently next time?**
    - More load testing, better monitoring, involve DBAs earlier

---

## Related Topics

- [Performance Optimization](../../sections/12-performance-optimization/README.md)
- [JVM Internals](../../sections/07-jvm-internals/README.md)
- [Database Design](../../sections/03-collections-framework/README.md)
- [Multithreading & Concurrency](../../sections/04-multithreading-concurrency/README.md)

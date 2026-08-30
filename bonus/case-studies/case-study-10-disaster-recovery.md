# Case Study 10: Disaster Recovery and Business Continuity

## Difficulty: Hard 🔴

---

## Scenario Context

### Company Background
**CloudScale** is a SaaS platform providing project management and collaboration tools to 100,000+ enterprise customers. The platform handles 1 million daily active users, 10 million projects, and processes 50 million API calls per day.

### Technical Stack
- **Application**: Spring Boot microservices (25 services)
- **Primary Region**: AWS us-east-1 (Virginia)
- **Database**: Amazon Aurora PostgreSQL (multi-AZ)
- **Cache**: Amazon ElastiCache Redis (cluster mode)
- **Storage**: Amazon S3 (with cross-region replication)
- **Message Queue**: Amazon MSK (Kafka)
- **Load Balancer**: AWS Application Load Balancer
- **CDN**: CloudFront
- **Monitoring**: Datadog, CloudWatch
- **Deployment**: Kubernetes with EKS

### Business Context
The company was experiencing:
- **Single Region Risk**: All infrastructure in us-east-1, creating single point of failure
- **RPO/RTO Gaps**: 1-hour RPO and 4-hour RTO not meeting enterprise SLA requirements
- **Compliance Requirements**: SOC 2 Type II requiring disaster recovery documentation
- **Customer Pressure**: Enterprise customers demanding 99.99% availability
- **Cost Concerns**: Full active-active setup too expensive

### Constraints
- **Budget**: $50K/month additional budget for DR
- **RPO**: 15-minute Recovery Point Objective
- **RTO**: 1-hour Recovery Time Objective
- **Data Consistency**: Must maintain data integrity across regions
- **Performance**: DR solution cannot impact primary region performance
- **Complexity**: Must be manageable with current team size

---

## Problem Statement

**Primary Challenge**: Design and implement a disaster recovery solution that meets enterprise SLA requirements (99.99% availability, 15-minute RPO, 1-hour RTO) within budget constraints.

**Specific Problems**:
1. How to achieve 15-minute RPO without full active-active?
2. How to failover to secondary region within 1 hour?
3. How to ensure data consistency across regions?
4. How to test DR without impacting production?
5. How to manage complexity within current team?

**Success Criteria**:
- RPO ≤ 15 minutes
- RTO ≤ 1 hour
- 99.99% availability target
- Monthly DR testing without production impact
- Budget ≤ $50K/month additional
- Automated failover capability

---

## Analysis Phase

### Current Architecture Assessment

**Primary Region (us-east-1)**:
```
┌─────────────────────────────────────────────────────────────┐
│                  AWS us-east-1 (Primary)                     │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   ALB        │  │   EKS        │  │   Aurora      │      │
│  │  (Public)    │  │  (K8s)       │  │  (PostgreSQL) │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  ElastiCache │  │     MSK      │  │     S3       │      │
│  │   (Redis)    │  │   (Kafka)    │  │  (Storage)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

**Risk Analysis**:

| Risk | Probability | Impact | Current Mitigation |
|------|-------------|--------|-------------------|
| Region outage | Low | Critical | None (single region) |
| AZ outage | Medium | High | Multi-AZ Aurora |
| Database failure | Low | Critical | Multi-AZ, automated failover |
| Network partition | Low | High | None |
| Human error | Medium | Medium | Backups, rollback |

**Gap Analysis**:

| Requirement | Current | Target | Gap |
|-------------|---------|--------|-----|
| Availability | 99.9% | 99.99% | 0.09% |
| RPO | 1 hour | 15 min | 45 min |
| RTO | 4 hours | 1 hour | 3 hours |
| DR Testing | Quarterly | Monthly | Frequency |
| Documentation | Basic | Comprehensive | Detail |

---

## Solution Approach

### Strategy: Pilot Light with Automated Failover

**Architecture Decision**: Pilot Light model with hot standby databases and cold compute
- **Hot Standby**: Databases, cache, message queues always running in secondary region
- **Cold Compute**: Kubernetes cluster provisioned but scaled to zero
- **Automated Failover**: Infrastructure as Code with automated deployment
- **Data Replication**: Continuous replication from primary to secondary

**DR Architecture**:
```
┌─────────────────────────────────────────────────────────────┐
│              AWS us-east-1 (Primary - Active)                │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   ALB        │  │   EKS        │  │   Aurora      │      │
│  │  (Active)    │  │  (Active)    │  │  (Primary)    │      │
│  └──────────────┘  └──────────────┘  └───┬──────────┘      │
│  ┌──────────────┐  ┌──────────────┐      │ Replication     │
│  │  ElastiCache │  │     MSK      │      │                 │
│  │   (Active)   │  │   (Active)   │      │                 │
│  └──────────────┘  └──────────────┘      │                 │
└───────────────────────────────────────────┼─────────────────┘
                                            │
                              ┌─────────────▼─────────────┐
                              │    Cross-Region Replication│
                              └─────────────┬─────────────┘
                                            │
┌───────────────────────────────────────────▼─────────────────┐
│              AWS us-west-2 (Secondary - Standby)             │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   ALB        │  │   EKS        │  │   Aurora      │      │
│  │  (Standby)   │  │ (Scaled to 0)│  │  (Replica)    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  ElastiCache │  │     MSK      │  │     S3       │      │
│  │  (Replica)   │  │  (Replica)   │  │  (Replicated)│      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation Details

### Phase 1: Database Replication Setup

#### Aurora Cross-Region Replica

```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.replica")
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public DataSource routingDataSource(
            @Qualifier("primaryDataSource") DataSource primary,
            @Qualifier("replicaDataSource") DataSource replica) {
        
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("primary", primary);
        targetDataSources.put("replica", replica);
        
        RoutingDataSource routingDataSource = new RoutingDataSource();
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(primary);
        
        return routingDataSource;
    }
}

public class RoutingDataSource extends AbstractRoutingDataSource {
    
    @Override
    protected Object determineCurrentLookupKey() {
        // Check if we're in disaster recovery mode
        if (DisasterRecoveryService.isActive()) {
            return "replica";
        }
        
        // Check if this is a read operation and we can use replica
        if (isReadOperation() && 
            FeatureFlagService.isEnabled("use-read-replica")) {
            return "replica";
        }
        
        return "primary";
    }
    
    private boolean isReadOperation() {
        String method = DatabaseContextHolder.getMethod();
        return method != null && method.startsWith("get") || 
               method.startsWith("find") || method.startsWith("query");
    }
}
```

#### Replication Lag Monitor

```java
@Service
public class ReplicationLagMonitor {
    
    @Autowired
    private JdbcTemplate primaryTemplate;
    
    @Autowired
    private JdbcTemplate replicaTemplate;
    
    @Autowired
    private AlertService alertService;
    
    @Scheduled(fixedDelay = 30000) // Every 30 seconds
    public void checkReplicationLag() {
        try {
            // Get primary position
            Long primaryLsn = primaryTemplate.queryForObject(
                "SELECT pg_current_wal_lsn()", Long.class);
            
            // Get replica position
            Long replicaLsn = replicaTemplate.queryForObject(
                "SELECT pg_last_wal_replay_lsn()", Long.class);
            
            // Calculate lag in bytes
            long lagBytes = primaryLsn - replicaLsn;
            
            // Convert to approximate time (rough estimate)
            long lagSeconds = estimateLagSeconds(lagBytes);
            
            log.info("Replication lag: {} seconds", lagSeconds);
            
            // Alert if lag exceeds threshold
            if (lagSeconds > 900) { // 15 minutes
                alertService.sendAlert(
                    "Replication lag exceeded threshold",
                    Map.of("lag_seconds", lagSeconds, "threshold", 900));
            }
            
            // Update metrics
            Metrics.gauge("database.replication.lag.seconds", lagSeconds);
            
        } catch (Exception e) {
            log.error("Failed to check replication lag", e);
            alertService.sendAlert("Replication lag check failed", e);
        }
    }
    
    private long estimateLagSeconds(long lagBytes) {
        // Rough estimate: 1MB ≈ 1 second of write activity
        // This should be calibrated based on actual workload
        return lagBytes / (1024 * 1024);
    }
}
```

### Phase 2: Cache and Message Queue Replication

#### Redis Cross-Region Replication

```java
@Configuration
public class CacheConfig {
    
    @Bean
    @Primary
    public RedisTemplate<String, Object> primaryRedisTemplate(
            RedisConnectionFactory primaryConnectionFactory) {
        
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(primaryConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
    
    @Bean
    public RedisTemplate<String, Object> replicaRedisTemplate(
            @Qualifier("replicaConnectionFactory") 
            RedisConnectionFactory replicaConnectionFactory) {
        
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(replicaConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
    
    @Bean
    public CacheManager cacheManager(
            RedisConnectionFactory primaryConnectionFactory) {
        
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .disableCachingNullValues();
        
        return RedisCacheManager.builder(primaryConnectionFactory)
            .cacheDefaults(config)
            .build();
    }
}

@Service
public class DualWriteCacheService {
    
    @Autowired
    @Qualifier("primaryRedisTemplate")
    private RedisTemplate<String, Object> primaryTemplate;
    
    @Autowired
    @Qualifier("replicaRedisTemplate")
    private RedisTemplate<String, Object> replicaTemplate;
    
    public void put(String key, Object value) {
        // Write to primary
        primaryTemplate.opsForValue().set(key, value);
        
        // Asynchronously write to replica
        CompletableFuture.runAsync(() -> {
            try {
                replicaTemplate.opsForValue().set(key, value);
            } catch (Exception e) {
                log.error("Failed to write to replica cache", e);
            }
        });
    }
    
    public Object get(String key) {
        // Try primary first
        Object value = primaryTemplate.opsForValue().get(key);
        
        if (value == null && DisasterRecoveryService.isActive()) {
            // Fallback to replica in DR mode
            value = replicaTemplate.opsForValue().get(key);
        }
        
        return value;
    }
}
```

#### Kafka Cross-Region Replication

```yaml
# kafka-mirror-maker.properties
clusters = primary, replica
primary.bootstrap.servers = primary-kafka.us-east-1.amazonaws.com:9092
replica.bootstrap.servers = replica-kafka.us-west-2.amazonaws.com:9092

primary->replica.enabled = true
primary->replica.topics = .*

# Replication policy
replication.policy.class = org.apache.kafka.connect.mirror.IdentityReplicationPolicy

# Offset sync
replication.offset.sync.interval.seconds = 5
replication.offset.sync.commit.interval.seconds = 5

# Heartbeat
replication.heartbeat.interval.seconds = 5
```

```java
@Service
public class KafkaReplicationMonitor {
    
    @Autowired
    private KafkaAdmin kafkaAdmin;
    
    @Scheduled(fixedDelay = 60000) // Every minute
    public void checkReplicationLag() {
        try {
            // Get consumer group offsets for primary
            Map<String, Map<TopicPartition, OffsetAndMetadata>> primaryOffsets = 
                getConsumerGroupOffsets("primary");
            
            // Get consumer group offsets for replica
            Map<String, Map<TopicPartition, OffsetAndMetadata>> replicaOffsets = 
                getConsumerGroupOffsets("replica");
            
            // Calculate lag for each topic
            for (String groupId : primaryOffsets.keySet()) {
                Map<TopicPartition, OffsetAndMetadata> primaryGroup = 
                    primaryOffsets.get(groupId);
                Map<TopicPartition, OffsetAndMetadata> replicaGroup = 
                    replicaOffsets.getOrDefault(groupId, Map.of());
                
                for (TopicPartition tp : primaryGroup.keySet()) {
                    long primaryOffset = primaryGroup.get(tp).offset();
                    long replicaOffset = replicaGroup.containsKey(tp) ? 
                        replicaGroup.get(tp).offset() : 0;
                    
                    long lag = primaryOffset - replicaOffset;
                    
                    if (lag > 10000) { // Threshold
                        alertService.sendAlert(
                            "Kafka replication lag high",
                            Map.of(
                                "topic", tp.topic(),
                                "partition", tp.partition(),
                                "lag", lag
                            ));
                    }
                }
            }
            
        } catch (Exception e) {
            log.error("Failed to check Kafka replication lag", e);
        }
    }
}
```

### Phase 3: Infrastructure as Code for Secondary Region

#### Terraform Configuration for Pilot Light

```hcl
# secondary-region/main.tf

provider "aws" {
  region = "us-west-2"
  alias  = "secondary"
}

# Aurora Replica (always running)
resource "aws_rds_cluster" "aurora_replica" {
  provider = aws.secondary
  
  engine                = "aurora-postgresql"
  engine_version        = "14.7"
  database_name         = "cloudscale"
  master_username       = var.db_username
  master_password       = var.db_password
  
  replication_source_identifier = "arn:aws:rds:us-east-1:123456789012:cluster:primary-aurora"
  
  backup_retention_period = 7
  preferred_backup_window = "03:00-04:00"
  
  db_instance_class = "db.r6g.xlarge"
  
  tags = {
    Environment = "secondary"
    Purpose     = "disaster-recovery"
  }
}

# ElastiCache Replica (always running)
resource "aws_elasticache_replication_group" "redis_replica" {
  provider = aws.secondary
  
  replication_group_id = "cloudscale-redis-replica"
  description          = "Redis replica for disaster recovery"
  
  node_type            = "cache.r6g.large"
  number_cache_clusters = 2
  
  automatic_failover_enabled = true
  multi_az_enabled          = true
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  
  auth_token = var.redis_auth_token
  
  tags = {
    Environment = "secondary"
    Purpose     = "disaster-recovery"
  }
}

# EKS Cluster (provisioned but scaled to zero)
resource "aws_eks_cluster" "secondary" {
  provider = aws.secondary
  
  name     = "cloudscale-secondary"
  role_arn = aws_iam_role.eks_role.arn
  
  version = "1.27"
  
  vpc_config {
    subnet_ids = var.private_subnet_ids
    security_group_ids = [aws_security_group.eks_sg.id]
  }
  
  tags = {
    Environment = "secondary"
    Purpose     = "disaster-recovery"
  }
}

# Node Group (initially zero nodes)
resource "aws_eks_node_group" "secondary" {
  provider = aws.secondary
  
  cluster_name = aws_eks_cluster.secondary.name
  node_group_name = "secondary-nodes"
  node_role_arn = aws_iam_role.node_role.arn
  
  subnet_ids = var.private_subnet_ids
  
  scaling_config {
    desired_size = 0  # Start with zero
    min_size     = 0
    max_size     = 20
  }
  
  instance_types = ["m6g.xlarge"]
  
  labels = {
    environment = "secondary"
  }
  
  tags = {
    Environment = "secondary"
    Purpose     = "disaster-recovery"
  }
}
```

#### Automated Failover Script

```java
@Service
public class DisasterRecoveryService {
    
    @Autowired
    private EksService eksService;
    
    @Autowired
    private RdsService rdsService;
    
    @Autowired
    private ElastiCacheService cacheService;
    
    @Autowired
    private Route53Service route53Service;
    
    private static boolean active = false;
    
    public void initiateFailover() {
        log.warn("Initiating disaster recovery failover");
        
        try {
            // Step 1: Promote Aurora replica
            log.info("Step 1: Promoting Aurora replica");
            rdsService.promoteReadReplica("cloudscale-aurora-replica");
            
            // Step 2: Scale up EKS cluster
            log.info("Step 2: Scaling up EKS cluster");
            eksService.scaleNodeGroup("secondary-nodes", 10);
            
            // Step 3: Wait for EKS to be ready
            log.info("Step 3: Waiting for EKS to be ready");
            waitForEksReady(Duration.ofMinutes(15));
            
            // Step 4: Deploy applications
            log.info("Step 4: Deploying applications");
            deployApplications();
            
            // Step 5: Update DNS
            log.info("Step 5: Updating DNS");
            route53Service.updateRecordSet(
                "api.cloudscale.com",
                "secondary-alb.us-west-2.elb.amazonaws.com");
            
            // Step 6: Verify health
            log.info("Step 6: Verifying health");
            verifySystemHealth();
            
            // Mark as active
            active = true;
            log.warn("Disaster recovery failover completed successfully");
            
        } catch (Exception e) {
            log.error("Failover failed, attempting rollback", e);
            rollbackFailover();
            throw new DisasterRecoveryException("Failover failed", e);
        }
    }
    
    private void deployApplications() {
        // Use Helm to deploy all applications
        List<String> charts = List.of(
            "api-gateway", "user-service", "project-service",
            "notification-service", "analytics-service");
        
        for (String chart : charts) {
            helmService.upgrade(chart, "secondary");
        }
    }
    
    private void waitForEksReady(Duration timeout) {
        Instant start = Instant.now();
        
        while (Duration.between(start, Instant.now()).compareTo(timeout) < 0) {
            if (eksService.isClusterReady("cloudscale-secondary")) {
                return;
            }
            try {
                Thread.sleep(30000); // Check every 30 seconds
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new DisasterRecoveryException("Interrupted while waiting for EKS");
            }
        }
        
        throw new DisasterRecoveryException("EKS did not become ready in time");
    }
    
    private void verifySystemHealth() {
        // Health check all critical services
        List<String> services = List.of(
            "http://secondary-alb.us-west-2.elb.amazonaws.com/health",
            "http://secondary-alb.us-west-2.elb.amazonaws.com/api/users/health",
            "http://secondary-alb.us-west-2.elb.amazonaws.com/api/projects/health"
        );
        
        for (String url : services) {
            if (!healthCheckService.isHealthy(url)) {
                throw new DisasterRecoveryException(
                    "Service health check failed: " + url);
            }
        }
    }
    
    public void rollbackFailover() {
        log.error("Initiating failover rollback");
        
        try {
            // Rollback DNS
            route53Service.updateRecordSet(
                "api.cloudscale.com",
                "primary-alb.us-east-1.elb.amazonaws.com");
            
            // Scale down EKS
            eksService.scaleNodeGroup("secondary-nodes", 0);
            
            // Demote Aurora (if promoted)
            // Note: Aurora doesn't support demotion, would need to recreate replica
            
            active = false;
            log.error("Rollback completed");
            
        } catch (Exception e) {
            log.error("Rollback failed", e);
            alertService.sendCriticalAlert("DR rollback failed", e);
        }
    }
    
    public static boolean isActive() {
        return active;
    }
}
```

### Phase 4: Automated Failover Trigger

#### Health Check and Failover Controller

```java
@RestController
@RequestMapping("/admin/dr")
public class DisasterRecoveryController {
    
    @Autowired
    private DisasterRecoveryService drService;
    
    @Autowired
    private HealthCheckService healthCheckService;
    
    @Autowired
    private ApprovalService approvalService;
    
    @PostMapping("/failover/initiate")
    public ResponseEntity<FailoverResponse> initiateFailover(
            @RequestBody FailoverRequest request) {
        
        log.warn("Failover requested by: {}", request.getRequester());
        
        // Verify approval for manual failover
        if (!request.isEmergency() && 
            !approvalService.isApproved(request.getRequester(), "dr-failover")) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN)
                .body(FailoverResponse.failed("Approval required"));
        }
        
        // Execute failover asynchronously
        CompletableFuture.runAsync(() -> {
            try {
                drService.initiateFailover();
            } catch (Exception e) {
                log.error("Failover failed", e);
            }
        });
        
        return ResponseEntity.ok(FailoverResponse.initiated());
    }
    
    @GetMapping("/status")
    public ResponseEntity<DRStatus> getStatus() {
        DRStatus status = new DRStatus();
        status.setActive(DisasterRecoveryService.isActive());
        status.setPrimaryRegionHealthy(healthCheckService.isRegionHealthy("us-east-1"));
        status.setSecondaryRegionHealthy(healthCheckService.isRegionHealthy("us-west-2"));
        status.setReplicationLag(getReplicationLag());
        status.setLastTestRun(getLastTestRun());
        
        return ResponseEntity.ok(status);
    }
}
```

#### Automated Health Check Monitor

```java
@Service
public class AutomatedFailoverMonitor {
    
    @Autowired
    private HealthCheckService healthCheckService;
    
    @Autowired
    private DisasterRecoveryService drService;
    
    @Autowired
    private AlertService alertService;
    
    @Value("${dr.failover.automated:false}")
    private boolean automatedFailoverEnabled;
    
    @Scheduled(fixedDelay = 60000) // Every minute
    public void monitorPrimaryRegion() {
        if (DisasterRecoveryService.isActive()) {
            // Already in DR mode, monitor secondary
            monitorSecondaryRegion();
            return;
        }
        
        // Check primary region health
        RegionHealth health = healthCheckService.checkRegionHealth("us-east-1");
        
        if (!health.isHealthy()) {
            log.error("Primary region health check failed: {}", health);
            
            // Count consecutive failures
            int failureCount = incrementFailureCount();
            
            if (failureCount >= 3) { // 3 consecutive failures
                log.error("Primary region unhealthy for 3 consecutive checks");
                
                if (automatedFailoverEnabled) {
                    log.warn("Initiating automated failover");
                    alertService.sendCriticalAlert(
                        "Automated failover initiated", health);
                    
                    try {
                        drService.initiateFailover();
                    } catch (Exception e) {
                        log.error("Automated failover failed", e);
                        alertService.sendCriticalAlert(
                            "Automated failover failed", e);
                    }
                } else {
                    alertService.sendCriticalAlert(
                        "Primary region down - manual failover required", health);
                }
            }
        } else {
            resetFailureCount();
        }
    }
    
    private int incrementFailureCount() {
        // Use Redis to track failure count across instances
        String key = "dr:failover:failure-count";
        Long count = redisTemplate.opsForValue().increment(key);
        redisTemplate.expire(key, Duration.ofMinutes(5));
        return count.intValue();
    }
    
    private void resetFailureCount() {
        redisTemplate.delete("dr:failover:failure-count");
    }
}
```

### Phase 5: DR Testing Framework

#### Monthly DR Test Orchestration

```java
@Service
public class DisasterRecoveryTestService {
    
    @Autowired
    private DisasterRecoveryService drService;
    
    @Autowired
    private Route53Service route53Service;
    
    @Autowired
    private TrafficService trafficService;
    
    @Scheduled(cron = "0 0 1 * * ?") // First day of month at midnight
    public void runMonthlyDrTest() {
        log.info("Starting monthly DR test");
        
        DrTestReport report = new DrTestReport();
        report.setTestDate(Instant.now());
        
        try {
            // Step 1: Notify stakeholders
            notificationService.notifyDrTestStart();
            
            // Step 2: Scale up secondary
            log.info("Scaling up secondary region");
            eksService.scaleNodeGroup("secondary-nodes", 5);
            waitForEksReady(Duration.ofMinutes(15));
            
            // Step 3: Deploy applications
            log.info("Deploying applications to secondary");
            deployApplications();
            
            // Step 4: Sync data
            log.info("Verifying data sync");
            verifyDataSync();
            
            // Step 5: Route test traffic
            log.info("Routing test traffic to secondary");
            String testDns = "test.cloudscale.com";
            route53Service.updateRecordSet(
                testDns, "secondary-alb.us-west-2.elb.amazonaws.com");
            
            // Step 6: Run synthetic tests
            log.info("Running synthetic tests");
            TestResults testResults = runSyntheticTests();
            report.setTestResults(testResults);
            
            // Step 7: Measure performance
            log.info("Measuring performance");
            PerformanceMetrics metrics = measurePerformance();
            report.setPerformanceMetrics(metrics);
            
            // Step 8: Verify RPO/RTO
            log.info("Verifying RPO/RTO");
            RpoRtoVerification verification = verifyRpoRto();
            report.setRpoRtoVerification(verification);
            
            // Step 9: Cleanup
            log.info("Cleaning up test environment");
            route53Service.updateRecordSet(
                testDns, "primary-alb.us-east-1.elb.amazonaws.com");
            eksService.scaleNodeGroup("secondary-nodes", 0);
            
            report.setStatus("PASSED");
            log.info("Monthly DR test completed successfully");
            
        } catch (Exception e) {
            log.error("DR test failed", e);
            report.setStatus("FAILED");
            report.setFailureReason(e.getMessage());
            
            // Ensure cleanup
            cleanupTestEnvironment();
        }
        
        // Save report
        drTestReportRepository.save(report);
        
        // Notify stakeholders
        notificationService.notifyDrTestComplete(report);
    }
    
    private TestResults runSyntheticTests() {
        TestResults results = new TestResults();
        
        // Test user authentication
        boolean authTest = testAuthentication();
        results.setAuthenticationTest(authTest);
        
        // Test project creation
        boolean projectTest = testProjectCreation();
        results.setProjectCreationTest(projectTest);
        
        // Test API endpoints
        boolean apiTest = testApiEndpoints();
        results.setApiTest(apiTest);
        
        // Test database connectivity
        boolean dbTest = testDatabaseConnectivity();
        results.setDatabaseTest(dbTest);
        
        return results;
    }
    
    private RpoRtoVerification verifyRpoRto() {
        RpoRtoVerification verification = new RpoRtoVerification();
        
        // Measure actual RPO
        Instant lastSync = getLastDataSyncTime();
        Duration rpo = Duration.between(lastSync, Instant.now());
        verification.setActualRpo(rpo);
        verification.setRpoTarget(Duration.ofMinutes(15));
        verification.setRpoMet(rpo.compareTo(Duration.ofMinutes(15)) <= 0);
        
        // Measure actual RTO (time from test start to ready)
        Duration rto = Duration.between(
            report.getTestDate(), 
            Instant.now());
        verification.setActualRto(rto);
        verification.setRtoTarget(Duration.ofHours(1));
        verification.setRtoMet(rto.compareTo(Duration.ofHours(1)) <= 0);
        
        return verification;
    }
}
```

### Phase 6: Documentation and Runbooks

#### DR Runbook Template

```java
@Component
public class DisasterRecoveryRunbook {
    
    public String generateRunbook() {
        return """
            # Disaster Recovery Runbook
            
            ## Overview
            This document outlines the procedures for disaster recovery at CloudScale.
            
            ## RPO/RTO Targets
            - RPO: 15 minutes
            - RTO: 1 hour
            
            ## Architecture
            - Primary Region: us-east-1 (Virginia)
            - Secondary Region: us-west-2 (Oregon)
            - DR Strategy: Pilot Light with automated failover
            
            ## Pre-Failover Checklist
            - [ ] Verify secondary region infrastructure is healthy
            - [ ] Verify replication lag is < 5 minutes
            - [ ] Verify backups are current
            - [ ] Notify stakeholders
            - [ ] Update status page
            
            ## Failover Procedure
            
            ### Automated Failover
            1. System detects primary region failure (3 consecutive health check failures)
            2. Automated failover initiates if enabled
            3. Monitor failover progress in DR dashboard
            
            ### Manual Failover
            1. Access DR dashboard: https://dr.cloudscale.com
            2. Click "Initiate Failover"
            3. Enter approval code (requires director-level approval)
            4. Monitor failover progress
            
            ### Failover Steps
            1. Promote Aurora replica (5 minutes)
            2. Scale up EKS cluster (10 minutes)
            3. Deploy applications (15 minutes)
            4. Update DNS (2 minutes)
            5. Verify health (5 minutes)
            
            Total estimated time: 37 minutes
            
            ## Post-Failover Checklist
            - [ ] Verify all services are healthy
            - [ ] Verify data integrity
            - [ ] Run smoke tests
            - [ ] Update status page
            - [ ] Notify customers
            - [ ] Monitor performance metrics
            - [ ] Document incident
            
            ## Failback Procedure
            1. Verify primary region is restored
            2. Resync data from secondary to primary
            3. Scale up primary region
            4. Deploy applications to primary
            5. Run smoke tests
            6. Update DNS back to primary
            7. Scale down secondary region
            
            ## Testing Schedule
            - Monthly automated DR test (non-disruptive)
            - Quarterly full failover test (scheduled maintenance)
            - Annual tabletop exercise
            
            ## Contact Information
            - DR Lead: dr-lead@cloudscale.com
            - Infrastructure Lead: infra-lead@cloudscale.com
            - On-Call: +1-555-DR-HELP
            """;
    }
}
```

---

## Results & Metrics

### Quantitative Outcomes

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Availability | 99.9% | 99.99% | 0.09% improvement |
| RPO | 1 hour | 5 minutes | 92% improvement |
| RTO | 4 hours | 37 minutes | 85% improvement |
| DR Testing | Quarterly | Monthly | 4x frequency |
| Documentation | Basic | Comprehensive | Full runbooks |
| Cost | $0 | $35K/month | Within budget |

### Qualitative Improvements

1. **Enterprise Readiness**: Met SOC 2 Type II requirements for DR
2. **Customer Confidence**: Enterprise customers renewed with 99.99% SLA
3. **Operational Excellence**: Monthly testing increased team confidence
4. **Risk Reduction**: Eliminated single region failure point
5. **Compliance**: Full documentation and audit trail

### Lessons Learned

#### What Worked Well

1. **Pilot Light Model**: Balanced cost and recovery time effectively
2. **Automated Testing**: Monthly tests caught issues before production
3. **Infrastructure as Code**: Enabled reliable and repeatable failover
4. **Comprehensive Monitoring**: Early detection of replication issues
5. **Clear Documentation**: Runbooks reduced failover time

#### What Didn't Work

1. **DNS Propagation Delay**: CloudFront caching caused some traffic to still route to primary
2. **Cold Start Time**: EKS scaling took longer than expected (15 min vs 10 min)
3. **Data Sync Complexity**: Some data inconsistencies during initial tests
4. **Notification Gaps**: Initial customer communication was delayed
5. **Testing Fatigue**: Monthly tests became repetitive, team engagement dropped

---

## Interview Takeaways

### Key Technical Concepts to Discuss

1. **DR Strategies**: Active-active, pilot light, warm standby, cold standby
2. **RPO/RTO**: Recovery Point Objective, Recovery Time Objective
3. **Database Replication**: Cross-region replication, replication lag monitoring
4. **Infrastructure as Code**: Terraform, automated provisioning
5. **Health Checking**: Multi-region health checks, automated failover
6. **DNS Management**: Route53, failover routing, TTL considerations
7. **Testing Strategy**: Monthly tests, chaos engineering, game days
8. **Documentation**: Runbooks, playbooks, incident response

### What to Emphasize in Interviews

1. **Trade-off Analysis**: Why pilot light over active-active (cost vs. RTO)
2. **Risk Mitigation**: How you addressed single points of failure
3. **Automation**: How you automated failover to reduce human error
4. **Testing**: How you validated DR without impacting production
5. **Cost Management**: How you met SLA requirements within budget
6. **Communication**: How you handled customer communication during incidents

### Sample Interview Answer Structure

"When designing disaster recovery for our SaaS platform, we chose a pilot light model to balance cost and recovery time. We implemented cross-region database replication with hot standby databases in us-west-2, while keeping compute resources scaled to zero. This approach gave us a 5-minute RPO and 37-minute RTO at $35K/month, well within our $50K budget. We automated failover with infrastructure as code and implemented monthly testing to validate the system. This design helped us achieve 99.99% availability and meet enterprise customer SLA requirements."

---

## Follow-up Questions

### Technical Questions

1. **How did you handle data consistency during failover?**
   - Replication lag monitoring, conflict resolution, data validation

2. **What about cached data during failover?**
   - Cache warming, TTL settings, cache invalidation

3. **How did you handle DNS propagation during failover?**
   - Route53 failover routing, low TTLs, CDN invalidation

4. **What happened to in-flight transactions during failover?**
   - Transaction rollback, idempotency, retry logic

5. **How did you test DR without impacting production?**
   - Separate test domain, synthetic traffic, read-only tests

### Architectural Questions

6. **Why pilot light instead of active-active?**
   - Cost considerations, complexity, RPO/RTO requirements

7. **How did you decide which services to replicate?**
   - Criticality analysis, dependency mapping, data classification

8. **What about third-party services during DR?**
   - Multi-region endpoints, fallback providers, graceful degradation

9. **How did you handle stateful services?**
   - State replication, session affinity, client-side state

### Leadership Questions

10. **How did you get approval for the DR budget?**
    - Risk analysis, ROI calculation, customer requirements

11. **How did you ensure team readiness for DR?**
    - Training, documentation, regular testing, on-call rotation

12. **What would you do differently next time?**
    - Active-active for critical services, better cache strategy, improved testing

---

## Related Topics

- [System Design: High Availability](../../sections/14-system-design/README.md)
- [Performance Optimization](../../sections/12-performance-optimization/README.md)
- [Spring Boot: Cloud Deployment](../../sections/13-spring-boot-advanced/README.md)
- [Behavioral: Crisis Management](../../sections/15-behavioral-scenarios/README.md)

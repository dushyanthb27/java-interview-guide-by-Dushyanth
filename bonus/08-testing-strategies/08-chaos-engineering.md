# Chaos Engineering

## 📋 Overview

Chaos engineering is the discipline of experimenting on a system to build confidence in its capability to withstand turbulent conditions. This guide covers chaos testing principles, fault injection patterns, resilience testing in Java, Chaos Monkey integration, and production chaos testing strategies.

## 🎯 Why Chaos Engineering?

Chaos engineering addresses the reality that distributed systems fail:
- **Proactive Resilience**: Find weaknesses before they cause outages
- **Confidence in Production**: Verify system handles failures gracefully
- **Reduce MTTR**: Practice incident response and recovery procedures
- **Validate Redundancy**: Ensure failover mechanisms actually work
- **Test Assumptions**: Challenge assumptions about system behavior

## 🔧 Chaos Engineering Principles

### The Chaos Engineering Process

1. **Define Steady State**: Measure normal system behavior
2. **Hypothesize**: Assume that system will remain in steady state
3. **Introduce Fault**: Inject real-world failures
4. **Observe**: Monitor system response
5. **Refine**: Improve system based on findings

## 🎭 Fault Injection Patterns

### Latency Injection

```java
@SpringBootTest
@DisplayName("Latency Injection Tests")
class LatencyInjectionTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @DisplayName("Should handle slow database responses")
    void shouldHandleSlowDatabaseResponses() {
        // Simulate slow database
        when(userRepository.findById(any()))
            .thenAnswer(invocation -> {
                Thread.sleep(2000); // 2 second delay
                return Optional.of(User.builder()
                    .id(1L)
                    .email("test@example.com")
                    .build());
            });
        
        // System should handle this gracefully
        long startTime = System.currentTimeMillis();
        User user = userService.findById(1L).orElseThrow();
        long duration = System.currentTimeMillis() - startTime;
        
        assertThat(user).isNotNull();
        assertThat(duration).isGreaterThan(2000);
        
        // Verify timeout handling
        // System should have appropriate timeout configuration
    }
}
```

### Exception Injection

```java
@SpringBootTest
@DisplayName("Exception Injection Tests")
class ExceptionInjectionTest {
    
    @Autowired
    private OrderService orderService;
    
    @MockBean
    private PaymentService paymentService;
    
    @Test
    @DisplayName("Should handle payment service failures")
    void shouldHandlePaymentServiceFailures() {
        when(paymentService.processPayment(any()))
            .thenThrow(new PaymentServiceUnavailableException());
        
        CreateOrderRequest request = CreateOrderRequest.builder()
            .customerId(1L)
            .items(List.of(OrderItem.builder()
                .productId(101L)
                .quantity(2)
                .build()))
            .build();
        
        // System should handle this gracefully
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(OrderCreationException.class)
            .hasMessageContaining("Payment service unavailable");
        
        // Verify order is not created
        // Verify customer is notified
    }
    
    @Test
    @DisplayName("Should handle database connection failures")
    void shouldHandleDatabaseConnectionFailures() {
        when(userRepository.findById(any()))
            .thenThrow(new DataAccessException("Connection timeout") {});
        
        // System should retry or fail gracefully
        assertThatThrownBy(() -> userService.findById(1L))
            .isInstanceOf(ServiceUnavailableException.class);
    }
}
```

### Resource Exhaustion

```java
@SpringBootTest
@DisplayName("Resource Exhaustion Tests")
class ResourceExhaustionTest {
    
    @Test
    @DisplayName("Should handle memory pressure")
    void shouldHandleMemoryPressure() {
        // Simulate memory pressure
        List<byte[]> memoryConsumer = new ArrayList<>();
        
        for (int i = 0; i < 1000; i++) {
            memoryConsumer.add(new byte[1024 * 1024]); // 1MB each
        }
        
        // System should handle OutOfMemoryError gracefully
        // or have proper limits in place
    }
    
    @Test
    @DisplayName("Should handle thread pool exhaustion")
    void shouldHandleThreadPoolExhaustion() {
        ExecutorService executor = Executors.newFixedThreadPool(10);
        
        // Submit more tasks than thread pool size
        List<Future<?>> futures = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            futures.add(executor.submit(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }));
        }
        
        // System should handle queue saturation
        // Either reject new tasks or have proper queueing
    }
}
```

## 🐒 Chaos Monkey Integration

### Spring Boot Chaos Monkey

```xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>chaos-monkey-spring-boot</artifactId>
    <version>3.0.1</version>
</dependency>
```

### Configuration

```yaml
# application.yml
chaos:
  monkey:
    enabled: true
    watcher:
      controller: true
      service: true
      repository: true
      rest: true
    assaults:
      level: 1 # Assault probability (1 = 100%)
      latencyActive: true
      latencyRangeStart: 1000
      latencyRangeEnd: 3000
      exceptionsActive: true
      killApplicationActive: false
      rate: 5 # Assault rate (5 per second)
```

### Custom Assaults

```java
@Component
public class CustomAssault implements ChaosMonkeyRequestAssault {
    
    @Override
    public void attack(RequestContext context) {
        // Custom chaos logic
        if (shouldAttack(context)) {
            injectLatency();
            orThrowException();
        }
    }
    
    private boolean shouldAttack(RequestContext context) {
        // Attack only specific endpoints
        return context.getMethod().equals("GET") &&
               context.getPath().contains("/api/critical");
    }
    
    private void injectLatency() {
        try {
            Thread.sleep(ThreadLocalRandom.current().nextInt(1000, 5000));
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## 🎯 Resilience Patterns Testing

### Circuit Breaker Testing

```java
@SpringBootTest
@DisplayName("Circuit Breaker Tests")
class CircuitBreakerTest {
    
    @Autowired
    private ExternalServiceClient externalServiceClient;
    
    @MockBean
    private ExternalService externalService;
    
    @Test
    @DisplayName("Should open circuit after failures")
    void shouldOpenCircuitAfterFailures() {
        // Simulate repeated failures
        when(externalService.call())
            .thenThrow(new ServiceUnavailableException());
        
        // Circuit should open after threshold
        for (int i = 0; i < 10; i++) {
            assertThatThrownBy(() -> externalServiceClient.call())
                .isInstanceOf(CircuitBreakerOpenException.class);
        }
        
        // Verify circuit is open
        // Subsequent calls should fail fast
    }
    
    @Test
    @DisplayName("Should close circuit after recovery")
    void shouldCloseCircuitAfterRecovery() {
        // First, open the circuit
        when(externalService.call())
            .thenThrow(new ServiceUnavailableException());
        
        for (int i = 0; i < 10; i++) {
            try {
                externalServiceClient.call();
            } catch (Exception e) {
                // Expected
            }
        }
        
        // Wait for circuit to attempt recovery
        await().atMost(30, TimeUnit.SECONDS)
            .until(() -> {
                when(externalService.call()).thenReturn("success");
                return "success".equals(externalServiceClient.call());
            });
    }
}
```

### Retry Pattern Testing

```java
@SpringBootTest
@DisplayName("Retry Pattern Tests")
class RetryPatternTest {
    
    @Autowired
    private RetryService retryService;
    
    @MockBean
    private ExternalService externalService;
    
    @Test
    @DisplayName("Should retry on transient failures")
    void shouldRetryOnTransientFailures() {
        AtomicInteger attemptCount = new AtomicInteger(0);
        
        when(externalService.call())
            .thenAnswer(invocation -> {
                int attempt = attemptCount.incrementAndGet();
                if (attempt < 3) {
                    throw new TransientException("Temporary failure");
                }
                return "success";
            });
        
        String result = retryService.callWithRetry();
        
        assertThat(result).isEqualTo("success");
        assertThat(attemptCount.get()).isEqualTo(3);
    }
    
    @Test
    @DisplayName("Should not retry on permanent failures")
    void shouldNotRetryOnPermanentFailures() {
        when(externalService.call())
            .thenThrow(new PermanentException("Permanent failure"));
        
        assertThatThrownBy(() -> retryService.callWithRetry())
            .isInstanceOf(PermanentException.class);
        
        verify(externalService, times(1)).call();
    }
}
```

### Bulkhead Pattern Testing

```java
@SpringBootTest
@DisplayName("Bulkhead Pattern Tests")
class BulkheadPatternTest {
    
    @Autowired
    private BulkheadService bulkheadService;
    
    @Test
    @DisplayName("Should limit concurrent executions")
    void shouldLimitConcurrentExecutions() throws InterruptedException {
        int bulkheadSize = 10;
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch completeLatch = new CountDownLatch(20);
        AtomicInteger successCount = new AtomicInteger(0);
        AtomicInteger rejectedCount = new AtomicInteger(0);
        
        // Submit 20 tasks to a bulkhead of size 10
        for (int i = 0; i < 20; i++) {
            new Thread(() -> {
                try {
                    startLatch.await();
                    bulkheadService.execute();
                    successCount.incrementAndGet();
                } catch (BulkheadFullException e) {
                    rejectedCount.incrementAndGet();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    completeLatch.countDown();
                }
            }).start();
        }
        
        startLatch.countDown();
        completeLatch.await(10, TimeUnit.SECONDS);
        
        // Some should succeed, some should be rejected
        assertThat(successCount.get()).isLessThanOrEqualTo(bulkheadSize);
        assertThat(rejectedCount.get()).isGreaterThan(0);
    }
}
```

## 🌐 Network Chaos Testing

### Network Latency Simulation

```java
@SpringBootTest
@DisplayName("Network Latency Tests")
class NetworkLatencyTest {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Test
    @DisplayName("Should handle high network latency")
    void shouldHandleHighNetworkLatency() {
        // Configure RestTemplate with timeout
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(5000);
        factory.setReadTimeout(5000);
        restTemplate.setRequestFactory(factory);
        
        // Simulate network latency using proxy or tool
        // This test would use a tool like Toxiproxy or tc (traffic control)
        
        ResponseEntity<String> response = restTemplate.getForEntity(
            "http://localhost:8080/api/resource",
            String.class
        );
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

### Packet Loss Simulation

```java
@SpringBootTest
@DisplayName("Packet Loss Tests")
class PacketLossTest {
    
    @Test
    @DisplayName("Should handle packet loss with retries")
    void shouldHandlePacketLossWithRetries() {
        // Simulate packet loss using network tools
        // This requires integration with network chaos tools
        
        // Application should retry on failure
        // Circuit breaker should handle sustained packet loss
    }
}
```

## 💾 Database Chaos Testing

### Database Connection Failures

```java
@SpringBootTest
@Testcontainers
@DisplayName("Database Chaos Tests")
class DatabaseChaosTest {
    
    @Container
    private static final PostgreSQLContainer<?> postgres = 
        new PostgreSQLContainer<>("postgres:15-alpine");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @DisplayName("Should handle database restart")
    void shouldHandleDatabaseRestart() {
        // Save data before restart
        User user = userRepository.save(
            User.builder()
                .email("test@example.com")
                .name("Test User")
                .build()
        );
        
        // Stop database
        postgres.stop();
        
        // Attempt operation - should fail gracefully
        assertThatThrownBy(() -> userRepository.findById(user.getId()))
            .isInstanceOf(DataAccessException.class);
        
        // Restart database
        postgres.start();
        
        // Wait for recovery
        await().atMost(30, TimeUnit.SECONDS)
            .until(() -> {
                try {
                    return userRepository.findById(user.getId()).isPresent();
                } catch (Exception e) {
                    return false;
                }
            });
        
        // Verify data after recovery
        Optional<User> recovered = userRepository.findById(user.getId());
        assertThat(recovered).isPresent();
    }
}
```

### Database Latency

```java
@Test
@DisplayName("Should handle slow database queries")
void shouldHandleSlowDatabaseQueries() {
    // Simulate slow query using PostgreSQL
    jdbcTemplate.execute("SELECT pg_sleep(5)");
    
    // Application should have appropriate timeouts
    // Connection pool should handle slow queries
}
```

## 🎯 Game Day Exercises

### Game Day Scenario Template

```java
@SpringBootTest
@DisplayName("Game Day: Service Degradation")
class GameDayServiceDegradation {
    
    @Autowired
    private MonitoringService monitoringService;
    
    @Test
    @DisplayName("Scenario: Payment Service Degradation")
    void paymentServiceDegradationScenario() {
        // 1. Establish baseline metrics
        Metrics baseline = monitoringService.captureBaseline();
        
        // 2. Inject fault: Degrade payment service
        chaosMonkey.degradeService("payment-service");
        
        // 3. Observe system behavior
        Metrics duringChaos = monitoringService.captureMetrics();
        
        // 4. Verify graceful degradation
        assertThat(duringChaos.getErrorRate()).isLessThan(0.05); // < 5% errors
        assertThat(duringChaos.getResponseTime()).isLessThan(5000); // < 5s
        
        // 5. Restore service
        chaosMonkey.restoreService("payment-service");
        
        // 6. Verify recovery
        Metrics afterRecovery = monitoringService.captureMetrics();
        assertThat(afterRecovery.getErrorRate()).isEqualTo(baseline.getErrorRate());
    }
}
```

## ⚡ Production Chaos Testing

### Gradual Rollout with Chaos

```java
@Component
public class ProductionChaosController {
    
    @Autowired
    private ChaosMonkey chaosMonkey;
    
    @Scheduled(cron = "0 0 2 * * ?") // 2 AM daily
    public void runNightlyChaos() {
        if (isProduction() && isSafeWindow()) {
            // Run chaos on small percentage of traffic
            chaosMonkey.enableForPercentage(1); // 1% of requests
            chaosMonkey.injectLatency(1000, 3000);
            
            // Monitor for 30 minutes
            await().atMost(30, TimeUnit.MINUTES)
                .until(() -> metrics.areHealthy());
            
            // Disable chaos
            chaosMonkey.disable();
        }
    }
}
```

## 🎯 Chaos Testing Best Practices

### DO ✅

- Start with small, controlled experiments
- Monitor everything during chaos experiments
- Have rollback plans ready
- Document findings and improvements
- Run chaos in off-peak hours initially
- Test in staging before production
- Involve SRE and operations teams
- Use feature flags for quick rollbacks

### DON'T ❌

- Run chaos experiments without monitoring
- Test critical systems during peak hours
- Start with destructive experiments
- Ignore alarms during chaos tests
- Run chaos without team awareness
- Test without clear hypotheses
- Skip post-incident reviews
- Assume systems will always recover

## ⚠️ Common Pitfalls

### 1. Testing Without Monitoring

```java
// Bad: Chaos without monitoring
chaosMonkey.killService();
// No monitoring - can't observe impact

// Good: Chaos with monitoring
chaosMonkey.killService();
monitoringService.captureMetrics();
monitoringService.verifyHealth();
```

### 2. Destructive Testing in Production

```java
// Bad: Destructive chaos in production
chaosMonkey.killDatabase(); // Don't do this!

// Good: Non-destructive chaos in production
chaosMonkey.injectLatency(100, 500); // Safe latency injection
chaosMonkey.degradeServiceGradually(); // Gradual degradation
```

### 3. No Rollback Plan

```java
// Bad: Chaos without rollback
chaosMonkey.enable();
// What if something goes wrong?

// Good: Chaos with rollback
try {
    chaosMonkey.enable();
    await().atMost(5, TimeUnit.MINUTES)
        .until(() -> metrics.areHealthy());
} catch (Exception e) {
    chaosMonkey.disable(); // Immediate rollback
    alertTeam();
}
```

## 🎓 Interview Discussion Points

When discussing chaos engineering in interviews:

1. **Philosophy**: Explain why chaos engineering is important for resilience
2. **Experience**: Share real examples of issues found through chaos testing
3. **Tools**: Compare Chaos Monkey, Gremlin, and other chaos tools
4. **Safety**: Describe how you ensure safe chaos experiments
5. **Culture**: Discuss how to build a chaos engineering culture
6. **Metrics**: Explain which metrics to monitor during chaos experiments

## 📚 Related Topics

- [Load Testing Strategies](./load-testing-strategies.md)
- [Integration Testing Patterns](./integration-testing-patterns.md)
- [Contract Testing](./contract-testing.md)

---

**Key Takeaway**: Chaos engineering is about building confidence in system resilience through controlled experimentation. Start small, monitor everything, and always have a rollback plan. The goal is to find weaknesses before they cause real outages.

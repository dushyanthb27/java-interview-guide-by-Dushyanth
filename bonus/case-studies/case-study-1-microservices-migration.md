# Case Study 1: Monolith to Microservices Migration

## Difficulty: Hard 🔴

---

## Scenario Context

### Company Background
**EcoMart** is a mid-sized e-commerce platform with 2 million monthly active users. The platform handles product catalog, shopping cart, checkout, order management, inventory, and user authentication.

### Technical Stack
- **Monolith**: Single Spring Boot application (3.2M LOC)
- **Database**: PostgreSQL (500GB, 200+ tables)
- **Cache**: Redis cluster
- **Message Queue**: RabbitMQ
- **Deployment**: On-premise Kubernetes cluster
- **Team Size**: 25 developers across 5 teams

### Business Context
The company was experiencing:
- **Deployment Pain**: Full deployments took 45 minutes, any bug required rolling back the entire system
- **Team Bottlenecks**: Changes to one module required coordination across all teams
- **Scaling Issues**: Couldn't scale individual components (checkout needed more resources than catalog)
- **Technology Debt**: Some teams wanted to adopt new technologies but were blocked by the monolith

### Constraints
- **Zero Downtime**: Platform must remain available 24/7
- **Data Consistency**: Cannot lose orders or have inventory mismatches
- **Budget**: Limited budget for new infrastructure
- **Timeline**: 18-month migration window
- **Team Skill**: Mixed experience with microservices

---

## Problem Statement

**Primary Challenge**: Migrate a 3.2M LOC monolithic e-commerce platform to microservices architecture without disrupting business operations.

**Specific Problems**:
1. How to extract services without breaking existing functionality?
2. How to handle data consistency across services?
3. How to manage the transition period where both architectures coexist?
4. How to prevent the migration from becoming a "big bang" failure?
5. How to maintain team productivity during the migration?

**Success Criteria**:
- Individual service deployments under 5 minutes
- Ability to scale services independently
- Zero data loss during migration
- Team autonomy in deployment decisions
- Maintain or improve system performance

---

## Analysis Phase

### Current Architecture Assessment

```
┌─────────────────────────────────────────────────────────────┐
│                     Monolith Application                     │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Catalog    │  │    Cart      │  │   Checkout   │      │
│  │   Module     │  │   Module     │  │   Module     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Orders     │  │  Inventory   │  │     User     │      │
│  │   Module     │  │   Module     │  │   Module     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
├─────────────────────────────────────────────────────────────┤
│              Shared Business Logic Layer                     │
├─────────────────────────────────────────────────────────────┤
│              Data Access Layer (JPA/Hibernate)              │
├─────────────────────────────────────────────────────────────┤
│                    PostgreSQL Database                       │
│         (200+ tables, complex relationships)                │
└─────────────────────────────────────────────────────────────┘
```

### Dependency Analysis

**Module Coupling Matrix**:

| Module | Catalog | Cart | Checkout | Orders | Inventory | User |
|--------|---------|------|----------|--------|-----------|------|
| Catalog | - | High | Medium | Low | High | Low |
| Cart | High | - | Very High | Medium | High | Medium |
| Checkout | Medium | Very High | - | Very High | Very High | High |
| Orders | Low | Medium | Very High | - | Very High | High |
| Inventory | High | High | Very High | Very High | - | Low |
| User | Low | Medium | High | High | Low | - |

**Key Findings**:
- Cart and Checkout are tightly coupled (shared transaction boundaries)
- Inventory is coupled with almost everything (real-time stock checks)
- User service is relatively independent
- Catalog has read-heavy patterns, good candidate for extraction

### Risk Assessment

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Data inconsistency during migration | Critical | High | Dual-write pattern with reconciliation |
| Performance degradation | High | Medium | Load testing, feature flags |
| Team productivity drop | Medium | High | Incremental migration, clear ownership |
| Increased operational complexity | High | High | Invest in observability early |
| Migration timeline overrun | Medium | Medium | MVP approach, prioritize high-value services |

---

## Solution Approach

### Strategy: Strangler Fig Pattern

Instead of a big-bang rewrite, we adopted the Strangler Fig pattern:
1. Identify a service boundary
2. Create new microservice alongside the monolith
3. Route traffic incrementally using feature flags
4. Migrate data gradually
5. Decommission monolith module when migration is complete

### Service Extraction Order

Based on dependency analysis and business value:

1. **User Service** (Month 1-3) - Lowest coupling, foundational
2. **Catalog Service** (Month 4-6) - Read-heavy, independent
3. **Inventory Service** (Month 7-9) - Critical for checkout
4. **Cart Service** (Month 10-12) - Complex state management
5. **Order Service** (Month 13-15) - Transaction-heavy
6. **Checkout Service** (Month 16-18) - Orchestrator, extracted last

### Architecture Decisions

**Communication Pattern**: Event-driven architecture with synchronous fallbacks
```java
// Event-driven communication for eventual consistency
@EventListener
public void handleOrderCreated(OrderCreatedEvent event) {
    inventoryService.reserveItems(event.getItems());
    notificationService.sendConfirmation(event.getUserId());
}

// Synchronous fallback for critical paths
@Service
public class CheckoutService {
    @CircuitBreaker(fallbackMethod = "getFallbackInventory")
    public InventoryStatus getInventoryStatus(String productId) {
        return inventoryServiceClient.checkStock(productId);
    }
}
```

**Data Strategy**: Database per service with shared database during transition
```java
// Dual-write pattern during migration
@Service
@Transactional
public class OrderService {
    
    @Autowired
    private OrderRepository monolithRepo;
    
    @Autowired
    private OrderMicroserviceRepository microRepo;
    
    public void createOrder(Order order) {
        // Write to both databases
        monolithRepo.save(order);
        try {
            microRepo.save(order);
        } catch (Exception e) {
            // Log for reconciliation
            reconciliationService.logFailure(order.getId(), e);
        }
    }
}
```

---

## Implementation Details

### Phase 1: User Service Extraction (Months 1-3)

#### Step 1: Define Service Boundary

```java
// User Service API
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody CreateUserRequest request) {
        User user = userService.create(request);
        return ResponseEntity.created(URI.create("/api/users/" + user.getId())).body(user);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(@PathVariable Long id, 
                                          @Valid @RequestBody UpdateUserRequest request) {
        return ResponseEntity.ok(userService.update(id, request));
    }
    
    @PostMapping("/{id}/deactivate")
    public ResponseEntity<Void> deactivateUser(@PathVariable Long id) {
        userService.deactivate(id);
        return ResponseEntity.noContent().build();
    }
}
```

#### Step 2: Implement Data Synchronization

```java
@Service
public class UserMigrationService {
    
    @Autowired
    private MonolithUserRepository monolithRepo;
    
    @Autowired
    private MicroserviceUserRepository microRepo;
    
    @Autowired
    private UserEventPublisher eventPublisher;
    
    @Scheduled(fixedDelay = 1000)
    public void syncUsers() {
        List<User> users = monolithRepo.findModifiedSince(lastSyncTime);
        
        for (User user : users) {
            try {
                microRepo.upsert(user);
                eventPublisher.publishUserSynced(user.getId());
            } catch (Exception e) {
                log.error("Failed to sync user {}", user.getId(), e);
                deadLetterQueue.add(user);
            }
        }
        
        lastSyncTime = Instant.now();
    }
    
    public void reconcile() {
        // Compare counts and identify discrepancies
        long monolithCount = monolithRepo.count();
        long microCount = microRepo.count();
        
        if (monolithCount != microCount) {
            log.warn("Count mismatch: monolith={}, micro={}", monolithCount, microCount);
            alertService.notifyMismatch("users", monolithCount, microCount);
        }
    }
}
```

#### Step 3: Implement Feature Flag Routing

```java
@Component
public class UserServiceRouter {
    
    @Autowired
    private FeatureFlagService featureFlagService;
    
    @Autowired
    private MonolithUserService monolithService;
    
    @Autowired
    private MicroserviceUserService microService;
    
    public User getUser(Long id) {
        if (featureFlagService.isEnabled("user-service-migration", id)) {
            return microService.findById(id);
        }
        return monolithService.findById(id);
    }
    
    public User createUser(CreateUserRequest request) {
        // Write to both during migration
        User monolithUser = monolithService.create(request);
        
        if (featureFlagService.isEnabled("user-service-write-migration")) {
            try {
                microService.create(request);
            } catch (Exception e) {
                log.error("Microservice write failed, will retry", e);
            }
        }
        
        return monolithUser;
    }
}
```

#### Step 4: Gradual Traffic Shift

```java
@Configuration
public class UserMigrationConfig {
    
    @Bean
    public FeatureFlagDefinition userServiceMigration() {
        return FeatureFlagDefinition.builder()
            .key("user-service-migration")
            .type(FeatureFlagType.ROLLOUT)
            .rolloutStrategy(RolloutStrategy.builder()
                .initialPercentage(0)
                .targetPercentage(100)
                .rolloutPeriod(Duration.ofDays(14))
                .segments(List.of(
                    Segment.builder()
                        .criteria("user.id % 100 < 5")  // Start with 5%
                        .build()
                ))
                .build())
            .build();
    }
}
```

### Phase 2: Catalog Service Extraction (Months 4-6)

#### Caching Strategy for Read-Heavy Service

```java
@Service
public class CatalogService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public Product getProduct(Long id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }
    
    @CacheEvict(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        Product updated = productRepository.save(product);
        
        // Publish cache invalidation event
        eventPublisher.publishProductUpdated(updated);
        
        return updated;
    }
    
    @Cacheable(value = "product-search", key = "#query.hashCode()")
    public List<Product> searchProducts(SearchQuery query) {
        return productRepository.search(query);
    }
}
```

#### Event-Driven Cache Invalidation

```java
@Component
public class CacheInvalidationListener {
    
    @Autowired
    private CacheManager cacheManager;
    
    @KafkaListener(topics = "product-updates")
    public void handleProductUpdate(ProductUpdatedEvent event) {
        Cache productsCache = cacheManager.getCache("products");
        if (productsCache != null) {
            productsCache.evict(event.getProductId());
        }
        
        // Invalidate search cache as well
        Cache searchCache = cacheManager.getCache("product-search");
        if (searchCache != null) {
            searchCache.clear();
        }
    }
}
```

### Phase 3: Inventory Service with Distributed Transactions (Months 7-9)

#### Saga Pattern Implementation

```java
@Service
public class OrderSagaOrchestrator {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private SagaLogRepository sagaLogRepository;
    
    public void executeOrderSaga(Order order) {
        String sagaId = UUID.randomUUID().toString();
        
        try {
            // Step 1: Reserve inventory
            sagaLogRepository.save(new SagaLog(sagaId, "INVENTORY_RESERVE_STARTED"));
            inventoryService.reserveItems(order.getItems(), sagaId);
            sagaLogRepository.save(new SagaLog(sagaId, "INVENTORY_RESERVE_COMPLETED"));
            
            // Step 2: Process payment
            sagaLogRepository.save(new SagaLog(sagaId, "PAYMENT_STARTED"));
            PaymentResult paymentResult = paymentService.processPayment(
                order.getPaymentDetails(), order.getTotal(), sagaId);
            
            if (paymentResult.isSuccess()) {
                sagaLogRepository.save(new SagaLog(sagaId, "PAYMENT_COMPLETED"));
                
                // Step 3: Confirm inventory
                inventoryService.confirmReservation(sagaId);
                sagaLogRepository.save(new SagaLog(sagaId, "INVENTORY_CONFIRMED"));
                
                // Step 4: Complete order
                order.setStatus(OrderStatus.COMPLETED);
                orderRepository.save(order);
                sagaLogRepository.save(new SagaLog(sagaId, "ORDER_COMPLETED"));
            } else {
                throw new PaymentFailedException(paymentResult.getErrorMessage());
            }
            
        } catch (Exception e) {
            // Compensating transactions
            log.error("Saga failed for {}", sagaId, e);
            compensateOrderSaga(sagaId, order);
        }
    }
    
    private void compensateOrderSaga(String sagaId, Order order) {
        // Compensate inventory
        try {
            inventoryService.cancelReservation(sagaId);
            sagaLogRepository.save(new SagaLog(sagaId, "INVENTORY_COMPENSATED"));
        } catch (Exception e) {
            log.error("Failed to compensate inventory for saga {}", sagaId, e);
        }
        
        // Compensate payment if already processed
        SagaLog paymentLog = sagaLogRepository.findBySagaIdAndStep(
            sagaId, "PAYMENT_COMPLETED");
        if (paymentLog != null) {
            try {
                paymentService.refundPayment(order.getPaymentDetails(), sagaId);
                sagaLogRepository.save(new SagaLog(sagaId, "PAYMENT_REFUNDED"));
            } catch (Exception e) {
                log.error("Failed to refund payment for saga {}", sagaId, e);
            }
        }
        
        order.setStatus(OrderStatus.FAILED);
        orderRepository.save(order);
        sagaLogRepository.save(new SagaLog(sagaId, "ORDER_FAILED"));
    }
}
```

#### Inventory Service Implementation

```java
@Service
public class InventoryService {
    
    @Autowired
    private InventoryRepository inventoryRepository;
    
    @Autowired
    private ReservationRepository reservationRepository;
    
    @Transactional
    public void reserveItems(List<OrderItem> items, String sagaId) {
        for (OrderItem item : items) {
            Inventory inventory = inventoryRepository.findByProductId(item.getProductId())
                .orElseThrow(() -> new InventoryNotFoundException(item.getProductId()));
            
            if (inventory.getAvailableQuantity() < item.getQuantity()) {
                throw new InsufficientInventoryException(item.getProductId(), 
                    inventory.getAvailableQuantity(), item.getQuantity());
            }
            
            // Create reservation
            Reservation reservation = new Reservation(
                sagaId, item.getProductId(), item.getQuantity());
            reservationRepository.save(reservation);
            
            // Decrease available quantity
            inventory.setAvailableQuantity(inventory.getAvailableQuantity() - item.getQuantity());
            inventory.setReservedQuantity(inventory.getReservedQuantity() + item.getQuantity());
            inventoryRepository.save(inventory);
        }
    }
    
    @Transactional
    public void confirmReservation(String sagaId) {
        List<Reservation> reservations = reservationRepository.findBySagaId(sagaId);
        
        for (Reservation reservation : reservations) {
            Inventory inventory = inventoryRepository.findByProductId(reservation.getProductId())
                .orElseThrow();
            
            // Move from reserved to sold
            inventory.setReservedQuantity(inventory.getReservedQuantity() - reservation.getQuantity());
            inventory.setSoldQuantity(inventory.getSoldQuantity() + reservation.getQuantity());
            inventoryRepository.save(inventory);
            
            reservationRepository.delete(reservation);
        }
    }
    
    @Transactional
    public void cancelReservation(String sagaId) {
        List<Reservation> reservations = reservationRepository.findBySagaId(sagaId);
        
        for (Reservation reservation : reservations) {
            Inventory inventory = inventoryRepository.findByProductId(reservation.getProductId())
                .orElseThrow();
            
            // Release reservation back to available
            inventory.setReservedQuantity(inventory.getReservedQuantity() - reservation.getQuantity());
            inventory.setAvailableQuantity(inventory.getAvailableQuantity() + reservation.getQuantity());
            inventoryRepository.save(inventory);
            
            reservationRepository.delete(reservation);
        }
    }
}
```

### Phase 4: API Gateway Implementation

```java
@Configuration
public class ApiGatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // User service routes
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .circuitBreaker(c -> c.setName("user-service"))
                    .retry(retryConfig -> retryConfig
                        .setRetries(3)
                        .setBackoff(Duration.ofMillis(100), Duration.ofSeconds(2), 2, true))
                )
                .uri("lb://user-service"))
            
            // Catalog service routes
            .route("catalog-service", r -> r
                .path("/api/products/**")
                .filters(f -> f
                    .circuitBreaker(c -> c.setName("catalog-service"))
                    .requestRateLimiter(rateLimiterConfig -> rateLimiterConfig
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver()))
                )
                .uri("lb://catalog-service"))
            
            // Inventory service routes
            .route("inventory-service", r -> r
                .path("/api/inventory/**")
                .filters(f -> f.circuitBreaker(c -> c.setName("inventory-service")))
                .uri("lb://inventory-service"))
            
            // Fallback to monolith for unextracted services
            .route("monolith", r -> r
                .path("/api/**")
                .uri("lb://monolith"))
            
            .build();
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(100, 200); // 100 requests per second, burst 200
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> exchange.getRequest().getHeaders()
            .getOrDefault("X-User-Id", List.of("anonymous"))
            .stream()
            .findFirst()
            .map(UUID::nameUUIDFromBytes)
            .orElse(UUID.randomUUID());
    }
}
```

---

## Results & Metrics

### Quantitative Outcomes

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Deployment Time | 45 minutes | 3 minutes (avg) | 93% reduction |
| Deployment Frequency | 2x/week | 20x/week | 10x increase |
| Lead Time for Changes | 5 days | 1 day | 80% reduction |
| System Availability | 99.5% | 99.95% | 0.45% improvement |
| P95 Latency | 850ms | 420ms | 51% reduction |
| Infrastructure Cost | $50K/month | $65K/month | 30% increase (acceptable) |
| Team Deployment Autonomy | 0% | 80% | Significant improvement |

### Qualitative Improvements

1. **Team Autonomy**: Each team could deploy their service independently
2. **Technology Flexibility**: Catalog team adopted GraphQL, User team adopted Kotlin
3. **Faster Iteration**: Checkout team could A/B test new flows without coordinating
4. **Better Scalability**: Scaled inventory service 5x during Black Friday without affecting others
5. **Improved Debugging**: Isolated issues to specific services quickly

### Lessons Learned

#### What Worked Well

1. **Incremental Approach**: Strangler Fig pattern prevented big-bang risks
2. **Feature Flags**: Enabled safe, gradual traffic migration
3. **Dual-Write Pattern**: Ensured data consistency during transition
4. **Investment in Observability**: Distributed tracing essential for debugging
5. **Team Ownership**: Clear service boundaries improved accountability

#### What Didn't Work

1. **Underestimated Data Migration**: Spent 40% more time on data sync than planned
2. **Insufficient Testing**: Integration tests caught issues in production
3. **Premature Service Extraction**: Extracted checkout too early, had to revert
4. **Lack of API Versioning**: Breaking changes caused compatibility issues
5. **Operational Complexity**: Monitoring 15 services vs 1 monolith was challenging

---

## Interview Takeaways

### Key Technical Concepts to Discuss

1. **Strangler Fig Pattern**: Gradual migration without big-bang rewrite
2. **Saga Pattern**: Distributed transactions without 2PC
3. **Dual-Write Pattern**: Data consistency during migration
4. **Feature Flags**: Safe deployment and traffic routing
5. **API Gateway**: Single entry point, routing, and cross-cutting concerns
6. **Event-Driven Architecture**: Loose coupling and eventual consistency
7. **Circuit Breaker Pattern**: Preventing cascading failures
8. **Database per Service**: Data ownership and independence

### What to Emphasize in Interviews

1. **Trade-off Analysis**: Why microservices over monolith (and vice versa)
2. **Incremental Delivery**: Delivering value while reducing risk
3. **Data Consistency**: How you handle distributed transactions
4. **Operational Readiness**: Monitoring, logging, tracing
5. **Team Structure**: Conway's Law and team alignment with architecture
6. **Migration Strategy**: How to approach large-scale refactoring

### Sample Interview Answer Structure

"When we migrated our e-commerce platform from monolith to microservices, we used the Strangler Fig pattern. We started with the User Service because it had the fewest dependencies. We implemented dual-write to ensure data consistency during the transition, using feature flags to gradually shift traffic. The biggest challenge was handling distributed transactions for the checkout flow, which we solved using the Saga pattern. This approach allowed us to maintain 99.95% availability during the 18-month migration while giving teams deployment autonomy."

---

## Follow-up Questions

### Technical Questions

1. **How did you handle database transactions across services?**
   - Discuss Saga pattern, compensating transactions, eventual consistency

2. **What happened when a service was down during migration?**
   - Circuit breakers, fallbacks, graceful degradation

3. **How did you ensure data consistency during dual-write?**
   - Reconciliation jobs, idempotent operations, dead letter queues

4. **How did you handle API versioning during migration?**
   - Versioning strategy, backward compatibility, deprecation timelines

5. **What monitoring and observability did you implement?**
   - Distributed tracing, metrics, logging, alerting

### Architectural Questions

6. **How did you decide service boundaries?**
   - Domain-driven design, bounded contexts, data ownership

7. **When would you NOT use microservices?**
   - Small teams, simple domains, early-stage startups

8. **How did you handle shared libraries?**
   - Library versioning, dependency management, breaking changes

9. **What about shared databases?**
   - Anti-pattern, but necessary during transition, how to eliminate

### Leadership Questions

10. **How did you keep teams productive during migration?**
    - Clear ownership, incremental goals, training

11. **How did you communicate progress to stakeholders?**
    - Metrics, demos, regular updates, risk transparency

12. **What would you do differently next time?**
    - More testing, better API versioning, invest in observability earlier

---

## Related Topics

- [System Design: Distributed Systems](../../sections/14-system-design/README.md)
- [Multithreading & Concurrency](../../sections/04-multithreading-concurrency/README.md)
- [Performance Optimization](../../sections/12-performance-optimization/README.md)
- [Behavioral: Architectural Decision-Making](../../sections/15-behavioral-scenarios/q06-architectural-decisions.md)

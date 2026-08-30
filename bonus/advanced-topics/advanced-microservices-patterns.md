# Advanced Microservices Patterns Deep-Dive

## 📋 Question
Explain advanced microservices patterns including Saga pattern for distributed transactions, CQRS (Command Query Responsibility Segregation), Event Sourcing, and service mesh patterns. How do you handle data consistency across services? Provide real-world implementation examples.

## Difficulty: 🔴 Expert

---

## 💡 Answer

Microservices architecture introduces complex challenges around data consistency, communication patterns, and system resilience. Advanced patterns like Saga, CQRS, and Event Sourcing address these challenges at scale.

### Saga Pattern for Distributed Transactions

The Saga pattern breaks a distributed transaction into a sequence of local transactions, each updating data within a single service. If one step fails, compensating transactions undo the changes from previous steps.

**Choreography-based Saga:**

```java
// Service A: Order Service
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. Create order (local transaction)
        Order order = new Order(request.getUserId(), request.getItems());
        order.setStatus(OrderStatus.PENDING);
        order = orderRepository.save(order);
        
        // 2. Publish event
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getUserId(),
            order.getTotal()
        );
        kafkaTemplate.send("order-created", event);
        
        return order;
    }
    
    // Compensating transaction
    @Transactional
    public void cancelOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new RuntimeException("Order not found"));
        
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
        
        // Publish cancellation event
        kafkaTemplate.send("order-cancelled", new OrderCancelledEvent(orderId));
    }
}

// Service B: Payment Service
@Service
public class PaymentService {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @KafkaListener(topics = "order-created")
    @Transactional
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            // Process payment
            Payment payment = new Payment(event.getOrderId(), event.getTotal());
            payment.setStatus(PaymentStatus.COMPLETED);
            paymentRepository.save(payment);
            
            // Publish success event
            kafkaTemplate.send("payment-completed", 
                new PaymentCompletedEvent(event.getOrderId()));
            
        } catch (Exception e) {
            // Compensating transaction
            kafkaTemplate.send("payment-failed", 
                new PaymentFailedEvent(event.getOrderId(), e.getMessage()));
        }
    }
    
    @KafkaListener(topics = "order-cancelled")
    @Transactional
    public void handleOrderCancelled(OrderCancelledEvent event) {
        // Refund payment
        Payment payment = paymentRepository.findByOrderId(event.getOrderId());
        if (payment != null && payment.getStatus() == PaymentStatus.COMPLETED) {
            payment.setStatus(PaymentStatus.REFUNDED);
            paymentRepository.save(payment);
        }
    }
}

// Service C: Inventory Service
@Service
public class InventoryService {
    
    @Autowired
    private InventoryRepository inventoryRepository;
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @KafkaListener(topics = "payment-completed")
    @Transactional
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        try {
            // Reserve inventory
            Order order = orderRepository.findById(event.getOrderId());
            for (OrderItem item : order.getItems()) {
                Inventory inventory = inventoryRepository.findByProductId(
                    item.getProductId());
                inventory.setQuantity(inventory.getQuantity() - item.getQuantity());
                inventoryRepository.save(inventory);
            }
            
            kafkaTemplate.send("inventory-reserved", 
                new InventoryReservedEvent(event.getOrderId()));
            
        } catch (InsufficientInventoryException e) {
            kafkaTemplate.send("inventory-failed", 
                new InventoryFailedEvent(event.getOrderId(), e.getMessage()));
        }
    }
    
    @KafkaListener(topics = "order-cancelled")
    @Transactional
    public void handleOrderCancelled(OrderCancelledEvent event) {
        // Release inventory
        // ... release reserved items
    }
}
```

**Orchestration-based Saga (with Saga orchestrator):**

```java
// Saga orchestrator using Spring State Machine
@Service
public class OrderSagaOrchestrator {
    
    @Autowired
    private StateMachineFactory<OrderStates, OrderEvents> stateMachineFactory;
    
    @Autowired
    private OrderRepository orderRepository;
    
    public void startSaga(Order order) {
        StateMachine<OrderStates, OrderEvents> sm = stateMachineFactory.getStateMachine(
            order.getId().toString());
        
        sm.start();
        
        // Define saga steps
        sm.sendEvent(OrderEvents.CREATE_ORDER);
        sm.sendEvent(OrderEvents.PROCESS_PAYMENT);
        sm.sendEvent(OrderEvents.RESERVE_INVENTORY);
        sm.sendEvent(OrderEvents.COMPLETE_ORDER);
    }
    
    @OnTransition(target = "PAYMENT_FAILED")
    public void handlePaymentFailed() {
        // Trigger compensating transactions
        // Cancel order, refund payment, release inventory
    }
}

enum OrderStates {
    CREATED, PAYMENT_PROCESSING, PAYMENT_COMPLETED, PAYMENT_FAILED,
    INVENTORY_RESERVING, INVENTORY_RESERVED, INVENTORY_FAILED,
    COMPLETED, CANCELLED
}

enum OrderEvents {
    CREATE_ORDER, PROCESS_PAYMENT, PAYMENT_SUCCESS, PAYMENT_FAILED,
    RESERVE_INVENTORY, INVENTORY_SUCCESS, INVENTORY_FAILED,
    COMPLETE_ORDER, CANCEL_ORDER
}
```

### CQRS (Command Query Responsibility Segregation)

CQRS separates read and write operations, using different models for each. This enables independent scaling and optimization.

**CQRS Implementation:**

```java
// === COMMAND SIDE (Write) ===
@Entity
public class OrderCommand {
    @Id
    private Long id;
    private Long userId;
    private String status;
    private List<OrderItem> items;
    
    // Command operations
    public void addItem(OrderItem item) {
        this.items.add(item);
    }
    
    public void complete() {
        this.status = "COMPLETED";
    }
}

@Repository
public interface OrderCommandRepository extends JpaRepository<OrderCommand, Long> {
}

@Service
public class OrderCommandService {
    
    @Autowired
    private OrderCommandRepository commandRepository;
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Transactional
    public OrderCommand createOrder(CreateOrderCommand command) {
        OrderCommand order = new OrderCommand();
        order.setUserId(command.getUserId());
        order.setStatus("CREATED");
        order = commandRepository.save(order);
        
        // Publish event for query side
        kafkaTemplate.send("order-events", new OrderCreatedEvent(order));
        
        return order;
    }
}

// === QUERY SIDE (Read) ===
// Denormalized read model optimized for queries
@Entity
public class OrderQuery {
    @Id
    private Long id;
    private Long userId;
    private String userName;  // Denormalized
    private String status;
    private Double total;
    private String itemsSummary;  // Pre-computed
    private LocalDateTime createdAt;
    
    // Read-only
}

@Repository
public interface OrderQueryRepository extends JpaRepository<OrderQuery, Long> {
    
    // Optimized for common queries
    List<OrderQuery> findByUserIdAndStatus(Long userId, String status);
    
    @Query("SELECT o FROM OrderQuery o WHERE o.userId = :userId " +
           "AND o.createdAt BETWEEN :start AND :end")
    List<OrderQuery> findByUserAndDateRange(
        @Param("userId") Long userId,
        @Param("start") LocalDateTime start,
        @Param("end") LocalDateTime end
    );
}

@Service
public class OrderQueryService {
    
    @Autowired
    private OrderQueryRepository queryRepository;
    
    // Read operations - no transactions needed
    public OrderQuery getOrderById(Long id) {
        return queryRepository.findById(id).orElse(null);
    }
    
    public List<OrderQuery> getUserOrders(Long userId) {
        return queryRepository.findByUserIdAndStatus(userId, "COMPLETED");
    }
}

// === EVENT HANDLER (Sync Command to Query) ===
@Service
public class OrderEventHandler {
    
    @Autowired
    private OrderQueryRepository queryRepository;
    
    @Autowired
    private UserRepository userRepository;
    
    @KafkaListener(topics = "order-events")
    @Transactional
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Update read model
        OrderQuery query = new OrderQuery();
        query.setId(event.getOrderId());
        query.setUserId(event.getUserId());
        
        // Denormalize user data
        User user = userRepository.findById(event.getUserId());
        query.setUserName(user.getName());
        
        query.setStatus("CREATED");
        query.setCreatedAt(LocalDateTime.now());
        
        queryRepository.save(query);
    }
    
    @KafkaListener(topics = "order-events")
    @Transactional
    public void handleOrderCompleted(OrderCompletedEvent event) {
        OrderQuery query = queryRepository.findById(event.getOrderId());
        query.setStatus("COMPLETED");
        query.setTotal(event.getTotal());
        queryRepository.save(query);
    }
}
```

### Event Sourcing

Event Sourcing stores all state changes as a sequence of events, rather than storing current state. The current state is derived by replaying events.

**Event Sourcing Implementation:**

```java
// === EVENT STORE ===
@Entity
public class Event {
    @Id
    private Long id;
    
    private String aggregateId;
    private String aggregateType;
    private String eventType;
    private String eventData;  // JSON
    private Long version;
    private LocalDateTime createdAt;
    
    @Version
    private Long optimisticLock;
}

@Repository
public interface EventRepository extends JpaRepository<Event, Long> {
    List<Event> findByAggregateIdOrderByVersion(String aggregateId);
}

// === AGGREGATE ===
public class OrderAggregate {
    private String id;
    private OrderState state;
    private List<OrderItem> items;
    private Long version;
    
    public List<Event> handleCommand(OrderCommand command) {
        List<Event> events = new ArrayList<>();
        
        if (command instanceof CreateOrderCommand) {
            events.add(new OrderCreatedEvent(
                command.getOrderId(),
                command.getUserId()
            ));
        } else if (command instanceof AddItemCommand) {
            events.add(new ItemAddedEvent(
                command.getOrderId(),
                command.getProductId(),
                command.getQuantity()
            ));
        }
        
        return events;
    }
    
    public void applyEvent(Event event) {
        if (event instanceof OrderCreatedEvent) {
            this.id = event.getAggregateId();
            this.state = OrderState.CREATED;
        } else if (event instanceof ItemAddedEvent) {
            this.items.add(new OrderItem(event.getProductId(), event.getQuantity()));
        }
        this.version = event.getVersion();
    }
    
    public static OrderAggregate rebuildFromEvents(List<Event> events) {
        OrderAggregate aggregate = new OrderAggregate();
        for (Event event : events) {
            aggregate.applyEvent(event);
        }
        return aggregate;
    }
}

// === EVENT STORE SERVICE ===
@Service
public class EventStoreService {
    
    @Autowired
    private EventRepository eventRepository;
    
    @Transactional
    public void saveEvents(String aggregateId, List<Event> events) {
        for (Event event : events) {
            event.setAggregateId(aggregateId);
            event.setCreatedAt(LocalDateTime.now());
            eventRepository.save(event);
        }
    }
    
    public List<Event> getEvents(String aggregateId) {
        return eventRepository.findByAggregateIdOrderByVersion(aggregateId);
    }
    
    public OrderAggregate getAggregate(String aggregateId) {
        List<Event> events = getEvents(aggregateId);
        return OrderAggregate.rebuildFromEvents(events);
    }
}

// === SNAPSHOTING (Optimization) ===
@Entity
public class OrderSnapshot {
    @Id
    private String aggregateId;
    private String stateData;  // Serialized aggregate state
    private Long version;
    private LocalDateTime createdAt;
}

@Repository
public interface OrderSnapshotRepository extends JpaRepository<OrderSnapshot, String> {
}

@Service
public class SnapshotService {
    
    @Autowired
    private OrderSnapshotRepository snapshotRepository;
    
    @Autowired
    private EventRepository eventRepository;
    
    public OrderAggregate getAggregateWithSnapshot(String aggregateId) {
        // Try to load from snapshot
        OrderSnapshot snapshot = snapshotRepository.findById(aggregateId).orElse(null);
        
        if (snapshot != null) {
            // Rebuild from snapshot + events since snapshot
            OrderAggregate aggregate = deserialize(snapshot.getStateData());
            List<Event> events = eventRepository.findByAggregateIdAndVersionAfter(
                aggregateId, snapshot.getVersion());
            
            for (Event event : events) {
                aggregate.applyEvent(event);
            }
            
            return aggregate;
        } else {
            // Rebuild from all events
            List<Event> events = eventRepository.findByAggregateIdOrderByVersion(aggregateId);
            return OrderAggregate.rebuildFromEvents(events);
        }
    }
    
    @Scheduled(fixedRate = 3600000)  // Every hour
    public void createSnapshots() {
        // Create snapshots for aggregates with many events
    }
}
```

### Service Mesh Patterns

Service mesh provides infrastructure-level features like service discovery, load balancing, encryption, and observability.

**Istio Configuration Example:**

```yaml
# VirtualService for routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: order-service
        subset: v2  # Canary version
      weight: 10
  - route:
    - destination:
        host: order-service
        subset: v1  # Stable version
      weight: 90

# DestinationRule for load balancing
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN  # Least connection load balancing
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

# Circuit breaker configuration
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 10
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

### Code Example: Complete Microservices Patterns Demo

```java
package com.interview.advanced.microservices;

import org.springframework.boot.*;
import org.springframework.boot.autoconfigure.*;
import org.springframework.context.annotation.*;
import org.springframework.stereotype.*;
import org.springframework.transaction.annotation.*;
import org.springframework.kafka.annotation.*;
import org.springframework.kafka.core.*;
import org.springframework.data.jpa.repository.*;
import org.springframework.beans.factory.annotation.*;
import java.util.*;

// === SAGA CHOREOGRAPHY ===
@Service
@Transactional
public class OrderSagaService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    public Order createOrder(OrderRequest request) {
        // Step 1: Create order
        Order order = new Order(request.getUserId(), request.getItems());
        order.setStatus(OrderStatus.PENDING);
        order = orderRepository.save(order);
        
        // Step 2: Publish event
        kafkaTemplate.send("order-created", new OrderCreatedEvent(
            order.getId(), order.getUserId(), order.getTotal()
        ));
        
        return order;
    }
    
    @KafkaListener(topics = "payment-failed")
    public void handlePaymentFailed(PaymentFailedEvent event) {
        // Compensating transaction
        Order order = orderRepository.findById(event.getOrderId());
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
        
        kafkaTemplate.send("order-cancelled", new OrderCancelledEvent(order.getId()));
    }
    
    @KafkaListener(topics = "inventory-failed")
    public void handleInventoryFailed(InventoryFailedEvent event) {
        // Compensating transaction
        Order order = orderRepository.findById(event.getOrderId());
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
        
        kafkaTemplate.send("order-cancelled", new OrderCancelledEvent(order.getId()));
    }
}

// === CQRS IMPLEMENTATION ===
@Service
public class OrderCQRSService {
    
    @Autowired
    private OrderCommandRepository commandRepository;
    
    @Autowired
    private OrderQueryRepository queryRepository;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    // Command side
    @Transactional
    public OrderCommand createOrderCommand(CreateOrderCommand command) {
        OrderCommand order = new OrderCommand();
        order.setUserId(command.getUserId());
        order.setStatus("CREATED");
        order = commandRepository.save(order);
        
        // Sync to query side via event
        kafkaTemplate.send("order-sync", new OrderSyncEvent(order));
        
        return order;
    }
    
    // Query side - no transaction
    public OrderQuery getOrderQuery(Long id) {
        return queryRepository.findById(id).orElse(null);
    }
    
    public List<OrderQuery> getUserOrders(Long userId) {
        return queryRepository.findByUserId(userId);
    }
}

// === EVENT SOURCING ===
@Service
public class EventSourcingService {
    
    @Autowired
    private EventRepository eventRepository;
    
    @Transactional
    public void saveEvents(String aggregateId, List<DomainEvent> events) {
        for (DomainEvent event : events) {
            Event storedEvent = new Event();
            storedEvent.setAggregateId(aggregateId);
            storedEvent.setEventType(event.getClass().getSimpleName());
            storedEvent.setEventData(serialize(event));
            storedEvent.setVersion(getNextVersion(aggregateId));
            eventRepository.save(storedEvent);
        }
    }
    
    public List<DomainEvent> getEvents(String aggregateId) {
        return eventRepository.findByAggregateIdOrderByVersion(aggregateId)
            .stream()
            .map(this::deserialize)
            .collect(Collectors.toList());
    }
    
    public OrderAggregate rebuildAggregate(String aggregateId) {
        List<DomainEvent> events = getEvents(aggregateId);
        OrderAggregate aggregate = new OrderAggregate();
        for (DomainEvent event : events) {
            aggregate.apply(event);
        }
        return aggregate;
    }
}

// === RESILIENCE PATTERNS ===
@Service
public class ResilientService {
    
    @Autowired
    private PaymentServiceClient paymentClient;
    
    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentClient.process(request);
    }
    
    public PaymentResult fallbackPayment(PaymentRequest request, Exception e) {
        return PaymentResult.failed("Payment service unavailable");
    }
    
    @Retry(name = "paymentService", maxAttempts = 3, backoff = @Backoff(delay = 1000))
    public PaymentResult processPaymentWithRetry(PaymentRequest request) {
        return paymentClient.process(request);
    }
    
    @Bulkhead(name = "paymentService", value = 10)
    public PaymentResult processPaymentWithBulkhead(PaymentRequest request) {
        return paymentClient.process(request);
    }
}

// === CONFIGURATION ===
@Configuration
public class KafkaConfig {
    
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### ⚠️ Common Mistakes

**1. Not handling saga failures properly**

```java
// ❌ No compensating transaction
public void handlePaymentFailed(PaymentFailedEvent event) {
    // Order remains in PENDING state
}

// ✅ Implement compensating transaction
public void handlePaymentFailed(PaymentFailedEvent event) {
    Order order = orderRepository.findById(event.getOrderId());
    order.setStatus(OrderStatus.CANCELLED);
    orderRepository.save(order);
    kafkaTemplate.send("order-cancelled", new OrderCancelledEvent(order.getId()));
}
```

**2. Eventual consistency not handled in UI**

```java
// ❌ Assuming immediate consistency
Order order = orderService.createOrder(request);
return order;  // Status might still be PENDING

// ✅ Handle eventual consistency
Order order = orderService.createOrder(request);
return CompletableFuture.supplyAsync(() -> {
    while (order.getStatus() == OrderStatus.PENDING) {
        Thread.sleep(100);
        order = orderService.getOrder(order.getId());
    }
    return order;
});
```

### ⚡ Performance Tip

**Use event batching** to reduce overhead. Group multiple events into a single batch for processing. Implement idempotent event handlers to safely retry failed events without side effects.

**Snapshotting** in event sourcing is essential for performance. Create snapshots after N events to avoid replaying entire event history.

### 🔧 Microservices Internals

The Saga pattern relies on **event-driven architecture** where services communicate asynchronously via message brokers. Choreography-based sagas have no central coordinator; each service reacts to events. Orchestration-based sagas use a central orchestrator to manage the transaction flow.

CQRS separates read and write models, often using different databases optimized for each workload. Event sourcing stores immutable events, enabling audit trails and temporal queries.

---

## 🎯 Interviewer Insight

Microservices patterns questions test your understanding of distributed systems challenges. Interviewers want to see that you:
- Understand the trade-offs between choreography and orchestration
- Can implement CQRS with event synchronization
- Know when to use event sourcing vs traditional CRUD
- Understand eventual consistency and how to handle it
- Can design resilient services with circuit breakers and retries

Strong candidates can explain the difference between choreography and orchestration and provide real-world examples of saga implementation.

## ⚡ Follow-up Questions

1. What is the difference between choreography and orchestration in the Saga pattern?
2. How do you handle duplicate events in event sourcing?
3. When would you choose CQRS over a traditional architecture?

## 📚 Related Topics

- [System Design](../../sections/14-system-design/README.md)
- [Spring Boot](../../sections/13-framework-integration/README.md)
- [Reactive Programming](../../sections/04-multithreading-concurrency/q25-reactive-basics.md)

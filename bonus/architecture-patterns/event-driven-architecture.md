# Event-Driven Architecture

[Home](../../README.md) > Bonus > Architecture Patterns > Event-Driven Architecture

---

## Overview
Event-Driven Architecture (EDA) is a pattern where components communicate through events. Instead of direct method calls, services publish events when something happens, and other services subscribe to those events they're interested in. This enables loose coupling, scalability, and real-time processing in distributed systems.

## Pattern Description

In event-driven architecture, producers publish events without knowing who will consume them. Consumers subscribe to events they care about and react accordingly. This decoupling allows systems to evolve independently and handle varying loads effectively.

### Architecture Diagram

```
┌─────────────┐    Publish    ┌─────────────────┐    Subscribe    ┌─────────────┐
│  Producer   │ ─────────────▶│  Event Broker   │◀─────────────── │  Consumer   │
│  Service    │   Events      │  (Kafka/RabbitMQ)│     Events      │  Service    │
└─────────────┘                └─────────────────┘                └─────────────┘
       │                                │                                │
       │                                │                                │
┌──────▼──────┐                ┌────────▼────────┐                ┌──────▼──────┐
│  Producer   │                │  Event Broker   │                │  Consumer   │
│  Service    │                │  (Kafka/RabbitMQ)│                │  Service    │
└─────────────┘                └─────────────────┘                └─────────────┘
       │                                │                                │
       │                                │                                │
┌──────▼──────┐                ┌────────▼────────┐                ┌──────▼──────┐
│  Producer   │                │  Event Broker   │                │  Consumer   │
│  Service    │                │  (Kafka/RabbitMQ)│                │  Service    │
└─────────────┘                └─────────────────┘                └─────────────┘

Event Flow:
1. Producer creates an event (state change or notification)
2. Producer publishes event to event broker
3. Event broker routes event to interested consumers
4. Consumers process event asynchronously
5. Consumers may publish new events (chaining)
```

### Event Types

**Domain Events**
- Represent something that happened in the domain
- Named in past tense (OrderCreated, PaymentProcessed)
- Contain all relevant data about the event
- Immutable once published

**Integration Events**
- Used for communication between bounded contexts
- May contain less detail than domain events
- Often versioned for compatibility
- May require transformation

**System Events**
- Technical events (ServiceStarted, ErrorOccurred)
- Used for monitoring and alerting
- Often handled by infrastructure
- May trigger automated responses

### Event Patterns

**Event Notification**
- Simple notification that something happened
- Consumers fetch additional data if needed
- Lightweight and simple
- Example: OrderPlaced event

**Event-Carried State Transfer**
- Event contains all necessary data
- Consumers don't need to call back to producer
- Reduces coupling between services
- Example: OrderCreated with full order details

**Event Sourcing**
- State is stored as sequence of events
- Current state derived by replaying events
- Enables temporal queries
- Complex but powerful

**CQRS with Events**
- Separate read and write models
- Events update read model asynchronously
- Optimized for different workloads
- Common in event-driven systems

## When to Use Event-Driven Architecture

### Ideal Scenarios
✅ High-throughput systems requiring loose coupling
✅ Real-time processing and notifications
✅ Systems with multiple independent services
✅ When scalability and resilience are critical
✅ Asynchronous processing requirements
✅ Complex business processes with multiple steps
✅ Systems requiring audit trails

### When to Avoid
❌ Simple CRUD applications (overkill)
❌ When strong consistency is required
❌ Teams unfamiliar with asynchronous programming
❌ When event ordering is critical
❌ Simple request-response patterns suffice
❌ When debugging complexity is a concern

## Java Implementation

### Project Structure
```
src/
├── main/
│   ├── java/
│   │   └── com/
│   │       └── example/
│   │           └── ecommerce/
│   │               ├── events/
│   │               │   ├── OrderCreatedEvent.java
│   │               │   ├── PaymentProcessedEvent.java
│   │               │   ├── OrderShippedEvent.java
│   │               │   └── InventoryUpdatedEvent.java
│   │               ├── producer/
│   │               │   ├── OrderService.java
│   │               │   └── PaymentService.java
│   │               ├── consumer/
│   │               │   ├── InventoryService.java
│   │               │   ├── NotificationService.java
│   │               │   └── AnalyticsService.java
│   │               ├── config/
│   │               │   ├── KafkaConfig.java
│   │               │   └── EventConfig.java
│   │               └── EcommerceApplication.java
```

### Event Models
```java
// Base Event Interface
package com.example.ecommerce.events;

import java.time.LocalDateTime;
import java.util.UUID;

public interface DomainEvent {
    UUID getEventId();
    String getEventType();
    LocalDateTime getOccurredOn();
    String getAggregateId();
    int getVersion();
}

// Domain Event
package com.example.ecommerce.events;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

public record OrderCreatedEvent(
    UUID eventId,
    String eventType,
    LocalDateTime occurredOn,
    String aggregateId,
    int version,
    UUID orderId,
    UUID customerId,
    List<OrderItem> items,
    BigDecimal totalAmount,
    String shippingAddress,
    String status
) implements DomainEvent {

    public OrderCreatedEvent {
        if (eventId == null) eventId = UUID.randomUUID();
        if (eventType == null) eventType = "OrderCreated";
        if (occurredOn == null) occurredOn = LocalDateTime.now();
        if (aggregateId == null) aggregateId = orderId.toString();
    }

    public static OrderCreatedEvent create(
            UUID orderId,
            UUID customerId,
            List<OrderItem> items,
            BigDecimal totalAmount,
            String shippingAddress) {
        return new OrderCreatedEvent(
            UUID.randomUUID(),
            "OrderCreated",
            LocalDateTime.now(),
            orderId.toString(),
            1,
            orderId,
            customerId,
            items,
            totalAmount,
            shippingAddress,
            "PENDING"
        );
    }

    public record OrderItem(
        UUID productId,
        String productName,
        int quantity,
        BigDecimal unitPrice,
        BigDecimal totalPrice
    ) {}
}

// Domain Event
package com.example.ecommerce.events;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public record PaymentProcessedEvent(
    UUID eventId,
    String eventType,
    LocalDateTime occurredOn,
    String aggregateId,
    int version,
    UUID paymentId,
    UUID orderId,
    BigDecimal amount,
    String paymentMethod,
    String status,
    String transactionId
) implements DomainEvent {

    public PaymentProcessedEvent {
        if (eventId == null) eventId = UUID.randomUUID();
        if (eventType == null) eventType = "PaymentProcessed";
        if (occurredOn == null) occurredOn = LocalDateTime.now();
        if (aggregateId == null) aggregateId = orderId.toString();
    }

    public static PaymentProcessedEvent create(
            UUID paymentId,
            UUID orderId,
            BigDecimal amount,
            String paymentMethod,
            String status,
            String transactionId) {
        return new PaymentProcessedEvent(
            UUID.randomUUID(),
            "PaymentProcessed",
            LocalDateTime.now(),
            orderId.toString(),
            1,
            paymentId,
            orderId,
            amount,
            paymentMethod,
            status,
            transactionId
        );
    }
}

// Domain Event
package com.example.ecommerce.events;

import java.time.LocalDateTime;
import java.util.UUID;

public record OrderShippedEvent(
    UUID eventId,
    String eventType,
    LocalDateTime occurredOn,
    String aggregateId,
    int version,
    UUID orderId,
    String trackingNumber,
    String carrier,
    LocalDateTime estimatedDelivery
) implements DomainEvent {

    public OrderShippedEvent {
        if (eventId == null) eventId = UUID.randomUUID();
        if (eventType == null) eventType = "OrderShipped";
        if (occurredOn == null) occurredOn = LocalDateTime.now();
        if (aggregateId == null) aggregateId = orderId.toString();
    }

    public static OrderShippedEvent create(
            UUID orderId,
            String trackingNumber,
            String carrier,
            LocalDateTime estimatedDelivery) {
        return new OrderShippedEvent(
            UUID.randomUUID(),
            "OrderShipped",
            LocalDateTime.now(),
            orderId.toString(),
            1,
            orderId,
            trackingNumber,
            carrier,
            estimatedDelivery
        );
    }
}
```

### Event Publisher
```java
package com.example.ecommerce.producer;

import com.example.ecommerce.events.DomainEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

@Component
public class EventPublisher {
    
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    public EventPublisher(KafkaTemplate<String, String> kafkaTemplate,
                         ObjectMapper objectMapper) {
        this.kafkaTemplate = kafkaTemplate;
        this.objectMapper = objectMapper;
    }

    public void publish(String topic, DomainEvent event) {
        try {
            String json = objectMapper.writeValueAsString(event);
            String key = event.getAggregateId();
            kafkaTemplate.send(topic, key, json);
            
            System.out.println("Published event: " + event.getEventType() + 
                             " to topic: " + topic);
        } catch (Exception e) {
            throw new RuntimeException("Failed to publish event", e);
        }
    }

    public void publishWithRetry(String topic, DomainEvent event, int maxRetries) {
        int attempts = 0;
        while (attempts < maxRetries) {
            try {
                publish(topic, event);
                return;
            } catch (Exception e) {
                attempts++;
                if (attempts >= maxRetries) {
                    throw new RuntimeException(
                        "Failed to publish event after " + maxRetries + " attempts", e
                    );
                }
                try {
                    Thread.sleep(1000 * attempts); // Exponential backoff
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException("Interrupted during retry", ie);
                }
            }
        }
    }
}
```

### Producer Services
```java
package com.example.ecommerce.producer;

import com.example.ecommerce.events.OrderCreatedEvent;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

@Service
@Transactional
public class OrderService {
    
    private final EventPublisher eventPublisher;

    public OrderService(EventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void createOrder(UUID orderId, UUID customerId, 
                           List<OrderCreatedEvent.OrderItem> items,
                           BigDecimal totalAmount, String shippingAddress) {
        // Business logic to create order
        System.out.println("Creating order: " + orderId);
        
        // Publish domain event
        OrderCreatedEvent event = OrderCreatedEvent.create(
            orderId,
            customerId,
            items,
            totalAmount,
            shippingAddress
        );
        
        eventPublisher.publish("order-events", event);
    }

    public void shipOrder(UUID orderId, String trackingNumber, 
                         String carrier, LocalDateTime estimatedDelivery) {
        // Business logic to ship order
        System.out.println("Shipping order: " + orderId);
        
        // Publish domain event
        OrderShippedEvent event = OrderShippedEvent.create(
            orderId,
            trackingNumber,
            carrier,
            estimatedDelivery
        );
        
        eventPublisher.publish("order-events", event);
    }
}

package com.example.ecommerce.producer;

import com.example.ecommerce.events.PaymentProcessedEvent;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.UUID;

@Service
@Transactional
public class PaymentService {
    
    private final EventPublisher eventPublisher;

    public PaymentService(EventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void processPayment(UUID paymentId, UUID orderId, 
                               BigDecimal amount, String paymentMethod) {
        // Business logic to process payment
        String transactionId = "TXN-" + UUID.randomUUID().toString().substring(0, 8);
        String status = "COMPLETED";
        
        System.out.println("Processing payment: " + paymentId);
        
        // Publish domain event
        PaymentProcessedEvent event = PaymentProcessedEvent.create(
            paymentId,
            orderId,
            amount,
            paymentMethod,
            status,
            transactionId
        );
        
        eventPublisher.publish("payment-events", event);
    }
}
```

### Event Consumers
```java
package com.example.ecommerce.consumer;

import com.example.ecommerce.events.OrderCreatedEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class InventoryService {
    
    private final ObjectMapper objectMapper;

    public InventoryService(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @KafkaListener(topics = "order-events", groupId = "inventory-group")
    public void handleOrderCreated(String message) {
        try {
            OrderCreatedEvent event = objectMapper.readValue(
                message, OrderCreatedEvent.class
            );
            
            System.out.println("Inventory Service: Handling OrderCreated event");
            System.out.println("Order ID: " + event.orderId());
            
            // Reserve inventory for each item
            for (OrderCreatedEvent.OrderItem item : event.items()) {
                reserveInventory(item.productId(), item.quantity());
                System.out.println("Reserved " + item.quantity() + 
                                 " units of product: " + item.productId());
            }
            
        } catch (Exception e) {
            System.err.println("Failed to process OrderCreated event: " + e.getMessage());
            // Implement error handling: dead letter queue, retry, etc.
        }
    }

    private void reserveInventory(UUID productId, int quantity) {
        // Business logic to reserve inventory
        System.out.println("Reserving inventory for product: " + productId);
    }
}

package com.example.ecommerce.consumer;

import com.example.ecommerce.events.OrderCreatedEvent;
import com.example.ecommerce.events.OrderShippedEvent;
import com.example.ecommerce.events.PaymentProcessedEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class NotificationService {
    
    private final ObjectMapper objectMapper;

    public NotificationService(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @KafkaListener(topics = "order-events", groupId = "notification-group")
    public void handleOrderCreated(String message) {
        try {
            OrderCreatedEvent event = objectMapper.readValue(
                message, OrderCreatedEvent.class
            );
            
            System.out.println("Notification Service: Sending order confirmation");
            sendEmail(event.customerId(), 
                     "Order Confirmation",
                     "Your order " + event.orderId() + " has been created");
            
        } catch (Exception e) {
            System.err.println("Failed to send notification: " + e.getMessage());
        }
    }

    @KafkaListener(topics = "payment-events", groupId = "notification-group")
    public void handlePaymentProcessed(String message) {
        try {
            PaymentProcessedEvent event = objectMapper.readValue(
                message, PaymentProcessedEvent.class
            );
            
            if ("COMPLETED".equals(event.status())) {
                System.out.println("Notification Service: Sending payment confirmation");
                sendEmail(event.orderId(), 
                         "Payment Confirmation",
                         "Payment of " + event.amount() + " completed");
            }
            
        } catch (Exception e) {
            System.err.println("Failed to send notification: " + e.getMessage());
        }
    }

    @KafkaListener(topics = "order-events", groupId = "notification-group")
    public void handleOrderShipped(String message) {
        try {
            OrderShippedEvent event = objectMapper.readValue(
                message, OrderShippedEvent.class
            );
            
            System.out.println("Notification Service: Sending shipping notification");
            sendEmail(event.orderId(),
                     "Order Shipped",
                     "Your order has been shipped. Tracking: " + event.trackingNumber());
            
        } catch (Exception e) {
            System.err.println("Failed to send notification: " + e.getMessage());
        }
    }

    private void sendEmail(UUID recipientId, String subject, String body) {
        // Business logic to send email
        System.out.println("Sending email to " + recipientId + 
                         ": " + subject);
    }
}

package com.example.ecommerce.consumer;

import com.example.ecommerce.events.OrderCreatedEvent;
import com.example.ecommerce.events.PaymentProcessedEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class AnalyticsService {
    
    private final ObjectMapper objectMapper;

    public AnalyticsService(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @KafkaListener(topics = "order-events", groupId = "analytics-group")
    public void handleOrderCreated(String message) {
        try {
            OrderCreatedEvent event = objectMapper.readValue(
                message, OrderCreatedEvent.class
            );
            
            System.out.println("Analytics Service: Recording order metrics");
            recordMetric("orders.created", 1);
            recordMetric("orders.total_amount", event.totalAmount().doubleValue());
            
        } catch (Exception e) {
            System.err.println("Failed to record analytics: " + e.getMessage());
        }
    }

    @KafkaListener(topics = "payment-events", groupId = "analytics-group")
    public void handlePaymentProcessed(String message) {
        try {
            PaymentProcessedEvent event = objectMapper.readValue(
                message, PaymentProcessedEvent.class
            );
            
            if ("COMPLETED".equals(event.status())) {
                System.out.println("Analytics Service: Recording payment metrics");
                recordMetric("payments.completed", 1);
                recordMetric("payments.amount", event.amount().doubleValue());
                recordMetric("payments.method." + event.paymentMethod(), 1);
            }
            
        } catch (Exception e) {
            System.err.println("Failed to record analytics: " + e.getMessage());
        }
    }

    private void recordMetric(String name, double value) {
        // Business logic to record metrics
        System.out.println("Recording metric: " + name + " = " + value);
    }
}
```

### Configuration
```java
package com.example.ecommerce.config;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.*;
import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
@EnableKafka
public class KafkaConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
                   org.apache.kafka.common.serialization.StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
                   org.apache.kafka.common.serialization.StringSerializer.class);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "ecommerce-group");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
                   org.apache.kafka.common.serialization.StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
                   org.apache.kafka.common.serialization.StringDeserializer.class);
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> 
            kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(
            org.springframework.kafka.listener.ContainerProperties.AckMode.MANUAL_IMMEDIATE
        );
        return factory;
    }
}
```

## Pros and Cons

### Advantages
✅ **Loose Coupling**: Producers and consumers don't know about each other
✅ **Scalability**: Consumers can scale independently based on load
✅ **Resilience**: Event broker buffers messages during outages
✅ **Flexibility**: Easy to add new consumers without changing producers
✅ **Real-time Processing**: Enables immediate reaction to events
✅ **Asynchronous**: Non-blocking communication improves performance
✅ **Audit Trail**: Event log provides history of all changes

### Disadvantages
❌ **Complexity**: More complex than synchronous communication
❌ **Eventual Consistency**: Data is not immediately consistent
❌ **Debugging**: Harder to trace request flow across services
❌ **Event Ordering**: No guarantee of event order across topics
❌ **Error Handling**: Complex error handling and retry logic
❌ **Learning Curve**: Requires understanding of async patterns
❌ **Infrastructure**: Requires event broker (Kafka, RabbitMQ)

## Real-World Use Cases

### E-commerce Platform
- **Events**: OrderCreated, PaymentProcessed, OrderShipped
- **Consumers**: Inventory, Notification, Analytics, Shipping
- **Benefits**: Real-time inventory updates, customer notifications

### Financial Trading System
- **Events**: TradeExecuted, PositionUpdated, RiskLimitBreached
- **Consumers**: Risk Management, Reporting, Compliance
- **Benefits**: Real-time risk monitoring, automated compliance

### IoT Platform
- **Events**: SensorReading, DeviceConnected, AlertTriggered
- **Consumers**: Analytics, Alerting, Storage
- **Benefits**: Real-time monitoring, scalable data ingestion

## Interview Discussion Points

### Common Questions

**Q: How do you handle duplicate events?**
A: Duplicate events are handled through:
- Idempotent consumers: Same event processed multiple times safely
- Event deduplication: Track processed event IDs
- Unique constraints: Database prevents duplicate state
- Exactly-once semantics: Kafka transactions for critical operations

**Q: How do you ensure event ordering?**
A: Event ordering strategies:
- Partition by key: Events with same key go to same partition
- Single partition per aggregate: Guarantees order per entity
- Sequence numbers: Order events within consumer
- Timestamp ordering: Sort events by timestamp if needed

**Q: How do you handle failed event processing?**
A: Error handling strategies:
- Retry with exponential backoff
- Dead letter queue (DLQ) for failed events
- Circuit breaker to stop cascading failures
- Manual intervention for critical failures
- Monitoring and alerting on error rates

**Q: What's the difference between event notification and event-carried state transfer?**
A: Event notification: Lightweight event with minimal data, consumers fetch additional data if needed. Event-carried state transfer: Event contains all necessary data, consumers don't need to call back. State transfer reduces coupling but increases event size and potential data inconsistency.

**Q: How do you handle schema evolution for events?**
A: Schema evolution strategies:
- Version events: Include version field in event
- Backward compatibility: New fields are optional
- Avro/Protobuf: Schema registry for versioning
- Multiple consumers: Support old and new versions
- Migration strategy: Gradual rollout of new schema

### Advanced Topics

**Event Sourcing**
- Store state as sequence of events
- Rebuild state by replaying events
- Enables temporal queries
- Complex but powerful for certain domains

**CQRS with Events**
- Separate read and write models
- Events update read model asynchronously
- Optimized for different workloads
- Common in event-driven systems

**Saga Pattern**
- Distributed transactions via events
- Compensating events for rollback
- Choreography vs orchestration
- Eventual consistency

## Best Practices

### Event Design
1. **Use past tense**: OrderCreated, not CreateOrder
2. **Include all data**: Event should be self-contained
3. **Immutable**: Events should never change
4. **Version events**: Include version for evolution
5. **Unique IDs**: Every event needs unique identifier

### Producer Design
1. **Publish after commit**: Events published after state change
2. **Transactional publishing**: Ensure event and state are atomic
3. **Error handling**: Handle publish failures gracefully
4. **Idempotent publishing**: Handle duplicate publishes
5. **Event validation**: Validate events before publishing

### Consumer Design
1. **Idempotent processing**: Handle duplicate events safely
2. **Error handling**: Robust error handling and retry
3. **Monitoring**: Track consumer lag and errors
4. **Scaling**: Horizontal scaling for high throughput
5. **Testing**: Test with various event scenarios

## Related Patterns

- **CQRS**: Often combined with event-driven architecture
- **Event Sourcing**: Advanced pattern using events as state
- **Saga Pattern**: Distributed transactions via events
- **Publish-Subscribe**: Fundamental pattern behind EDA

## Technology Choices

### Event Brokers
- **Apache Kafka**: High throughput, durable, scalable
- **RabbitMQ**: Flexible routing, reliable messaging
- **Amazon SQS/SNS**: Managed cloud service
- **Apache Pulsar**: Multi-tenancy, geo-replication

### Java Frameworks
- **Spring Kafka**: Kafka integration with Spring
- **Spring Cloud Stream**: Abstraction over messaging
- **Axon Framework**: CQRS and event sourcing
- **Micronaut**: Lightweight framework with messaging

---

**Next**: Learn about [CQRS Pattern](./cqrs-pattern.md) for separating read and write operations

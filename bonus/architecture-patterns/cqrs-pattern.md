# CQRS Pattern

[Home](../../README.md) > Bonus > Architecture Patterns > CQRS Pattern

---

## Overview
Command Query Responsibility Segregation (CQRS) is a pattern that separates read and write operations into different models. Unlike traditional CRUD where the same model handles both reads and writes, CQRS uses separate models optimized for their specific operations. This pattern is particularly useful for complex domains with different read and write performance requirements.

## Pattern Description

CQRS separates the application into two parts:
- **Command side**: Handles writes (creates, updates, deletes)
- **Query side**: Handles reads (queries, searches, reporting)

This separation allows each side to be optimized independently, use different data stores, and scale according to their specific needs.

### Architecture Diagram

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ├─────────────────┬─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  Command    │   │  Command    │   │  Query      │
│  Handler    │   │  Handler    │   │  Handler    │
└──────┬──────┘   └──────┬──────┘   └──────┬──────┘
       │                 │                 │
       ▼                 ▼                 ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  Write      │   │  Event      │   │  Read       │
│  Model      │   │  Bus        │   │  Model      │
└──────┬──────┘   └──────┬──────┘   └──────┬──────┘
       │                 │                 │
       ▼                 │                 ▼
┌─────────────┐           │         ┌─────────────┐
│  Write      │           │         │  Read       │
│  Database   │           │         │  Database   │
│  (NoSQL/SQL)│           │         │  (SQL/NoSQL)│
└─────────────┘           │         └─────────────┘
                          │
                          ▼
                   ┌─────────────┐
                   │  Event      │
                   │  Store     │
                   │  (Optional)│
                   └─────────────┘

Command Flow:
1. Client sends command (CreateOrder, UpdateProduct)
2. Command handler validates and processes
3. Write model updated in write database
4. Domain event published to event bus
5. Event updates read model asynchronously
6. Query handler reads from read database
```

### Key Concepts

**Commands**
- Represent intent to change state
- Named in imperative mood (CreateOrder, UpdateProduct)
- Return success/failure, not data
- Validated before execution

**Queries**
- Represent request for data
- Named in interrogative mood (GetOrder, SearchProducts)
- Return data, not commands
- Never modify state

**Write Model**
- Optimized for writes
- Normalized for consistency
- Enforces business rules
- Often relational database

**Read Model**
- Optimized for reads
- Denormalized for performance
- Tailored to specific queries
- Often document database or search engine

**Event Bus**
- Optional but common
- Bridges write and read models
- Enables eventual consistency
- Supports multiple read models

## When to Use CQRS

### Ideal Scenarios
✅ Read-heavy vs write-heavy workloads
✅ Complex business logic with different read/write models
✅ When read and write performance requirements differ
✅ Systems requiring complex query capabilities
✅ High-throughput applications needing scalability
✅ Real-time dashboards and reporting
✅ Systems with multiple data access patterns

### When to Avoid
❌ Simple CRUD applications (overkill)
❌ When strong consistency is required
❌ Teams unfamiliar with the pattern
❌ Small applications with simple queries
❌ When development time is constrained
❌ When read and write patterns are similar

## Java Implementation

### Project Structure
```
src/
├── main/
│   ├── java/
│   │   └── com/
│   │       └── example/
│   │           └── ecommerce/
│   │               ├── command/
│   │               │   ├── model/
│   │               │   │   ├── Order.java
│   │               │   │   ├── OrderItem.java
│   │               │   │   └── Customer.java
│   │               │   ├── repository/
│   │               │   │   └── OrderWriteRepository.java
│   │               │   ├── handler/
│   │               │   │   ├── CreateOrderCommandHandler.java
│   │               │   │   ├── UpdateOrderCommandHandler.java
│   │               │   │   └── CancelOrderCommandHandler.java
│   │               │   └── dto/
│   │               │       ├── CreateOrderCommand.java
│   │               │       ├── UpdateOrderCommand.java
│   │               │       └── CancelOrderCommand.java
│   │               ├── query/
│   │               │   ├── model/
│   │               │   │   ├── OrderView.java
│   │               │   │   └── OrderSummaryView.java
│   │               │   ├── repository/
│   │               │   │   └── OrderReadRepository.java
│   │               │   ├── handler/
│   │               │   │   ├── GetOrderQueryHandler.java
│   │               │   │   └── SearchOrdersQueryHandler.java
│   │               │   └── dto/
│   │               │       ├── GetOrderQuery.java
│   │               │       └── SearchOrdersQuery.java
│   │               ├── event/
│   │               │   ├── OrderCreatedEvent.java
│   │               │   ├── OrderUpdatedEvent.java
│   │               │   └── OrderCancelledEvent.java
│   │               ├── projection/
│   │               │   └── OrderProjection.java
│   │               └── controller/
│   │                   ├── OrderCommandController.java
│   │                   └── OrderQueryController.java
```

### Command Side
```java
// Command DTO
package com.example.ecommerce.command.dto;

import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

public record CreateOrderCommand(
    UUID orderId,
    UUID customerId,
    List<OrderItemCommand> items,
    String shippingAddress
) {
    public record OrderItemCommand(
        UUID productId,
        int quantity,
        BigDecimal unitPrice
    ) {}

    public CreateOrderCommand {
        if (orderId == null) {
            throw new IllegalArgumentException("Order ID is required");
        }
        if (customerId == null) {
            throw new IllegalArgumentException("Customer ID is required");
        }
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }
    }
}

// Command DTO
package com.example.ecommerce.command.dto;

import java.util.UUID;

public record CancelOrderCommand(
    UUID orderId,
    UUID customerId,
    String reason
) {
    public CancelOrderCommand {
        if (orderId == null) {
            throw new IllegalArgumentException("Order ID is required");
        }
        if (customerId == null) {
            throw new IllegalArgumentException("Customer ID is required");
        }
    }
}

// Write Model
package com.example.ecommerce.command.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

public class Order {
    private UUID id;
    private UUID customerId;
    private List<OrderItem> items;
    private String shippingAddress;
    private OrderStatus status;
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    public enum OrderStatus {
        PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
    }

    public Order(UUID id, UUID customerId, List<OrderItem> items, 
                 String shippingAddress) {
        this.id = id;
        this.customerId = customerId;
        this.items = new ArrayList<>(items);
        this.shippingAddress = shippingAddress;
        this.status = OrderStatus.PENDING;
        this.totalAmount = calculateTotal();
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException(
                "Only pending orders can be confirmed"
            );
        }
        this.status = OrderStatus.CONFIRMED;
        this.updatedAt = LocalDateTime.now();
    }

    public void cancel(String reason) {
        if (status == OrderStatus.DELIVERED || status == OrderStatus.CANCELLED) {
            throw new IllegalStateException(
                "Cannot cancel delivered or already cancelled orders"
            );
        }
        this.status = OrderStatus.CANCELLED;
        this.updatedAt = LocalDateTime.now();
    }

    public void ship() {
        if (status != OrderStatus.CONFIRMED) {
            throw new IllegalStateException(
                "Only confirmed orders can be shipped"
            );
        }
        this.status = OrderStatus.SHIPPED;
        this.updatedAt = LocalDateTime.now();
    }

    private BigDecimal calculateTotal() {
        return items.stream()
                .map(OrderItem::getTotalPrice)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    // Getters
    public UUID getId() { return id; }
    public UUID getCustomerId() { return customerId; }
    public List<OrderItem> getItems() { return items; }
    public String getShippingAddress() { return shippingAddress; }
    public OrderStatus getStatus() { return status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}

// Write Model
package com.example.ecommerce.command.model;

import java.math.BigDecimal;
import java.util.UUID;

public class OrderItem {
    private UUID productId;
    private String productName;
    private int quantity;
    private BigDecimal unitPrice;
    private BigDecimal totalPrice;

    public OrderItem(UUID productId, String productName, 
                    int quantity, BigDecimal unitPrice) {
        this.productId = productId;
        this.productName = productName;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
        this.totalPrice = unitPrice.multiply(BigDecimal.valueOf(quantity));
    }

    // Getters
    public UUID getProductId() { return productId; }
    public String getProductName() { return productName; }
    public int getQuantity() { return quantity; }
    public BigDecimal getUnitPrice() { return unitPrice; }
    public BigDecimal getTotalPrice() { return totalPrice; }
}

// Write Repository Interface
package com.example.ecommerce.command.repository;

import com.example.ecommerce.command.model.Order;
import java.util.Optional;
import java.util.UUID;

public interface OrderWriteRepository {
    Order save(Order order);
    Optional<Order> findById(UUID id);
    void delete(UUID id);
}

// Command Handler
package com.example.ecommerce.command.handler;

import com.example.ecommerce.command.dto.CreateOrderCommand;
import com.example.ecommerce.command.dto.OrderItemCommand;
import com.example.ecommerce.command.model.Order;
import com.example.ecommerce.command.model.OrderItem;
import com.example.ecommerce.command.repository.OrderWriteRepository;
import com.example.ecommerce.event.OrderCreatedEvent;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.context.ApplicationEventPublisher;

import java.util.List;
import java.util.stream.Collectors;

@Service
@Transactional
public class CreateOrderCommandHandler {
    
    private final OrderWriteRepository orderWriteRepository;
    private final ApplicationEventPublisher eventPublisher;

    public CreateOrderCommandHandler(OrderWriteRepository orderWriteRepository,
                                     ApplicationEventPublisher eventPublisher) {
        this.orderWriteRepository = orderWriteRepository;
        this.eventPublisher = eventPublisher;
    }

    public void handle(CreateOrderCommand command) {
        // Convert command items to domain items
        List<OrderItem> items = command.items().stream()
                .map(this::toOrderItem)
                .collect(Collectors.toList());

        // Create order
        Order order = new Order(
            command.orderId(),
            command.customerId(),
            items,
            command.shippingAddress()
        );

        // Save to write database
        Order savedOrder = orderWriteRepository.save(order);

        // Publish domain event
        OrderCreatedEvent event = new OrderCreatedEvent(
            savedOrder.getId(),
            savedOrder.getCustomerId(),
            savedOrder.getItems(),
            savedOrder.getTotalAmount(),
            savedOrder.getShippingAddress(),
            savedOrder.getStatus()
        );
        eventPublisher.publishEvent(event);
    }

    private OrderItem toOrderItem(OrderItemCommand itemCommand) {
        return new OrderItem(
            itemCommand.productId(),
            "Product-" + itemCommand.productId(), // Would fetch from product service
            itemCommand.quantity(),
            itemCommand.unitPrice()
        );
    }
}

// Command Handler
package com.example.ecommerce.command.handler;

import com.example.ecommerce.command.dto.CancelOrderCommand;
import com.example.ecommerce.command.model.Order;
import com.example.ecommerce.command.repository.OrderWriteRepository;
import com.example.ecommerce.event.OrderCancelledEvent;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.context.ApplicationEventPublisher;

import java.util.Optional;

@Service
@Transactional
public class CancelOrderCommandHandler {
    
    private final OrderWriteRepository orderWriteRepository;
    private final ApplicationEventPublisher eventPublisher;

    public CancelOrderCommandHandler(OrderWriteRepository orderWriteRepository,
                                    ApplicationEventPublisher eventPublisher) {
        this.orderWriteRepository = orderWriteRepository;
        this.eventPublisher = eventPublisher;
    }

    public void handle(CancelOrderCommand command) {
        // Load order
        Order order = orderWriteRepository.findById(command.orderId())
                .orElseThrow(() -> new IllegalArgumentException(
                    "Order not found: " + command.orderId()
                ));

        // Verify ownership
        if (!order.getCustomerId().equals(command.customerId())) {
            throw new IllegalArgumentException(
                "Customer does not own this order"
            );
        }

        // Cancel order
        order.cancel(command.reason());

        // Save to write database
        orderWriteRepository.save(order);

        // Publish domain event
        OrderCancelledEvent event = new OrderCancelledEvent(
            order.getId(),
            order.getCustomerId(),
            command.reason()
        );
        eventPublisher.publishEvent(event);
    }
}
```

### Query Side
```java
// Query DTO
package com.example.ecommerce.query.dto;

import java.util.UUID;

public record GetOrderQuery(
    UUID orderId
) {
    public GetOrderQuery {
        if (orderId == null) {
            throw new IllegalArgumentException("Order ID is required");
        }
    }
}

// Query DTO
package com.example.ecommerce.query.dto;

import java.time.LocalDateTime;
import java.util.UUID;

public record SearchOrdersQuery(
    UUID customerId,
    OrderStatus status,
    LocalDateTime startDate,
    LocalDateTime endDate,
    int page,
    int size
) {
    public enum OrderStatus {
        PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
    }
}

// Read Model
package com.example.ecommerce.query.model;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

public class OrderView {
    private UUID id;
    private UUID customerId;
    private String customerName;
    private List<OrderItemView> items;
    private String shippingAddress;
    private String status;
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    public OrderView(UUID id, UUID customerId, String customerName,
                     List<OrderItemView> items, String shippingAddress,
                     String status, BigDecimal totalAmount,
                     LocalDateTime createdAt, LocalDateTime updatedAt) {
        this.id = id;
        this.customerId = customerId;
        this.customerName = customerName;
        this.items = items;
        this.shippingAddress = shippingAddress;
        this.status = status;
        this.totalAmount = totalAmount;
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
    }

    // Getters
    public UUID getId() { return id; }
    public UUID getCustomerId() { return customerId; }
    public String getCustomerName() { return customerName; }
    public List<OrderItemView> getItems() { return items; }
    public String getShippingAddress() { return shippingAddress; }
    public String getStatus() { return status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}

// Read Model
package com.example.ecommerce.query.model;

import java.math.BigDecimal;
import java.util.UUID;

public class OrderItemView {
    private UUID productId;
    private String productName;
    private int quantity;
    private BigDecimal unitPrice;
    private BigDecimal totalPrice;

    public OrderItemView(UUID productId, String productName,
                        int quantity, BigDecimal unitPrice,
                        BigDecimal totalPrice) {
        this.productId = productId;
        this.productName = productName;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
        this.totalPrice = totalPrice;
    }

    // Getters
    public UUID getProductId() { return productId; }
    public String getProductName() { return productName; }
    public int getQuantity() { return quantity; }
    public BigDecimal getUnitPrice() { return unitPrice; }
    public BigDecimal getTotalPrice() { return totalPrice; }
}

// Read Repository Interface
package com.example.ecommerce.query.repository;

import com.example.ecommerce.query.model.OrderView;
import com.example.ecommerce.query.dto.SearchOrdersQuery;
import java.util.Optional;
import java.util.UUID;
import java.util.List;

public interface OrderReadRepository {
    Optional<OrderView> findById(UUID id);
    List<OrderView> search(SearchOrdersQuery query);
    List<OrderView> findByCustomerId(UUID customerId);
}

// Query Handler
package com.example.ecommerce.query.handler;

import com.example.ecommerce.query.dto.GetOrderQuery;
import com.example.ecommerce.query.model.OrderView;
import com.example.ecommerce.query.repository.OrderReadRepository;
import org.springframework.stereotype.Service;

import java.util.Optional;
import java.util.UUID;

@Service
public class GetOrderQueryHandler {
    
    private final OrderReadRepository orderReadRepository;

    public GetOrderQueryHandler(OrderReadRepository orderReadRepository) {
        this.orderReadRepository = orderReadRepository;
    }

    public Optional<OrderView> handle(GetOrderQuery query) {
        return orderReadRepository.findById(query.orderId());
    }
}

// Query Handler
package com.example.ecommerce.query.handler;

import com.example.ecommerce.query.dto.SearchOrdersQuery;
import com.example.ecommerce.query.model.OrderView;
import com.example.ecommerce.query.repository.OrderReadRepository;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class SearchOrdersQueryHandler {
    
    private final OrderReadRepository orderReadRepository;

    public SearchOrdersQueryHandler(OrderReadRepository orderReadRepository) {
        this.orderReadRepository = orderReadRepository;
    }

    public List<OrderView> handle(SearchOrdersQuery query) {
        return orderReadRepository.search(query);
    }
}
```

### Event and Projection
```java
// Domain Event
package com.example.ecommerce.event;

import com.example.ecommerce.command.model.OrderItem;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

public record OrderCreatedEvent(
    UUID orderId,
    UUID customerId,
    List<OrderItem> items,
    BigDecimal totalAmount,
    String shippingAddress,
    Order.OrderStatus status
) {
    public record OrderItem(
        UUID productId,
        String productName,
        int quantity,
        BigDecimal unitPrice,
        BigDecimal totalPrice
    ) {}
}

// Domain Event
package com.example.ecommerce.event;

import java.util.UUID;

public record OrderCancelledEvent(
    UUID orderId,
    UUID customerId,
    String reason
) {}

// Projection (Updates Read Model)
package com.example.ecommerce.projection;

import com.example.ecommerce.event.OrderCancelledEvent;
import com.example.ecommerce.event.OrderCreatedEvent;
import com.example.ecommerce.query.model.OrderItemView;
import com.example.ecommerce.query.model.OrderView;
import com.example.ecommerce.query.repository.OrderReadRepository;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

import java.util.List;
import java.util.stream.Collectors;

@Component
public class OrderProjection {
    
    private final OrderReadRepository orderReadRepository;

    public OrderProjection(OrderReadRepository orderReadRepository) {
        this.orderReadRepository = orderReadRepository;
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(OrderCreatedEvent event) {
        // Convert to read model
        List<OrderItemView> itemViews = event.items().stream()
                .map(item -> new OrderItemView(
                    item.productId(),
                    item.productName(),
                    item.quantity(),
                    item.unitPrice(),
                    item.totalPrice()
                ))
                .collect(Collectors.toList());

        OrderView orderView = new OrderView(
            event.orderId(),
            event.customerId(),
            "Customer-" + event.customerId(), // Would fetch from customer service
            itemViews,
            event.shippingAddress(),
            event.status().name(),
            event.totalAmount(),
            LocalDateTime.now(),
            LocalDateTime.now()
        );

        // Save to read database
        saveToReadDatabase(orderView);
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(OrderCancelledEvent event) {
        // Update read model
        updateOrderStatusInReadDatabase(event.orderId(), "CANCELLED");
    }

    private void saveToReadDatabase(OrderView orderView) {
        // Implementation depends on read database technology
        // Could be MongoDB, Elasticsearch, or denormalized SQL
        System.out.println("Saving order to read database: " + orderView.getId());
    }

    private void updateOrderStatusInReadDatabase(UUID orderId, String status) {
        // Update status in read database
        System.out.println("Updating order status in read database: " + orderId);
    }
}
```

### Controllers
```java
// Command Controller
package com.example.ecommerce.controller;

import com.example.ecommerce.command.dto.CancelOrderCommand;
import com.example.ecommerce.command.dto.CreateOrderCommand;
import com.example.ecommerce.command.handler.CancelOrderCommandHandler;
import com.example.ecommerce.command.handler.CreateOrderCommandHandler;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/commands/orders")
public class OrderCommandController {
    
    private final CreateOrderCommandHandler createOrderHandler;
    private final CancelOrderCommandHandler cancelOrderHandler;

    public OrderCommandController(CreateOrderCommandHandler createOrderHandler,
                                 CancelOrderCommandHandler cancelOrderHandler) {
        this.createOrderHandler = createOrderHandler;
        this.cancelOrderHandler = cancelOrderHandler;
    }

    @PostMapping
    public ResponseEntity<Void> createOrder(
            @Valid @RequestBody CreateOrderCommand command) {
        createOrderHandler.handle(command);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }

    @PostMapping("/{orderId}/cancel")
    public ResponseEntity<Void> cancelOrder(
            @PathVariable UUID orderId,
            @RequestBody CancelReasonRequest request) {
        CancelOrderCommand command = new CancelOrderCommand(
            orderId,
            request.customerId(),
            request.reason()
        );
        cancelOrderHandler.handle(command);
        return ResponseEntity.ok().build();
    }

    record CancelReasonRequest(UUID customerId, String reason) {}
}

// Query Controller
package com.example.ecommerce.controller;

import com.example.ecommerce.query.dto.GetOrderQuery;
import com.example.ecommerce.query.dto.SearchOrdersQuery;
import com.example.ecommerce.query.handler.GetOrderQueryHandler;
import com.example.ecommerce.query.handler.SearchOrdersQueryHandler;
import com.example.ecommerce.query.model.OrderView;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

@RestController
@RequestMapping("/api/queries/orders")
public class OrderQueryController {
    
    private final GetOrderQueryHandler getOrderHandler;
    private final SearchOrdersQueryHandler searchOrdersHandler;

    public OrderQueryController(GetOrderQueryHandler getOrderHandler,
                               SearchOrdersQueryHandler searchOrdersHandler) {
        this.getOrderHandler = getOrderHandler;
        this.searchOrdersHandler = searchOrdersHandler;
    }

    @GetMapping("/{orderId}")
    public ResponseEntity<OrderView> getOrder(@PathVariable UUID orderId) {
        GetOrderQuery query = new GetOrderQuery(orderId);
        Optional<OrderView> orderView = getOrderHandler.handle(query);
        return orderView.map(ResponseEntity::ok)
                       .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping("/search")
    public ResponseEntity<List<OrderView>> searchOrders(
            @RequestParam(required = false) UUID customerId,
            @RequestParam(required = false) String status,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        SearchOrdersQuery query = new SearchOrdersQuery(
            customerId,
            status != null ? SearchOrdersQuery.OrderStatus.valueOf(status) : null,
            null,
            null,
            page,
            size
        );
        List<OrderView> orders = searchOrdersHandler.handle(query);
        return ResponseEntity.ok(orders);
    }
}
```

## Pros and Cons

### Advantages
✅ **Optimized Performance**: Read and write models optimized separately
✅ **Scalability**: Scale read and write independently
✅ **Flexibility**: Use different databases for each model
✅ **Complex Queries**: Read model can be tailored for complex queries
✅ **Separation of Concerns**: Clear separation between commands and queries
✅ **Event-Driven**: Natural fit with event-driven architecture
✅ **Caching**: Read model can be cached effectively

### Disadvantages
❌ **Complexity**: More complex than traditional CRUD
❌ **Eventual Consistency**: Read model may be temporarily stale
❌ **Development Overhead**: Need to maintain two models
❌ **Learning Curve**: Team must understand the pattern
❌ **Debugging**: Harder to trace data flow
❌ **Data Duplication**: Data stored in multiple places
❌ **Synchronization**: Need to keep models in sync

## Real-World Use Cases

### E-commerce Platform
- **Write**: Order creation, payment processing, inventory updates
- **Read**: Product search, order history, customer dashboards
- **Benefits**: Fast product search, scalable order processing

### Social Media Platform
- **Write**: Post creation, likes, comments, follows
- **Read**: News feed, user profiles, analytics
- **Benefits**: Optimized feed generation, real-time analytics

### Financial Trading System
- **Write**: Trade execution, position updates, risk calculations
- **Read**: Portfolio views, historical reports, real-time quotes
- **Benefits**: Fast trade execution, complex reporting

## Interview Discussion Points

### Common Questions

**Q: How do you handle eventual consistency in CQRS?**
A: Eventual consistency is handled through:
- Event-driven updates from write to read model
- Idempotent event processing
- Retry mechanisms for failed updates
- Monitoring consistency lag
- User communication about potential delays

**Q: When should you use CQRS vs traditional CRUD?**
A: Use CQRS when:
- Read and write performance requirements differ significantly
- Complex queries are needed that don't match write model
- Different data stores are beneficial for reads vs writes
- High throughput requires independent scaling
- Use CRUD for simple applications with similar read/write patterns

**Q: How do you handle validation in CQRS?**
A: Validation occurs at multiple levels:
- Command validation: Format and business rule validation
- Write model validation: Domain invariants
- Query validation: Input sanitization
- Read model validation: Data integrity checks

**Q: What's the relationship between CQRS and Event Sourcing?**
A: They're complementary but independent:
- CQRS: Separates read and write models
- Event Sourcing: Stores state as event sequence
- Often used together: Event sourcing on write side, CQRS for separation
- Can use CQRS without event sourcing (traditional database)

**Q: How do you handle database transactions in CQRS?**
A: Transaction strategies:
- Write side: ACID transactions on write database
- Event publishing: Transactional event publishing
- Read side: Eventual consistency, no distributed transactions
- Compensation: Saga pattern for distributed transactions

### Advanced Topics

**Multiple Read Models**
- Different read models for different use cases
- Search-optimized model (Elasticsearch)
- Reporting-optimized model (data warehouse)
- Real-time model (Redis cache)

**Event Sourcing Integration**
- Store events instead of current state
- Rebuild read models from events
- Enable temporal queries
- Complex but powerful

**CQRS with Microservices**
- Each service has its own CQRS implementation
- Event-driven communication between services
- Independent scaling per service
- Eventual consistency across services

## Best Practices

### Command Design
1. **Validate early**: Validate commands before processing
2. **Use immutability**: Commands should be immutable
3. **Clear intent**: Command names should express intent
4. **Return results**: Return success/failure, not data
5. **Keep focused**: Each command should do one thing

### Query Design
1. **Optimize for reads**: Design queries for specific use cases
2. **Denormalize**: Read model can be denormalized
3. **Use projections**: Return only needed data
4. **Support pagination**: Handle large result sets
5. **Cache effectively**: Cache read model results

### Event Design
1. **Use past tense**: OrderCreated, not CreateOrder
2. **Include all data**: Events should be self-contained
3. **Immutable**: Events should never change
4. **Version events**: Support schema evolution
5. **Unique IDs**: Every event needs unique identifier

## Related Patterns

- **Event Sourcing**: Often used with CQRS
- **Event-Driven Architecture**: Natural fit with CQRS
- **Saga Pattern**: For distributed transactions
- **Materialized View**: Read model as materialized view

## Technology Choices

### Write Side
- **PostgreSQL**: Relational with strong consistency
- **MongoDB**: Document store for complex models
- **Cassandra**: High write throughput
- **Event Store**: For event sourcing

### Read Side
- **Elasticsearch**: Full-text search and analytics
- **PostgreSQL**: With materialized views
- **MongoDB**: Document store for flexible queries
- **Redis**: Fast caching layer

### Java Frameworks
- **Axon Framework**: CQRS and event sourcing
- **Spring Boot**: With custom implementation
- **Micronaut**: Lightweight framework
- **Quarkus**: Cloud-native framework

---

**Next**: Learn about [Saga Pattern](./saga-pattern.md) for distributed transactions

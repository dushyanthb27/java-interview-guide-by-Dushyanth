# Domain-Driven Design

[Home](../../README.md) > Bonus > Architecture Patterns > Domain-Driven Design

---

## Overview
Domain-Driven Design (DDD) is an approach to software development that focuses on creating a deep understanding of the business domain and modeling the software to reflect that domain. Introduced by Eric Evans in his 2003 book, DDD emphasizes collaboration between technical experts and domain experts to build software that solves real business problems effectively.

## Pattern Description

DDD provides a set of principles and patterns for designing software based on the business domain. It's not just about technical architecture—it's about creating a shared language between developers and domain experts, and structuring the code to reflect business concepts and relationships.

### Core DDD Layers

```
┌─────────────────────────────────────────┐
│         User Interface Layer            │  ← Handles user interaction
├─────────────────────────────────────────┤
│         Application Layer               │  ← Orchestrates use cases
├─────────────────────────────────────────┤
│         Domain Layer                    │  ← Core business logic
│  ┌─────────────────────────────────┐   │
│  │   Bounded Contexts              │   │
│  │   ┌──────────────┐              │   │
│  │   │   Entities   │              │   │
│  │   ├──────────────┤              │   │
│  │   │ Value Objects│              │   │
│  │   ├──────────────┤              │   │
│  │   │  Aggregates  │              │   │
│  │   ├──────────────┤              │   │
│  │   │  Repositories│              │   │
│  │   ├──────────────┤              │   │
│  │   │ Domain Events│              │   │
│  │   └──────────────┘              │   │
│  └─────────────────────────────────┘   │
├─────────────────────────────────────────┤
│         Infrastructure Layer            │  ← Technical implementations
└─────────────────────────────────────────┘
```

### Key DDD Concepts

**Ubiquitous Language**
- A shared language used by both developers and domain experts
- Used in conversations, documentation, and code
- Eliminates translation between business and technical terms
- Evolves as understanding of the domain deepens

**Bounded Context**
- A specific part of the domain with clear boundaries
- Has its own ubiquitous language
- Can be implemented as a separate module or service
- Different contexts may have different models for the same concept

**Aggregate**
- A cluster of domain objects treated as a single unit
- Has a root entity (aggregate root) that provides access
- Enforces consistency boundaries
- Controls all access to objects within the aggregate

**Entity**
- An object with a unique identity that persists over time
- Identified by ID, not by attributes
- Has lifecycle and state changes
- Example: Customer, Order, Product

**Value Object**
- An object defined by its attributes, not identity
- Immutable when possible
- Can be shared freely
- Example: Money, Address, DateRange

**Domain Event**
- Something that happened in the domain
- Represents a business-relevant event
- Published when an important state change occurs
- Enables loose coupling between bounded contexts

**Repository**
- Provides collection-like access to aggregates
- Hides the details of data persistence
- Implemented in infrastructure layer
- Interface defined in domain layer

## When to Use Domain-Driven Design

### Ideal Scenarios
✅ Complex business domains with rich business logic
✅ When business logic is critical and complex
✅ Teams with domain experts available for collaboration
✅ Long-lived projects requiring maintainability
✅ Multiple teams working on different domains
✅ When the domain is the competitive advantage

### When to Avoid
❌ Simple CRUD applications (overkill)
❌ When domain experts are not available
❌ Short-term projects with limited lifespan
❌ When technical complexity outweighs domain complexity
❌ Teams without DDD experience and no time to learn

## Java Implementation

### Project Structure
```
src/
├── main/
│   ├── java/
│   │   └── com/
│   │       └── example/
│   │           └── orders/
│   │               ├── OrdersApplication.java
│   │               ├── application/
│   │               │   ├── service/
│   │               │   │   └── OrderService.java
│   │               │   └── dto/
│   │               │       ├── CreateOrderRequest.java
│   │               │       └── OrderResponse.java
│   │               ├── domain/
│   │               │   ├── model/
│   │               │   │   ├── Order.java
│   │               │   │   ├── OrderItem.java
│   │               │   │   ├── Customer.java
│   │               │   │   ├── Product.java
│   │               │   │   ├── Money.java
│   │               │   │   └── Address.java
│   │               │   ├── repository/
│   │               │   │   ├── OrderRepository.java
│   │               │   │   └── CustomerRepository.java
│   │               │   └── event/
│   │               │       ├── OrderCreated.java
│   │               │       └── OrderCancelled.java
│   │               └── infrastructure/
│   │                   ├── persistence/
│   │                   │   ├── JpaOrderRepository.java
│   │                   │   ├── OrderEntity.java
│   │                   │   └── OrderItemEntity.java
│   │                   └── messaging/
│   │                       └── KafkaEventPublisher.java
```

### Domain Layer - Value Objects
```java
// Money Value Object
package com.example.orders.domain.model;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Objects;

public final class Money {
    private final BigDecimal amount;
    private final String currency;

    public Money(BigDecimal amount, String currency) {
        if (amount == null) {
            throw new IllegalArgumentException("Amount cannot be null");
        }
        if (currency == null || currency.trim().isEmpty()) {
            throw new IllegalArgumentException("Currency cannot be null or empty");
        }
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency.toUpperCase();
    }

    public static Money of(BigDecimal amount, String currency) {
        return new Money(amount, currency);
    }

    public static Money zero(String currency) {
        return new Money(BigDecimal.ZERO, currency);
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "Cannot add money with different currencies"
            );
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money subtract(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "Cannot subtract money with different currencies"
            );
        }
        BigDecimal result = this.amount.subtract(other.amount);
        if (result.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException(
                "Result would be negative"
            );
        }
        return new Money(result, this.currency);
    }

    public Money multiply(int multiplier) {
        return new Money(
            this.amount.multiply(BigDecimal.valueOf(multiplier)),
            this.currency
        );
    }

    public boolean isGreaterThan(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "Cannot compare money with different currencies"
            );
        }
        return this.amount.compareTo(other.amount) > 0;
    }

    public boolean isLessThan(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "Cannot compare money with different currencies"
            );
        }
        return this.amount.compareTo(other.amount) < 0;
    }

    // Getters
    public BigDecimal getAmount() {
        return amount;
    }

    public String getCurrency() {
        return currency;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Money money = (Money) o;
        return Objects.equals(amount, money.amount) &&
               Objects.equals(currency, money.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }

    @Override
    public String toString() {
        return amount + " " + currency;
    }
}

// Address Value Object
package com.example.orders.domain.model;

import java.util.Objects;

public final class Address {
    private final String street;
    private final String city;
    private final String state;
    private final String postalCode;
    private final String country;

    public Address(String street, String city, String state,
                   String postalCode, String country) {
        if (street == null || street.trim().isEmpty()) {
            throw new IllegalArgumentException("Street is required");
        }
        if (city == null || city.trim().isEmpty()) {
            throw new IllegalArgumentException("City is required");
        }
        if (postalCode == null || postalCode.trim().isEmpty()) {
            throw new IllegalArgumentException("Postal code is required");
        }
        
        this.street = street.trim();
        this.city = city.trim();
        this.state = state != null ? state.trim() : "";
        this.postalCode = postalCode.trim();
        this.country = country != null ? country.trim() : "";
    }

    public static Address of(String street, String city, String state,
                            String postalCode, String country) {
        return new Address(street, city, state, postalCode, country);
    }

    // Getters
    public String getStreet() { return street; }
    public String getCity() { return city; }
    public String getState() { return state; }
    public String getPostalCode() { return postalCode; }
    public String getCountry() { return country; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Address address = (Address) o;
        return Objects.equals(street, address.street) &&
               Objects.equals(city, address.city) &&
               Objects.equals(state, address.state) &&
               Objects.equals(postalCode, address.postalCode) &&
               Objects.equals(country, address.country);
    }

    @Override
    public int hashCode() {
        return Objects.hash(street, city, state, postalCode, country);
    }

    @Override
    public String toString() {
        return street + ", " + city + ", " + state + " " + postalCode;
    }
}
```

### Domain Layer - Entities and Aggregates
```java
// Order Aggregate Root
package com.example.orders.domain.model;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

public class Order {
    private final UUID id;
    private final UUID customerId;
    private final List<OrderItem> items;
    private OrderStatus status;
    private final Money totalAmount;
    private final Address shippingAddress;
    private final LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private LocalDateTime shippedAt;
    private LocalDateTime deliveredAt;

    public Order(UUID id, UUID customerId, List<OrderItem> items,
                 Address shippingAddress) {
        if (id == null) {
            throw new IllegalArgumentException("Order ID cannot be null");
        }
        if (customerId == null) {
            throw new IllegalArgumentException("Customer ID cannot be null");
        }
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }
        if (shippingAddress == null) {
            throw new IllegalArgumentException("Shipping address is required");
        }

        this.id = id;
        this.customerId = customerId;
        this.items = new ArrayList<>(items);
        this.status = OrderStatus.PENDING;
        this.totalAmount = calculateTotal(items);
        this.shippingAddress = shippingAddress;
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
        
        validate();
    }

    private Money calculateTotal(List<OrderItem> items) {
        Money total = Money.zero("USD");
        for (OrderItem item : items) {
            total = total.add(item.getSubtotal());
        }
        return total;
    }

    public void ship() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException(
                "Only pending orders can be shipped. Current status: " + status
            );
        }
        this.status = OrderStatus.SHIPPED;
        this.shippedAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    public void deliver() {
        if (status != OrderStatus.SHIPPED) {
            throw new IllegalStateException(
                "Only shipped orders can be delivered. Current status: " + status
            );
        }
        this.status = OrderStatus.DELIVERED;
        this.deliveredAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    public void cancel() {
        if (status == OrderStatus.DELIVERED) {
            throw new IllegalStateException(
                "Delivered orders cannot be cancelled"
            );
        }
        if (status == OrderStatus.CANCELLED) {
            throw new IllegalStateException(
                "Order is already cancelled"
            );
        }
        this.status = OrderStatus.CANCELLED;
        this.updatedAt = LocalDateTime.now();
    }

    public void addItem(OrderItem item) {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException(
                "Can only add items to pending orders"
            );
        }
        this.items.add(item);
        this.updatedAt = LocalDateTime.now();
    }

    public void removeItem(UUID itemId) {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException(
                "Can only remove items from pending orders"
            );
        }
        this.items.removeIf(item -> item.getId().equals(itemId));
        if (this.items.isEmpty()) {
            throw new IllegalStateException(
                "Order must have at least one item"
            );
        }
        this.updatedAt = LocalDateTime.now();
    }

    private void validate() {
        // Ensure all items have the same currency
        String currency = items.get(0).getPrice().getCurrency();
        for (OrderItem item : items) {
            if (!item.getPrice().getCurrency().equals(currency)) {
                throw new IllegalArgumentException(
                    "All items must have the same currency"
                );
            }
        }
    }

    // Getters
    public UUID getId() { return id; }
    public UUID getCustomerId() { return customerId; }
    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }
    public OrderStatus getStatus() { return status; }
    public Money getTotalAmount() { return totalAmount; }
    public Address getShippingAddress() { return shippingAddress; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public LocalDateTime getShippedAt() { return shippedAt; }
    public LocalDateTime getDeliveredAt() { return deliveredAt; }
}

// Order Item Entity (part of Order Aggregate)
package com.example.orders.domain.model;

import java.util.Objects;
import java.util.UUID;

public class OrderItem {
    private final UUID id;
    private final UUID productId;
    private final String productName;
    private final Money price;
    private final int quantity;

    public OrderItem(UUID id, UUID productId, String productName,
                    Money price, int quantity) {
        if (id == null) {
            throw new IllegalArgumentException("Item ID cannot be null");
        }
        if (productId == null) {
            throw new IllegalArgumentException("Product ID cannot be null");
        }
        if (productName == null || productName.trim().isEmpty()) {
            throw new IllegalArgumentException("Product name is required");
        }
        if (price == null) {
            throw new IllegalArgumentException("Price cannot be null");
        }
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }

        this.id = id;
        this.productId = productId;
        this.productName = productName.trim();
        this.price = price;
        this.quantity = quantity;
    }

    public Money getSubtotal() {
        return price.multiply(quantity);
    }

    // Getters
    public UUID getId() { return id; }
    public UUID getProductId() { return productId; }
    public String getProductName() { return productName; }
    public Money getPrice() { return price; }
    public int getQuantity() { return quantity; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        OrderItem orderItem = (OrderItem) o;
        return Objects.equals(id, orderItem.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

// Order Status Enum
package com.example.orders.domain.model;

public enum OrderStatus {
    PENDING,
    SHIPPED,
    DELIVERED,
    CANCELLED
}
```

### Domain Layer - Repository Interface
```java
package com.example.orders.domain.repository;

import com.example.orders.domain.model.Order;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(UUID id);
    List<Order> findByCustomerId(UUID customerId);
    List<Order> findByStatus(OrderStatus status);
    void delete(UUID id);
    boolean existsById(UUID id);
}
```

### Domain Layer - Events
```java
package com.example.orders.domain.event;

import java.time.LocalDateTime;
import java.util.UUID;

public class OrderCreated {
    private final UUID orderId;
    private final UUID customerId;
    private final double totalAmount;
    private final LocalDateTime occurredAt;

    public OrderCreated(UUID orderId, UUID customerId, double totalAmount) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.totalAmount = totalAmount;
        this.occurredAt = LocalDateTime.now();
    }

    // Getters
    public UUID getOrderId() { return orderId; }
    public UUID getCustomerId() { return customerId; }
    public double getTotalAmount() { return totalAmount; }
    public LocalDateTime getOccurredAt() { return occurredAt; }
}

package com.example.orders.domain.event;

import java.time.LocalDateTime;
import java.util.UUID;

public class OrderCancelled {
    private final UUID orderId;
    private final UUID customerId;
    private final LocalDateTime occurredAt;

    public OrderCancelled(UUID orderId, UUID customerId) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.occurredAt = LocalDateTime.now();
    }

    // Getters
    public UUID getOrderId() { return orderId; }
    public UUID getCustomerId() { return customerId; }
    public LocalDateTime getOccurredAt() { return occurredAt; }
}
```

### Application Layer - Service
```java
package com.example.orders.application.service;

import com.example.orders.application.dto.CreateOrderRequest;
import com.example.orders.application.dto.OrderResponse;
import com.example.orders.domain.event.OrderCreated;
import com.example.orders.domain.model.*;
import com.example.orders.domain.repository.OrderRepository;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;

@Service
@Transactional
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public OrderService(OrderRepository orderRepository,
                       ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    public OrderResponse createOrder(CreateOrderRequest request) {
        List<OrderItem> items = request.getItems().stream()
            .map(item -> new OrderItem(
                UUID.randomUUID(),
                item.getProductId(),
                item.getProductName(),
                Money.of(item.getPrice(), "USD"),
                item.getQuantity()
            ))
            .collect(Collectors.toList());

        Address shippingAddress = Address.of(
            request.getStreet(),
            request.getCity(),
            request.getState(),
            request.getPostalCode(),
            request.getCountry()
        );

        Order order = new Order(
            UUID.randomUUID(),
            request.getCustomerId(),
            items,
            shippingAddress
        );

        Order saved = orderRepository.save(order);
        
        // Publish domain event
        eventPublisher.publishEvent(new OrderCreated(
            saved.getId(),
            saved.getCustomerId(),
            saved.getTotalAmount().getAmount().doubleValue()
        ));

        return toResponse(saved);
    }

    public OrderResponse getOrder(UUID id) {
        return orderRepository.findById(id)
                .map(this::toResponse)
                .orElseThrow(() -> new IllegalArgumentException(
                    "Order not found: " + id
                ));
    }

    public List<OrderResponse> getCustomerOrders(UUID customerId) {
        return orderRepository.findByCustomerId(customerId).stream()
                .map(this::toResponse)
                .collect(Collectors.toList());
    }

    public void shipOrder(UUID id) {
        Order order = orderRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException(
                    "Order not found: " + id
                ));
        
        order.ship();
        orderRepository.save(order);
    }

    public void cancelOrder(UUID id) {
        Order order = orderRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException(
                    "Order not found: " + id
                ));
        
        order.cancel();
        orderRepository.save(order);
        
        // Publish domain event
        eventPublisher.publishEvent(new OrderCancelled(
            order.getId(),
            order.getCustomerId()
        ));
    }

    private OrderResponse toResponse(Order order) {
        return new OrderResponse(
            order.getId(),
            order.getCustomerId(),
            order.getItems().stream()
                .map(item -> new OrderResponse.ItemResponse(
                    item.getId(),
                    item.getProductId(),
                    item.getProductName(),
                    item.getPrice().getAmount(),
                    item.getQuantity()
                ))
                .collect(Collectors.toList()),
            order.getTotalAmount().getAmount(),
            order.getStatus().name(),
            order.getShippingAddress().getStreet(),
            order.getShippingAddress().getCity(),
            order.getShippingAddress().getState(),
            order.getShippingAddress().getPostalCode(),
            order.getShippingAddress().getCountry(),
            order.getCreatedAt(),
            order.getUpdatedAt()
        );
    }
}
```

### Infrastructure Layer - Repository Implementation
```java
package com.example.orders.infrastructure.persistence;

import com.example.orders.domain.model.Order;
import com.example.orders.domain.model.OrderStatus;
import com.example.orders.domain.repository.OrderRepository;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;
import java.util.UUID;
import java.util.stream.Collectors;

@Repository
@Transactional
public class JpaOrderRepository implements OrderRepository {
    
    private final SpringDataOrderRepository springDataRepo;
    private final OrderMapper mapper;

    public JpaOrderRepository(SpringDataOrderRepository springDataRepo,
                             OrderMapper mapper) {
        this.springDataRepo = springDataRepo;
        this.mapper = mapper;
    }

    @Override
    public Order save(Order order) {
        OrderEntity entity = mapper.toEntity(order);
        OrderEntity saved = springDataRepo.save(entity);
        return mapper.toDomain(saved);
    }

    @Override
    public Optional<Order> findById(UUID id) {
        return springDataRepo.findById(id)
                .map(mapper::toDomain);
    }

    @Override
    public List<Order> findByCustomerId(UUID customerId) {
        return springDataRepo.findByCustomerId(customerId).stream()
                .map(mapper::toDomain)
                .collect(Collectors.toList());
    }

    @Override
    public List<Order> findByStatus(OrderStatus status) {
        return springDataRepo.findByStatus(status.name()).stream()
                .map(mapper::toDomain)
                .collect(Collectors.toList());
    }

    @Override
    public void delete(UUID id) {
        springDataRepo.deleteById(id);
    }

    @Override
    public boolean existsById(UUID id) {
        return springDataRepo.existsById(id);
    }
}
```

## Pros and Cons

### Advantages
✅ **Business Alignment**: Code directly reflects business concepts
✅ **Shared Language**: Ubiquitous language improves communication
✅ **Maintainability**: Clear boundaries make code easier to maintain
✅ **Testability**: Domain logic can be tested independently
✅ **Flexibility**: Bounded contexts can evolve independently
✅ **Scalability**: Different contexts can be deployed separately
✅ **Quality**: Focus on domain leads to better business solutions

### Disadvantages
❌ **Learning Curve**: Requires significant DDD knowledge
❌ **Initial Effort**: More upfront design and modeling work
❌ **Overhead**: Can be overkill for simple domains
❌ **Team Requirements**: Needs domain expert collaboration
❌ **Complexity**: Adds architectural complexity
❌ **Time Investment**: Slower initial development
❌ **Cultural Change**: Requires mindset shift in teams

## Real-World Use Cases

### E-commerce Platform
- **Order Context**: Order management, fulfillment
- **Inventory Context**: Stock management, warehousing
- **Payment Context**: Payment processing, refunds
- **Customer Context**: Customer profiles, authentication

### Banking System
- **Account Context**: Account management, transactions
- **Lending Context**: Loan applications, approvals
- **Risk Context**: Risk assessment, compliance
- **Reporting Context**: Financial reports, analytics

### Healthcare System
- **Patient Context**: Patient records, medical history
- **Appointment Context**: Scheduling, calendar management
- **Billing Context**: Insurance claims, payments
- **Prescription Context**: Medication management

## Interview Discussion Points

### Common Questions

**Q: What's the difference between an entity and a value object?**
A: Entities have identity that persists over time and are identified by ID, not attributes. Value objects are defined by their attributes and are immutable. Two value objects with the same attributes are equal, while two entities with the same attributes are different if they have different IDs.

**Q: What is a bounded context and why is it important?**
A: A bounded context is a specific part of the domain with clear boundaries and its own ubiquitous language. It's important because it allows different parts of a large system to have different models for the same concept (e.g., "Customer" in Sales vs. "Customer" in Support), preventing ambiguity and enabling independent evolution.

**Q: How do you decide what should be an aggregate root?**
A: An aggregate root should:
- Represent a meaningful business concept
- Need to maintain consistency invariants
- Be accessed as a single unit from outside
- Have a clear lifecycle
- Be the only way to access objects within the aggregate

**Q: What's the role of ubiquitous language in DDD?**
A: Ubiquitous language is a shared language used by developers and domain experts in conversations, documentation, and code. It eliminates translation between business and technical terms, reduces misunderstandings, and ensures the code reflects the business domain accurately.

**Q: How do you handle relationships between bounded contexts?**
A: Relationships between bounded contexts are handled through:
- **Context Mapping**: Define how contexts relate (partnership, anti-corruption layer, etc.)
- **Domain Events**: Asynchronous communication via events
- **APIs**: Synchronous communication through well-defined interfaces
- **Shared Kernel**: Carefully selected shared concepts when necessary

### Advanced Topics

**Strategic Design**
- Domain modeling sessions with domain experts
- Context mapping to understand relationships
- Bounded context identification
- Core domain vs supporting domains vs generic domains

**Tactical Patterns**
- Aggregates and aggregate roots
- Domain services for operations that don't fit entities
- Factories for complex object creation
- Specifications for business rules

**Event-Driven DDD**
- Domain events for loose coupling
- Event sourcing for audit trails
- CQRS for complex read/write requirements
- Event storming for collaborative modeling

## Best Practices

### Domain Modeling
1. **Collaborate with domain experts**: Regular modeling sessions
2. **Focus on the core domain**: Invest where it matters most
3. **Use the ubiquitous language**: Speak the same language in code
4. **Model behavior, not just data**: Rich domain models
5. **Keep aggregates small**: Better performance and consistency

### Code Organization
1. **Package by bounded context**: Clear module boundaries
2. **Separate domain from infrastructure**: Clean architecture
3. **Use modules for bounded contexts**: Physical separation
4. **Define clear interfaces**: Between layers and contexts

### Team Collaboration
1. **Include domain experts in design**: They understand the business
2. **Regular knowledge sharing**: Cross-team communication
3. **Document the ubiquitous language**: Glossary and diagrams
4. **Iterative refinement**: Models evolve with understanding

## Related Patterns

- **Hexagonal Architecture**: Complements DDD with clear layer boundaries
- **Event-Driven Architecture**: Works well with domain events
- **CQRS**: Often used with DDD for complex read/write scenarios
- **Event Sourcing**: Natural fit with domain events
- **Microservices**: Bounded contexts can become microservices

## Migration Strategies

### From Anemic Domain Model
1. Identify business logic scattered in services
2. Move logic to appropriate domain entities
3. Introduce value objects for concepts
4. Define aggregates and consistency boundaries
5. Implement repositories for aggregate access

### From Monolith to Bounded Contexts
1. Identify natural boundaries in the domain
2. Define ubiquitous language for each context
3. Create modules for each bounded context
4. Implement context mapping
5. Gradually extract into separate services if needed

---

**Next**: Learn about [Clean Architecture](./clean-architecture.md) for another approach to organizing complex systems

# Transaction Management

## Overview

Transaction management is critical for maintaining data consistency and integrity in database applications. A transaction is a sequence of operations performed as a single logical unit of work - either all operations succeed, or none do (atomicity).

## ACID Properties

Every transaction must satisfy the ACID properties:

### Atomicity
All operations in a transaction succeed or fail together. No partial updates.

```java
// ❌ Without atomicity - partial update possible
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    accountRepository.withdraw(fromId, amount);  // Success
    // Network error here!
    accountRepository.deposit(toId, amount);     // Never executes
    // Money lost!
}

// ✅ With atomicity - both or none
@Transactional
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    accountRepository.withdraw(fromId, amount);
    accountRepository.deposit(toId, amount);
    // Both succeed or both rollback
}
```

### Consistency
Transaction brings database from one valid state to another valid state.

```java
@Transactional
public void createOrder(Order order) {
    // Business rule: order total must equal sum of items
    validateOrderTotal(order);
    
    orderRepository.save(order);
    orderItemRepository.saveAll(order.getItems());
    
    // Database constraints ensure consistency
    // e.g., NOT NULL, FOREIGN KEY, CHECK constraints
}
```

### Isolation
Concurrent transactions don't interfere with each other.

```java
// Without proper isolation:
// Transaction A reads balance = 100
// Transaction B reads balance = 100
// Transaction A deducts 50, writes 50
// Transaction B deducts 30, writes 70
// Final balance = 70 (should be 20!)

// With proper isolation (SERIALIZABLE):
// One transaction waits for the other to complete
```

### Durability
Once committed, changes persist even after system failure.

```java
@Transactional
public void processPayment(Payment payment) {
    paymentRepository.save(payment);
    // After commit, payment is saved to disk
    // Even if server crashes immediately after
}
```

## Transaction Isolation Levels

### Read Uncommitted

Lowest isolation level. Transactions can read uncommitted changes from other transactions.

```java
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
public void readUncommittedExample() {
    // Can read data that might be rolled back
    // Dirty reads possible
    Account account = accountRepository.findById(1L);
    // Value might be from uncommitted transaction
}
```

**Problems:**
- Dirty reads: Reading uncommitted data
- Non-repeatable reads: Same query returns different results
- Phantom reads: New rows appear in subsequent queries

**Use case:** Rarely used, only for analytics where accuracy isn't critical.

### Read Committed

Default in many databases (PostgreSQL, SQL Server, Oracle). Transactions can only read committed changes.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void readCommittedExample() {
    // Can only read committed data
    // No dirty reads
    // But non-repeatable reads and phantom reads possible
}
```

**Problems:**
- Non-repeatable reads: Data changes between reads
- Phantom reads: New rows appear

**Use case:** Most general-purpose applications.

### Repeatable Read

Ensures that if a row is read twice, same values are returned.

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void repeatableReadExample() {
    // First read
    Account account1 = accountRepository.findById(1L);
    
    // Another transaction commits changes
    // ...
    
    // Second read - same values as first
    Account account2 = accountRepository.findById(1L);
    // account1 == account2 (same values)
}
```

**Problems:**
- Phantom reads still possible (new rows in range queries)

**Use case:** Default in MySQL InnoDB. Good for reporting.

### Serializable

Highest isolation level. Transactions are completely isolated.

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void serializableExample() {
    // Complete isolation
    // No dirty reads, no non-repeatable reads, no phantom reads
    // But poor performance due to locking
}
```

**Problems:**
- Poor performance due to extensive locking
- Deadlocks more likely

**Use case:** Financial transactions, critical operations.

## Isolation Level Comparison

| Isolation Level | Dirty Reads | Non-repeatable Reads | Phantom Reads | Performance |
|----------------|-------------|----------------------|----------------|-------------|
| Read Uncommitted | Possible | Possible | Possible | Best |
| Read Committed | Prevented | Possible | Possible | Good |
| Repeatable Read | Prevented | Prevented | Possible | Fair |
| Serializable | Prevented | Prevented | Prevented | Poor |

## Spring Transaction Management

### Declarative Transactions with @Transactional

```java
@Service
public class OrderService {

    @Transactional
    public Order createOrder(OrderDTO dto) {
        // All database operations in this method are transactional
        Order order = new Order();
        order.setCustomerId(dto.getCustomerId());
        order.setTotal(dto.getTotal());
        
        order = orderRepository.save(order);
        
        for (OrderItemDTO itemDto : dto.getItems()) {
            OrderItem item = new OrderItem();
            item.setOrder(order);
            item.setProductId(itemDto.getProductId());
            item.setQuantity(itemDto.getQuantity());
            item.setPrice(itemDto.getPrice());
            orderItemRepository.save(item);
        }
        
        // Update inventory
        inventoryService.updateStock(dto.getItems());
        
        return order;
    }
}
```

### Transaction Propagation

Defines how transactions relate to each other.

#### REQUIRED (Default)

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
    // If no transaction exists, create one
    // If transaction exists, join it
    methodB(); // Joins same transaction
}

@Transactional(propagation = Propagation.REQUIRED)
public void methodB() {
    // Runs in same transaction as methodA
}
```

#### REQUIRES_NEW

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void methodA() {
    // Creates new transaction
    methodB(); // Suspends current, creates new transaction
    // Resumes original transaction
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void methodB() {
    // Runs in its own transaction
    // Commits independently
}
```

**Use case:** Logging, auditing, independent operations.

#### SUPPORTS

```java
@Transactional(propagation = Propagation.SUPPORTS)
public void methodA() {
    // Runs in existing transaction if present
    // Runs non-transactionally if no transaction exists
    methodB();
}

@Transactional(propagation = Propagation.SUPPORTS)
public void methodB() {
    // Supports but doesn't require transaction
}
```

**Use case:** Read-only operations that can run with or without transaction.

#### NOT_SUPPORTED

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void methodA() {
    // Suspends existing transaction if present
    // Runs non-transactionally
    methodB();
}

@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void methodB() {
    // Always runs non-transactionally
}
```

**Use case:** Operations that shouldn't be in transaction (e.g., sending emails).

#### MANDATORY

```java
@Transactional(propagation = Propagation.MANDATORY)
public void methodA() {
    // Must run within existing transaction
    // Throws exception if no transaction exists
    methodB();
}

@Transactional(propagation = Propagation.MANDATORY)
public void methodB() {
    // Requires existing transaction
}
```

**Use case:** Critical operations that must be transactional.

#### NEVER

```java
@Transactional(propagation = Propagation.NEVER)
public void methodA() {
    // Must not run within transaction
    // Throws exception if transaction exists
    methodB();
}

@Transactional(propagation = Propagation.NEVER)
public void methodB() {
    // Never runs in transaction
}
```

**Use case:** Operations that explicitly shouldn't be transactional.

#### NESTED

```java
@Transactional(propagation = Propagation.NESTED)
public void methodA() {
    // Runs in nested transaction if parent exists
    // Creates new transaction if no parent exists
    try {
        methodB();
    } catch (Exception e) {
        // Can rollback nested without rolling back parent
    }
}

@Transactional(propagation = Propagation.NESTED)
public void methodB() {
    // Nested transaction
    // Can rollback independently
}
```

**Use case:** Partial rollback scenarios (requires savepoint support).

### Transaction Configuration Examples

```java
@Service
public class TransactionalService {

    // Basic transaction
    @Transactional
    public void basicTransaction() {
        // Default: REQUIRED, READ_COMMITTED
    }

    // Read-only transaction (performance optimization)
    @Transactional(readOnly = true)
    public List<Order> getOrders() {
        return orderRepository.findAll();
    }

    // Custom isolation level
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void criticalOperation() {
        // Highest isolation
    }

    // Custom timeout (seconds)
    @Transactional(timeout = 30)
    public void longRunningOperation() {
        // Timeout after 30 seconds
    }

    // Specific exception for rollback
    @Transactional(rollbackFor = {SQLException.class, CustomException.class})
    public void customRollback() {
        // Rollback on these exceptions
    }

    // No rollback on specific exception
    @Transactional(noRollbackFor = {BusinessException.class})
    public void noRollbackOnBusiness() {
        // Don't rollback on BusinessException
    }

    // Combined configuration
    @Transactional(
        propagation = Propagation.REQUIRED,
        isolation = Isolation.READ_COMMITTED,
        timeout = 30,
        readOnly = false,
        rollbackFor = {Exception.class},
        noRollbackFor = {BusinessException.class}
    )
    public void fullyConfigured() {
        // All options specified
    }
}
```

## Programmatic Transaction Management

### TransactionTemplate

```java
@Service
public class ProgrammaticTransactionService {

    @Autowired
    private TransactionTemplate transactionTemplate;

    @Autowired
    private OrderRepository orderRepository;

    public Order createOrderWithTemplate(OrderDTO dto) {
        return transactionTemplate.execute(status -> {
            Order order = new Order();
            order.setCustomerId(dto.getCustomerId());
            order = orderRepository.save(order);
            
            for (OrderItemDTO itemDto : dto.getItems()) {
                OrderItem item = new OrderItem();
                item.setOrder(order);
                item.setProductId(itemDto.getProductId());
                orderItemRepository.save(item);
            }
            
            return order;
        });
    }

    public void createOrderWithRollback(OrderDTO dto) {
        transactionTemplate.executeWithoutResult(status -> {
            try {
                Order order = new Order();
                order.setCustomerId(dto.getCustomerId());
                order = orderRepository.save(order);
                
                // Some condition that should trigger rollback
                if (dto.getTotal().compareTo(BigDecimal.ZERO) <= 0) {
                    status.setRollbackOnly();
                    return;
                }
                
                // Continue processing...
            } catch (Exception e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }
}
```

### PlatformTransactionManager

```java
@Service
public class LowLevelTransactionService {

    @Autowired
    private PlatformTransactionManager transactionManager;

    @Autowired
    private OrderRepository orderRepository;

    public void createOrderLowLevel(OrderDTO dto) {
        TransactionDefinition def = new DefaultTransactionDefinition();
        TransactionStatus status = transactionManager.getTransaction(def);
        
        try {
            Order order = new Order();
            order.setCustomerId(dto.getCustomerId());
            order = orderRepository.save(order);
            
            for (OrderItemDTO itemDto : dto.getItems()) {
                OrderItem item = new OrderItem();
                item.setOrder(order);
                item.setProductId(itemDto.getProductId());
                orderItemRepository.save(item);
            }
            
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

## Transaction Rollback Strategies

### Default Rollback Behavior

```java
// By default, Spring rolls back on RuntimeException and Error
@Transactional
public void defaultRollback() {
    // RuntimeException triggers rollback
    throw new RuntimeException("Error");
}

@Transactional
public void noRollbackOnChecked() {
    // Checked exception does NOT trigger rollback by default
    throw new SQLException("Error"); // No rollback!
}
```

### Custom Rollback Configuration

```java
// Rollback on specific checked exceptions
@Transactional(rollbackFor = {SQLException.class, IOException.class})
public void customRollback() throws SQLException {
    // Now SQLException triggers rollback
    throw new SQLException("Error");
}

// Don't rollback on specific runtime exceptions
@Transactional(noRollbackFor = {BusinessException.class})
public void noRollbackOnRuntime() {
    // BusinessException does NOT trigger rollback
    throw new BusinessException("Business rule violation");
}
```

### Conditional Rollback

```java
@Transactional
public void conditionalRollback(OrderDTO dto) {
    Order order = orderRepository.save(dto.toEntity());
    
    // Business logic that might need rollback
    if (order.getTotal().compareTo(BigDecimal.ZERO) <= 0) {
        // Manually trigger rollback
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        throw new BusinessException("Invalid order total");
    }
}
```

## Distributed Transactions

### Two-Phase Commit (2PC)

```java
// Requires JTA transaction manager
@Transactional
public void distributedTransaction() {
    // Phase 1: Prepare
    // Both resources prepare to commit
    dataSource1.execute("UPDATE account SET balance = balance - 100");
    dataSource2.execute("UPDATE account SET balance = balance + 100");
    
    // Phase 2: Commit
    // Both resources commit or both rollback
}
```

### Saga Pattern (Alternative to 2PC)

```java
@Service
public class SagaOrderService {

    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;

    @Transactional
    public Order createOrder(OrderDTO dto) {
        // Step 1: Create order
        Order order = orderRepository.save(dto.toEntity());
        
        try {
            // Step 2: Process payment
            paymentService.processPayment(order);
            
            try {
                // Step 3: Reserve inventory
                inventoryService.reserveInventory(order);
                
                order.setStatus("COMPLETED");
                return orderRepository.save(order);
                
            } catch (InventoryException e) {
                // Compensating transaction: refund payment
                paymentService.refundPayment(order);
                order.setStatus("PAYMENT_REFUNDED");
                throw e;
            }
        } catch (PaymentException e) {
            // Compensating transaction: cancel order
            order.setStatus("CANCELLED");
            throw e;
        }
    }
}
```

## Transaction Synchronization

### Before/After Commit Callbacks

```java
@Service
public class TransactionSyncService {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private EmailService emailService;

    @Transactional
    public void createOrderAndNotify(OrderDTO dto) {
        Order order = orderRepository.save(dto.toEntity());
        
        // Register synchronization callback
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                @Override
                public void beforeCommit() {
                    log.info("About to commit transaction for order {}", order.getId());
                }

                @Override
                public void afterCommit() {
                    // Send email only after successful commit
                    emailService.sendOrderConfirmation(order);
                }

                @Override
                public void afterCompletion(int status) {
                    if (status == STATUS_COMMITTED) {
                        log.info("Transaction committed for order {}", order.getId());
                    } else {
                        log.warn("Transaction rolled back for order {}", order.getId());
                    }
                }
            }
        );
    }
}
```

## Transaction Monitoring

### Custom Transaction Metrics

```java
@Aspect
@Component
public class TransactionMonitor {

    @Autowired
    private MeterRegistry meterRegistry;

    @Around("@annotation(org.springframework.transaction.annotation.Transactional)")
    public Object monitorTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            Object result = joinPoint.proceed();
            sample.stop(Timer.builder("transaction.duration")
                .tag("method", methodName)
                .tag("status", "success")
                .register(meterRegistry));
            return result;
        } catch (Exception e) {
            sample.stop(Timer.builder("transaction.duration")
                .tag("method", methodName)
                .tag("status", "error")
                .tag("exception", e.getClass().getSimpleName())
                .register(meterRegistry));
            throw e;
        }
    }
}
```

## Common Pitfalls

### 1. @Transactional on Private Methods

```java
// ❌ Doesn't work - private method
@Transactional
private void privateMethod() {
    // Spring AOP can't intercept private methods
}

// ✅ Use public method
@Transactional
public void publicMethod() {
    // Works correctly
}
```

### 2. Self-Invocation

```java
// ❌ Doesn't work - self-invocation bypasses proxy
@Service
public class OrderService {
    
    @Transactional
    public void methodA() {
        methodB(); // Direct call, bypasses transaction proxy
    }
    
    @Transactional
    public void methodB() {
        // Not transactional when called from methodA
    }
}

// ✅ Inject self or use TransactionTemplate
@Service
public class OrderService {
    
    @Autowired
    private OrderService self; // Inject proxy
    
    @Transactional
    public void methodA() {
        self.methodB(); // Call through proxy
    }
    
    @Transactional
    public void methodB() {
        // Transactional
    }
}
```

### 3. Long-Running Transactions

```java
// ❌ Bad: External API call in transaction
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);
    externalService.notifyCustomer(order); // Slow external call
    // Holds database locks during API call!
}

// ✅ Good: Short transactions
public void processOrder(Order order) {
    transactionTemplate.execute(status -> {
        orderRepository.save(order);
        return null;
    });
    externalService.notifyCustomer(order); // Outside transaction
}
```

### 4. Wrong Isolation Level

```java
// ❌ Bad: SERIALIZABLE for read operations
@Transactional(isolation = Isolation.SERIALIZABLE, readOnly = true)
public List<Order> getAllOrders() {
    // Unnecessary overhead
}

// ✅ Good: READ_COMMITTED for reads
@Transactional(isolation = Isolation.READ_COMMITTED, readOnly = true)
public List<Order> getAllOrders() {
    // Appropriate isolation
}
```

### 5. Not Handling Exceptions

```java
// ❌ Bad: Swallows exception, no rollback
@Transactional
public void badExample() {
    try {
        orderRepository.save(order);
    } catch (Exception e) {
        log.error("Error", e);
        // Exception swallowed, transaction commits!
    }
}

// ✅ Good: Re-throw exception
@Transactional
public void goodExample() {
    try {
        orderRepository.save(order);
    } catch (Exception e) {
        log.error("Error", e);
        throw e; // Triggers rollback
    }
}
```

## Best Practices

1. **Keep transactions short** - Minimize lock holding time
2. **Use appropriate isolation levels** - Balance consistency and performance
3. **Specify rollback behavior** - Explicitly declare rollbackFor
4. **Avoid external calls in transactions** - API calls, file I/O
5. **Use readOnly for queries** - Performance optimization
6. **Monitor transaction duration** - Alert on long transactions
7. **Understand propagation** - Choose the right propagation behavior
8. **Test rollback scenarios** - Verify rollback behavior

## Performance Implications

| Isolation Level | Performance Impact | Lock Contention | Use Case |
|----------------|-------------------|-----------------|----------|
| Read Uncommitted | Minimal | Low | Analytics |
| Read Committed | Low | Low-Medium | General purpose |
| Repeatable Read | Medium | Medium | Reporting |
| Serializable | High | High | Critical operations |

## Interview Questions

1. Explain the ACID properties with examples
2. What are the different transaction isolation levels?
3. How does Spring's @Transactional work internally?
4. What is the difference between REQUIRED and REQUIRES_NEW?
5. How do you handle distributed transactions?
6. What is the Saga pattern?
7. Why shouldn't you make external API calls in transactions?
8. How do you monitor transaction performance?

## Related Topics

- [Connection Pooling](./connection-pooling-patterns.md)
- [ORM Patterns](./orm-patterns.md)
- [Data Migration Patterns](./data-migration-patterns.md)

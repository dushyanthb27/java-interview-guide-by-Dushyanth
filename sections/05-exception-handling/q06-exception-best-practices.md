# Q6: What Are Exception Handling Best Practices?

[Home](../../README.md) > [Exception Handling](./README.md) > Exception Best Practices

---

## Difficulty: 🟠 Hard

## Question
> What are the key best practices for exception handling in Java? What patterns and anti-patterns should developers be aware of?

---

## Answer

### 1. Don't Use Exceptions for Flow Control

```java
// ❌ Using exceptions as control flow - VERY expensive
public boolean isInteger(String input) {
    try {
        Integer.parseInt(input);
        return true;
    } catch (NumberFormatException e) {
        return false;
    }
}

// ✅ Use validation instead
public boolean isInteger(String input) {
    if (input == null || input.isEmpty()) return false;
    return input.chars().allMatch(Character::isDigit);
}

// ❌ Using exceptions to break out of loops
try {
    for (int i = 0; ; i++) {
        array[i].process(); // relies on ArrayIndexOutOfBoundsException to stop
    }
} catch (ArrayIndexOutOfBoundsException e) { }

// ✅ Use proper loop bounds
for (int i = 0; i < array.length; i++) {
    array[i].process();
}
```

---

### 2. Catch Specific Exceptions

```java
// ❌ Catching overly broad exception types
try {
    String content = readFile(path);
    User user = parseJson(content);
    saveToDatabase(user);
} catch (Exception e) {
    log.error("Something went wrong", e); // Which operation failed?
}

// ✅ Catch specific exceptions with targeted handling
try {
    String content = readFile(path);
    User user = parseJson(content);
    saveToDatabase(user);
} catch (FileNotFoundException e) {
    log.error("Config file not found: {}", path);
    throw new AppStartupException("Missing config", e);
} catch (JsonParseException e) {
    log.error("Invalid JSON in file: {}", path);
    throw new ConfigException("Malformed config", e);
} catch (SQLException e) {
    log.error("Database save failed for user");
    throw new DataAccessException("Save failed", e);
}
```

---

### 3. Never Swallow Exceptions

```java
// ❌ Swallowing exceptions - bugs hide here
try {
    criticalOperation();
} catch (Exception e) {
    // Nothing! The error is completely lost
}

// ❌ Logging and swallowing - often just as bad
try {
    processPayment(order);
} catch (PaymentException e) {
    log.error("Payment failed", e);
    // But then what? Order is left in inconsistent state!
}

// ✅ Handle meaningfully OR rethrow
try {
    processPayment(order);
} catch (PaymentException e) {
    log.error("Payment failed for order: {}", order.getId(), e);
    order.setStatus(OrderStatus.PAYMENT_FAILED);
    notifyCustomer(order, e);
    throw e; // Let the caller know too
}
```

---

### 4. Prefer Standard Exceptions

```java
// ✅ Use JDK exceptions for common cases - everyone knows them
// IllegalArgumentException - bad input
public void setPort(int port) {
    if (port < 0 || port > 65535) {
        throw new IllegalArgumentException("Port out of range: " + port);
    }
}

// IllegalStateException - wrong state
public void start() {
    if (isRunning) {
        throw new IllegalStateException("Server already running");
    }
}

// NullPointerException - null where not allowed
public void process(Request request) {
    Objects.requireNonNull(request, "Request must not be null");
}

// UnsupportedOperationException - operation not supported
@Override
public void delete(Long id) {
    throw new UnsupportedOperationException("Delete not supported for audit records");
}

// ConcurrentModificationException - concurrent modification detected
// IndexOutOfBoundsException - index out of range
```

---

### 5. Document Exceptions with @throws

```java
// ✅ Document all thrown exceptions (checked AND significant unchecked)
/**
 * Transfers funds between two accounts.
 *
 * @param fromAccountId the source account ID
 * @param toAccountId   the destination account ID
 * @param amount        the amount to transfer (must be positive)
 * @throws AccountNotFoundException if either account does not exist
 * @throws InsufficientFundsException if the source account has insufficient balance
 * @throws IllegalArgumentException if amount is zero or negative
 * @throws TransferException if the transfer fails for a system reason
 */
public TransferResult transfer(String fromAccountId, String toAccountId, BigDecimal amount)
        throws TransferException {
    // ...
}
```

---

### 6. Exception Translation (Abstraction Layers)

```java
// ✅ Translate low-level exceptions to domain-level exceptions
public class UserRepository {

    public User findById(Long id) {
        try {
            return jdbcTemplate.queryForObject(
                "SELECT * FROM users WHERE id = ?",
                userRowMapper, id);
        } catch (EmptyResultDataAccessException e) {
            // Translate data access exception to domain exception
            throw new UserNotFoundException("User not found: " + id, e);
        } catch (DataAccessException e) {
            // Translate generic data access to repository exception
            throw new RepositoryException("Failed to query user: " + id, e);
        }
    }
}

// ❌ Leaking implementation details
public class UserService {
    public User getUser(Long id) throws SQLException { // Leaks JDBC!
        return userDao.findById(id);
    }
}

// ✅ Translate at the boundary
public class UserService {
    public User getUser(Long id) {
        try {
            return userRepository.findById(id);
        } catch (RepositoryException e) {
            throw new ServiceException("Cannot retrieve user", e);
        }
    }
}
```

---

### 7. Logging Strategy

```java
// ❌ Log AND throw - causes duplicate log entries
try {
    process();
} catch (Exception e) {
    log.error("Processing failed", e);  // Logged here
    throw e;                             // Logged AGAIN at the caller
}

// ✅ Rule: Log OR throw, not both
// Option A: Handle and log (don't rethrow)
try {
    process();
} catch (ProcessException e) {
    log.error("Processing failed, using fallback", e);
    return fallbackValue;
}

// Option B: Translate and throw (don't log, let caller log)
try {
    process();
} catch (ProcessException e) {
    throw new ServiceException("Processing failed", e);
}

// ✅ Exception: Log at the boundary (controllers, message listeners)
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception e) {
        log.error("Unhandled exception", e); // Log at the outermost boundary
        return ResponseEntity.status(500).body(new ErrorResponse("Internal Error"));
    }
}
```

---

### 8. Fail-Fast Principle

```java
// ✅ Validate inputs immediately - fail fast
public Order createOrder(String customerId, List<OrderItem> items, String shippingAddress) {
    // Validate everything upfront
    Objects.requireNonNull(customerId, "customerId must not be null");
    if (items == null || items.isEmpty()) {
        throw new IllegalArgumentException("Order must have at least one item");
    }
    if (shippingAddress == null || shippingAddress.isBlank()) {
        throw new IllegalArgumentException("Shipping address is required");
    }

    // All validations passed - proceed with business logic
    return processOrder(customerId, items, shippingAddress);
}

// ❌ Failing late - harder to debug, state may be partially modified
public Order createOrder(String customerId, List<OrderItem> items, String shippingAddress) {
    Order order = new Order();
    order.setCustomerId(customerId);       // Could be null - NullPointerException later
    for (OrderItem item : items) {          // items could be null - NPE here
        order.addItem(item);
    }
    order.setShippingAddress(shippingAddress); // Blank address fails during shipping
    return orderRepository.save(order);         // Fails in DB - too late!
}
```

---

### 9. Provide Context in Exception Messages

```java
// ❌ Useless error messages
throw new RuntimeException("Error");
throw new RuntimeException("Operation failed");
throw new RuntimeException("Invalid input");

// ✅ Include relevant context
throw new OrderNotFoundException(
    String.format("Order not found [orderId=%s, customerId=%s]", orderId, customerId));

throw new ValidationException(
    String.format("Invalid age: %d. Must be between %d and %d", age, MIN_AGE, MAX_AGE));

throw new ConnectionException(
    String.format("Failed to connect to %s:%d after %d retries", host, port, retryCount));
```

---

### 10. Clean Up Resources Properly

```java
// ❌ Resource leak on exception
public void processFile(String path) throws IOException {
    FileInputStream fis = new FileInputStream(path);
    // If process() throws, fis is NEVER closed
    process(fis);
    fis.close();
}

// ✅ Always use try-with-resources
public void processFile(String path) throws IOException {
    try (FileInputStream fis = new FileInputStream(path)) {
        process(fis);
    }
}
```

---

### Summary Cheat Sheet

| Practice | Rule |
|----------|------|
| Flow control | Never use exceptions for flow control |
| Catch scope | Catch the most specific exception possible |
| Swallowing | Never catch without handling or rethrowing |
| Standard types | Prefer JDK exceptions for common scenarios |
| Documentation | Use `@throws` for all significant exceptions |
| Translation | Translate exceptions at layer boundaries |
| Logging | Log OR throw, never both |
| Fail-fast | Validate inputs at method entry |
| Messages | Include relevant context in error messages |
| Resources | Always use try-with-resources |

---

## Interview Tips
- Lead with the "don't use exceptions for flow control" principle
- Discuss the "log or throw, not both" guideline with examples
- Explain exception translation across architectural layers
- Mention the fail-fast principle and its connection to clean code
- Show awareness of the performance cost of exceptions (see Q11)

---

[<< Previous: Q5 - Custom Exceptions](./q05-custom-exceptions.md) | [Next: Q7 - Error vs Exception >>](./q07-error-vs-exception.md)

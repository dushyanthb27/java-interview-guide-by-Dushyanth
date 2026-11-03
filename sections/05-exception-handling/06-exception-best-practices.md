# Q6 🟠 What Are Exception Handling Best Practices?

**Question:** What are the key best practices for exception handling in Java? What patterns and anti-patterns should developers be aware of?

---

## Answer

### 1. Don't Use Exceptions for Flow Control

```java
// ❌ Using exceptions as control flow — VERY expensive
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

> **🎯 Interviewer Insight:** Exception creation is expensive — it captures the full stack trace. Using exceptions for control flow (parsing, loop termination) is an anti-pattern that kills performance. 

Always prefer validation, regex, or proper loop bounds.

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

> **⚠️ Common Mistake:** `catch (Exception e)` or `catch (Throwable t)` masks programming bugs (`NullPointerException`, `IllegalArgumentException`) alongside legitimate errors. 

Catch the most specific exception possible.

---

### 3. Never Swallow Exceptions

```java
// ❌ Swallowing exceptions — bugs hide here
try {
    criticalOperation();
} catch (Exception e) {
    // Nothing! The error is completely lost
}

// ❌ Logging and swallowing — often just as bad
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

> **🏭 Production Consideration:** Swallowed exceptions are the #1 cause of "it works on my machine" bugs in production. 

If you catch an exception, you MUST either handle it (retry, fallback, compensate) or rethrow it. Logging alone is not handling.

---

### 4. Prefer Standard Exceptions

```java
// ✅ Use JDK exceptions for common cases — everyone knows them

// IllegalArgumentException — bad input
public void setPort(int port) {
    if (port < 0 || port > 65535) {
        throw new IllegalArgumentException("Port out of range: " + port);
    }
}

// IllegalStateException — wrong state
public void start() {
    if (isRunning) {
        throw new IllegalStateException("Server already running");
    }
}

// NullPointerException — null where not allowed
public void process(Request request) {
    Objects.requireNonNull(request, "Request must not be null");
}

// UnsupportedOperationException — operation not supported
@Override
public void delete(Long id) {
    throw new UnsupportedOperationException("Delete not supported for audit records");
}

// ConcurrentModificationException — concurrent modification detected
// IndexOutOfBoundsException — index out of range
```

> **🎯 Interviewer Insight:** Don't create custom exceptions for standard cases. `IllegalArgumentException`, `IllegalStateException`, `NullPointerException`, `UnsupportedOperationException` are universally understood. Custom exceptions are for domain-specific errors (e.g., `InsufficientFundsException`, `OrderNotFoundException`).

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

> **🏭 Production Consideration:** IDEs and tools (SonarQube, Checkstyle) can enforce `@throws` documentation. Documenting unchecked exceptions is optional but highly recommended for public APIs — it tells callers what can go wrong without forcing them to read the source.

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

> **🎯 Interviewer Insight:** This is the **exception translation pattern** — a cornerstone of clean architecture. Each layer (DAO → Repository → Service → Controller) translates exceptions to its own domain. The controller never sees `SQLException`; it sees `ServiceException` or `UserNotFoundException`.

---

### 7. Logging Strategy — Log OR Throw, Not Both

```java
// ❌ Log AND throw — causes duplicate log entries
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

> **⚠️ Common Mistake:** Logging at every layer creates duplicate log entries and noise. **Log once at the boundary** (controller, message listener, scheduled job entry point). Inner layers should translate and rethrow.

---

### 8. Fail-Fast Principle

```java
// ✅ Validate inputs immediately — fail fast
public Order createOrder(String customerId, List<OrderItem> items, String shippingAddress) {
    // Validate everything upfront
    Objects.requireNonNull(customerId, "customerId must not be null");
    if (items == null || items.isEmpty()) {
        throw new IllegalArgumentException("Order must have at least one item");
    }
    if (shippingAddress == null || shippingAddress.isBlank()) {
        throw new IllegalArgumentException("Shipping address is required");
    }

    // All validations passed — proceed with business logic
    return processOrder(customerId, items, shippingAddress);
}

// ❌ Failing late — harder to debug, state may be partially modified
public Order createOrder(String customerId, List<OrderItem> items, String shippingAddress) {
    Order order = new Order();
    order.setCustomerId(customerId);       // Could be null — NullPointerException later
    for (OrderItem item : items) {          // items could be null — NPE here
        order.addItem(item);
    }
    order.setShippingAddress(shippingAddress); // Blank address fails during shipping
    return orderRepository.save(order);         // Fails in DB — too late!
}
```

> **🏭 Production Consideration:** Fail-fast prevents partially constructed objects and inconsistent state. Validate at method entry, not halfway through. This is especially critical for constructors and factory methods.

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

> **🎯 Interviewer Insight:** A good exception message answers: **What failed? What was the input? What was expected?** This turns a 30-minute debugging session into a 30-second fix.

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

## 🎯 Interviewer Insight

Interviewers test these specific areas:

1. **Flow control anti-pattern** — parsing with try-catch vs regex/validation
2. **Catch specificity** — why `catch (Exception e)` is bad
3. **Swallowing** — log AND rethrow, or handle completely
4. **Exception translation** — layer boundaries, don't leak implementation details
5. **Logging strategy** — log once at the boundary
6. **Fail-fast** — validate at entry, not halfway through
7. **Context in messages** — what, input, expected

---

## ⚡ Follow-up Questions

> **⚡ Follow-up:** Why is using exceptions for control flow a performance problem?

> **⚡ Follow-up:** What's the difference between "handle and log" vs "translate and throw"?

> **⚡ Follow-up:** How do you decide which exceptions to document with `@throws`?

> **⚡ Follow-up:** When would you create a custom exception vs using a standard JDK one?

> **⚡ Follow-up:** How does exception translation work in Spring's `@ControllerAdvice`?

---

## 📚 Related Topics

- [Custom Exceptions](./05-custom-exceptions.md)
- [Error vs Exception →](./07-error-vs-exception.md)
- [Exception Chaining →](./08-exception-chaining.md)
- [Exception Performance →](./11-exception-performance.md)

---

[← Back to Section Index](./README.md) · [Previous: Custom Exceptions ←](./05-custom-exceptions.md) · [Next: Error vs Exception →](./07-error-vs-exception.md)

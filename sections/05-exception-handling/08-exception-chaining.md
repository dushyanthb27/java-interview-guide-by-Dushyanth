# Q8: What Is Exception Chaining?

[Home](../../README.md) > [Exception Handling](./README.md) > Exception Chaining

---

## Difficulty: 🟡 Medium

## Question
> What is exception chaining in Java? How do you preserve the root cause when translating exceptions across layers?

---

## Answer

### What Is Exception Chaining?

Exception chaining (or exception wrapping) is the practice of catching one exception and throwing a new one while preserving the original as the **cause**. This allows you to translate low-level exceptions into domain-specific ones without losing diagnostic information.

---

### Two Ways to Set the Cause

```java
// ✅ Method 1: Constructor with Throwable cause
try {
    loadConfigFile("app.yml");
} catch (IOException e) {
    throw new ConfigurationException("Failed to load config", e);
    //                                                        ^ cause
}

// ✅ Method 2: initCause() - for exceptions without a cause constructor
try {
    parseData(input);
} catch (ParseException e) {
    ConfigurationException ce = new ConfigurationException("Bad config format");
    ce.initCause(e);  // Set cause after construction
    throw ce;
}

// ❌ initCause() can only be called ONCE
Exception ex = new Exception("test");
ex.initCause(new IOException("first"));
ex.initCause(new IOException("second")); // throws IllegalStateException!
```

---

### Exception Translation Pattern

```java
// ✅ Layer-by-layer exception translation preserving the full chain

// === Data Access Layer ===
public class UserRepository {

    public User findById(Long id) {
        try {
            return jdbcTemplate.queryForObject(
                "SELECT * FROM users WHERE id = ?",
                userMapper, id);
        } catch (EmptyResultDataAccessException e) {
            throw new UserNotFoundException("User not found: " + id, e);
        } catch (DataAccessException e) {
            throw new RepositoryException("Database query failed", e);
        }
    }
}

// === Service Layer ===
public class UserService {

    public UserDTO getUser(Long id) {
        try {
            User user = userRepository.findById(id);
            return userMapper.toDTO(user);
        } catch (UserNotFoundException e) {
            throw e; // Let it propagate as-is (already domain-level)
        } catch (RepositoryException e) {
            throw new ServiceException("Failed to retrieve user: " + id, e);
        }
    }
}

// === Controller Layer ===
@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
        try {
            return ResponseEntity.ok(userService.getUser(id));
        } catch (UserNotFoundException e) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND, e.getMessage(), e);
        } catch (ServiceException e) {
            throw new ResponseStatusException(HttpStatus.INTERNAL_SERVER_ERROR,
                "Service error", e);
        }
    }
}

// The exception chain now looks like:
// ResponseStatusException: "Service error"
//   └── caused by ServiceException: "Failed to retrieve user: 42"
//       └── caused by RepositoryException: "Database query failed"
//           └── caused by DataAccessException: "Connection refused"
```

---

### Traversing the Cause Chain

```java
public class ExceptionChainUtils {

    // ✅ Get the root cause of an exception chain
    public static Throwable getRootCause(Throwable throwable) {
        Throwable cause = throwable;
        while (cause.getCause() != null && cause.getCause() != cause) {
            cause = cause.getCause();
        }
        return cause;
    }

    // ✅ Get the full chain as a list
    public static List<Throwable> getCauseChain(Throwable throwable) {
        List<Throwable> chain = new ArrayList<>();
        Throwable current = throwable;
        while (current != null && !chain.contains(current)) {
            chain.add(current);
            current = current.getCause();
        }
        return Collections.unmodifiableList(chain);
    }

    // ✅ Check if a specific exception type is anywhere in the chain
    public static <T extends Throwable> boolean hasCauseOfType(
            Throwable throwable, Class<T> type) {
        Throwable current = throwable;
        while (current != null) {
            if (type.isInstance(current)) return true;
            current = current.getCause();
        }
        return false;
    }

    // ✅ Find first exception of a specific type in the chain
    @SuppressWarnings("unchecked")
    public static <T extends Throwable> Optional<T> findCause(
            Throwable throwable, Class<T> type) {
        Throwable current = throwable;
        while (current != null) {
            if (type.isInstance(current)) return Optional.of((T) current);
            current = current.getCause();
        }
        return Optional.empty();
    }
}

// Usage:
try {
    userService.getUser(42L);
} catch (ServiceException e) {
    Throwable root = ExceptionChainUtils.getRootCause(e);
    System.out.println("Root cause: " + root.getClass().getSimpleName());

    if (ExceptionChainUtils.hasCauseOfType(e, ConnectException.class)) {
        System.out.println("Database connection issue detected");
    }

    ExceptionChainUtils.findCause(e, SQLException.class)
        .ifPresent(sql -> System.out.println("SQL State: " + sql.getSQLState()));
}
```

---

### Using Apache Commons Lang (Shortcut)

```java
import org.apache.commons.lang3.exception.ExceptionUtils;

try {
    processOrder(order);
} catch (Exception e) {
    // Get root cause
    Throwable root = ExceptionUtils.getRootCause(e);

    // Get root cause message
    String rootMessage = ExceptionUtils.getRootCauseMessage(e);

    // Get full stack trace as string
    String fullTrace = ExceptionUtils.getStackTrace(e);

    // Get chain of throwables
    List<Throwable> chain = ExceptionUtils.getThrowableList(e);

    // Check if specific type is in the chain
    int index = ExceptionUtils.indexOfType(e, SQLException.class);
    if (index >= 0) {
        System.out.println("SQL error found in chain");
    }
}
```

---

### Practical Example: REST API Error Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception ex) {
        // ✅ Walk the cause chain to build a meaningful response
        Throwable root = getRootCause(ex);

        ErrorResponse response = new ErrorResponse();
        response.setMessage(ex.getMessage());
        response.setRootCause(root.getClass().getSimpleName() + ": " + root.getMessage());
        response.setTimestamp(Instant.now());

        // ✅ Determine HTTP status from exception chain
        int status = determineStatus(ex);
        response.setStatus(status);

        // ✅ Log the full chain for debugging
        log.error("Request failed: {} -> Root cause: {}",
            ex.getMessage(), root.getMessage(), ex);

        return ResponseEntity.status(status).body(response);
    }

    private int determineStatus(Throwable ex) {
        Throwable current = ex;
        while (current != null) {
            if (current instanceof ResourceNotFoundException) return 404;
            if (current instanceof ValidationException) return 400;
            if (current instanceof AuthorizationException) return 403;
            current = current.getCause();
        }
        return 500;
    }
}
```

---

### Anti-Patterns

```java
// ❌ Losing the original cause - destroys diagnostic information
try {
    connectToDatabase();
} catch (SQLException e) {
    throw new ServiceException("Connection failed");
    // Original SQLException is LOST! No way to diagnose the problem
}

// ✅ Always preserve the cause
try {
    connectToDatabase();
} catch (SQLException e) {
    throw new ServiceException("Connection failed", e);
}

// ❌ Wrapping in generic RuntimeException with no context
try {
    processOrder(order);
} catch (Exception e) {
    throw new RuntimeException(e); // No message, no context
}

// ✅ Provide context when wrapping
try {
    processOrder(order);
} catch (Exception e) {
    throw new OrderProcessingException(
        "Failed to process order: " + order.getId(), e);
}

// ❌ Circular cause chain (don't do this!)
Exception a = new Exception("A");
Exception b = new Exception("B");
a.initCause(b);
b.initCause(a); // throws IllegalArgumentException in modern Java

// ❌ Catching, logging, AND rethrowing (duplicate logs)
try {
    process();
} catch (Exception e) {
    log.error("Failed", e);    // Logged here
    throw new ServiceException("Failed", e);  // Will be logged again upstream
}
```

---

### Stack Trace Output with Chaining

```
ServiceException: Failed to retrieve user: 42
    at com.app.UserService.getUser(UserService.java:25)
    at com.app.UserController.getUser(UserController.java:18)
    ...
Caused by: RepositoryException: Database query failed
    at com.app.UserRepository.findById(UserRepository.java:30)
    at com.app.UserService.getUser(UserService.java:22)
    ... 10 more
Caused by: java.sql.SQLException: Connection refused
    at com.mysql.jdbc.ConnectionImpl.connect(ConnectionImpl.java:456)
    at com.app.UserRepository.findById(UserRepository.java:27)
    ... 12 more
```

The `"... N more"` means those stack frames are identical to frames already shown above.

---

## Interview Tips
- Explain that exception chaining preserves the root cause for debugging
- Know both approaches: constructor parameter and `initCause()`
- Describe the exception translation pattern across architectural layers
- Mention that `getCause()` returns `null` at the end of the chain
- Emphasize: **never** create a new exception without including the original as the cause

---

[<< Previous: Q7 - Error vs Exception](./q07-error-vs-exception.md) | [Next: Q9 - Suppressed Exceptions >>](./q09-suppressed-exceptions.md)

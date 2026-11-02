# Q5 🟡 How to Create Custom Exceptions?

**Question:** How do you create custom exceptions in Java? What are the constructor patterns and design considerations?

---

## Answer

### Choosing the Superclass

```java
// Checked exception — caller MUST handle or declare
public class OrderNotFoundException extends Exception { }

// Unchecked exception — caller handling is OPTIONAL
public class InvalidOrderException extends RuntimeException { }
```

| Extend | When |
|--------|------|
| `Exception` | Recoverable condition; force caller to handle |
| `RuntimeException` | Programming error; bug in caller's code |

> **🎯 Interviewer Insight:** The choice between checked and unchecked is a design decision. Checked = "caller can recover" (file not found, network timeout). Unchecked = "caller wrote buggy code" (null argument, invalid state). Most modern frameworks (Spring, Kotlin) favor unchecked with good documentation.

---

### Standard Constructor Patterns — The 4-Constructor Rule

```java
// ✅ Follow the 4-constructor pattern (same as JDK exceptions)
public class OrderNotFoundException extends RuntimeException {

    // 1. No-arg constructor
    public OrderNotFoundException() {
        super();
    }

    // 2. Message-only constructor
    public OrderNotFoundException(String message) {
        super(message);
    }

    // 3. Message + cause constructor
    public OrderNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }

    // 4. Cause-only constructor
    public OrderNotFoundException(Throwable cause) {
        super(cause);
    }
}
```

> **⚠️ Common Mistake:** Forgetting the cause constructors. Without `super(message, cause)`, you lose the original exception's stack trace when wrapping. **Always include all four constructors.**

---

### Adding Error Codes and Metadata

```java
public class BusinessException extends RuntimeException {

    private final String errorCode;
    private final int httpStatus;

    public BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = 500;
    }

    public BusinessException(String errorCode, String message, int httpStatus) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }

    public BusinessException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.httpStatus = 500;
    }

    public String getErrorCode() { return errorCode; }
    public int getHttpStatus() { return httpStatus; }
}

// Usage:
throw new BusinessException("ORD-404", "Order not found", 404);
throw new BusinessException("PAY-001", "Payment declined", 402);
```

---

### Custom Exception Hierarchy for Applications

```java
// ✅ Base exception for the entire application
public abstract class AppException extends RuntimeException {

    private final ErrorCode errorCode;

    protected AppException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    protected AppException(ErrorCode errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public ErrorCode getErrorCode() { return errorCode; }
}

// Error code enum — single source of truth for HTTP status mapping
public enum ErrorCode {
    RESOURCE_NOT_FOUND(404),
    VALIDATION_FAILED(400),
    UNAUTHORIZED(401),
    FORBIDDEN(403),
    CONFLICT(409),
    INTERNAL_ERROR(500);

    private final int httpStatus;

    ErrorCode(int httpStatus) { this.httpStatus = httpStatus; }
    public int getHttpStatus() { return httpStatus; }
}

// Specific exception types
public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String resource, Object id) {
        super(ErrorCode.RESOURCE_NOT_FOUND,
              String.format("%s not found with id: %s", resource, id));
    }
}

public class ValidationException extends AppException {
    private final Map<String, String> fieldErrors;

    public ValidationException(Map<String, String> fieldErrors) {
        super(ErrorCode.VALIDATION_FAILED, "Validation failed");
        this.fieldErrors = Collections.unmodifiableMap(fieldErrors);
    }

    public Map<String, String> getFieldErrors() { return fieldErrors; }
}

public class UnauthorizedException extends AppException {
    public UnauthorizedException(String message) {
        super(ErrorCode.UNAUTHORIZED, message);
    }
}

// Usage:
throw new ResourceNotFoundException("User", userId);
throw new ValidationException(Map.of("email", "Invalid format", "age", "Must be positive"));
```

> **🏭 Production Consideration:** A unified exception hierarchy with error codes enables consistent error responses across your API. The `ErrorCode` enum maps directly to HTTP status codes, making REST controllers trivial.

---

### Using Custom Exceptions with Spring

```java
// ✅ Global exception handler in Spring
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            ex.getErrorCode().name(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity
            .status(ex.getErrorCode().getHttpStatus())
            .body(error);
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException ex) {
        ErrorResponse error = new ErrorResponse(
            ex.getErrorCode().name(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        error.setFieldErrors(ex.getFieldErrors());
        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(AppException.class)
    public ResponseEntity<ErrorResponse> handleAppException(AppException ex) {
        ErrorResponse error = new ErrorResponse(
            ex.getErrorCode().name(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity
            .status(ex.getErrorCode().getHttpStatus())
            .body(error);
    }
}

// Error response DTO
public record ErrorResponse(
    String errorCode,
    String message,
    LocalDateTime timestamp,
    Map<String, String> fieldErrors
) {
    public ErrorResponse(String errorCode, String message, LocalDateTime timestamp) {
        this(errorCode, message, timestamp, Map.of());
    }
}
```

---

### Serialization Considerations

```java
// ✅ Exceptions implement Serializable via Throwable
// Add serialVersionUID for version control
public class OrderException extends RuntimeException {

    @Serial
    private static final long serialVersionUID = 1L;

    private final String orderId;

    public OrderException(String orderId, String message) {
        super(message);
        this.orderId = orderId;
    }

    public String getOrderId() { return orderId; }
}

// ❌ Avoid non-serializable fields in exceptions
public class BadException extends RuntimeException {
    private final Connection dbConnection;  // NOT serializable!
    // This will cause issues in distributed systems (RMI, JMS)
}

// ✅ Use transient for non-serializable fields if necessary
public class SafeException extends RuntimeException {
    private transient final Connection dbConnection; // Won't be serialized
}
```

> **🎯 Interviewer Insight:** `Throwable` implements `Serializable`, so all exceptions are serializable by default. In distributed systems (RMI, JMS, Spring Cloud), non-serializable fields cause `NotSerializableException`. Always mark non-serializable fields as `transient` or avoid them entirely.

---

### Factory Methods for Clean Exception Creation

```java
public class ApiException extends RuntimeException {

    private final int statusCode;

    private ApiException(int statusCode, String message) {
        super(message);
        this.statusCode = statusCode;
    }

    private ApiException(int statusCode, String message, Throwable cause) {
        super(message, cause);
        this.statusCode = statusCode;
    }

    // ✅ Static factory methods — more readable than constructors
    public static ApiException notFound(String resource, Object id) {
        return new ApiException(404,
            String.format("%s with id '%s' not found", resource, id));
    }

    public static ApiException badRequest(String detail) {
        return new ApiException(400, "Bad request: " + detail);
    }

    public static ApiException conflict(String detail) {
        return new ApiException(409, "Conflict: " + detail);
    }

    public static ApiException internal(String detail, Throwable cause) {
        return new ApiException(500, "Internal error: " + detail, cause);
    }

    public int getStatusCode() { return statusCode; }
}

// Usage — reads like English:
throw ApiException.notFound("User", userId);
throw ApiException.badRequest("Email is required");
throw ApiException.conflict("Username already exists");
```

> **🎯 Interviewer Insight:** Factory methods make exception creation self-documenting. `throw ApiException.notFound("User", id)` is instantly readable vs `throw new ApiException(404, "User not found")`. This is the pattern used by Spring's `ResponseStatusException` and similar libraries.

---

### Anti-Patterns

```java
// ❌ Exception with no context
throw new RuntimeException(); // What went wrong?!

// ❌ Generic exception with just a string
throw new Exception("something failed"); // No type, no code, no details

// ❌ Too many custom exception classes
public class UserNotFoundException extends RuntimeException { }
public class ProductNotFoundException extends RuntimeException { }
public class OrderNotFoundException extends RuntimeException { }
// Better: use ONE ResourceNotFoundException with a resource type field

// ❌ Mutable exception fields
public class MutableException extends RuntimeException {
    private String code;
    public void setCode(String code) { this.code = code; } // BAD!
}

// ✅ Keep exception fields final (immutable)
public class ImmutableException extends RuntimeException {
    private final String code;
    public ImmutableException(String code, String msg) {
        super(msg);
        this.code = code;
    }
    public String getCode() { return code; }
}
```

> **⚠️ Common Mistake:** Creating a separate exception class for every entity (`UserNotFoundException`, `ProductNotFoundException`, `OrderNotFoundException`). This bloats the codebase. Use a parameterized exception like `ResourceNotFoundException(String resource, Object id)` instead.

---

## 🎯 Interviewer Insight

Interviewers test these specific areas:

1. **Checked vs unchecked choice** — recoverable vs programming error
2. **Four constructors** — message, cause, message+cause, no-arg
3. **Error codes** — enum-based, maps to HTTP status
4. **Serialization** — `serialVersionUID`, `transient` fields
5. **Factory methods** — readable exception creation
6. **Exception hierarchy** — base class with error code, specific subclasses

---

## ⚡ Follow-up Questions

> **⚡ Follow-up:** When would you extend `Exception` instead of `RuntimeException`?

> **⚡ Follow-up:** Why is it important to include the cause constructor?

> **⚡ Follow-up:** How do you handle exception serialization in distributed systems?

> **⚡ Follow-up:** What's the benefit of factory methods over constructors?

> **⚡ Follow-up:** How would you design a validation exception that carries field-level errors?

---

## 📚 Related Topics

- [Try-With-Resources](./04-try-with-resources.md)
- [Exception Best Practices →](./06-exception-best-practices.md)
- [Exception Chaining →](./08-exception-chaining.md)

---

[← Back to Section Index](./README.md) · [Previous: Try-With-Resources ←](./04-try-with-resources.md) · [Next: Exception Best Practices →](./06-exception-best-practices.md)

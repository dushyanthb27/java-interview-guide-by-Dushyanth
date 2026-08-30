# Q5: How to Create Custom Exceptions?

[Home](../../README.md) > [Exception Handling](./README.md) > Custom Exceptions

---

## Difficulty: 🟡 Medium

## Question
> How do you create custom exceptions in Java? What are the constructor patterns and design considerations?

---

## Answer

### Choosing the Superclass

```java
// Checked exception - caller MUST handle or declare
public class OrderNotFoundException extends Exception { }

// Unchecked exception - caller handling is OPTIONAL
public class InvalidOrderException extends RuntimeException { }
```

| Extend | When |
|--------|------|
| `Exception` | Recoverable condition; force caller to handle |
| `RuntimeException` | Programming error; bug in caller's code |

---

### Standard Constructor Patterns

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

---

### Adding Error Codes

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

// Error code enum
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

    // ✅ Static factory methods - more readable than constructors
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

// Usage - reads like English:
throw ApiException.notFound("User", userId);
throw ApiException.badRequest("Email is required");
throw ApiException.conflict("Username already exists");
```

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

---

## Interview Tips
- Always follow the 4-constructor pattern for custom exceptions
- Explain when to extend `Exception` vs `RuntimeException`
- Show how to add error codes and use enums for categorization
- Mention `serialVersionUID` for distributed system compatibility
- Demonstrate the factory method pattern for clean exception creation

---

[<< Previous: Q4 - Try-With-Resources](./q04-try-with-resources.md) | [Next: Q6 - Exception Best Practices >>](./q06-exception-best-practices.md)

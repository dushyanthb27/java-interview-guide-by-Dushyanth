# Custom Exception Template

## Overview
Exception hierarchy design and best practices for creating maintainable, informative custom exceptions.

## Use Cases
- Domain-specific error handling
- Structured error information
- Error codes and messages
- Logging and monitoring
- API error responses
- Retry logic based on exception types

## Implementation

Complete exception hierarchy with best practices.

## Code

```java
import java.io.Serial;
import java.time.LocalDateTime;
import java.util.*;

/**
 * SIMPLE VERSION: Basic Custom Exceptions
 * - Extend Exception or RuntimeException
 * - Add custom fields
 * - Provide meaningful messages
 */
public class SimpleExceptions {
    /**
     * Base custom exception
     */
    public static class ApplicationException extends Exception {
        public ApplicationException(String message) {
            super(message);
        }

        public ApplicationException(String message, Throwable cause) {
            super(message, cause);
        }
    }

    /**
     * Specific exceptions
     */
    public static class UserNotFoundException extends ApplicationException {
        public UserNotFoundException(String userId) {
            super("User not found: " + userId);
        }
    }

    public static class InvalidInputException extends ApplicationException {
        public InvalidInputException(String field, String reason) {
            super("Invalid input for field '" + field + "': " + reason);
        }
    }

    public static class DatabaseException extends ApplicationException {
        public DatabaseException(String message, Throwable cause) {
            super(message, cause);
        }
    }
}

/**
 * PRODUCTION VERSION: Advanced Exception Hierarchy
 * - Error codes
 * - Structured error information
 * - Logging integration
 * - Serialization support
 */
public class ProductionExceptions {
    /**
     * Error code enumeration
     */
    public enum ErrorCode {
        // User errors (1000-1999)
        USER_NOT_FOUND(1001, "User not found"),
        USER_ALREADY_EXISTS(1002, "User already exists"),
        INVALID_CREDENTIALS(1003, "Invalid credentials"),
        USER_INACTIVE(1004, "User account is inactive"),

        // Validation errors (2000-2999)
        INVALID_INPUT(2001, "Invalid input provided"),
        MISSING_REQUIRED_FIELD(2002, "Required field is missing"),
        INVALID_EMAIL_FORMAT(2003, "Invalid email format"),
        INVALID_PHONE_FORMAT(2004, "Invalid phone format"),

        // Database errors (3000-3999)
        DATABASE_CONNECTION_FAILED(3001, "Database connection failed"),
        DATABASE_QUERY_FAILED(3002, "Database query failed"),
        TRANSACTION_FAILED(3003, "Transaction failed"),
        CONSTRAINT_VIOLATION(3004, "Database constraint violation"),

        // Business logic errors (4000-4999)
        INSUFFICIENT_BALANCE(4001, "Insufficient balance"),
        OPERATION_NOT_ALLOWED(4002, "Operation not allowed"),
        RESOURCE_LOCKED(4003, "Resource is locked"),
        QUOTA_EXCEEDED(4004, "Quota exceeded"),

        // System errors (5000-5999)
        INTERNAL_SERVER_ERROR(5001, "Internal server error"),
        SERVICE_UNAVAILABLE(5002, "Service unavailable"),
        TIMEOUT(5003, "Operation timeout"),
        CONFIGURATION_ERROR(5004, "Configuration error");

        public final int code;
        public final String defaultMessage;

        ErrorCode(int code, String defaultMessage) {
            this.code = code;
            this.defaultMessage = defaultMessage;
        }
    }

    /**
     * Base custom exception with error code
     */
    public static class ApplicationException extends Exception {
        @Serial
        private static final long serialVersionUID = 1L;

        protected final ErrorCode errorCode;
        protected final Map<String, Object> context;
        protected final LocalDateTime timestamp;
        protected final String requestId;

        public ApplicationException(ErrorCode errorCode) {
            this(errorCode, errorCode.defaultMessage, null);
        }

        public ApplicationException(ErrorCode errorCode, String message) {
            this(errorCode, message, null);
        }

        public ApplicationException(ErrorCode errorCode, String message, 
                                   Throwable cause) {
            super(message, cause);
            this.errorCode = errorCode;
            this.context = new HashMap<>();
            this.timestamp = LocalDateTime.now();
            this.requestId = UUID.randomUUID().toString();
        }

        /**
         * Add context information
         */
        public ApplicationException withContext(String key, Object value) {
            context.put(key, value);
            return this;
        }

        /**
         * Get error code
         */
        public ErrorCode getErrorCode() {
            return errorCode;
        }

        /**
         * Get error code value
         */
        public int getErrorCodeValue() {
            return errorCode.code;
        }

        /**
         * Get context
         */
        public Map<String, Object> getContext() {
            return Collections.unmodifiableMap(context);
        }

        /**
         * Get timestamp
         */
        public LocalDateTime getTimestamp() {
            return timestamp;
        }

        /**
         * Get request ID for tracing
         */
        public String getRequestId() {
            return requestId;
        }

        /**
         * Convert to error response
         */
        public ErrorResponse toErrorResponse() {
            return new ErrorResponse(
                errorCode.code,
                getMessage(),
                requestId,
                timestamp,
                context
            );
        }

        @Override
        public String toString() {
            return "ApplicationException{" +
                    "errorCode=" + errorCode +
                    ", code=" + errorCode.code +
                    ", message='" + getMessage() + '\'' +
                    ", requestId='" + requestId + '\'' +
                    ", timestamp=" + timestamp +
                    ", context=" + context +
                    '}';
        }
    }

    /**
     * Checked exception for recoverable errors
     */
    public static class CheckedException extends ApplicationException {
        public CheckedException(ErrorCode errorCode) {
            super(errorCode);
        }

        public CheckedException(ErrorCode errorCode, String message) {
            super(errorCode, message);
        }

        public CheckedException(ErrorCode errorCode, String message, 
                               Throwable cause) {
            super(errorCode, message, cause);
        }
    }

    /**
     * Unchecked exception for programming errors
     */
    public static class UncheckedException extends RuntimeException {
        @Serial
        private static final long serialVersionUID = 1L;

        protected final ErrorCode errorCode;
        protected final Map<String, Object> context;
        protected final LocalDateTime timestamp;
        protected final String requestId;

        public UncheckedException(ErrorCode errorCode) {
            this(errorCode, errorCode.defaultMessage, null);
        }

        public UncheckedException(ErrorCode errorCode, String message) {
            this(errorCode, message, null);
        }

        public UncheckedException(ErrorCode errorCode, String message, 
                                 Throwable cause) {
            super(message, cause);
            this.errorCode = errorCode;
            this.context = new HashMap<>();
            this.timestamp = LocalDateTime.now();
            this.requestId = UUID.randomUUID().toString();
        }

        public UncheckedException withContext(String key, Object value) {
            context.put(key, value);
            return this;
        }

        public ErrorCode getErrorCode() {
            return errorCode;
        }

        public int getErrorCodeValue() {
            return errorCode.code;
        }

        public Map<String, Object> getContext() {
            return Collections.unmodifiableMap(context);
        }

        public LocalDateTime getTimestamp() {
            return timestamp;
        }

        public String getRequestId() {
            return requestId;
        }

        public ErrorResponse toErrorResponse() {
            return new ErrorResponse(
                errorCode.code,
                getMessage(),
                requestId,
                timestamp,
                context
            );
        }
    }

    /**
     * Specific exception types
     */
    public static class UserNotFoundException extends ApplicationException {
        public UserNotFoundException(String userId) {
            super(ErrorCode.USER_NOT_FOUND, 
                  "User not found: " + userId);
            withContext("userId", userId);
        }
    }

    public static class InvalidInputException extends ApplicationException {
        public InvalidInputException(String field, String reason) {
            super(ErrorCode.INVALID_INPUT, 
                  "Invalid input for field '" + field + "': " + reason);
            withContext("field", field);
            withContext("reason", reason);
        }
    }

    public static class DatabaseException extends ApplicationException {
        public DatabaseException(String message, Throwable cause) {
            super(ErrorCode.DATABASE_QUERY_FAILED, message, cause);
        }
    }

    public static class InsufficientBalanceException extends UncheckedException {
        public InsufficientBalanceException(double required, double available) {
            super(ErrorCode.INSUFFICIENT_BALANCE, 
                  "Insufficient balance. Required: " + required + 
                  ", Available: " + available);
            withContext("required", required);
            withContext("available", available);
        }
    }

    public static class TimeoutException extends UncheckedException {
        public TimeoutException(long timeoutMs, String operation) {
            super(ErrorCode.TIMEOUT, 
                  "Operation '" + operation + "' timed out after " + 
                  timeoutMs + "ms");
            withContext("timeoutMs", timeoutMs);
            withContext("operation", operation);
        }
    }

    /**
     * Error response for API
     */
    public static class ErrorResponse {
        public final int code;
        public final String message;
        public final String requestId;
        public final LocalDateTime timestamp;
        public final Map<String, Object> context;

        public ErrorResponse(int code, String message, String requestId,
                           LocalDateTime timestamp, Map<String, Object> context) {
            this.code = code;
            this.message = message;
            this.requestId = requestId;
            this.timestamp = timestamp;
            this.context = context;
        }

        @Override
        public String toString() {
            return "ErrorResponse{" +
                    "code=" + code +
                    ", message='" + message + '\'' +
                    ", requestId='" + requestId + '\'' +
                    ", timestamp=" + timestamp +
                    ", context=" + context +
                    '}';
        }
    }
}

/**
 * EXCEPTION HANDLING PATTERNS
 */
public class ExceptionHandling {
    /**
     * Retry logic based on exception type
     */
    public static class RetryableException extends RuntimeException {
        private final int maxRetries;
        private final long delayMs;

        public RetryableException(String message, int maxRetries, long delayMs) {
            super(message);
            this.maxRetries = maxRetries;
            this.delayMs = delayMs;
        }

        public int getMaxRetries() {
            return maxRetries;
        }

        public long getDelayMs() {
            return delayMs;
        }
    }

    /**
     * Retry executor
     */
    public static class RetryExecutor {
        public static <T> T executeWithRetry(
                java.util.function.Supplier<T> operation,
                int maxRetries,
                long delayMs) throws Exception {

            int attempt = 0;
            Exception lastException = null;

            while (attempt < maxRetries) {
                try {
                    return operation.get();
                } catch (RetryableException e) {
                    lastException = e;
                    attempt++;
                    if (attempt < maxRetries) {
                        Thread.sleep(delayMs);
                    }
                } catch (Exception e) {
                    // Non-retryable exception
                    throw e;
                }
            }

            throw lastException;
        }
    }

    /**
     * Exception translator
     */
    public static class ExceptionTranslator {
        public static ProductionExceptions.ApplicationException translate(
                Exception e) {

            if (e instanceof ProductionExceptions.ApplicationException) {
                return (ProductionExceptions.ApplicationException) e;
            }

            if (e instanceof IllegalArgumentException) {
                return new ProductionExceptions.ApplicationException(
                    ProductionExceptions.ErrorCode.INVALID_INPUT,
                    e.getMessage(),
                    e);
            }

            if (e instanceof java.sql.SQLException) {
                return new ProductionExceptions.ApplicationException(
                    ProductionExceptions.ErrorCode.DATABASE_QUERY_FAILED,
                    "Database error: " + e.getMessage(),
                    e);
            }

            if (e instanceof java.net.ConnectException) {
                return new ProductionExceptions.ApplicationException(
                    ProductionExceptions.ErrorCode.SERVICE_UNAVAILABLE,
                    "Service unavailable: " + e.getMessage(),
                    e);
            }

            if (e instanceof java.util.concurrent.TimeoutException) {
                return new ProductionExceptions.ApplicationException(
                    ProductionExceptions.ErrorCode.TIMEOUT,
                    "Operation timeout: " + e.getMessage(),
                    e);
            }

            // Default to internal server error
            return new ProductionExceptions.ApplicationException(
                ProductionExceptions.ErrorCode.INTERNAL_SERVER_ERROR,
                "Unexpected error: " + e.getMessage(),
                e);
        }
    }
}

/**
 * EXCEPTION HIERARCHY SUMMARY
 * 
 * Exception (Checked)
 * ├── ApplicationException (Checked)
 * │   ├── UserNotFoundException
 * │   ├── InvalidInputException
 * │   ├── DatabaseException
 * │   └── ... (other checked exceptions)
 * │
 * RuntimeException (Unchecked)
 * ├── UncheckedException
 * │   ├── InsufficientBalanceException
 * │   ├── TimeoutException
 * │   └── ... (other unchecked exceptions)
 * │
 * └── RetryableException (for retry logic)
 */
```

## Testing

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

public class CustomExceptionTest {

    @Test
    public void testSimpleException() {
        assertThrows(SimpleExceptions.UserNotFoundException.class, () -> {
            throw new SimpleExceptions.UserNotFoundException("user123");
        });
    }

    @Test
    public void testProductionExceptionWithErrorCode() {
        ProductionExceptions.UserNotFoundException ex = 
            new ProductionExceptions.UserNotFoundException("user123");

        assertEquals(ProductionExceptions.ErrorCode.USER_NOT_FOUND, 
                    ex.getErrorCode());
        assertEquals(1001, ex.getErrorCodeValue());
        assertNotNull(ex.getRequestId());
        assertNotNull(ex.getTimestamp());
    }

    @Test
    public void testExceptionContext() {
        ProductionExceptions.InvalidInputException ex = 
            new ProductionExceptions.InvalidInputException("email", 
                "Invalid format");

        ex.withContext("userId", "user123")
          .withContext("attemptCount", 3);

        assertEquals("user123", ex.getContext().get("userId"));
        assertEquals(3, ex.getContext().get("attemptCount"));
    }

    @Test
    public void testExceptionToErrorResponse() {
        ProductionExceptions.UserNotFoundException ex = 
            new ProductionExceptions.UserNotFoundException("user123");

        ProductionExceptions.ErrorResponse response = ex.toErrorResponse();

        assertEquals(1001, response.code);
        assertNotNull(response.requestId);
        assertNotNull(response.timestamp);
    }

    @Test
    public void testUncheckedExceptionInheritance() {
        assertThrows(ProductionExceptions.InsufficientBalanceException.class, 
            () -> {
                throw new ProductionExceptions.InsufficientBalanceException(
                    100.0, 50.0);
            });
    }

    @Test
    public void testExceptionTranslator() {
        Exception sqlException = new java.sql.SQLException("Connection failed");
        ProductionExceptions.ApplicationException translated = 
            ExceptionHandling.ExceptionTranslator.translate(sqlException);

        assertEquals(ProductionExceptions.ErrorCode.DATABASE_QUERY_FAILED,
                    translated.getErrorCode());
    }

    @Test
    public void testRetryExecutor() throws Exception {
        java.util.concurrent.atomic.AtomicInteger attempts = 
            new java.util.concurrent.atomic.AtomicInteger(0);

        String result = ExceptionHandling.RetryExecutor.executeWithRetry(
            () -> {
                attempts.incrementAndGet();
                if (attempts.get() < 3) {
                    throw new ExceptionHandling.RetryableException(
                        "Retry", 3, 10);
                }
                return "Success";
            },
            3,
            10
        );

        assertEquals("Success", result);
        assertEquals(3, attempts.get());
    }

    @Test
    public void testExceptionChaining() {
        Exception cause = new RuntimeException("Root cause");
        ProductionExceptions.DatabaseException ex = 
            new ProductionExceptions.DatabaseException(
                "Failed to execute query", cause);

        assertSame(cause, ex.getCause());
        assertEquals("Failed to execute query", ex.getMessage());
    }
}
```

## Performance Considerations

### Time Complexity
- **Exception creation**: O(1)
- **Context addition**: O(1) per context entry
- **Exception translation**: O(1) with switch/if-else

### Space Complexity
- **O(n)** where n = number of context entries

### Optimization Tips
1. **Lazy context**: Only add context when needed
2. **Exception pooling**: Reuse exception objects (careful!)
3. **Avoid excessive nesting**: Keep stack traces readable
4. **Efficient logging**: Log once, not multiple times
5. **Context limits**: Limit context size to prevent memory issues

## Common Pitfalls

1. **Swallowing exceptions**: Catching without logging
2. **Generic exceptions**: Using Exception instead of specific types
3. **No context**: Exceptions without helpful information
4. **Circular dependencies**: Exceptions depending on each other
5. **Memory leaks**: Holding references in exception context
6. **Thread safety**: Mutable context in concurrent scenarios
7. **Serialization issues**: Non-serializable context objects

## Best Practices

1. **Use checked exceptions for recoverable errors**
2. **Use unchecked exceptions for programming errors**
3. **Provide meaningful error messages**
4. **Include context information**
5. **Use error codes for categorization**
6. **Chain exceptions to preserve stack traces**
7. **Log exceptions at appropriate levels**
8. **Translate low-level exceptions to domain exceptions**
9. **Support retry logic for transient failures**
10. **Make exceptions serializable if needed**

## Variations

1. **Typed Exceptions**: Different exception per error type
2. **Hierarchical Exceptions**: Multiple inheritance levels
3. **Contextual Exceptions**: Rich context information
4. **Translating Exceptions**: Convert between exception types
5. **Retryable Exceptions**: Support for retry logic
6. **Async Exceptions**: Handle exceptions in async code

# Q8: SLF4J Logging Best Practices 🟢

[Home](../../README.md) > [Framework Integration](./README.md) > SLF4J Logging

---

## Overview

**SLF4J (Simple Logging Facade for Java)** is a logging abstraction that decouples your application code from the underlying logging implementation. You code against SLF4J, and swap implementations (Logback, Log4j2, JUL) without changing a single line of application code.

---

## The Facade Pattern

```
┌─────────────────────────────────────────────────────┐
│                  Your Application                   │
│               (imports org.slf4j.*)                 │
└──────────────────────┬──────────────────────────────┘
                       │ SLF4J API
                       ▼
┌──────────────────────────────────────────────────────┐
│                    SLF4J API                         │
│          (slf4j-api.jar — the facade)                │
└───────┬──────────────┬───────────────┬───────────────┘
        │              │               │
        ▼              ▼               ▼
   ┌─────────┐   ┌──────────┐   ┌───────────┐
   │ Logback │   │ Log4j2   │   │  JUL      │
   │ (native)│   │ (bridge) │   │ (bridge)  │
   └─────────┘   └──────────┘   └───────────┘
```

**Why a facade?**
- Library authors use SLF4J → library works with ANY logging framework
- Application owners choose the implementation at deployment time
- No code changes when switching implementations

---

## Log Levels

```
┌──────────┬──────────────────────────────────────────┐
│  Level   │  Purpose                                 │
├──────────┼──────────────────────────────────────────┤
│  TRACE   │  Finest-grained; method entry/exit       │
│  DEBUG   │  Diagnostic info for developers          │
│  INFO    │  Normal operational events               │
│  WARN    │  Potential problem, but recoverable      │
│  ERROR   │  Failure requiring attention             │
└──────────┴──────────────────────────────────────────┘

  TRACE < DEBUG < INFO < WARN < ERROR

  Setting level = WARN means only WARN and ERROR are logged.
```

---

## Basic Usage

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class UserService {

    // ✅ One logger per class — private static final
    private static final Logger log =
        LoggerFactory.getLogger(UserService.class);

    public User createUser(String name, String email) {
        log.info("Creating user: name={}, email={}", name, email);

        try {
            User user = repository.save(new User(name, email));
            log.debug("User persisted with id={}", user.getId());
            return user;
        } catch (DataAccessException e) {
            log.error("Failed to create user: name={}", name, e);
            throw new ServiceException("User creation failed", e);
        }
    }
}
```

```java
// ✅ With Lombok — even cleaner
@Slf4j
@Service
public class UserService {

    public User createUser(String name, String email) {
        log.info("Creating user: name={}, email={}", name, email);
        // ...
    }
}
```

---

## Parameterized Logging

```java
// ❌ Anti-pattern: string concatenation — always evaluates, even if level is off
log.debug("User " + user.getName() + " has " + orders.size() + " orders");

// ✅ Best practice: parameterized messages — arguments evaluated only if level is enabled
log.debug("User {} has {} orders", user.getName(), orders.size());

// ✅ For expensive computations, guard with isEnabled check
if (log.isDebugEnabled()) {
    log.debug("Order details: {}", order.toDetailedString());
}

// ✅ SLF4J 2.0+ fluent API (lazy evaluation built-in)
log.atDebug()
   .addArgument(() -> order.toDetailedString())  // lambda — lazy
   .log("Order details: {}");
```

**Why parameterized logging matters:**

```java
// String concatenation:
// 1. toString() called on every argument
// 2. StringBuilder allocates memory
// 3. All happens EVEN if DEBUG is disabled in production

// Parameterized logging:
// 1. Only checks if level is enabled
// 2. If disabled → no toString(), no allocation, no cost
```

---

## Logging Exceptions

```java
// ✅ Correct: exception as the LAST argument (no placeholder needed)
try {
    processPayment(order);
} catch (PaymentException e) {
    log.error("Payment failed for order {}", order.getId(), e);
    //                                                      ^^^
    //                     SLF4J automatically prints the full stack trace
}

// ❌ Anti-pattern: logging and rethrowing (duplicate stack traces)
catch (Exception e) {
    log.error("Error", e);    // logged once here
    throw e;                  // logged AGAIN by caller → noise in logs
}

// ✅ Best practice: log OR rethrow, not both
catch (Exception e) {
    throw new ServiceException("Payment processing failed", e);
    // Let the top-level handler log it once
}

// ❌ Anti-pattern: swallowing the exception
catch (Exception e) {
    log.error("Something went wrong");   // no stack trace, no context!
}

// ✅ Best practice: include the exception object
catch (Exception e) {
    log.error("Payment failed for order {}", orderId, e);
}
```

---

## MDC (Mapped Diagnostic Context)

MDC attaches contextual key-value pairs to the current thread, making them available in every log line — invaluable for tracing requests.

```java
@Component
public class RequestFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res,
                         FilterChain chain) throws IOException, ServletException {
        try {
            // ✅ Set context at the start of the request
            MDC.put("requestId", UUID.randomUUID().toString());
            MDC.put("userId", extractUserId(req));

            chain.doFilter(req, res);
        } finally {
            // ✅ Always clear MDC to prevent memory leaks in thread pools
            MDC.clear();
        }
    }
}
```

```xml
<!-- logback.xml — include MDC values in pattern -->
<pattern>
    %d{HH:mm:ss.SSS} [%thread] [%X{requestId}] [%X{userId}] %-5level %logger{36} - %msg%n
</pattern>
```

Output:
```
14:23:45.123 [http-nio-8080-exec-1] [a1b2c3d4] [user-42] INFO  UserService - Creating user
14:23:45.156 [http-nio-8080-exec-1] [a1b2c3d4] [user-42] DEBUG UserService - User persisted
```

---

## Logback Configuration

```xml
<!-- src/main/resources/logback-spring.xml -->
<configuration>

    <!-- ✅ Console appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- ✅ Rolling file appender -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/app-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>10MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- ✅ Package-level logging control -->
    <logger name="com.myapp" level="DEBUG"/>
    <logger name="org.springframework" level="WARN"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>

</configuration>
```

```yaml
# ✅ Spring Boot simplified config (application.yml)
logging:
  level:
    root: INFO
    com.myapp: DEBUG
    org.springframework: WARN
    org.hibernate.SQL: DEBUG
  file:
    name: logs/application.log
  pattern:
    console: "%d{HH:mm:ss} %-5level %logger{36} - %msg%n"
```

---

## What to Log (and What Not To)

```java
// ✅ DO log: business events
log.info("Order {} placed by user {} — total: {}", orderId, userId, total);

// ✅ DO log: external system calls
log.info("Calling payment gateway for order {}", orderId);
log.info("Payment gateway responded in {}ms — status: {}", elapsed, status);

// ✅ DO log: state transitions
log.info("Order {} status changed: {} → {}", orderId, oldStatus, newStatus);

// ❌ DON'T log: sensitive data
log.info("User login: email={}, password={}", email, password);     // NEVER!
log.info("Credit card: {}", cardNumber);                            // NEVER!

// ❌ DON'T log: inside tight loops
for (int i = 0; i < 1_000_000; i++) {
    log.debug("Processing item {}", i);  // floods logs, kills performance
}

// ✅ DO: log summary after loop
log.info("Processed {} items in {}ms", count, elapsed);
```

---

## Anti-Patterns vs Best Practices

```java
// ❌ Anti-pattern: System.out.println for logging
System.out.println("User created: " + user);

// ✅ Best practice: structured logging through SLF4J
log.info("User created: id={}, name={}", user.getId(), user.getName());

// ❌ Anti-pattern: wrong log level
log.error("User not found");        // not an error — expected business case
log.info("NullPointerException");   // this IS an error

// ✅ Best practice: appropriate levels
log.warn("User {} not found — returning empty result", userId);
log.error("Unexpected error processing order {}", orderId, exception);

// ❌ Anti-pattern: logger from wrong class
private static final Logger log = LoggerFactory.getLogger(SomeOtherClass.class);

// ✅ Best practice: always use the enclosing class (or @Slf4j)
private static final Logger log = LoggerFactory.getLogger(MyService.class);
```

---

## Choosing the Right Level

| Level | When to Use | Example |
|-------|-------------|---------|
| `ERROR` | Unexpected failure needing immediate attention | Database connection lost |
| `WARN` | Potential issue, but operation continues | Retry attempt, fallback used |
| `INFO` | Normal business operations | User registered, order placed |
| `DEBUG` | Technical details for developers | SQL query, cache hit/miss |
| `TRACE` | Ultra-detailed flow tracing | Method entry/exit, variable values |

---

## Interview Tips

| Topic | Key Point |
|-------|-----------|
| Why SLF4J? | Facade pattern — decouple code from logging implementation |
| Parameterized logging | `{}` placeholders avoid string concatenation cost when level is off |
| MDC | Thread-local context for request tracing — always clear in `finally` |
| Log levels in production | Typically `INFO` for app, `WARN` for frameworks |
| Exception logging | Pass exception as last arg; log OR rethrow, not both |

---

[← Previous: Q7 - Lombok](./q07-lombok.md) | [Back to Section Index](./README.md)

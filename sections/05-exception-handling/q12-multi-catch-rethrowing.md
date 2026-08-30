# Q12: Multi-Catch and Rethrowing Exceptions

[Home](../../README.md) > [Exception Handling](./README.md) > Multi-Catch and Rethrowing

---

## Difficulty: 🟡 Medium

## Question
> How does multi-catch work in Java 7+? What are the rules for rethrowing exceptions with more inclusive type checking?

---

## Answer

### Multi-Catch Syntax (Java 7+)

```java
// ❌ Before Java 7: duplicate catch blocks
try {
    reflectiveOperation();
} catch (ClassNotFoundException e) {
    log.error("Reflection error", e);
    throw new AppException("Reflection failed", e);
} catch (NoSuchMethodException e) {
    log.error("Reflection error", e);           // Same handling!
    throw new AppException("Reflection failed", e);
} catch (IllegalAccessException e) {
    log.error("Reflection error", e);           // Same handling!
    throw new AppException("Reflection failed", e);
}

// ✅ Java 7+: multi-catch with pipe (|) operator
try {
    reflectiveOperation();
} catch (ClassNotFoundException | NoSuchMethodException | IllegalAccessException e) {
    log.error("Reflection error", e);
    throw new AppException("Reflection failed", e);
}
```

---

### Multi-Catch Rules

#### Rule 1: Exception Types Must Be Unrelated

```java
// ✅ Unrelated exception types - valid
try {
    parse(input);
} catch (IOException | ParseException e) {
    // OK - IOException and ParseException are not related
}

// ❌ Won't compile - one is a subclass of the other
try {
    readFile(path);
} catch (FileNotFoundException | IOException e) {
    // Compile error! FileNotFoundException IS-A IOException
    // FileNotFoundException is already caught by IOException
}

// ✅ Just use the parent type
try {
    readFile(path);
} catch (IOException e) {
    // Catches both IOException and FileNotFoundException
}
```

#### Rule 2: The Variable Is Effectively Final

```java
// In multi-catch, the exception variable is implicitly FINAL

// ❌ Cannot reassign the exception variable
try {
    process();
} catch (IOException | SQLException e) {
    e = new IOException("replaced"); // Compile error! e is final
    throw e;
}

// ✅ In single-catch, reassignment is allowed (though not recommended)
try {
    process();
} catch (IOException e) {
    e = new IOException("replaced"); // Compiles (but don't do this)
    throw e;
}
```

#### Rule 3: The Variable's Type Is the Union

```java
// The type of e is the "union" (common supertype) of all listed exceptions

try {
    process();
} catch (IOException | SQLException e) {
    // e's compile-time type is: Exception (common parent)
    // But the JVM knows the exact type at runtime

    // ✅ Can call methods from the common parent (Exception/Throwable)
    String message = e.getMessage();
    Throwable cause = e.getCause();

    // ❌ Cannot call type-specific methods without casting
    // e.getSQLState(); // Compile error! Not available on the union type

    // ✅ Use instanceof if you need type-specific behavior
    if (e instanceof SQLException sql) {
        log.error("SQL State: {}", sql.getSQLState());
    }
}
```

---

### Rethrowing with More Inclusive Type Checking (Java 7+)

```java
// Java 7 introduced "more inclusive" type checking for rethrown exceptions
// The compiler tracks WHICH exceptions can actually be thrown

// ✅ Java 7+: Compiler knows exactly which exceptions can be rethrown
public void processData(String input) throws IOException, ParseException {
    try {
        readFile(input);         // throws IOException
        parseContent(input);     // throws ParseException
    } catch (Exception e) {
        // Even though we catch Exception (broad),
        // the compiler knows only IOException or ParseException can reach here
        log.error("Processing failed", e);
        throw e; // ✅ Compiler allows this - knows it's IOException | ParseException
    }
}

// In Java 6, the above would require: throws Exception
// because the compiler treated 'e' as type Exception
```

---

### Final Rethrow Pattern

```java
// ✅ The "final rethrow" pattern - catch broad, rethrow precisely

public void doWork() throws IOException, SQLException {
    try {
        step1(); // throws IOException
        step2(); // throws SQLException
        step3(); // throws IOException
    } catch (final Exception e) {
        // 'final' keyword ensures we don't reassign e
        // Compiler knows e can only be IOException or SQLException
        log.error("Work failed at some step", e);
        throw e; // ✅ Only needs to declare IOException, SQLException
    }
}

// ❌ Without 'final', if you reassign e, the compiler loses type info
public void doWorkBroken() throws Exception { // Must declare Exception!
    try {
        step1();
        step2();
    } catch (Exception e) {
        e = new RuntimeException("replaced"); // Reassignment!
        throw e; // Now compiler must assume any Exception
    }
}
```

---

### Practical Multi-Catch Examples

#### Example 1: Reflection Operations

```java
public Object invokeMethod(Object target, String methodName, Object... args) {
    try {
        Method method = target.getClass().getMethod(methodName);
        method.setAccessible(true);
        return method.invoke(target, args);
    } catch (NoSuchMethodException | IllegalAccessException |
             InvocationTargetException e) {
        throw new ReflectionException(
            "Failed to invoke " + methodName + " on " + target.getClass().getName(), e);
    }
}
```

#### Example 2: Data Parsing

```java
public Config parseConfig(String source) {
    try {
        String content = readSource(source);
        JsonNode root = objectMapper.readTree(content);
        return buildConfig(root);
    } catch (IOException | JsonProcessingException e) {
        throw new ConfigException("Cannot parse config from: " + source, e);
    } catch (IllegalArgumentException | NullPointerException e) {
        throw new ConfigException("Invalid config structure: " + source, e);
    }
}
```

#### Example 3: Resource Initialization

```java
public DataSource initializeDataSource(Properties props) {
    try {
        Class.forName(props.getProperty("driver"));
        DataSource ds = createDataSource(props);
        testConnection(ds);
        return ds;
    } catch (ClassNotFoundException | SQLException e) {
        throw new DatabaseInitException("Failed to initialize database", e);
    }
}
```

---

### Type Checking in Multi-Catch

```java
// ✅ Using instanceof with pattern matching (Java 16+)
try {
    processOrder(order);
} catch (PaymentException | InventoryException | ShippingException e) {

    // Common handling for all
    log.error("Order processing failed: {}", e.getMessage());
    order.setStatus(OrderStatus.FAILED);

    // Type-specific handling
    if (e instanceof PaymentException pe) {
        refundService.initiateRefund(pe.getTransactionId());
    } else if (e instanceof InventoryException ie) {
        inventoryService.releaseReservation(ie.getReservationId());
    } else if (e instanceof ShippingException se) {
        shippingService.cancelLabel(se.getTrackingNumber());
    }

    throw new OrderException("Order failed: " + order.getId(), e);
}
```

---

### Combining Multi-Catch with Try-With-Resources

```java
// ✅ Multi-catch works perfectly with try-with-resources
try (
    Connection conn = dataSource.getConnection();
    PreparedStatement ps = conn.prepareStatement(SQL);
    ResultSet rs = ps.executeQuery()
) {
    while (rs.next()) {
        results.add(mapRow(rs));
    }
} catch (SQLException | DataFormatException e) {
    throw new RepositoryException("Query failed: " + SQL, e);
}
```

---

### Anti-Patterns

```java
// ❌ Using multi-catch for unrelated handling
try {
    process();
} catch (IOException | NullPointerException e) {
    // These need completely different handling!
    // IOException -> retry or fallback
    // NullPointerException -> it's a bug, fix the code
    log.error("Error", e);
}

// ✅ Separate catch blocks for different handling strategies
try {
    process();
} catch (IOException e) {
    log.warn("I/O failure, using fallback", e);
    return fallbackValue;
} catch (NullPointerException e) {
    // This is a bug - fail fast
    throw new IllegalStateException("Unexpected null in processing", e);
}

// ❌ Overly broad multi-catch that hides specific types
try {
    complexOperation();
} catch (Exception e) {
    // Don't know which exception - can't handle appropriately
    handleError(e);
}

// ✅ Group only exceptions with the SAME handling strategy
try {
    complexOperation();
} catch (TimeoutException | ConnectException e) {
    // Both are connectivity issues - same retry logic
    retryWithBackoff(e);
} catch (ValidationException e) {
    // Different handling - return error to user
    return ErrorResponse.badRequest(e.getMessage());
}

// ❌ Catching and wrapping without adding value
try {
    step();
} catch (IOException e) {
    throw new RuntimeException(e); // No context added, no translation
}

// ✅ Add meaningful context when wrapping
try {
    step();
} catch (IOException e) {
    throw new FileProcessingException("Failed at step for file: " + filename, e);
}
```

---

### Quick Reference

```java
// Multi-catch syntax
catch (TypeA | TypeB | TypeC e) { }

// Rules:
// 1. Types must be unrelated (no parent-child)
// 2. Variable 'e' is implicitly final
// 3. 'e' type is the common parent (usually Exception or Throwable)
// 4. Compiler tracks actual types for precise rethrow

// Rethrowing:
// - Java 7+: compiler uses "more inclusive type checking"
// - 'final' or effectively final variable enables precise rethrow
// - Method only needs to declare the specific exceptions, not Exception
```

---

## Interview Tips
- Know the pipe (`|`) syntax and the rules (unrelated types, effectively final variable)
- Explain "more inclusive type checking" for rethrowing - a Java 7 improvement
- Demonstrate the `final` rethrow pattern for precise exception declarations
- Show when to use multi-catch (same handling) vs separate blocks (different handling)
- Mention that multi-catch generates more efficient bytecode than duplicate blocks

---

[<< Previous: Q11 - Exception Performance](./q11-exception-performance.md) | [Back to Section Index](./README.md)

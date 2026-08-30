# Exception Handling - Cheat Sheet

Complete guide to Java exception handling with patterns, best practices, and examples.

## 🏗️ Exception Hierarchy

```
Throwable
├── Error (JVM errors - don't catch)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── VirtualMachineError
├── Exception (recoverable)
│   ├── Checked Exceptions (must handle)
│   │   ├── IOException
│   │   ├── SQLException
│   │   ├── ClassNotFoundException
│   │   └── InterruptedException
│   └── Unchecked Exceptions (optional)
│       ├── RuntimeException
│       │   ├── NullPointerException
│       │   ├── IllegalArgumentException
│       │   ├── IndexOutOfBoundsException
│       │   ├── ClassCastException
│       │   ├── ConcurrentModificationException
│       │   └── UnsupportedOperationException
│       └── Other RuntimeExceptions
```

## 🎯 Try-Catch-Finally

### Basic Structure
```java
try {
    // Code that might throw exception
    int result = 10 / 0;
} catch (ArithmeticException e) {
    // Handle specific exception
    System.out.println("Cannot divide by zero");
} catch (Exception e) {
    // Catch-all (less specific)
    System.out.println("Unexpected error");
} finally {
    // Always executes (optional)
    System.out.println("Cleanup");
}
```

### Multiple Catch Blocks
```java
try {
    // Risky code
} catch (FileNotFoundException e) {
    // Handle file not found
} catch (IOException e) {
    // Handle other IO errors
} catch (Exception e) {
    // Catch-all for other exceptions
}

// Order matters: specific to general
// ❌ Wrong - unreachable catch
try {
    // code
} catch (Exception e) {
    // Catches everything
} catch (IOException e) {
    // Unreachable!
}
```

### Multi-Catch (Java 7+)
```java
try {
    // Risky code
} catch (FileNotFoundException | IOException e) {
    // Handle multiple exception types
    System.out.println("IO error: " + e.getMessage());
}

// Useful for similar handling
try {
    // code
} catch (IllegalArgumentException | IllegalStateException e) {
    System.out.println("Invalid state: " + e.getMessage());
}
```

### Try-With-Resources (Java 7+)
```java
// Automatically closes resources
try (FileReader reader = new FileReader("file.txt");
     BufferedReader br = new BufferedReader(reader)) {
    String line = br.readLine();
} catch (IOException e) {
    System.out.println("Error reading file: " + e.getMessage());
}

// Multiple resources
try (FileInputStream fis = new FileInputStream("file.txt");
     FileOutputStream fos = new FileOutputStream("output.txt")) {
    // Use resources
} catch (IOException e) {
    // Handle exception
}

// Custom AutoCloseable
class Resource implements AutoCloseable {
    @Override
    public void close() throws Exception {
        System.out.println("Closing resource");
    }
}

try (Resource res = new Resource()) {
    // Use resource
} catch (Exception e) {
    // Handle exception
}

// Benefits:
// - Automatic resource closing
// - Suppressed exceptions handled
// - Cleaner code
```

## 🚨 Exception Handling Patterns

### Throw and Catch
```java
public void validateAge(int age) throws IllegalArgumentException {
    if (age < 0) {
        throw new IllegalArgumentException("Age cannot be negative");
    }
}

try {
    validateAge(-5);
} catch (IllegalArgumentException e) {
    System.out.println("Invalid input: " + e.getMessage());
}
```

### Wrap and Rethrow
```java
public void processFile(String filename) throws CustomException {
    try {
        FileReader reader = new FileReader(filename);
        // Process file
    } catch (IOException e) {
        // Wrap checked exception as unchecked
        throw new CustomException("Failed to process file", e);
    }
}

// Caller doesn't need to handle checked exception
processFile("file.txt");  // No try-catch needed
```

### Log and Continue
```java
for (String item : items) {
    try {
        processItem(item);
    } catch (Exception e) {
        logger.error("Failed to process item: " + item, e);
        // Continue with next item
    }
}
```

### Log and Rethrow
```java
try {
    riskyOperation();
} catch (Exception e) {
    logger.error("Operation failed", e);
    throw e;  // Rethrow after logging
}
```

### Default Value on Exception
```java
public int parseInteger(String value) {
    try {
        return Integer.parseInt(value);
    } catch (NumberFormatException e) {
        return 0;  // Default value
    }
}

// Or with Optional
public Optional<Integer> parseInteger(String value) {
    try {
        return Optional.of(Integer.parseInt(value));
    } catch (NumberFormatException e) {
        return Optional.empty();
    }
}
```

## 🏭 Custom Exceptions

### Simple Custom Exception
```java
public class InvalidAgeException extends Exception {
    public InvalidAgeException(String message) {
        super(message);
    }
    
    public InvalidAgeException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Usage
throw new InvalidAgeException("Age must be between 0 and 150");
```

### Custom Unchecked Exception
```java
public class InvalidConfigurationException extends RuntimeException {
    public InvalidConfigurationException(String message) {
        super(message);
    }
    
    public InvalidConfigurationException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Usage - no need to declare in throws
throw new InvalidConfigurationException("Config file not found");
```

### Exception with Custom Fields
```java
public class PaymentException extends Exception {
    private final String transactionId;
    private final BigDecimal amount;
    
    public PaymentException(String message, String transactionId, BigDecimal amount) {
        super(message);
        this.transactionId = transactionId;
        this.amount = amount;
    }
    
    public String getTransactionId() {
        return transactionId;
    }
    
    public BigDecimal getAmount() {
        return amount;
    }
}

// Usage
try {
    processPayment(amount);
} catch (PaymentException e) {
    logger.error("Payment failed for transaction: " + e.getTransactionId());
}
```

## 📋 Common Exception Types

### Checked Exceptions (Must Handle)
```java
// IOException
try {
    FileReader reader = new FileReader("file.txt");
} catch (IOException e) {
    System.out.println("File not found or cannot read");
}

// SQLException
try {
    Connection conn = DriverManager.getConnection(url);
} catch (SQLException e) {
    System.out.println("Database connection failed");
}

// InterruptedException
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}

// ClassNotFoundException
try {
    Class.forName("com.example.MyClass");
} catch (ClassNotFoundException e) {
    System.out.println("Class not found");
}
```

### Unchecked Exceptions (Optional)
```java
// NullPointerException
String str = null;
str.length();  // NullPointerException

// IllegalArgumentException
if (age < 0) {
    throw new IllegalArgumentException("Age cannot be negative");
}

// IndexOutOfBoundsException
List<String> list = new ArrayList<>();
list.get(10);  // IndexOutOfBoundsException

// ClassCastException
Object obj = "string";
Integer num = (Integer) obj;  // ClassCastException

// ConcurrentModificationException
List<String> list = new ArrayList<>();
for (String item : list) {
    list.remove(item);  // ConcurrentModificationException
}

// UnsupportedOperationException
List<String> list = Collections.unmodifiableList(new ArrayList<>());
list.add("item");  // UnsupportedOperationException
```

## 🎯 Best Practices

### 1. Catch Specific Exceptions
```java
// ❌ Wrong - too broad
try {
    // code
} catch (Exception e) {
    System.out.println("Error");
}

// ✅ Correct - specific
try {
    // code
} catch (FileNotFoundException e) {
    System.out.println("File not found");
} catch (IOException e) {
    System.out.println("IO error");
}
```

### 2. Don't Ignore Exceptions
```java
// ❌ Wrong - silent failure
try {
    riskyOperation();
} catch (Exception e) {
    // Ignored
}

// ✅ Correct - log or handle
try {
    riskyOperation();
} catch (Exception e) {
    logger.error("Operation failed", e);
    throw new RuntimeException("Operation failed", e);
}
```

### 3. Provide Context
```java
// ❌ Wrong - no context
throw new Exception("Error");

// ✅ Correct - meaningful message
throw new InvalidAgeException("Age must be between 0 and 150, got: " + age);
```

### 4. Include Cause
```java
// ❌ Wrong - loses original exception
try {
    readFile();
} catch (IOException e) {
    throw new RuntimeException("Failed to read file");
}

// ✅ Correct - preserves stack trace
try {
    readFile();
} catch (IOException e) {
    throw new RuntimeException("Failed to read file", e);
}
```

### 5. Use Finally for Cleanup
```java
FileReader reader = null;
try {
    reader = new FileReader("file.txt");
    // Process file
} catch (IOException e) {
    System.out.println("Error reading file");
} finally {
    if (reader != null) {
        try {
            reader.close();
        } catch (IOException e) {
            System.out.println("Error closing file");
        }
    }
}

// Better: Use try-with-resources
try (FileReader reader = new FileReader("file.txt")) {
    // Process file
} catch (IOException e) {
    System.out.println("Error reading file");
}
```

### 6. Don't Catch Throwable or Error
```java
// ❌ Wrong - catches Error too
try {
    // code
} catch (Throwable t) {
    // Handles OutOfMemoryError, StackOverflowError, etc.
}

// ✅ Correct - only catch Exception
try {
    // code
} catch (Exception e) {
    // Handles recoverable exceptions
}
```

## 🔄 Exception Translation

### Wrapping Checked as Unchecked
```java
public List<String> readLines(String filename) {
    try {
        return Files.readAllLines(Paths.get(filename));
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}

// Caller doesn't need try-catch
List<String> lines = readLines("file.txt");
```

### Chaining Exceptions
```java
try {
    processData();
} catch (DataProcessingException e) {
    throw new ApplicationException("Failed to process data", e);
}

// Stack trace shows both exceptions
```

## 📊 Exception Handling Patterns Table

| Pattern | Use Case | Example |
|---------|----------|---------|
| Try-Catch | Handle expected exceptions | File I/O, parsing |
| Try-Finally | Cleanup resources | Close connections |
| Try-With-Resources | Auto-close resources | File operations |
| Throw | Signal error to caller | Validation failures |
| Wrap & Rethrow | Add context | Translate exception types |
| Log & Continue | Non-critical errors | Batch processing |
| Default Value | Graceful degradation | Parsing with fallback |

## ⚠️ Common Pitfalls

1. **Swallowing exceptions**
   ```java
   // ❌ Wrong
   try {
       riskyOperation();
   } catch (Exception e) {
       // Ignored
   }
   
   // ✅ Correct
   try {
       riskyOperation();
   } catch (Exception e) {
       logger.error("Operation failed", e);
   }
   ```

2. **Catching too broad**
   ```java
   // ❌ Wrong
   try {
       // code
   } catch (Exception e) {
       // Catches everything including programming errors
   }
   
   // ✅ Correct
   try {
       // code
   } catch (SpecificException e) {
       // Only catches expected exceptions
   }
   ```

3. **Not preserving cause**
   ```java
   // ❌ Wrong
   catch (IOException e) {
       throw new RuntimeException("Failed");  // Loses original exception
   }
   
   // ✅ Correct
   catch (IOException e) {
       throw new RuntimeException("Failed", e);  // Preserves stack trace
   }
   ```

## 🎓 Interview Tips

- **Know the hierarchy**: Checked vs unchecked exceptions
- **Understand try-with-resources**: Automatic resource management
- **Use specific exceptions**: Catch what you expect
- **Preserve context**: Include cause and meaningful messages
- **Avoid silent failures**: Always log or handle exceptions
- **Know common exceptions**: NullPointerException, IOException, etc.

---

**Related**: See Testing cheat sheet for testing exception handling

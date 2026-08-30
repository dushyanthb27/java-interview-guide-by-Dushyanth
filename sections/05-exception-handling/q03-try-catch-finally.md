# Q3: How Does Try-Catch-Finally Work?

[Home](../../README.md) > [Exception Handling](./README.md) > Try-Catch-Finally

---

## Difficulty: 🟢 Easy

## Question
> How does the try-catch-finally mechanism work in Java? What are the execution flow rules and common gotchas?

---

## Answer

### Basic Structure

```java
try {
    // Code that may throw an exception
    riskyOperation();
} catch (SpecificException e) {
    // Handle the specific exception
    handleError(e);
} finally {
    // ALWAYS executes (with rare exceptions)
    cleanup();
}
```

---

### Execution Flow

#### Path 1: No Exception Thrown

```java
public void normalFlow() {
    System.out.println("1 - Before try");
    try {
        System.out.println("2 - Inside try");
        // No exception here
    } catch (Exception e) {
        System.out.println("X - SKIPPED");
    } finally {
        System.out.println("3 - Finally block");
    }
    System.out.println("4 - After try-catch-finally");
}
// Output: 1, 2, 3, 4
```

#### Path 2: Exception Caught

```java
public void caughtExceptionFlow() {
    System.out.println("1 - Before try");
    try {
        System.out.println("2 - Inside try");
        int result = 10 / 0; // ArithmeticException
        System.out.println("X - SKIPPED (after exception)");
    } catch (ArithmeticException e) {
        System.out.println("3 - Exception caught: " + e.getMessage());
    } finally {
        System.out.println("4 - Finally block");
    }
    System.out.println("5 - After try-catch-finally");
}
// Output: 1, 2, 3, 4, 5
```

#### Path 3: Exception NOT Caught

```java
public void uncaughtExceptionFlow() {
    System.out.println("1 - Before try");
    try {
        System.out.println("2 - Inside try");
        int result = 10 / 0; // ArithmeticException
    } catch (NullPointerException e) {
        System.out.println("X - SKIPPED (wrong type)");
    } finally {
        System.out.println("3 - Finally still runs!");
    }
    System.out.println("X - SKIPPED (exception propagates)");
}
// Output: 1, 2, 3 ... then ArithmeticException propagates up
```

---

### Multiple Catch Blocks - Order Matters

```java
// ✅ Correct: Most specific first
try {
    readFile("data.txt");
} catch (FileNotFoundException e) {
    System.out.println("File not found");       // Specific
} catch (IOException e) {
    System.out.println("I/O error");            // Broader
} catch (Exception e) {
    System.out.println("Unexpected error");     // Broadest
}

// ❌ Won't compile: Broader type before specific type
try {
    readFile("data.txt");
} catch (IOException e) {
    System.out.println("I/O error");
} catch (FileNotFoundException e) {         // Compile error!
    // FileNotFoundException is already caught by IOException above
    System.out.println("File not found");
}
```

---

### Finally Always Executes (Almost)

```java
// ✅ Finally runs even when exception is thrown
try {
    throw new RuntimeException("boom");
} finally {
    System.out.println("Finally runs!"); // This WILL execute
}

// ✅ Finally runs even when return is in try
public int getNumber() {
    try {
        return 42;
    } finally {
        System.out.println("Finally runs before return!");
    }
}

// ✅ Finally runs even when return is in catch
public int getNumber() {
    try {
        throw new Exception();
    } catch (Exception e) {
        return -1;
    } finally {
        System.out.println("Finally runs before catch's return!");
    }
}
```

#### When Finally Does NOT Execute

```java
// 1. System.exit() is called
try {
    System.exit(0); // JVM terminates immediately
} finally {
    System.out.println("NEVER printed");
}

// 2. JVM crashes (OutOfMemoryError, etc.)
// 3. Thread is killed (Thread.stop() - deprecated)
// 4. Infinite loop or deadlock in try block
try {
    while (true) { } // Finally never reached
} finally {
    System.out.println("NEVER printed");
}
```

---

### Return Value Gotcha in Finally (Classic Interview Question)

```java
// ❌ GOTCHA: return in finally OVERRIDES return in try/catch
public int trickyReturn() {
    try {
        return 1;       // This value is prepared...
    } finally {
        return 2;       // ...but THIS value is actually returned!
    }
}
// Returns 2, NOT 1

// ❌ GOTCHA: finally return swallows exceptions
public int silentException() {
    try {
        throw new RuntimeException("Error!");  // Exception thrown
    } finally {
        return 42;   // Exception is SWALLOWED, 42 is returned
    }
}
// Returns 42 - RuntimeException is completely lost!

// ✅ Best Practice: NEVER return from finally
public int correctApproach() {
    int result;
    try {
        result = computeValue();
    } catch (Exception e) {
        result = defaultValue();
    } finally {
        cleanup(); // No return here!
    }
    return result;
}
```

---

### Nested Try-Catch

```java
public void nestedTryCatch() {
    try {
        System.out.println("Outer try");

        try {
            System.out.println("Inner try");
            int result = 10 / 0;
        } catch (ArithmeticException e) {
            System.out.println("Inner catch: " + e.getMessage());
            // Exception is handled here; outer catch is NOT triggered
        } finally {
            System.out.println("Inner finally");
        }

        System.out.println("Outer try continues (inner exception was handled)");

    } catch (Exception e) {
        System.out.println("Outer catch");
    } finally {
        System.out.println("Outer finally");
    }
}
// Output:
// Outer try
// Inner try
// Inner catch: / by zero
// Inner finally
// Outer try continues (inner exception was handled)
// Outer finally
```

---

### Try-Finally Without Catch

```java
// ✅ Valid syntax - ensures cleanup even without handling the exception
public void lockExample() {
    Lock lock = new ReentrantLock();
    lock.lock();
    try {
        // Critical section
        modifySharedState();
    } finally {
        lock.unlock(); // Always release the lock
    }
    // Any exception propagates up, but lock is always released
}
```

---

### Exception in Finally Block

```java
// ❌ GOTCHA: Exception in finally replaces the original exception
public void finallyException() throws Exception {
    try {
        throw new IOException("Original error");
    } finally {
        throw new IllegalStateException("Finally error");
        // IOException is LOST! Only IllegalStateException propagates
    }
}

// ✅ Guard against exceptions in finally
public void safeFinally() throws IOException {
    InputStream is = null;
    try {
        is = new FileInputStream("data.txt");
        // process stream
    } finally {
        if (is != null) {
            try {
                is.close();
            } catch (IOException e) {
                // Log but don't throw - preserves original exception
                log.warn("Failed to close stream", e);
            }
        }
    }
}

// ✅ Better approach: use try-with-resources (see Q4)
```

---

### Variable Scope in Try-Catch

```java
// ❌ Variable declared inside try is NOT accessible outside
try {
    String data = readFile("input.txt");
} catch (IOException e) {
    // handle
}
System.out.println(data); // Compile error! 'data' out of scope

// ✅ Declare variable before try block
String data = null;
try {
    data = readFile("input.txt");
} catch (IOException e) {
    data = "default";
}
System.out.println(data); // Works!
```

---

### Anti-Patterns Summary

```java
// ❌ Empty catch block (swallowing exceptions)
try { riskyOp(); } catch (Exception e) { }

// ❌ Return in finally
try { return 1; } finally { return 2; }

// ❌ Catching Exception/Throwable
try { specificOp(); } catch (Throwable t) { }

// ❌ Using try-catch for flow control
try {
    int value = Integer.parseInt(input);
} catch (NumberFormatException e) {
    // Using exception as an if-else - expensive!
}

// ✅ Prefer validation over exception-based flow control
if (input.matches("-?\\d+")) {
    int value = Integer.parseInt(input);
}
```

---

## Interview Tips
- Know all three execution paths: no exception, caught, uncaught
- Explain the return-in-finally gotcha - a favorite interview question
- Mention that `System.exit()` prevents `finally` from running
- Emphasize: catch blocks must be ordered from most specific to most general
- Recommend try-with-resources over try-finally for resource management

---

[<< Previous: Q2 - Checked vs Unchecked](./q02-checked-vs-unchecked.md) | [Next: Q4 - Try-With-Resources >>](./q04-try-with-resources.md)

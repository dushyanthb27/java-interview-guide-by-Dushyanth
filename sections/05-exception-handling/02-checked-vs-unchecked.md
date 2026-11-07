# Q2 🟢 What's the Difference Between Checked and Unchecked Exceptions?

**Question:** What is the difference between checked and unchecked exceptions in Java? When should you use each?

---

## Answer

### Side-by-Side Comparison

| Feature | Checked Exception | Unchecked Exception |
|---------|-------------------|---------------------|
| **Superclass** | `Exception` (not `RuntimeException`) | `RuntimeException` |
| **Compile-time check** | ✅ Yes — must handle or declare | ❌ No — optional handling |
| **Keyword** | `throws` required in method signature | `throws` not required |
| **Typical cause** | External/recoverable conditions | Programming errors/bugs |
| **Examples** | `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |

---

### Checked Exceptions — Must Handle or Declare

```java
// ✅ Option 1: Handle with try-catch
public String readFile(String path) {
    try {
        return Files.readString(Path.of(path));
    } catch (IOException e) {
        return "default content";
    }
}

// ✅ Option 2: Declare with throws
public String readFile(String path) throws IOException {
    return Files.readString(Path.of(path));
}

// ❌ Won't compile — checked exception not handled or declared
public String readFile(String path) {
    return Files.readString(Path.of(path)); // Compilation error!
}
```

### Unchecked Exceptions — No Handling Required

```java
// ✅ Compiles fine — no try-catch or throws needed
public int divide(int a, int b) {
    return a / b; // May throw ArithmeticException — no handling required
}

// ✅ Can optionally handle unchecked exceptions
public int divide(int a, int b) {
    try {
        return a / b;
    } catch (ArithmeticException e) {
        throw new IllegalArgumentException("Divisor cannot be zero", e);
    }
}
```

---

### When to Use Each

#### Use Checked Exceptions When: The Caller Can Reasonably Recover

```java
// ✅ File operations — caller can use a default or retry
public Config loadConfig(String path) throws ConfigNotFoundException {
    File file = new File(path);
    if (!file.exists()) {
        throw new ConfigNotFoundException("Config not found: " + path);
    }
    return parseConfig(file);
}

// ✅ Network operations — caller can retry or use fallback
public UserData fetchUser(String id) throws ServiceUnavailableException {
    Response response = httpClient.get("/users/" + id);
    if (response.status() == 503) {
        throw new ServiceUnavailableException("Service down");
    }
    return response.body(UserData.class);
}
```

#### Use Unchecked Exceptions When: The Problem Indicates a Programming Bug

```java
// ✅ Precondition violation — caller passed bad data
public void setAge(int age) {
    if (age < 0 || age > 150) {
        throw new IllegalArgumentException("Invalid age: " + age);
    }
    this.age = age;
}

// ✅ Illegal state — method called at wrong time
public void withdraw(BigDecimal amount) {
    if (this.closed) {
        throw new IllegalStateException("Account is closed");
    }
    this.balance = this.balance.subtract(amount);
}

// ✅ Null checks — fail fast on invalid input
public void process(Order order) {
    Objects.requireNonNull(order, "Order must not be null");
    // ...
}
```

> **🎯 Interviewer Insight:** The key distinction: **checked = "caller must deal with this" (recoverable)**, **unchecked = "you wrote buggy code" (programming error)**. 

If the caller can meaningfully recover (retry, use fallback, show user-friendly error), use checked. If it's a bug (null, bad argument, wrong state), use unchecked. The fix is in the code, not in exception handling.

---

### The Checked Exceptions Debate

#### Arguments Against Checked Exceptions

```java
// Problem 1: Catch-and-swallow anti-pattern
// Developers get lazy and write:
try {
    riskyOperation();
} catch (Exception e) {
    // ❌ Swallowed! Bug hiding in plain sight
}

// Problem 2: Throws clause pollution
// Every method in the call chain must declare throws
public void methodA() throws IOException, SQLException {
    methodB();
}
public void methodB() throws IOException, SQLException {
    methodC();
}
public void methodC() throws IOException, SQLException {
    // actual work
}

// Problem 3: Incompatible with functional interfaces
// Won't compile — Function doesn't declare throws
Function<String, String> reader = path -> Files.readString(Path.of(path));
```

#### Arguments For Checked Exceptions

```java
// ✅ Self-documenting API — callers KNOW what can go wrong
public Connection connect(String url) throws ConnectionException {
    // The method signature tells the story
}

// ✅ Forces consideration of failure scenarios
// The compiler ensures you at least acknowledge the exception
```

> **🏭 Production Consideration:** Checked exceptions force API consumers to acknowledge failure modes. In large teams, this prevents "it works on my machine" surprises. However, they create boilerplate.

Modern frameworks (Spring) and languages (Kotlin) favor unchecked exceptions with good documentation.

---

### How Spring Framework Handles This

```java
// Spring wraps checked exceptions in unchecked equivalents

// JDBC: SQLException (checked) → DataAccessException (unchecked)
// JPA:  PersistenceException is already unchecked
// I/O:  IOException (checked) → UncategorizedIOException (unchecked)

// ❌ Traditional JDBC
public User findById(long id) throws SQLException {
    PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
    // ...
}

// ✅ Spring JDBC
public User findById(long id) {
    // DataAccessException is unchecked — no throws clause needed
    return jdbcTemplate.queryForObject(
        "SELECT * FROM users WHERE id = ?",
        userRowMapper,
        id
    );
}
```

> **🎯 Interviewer Insight:** Spring's approach is pragmatic. Wrap checked exceptions in unchecked ones at the framework boundary. This gives you the best of both worlds: the framework handles the boilerplate, and your business logic stays clean. 

Know this pattern. It's widely used in enterprise Java.

---

### Kotlin's Approach — No Checked Exceptions

```kotlin
// Kotlin does NOT have checked exceptions at the language level
// All exceptions are unchecked

// No throws clause needed — compiles fine
fun readFile(path: String): String {
    return File(path).readText() // IOException is unchecked in Kotlin
}

// @Throws annotation only for Java interop
@Throws(IOException::class)
fun readFileForJava(path: String): String {
    return File(path).readText()
}
```

---

### Converting Between Checked and Unchecked

```java
// ✅ Wrapping checked in unchecked (common pattern)
public String loadData(String path) {
    try {
        return Files.readString(Path.of(path));
    } catch (IOException e) {
        throw new UncheckedIOException("Failed to load: " + path, e);
        // UncheckedIOException is a JDK class since Java 8
    }
}

// ✅ Generic utility for wrapping
@FunctionalInterface
public interface CheckedSupplier<T> {
    T get() throws Exception;
}

public static <T> T uncheckCall(CheckedSupplier<T> supplier) {
    try {
        return supplier.get();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}

// Usage:
String content = uncheckCall(() -> Files.readString(Path.of("data.txt")));
```

---

### Common Anti-Patterns

```java
// ❌ Catching Exception instead of specific type
try {
    parseAndSave(data);
} catch (Exception e) {
    log.error("Error", e); // Which exception? IOException? NPE?
}

// ❌ Declaring throws Exception
public void doWork() throws Exception {  // Too vague!
    // ...
}

// ❌ Wrapping without preserving cause
try {
    readConfig();
} catch (IOException e) {
    throw new RuntimeException("Config error"); // Lost the cause!
}

// ✅ Always preserve the cause
try {
    readConfig();
} catch (IOException e) {
    throw new RuntimeException("Config error", e); // Cause preserved
}
```

> **⚠️ Common Mistake:** Wrapping an exception without passing the cause (`throw new RuntimeException("msg")` instead of `throw new RuntimeException("msg", e)`). This destroys the stack trace. It makes debugging production issues nearly impossible. 

**Always preserve the cause.**

---

### Quick Decision Guide

```
Is the problem caused by the CALLER passing bad data?
  → Yes → Use unchecked (IllegalArgumentException)

Is the problem a programming bug (null, bad cast)?
  → Yes → Use unchecked (NullPointerException, etc.)

Is the problem external and potentially recoverable?
  → Yes → Use checked (IOException, etc.)
  → ... but are you building a framework/library?
      → Consider unchecked with good documentation
```

---

## 🎯 Interviewer Insight

Interviewers want you to demonstrate understanding of the exception hierarchy (`Exception` → checked, `RuntimeException` → unchecked) and the practical implications. Strong candidates mention:

- The ongoing debate around checked exceptions (Spring/Kotlin perspective)
- That checked exceptions force API consumers to handle failure cases
- The `UncheckedIOException` wrapper class added in Java 8
- How to handle checked exceptions in functional interfaces and lambdas

Be prepared to argue **both sides** of the checked exceptions debate.

---

## ⚡ Follow-up Questions

> **⚡ Follow-up:** Why doesn't Kotlin have checked exceptions, and what are the arguments for and against this design?

> **⚡ Follow-up:** When would you create a custom checked vs unchecked exception, and what are the naming conventions?

> **⚡ Follow-up:** How does Spring handle checked exceptions in its data access layer, and what is the benefit of this approach?

> **⚡ Follow-up:** What is the difference between `UncheckedIOException` and `RuntimeException` for wrapping checked exceptions?

> **⚡ Follow-up:** How do you handle checked exceptions in functional interfaces and lambda expressions?

---

## 📚 Related Topics

- [Exception Hierarchy](./01-exception-hierarchy.md)
- [Try-Catch-Finally →](./03-try-catch-finally.md)
- [Exception Best Practices →](./06-exception-best-practices.md)

\n
### ⚠️ Gotchas

- **Memory Leaks & Object Reference Retention**: Unintentional object retention via static references, unclosed resource handles, or long-lived caches preventing Garbage Collection.
- **Concurrency & Thread Safety**: Assuming thread safety on standard collections (`ArrayList`, `HashMap`) or primitives, leading to race conditions and data corruption.
- **Exception Masking & Swallowing**: Catching generic `Exception` or `Throwable` without proper logging, rethrowing, or handling suppressed exceptions in try-with-resources.
- **Performance Antipatterns**: Excessive object allocation in tight loops, improper stream boxing/unboxing overhead, or inefficient synchronization blocks causing thread contention.
\n---\n

[← Back to Section Index](./README.md) · [Previous: Exception Hierarchy ←](./01-exception-hierarchy.md) · [Next: Try-Catch-Finally →](./03-try-catch-finally.md)

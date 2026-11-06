# Interfaces vs Abstract Classes

[Home](../../README.md) > [Object-Oriented Programming](./README.md) > Interfaces vs Abstract Classes

---

## Q10 🟡 Interfaces vs Abstract Classes — When to Use Each?

**Question** — What are the differences between interfaces and abstract classes in Java? When should you use each, and how have interfaces evolved since Java 8?

### Answer

Both interfaces and abstract classes provide abstraction, but they serve different design purposes and have distinct capabilities. Choose based on whether you need to define a contract (CAN-DO) or a base type (IS-A), whether you need state, and whether you need multiple inheritance.

**Comprehensive Comparison**

| Feature | Interface | Abstract Class |
|---------|-----------|---------------|
| Instantiation | ❌ Cannot | ❌ Cannot |
| Constructors | ❌ No | ✅ Yes |
| State (instance fields) | ❌ Only `public static final` constants | ✅ Any fields |
| Abstract methods | ✅ Yes (implicitly `public abstract`) | ✅ Yes |
| Concrete methods | ✅ `default`, `static`, `private` (Java 8/9+) | ✅ Yes |
| Multiple inheritance | ✅ A class can implement many | ❌ A class can extend only one |
| Access modifiers on methods | `public` (or `private` in Java 9+) | Any modifier |
| Inheritance keyword | `implements` | `extends` |
| Purpose | Define a **contract** (CAN-DO) | Define a **base type** (IS-A) |

**When to Use Interfaces**

Use interfaces to define a capability or contract, especially when you need multiple inheritance of type. A class can implement many interfaces but extend only one abstract class.

```java
// USE INTERFACE: Define a capability/contract
public interface Sortable<T> {
    int compareTo(T other);
}

// USE INTERFACE: Multiple inheritance of type
public interface Flyable {
    void fly();
}

public interface Swimmable {
    void swim();
}

// A Duck CAN fly AND CAN swim
public class Duck implements Flyable, Swimmable {
    @Override
    public void fly() { System.out.println("Duck flying"); }

    @Override
    public void swim() { System.out.println("Duck swimming"); }
}
```

**When to Use Abstract Classes**

Use abstract classes when you need shared state, partial implementation, or constructor initialization. They provide a template method pattern with common behavior that subclasses can customize.

```java
// USE ABSTRACT CLASS: Shared state + partial implementation
public abstract class AbstractHttpHandler {
    // Shared state
    protected final Logger logger = LoggerFactory.getLogger(getClass());
    protected final ObjectMapper mapper = new ObjectMapper();
    private int requestCount = 0;

    // Constructor for initialization
    protected AbstractHttpHandler() {
        logger.info("Handler initialized: {}", getClass().getSimpleName());
    }

    // Template method - defines the algorithm skeleton
    public final Response handle(Request request) {
        requestCount++;
        validate(request);
        logger.info("Processing request #{}", requestCount);
        Response response = doHandle(request);
        postProcess(response);
        return response;
    }

    // Shared concrete behavior
    protected void validate(Request request) {
        if (request == null) throw new IllegalArgumentException("Request is null");
    }

    protected void postProcess(Response response) {
        response.addHeader("X-Request-Count", String.valueOf(requestCount));
    }

    // Abstract hook - subclass provides specific logic
    protected abstract Response doHandle(Request request);
}

public class UserHandler extends AbstractHttpHandler {
    @Override
    protected Response doHandle(Request request) {
        // Only implement the specific part
        return new Response(200, "User data");
    }
}
```

**Java 8+ Interface Evolution**

Interfaces have evolved significantly. Java 8 added default and static methods, Java 9 added private methods for code reuse, and Java 17 added sealed interfaces for controlled implementation.

```java
// Pre-Java 8: interfaces could only have abstract methods
public interface OldStyle {
    void doSomething();
}

// Java 8+: default and static methods
public interface Collection<E> {
    // Abstract method (traditional)
    boolean add(E element);

    // Default method (Java 8) - provides implementation
    default void forEach(Consumer<? super E> action) {
        for (E element : this) {
            action.accept(element);
        }
    }

    // Static method (Java 8) - utility
    static <T> Collection<T> empty() {
        return List.of();
    }
}

// Java 9+: private methods for code reuse within default methods
public interface Loggable {
    default void logInfo(String message) {
        log("INFO", message);
    }

    default void logError(String message) {
        log("ERROR", message);
    }

    // Private helper - reduces code duplication in default methods
    private void log(String level, String message) {
        System.out.printf("[%s] %s: %s%n", level, getClass().getSimpleName(), message);
    }
}
```

**Decision Framework**

Use this decision tree to choose between interfaces and abstract classes:

```
                        Need to share state (fields)?
                               /        \
                             YES         NO
                              |           |
                     Abstract Class    Need multiple
                                       inheritance?
                                        /      \
                                      YES       NO
                                       |         |
                                   Interface   Either works,
                                              prefer Interface
```

**Template Method Pattern (Abstract Class)**

Abstract classes excel at the template method pattern — defining an algorithm skeleton in a base class with hooks that subclasses override.

```java
public abstract class DataProcessor {
    // Template method - invariant algorithm structure
    public final void process() {
        List<String> data = readData();
        List<String> transformed = transformData(data);
        writeData(transformed);
    }

    protected abstract List<String> readData();
    protected abstract List<String> transformData(List<String> data);
    protected abstract void writeData(List<String> data);
}

public class CsvProcessor extends DataProcessor {
    @Override
    protected List<String> readData() { return List.of("csv", "data"); }
    @Override
    protected List<String> transformData(List<String> data) { return data; }
    @Override
    protected void writeData(List<String> data) { System.out.println(data); }
}
```

**API Design — Interface-First Approach**

The best practice is to define the contract with an interface first, optionally provide an abstract base class if there's shared logic, then implement concrete classes.

```java
// Step 1: Define the contract with an interface
public interface UserService {
    User findById(long id);
    User save(User user);
    void delete(long id);
}

// Step 2: Provide abstract base if there's shared logic
public abstract class AbstractUserService implements UserService {
    protected void validateUser(User user) {
        if (user.getName() == null) throw new IllegalArgumentException("Name required");
    }
}

// Step 3: Concrete implementation
public class DatabaseUserService extends AbstractUserService {
    @Override
    public User findById(long id) { /* DB query */ return null; }

    @Override
    public User save(User user) {
        validateUser(user);
        /* DB save */
        return user;
    }

    @Override
    public void delete(long id) { /* DB delete */ }
}
```

> **⚠️ Common Mistake** — Using abstract class when you only need a contract, or adding too many default methods to interfaces (turning them into "fat interfaces" that violate Interface Segregation).

```java
// Using abstract class when you only need a contract
public abstract class Printable {
    public abstract void print(); // No state, no shared logic
}

// Use an interface - simpler and allows multiple implementation
public interface Printable {
    void print();
}

// Adding too many default methods to interfaces
// (turns them into "fat interfaces" violating Interface Segregation)
public interface Kitchen {
    default void cook() { }
    default void wash() { }
    default void clean() { }
    default void serve() { }
    // This is becoming an abstract class in disguise!
}
```

> **🔧 Performance Tip** — Interface method dispatch (`invokeinterface`) involves an extra lookup compared to virtual dispatch (`invokevirtual`), but the JIT compiler optimizes this in hot paths. Choose based on design needs, not performance concerns.

> **🎯 Interviewer Insight** — Interviewers expect you to articulate clear decision criteria, not just list differences. Show you understand when each is appropriate and can discuss the evolution of interfaces in modern Java.

> **⚡ Follow-up** — Can an abstract class implement an interface without providing all methods, and what are the implications for subclasses?

> **⚡ Follow-up** — How do sealed interfaces (Java 17) change the design landscape compared to traditional interfaces?

> **⚡ Follow-up** — When would you use both an interface AND an abstract class together in a design pattern?

> **⚡ Follow-up** — How does the introduction of default methods (Java 8) affect the diamond problem resolution?

> **⚡ Follow-up** — What are the performance implications of interface method dispatch vs abstract class method dispatch?

> **📚 Related** — [Abstraction](./06-abstraction.md), [Default Methods in Interfaces](./16-default-methods-interfaces.md), [Diamond Problem](./18-diamond-problem.md)

---

[← Previous: Abstraction](./06-abstraction.md) · [Next: Method Overloading →](./08-method-overloading.md)


### ⚠️ Gotchas
- **Common Misconception**: Ensure you distinguish compile-time behavior from runtime execution in production environments.
- **Null Safety & Edge Cases**: Watch out for implicit null unboxing and edge boundary conditions when scaling under high concurrency.
- **Performance Pitfall**: Inappropriate usage of default APIs without profiling can lead to hidden bottlenecks and memory overhead.

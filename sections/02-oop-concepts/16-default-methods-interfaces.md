# Default Methods in Interfaces

[Home](../../README.md) > [Object-Oriented Programming](./README.md) > Default Methods in Interfaces

---

## Q19 🟠 How Do Default Methods Work in Interfaces?

**Question** — What are default methods in interfaces? Explain their purpose, multiple inheritance conflict resolution, when NOT to use them, and how they changed Java's interface design.

### Answer

**Default methods** (Java 8+) allow interfaces to provide method implementations using the `default` keyword. They were introduced primarily to enable **interface evolution** — adding new methods to existing interfaces without breaking all implementing classes.

**Basic Syntax**

Default methods provide a default implementation that implementing classes can use or override.

```java
public interface Loggable {
    // Abstract method (traditional)
    String getLogPrefix();

    // Default method - provides a default implementation
    default void log(String message) {
        System.out.printf("[%s] %s: %s%n",
            java.time.LocalTime.now(), getLogPrefix(), message);
    }

    default void logError(String message, Exception e) {
        log("ERROR - " + message + ": " + e.getMessage());
    }
}

// Implementing class gets default methods for free
public class UserService implements Loggable {
    @Override
    public String getLogPrefix() {
        return "UserService";
    }

    public void createUser(String name) {
        log("Creating user: " + name); // Uses default method
    }
}

// Can also override default methods
public class AuditService implements Loggable {
    @Override
    public String getLogPrefix() {
        return "AuditService";
    }

    @Override
    public void log(String message) {
        // Override with custom implementation
        System.out.println("AUDIT: " + message);
        writeToAuditLog(message);
    }

    private void writeToAuditLog(String message) { /* ... */ }
}
```

**Interface Evolution (The Primary Motivation)**

Before Java 8, adding a method to an interface would break all implementing classes. Default methods solve this by providing a default implementation that existing classes automatically inherit.

```java
// Before Java 8: adding forEach() to Iterable would break ALL implementations
public interface Iterable<T> {
    Iterator<T> iterator();

    // Java 8 added this without breaking existing code
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }

    // Java 8: also added spliterator()
    default Spliterator<T> spliterator() {
        return Spliterators.spliteratorUnknownSize(iterator(), 0);
    }
}

// All existing Iterable implementations (ArrayList, HashSet, etc.)
// got forEach() and spliterator() for free!
```

**Multiple Inheritance Conflict Resolution**

When a class implements multiple interfaces with the same default method, the class must explicitly override to resolve the conflict.

```java
public interface Flyable {
    default String move() {
        return "Flying through the air";
    }
}

public interface Swimmable {
    default String move() {
        return "Swimming through water";
    }
}

// COMPILE ERROR: class inherits unrelated defaults for move()
// public class Duck implements Flyable, Swimmable { }

// Must explicitly override to resolve the conflict
public class Duck implements Flyable, Swimmable {
    @Override
    public String move() {
        // Option 1: Choose one
        return Flyable.super.move();

        // Option 2: Combine both
        // return Flyable.super.move() + " and " + Swimmable.super.move();

        // Option 3: Provide entirely new implementation
        // return "Waddling on land";
    }
}
```

**Resolution Rules (Priority Order)**

The compiler follows three rules in strict order — class methods win over interface default methods, more specific interface methods win over less specific, and if still ambiguous, the implementing class must override.

**Rule 1 — Class Methods Win Over Interface Defaults**

```java
public interface Greeting {
    default String greet() { return "Hello from interface"; }
}

public class Parent {
    public String greet() { return "Hello from class"; }
}

// Class method wins - no ambiguity
public class Child extends Parent implements Greeting {
    // greet() returns "Hello from class"
    // Parent.greet() takes priority over Greeting.greet()
}
```

**Rule 2 — More Specific Interface Wins**

```java
public interface A {
    default String greet() { return "Hello from A"; }
}

public interface B extends A {
    @Override
    default String greet() { return "Hello from B"; } // More specific
}

// B.greet() wins because B extends A (more specific)
public class MyClass implements A, B {
    // No override needed - B.greet() is automatically used
}
```

**Rule 3 — If Still Ambiguous, Must Override**

When two unrelated interfaces have the same default method with no inheritance relationship, the implementing class must explicitly override and choose.

**Real-World Use — Collection.forEach**

The JDK extensively uses default methods to enhance collections without breaking existing code.

```java
public class DefaultMethodInAction {
    public static void main(String[] args) {
        List<String> names = List.of("Alice", "Bob", "Charlie");

        // forEach is a default method in Iterable interface
        names.forEach(System.out::println);

        // stream() is a default method in Collection interface
        names.stream()
             .filter(n -> n.length() > 3)
             .forEach(System.out::println);

        // Map.forEach is a default method
        Map<String, Integer> scores = Map.of("Alice", 95, "Bob", 87);
        scores.forEach((name, score) ->
            System.out.printf("%s: %d%n", name, score));

        // List.sort is a default method (Java 8)
        var mutableList = new ArrayList<>(names);
        mutableList.sort(Comparator.naturalOrder());
    }
}
```

**When NOT to Use Default Methods**

Don't use default methods to simulate abstract class behavior, especially when you need instance state. Default methods cannot access instance fields (interfaces can't have instance state).

```java
// Don't use defaults to simulate abstract class behavior
public interface Repository<T> {
    default T findById(long id) {
        // Complex implementation with state management
        // This belongs in an abstract class, not an interface
        throw new UnsupportedOperationException();
    }
}

// Don't use defaults when you need instance state
public interface Cacheable {
    // Interfaces can't have instance fields!
    // Map<String, Object> cache = new HashMap<>();  // This is public static final!
    default Object getCached(String key) {
        // Can't maintain per-instance cache in an interface
        return null;
    }
}

// Use abstract class when you need state
public abstract class AbstractCacheable {
    private final Map<String, Object> cache = new HashMap<>();

    protected Object getCached(String key) {
        return cache.get(key);
    }

    protected void putCache(String key, Object value) {
        cache.put(key, value);
    }
}
```

**Default Methods vs Abstract Methods**

| Aspect | Default Method | Abstract Method |
|--------|---------------|-----------------|
| Has body | ✅ Yes | ❌ No |
| Override required | ❌ Optional | ✅ Required |
| Can access state | ❌ No instance fields | ✅ Yes (in abstract class) |
| Purpose | Provide optional behavior | Define required contract |
| Added in | Java 8 | Java 1.0 |

> **⚠️ Common Mistake** — Assuming default methods can be called on the interface itself, or breaking existing implementations by changing a default method's contract in a way that depends on it.

```java
// Assuming default methods can be called on the interface itself
public interface MyInterface {
    default void doSomething() { System.out.println("doing"); }
}

// MyInterface.doSomething();  // COMPILE ERROR - need an instance!
// Default methods are instance methods, not static

// Breaking existing implementations by changing a default method's contract
public interface Versioned {
    // v1: returns version as integer
    default int getVersion() { return 1; }

    // v2: changing semantics silently breaks all implementations
    // that relied on v1 behavior
}
```

> **🔧 Performance Tip** — Default methods have the same performance as regular interface method calls. The JVM uses `invokeinterface` for dispatch, and the JIT compiler can inline monomorphic and bimorphic call sites. There is no additional overhead compared to abstract class methods.

> **🎯 Interviewer Insight** — Interviewers want to hear about the **motivation** (interface evolution), the **conflict resolution rules** (class > specific interface > general interface), and the **limitations** (no state). Understanding when NOT to use defaults distinguishes strong candidates.

> **⚡ Follow-up** — Can a default method call an abstract method on the same interface?

> **⚡ Follow-up** — How do default methods affect the diamond problem in Java?

> **⚡ Follow-up** — What is the difference between `default` methods and `static` methods in interfaces?

> **📚 Related** — [Interfaces vs Abstract Classes](./07-interfaces-vs-abstract-classes.md), [Diamond Problem](./18-diamond-problem.md), [Static and Private Methods in Interfaces](./17-static-private-methods-interfaces.md)

---

[← Previous: Functional Interfaces](./15-functional-interfaces.md) · [Next: Static and Private Methods in Interfaces →](./17-static-private-methods-interfaces.md)

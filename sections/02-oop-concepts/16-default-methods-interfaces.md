# Q16: How Do Default Methods Work in Interfaces?

## Question
What are default methods in interfaces? Explain their purpose, multiple inheritance conflict resolution, when NOT to use them, and how they changed Java's interface design.

## Difficulty: 🟠 Advanced

---

## Answer

**Default methods** (Java 8+) allow interfaces to provide method implementations using the `default` keyword. They were introduced primarily to enable **interface evolution** - adding new methods to existing interfaces without breaking all implementing classes.

### Basic Syntax

```java
public interface Loggable {
    // Abstract method (traditional)
    String getLogPrefix();

    // ✅ Default method - provides a default implementation
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
        log("Creating user: " + name); // ✅ Uses default method
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

### Interface Evolution (The Primary Motivation)

```java
// Before Java 8: adding forEach() to Iterable would break ALL implementations
public interface Iterable<T> {
    Iterator<T> iterator();

    // ✅ Java 8 added this without breaking existing code
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }

    // ✅ Java 8: also added spliterator()
    default Spliterator<T> spliterator() {
        return Spliterators.spliteratorUnknownSize(iterator(), 0);
    }
}

// All existing Iterable implementations (ArrayList, HashSet, etc.)
// got forEach() and spliterator() for free!
```

### Multiple Inheritance Conflict Resolution

When a class implements multiple interfaces with the same default method:

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

// ❌ COMPILE ERROR: class inherits unrelated defaults for move()
// public class Duck implements Flyable, Swimmable { }

// ✅ Must explicitly override to resolve the conflict
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

### Resolution Rules (Priority Order)

```
1. Class methods WIN over interface default methods
2. More specific interface methods WIN over less specific
3. If still ambiguous, the implementing class MUST override
```

```java
public interface A {
    default String greet() { return "Hello from A"; }
}

public interface B extends A {
    @Override
    default String greet() { return "Hello from B"; } // More specific
}

// Rule 2: B.greet() wins because B extends A (more specific)
public class MyClass implements A, B {
    // No override needed - B.greet() is automatically used
}

// Rule 1: class method always wins
public class Parent {
    public String greet() { return "Hello from Parent"; }
}

public class Child extends Parent implements A {
    // Parent.greet() wins over A.greet() - class > interface
}
```

### Real-World Use: Collection.forEach

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

### When NOT to Use Default Methods

```java
// ❌ Don't use defaults to simulate abstract class behavior
public interface Repository<T> {
    default T findById(long id) {
        // Complex implementation with state management
        // This belongs in an abstract class, not an interface
        throw new UnsupportedOperationException();
    }
}

// ❌ Don't use defaults when you need instance state
public interface Cacheable {
    // Interfaces can't have instance fields!
    // Map<String, Object> cache = new HashMap<>();  // This is public static final!
    default Object getCached(String key) {
        // Can't maintain per-instance cache in an interface
        return null;
    }
}

// ✅ Use abstract class when you need state
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

### Default Methods vs Abstract Methods

| Aspect | Default Method | Abstract Method |
|--------|---------------|-----------------|
| Has body | ✅ Yes | ❌ No |
| Override required | ❌ Optional | ✅ Required |
| Can access state | ❌ No instance fields | ✅ Yes (in abstract class) |
| Purpose | Provide optional behavior | Define required contract |
| Added in | Java 8 | Java 1.0 |

### Testing Default Methods

```java
// ✅ Testing default methods directly
public interface Calculable {
    double getValue();

    default double doubleValue() {
        return getValue() * 2;
    }

    default boolean isPositive() {
        return getValue() > 0;
    }
}

// Test using anonymous implementation
class CalculableTest {
    @Test
    void testDefaultMethods() {
        Calculable calc = () -> 5.0; // Lambda provides getValue()

        assertEquals(10.0, calc.doubleValue());
        assertTrue(calc.isPositive());
    }

    @Test
    void testNegativeValue() {
        Calculable calc = () -> -3.0;

        assertFalse(calc.isPositive());
        assertEquals(-6.0, calc.doubleValue());
    }
}
```

### Common Mistake

```java
// ❌ Assuming default methods can be called on the interface itself
public interface MyInterface {
    default void doSomething() { System.out.println("doing"); }
}

// MyInterface.doSomething();  // COMPILE ERROR - need an instance!
// Default methods are instance methods, not static

// ❌ Breaking existing implementations by changing a default method's contract
public interface Versioned {
    // v1: returns version as integer
    default int getVersion() { return 1; }

    // v2: changing semantics silently breaks all implementations
    // that relied on v1 behavior
}
```

### Performance Tip
Default methods have the same performance as regular interface method calls. The JVM uses `invokeinterface` for dispatch, and the JIT compiler can inline monomorphic and bimorphic call sites. There is no additional overhead compared to abstract class methods.

---

## Interviewer Insight
Interviewers want to hear about the **motivation** (interface evolution), the **conflict resolution rules** (class > specific interface > general interface), and the **limitations** (no state). Understanding when NOT to use defaults distinguishes strong candidates.

## Follow-up Questions
1. Can a default method call an abstract method on the same interface?
2. How do default methods affect the diamond problem in Java?
3. What is the difference between `default` methods and `static` methods in interfaces?

## Related Topics
- [Interfaces vs Abstract Classes](./q07-interfaces-vs-abstract-classes.md)
- [Diamond Problem](./q18-diamond-problem.md)
- [Static and Private Methods in Interfaces](./q17-static-private-methods-interfaces.md)

# Diamond Problem

[Home](../../README.md) > [Object-Oriented Programming](./README.md) > Diamond Problem

---

## Q21 🟠 How Does Java Solve the Diamond Problem?

**Question** — What is the Diamond Problem? How does Java handle it with interfaces and default methods? Explain the resolution rules, `InterfaceName.super` syntax, and compare with C++ multiple inheritance.

### Answer

The **Diamond Problem** occurs when a class inherits from two types that both provide an implementation of the same method, creating ambiguity about which version to use. The name comes from the diamond shape of the inheritance diagram.

**The Diamond Shape**

```
         Interface A
        /           \
       /             \
  Interface B    Interface C
       \             /
        \           /
         Class D
```

If both B and C provide a default implementation of a method from A, which does D inherit?

**Diamond Problem in Java**

Java requires explicit conflict resolution when implementing multiple interfaces with the same default method. This prevents ambiguity at compile time.

```java
public interface Animal {
    default String sound() {
        return "Some animal sound";
    }
}

public interface Pet extends Animal {
    @Override
    default String sound() {
        return "Friendly sound";
    }
}

public interface WildAnimal extends Animal {
    @Override
    default String sound() {
        return "Wild roar";
    }
}

// COMPILE ERROR: class ExoticPet inherits unrelated defaults for sound()
// public class ExoticPet implements Pet, WildAnimal { }

// Must explicitly resolve the conflict
public class ExoticPet implements Pet, WildAnimal {
    @Override
    public String sound() {
        // Option 1: Choose one parent's implementation
        return Pet.super.sound();

        // Option 2: Choose the other
        // return WildAnimal.super.sound();

        // Option 3: Combine both
        // return Pet.super.sound() + " / " + WildAnimal.super.sound();

        // Option 4: Completely new implementation
        // return "Exotic sound";
    }
}
```

**Java's Resolution Rules**

The compiler follows three rules in strict order to resolve conflicts.

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

public class Demo {
    public static void main(String[] args) {
        Child c = new Child();
        System.out.println(c.greet()); // "Hello from class"
    }
}
```

**Rule 2 — More Specific Interface Wins**

```java
public interface Base {
    default String identify() { return "Base"; }
}

public interface Specific extends Base {
    @Override
    default String identify() { return "Specific"; } // More specific
}

// Specific.identify() wins because Specific extends Base
public class MyClass implements Base, Specific {
    // identify() returns "Specific" - no need to override
}

public class Demo {
    public static void main(String[] args) {
        MyClass obj = new MyClass();
        System.out.println(obj.identify()); // "Specific"
    }
}
```

**Rule 3 — If Still Ambiguous, Must Override**

When two interfaces are unrelated (neither extends the other) and both provide the same default method, the implementing class must explicitly override.

```java
public interface Left {
    default String side() { return "Left"; }
}

public interface Right {
    default String side() { return "Right"; }
}

// Neither Left nor Right is more specific - MUST override
public class Center implements Left, Right {
    @Override
    public String side() {
        return "Center"; // Or delegate to one: Left.super.side()
    }
}
```

**InterfaceName.super Syntax**

When implementing multiple interfaces with conflicting defaults, use `InterfaceName.super.method()` to explicitly call a specific interface's default method.

```java
public interface Logger {
    default void log(String msg) {
        System.out.println("[LOG] " + msg);
    }
}

public interface Auditor {
    default void log(String msg) {
        System.out.println("[AUDIT] " + msg);
    }
}

public class SecureService implements Logger, Auditor {
    @Override
    public void log(String msg) {
        // Call specific interface's default method
        Logger.super.log(msg);   // "[LOG] message"
        Auditor.super.log(msg);  // "[AUDIT] message"
    }

    // Can also delegate in separate methods
    public void logInfo(String msg) {
        Logger.super.log(msg);
    }

    public void audit(String msg) {
        Auditor.super.log(msg);
    }
}
```

**Complex Diamond Scenario**

When multiple interfaces extend a common interface and all provide default implementations, the most specific wins automatically. Only when there's no clear specificity does manual resolution become necessary.

```java
public interface Printable {
    default String format() { return "Printable format"; }
}

public interface Displayable extends Printable {
    @Override
    default String format() { return "Displayable format"; }
}

public interface Exportable extends Printable {
    @Override
    default String format() { return "Exportable format"; }
}

// Conflict between Displayable and Exportable (both extend Printable)
public class Report implements Displayable, Exportable {
    @Override
    public String format() {
        // Must resolve - both are equally specific
        return Displayable.super.format(); // Choose one
    }
}

// But if Report only implemented one:
public class SimpleReport implements Displayable {
    // No conflict - Displayable.format() wins over Printable.format()
    // (Rule 2: more specific wins)
}
```

**Comparison — Java vs C++ Multiple Inheritance**

Java avoids the complexity of C++ virtual inheritance by restricting multiple inheritance to interfaces only (no state). C++ allows full multiple inheritance with state, which requires virtual inheritance to solve the diamond problem.

| Feature | Java | C++ |
|---------|------|-----|
| Multiple class inheritance | ❌ Not allowed | ✅ Allowed |
| Multiple interface/type inheritance | ✅ Interfaces | ✅ (no direct equivalent) |
| State from multiple parents | ❌ No (interfaces have no state) | ✅ Yes (can be problematic) |
| Diamond resolution | Explicit override required | Virtual inheritance |
| Ambiguity handling | Compile-time error | Compile-time error (unless virtual) |

```cpp
// C++ Diamond Problem
class Animal { public: virtual void sound() { cout << "Animal"; } };
class Pet : virtual public Animal { public: void sound() override { cout << "Pet"; } };
class Wild : virtual public Animal { public: void sound() override { cout << "Wild"; } };
class Exotic : public Pet, public Wild {
    // Must override to resolve ambiguity
    void sound() override { Pet::sound(); }
};
```

```java
// Java approach - cleaner, no virtual inheritance needed
interface Animal { default void sound() { System.out.println("Animal"); } }
interface Pet extends Animal { default void sound() { System.out.println("Pet"); } }
interface Wild extends Animal { default void sound() { System.out.println("Wild"); } }
class Exotic implements Pet, Wild {
    @Override
    public void sound() { Pet.super.sound(); }
}
```

**Real-World Diamond Scenario**

In real APIs, diamond problems can occur when different aspects of functionality (caching, auditing, serialization) all extend a common base interface.

```java
public interface Serializable {
    default byte[] serialize() {
        // Default JSON serialization
        return toJson().getBytes();
    }
    default String toJson() { return "{}"; }
}

public interface Cacheable extends Serializable {
    @Override
    default byte[] serialize() {
        // Add cache key prefix
        return ("CACHE:" + toJson()).getBytes();
    }
}

public interface Auditable extends Serializable {
    @Override
    default byte[] serialize() {
        // Add timestamp
        return (System.currentTimeMillis() + ":" + toJson()).getBytes();
    }
}

// Real entity needs both caching and auditing
public class UserProfile implements Cacheable, Auditable {
    private final String name;
    private final String email;

    public UserProfile(String name, String email) {
        this.name = name;
        this.email = email;
    }

    @Override
    public String toJson() {
        return "{\"name\":\"%s\",\"email\":\"%s\"}".formatted(name, email);
    }

    @Override
    public byte[] serialize() {
        // Explicitly choose strategy or combine
        return Cacheable.super.serialize(); // Use cache-friendly format
    }
}
```

> **⚠️ Common Mistake** — Assuming diamond resolution works like method overloading (the compiler can "pick one"), or trying to call `super.super` to skip a level in the hierarchy.

```java
// Assuming diamond resolution works like method overloading
public interface A {
    default void process(String s) { System.out.println("A: " + s); }
}
public interface B {
    default void process(String s) { System.out.println("B: " + s); }
}
public class C implements A, B {
    // You MUST override process() - the compiler cannot "pick one"
    // This class won't compile without the override
    @Override
    public void process(String s) {
        A.super.process(s); // Explicitly choose
    }
}

// Trying to call super.super (skipping a level)
public interface Base { default void run() { } }
public interface Mid extends Base { default void run() { } }
public class Impl implements Mid {
    @Override
    public void run() {
        // Base.super.run();  // Can only call directly implemented interfaces
        Mid.super.run();      // This is the immediate super
    }
}
```

> **🔧 Performance Tip** — Diamond resolution is entirely a compile-time concern. Once the compiler resolves which method to use, the bytecode contains a direct reference. There is no runtime overhead for diamond resolution. The JVM uses the same `invokeinterface` instruction regardless of how many interfaces are involved.

> **🎯 Interviewer Insight** — This is a design question that tests understanding of Java's type system. Interviewers want to hear the three resolution rules in order (class > specific > explicit), the `InterfaceName.super` syntax, and why Java's approach avoids the complexity of C++ virtual inheritance.

> **⚡ Follow-up** — What happens if two interfaces have the same default method with different return types?

> **⚡ Follow-up** — Can the diamond problem occur with abstract methods?

> **⚡ Follow-up** — How do sealed interfaces (Java 17) affect the diamond problem?

> **📚 Related** — [Default Methods in Interfaces](./16-default-methods-interfaces.md), [Interfaces vs Abstract Classes](./07-interfaces-vs-abstract-classes.md), [Inheritance](./03-inheritance.md)

---

[← Previous: Static and Private Methods in Interfaces](./17-static-private-methods-interfaces.md) · [Back to Section Index](./README.md)


### ⚠️ Gotchas
- **Common Misconception**: Ensure you distinguish compile-time behavior from runtime execution in production environments.
- **Null Safety & Edge Cases**: Watch out for implicit null unboxing and edge boundary conditions when scaling under high concurrency.
- **Performance Pitfall**: Inappropriate usage of default APIs without profiling can lead to hidden bottlenecks and memory overhead.

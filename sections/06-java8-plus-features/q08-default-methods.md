# Q8: How Do Default Methods Enable Interface Evolution?

## 📋 Question
How do default methods work in Java interfaces? Explain their purpose for interface evolution, the diamond problem resolution, and when to use them vs abstract classes.

## Difficulty: 🟡 Medium

---

## 💡 Answer

**Default methods** (Java 8) allow interfaces to provide **method implementations** using the `default` keyword. Their primary purpose is **interface evolution** — adding new methods to existing interfaces without breaking all implementing classes. Before Java 8, adding a method to an interface forced every implementor to provide an implementation, making API evolution nearly impossible.

### Default Method Syntax and Usage

```java
package com.interview.java8;

import java.util.List;

public class DefaultMethodDemo {

    // ✅ Interface with default method
    interface Greeting {
        String greet(String name);  // Abstract — must be implemented

        // Default method — provides implementation, can be overridden
        default String greetAll(List<String> names) {
            StringBuilder sb = new StringBuilder();
            for (String name : names) {
                if (sb.length() > 0) sb.append(", ");
                sb.append(greet(name));
            }
            return sb.toString();
        }

        // Another default method
        default String greetWithTitle(String name, String title) {
            return greet(title + " " + name);
        }
    }

    // ✅ Implementing class — only needs to implement abstract methods
    static class FormalGreeting implements Greeting {
        @Override
        public String greet(String name) {
            return "Good day, " + name;
        }
        // greetAll() and greetWithTitle() are inherited from the interface
    }

    // ✅ Can override default methods
    static class CasualGreeting implements Greeting {
        @Override
        public String greet(String name) {
            return "Hey, " + name + "!";
        }

        @Override
        public String greetAll(List<String> names) {
            return "Hey everyone: " + String.join(", ", names) + "!";
        }
    }

    public static void main(String[] args) {
        Greeting formal = new FormalGreeting();
        System.out.println(formal.greet("Alice"));                    // Good day, Alice
        System.out.println(formal.greetAll(List.of("Alice", "Bob"))); // Good day, Alice, Good day, Bob

        Greeting casual = new CasualGreeting();
        System.out.println(casual.greetAll(List.of("Alice", "Bob"))); // Hey everyone: Alice, Bob!
    }
}
```

### Interface Evolution — The Real Motivation

```java
package com.interview.java8;

import java.util.*;
import java.util.stream.*;

public class InterfaceEvolutionDemo {

    // Before Java 8: adding a method to Collection would break ALL implementations
    // Java 8 solution: add as default method

    // Real examples from the JDK:
    // ✅ Iterable.forEach(Consumer)        — added in Java 8
    // ✅ Collection.stream()               — added in Java 8
    // ✅ Collection.removeIf(Predicate)    — added in Java 8
    // ✅ List.sort(Comparator)             — added in Java 8
    // ✅ Map.getOrDefault(K, V)            — added in Java 8
    // ✅ Map.putIfAbsent(K, V)             — added in Java 8
    // ✅ Map.forEach(BiConsumer)           — added in Java 8
    // ✅ Map.replaceAll(BiFunction)        — added in Java 8

    public static void main(String[] args) {
        List<String> names = new ArrayList<>(List.of("Charlie", "Alice", "Bob"));

        // ✅ These are all default methods added in Java 8
        names.forEach(System.out::println);                // Iterable.forEach
        names.sort(Comparator.naturalOrder());             // List.sort
        names.removeIf(n -> n.startsWith("C"));            // Collection.removeIf
        Stream<String> stream = names.stream();            // Collection.stream

        Map<String, Integer> map = new HashMap<>();
        map.put("Alice", 90);
        int score = map.getOrDefault("Bob", 0);            // Map.getOrDefault
        map.putIfAbsent("Bob", 85);                        // Map.putIfAbsent
        map.forEach((k, v) -> System.out.println(k + "=" + v));  // Map.forEach
    }
}
```

### Multiple Inheritance — The Diamond Problem

```java
package com.interview.java8;

public class DiamondProblemDemo {

    interface Flyable {
        default String move() { return "Flying"; }
    }

    interface Swimmable {
        default String move() { return "Swimming"; }
    }

    // ❌ Compile error — ambiguous: which move() to use?
    // class Duck implements Flyable, Swimmable { }

    // ✅ Resolution: class MUST override the conflicting method
    static class Duck implements Flyable, Swimmable {
        @Override
        public String move() {
            // Option 1: Provide own implementation
            return "Walking";
        }
    }

    // ✅ Can delegate to a specific interface's default
    static class FlyingFish implements Flyable, Swimmable {
        @Override
        public String move() {
            return Flyable.super.move();  // Explicitly choose Flyable's default
        }
    }

    // ═══════════════════════════════════════════════════════
    // Resolution Rules (priority order):
    // 1. Class methods WIN over interface default methods
    // 2. More SPECIFIC interface wins (sub-interface over parent)
    // 3. If still ambiguous → compile error, must override
    // ═══════════════════════════════════════════════════════

    interface Animal {
        default String sound() { return "..."; }
    }

    interface Pet extends Animal {
        @Override
        default String sound() { return "Purr"; }  // More specific
    }

    // Pet.sound() wins because Pet is more specific than Animal
    static class Cat implements Animal, Pet {
        // Inherits Pet.sound() → "Purr" (no override needed)
    }

    // Class method always wins over any default
    static class Dog implements Animal {
        @Override
        public String sound() { return "Bark"; }  // Class wins
    }

    public static void main(String[] args) {
        System.out.println(new Duck().move());        // Walking
        System.out.println(new FlyingFish().move());  // Flying
        System.out.println(new Cat().sound());        // Purr
        System.out.println(new Dog().sound());        // Bark
    }
}
```

### Default Methods vs Abstract Classes

| Feature | Default Methods | Abstract Classes |
|---------|----------------|-----------------|
| **State** | No instance fields | Can have instance fields |
| **Constructors** | No constructors | Can have constructors |
| **Multiple inheritance** | Class can implement multiple interfaces | Single inheritance only |
| **Access modifiers** | Implicitly `public` | Any access modifier |
| **Purpose** | Interface evolution, mix-in behavior | Shared base implementation |
| **`this` reference** | Refers to implementing class | Refers to abstract class |

```java
package com.interview.java8;

public class DefaultVsAbstractDemo {

    // ✅ Use default methods for: stateless mix-in behavior
    interface Loggable {
        default void log(String message) {
            System.out.println("[" + getClass().getSimpleName() + "] " + message);
        }
    }

    interface Auditable {
        default void audit(String action) {
            System.out.println("AUDIT: " + action + " by " + getClass().getSimpleName());
        }
    }

    // ✅ Class can mix in BOTH behaviors (impossible with abstract classes)
    static class UserService implements Loggable, Auditable {
        public void createUser(String name) {
            log("Creating user: " + name);      // From Loggable
            audit("CREATE_USER");               // From Auditable
        }
    }

    // ✅ Use abstract class for: shared state + template pattern
    static abstract class AbstractRepository<T> {
        protected final String tableName;  // ❌ Interfaces can't have this

        protected AbstractRepository(String tableName) {
            this.tableName = tableName;    // ❌ Interfaces can't have constructors
        }

        public final void save(T entity) {  // Template method
            validate(entity);
            persist(entity);
        }

        protected abstract void validate(T entity);
        protected abstract void persist(T entity);
    }

    public static void main(String[] args) {
        UserService service = new UserService();
        service.createUser("Alice");
    }
}
```

### ⚠️ Common Mistake
**Treating interfaces with default methods as abstract classes**: Default methods should provide **sensible defaults** or **convenience methods** built on top of abstract methods. They should NOT contain complex business logic, maintain state (via workarounds), or create deep default method hierarchies. If you need state, constructors, or protected methods, use an abstract class. Default methods are for **horizontal composition** (mix-ins), not vertical inheritance.

### ⚡ Performance Tip
Default methods are dispatched via `invokevirtual` (same as regular instance methods) when the implementing class doesn't override them. There is **no performance penalty** compared to concrete class methods. The JVM's JIT compiler can inline default methods just as effectively as regular methods. However, avoid creating default methods that call other default methods in deep chains — this adds indirection that makes the code harder to debug, even though the JIT compiler typically handles it efficiently.

### 🔧 JVM Internals
When a class inherits a default method without overriding it, the JVM creates a **bridge** in the class's vtable (virtual method table) that points to the interface's default implementation. The resolution follows the **most specific** rule at the bytecode level — the compiler determines which default to use and encodes it in the class file. At runtime, `invokevirtual` dispatches just like a regular method call. The `InterfaceName.super.method()` syntax compiles to `invokespecial`, directly targeting the specific interface's default implementation.

---

## 🎯 Interviewer Insight
Interviewers want you to explain the **motivation** for default methods (backward-compatible interface evolution — e.g., adding `stream()` to `Collection`) and the **diamond problem resolution rules**. Knowing the three resolution rules (class wins, specific wins, otherwise error) is essential. A strong answer contrasts default methods with abstract classes and identifies appropriate use cases for each.

## ⚡ Follow-up Questions
1. Can a default method call an abstract method of the same interface?
2. What happens if a class inherits the same default method from an interface directly and via a superclass?
3. How did the JDK team add `forEach` to `Iterable` without breaking existing code?

## 📚 Related Topics
- [Static Interface Methods](./q09-static-interface-methods.md)
- [Private Interface Methods](./q10-private-interface-methods.md)
- [Sealed Classes](./q16-sealed-classes-detail.md)

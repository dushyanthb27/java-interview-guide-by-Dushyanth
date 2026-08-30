# Q9: Static Methods in Interfaces

## 📋 Question
What are static methods in interfaces? How are they used for factory methods and utility functions, and how do they differ from helper/utility classes?

## Difficulty: 🟡 Medium

---

## 💡 Answer

**Static interface methods** (Java 8) allow interfaces to contain static methods with implementations. Unlike default methods, static methods **cannot be overridden** and are called directly on the interface type (`InterfaceName.method()`). Their primary use cases are **factory methods**, **utility methods**, and keeping related functionality **cohesive** within the interface that defines the type.

### Static Method Syntax and Usage

```java
package com.interview.java8;

import java.util.List;
import java.util.Comparator;

public class StaticInterfaceMethodDemo {

    // ✅ Interface with static factory and utility methods
    interface Validator<T> {
        boolean validate(T value);
        String errorMessage();

        // ✅ Static factory methods — create common validators
        static Validator<String> notEmpty() {
            return new Validator<>() {
                @Override
                public boolean validate(String value) {
                    return value != null && !value.isEmpty();
                }

                @Override
                public String errorMessage() {
                    return "Value must not be empty";
                }
            };
        }

        static Validator<String> maxLength(int max) {
            return new Validator<>() {
                @Override
                public boolean validate(String value) {
                    return value != null && value.length() <= max;
                }

                @Override
                public String errorMessage() {
                    return "Value must not exceed " + max + " characters";
                }
            };
        }

        // ✅ Static utility method — compose validators
        static <T> Validator<T> allOf(List<Validator<T>> validators) {
            return new Validator<>() {
                @Override
                public boolean validate(T value) {
                    return validators.stream().allMatch(v -> v.validate(value));
                }

                @Override
                public String errorMessage() {
                    return validators.stream()
                            .map(Validator::errorMessage)
                            .reduce((a, b) -> a + "; " + b)
                            .orElse("Validation failed");
                }
            };
        }
    }

    public static void main(String[] args) {
        // ✅ Called on the INTERFACE, not on implementing classes
        Validator<String> nameValidator = Validator.allOf(List.of(
                Validator.notEmpty(),
                Validator.maxLength(50)
        ));

        System.out.println(nameValidator.validate("Alice"));   // true
        System.out.println(nameValidator.validate(""));         // false
        System.out.println(nameValidator.errorMessage());
    }
}
```

### Real JDK Examples

```java
package com.interview.java8;

import java.util.*;
import java.util.stream.*;
import java.util.function.*;
import java.nio.file.*;

public class JdkStaticMethodExamples {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // Comparator static factory methods
        // ═══════════════════════════════════════════════════
        Comparator<String> byLength = Comparator.comparingInt(String::length);
        Comparator<String> natural = Comparator.naturalOrder();
        Comparator<String> reversed = Comparator.reverseOrder();
        Comparator<String> nullsFirst = Comparator.nullsFirst(natural);

        List<String> names = new ArrayList<>(List.of("Charlie", "Alice", "Bob"));
        names.sort(Comparator.comparingInt(String::length)
                .thenComparing(Comparator.naturalOrder()));
        System.out.println(names);  // [Bob, Alice, Charlie]

        // ═══════════════════════════════════════════════════
        // Stream static factory methods
        // ═══════════════════════════════════════════════════
        Stream<String> ofValues = Stream.of("a", "b", "c");
        Stream<String> empty = Stream.empty();
        Stream<Integer> generated = Stream.iterate(0, n -> n + 2).limit(5);
        Stream<Double> randoms = Stream.generate(Math::random).limit(3);

        // ═══════════════════════════════════════════════════
        // List / Set / Map static factory methods (Java 9+)
        // ═══════════════════════════════════════════════════
        List<String> immutableList = List.of("a", "b", "c");
        Set<Integer> immutableSet = Set.of(1, 2, 3);
        Map<String, Integer> immutableMap = Map.of("a", 1, "b", 2);
        Map<String, Integer> entries = Map.ofEntries(
                Map.entry("alice", 90),
                Map.entry("bob", 85)
        );

        // ═══════════════════════════════════════════════════
        // Predicate / Function static methods
        // ═══════════════════════════════════════════════════
        Predicate<String> notNull = Predicate.not(Objects::isNull); // Java 11
        Function<String, String> identity = Function.identity();
        UnaryOperator<String> identityOp = UnaryOperator.identity();

        // ═══════════════════════════════════════════════════
        // Path static factory method
        // ═══════════════════════════════════════════════════
        Path path = Path.of("src", "main", "java");  // Java 11+
    }
}
```

### Interface as a Cohesive Unit

```java
package com.interview.java8;

import java.util.*;
import java.util.function.*;

public class CohesiveInterfaceDemo {

    // ✅ BEFORE Java 8: Interface + companion utility class
    // interface Shape { double area(); }
    // class ShapeUtils { static Shape circle(double r) { ... } }  // Separate class

    // ✅ AFTER Java 8: Everything in one place — cohesive!
    interface Shape {
        double area();
        double perimeter();

        // Static factory methods — no need for separate ShapeFactory
        static Shape circle(double radius) {
            return new Shape() {
                @Override
                public double area() { return Math.PI * radius * radius; }
                @Override
                public double perimeter() { return 2 * Math.PI * radius; }
                @Override
                public String toString() { return "Circle(r=" + radius + ")"; }
            };
        }

        static Shape rectangle(double width, double height) {
            return new Shape() {
                @Override
                public double area() { return width * height; }
                @Override
                public double perimeter() { return 2 * (width + height); }
                @Override
                public String toString() { return "Rectangle(" + width + "x" + height + ")"; }
            };
        }

        // Static utility method
        static Shape largest(List<Shape> shapes) {
            return shapes.stream()
                    .max(Comparator.comparingDouble(Shape::area))
                    .orElseThrow(() -> new IllegalArgumentException("Empty list"));
        }

        // Static comparison factory
        static Comparator<Shape> byArea() {
            return Comparator.comparingDouble(Shape::area);
        }
    }

    public static void main(String[] args) {
        // ✅ Factory methods called directly on the interface
        Shape circle = Shape.circle(5.0);
        Shape rect = Shape.rectangle(4.0, 6.0);

        List<Shape> shapes = List.of(circle, rect, Shape.circle(3.0));
        Shape biggest = Shape.largest(shapes);

        shapes.stream()
                .sorted(Shape.byArea())
                .forEach(s -> System.out.println(s + " → area=" + s.area()));
    }
}
```

### Static Methods Cannot Be Overridden

```java
package com.interview.java8;

public class StaticNotOverriddenDemo {

    interface Logger {
        void log(String msg);

        static Logger create() {
            return msg -> System.out.println("[DEFAULT] " + msg);
        }
    }

    interface FileLogger extends Logger {
        // ✅ This is a NEW static method — NOT an override
        static Logger create() {
            return msg -> System.out.println("[FILE] " + msg);
        }
    }

    static class ConsoleLogger implements Logger {
        @Override
        public void log(String msg) {
            System.out.println("[CONSOLE] " + msg);
        }

        // ❌ Cannot override interface static methods
        // Defining create() here is a class-level static — unrelated
    }

    public static void main(String[] args) {
        // ✅ Static methods are called on the INTERFACE name
        Logger defaultLogger = Logger.create();       // [DEFAULT]
        Logger fileLogger = FileLogger.create();      // [FILE]

        // ❌ Cannot call via implementing class
        // ConsoleLogger.create(); // Not inherited

        // ❌ Cannot call via instance reference
        // Logger logger = new ConsoleLogger();
        // logger.create(); // ❌ Compile error
    }
}
```

### Comparison: Static Interface Methods vs Utility Classes

| Feature | Static Interface Methods | Utility Classes |
|---------|------------------------|-----------------|
| **Location** | Inside the type definition | Separate class |
| **Cohesion** | High — co-located with type | Low — scattered |
| **Discoverability** | IDE auto-suggests from type | Must know class name |
| **Inheritance** | Not inherited by subtypes | N/A |
| **Private helpers** | Can use private methods (Java 9) | Can use private methods |
| **State** | No fields | Can have static fields |
| **Use case** | Factories, common operations | Cross-cutting utilities |

### ⚠️ Common Mistake
**Expecting static interface methods to be inherited**: Unlike static methods in classes, static interface methods are **NOT inherited** by implementing classes or sub-interfaces. You must call them using the declaring interface name: `MyInterface.staticMethod()`. Attempting to call them via `implementingClass.staticMethod()` results in a compile error. This is by design — it prevents ambiguity when a class implements multiple interfaces with same-named static methods.

### ⚡ Performance Tip
Static interface methods compile to `invokestatic` bytecode — the same as static methods in classes. There is **zero overhead** compared to utility class methods. They are resolved at compile time, not runtime, and are prime candidates for JIT inlining. Using static factory methods (`List.of()`, `Map.of()`) instead of constructors also enables internal optimizations — the JDK returns optimized immutable implementations based on the number of elements.

### 🔧 JVM Internals
At the bytecode level, static interface methods are invoked using `invokestatic` with the interface's class reference. They are stored in the interface's class file alongside default methods. Since they cannot be overridden, no virtual dispatch is needed — the call target is always known at compile time. This makes them slightly more efficient than default methods (which use `invokevirtual`) in theory, though the JIT compiler's inlining typically eliminates this difference.

---

## 🎯 Interviewer Insight
Interviewers want to see that you understand the **cohesion benefit** of static interface methods — putting factory methods where they logically belong, alongside the type they create. Knowing real JDK examples (`Comparator.comparing()`, `Stream.of()`, `List.of()`) demonstrates practical familiarity. The key insight to mention: static interface methods replaced the old "companion utility class" pattern (e.g., `Collections` companion to `Collection`).

## ⚡ Follow-up Questions
1. Why aren't static interface methods inherited by implementing classes?
2. How do `List.of()` and `Map.of()` (Java 9) leverage static interface methods?
3. Can static interface methods access default methods of the same interface?

## 📚 Related Topics
- [Default Methods](./q08-default-methods.md)
- [Private Interface Methods](./q10-private-interface-methods.md)
- [Functional Interfaces](./q03-functional-interfaces-detail.md)

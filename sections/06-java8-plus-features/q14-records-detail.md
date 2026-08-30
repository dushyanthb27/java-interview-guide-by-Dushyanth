# Q14: Records in Depth (Java 14+)

## 📋 Question
What are Records in Java? Explain record components, constructors (canonical, compact, custom), generic records, records with interfaces, record patterns, serialization, and restrictions.

## Difficulty: 🟠 Senior

---

## 💡 Answer

**Records** (Java 14 preview, Java 16 final) are a special class type that provides a **transparent, immutable data carrier** with auto-generated `equals()`, `hashCode()`, `toString()`, accessors, and a canonical constructor. They dramatically reduce the boilerplate of simple data classes — what previously required 50+ lines of code now takes one line.

### Record Basics

```java
package com.interview.java14;

// ✅ One-line record replaces ~50 lines of boilerplate
// Compiler generates: constructor, accessors, equals, hashCode, toString
public record Point(int x, int y) {}

// What the compiler generates (conceptually):
// public final class Point {
//     private final int x;
//     private final int y;
//     public Point(int x, int y) { this.x = x; this.y = y; }
//     public int x() { return x; }          // NOT getX()!
//     public int y() { return y; }          // NOT getY()!
//     public boolean equals(Object o) { ... component-based equality ... }
//     public int hashCode() { ... based on x, y ... }
//     public String toString() { return "Point[x=" + x + ", y=" + y + "]"; }
// }
```

### Constructors: Canonical, Compact, and Custom

```java
package com.interview.java14;

import java.util.Objects;

public class RecordConstructorsDemo {

    // ═══════════════════════════════════════════════════
    // Canonical constructor (auto-generated — can be overridden)
    // ═══════════════════════════════════════════════════
    record Range(int min, int max) {
        // ✅ Explicit canonical constructor with validation
        public Range(int min, int max) {
            if (min > max) throw new IllegalArgumentException(
                    "min (" + min + ") must be <= max (" + max + ")");
            this.min = min;
            this.max = max;
        }
    }

    // ═══════════════════════════════════════════════════
    // Compact constructor — preferred for validation (no params)
    // ═══════════════════════════════════════════════════
    record Email(String value) {
        // ✅ Compact constructor — no parameter list, auto-assigns fields at end
        public Email {
            Objects.requireNonNull(value, "Email cannot be null");
            value = value.trim().toLowerCase();  // Normalize before assignment
            if (!value.contains("@")) {
                throw new IllegalArgumentException("Invalid email: " + value);
            }
            // Implicit: this.value = value; (auto-generated at end)
        }
    }

    // ═══════════════════════════════════════════════════
    // Custom additional constructors (must delegate to canonical)
    // ═══════════════════════════════════════════════════
    record Temperature(double celsius) {
        // ✅ Compact constructor for validation
        public Temperature {
            if (celsius < -273.15) {
                throw new IllegalArgumentException("Below absolute zero!");
            }
        }

        // ✅ Additional factory constructors
        public static Temperature fromFahrenheit(double f) {
            return new Temperature((f - 32) * 5.0 / 9.0);
        }

        public static Temperature fromKelvin(double k) {
            return new Temperature(k - 273.15);
        }

        // ✅ Custom accessor (instance method)
        public double fahrenheit() {
            return celsius * 9.0 / 5.0 + 32;
        }
    }

    public static void main(String[] args) {
        var range = new Range(1, 10);
        System.out.println(range);  // Range[min=1, max=10]
        // new Range(10, 1);  // ❌ IllegalArgumentException

        var email = new Email("  Alice@Example.COM  ");
        System.out.println(email.value());  // alice@example.com

        var temp = Temperature.fromFahrenheit(98.6);
        System.out.printf("%.1f°C = %.1f°F%n", temp.celsius(), temp.fahrenheit());
    }
}
```

### Records with Generics and Interfaces

```java
package com.interview.java14;

import java.util.Comparator;

public class RecordGenericsDemo {

    // ═══════════════════════════════════════════════════
    // Generic records
    // ═══════════════════════════════════════════════════
    record Pair<A, B>(A first, B second) {
        public <C> Pair<A, C> mapSecond(java.util.function.Function<B, C> f) {
            return new Pair<>(first, f.apply(second));
        }
    }

    record Result<T>(T value, String error) {
        public static <T> Result<T> success(T value) {
            return new Result<>(value, null);
        }

        public static <T> Result<T> failure(String error) {
            return new Result<>(null, error);
        }

        public boolean isSuccess() {
            return error == null;
        }
    }

    // ═══════════════════════════════════════════════════
    // Records implementing interfaces
    // ═══════════════════════════════════════════════════
    interface Printable {
        String prettyPrint();
    }

    record Employee(String name, String dept, double salary)
            implements Printable, Comparable<Employee> {

        @Override
        public String prettyPrint() {
            return String.format("%s (%s) - $%,.2f", name, dept, salary);
        }

        @Override
        public int compareTo(Employee other) {
            return Double.compare(this.salary, other.salary);
        }
    }

    // ═══════════════════════════════════════════════════
    // Records with static fields and methods
    // ═══════════════════════════════════════════════════
    record Money(double amount, String currency) {
        // ✅ Static fields are allowed
        public static final Money ZERO_USD = new Money(0, "USD");

        // ✅ Static methods are allowed
        public static Money usd(double amount) {
            return new Money(amount, "USD");
        }

        // ✅ Instance methods are allowed
        public Money add(Money other) {
            if (!this.currency.equals(other.currency)) {
                throw new IllegalArgumentException("Currency mismatch");
            }
            return new Money(this.amount + other.amount, this.currency);
        }
    }

    public static void main(String[] args) {
        var pair = new Pair<>("Alice", 42);
        System.out.println(pair);  // Pair[first=Alice, second=42]

        var mapped = pair.mapSecond(n -> n * 2);
        System.out.println(mapped);  // Pair[first=Alice, second=84]

        var emp = new Employee("Alice", "Engineering", 95000);
        System.out.println(emp.prettyPrint());  // Alice (Engineering) - $95,000.00

        var total = Money.usd(100).add(Money.usd(50));
        System.out.println(total);  // Money[amount=150.0, currency=USD]
    }
}
```

### Record Patterns (Java 21)

```java
package com.interview.java21;

public class RecordPatternsDemo {

    sealed interface Shape permits Circle, Rectangle {}
    record Circle(double radius) implements Shape {}
    record Rectangle(double width, double height) implements Shape {}
    record Point(double x, double y) {}
    record Line(Point start, Point end) {}

    // ═══════════════════════════════════════════════════
    // Record pattern in instanceof
    // ═══════════════════════════════════════════════════
    static double area(Shape shape) {
        if (shape instanceof Circle(double r)) {
            return Math.PI * r * r;
        } else if (shape instanceof Rectangle(double w, double h)) {
            return w * h;
        }
        throw new IllegalArgumentException("Unknown shape");
    }

    // ═══════════════════════════════════════════════════
    // Record pattern in switch (Java 21)
    // ═══════════════════════════════════════════════════
    static String describe(Shape shape) {
        return switch (shape) {
            case Circle(var r) when r > 10 -> "Large circle (r=" + r + ")";
            case Circle(var r) -> "Small circle (r=" + r + ")";
            case Rectangle(var w, var h) when w == h -> "Square (" + w + "x" + h + ")";
            case Rectangle(var w, var h) -> "Rectangle (" + w + "x" + h + ")";
        };
    }

    // ═══════════════════════════════════════════════════
    // Nested record patterns
    // ═══════════════════════════════════════════════════
    static double lineLength(Line line) {
        // Deconstructs Line → Point → coordinates in ONE pattern
        if (line instanceof Line(Point(var x1, var y1), Point(var x2, var y2))) {
            return Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
        }
        return 0;
    }

    public static void main(String[] args) {
        System.out.println(area(new Circle(5)));            // 78.54
        System.out.println(area(new Rectangle(4, 6)));      // 24.0
        System.out.println(describe(new Circle(15)));       // Large circle (r=15.0)
        System.out.println(describe(new Rectangle(5, 5)));  // Square (5.0x5.0)

        var line = new Line(new Point(0, 0), new Point(3, 4));
        System.out.println("Length: " + lineLength(line));   // 5.0
    }
}
```

### Records and Serialization

```java
package com.interview.java14;

import java.io.*;

public class RecordSerializationDemo {

    // ✅ Records have special serialization support
    // - Uses canonical constructor for deserialization (not readObject)
    // - Cannot customize with readObject/writeObject
    // - More secure than regular class serialization
    record User(String name, int age) implements Serializable {
        // Compact constructor validation runs during deserialization too!
        public User {
            if (age < 0) throw new IllegalArgumentException("Age cannot be negative");
        }
    }

    public static void main(String[] args) throws Exception {
        var user = new User("Alice", 30);

        // Serialize
        var baos = new ByteArrayOutputStream();
        try (var oos = new ObjectOutputStream(baos)) {
            oos.writeObject(user);
        }

        // Deserialize — goes through canonical constructor
        var bais = new ByteArrayInputStream(baos.toByteArray());
        try (var ois = new ObjectInputStream(bais)) {
            var deserialized = (User) ois.readObject();
            System.out.println(deserialized);               // User[name=Alice, age=30]
            System.out.println(user.equals(deserialized));  // true
        }
    }
}
```

### Record Restrictions

| Restriction | Reason |
|------------|--------|
| Cannot extend a class (implicitly extends `Record`) | Records are final data carriers |
| All fields are `final` (immutable) | Transparency guarantee |
| Cannot declare instance fields beyond components | Components define the state |
| Implicitly `final` — cannot be subclassed | Data integrity |
| Cannot be `abstract` | Must be instantiable |
| Can implement interfaces | ✅ Allowed |
| Can have static fields and methods | ✅ Allowed |
| Can have instance methods | ✅ Allowed |
| Can be local (declared inside methods) | ✅ Allowed (Java 16+) |
| Can be generic | ✅ Allowed |

### When to Use Records vs When Not To

```java
// ✅ USE records for:
record ApiResponse(int status, String body, Map<String, String> headers) {}
record Coordinate(double lat, double lon) {}
record DateRange(LocalDate start, LocalDate end) {}
record Event(String type, Instant timestamp, Map<String, Object> payload) {}

// ❌ DON'T use records for:
// - Mutable domain entities (JPA entities — need setters, default constructor)
// - Classes needing inheritance hierarchy
// - Classes with complex internal state management
// - DTOs requiring field-level serialization customization
```

### ⚠️ Common Mistake
**Trying to use records as JPA entities**: Records are `final`, have no default constructor, and no setters — all of which JPA requires. Use records for **DTOs, value objects, API responses, and method return types**, not for Hibernate/JPA entities. Also, record accessors are `name()` not `getName()` — this breaks frameworks expecting JavaBean conventions. Some frameworks (Jackson, Spring) have added record support, but not all.

### ⚡ Performance Tip
Records are **value-based classes** designed to benefit from future JVM optimizations (Project Valhalla's value types/inline classes). Their immutability enables safe sharing across threads without synchronization. The auto-generated `hashCode()` computes from all components, making records excellent `Map` keys and `Set` elements. However, records with many components generate hash codes that combine all fields — for records used as map keys in hot paths, consider providing a custom `hashCode()` that uses fewer fields.

### 🔧 JVM Internals
Records compile to regular `final` classes that extend `java.lang.Record`. The `equals()`, `hashCode()`, and `toString()` methods are generated using `invokedynamic` calling `ObjectMethods.bootstrap()` — a runtime-generated implementation that avoids the overhead of hand-coded reflection. Deserialization uses the canonical constructor (not field injection), making record serialization **more secure** than traditional serialization — validation in the constructor runs during deserialization, preventing construction of invalid objects from malicious byte streams.

---

## 🎯 Interviewer Insight
Interviewers test whether you know records beyond the basics. Key differentiators: explaining **compact constructors** and their validation role, knowing that record accessors use `name()` not `getName()`, understanding the serialization advantage (canonical constructor), and being able to discuss **record patterns** (Java 21). Mentioning that records can't be JPA entities shows practical awareness.

## ⚡ Follow-up Questions
1. Can a record have a custom `equals()` method? Should it?
2. How do records differ from Lombok's `@Data` or `@Value`?
3. What is a compact constructor and when is it preferred over the canonical constructor?

## 📚 Related Topics
- [Pattern Matching](./q15-pattern-matching.md)
- [Sealed Classes](./q16-sealed-classes-detail.md)
- [Text Blocks](./q17-text-blocks.md)

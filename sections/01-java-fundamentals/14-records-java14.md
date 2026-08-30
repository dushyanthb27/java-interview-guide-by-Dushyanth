# Q14: What Are Records in Java 14+ and When Should You Use Them?

## 📋 Question
What are Records in Java 14+ and when should you use them? How do they compare to traditional POJOs, and what are their limitations?

## Difficulty: 🟠 Intermediate

---

## 💡 Answer

**Records** (finalized in Java 16, previewed in Java 14) are a special kind of class designed to be **transparent carriers of immutable data**. They dramatically reduce boilerplate by automatically generating: `private final` fields, a canonical constructor, accessor methods (named after the components, not `getXxx()`), `equals()`, `hashCode()`, and `toString()`.

A record declaration like `record Point(int x, int y) {}` replaces approximately 50-70 lines of traditional POJO code. Records embody the principle that **the API of a data carrier class should be derived from its state** — the components you declare ARE the API.

### What Records Generate Automatically

For `record Point(int x, int y) {}`, the compiler generates:
1. **Private final fields**: `private final int x; private final int y;`
2. **Canonical constructor**: `Point(int x, int y)` that assigns parameters to fields
3. **Accessor methods**: `x()` and `y()` (NOT `getX()` and `getY()`)
4. **`equals()`**: Component-wise equality (two Points are equal if `x` and `y` are equal)
5. **`hashCode()`**: Based on all components
6. **`toString()`**: `Point[x=1, y=2]`

### Key Restrictions
- Fields are **always `private final`** — records are immutable by design
- Records **cannot extend classes** (they implicitly extend `java.lang.Record`)
- Records **cannot be abstract** (they represent concrete data)
- Records **cannot declare additional instance fields** (only static fields allowed)
- Records **can implement interfaces**
- Records **can declare static fields, static methods, and instance methods**
- Records **can have custom constructors** (compact, canonical, and custom)

### Code Example

```java
package com.interview.fundamentals;

import java.util.Objects;

// === BASIC RECORD ===
record Point(int x, int y) {
    // That's it! equals(), hashCode(), toString(), x(), y() are all generated
}

// === RECORD WITH VALIDATION (Compact Constructor) ===
record Temperature(double value, String unit) {

    // Compact constructor — no parameter list, no explicit assignments
    // Parameters are implicitly assigned to fields AFTER this block
    Temperature {
        if (unit == null || unit.isBlank()) {
            throw new IllegalArgumentException("Unit cannot be null or blank");
        }
        // Normalize the unit
        unit = unit.toUpperCase();  // Modifies the parameter before auto-assignment

        if (unit.equals("K") && value < 0) {
            throw new IllegalArgumentException("Kelvin cannot be negative");
        }
    }

    // ✅ Custom instance methods
    public Temperature toCelsius() {
        return switch (unit) {
            case "F" -> new Temperature((value - 32) * 5.0 / 9.0, "C");
            case "K" -> new Temperature(value - 273.15, "C");
            case "C" -> this;
            default -> throw new IllegalStateException("Unknown unit: " + unit);
        };
    }

    // ✅ Static factory method
    public static Temperature ofCelsius(double value) {
        return new Temperature(value, "C");
    }

    public static Temperature ofFahrenheit(double value) {
        return new Temperature(value, "F");
    }
}

// === RECORD IMPLEMENTING INTERFACE ===
sealed interface Shape permits Circle, Rectangle {
    double area();
}

record Circle(double radius) implements Shape {
    // Compact constructor with validation
    Circle {
        if (radius < 0) throw new IllegalArgumentException("Radius must be non-negative");
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

record Rectangle(double width, double height) implements Shape {
    Rectangle {
        if (width < 0 || height < 0) {
            throw new IllegalArgumentException("Dimensions must be non-negative");
        }
    }

    @Override
    public double area() {
        return width * height;
    }

    // ✅ Additional instance methods
    public double perimeter() {
        return 2 * (width + height);
    }

    // ✅ Custom toString() override (optional)
    @Override
    public String toString() {
        return "Rectangle(%.1f x %.1f, area=%.1f)".formatted(width, height, area());
    }
}

public class RecordDemo {

    public static void main(String[] args) {
        // === BASIC USAGE ===
        Point p1 = new Point(3, 4);
        Point p2 = new Point(3, 4);

        System.out.println(p1);              // Point[x=3, y=4]
        System.out.println(p1.x());          // 3 (accessor, not getX())
        System.out.println(p1.y());          // 4
        System.out.println(p1.equals(p2));   // true (component-wise equality)
        System.out.println(p1.hashCode() == p2.hashCode());  // true
        System.out.println(p1 == p2);        // false (different objects)

        // === RECORDS WITH VALIDATION ===
        Temperature temp = Temperature.ofFahrenheit(98.6);
        System.out.println(temp);                    // Temperature[value=98.6, unit=F]
        System.out.println(temp.toCelsius());        // Temperature[value=37.0, unit=C]

        // Validation in action
        try {
            new Temperature(-10, "K");  // ❌ Throws IllegalArgumentException
        } catch (IllegalArgumentException e) {
            System.out.println("Caught: " + e.getMessage());
        }

        // === RECORDS WITH INTERFACES ===
        Shape circle = new Circle(5);
        Shape rect = new Rectangle(4, 6);
        System.out.println(circle.area());  // 78.54
        System.out.println(rect.area());    // 24.0
        System.out.println(rect);           // Rectangle(4.0 x 6.0, area=24.0)

        // === RECORDS AS MAP KEYS (excellent due to auto-generated equals/hashCode) ===
        var distances = new java.util.HashMap<Point, Double>();
        distances.put(new Point(0, 0), 0.0);
        distances.put(new Point(3, 4), 5.0);
        System.out.println(distances.get(new Point(3, 4)));  // 5.0 (works because of equals/hashCode)
    }
}
```

### Record vs POJO Comparison

```java
// === TRADITIONAL POJO — ~60 lines ===
class PointPojo {
    private final int x;
    private final int y;

    public PointPojo(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int getX() { return x; }
    public int getY() { return y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        PointPojo that = (PointPojo) o;
        return x == that.x && y == that.y;
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);
    }

    @Override
    public String toString() {
        return "PointPojo{x=" + x + ", y=" + y + "}";
    }
}

// === RECORD — 1 line ===
record PointRecord(int x, int y) {}

// Same functionality, 1/60th the code!
```

### When NOT to Use Records

```java
// ❌ Records are NOT suitable for:

// 1. Mutable objects (entities with setters)
//    Use class: JPA entities need setters and no-arg constructor
//    record User(String name) {}  // Cannot set name after creation

// 2. Objects with additional hidden state
//    Records: state = components only. Cannot add instance fields.
//    record Bad(int x) { int hidden = 0; }  // ❌ Compile error

// 3. Objects requiring inheritance
//    record Base(int x) {}
//    record Child(int x, int y) extends Base(x) {}  // ❌ Compile error

// 4. JPA/Hibernate entities (as of 2024)
//    JPA requires: no-arg constructor, mutable fields, proxying
//    Records don't support any of these
//    ✅ Use records for DTOs and projections, not entities

// ✅ Records ARE ideal for:
// - DTOs (Data Transfer Objects)
// - API request/response payloads
// - Value objects (DDD)
// - Composite map keys
// - Return types for methods that need to return multiple values
// - Event payloads
// - Configuration/settings snapshots
```

### ⚠️ Common Mistake
**Using records as JPA entities**: Records are immutable and have no no-arg constructor, making them incompatible with JPA/Hibernate entity requirements. Use records for **DTOs and projections**, not entities. Another mistake: assuming records are deeply immutable. If a record component is a mutable object (like a `List`), the record holds an immutable **reference** to a mutable object — the list contents can still be modified. For true deep immutability, use `List.of()` or `Collections.unmodifiableList()` in the compact constructor.

```java
// ❌ Shallow immutability trap
record Team(String name, List<String> members) {}

var members = new ArrayList<>(List.of("Alice", "Bob"));
var team = new Team("Alpha", members);
members.add("Charlie");  // Modifies the list inside the record!

// ✅ Defensive copy in compact constructor
record SafeTeam(String name, List<String> members) {
    SafeTeam {
        members = List.copyOf(members);  // Deep copy, immutable
    }
}
```

### ⚡ Performance Tip
Records have the **same runtime performance** as equivalent hand-written classes — the compiler generates identical bytecode for constructors, accessors, equals, and hashCode. There is no reflection or proxy overhead. Records are excellent as **HashMap keys** because their auto-generated `equals()` and `hashCode()` are correct and consistent by definition, eliminating a common source of bugs. Since record components are `final`, the JVM's escape analysis and scalar replacement optimizations apply well — short-lived records may be entirely stack-allocated by the JIT compiler.

---

## 🎯 Interviewer Insight
Interviewers ask about records to assess your knowledge of modern Java. They want you to articulate the **trade-offs**: what records give you (conciseness, correctness, immutability) and what they restrict (no inheritance, no mutable state, no extra fields). Strong candidates know that records use `invokedynamic` for equals/hashCode/toString, understand the compact constructor syntax, and can explain when records are inappropriate (JPA entities, mutable objects).

## ⚡ Follow-up Questions
1. How do records interact with serialization, and why are they considered safer than regular classes for serialization?
2. Can you customize the `equals()` and `hashCode()` of a record? Should you?
3. How do record patterns work in Java 21+ switch expressions?

## 📚 Related Topics
- [Sealed Classes](./15-sealed-classes-java15.md)
- [Final Keyword](./11-final-keyword.md)
- [Enums in Java](./12-enums-in-java.md)

# Q15: Sealed Classes in Java 17+ - Advanced Usage

## 📋 Question
Explain advanced usage of sealed classes in Java 17+, including the permits clause, exhaustive pattern matching, and integration with records.

## Difficulty: 🟠 Intermediate

---

## 💡 Answer

Sealed classes in Java 17+ provide controlled inheritance, enabling exhaustive pattern matching and type-safe hierarchies.

### Basic Sealed Class

```java
// Sealed class with permitted subclasses
sealed class Shape permits Circle, Rectangle, Triangle {
    abstract double area();
}

final class Circle extends Shape {
    private final double radius;
    
    Circle(double radius) {
        this.radius = radius;
    }
    
    @Override
    double area() {
        return Math.PI * radius * radius;
    }
}

final class Rectangle extends Shape {
    private final double width, height;
    
    Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
    
    @Override
    double area() {
        return width * height;
    }
}
```

### Sealed Interfaces

```java
// Sealed interface with permitted implementations
sealed interface Expression permits Literal, Add, Multiply {
    double evaluate();
}

record Literal(double value) implements Expression {
    @Override
    public double evaluate() {
        return value;
    }
}

record Add(Expression left, Expression right) implements Expression {
    @Override
    public double evaluate() {
        return left.evaluate() + right.evaluate();
    }
}
```

### Mixed Modifiers

```java
// Sealed class with final, sealed, and non-sealed subclasses
sealed class Vehicle permits Car, Truck, Motorcycle {
    abstract int numberOfWheels();
}

final class Motorcycle extends Vehicle {
    @Override
    int numberOfWheels() { return 2; }
}

sealed class Car extends Vehicle permits Sedan, SUV {
    @Override
    int numberOfWheels() { return 4; }
}

final class Sedan extends Car { }

non-sealed class Truck extends Vehicle {
    @Override
    int numberOfWheels() { return 6; }
}

// Anyone can extend Truck since it's non-sealed
class PickupTruck extends Truck {
    @Override
    int numberOfWheels() { return 4; }
}
```

### Exhaustive Pattern Matching

```java
// The compiler ensures all permitted types are handled
static String describe(Shape shape) {
    return switch (shape) {
        case Circle c -> "Circle with radius " + c.radius;
        case Rectangle r -> "Rectangle " + r.width + "x" + r.height;
        case Triangle t -> "Triangle";
        // No default needed - compiler enforces exhaustiveness
    };
}
```

### Key Rules

1. **Same package/module**: Permitted subclasses must be in the same package or module
2. **Three modifiers**: `final`, `sealed`, or `non-sealed`
3. **Implicit permits**: If all subclasses are in the same file, permits can be omitted
4. **Interface support**: Both classes and interfaces can be sealed

### Sealed Classes vs Enums

| Feature | Sealed Classes | Enums |
|---------|---------------|-------|
| Different state per type | ✅ | ❌ |
| Multiple instances per type | ✅ | ❌ (singletons) |
| Inheritance | ✅ (controlled) | ❌ |
| Pattern matching | ✅ | ✅ |
| Use case | Different data/behavior | Fixed set of values |

### Use Cases

✅ **Good for:**
- Domain modeling with fixed type sets
- API response types (Success, Error, Loading)
- Expression trees (AST nodes)
- State machines with fixed states

❌ **Not for:**
- Open-ended hierarchies
- Plugin systems
- Frameworks requiring extension

---

## 🎯 Interviewer Insight
Interviewers want to see that you understand sealed classes enable exhaustive pattern matching and type-safe hierarchies. Strong candidates can explain the three modifiers (final, sealed, non-sealed) and when to use sealed classes vs enums.

## ⚡ Follow-up Questions
1. What happens if you try to extend a sealed class without being in the permits clause?
2. Can sealed classes be used across different modules?
3. How do sealed classes improve pattern matching?

## 📚 Related Topics
- [Sealed Classes in Java 15+](../01-java-fundamentals/15-sealed-classes-java15.md)
- [Records](../01-java-fundamentals/14-records-java14.md)
- [Pattern Matching](./15-pattern-matching.md)

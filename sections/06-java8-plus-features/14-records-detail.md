# Q14: Records in Java 16+ - Advanced Usage

## 📋 Question
Explain advanced usage of records in Java 16+, including compact constructors, custom methods, and integration with pattern matching.

## Difficulty: 🟠 Intermediate

---

## 💡 Answer

Records in Java 16+ provide advanced features beyond simple data carriers, including compact constructors, custom methods, and integration with pattern matching.

### Compact Constructors

```java
// Compact constructor - no parameter list, no explicit assignments
record Point(int x, int y) {
    Point {
        if (x < 0 || y < 0) {
            throw new IllegalArgumentException("Coordinates must be non-negative");
        }
        // Parameters are automatically assigned to fields after this block
    }
}
```

### Custom Methods in Records

```java
record Rectangle(double width, double height) {
    // Custom instance method
    public double area() {
        return width * height;
    }

    // Custom static factory method
    public static Rectangle square(double side) {
        return new Rectangle(side, side);
    }

    // Override toString()
    @Override
    public String toString() {
        return "Rectangle[%.1f x %.1f, area=%.1f]".formatted(width, height, area());
    }
}
```

### Records with Interfaces

```java
interface Shape {
    double area();
}

record Circle(double radius) implements Shape {
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

record Square(double side) implements Shape {
    @Override
    public double area() {
        return side * side;
    }
}
```

### Pattern Matching with Records (Java 21+)

```java
// Destructuring pattern matching
static void printShape(Shape shape) {
    switch (shape) {
        case Circle(double r) -> 
            System.out.printf("Circle with radius %.1f, area %.2f%n", r, shape.area());
        case Square(double s) -> 
            System.out.printf("Square with side %.1f, area %.2f%n", s, shape.area());
    }
}
```

### Records vs Traditional Classes

| Feature | Record | Traditional Class |
|---------|--------|-------------------|
| Boilerplate | Minimal | High |
| Immutability | Built-in | Manual |
| equals/hashCode | Auto-generated | Manual |
| Pattern matching | Supported | Limited |
| Inheritance | No extension | Full support |

### Use Cases

✅ **Good for:**
- DTOs (Data Transfer Objects)
- API request/response objects
- Value objects in DDD
- Configuration objects
- Tuple-like return values

❌ **Not for:**
- JPA entities (need mutability)
- Objects with complex inheritance
- Classes requiring additional hidden state

---

## 🎯 Interviewer Insight
Interviewers want to see that you understand records beyond the basics - compact constructors, custom methods, and pattern matching integration. Strong candidates can explain when to use records vs traditional classes and understand the immutability guarantees.

## ⚡ Follow-up Questions
1. Can records have additional instance fields beyond the components?
2. How do compact constructors differ from canonical constructors?
3. When would you choose a record over a traditional class?

## 📚 Related Topics
- [Records in Java 14+](../01-java-fundamentals/14-records-java14.md)
- [Sealed Classes](../01-java-fundamentals/15-sealed-classes-java15.md)
- [Pattern Matching](./15-pattern-matching.md)

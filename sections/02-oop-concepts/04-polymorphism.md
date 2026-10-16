# Q4: Explain Polymorphism with Examples

## 📋 Question
What is polymorphism in Java? Explain compile-time and runtime polymorphism, dynamic method dispatch, and covariant return types with examples.

## Difficulty: 🟡 Intermediate

---

## Answer

**Polymorphism** (Greek for "many forms") allows objects to be treated as instances of their parent type while retaining their actual behavior. It is one of the four pillars of OOP.

### Two Types of Polymorphism

| Type | Mechanism | Binding | Also Called |
|------|-----------|---------|-------------|
| Compile-time | Method overloading | Static (at compile time) | Static polymorphism |
| Runtime | Method overriding | Dynamic (at runtime) | Dynamic polymorphism |

### 1. Compile-Time Polymorphism (Overloading)

The compiler decides which method to call based on the **method signature**:

```java
public class Calculator {
    // ✅ Same name, different parameter lists
    public int add(int a, int b) {
        return a + b;
    }

    public double add(double a, double b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }

    public String add(String a, String b) {
        return a + b; // Concatenation
    }
}
```

### 2. Runtime Polymorphism (Overriding + Dynamic Dispatch)

The JVM decides which method to call based on the **actual object type** at runtime:

```java
public abstract class Shape {
    protected String color;

    public Shape(String color) {
        this.color = color;
    }

    // Method to be overridden
    public abstract double area();

    @Override
    public String toString() {
        return "%s %s (area=%.2f)".formatted(color, getClass().getSimpleName(), area());
    }
}

public class Circle extends Shape {
    private final double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

public class Rectangle extends Shape {
    private final double width, height;

    public Rectangle(String color, double width, double height) {
        super(color);
        this.width = width;
        this.height = height;
    }

    @Override
    public double area() {
        return width * height;
    }
}
```

### Dynamic Method Dispatch

```java
public class PolymorphismDemo {
    public static void main(String[] args) {
        // Reference type: Shape | Actual type: Circle, Rectangle
        List<Shape> shapes = List.of(
            new Circle("Red", 5.0),
            new Rectangle("Blue", 4.0, 6.0),
            new Circle("Green", 3.0)
        );

        // ✅ Dynamic dispatch: JVM calls the correct area() at runtime
        for (Shape shape : shapes) {
            System.out.println(shape); // Calls correct toString() -> area()
        }

        // Output:
        // Red Circle (area=78.54)
        // Blue Rectangle (area=24.00)
        // Green Circle (area=28.27)
    }

    // ✅ Polymorphic method - works with ANY Shape subtype
    public static double totalArea(List<Shape> shapes) {
        return shapes.stream()
                     .mapToDouble(Shape::area)
                     .sum();
    }
}
```

### How It Works: Virtual Method Table (vtable)

```
Shape reference ──▶ Circle Object
                    ┌─────────────────────┐
                    │ vtable pointer ──────┼──▶ ┌─────────────┐
                    │ color: "Red"         │    │ area() ──▶ Circle.area()
                    │ radius: 5.0          │    │ toString()──▶ Shape.toString()
                    └─────────────────────┘    └─────────────┘

Shape reference ──▶ Rectangle Object
                    ┌─────────────────────┐
                    │ vtable pointer ──────┼──▶ ┌─────────────┐
                    │ color: "Blue"        │    │ area() ──▶ Rectangle.area()
                    │ width: 4.0           │    │ toString()──▶ Shape.toString()
                    │ height: 6.0          │    └─────────────┘
                    └─────────────────────┘
```

Each class has a virtual method table that maps method signatures to actual implementations. The JVM uses the vtable to dispatch calls to the correct method at runtime.

### Covariant Return Types

A subclass can override a method and return a **more specific type**:

```java
public class AnimalFactory {
    public Animal create() {
        return new Animal("Generic");
    }
}

public class DogFactory extends AnimalFactory {
    // ✅ Covariant return: Dog is a subtype of Animal
    @Override
    public Dog create() {
        return new Dog("Buddy", 3, "Labrador");
    }
}
```

### Polymorphism with Interfaces

```java
public interface Notification {
    void send(String message, String recipient);
}

public class EmailNotification implements Notification {
    @Override
    public void send(String message, String recipient) {
        System.out.println("Email to " + recipient + ": " + message);
    }
}

public class SmsNotification implements Notification {
    @Override
    public void send(String message, String recipient) {
        System.out.println("SMS to " + recipient + ": " + message);
    }
}

public class PushNotification implements Notification {
    @Override
    public void send(String message, String recipient) {
        System.out.println("Push to " + recipient + ": " + message);
    }
}

// ✅ Client code depends on the interface, not the implementation
public class NotificationService {
    private final List<Notification> channels;

    public NotificationService(List<Notification> channels) {
        this.channels = channels;
    }

    public void notifyAll(String message, String recipient) {
        channels.forEach(ch -> ch.send(message, recipient));
    }
}
```

### Pattern Matching with Polymorphism (Java 17+)

```java
public static String describeShape(Shape shape) {
    // ✅ Pattern matching for instanceof (Java 16+)
    if (shape instanceof Circle c) {
        return "Circle with radius: " + c.getRadius();
    } else if (shape instanceof Rectangle r) {
        return "Rectangle %s x %s".formatted(r.getWidth(), r.getHeight());
    }
    return "Unknown shape";
}
```

### ⚠️ Common Mistake
**Confusing reference type with object type**: You can only call methods that exist in the reference type, not the actual object type. This leads to compile errors when trying to access subclass-specific methods through a superclass reference.

```java
// ❌ Confusing reference type with object type
Animal animal = new Dog("Rex", 5, "Shepherd");
// animal.fetch();  // COMPILE ERROR - Animal reference doesn't see Dog methods

// ✅ Cast when you need subclass-specific behavior
if (animal instanceof Dog dog) {
    dog.fetch(); // Now accessible
}
```

### ⚡ Performance Tip
Virtual method dispatch has negligible overhead in modern JVMs. The JIT compiler aggressively inlines monomorphic (single-target) and bimorphic (two-target) call sites, making polymorphic calls nearly as fast as direct calls.

---

## 🎯 Interviewer Insight
Interviewers want to see you distinguish between compile-time and runtime polymorphism clearly. Bonus points for explaining the vtable mechanism, covariant returns, and demonstrating real-world use cases like the Strategy or Template patterns.

## ⚡ Follow-up Questions
1. Can you achieve polymorphism without inheritance? If so, how?
2. What is the difference between method hiding (static methods) and method overriding (instance methods)?
3. How does the JIT compiler optimize polymorphic call sites through inlining and devirtualization?
4. What is the difference between static binding (compile-time) and dynamic binding (runtime)?
5. How do pattern matching (Java 16+) and sealed classes (Java 17+) change polymorphism patterns?

## 📚 Related Topics
- [Method Overloading](./08-method-overloading.md)
- [Method Overriding](./09-method-overriding.md)
- [Interfaces vs Abstract Classes](./07-interfaces-vs-abstract-classes.md)

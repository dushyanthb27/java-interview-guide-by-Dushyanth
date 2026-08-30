# Q15: What Are Sealed Classes in Java 15+ and What Problems Do They Solve?

## 📋 Question
What are Sealed Classes in Java 15+ and what problems do they solve? How do they combine with records and pattern matching? When should you use sealed classes vs enums vs interfaces?

## Difficulty: 🟠 Intermediate

---

## 💡 Answer

**Sealed classes** (finalized in Java 17, previewed in Java 15) allow a class or interface to **explicitly control which classes can extend or implement it** using the `permits` clause. This fills a gap in Java's type system: before sealed classes, you could either allow **any** class to extend yours (`public class`) or **none** (`final class`) — there was no middle ground for "only these specific classes."

Sealed classes solve the problem of **restricted type hierarchies** — situations where you know the complete set of subtypes and want the compiler to enforce this constraint. They are the foundation for **algebraic data types (ADTs)** in Java, enabling exhaustive pattern matching in `switch` expressions.

### Key Rules
1. A sealed class/interface uses the `sealed` modifier and `permits` clause to list allowed subtypes
2. Permitted subclasses must be in the **same package** (or same module) as the sealed parent
3. Each permitted subclass must be exactly one of:
   - `final` — cannot be extended further
   - `sealed` — continues the sealed hierarchy
   - `non-sealed` — reopens for arbitrary extension
4. If all permitted subclasses are in the same source file, the `permits` clause can be **omitted** (compiler infers it)

### The Hierarchy Modifiers

```
sealed class Shape permits Circle, Rectangle, Triangle
   │
   ├── final class Circle          → Cannot be extended (leaf)
   ├── sealed class Rectangle      → Can only be extended by permitted classes
   │   └── final class Square      → Leaf
   └── non-sealed class Triangle   → Open for extension by anyone
       ├── class RightTriangle     → Anyone can extend
       └── class EquilateralTriangle
```

### Code Example

```java
package com.interview.fundamentals;

// === SEALED INTERFACE WITH RECORDS (Algebraic Data Type) ===
// This models a complete, closed set of expression types

sealed interface Expression permits
    Literal, Add, Multiply, Negate {
    // Common method for all expressions
    double evaluate();
}

// ✅ final record — leaf type, cannot be extended
record Literal(double value) implements Expression {
    @Override
    public double evaluate() {
        return value;
    }
}

// ✅ final record — leaf type
record Add(Expression left, Expression right) implements Expression {
    @Override
    public double evaluate() {
        return left.evaluate() + right.evaluate();
    }
}

// ✅ final record — leaf type
record Multiply(Expression left, Expression right) implements Expression {
    @Override
    public double evaluate() {
        return left.evaluate() * right.evaluate();
    }
}

// ✅ final record — leaf type
record Negate(Expression operand) implements Expression {
    @Override
    public double evaluate() {
        return -operand.evaluate();
    }
}

// ❌ Cannot add new Expression types without modifying the sealed interface
// record Subtract(...) implements Expression { }  // Compile error!

public class SealedClassDemo {

    public static void main(String[] args) {
        // Build expression: -(3 + 4) * 2
        Expression expr = new Multiply(
            new Negate(
                new Add(new Literal(3), new Literal(4))
            ),
            new Literal(2)
        );

        System.out.println("Result: " + expr.evaluate());  // -14.0

        // === EXHAUSTIVE PATTERN MATCHING (Java 21+) ===
        System.out.println("Formatted: " + format(expr));

        // === EXHAUSTIVE SWITCH — compiler ensures ALL cases are handled ===
        printType(new Literal(42));
        printType(new Add(new Literal(1), new Literal(2)));
    }

    // ✅ Pattern matching with sealed types — no default needed!
    // The compiler KNOWS all possible subtypes
    static String format(Expression expr) {
        return switch (expr) {
            case Literal(var v)             -> String.valueOf(v);
            case Add(var l, var r)          -> "(%s + %s)".formatted(format(l), format(r));
            case Multiply(var l, var r)     -> "(%s * %s)".formatted(format(l), format(r));
            case Negate(var op)             -> "(-%s)".formatted(format(op));
            // No default needed — all cases are covered (exhaustive!)
            // If you add a new permitted class, this won't compile until you handle it
        };
    }

    // ✅ Exhaustive switch with type patterns
    static void printType(Expression expr) {
        switch (expr) {
            case Literal l    -> System.out.println("Literal: " + l.value());
            case Add a        -> System.out.println("Addition");
            case Multiply m   -> System.out.println("Multiplication");
            case Negate n     -> System.out.println("Negation");
            // Compiler error if any permitted type is missing
        }
    }
}
```

### Sealed Class with Mixed Subtypes

```java
package com.interview.fundamentals;

// === SEALED CLASS WITH final, sealed, AND non-sealed subtypes ===

sealed abstract class Vehicle permits Car, Truck, Motorcycle {
    private final String make;
    private final int year;

    protected Vehicle(String make, int year) {
        this.make = make;
        this.year = year;
    }

    public String make() { return make; }
    public int year() { return year; }
    public abstract int numberOfWheels();
}

// final — no further extension allowed
final class Motorcycle extends Vehicle {
    public Motorcycle(String make, int year) {
        super(make, year);
    }

    @Override
    public int numberOfWheels() { return 2; }
}

// sealed — restricted further extension
sealed class Car extends Vehicle permits Sedan, SUV {
    private final int doors;

    public Car(String make, int year, int doors) {
        super(make, year);
        this.doors = doors;
    }

    public int doors() { return doors; }

    @Override
    public int numberOfWheels() { return 4; }
}

final class Sedan extends Car {
    public Sedan(String make, int year) {
        super(make, year, 4);
    }
}

final class SUV extends Car {
    private final boolean allWheelDrive;

    public SUV(String make, int year, boolean allWheelDrive) {
        super(make, year, 5);
        this.allWheelDrive = allWheelDrive;
    }

    public boolean isAllWheelDrive() { return allWheelDrive; }
}

// non-sealed — open for extension by anyone
non-sealed class Truck extends Vehicle {
    public Truck(String make, int year) {
        super(make, year);
    }

    @Override
    public int numberOfWheels() { return 6; }
}

// ✅ Anyone can extend Truck (it's non-sealed)
class PickupTruck extends Truck {
    public PickupTruck(String make, int year) {
        super(make, year);
    }

    @Override
    public int numberOfWheels() { return 4; }
}
```

### Sealed Classes vs Enums vs Interfaces

```
┌────────────────────────┬──────────────┬─────────────────┬───────────────────┐
│ Feature                │ Enum         │ Sealed Class    │ Interface         │
├────────────────────────┼──────────────┼─────────────────┼───────────────────┤
│ Fixed set of types     │ ✅ (values)  │ ✅ (permits)    │ ❌ (open)         │
│ Can have state         │ ✅ (limited) │ ✅ (full)       │ ❌ (no instance)  │
│ Different fields/type  │ ❌ (same)    │ ✅ (per subtype)│ N/A               │
│ Pattern matching       │ ✅ (values)  │ ✅ (types+data) │ ❌ (no exhaust.)  │
│ Exhaustive switch      │ ✅           │ ✅              │ ❌ (need default) │
│ Singleton instances    │ ✅           │ ❌              │ N/A               │
│ Inheritance            │ ❌           │ ✅ (multi-level)│ ✅                │
│ Can carry data         │ ✅ (uniform) │ ✅ (per subtype)│ N/A               │
└────────────────────────┴──────────────┴─────────────────┴───────────────────┘

Use ENUM when:
  → Fixed set of SINGLETON values with the SAME structure (Season, Color, HttpStatus)

Use SEALED CLASS when:
  → Fixed set of TYPES, each with DIFFERENT state/behavior (Shape→Circle, Rectangle)
  → Need pattern matching with data extraction (destructuring)

Use INTERFACE when:
  → Open for extension by anyone (Collection, Comparable, Serializable)
  → You don't know or control all implementations
```

### Real-World Use Cases

```java
// ✅ Use Case 1: API Response modeling
sealed interface ApiResponse<T> permits Success, Failure, Loading {
}

record Success<T>(T data, int statusCode) implements ApiResponse<T> {}
record Failure<T>(String errorMessage, int statusCode) implements ApiResponse<T> {}
record Loading<T>() implements ApiResponse<T> {}

// ✅ Use Case 2: Payment method domain modeling
sealed interface PaymentMethod permits CreditCard, BankTransfer, DigitalWallet {
    String displayName();
}

record CreditCard(String number, String expiry, String cvv) implements PaymentMethod {
    public String displayName() { return "Card ending " + number.substring(number.length() - 4); }
}

record BankTransfer(String iban, String swift) implements PaymentMethod {
    public String displayName() { return "Bank " + iban.substring(0, 8) + "..."; }
}

record DigitalWallet(String provider, String email) implements PaymentMethod {
    public String displayName() { return provider + " (" + email + ")"; }
}

// ✅ Usage with exhaustive switch
// static String processPayment(PaymentMethod method) {
//     return switch (method) {
//         case CreditCard cc    -> "Charging card " + cc.displayName();
//         case BankTransfer bt  -> "Transferring via " + bt.displayName();
//         case DigitalWallet dw -> "Paying via " + dw.displayName();
//     };
// }
```

### ⚠️ Common Mistake
**Using `non-sealed` when you actually want `final`**: Making a subclass `non-sealed` reopens the hierarchy, allowing anyone to extend it and defeating the purpose of the sealed parent. Use `non-sealed` only when you intentionally want to allow unrestricted extension of a specific branch. Another common mistake: trying to use sealed classes across different packages or modules without understanding the co-location constraint — all permitted subtypes must be in the **same package** (or accessible in the same module). This constraint ensures the compiler can verify exhaustiveness.

### ⚡ Performance Tip
Sealed classes enable the JVM to perform **exhaustive type checking at compile time**, which means `switch` expressions over sealed types don't need a `default` branch. This eliminates dead-code paths and allows the JIT compiler to optimize the type dispatch more aggressively. Record patterns combined with sealed types enable the JIT to perform **scalar replacement** — decomposing record objects into their component values on the stack, avoiding heap allocation entirely. In performance-critical path processing (like AST evaluation or protocol parsing), sealed hierarchies with records can be as fast as manual `if-else instanceof` chains while being dramatically more maintainable.

---

## 🎯 Interviewer Insight
Sealed classes test a candidate's knowledge of **modern Java type system design**. Interviewers want to see that you understand the problem sealed classes solve (the gap between `final` and fully open extension), how they enable exhaustive pattern matching, and the synergy with records for algebraic data types. Strong candidates can articulate the three modifiers (`final`, `sealed`, `non-sealed`) and explain when to choose sealed classes over enums. Demonstrating real-world use cases (API response types, domain modeling) and connecting sealed classes to functional programming concepts (ADTs, discriminated unions) shows both depth and breadth.

## ⚡ Follow-up Questions
1. How do sealed classes interact with the module system (JPMS)?
2. Can a sealed interface have `non-sealed` implementations? What are the implications?
3. How would you model a binary tree using sealed classes and records?

## 📚 Related Topics
- [Records in Java 14+](./14-records-java14.md)
- [Enums in Java](./12-enums-in-java.md)
- [Control Flow — Switch Expressions](./05-control-flow-statements.md)

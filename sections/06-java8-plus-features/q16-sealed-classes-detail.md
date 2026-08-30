# Q16: Sealed Classes and Interfaces

## 📋 Question
What are sealed classes and interfaces in Java? Explain the sealed keyword, permits clause, permitted subclass modifiers, combining sealed types with records, exhaustive switch, and algebraic data types.

## Difficulty: 🟠 Senior

---

## 💡 Answer

**Sealed classes** (Java 15 preview, Java 17 final) restrict which classes or interfaces can extend or implement them. Using the `sealed` keyword with a `permits` clause, you explicitly declare the **complete set of subtypes** — the compiler enforces that no other class can extend the sealed type. This enables **exhaustive pattern matching** and models **closed type hierarchies** (algebraic data types) in Java.

### Basic Sealed Class Syntax

```java
package com.interview.java17;

// ✅ Sealed class — only Circle, Rectangle, Triangle can extend it
public sealed class Shape permits Circle, Rectangle, Triangle {
    // Common behavior
    public abstract double area();
    public abstract double perimeter();
}

// ✅ final — cannot be further extended
final class Circle extends Shape {
    private final double radius;

    Circle(double radius) { this.radius = radius; }

    @Override
    public double area() { return Math.PI * radius * radius; }

    @Override
    public double perimeter() { return 2 * Math.PI * radius; }
}

// ✅ non-sealed — open for extension by anyone
non-sealed class Rectangle extends Shape {
    private final double width, height;

    Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public double area() { return width * height; }

    @Override
    public double perimeter() { return 2 * (width + height); }
}

// ✅ sealed — can be further restricted with its own permits
sealed class Triangle extends Shape permits EquilateralTriangle, RightTriangle {
    protected final double base, height2;

    Triangle(double base, double height) {
        this.base = base;
        this.height2 = height;
    }

    @Override
    public double area() { return 0.5 * base * height2; }

    @Override
    public double perimeter() { return base + height2 + Math.sqrt(base * base + height2 * height2); }
}

final class EquilateralTriangle extends Triangle {
    EquilateralTriangle(double side) { super(side, side * Math.sqrt(3) / 2); }
}

final class RightTriangle extends Triangle {
    RightTriangle(double base, double height) { super(base, height); }
}
```

### Permitted Subclass Rules

| Modifier | Meaning | Who Can Extend? |
|----------|---------|-----------------|
| `final` | Hierarchy ends here | Nobody |
| `sealed` | Further restricts subclasses | Only its own permitted subclasses |
| `non-sealed` | Opens up the hierarchy | Anyone |

Every permitted subclass **must** use one of these three modifiers — no class can simply `extend` a sealed class without declaring its own extension policy.

```java
package com.interview.java17;

public class SealedRulesDemo {

    // ✅ Permitted subclasses must be in the same module (or same file for unnamed modules)
    // ✅ If all subtypes are in the same file, permits clause is optional
    sealed interface Payment {
        // permits CreditCard, DebitCard, BankTransfer — inferred from same file
    }

    record CreditCard(String number, String expiry) implements Payment {}
    record DebitCard(String number, String pin) implements Payment {}
    record BankTransfer(String iban, String bic) implements Payment {}
    // No other class can implement Payment
}
```

### Sealed Interfaces with Records — The Power Combo

```java
package com.interview.java17;

public class SealedWithRecordsDemo {

    // ═══════════════════════════════════════════════════
    // Algebraic Data Type: Expression tree
    // ═══════════════════════════════════════════════════
    sealed interface Expr permits Literal, Add, Multiply, Negate {
        // No methods needed — pattern matching handles everything
    }

    record Literal(double value) implements Expr {}
    record Add(Expr left, Expr right) implements Expr {}
    record Multiply(Expr left, Expr right) implements Expr {}
    record Negate(Expr operand) implements Expr {}

    // ✅ Exhaustive evaluation using pattern matching switch
    static double evaluate(Expr expr) {
        return switch (expr) {
            case Literal(var value)       -> value;
            case Add(var left, var right) -> evaluate(left) + evaluate(right);
            case Multiply(var l, var r)   -> evaluate(l) * evaluate(r);
            case Negate(var operand)      -> -evaluate(operand);
            // No default needed! Compiler knows all cases are covered
        };
    }

    // ✅ Exhaustive pretty printing
    static String prettyPrint(Expr expr) {
        return switch (expr) {
            case Literal(var v)           -> String.valueOf(v);
            case Add(var l, var r)        -> "(" + prettyPrint(l) + " + " + prettyPrint(r) + ")";
            case Multiply(var l, var r)   -> "(" + prettyPrint(l) + " * " + prettyPrint(r) + ")";
            case Negate(var op)           -> "-(" + prettyPrint(op) + ")";
        };
    }

    public static void main(String[] args) {
        // Expression: -(3 + 4) * 2
        Expr expr = new Multiply(
                new Negate(new Add(new Literal(3), new Literal(4))),
                new Literal(2)
        );

        System.out.println(prettyPrint(expr));    // (-(3.0 + 4.0) * 2.0)
        System.out.println(evaluate(expr));       // -14.0
    }
}
```

### Exhaustive Switch

```java
package com.interview.java17;

public class ExhaustiveSwitchDemo {

    sealed interface HttpResponse permits Success, ClientError, ServerError {}
    record Success(int code, String body) implements HttpResponse {}
    record ClientError(int code, String message) implements HttpResponse {}
    record ServerError(int code, String message, Exception cause) implements HttpResponse {}

    // ✅ Switch expression is EXHAUSTIVE — all sealed subtypes covered
    static String handleResponse(HttpResponse response) {
        return switch (response) {
            case Success(var code, var body)
                    -> "✅ " + code + ": " + body;
            case ClientError(var code, var msg) when code == 404
                    -> "🔍 Not found: " + msg;
            case ClientError(var code, var msg)
                    -> "❌ Client error " + code + ": " + msg;
            case ServerError(var code, var msg, var cause)
                    -> "🔥 Server error " + code + ": " + msg + " (" + cause.getMessage() + ")";
        };
    }

    // ✅ If a new subtype is added to HttpResponse without updating this switch,
    //    the compiler emits an ERROR — forces you to handle the new case!

    public static void main(String[] args) {
        System.out.println(handleResponse(new Success(200, "OK")));
        System.out.println(handleResponse(new ClientError(404, "User")));
        System.out.println(handleResponse(
                new ServerError(500, "Internal", new RuntimeException("DB down"))));
    }
}
```

### Real-World Domain Modeling

```java
package com.interview.java17;

import java.time.LocalDate;
import java.util.*;
import java.util.stream.*;

public class DomainModelingDemo {

    // ═══════════════════════════════════════════════════
    // Result type — success or failure (like Rust's Result<T, E>)
    // ═══════════════════════════════════════════════════
    sealed interface Result<T> permits Ok, Err {
        default <R> Result<R> map(java.util.function.Function<T, R> fn) {
            return switch (this) {
                case Ok<T>(var value) -> new Ok<>(fn.apply(value));
                case Err<T>(var error) -> new Err<>(error);
            };
        }

        default T orElse(T fallback) {
            return switch (this) {
                case Ok<T>(var value) -> value;
                case Err<T> e -> fallback;
            };
        }
    }

    record Ok<T>(T value) implements Result<T> {}
    record Err<T>(String error) implements Result<T> {}

    // ═══════════════════════════════════════════════════
    // State machine for order processing
    // ═══════════════════════════════════════════════════
    sealed interface OrderState
            permits Pending, Confirmed, Shipped, Delivered, Cancelled {}

    record Pending(LocalDate created) implements OrderState {}
    record Confirmed(LocalDate confirmed, String paymentId) implements OrderState {}
    record Shipped(LocalDate shipped, String trackingNumber) implements OrderState {}
    record Delivered(LocalDate delivered, String signature) implements OrderState {}
    record Cancelled(LocalDate cancelled, String reason) implements OrderState {}

    static String describeOrder(OrderState state) {
        return switch (state) {
            case Pending(var date)
                    -> "Order pending since " + date;
            case Confirmed(var date, var paymentId)
                    -> "Confirmed on " + date + " (payment: " + paymentId + ")";
            case Shipped(var date, var tracking)
                    -> "Shipped on " + date + " (tracking: " + tracking + ")";
            case Delivered(var date, var sig)
                    -> "Delivered on " + date + " (signed by: " + sig + ")";
            case Cancelled(var date, var reason)
                    -> "Cancelled on " + date + " (" + reason + ")";
        };
    }

    public static void main(String[] args) {
        // Result type usage
        Result<Integer> success = new Ok<>(42);
        Result<Integer> failure = new Err<>("Not found");

        System.out.println(success.map(x -> x * 2).orElse(0));  // 84
        System.out.println(failure.map(x -> x * 2).orElse(0));  // 0

        // State machine usage
        OrderState state = new Shipped(LocalDate.now(), "TRK-12345");
        System.out.println(describeOrder(state));
    }
}
```

### Sealed vs Other Approaches

| Approach | Extensibility | Exhaustiveness | Type Safety |
|----------|--------------|----------------|-------------|
| **Sealed** | Closed (controlled) | ✅ Compiler-enforced | ✅ Full |
| **Abstract class** | Open (anyone can extend) | ❌ No guarantee | ✅ Full |
| **Interface** | Open (anyone implements) | ❌ No guarantee | ✅ Full |
| **Enum** | Closed (fixed constants) | ✅ Yes | ✅ But no hierarchy |
| **Visitor pattern** | Manually closed | ⚠️ Runtime only | ✅ But verbose |

### ⚠️ Common Mistake
**Forgetting that permitted subclasses must declare `final`, `sealed`, or `non-sealed`**: Every direct subclass of a sealed type must explicitly declare its extension policy. Omitting this modifier causes a compile error. Also, **all permitted subclasses must be in the same module** (or same package for unnamed modules). If a permitted subclass is in a different module, the code won't compile — this is a deliberate design choice to keep the type hierarchy co-located and maintainable.

### ⚡ Performance Tip
Sealed classes enable JVM optimizations that aren't possible with open hierarchies. The JIT compiler can perform **complete class hierarchy analysis** on sealed types — since it knows all possible subtypes, it can devirtualize method calls and inline them more aggressively. For exhaustive switch expressions on sealed types, the JVM can use more efficient dispatch mechanisms because it knows the complete type set at compile time.

### 🔧 JVM Internals
Sealed classes are enforced at both compile time and runtime. The class file stores the permitted subclasses in the `PermittedSubclasses` attribute (JEP 360). At class-loading time, the JVM verifies that any class attempting to extend a sealed class is in the permitted list — attempting to bypass this at runtime with bytecode manipulation results in `IncompatibleClassChangeError`. The `sealed` and `non-sealed` keywords are new **context-sensitive keywords** — they're only keywords in class/interface declarations, so existing code using `sealed` as a variable name continues to compile.

---

## 🎯 Interviewer Insight
Interviewers look for your understanding of **why** sealed classes were added — not just the syntax. The key insight: sealed classes + records + pattern matching together form Java's answer to **algebraic data types** (sum types) found in Scala, Kotlin, and Rust. A strong answer models a real-world problem (Result type, state machine, AST) using this trio and explains how exhaustive switch prevents bugs when new subtypes are added.

## ⚡ Follow-up Questions
1. What's the difference between sealed classes and enums?
2. Can a permitted subclass be in a different package?
3. How do sealed classes interact with reflection and serialization?

## 📚 Related Topics
- [Records](./q14-records-detail.md)
- [Pattern Matching](./q15-pattern-matching.md)
- [Default Methods](./q08-default-methods.md)

# Q15: Pattern Matching for instanceof and switch

## 📋 Question
Explain pattern matching in Java. Cover pattern matching for instanceof (Java 16), pattern matching for switch (Java 21), guarded patterns, record patterns, nested patterns, exhaustiveness, and null handling.

## Difficulty: 🟠 Senior

---

## 💡 Answer

**Pattern matching** progressively eliminates the boilerplate of type-checking and casting in Java. It started with **instanceof** (Java 16), extended to **switch** (Java 21), and integrates with **record deconstruction** for deep structural matching. Together with sealed classes and records, pattern matching enables Java to model algebraic data types elegantly.

### Pattern Matching for instanceof (Java 16)

```java
package com.interview.java16;

public class InstanceofPatternDemo {

    // ═══════════════════════════════════════════════════
    // BEFORE Java 16: Verbose cast-after-check pattern
    // ═══════════════════════════════════════════════════
    static String describeOld(Object obj) {
        if (obj instanceof String) {
            String s = (String) obj;            // Redundant cast
            return "String of length " + s.length();
        } else if (obj instanceof Integer) {
            Integer i = (Integer) obj;          // Redundant cast
            return "Integer: " + i;
        }
        return "Unknown: " + obj;
    }

    // ═══════════════════════════════════════════════════
    // AFTER Java 16: Pattern variable eliminates the cast
    // ═══════════════════════════════════════════════════
    static String describeNew(Object obj) {
        if (obj instanceof String s) {           // Test + declare + cast in one
            return "String of length " + s.length();
        } else if (obj instanceof Integer i) {
            return "Integer: " + i;
        } else if (obj instanceof int[] arr) {
            return "int array of length " + arr.length;
        }
        return "Unknown: " + obj;
    }

    public static void main(String[] args) {
        System.out.println(describeNew("Hello"));    // String of length 5
        System.out.println(describeNew(42));          // Integer: 42
        System.out.println(describeNew(3.14));        // Unknown: 3.14
    }
}
```

### Pattern Variable Scope

```java
package com.interview.java16;

public class PatternVariableScopeDemo {

    static void scopeRules(Object obj) {

        // ✅ Pattern variable is in scope when the compiler can PROVE the type
        if (obj instanceof String s) {
            // s is in scope here — obj IS a String
            System.out.println(s.toUpperCase());
        }
        // s is NOT in scope here — obj might not be a String

        // ✅ Flow scoping with negation
        if (!(obj instanceof String s)) {
            // s is NOT in scope here
            return;
        }
        // s IS in scope here — because we returned above if not String
        System.out.println("Definitely a string: " + s.length());

        // ✅ Combined with logical AND
        if (obj instanceof String str && str.length() > 5) {
            System.out.println("Long string: " + str);
        }

        // ❌ Cannot use with OR — variable might not be initialized
        // if (obj instanceof String str || str.isEmpty()) { } // ❌ Compile error

        // ✅ In ternary
        String result = (obj instanceof String str) ? str.toUpperCase() : "not a string";
    }

    // ✅ Combining with &&
    static boolean isNonEmptyString(Object obj) {
        return obj instanceof String s && !s.isEmpty();
    }
}
```

### Pattern Matching for switch (Java 21)

```java
package com.interview.java21;

public class SwitchPatternDemo {

    sealed interface Shape permits Circle, Rectangle, Triangle {}
    record Circle(double radius) implements Shape {}
    record Rectangle(double width, double height) implements Shape {}
    record Triangle(double base, double height) implements Shape {}

    // ═══════════════════════════════════════════════════
    // Type patterns in switch
    // ═══════════════════════════════════════════════════
    static double area(Shape shape) {
        return switch (shape) {
            case Circle c    -> Math.PI * c.radius() * c.radius();
            case Rectangle r -> r.width() * r.height();
            case Triangle t  -> 0.5 * t.base() * t.height();
            // No default needed — sealed interface makes switch exhaustive
        };
    }

    // ═══════════════════════════════════════════════════
    // Guarded patterns with 'when' clause
    // ═══════════════════════════════════════════════════
    static String classify(Shape shape) {
        return switch (shape) {
            case Circle c when c.radius() > 100  -> "Large circle";
            case Circle c when c.radius() > 10   -> "Medium circle";
            case Circle c                         -> "Small circle";
            case Rectangle r when r.width() == r.height() -> "Square";
            case Rectangle r                      -> "Rectangle";
            case Triangle t when t.base() == t.height() -> "Isoceles-ish triangle";
            case Triangle t                       -> "Triangle";
        };
    }

    // ═══════════════════════════════════════════════════
    // Multiple types and primitives
    // ═══════════════════════════════════════════════════
    static String format(Object obj) {
        return switch (obj) {
            case Integer i  -> "int: " + i;
            case Long l     -> "long: " + l;
            case Double d   -> String.format("double: %.2f", d);
            case String s   -> "string: \"" + s + "\"";
            case int[] arr  -> "int array[" + arr.length + "]";
            case null       -> "null!";
            default         -> "unknown: " + obj.getClass().getSimpleName();
        };
    }

    public static void main(String[] args) {
        System.out.println(area(new Circle(5)));            // 78.54
        System.out.println(area(new Rectangle(4, 6)));      // 24.0
        System.out.println(classify(new Circle(50)));       // Medium circle
        System.out.println(classify(new Rectangle(5, 5)));  // Square
        System.out.println(format(42));                     // int: 42
        System.out.println(format("hi"));                   // string: "hi"
        System.out.println(format(null));                    // null!
    }
}
```

### Record Patterns (Deconstruction)

```java
package com.interview.java21;

public class RecordPatternsDemo {

    record Point(double x, double y) {}
    record Line(Point start, Point end) {}
    record ColoredPoint(Point point, String color) {}

    // ═══════════════════════════════════════════════════
    // Simple record deconstruction
    // ═══════════════════════════════════════════════════
    static String describePoint(Object obj) {
        return switch (obj) {
            case Point(var x, var y) when x == 0 && y == 0 -> "Origin";
            case Point(var x, var y) when x == 0 -> "On Y-axis at y=" + y;
            case Point(var x, var y) when y == 0 -> "On X-axis at x=" + x;
            case Point(var x, var y) -> String.format("Point(%.1f, %.1f)", x, y);
            default -> "Not a point";
        };
    }

    // ═══════════════════════════════════════════════════
    // Nested record patterns — deep deconstruction
    // ═══════════════════════════════════════════════════
    static double lineLength(Object obj) {
        if (obj instanceof Line(Point(var x1, var y1), Point(var x2, var y2))) {
            return Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
        }
        return -1;
    }

    // ═══════════════════════════════════════════════════
    // Nested patterns in switch
    // ═══════════════════════════════════════════════════
    static String describeColored(Object obj) {
        return switch (obj) {
            case ColoredPoint(Point(var x, var y), var color)
                    when color.equals("red") -> "Red point at (" + x + "," + y + ")";
            case ColoredPoint(Point(var x, var y), var color)
                    -> color + " point at (" + x + "," + y + ")";
            default -> "Not a colored point";
        };
    }

    public static void main(String[] args) {
        System.out.println(describePoint(new Point(0, 5)));     // On Y-axis at y=5.0
        System.out.println(describePoint(new Point(3, 4)));     // Point(3.0, 4.0)

        var line = new Line(new Point(0, 0), new Point(3, 4));
        System.out.println("Length: " + lineLength(line));       // 5.0

        var cp = new ColoredPoint(new Point(1, 2), "red");
        System.out.println(describeColored(cp));                 // Red point at (1.0,2.0)
    }
}
```

### Exhaustiveness and Null Handling

```java
package com.interview.java21;

public class ExhaustivenessDemo {

    sealed interface Result<T> permits Success, Failure {}
    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(String error) implements Result<T> {}

    // ═══════════════════════════════════════════════════
    // Exhaustive switch — compiler ensures all cases covered
    // ═══════════════════════════════════════════════════
    static <T> String handleResult(Result<T> result) {
        return switch (result) {
            case Success<T>(var value) -> "Success: " + value;
            case Failure<T>(var error) -> "Error: " + error;
            // No default needed — sealed hierarchy is exhaustive
            // Adding a new subtype to Result forces updating this switch
        };
    }

    // ═══════════════════════════════════════════════════
    // Null handling in switch (Java 21)
    // ═══════════════════════════════════════════════════
    static String processInput(Object input) {
        return switch (input) {
            case null          -> "Received null";          // ✅ Explicit null case
            case String s      -> "String: " + s;
            case Integer i     -> "Integer: " + i;
            default            -> "Other: " + input;
        };
        // Without 'case null', null throws NullPointerException (traditional behavior)
    }

    // ═══════════════════════════════════════════════════
    // Null combined with other patterns
    // ═══════════════════════════════════════════════════
    static String handleNullable(String input) {
        return switch (input) {
            case null, String s when s.isBlank() -> "Empty or null";
            case String s when s.length() > 10   -> "Long: " + s;
            case String s                         -> "Normal: " + s;
        };
    }

    // ═══════════════════════════════════════════════════
    // Dominance — order matters (more specific patterns first)
    // ═══════════════════════════════════════════════════
    static String ordered(Object obj) {
        return switch (obj) {
            // ✅ Specific cases BEFORE general cases
            case String s when s.isEmpty() -> "empty string";  // More specific
            case String s                  -> "string: " + s;  // Less specific
            case Number n                  -> "number: " + n;  // Supertype
            default                        -> "other";
        };
        // ❌ Would fail: case String s -> ... (general) BEFORE case String s when ... (specific)
    }

    public static void main(String[] args) {
        System.out.println(handleResult(new Success<>("data")));  // Success: data
        System.out.println(handleResult(new Failure<>("timeout")));  // Error: timeout
        System.out.println(processInput(null));                    // Received null
        System.out.println(processInput("hello"));                 // String: hello
    }
}
```

### Pattern Matching Evolution Timeline

| Java Version | Feature | Status |
|-------------|---------|--------|
| Java 14 | Pattern matching for `instanceof` | Preview |
| Java 16 | Pattern matching for `instanceof` | **Final** |
| Java 17 | Pattern matching for `switch` | Preview |
| Java 21 | Pattern matching for `switch` | **Final** |
| Java 21 | Record patterns | **Final** |
| Java 21 | Guarded patterns (`when`) | **Final** |

### ⚠️ Common Mistake
**Incorrect case ordering in pattern switch**: Pattern cases are checked **top-to-bottom**, and the compiler enforces **dominance rules** — a more general pattern cannot appear before a more specific one. Placing `case String s` before `case String s when s.isEmpty()` causes a compile error because the general case dominates (would always match first). Always order from most specific to most general. Also, remember that `case null` must be explicitly added — without it, null input throws `NullPointerException`, just like traditional switch.

### ⚡ Performance Tip
Pattern matching switch compiles to efficient bytecode using `tableswitch`/`lookupswitch` for primitive cases and type-check chains for object patterns. The JIT compiler optimizes common patterns — for sealed hierarchies, it can even eliminate type checks when the class is provably known. Record patterns with nested deconstruction are optimized to avoid creating intermediate objects — the JVM directly accesses the nested components.

### 🔧 JVM Internals
Pattern matching for switch uses the `invokedynamic` instruction with `SwitchBootstraps.typeSwitch()` as the bootstrap method. This runtime bootstrap generates an efficient dispatch table based on the pattern types. For sealed classes, the compiler encodes the exhaustiveness information in the class file, enabling the JVM to throw `MatchException` (not `IncompatibleClassChangeError`) if a new permitted subclass is added without recompiling the switch. Pattern variables are implemented as regular local variables in the bytecode — the "smart cast" is purely a compile-time feature.

---

## 🎯 Interviewer Insight
Interviewers love pattern matching questions because they combine multiple modern Java features. A strong answer demonstrates: pattern matching for instanceof (Java 16) with **flow scoping**, pattern matching for switch with **guarded patterns** (`when`), **record deconstruction patterns** for deep matching, and **exhaustiveness** with sealed types. The trio of sealed classes + records + pattern matching for expressing algebraic data types is a powerful talking point.

## ⚡ Follow-up Questions
1. What happens if a new subclass is added to a sealed hierarchy but the switch is not updated?
2. Can you use pattern matching with generics?
3. How does the `when` guard differ from using an `if` inside the case body?

## 📚 Related Topics
- [Records](./q14-records-detail.md)
- [Sealed Classes](./q16-sealed-classes-detail.md)
- [Lambda Expressions](./q01-lambda-expressions.md)

# Q12: How Do Enums Work in Java?

## 📋 Question
How do Enums work in Java? Explain enum basics, enums with fields and methods, enum implementing interfaces, and when to use enums vs constants.

## Difficulty: 🟡 Medium

---

## 💡 Answer

Java **enums** (enumerated types) are special classes that represent a **fixed set of constants**. Introduced in Java 5, they are far more powerful than enums in most other languages — Java enums are full-fledged classes that can have **fields, constructors, methods**, and can **implement interfaces**.

Every enum implicitly extends `java.lang.Enum<E>`, which means: (1) enums cannot extend other classes (single inheritance already used); (2) enums are implicitly `final` (cannot be subclassed, with a nuance for constant-specific bodies); (3) each enum constant is a `public static final` instance of the enum type, created once during class loading.

### Key Enum Features
- **Type safety**: Unlike `int` constants, enums are typed — you cannot pass a `Season` where a `Color` is expected.
- **Namespace**: Enum constants are scoped to the enum type — `Season.SPRING` vs `Color.RED`.
- **Singleton guarantee**: Each enum constant is a singleton — exactly one instance exists per JVM (even surviving serialization/deserialization).
- **Built-in methods**: `values()`, `valueOf(String)`, `name()`, `ordinal()`, `compareTo()`.
- **Usable in switch**: Enums work seamlessly with switch statements and expressions.

### Code Example

```java
package com.interview.fundamentals;

// === BASIC ENUM ===
enum Season {
    SPRING, SUMMER, AUTUMN, WINTER
}

// === ENUM WITH FIELDS, CONSTRUCTOR, AND METHODS ===
enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS(4.869e+24, 6.0518e6),
    EARTH(5.976e+24, 6.37814e6),
    MARS(6.421e+23, 3.3972e6);

    // Fields (should be final for immutability)
    private final double mass;    // in kilograms
    private final double radius;  // in meters

    // Constructor — ALWAYS private (implicitly or explicitly)
    // ❌ public Planet(...) — compile error
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    // Getters
    public double mass() { return mass; }
    public double radius() { return radius; }

    // Custom method
    public double surfaceGravity() {
        final double G = 6.67300E-11;
        return G * mass / (radius * radius);
    }

    public double surfaceWeight(double otherMass) {
        return otherMass * surfaceGravity();
    }
}

public class EnumDemo {

    public static void main(String[] args) {
        // === BASIC USAGE ===
        Season current = Season.SUMMER;
        System.out.println("Current season: " + current);           // SUMMER
        System.out.println("Name: " + current.name());              // SUMMER
        System.out.println("Ordinal: " + current.ordinal());        // 1 (0-based position)

        // === Built-in methods ===
        // values() — returns array of all constants
        for (Season s : Season.values()) {
            System.out.println(s.ordinal() + ": " + s.name());
        }

        // valueOf() — converts String to enum (case-sensitive, exact match)
        Season spring = Season.valueOf("SPRING");  // ✅
        // Season bad = Season.valueOf("spring");   // ❌ IllegalArgumentException

        // === SWITCH WITH ENUM ===
        String description = switch (current) {
            case SPRING -> "Flowers blooming";
            case SUMMER -> "Beach time";
            case AUTUMN -> "Leaves falling";
            case WINTER -> "Snow falling";
        };  // Exhaustive — compiler ensures all values covered
        System.out.println(description);

        // === ENUM WITH METHODS ===
        double earthWeight = 75.0;  // kg
        for (Planet p : Planet.values()) {
            System.out.printf("Your weight on %s is %6.2f N%n",
                p, p.surfaceWeight(earthWeight));
        }

        // === ENUM COMPARISON ===
        // ✅ Use == for enum comparison (always works, null-safe on left side)
        Season s1 = Season.SPRING;
        Season s2 = Season.SPRING;
        System.out.println(s1 == s2);       // true (same singleton instance)
        System.out.println(s1.equals(s2));  // true (but == is preferred)
        // == is preferred because: (1) faster, (2) null-safe, (3) compile-time type check
    }
}
```

### Enum Implementing Interfaces

```java
package com.interview.fundamentals;

// ✅ Enums can implement interfaces — enabling polymorphism
interface Printable {
    String format();
}

// Enum with constant-specific method implementations
enum HttpStatus implements Printable {
    OK(200, "OK") {
        @Override
        public String format() {
            return code() + " ✅ " + message();
        }
    },
    NOT_FOUND(404, "Not Found") {
        @Override
        public String format() {
            return code() + " ❌ " + message();
        }
    },
    INTERNAL_ERROR(500, "Internal Server Error") {
        @Override
        public String format() {
            return code() + " 🔥 " + message();
        }
    };

    private final int code;
    private final String message;

    HttpStatus(int code, String message) {
        this.code = code;
        this.message = message;
    }

    public int code() { return code; }
    public String message() { return message; }

    // Find by code — common pattern
    public static HttpStatus fromCode(int code) {
        for (HttpStatus status : values()) {
            if (status.code == code) return status;
        }
        throw new IllegalArgumentException("Unknown HTTP status code: " + code);
    }
}
```

### EnumSet and EnumMap

```java
package com.interview.fundamentals;

import java.util.EnumMap;
import java.util.EnumSet;

enum Permission { READ, WRITE, EXECUTE, DELETE }

public class EnumCollectionsDemo {

    public static void main(String[] args) {

        // === EnumSet — specialized Set for enums (backed by bit vector) ===
        // Extremely fast and memory-efficient

        EnumSet<Permission> readOnly = EnumSet.of(Permission.READ);
        EnumSet<Permission> readWrite = EnumSet.of(Permission.READ, Permission.WRITE);
        EnumSet<Permission> all = EnumSet.allOf(Permission.class);
        EnumSet<Permission> none = EnumSet.noneOf(Permission.class);
        EnumSet<Permission> complement = EnumSet.complementOf(readOnly);
        // complement = {WRITE, EXECUTE, DELETE}

        // ✅ Check permissions
        if (readWrite.contains(Permission.WRITE)) {
            System.out.println("Write access granted");
        }

        // === EnumMap — specialized Map with enum keys (backed by array) ===
        // Faster and more memory-efficient than HashMap for enum keys

        EnumMap<Permission, String> descriptions = new EnumMap<>(Permission.class);
        descriptions.put(Permission.READ, "View files");
        descriptions.put(Permission.WRITE, "Modify files");
        descriptions.put(Permission.EXECUTE, "Run programs");
        descriptions.put(Permission.DELETE, "Remove files");

        descriptions.forEach((perm, desc) ->
            System.out.println(perm + ": " + desc));
        // Iteration order follows enum declaration order (guaranteed)
    }
}
```

### Enum Singleton Pattern

```java
package com.interview.fundamentals;

// ✅ Enum singleton — the BEST way to implement singleton in Java
// (Recommended by Joshua Bloch in "Effective Java")
enum DatabaseConnection {
    INSTANCE;  // Single enum constant = singleton

    private String url;

    // Initialization
    DatabaseConnection() {
        this.url = "jdbc:postgresql://localhost:5432/mydb";
    }

    public void connect() {
        System.out.println("Connected to: " + url);
    }

    public void query(String sql) {
        System.out.println("Executing: " + sql);
    }
}

class EnumSingletonUsage {
    public static void main(String[] args) {
        // ✅ Thread-safe, serialization-safe, reflection-proof
        DatabaseConnection.INSTANCE.connect();
        DatabaseConnection.INSTANCE.query("SELECT * FROM users");

        // Why enum singleton is superior:
        // 1. Thread-safe by JVM guarantee (class loading is synchronized)
        // 2. Serialization-safe (enum deserialization returns existing instance)
        // 3. Reflection-proof (cannot create new instances via reflection)
        // 4. Concise and readable
    }
}
```

### ⚠️ Common Mistake
**Relying on `ordinal()` for logic or persistence**: The ordinal is the position of the constant in the declaration (0-based). If you add, remove, or reorder enum constants, all ordinals change — breaking any code or database that depends on them. Use the `name()` string representation or a custom field instead. Also, **never use `int` constants** (`public static final int STATUS_OK = 0`) when an enum is appropriate — `int` constants have no type safety, no namespace, no methods, and no IDE support.

```java
// ❌ Anti-pattern: using ordinal for storage
int saved = Season.SUMMER.ordinal();  // 1 — breaks if enum is reordered

// ✅ Use name() or a custom code field for persistence
String saved2 = Season.SUMMER.name();  // "SUMMER" — stable
Season restored = Season.valueOf(saved2);
```

### ⚡ Performance Tip
`EnumSet` is internally implemented as a **bit vector** — each enum constant maps to a single bit. Operations like `contains`, `add`, `remove`, and set operations (union, intersection) execute in **O(1)** time using bitwise operations. For enums with 64 or fewer constants, `EnumSet` uses a single `long` (64 bits). This makes it orders of magnitude faster than `HashSet<MyEnum>`. Similarly, `EnumMap` uses a **simple array** indexed by `ordinal()`, making it faster and more memory-efficient than `HashMap<MyEnum, V>`. Always prefer `EnumSet` and `EnumMap` when keys are enum types.

### 🔧 JVM Internals
At the bytecode level, each enum class compiles to a class that extends `java.lang.Enum`. Each constant becomes a `public static final` field of the enum type, initialized in a `static {}` block. The `values()` method returns a **clone** of an internal array (so callers cannot corrupt the original). The enum constructor is always `private` — the JVM prevents reflective creation of new enum instances by throwing `IllegalArgumentException` in `Constructor.newInstance()`. During deserialization, the JVM calls `Enum.valueOf()` instead of creating a new instance, ensuring the singleton property is preserved. The `switch` on enum compiles to a `tableswitch` on ordinals, wrapped in a synthetic `$SwitchMap` array that maps ordinals to case indices — this indirection allows the switch to work even when enum classes are compiled separately.

---

## 🎯 Interviewer Insight
Interviewers expect you to go beyond basic enum usage. Discussing enums with fields, constructors, and methods shows you understand they are full classes. Mentioning the enum singleton pattern (Effective Java) demonstrates awareness of best practices. Knowing about `EnumSet`/`EnumMap` and their performance advantages shows practical experience. The key insight is explaining why enums are **type-safe** alternatives to `int` constants and when to use enums vs sealed classes (Java 17+).

## ⚡ Follow-up Questions
1. Can an enum extend a class? Can it implement multiple interfaces?
2. How does enum serialization work, and why is it safe for singletons?
3. When would you use a sealed class instead of an enum?

## 📚 Related Topics
- [Final Keyword](./q11-final-keyword.md)
- [Sealed Classes](./q15-sealed-classes-java15.md)
- [Records in Java 14+](./q14-records-java14.md)

# Q3: Explain Java's Primitive Data Types and Their Wrapper Classes

## 📋 Question
Explain Java's primitive data types and their wrapper classes. How do autoboxing and unboxing work, and what are the pitfalls?

## Difficulty: 🟢 Easy

---

## 💡 Answer

Java has **8 primitive data types** that are not objects — they hold values directly on the stack (when local) rather than as references on the heap. Each primitive has a corresponding **wrapper class** in `java.lang` that allows primitives to be used in object-oriented contexts (collections, generics, etc.).

### The 8 Primitive Types

| Type | Size | Min Value | Max Value | Default | Wrapper Class |
|------|------|-----------|-----------|---------|---------------|
| `byte` | 8 bits | -128 | 127 | `0` | `Byte` |
| `short` | 16 bits | -32,768 | 32,767 | `0` | `Short` |
| `int` | 32 bits | -2,147,483,648 | 2,147,483,647 | `0` | `Integer` |
| `long` | 64 bits | -9.2 × 10¹⁸ | 9.2 × 10¹⁸ | `0L` | `Long` |
| `float` | 32 bits | ~1.4 × 10⁻⁴⁵ | ~3.4 × 10³⁸ | `0.0f` | `Float` |
| `double` | 64 bits | ~4.9 × 10⁻³²⁴ | ~1.8 × 10³⁰⁸ | `0.0d` | `Double` |
| `char` | 16 bits | '\u0000' (0) | '\uffff' (65,535) | `'\u0000'` | `Character` |
| `boolean` | JVM-specific | `false` | `true` | `false` | `Boolean` |

**Autoboxing** is the automatic conversion from a primitive to its wrapper class (`int` → `Integer`). **Unboxing** is the reverse (`Integer` → `int`). Introduced in Java 5, these conversions happen implicitly but carry performance costs — each autobox creates (or retrieves) an object on the heap.

**Integer Cache**: Java caches `Integer` objects for values **-128 to 127** (and `Byte`, `Short`, `Long` in the same range, plus `Character` 0-127 and `Boolean`). This means `Integer.valueOf(127) == Integer.valueOf(127)` returns `true`, but `Integer.valueOf(128) == Integer.valueOf(128)` returns `false` — a classic interview trap.

Primitives cannot be `null` and cannot be used as generic type parameters (`List<int>` is illegal; use `List<Integer>`). They also don't have methods — you must use the wrapper class for utility methods like `Integer.parseInt()`, `Double.isNaN()`, etc.

### Code Example

```java
package com.interview.fundamentals;

public class PrimitiveTypesDemo {

    public static void main(String[] args) {
        // ✅ Primitive declarations with literal notations
        byte b = 127;                    // 8-bit signed
        short s = 32_767;                // 16-bit signed (underscores for readability)
        int i = 2_147_483_647;           // 32-bit signed
        long l = 9_223_372_036_854_775_807L;  // 64-bit signed (L suffix)
        float f = 3.14f;                 // 32-bit floating point (f suffix required)
        double d = 3.141592653589793;    // 64-bit floating point
        char c = 'A';                    // 16-bit Unicode character
        boolean flag = true;             // true or false

        // ✅ Numeric literals in different bases
        int decimal = 42;               // Decimal
        int hex = 0x2A;                 // Hexadecimal (= 42)
        int octal = 052;               // Octal (= 42)
        int binary = 0b00101010;        // Binary (= 42, Java 7+)

        // ✅ Autoboxing: primitive → wrapper (implicit)
        Integer autoBoxed = 42;          // Compiler converts to: Integer.valueOf(42)
        Double autoBoxedD = 3.14;        // Compiler converts to: Double.valueOf(3.14)

        // ✅ Unboxing: wrapper → primitive (implicit)
        int unboxed = autoBoxed;         // Compiler converts to: autoBoxed.intValue()

        // ✅ Wrapper utility methods
        int parsed = Integer.parseInt("42");
        String binary_str = Integer.toBinaryString(42);  // "101010"
        int max = Integer.MAX_VALUE;
        int min = Integer.MIN_VALUE;

        System.out.println("Parsed: " + parsed);
        System.out.println("Binary: " + binary_str);
        System.out.println("Max int: " + max);
        System.out.println("Min int: " + min);
    }
}
```

### The Integer Cache Gotcha

```java
package com.interview.fundamentals;

public class IntegerCacheGotcha {

    public static void main(String[] args) {
        // === THE CACHE TRAP ===

        // ✅ Values -128 to 127 are CACHED — same object returned
        Integer a = 127;                // Integer.valueOf(127) — cached
        Integer b = 127;                // Integer.valueOf(127) — same cached object
        System.out.println(a == b);     // true (same reference from cache)
        System.out.println(a.equals(b)); // true (same value)

        // ❌ Values outside cache range — DIFFERENT objects
        Integer x = 128;                // Integer.valueOf(128) — new object
        Integer y = 128;                // Integer.valueOf(128) — different new object
        System.out.println(x == y);     // false! (different references)
        System.out.println(x.equals(y)); // true  (same value)

        // ✅ ALWAYS use .equals() to compare wrapper objects
        // ❌ NEVER use == to compare wrapper objects

        // === UNBOXING NULL TRAP ===
        Integer nullValue = null;
        // int danger = nullValue;      // ❌ NullPointerException at runtime!
        //                              //    Compiler won't warn you

        // ✅ Safe unboxing
        int safe = (nullValue != null) ? nullValue : 0;
        System.out.println("Safe value: " + safe);

        // === TYPE WIDENING ===
        byte smallByte = 42;
        int widened = smallByte;         // ✅ Implicit widening: byte → int (no data loss)
        // byte narrowed = widened;      // ❌ Compile error: possible lossy conversion
        byte narrowed = (byte) widened;  // ✅ Explicit cast needed for narrowing
    }
}
```

### ⚠️ Common Mistake
**Using `==` to compare wrapper objects**: `==` checks reference equality, not value equality, for objects. Due to the Integer cache, `==` works for small values (-128 to 127) but fails silently for larger values. This creates bugs that are hard to detect in testing (small test values work) but fail in production (real values exceed the cache range). Always use `.equals()` for wrapper comparison.

### ⚡ Performance Tip
Autoboxing in tight loops can cause **massive garbage collection pressure**. Each autobox creates a new object on the heap (outside the cache range). Prefer primitives for computation-heavy code. For collections requiring objects, consider specialized collections like Eclipse Collections' `IntList` or `IntObjectMap` that avoid boxing entirely. In Java 21+ (preview), **Project Valhalla** aims to add value types and primitive generics (`List<int>`) to solve this problem at the language level.

```java
// ❌ Bad: autoboxing in a loop creates ~10 million Integer objects
Long sum = 0L;
for (long i = 0; i < 10_000_000; i++) {
    sum += i;  // Unbox sum, add i, autobox result — every iteration
}

// ✅ Good: use primitive — zero allocations
long sum2 = 0L;
for (long i = 0; i < 10_000_000; i++) {
    sum2 += i;  // Pure stack operation, no heap allocation
}
```

### 🔧 JVM Internals
Primitives stored as local variables live on the **thread's stack** frame — they are allocated and freed automatically when the method enters and exits, with zero GC overhead. Wrapper objects live on the **heap** and are managed by the garbage collector. The JVM's **escape analysis** can sometimes optimize wrapper objects: if the JIT compiler proves an `Integer` object doesn't escape the method, it may **scalar-replace** it (store the `int` value directly on the stack, eliminating the heap allocation entirely). The Integer cache is implemented in `Integer.valueOf()` using a static inner class `IntegerCache` that pre-initializes an `Integer[]` array. The upper bound of the cache can be extended with `-XX:AutoBoxCacheMax=<size>`.

---

## 🎯 Interviewer Insight
This question tests your understanding of Java's type system and memory model. Interviewers expect you to know the Integer cache behavior, the autoboxing/unboxing mechanics, and the performance implications. Being able to explain **why** `==` works for `Integer(127)` but fails for `Integer(128)` demonstrates deep understanding. Mentioning the `NullPointerException` risk from unboxing `null` shows production experience.

## ⚡ Follow-up Questions
1. Why can't you use primitive types as generic type parameters in Java?
2. What happens when you mix primitives and wrappers in arithmetic expressions?
3. How does `float` vs `double` affect precision, and why should you never use `float`/`double` for monetary calculations?

## 📚 Related Topics
- [Operators and Expressions](./q04-operators-and-expressions.md)
- [Strings and String Pool](./q07-strings-and-string-pool.md)
- [Generics & Type System](../09-generics-type-system/README.md)

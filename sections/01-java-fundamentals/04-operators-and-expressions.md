# Q4: How Do Operators and Expressions Work in Java?

## 📋 Question
How do operators and expressions work in Java? Explain the different categories of operators, operator precedence, and short-circuit evaluation.

## Difficulty: 🟢 Easy

---

## 💡 Answer

Java provides a rich set of **operators** that work on operands to produce results. Operators are categorized by function: arithmetic, relational, logical, bitwise, assignment, ternary, and type-checking. Understanding **operator precedence** (which operator evaluates first) and **short-circuit evaluation** (skipping unnecessary evaluations) is essential for writing correct and efficient code.

### Operator Categories

**Arithmetic Operators**: `+`, `-`, `*`, `/`, `%` (modulo). Integer division truncates (`7 / 2 = 3`). The `%` operator returns the remainder. The `+` operator is overloaded for String concatenation.

**Relational Operators**: `==`, `!=`, `>`, `<`, `>=`, `<=`. For primitives, `==` compares values. For objects, `==` compares **references** (memory addresses), not content — use `.equals()` for value comparison.

**Logical Operators**: `&&` (short-circuit AND), `||` (short-circuit OR), `!` (NOT). Also `&` and `|` for non-short-circuit logical operations (both sides always evaluated).

**Bitwise Operators**: `&` (AND), `|` (OR), `^` (XOR), `~` (complement), `<<` (left shift), `>>` (signed right shift), `>>>` (unsigned right shift). Used for low-level operations, flags, and performance-critical code.

**Assignment Operators**: `=`, `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `|=`, `^=`, `<<=`, `>>=`, `>>>=`. Compound assignments include an implicit cast: `byte b = 10; b += 5;` works, but `b = b + 5;` requires explicit `(byte)` cast because `b + 5` is promoted to `int`.

**Ternary Operator**: `condition ? valueIfTrue : valueIfFalse` — a concise alternative to simple `if-else` statements.

**instanceof Operator**: Tests whether an object is an instance of a class or interface. Java 16+ introduced **pattern matching** with `instanceof`, which combines type checking and casting in one step.

### Code Example

```java
package com.interview.fundamentals;

public class OperatorsDemo {

    public static void main(String[] args) {

        // === ARITHMETIC OPERATORS ===
        int a = 17, b = 5;
        System.out.println("17 / 5  = " + (a / b));     // 3 (integer division truncates)
        System.out.println("17 % 5  = " + (a % b));     // 2 (remainder)
        System.out.println("17.0 / 5 = " + (17.0 / 5)); // 3.4 (floating-point division)

        // ❌ Pitfall: integer overflow wraps around silently
        int max = Integer.MAX_VALUE;
        System.out.println("MAX_VALUE + 1 = " + (max + 1)); // -2147483648 (overflow!)
        // ✅ Use Math.addExact() to detect overflow
        // Math.addExact(max, 1);  // Throws ArithmeticException

        // === SHORT-CIRCUIT EVALUATION ===
        String text = null;

        // ✅ Short-circuit && : second condition NOT evaluated if first is false
        if (text != null && text.length() > 5) {
            System.out.println("Long text");
        }
        // If text is null, text.length() is NEVER called — no NPE

        // ❌ Non-short-circuit & : BOTH sides always evaluated
        // if (text != null & text.length() > 5) { }  // NPE! text.length() is evaluated

        // ✅ Short-circuit || : second condition NOT evaluated if first is true
        boolean hasDefault = true;
        if (hasDefault || expensiveCheck()) {
            System.out.println("Using default");  // expensiveCheck() never called
        }

        // === TERNARY OPERATOR ===
        int score = 85;
        String grade = score >= 90 ? "A" : score >= 80 ? "B" : "C";
        System.out.println("Grade: " + grade);  // B

        // === INSTANCEOF WITH PATTERN MATCHING (Java 16+) ===
        Object obj = "Hello, Java 17!";

        // ❌ Old way: separate instanceof check and cast
        if (obj instanceof String) {
            String s = (String) obj;  // Redundant cast
            System.out.println("Old way length: " + s.length());
        }

        // ✅ New way: pattern matching — check and cast in one step
        if (obj instanceof String s) {
            System.out.println("New way length: " + s.length());  // s is already String
        }

        // ✅ Pattern matching with logical operators
        if (obj instanceof String s && s.length() > 5) {
            System.out.println("Long string: " + s);
        }

        // === BITWISE OPERATORS ===
        int flags = 0b0000;
        int READ    = 0b0001;  // 1
        int WRITE   = 0b0010;  // 2
        int EXECUTE = 0b0100;  // 4

        flags |= READ;           // Set READ flag:    0001
        flags |= WRITE;          // Set WRITE flag:   0011
        System.out.println("Has READ?  " + ((flags & READ) != 0));    // true
        System.out.println("Has EXEC?  " + ((flags & EXECUTE) != 0)); // false
        flags &= ~READ;          // Clear READ flag:  0010
        flags ^= WRITE;          // Toggle WRITE flag: 0000
    }

    private static boolean expensiveCheck() {
        System.out.println("This should NOT print");
        return false;
    }
}
```

### Operator Precedence Table (High to Low)

```
Precedence (highest to lowest):
─────────────────────────────────────────────────
 1.  ()  []  .                    Parentheses, array access, member access
 2.  ++ --  +  -  ~  !  (type)   Unary, cast
 3.  *  /  %                      Multiplicative
 4.  +  -                         Additive, string concatenation
 5.  <<  >>  >>>                  Shift
 6.  <  >  <=  >=  instanceof     Relational
 7.  ==  !=                       Equality
 8.  &                            Bitwise AND
 9.  ^                            Bitwise XOR
10.  |                            Bitwise OR
11.  &&                           Logical AND (short-circuit)
12.  ||                           Logical OR (short-circuit)
13.  ? :                          Ternary
14.  =  +=  -=  *=  /=  ...      Assignment
─────────────────────────────────────────────────
```

### Compound Assignment Implicit Cast

```java
package com.interview.fundamentals;

public class CompoundAssignmentCast {
    public static void main(String[] args) {
        byte b = 10;

        // ❌ Compile error: int cannot be narrowed to byte
        // b = b + 5;  // (b + 5) promotes to int

        // ✅ Compound assignment includes implicit cast
        b += 5;        // Equivalent to: b = (byte)(b + 5);

        // Same applies to short and char
        short s = 100;
        s *= 2;        // Equivalent to: s = (short)(s * 2);

        char c = 'A';
        c += 1;        // Equivalent to: c = (char)(c + 1); → 'B'
        System.out.println(c); // B
    }
}
```

### ⚠️ Common Mistake
**Confusing `==` with `.equals()` for objects**: `==` compares references for objects, not values. This is especially dangerous with `String` and wrapper types. Also, using `&` instead of `&&` in conditions can cause `NullPointerException` because `&` does not short-circuit — both operands are always evaluated. Another trap: `a == b == c` does not do what you expect — it evaluates `(a == b)` first (a boolean), then compares that boolean to `c`.

### ⚡ Performance Tip
Short-circuit evaluation (`&&`, `||`) is not just about correctness (null checks) — it's also a performance tool. Place the **cheapest or most-likely-to-short-circuit** condition first. For example: `if (cachedResult != null || expensiveDatabaseLookup())` avoids the database call when the cache hit occurs. Bitwise operations (`<<`, `>>`, `&`) are faster than arithmetic equivalents (`* 2`, `/ 2`, `% 2`) but modern JIT compilers typically optimize these automatically — prefer readability.

### 🔧 JVM Internals
At the bytecode level, Java arithmetic operations work on the **operand stack**. The JVM has typed instructions: `iadd` for `int` addition, `ladd` for `long`, `fadd` for `float`, `dadd` for `double`. There are **no** bytecode instructions for `byte`, `short`, or `char` arithmetic — they are all promoted to `int` before computation (using `i2b`, `i2s`, `i2c` for narrowing afterward). This is why `byte + byte` produces an `int` and requires explicit casting. The `instanceof` pattern matching compiles to a `checkcast` bytecode followed by an `astore` — the JIT compiler can often eliminate the redundant type check when the pattern is used within the same branch.

---

## 🎯 Interviewer Insight
Interviewers use this question to check your attention to detail — especially around integer division truncation, short-circuit evaluation, and the `==` vs `.equals()` distinction. Knowing about compound assignment implicit casting and the newer pattern matching `instanceof` (Java 16+) signals that you stay current with the language. A candidate who can explain **why** `byte + byte = int` at the bytecode level stands out.

## ⚡ Follow-up Questions
1. What is the difference between `>>` (signed right shift) and `>>>` (unsigned right shift)?
2. How does Java handle integer overflow, and how can you detect it?
3. Explain how the `+` operator works differently for `String` concatenation vs numeric addition.

## 📚 Related Topics
- [Data Types and Primitives](./q03-data-types-primitives.md)
- [Control Flow Statements](./q05-control-flow-statements.md)
- [Strings and String Pool](./q07-strings-and-string-pool.md)

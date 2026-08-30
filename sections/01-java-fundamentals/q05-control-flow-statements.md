# Q5: Explain Control Flow Statements in Java

## 📋 Question
Explain control flow statements in Java. Cover conditional statements, loops, and modern switch expressions introduced in Java 14+.

## Difficulty: 🟢 Easy

---

## 💡 Answer

Control flow statements determine the **order of execution** in a Java program. They fall into three categories: **selection** (choosing which code to execute), **iteration** (repeating code), and **transfer** (jumping to a different point in execution).

### Selection Statements
- **`if` / `else if` / `else`**: The fundamental conditional construct. Conditions must be boolean expressions (Java doesn't allow truthy/falsy values like JavaScript/Python).
- **`switch` (traditional)**: Matches a value against multiple `case` labels. Supports `byte`, `short`, `int`, `char`, `String` (Java 7+), and enums. Falls through to the next case without `break`.
- **`switch` expressions (Java 14+)**: Modern arrow syntax (`->`) with no fall-through, can return values, supports `yield` for multi-statement blocks, and the compiler enforces exhaustiveness.

### Iteration Statements
- **`for`**: Classic C-style loop with initialization, condition, and update.
- **`for-each` (enhanced for)**: Iterates over arrays and `Iterable` objects. Cleaner syntax, no index management, but cannot modify the collection or access the index.
- **`while`**: Condition checked before each iteration — may never execute.
- **`do-while`**: Condition checked after each iteration — always executes at least once.

### Transfer Statements
- **`break`**: Exits the current loop or switch statement. With a label, exits the labeled outer loop.
- **`continue`**: Skips the rest of the current iteration and moves to the next. With a label, continues the labeled outer loop.
- **`return`**: Exits the current method, optionally returning a value.

### Code Example

```java
package com.interview.fundamentals;

import java.util.List;

public class ControlFlowDemo {

    public static void main(String[] args) {

        // === TRADITIONAL SWITCH (fall-through behavior) ===
        int dayNum = 3;
        String dayName;
        switch (dayNum) {
            case 1:  dayName = "Monday";    break;
            case 2:  dayName = "Tuesday";   break;
            case 3:  dayName = "Wednesday"; break;
            case 4:  dayName = "Thursday";  break;
            case 5:  dayName = "Friday";    break;
            default: dayName = "Weekend";   break;
        }
        // ❌ Forgetting `break` causes fall-through — a common bug source

        // === ENHANCED SWITCH EXPRESSION (Java 14+) — NO fall-through ===
        String dayType = switch (dayNum) {
            case 1, 2, 3, 4, 5 -> "Weekday";    // Arrow syntax, no break needed
            case 6, 7          -> "Weekend";
            default            -> "Invalid";
        };  // Note: semicolon required — this is an expression (returns a value)
        System.out.println(dayName + " is a " + dayType);

        // === SWITCH EXPRESSION WITH yield (multi-statement blocks) ===
        String description = switch (dayNum) {
            case 1 -> "Start of the week";
            case 5 -> "TGIF!";
            case 6, 7 -> "Weekend vibes";
            default -> {
                // Multi-statement block requires `yield` to return a value
                String msg = "Day " + dayNum + " of the week";
                yield msg;  // yield replaces return in switch expression blocks
            }
        };
        System.out.println(description);

        // === SWITCH WITH PATTERN MATCHING (Java 21+) ===
        Object value = 42;
        String result = switch (value) {
            case Integer i when i > 0  -> "Positive integer: " + i;
            case Integer i             -> "Non-positive integer: " + i;
            case String s              -> "String: " + s;
            case null                  -> "Null value";
            default                    -> "Other: " + value;
        };
        System.out.println(result);

        // === FOR-EACH LOOP ===
        List<String> languages = List.of("Java", "Kotlin", "Scala");
        for (String lang : languages) {
            System.out.println("Language: " + lang);
        }
        // ❌ Cannot modify list during for-each — ConcurrentModificationException

        // === LABELED BREAK AND CONTINUE ===
        labeledLoopDemo();
    }

    // === LABELED LOOPS — breaking/continuing outer loops ===
    static void labeledLoopDemo() {
        System.out.println("\n=== Labeled Loop Demo ===");

        // ✅ Labeled break — exits the OUTER loop entirely
        outer:
        for (int i = 0; i < 5; i++) {
            for (int j = 0; j < 5; j++) {
                if (i == 2 && j == 3) {
                    System.out.println("Breaking outer at i=" + i + ", j=" + j);
                    break outer;  // Exits both loops
                }
                System.out.print("(" + i + "," + j + ") ");
            }
            System.out.println();
        }

        System.out.println("\n--- Continue Demo ---");

        // ✅ Labeled continue — skips to next iteration of OUTER loop
        outer:
        for (int i = 0; i < 3; i++) {
            for (int j = 0; j < 3; j++) {
                if (j == 1) {
                    continue outer;  // Skips rest of inner loop, goes to next i
                }
                System.out.print("(" + i + "," + j + ") ");
            }
            System.out.println();  // This line is skipped when continue outer fires
        }
        System.out.println();
    }
}
```

### While vs Do-While

```java
package com.interview.fundamentals;

public class LoopComparison {
    public static void main(String[] args) {

        // ✅ while — condition checked BEFORE body (may never execute)
        int count = 5;
        while (count < 5) {
            System.out.println("While: " + count);  // Never executes
            count++;
        }

        // ✅ do-while — condition checked AFTER body (always executes at least once)
        count = 5;
        do {
            System.out.println("Do-While: " + count);  // Executes once (prints 5)
            count++;
        } while (count < 5);

        // ✅ Infinite loop patterns
        // while (true) { ... }       // Most common
        // for (;;) { ... }           // Also valid, same bytecode
        // do { ... } while (true);   // Rarely used
    }
}
```

### Switch Expression Exhaustiveness

```java
package com.interview.fundamentals;

// ✅ Switch expressions with enums MUST be exhaustive
enum Season { SPRING, SUMMER, AUTUMN, WINTER }

public class ExhaustiveSwitchDemo {
    static String getActivity(Season season) {
        // Compiler enforces that ALL enum values are covered
        return switch (season) {
            case SPRING -> "Gardening";
            case SUMMER -> "Swimming";
            case AUTUMN -> "Hiking";
            case WINTER -> "Skiing";
            // No default needed — all cases covered
            // If you add a new enum value, this won't compile until you handle it
        };
    }

    // ❌ Anti-pattern: Using if-else chain instead of switch for enum values
    static String getActivityBad(Season season) {
        if (season == Season.SPRING) return "Gardening";
        else if (season == Season.SUMMER) return "Swimming";
        else if (season == Season.AUTUMN) return "Hiking";
        else return "Skiing";
        // No compile-time exhaustiveness check — new enum values silently fall to else
    }
}
```

### ⚠️ Common Mistake
**Forgetting `break` in traditional `switch`** causes fall-through — execution continues into the next `case` block. This is the #1 reason to prefer the enhanced switch expression (`->` syntax) which has **no fall-through by design**. Another common mistake: modifying a collection inside a `for-each` loop throws `ConcurrentModificationException`. Use `Iterator.remove()`, `List.removeIf()`, or collect changes in a separate list.

### ⚡ Performance Tip
The JVM optimizes `switch` statements using **two bytecode strategies**: `tableswitch` (constant-time O(1) jump table for dense, sequential cases like 1, 2, 3, 4, 5) and `lookupswitch` (binary search O(log n) for sparse cases like 1, 100, 500). Dense `switch` statements are faster than equivalent `if-else` chains because of the jump table. String-based `switch` uses `hashCode()` first for O(1) bucket selection, then `.equals()` for verification.

### 🔧 JVM Internals
At the bytecode level, the enhanced switch expression (`->`) compiles to the **same** `tableswitch`/`lookupswitch` bytecodes as the traditional switch — the improvement is purely syntactic and safety-oriented (no fall-through, exhaustiveness checking). Pattern matching in `switch` (Java 21+) generates `instanceof` checks and casts in the bytecode, combined with the switch structure. The `for-each` loop over a collection compiles to an `Iterator`-based `while` loop: `Iterator<E> it = collection.iterator(); while (it.hasNext()) { E e = it.next(); ... }`. For arrays, it compiles to a classic indexed `for` loop.

---

## 🎯 Interviewer Insight
Interviewers want to see that you understand modern switch expressions (Java 14+) and can explain why they are superior to traditional switch (no fall-through, return values, exhaustiveness). Demonstrating knowledge of labeled loops shows you understand complex iteration control. Being able to discuss `tableswitch` vs `lookupswitch` JVM optimizations shows genuine depth.

## ⚡ Follow-up Questions
1. What is the difference between `tableswitch` and `lookupswitch` at the bytecode level?
2. Can you use `switch` with `long`, `float`, or `double` types? Why or why not?
3. How does pattern matching in `switch` (Java 21) differ from traditional `instanceof` chains?

## 📚 Related Topics
- [Operators and Expressions](./q04-operators-and-expressions.md)
- [Enums in Java](./q12-enums-in-java.md)
- [Sealed Classes](./q15-sealed-classes-java15.md)

# Q13: Local Variable Type Inference (var)

## 📋 Question
What is the `var` keyword in Java? Explain LVTI rules, where `var` can and cannot be used, interactions with the diamond operator and lambdas, and best practices for readability.

## Difficulty: 🟡 Medium

---

## 💡 Answer

The `var` keyword (Java 10) enables **Local Variable Type Inference (LVTI)** — the compiler infers the type of a local variable from its initializer expression. It is **not** a dynamic type — Java remains statically typed. The inferred type is determined at **compile time** and cannot change at runtime. `var` is a **reserved type name**, not a keyword — you can still name variables or methods `var` (though you shouldn't).

### Basic Usage

```java
package com.interview.java10;

import java.util.*;
import java.util.stream.*;

public class VarBasicsDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // ✅ Simple type inference from initializer
        // ═══════════════════════════════════════════════════
        var name = "Alice";              // Inferred as String
        var age = 30;                    // Inferred as int
        var salary = 95_000.50;          // Inferred as double
        var active = true;               // Inferred as boolean

        // ═══════════════════════════════════════════════════
        // ✅ Complex types — var shines here
        // ═══════════════════════════════════════════════════
        var employees = new ArrayList<String>();  // ArrayList<String>
        var nameToScore = new HashMap<String, List<Integer>>();  // HashMap<String, List<Integer>>

        // Without var:
        // HashMap<String, List<Integer>> nameToScore = new HashMap<String, List<Integer>>();

        // ✅ Iterating with var
        var numbers = List.of(1, 2, 3, 4, 5);
        for (var num : numbers) {        // int (inferred)
            System.out.print(num + " ");
        }
        System.out.println();

        // ✅ Traditional for loop
        for (var i = 0; i < 10; i++) {   // int
            // ...
        }

        // ✅ Try-with-resources
        try (var reader = new java.io.BufferedReader(
                new java.io.StringReader("Hello"))) {
            var line = reader.readLine();
            System.out.println(line);
        } catch (Exception e) {
            e.printStackTrace();
        }

        // ✅ Streams — reduces type noise
        var result = numbers.stream()
                .filter(n -> n > 2)
                .map(n -> n * 10)
                .collect(Collectors.toList());  // Inferred as List<Integer>
    }
}
```

### Where var CANNOT Be Used

```java
package com.interview.java10;

public class VarRestrictionsDemo {

    // ❌ Cannot use var for fields (instance or static)
    // var instanceField = "hello";  // ❌ Compile error

    // ❌ Cannot use var for method parameters
    // void process(var input) { }  // ❌ Compile error

    // ❌ Cannot use var for method return types
    // var getName() { return "Alice"; }  // ❌ Compile error

    // ❌ Cannot use var without initializer
    // var x;  // ❌ Cannot infer type
    // x = 42;

    // ❌ Cannot use var with null initializer
    // var nothing = null;  // ❌ Type is null — not useful

    // ❌ Cannot use var for array initializer without new
    // var arr = {1, 2, 3};  // ❌ Compile error
    var arr = new int[]{1, 2, 3};  // ✅ This works

    // ❌ Cannot use var for lambda expressions (no target type)
    // var runnable = () -> System.out.println("hi");  // ❌ Which functional interface?

    // ❌ Cannot use var for method references
    // var func = String::length;  // ❌ Which functional interface?

    // ❌ Cannot use var for catch parameters
    // try { } catch (var e) { }  // ❌ Compile error

    void validExamples() {
        // ✅ These all work:
        var list = new java.util.ArrayList<String>();   // Local variable
        var stream = list.stream();                      // Local variable
        var opt = java.util.Optional.of("hello");        // Local variable
    }
}
```

### var with Diamond Operator

```java
package com.interview.java10;

import java.util.*;

public class VarWithDiamondDemo {

    public static void main(String[] args) {

        // ⚠️ var + diamond = loss of generic type info
        var list1 = new ArrayList<>();       // ❌ Inferred as ArrayList<Object> — too broad!
        var list2 = new ArrayList<String>();  // ✅ Inferred as ArrayList<String> — explicit type

        // ✅ If the type can be inferred from context:
        var list3 = List.of("a", "b", "c");  // ✅ List<String> — inferred from elements
        var list4 = List.of(1, 2, 3);        // ✅ List<Integer> — inferred from elements

        // ⚠️ Mixed types lead to unexpected inference
        var mixed = List.of("hello", 42);    // List<Serializable & Comparable<?>> — surprising!

        // ✅ Best practice: be explicit when combining var with diamond
        var map1 = new HashMap<String, Integer>();   // ✅ Clear
        var map2 = new HashMap<>();                  // ❌ HashMap<Object, Object>
    }
}
```

### var with Lambdas (Java 11)

```java
package com.interview.java11;

import java.util.function.*;
import java.lang.annotation.*;

public class VarInLambdasDemo {

    @Target(ElementType.PARAMETER)
    @Retention(RetentionPolicy.RUNTIME)
    @interface NonNull {}

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // Java 11: var allowed in lambda parameters
        // Primary use case: adding annotations to lambda params
        // ═══════════════════════════════════════════════════

        // ✅ Before Java 11 — no way to annotate lambda parameters
        BiFunction<String, String, String> concat1 = (a, b) -> a + b;

        // ✅ Java 11 — var enables annotations on lambda params
        BiFunction<String, String, String> concat2 =
                (@NonNull var a, @NonNull var b) -> a + b;

        // ⚠️ Rules: must use var for ALL params or NONE
        // (var a, b) -> a + b;         // ❌ Cannot mix var and implicit
        // (var a, String b) -> a + b;  // ❌ Cannot mix var and explicit
        // (var a, var b) -> a + b;     // ✅ All var
        // (a, b) -> a + b;            // ✅ All implicit (no var)

        System.out.println(concat2.apply("Hello, ", "World!"));
    }
}
```

### Readability Best Practices

```java
package com.interview.java10;

import java.util.*;
import java.util.stream.*;

public class VarBestPracticesDemo {

    public static void main(String[] args) {

        // ✅ GOOD: Type is obvious from the right-hand side
        var name = "Alice";                              // Obviously String
        var count = 42;                                  // Obviously int
        var employees = new ArrayList<String>();          // Obviously ArrayList<String>
        var reader = new java.io.BufferedReader(          // Obviously BufferedReader
                new java.io.InputStreamReader(System.in));

        // ✅ GOOD: Very long type names — var reduces clutter
        var entrySet = Map.of("a", List.of(1, 2)).entrySet();
        // Without var: Set<Map.Entry<String, List<Integer>>> entrySet = ...

        // ❌ BAD: Type is NOT obvious from the right side
        // var result = computeResult();   // What type is result?
        // var data = process(input);      // What type is data?
        // var x = getConfig();            // What type is x?

        // ✅ BETTER: Use descriptive variable names with var
        var userProfile = fetchUserProfile("alice");    // Name hints at type
        var connectionPool = createConnectionPool();     // Name describes the type

        // ❌ BAD: Single-letter variable names with var
        // var x = getData();  // Unreadable
        // var m = getMapping();  // Unreadable

        // ✅ GOOD: Numeric literals — obvious types
        var pi = 3.14159;   // double
        var count2 = 100L;  // long (L suffix makes type clear)

        // ⚠️ CAUTION: Primitives vs wrappers
        var number = 42;              // int (primitive)
        var boxed = Integer.valueOf(42);  // Integer (wrapper) — different!
    }

    static String fetchUserProfile(String id) { return "Profile:" + id; }
    static Object createConnectionPool() { return new Object(); }
}
```

### Summary: When to Use var

| Scenario | Use var? | Reason |
|----------|----------|--------|
| Type obvious from constructor | ✅ Yes | `var list = new ArrayList<String>()` |
| Type obvious from literal | ✅ Yes | `var name = "Alice"` |
| Complex generic types | ✅ Yes | Reduces verbosity |
| Iterator in enhanced for | ✅ Yes | `for (var item : collection)` |
| Try-with-resources | ✅ Yes | `try (var reader = ...)` |
| Type NOT obvious from RHS | ❌ No | `var x = compute()` — unclear |
| Numeric edge cases | ❌ No | `var x = 0` — int? long? |
| Fields, parameters, returns | ❌ N/A | Not allowed |

### ⚠️ Common Mistake
**Using `var` with the diamond operator on constructors**: `var list = new ArrayList<>()` infers `ArrayList<Object>` — the generic type information is lost because neither side specifies it. Always provide the type parameter when using `var` with constructors: `var list = new ArrayList<String>()`. Similarly, avoid `var` when the right-hand side is a method call whose return type isn't immediately obvious — it forces readers to look up the method signature.

### ⚡ Performance Tip
`var` has **zero runtime impact** — it's purely a compile-time feature. The compiler resolves the type and emits the same bytecode as if you'd written the type explicitly. The compiled class file contains the full type information. There's no boxing, no reflection, no dynamic dispatch — `var` is syntactic sugar that disappears at compilation. Use it freely without performance concerns.

### 🔧 JVM Internals
`var` is not a keyword — it's a **reserved type name**. This means you can still have variables named `var` (legacy code compatibility), but you cannot create a class, interface, or enum named `var`. The compiler performs **flow-sensitive type inference** using the initializer expression. The inferred type is the **most specific concrete type** — `var x = new ArrayList<String>()` infers `ArrayList<String>`, not `List<String>`. This is a subtle but important difference from explicit typing, where you'd typically program to the interface.

---

## 🎯 Interviewer Insight
Interviewers want to see that you understand `var` is **compile-time syntactic sugar**, not a dynamic type. Key points to mention: the restrictions (can't use for fields, parameters, null, lambdas without target type), the diamond operator gotcha, and the readability trade-off. A strong answer discusses **when NOT to use var** — code readability should never be sacrificed for brevity. Mentioning that `var` infers the concrete type (not the interface) shows deep understanding.

## ⚡ Follow-up Questions
1. What type does `var list = new ArrayList<>()` infer?
2. Why can't `var` be used for method return types?
3. Is `var` a keyword or a reserved type name? What's the difference?

## 📚 Related Topics
- [Lambda Expressions](./q01-lambda-expressions.md)
- [Records](./q14-records-detail.md)
- [Text Blocks](./q17-text-blocks.md)

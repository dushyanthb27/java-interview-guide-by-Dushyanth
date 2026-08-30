# Q1: How Do Lambda Expressions Work?

## 📋 Question
How do lambda expressions work in Java? Explain the syntax, target typing, variable capture rules, and how they differ from anonymous classes.

## Difficulty: 🟡 Medium

---

## 💡 Answer

Lambda expressions, introduced in **Java 8**, provide a concise way to represent **single-method interface implementations** (functional interfaces). They enable a functional programming style in Java by treating behavior as a value that can be passed around.

### Lambda Syntax Forms

```java
package com.interview.java8;

import java.util.Comparator;
import java.util.function.*;

public class LambdaSyntaxDemo {

    public static void main(String[] args) {

        // ✅ 1. Expression lambda — single expression, implicit return
        Function<String, Integer> strLength = s -> s.length();

        // ✅ 2. Block lambda — multiple statements, explicit return
        Function<String, Integer> wordCount = s -> {
            if (s == null || s.isBlank()) return 0;
            return s.split("\\s+").length;
        };

        // ✅ 3. No-parameter lambda
        Runnable greet = () -> System.out.println("Hello, Lambda!");

        // ✅ 4. Multi-parameter lambda (parentheses required)
        Comparator<String> byLength = (a, b) -> Integer.compare(a.length(), b.length());

        // ✅ 5. Explicit parameter types (usually inferred)
        BiFunction<String, Integer, String> repeat =
                (String s, Integer n) -> s.repeat(n);

        System.out.println(strLength.apply("Lambda"));          // 6
        System.out.println(wordCount.apply("Hello World"));     // 2
        greet.run();                                             // Hello, Lambda!
        System.out.println(byLength.compare("ab", "xyz"));      // -1
        System.out.println(repeat.apply("Go", 3));               // GoGoGo
    }
}
```

### Target Typing

The **type of a lambda** is determined by the context in which it appears — this is called **target typing**. The compiler infers which functional interface the lambda implements based on where it's assigned, passed, or returned.

```java
package com.interview.java8;

import java.util.concurrent.Callable;
import java.util.function.*;

public class TargetTypingDemo {

    // Same lambda body, different target types
    public static void main(String[] args) {
        // ✅ Target type is Runnable (void, no params)
        Runnable r = () -> System.out.println("run");

        // ✅ Target type is Callable<String> (returns String, no params)
        Callable<String> c = () -> "result";

        // ✅ Target type inferred from method parameter
        processAction(() -> System.out.println("action"));
    }

    static void processAction(Runnable action) {
        action.run();
    }

    // ❌ Ambiguous target type — won't compile without a cast
    // static void ambiguous(Runnable r) { }
    // static void ambiguous(Callable<Void> c) { }
    // ambiguous(() -> System.out.println("?")); // ❌ Compile error
}
```

### Effectively Final and Variable Capture

Lambdas can capture variables from the enclosing scope, but those variables must be **effectively final** — assigned exactly once and never modified after initialization.

```java
package com.interview.java8;

import java.util.function.Supplier;

public class VariableCaptureDemo {

    private String instanceField = "instance";

    public void demonstrate() {
        String localVar = "captured";     // effectively final — never reassigned
        int counter = 10;                 // effectively final

        // ✅ Capturing effectively final local variable
        Supplier<String> supplier = () -> localVar + " value: " + counter;

        // ✅ Capturing instance field (via 'this' reference — always allowed)
        Runnable useField = () -> System.out.println(this.instanceField);

        // ❌ Cannot capture a variable that is later modified
        // int mutableCounter = 0;
        // Runnable bad = () -> System.out.println(mutableCounter); // ❌ Compile error
        // mutableCounter++;  // This modification makes it non-effectively-final

        // ✅ Workaround: use a mutable container
        int[] mutableWrapper = {0};
        Runnable workaround = () -> mutableWrapper[0]++;  // Array ref is final
        workaround.run();
        System.out.println(mutableWrapper[0]);  // 1
    }

    public static void main(String[] args) {
        new VariableCaptureDemo().demonstrate();
    }
}
```

### Lambda vs Anonymous Class

| Feature | Lambda | Anonymous Class |
|---------|--------|-----------------|
| **`this` reference** | Refers to enclosing class | Refers to anonymous class itself |
| **Variable shadowing** | Cannot shadow enclosing variables | Can shadow enclosing variables |
| **Fields/state** | No instance fields | Can have instance fields |
| **Multiple methods** | Single abstract method only | Can implement multiple methods |
| **Implementation** | `invokedynamic` bytecode | Generates a separate `.class` file |
| **Performance** | Lazy class generation, no extra `.class` file | Eager class generation at compile time |
| **Serialization** | Optional (cast to Serializable) | Standard serialization |

```java
package com.interview.java8;

import java.util.function.Function;

public class LambdaVsAnonymousDemo {

    private String name = "Outer";

    public void showDifference() {

        // ✅ Lambda: 'this' refers to the ENCLOSING class (LambdaVsAnonymousDemo)
        Runnable lambda = () -> System.out.println("Lambda this: " + this.name);

        // ✅ Anonymous class: 'this' refers to the ANONYMOUS CLASS instance
        Runnable anon = new Runnable() {
            private String name = "Anonymous";

            @Override
            public void run() {
                System.out.println("Anon this: " + this.name);  // "Anonymous"
            }
        };

        lambda.run();  // Lambda this: Outer
        anon.run();    // Anon this: Anonymous

        // ❌ Lambda CANNOT shadow enclosing scope variables
        String value = "outer";
        // Function<String, String> shadower = (value) -> value; // ❌ Compile error
    }

    public static void main(String[] args) {
        new LambdaVsAnonymousDemo().showDifference();
    }
}
```

### Desugaring and invokedynamic

Under the hood, the Java compiler translates lambdas into **private static or instance methods** in the enclosing class (desugaring). At runtime, the `invokedynamic` bytecode instruction calls `LambdaMetafactory.metafactory()` to generate the functional interface implementation lazily on first use. This avoids creating a separate `.class` file for each lambda.

```java
// What the compiler generates (conceptually):
// 1. Lambda body becomes a private method:
//    private static Integer lambda$main$0(String s) { return s.length(); }
//
// 2. An invokedynamic instruction at the call site calls LambdaMetafactory
//    which generates a class implementing Function<String, Integer>
//    that delegates to lambda$main$0
//
// 3. The generated class is cached — subsequent calls reuse the same instance
//    (for non-capturing lambdas, a singleton instance is used)
```

### Serializable Lambdas

```java
package com.interview.java8;

import java.io.Serializable;
import java.util.function.Function;

public class SerializableLambdaDemo {

    // ✅ Cast to both functional interface AND Serializable
    Function<String, Integer> serializableLambda =
            (Function<String, Integer> & Serializable) String::length;

    // ⚠️ Use sparingly — serializable lambdas capture the entire
    // enclosing scope and are fragile across code changes
}
```

### ⚠️ Common Mistake
**Modifying captured variables via mutable wrappers**: While using `int[]` or `AtomicInteger` as a workaround for effectively-final constraints is technically valid, it defeats the purpose of the restriction. The effectively-final rule exists to prevent concurrency bugs — if you're mutating shared state inside a lambda (especially in parallel streams), you've introduced a race condition. Prefer `reduce()`, `collect()`, or `AtomicInteger` for thread-safe accumulation.

### ⚡ Performance Tip
Non-capturing lambdas (those that don't reference any variables from the enclosing scope) are **cached as singletons** by the JVM. A lambda like `s -> s.length()` used in a loop creates **zero garbage** — the same instance is reused on every iteration. Capturing lambdas allocate a new object each time to hold the captured values, which adds GC pressure in hot loops.

### 🔧 JVM Internals
The `invokedynamic` approach was chosen over anonymous inner classes for three key reasons: (1) **deferred class generation** — the lambda implementation class is created at runtime, not compile time, enabling future JVM optimizations without recompilation; (2) **no extra .class files** — a project with 1,000 lambdas doesn't produce 1,000 additional class files; (3) **optimization potential** — the JVM can inline lambda bodies directly into call sites, eliminating virtual dispatch overhead entirely.

---

## 🎯 Interviewer Insight
Interviewers want to see that you understand lambdas beyond basic syntax. Key differentiators: explaining `this` semantics vs anonymous classes, knowing the effectively-final rule and **why** it exists (thread safety), and understanding that lambdas use `invokedynamic` rather than generating inner classes. Mentioning the singleton caching of non-capturing lambdas demonstrates deep JVM awareness.

## ⚡ Follow-up Questions
1. Why does Java require captured variables to be effectively final?
2. Can a lambda throw checked exceptions? How do you handle that?
3. How does `invokedynamic` differ from `invokevirtual` for lambda dispatch?

## 📚 Related Topics
- [Method References](./q02-method-references.md)
- [Functional Interfaces](./q03-functional-interfaces-detail.md)
- [Stream API Operations](./q04-stream-api-operations.md)

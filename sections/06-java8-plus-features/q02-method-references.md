# Q2: What Are Method References?

## 📋 Question
What are method references in Java? Explain the four types, how type inference works, and when to prefer method references over lambdas.

## Difficulty: 🟡 Medium

---

## 💡 Answer

Method references are a **shorthand notation** for lambdas that call a single existing method. Introduced in Java 8, they make code more readable by replacing `x -> SomeClass.method(x)` with `SomeClass::method`. The compiler resolves the target functional interface through **type inference**, just like it does for lambdas.

### The Four Types of Method References

```java
package com.interview.java8;

import java.util.List;
import java.util.function.*;

public class MethodReferenceTypesDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // TYPE 1: Static method reference — ClassName::staticMethod
        // ═══════════════════════════════════════════════════
        // Lambda equivalent: (s) -> Integer.parseInt(s)
        Function<String, Integer> parser = Integer::parseInt;
        System.out.println(parser.apply("42"));  // 42

        // Lambda equivalent: (a, b) -> Math.max(a, b)
        BinaryOperator<Integer> max = Math::max;
        System.out.println(max.apply(10, 20));   // 20


        // ═══════════════════════════════════════════════════
        // TYPE 2: Instance method on a specific object — obj::instanceMethod
        // ═══════════════════════════════════════════════════
        String greeting = "Hello, World!";

        // Lambda equivalent: () -> greeting.toUpperCase()
        Supplier<String> upper = greeting::toUpperCase;
        System.out.println(upper.get());  // HELLO, WORLD!

        // Lambda equivalent: (index) -> greeting.charAt(index)
        Function<Integer, Character> charAt = greeting::charAt;
        System.out.println(charAt.apply(0));  // H


        // ═══════════════════════════════════════════════════
        // TYPE 3: Instance method on an arbitrary object of a type — ClassName::instanceMethod
        // ═══════════════════════════════════════════════════
        // Lambda equivalent: (s) -> s.toUpperCase()
        // The FIRST parameter becomes the receiver of the method call
        Function<String, String> toUpper = String::toUpperCase;
        System.out.println(toUpper.apply("java"));  // JAVA

        // Lambda equivalent: (s1, s2) -> s1.compareTo(s2)
        // s1 is the receiver, s2 is the argument
        Comparator<String> comparator = String::compareTo;
        System.out.println(comparator.compare("a", "b"));  // -1


        // ═══════════════════════════════════════════════════
        // TYPE 4: Constructor reference — ClassName::new
        // ═══════════════════════════════════════════════════
        // Lambda equivalent: () -> new StringBuilder()
        Supplier<StringBuilder> sbFactory = StringBuilder::new;

        // Lambda equivalent: (capacity) -> new StringBuilder(capacity)
        Function<Integer, StringBuilder> sbWithCap = StringBuilder::new;

        // Lambda equivalent: (s) -> new String(s)
        Function<String, String> strFactory = String::new;

        System.out.println(sbFactory.get().append("Built!"));   // Built!
        System.out.println(sbWithCap.apply(100).capacity());    // 100
    }
}
```

### Method Reference Summary Table

| Type | Syntax | Lambda Equivalent | Example |
|------|--------|-------------------|---------|
| **Static** | `ClassName::staticMethod` | `(args) -> ClassName.staticMethod(args)` | `Integer::parseInt` |
| **Bound instance** | `obj::instanceMethod` | `(args) -> obj.instanceMethod(args)` | `str::toUpperCase` |
| **Unbound instance** | `ClassName::instanceMethod` | `(obj, args) -> obj.instanceMethod(args)` | `String::toUpperCase` |
| **Constructor** | `ClassName::new` | `(args) -> new ClassName(args)` | `ArrayList::new` |

### Type Inference in Action

The compiler resolves which overloaded method to call based on the **target functional interface**:

```java
package com.interview.java8;

import java.util.function.*;

public class MethodRefTypeInferenceDemo {

    public static void main(String[] args) {

        // ✅ String::valueOf is overloaded — compiler picks the right one
        // Resolves to String.valueOf(int) because target is Function<Integer, String>
        Function<Integer, String> intToStr = String::valueOf;

        // Resolves to String.valueOf(boolean) because target is Function<Boolean, String>
        Function<Boolean, String> boolToStr = String::valueOf;

        // Resolves to String.valueOf(char[]) because target is Function<char[], String>
        Function<char[], String> charsToStr = String::valueOf;

        System.out.println(intToStr.apply(42));           // "42"
        System.out.println(boolToStr.apply(true));        // "true"
        System.out.println(charsToStr.apply(new char[]{'a','b'})); // "ab"
    }
}
```

### Practical Usage with Collections

```java
package com.interview.java8;

import java.util.*;
import java.util.stream.*;

public class MethodRefPracticalDemo {

    record Employee(String name, String department, double salary) {}

    public static void main(String[] args) {
        List<String> names = List.of("Charlie", "Alice", "Bob", "David");

        // ✅ Method references in stream operations
        names.stream()
             .map(String::toUpperCase)            // Unbound instance reference
             .sorted(String::compareTo)           // Unbound instance reference
             .forEach(System.out::println);        // Bound instance reference

        // ✅ Constructor reference to create objects
        List<String> rawData = List.of("Alice,Engineering,95000",
                                       "Bob,Marketing,85000");

        List<Employee> employees = rawData.stream()
                .map(line -> line.split(","))
                .map(parts -> new Employee(parts[0], parts[1],
                        Double.parseDouble(parts[2])))
                .collect(Collectors.toList());

        // ✅ Method references for predicate composition
        List<String> filtered = names.stream()
                .filter(Objects::nonNull)           // Static reference as Predicate
                .filter(s -> !s.isEmpty())           // Lambda (no direct method ref)
                .collect(Collectors.toList());

        // ✅ Array constructor reference
        String[] nameArray = names.stream()
                .toArray(String[]::new);             // int -> String[] constructor
    }
}
```

### When to Prefer Method References vs Lambdas

```java
package com.interview.java8;

import java.util.function.*;

public class WhenToUseMethodRefDemo {

    // ✅ PREFER method reference — cleaner and self-documenting
    Function<String, Integer> good1 = Integer::parseInt;

    // ❌ AVOID equivalent lambda when method ref exists — adds noise
    Function<String, Integer> bad1 = s -> Integer.parseInt(s);

    // ✅ PREFER lambda — method reference would be less readable
    Function<String, String> good2 = s -> "prefix_" + s;

    // ✅ PREFER lambda — multi-step logic can't be a method reference
    Function<String, String> good3 = s -> s.trim().toUpperCase();

    // ✅ PREFER lambda — when you need argument reordering
    BiFunction<String, String, String> good4 = (a, b) -> b + a;  // Reversed

    // ✅ PREFER method reference — for common patterns
    Predicate<String> good5 = String::isEmpty;      // Clearer intent
    Predicate<String> bad5  = s -> s.isEmpty();      // Equivalent but verbose

    // ⚠️ WATCH OUT — bound instance method reference captures 'this'
    // This prevents garbage collection of the enclosing object
}
```

### Rules of Thumb

| Scenario | Recommendation |
|----------|---------------|
| Direct delegation to one method | ✅ Use method reference |
| Argument transformation needed | ✅ Use lambda |
| Multiple method calls or chaining | ✅ Use lambda |
| Argument reordering | ✅ Use lambda |
| Predicate/Consumer on existing method | ✅ Use method reference |
| Readability is unclear with method ref | ✅ Use lambda |

### ⚠️ Common Mistake
**Confusing unbound instance references with static references**: `String::toUpperCase` looks like a static call, but it's actually an unbound instance method reference. It's equivalent to `(String s) -> s.toUpperCase()`. The key tell: `toUpperCase()` is an instance method of `String`, not a static method. If the method is instance-level, the first parameter of the functional interface becomes the receiver object.

### ⚡ Performance Tip
Method references and equivalent lambdas compile to the **same `invokedynamic` bytecode** and have identical runtime performance. The choice between them is purely a readability decision. However, bound instance method references (`obj::method`) capture the object reference and may prevent its garbage collection — be cautious with long-lived functional objects referencing short-lived instances.

### 🔧 JVM Internals
Method references use the same `LambdaMetafactory` infrastructure as lambdas. The `invokedynamic` call site is linked to a bootstrap method that generates a lightweight class implementing the target functional interface. For non-capturing method references (static and unbound instance types), the JVM creates a **singleton instance** — no allocation on each use. For capturing references (bound instance like `obj::method`), a new instance is created each time to hold the captured object reference.

---

## 🎯 Interviewer Insight
Interviewers test whether you can correctly identify the four types of method references and explain the subtle difference between bound and unbound instance references. A strong answer demonstrates knowing that `String::length` maps to `(String s) -> s.length()` (the first argument becomes the receiver). Bonus points for explaining when lambdas are more readable than method references.

## ⚡ Follow-up Questions
1. Can you use method references with overloaded methods? How does the compiler resolve ambiguity?
2. What's the difference between `this::method` and `ClassName::method`?
3. How does `String[]::new` work as an array constructor reference?

## 📚 Related Topics
- [Lambda Expressions](./q01-lambda-expressions.md)
- [Functional Interfaces](./q03-functional-interfaces-detail.md)
- [Stream API Operations](./q04-stream-api-operations.md)

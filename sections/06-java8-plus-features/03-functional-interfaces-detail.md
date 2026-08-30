# Q3: Deep Dive into Functional Interfaces

## 📋 Question
What are functional interfaces in Java? Explain the built-in functional interfaces, their primitive specializations, composition methods, and how to create custom functional interfaces.

## Difficulty: 🟡 Medium

---

## 💡 Answer

A **functional interface** is an interface with exactly **one abstract method** (SAM — Single Abstract Method). It can have any number of default, static, or private methods. The `@FunctionalInterface` annotation is optional but recommended — it makes the compiler enforce the single-abstract-method contract.

### Core Built-in Functional Interfaces

```java
package com.interview.java8;

import java.util.function.*;

public class CoreFunctionalInterfacesDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // Function<T, R> — takes T, returns R
        // ═══════════════════════════════════════════════════
        Function<String, Integer> length = String::length;
        System.out.println(length.apply("Hello"));  // 5

        // ═══════════════════════════════════════════════════
        // BiFunction<T, U, R> — takes T and U, returns R
        // ═══════════════════════════════════════════════════
        BiFunction<String, String, String> concat = String::concat;
        System.out.println(concat.apply("Hello, ", "World!"));  // Hello, World!

        // ═══════════════════════════════════════════════════
        // Predicate<T> — takes T, returns boolean
        // ═══════════════════════════════════════════════════
        Predicate<String> isLong = s -> s.length() > 10;
        System.out.println(isLong.test("Short"));         // false
        System.out.println(isLong.test("A longer string")); // true

        // ═══════════════════════════════════════════════════
        // Consumer<T> — takes T, returns void (side effect)
        // ═══════════════════════════════════════════════════
        Consumer<String> printer = System.out::println;
        printer.accept("Consumed!");  // Consumed!

        // ═══════════════════════════════════════════════════
        // Supplier<T> — takes nothing, returns T
        // ═══════════════════════════════════════════════════
        Supplier<Double> random = Math::random;
        System.out.println("Random: " + random.get());

        // ═══════════════════════════════════════════════════
        // UnaryOperator<T> — takes T, returns T (extends Function<T, T>)
        // ═══════════════════════════════════════════════════
        UnaryOperator<String> shout = s -> s.toUpperCase() + "!";
        System.out.println(shout.apply("hello"));  // HELLO!

        // ═══════════════════════════════════════════════════
        // BinaryOperator<T> — takes T, T, returns T (extends BiFunction<T, T, T>)
        // ═══════════════════════════════════════════════════
        BinaryOperator<Integer> sum = Integer::sum;
        System.out.println(sum.apply(3, 7));  // 10
    }
}
```

### Summary Table

| Interface | Method | Parameters | Return |
|-----------|--------|------------|--------|
| `Function<T,R>` | `apply(T)` | 1 | `R` |
| `BiFunction<T,U,R>` | `apply(T, U)` | 2 | `R` |
| `Predicate<T>` | `test(T)` | 1 | `boolean` |
| `BiPredicate<T,U>` | `test(T, U)` | 2 | `boolean` |
| `Consumer<T>` | `accept(T)` | 1 | `void` |
| `BiConsumer<T,U>` | `accept(T, U)` | 2 | `void` |
| `Supplier<T>` | `get()` | 0 | `T` |
| `UnaryOperator<T>` | `apply(T)` | 1 | `T` |
| `BinaryOperator<T>` | `apply(T, T)` | 2 | `T` |

### Primitive Specializations (Avoiding Autoboxing)

```java
package com.interview.java8;

import java.util.function.*;

public class PrimitiveSpecializationsDemo {

    public static void main(String[] args) {

        // ✅ Primitive-to-primitive — avoids ALL boxing
        IntUnaryOperator doubleIt = x -> x * 2;
        System.out.println(doubleIt.applyAsInt(21));  // 42

        // ✅ Primitive-to-object
        IntFunction<String> intToStr = Integer::toString;
        System.out.println(intToStr.apply(42));  // "42"

        // ✅ Object-to-primitive
        ToIntFunction<String> strToInt = String::length;
        System.out.println(strToInt.applyAsInt("Hello"));  // 5

        // ✅ Primitive predicates
        IntPredicate isEven = n -> n % 2 == 0;
        System.out.println(isEven.test(4));  // true

        // ✅ Primitive consumers
        IntConsumer printInt = System.out::println;
        printInt.accept(42);  // 42

        // ✅ Primitive suppliers
        IntSupplier randomInt = () -> (int) (Math.random() * 100);
        System.out.println("Random int: " + randomInt.getAsInt());

        // ✅ Primitive binary operators
        IntBinaryOperator intMax = Math::max;
        System.out.println(intMax.applyAsInt(10, 20));  // 20

        // ✅ Two-type primitive conversions
        ToIntBiFunction<String, String> totalLength =
                (a, b) -> a.length() + b.length();
        System.out.println(totalLength.applyAsInt("Hi", "World"));  // 7

        // ❌ Bad: uses boxed Integer — autoboxing on every call
        Function<Integer, Integer> boxedDouble = x -> x * 2;

        // ✅ Good: uses primitive int — zero boxing overhead
        IntUnaryOperator primitiveDouble = x -> x * 2;
    }
}
```

### Primitive Specialization Naming Pattern

| Pattern | Example | Signature |
|---------|---------|-----------|
| `IntXxx` | `IntFunction<R>` | `int → R` |
| `LongXxx` | `LongPredicate` | `long → boolean` |
| `DoubleXxx` | `DoubleConsumer` | `double → void` |
| `ToIntXxx` | `ToIntFunction<T>` | `T → int` |
| `XxxToInt` | `LongToIntFunction` | `long → int` |
| `ObjIntXxx` | `ObjIntConsumer<T>` | `(T, int) → void` |

### Composition Methods

```java
package com.interview.java8;

import java.util.List;
import java.util.function.*;
import java.util.stream.Collectors;

public class CompositionDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // Function composition: andThen, compose
        // ═══════════════════════════════════════════════════
        Function<String, String> trim = String::trim;
        Function<String, String> upper = String::toUpperCase;
        Function<String, Integer> length = String::length;

        // andThen: trim THEN upper → "  hello  " → "hello" → "HELLO"
        Function<String, String> trimThenUpper = trim.andThen(upper);
        System.out.println(trimThenUpper.apply("  hello  "));  // HELLO

        // compose: trim COMPOSED WITH upper → upper first, then trim
        // upper("  hello  ") = "  HELLO  ", then trim = "HELLO"
        Function<String, String> upperComposeTrim = trim.compose(upper);
        System.out.println(upperComposeTrim.apply("  hello  "));  // HELLO

        // Chaining: trim → upper → length
        Function<String, Integer> pipeline = trim.andThen(upper).andThen(length);
        System.out.println(pipeline.apply("  hello  "));  // 5

        // ═══════════════════════════════════════════════════
        // Predicate composition: and, or, negate
        // ═══════════════════════════════════════════════════
        Predicate<String> isNotEmpty = s -> !s.isEmpty();
        Predicate<String> isShort = s -> s.length() <= 5;
        Predicate<String> startsWithJ = s -> s.startsWith("J");

        // AND: both conditions must be true
        Predicate<String> shortAndStartsJ = isShort.and(startsWithJ);
        System.out.println(shortAndStartsJ.test("Java"));     // true
        System.out.println(shortAndStartsJ.test("JavaScript")); // false

        // OR: either condition
        Predicate<String> shortOrStartsJ = isShort.or(startsWithJ);
        System.out.println(shortOrStartsJ.test("JavaScript")); // true

        // NEGATE: invert
        Predicate<String> isEmpty = isNotEmpty.negate();
        System.out.println(isEmpty.test(""));  // true

        // Practical: complex filter with composition
        List<String> languages = List.of("Java", "JavaScript", "Python", "Go", "");
        List<String> result = languages.stream()
                .filter(isNotEmpty.and(isShort).and(startsWithJ))
                .collect(Collectors.toList());
        System.out.println(result);  // [Java]

        // ═══════════════════════════════════════════════════
        // Consumer composition: andThen
        // ═══════════════════════════════════════════════════
        Consumer<String> log = s -> System.out.println("LOG: " + s);
        Consumer<String> audit = s -> System.out.println("AUDIT: " + s);

        Consumer<String> logAndAudit = log.andThen(audit);
        logAndAudit.accept("User login");
        // LOG: User login
        // AUDIT: User login
    }
}
```

### Custom Functional Interfaces

```java
package com.interview.java8;

// ✅ Custom functional interface with @FunctionalInterface annotation
@FunctionalInterface
public interface Transformer<T> {
    T transform(T input);

    // ✅ Default methods are allowed — don't count toward SAM
    default Transformer<T> andThen(Transformer<T> after) {
        return input -> after.transform(this.transform(input));
    }

    // ✅ Static factory methods are allowed
    static <T> Transformer<T> identity() {
        return input -> input;
    }

    // ❌ Cannot add a second abstract method — would break @FunctionalInterface
    // String otherMethod();
}

// ✅ Functional interface with checked exception support
@FunctionalInterface
interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;

    // Wraps to standard Function, converting checked to unchecked
    static <T, R> java.util.function.Function<T, R> unchecked(ThrowingFunction<T, R> f) {
        return t -> {
            try {
                return f.apply(t);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        };
    }
}

// Usage:
// list.stream()
//     .map(ThrowingFunction.unchecked(item -> riskyOperation(item)))
//     .collect(Collectors.toList());
```

### ⚠️ Common Mistake
**Creating custom functional interfaces when a built-in one exists**: Java provides 43 built-in functional interfaces in `java.util.function`. Before creating `interface StringProcessor { String process(String s); }`, check if `UnaryOperator<String>` already fits. Custom interfaces are only warranted when you need a descriptive name for domain clarity, checked exception support, or more than two parameters.

### ⚡ Performance Tip
Always prefer **primitive specializations** (`IntPredicate`, `LongFunction`, `ToDoubleFunction`) over their generic counterparts when working with primitives. Using `Predicate<Integer>` instead of `IntPredicate` in a tight loop forces autoboxing/unboxing on every call — this can cause 3-5x slower performance and heavy garbage collection pressure due to the constant creation of `Integer` wrapper objects.

### 🔧 JVM Internals
The `@FunctionalInterface` annotation is purely a **compile-time check** — it has no effect on runtime behavior. Any interface with a single abstract method can be used as a lambda target, with or without the annotation. However, the annotation prevents accidental addition of a second abstract method and serves as documentation. Note that methods from `Object` (`toString`, `equals`, `hashCode`) don't count as abstract methods in functional interfaces because every class already inherits their implementation from `Object`.

---

## 🎯 Interviewer Insight
Interviewers expect you to name the core functional interfaces (`Function`, `Predicate`, `Consumer`, `Supplier`) and their method signatures from memory. A strong answer demonstrates knowledge of primitive specializations and their performance benefits, plus the ability to compose predicates and functions using `andThen`/`compose`/`and`/`or`. Mentioning the `ThrowingFunction` pattern for checked exceptions shows real-world experience.

## ⚡ Follow-up Questions
1. Why does `Function` have both `andThen` and `compose`? What's the execution order difference?
2. How many built-in functional interfaces does Java provide?
3. Can a functional interface extend another interface?

## 📚 Related Topics
- [Lambda Expressions](./q01-lambda-expressions.md)
- [Method References](./q02-method-references.md)
- [Stream API Operations](./q04-stream-api-operations.md)

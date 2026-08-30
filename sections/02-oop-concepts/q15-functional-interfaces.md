# Q15: What Are Functional Interfaces?

## Question
What are functional interfaces in Java? Explain `@FunctionalInterface`, the built-in functional interfaces, composition methods, custom functional interfaces, and primitive specializations.

## Difficulty: 🟡 Intermediate

---

## Answer

A **functional interface** is an interface with exactly **one abstract method** (SAM - Single Abstract Method). It can have any number of default, static, or private methods. Functional interfaces are the target types for lambda expressions and method references.

### @FunctionalInterface Annotation

```java
// ✅ Valid functional interface
@FunctionalInterface
public interface Converter<F, T> {
    T convert(F from); // Single abstract method

    // These don't count against SAM rule:
    default Converter<F, T> andThen(Converter<T, ?> after) { return null; }
    static <T> Converter<T, T> identity() { return t -> t; }

    // Object methods don't count either:
    boolean equals(Object obj);
    String toString();
}

// ❌ NOT a functional interface - two abstract methods
// @FunctionalInterface  // COMPILE ERROR
public interface InvalidFI {
    void method1();
    void method2();
}
```

### Built-in Functional Interfaces (java.util.function)

| Interface | Method | Signature | Use Case |
|-----------|--------|-----------|----------|
| `Function<T,R>` | `apply` | `T → R` | Transform data |
| `Predicate<T>` | `test` | `T → boolean` | Filter/validate |
| `Consumer<T>` | `accept` | `T → void` | Process/consume |
| `Supplier<T>` | `get` | `() → T` | Generate/provide |
| `UnaryOperator<T>` | `apply` | `T → T` | Transform same type |
| `BinaryOperator<T>` | `apply` | `(T, T) → T` | Combine two values |
| `BiFunction<T,U,R>` | `apply` | `(T, U) → R` | Two-input transform |
| `BiPredicate<T,U>` | `test` | `(T, U) → boolean` | Two-input filter |
| `BiConsumer<T,U>` | `accept` | `(T, U) → void` | Two-input process |

### Function<T, R>

```java
public class FunctionDemo {
    public static void main(String[] args) {
        Function<String, Integer> length = String::length;
        Function<Integer, String> intToStr = i -> "Number: " + i;

        // Apply
        System.out.println(length.apply("Hello")); // 5

        // ✅ Compose functions
        Function<String, String> lengthThenFormat = length.andThen(intToStr);
        System.out.println(lengthThenFormat.apply("Hello")); // "Number: 5"

        // ✅ compose (reverse order)
        Function<String, String> sameResult = intToStr.compose(length);
        System.out.println(sameResult.apply("Hello")); // "Number: 5"

        // ✅ Identity function
        Function<String, String> identity = Function.identity();
        System.out.println(identity.apply("unchanged")); // "unchanged"
    }
}
```

### Predicate<T>

```java
public class PredicateDemo {
    public static void main(String[] args) {
        Predicate<String> isNotEmpty = s -> !s.isEmpty();
        Predicate<String> isLong = s -> s.length() > 5;
        Predicate<String> startsWithA = s -> s.startsWith("A");

        // ✅ Combine predicates with and(), or(), negate()
        Predicate<String> complexCheck = isNotEmpty
            .and(isLong)
            .and(startsWithA.negate()); // Not starting with A

        var names = List.of("Alice", "Alexander", "Bob", "Charlie", "");
        var filtered = names.stream()
                            .filter(complexCheck)
                            .toList();
        System.out.println(filtered); // [Charlie]

        // ✅ Predicate.isEqual()
        Predicate<String> isHello = Predicate.isEqual("Hello");
        System.out.println(isHello.test("Hello")); // true

        // ✅ Predicate.not() (Java 11+)
        var nonEmpty = names.stream()
                            .filter(Predicate.not(String::isEmpty))
                            .toList();
    }
}
```

### Consumer<T> and Supplier<T>

```java
public class ConsumerSupplierDemo {
    public static void main(String[] args) {
        // Consumer: takes input, returns nothing
        Consumer<String> printer = System.out::println;
        Consumer<String> logger = s -> System.out.println("[LOG] " + s);

        // ✅ Chain consumers with andThen
        Consumer<String> printAndLog = printer.andThen(logger);
        printAndLog.accept("Hello");
        // Hello
        // [LOG] Hello

        // Supplier: takes nothing, returns a value
        Supplier<List<String>> listFactory = ArrayList::new;
        Supplier<Double> randomSupplier = Math::random;

        // ✅ Lazy initialization
        List<String> newList = listFactory.get();
        System.out.println(randomSupplier.get()); // Random number
    }
}
```

### UnaryOperator<T> and BinaryOperator<T>

```java
public class OperatorDemo {
    public static void main(String[] args) {
        // UnaryOperator: T → T (special case of Function<T,T>)
        UnaryOperator<String> toUpper = String::toUpperCase;
        UnaryOperator<String> trim = String::trim;
        UnaryOperator<String> pipeline = trim.andThen(toUpper);

        System.out.println(pipeline.apply("  hello  ")); // "HELLO"

        // BinaryOperator: (T, T) → T (special case of BiFunction<T,T,T>)
        BinaryOperator<Integer> max = Integer::max;
        BinaryOperator<String> concat = String::concat;

        System.out.println(max.apply(10, 20)); // 20
        System.out.println(concat.apply("Hello, ", "World")); // "Hello, World"

        // ✅ Used in Stream.reduce()
        var numbers = List.of(1, 2, 3, 4, 5);
        int sum = numbers.stream().reduce(0, Integer::sum);
        System.out.println(sum); // 15
    }
}
```

### Custom Functional Interfaces

```java
// ✅ Custom functional interface with checked exception
@FunctionalInterface
public interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;

    // Wrapper to convert to standard Function
    static <T, R> Function<T, R> unchecked(ThrowingFunction<T, R> fn) {
        return t -> {
            try {
                return fn.apply(t);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        };
    }
}

// Usage
List<String> urls = List.of("https://example.com", "https://google.com");
var contents = urls.stream()
    .map(ThrowingFunction.unchecked(url -> new URL(url).openStream().toString()))
    .toList();
```

### Composition (andThen, compose)

```java
public class CompositionDemo {
    public static void main(String[] args) {
        Function<String, String> trim = String::trim;
        Function<String, String> lower = String::toLowerCase;
        Function<String, String> sanitize = s -> s.replaceAll("[^a-z0-9]", "");

        // ✅ Build a processing pipeline
        Function<String, String> pipeline = trim
            .andThen(lower)
            .andThen(sanitize);

        System.out.println(pipeline.apply("  Hello, World! 123  ")); // "helloworld123"

        // ✅ Predicate composition for validation
        Predicate<String> isEmail = s -> s.contains("@");
        Predicate<String> hasMinLength = s -> s.length() >= 5;
        Predicate<String> noDots = s -> !s.startsWith(".");

        Predicate<String> isValidEmail = isEmail
            .and(hasMinLength)
            .and(noDots);

        System.out.println(isValidEmail.test("a@b.com")); // true
        System.out.println(isValidEmail.test("ab"));       // false
    }
}
```

### Primitive Specializations

```java
// ✅ Avoid autoboxing overhead with primitive specializations
public class PrimitiveDemo {
    public static void main(String[] args) {
        // Instead of Function<Integer, Integer>:
        IntUnaryOperator doubleIt = x -> x * 2;
        System.out.println(doubleIt.applyAsInt(5)); // 10

        // Instead of Predicate<Integer>:
        IntPredicate isEven = x -> x % 2 == 0;
        System.out.println(isEven.test(4)); // true

        // Instead of Consumer<Integer>:
        IntConsumer printer = System.out::println;
        printer.accept(42); // 42

        // Instead of Supplier<Integer>:
        IntSupplier randomInt = () -> (int) (Math.random() * 100);
        System.out.println(randomInt.getAsInt());

        // ToIntFunction: T → int
        ToIntFunction<String> stringLength = String::length;

        // IntToDoubleFunction: int → double
        IntToDoubleFunction half = x -> x / 2.0;

        // IntStream uses these for efficiency
        int sum = IntStream.rangeClosed(1, 100)
                           .filter(isEven)
                           .map(doubleIt)
                           .sum();
        System.out.println(sum); // 5100
    }
}
```

### Common Mistake

```java
// ❌ Creating unnecessary custom functional interfaces
@FunctionalInterface
interface StringProcessor {
    String process(String input);
}
// ✅ Just use UnaryOperator<String> or Function<String, String>

// ❌ Overcomplicating with too many generic type parameters
@FunctionalInterface
interface TriFunction<A, B, C, R> {
    R apply(A a, B b, C c);
}
// ✅ Consider using a parameter object or method instead

// ❌ Lambda that captures mutable state
int[] counter = {0};
list.forEach(item -> counter[0]++); // Side effect - hard to parallelize
// ✅ Use Stream.count() or Collectors instead
```

### Performance Tip
Always prefer **primitive specializations** (`IntFunction`, `LongPredicate`, `DoubleConsumer`, etc.) over their generic counterparts when working with primitives. This avoids autoboxing/unboxing overhead, which can be significant in tight loops or stream pipelines.

---

## Interviewer Insight
Interviewers expect you to know the core four (`Function`, `Predicate`, `Consumer`, `Supplier`) and explain composition (`andThen`, `compose`). Understanding primitive specializations and when to create custom functional interfaces shows depth.

## Follow-up Questions
1. What is the difference between `Function.andThen()` and `Function.compose()`?
2. Why can't lambda expressions throw checked exceptions by default?
3. How do functional interfaces enable the Strategy pattern?

## Related Topics
- [Lambdas and OOP](./q14-lambdas-and-oop.md)
- [Default Methods in Interfaces](./q16-default-methods-interfaces.md)
- [Interfaces vs Abstract Classes](./q07-interfaces-vs-abstract-classes.md)

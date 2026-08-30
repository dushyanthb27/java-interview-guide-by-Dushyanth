# Q14: How Do Lambdas Relate to OOP?

## Question
How do lambda expressions relate to Object-Oriented Programming in Java? Explain closure semantics, effectively final variables, and the difference between lambdas and anonymous classes.

## Difficulty: 🟡 Intermediate

---

## Answer

**Lambda expressions** (Java 8+) are concise implementations of **functional interfaces** (interfaces with a single abstract method). They bridge OOP and functional programming by treating behavior as a first-class concept that can be passed around like data.

### Lambdas as Interface Implementations

```java
// A lambda IS an instance of a functional interface
@FunctionalInterface
public interface Transformer<T, R> {
    R transform(T input);
}

public class LambdaDemo {
    public static void main(String[] args) {
        // ✅ Lambda - concise syntax for single-method interface
        Transformer<String, Integer> lengthFn = s -> s.length();

        // ✅ Method reference - even more concise
        Transformer<String, Integer> lengthRef = String::length;

        // These are equivalent to:
        Transformer<String, Integer> anon = new Transformer<>() {
            @Override
            public Integer transform(String input) {
                return input.length();
            }
        };

        System.out.println(lengthFn.transform("Hello")); // 5
    }
}
```

### Functional Programming Meets OOP

```java
// ✅ Passing behavior as parameters (Strategy pattern simplified)
public class DataPipeline {
    private final List<String> data;

    public DataPipeline(List<String> data) {
        this.data = new ArrayList<>(data);
    }

    // Accept behavior as a parameter
    public DataPipeline filter(Predicate<String> condition) {
        var filtered = data.stream().filter(condition).toList();
        return new DataPipeline(filtered);
    }

    public DataPipeline transform(UnaryOperator<String> transformation) {
        var transformed = data.stream().map(transformation).toList();
        return new DataPipeline(transformed);
    }

    public void forEach(Consumer<String> action) {
        data.forEach(action);
    }
}

// Usage - behavior passed as lambdas
new DataPipeline(List.of("Alice", "Bob", "Charlie", "Dave"))
    .filter(name -> name.length() > 3)          // Predicate
    .transform(String::toUpperCase)              // UnaryOperator
    .forEach(name -> System.out.println(name));  // Consumer
// Output: ALICE, CHARLIE, DAVE
```

### Closure Semantics

A lambda can **capture** variables from its enclosing scope. This creates a **closure**:

```java
public class ClosureDemo {
    private String prefix = "Hello";

    public List<String> greet(List<String> names) {
        String punctuation = "!"; // Local variable

        return names.stream()
            .map(name -> prefix + ", " + name + punctuation) // Captures prefix and punctuation
            .toList();

        // Lambda captures:
        // 1. 'this.prefix' - instance field (via implicit 'this' reference)
        // 2. 'punctuation' - local variable (must be effectively final)
    }
}
```

### Effectively Final Variables

Captured local variables must be **effectively final** (assigned once, never modified after):

```java
public class EffectivelyFinalDemo {
    public void demonstrate() {
        String greeting = "Hello"; // Effectively final (never reassigned)
        int count = 0;             // Will be modified - NOT effectively final

        // ✅ OK - greeting is effectively final
        Runnable r1 = () -> System.out.println(greeting);

        // ❌ COMPILE ERROR - count is modified later
        // Runnable r2 = () -> System.out.println(count);

        count++; // This assignment makes 'count' not effectively final

        // ✅ Workaround: use an array or AtomicInteger
        var counter = new AtomicInteger(0);
        Runnable r3 = () -> System.out.println(counter.incrementAndGet());
    }
}
```

### Lambda vs Anonymous Class - Key Differences

```java
public class DifferenceDemo {
    private String name = "DifferenceDemo";

    public void show() {
        // 1. 'this' reference
        Runnable lambda = () -> {
            System.out.println(this.name);  // "DifferenceDemo" - enclosing class
        };

        Runnable anon = new Runnable() {
            @Override
            public void run() {
                // System.out.println(this.name);  // ERROR - 'this' is the anonymous class
                System.out.println(this.getClass()); // Anonymous class
            }
        };

        // 2. Compilation
        // Lambda: uses invokedynamic (no extra .class file)
        // Anonymous: generates DifferenceDemo$1.class

        // 3. Shadowing
        String x = "outer";
        Runnable lambdaShadow = () -> {
            // String x = "inner";  // ❌ COMPILE ERROR - cannot shadow in lambda
        };

        Runnable anonShadow = new Runnable() {
            @Override
            public void run() {
                String x = "inner";  // ✅ OK - anonymous class has its own scope
            }
        };
    }
}
```

### Method References

Four types of method references provide even more concise lambda syntax:

```java
public class MethodRefDemo {
    public static void main(String[] args) {
        List<String> names = List.of("Charlie", "Alice", "Bob");

        // 1. Static method reference
        // Lambda: (s) -> Integer.parseInt(s)
        Function<String, Integer> parser = Integer::parseInt;

        // 2. Instance method of a particular object
        // Lambda: (s) -> System.out.println(s)
        Consumer<String> printer = System.out::println;

        // 3. Instance method of an arbitrary object of a type
        // Lambda: (s) -> s.toUpperCase()
        Function<String, String> upper = String::toUpperCase;

        // 4. Constructor reference
        // Lambda: (s) -> new StringBuilder(s)
        Function<String, StringBuilder> sbCreator = StringBuilder::new;

        // Using them together
        names.stream()
             .map(String::toUpperCase)        // Type 3
             .sorted(String::compareTo)       // Type 3
             .forEach(System.out::println);   // Type 2
    }
}
```

### Lambdas in Design Patterns

```java
// ✅ Strategy pattern without boilerplate classes
public class Validator {
    private final List<Function<String, Optional<String>>> rules = new ArrayList<>();

    public Validator addRule(Function<String, Optional<String>> rule) {
        rules.add(rule);
        return this;
    }

    public List<String> validate(String input) {
        return rules.stream()
                    .map(rule -> rule.apply(input))
                    .filter(Optional::isPresent)
                    .map(Optional::get)
                    .toList();
    }
}

// Usage - strategies defined as lambdas
var validator = new Validator()
    .addRule(s -> s.length() < 3 ? Optional.of("Too short") : Optional.empty())
    .addRule(s -> s.contains(" ") ? Optional.of("No spaces") : Optional.empty())
    .addRule(s -> s.equals(s.toLowerCase()) ? Optional.of("Need uppercase") : Optional.empty());

List<String> errors = validator.validate("hi");
// ["Too short", "Need uppercase"]
```

### Common Mistake

```java
// ❌ Modifying external state in lambdas (side effects)
public class SideEffectBug {
    public void process(List<String> items) {
        var results = new ArrayList<String>();

        // ❌ Mutating shared state from parallel stream lambda - NOT thread-safe!
        items.parallelStream().forEach(item -> results.add(item.toUpperCase()));

        // ✅ Use a functional approach instead
        var safeResults = items.parallelStream()
                               .map(String::toUpperCase)
                               .toList(); // Thread-safe collection
    }
}

// ❌ Heavy lambdas that should be methods
Comparator<Employee> comp = (e1, e2) -> {
    if (e1.getDepartment().equals(e2.getDepartment())) {
        if (e1.getSalary() != e2.getSalary()) {
            return Double.compare(e2.getSalary(), e1.getSalary());
        }
        return e1.getName().compareTo(e2.getName());
    }
    return e1.getDepartment().compareTo(e2.getDepartment());
};

// ✅ Extract complex logic into a method and use method reference
Comparator<Employee> comp = Comparator
    .comparing(Employee::getDepartment)
    .thenComparing(Employee::getSalary, Comparator.reverseOrder())
    .thenComparing(Employee::getName);
```

### Performance Tip
Lambdas use `invokedynamic` at the bytecode level, which the JVM optimizes at runtime. On first call, the JVM generates a lightweight implementation class. Subsequent calls are as fast as direct method calls. Lambdas are generally faster than anonymous classes due to avoiding class loading.

---

## Interviewer Insight
Interviewers want to see that you understand lambdas as **functional interface instances**, know the `this` scoping difference from anonymous classes, and can explain effectively final semantics. Demonstrating how lambdas simplify design patterns shows practical experience.

## Follow-up Questions
1. Can a lambda expression throw checked exceptions?
2. What is the difference between `invokedynamic` and `invokeinterface`?
3. How do lambdas interact with serialization?

## Related Topics
- [Functional Interfaces](./q15-functional-interfaces.md)
- [Inner Classes](./q13-inner-classes.md)
- [Default Methods in Interfaces](./q16-default-methods-interfaces.md)

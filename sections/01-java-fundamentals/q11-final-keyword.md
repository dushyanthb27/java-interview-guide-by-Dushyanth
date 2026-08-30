# Q11: Explain the final Keyword and Its Various Uses

## 📋 Question
Explain the `final` keyword in Java and its various uses. How does `final` apply to variables, methods, classes, and parameters? What is the difference between `final` and immutability?

## Difficulty: 🟡 Medium

---

## 💡 Answer

The `final` keyword in Java is a **non-access modifier** that means "cannot be changed after initialization." Its exact behavior depends on where it's applied:

1. **`final` variable**: The value (or reference) cannot be reassigned after initialization. For primitives, the value is constant. For objects, the **reference** is constant, but the **object's internal state can still be modified** — `final` does NOT mean immutable.

2. **`final` method**: The method cannot be **overridden** by subclasses. Used to prevent subclasses from changing critical behavior (e.g., template method pattern's fixed steps).

3. **`final` class**: The class cannot be **extended** (no subclasses). Examples: `String`, `Integer`, all wrapper classes, `System`, `Math`. Used when the class's design is not meant for inheritance.

4. **`final` parameter**: The parameter cannot be reassigned inside the method. Signals intent and prevents accidental reassignment.

5. **Effectively final** (Java 8+): A variable that is not declared `final` but is never reassigned after initialization. Effectively final variables can be used in **lambda expressions** and **anonymous inner classes** without explicitly adding the `final` keyword.

### The Critical Distinction: final ≠ Immutable

`final` prevents **reassignment of the reference**, not modification of the object. A `final List` can still have elements added or removed — only the reference to the list is fixed.

### Code Example

```java
package com.interview.fundamentals;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class FinalKeywordDemo {

    // === FINAL VARIABLES ===

    // Compile-time constant (static final + literal)
    public static final double PI = 3.14159265358979;
    public static final String APP_NAME = "InterviewGuide";

    // Blank final — initialized in constructor (different per instance)
    private final String userId;
    private final long createdAt;

    // Static blank final — initialized in static block
    private static final String CONFIG_VALUE;

    static {
        // Simulate loading from configuration
        CONFIG_VALUE = System.getProperty("app.config", "default");
    }

    public FinalKeywordDemo(String userId) {
        this.userId = userId;        // ✅ Blank final must be initialized here
        this.createdAt = System.currentTimeMillis();
        // this.userId = "changed";  // ❌ Compile error: already assigned
    }

    public void demonstrate() {
        // === FINAL LOCAL VARIABLES ===
        final int maxRetries = 3;
        // maxRetries = 5;  // ❌ Compile error: cannot reassign

        // === FINAL vs IMMUTABILITY — The Critical Difference ===

        // final prevents REFERENCE reassignment, NOT object mutation
        final List<String> list = new ArrayList<>();
        list.add("Hello");       // ✅ Modifying the object is allowed
        list.add("World");       // ✅ Still allowed — the reference hasn't changed
        // list = new ArrayList<>(); // ❌ Compile error: cannot reassign reference

        // ✅ For true immutability, use unmodifiable/immutable collections
        List<String> immutable = List.of("Hello", "World");       // Java 9+
        // immutable.add("Nope");  // ❌ UnsupportedOperationException at runtime

        List<String> unmodifiable = Collections.unmodifiableList(list);
        // unmodifiable.add("Nope"); // ❌ UnsupportedOperationException at runtime

        // === EFFECTIVELY FINAL (Java 8+) ===
        String greeting = "Hello";  // Not declared final, but never reassigned
        // greeting is "effectively final" — can be used in lambdas

        Runnable r = () -> System.out.println(greeting);  // ✅ Works
        r.run();

        // If we uncomment the next line, the lambda above would NOT compile:
        // greeting = "Hi";  // This would make greeting NOT effectively final
    }

    // === FINAL PARAMETERS ===
    public void process(final String input, final int count) {
        // input = "modified";  // ❌ Compile error: final parameter
        // count = 0;           // ❌ Compile error: final parameter
        System.out.println(input.repeat(count));
    }

    public static void main(String[] args) {
        var demo = new FinalKeywordDemo("user-001");
        demo.demonstrate();
        demo.process("Ha", 3);
    }
}
```

### Final Methods and Classes

```java
package com.interview.fundamentals;

// === FINAL CLASS — cannot be subclassed ===
// public final class String { ... }   // Real JDK — cannot extend String
// public final class Integer { ... }  // Real JDK — cannot extend Integer

final class SecurityValidator {
    public boolean validate(String token) {
        // Critical security logic that must not be overridden
        return token != null && token.startsWith("Bearer ");
    }
}

// class CustomValidator extends SecurityValidator { }  // ❌ Compile error

// === FINAL METHOD — cannot be overridden ===
class BaseTemplate {

    // Template method pattern: final method defines the algorithm skeleton
    public final void execute() {
        validate();
        process();    // Subclasses customize this step
        cleanup();
    }

    // ❌ Cannot be overridden — fixed behavior
    // @Override execute() in a subclass → Compile error

    protected void validate() {
        System.out.println("Default validation");
    }

    // Hook method — subclasses MUST override
    protected void process() {
        throw new UnsupportedOperationException("Subclass must implement");
    }

    private void cleanup() {
        System.out.println("Cleanup complete");
    }
}

class ConcreteTask extends BaseTemplate {

    // ✅ Can override non-final methods
    @Override
    protected void process() {
        System.out.println("Custom processing logic");
    }

    // ❌ Cannot override final method
    // @Override public void execute() { }  // Compile error
}
```

### Effectively Final Deep Dive

```java
package com.interview.fundamentals;

import java.util.function.Function;

public class EffectivelyFinalDemo {

    public static void main(String[] args) {

        // ✅ Effectively final — never reassigned, usable in lambdas
        String prefix = "Hello";
        Function<String, String> greeter = name -> prefix + " " + name;
        System.out.println(greeter.apply("Alice"));  // "Hello Alice"

        // ❌ NOT effectively final — reassigned, cannot use in lambda
        String mutablePrefix = "Hi";
        mutablePrefix = "Hey";  // Reassignment makes it NOT effectively final
        // Function<String, String> broken = name -> mutablePrefix + " " + name;
        // Compile error: Variable used in lambda should be final or effectively final

        // ✅ Workaround: use a final wrapper (array or AtomicReference)
        final int[] counter = {0};  // Array is final, but contents are mutable
        Runnable incrementer = () -> counter[0]++;  // ✅ Works
        incrementer.run();
        incrementer.run();
        System.out.println("Counter: " + counter[0]);  // 2
        // ⚠️ This is a code smell — consider AtomicInteger for thread safety
    }
}
```

### Constant Naming and Best Practices

```java
public class FinalBestPractices {

    // ✅ Constants: static final with SCREAMING_SNAKE_CASE
    public static final int MAX_CONNECTIONS = 100;
    public static final String DEFAULT_CHARSET = "UTF-8";
    public static final List<String> SUPPORTED_FORMATS =
        List.of("JSON", "XML", "CSV");  // Immutable list — truly constant

    // ❌ Anti-pattern: mutable static final (looks constant but isn't!)
    public static final List<String> BAD_CONSTANT = new ArrayList<>();
    // Anyone can call BAD_CONSTANT.add("oops") — defeats the purpose

    // ✅ Best practices:
    // 1. Make fields final whenever possible (signals intent, enables optimizations)
    // 2. Use final for method parameters (prevents accidental reassignment)
    // 3. Prefer final classes unless you explicitly design for extension
    // 4. Use final + unmodifiable for true constant collections
    // 5. Don't use final on local variables just for the sake of it
    //    (effectively final is sufficient for lambdas)
}
```

### ⚠️ Common Mistake
**Assuming `final` means immutable**: A `final List<String>` can still be modified (`add`, `remove`, `clear`). Only the **reference** is frozen. For true immutability, combine `final` with an immutable collection (`List.of()`, `Map.of()`, or `Collections.unmodifiable*()`). Another mistake: declaring a `static final` field with a mutable object (like `ArrayList`) as a "constant" — any code can modify its contents, leading to subtle bugs. Always use `List.of()` or `Collections.unmodifiableList()` for constant collections.

### ⚡ Performance Tip
The JVM treats `static final` primitive and String constants as **compile-time constants** — their values are **inlined** directly into the bytecode of every class that references them. This means: (1) changing a constant requires **recompiling all dependent classes**, not just the class that declares it; (2) accessing a compile-time constant has zero overhead — no field lookup at all. `final` fields also enable the JVM's **memory model guarantees**: a `final` field is guaranteed to be visible to all threads once the constructor completes, without explicit synchronization. This is why `final` fields are essential for thread-safe immutable objects.

### 🔧 JVM Internals
At the bytecode level, `final` fields are marked with the `ACC_FINAL` flag in the class file's field access flags. For `static final` fields initialized with constant expressions (primitives and Strings), the value is stored in the class file's **ConstantValue** attribute and loaded directly — the static initializer block is not needed. The JVM's memory model provides a **freeze** action for `final` fields: when a constructor completes, all writes to `final` fields are guaranteed to be visible to any thread that obtains a reference to the object, even without synchronization. This is the foundation of safely publishing immutable objects across threads. The JIT compiler can also perform more aggressive optimizations on `final` fields (constant folding, loop-invariant code motion) because it knows the value cannot change after initialization.

---

## 🎯 Interviewer Insight
Interviewers use this question to assess your understanding of the difference between **reference immutability** (`final`) and **object immutability** (unmodifiable + no setters). The best candidates explain that `final` is necessary but not sufficient for immutability, and can articulate the JMM (Java Memory Model) guarantee for `final` fields. Knowing that compile-time constants are inlined (and the recompilation implications) shows deep understanding. Mentioning effectively final and its role in lambda expressions demonstrates modern Java knowledge.

## ⚡ Follow-up Questions
1. What is the difference between a `final` field and an immutable object?
2. If you change the value of a `static final` constant, do you need to recompile dependent classes? Why?
3. How does the `final` keyword interact with the Java Memory Model for thread safety?

## 📚 Related Topics
- [Strings and String Pool](./q07-strings-and-string-pool.md)
- [Records in Java 14+](./q14-records-java14.md)
- [Sealed Classes](./q15-sealed-classes-java15.md)

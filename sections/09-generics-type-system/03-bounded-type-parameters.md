# Q3: What Are Bounded Type Parameters?

[Home](../../README.md) > [Generics & Type System](./README.md) > Bounded Type Parameters

---

## 🟡 Difficulty: Intermediate

## Question
> What are bounded type parameters in Java generics? How do upper bounds, multiple bounds, and recursive type bounds work?

---

## Quick Answer
**Bounded type parameters** restrict the types that can be used as type arguments. An **upper bound** (`<T extends Number>`) limits T to Number or its subclasses. **Multiple bounds** (`<T extends Comparable<T> & Serializable>`) require T to satisfy all listed types. **Recursive type bounds** (`<T extends Comparable<T>>`) constrain T to be comparable with itself — a pattern used in sorting and fluent APIs.

---

## Detailed Explanation

### Upper-Bounded Type Parameters

```java
// T must be Number or a subclass of Number
public class MathBox<T extends Number> {
    private T value;

    public MathBox(T value) {
        this.value = value;
    }

    // Can call Number methods on T
    public double doubleValue() {
        return value.doubleValue();
    }

    public int intValue() {
        return value.intValue();
    }
}

// Usage
MathBox<Integer> intBox = new MathBox<>(42);           // ✅ Integer extends Number
MathBox<Double> doubleBox = new MathBox<>(3.14);       // ✅ Double extends Number
// MathBox<String> stringBox = new MathBox<>("hello"); // ❌ COMPILE ERROR
```

### Bounded Generic Methods

```java
public class Calculator {

    // T must extend Number — allows calling Number methods
    public static <T extends Number> double sum(List<T> numbers) {
        double total = 0;
        for (T num : numbers) {
            total += num.doubleValue();  // Safe — T is at least Number
        }
        return total;
    }

    // T must be Comparable — allows natural ordering
    public static <T extends Comparable<T>> T findMax(List<T> items) {
        if (items.isEmpty()) throw new IllegalArgumentException("Empty list");
        T max = items.get(0);
        for (T item : items) {
            if (item.compareTo(max) > 0) {
                max = item;
            }
        }
        return max;
    }
}

// Usage
double total = Calculator.sum(List.of(1, 2, 3, 4.5));     // 10.5
String maxStr = Calculator.findMax(List.of("a", "z", "m")); // "z"
```

### Multiple Bounds

```java
// T must extend Comparable AND implement Serializable
// Syntax: class first, then interfaces (class must come first)
public class SortableCache<T extends Comparable<T> & Serializable> {
    private final List<T> items = new ArrayList<>();

    public void add(T item) {
        items.add(item);
        Collections.sort(items);  // Safe — T is Comparable
    }

    public byte[] serialize() throws IOException {
        // Safe — T is Serializable
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(new ArrayList<>(items));
        return baos.toByteArray();
    }
}

// String implements both Comparable<String> and Serializable
SortableCache<String> cache = new SortableCache<>();  // ✅
```

### Multiple Bounds — Ordering Rules

```java
// ✅ Class bound MUST come first, followed by interface bounds
<T extends AbstractClass & InterfaceA & InterfaceB>

// ❌ COMPILE ERROR — interface before class
// <T extends Serializable & Number>

// ✅ Correct ordering — class first
<T extends Number & Serializable>

// ✅ Multiple interface bounds (no class bound)
<T extends Comparable<T> & Serializable & Cloneable>

// ❌ Cannot have more than one class bound
// <T extends Number & String>  // COMPILE ERROR — only one class allowed
```

### Recursive Type Bounds

```java
// T must be comparable with itself
// This is the standard pattern for natural ordering
public static <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}

// Why recursive? Because T appears in its own bound:
//   T extends Comparable<T>
//   ↑                    ↑
//   same T ──────────────┘
```

### Recursive Bounds — Fluent Builder Pattern

```java
// Base builder with recursive type bound
public abstract class Builder<T extends Builder<T>> {
    private String name;
    private int age;

    @SuppressWarnings("unchecked")
    protected T self() {
        return (T) this;
    }

    public T name(String name) {
        this.name = name;
        return self();  // Returns the concrete subtype, not Builder
    }

    public T age(int age) {
        this.age = age;
        return self();
    }

    public abstract Object build();
}

// Concrete builder
public class UserBuilder extends Builder<UserBuilder> {
    private String email;

    @Override
    protected UserBuilder self() {
        return this;
    }

    public UserBuilder email(String email) {
        this.email = email;
        return this;
    }

    @Override
    public User build() {
        return new User(/* fields */);
    }
}

// Fluent chaining returns correct type at every step
User user = new UserBuilder()
    .name("Alice")         // Returns UserBuilder, not Builder
    .age(30)               // Returns UserBuilder, not Builder
    .email("a@test.com")   // Returns UserBuilder
    .build();
```

### Recursive Bounds — Enum Pattern

```java
// Java's Enum class uses recursive type bounds!
// public abstract class Enum<E extends Enum<E>> implements Comparable<E>

// Every enum implicitly extends Enum<itself>
public enum Color { RED, GREEN, BLUE }
// Compiler generates: class Color extends Enum<Color>

// This is why enums are naturally Comparable with themselves
Color c = Collections.max(List.of(Color.RED, Color.BLUE)); // BLUE
```

### Bounded Types and Method Access

```java
public class Demo {

    // Unbounded — can only call Object methods on T
    public static <T> String describe(T item) {
        return item.toString();    // ✅ toString() is on Object
        // item.doubleValue();     // ❌ not available
    }

    // Bounded by Number — can call Number methods
    public static <T extends Number> double toDouble(T num) {
        return num.doubleValue();  // ✅ doubleValue() is on Number
    }

    // Bounded by Comparable — can call compareTo()
    public static <T extends Comparable<T>> int compare(T a, T b) {
        return a.compareTo(b);     // ✅ compareTo() is on Comparable
    }

    // Multiple bounds — can call methods from all bounds
    public static <T extends Number & Comparable<T>> T max(T a, T b) {
        double da = a.doubleValue();  // ✅ from Number
        int cmp = a.compareTo(b);     // ✅ from Comparable
        return cmp >= 0 ? a : b;
    }
}
```

### Bounded Type in Inheritance

```java
// Generic interface with bound
public interface Summable<T extends Number> {
    T sum(T a, T b);
}

// Implementation fixes the type
public class IntegerSummable implements Summable<Integer> {
    @Override
    public Integer sum(Integer a, Integer b) {
        return a + b;
    }
}

// Implementation preserves the bound
public class GenericSummable<T extends Number> implements Summable<T> {
    @Override
    public T sum(T a, T b) {
        // Cannot use + operator on T — need workaround
        throw new UnsupportedOperationException();
    }
}
```

---

## Anti-Patterns

```java
// ❌ Overly restrictive bounds — limits reusability
public static <T extends Integer> void process(T value) { }
// Integer is final — T can ONLY be Integer, so generics add nothing

// ❌ Missing recursive bound causes type-safety gap
public static <T extends Comparable> T max(T a, T b) {  // Raw Comparable
    return a.compareTo(b) >= 0 ? a : b;  // Unchecked — can compare anything
}

// ❌ Forgetting that erasure replaces T with the first bound
public class Thing<T extends Serializable & Comparable<T>> {
    // T erases to Serializable (first bound)
    // Put the most-used type first if possible
}
```

## Best Practices

```java
// ✅ Use Comparable<? super T> for maximum flexibility
public static <T extends Comparable<? super T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
// Works with classes that inherit Comparable from a parent

// ✅ Place the class bound first (required by syntax)
<T extends Number & Comparable<T> & Serializable>

// ✅ Use recursive type bounds for fluent APIs
public abstract class Builder<B extends Builder<B>> {
    public abstract B self();
}

// ✅ Put the most frequently called bound first (better erased performance)
<T extends Comparable<T> & Serializable>  // compareTo() calls need no cast
```

---

## Interview Tips
- Upper bounds restrict acceptable type arguments and expose methods of the bound
- Multiple bounds use `&` syntax; **at most one class**, which must come first
- Recursive bounds (`T extends Comparable<T>`) are used in enums, builders, and sort APIs
- After erasure, T is replaced by its **first bound** (or `Object` if unbounded)
- `<T extends Comparable<? super T>>` is the idiomatic pattern for sort methods

---

[← Q2: Type Erasure](./q02-type-erasure.md) | [Next: Q4 - Upper-Bounded Wildcards →](./q04-wildcards-extends.md)

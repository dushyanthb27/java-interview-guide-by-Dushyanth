# Q1: How Do Generic Classes and Methods Work?

[Home](../../README.md) > [Generics & Type System](./README.md) > Generic Classes & Methods

---

## 🟢 Difficulty: Basic

## Question
> How do generic classes and methods work in Java? What are the conventions and scope rules for type parameters?

---

## Quick Answer
Generics allow classes, interfaces, and methods to operate on parameterized types. A **generic class** declares type parameters in angle brackets after the class name (`class Box<T>`). A **generic method** declares type parameters before the return type (`<T> T method(T t)`). Type parameters are replaced with actual types at compile time, providing type safety without casting.

---

## Detailed Explanation

### Generic Class Syntax

```java
// Basic generic class with one type parameter
public class Box<T> {
    private T content;

    public Box(T content) {
        this.content = content;
    }

    public T getContent() {
        return content;
    }

    public void setContent(T content) {
        this.content = content;
    }
}

// Usage
Box<String> stringBox = new Box<>("Hello");
String value = stringBox.getContent();  // No casting needed

Box<Integer> intBox = new Box<>(42);
Integer number = intBox.getContent();   // Type-safe
```

### Multiple Type Parameters

```java
// Generic class with multiple type parameters
public class Pair<K, V> {
    private final K key;
    private final V value;

    public Pair(K key, V value) {
        this.key = key;
        this.value = value;
    }

    public K getKey()   { return key; }
    public V getValue() { return value; }

    @Override
    public String toString() {
        return "(" + key + ", " + value + ")";
    }
}

// Triple with three type parameters
public class Triple<A, B, C> {
    private final A first;
    private final B second;
    private final C third;

    public Triple(A first, B second, C third) {
        this.first = first;
        this.second = second;
        this.third = third;
    }

    public A getFirst()  { return first; }
    public B getSecond() { return second; }
    public C getThird()  { return third; }
}

// Usage
Pair<String, Integer> entry = new Pair<>("age", 30);
Triple<String, Integer, Boolean> record = new Triple<>("Alice", 30, true);
```

### Generic Methods

```java
public class Utility {

    // Generic method — type parameter <T> declared before return type
    public static <T> T firstNonNull(T first, T second) {
        return first != null ? first : second;
    }

    // Generic method with different return type
    public static <T> List<T> singletonList(T element) {
        List<T> list = new ArrayList<>();
        list.add(element);
        return list;
    }

    // Multiple type parameters in a method
    public static <K, V> Map<K, V> mapOf(K key, V value) {
        Map<K, V> map = new HashMap<>();
        map.put(key, value);
        return map;
    }

    // Generic method in a non-generic class
    public static <T extends Comparable<T>> T max(T a, T b) {
        return a.compareTo(b) >= 0 ? a : b;
    }
}

// Usage — type inferred automatically
String name = Utility.firstNonNull(null, "default");
Integer bigger = Utility.max(10, 20);    // T inferred as Integer

// Explicit type witness (rarely needed)
String result = Utility.<String>firstNonNull(null, "fallback");
```

### Generic Interfaces

```java
// Generic interface
public interface Repository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    void save(T entity);
    void deleteById(ID id);
}

// Concrete implementation fixes the type parameters
public class UserRepository implements Repository<User, Long> {
    @Override
    public User findById(Long id) { /* ... */ return null; }

    @Override
    public List<User> findAll() { /* ... */ return List.of(); }

    @Override
    public void save(User entity) { /* ... */ }

    @Override
    public void deleteById(Long id) { /* ... */ }
}

// Generic implementation preserves type parameters
public abstract class AbstractRepository<T, ID> implements Repository<T, ID> {
    protected final Map<ID, T> store = new HashMap<>();

    @Override
    public T findById(ID id) {
        return store.get(id);
    }

    @Override
    public List<T> findAll() {
        return new ArrayList<>(store.values());
    }
}
```

### Type Parameter Naming Conventions

| Letter | Convention | Example |
|--------|-----------|---------|
| `T` | Type (general) | `Box<T>` |
| `E` | Element (collections) | `List<E>` |
| `K` | Key | `Map<K, V>` |
| `V` | Value | `Map<K, V>` |
| `R` | Return type | `Function<T, R>` |
| `N` | Number | `Calculator<N>` |
| `S, U` | 2nd, 3rd types | `Pair<S, U>` |

### Type Parameter Scope

```java
public class Container<T> {           // T is class-level scope

    private T item;                    // T visible in fields

    public T getItem() {               // T visible in instance methods
        return item;
    }

    // Method-level type parameter <U> — separate from class <T>
    public <U> Pair<T, U> pairWith(U other) {
        return new Pair<>(item, other);
    }

    // Static methods CANNOT use class-level T
    // public static T broken() { }   // ❌ COMPILE ERROR

    // Static methods must declare their own type parameters
    public static <S> S staticMethod(S input) {  // ✅ OK
        return input;
    }
}
```

### Generic Constructors

```java
public class Wrapper<T> {
    private T value;

    // Constructor can have its own type parameter
    public <S extends T> Wrapper(S value) {
        this.value = value;
    }
}

// Usage
Wrapper<Number> wrapper = new Wrapper<>(42);  // S=Integer, T=Number
```

---

## Anti-Patterns

```java
// ❌ Using raw types — loses all type safety
List list = new ArrayList();
list.add("hello");
list.add(42);                     // No compile error
String s = (String) list.get(1);  // ClassCastException at runtime!

// ❌ Unnecessary casting with generics
List<String> names = new ArrayList<>();
String name = (String) names.get(0);  // Cast is redundant

// ❌ Using Object instead of generics
public class ObjectBox {
    private Object content;
    public Object getContent() { return content; }  // Callers must cast
}

// ❌ Shadowing class type parameter in a method
public class Outer<T> {
    // This <T> shadows the class-level T — confusing!
    public <T> T confusingMethod(T t) { return t; }
}
```

## Best Practices

```java
// ✅ Always use parameterized types
List<String> names = new ArrayList<>();

// ✅ Use descriptive type parameter names for complex generics
public interface Converter<SOURCE, TARGET> {
    TARGET convert(SOURCE source);
}

// ✅ Prefer generic methods over raw-type utility methods
public static <T> List<T> merge(List<T> a, List<T> b) {
    List<T> result = new ArrayList<>(a);
    result.addAll(b);
    return result;
}

// ✅ Use different names for method and class type parameters
public class Cache<T> {
    public <U> Pair<T, U> associate(U metadata) {
        // U is clearly different from T
        return new Pair<>(/* ... */, metadata);
    }
}
```

---

## Interview Tips
- Generics are a **compile-time** feature — erased at runtime
- Type parameters on **static** members must be declared on the method itself
- Always use parameterized types; raw types exist only for backward compatibility
- The compiler inserts casts automatically when you use generics

---

[Next: Q2 - Type Erasure →](./q02-type-erasure.md)

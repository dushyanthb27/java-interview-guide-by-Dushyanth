# Q14: How to Design Custom Generic Types?

[Home](../../README.md) > [Generics & Type System](./Generics & Type System) > Custom Generic Types

---

## 🟠 Difficulty: Advanced

## Question

> How do you design custom generic classes, interfaces, and methods? What advanced patterns (such as type-safe heterogeneous containers, generic builders, and recursive type bounds) enable robust, reusable APIs?

---

## Quick Answer

Designing custom generic types involves selecting precise type parameter names (`T`, `E`, `K`, `V`), applying appropriate bounds (`<T extends Comparable<T>>`), adhering to PECS for method parameters, and employing advanced architectural patterns like **type-safe heterogeneous containers** (using `Class<T>` tokens) and **recursive type bounds** (self-types for fluent builders). Good generic design keeps APIs intuitive while eliminating unsafe casting boilerplate.

---

## Detailed Explanation

### 1. Type-Safe Heterogeneous Container Pattern

When you need a map that can store values of completely different types associated with their respective `Class` tokens type-safely:

```java
public class TypeSafeRegistry {
    private final Map<Class<?>, Object> map = new ConcurrentHashMap<>();

    public <T> void put(Class<T> type, T instance) {
        map.put(Objects.requireNonNull(type), type.cast(instance));
    }

    public <T> T get(Class<T> type) {
        return type.cast(map.get(type));
    }
}
```

### 2. Generic Repository Pattern

```java
public interface Repository<T, ID> {
    Optional<T> findById(ID id);
    T save(T entity);
    void deleteById(ID id);
}

public abstract class AbstractRepository<T, ID> implements Repository<T, ID> {
    protected final Map<ID, T> store = new ConcurrentHashMap<>();
    protected abstract ID extractId(T entity);

    @Override
    public T save(T entity) {
        store.put(extractId(entity), entity);
        return entity;
    }
}
```

### 3. Recursive Type Bounds (Fluent Self-Type Pattern)

```java
public abstract class AbstractBuilder<T, B extends AbstractBuilder<T, B>> {
    protected String name;

    @SuppressWarnings("unchecked")
    protected B self() {
        return (B) this;
    }

    public B name(String name) {
        this.name = name;
        return self();
    }

    public abstract T build();
}
```

---

## Gotchas

- **Type Token Erasure Limits**: `Class<List<String>>` is illegal because generic type arguments are erased at runtime; you cannot use parameterized types directly as class token keys without super-type token workarounds (e.g., `TypeLiteral`).
- **Over-Genericizing Simple Classes**: Adding unnecessary type parameters to classes or methods where a wildcard or concrete type suffices creates maintenance overhead and illegible signatures.
- **Static Context Blindness**: Type parameters defined on a generic class cannot be referenced inside `static` methods or fields of that class.
- **Raw Type Leakage in Subclasses**: Extending a generic class without specifying type parameters implicitly creates raw types, reintroducing compiler warnings.

---

## Anti-Patterns

```java
// ❌ Using Object instead of generic type parameters
public class Box {
    private Object item;
    public Object getItem() { return item; } // Requires unsafe casts for callers
}

// ❌ Adding unused type parameters to classes
public class Utility<T> {
    public static void print(String msg) { ... } // T is completely unused
}
```

## Best Practices

```java
// ✅ Use standard single-letter type parameter conventions (T, E, K, V, R)
public interface Processor<I, O> {
    O process(I input);
}

// ✅ Use Class<T> type tokens for runtime generic type safety
public <T> T load(Class<T> clazz, long id) { ... }

// ✅ Apply recursive type bounds for fluent builder inheritance
```

## Interview Tips

- Use **`Class<T>` tokens** for runtime type-safe casting and heterogeneous storage.
- **Recursive type bounds** (`<T extends Comparable<T>>` or `SELF extends Builder<SELF>`) enable fluent method chaining in subclassed builders.
- Generic type parameters **cannot** be used in `static` contexts because parameters belong to instance instantiation.
- Always apply **PECS** when designing consumer/producer generic method parameters.

\n
### ⚠️ Gotchas

- **Memory Leaks & Object Reference Retention**: Unintentional object retention via static references, unclosed resource handles, or long-lived caches preventing Garbage Collection.
- **Concurrency & Thread Safety**: Assuming thread safety on standard collections (`ArrayList`, `HashMap`) or primitives, leading to race conditions and data corruption.
- **Exception Masking & Swallowing**: Catching generic `Exception` or `Throwable` without proper logging, rethrowing, or handling suppressed exceptions in try-with-resources.
- **Performance Antipatterns**: Excessive object allocation in tight loops, improper stream boxing/unboxing overhead, or inefficient synchronization blocks causing thread contention.
\n---\n

[← Q13: Covariance & Contravariance](./13-covariance-contravariance.md) | [Next: Q15 - Advanced Generics Patterns →](./15-advanced-generics-patterns.md)

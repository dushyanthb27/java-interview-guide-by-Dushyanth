# Q13: Explain Covariance and Contravariance

[Home](../../README.md) > [Generics & Type System](./Generics & Type System) > Covariance & Contravariance

---

## 🟠 Difficulty: Advanced

## Question

> What are covariance, contravariance, and invariance? How do they apply to Java arrays, generics, and wildcards, and what is the PECS principle?

---

## Quick Answer

**Covariance** preserves the subtype hierarchy (`Dog extends Animal` $\implies$ `List<? extends Animal>` accepts `List<Dog>`). **Contravariance** reverses the subtype hierarchy (`Dog extends Animal` $\implies$ `List<? super Dog>` accepts `List<Animal>`). **Invariance** enforces exact type matching (`List<Dog>` is unrelated to `List<Animal>`). Java arrays are covariant by language design, generics are invariant by default, and wildcards enable **use-site variance**. The golden rule is **PECS**: **P**roducer **E**xtends, **C**onsumer **S**uper.

---

## Detailed Explanation

### The Three Variance Models

```
Given: Dog extends Animal

1. COVARIANT (Preserves hierarchy direction):
   List<? extends Animal> can be assigned a List<Dog> or List<Cat>.
   - Safe for READING (producing) items as Animal.
   - Unsafe for WRITING (except null).

2. CONTRAVARIANT (Reverses hierarchy direction):
   List<? super Dog> can be assigned a List<Dog>, List<Animal>, or List<Object>.
   - Safe for WRITING (consuming) Dog instances.
   - Reading only returns Object.

3. INVARIANT (No subtype relationship):
   List<Dog> cannot be assigned to List<Animal> (and vice versa).
   - Fully safe for both reading and writing exact type Dog.
```

### The PECS Principle (Producer Extends, Consumer Super)

- **PECS**: **P**roducer **E**xtends, **C**onsumer **S**uper.
- **Producer Extends (`? extends T`)**: Use this wildcard when your method **gets/reads** data out of a structure. The structure produces items of type `T` (or its subtypes).
- **Consumer Super (`? super T`)**: Use this wildcard when your method **puts/writes** data into a structure. The structure consumes items of type `T` (or its supertypes).

```java
public class CollectionUtils {
    // Src is a Producer of T -> use ? extends T
    // Dest is a Consumer of T -> use ? super T
    public static <T> void copy(List<? super T> dest, List<? extends T> src) {
        for (T item : src) {
            dest.add(item);
        }
    }
}
```

### Variance in Functional Interfaces

```java
// Function<T, R>: T is consumed (contravariant), R is produced (covariant)
Function<Animal, String> f1 = Animal::getName;
Function<? super Dog, ? extends CharSequence> f2 = f1; // ✅ Valid variance mapping
```

---

## Gotchas

- **Wildcard Capture Errors**: Trying to write to a `List<? extends Animal>` directly inside a method results in a compilation error because the exact subtype (`Dog`, `Cat`, etc.) cannot be guaranteed at compile time.
- **Get vs. Put Principle Confusion**: Forgetting PECS leads to overly restrictive method signatures that reject valid subtyping assignments.
- **Array Covariance Runtime Traps**: Assuming Java arrays are type-safe because they are covariant results in `ArrayStoreException` at runtime.
- **Wildcard Capture with Helper Methods**: When compiler type inference fails on wildcards (e.g., in `List<?>`), helper methods are required to capture the wildcard type parameter explicitly.

---

## Anti-Patterns

```java
// ❌ Using extends when you need to write/consume items
public void addAll(List<? extends Number> list) {
    list.add(42); // ❌ COMPILE ERROR! Cannot add to ? extends
}

// ❌ Using super when you only need to read items
public double sum(List<? super Number> list) {
    // return list.stream().mapToDouble(...); // ❌ Only returns Object, arithmetic fails
}
```

## Best Practices

```java
// ✅ Follow PECS strictly for generic method parameters
public void processItems(List<? extends Item> producer, List<? super Item> consumer) {
    consumer.addAll(producer);
}

// ✅ Use bounded wildcards on API boundaries to maximize flexibility
public void printNames(List<? extends NamedEntity> entities) {
    entities.forEach(e -> System.out.println(e.getName()));
}
```

## Interview Tips

- Master **PECS**: **P**roducer **E**xtends, **C**onsumer **S**uper.
- Arrays are **covariant** (with runtime checks); generics are **invariant** (with compile-time safety).
- Covariance (`? extends`) allows safe **reading**; contravariance (`? super`) allows safe **writing**.
- Java uses **use-site** variance (wildcards), whereas Kotlin/Scala support **declaration-site** variance (`out`/`in`).

---

[← Q12: Generics & Legacy Code](./12-generics-legacy-code.md) | [Next: Q14 - Custom Generic Types →](./14-custom-generic-types.md)

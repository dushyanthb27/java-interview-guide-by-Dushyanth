# Q12: How to Mix Generics with Legacy Code?

[Home](../../README.md) > [Generics & Type System](./README.md) > Generics & Legacy Code

---

## 🟡 Difficulty: Intermediate

## Question

> How do you safely integrate modern generic code with pre-Java 5 legacy code using raw types? What are unchecked warnings, heap pollution risks, and migration strategies?

---

## Quick Answer

Java generics maintain backward compatibility through **type erasure**, allowing generic code and pre-Java 5 raw types (`List` instead of `List<String>`) to interoperate within the same JVM. However, mixing them introduces **unchecked warnings** and risks **heap pollution** if legacy code inserts incorrect types into typed collections. Migration requires incremental generification, narrow use of `@SuppressWarnings("unchecked")`, and `Collections.checkedXxx()` runtime safety wrappers when interacting with untrusted legacy boundaries.

---

## Detailed Explanation

### Raw Types vs. Parameterized Types

```java
// Raw type (pre-Java 5 style — bypasses generic type checks)
List rawList = new ArrayList();
rawList.add("hello");
rawList.add(42); // Allowed on raw type!

// Parameterized type (type-safe)
List<String> typedList = new ArrayList<>();
typedList.add("hello");
// typedList.add(42); // ❌ COMPILE ERROR
```

### Passing Generics to Legacy Code (Risk of Heap Pollution)

```java
public class LegacySystem {
    public static void corrupt(List list) {
        list.add(100); // Legacy code doesn't know about List<String>
    }
}

// Modern caller:
List<String> names = new ArrayList<>(List.of("Alice"));
LegacySystem.corrupt(names); // Heap pollution occurs!
// String s = names.get(0); // Works for "Alice"
// String t = names.get(1); // 💥 ClassCastException when accessing Integer 100!
```

### `Collections.checkedXxx()` Runtime Wrappers

To protect generic collections passed into legacy code that might corrupt them, use Java's checked collection wrappers:

```java
List<String> safeNames = Collections.checkedList(new ArrayList<>(), String.class);

// Pass to legacy code:
LegacySystem.corrupt(safeNames); 
// 💥 Throws ClassCastException immediately upon insertion of Integer, 
// rather than failing later during retrieval!
```
Available wrappers include `Collections.checkedList`, `checkedSet`, `checkedMap`, and `checkedQueue`.

### Narrowing `@SuppressWarnings("unchecked")`

```java
// ❌ BAD: Suppressing warnings across an entire method or class
@SuppressWarnings("unchecked")
public void processAll() {
    List<String> a = legacy1();
    List<Integer> b = legacy2();
    // Hides all new type errors here!
}

// ✅ GOOD: Scoped strictly to the assignment line with justification
public List<User> loadUsers() {
    @SuppressWarnings("unchecked")
    // Safe: Legacy DAO guarantees returned list contains only User instances
    List<User> users = (List<User>) legacyDao.findAllRaw();
    return users;
}
```

---

## Gotchas

- **Silent Binary Compatibility Traps**: Generic code and raw type code compile to identical bytecode under erasure, meaning binary compatibility is preserved even when type safety is violated at boundaries.
- **Unchecked Conversion on Return**: Returning a raw type from a method assigned to a generic variable triggers an unchecked conversion warning that, if ignored without verification, causes downstream `ClassCastException` bugs.
- **Checked Wrapper Performance Overhead**: `Collections.checkedList` performs runtime `ClassCastException` checks on every `add` or `set` operation, introducing minor overhead.
- **Type Erasure Reflection Blanks**: Inspecting legacy raw collections via reflection at runtime yields raw `Object` types, losing generic parameter metadata.

---

## Anti-Patterns

```java
// ❌ Using raw types in new codebases
List items = new ArrayList(); // Avoid raw types entirely

// ❌ Blindly suppressing warnings without verifying the legacy contract
@SuppressWarnings("unchecked")
List<String> data = (List<String>) unsafeSource.getData(); // High risk of ClassCastException
```

## Best Practices

```java
// ✅ Never use raw types in modern code
List<String> names = new ArrayList<>();

// ✅ Use Collections.checkedList when interfacing with mutable legacy APIs
List<String> protectedList = Collections.checkedList(new ArrayList<>(), String.class);

// ✅ Keep @SuppressWarnings("unchecked") narrow and commented
```

## Interview Tips

- Raw types exist solely for **backward compatibility** with pre-Java 5 code.
- Type erasure guarantees **binary compatibility** between generic and raw `.class` files.
- `Collections.checkedList()` enforces **runtime** type checks to catch heap pollution early.
- `@SuppressWarnings("unchecked")` must always be scoped as narrowly as possible.

---

[← Q11: Generic Arrays](./q11-generic-arrays.md) | [Next: Q13 - Covariance & Contravariance →](./q13-covariance-contravariance.md)

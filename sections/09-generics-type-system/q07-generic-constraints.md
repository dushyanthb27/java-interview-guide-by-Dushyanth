# Q7: What Are Generic Constraints and Limitations?

[Home](../../README.md) > [Generics & Type System](./README.md) > Generic Constraints

---

## 🟡 Difficulty: Intermediate

## Question
> What are the constraints and limitations of Java generics? What can't you do with type parameters?

---

## Quick Answer
Due to type erasure, Java generics have several limitations: you cannot instantiate type parameters (`new T()`), use primitives as type arguments, create generic arrays (`new T[]`), use `instanceof` with parameterized types, declare static fields of a type parameter, or catch/throw type parameters. Each limitation has established workarounds.

---

## Detailed Explanation

### 1. Cannot Instantiate Type Parameters

```java
public class Factory<T> {
    // ❌ COMPILE ERROR — T is erased to Object at runtime
    // public T create() {
    //     return new T();
    // }

    // ✅ Workaround 1: Pass Class<T> token
    private final Class<T> type;

    public Factory(Class<T> type) {
        this.type = type;
    }

    public T create() throws Exception {
        return type.getDeclaredConstructor().newInstance();
    }

    // ✅ Workaround 2: Pass Supplier<T>
    public static <T> T createWith(Supplier<T> supplier) {
        return supplier.get();
    }
}

// Usage
Factory<StringBuilder> factory = new Factory<>(StringBuilder.class);
StringBuilder sb = factory.create();

String s = Factory.createWith(String::new);
```

### 2. Cannot Use Primitive Types

```java
// ❌ Primitives are not allowed as type arguments
// List<int> ints = new ArrayList<>();         // COMPILE ERROR
// Map<int, double> map = new HashMap<>();     // COMPILE ERROR

// ✅ Use wrapper types instead (autoboxing handles conversion)
List<Integer> ints = new ArrayList<>();
ints.add(42);                                  // Autoboxing: int → Integer
int value = ints.get(0);                       // Unboxing: Integer → int

Map<Integer, Double> map = new HashMap<>();
map.put(1, 3.14);                              // Autoboxing both

// ⚠️ Performance consideration: autoboxing creates objects
// For performance-critical code, use specialized collections:
// IntStream, LongStream, DoubleStream
// or third-party: Eclipse Collections IntList, Trove TIntArrayList
```

### 3. Cannot Create Generic Arrays

```java
// ❌ COMPILE ERROR — cannot create array of parameterized type
// List<String>[] arrayOfLists = new List<String>[10];

// ❌ COMPILE ERROR — cannot create array of type parameter
public class Container<T> {
    // T[] items = new T[10];  // COMPILE ERROR
}

// ✅ Workaround 1: Use Object array with cast
@SuppressWarnings("unchecked")
public class Container<T> {
    private T[] items;

    public Container(int size) {
        items = (T[]) new Object[size];  // Unchecked cast
    }
}

// ✅ Workaround 2: Use Array.newInstance()
public class SafeContainer<T> {
    private T[] items;

    @SuppressWarnings("unchecked")
    public SafeContainer(Class<T> type, int size) {
        items = (T[]) java.lang.reflect.Array.newInstance(type, size);
    }
}

// ✅ Workaround 3: Use List<T> instead (preferred)
public class BestContainer<T> {
    private List<T> items = new ArrayList<>();
}
```

### 4. Cannot Use instanceof with Parameterized Types

```java
public static <T> void check(Object obj) {
    // ❌ COMPILE ERROR — parameterized type erased at runtime
    // if (obj instanceof List<String>) { }
    // if (obj instanceof T) { }

    // ✅ Can check raw type
    if (obj instanceof List<?>) {
        System.out.println("It's a List");
    }

    // ✅ Can check unbounded wildcard
    if (obj instanceof Map<?, ?>) {
        System.out.println("It's a Map");
    }

    // ✅ Workaround: pass Class<T> for runtime type check
}

public static <T> boolean isType(Object obj, Class<T> type) {
    return type.isInstance(obj);  // ✅ Runtime check via Class token
}

// Usage
boolean isString = isType("hello", String.class);   // true
boolean isInteger = isType("hello", Integer.class);  // false
```

### 5. Cannot Use Type Parameters in Static Context

```java
public class Singleton<T> {
    // ❌ COMPILE ERROR — static field cannot use instance type parameter
    // private static T instance;

    // ❌ COMPILE ERROR — static method cannot use instance type parameter
    // public static T getInstance() { return instance; }

    // Why? T belongs to each INSTANCE, but static belongs to the CLASS.
    // Singleton<String> and Singleton<Integer> share the SAME static members.

    // ✅ Static methods can declare their OWN type parameters
    public static <U> U defaultValue(Class<U> type) throws Exception {
        return type.getDeclaredConstructor().newInstance();
    }
}
```

### 6. Cannot Catch or Throw Type Parameters

```java
// ❌ Generic class cannot extend Throwable
// class GenericException<T> extends Exception { }   // COMPILE ERROR

// ❌ Cannot catch type parameter
// public <T extends Exception> void handle() {
//     try {
//         riskyOperation();
//     } catch (T e) { }  // COMPILE ERROR — T is erased
// }

// ✅ CAN use type parameter in throws clause
public <T extends Exception> void doWork(T exception) throws T {
    throw exception;  // ✅ Valid
}

// ✅ CAN use specific exception types
public <T extends Exception> void tryOrThrow(
        Callable<?> task, Supplier<T> exceptionSupplier) throws T {
    try {
        task.call();
    } catch (Exception e) {
        throw exceptionSupplier.get();
    }
}
```

### 7. Cannot Overload Methods That Erase to Same Signature

```java
public class Processor {
    // ❌ COMPILE ERROR — both erase to process(List)
    // public void process(List<String> strings) { }
    // public void process(List<Integer> integers) { }

    // ✅ Workaround 1: different method names
    public void processStrings(List<String> strings) { }
    public void processIntegers(List<Integer> integers) { }

    // ✅ Workaround 2: generic method
    public <T> void process(List<T> items) { }

    // ✅ Workaround 3: use different parameter lists
    public void process(List<String> strings, String marker) { }
    public void process(List<Integer> integers, Integer marker) { }
}
```

### 8. Cannot Create Generic Enum

```java
// ❌ Enums cannot have type parameters
// enum Response<T> {
//     SUCCESS(data), ERROR(message);
//     T payload;
// }

// ✅ Workaround: use generic methods or interfaces
interface Response<T> {
    T getData();
    boolean isSuccess();
}

class SuccessResponse<T> implements Response<T> {
    private final T data;
    SuccessResponse(T data) { this.data = data; }
    public T getData() { return data; }
    public boolean isSuccess() { return true; }
}
```

### Summary Table

| Limitation | Reason | Workaround |
|-----------|--------|------------|
| `new T()` | T erased to Object | `Class<T>` token or `Supplier<T>` |
| `List<int>` | Primitives not objects | Wrapper types (`Integer`) |
| `new T[]` | Array stores runtime type | `Object[]` cast or `Array.newInstance()` |
| `instanceof T` | T erased at runtime | `Class<T>.isInstance()` |
| `static T field` | T is per-instance | Separate type param on static method |
| `catch (T e)` | T erased at runtime | Catch concrete types |
| Overload erasure | Both erase to same sig | Different method names |
| `enum E<T>` | Enum spec forbids it | Generic interface + classes |

---

## Anti-Patterns

```java
// ❌ Suppressing warnings without understanding the root cause
@SuppressWarnings("unchecked")
public <T> T unsafeCast(Object obj) {
    return (T) obj;  // Dangerous — no runtime check
}

// ❌ Using raw types to bypass restrictions
List rawList = new ArrayList();  // Loses ALL type safety
rawList.add("string");
rawList.add(42);

// ❌ Ignoring heap pollution warnings
@SafeVarargs  // Only valid if method truly is safe
public static <T> List<T> asList(T... elements) {
    return Arrays.asList(elements);  // Potential heap pollution
}
```

## Best Practices

```java
// ✅ Use Class<T> token for runtime type operations
public static <T> T safeCast(Object obj, Class<T> type) {
    if (type.isInstance(obj)) {
        return type.cast(obj);
    }
    throw new ClassCastException("Cannot cast to " + type.getName());
}

// ✅ Prefer collections over arrays for generic storage
public class TypeSafeStore<T> {
    private final List<T> items = new ArrayList<>();  // Preferred over T[]
}

// ✅ Document @SuppressWarnings with justification
@SuppressWarnings("unchecked")  // Safe: array only accessed via typed methods
private T[] createArray(int size) {
    return (T[]) new Object[size];
}

// ✅ Use Supplier<T> for deferred instantiation
public static <T> T createOrDefault(Supplier<T> factory, T defaultValue) {
    try {
        return factory.get();
    } catch (Exception e) {
        return defaultValue;
    }
}
```

---

## Interview Tips
- All generic limitations stem from **type erasure** (backward compatibility with pre-Java 5)
- The `Class<T>` token is the universal workaround for runtime type info
- `@SuppressWarnings("unchecked")` should be as narrow as possible with a comment
- Prefer `List<T>` over `T[]` — it avoids the generic array creation problem
- Know at least 4-5 limitations and their workarounds for interviews

---

[← Q6: PECS Principle](./q06-pecs-principle.md) | [Next: Q8 - Type Inference →](./q08-type-inference.md)

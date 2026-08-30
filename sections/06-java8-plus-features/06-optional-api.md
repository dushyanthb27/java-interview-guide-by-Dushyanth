# Q6: How to Use Optional API Effectively?

## 📋 Question
How should the Optional API be used in Java? Explain creation, transformation, retrieval methods, and the common anti-patterns to avoid.

## Difficulty: 🟡 Medium

---

## 💡 Answer

`Optional<T>` (Java 8) is a **container object** that may or may not hold a non-null value. It was designed to express the **possibility of absence** in return types, replacing the error-prone practice of returning `null`. Optional forces callers to explicitly handle the "no value" case, making code more readable and reducing `NullPointerException` bugs.

### Creating Optionals

```java
package com.interview.java8;

import java.util.Optional;

public class OptionalCreationDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // of(value) — wraps a NON-NULL value
        // ═══════════════════════════════════════════════════
        Optional<String> present = Optional.of("Hello");
        // Optional.of(null);  // ❌ Throws NullPointerException immediately!

        // ═══════════════════════════════════════════════════
        // ofNullable(value) — wraps a value that MIGHT be null
        // ═══════════════════════════════════════════════════
        String name = getUserName();  // might return null
        Optional<String> maybe = Optional.ofNullable(name);

        // ═══════════════════════════════════════════════════
        // empty() — explicitly represents absence
        // ═══════════════════════════════════════════════════
        Optional<String> absent = Optional.empty();

        System.out.println(present.isPresent());  // true
        System.out.println(absent.isPresent());   // false
        System.out.println(absent.isEmpty());     // true (Java 11+)
    }

    static String getUserName() { return null; }
}
```

### Transformation Methods

```java
package com.interview.java8;

import java.util.Optional;

public class OptionalTransformDemo {

    record User(String name, Address address) {}
    record Address(String city, String zip) {}

    public static void main(String[] args) {
        Optional<User> user = findUser("alice");

        // ═══════════════════════════════════════════════════
        // map() — transforms value if present, wraps result in Optional
        // ═══════════════════════════════════════════════════
        Optional<String> cityName = user
                .map(User::address)        // Optional<Address>
                .map(Address::city);       // Optional<String>
        System.out.println(cityName.orElse("Unknown"));

        // ═══════════════════════════════════════════════════
        // flatMap() — use when transformation itself returns Optional
        // ═══════════════════════════════════════════════════
        Optional<String> zip = user
                .flatMap(OptionalTransformDemo::getAddress)  // returns Optional<Address>
                .map(Address::zip);
        System.out.println(zip.orElse("N/A"));

        // ═══════════════════════════════════════════════════
        // filter() — keep value only if predicate matches
        // ═══════════════════════════════════════════════════
        Optional<User> longName = user
                .filter(u -> u.name().length() > 3);

        // ✅ Chain operations — elegant null-safe navigation
        String result = findUser("alice")
                .flatMap(OptionalTransformDemo::getAddress)
                .map(Address::city)
                .filter(city -> city.length() > 2)
                .map(String::toUpperCase)
                .orElse("UNKNOWN");
        System.out.println("City: " + result);
    }

    static Optional<User> findUser(String id) {
        return Optional.of(new User("alice", new Address("Seattle", "98101")));
    }

    static Optional<Address> getAddress(User user) {
        return Optional.ofNullable(user.address());
    }
}
```

### Retrieval Methods

```java
package com.interview.java8;

import java.util.Optional;
import java.util.NoSuchElementException;

public class OptionalRetrievalDemo {

    public static void main(String[] args) {
        Optional<String> present = Optional.of("Hello");
        Optional<String> absent = Optional.empty();

        // ═══════════════════════════════════════════════════
        // orElse(default) — returns value or default (always evaluated)
        // ═══════════════════════════════════════════════════
        String value1 = absent.orElse("default");              // "default"
        String value2 = present.orElse("default");             // "Hello"

        // ═══════════════════════════════════════════════════
        // orElseGet(Supplier) — returns value or computes default lazily
        // ═══════════════════════════════════════════════════
        String value3 = absent.orElseGet(() -> computeExpensiveDefault());
        // ✅ Supplier is ONLY called when value is absent

        // ⚠️ Key difference: orElse vs orElseGet
        // orElse ALWAYS evaluates the argument (even when value is present)
        present.orElse(computeExpensiveDefault());    // ❌ Default computed unnecessarily!
        present.orElseGet(() -> computeExpensiveDefault()); // ✅ Supplier NOT called

        // ═══════════════════════════════════════════════════
        // orElseThrow() — throws exception if absent
        // ═══════════════════════════════════════════════════
        String value4 = present.orElseThrow();  // Java 10+ — throws NoSuchElementException
        String value5 = present.orElseThrow(
                () -> new IllegalStateException("Value is required!"));

        // ═══════════════════════════════════════════════════
        // get() — ❌ AVOID — throws NoSuchElementException if absent
        // ═══════════════════════════════════════════════════
        // String bad = absent.get(); // ❌ NoSuchElementException — no better than NPE!

        // ═══════════════════════════════════════════════════
        // ifPresent / ifPresentOrElse
        // ═══════════════════════════════════════════════════
        present.ifPresent(v -> System.out.println("Got: " + v));

        // Java 9+: ifPresentOrElse
        absent.ifPresentOrElse(
                v -> System.out.println("Got: " + v),
                () -> System.out.println("No value!")
        );
    }

    static String computeExpensiveDefault() {
        System.out.println("  Computing expensive default...");
        return "expensive";
    }
}
```

### Java 9+ Enhancements

```java
package com.interview.java8;

import java.util.*;
import java.util.stream.*;

public class OptionalJava9PlusDemo {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════════
        // or(Supplier<Optional>) — Java 9: fallback to another Optional
        // ═══════════════════════════════════════════════════
        Optional<String> result = findInCache("key")
                .or(() -> findInDatabase("key"))
                .or(() -> findInRemoteService("key"));
        System.out.println(result.orElse("not found"));

        // ═══════════════════════════════════════════════════
        // stream() — Java 9: convert Optional to Stream (0 or 1 elements)
        // ═══════════════════════════════════════════════════
        List<Optional<String>> optionals = List.of(
                Optional.of("Alice"),
                Optional.empty(),
                Optional.of("Bob"),
                Optional.empty(),
                Optional.of("Charlie")
        );

        // ✅ Flatten list of Optionals into list of present values
        List<String> presentValues = optionals.stream()
                .flatMap(Optional::stream)   // Empty optionals produce empty stream
                .collect(Collectors.toList());
        System.out.println(presentValues);  // [Alice, Bob, Charlie]
    }

    static Optional<String> findInCache(String key) { return Optional.empty(); }
    static Optional<String> findInDatabase(String key) { return Optional.of("DB:" + key); }
    static Optional<String> findInRemoteService(String key) { return Optional.empty(); }
}
```

### Anti-Patterns to Avoid

```java
package com.interview.java8;

import java.util.Optional;

public class OptionalAntiPatternsDemo {

    // ❌ ANTI-PATTERN 1: Optional.get() without isPresent()
    void bad1(Optional<String> opt) {
        // String value = opt.get();  // ❌ NoSuchElementException if empty
        String value = opt.orElseThrow();  // ✅ At least explicit about the risk
    }

    // ❌ ANTI-PATTERN 2: Optional as a field
    // Optional is not Serializable — it's NOT meant for fields
    // private Optional<String> middleName;  // ❌ Don't do this
    private String middleName;  // ✅ Use null and document nullability

    // ❌ ANTI-PATTERN 3: Optional as method parameter
    // void process(Optional<String> name) { }  // ❌ Forces caller to wrap
    void process(String name) { }  // ✅ Use @Nullable annotation if needed

    // ❌ ANTI-PATTERN 4: Using Optional.of() for nullable values
    Optional<String> bad4(String input) {
        // return Optional.of(input);  // ❌ NPE if input is null
        return Optional.ofNullable(input);  // ✅ Safe for nullable
    }

    // ❌ ANTI-PATTERN 5: isPresent() + get() instead of functional style
    void bad5(Optional<String> opt) {
        // ❌ Imperative style — defeats the purpose of Optional
        // if (opt.isPresent()) {
        //     System.out.println(opt.get());
        // } else {
        //     System.out.println("empty");
        // }

        // ✅ Functional style — concise and expressive
        opt.ifPresentOrElse(
                System.out::println,
                () -> System.out.println("empty")
        );
    }

    // ❌ ANTI-PATTERN 6: Wrapping collection in Optional
    // Optional<List<String>> bad — empty list suffices
    // ✅ Return empty collection instead: Collections.emptyList()

    // ✅ CORRECT: Optional as return type for single-value queries
    Optional<String> findUserById(String id) {
        // return Optional.ofNullable(database.find(id));
        return Optional.empty();
    }
}
```

### When to Use Optional

| Scenario | Use Optional? | Alternative |
|----------|--------------|-------------|
| Method return type (single value) | ✅ Yes | — |
| Stream operations | ✅ Yes | — |
| Class fields | ❌ No | Use `null` with `@Nullable` |
| Method parameters | ❌ No | Overloaded methods or `@Nullable` |
| Collections that might be empty | ❌ No | Return empty collection |
| Performance-critical hot paths | ❌ No | Use null checks (avoid object creation) |

### ⚠️ Common Mistake
**Using `orElse()` with an expensive computation**: `orElse()` **always evaluates** its argument, even when the Optional has a value. This means `opt.orElse(expensiveQuery())` runs the query even when `opt` is present. Use `orElseGet(() -> expensiveQuery())` instead — the `Supplier` is only invoked when the value is actually absent. This is a subtle but important performance difference in production code.

### ⚡ Performance Tip
Each `Optional.of()` or `Optional.ofNullable()` with a non-null value allocates a new object on the heap. In hot loops or high-throughput code paths, this GC pressure can be significant. `Optional.empty()` returns a cached singleton (no allocation), but wrapping a value always allocates. For performance-critical code, consider using null checks directly. The JVM's escape analysis can sometimes eliminate Optional allocations, but this optimization is not guaranteed.

### 🔧 JVM Internals
`Optional` is a **value-based class** — its identity should not be relied upon. Do not use `==` to compare Optionals, do not synchronize on them, and do not serialize them. Java's Project Valhalla plans to make Optional a **value type** (inline class), which would eliminate the heap allocation overhead entirely. Until then, Optional has the same memory footprint as any small object: 16 bytes header + 8 bytes reference field = 24 bytes per instance.

---

## 🎯 Interviewer Insight
Interviewers test whether you know the **anti-patterns** as well as the correct usage. The key signals: never using `Optional.get()` without checking, knowing the `orElse` vs `orElseGet` difference, and understanding that Optional is for **return types only** — not fields or parameters. Demonstrating the `or()` chaining pattern (Java 9) and `Optional::stream` for flattening shows modern Java fluency.

## ⚡ Follow-up Questions
1. Why is Optional not Serializable?
2. When should you use `orElse` vs `orElseGet` vs `orElseThrow`?
3. How does Optional interact with Stream's `flatMap`?

## 📚 Related Topics
- [Stream API Operations](./q04-stream-api-operations.md)
- [Stream Collectors](./q05-stream-collectors.md)
- [Records](./q14-records-detail.md)

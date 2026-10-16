# Q4: Explain Stream API Operations

## 📋 Question
Explain the Stream API operations in Java. Differentiate between intermediate and terminal operations, and describe lazy evaluation, short-circuiting, and stream pipeline optimization.

## Difficulty: 🟠 Senior

---

## 💡 Answer

The Stream API (Java 8) provides a **declarative, functional pipeline** for processing collections of data. A stream pipeline consists of a **source**, zero or more **intermediate operations** (lazy, return a new Stream), and a **terminal operation** (eager, triggers execution and produces a result).

### Intermediate Operations (Lazy — Return Stream)

```java
package com.interview.java8;

import java.util.*;
import java.util.stream.*;

public class IntermediateOpsDemo {

    record Employee(String name, String dept, double salary) {}

    public static void main(String[] args) {
        List<Employee> employees = List.of(
            new Employee("Alice", "Engineering", 95000),
            new Employee("Bob", "Marketing", 85000),
            new Employee("Charlie", "Engineering", 110000),
            new Employee("Diana", "Marketing", 92000),
            new Employee("Eve", "Engineering", 95000),
            new Employee("Frank", "HR", 78000)
        );

        // ═══════════════════════════════════════════════════
        // filter(Predicate<T>) — keeps elements matching the predicate
        // ═══════════════════════════════════════════════════
        employees.stream()
                .filter(e -> e.salary() > 90000)
                .forEach(e -> System.out.println("High earner: " + e.name()));

        // ═══════════════════════════════════════════════════
        // map(Function<T, R>) — transforms each element
        // ═══════════════════════════════════════════════════
        List<String> names = employees.stream()
                .map(Employee::name)
                .collect(Collectors.toList());

        // ═══════════════════════════════════════════════════
        // flatMap(Function<T, Stream<R>>) — one-to-many transformation
        // ═══════════════════════════════════════════════════
        List<List<String>> nested = List.of(
                List.of("a", "b"), List.of("c", "d"), List.of("e"));
        List<String> flat = nested.stream()
                .flatMap(Collection::stream)   // Flatten nested lists
                .collect(Collectors.toList());
        System.out.println(flat);  // [a, b, c, d, e]

        // ═══════════════════════════════════════════════════
        // peek(Consumer<T>) — side-effect without modifying (for debugging)
        // ═══════════════════════════════════════════════════
        List<String> debugged = names.stream()
                .peek(n -> System.out.println("Before filter: " + n))
                .filter(n -> n.startsWith("A"))
                .peek(n -> System.out.println("After filter: " + n))
                .collect(Collectors.toList());

        // ═══════════════════════════════════════════════════
        // distinct() — removes duplicates (uses equals/hashCode)
        // ═══════════════════════════════════════════════════
        List<Double> uniqueSalaries = employees.stream()
                .map(Employee::salary)
                .distinct()
                .collect(Collectors.toList());

        // ═══════════════════════════════════════════════════
        // sorted() / sorted(Comparator) — sorts elements
        // ═══════════════════════════════════════════════════
        List<Employee> bySalary = employees.stream()
                .sorted(Comparator.comparingDouble(Employee::salary).reversed())
                .collect(Collectors.toList());

        // ═══════════════════════════════════════════════════
        // limit(n) / skip(n) — pagination
        // ═══════════════════════════════════════════════════
        List<Employee> page2 = employees.stream()
                .skip(2)      // Skip first 2
                .limit(2)     // Take next 2
                .collect(Collectors.toList());

        // ═══════════════════════════════════════════════════
        // mapToInt / mapToLong / mapToDouble — primitive streams
        // ═══════════════════════════════════════════════════
        double totalSalary = employees.stream()
                .mapToDouble(Employee::salary)  // Stream<Employee> → DoubleStream
                .sum();                          // Primitive sum — no boxing
        System.out.println("Total salary: " + totalSalary);
    }
}
```

### Terminal Operations (Eager — Trigger Execution)

```java
package com.interview.java8;

import java.util.*;
import java.util.stream.*;

public class TerminalOpsDemo {

    public static void main(String[] args) {
        List<Integer> numbers = List.of(3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5);

        // ═══════════════════════════════════════════════════
        // collect(Collector) — accumulates into a collection/result
        // ═══════════════════════════════════════════════════
        Set<Integer> uniqueNums = numbers.stream()
                .collect(Collectors.toSet());

        // ═══════════════════════════════════════════════════
        // forEach(Consumer) — performs action on each element
        // ═══════════════════════════════════════════════════
        numbers.stream()
                .filter(n -> n > 4)
                .forEach(n -> System.out.print(n + " "));  // 5 9 6 5 5
        System.out.println();

        // ═══════════════════════════════════════════════════
        // reduce — combines elements into a single result
        // ═══════════════════════════════════════════════════
        // With identity: always returns a value
        int sum = numbers.stream().reduce(0, Integer::sum);
        System.out.println("Sum: " + sum);  // 44

        // Without identity: returns Optional (stream might be empty)
        Optional<Integer> product = numbers.stream()
                .reduce((a, b) -> a * b);
        product.ifPresent(p -> System.out.println("Product: " + p));

        // ═══════════════════════════════════════════════════
        // count, min, max
        // ═══════════════════════════════════════════════════
        long count = numbers.stream().filter(n -> n > 3).count();
        Optional<Integer> min = numbers.stream().min(Integer::compareTo);
        Optional<Integer> max = numbers.stream().max(Integer::compareTo);
        System.out.println("Count>3: " + count + ", Min: " + min.orElse(0)
                + ", Max: " + max.orElse(0));

        // ═══════════════════════════════════════════════════
        // findFirst / findAny — short-circuiting
        // ═══════════════════════════════════════════════════
        Optional<Integer> first = numbers.stream()
                .filter(n -> n > 5)
                .findFirst();  // Deterministic order
        System.out.println("First > 5: " + first.orElse(-1));  // 9

        Optional<Integer> any = numbers.parallelStream()
                .filter(n -> n > 5)
                .findAny();   // Non-deterministic in parallel
        System.out.println("Any > 5: " + any.orElse(-1));

        // ═══════════════════════════════════════════════════
        // anyMatch, allMatch, noneMatch — short-circuiting predicates
        // ═══════════════════════════════════════════════════
        boolean hasLarge = numbers.stream().anyMatch(n -> n > 8);     // true
        boolean allPositive = numbers.stream().allMatch(n -> n > 0);  // true
        boolean noNegative = numbers.stream().noneMatch(n -> n < 0);  // true

        // ═══════════════════════════════════════════════════
        // toArray — collects into an array
        // ═══════════════════════════════════════════════════
        Integer[] arr = numbers.stream()
                .filter(n -> n > 3)
                .toArray(Integer[]::new);
        System.out.println("Array: " + Arrays.toString(arr));
    }
}
```

### Lazy Evaluation and Pipeline Optimization

```java
package com.interview.java8;

import java.util.List;
import java.util.stream.Collectors;

public class LazyEvaluationDemo {

    public static void main(String[] args) {
        List<String> names = List.of("Charlie", "Alice", "Bob", "David", "Eve");

        // ✅ Lazy evaluation — operations execute ONLY when terminal op is called
        // NOTHING happens until .collect() is called
        var stream = names.stream()
                .filter(n -> {
                    System.out.println("  filter: " + n);
                    return n.length() > 3;
                })
                .map(n -> {
                    System.out.println("  map: " + n);
                    return n.toUpperCase();
                });

        System.out.println("Stream created — no operations executed yet!");
        System.out.println("Now triggering terminal operation...");

        // Elements are processed ONE AT A TIME through the pipeline:
        // filter("Charlie") → true → map("Charlie") → "CHARLIE"
        // filter("Alice")   → true → map("Alice")   → "ALICE"
        // filter("Bob")     → false (skip map)
        // filter("David")   → true → map("David")   → "DAVID"
        // filter("Eve")     → false (skip map)
        List<String> result = stream.collect(Collectors.toList());
        System.out.println(result);  // [CHARLIE, ALICE, DAVID]
    }
}
```

### Short-Circuiting Operations

```java
package com.interview.java8;

import java.util.stream.Stream;

public class ShortCircuitDemo {

    public static void main(String[] args) {
        // ✅ Short-circuiting: stops processing as soon as result is determined

        // findFirst — stops after finding the first match
        Stream.of("apple", "banana", "cherry", "date")
                .peek(s -> System.out.println("Processing: " + s))
                .filter(s -> s.startsWith("c"))
                .findFirst();
        // Output: Processing: apple, Processing: banana, Processing: cherry
        // "date" is NEVER processed

        // limit — stops after collecting n elements
        Stream.iterate(1, n -> n + 1)  // Infinite stream!
                .filter(n -> n % 2 == 0)
                .limit(5)              // Without this, it would run forever
                .forEach(n -> System.out.print(n + " "));  // 2 4 6 8 10
        System.out.println();

        // anyMatch — stops at first true
        boolean found = Stream.of(1, 2, 3, 4, 5)
                .peek(n -> System.out.println("Checking: " + n))
                .anyMatch(n -> n == 3);
        // Only checks 1, 2, 3 — stops immediately
    }
}
```

### Operation Classification Table

| Operation | Type | Short-Circuit | Returns |
|-----------|------|---------------|---------|
| `filter` | Intermediate | No | `Stream<T>` |
| `map` | Intermediate | No | `Stream<R>` |
| `flatMap` | Intermediate | No | `Stream<R>` |
| `peek` | Intermediate | No | `Stream<T>` |
| `distinct` | Intermediate (Stateful) | No | `Stream<T>` |
| `sorted` | Intermediate (Stateful) | No | `Stream<T>` |
| `limit` | Intermediate | **Yes** | `Stream<T>` |
| `skip` | Intermediate (Stateful) | No | `Stream<T>` |
| `collect` | Terminal | No | `R` |
| `forEach` | Terminal | No | `void` |
| `reduce` | Terminal | No | `Optional<T>` / `T` |
| `count` | Terminal | No | `long` |
| `findFirst` | Terminal | **Yes** | `Optional<T>` |
| `findAny` | Terminal | **Yes** | `Optional<T>` |
| `anyMatch` | Terminal | **Yes** | `boolean` |
| `allMatch` | Terminal | **Yes** | `boolean` |
| `noneMatch` | Terminal | **Yes** | `boolean` |

### ⚠️ Common Mistake
**Reusing a stream**: Streams are **single-use** — once a terminal operation is called, the stream is consumed. Calling a second terminal operation on the same stream throws `IllegalStateException`. If you need to perform multiple operations on the same data, either create a new stream each time or collect intermediate results. Also, **`peek` should never be used for business logic** — only for debugging. Its execution is not guaranteed by the JVM in all scenarios (e.g., short-circuiting may skip elements).

### ⚡ Performance Tip
**Order your operations for maximum short-circuiting**: Place `filter()` before `map()` and `sorted()` to reduce the number of elements processed by expensive operations. Use `mapToInt/Long/Double` to avoid autoboxing in numeric pipelines. Avoid `sorted()` on large streams unless necessary — it's a stateful operation that must buffer all elements (O(n log n) time and O(n) space). Prefer `parallel()` streams only for CPU-intensive operations on large datasets (>10,000 elements); the overhead of parallel setup outweighs gains for small collections or I/O-bound operations.

### 🔧 JVM Internals
The Stream API uses **loop fusion** internally — multiple intermediate operations are merged into a single pass over the data rather than creating intermediate collections. The `Spliterator` interface underlies the splitting and traversal logic, enabling parallel decomposition. Stateful operations (`sorted`, `distinct`) introduce **pipeline barriers** that force buffering of upstream elements. The JVM can also optimize `stream().count()` on `ArrayList` to return `.size()` directly, bypassing the entire pipeline when no intermediate operations are present.

---

## 🎯 Interviewer Insight
Interviewers love to test your understanding of **lazy evaluation** and **short-circuiting**. A strong candidate can trace through a stream pipeline and predict exactly which elements are processed and in what order. Knowing the difference between stateless (`filter`, `map`) and stateful (`sorted`, `distinct`) operations is crucial for performance discussions. Be ready to explain why `parallelStream()` isn't always faster.

## ⚡ Follow-up Questions
1. What happens if you call two terminal operations on the same stream, and how do you reuse stream logic?
2. When is `parallelStream()` beneficial, and when is it harmful? What are the overhead considerations?
3. How does `flatMap` differ from `map` in practical usage, and when would you choose one over the other?
4. What is the difference between stateless and stateful stream operations, and how do they affect performance?
5. How does the JVM perform loop fusion optimization on stream pipelines, and what operations prevent it?
6. What is the difference between `findFirst()` and `findAny()` in sequential vs parallel streams?
7. How do primitive streams (IntStream, LongStream, DoubleStream) improve performance by avoiding boxing?

## 📚 Related Topics
- [Stream Collectors](./q05-stream-collectors.md)
- [Functional Interfaces](./q03-functional-interfaces-detail.md)
- [Optional API](./q06-optional-api.md)

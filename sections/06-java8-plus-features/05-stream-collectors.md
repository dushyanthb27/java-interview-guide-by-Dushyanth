# Q5: How Do Stream Collectors Work?

## 📋 Question
How do Stream Collectors work in Java? Explain the built-in collectors, downstream collectors, groupingBy/partitioningBy, and how to implement a custom Collector.

## Difficulty: 🟠 Senior

---

## 💡 Answer

`Collectors` is a utility class providing **ready-made reductions** for Stream's `collect()` terminal operation. A Collector defines four operations: **supplier** (creates accumulator), **accumulator** (adds element), **combiner** (merges two accumulators for parallel), and **finisher** (produces final result).

### Basic Collectors

```java
package com.interview.java8;

import java.util.*;
import java.util.stream.*;

public class BasicCollectorsDemo {

    public static void main(String[] args) {
        List<String> names = List.of("Alice", "Bob", "Charlie", "Alice", "David");

        // ═══════════════════════════════════════════════════
        // toList(), toSet(), toCollection()
        // ═══════════════════════════════════════════════════
        List<String> list = names.stream().collect(Collectors.toList());       // Mutable
        Set<String> set = names.stream().collect(Collectors.toSet());          // No dupes
        TreeSet<String> sorted = names.stream()
                .collect(Collectors.toCollection(TreeSet::new));               // Sorted set

        // ✅ Java 10+: toUnmodifiableList/Set — returns immutable collection
        List<String> immutable = names.stream()
                .collect(Collectors.toUnmodifiableList());
        // immutable.add("Eve");  // ❌ UnsupportedOperationException

        // ✅ Java 16+: Stream.toList() shorthand (returns unmodifiable list)
        List<String> shorthand = names.stream().filter(n -> n.length() > 3).toList();

        // ═══════════════════════════════════════════════════
        // toMap() — collects into a Map
        // ═══════════════════════════════════════════════════
        Map<String, Integer> nameLengths = names.stream()
                .distinct()
                .collect(Collectors.toMap(
                        name -> name,              // Key mapper
                        String::length             // Value mapper
                ));
        System.out.println(nameLengths);  // {Alice=5, Bob=3, Charlie=7, David=5}

        // ⚠️ toMap throws on duplicate keys — use merge function
        Map<String, Integer> withDupes = names.stream()
                .collect(Collectors.toMap(
                        name -> name,
                        String::length,
                        (existing, replacement) -> existing  // Keep first on conflict
                ));

        // Specific map type
        LinkedHashMap<String, Integer> orderedMap = names.stream()
                .distinct()
                .collect(Collectors.toMap(
                        name -> name, String::length,
                        (a, b) -> a, LinkedHashMap::new
                ));

        // ═══════════════════════════════════════════════════
        // joining() — concatenates strings
        // ═══════════════════════════════════════════════════
        String joined = names.stream()
                .collect(Collectors.joining(", ", "[", "]"));
        System.out.println(joined);  // [Alice, Bob, Charlie, Alice, David]
    }
}
```

### groupingBy and partitioningBy

```java
package com.interview.java8;

import java.util.*;
import java.util.stream.*;

public class GroupingCollectorsDemo {

    record Employee(String name, String dept, double salary) {}

    public static void main(String[] args) {
        List<Employee> employees = List.of(
            new Employee("Alice", "Engineering", 95000),
            new Employee("Bob", "Marketing", 85000),
            new Employee("Charlie", "Engineering", 110000),
            new Employee("Diana", "Marketing", 92000),
            new Employee("Eve", "Engineering", 88000),
            new Employee("Frank", "HR", 78000)
        );

        // ═══════════════════════════════════════════════════
        // Simple groupingBy — groups into Map<K, List<T>>
        // ═══════════════════════════════════════════════════
        Map<String, List<Employee>> byDept = employees.stream()
                .collect(Collectors.groupingBy(Employee::dept));
        // {Engineering=[Alice, Charlie, Eve], Marketing=[Bob, Diana], HR=[Frank]}

        // ═══════════════════════════════════════════════════
        // groupingBy with downstream collector
        // ═══════════════════════════════════════════════════
        // Count per department
        Map<String, Long> countByDept = employees.stream()
                .collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));
        System.out.println(countByDept);  // {Engineering=3, Marketing=2, HR=1}

        // Average salary per department
        Map<String, Double> avgSalary = employees.stream()
                .collect(Collectors.groupingBy(Employee::dept,
                        Collectors.averagingDouble(Employee::salary)));
        System.out.println(avgSalary);  // {Engineering=97666.67, Marketing=88500.0, HR=78000.0}

        // Sum salary per department
        Map<String, Double> totalSalary = employees.stream()
                .collect(Collectors.groupingBy(Employee::dept,
                        Collectors.summingDouble(Employee::salary)));

        // Max salary employee per department
        Map<String, Optional<Employee>> topEarner = employees.stream()
                .collect(Collectors.groupingBy(Employee::dept,
                        Collectors.maxBy(Comparator.comparingDouble(Employee::salary))));

        // Names per department (mapping downstream)
        Map<String, List<String>> namesByDept = employees.stream()
                .collect(Collectors.groupingBy(Employee::dept,
                        Collectors.mapping(Employee::name, Collectors.toList())));
        System.out.println(namesByDept);
        // {Engineering=[Alice, Charlie, Eve], Marketing=[Bob, Diana], HR=[Frank]}

        // ═══════════════════════════════════════════════════
        // Multi-level grouping
        // ═══════════════════════════════════════════════════
        Map<String, Map<String, List<Employee>>> multilevel = employees.stream()
                .collect(Collectors.groupingBy(Employee::dept,
                        Collectors.groupingBy(e -> e.salary() > 90000 ? "High" : "Low")));

        // ═══════════════════════════════════════════════════
        // partitioningBy — splits into true/false groups
        // ═══════════════════════════════════════════════════
        Map<Boolean, List<Employee>> partitioned = employees.stream()
                .collect(Collectors.partitioningBy(e -> e.salary() > 90000));
        System.out.println("High earners: " + partitioned.get(true).size());   // 3
        System.out.println("Lower earners: " + partitioned.get(false).size()); // 3

        // partitioningBy with downstream
        Map<Boolean, Long> partitionCount = employees.stream()
                .collect(Collectors.partitioningBy(
                        e -> e.salary() > 90000, Collectors.counting()));
    }
}
```

### Advanced Collectors

```java
package com.interview.java8;

import java.util.*;
import java.util.stream.*;

public class AdvancedCollectorsDemo {

    record Employee(String name, String dept, double salary) {}

    public static void main(String[] args) {
        List<Employee> employees = List.of(
            new Employee("Alice", "Engineering", 95000),
            new Employee("Bob", "Marketing", 85000),
            new Employee("Charlie", "Engineering", 110000)
        );

        // ═══════════════════════════════════════════════════
        // collectingAndThen — apply finisher to collector result
        // ═══════════════════════════════════════════════════
        List<String> unmodifiable = employees.stream()
                .map(Employee::name)
                .collect(Collectors.collectingAndThen(
                        Collectors.toList(),
                        Collections::unmodifiableList   // Finisher
                ));

        // Count as int instead of long
        int count = employees.stream()
                .collect(Collectors.collectingAndThen(Collectors.counting(), Long::intValue));

        // ═══════════════════════════════════════════════════
        // reducing — general-purpose reduction as collector
        // ═══════════════════════════════════════════════════
        Optional<Employee> highestPaid = employees.stream()
                .collect(Collectors.reducing(
                        BinaryOperator.maxBy(Comparator.comparingDouble(Employee::salary))
                ));

        // With identity and mapping
        double totalSalary = employees.stream()
                .collect(Collectors.reducing(0.0, Employee::salary, Double::sum));

        // ═══════════════════════════════════════════════════
        // summarizingInt/Long/Double — statistics in one pass
        // ═══════════════════════════════════════════════════
        DoubleSummaryStatistics stats = employees.stream()
                .collect(Collectors.summarizingDouble(Employee::salary));
        System.out.println("Count: " + stats.getCount());
        System.out.println("Sum: " + stats.getSum());
        System.out.println("Min: " + stats.getMin());
        System.out.println("Max: " + stats.getMax());
        System.out.println("Avg: " + stats.getAverage());

        // ═══════════════════════════════════════════════════
        // teeing (Java 12) — process stream with two collectors
        // ═══════════════════════════════════════════════════
        var result = employees.stream()
                .collect(Collectors.teeing(
                        Collectors.minBy(Comparator.comparingDouble(Employee::salary)),
                        Collectors.maxBy(Comparator.comparingDouble(Employee::salary)),
                        (min, max) -> "Range: " + min.map(Employee::name).orElse("?")
                                + " to " + max.map(Employee::name).orElse("?")
                ));
        System.out.println(result);  // Range: Bob to Charlie
    }
}
```

### Custom Collector Implementation

```java
package com.interview.java8;

import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class CustomCollectorDemo {

    // ✅ Custom collector that collects elements into a comma-separated string
    // with a maximum length, appending "..." if truncated
    static Collector<String, StringBuilder, String> truncatedJoining(
            String delimiter, int maxLength) {

        return new Collector<>() {
            @Override
            public Supplier<StringBuilder> supplier() {
                return StringBuilder::new;                     // Create accumulator
            }

            @Override
            public BiConsumer<StringBuilder, String> accumulator() {
                return (sb, s) -> {
                    if (sb.length() > 0) sb.append(delimiter); // Add delimiter
                    sb.append(s);                               // Add element
                };
            }

            @Override
            public BinaryOperator<StringBuilder> combiner() {
                return (sb1, sb2) -> {
                    if (sb1.length() > 0 && sb2.length() > 0)
                        sb1.append(delimiter);
                    return sb1.append(sb2);                    // Merge for parallel
                };
            }

            @Override
            public Function<StringBuilder, String> finisher() {
                return sb -> {
                    String result = sb.toString();
                    if (result.length() > maxLength) {
                        return result.substring(0, maxLength - 3) + "...";  // Truncate
                    }
                    return result;
                };
            }

            @Override
            public Set<Characteristics> characteristics() {
                return Set.of();  // No special characteristics
                // CONCURRENT — accumulator is thread-safe
                // UNORDERED — result doesn't depend on encounter order
                // IDENTITY_FINISH — finisher is identity (cast only)
            }
        };
    }

    public static void main(String[] args) {
        String result = List.of("Java", "Python", "JavaScript", "Go", "Rust")
                .stream()
                .collect(truncatedJoining(", ", 25));
        System.out.println(result);  // Java, Python, JavaScr...
    }
}
```

### Collector Characteristics

| Characteristic | Meaning | Effect |
|---------------|---------|--------|
| `CONCURRENT` | Accumulator is thread-safe | Enables concurrent accumulation without combiner |
| `UNORDERED` | Result doesn't depend on element order | Enables optimizations in parallel streams |
| `IDENTITY_FINISH` | Finisher is identity function | Skips finisher step (cast accumulator to result) |

### ⚠️ Common Mistake
**Using `toMap()` without a merge function when duplicates exist**: `Collectors.toMap(keyMapper, valueMapper)` throws `IllegalStateException` on duplicate keys. Always provide a merge function `(v1, v2) -> ...` when key collisions are possible. Similarly, **`groupingBy` cannot have null keys** — if your classifier function returns null, use a wrapper or filter nulls before grouping.

### ⚡ Performance Tip
Use `Collectors.toUnmodifiableList()` or `Stream.toList()` (Java 16+) when you don't need to modify the resulting list — unmodifiable collections use less memory and enable internal optimizations. For large-scale aggregations, `summarizingDouble()` computes count, sum, min, max, and average in a **single pass** rather than requiring five separate terminal operations. When using `groupingBy` with `Collectors.toSet()` as a downstream, the resulting sets are `HashSet`s by default — use `toCollection(TreeSet::new)` if you need sorted results.

### 🔧 JVM Internals
The `Collector` interface is the bridge between the Stream API's `collect()` method and the reduction logic. For parallel streams, each thread gets its own accumulator from the supplier, fills it independently, and the combiner merges them. The `IDENTITY_FINISH` characteristic allows the JVM to skip the finisher step entirely — the accumulator itself is cast to the result type. This is why `Collectors.toList()` has `IDENTITY_FINISH` — the internal `ArrayList` accumulator is the final result, requiring no transformation.

---

## 🎯 Interviewer Insight
Interviewers expect you to fluently use `groupingBy` with downstream collectors — it's the most common real-world scenario. A strong answer chains downstream collectors (e.g., `groupingBy(dept, mapping(name, toList()))`). Being able to implement a custom `Collector` with all four functions demonstrates mastery. Know the `teeing` collector (Java 12) — it impresses interviewers and solves dual-aggregation problems elegantly.

## ⚡ Follow-up Questions
1. What's the difference between `reduce()` and `collect()`?
2. How does `groupingByConcurrent` differ from `groupingBy`?
3. When would you implement a custom Collector vs using built-in ones?

## 📚 Related Topics
- [Stream API Operations](./q04-stream-api-operations.md)
- [Functional Interfaces](./q03-functional-interfaces-detail.md)
- [Optional API](./q06-optional-api.md)

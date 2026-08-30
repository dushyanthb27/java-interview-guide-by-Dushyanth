# Stream API - Cheat Sheet

Complete reference for Java Streams with operations, collectors, and performance considerations.

## 🌊 Stream Basics

### Creating Streams
```java
// From collection
List<String> list = Arrays.asList("a", "b", "c");
Stream<String> stream = list.stream();
Stream<String> parallel = list.parallelStream();

// From array
String[] array = {"a", "b", "c"};
Stream<String> stream = Arrays.stream(array);

// From values
Stream<String> stream = Stream.of("a", "b", "c");

// Empty stream
Stream<String> empty = Stream.empty();

// Infinite streams
Stream<Integer> infinite = Stream.iterate(0, n -> n + 1);
Stream<Integer> infinite2 = Stream.generate(() -> new Random().nextInt());

// From file
Stream<String> lines = Files.lines(Paths.get("file.txt"));

// From range
IntStream range = IntStream.range(0, 10);        // 0-9
IntStream rangeClosed = IntStream.rangeClosed(0, 10);  // 0-10
```

## 🔄 Intermediate Operations

### Filter
```java
list.stream()
    .filter(s -> s.length() > 2)
    .forEach(System.out::println);

// Performance: O(n), lazy evaluation
```

### Map
```java
// Transform elements
list.stream()
    .map(String::toUpperCase)
    .forEach(System.out::println);

// Map to int/long/double
list.stream()
    .mapToInt(String::length)
    .sum();

// Flat map (flatten nested streams)
list.stream()
    .flatMap(s -> Stream.of(s.split("")))
    .forEach(System.out::println);

// Performance: O(n), lazy evaluation
```

### Distinct
```java
list.stream()
    .distinct()
    .forEach(System.out::println);

// Performance: O(n) with O(n) space for tracking
```

### Sorted
```java
// Natural order
list.stream()
    .sorted()
    .forEach(System.out::println);

// Custom comparator
list.stream()
    .sorted(Comparator.reverseOrder())
    .forEach(System.out::println);

// Sort by property
people.stream()
    .sorted(Comparator.comparing(Person::getAge))
    .forEach(System.out::println);

// Multiple sort criteria
people.stream()
    .sorted(Comparator.comparing(Person::getAge)
                      .thenComparing(Person::getName))
    .forEach(System.out::println);

// Performance: O(n log n), eager operation
```

### Limit & Skip
```java
// Take first n elements
list.stream()
    .limit(5)
    .forEach(System.out::println);

// Skip first n elements
list.stream()
    .skip(5)
    .forEach(System.out::println);

// Pagination
list.stream()
    .skip(pageNumber * pageSize)
    .limit(pageSize)
    .forEach(System.out::println);

// Performance: O(n), lazy evaluation
```

### Peek (Debugging)
```java
list.stream()
    .peek(s -> System.out.println("Before: " + s))
    .filter(s -> s.length() > 2)
    .peek(s -> System.out.println("After: " + s))
    .forEach(System.out::println);

// Performance: O(n), lazy evaluation
// Use case: Debugging only, not for side effects
```

## 🎯 Terminal Operations

### ForEach
```java
list.stream()
    .forEach(System.out::println);

// Ordered version (sequential only)
list.stream()
    .forEachOrdered(System.out::println);
```

### Collect
```java
// To list
List<String> result = list.stream()
    .filter(s -> s.length() > 2)
    .collect(Collectors.toList());

// To set
Set<String> result = list.stream()
    .collect(Collectors.toSet());

// To map
Map<String, Integer> result = list.stream()
    .collect(Collectors.toMap(
        Function.identity(),
        String::length
    ));

// To custom collection
TreeSet<String> result = list.stream()
    .collect(Collectors.toCollection(TreeSet::new));

// Performance: Terminal operation, O(n)
```

### Reduce
```java
// Sum
int sum = numbers.stream()
    .reduce(0, Integer::sum);

// Product
int product = numbers.stream()
    .reduce(1, (a, b) -> a * b);

// Max
Optional<Integer> max = numbers.stream()
    .reduce(Integer::max);

// Concatenate strings
String result = list.stream()
    .reduce("", String::concat);

// Performance: O(n), terminal operation
```

### Count, Min, Max
```java
long count = list.stream()
    .count();

Optional<String> max = list.stream()
    .max(Comparator.naturalOrder());

Optional<String> min = list.stream()
    .min(Comparator.naturalOrder());

// For numeric streams
int sum = IntStream.range(0, 10)
    .sum();

double average = IntStream.range(0, 10)
    .average()
    .orElse(0);

IntSummaryStatistics stats = IntStream.range(0, 10)
    .summaryStatistics();
// stats.getCount(), getSum(), getAverage(), getMin(), getMax()
```

### Match Operations
```java
// All match
boolean allPositive = numbers.stream()
    .allMatch(n -> n > 0);

// Any match
boolean anyNegative = numbers.stream()
    .anyMatch(n -> n < 0);

// None match
boolean noNegative = numbers.stream()
    .noneMatch(n -> n < 0);

// Performance: O(n) worst case, may short-circuit
```

### Find Operations
```java
// Find any (unordered)
Optional<String> any = list.stream()
    .filter(s -> s.length() > 2)
    .findAny();

// Find first (ordered)
Optional<String> first = list.stream()
    .filter(s -> s.length() > 2)
    .findFirst();

// Performance: O(1) best case, may short-circuit
```

## 📦 Collectors

### Grouping
```java
// Group by single property
Map<Integer, List<String>> byLength = list.stream()
    .collect(Collectors.groupingBy(String::length));

// Group by with custom collector
Map<Integer, Set<String>> byLength = list.stream()
    .collect(Collectors.groupingBy(
        String::length,
        Collectors.toSet()
    ));

// Group by with counting
Map<Integer, Long> countByLength = list.stream()
    .collect(Collectors.groupingBy(
        String::length,
        Collectors.counting()
    ));

// Nested grouping
Map<Integer, Map<String, List<String>>> nested = list.stream()
    .collect(Collectors.groupingBy(
        String::length,
        Collectors.groupingBy(s -> s.substring(0, 1))
    ));

// Performance: O(n), terminal operation
```

### Partitioning
```java
// Partition by predicate
Map<Boolean, List<String>> partitioned = list.stream()
    .collect(Collectors.partitioningBy(
        s -> s.length() > 2
    ));
// {true: [items with length > 2], false: [items with length <= 2]}

// Partition with custom collector
Map<Boolean, Set<String>> partitioned = list.stream()
    .collect(Collectors.partitioningBy(
        s -> s.length() > 2,
        Collectors.toSet()
    ));

// Performance: O(n), terminal operation
```

### Joining
```java
// Join strings
String result = list.stream()
    .collect(Collectors.joining());  // "abc"

// With delimiter
String result = list.stream()
    .collect(Collectors.joining(", "));  // "a, b, c"

// With prefix and suffix
String result = list.stream()
    .collect(Collectors.joining(", ", "[", "]"));  // "[a, b, c]"

// Performance: O(n), terminal operation
```

### Mapping
```java
// Map then collect
Set<Integer> lengths = list.stream()
    .collect(Collectors.mapping(
        String::length,
        Collectors.toSet()
    ));

// Map then join
String result = list.stream()
    .collect(Collectors.mapping(
        String::toUpperCase,
        Collectors.joining(", ")
    ));

// Performance: O(n), terminal operation
```

### Custom Collector
```java
// Collect to custom collection
TreeSet<String> result = list.stream()
    .collect(Collectors.toCollection(TreeSet::new));

// Using Collector.of()
Collector<String, ?, String> customCollector = Collector.of(
    StringBuilder::new,
    (sb, s) -> sb.append(s).append(","),
    (sb1, sb2) -> sb1.append(sb2),
    StringBuilder::toString
);

String result = list.stream()
    .collect(customCollector);
```

## 📊 Numeric Streams

### IntStream, LongStream, DoubleStream
```java
// Create
IntStream stream = IntStream.range(0, 10);
IntStream stream2 = IntStream.rangeClosed(0, 10);
IntStream stream3 = IntStream.of(1, 2, 3, 4, 5);

// Operations
int sum = IntStream.range(0, 10).sum();
double average = IntStream.range(0, 10).average().orElse(0);
int max = IntStream.range(0, 10).max().orElse(0);
int min = IntStream.range(0, 10).min().orElse(0);

// Box to Stream<Integer>
Stream<Integer> boxed = IntStream.range(0, 10).boxed();

// Performance: More efficient than Stream<Integer>
```

## ⚡ Performance Considerations

### Lazy vs Eager
```java
// Lazy evaluation - nothing happens until terminal operation
list.stream()
    .filter(s -> s.length() > 2)
    .map(String::toUpperCase);  // No work done yet

// Terminal operation triggers evaluation
list.stream()
    .filter(s -> s.length() > 2)
    .map(String::toUpperCase)
    .forEach(System.out::println);  // Now work is done
```

### Parallel Streams
```java
// Sequential (default)
list.stream()
    .filter(s -> s.length() > 2)
    .forEach(System.out::println);

// Parallel
list.parallelStream()
    .filter(s -> s.length() > 2)
    .forEach(System.out::println);

// Or convert
list.stream()
    .parallel()
    .filter(s -> s.length() > 2)
    .forEach(System.out::println);

// Performance: Use for large datasets (>1000 elements)
// Overhead for small datasets
// Avoid side effects in parallel streams
```

### Stateless vs Stateful
```java
// Stateless - each element independent
stream.filter(x -> x > 0)
      .map(x -> x * 2);

// Stateful - depends on previous elements
stream.distinct()
      .sorted()
      .limit(10);

// Performance: Stateless operations are more efficient
```

## 🔄 Common Patterns

### Filter and Map
```java
List<Integer> lengths = list.stream()
    .filter(s -> s.length() > 2)
    .map(String::length)
    .collect(Collectors.toList());
```

### Group and Count
```java
Map<Integer, Long> counts = list.stream()
    .collect(Collectors.groupingBy(
        String::length,
        Collectors.counting()
    ));
```

### Find First Match
```java
Optional<String> result = list.stream()
    .filter(s -> s.startsWith("a"))
    .findFirst();
```

### Reduce to Single Value
```java
int sum = numbers.stream()
    .reduce(0, Integer::sum);

String concatenated = list.stream()
    .reduce("", String::concat);
```

### Nested Iteration
```java
List<List<Integer>> matrix = Arrays.asList(
    Arrays.asList(1, 2),
    Arrays.asList(3, 4)
);

List<Integer> flattened = matrix.stream()
    .flatMap(List::stream)
    .collect(Collectors.toList());
```

## ⚠️ Common Pitfalls

1. **Consuming stream twice**
   ```java
   // ❌ Wrong - stream already consumed
   Stream<String> stream = list.stream();
   stream.forEach(System.out::println);
   stream.forEach(System.out::println);  // IllegalStateException
   
   // ✅ Correct
   list.stream().forEach(System.out::println);
   list.stream().forEach(System.out::println);
   ```

2. **Side effects in streams**
   ```java
   // ❌ Wrong - side effects in parallel stream
   list.parallelStream()
       .forEach(s -> sharedList.add(s));  // Race condition
   
   // ✅ Correct
   List<String> result = list.parallelStream()
       .collect(Collectors.toList());
   ```

3. **Forgetting terminal operation**
   ```java
   // ❌ Wrong - nothing happens
   list.stream()
       .filter(s -> s.length() > 2)
       .map(String::toUpperCase);
   
   // ✅ Correct
   list.stream()
       .filter(s -> s.length() > 2)
       .map(String::toUpperCase)
       .collect(Collectors.toList());
   ```

4. **Expensive operations in parallel**
   ```java
   // ❌ Wrong - sorting negates parallelism benefit
   list.parallelStream()
       .sorted()
       .collect(Collectors.toList());
   
   // ✅ Correct - sort after collecting
   list.parallelStream()
       .collect(Collectors.toList())
       .sort(Comparator.naturalOrder());
   ```

## 🎓 Interview Tips

- **Know the difference**: Intermediate vs terminal operations
- **Understand lazy evaluation**: Streams don't execute until terminal operation
- **Use appropriate collectors**: groupingBy, partitioningBy, joining
- **Avoid side effects**: Keep streams functional
- **Consider performance**: Parallel streams for large datasets only
- **Know the API**: map, filter, reduce, collect are most common

---

**Related**: See Java 8+ Features cheat sheet for lambda expressions and functional interfaces

# Q15: Streams API with Collections

[Back to Collections Framework](./README.md) | **Difficulty**: 🟡 Medium

---

## Question
> How does the Streams API work with Collections? Explain key intermediate and terminal operations.

---

## Answer

### Creating Streams from Collections

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "David", "Eve");

// From Collection
Stream<String> stream = names.stream();
Stream<String> parallelStream = names.parallelStream();

// From arrays
Stream<String> arrayStream = Arrays.stream(new String[]{"A", "B", "C"});

// From values
Stream<String> ofStream = Stream.of("A", "B", "C");

// From Map
Map<String, Integer> scores = Map.of("Alice", 85, "Bob", 92);
Stream<Map.Entry<String, Integer>> entryStream = scores.entrySet().stream();
Stream<String> keyStream = scores.keySet().stream();
Stream<Integer> valueStream = scores.values().stream();

// Other sources
Stream<Integer> range = IntStream.rangeClosed(1, 10).boxed();
Stream<String> lines = Files.lines(Path.of("file.txt"));  // Auto-closeable
Stream<String> generated = Stream.generate(() -> "hello").limit(5);
Stream<Integer> iterated = Stream.iterate(0, n -> n + 2).limit(10); // 0,2,4,...,18
```

### Intermediate Operations (Lazy - Return Stream)

```java
List<Employee> employees = getEmployees();

// ✅ filter - keep elements matching predicate
List<Employee> seniors = employees.stream()
    .filter(e -> e.getAge() > 30)
    .collect(Collectors.toList());

// ✅ map - transform each element
List<String> names = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toList());

// ✅ flatMap - flatten nested collections
List<List<String>> nested = List.of(List.of("A", "B"), List.of("C", "D"));
List<String> flat = nested.stream()
    .flatMap(Collection::stream)
    .collect(Collectors.toList());
// [A, B, C, D]

// ✅ distinct - remove duplicates (uses equals/hashCode)
List<String> unique = List.of("A", "B", "A", "C", "B").stream()
    .distinct()
    .collect(Collectors.toList());
// [A, B, C]

// ✅ sorted - sort elements
List<String> sorted = names.stream()
    .sorted()                                    // Natural order
    .sorted(Comparator.reverseOrder())           // Reverse order
    .sorted(Comparator.comparingInt(String::length))  // By length
    .collect(Collectors.toList());

// ✅ peek - inspect elements (for debugging, not side effects)
List<String> result = names.stream()
    .peek(n -> System.out.println("Before filter: " + n))
    .filter(n -> n.length() > 3)
    .peek(n -> System.out.println("After filter: " + n))
    .collect(Collectors.toList());

// ✅ limit and skip
List<String> page = employees.stream()
    .skip(10)       // Skip first 10
    .limit(5)       // Take next 5
    .map(Employee::getName)
    .collect(Collectors.toList());

// ✅ mapToInt/mapToLong/mapToDouble - avoid boxing
IntStream ages = employees.stream().mapToInt(Employee::getAge);
double avgAge = employees.stream().mapToInt(Employee::getAge).average().orElse(0);
```

### Terminal Operations (Eager - Trigger Processing)

```java
List<Employee> employees = getEmployees();

// ✅ collect - accumulate into a collection
List<String> names = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toList());

// ✅ forEach - perform action on each element
employees.stream()
    .filter(e -> e.isActive())
    .forEach(e -> sendEmail(e));

// ✅ reduce - combine all elements into one
int totalSalary = employees.stream()
    .mapToInt(Employee::getSalary)
    .reduce(0, Integer::sum);

Optional<String> concatenated = names.stream()
    .reduce((a, b) -> a + ", " + b);

// ✅ count
long activeCount = employees.stream()
    .filter(Employee::isActive)
    .count();

// ✅ findFirst / findAny
Optional<Employee> first = employees.stream()
    .filter(e -> e.getDepartment().equals("Engineering"))
    .findFirst();

Optional<Employee> any = employees.parallelStream()
    .filter(e -> e.getSalary() > 100000)
    .findAny();  // Non-deterministic in parallel

// ✅ anyMatch / allMatch / noneMatch
boolean hasHighEarner = employees.stream()
    .anyMatch(e -> e.getSalary() > 200000);

boolean allActive = employees.stream()
    .allMatch(Employee::isActive);

boolean noneRetired = employees.stream()
    .noneMatch(e -> e.getAge() > 65);

// ✅ min / max
Optional<Employee> youngest = employees.stream()
    .min(Comparator.comparingInt(Employee::getAge));

Optional<Employee> highestPaid = employees.stream()
    .max(Comparator.comparingDouble(Employee::getSalary));

// ✅ toArray
String[] nameArray = employees.stream()
    .map(Employee::getName)
    .toArray(String[]::new);
```

### Key Collectors

```java
List<Employee> employees = getEmployees();

// ✅ toList / toSet / toMap
List<String> nameList = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toList());

Set<String> departments = employees.stream()
    .map(Employee::getDepartment)
    .collect(Collectors.toSet());

Map<Integer, String> idToName = employees.stream()
    .collect(Collectors.toMap(Employee::getId, Employee::getName));

// Handle duplicate keys:
Map<String, Employee> byName = employees.stream()
    .collect(Collectors.toMap(
        Employee::getName,
        Function.identity(),
        (existing, replacement) -> existing  // Keep first on conflict
    ));

// ✅ groupingBy - group elements
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// Downstream collectors:
Map<String, Long> deptCount = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment, Collectors.counting()));

Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));

Map<String, Optional<Employee>> topEarnerByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary))
    ));

// ✅ partitioningBy - split into two groups (true/false)
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 50000));
List<Employee> highEarners = partitioned.get(true);
List<Employee> others = partitioned.get(false);

// ✅ joining - concatenate strings
String csv = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", "));
// "Alice, Bob, Charlie"

String csvWithBrackets = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", ", "[", "]"));
// "[Alice, Bob, Charlie]"

// ✅ summarizingInt/Long/Double
IntSummaryStatistics stats = employees.stream()
    .collect(Collectors.summarizingInt(Employee::getAge));
stats.getCount();     // number of employees
stats.getSum();       // sum of ages
stats.getMin();       // min age
stats.getMax();       // max age
stats.getAverage();   // average age

// ✅ toUnmodifiableList/Set/Map (Java 10+)
List<String> immutable = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toUnmodifiableList());
```

### Parallel Streams

```java
// Parallel stream for CPU-intensive operations on large datasets
List<Integer> numbers = IntStream.rangeClosed(1, 1_000_000)
    .boxed()
    .collect(Collectors.toList());

// Sequential
long seqSum = numbers.stream()
    .filter(n -> n % 2 == 0)
    .mapToLong(Integer::longValue)
    .sum();

// Parallel
long parSum = numbers.parallelStream()
    .filter(n -> n % 2 == 0)
    .mapToLong(Integer::longValue)
    .sum();

// ⚠️ When NOT to use parallel streams:
// - Small collections (overhead > benefit)
// - I/O operations (threads block, no benefit)
// - Ordered operations (ordering overhead)
// - Shared mutable state (race conditions)
// - LinkedList (poor splitting performance)

// ⚠️ Use thread-safe collectors with parallel streams
List<String> result = employees.parallelStream()
    .map(Employee::getName)
    .collect(Collectors.toList());  // ✅ toList() is safe for parallel
```

### Common Anti-Patterns

```java
// ❌ Using streams for simple iteration
employees.stream().forEach(System.out::println);
// ✅ Use direct forEach
employees.forEach(System.out::println);

// ❌ Side effects in stream operations
List<String> results = new ArrayList<>();
employees.stream()
    .map(Employee::getName)
    .forEach(results::add);  // ❌ Side effect! Not safe with parallel
// ✅ Use collect instead
List<String> results = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toList());

// ❌ Reusing a stream (streams are single-use!)
Stream<String> stream = names.stream();
stream.forEach(System.out::println);
stream.count();  // ❌ IllegalStateException: stream already operated upon

// ❌ Using peek for business logic (it's for debugging)
employees.stream()
    .peek(e -> e.setSalary(e.getSalary() * 1.1))  // ❌ Side effect in peek!
    .collect(Collectors.toList());

// ❌ Nested streams where flatMap would work
// ❌ orders.stream().map(o -> o.getItems().stream()) -> Stream<Stream<Item>>
// ✅ orders.stream().flatMap(o -> o.getItems().stream()) -> Stream<Item>
```

### Best Practices

```java
// ✅ Use method references when possible
employees.stream().map(Employee::getName)   // Cleaner than e -> e.getName()

// ✅ Use primitive streams to avoid boxing
employees.stream().mapToInt(Employee::getAge).sum()   // No boxing

// ✅ Use Collectors.toList() over manual accumulation
// ✅ Use collect(Collectors.toUnmodifiableList()) for immutable results (Java 10+)

// ✅ Short-circuit when possible
employees.stream().anyMatch(e -> e.getSalary() > 200000)  // Stops at first match

// ✅ Order operations to minimize work
employees.stream()
    .filter(e -> e.isActive())           // Filter FIRST (reduces elements)
    .filter(e -> e.getSalary() > 50000)  // Then more specific filter
    .map(Employee::getName)              // Map AFTER filtering
    .sorted()                            // Sort LAST (fewer elements)
    .collect(Collectors.toList());

// ✅ Close streams backed by I/O resources
try (Stream<String> lines = Files.lines(Path.of("data.txt"))) {
    lines.filter(line -> !line.isEmpty()).forEach(System.out::println);
}
```

---

## Interview Tips
- Streams are lazy (intermediate ops) and eager (terminal ops)
- Streams are single-use - cannot be reused after terminal operation
- Know the difference between map and flatMap
- Parallel streams use ForkJoinPool.commonPool() by default
- Order of operations matters for performance (filter before map)
- Avoid side effects in stream operations
- Know key Collectors: toList, groupingBy, partitioningBy, joining

---

[Prev: Q14 - Collections Utility](./q14-collections-utility.md) | [Next: Q16 - Optional Handling](./q16-optional-handling.md)

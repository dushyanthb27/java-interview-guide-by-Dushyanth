# Q16: When Should You Use Parallel Streams?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
When should you use parallel streams? What are the performance pitfalls and how does it work under the hood?

---

## Answer

### How Parallel Streams Work

```
Sequential Stream:
  [1, 2, 3, 4, 5, 6, 7, 8] ──► Single thread processes all elements

Parallel Stream:
  [1, 2, 3, 4, 5, 6, 7, 8]
       │         │
  ┌────▼────┐  ┌─▼─────────┐
  │[1,2,3,4]│  │[5, 6, 7, 8]│    Split (ForkJoinPool)
  └────┬────┘  └──┬─────────┘
  ┌────▼────┐  ┌──▼─────────┐
  │[1,2][3,4]│ │[5,6] [7,8] │    Split more
  └─┬──┬────┘ └─┬───┬──────┘
    ▼  ▼       ▼   ▼
  Process in parallel across cores    Process
    │  │       │   │
    ▼  ▼       ▼   ▼
  Combine ◄─────── Combine           Merge results
```

Parallel streams use the **ForkJoinPool.commonPool()** by default.

### Basic Usage

```java
public class ParallelStreamBasics {
    public static void main(String[] args) {
        List<Integer> numbers = IntStream.rangeClosed(1, 1_000_000)
            .boxed()
            .collect(Collectors.toList());

        // Sequential
        long seqSum = numbers.stream()
            .mapToLong(Integer::longValue)
            .sum();

        // Parallel
        long parSum = numbers.parallelStream()
            .mapToLong(Integer::longValue)
            .sum();

        // Convert sequential to parallel
        long parSum2 = numbers.stream()
            .parallel()
            .mapToLong(Integer::longValue)
            .sum();

        // Check if parallel
        boolean isParallel = numbers.parallelStream().isParallel(); // true
    }
}
```

### The NQ Model: When to Go Parallel

```
NQ Model: Use parallel streams when N * Q > 10,000

N = number of elements
Q = cost per element (operations)

┌──────────────┬─────────────┬──────────────────────┐
│ N (elements) │ Q (cost)    │ Parallel?            │
├──────────────┼─────────────┼──────────────────────┤
│ 10           │ 10          │ ❌ No (100)          │
│ 100          │ 10          │ ❌ No (1,000)        │
│ 10,000       │ 1           │ ⚠️ Maybe (10,000)    │
│ 100,000      │ 1           │ ✅ Yes (100,000)     │
│ 1,000        │ 100         │ ✅ Yes (100,000)     │
│ 100          │ 10,000      │ ✅ Yes (1,000,000)   │
└──────────────┴─────────────┴──────────────────────┘
```

### CPU-Bound vs I/O-Bound

```java
public class BoundTypeDemo {

    // ✅ CPU-bound: Parallel streams shine here
    public long cpuBoundParallel(List<String> data) {
        return data.parallelStream()
            .map(s -> expensiveComputation(s))
            .mapToLong(String::length)
            .sum();
    }

    // ❌ I/O-bound: DO NOT use parallel streams!
    public List<String> ioBoundBad(List<String> urls) {
        return urls.parallelStream()
            .map(url -> fetchFromNetwork(url)) // ❌ Blocks common pool threads!
            .collect(Collectors.toList());
        // This can starve the common pool, affecting the ENTIRE JVM!
    }

    // ✅ I/O-bound: Use CompletableFuture with custom executor instead
    public List<String> ioBoundGood(List<String> urls) {
        ExecutorService ioExecutor = Executors.newFixedThreadPool(20);
        List<CompletableFuture<String>> futures = urls.stream()
            .map(url -> CompletableFuture.supplyAsync(
                () -> fetchFromNetwork(url), ioExecutor))
            .collect(Collectors.toList());

        return futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
    }

    private String expensiveComputation(String s) {
        // CPU-intensive work
        return s.chars().mapToObj(c -> String.valueOf((char)c))
            .sorted().collect(Collectors.joining());
    }

    private String fetchFromNetwork(String url) {
        try { Thread.sleep(100); } catch (InterruptedException e) {}
        return "response from " + url;
    }
}
```

### Custom ForkJoinPool

```java
public class CustomPoolDemo {
    public static void main(String[] args) throws Exception {
        List<Integer> numbers = IntStream.rangeClosed(1, 1_000_000)
            .boxed()
            .collect(Collectors.toList());

        // Default: uses ForkJoinPool.commonPool()
        // Parallelism = Runtime.getRuntime().availableProcessors() - 1

        // Custom pool with specific parallelism
        ForkJoinPool customPool = new ForkJoinPool(8);

        try {
            long result = customPool.submit(() ->
                numbers.parallelStream()
                    .mapToLong(n -> n * n)
                    .sum()
            ).get();

            System.out.println("Sum of squares: " + result);
        } finally {
            customPool.shutdown();
        }
    }
}
```

### Data Source Matters

```java
// Splittability affects parallel performance:

// ✅ GOOD - easily split in half
ArrayList<Integer> arrayList;     // Excellent
int[] array;                      // Excellent
IntStream.range(0, n);           // Excellent

// ⚠️ MODERATE
HashSet<Integer> hashSet;        // Decent
TreeSet<Integer> treeSet;        // Decent

// ❌ POOR - hard to split evenly
LinkedList<Integer> linkedList;  // Bad! Must traverse to find midpoint
BufferedReader.lines();          // Bad! Sequential source
Stream.iterate(0, i -> i + 1);  // Bad! Inherently sequential

// Rule: Use data structures with O(1) random access for parallel streams
```

### Ordered vs Unordered

```java
public class OrderingDemo {
    public static void main(String[] args) {
        List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8);

        // ❌ Maintaining order in parallel has overhead
        List<Integer> orderedResult = numbers.parallelStream()
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toList()); // Preserves encounter order

        // ✅ Unordered allows better parallelism
        Set<Integer> unorderedResult = numbers.parallelStream()
            .unordered() // Hint: order doesn't matter
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toSet());

        // findAny() is better than findFirst() in parallel
        Optional<Integer> any = numbers.parallelStream()
            .filter(n -> n > 3)
            .findAny(); // ✅ Any matching element (faster in parallel)

        Optional<Integer> first = numbers.parallelStream()
            .filter(n -> n > 3)
            .findFirst(); // ❌ Must find the first in encounter order (slower)

        // forEachOrdered preserves order (negates parallelism benefit)
        numbers.parallelStream()
            .forEachOrdered(System.out::println); // ❌ Sequential printing
    }
}
```

### Performance Pitfalls

```java
public class ParallelPitfalls {

    // ❌ Pitfall 1: Shared mutable state
    public void sharedStateBug() {
        List<Integer> results = new ArrayList<>(); // NOT thread-safe!
        IntStream.rangeClosed(1, 1000)
            .parallel()
            .forEach(results::add); // ❌ ConcurrentModificationException or lost elements!

        // ✅ Fix: Use collect() or thread-safe collection
        List<Integer> safe = IntStream.rangeClosed(1, 1000)
            .parallel()
            .boxed()
            .collect(Collectors.toList()); // Thread-safe reduction
    }

    // ❌ Pitfall 2: Auto-boxing overhead
    public long boxingOverhead(List<Integer> numbers) {
        return numbers.parallelStream()
            .reduce(0, Integer::sum); // ❌ Auto-boxing in every addition

        // ✅ Fix
        // return numbers.parallelStream().mapToInt(Integer::intValue).sum();
    }

    // ❌ Pitfall 3: Small dataset
    public long smallDataset() {
        List<Integer> tiny = List.of(1, 2, 3, 4, 5);
        return tiny.parallelStream()
            .mapToLong(Integer::longValue)
            .sum(); // ❌ Overhead > benefit for small datasets
    }

    // ❌ Pitfall 4: Operations with side effects
    public void sideEffects() {
        AtomicInteger counter = new AtomicInteger(0);
        IntStream.rangeClosed(1, 1000)
            .parallel()
            .peek(n -> counter.incrementAndGet()) // ❌ Side effect in stream
            .sum();
    }
}
```

### Decision Checklist

```
Should I use parallel streams?

 □ Is N * Q > 10,000?
 □ Is the task CPU-bound (not I/O)?
 □ Is the data source easily splittable (Array, ArrayList)?
 □ Is there no shared mutable state?
 □ Are operations stateless and independent?
 □ Is order unimportant (or using unordered operations)?
 □ Have I measured and confirmed improvement?

If ALL boxes are checked → ✅ Use parallel streams
If ANY box is unchecked → ⚠️ Benchmark first or use sequential
```

---

## Anti-Patterns

❌ **Using parallel streams for I/O operations**  
❌ **Modifying shared mutable state inside parallel stream operations**  
❌ **Using parallel streams for small datasets**  
❌ **Using LinkedList or other poorly-splittable sources**  
❌ **Using parallel streams without benchmarking**

## Best Practices

✅ Benchmark before and after parallelization (use JMH)  
✅ Use `ArrayList`, arrays, or `IntStream.range()` as data sources  
✅ Prefer stateless, side-effect-free operations  
✅ Use `Collectors.toList()` instead of manual accumulation  
✅ Use custom `ForkJoinPool` to isolate from other parallel operations  
✅ Use `CompletableFuture` with custom executor for I/O-bound work

---

[← CompletableFuture](./q15-completable-future.md) | [Next: Concurrent Collections →](./q17-concurrent-collections.md)

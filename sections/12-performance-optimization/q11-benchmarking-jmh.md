# Benchmarking with JMH

[Home](../../README.md) > [Performance Optimization](./README.md) > Benchmarking with JMH

---

## Overview

JMH (Java Microbenchmark Harness) is the definitive tool for writing reliable Java benchmarks. Developed by the same team that builds the JVM, JMH handles warmup, JIT compilation, dead code elimination, and other pitfalls that make manual benchmarking unreliable.

---

## 1. JMH Setup

```xml
<!-- Maven dependency -->
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>1.37</version>
</dependency>
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>1.37</version>
    <scope>provided</scope>
</dependency>

<!-- Maven plugin to build benchmarks -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.5.1</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
            <configuration>
                <transformers>
                    <transformer implementation=
                        "org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>org.openjdk.jmh.Main</mainClass>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```bash
# Generate JMH project from archetype
mvn archetype:generate \
    -DinteractiveMode=false \
    -DarchetypeGroupId=org.openjdk.jmh \
    -DarchetypeArtifactId=jmh-java-benchmark-archetype \
    -DgroupId=com.example \
    -DartifactId=benchmarks \
    -Dversion=1.0

# Build and run
mvn clean package
java -jar target/benchmarks.jar
```

---

## 2. Core Annotations

### @Benchmark

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)           // measure average time per operation
@OutputTimeUnit(TimeUnit.NANOSECONDS)       // report in nanoseconds
@Warmup(iterations = 5, time = 1)          // 5 warmup iterations, 1 second each
@Measurement(iterations = 10, time = 1)    // 10 measurement iterations
@Fork(2)                                    // 2 separate JVM forks
public class StringBenchmark {

    @Benchmark
    public String concatOperator() {
        String result = "";
        for (int i = 0; i < 100; i++) {
            result += "item" + i;
        }
        return result;
    }

    @Benchmark
    public String stringBuilder() {
        StringBuilder sb = new StringBuilder(600);
        for (int i = 0; i < 100; i++) {
            sb.append("item").append(i);
        }
        return sb.toString();
    }
}
```

### @State

```java
// Shared state across benchmark methods
@State(Scope.Benchmark)  // one instance shared by all threads
public class MapBenchmark {

    private Map<String, Integer> hashMap;
    private Map<String, Integer> treeMap;
    private String[] keys;

    @Setup(Level.Trial)  // run once before all iterations
    public void setup() {
        hashMap = new HashMap<>();
        treeMap = new TreeMap<>();
        keys = new String[10_000];

        for (int i = 0; i < 10_000; i++) {
            String key = "key_" + i;
            keys[i] = key;
            hashMap.put(key, i);
            treeMap.put(key, i);
        }
    }

    @TearDown(Level.Trial)  // run once after all iterations
    public void tearDown() {
        hashMap = null;
        treeMap = null;
    }

    @Benchmark
    public Integer hashMapGet() {
        return hashMap.get(keys[5000]);
    }

    @Benchmark
    public Integer treeMapGet() {
        return treeMap.get(keys[5000]);
    }
}

// State scopes:
// Scope.Benchmark - one instance shared by all threads
// Scope.Thread    - one instance per thread (no sharing)
// Scope.Group     - one instance per thread group
```

### @Param

```java
@State(Scope.Benchmark)
public class CollectionSizeBenchmark {

    @Param({"10", "100", "1000", "10000"})  // test multiple sizes
    private int size;

    private List<Integer> arrayList;
    private List<Integer> linkedList;

    @Setup(Level.Trial)
    public void setup() {
        arrayList = new ArrayList<>();
        linkedList = new LinkedList<>();
        for (int i = 0; i < size; i++) {
            arrayList.add(i);
            linkedList.add(i);
        }
    }

    @Benchmark
    public int arrayListGet() {
        return arrayList.get(size / 2);  // middle element
    }

    @Benchmark
    public int linkedListGet() {
        return linkedList.get(size / 2); // middle element
    }
}

// Output shows results for each parameter combination:
// CollectionSizeBenchmark.arrayListGet   size=10      3.5 ns
// CollectionSizeBenchmark.arrayListGet   size=100     3.5 ns
// CollectionSizeBenchmark.arrayListGet   size=1000    3.5 ns   (O(1))
// CollectionSizeBenchmark.linkedListGet  size=10      15 ns
// CollectionSizeBenchmark.linkedListGet  size=100     120 ns
// CollectionSizeBenchmark.linkedListGet  size=1000    1200 ns  (O(n))
```

---

## 3. Benchmark Modes

```java
public class BenchmarkModes {

    // Throughput: operations per second
    @Benchmark
    @BenchmarkMode(Mode.Throughput)
    @OutputTimeUnit(TimeUnit.SECONDS)
    public int throughputMode() {
        return computeSomething();
    }
    // Output: 15,234,567.890 ops/s

    // Average Time: average time per operation
    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.NANOSECONDS)
    public int avgTimeMode() {
        return computeSomething();
    }
    // Output: 65.7 ns/op

    // Sample Time: distribution of times (percentiles)
    @Benchmark
    @BenchmarkMode(Mode.SampleTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public int sampleTimeMode() {
        return computeSomething();
    }
    // Output: p50=0.065 μs, p90=0.082 μs, p99=0.150 μs, p99.9=12.5 μs

    // Single Shot: one cold invocation (no warmup) - for startup time
    @Benchmark
    @BenchmarkMode(Mode.SingleShotTime)
    @OutputTimeUnit(TimeUnit.MILLISECONDS)
    public void singleShotMode() {
        initializeHeavyFramework();
    }
    // Output: 1234.5 ms/op

    // All modes at once
    @Benchmark
    @BenchmarkMode(Mode.All)
    public int allModes() {
        return computeSomething();
    }
}
```

---

## 4. Warmup Configuration

```java
@Warmup(iterations = 5, time = 2, timeUnit = TimeUnit.SECONDS)
// 5 warmup iterations, 2 seconds each = 10 seconds warmup total
// Purpose: Let JIT compile hot methods before measuring

@Measurement(iterations = 10, time = 2, timeUnit = TimeUnit.SECONDS)
// 10 measurement iterations, 2 seconds each = 20 seconds of data

@Fork(2)
// Run in 2 separate JVM processes to avoid profile pollution
// Each fork does its own warmup + measurement

// Why fork matters:
// Fork 1: JIT may compile methods differently
// Fork 2: Different compilation order may yield different performance
// JMH averages across forks for reliable results
```

---

## 5. Blackhole Consumption

```java
import org.openjdk.jmh.infra.Blackhole;

public class BlackholeBenchmark {

    // BAD: JIT may eliminate dead code (result is unused)
    @Benchmark
    public void deadCode() {
        Math.log(42);  // JIT sees result is unused -> eliminates the call!
        // Benchmark measures NOTHING
    }

    // GOOD: Return value (JMH consumes it automatically)
    @Benchmark
    public double returnValue() {
        return Math.log(42);  // JMH captures the return value
    }

    // GOOD: Blackhole for multiple values
    @Benchmark
    public void multipleValues(Blackhole bh) {
        bh.consume(Math.log(42));     // force computation
        bh.consume(Math.sqrt(42));    // force computation
        bh.consume(Math.sin(42));     // force computation
    }

    // BAD: Accumulating results (folds into constant)
    @Benchmark
    public double foldedComputation() {
        double sum = 0;
        for (int i = 0; i < 100; i++) {
            sum += Math.log(i + 1);  // JIT may pre-compute this constant loop
        }
        return sum;
    }

    // GOOD: Use @State field to prevent constant folding
    @State(Scope.Thread)
    public static class Input {
        int value = 42;  // JIT cannot assume this is constant
    }

    @Benchmark
    public double withState(Input input) {
        return Math.log(input.value); // cannot be constant-folded
    }
}
```

---

## 6. Common Pitfalls

### Dead Code Elimination

```java
// PITFALL: Benchmark measures nothing
@Benchmark
public void deadCodeBenchmark() {
    new ArrayList<>(100);  // result unused, JIT eliminates allocation
}

// FIX: Return or consume the result
@Benchmark
public List<Object> fixedBenchmark() {
    return new ArrayList<>(100);  // JMH uses return value
}
```

### Constant Folding

```java
// PITFALL: JIT computes result at compile time
@Benchmark
public int constantFolding() {
    return 2 + 3;  // JIT replaces with: return 5
}

// FIX: Use @State fields
@State(Scope.Thread)
public static class Operands {
    int a = 2;
    int b = 3;
}

@Benchmark
public int fixedFolding(Operands ops) {
    return ops.a + ops.b;  // values not known at compile time
}
```

### Loop Optimizations

```java
// PITFALL: JIT may unroll, vectorize, or eliminate the loop
@Benchmark
public int loopBenchmark() {
    int sum = 0;
    for (int i = 0; i < 1000; i++) {
        sum += i;  // JIT replaces with: return 499500
    }
    return sum;
}

// FIX: Use @Param or @State to make loop bounds opaque
@State(Scope.Thread)
public static class LoopState {
    @Param({"100", "1000", "10000"})
    int size;
    int[] data;

    @Setup
    public void setup() {
        data = new int[size];
        for (int i = 0; i < size; i++) data[i] = i;
    }
}

@Benchmark
public int fixedLoop(LoopState state) {
    int sum = 0;
    for (int i = 0; i < state.data.length; i++) {
        sum += state.data[i];
    }
    return sum;
}
```

---

## 7. Complete Benchmark Example

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(2)
@State(Scope.Benchmark)
public class SortingBenchmark {

    @Param({"100", "1000", "10000", "100000"})
    private int size;

    private int[] data;
    private int[] dataCopy;

    @Setup(Level.Trial)
    public void setup() {
        data = new Random(42).ints(size).toArray();
    }

    @Setup(Level.Invocation) // fresh copy before each invocation
    public void copyData() {
        dataCopy = Arrays.copyOf(data, data.length);
    }

    @Benchmark
    public int[] arraysSort() {
        Arrays.sort(dataCopy); // dual-pivot quicksort
        return dataCopy;
    }

    @Benchmark
    public int[] parallelSort() {
        Arrays.parallelSort(dataCopy); // parallel merge sort
        return dataCopy;
    }

    // Runner
    public static void main(String[] args) throws Exception {
        Options opt = new OptionsBuilder()
            .include(SortingBenchmark.class.getSimpleName())
            .forks(2)
            .build();
        new Runner(opt).run();
    }
}
```

### Expected Output

```
Benchmark                     (size)  Mode  Cnt     Score     Error  Units
SortingBenchmark.arraysSort      100  avgt   20     785.3 ±   12.4  ns/op
SortingBenchmark.arraysSort     1000  avgt   20   12456.7 ±  234.5  ns/op
SortingBenchmark.arraysSort    10000  avgt   20  164532.1 ± 3456.7  ns/op
SortingBenchmark.arraysSort   100000  avgt   20  2345678.9 ± 45678  ns/op
SortingBenchmark.parallelSort    100  avgt   20    1234.5 ±   23.4  ns/op
SortingBenchmark.parallelSort   1000  avgt   20   11234.5 ±  456.7  ns/op
SortingBenchmark.parallelSort  10000  avgt   20   95678.9 ± 2345.6  ns/op
SortingBenchmark.parallelSort 100000  avgt   20  856789.0 ± 12345  ns/op
```

---

## 8. Running Benchmarks

```bash
# Run all benchmarks
java -jar target/benchmarks.jar

# Run specific benchmark class
java -jar target/benchmarks.jar StringBenchmark

# Run with regex filter
java -jar target/benchmarks.jar ".*Sort.*"

# Override warmup and measurement
java -jar target/benchmarks.jar -wi 3 -i 5 -f 1

# Output in JSON format
java -jar target/benchmarks.jar -rf json -rff results.json

# List available benchmarks
java -jar target/benchmarks.jar -l

# Profile while benchmarking (with async-profiler)
java -jar target/benchmarks.jar -prof async:libPath=/path/to/libasyncProfiler.so

# GC profiling
java -jar target/benchmarks.jar -prof gc

# Stack profiler
java -jar target/benchmarks.jar -prof stack
```

---

## 9. JMH Best Practices

| Practice                           | Why                                         |
|--------------------------------------|---------------------------------------------|
| Always use `@Fork(>=1)`              | Isolate JIT profiles between runs           |
| Return or consume benchmark results  | Prevent dead code elimination               |
| Use `@State` for inputs              | Prevent constant folding                    |
| Pre-size collections in `@Setup`     | Don't benchmark setup, only the operation   |
| Use `Level.Invocation` carefully     | High overhead; use `Level.Trial` when possible |
| Compare relative, not absolute       | Times vary across machines                  |
| Check `Error` column                 | High error means unstable benchmark         |
| Use `@Param` for scaling tests       | Shows how performance scales with input     |

---

## Interview Tips

- JMH is developed by the JVM team; it's the only reliable way to micro-benchmark Java
- Always return or `Blackhole.consume()` benchmark results to prevent dead code elimination
- `@Fork` isolates JIT compilation profiles between benchmark runs
- `@State(Scope.Thread)` avoids thread contention in the benchmark itself
- `Level.Invocation` has high overhead; prefer `Level.Trial` or `Level.Iteration`
- Never use `System.nanoTime()` loops for benchmarking; JMH handles timing correctly
- Check the `Error` (±) column; if it's large relative to `Score`, results are unreliable

---

[<< Previous: Profiling Tools](./q10-profiling-tools.md) | [Next: Micro-Optimizations >>](./q12-micro-optimizations.md)

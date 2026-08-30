# Are Micro-Optimizations Worth It?

[Home](../../README.md) > [Performance Optimization](./README.md) > Micro-Optimizations

---

## Overview

Micro-optimizations are small code-level changes aimed at reducing CPU cycles or memory usage. While rarely the solution to real performance problems, understanding them deepens knowledge of JVM internals and helps in performance-critical code paths.

---

## 1. When Micro-Optimizations Matter vs Premature Optimization

### Donald Knuth's Rule

```
"Premature optimization is the root of all evil."
  - But the full quote continues:
"Yet we should not pass up our opportunities in that critical 3%."
```

### Decision Framework

```
Should you micro-optimize?

1. Have you profiled? (No -> STOP, profile first)
2. Is this code in a hot path? (No -> don't bother)
3. Is the optimization measurable with JMH? (No -> don't bother)
4. Does it hurt readability? (Yes -> probably not worth it)
5. Is it the bottleneck? (No -> optimize the real bottleneck)

Hot path examples:
  - Inner loop processing millions of records
  - Serialization/deserialization in high-throughput services
  - Hash computation in custom data structures
  - Tight numerical computation loops
```

---

## 2. JIT Optimizations That Make Manual Optimization Unnecessary

The JIT compiler already performs these optimizations. Doing them manually is wasteful.

```java
// 1. Method inlining - JIT inlines small/hot methods automatically
// Manual "optimization" by avoiding method calls is UNNECESSARY
public int getX() { return x; }
// JIT inlines this: obj.getX() becomes direct field access

// 2. Bounds check elimination
int[] arr = new int[100];
for (int i = 0; i < arr.length; i++) {
    arr[i] = i;  // JIT eliminates bounds check after proving i < arr.length
}

// 3. Loop unrolling
for (int i = 0; i < 4; i++) {
    sum += data[i];
}
// JIT may transform to:
// sum += data[0] + data[1] + data[2] + data[3];

// 4. Null check elimination
if (obj != null) {
    obj.method();   // JIT knows obj is non-null here, skips null check
}

// 5. Common subexpression elimination
double a = Math.sqrt(x * x + y * y);
double b = Math.sqrt(x * x + y * y);  // JIT reuses result from first computation

// 6. Dead code elimination
if (false) {
    expensiveOperation();  // JIT removes entirely
}

// 7. Escape analysis + scalar replacement (covered in q03)
Point p = new Point(3, 4);
int dist = p.x + p.y;  // JIT may eliminate Point allocation
```

---

## 3. Loop Optimization

```java
// Loop-invariant code motion (JIT does this, but being explicit helps readability)

// BEFORE: recomputation every iteration
for (int i = 0; i < list.size(); i++) {  // list.size() called each iteration
    double result = Math.PI * radius;     // same computation each iteration
    process(list.get(i), result);
}

// AFTER: hoist invariants out of loop
int size = list.size();                    // computed once
double circumferenceFactor = Math.PI * radius; // computed once
for (int i = 0; i < size; i++) {
    process(list.get(i), circumferenceFactor);
}

// Loop interchange for cache efficiency
int[][] matrix = new int[1000][1000];

// BAD: Column-major traversal (cache-unfriendly in row-major Java)
for (int col = 0; col < 1000; col++) {
    for (int row = 0; row < 1000; row++) {
        matrix[row][col] = row + col;  // jumps 1000 ints per access
    }
}

// GOOD: Row-major traversal (cache-friendly)
for (int row = 0; row < 1000; row++) {
    for (int col = 0; col < 1000; col++) {
        matrix[row][col] = row + col;  // sequential memory access
    }
}
// Row-major is ~3-5x faster due to CPU cache line utilization
```

---

## 4. Branch Prediction

```java
// CPU branch predictor learns common branch outcomes
// Sorted data leads to predictable branches

// Example: summing values > 128 from an array
int[] data = new int[32768];
// Fill with random values 0-255

// SORTED array: branch predictor learns pattern
// [0,0,0,...,128,129,130,...,255,255,255]
// First half: always skip (predictor: "don't take")
// Second half: always sum (predictor: "take")
// Branch prediction accuracy: ~99%

// UNSORTED array: branches are random
// [203,45,178,22,199,...] - unpredictable
// Branch prediction accuracy: ~50% -> many mispredictions

// Benchmark results:
// Sorted:   ~2.5 ms   (branch prediction success)
// Unsorted: ~8.0 ms   (branch misprediction penalty ~15 cycles each)

// Alternative: branchless code (avoid the branch entirely)
for (int i = 0; i < data.length; i++) {
    // Branching version
    // if (data[i] >= 128) sum += data[i];

    // Branchless version (no branch prediction needed)
    int mask = (data[i] - 128) >> 31;  // 0 if >= 128, -1 if < 128
    sum += ~mask & data[i];
}
```

---

## 5. Locality of Reference

```java
// Spatial locality: access nearby memory addresses
// Temporal locality: access same memory addresses repeatedly

// GOOD spatial locality: arrays (contiguous memory)
int[] array = new int[1_000_000];
long sum = 0;
for (int i = 0; i < array.length; i++) {
    sum += array[i];  // sequential, cache-friendly
}

// POOR spatial locality: linked list (scattered nodes)
LinkedList<Integer> list = new LinkedList<>();
// Each node is a separate heap object at unpredictable addresses
long sum = 0;
for (int val : list) {
    sum += val;  // pointer chasing, cache-unfriendly
}

// Benchmark: Array vs LinkedList sequential sum (1M elements)
// int[]:              ~0.5 ms   (cache line prefetching)
// LinkedList<Integer>: ~8.0 ms   (cache misses on every node)
```

### Structure of Arrays vs Array of Structures

```java
// Array of Structures (AoS) - traditional OOP
class Particle {
    double x, y, z;      // position
    double vx, vy, vz;   // velocity
    double mass;
}
Particle[] particles = new Particle[100_000];
// Each Particle is a separate heap object
// Updating positions requires loading full objects (waste cache)

// Structure of Arrays (SoA) - data-oriented design
class ParticleSystem {
    double[] x, y, z;       // position arrays
    double[] vx, vy, vz;    // velocity arrays
    double[] mass;

    // Update positions: only x,y,z and vx,vy,vz arrays are accessed
    void updatePositions(double dt) {
        for (int i = 0; i < x.length; i++) {
            x[i] += vx[i] * dt;  // sequential access, cache-friendly
            y[i] += vy[i] * dt;
            z[i] += vz[i] * dt;
        }
    }
}
// SoA is ~2-4x faster for operations that access only a few fields
```

---

## 6. CPU Cache Effects

```java
// L1 cache: ~32 KB,  ~1 ns access
// L2 cache: ~256 KB, ~4 ns access
// L3 cache: ~8 MB,   ~12 ns access
// Main RAM:           ~65 ns access

// Cache line size: typically 64 bytes
// Loading one int loads 16 ints (64 / 4 bytes)

// Stride access pattern matters:
int[][] matrix = new int[4096][4096];

// Stride-1 (every element): ~50 ms
for (int i = 0; i < 4096; i++)
    for (int j = 0; j < 4096; j++)
        sum += matrix[i][j];

// Stride-16 (every 16th element): ~45 ms (still within cache line)
for (int i = 0; i < 4096; i++)
    for (int j = 0; j < 4096; j += 16)
        sum += matrix[i][j];

// Stride-4096 (column traversal): ~200 ms (cache miss every access)
for (int j = 0; j < 4096; j++)
    for (int i = 0; i < 4096; i++)
        sum += matrix[i][j];
```

---

## 7. Worthwhile Micro-Optimizations

These are micro-optimizations that are almost always worth doing:

```java
// 1. Use primitives instead of wrappers in hot paths
// BAD
Long sum = 0L;
for (Long value : values) { sum += value; } // autoboxing

// GOOD
long sum = 0L;
for (int i = 0; i < values.length; i++) { sum += values[i]; }

// 2. Bit manipulation instead of division/modulo for powers of 2
int mod = value % 16;        // slow: division
int mod = value & 15;        // fast: bitwise AND (when divisor is power of 2)

int div = value / 8;         // slow: division
int div = value >> 3;        // fast: right shift (divide by 2^3)

boolean isEven = (value % 2 == 0);  // slow
boolean isEven = (value & 1) == 0;  // fast

// 3. Avoid autoboxing in hot loops
// BAD
Map<Integer, Integer> map = new HashMap<>();
for (int i = 0; i < 1_000_000; i++) {
    map.put(i, i * 2);  // autoboxes BOTH key and value
}

// GOOD: Use primitive collection
IntIntHashMap map = new IntIntHashMap();  // Eclipse Collections
for (int i = 0; i < 1_000_000; i++) {
    map.put(i, i * 2);  // no boxing
}

// 4. switch expression vs if-else chain (JIT optimizes both, but switch is cleaner)
// JIT may use a jump table for dense switch cases

// 5. final keyword for JIT hints
// Marking classes/methods final helps JIT with devirtualization
public final class FastMath {
    public static final double TWO_PI = 2.0 * Math.PI;

    public final double compute(double x) {
        return x * TWO_PI; // JIT can inline more aggressively
    }
}
```

---

## 8. Optimizations That DON'T Help

```java
// 1. Manual method inlining (JIT does this better)
// DON'T copy method body to avoid call overhead
// JIT inlines hot methods automatically (up to 35 bytes by default)

// 2. Using ++ vs += 1 (identical bytecode)
i++;      // same as
i += 1;   // same as
i = i + 1;

// 3. StringBuilder outside loops for simple concatenation
// The compiler already uses StringBuilder for: "a" + b + "c"

// 4. Replacing ArrayList with arrays for "speed"
// ArrayList is backed by an array; get(i) is array[i] (after bounds check)
// The bounds check is often eliminated by JIT

// 5. Rewriting for-each as indexed loop
// JIT optimizes Iterator-based for-each for ArrayList to indexed access

// 6. Caching array.length in a local variable
// JIT hoists the length load out of the loop automatically

// 7. Using multiplication instead of Math.pow for small integers
// Math.pow(x, 2) vs x * x : JIT optimizes Math.pow for small integer exponents
```

---

## 9. Micro-Optimization Decision Matrix

| Optimization                     | Worth It?       | When?                        |
|----------------------------------|-----------------|------------------------------|
| StringBuilder in loops           | Always          | Any loop with string concat  |
| Pre-size collections             | Always          | Known size collections       |
| Primitives over wrappers         | Usually         | Hot paths, large data        |
| Bit manipulation                 | Sometimes       | High-frequency operations    |
| SoA layout                       | Sometimes       | Data-intensive computation   |
| Row-major array traversal        | Yes             | Matrix operations            |
| Manual inlining                  | Never           | JIT does this better         |
| Avoiding method calls            | Never           | JIT inlines automatically    |
| Replacing for-each with index    | Rarely          | Only after profiling         |
| Cache-aware data layout          | In hot paths    | Millions of iterations       |

---

## Interview Tips

- Quote Knuth correctly: optimize the critical 3%, not everything
- JIT already performs inlining, dead code elimination, escape analysis, and loop optimization
- The biggest wins come from algorithmic improvements, not micro-optimizations
- Cache locality matters: sequential array access is 10-20x faster than linked list traversal
- Branch prediction effects are real: sorted data can be 3x faster to process
- Autoboxing in hot loops is one of the few micro-optimizations always worth fixing
- Always validate with JMH; intuition about performance is often wrong

---

[<< Previous: Benchmarking with JMH](./q11-benchmarking-jmh.md) | [Next: Algorithm Selection >>](./q13-algorithm-selection.md)

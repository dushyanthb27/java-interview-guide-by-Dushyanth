# Q7: How Does JIT Compilation Work?

## 📋 Question
Explain how Just-In-Time (JIT) compilation works in the JVM. What are C1 and C2 compilers, and what optimizations does the JIT perform?

## Difficulty: 🟠 Hard

---

## 💡 Answer

JIT (Just-In-Time) compilation is the JVM's strategy for converting frequently-executed **bytecode** into optimized **native machine code** at runtime. This is why Java achieves near-native performance despite being a bytecode-interpreted language.

### Interpreter vs JIT Compiler

```
┌─────────────────────────────────────────────────────────────────┐
│                   EXECUTION PIPELINE                            │
│                                                                 │
│  .class (bytecode)                                              │
│        │                                                        │
│        ├──▶ INTERPRETER (always available)                      │
│        │    • Reads each bytecode instruction                   │
│        │    • Executes immediately                              │
│        │    • ✅ Fast startup (no compilation delay)             │
│        │    • ❌ Slow execution (decode each instruction)        │
│        │                                                        │
│        └──▶ JIT COMPILER (for hot code)                         │
│             • Compiles entire methods to native code            │
│             • Stores in Code Cache                              │
│             • ❌ Slow startup (compilation overhead)             │
│             • ✅ Fast execution (runs native machine code)       │
│                                                                 │
│  Method first call:    Interpreter (quick start)                │
│  Method 10000th call:  JIT-compiled native code (blazing fast)  │
└─────────────────────────────────────────────────────────────────┘
```

### Tiered Compilation (Default Since Java 8)

```
┌──────────────────────────────────────────────────────────────┐
│                    TIERED COMPILATION                         │
│           (-XX:+TieredCompilation — on by default)           │
│                                                              │
│  Level 0: Interpreter                                        │
│           └── Bytecode interpretation, profiling counters    │
│                    │                                         │
│                    ▼                                         │
│  Level 1: C1 — Simple compile (no profiling)                 │
│           └── Quick native code, basic optimizations         │
│                    │                                         │
│                    ▼                                         │
│  Level 2: C1 — Limited profiling                             │
│           └── Native code with basic profiling hooks         │
│                    │                                         │
│                    ▼                                         │
│  Level 3: C1 — Full profiling                                │
│           └── Native code with comprehensive profiling       │
│               (branch frequencies, type profiles)            │
│                    │                                         │
│                    ▼                                         │
│  Level 4: C2 — Full optimization                             │
│           └── Aggressively optimized native code             │
│               (inlining, escape analysis, loop unrolling)    │
│                                                              │
│  Typical path: Level 0 → Level 3 → Level 4                  │
│  Fast path:    Level 0 → Level 1 (trivial methods)           │
└──────────────────────────────────────────────────────────────┘
```

### C1 vs C2 Compilers

| Feature | C1 (Client) | C2 (Server) |
|---------|-------------|-------------|
| **Speed** | Fast compilation | Slow compilation |
| **Optimization** | Basic | Aggressive |
| **When** | Methods become warm | Methods become hot |
| **Profiling** | Collects profiling data | Uses profiling data |
| **Inlining** | Shallow | Deep |
| **Escape Analysis** | No | Yes |
| **Loop Unrolling** | Basic | Aggressive |
| **Use Case** | Quick startup, GUI apps | Long-running servers |

### Hot Spot Detection

The JVM identifies "hot" methods and loops using **invocation counters** and **back-edge counters**:

```java
public class HotSpotDetection {

    // Method invocation counter tracks call frequency
    public int computeValue(int x) {  // Called 10,000+ times → HOT
        return x * x + 2 * x + 1;
    }

    // Back-edge counter tracks loop iterations
    public long sumArray(int[] data) {
        long sum = 0;
        for (int val : data) {         // Loop body executed millions of times
            sum += val;                // Back-edge counter triggers compilation
        }
        return sum;
    }
}

// Default threshold: -XX:CompileThreshold=10000 (for non-tiered)
// With tiered compilation, thresholds are more nuanced:
//   Level 3 (C1): ~2000 invocations
//   Level 4 (C2): ~15000 invocations
```

### On-Stack Replacement (OSR)

```java
// What if a method has a hot LOOP but is only called once?
// Normal JIT waits for the next method invocation to use compiled code.
// OSR replaces the RUNNING method's interpreter frame with compiled code.

public class OSRExample {
    public static void main(String[] args) {
        long sum = 0;
        // This loop runs millions of iterations but main() is called only once
        for (int i = 0; i < 10_000_000; i++) {
            sum += i;
            // After ~10,000 iterations (back-edge counter threshold):
            //   1. JIT compiles the loop
            //   2. OSR: transfers execution from interpreter to compiled code
            //   3. Remaining iterations run as native code
        }
        System.out.println(sum);
    }
}

// Observe OSR with:
// java -XX:+PrintCompilation OSRExample
// Output includes '%' for OSR-compiled methods:
//   123   3 %     OSRExample::main @ 8 (42 bytes)
//                 ^ OSR marker     ^ bytecode index where OSR happens
```

### Key JIT Optimizations

#### 1. Method Inlining

```java
// Before inlining:
public int calculate(int x) {
    return square(x) + double_(x) + 1;  // 2 method calls
}
private int square(int x) { return x * x; }
private int double_(int x) { return x * 2; }

// After JIT inlining:
public int calculate(int x) {
    return (x * x) + (x * 2) + 1;  // No method calls — all inlined
}

// Inlining eliminates method call overhead (stack frame, parameter passing)
// and enables further optimizations on the combined code

// Flags:
// -XX:MaxInlineSize=35           (max bytecode size for automatic inlining)
// -XX:FreqInlineSize=325         (max size for hot methods)
// -XX:MaxInlineLevel=15          (max depth of inlining chain)
```

#### 2. Escape Analysis

```java
// Escape analysis determines if an object "escapes" the method/thread

public int sumPoints(int x1, int y1, int x2, int y2) {
    // This Point does NOT escape the method
    Point p1 = new Point(x1, y1);  // Candidate for stack allocation
    Point p2 = new Point(x2, y2);  // Candidate for stack allocation

    return p1.x + p1.y + p2.x + p2.y;
}

// JIT optimizations enabled by escape analysis:

// 1. Stack Allocation — allocate on stack instead of heap (no GC needed)
//    Object doesn't escape method → put it on the stack

// 2. Scalar Replacement — break object into fields, use registers
//    Point p1 = new Point(x1, y1) → just use x1 and y1 directly

// 3. Lock Elimination — remove unnecessary synchronization
//    synchronized(new Object()) { ... } → lock is useless, remove it

// After scalar replacement:
public int sumPoints(int x1, int y1, int x2, int y2) {
    return x1 + y1 + x2 + y2;  // No objects created at all!
}

// Flags:
// -XX:+DoEscapeAnalysis       (enabled by default)
// -XX:+EliminateAllocations   (scalar replacement, enabled by default)
// -XX:+EliminateLocks         (lock elimination, enabled by default)
```

#### 3. Loop Unrolling

```java
// Before unrolling:
for (int i = 0; i < 100; i++) {
    sum += array[i];  // 100 iterations, 100 branch checks
}

// After loop unrolling (JIT optimization):
for (int i = 0; i < 100; i += 4) {
    sum += array[i];      // Process 4 elements per iteration
    sum += array[i + 1];  // Only 25 branch checks
    sum += array[i + 2];  // Better CPU pipeline utilization
    sum += array[i + 3];  // Enables SIMD vectorization
}

// Flags:
// -XX:LoopUnrollLimit=50   (max number of unrolled iterations)
```

#### 4. Dead Code Elimination

```java
// Before:
public int compute(int x) {
    int unused = x * 100;    // Never used
    int debug = x + 42;      // Never used
    return x * 2;
}

// After JIT:
public int compute(int x) {
    return x * 2;  // Dead code removed entirely
}
```

#### 5. Devirtualization

```java
// Virtual method calls require vtable lookup — expensive
interface Shape { int area(); }

class Square implements Shape {
    int side;
    public int area() { return side * side; }
}

// If profiling shows area() is ALWAYS called on Square:
Shape s = getShape();
int a = s.area();          // Virtual call → vtable lookup

// JIT can devirtualize (speculative):
int a = ((Square) s).area();  // Direct call, can be inlined
// With a type guard:
if (s instanceof Square) {
    int a = s.side * s.side;  // Fully inlined!
} else {
    // Deoptimize — fall back to interpreter
}
```

### Code Cache

```
┌─────────────────────────────────────────┐
│           CODE CACHE                    │
│   (Stores JIT-compiled native code)     │
│                                         │
│   ┌──────────────────────┐              │
│   │ Non-method code      │ JVM internal │
│   ├──────────────────────┤              │
│   │ Profiled code (C1)   │ With profile │
│   ├──────────────────────┤              │
│   │ Non-profiled code(C2)│ Optimized    │
│   └──────────────────────┘              │
│                                         │
│   Default size: 240 MB (Java 11+)       │
│   -XX:ReservedCodeCacheSize=256m        │
│                                         │
│   If code cache fills up:               │
│   • JIT compilation STOPS               │
│   • App falls back to interpreter       │
│   • ⚠️ Significant performance drop     │
└─────────────────────────────────────────┘
```

### Graal Compiler (Alternative to C2)

```bash
# GraalVM's Graal compiler can replace C2
# Written in Java (unlike C2 which is written in C++)
# Benefits:
#   - Better escape analysis
#   - Partial escape analysis (unique to Graal)
#   - More aggressive inlining
#   - Ahead-of-time compilation (native-image)

# Use Graal as JIT compiler in HotSpot:
java -XX:+UnlockExperimentalVMOptions -XX:+UseJVMCICompiler -jar myapp.jar

# GraalVM native image (AOT compilation):
native-image -jar myapp.jar    # Produces native executable
# ✅ Instant startup, low memory
# ❌ No runtime profiling/optimization, limited reflection
```

### Observing JIT Compilation

```bash
# Print compilation events
java -XX:+PrintCompilation MyApp

# Output format:
#  timestamp compile_id attributes (tiered_level) method_name size deopt_type
#    123    1    b  3       java.lang.String::hashCode (55 bytes)
#    124    2    b  4       MyApp::hotMethod (23 bytes)
#    125    3  % b  4       MyApp::main @ 15 (120 bytes)
#
# Attributes: b=blocking, %=OSR, s=synchronized, !=has exception handler
# Level: 1-3=C1, 4=C2

# Print inlining decisions
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining MyApp

# Print assembly code (requires hsdis library)
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly MyApp
```

### ⚠️ Common Mistake
Writing microbenchmarks without JIT warm-up. The first few thousand invocations of a method run in the interpreter, which is much slower than JIT-compiled code. Always use **JMH (Java Microbenchmark Harness)** for benchmarks — it handles warm-up, dead code elimination prevention, and statistical analysis:

```java
@Benchmark
@Warmup(iterations = 5, time = 1)  // Let JIT compile before measuring
@Measurement(iterations = 5, time = 1)
public int benchmarkMethod() {
    return computeValue(42);
}
```

### ⚡ Performance Tip
Monitor code cache usage (`jcmd <pid> Compiler.codecache`) — if the code cache fills up, the JIT stops compiling and performance degrades severely. Increase it with `-XX:ReservedCodeCacheSize=512m` for large applications. Also, very large methods (> 8000 bytes of bytecode) are never JIT-compiled — keep methods small and focused to enable JIT optimizations.

---

## 🎯 Interviewer Insight
Interviewers want to hear about tiered compilation (C1 + C2), hot spot detection via counters, and at least two JIT optimizations (inlining and escape analysis are the most important). Strong candidates explain OSR, devirtualization, and the code cache. Mentioning Graal and AOT compilation shows awareness of cutting-edge JVM technology.

## ⚡ Follow-up Questions
1. What is the difference between AOT and JIT compilation?
2. How does escape analysis eliminate object allocations?
3. What happens when the code cache is full?

## 📚 Related Topics
- [JVM Architecture](./q01-jvm-architecture.md)
- [Bytecode Understanding](./q08-bytecode-understanding.md)
- [JVM Tuning Parameters](./q12-jvm-tuning-parameters.md)

# JVM Internals

[Home](../../README.md) > JVM Internals

---

## Overview
Understanding JVM internals separates senior developers from juniors. This section covers class loading, memory model, garbage collection, JIT compilation, and JVM tuning.

## Prerequisites
- [Java Fundamentals](../01-java-fundamentals/README.md)
- [Java Memory Management](../10-memory-management/README.md)

## Question Index
| # | Question | Difficulty | File |
|---|----------|------------|------|
| 1 | Explain JVM architecture | 🟡 | [q01](./q01-jvm-architecture.md) |
| 2 | How does class loading work? | 🟠 | [q02](./q02-class-loading-mechanism.md) |
| 3 | What are the different class loaders? | 🟠 | [q03](./q03-class-loaders.md) |
| 4 | Explain the Java Memory Model (JMM) | 🔴 | [q04](./q04-java-memory-model.md) |
| 5 | What GC algorithms are used? | 🟠 | [q05](./q05-gc-algorithms.md) |
| 6 | Compare garbage collectors (G1, ZGC, Shenandoah) | 🔴 | [q06](./q06-gc-collectors.md) |
| 7 | How does JIT compilation work? | 🟠 | [q07](./q07-jit-compilation.md) |
| 8 | How to understand Java bytecode? | 🟠 | [q08](./q08-bytecode-understanding.md) |
| 9 | Stack vs Heap in JVM | 🟡 | [q09](./q09-stack-vs-heap.md) |
| 10 | What is Metaspace? | 🟡 | [q10](./q10-metaspace.md) |
| 11 | PermGen vs Metaspace evolution | 🟡 | [q11](./q11-permgen-vs-metaspace.md) |
| 12 | Key JVM tuning parameters | 🟠 | [q12](./q12-jvm-tuning-parameters.md) |
| 13 | How to monitor JVM performance? | 🟠 | [q13](./q13-monitoring-jvm.md) |
| 14 | JVM security architecture | 🟡 | [q14](./q14-jvm-security.md) |
| 15 | Native methods and JNI | 🟠 | [q15](./q15-native-methods-jni.md) |

## Key Takeaways
- JVM uses a three-tier class loading delegation model
- Java Memory Model defines happens-before relationships for thread safety
- G1 is the default GC; ZGC offers sub-millisecond pauses for large heaps
- JIT compiles hot methods to native code with C1 (quick) and C2 (optimizing) compilers
- Metaspace replaced PermGen in Java 8 with native memory allocation

## Related Sections
- [Memory Management](../10-memory-management/README.md)
- [Performance Optimization](../12-performance-optimization/README.md)
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md)

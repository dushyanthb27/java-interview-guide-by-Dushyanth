# Java Memory Management

[Home](../../README.md) > Java Memory Management

---

## Overview
Understanding memory management is crucial for building performant, leak-free Java applications. This section covers the object lifecycle, reference types, memory leaks, and profiling techniques.

## Prerequisites
- [Java Fundamentals](../01-java-fundamentals/README.md)
- [JVM Internals](../07-jvm-internals/README.md)

## Question Index
| # | Question | Difficulty | File |
|---|----------|------------|------|
| 1 | How does stack vs heap memory work? | 🟡 | [q01](./q01-stack-vs-heap.md) |
| 2 | Explain the object lifecycle in Java | 🟡 | [q02](./q02-object-lifecycle.md) |
| 3 | How do memory leaks occur in Java? | 🟠 | [q03](./q03-memory-leaks.md) |
| 4 | What are weak references? | 🟠 | [q04](./q04-weak-references.md) |
| 5 | What are soft references? | 🟠 | [q05](./q05-soft-references.md) |
| 6 | What are phantom references? | 🔴 | [q06](./q06-phantom-references.md) |
| 7 | How do reference queues work? | 🔴 | [q07](./q07-reference-queues.md) |
| 8 | When should you use object pooling? | 🟠 | [q08](./q08-object-pooling.md) |
| 9 | How does the String Pool work? | 🟡 | [q09](./q09-string-pool.md) |
| 10 | Memory optimization techniques | 🟠 | [q10](./q10-memory-optimization.md) |
| 11 | How to profile memory usage? | 🟠 | [q11](./q11-profiling-memory.md) |
| 12 | Heap dump analysis techniques | 🔴 | [q12](./q12-heap-dump-analysis.md) |

## Key Takeaways
- Java manages memory automatically but leaks still occur through unintended references
- WeakReferences are collected at next GC; SoftReferences are kept until memory pressure
- String pool is in the heap (since Java 7) and uses weak references internally
- Object pooling is rarely beneficial in modern JVMs (except for expensive resources like DB connections)
- Heap dump analysis with MAT/VisualVM is essential for diagnosing memory issues

## Related Sections
- [JVM Internals](../07-jvm-internals/README.md)
- [Performance Optimization](../12-performance-optimization/README.md)
- [Collections Framework](../03-collections-framework/README.md)

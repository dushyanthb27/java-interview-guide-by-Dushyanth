# Collections Framework

[Home](../../README.md) > Collections Framework

---

## Overview
The Collections Framework is one of the most heavily tested topics in Java interviews. Understanding data structure internals, performance characteristics, and thread-safety is crucial.

## Prerequisites
- [Java Fundamentals](../01-java-fundamentals/README.md)
- [Object-Oriented Programming](../02-oop-concepts/README.md)

## Question Index
| # | Question | Difficulty | File |
|---|----------|------------|------|
| 1 | Explain the Java Collection hierarchy | 🟢 | [q01](./q01-collection-hierarchy.md) |
| 2 | How does ArrayList work internally? | 🟡 | [q02](./q02-arraylist.md) |
| 3 | How does LinkedList work and when to use it? | 🟡 | [q03](./q03-linkedlist.md) |
| 4 | Are Vector and Stack still relevant? | 🟡 | [q04](./q04-vector-stack.md) |
| 5 | How does HashSet work internally? | 🟡 | [q05](./q05-hashset.md) |
| 6 | Compare LinkedHashSet and TreeSet | 🟡 | [q06](./q06-linkedhashset-treeset.md) |
| 7 | How does HashMap work internally? | 🟠 | [q07](./q07-hashmap-internals.md) |
| 8 | Compare LinkedHashMap and TreeMap | 🟡 | [q08](./q08-linkedhashmap-treemap.md) |
| 9 | How does ConcurrentHashMap work? | 🔴 | [q09](./q09-concurrenthashmap.md) |
| 10 | Explain Queue and Deque implementations | 🟡 | [q10](./q10-queue-deque.md) |
| 11 | Iterator vs ListIterator | 🟢 | [q11](./q11-iterator-listiterator.md) |
| 12 | What is fail-fast vs fail-safe behavior? | 🟠 | [q12](./q12-fail-fast-fail-safe.md) |
| 13 | Comparable vs Comparator | 🟡 | [q13](./q13-comparable-comparator.md) |
| 14 | Collections utility class methods | 🟢 | [q14](./q14-collections-utility.md) |
| 15 | Streams API with Collections | 🟡 | [q15](./q15-streams-api-collections.md) |
| 16 | Optional handling and best practices | 🟡 | [q16](./q16-optional-handling.md) |
| 17 | Immutable Collections in Java | 🟠 | [q17](./q17-immutable-collections.md) |
| 18 | Collection performance comparison | 🟠 | [q18](./q18-collection-performance.md) |
| 19 | Thread-safe collection options | 🟠 | [q19](./q19-thread-safe-collections.md) |
| 20 | How to implement a custom Collection? | 🔴 | [q20](./q20-custom-collections.md) |

## Key Takeaways
- HashMap uses array + linked list + red-black tree (after threshold of 8)
- ArrayList is O(1) for random access, LinkedList is O(1) for insertion at known position
- ConcurrentHashMap uses CAS operations and synchronized blocks per node (Java 8+)
- Always override equals() and hashCode() together for collection keys
- Prefer immutable collections (List.of(), Map.of()) for thread safety

## Related Sections
- [Generics & Type System](../09-generics-type-system/README.md)
- [Java 8+ Features](../06-java8-plus-features/README.md)
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md)

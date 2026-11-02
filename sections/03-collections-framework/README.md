# Collections Framework

[Home](../../README.md) > Collections Framework

---

## Overview

The Collections Framework is one of the most heavily tested topics in Java interviews — and for good reason. Every senior Java developer needs to understand not just *which* collection to use, but *why* it behaves the way it does internally. This section digs into the mechanics of ArrayList's growth strategy, HashMap's collision resolution, ConcurrentHashMap's thread-safety guarantees, and the trade-offs that matter in production systems.

You'll learn how these data structures work under the hood, their performance characteristics, and when to choose one over another. We'll cover everything from the basic List/Set/Map hierarchy to advanced topics like custom collections, thread-safety patterns, and modern Java 8+ and 21+ features. The questions here range from fundamental concepts to deep-dive internals that separate junior from senior developers.

## Prerequisites

- [Java Fundamentals](../01-java-fundamentals/README.md)
- [Object-Oriented Programming](../02-oop-concepts/README.md)

## Question Index

| # | Question | Difficulty | File |
|---|----------|------------|------|
| Q1 | Explain the Java Collection hierarchy | 🟢 | [01-collection-hierarchy.md](./01-collection-hierarchy.md) |
| Q2 | How does ArrayList work internally? | 🟡 | [02-arraylist.md](./02-arraylist.md) |
| Q3 | How does LinkedList work and when to use it? | 🟡 | [03-linkedlist.md](./03-linkedlist.md) |
| Q4 | Are Vector and Stack still relevant? | 🟡 | [04-vector-stack.md](./04-vector-stack.md) |
| Q5 | How does HashSet work internally? | 🟡 | [05-hashset.md](./05-hashset.md) |
| Q6 | Compare LinkedHashSet and TreeSet | 🟡 | [06-linkedhashset-treeset.md](./06-linkedhashset-treeset.md) |
| Q7 | How does HashMap work internally? | 🟠 | [07-hashmap-internals.md](./07-hashmap-internals.md) |
| Q8 | Compare LinkedHashMap and TreeMap | 🟡 | [08-linkedhashmap-treemap.md](./08-linkedhashmap-treemap.md) |
| Q9 | How does ConcurrentHashMap work? | 🔴 | [09-concurrenthashmap.md](./09-concurrenthashmap.md) |
| Q10 | Explain Queue and Deque implementations | 🟡 | [10-queue-deque.md](./10-queue-deque.md) |
| Q11 | Iterator vs ListIterator | 🟢 | [11-iterator-listiterator.md](./11-iterator-listiterator.md) |
| Q12 | What is fail-fast vs fail-safe behavior? | 🟠 | [12-fail-fast-fail-safe.md](./12-fail-fast-fail-safe.md) |
| Q13 | Comparable vs Comparator | 🟡 | [13-comparable-comparator.md](./13-comparable-comparator.md) |
| Q14 | Collections utility class methods | 🟢 | [14-collections-utility.md](./14-collections-utility.md) |
| Q15 | Streams API with Collections | 🟡 | [15-streams-api-collections.md](./15-streams-api-collections.md) |
| Q16 | Optional handling and best practices | 🟡 | [16-optional-handling.md](./16-optional-handling.md) |
| Q17 | Immutable Collections in Java | 🟠 | [17-immutable-collections.md](./17-immutable-collections.md) |
| Q18 | Collection performance comparison | 🟠 | [18-collection-performance.md](./18-collection-performance.md) |
| Q19 | Thread-safe collection options | 🟠 | [19-thread-safe-collections.md](./19-thread-safe-collections.md) |
| Q20 | How to implement a custom Collection? | 🔴 | [20-custom-collections.md](./20-custom-collections.md) |

## Key Takeaways

- **HashMap** internally uses an array with linked list buckets that convert to red-black trees when a bucket exceeds 8 elements — this optimization prevents O(n) degradation in hash collision scenarios
- **ArrayList** provides O(1) random access via direct array indexing, while **LinkedList** offers O(1) insertions at known positions but O(n) random access — the choice depends on your access pattern
- **ConcurrentHashMap** uses CAS operations and synchronized blocks per bucket (Java 8+) instead of locking the entire map, enabling high concurrency with minimal contention
- Always override **equals() and hashCode() together** when using objects as collection keys — inconsistent implementations break the contract and cause unpredictable behavior
- Prefer **immutable collections** (`List.of()`, `Map.of()`, `Set.of()`) for thread safety — they're inherently thread-safe without synchronization overhead
- **Java 21** introduced Sequenced Collections API, providing unified methods for accessing first/last elements across Lists, Sets, and Maps
- Understanding **fail-fast vs fail-safe** iterators is crucial for debugging concurrent modification issues in production code

## Related Sections

- [Generics & Type System](../09-generics-type-system/README.md) — Type safety and wildcards in collections
- [Java 8+ Features](../06-java8-plus-features/README.md) — Streams API and functional collection operations
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md) — Thread-safety and concurrent programming

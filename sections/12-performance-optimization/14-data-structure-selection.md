# Data Structure Selection Guide

[Home](../../README.md) > [Performance Optimization](./README.md) > Data Structure Selection

---

## Overview

Choosing the right data structure is one of the most impactful performance decisions. The wrong choice can make operations 10-1000x slower. This guide provides a systematic approach to selecting the best data structure for each use case.

---

## 1. Decision Matrix for Data Structure Selection

```
What is the primary operation?
│
├── Key-Value Lookup
│   ├── Exact key match only?
│   │   ├── High concurrency? ──────────> ConcurrentHashMap
│   │   ├── Enum keys? ────────────────> EnumMap
│   │   ├── Insertion order needed? ───> LinkedHashMap
│   │   └── General purpose ──────────> HashMap
│   ├── Range queries / sorted? ──────> TreeMap
│   └── LRU cache behavior? ─────────> LinkedHashMap (access-order)
│
├── Ordered Sequence
│   ├── Random access by index? ─────> ArrayList
│   ├── Frequent insert/remove at ends?─> ArrayDeque
│   ├── Thread-safe list?
│   │   ├── Read-heavy? ────────────> CopyOnWriteArrayList
│   │   └── Write-heavy? ──────────> Collections.synchronizedList
│   └── Priority ordering? ─────────> PriorityQueue
│
├── Unique Elements
│   ├── No ordering needed? ─────────> HashSet
│   ├── Sorted order? ──────────────> TreeSet
│   ├── Insertion order? ──────────> LinkedHashSet
│   └── Enum values? ──────────────> EnumSet
│
└── Producer-Consumer / Queue
    ├── Bounded blocking? ──────────> ArrayBlockingQueue
    ├── Unbounded blocking? ────────> LinkedBlockingQueue
    ├── Non-blocking? ─────────────> ConcurrentLinkedQueue
    ├── Priority? ─────────────────> PriorityBlockingQueue
    ├── Delayed elements? ─────────> DelayQueue
    └── Work-stealing? ───────────> LinkedTransferQueue
```

---

## 2. Performance Characteristics Table

### List Implementations

| Operation          | ArrayList | LinkedList | CopyOnWriteArrayList |
|-------------------|-----------|------------|----------------------|
| `get(index)`      | O(1)      | O(n)       | O(1)                 |
| `add(end)`        | O(1)*     | O(1)       | O(n) (copy)          |
| `add(index)`      | O(n)      | O(1)**     | O(n) (copy)          |
| `remove(index)`   | O(n)      | O(1)**     | O(n) (copy)          |
| `contains()`      | O(n)      | O(n)       | O(n)                 |
| `iterator`        | O(n)      | O(n)       | O(n) snapshot        |
| Cache friendly    | Yes       | No         | Yes                  |
| Thread-safe       | No        | No         | Yes                  |

_* amortized  ** if already at position (finding position is O(n))_

### Map Implementations

| Operation       | HashMap  | TreeMap   | LinkedHashMap | ConcurrentHashMap |
|----------------|----------|-----------|---------------|-------------------|
| `put()`        | O(1)     | O(log n)  | O(1)          | O(1)              |
| `get()`        | O(1)     | O(log n)  | O(1)          | O(1)              |
| `remove()`     | O(1)     | O(log n)  | O(1)          | O(1)              |
| `containsKey()`| O(1)     | O(log n)  | O(1)          | O(1)              |
| Ordered        | No       | Sorted    | Insertion     | No                |
| Null keys      | 1 null   | No*       | 1 null        | No                |
| Thread-safe    | No       | No        | No            | Yes               |

_* TreeMap requires Comparable keys or a Comparator_

### Set Implementations

| Operation      | HashSet  | TreeSet   | LinkedHashSet | EnumSet         |
|---------------|----------|-----------|---------------|-----------------|
| `add()`       | O(1)     | O(log n)  | O(1)          | O(1)            |
| `contains()`  | O(1)     | O(log n)  | O(1)          | O(1) (bit test) |
| `remove()`    | O(1)     | O(log n)  | O(1)          | O(1)            |
| Ordered       | No       | Sorted    | Insertion     | Natural order   |
| Memory/entry  | ~48 bytes| ~56 bytes | ~64 bytes     | ~1 bit!         |

### Queue Implementations

| Queue                    | Bounded | Blocking | Ordering    | Thread-safe |
|--------------------------|---------|----------|-------------|-------------|
| `ArrayDeque`             | No*     | No       | FIFO/LIFO   | No          |
| `PriorityQueue`          | No      | No       | Priority    | No          |
| `ArrayBlockingQueue`     | Yes     | Yes      | FIFO        | Yes         |
| `LinkedBlockingQueue`    | Opt.    | Yes      | FIFO        | Yes         |
| `PriorityBlockingQueue`  | No      | Yes      | Priority    | Yes         |
| `ConcurrentLinkedQueue`  | No      | No       | FIFO        | Yes         |
| `SynchronousQueue`       | 0       | Yes      | N/A         | Yes         |

_* grows dynamically_

---

## 3. Memory Overhead Comparison

```java
// Memory per entry (approximate, 64-bit JVM, compressed oops)

// ArrayList<Integer>:
//   Array slot: 4 bytes (reference)
//   Integer object: 16 bytes (12 header + 4 int)
//   Total: ~20 bytes per element

// LinkedList<Integer>:
//   Node object: 24 bytes (12 header + 4 item ref + 4 prev + 4 next)
//   Integer object: 16 bytes
//   Total: ~40 bytes per element (2x ArrayList)

// HashMap<String, String>:
//   Entry object: 32 bytes (12 header + 4 hash + 4 key + 4 value + 4 next + padding)
//   Key String: ~48 bytes (depending on length)
//   Value String: ~48 bytes
//   Array slot: 4 bytes (reference in bucket array)
//   Total: ~132 bytes per entry

// int[] (primitive array):
//   4 bytes per element (no object overhead!)
//   1 million ints: ~4 MB

// ArrayList<Integer>:
//   ~20 bytes per element
//   1 million ints: ~20 MB (5x more than int[])
```

```java
// Measuring actual memory usage
public class MemoryMeasurement {
    public static long getObjectSize() {
        Runtime runtime = Runtime.getRuntime();
        runtime.gc();
        long before = runtime.totalMemory() - runtime.freeMemory();

        // Create structure
        Map<Integer, String> map = new HashMap<>();
        for (int i = 0; i < 100_000; i++) {
            map.put(i, "value_" + i);
        }

        runtime.gc();
        long after = runtime.totalMemory() - runtime.freeMemory();
        return after - before; // approximate memory used
    }
}
```

---

## 4. Concurrent vs Non-Concurrent

```java
// Non-concurrent: use when single-threaded or externally synchronized
List<String> list = new ArrayList<>();
Map<String, Integer> map = new HashMap<>();
Set<String> set = new HashSet<>();

// Synchronized wrappers: simple but coarse-grained locking
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
// WARNING: iteration still needs external synchronization!
synchronized (syncList) {
    for (String s : syncList) { /* ... */ }
}

// Concurrent collections: fine-grained or lock-free
ConcurrentHashMap<String, Integer> concMap = new ConcurrentHashMap<>();
CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();
ConcurrentLinkedQueue<String> clQueue = new ConcurrentLinkedQueue<>();
BlockingQueue<String> bQueue = new LinkedBlockingQueue<>(1000);
```

### Concurrent Collection Selection

| Scenario                           | Collection                    |
|------------------------------------|-------------------------------|
| Shared map, read + write           | `ConcurrentHashMap`           |
| Shared list, mostly reads          | `CopyOnWriteArrayList`        |
| Shared list, frequent writes       | `ConcurrentLinkedDeque`       |
| Producer-consumer (bounded)        | `ArrayBlockingQueue`          |
| Producer-consumer (unbounded)      | `LinkedBlockingQueue`         |
| Multiple producers, one consumer   | `ConcurrentLinkedQueue`       |
| Sorted concurrent map              | `ConcurrentSkipListMap`       |
| Sorted concurrent set              | `ConcurrentSkipListSet`       |

---

## 5. Specialized Collections

```java
// BitSet: compact boolean array (1 bit per element)
BitSet primes = new BitSet(1_000_000);
primes.set(0, 1_000_000); // all true initially
primes.clear(0);
primes.clear(1);
for (int i = 2; i * i < 1_000_000; i++) {
    if (primes.get(i)) {
        for (int j = i * i; j < 1_000_000; j += i) {
            primes.clear(j);
        }
    }
}
// 1 million booleans: ~125 KB (vs 1 MB for boolean[])

// EnumSet: bit-vector implementation for enum sets
Set<DayOfWeek> weekdays = EnumSet.range(DayOfWeek.MONDAY, DayOfWeek.FRIDAY);
// Entire set fits in a single long (64 bits)

// EnumMap: array-indexed by enum ordinal
Map<DayOfWeek, List<Meeting>> schedule = new EnumMap<>(DayOfWeek.class);
// Direct array access: no hashing, no collision handling

// WeakHashMap: entries can be garbage collected
Map<ClassLoader, Metadata> classMetadata = new WeakHashMap<>();
// Keys are weakly referenced; GC can collect when no other references exist

// IdentityHashMap: uses == instead of equals() for keys
Map<Object, String> identityMap = new IdentityHashMap<>();
// Useful for tracking object identity, not logical equality
```

---

## 6. Array-Based vs Pointer-Based

```java
// Array-based (contiguous memory): ArrayList, ArrayDeque, HashMap (bucket array)
// + Cache-friendly (spatial locality)
// + Low memory overhead
// + Fast random access
// - Expensive insert/delete in middle
// - Resize requires copy

// Pointer-based (linked nodes): LinkedList, TreeMap, LinkedBlockingQueue
// + O(1) insert/delete at known position
// + No resize needed
// - Cache-unfriendly (pointer chasing)
// - High memory overhead (node objects)
// - No random access

// RULE: Array-based structures almost always outperform pointer-based
// due to CPU cache effects, even for operations where linked structures
// have better theoretical complexity.
```

### Cache Effect Benchmark

```
Sequential iteration over 1 million integers:
  int[]:                   0.4 ms   (contiguous, perfect cache utilization)
  ArrayList<Integer>:      3.5 ms   (object references, some indirection)
  LinkedList<Integer>:     12 ms    (pointer chasing, cache misses)

Random access 100K times:
  ArrayList.get(i):        0.3 ms   (O(1), direct array offset)
  LinkedList.get(i):       2,800 ms (O(n), traverse from head each time)
```

---

## 7. Common Anti-Patterns

```java
// Anti-pattern 1: LinkedList for anything
// Almost NEVER use LinkedList. Use ArrayList or ArrayDeque instead.
Deque<String> stack = new ArrayDeque<>();     // NOT new LinkedList<>()
Queue<String> queue = new ArrayDeque<>();     // NOT new LinkedList<>()
List<String> list = new ArrayList<>();        // NOT new LinkedList<>()

// Anti-pattern 2: TreeMap when HashMap suffices
// Use TreeMap ONLY when you need sorted keys or range operations
Map<String, User> lookup = new HashMap<>();   // NOT new TreeMap<>()

// Anti-pattern 3: Vector or Hashtable (legacy synchronized)
List<String> list = new ArrayList<>();        // NOT new Vector<>()
Map<String, String> map = new HashMap<>();    // NOT new Hashtable<>()
// Use ConcurrentHashMap if thread safety is needed

// Anti-pattern 4: List.contains() for membership testing
Set<String> allowed = new HashSet<>(List.of("admin", "user", "mod"));
if (allowed.contains(role)) { ... }  // O(1) not O(n)

// Anti-pattern 5: Sorting an array to find min/max
int max = Collections.max(list);   // O(n) - scan once
// NOT: Collections.sort(list); int max = list.get(list.size()-1); // O(n log n)
```

---

## 8. Data Structure Selection Quick Reference

| Need                              | Best Choice                 | Why                          |
|-----------------------------------|-----------------------------|------------------------------|
| Fast lookup by key                | `HashMap`                   | O(1) average                 |
| Sorted key-value pairs            | `TreeMap`                   | O(log n), range queries      |
| Unique elements                   | `HashSet`                   | O(1) add/contains            |
| Ordered unique elements           | `LinkedHashSet`             | O(1) + insertion order       |
| Stack (LIFO)                      | `ArrayDeque`                | Faster than Stack/LinkedList |
| Queue (FIFO)                      | `ArrayDeque`                | Faster than LinkedList       |
| Priority queue                    | `PriorityQueue`             | O(log n) insert/remove min   |
| Thread-safe map                   | `ConcurrentHashMap`         | Fine-grained locking         |
| Thread-safe queue                 | `ArrayBlockingQueue`        | Bounded, blocking            |
| Boolean flags for integers        | `BitSet`                    | 1 bit per element            |
| Enum keys/values                  | `EnumMap` / `EnumSet`       | Array/bit-vector backed      |
| Cache with eviction               | `Caffeine Cache`            | W-TinyLFU eviction           |
| Large primitive collections       | Eclipse Collections         | No boxing overhead           |

---

## Interview Tips

- ArrayList beats LinkedList in nearly all real-world scenarios (cache locality)
- ArrayDeque is the best stack AND queue implementation
- HashMap is O(1) average but degrades to O(log n) worst case (tree bins since Java 8)
- EnumMap/EnumSet are the fastest Map/Set when keys are enum constants
- ConcurrentHashMap is the go-to for multi-threaded map access
- Never use Vector, Hashtable, or Stack (legacy classes, use modern alternatives)
- Memory overhead matters: LinkedList uses 10x more memory than ArrayList per element
- Always consider: Will this data structure fit in CPU cache for my dataset size?

---

[<< Previous: Algorithm Selection](./q13-algorithm-selection.md) | [Next: Startup Optimization >>](./q15-startup-optimization.md)

# Collection Performance Optimization

[Home](../../README.md) > [Performance Optimization](./README.md) > Collection Performance

---

## Overview

Choosing the right collection and configuring it properly can drastically impact performance. Understanding time complexity, memory overhead, and internal implementation details is essential for writing high-performance Java code.

---

## 1. Initial Capacity and Load Factor

```java
// BAD: Default capacity causes multiple resizes
Map<String, User> users = new HashMap<>();  // default capacity: 16
// Adding 10,000 entries causes ~10 resize operations (16 -> 32 -> 64 -> ... -> 16384)
// Each resize: allocate new array + rehash all entries

// GOOD: Pre-sized to avoid resizing
// Formula: expectedSize / loadFactor + 1
int expectedSize = 10_000;
Map<String, User> users = new HashMap<>(expectedSize * 4 / 3 + 1);
// Or simpler: just pass expected size (HashMap rounds to power of 2)
Map<String, User> users = new HashMap<>(expectedSize);

// GOOD: Pre-sized list
List<String> names = new ArrayList<>(10_000); // avoids grow + copy

// Java 19+: factory methods for known sizes
Map<String, User> users = HashMap.newHashMap(10_000); // exact sizing
List<String> names = ArrayList.newArrayList(10_000);
```

### Resize Cost Visualization

```
HashMap resize triggers when: size > capacity * loadFactor (default 0.75)

Capacity:  16  ->  32  ->  64  -> 128 -> 256 -> ... -> 16384
Threshold: 12  ->  24  ->  48  ->  96 -> 192 -> ... -> 12288
Cost:      O(n) rehash at each resize

Pre-sizing to 16384 eliminates all intermediate resizes.
```

---

## 2. Right Collection for the Job

### HashMap vs TreeMap vs LinkedHashMap

```java
// HashMap: O(1) average lookup - MOST COMMON CHOICE
Map<String, Integer> wordCount = new HashMap<>();
// Best for: fast lookup by key, no ordering needed

// TreeMap: O(log n) lookup - SORTED ORDER
Map<String, Integer> sortedWords = new TreeMap<>();
// Best for: range queries, sorted iteration, navigable operations
NavigableMap<Integer, String> scores = new TreeMap<>();
scores.subMap(80, 100); // all scores between 80-99

// LinkedHashMap: O(1) lookup + insertion order
Map<String, Integer> accessOrder = new LinkedHashMap<>(16, 0.75f, true);
// Best for: LRU cache implementation, predictable iteration order
```

### Performance Comparison

| Operation      | HashMap   | TreeMap     | LinkedHashMap |
|----------------|-----------|-------------|---------------|
| `put()`        | O(1) avg  | O(log n)    | O(1) avg      |
| `get()`        | O(1) avg  | O(log n)    | O(1) avg      |
| `remove()`     | O(1) avg  | O(log n)    | O(1) avg      |
| `containsKey()`| O(1) avg  | O(log n)    | O(1) avg      |
| Iteration      | O(capacity)| O(n)       | O(n)          |
| Memory/entry   | ~48 bytes | ~56 bytes   | ~64 bytes     |

---

## 3. ArrayList vs LinkedList

**ArrayList almost always wins.** This is one of the most important performance facts in Java.

```java
// ArrayList: contiguous memory, cache-friendly
List<Integer> arrayList = new ArrayList<>();

// LinkedList: scattered nodes, cache-unfriendly
List<Integer> linkedList = new LinkedList<>();
```

### Benchmark Results (1 million elements)

```
Operation              ArrayList    LinkedList    Winner
------------------------------------------------------------------
Sequential add         15 ms        25 ms        ArrayList
Random get(index)      0.01 ms      500 ms       ArrayList (O(1) vs O(n))
Sequential iteration   5 ms         8 ms         ArrayList (cache locality)
Add at beginning       800 ms       0.02 ms      LinkedList
Add in middle          400 ms       0.02 ms*     LinkedList*
Remove from middle     400 ms       0.02 ms*     LinkedList*

* LinkedList insertion is O(1) ONLY if you already have the node reference.
  Finding the position is O(n), making total O(n) in practice.
```

```java
// When LinkedList wins: Queue/Deque operations
Deque<Task> taskQueue = new ArrayDeque<>(); // STILL better than LinkedList!

// ArrayDeque is faster than LinkedList for both stack and queue usage
// LinkedList has virtually NO practical advantage over ArrayList/ArrayDeque
```

### Memory Overhead Comparison

```
ArrayList:  ~4 bytes per element (just the reference in the array)
LinkedList: ~48 bytes per element (Node object: prev + next + item + header)

1 million elements:
  ArrayList:  ~4 MB
  LinkedList: ~48 MB (12x more memory!)
```

---

## 4. EnumSet and EnumMap for Enum Keys

```java
public enum Permission {
    READ, WRITE, EXECUTE, DELETE, ADMIN
}

// BAD: HashSet for enums
Set<Permission> permissions = new HashSet<>();
permissions.add(Permission.READ);
permissions.add(Permission.WRITE);

// GOOD: EnumSet - backed by bit vector, extremely fast
Set<Permission> permissions = EnumSet.of(Permission.READ, Permission.WRITE);
// Operations are bitwise: O(1) and no hashing!

// BAD: HashMap for enum keys
Map<Permission, String> descriptions = new HashMap<>();

// GOOD: EnumMap - backed by array indexed by ordinal
Map<Permission, String> descriptions = new EnumMap<>(Permission.class);
descriptions.put(Permission.READ, "Can read files");
// Array access: O(1) with no hashing, minimal memory
```

### Performance Comparison

```
Set operations (contains, add, remove):
  HashSet<Permission>: ~20 ns per operation
  EnumSet<Permission>: ~3 ns per operation  (6-7x faster)

Map operations (get, put):
  HashMap<Permission, V>: ~25 ns per operation
  EnumMap<Permission, V>:  ~5 ns per operation  (5x faster)

Memory:
  HashSet (5 elements): ~320 bytes
  EnumSet (5 elements): ~24 bytes  (13x less memory)
```

---

## 5. Primitive Collections

Standard Java collections box primitives, wasting memory and CPU.

```java
// BAD: Autoboxing overhead
Map<Integer, Double> scores = new HashMap<>();
for (int i = 0; i < 1_000_000; i++) {
    scores.put(i, (double) i * 1.5);   // autoboxing int -> Integer, double -> Double
}
// Memory: ~72 bytes per entry (Integer + Double + Node + pointers)

// GOOD: Eclipse Collections primitive maps
import org.eclipse.collections.impl.map.mutable.primitive.IntDoubleHashMap;

IntDoubleHashMap scores = new IntDoubleHashMap(1_000_000);
for (int i = 0; i < 1_000_000; i++) {
    scores.put(i, i * 1.5);   // no boxing!
}
// Memory: ~20 bytes per entry (raw int + raw double + overhead)
```

### Primitive Collection Libraries

```xml
<!-- Eclipse Collections -->
<dependency>
    <groupId>org.eclipse.collections</groupId>
    <artifactId>eclipse-collections</artifactId>
    <version>11.1.0</version>
</dependency>
```

```java
// Eclipse Collections examples
import org.eclipse.collections.api.list.primitive.IntList;
import org.eclipse.collections.impl.list.mutable.primitive.IntArrayList;
import org.eclipse.collections.impl.set.mutable.primitive.LongHashSet;

// Primitive list (no boxing)
IntList numbers = IntArrayList.newListWith(1, 2, 3, 4, 5);
int sum = numbers.sum();  // primitive operations

// Primitive set
LongHashSet ids = new LongHashSet(1_000_000);
ids.add(12345L);
boolean contains = ids.contains(12345L); // no Long boxing

// Memory savings:
// ArrayList<Integer> (1M elements): ~20 MB
// IntArrayList       (1M elements): ~4 MB  (5x less!)
```

---

## 6. Collection Selection Decision Guide

```
Need key-value lookup?
  ├── Yes
  │   ├── Keys are enums? -> EnumMap
  │   ├── Keys are primitives? -> Eclipse Collections primitive map
  │   ├── Need sorted order? -> TreeMap
  │   ├── Need insertion order? -> LinkedHashMap
  │   ├── Concurrent access? -> ConcurrentHashMap
  │   └── General purpose -> HashMap
  └── No
      ├── Need uniqueness?
      │   ├── Yes
      │   │   ├── Elements are enums? -> EnumSet
      │   │   ├── Need sorted order? -> TreeSet
      │   │   ├── Concurrent? -> ConcurrentSkipListSet / CopyOnWriteArraySet
      │   │   └── General purpose -> HashSet
      │   └── No
      │       ├── Need FIFO/LIFO? -> ArrayDeque
      │       ├── Concurrent queue? -> ConcurrentLinkedQueue / LinkedBlockingQueue
      │       ├── Priority ordering? -> PriorityQueue
      │       └── General purpose -> ArrayList
```

---

## 7. Iteration Performance

```java
// Fastest: for-each (or indexed for ArrayList)
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));

// Good: enhanced for-each
for (String s : list) { process(s); }

// Good for ArrayList: indexed loop (avoids iterator object)
for (int i = 0, n = list.size(); i < n; i++) {
    process(list.get(i));
}

// Slower: Stream (overhead of pipeline setup)
list.stream().forEach(s -> process(s));

// Parallel stream: only for large datasets + CPU-bound operations
list.parallelStream()
    .filter(s -> s.length() > 3)
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// Map iteration: entrySet() is fastest
Map<String, Integer> map = new HashMap<>();
// GOOD
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    process(entry.getKey(), entry.getValue());
}
// BAD: double lookup
for (String key : map.keySet()) {
    process(key, map.get(key)); // extra hash lookup!
}
```

---

## 8. Common Collection Anti-Patterns

```java
// Anti-pattern 1: Using List.contains() for lookups (O(n))
List<String> allowedUsers = List.of("alice", "bob", "charlie");
if (allowedUsers.contains(username)) { ... } // O(n) scan

// Fix: Use HashSet (O(1))
Set<String> allowedUsers = Set.of("alice", "bob", "charlie");
if (allowedUsers.contains(username)) { ... } // O(1) lookup

// Anti-pattern 2: Removing from list while iterating
List<String> items = new ArrayList<>(Arrays.asList("a", "b", "c"));
// ConcurrentModificationException!
for (String item : items) {
    if (shouldRemove(item)) items.remove(item);
}
// Fix: Use removeIf()
items.removeIf(item -> shouldRemove(item));

// Anti-pattern 3: Creating unnecessary intermediate collections
list.stream()
    .collect(Collectors.toList())    // intermediate list (unnecessary)
    .stream()
    .filter(x -> x > 0)
    .collect(Collectors.toList());
// Fix: single pipeline
list.stream().filter(x -> x > 0).collect(Collectors.toList());
```

---

## Interview Tips

- Always pre-size collections when the expected size is known
- ArrayList beats LinkedList in almost every real-world scenario due to cache locality
- EnumSet/EnumMap are the fastest collections for enum-typed keys
- Use primitive collections (Eclipse Collections) to avoid boxing overhead
- `HashMap.get()` is O(1) average but O(log n) worst case (tree bins since Java 8)
- Iterate maps with `entrySet()`, never `keySet()` + `get()`
- `ArrayDeque` is faster than `LinkedList` for both stack and queue operations

---

[<< Previous: Concurrency Optimization](./q04-concurrency-optimization.md) | [Next: String Optimization >>](./q06-string-optimization.md)

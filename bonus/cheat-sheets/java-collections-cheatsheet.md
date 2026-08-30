# Java Collections Framework - Cheat Sheet

Complete quick reference for Java Collections with performance characteristics and examples.

## 📋 Collections Hierarchy

```
Collection
├── List (ordered, allows duplicates)
│   ├── ArrayList - O(1) access, O(n) insert/delete
│   ├── LinkedList - O(n) access, O(1) insert/delete at ends
│   └── CopyOnWriteArrayList - thread-safe, expensive writes
├── Set (unordered, no duplicates)
│   ├── HashSet - O(1) average, unordered
│   ├── LinkedHashSet - O(1) average, insertion order
│   ├── TreeSet - O(log n), sorted
│   └── ConcurrentHashMap.newKeySet() - thread-safe
└── Queue (FIFO/Priority)
    ├── LinkedList - FIFO, O(1) operations
    ├── PriorityQueue - heap-based, O(log n)
    └── ConcurrentLinkedQueue - thread-safe

Map (key-value pairs)
├── HashMap - O(1) average, unordered
├── LinkedHashMap - O(1) average, insertion order
├── TreeMap - O(log n), sorted by key
├── WeakHashMap - weak references to keys
├── IdentityHashMap - uses == instead of equals()
├── ConcurrentHashMap - thread-safe, segment locking
├── Hashtable - synchronized (legacy)
└── Collections.synchronizedMap() - synchronized wrapper
```

## 🚀 List Interface

### ArrayList
```java
// Creation and basic operations
List<String> list = new ArrayList<>();
list.add("item1");                          // O(1) amortized
list.add(0, "item0");                       // O(n)
list.get(0);                                // O(1)
list.remove(0);                             // O(n)
list.set(0, "newValue");                    // O(1)

// Iteration (fastest)
for (String item : list) { }                // Enhanced for loop

// Capacity management
ArrayList<String> list2 = new ArrayList<>(100);  // Pre-allocate
list.trimToSize();                          // Reduce capacity

// Performance: O(1) access, O(n) insert/delete
// Memory: Contiguous array, ~10% overhead
```

### LinkedList
```java
// Creation and operations
LinkedList<String> list = new LinkedList<>();
list.add("item");                           // O(1)
list.addFirst("first");                     // O(1)
list.addLast("last");                       // O(1)
list.getFirst();                            // O(1)
list.removeFirst();                         // O(1)
list.removeLast();                          // O(1)

// Queue operations
list.offer("item");                         // add to end, O(1)
list.poll();                                // remove from front, O(1)
list.peek();                                // view front, O(1)

// Performance: O(n) access, O(1) insert/delete at ends
// Use case: Queue, Deque operations
```

### CopyOnWriteArrayList
```java
// Thread-safe list for read-heavy workloads
List<String> list = new CopyOnWriteArrayList<>();
list.add("item");                           // O(n) - copies array
list.get(0);                                // O(1) - no lock

// Best for: Few writes, many reads
// Worst for: Frequent modifications
```

## 🎯 Set Interface

### HashSet
```java
// Unordered, no duplicates
Set<String> set = new HashSet<>();
set.add("item1");                           // O(1) average
set.contains("item1");                      // O(1) average
set.remove("item1");                        // O(1) average

// Iteration order is unpredictable
for (String item : set) { }

// Performance: O(1) average, O(n) worst case
// Load factor: 0.75 (resize at 75% capacity)
```

### LinkedHashSet
```java
// Maintains insertion order
Set<String> set = new LinkedHashSet<>();
set.add("first");
set.add("second");
// Iteration: first, second (insertion order)

// Performance: O(1) average, slightly slower than HashSet
// Use case: When order matters but no sorting needed
```

### TreeSet
```java
// Sorted set, implements NavigableSet
Set<Integer> set = new TreeSet<>();
set.add(3);
set.add(1);
set.add(2);
// Iteration: 1, 2, 3 (sorted)

// NavigableSet operations
set.floor(2);                               // ≤ 2 → 2
set.ceiling(2);                             // ≥ 2 → 2
set.lower(2);                               // < 2 → 1
set.higher(2);                              // > 2 → 3
set.headSet(2);                             // elements < 2
set.tailSet(2);                             // elements ≥ 2
set.subSet(1, 3);                           // 1 ≤ x < 3

// Custom comparator
Set<String> set2 = new TreeSet<>(
    Comparator.reverseOrder()
);

// Performance: O(log n) all operations
// Use case: Sorted unique elements, range queries
```

## 🗺️ Map Interface

### HashMap
```java
// Unordered key-value pairs
Map<String, Integer> map = new HashMap<>();
map.put("key1", 100);                       // O(1) average
map.get("key1");                            // O(1) average
map.remove("key1");                         // O(1) average
map.containsKey("key1");                    // O(1) average
map.getOrDefault("key2", 0);                // O(1) average

// Iteration
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    String key = entry.getKey();
    Integer value = entry.getValue();
}

// Performance: O(1) average, O(n) worst case
// Null handling: One null key, multiple null values allowed
```

### LinkedHashMap
```java
// Maintains insertion order
Map<String, Integer> map = new LinkedHashMap<>();
map.put("first", 1);
map.put("second", 2);
// Iteration: first, second (insertion order)

// Access-order LinkedHashMap (LRU cache)
Map<String, Integer> lru = new LinkedHashMap<String, Integer>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > 100;                // Keep max 100 entries
    }
};

// Performance: O(1) average, slightly slower than HashMap
```

### TreeMap
```java
// Sorted by key, implements NavigableMap
Map<Integer, String> map = new TreeMap<>();
map.put(3, "three");
map.put(1, "one");
map.put(2, "two");
// Iteration: 1→one, 2→two, 3→three (sorted)

// NavigableMap operations
map.floorKey(2);                            // ≤ 2 → 2
map.ceilingKey(2);                          // ≥ 2 → 2
map.lowerKey(2);                            // < 2 → 1
map.higherKey(2);                           // > 2 → 3
map.headMap(2);                             // keys < 2
map.tailMap(2);                             // keys ≥ 2
map.subMap(1, 3);                           // 1 ≤ k < 3

// Custom comparator
Map<String, Integer> map2 = new TreeMap<>(
    Comparator.reverseOrder()
);

// Performance: O(log n) all operations
```

### ConcurrentHashMap
```java
// Thread-safe without full synchronization
Map<String, Integer> map = new ConcurrentHashMap<>();
map.put("key1", 100);                       // O(1) average
map.get("key1");                            // O(1) average
map.putIfAbsent("key2", 200);               // O(1) average

// Atomic operations
map.computeIfAbsent("key3", k -> 300);      // O(1) average
map.computeIfPresent("key1", (k, v) -> v + 1);

// Iteration is weakly consistent (safe but may miss updates)
for (Map.Entry<String, Integer> entry : map.entrySet()) { }

// Performance: O(1) average, segment-based locking
// Use case: Multi-threaded applications
```

## 📊 Performance Comparison Table

| Operation | ArrayList | LinkedList | HashSet | TreeSet | HashMap | TreeMap |
|-----------|-----------|-----------|---------|---------|---------|---------|
| Access | O(1) | O(n) | - | - | O(1) | O(log n) |
| Insert | O(n) | O(1)* | O(1) | O(log n) | O(1) | O(log n) |
| Delete | O(n) | O(1)* | O(1) | O(log n) | O(1) | O(log n) |
| Search | O(n) | O(n) | O(1) | O(log n) | O(1) | O(log n) |
| Space | O(n) | O(n) | O(n) | O(n) | O(n) | O(n) |

*LinkedList: O(1) at ends, O(n) in middle

## 🔄 Queue Interface

### LinkedList as Queue
```java
Queue<String> queue = new LinkedList<>();
queue.offer("item1");                       // Add to end
queue.poll();                               // Remove from front
queue.peek();                               // View front

// Throws exception vs returns null
queue.add("item");                          // Throws exception if full
queue.offer("item");                        // Returns false if full
queue.remove();                             // Throws exception if empty
queue.poll();                               // Returns null if empty
queue.element();                            // Throws exception if empty
queue.peek();                               // Returns null if empty
```

### PriorityQueue
```java
// Min-heap by default
Queue<Integer> pq = new PriorityQueue<>();
pq.offer(3);
pq.offer(1);
pq.offer(2);
pq.poll();                                  // 1
pq.poll();                                  // 2
pq.poll();                                  // 3

// Max-heap with custom comparator
Queue<Integer> maxPq = new PriorityQueue<>(
    Comparator.reverseOrder()
);

// Performance: O(log n) insert/delete, O(1) peek
// Use case: Dijkstra's algorithm, task scheduling
```

## 🛡️ Thread-Safe Collections

```java
// Synchronized wrappers (legacy)
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
Set<String> syncSet = Collections.synchronizedSet(new HashSet<>());

// Concurrent collections (preferred)
List<String> list = new CopyOnWriteArrayList<>();
Map<String, Integer> map = new ConcurrentHashMap<>();
Queue<String> queue = new ConcurrentLinkedQueue<>();
Set<String> set = ConcurrentHashMap.newKeySet();

// Immutable collections (thread-safe by nature)
List<String> immutable = Collections.unmodifiableList(new ArrayList<>());
Map<String, Integer> immutable2 = Collections.unmodifiableMap(new HashMap<>());
```

## ⚡ Common Patterns

### Counting Occurrences
```java
Map<String, Integer> counts = new HashMap<>();
for (String item : items) {
    counts.put(item, counts.getOrDefault(item, 0) + 1);
}

// Or with merge
for (String item : items) {
    counts.merge(item, 1, Integer::sum);
}
```

### Group By
```java
Map<String, List<String>> grouped = new HashMap<>();
for (String item : items) {
    grouped.computeIfAbsent(category(item), k -> new ArrayList<>())
           .add(item);
}
```

### LRU Cache
```java
Map<String, Integer> cache = new LinkedHashMap<String, Integer>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > 100;
    }
};
```

## ⚠️ Common Pitfalls

1. **ConcurrentModificationException**: Don't modify collection while iterating
   ```java
   // ❌ Wrong
   for (String item : list) {
       if (item.equals("remove")) list.remove(item);
   }
   
   // ✅ Correct
   Iterator<String> it = list.iterator();
   while (it.hasNext()) {
       if (it.next().equals("remove")) it.remove();
   }
   ```

2. **HashMap with mutable keys**: Keys must be immutable
   ```java
   // ❌ Wrong - StringBuilder is mutable
   Map<StringBuilder, Integer> map = new HashMap<>();
   
   // ✅ Correct - String is immutable
   Map<String, Integer> map = new HashMap<>();
   ```

3. **Null handling**: Different collections handle nulls differently
   ```java
   // HashMap allows nulls
   HashMap<String, Integer> map = new HashMap<>();
   map.put(null, 1);
   
   // TreeMap doesn't allow null keys
   TreeMap<String, Integer> map2 = new TreeMap<>();
   map2.put(null, 1);  // NullPointerException
   ```

## 🎓 Interview Tips

- **Know the complexity**: Be able to explain O(1) vs O(n) operations
- **Understand trade-offs**: ArrayList vs LinkedList, HashMap vs TreeMap
- **Thread safety**: Know which collections are thread-safe
- **Choose wisely**: Pick the right collection for the use case
- **Memory efficiency**: Consider space complexity too

---

**Related**: See Stream API cheat sheet for functional operations on collections

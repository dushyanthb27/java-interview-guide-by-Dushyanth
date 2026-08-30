# Q9: How Does ConcurrentHashMap Work? (Deep Dive)

[Back to Collections Framework](./README.md) | **Difficulty**: 🔴 Expert

---

## Question
> How does ConcurrentHashMap work internally? How does it achieve thread-safety without locking the entire map?

---

## Answer

### Evolution: Java 7 vs Java 8+

```java
// Java 7 ConcurrentHashMap:
// - Segment-based locking (16 segments by default)
// - Each segment = mini HashMap with its own lock
// - Concurrency level = number of segments
// - Up to 16 threads can write simultaneously (one per segment)

// Java 8+ ConcurrentHashMap (current):
// - No more segments!
// - CAS (Compare-And-Swap) + synchronized per NODE
// - Fine-grained locking at individual bucket level
// - Much better scalability under high concurrency
```

### Java 8+ Internal Structure

```java
// Simplified internal structure
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
        implements ConcurrentMap<K,V>, Serializable {

    transient volatile Node<K,V>[] table;  // volatile for visibility
    private transient volatile int sizeCtl; // controls initialization and resizing

    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;       // volatile for visibility
        volatile Node<K,V> next; // volatile for visibility
    }

    // TreeBin wraps TreeNodes when bucket is treeified
    static final class TreeBin<K,V> extends Node<K,V> {
        TreeNode<K,V> root;
        volatile TreeNode<K,V> first;
        volatile Thread waiter;
        volatile int lockState;
    }
}
```

### CAS + Synchronized Per Node

```java
// put() operation (simplified from OpenJDK source):
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // ❌ Unlike HashMap, null keys and values are NOT allowed
    if (key == null || value == null) throw new NullPointerException();

    int hash = spread(key.hashCode());

    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;

        if (tab == null || (n = tab.length) == 0) {
            tab = initTable();  // CAS-based lazy initialization
        }
        // CASE 1: Bucket is empty -> CAS to insert new Node (no lock!)
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                break;  // CAS succeeded - no locking needed!
        }
        // CASE 2: Bucket is being resized (MOVED marker)
        else if ((fh = f.hash) == MOVED) {
            tab = helpTransfer(tab, f);  // Help with resize
        }
        // CASE 3: Bucket has entries -> synchronized on HEAD node
        else {
            synchronized (f) {  // Lock ONLY this bucket's head node
                // ... traverse and insert/update ...
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

### Visual: Concurrent Access Pattern

```
Thread 1: put("A", 1)          Thread 2: put("B", 2)
    |                               |
    v                               v
  hash("A") -> bucket[3]         hash("B") -> bucket[7]
    |                               |
    v                               v
  CAS or sync(bucket[3])         CAS or sync(bucket[7])
    |                               |
    v                               v
  SUCCESS (no contention!)        SUCCESS (no contention!)

Thread 3: put("C", 3)  ->  hash("C") -> bucket[3]  (same bucket as "A"!)
    |
    v
  synchronized(bucket[3].head)  // waits for Thread 1 if still working
    |
    v
  Append to chain in bucket[3]
```

### No Null Keys or Values (Why?)

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// ❌ Both null key and null value throw NullPointerException
map.put(null, 1);        // NullPointerException!
map.put("key", null);    // NullPointerException!

// WHY? Ambiguity in concurrent context:
// map.get("key") returns null
// Does it mean:
//   (a) key doesn't exist?     <- could be true
//   (b) value is null?         <- could also be true
// In a concurrent map, another thread might add/remove the key
// between containsKey() and get() calls, making it impossible
// to distinguish between the two cases safely.

// HashMap allows null because it's single-threaded:
// You can safely call containsKey() then get() without interference
```

### Atomic Operations (ConcurrentMap Interface)

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// ✅ putIfAbsent - atomic check-and-insert
map.putIfAbsent("counter", 0);
// Equivalent to (but atomic):
// if (!map.containsKey("counter")) { map.put("counter", 0); }

// ✅ computeIfAbsent - atomic compute-if-missing
map.computeIfAbsent("list", key -> new ArrayList<>());
// Thread-safe lazy initialization of values

// ✅ computeIfPresent - atomic update-if-exists
map.computeIfPresent("counter", (key, val) -> val + 1);

// ✅ compute - atomic compute (insert, update, or remove)
map.compute("counter", (key, val) -> val == null ? 1 : val + 1);

// ✅ merge - atomic merge operation
map.merge("counter", 1, Integer::sum);  // Atomic increment
// Equivalent to: map.put("counter", map.getOrDefault("counter", 0) + 1)

// ✅ replace - atomic conditional replace
map.replace("key", "oldValue", "newValue");
// Only replaces if current value equals "oldValue"
```

### Bulk Operations (Java 8+)

```java
ConcurrentHashMap<String, Integer> scores = new ConcurrentHashMap<>();
scores.put("Alice", 85);
scores.put("Bob", 92);
scores.put("Charlie", 78);
scores.put("Diana", 95);

// forEach with parallelism threshold
// If map has more entries than threshold, operations run in parallel
long parallelismThreshold = 1; // 1 = always parallel, Long.MAX_VALUE = sequential

scores.forEach(parallelismThreshold, (key, value) -> {
    System.out.println(key + " = " + value);
});

// search - find first match (returns as soon as non-null result found)
String topStudent = scores.search(parallelismThreshold, (key, value) -> {
    return value >= 90 ? key : null;
});
// Could return "Bob" or "Diana" (non-deterministic in parallel)

// reduce - aggregate values
int totalScore = scores.reduce(parallelismThreshold,
    (key, value) -> value,    // transformation
    Integer::sum               // reduction
);
// totalScore = 350

// reduceValues
int maxScore = scores.reduceValues(parallelismThreshold, Integer::max);
// maxScore = 95

// forEach with transformation
scores.forEach(parallelismThreshold,
    (key, value) -> key + "=" + value,  // transformer
    System.out::println                  // action
);

// reduceKeys
String allNames = scores.reduceKeys(parallelismThreshold,
    (a, b) -> a + ", " + b
);
// "Alice, Bob, Charlie, Diana"
```

### Size Estimation

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// size() returns int - can overflow for very large maps
int size = map.size();  // Accurate but limited to Integer.MAX_VALUE

// ✅ mappingCount() returns long - preferred for large maps
long count = map.mappingCount();

// Both return an ESTIMATE in concurrent scenarios
// Because entries can be added/removed during counting
// The count is exact only if no concurrent modifications occur
```

### ConcurrentHashMap vs Alternatives

```java
// Option 1: Collections.synchronizedMap
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
// - Wraps HashMap with synchronized on EVERY method
// - ENTIRE map is locked for each operation
// - Must manually synchronize iteration:
synchronized (syncMap) {
    for (Map.Entry<String, Integer> e : syncMap.entrySet()) { /* ... */ }
}
// - Allows null keys and values
// - Poor concurrency: one thread at a time

// Option 2: Hashtable (legacy)
Map<String, Integer> hashtable = new Hashtable<>();
// - Every method is synchronized (like Vector for lists)
// - Same lock contention issues as synchronizedMap
// - No null keys or values
// - Legacy - NEVER use in new code

// Option 3: ConcurrentHashMap (modern)
ConcurrentHashMap<String, Integer> chm = new ConcurrentHashMap<>();
// - CAS + per-node synchronization
// - Multiple threads can read AND write simultaneously
// - No null keys or values
// - Weakly consistent iterators (no ConcurrentModificationException)
// - Atomic compound operations (computeIfAbsent, merge, etc.)
// - Bulk parallel operations (forEach, search, reduce)
```

### Weakly Consistent Iterators

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("A", 1);
map.put("B", 2);
map.put("C", 3);

// ✅ Iterator is weakly consistent:
// - Does NOT throw ConcurrentModificationException
// - May or may not reflect concurrent modifications
// - Guaranteed to traverse each element at most once
Iterator<Map.Entry<String, Integer>> it = map.entrySet().iterator();

// Another thread adds: map.put("D", 4);
// The iterator MAY or MAY NOT see "D"

// This is safe - no exception!
while (it.hasNext()) {
    Map.Entry<String, Integer> entry = it.next();
    System.out.println(entry);
}

// Compare with HashMap:
// HashMap iterator is FAIL-FAST -> ConcurrentModificationException
```

### Real-World Concurrent Patterns

```java
// ✅ Pattern 1: Thread-safe cache with computeIfAbsent
ConcurrentHashMap<String, ExpensiveObject> cache = new ConcurrentHashMap<>();

public ExpensiveObject getOrCreate(String key) {
    return cache.computeIfAbsent(key, k -> {
        // This lambda executes atomically for this key
        // Only ONE thread will compute the value for a given key
        return new ExpensiveObject(k);
    });
}

// ✅ Pattern 2: Concurrent word counter
ConcurrentHashMap<String, LongAdder> wordCounts = new ConcurrentHashMap<>();

public void countWord(String word) {
    wordCounts.computeIfAbsent(word, k -> new LongAdder()).increment();
}

// ✅ Pattern 3: Concurrent set (no ConcurrentHashSet exists!)
Set<String> concurrentSet = ConcurrentHashMap.newKeySet();
// Or:
Set<String> concurrentSet2 = new ConcurrentHashMap<String, Boolean>().keySet(true);

concurrentSet.add("item");
concurrentSet.contains("item");
concurrentSet.remove("item");

// ✅ Pattern 4: Registration/Subscription pattern
ConcurrentHashMap<String, List<EventListener>> listeners = new ConcurrentHashMap<>();

public void register(String event, EventListener listener) {
    listeners.computeIfAbsent(event, k -> new CopyOnWriteArrayList<>())
             .add(listener);
}
```

### Common Anti-Patterns

```java
// ❌ Check-then-act without atomic operation
if (!map.containsKey("key")) {   // Thread A checks
    map.put("key", value);        // Thread B might insert between these lines!
}
// ✅ Use putIfAbsent or computeIfAbsent instead

// ❌ Non-atomic read-modify-write
Integer count = map.get("key");
map.put("key", count + 1);  // Lost update! Another thread may increment too
// ✅ Use merge or compute instead
map.merge("key", 1, Integer::sum);

// ❌ Assuming size() is exact under concurrency
if (map.size() == 0) {  // Another thread might add an entry right now
    // Don't assume map is still empty here!
}

// ❌ Using ConcurrentHashMap where HashMap suffices
// ConcurrentHashMap has overhead - don't use it for single-threaded code
Map<String, Integer> singleThreaded = new HashMap<>();  // ✅ Faster
```

### Best Practices

```java
// ✅ Use ConcurrentHashMap for shared mutable maps in multithreaded code
ConcurrentHashMap<String, Data> shared = new ConcurrentHashMap<>();

// ✅ Use atomic operations for compound actions
shared.computeIfAbsent("key", k -> expensiveCreate(k));
shared.merge("counter", 1, Integer::sum);

// ✅ Pre-size when possible (avoids concurrent resizing)
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>(expectedSize);

// ✅ Use LongAdder/LongAccumulator for concurrent counters
ConcurrentHashMap<String, LongAdder> counters = new ConcurrentHashMap<>();

// ✅ Use newKeySet() for concurrent Set
Set<String> concurrentSet = ConcurrentHashMap.newKeySet();

// ✅ Prefer ConcurrentHashMap over synchronizedMap for concurrent access
```

---

## Performance Summary

| Aspect | synchronizedMap | Hashtable | ConcurrentHashMap |
|--------|----------------|-----------|-------------------|
| Locking | Entire map | Entire map | Per-node/CAS |
| Read concurrency | 1 thread | 1 thread | N threads |
| Write concurrency | 1 thread | 1 thread | N threads* |
| Null keys/values | Yes | No | No |
| Iterator | Fail-fast | Fail-fast | Weakly consistent |
| Atomic ops | No | No | Yes |
| Java version | 1.2 | 1.0 | 1.5 (rewritten in 8) |

\* Multiple threads can write to different buckets simultaneously

---

## Interview Tips
- Know the evolution: Java 7 (segments) vs Java 8+ (CAS + per-node sync)
- Explain CAS (Compare-And-Swap) as a lock-free operation
- Know why null keys/values are not allowed (ambiguity in concurrent context)
- Understand weakly consistent iterators vs fail-fast
- Know atomic operations: computeIfAbsent, merge, putIfAbsent
- Be ready to implement a concurrent cache using ConcurrentHashMap
- Mention bulk operations (forEach, reduce, search) with parallelism threshold

---

[Prev: Q8 - LinkedHashMap and TreeMap](./q08-linkedhashmap-treemap.md) | [Next: Q10 - Queue and Deque](./q10-queue-deque.md)

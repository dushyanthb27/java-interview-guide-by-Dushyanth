# Q8: Compare LinkedHashMap and TreeMap

[Back to Collections Framework](./README.md) | **Difficulty**: 🟡 Medium

---

## Question
> Compare LinkedHashMap and TreeMap. When would you use each?

---

## Answer

### LinkedHashMap - HashMap + Insertion/Access Order

LinkedHashMap extends HashMap and maintains a **doubly-linked list** across all entries to preserve order.

```java
// Simplified internal structure
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V> {

    // Each entry has additional links for order maintenance
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before;  // previous in insertion/access order
        Entry<K,V> after;   // next in insertion/access order
    }

    transient LinkedHashMap.Entry<K,V> head;  // oldest entry
    transient LinkedHashMap.Entry<K,V> tail;  // newest entry
    final boolean accessOrder;  // false = insertion order, true = access order
}
```

### LinkedHashMap - Insertion Order (Default)

```java
Map<String, Integer> map = new LinkedHashMap<>();
map.put("Charlie", 3);
map.put("Alice", 1);
map.put("Bob", 2);

for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}
// Output (insertion order preserved):
// Charlie = 3
// Alice = 1
// Bob = 2

// Updating a value does NOT change order
map.put("Alice", 10);
// Order: Charlie, Alice, Bob (Alice stays in original position)
```

### LinkedHashMap - Access Order (LRU Cache)

```java
// accessOrder = true: most recently accessed entries move to tail
Map<String, Integer> accessMap = new LinkedHashMap<>(16, 0.75f, true);
accessMap.put("A", 1);
accessMap.put("B", 2);
accessMap.put("C", 3);
// Order: A -> B -> C

accessMap.get("A");    // Access "A" - moves to tail
// Order: B -> C -> A

accessMap.get("B");    // Access "B" - moves to tail
// Order: C -> A -> B

accessMap.put("C", 30);  // Update "C" - moves to tail
// Order: A -> B -> C
```

### LRU Cache with removeEldestEntry()

```java
// Simple LRU (Least Recently Used) cache
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;

    public LRUCache(int maxSize) {
        super(maxSize * 4 / 3 + 1, 0.75f, true);  // accessOrder = true
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;  // Remove oldest when exceeding max size
    }
}

// Usage
LRUCache<String, String> cache = new LRUCache<>(3);
cache.put("A", "1");
cache.put("B", "2");
cache.put("C", "3");
// Cache: [A=1, B=2, C=3]

cache.get("A");         // Access A - moves to end
// Cache: [B=2, C=3, A=1]

cache.put("D", "4");   // Exceeds max size - removes eldest (B)
// Cache: [C=3, A=1, D=4]

cache.get("C");         // Access C
// Cache: [A=1, D=4, C=3]

cache.put("E", "5");   // Removes eldest (A)
// Cache: [D=4, C=3, E=5]
```

### TreeMap - Red-Black Tree (Sorted Map)

TreeMap stores entries in a **self-balancing Red-Black tree**, sorted by keys.

```java
// Internal structure
public class TreeMap<K,V> extends AbstractMap<K,V>
        implements NavigableMap<K,V>, Cloneable, Serializable {

    private final Comparator<? super K> comparator;
    private transient Entry<K,V> root;
    private transient int size = 0;

    static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;
        Entry<K,V> right;
        Entry<K,V> parent;
        boolean color = BLACK;
    }
}
```

### TreeMap Usage

```java
// Natural ordering (keys must implement Comparable)
TreeMap<String, Integer> treeMap = new TreeMap<>();
treeMap.put("Charlie", 3);
treeMap.put("Alice", 1);
treeMap.put("Bob", 2);

for (Map.Entry<String, Integer> entry : treeMap.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}
// Output (sorted by key):
// Alice = 1
// Bob = 2
// Charlie = 3

// Custom comparator
TreeMap<String, Integer> reverseMap = new TreeMap<>(Comparator.reverseOrder());
reverseMap.putAll(treeMap);
// Output: Charlie, Bob, Alice
```

### NavigableMap Methods (TreeMap-Specific)

```java
TreeMap<Integer, String> scores = new TreeMap<>();
scores.put(60, "D");
scores.put(70, "C");
scores.put(80, "B");
scores.put(90, "A");
scores.put(100, "A+");

// First and last entries
scores.firstKey();                    // 60
scores.lastKey();                     // 100
scores.firstEntry();                  // 60=D
scores.lastEntry();                   // 100=A+

// Closest key queries
scores.lowerKey(80);                  // 70 (strictly less)
scores.floorKey(80);                  // 80 (less or equal)
scores.ceilingKey(85);                // 90 (greater or equal)
scores.higherKey(80);                 // 90 (strictly greater)

// Polling (retrieve and remove)
Map.Entry<Integer, String> lowest = scores.pollFirstEntry();  // 60=D
Map.Entry<Integer, String> highest = scores.pollLastEntry();  // 100=A+

// Range views (these are LIVE views!)
SortedMap<Integer, String> head = scores.headMap(80);         // {70=C} (exclusive)
SortedMap<Integer, String> tail = scores.tailMap(80);         // {80=B, 90=A}
SortedMap<Integer, String> sub  = scores.subMap(70, 90);      // {70=C, 80=B}

// NavigableMap range with inclusive flags
NavigableMap<Integer, String> sub2 = scores.subMap(70, true, 90, true);
// {70=C, 80=B, 90=A}

// Descending view
NavigableMap<Integer, String> desc = scores.descendingMap();
// {90=A, 80=B, 70=C}
```

### Real-World Use Cases

```java
// ✅ TreeMap: Grade calculator
TreeMap<Integer, String> gradeScale = new TreeMap<>();
gradeScale.put(90, "A");
gradeScale.put(80, "B");
gradeScale.put(70, "C");
gradeScale.put(60, "D");
gradeScale.put(0, "F");

public String getGrade(int score) {
    Map.Entry<Integer, String> entry = gradeScale.floorEntry(score);
    return entry != null ? entry.getValue() : "F";
}
getGrade(85);  // "B" (floor of 85 is 80)
getGrade(92);  // "A" (floor of 92 is 90)
getGrade(55);  // "F" (floor of 55 is 0)

// ✅ TreeMap: IP range lookup
TreeMap<Long, String> ipRanges = new TreeMap<>();
ipRanges.put(toIpLong("10.0.0.0"), "Private-A");
ipRanges.put(toIpLong("172.16.0.0"), "Private-B");
ipRanges.put(toIpLong("192.168.0.0"), "Private-C");

String network = ipRanges.floorEntry(toIpLong("192.168.1.100")).getValue();
// "Private-C"

// ✅ LinkedHashMap: Maintaining config insertion order
Map<String, String> config = new LinkedHashMap<>();
config.put("host", "localhost");
config.put("port", "8080");
config.put("context", "/api");
// Writing config preserves order - important for readability

// ✅ LinkedHashMap: Building ordered JSON-like structures
Map<String, Object> response = new LinkedHashMap<>();
response.put("status", 200);
response.put("message", "OK");
response.put("data", resultList);
// JSON output: {"status":200,"message":"OK","data":[...]}
```

### Null Handling

```java
// LinkedHashMap: allows one null key and multiple null values
LinkedHashMap<String, Integer> lhm = new LinkedHashMap<>();
lhm.put(null, 1);      // ✅ OK
lhm.put("key", null);  // ✅ OK

// TreeMap: does NOT allow null keys (needs comparison)
TreeMap<String, Integer> tm = new TreeMap<>();
tm.put(null, 1);        // ❌ NullPointerException
tm.put("key", null);    // ✅ null values are OK
```

### Performance Comparison

```java
// Operation          | HashMap  | LinkedHashMap | TreeMap
// -------------------|----------|---------------|--------
// put(k,v)           | O(1)     | O(1)          | O(log n)
// get(k)             | O(1)     | O(1)          | O(log n)
// remove(k)          | O(1)     | O(1)          | O(log n)
// containsKey(k)     | O(1)     | O(1)          | O(log n)
// firstKey/lastKey   | N/A      | O(1)          | O(log n)
// floor/ceiling      | N/A      | N/A           | O(log n)
// Iteration order    | None     | Insert/Access | Sorted
// Null key           | Yes      | Yes           | No*
// Memory overhead    | Low      | Medium        | High

// * Unless using null-safe Comparator
```

### Common Anti-Patterns

```java
// ❌ Using TreeMap when you don't need sorted order
Map<String, Integer> map = new TreeMap<>();  // O(log n) ops
// Use HashMap for O(1) operations when order doesn't matter

// ❌ Modifying TreeMap keys after insertion
TreeMap<StringBuilder, Integer> bad = new TreeMap<>();
StringBuilder key = new StringBuilder("A");
bad.put(key, 1);
key.append("B");  // Tree ordering is now corrupted!

// ❌ Using LinkedHashMap(accessOrder=true) without removeEldestEntry
// Memory will grow without bound - you probably wanted an LRU cache

// ❌ Iterating LinkedHashMap with get() for access-ordered map
Map<String, Integer> accessMap = new LinkedHashMap<>(16, 0.75f, true);
accessMap.put("A", 1); accessMap.put("B", 2); accessMap.put("C", 3);
for (String key : accessMap.keySet()) {
    accessMap.get(key);  // ❌ ConcurrentModificationException!
    // get() modifies structure in access-order mode
}
```

### Best Practices

```java
// ✅ Use HashMap when order doesn't matter (fastest)
Map<String, Integer> counts = new HashMap<>();

// ✅ Use LinkedHashMap when insertion order matters
Map<String, String> orderedConfig = new LinkedHashMap<>();

// ✅ Use TreeMap when sorted keys or range queries are needed
NavigableMap<LocalDate, Event> calendar = new TreeMap<>();

// ✅ Use LRU cache pattern for bounded caches
Map<String, Data> cache = new LinkedHashMap<>(100, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Data> eldest) {
        return size() > 100;
    }
};

// ✅ Use Collections.unmodifiableMap for read-only views
Map<String, Integer> readOnly = Collections.unmodifiableMap(map);

// ✅ Use Map.of() / Map.ofEntries() for immutable maps (Java 9+)
Map<String, Integer> constants = Map.of("A", 1, "B", 2, "C", 3);
```

---

## Interview Tips
- LinkedHashMap = HashMap + doubly-linked list for order
- TreeMap = Red-Black tree, always sorted, O(log n) operations
- Know the LRU cache implementation with `removeEldestEntry()`
- NavigableMap methods (floorKey, ceilingKey, subMap) are great for range queries
- TreeMap doesn't use hashCode - it uses Comparable/Comparator
- LinkedHashMap supports both insertion-order and access-order modes

---

[Prev: Q7 - HashMap Internals](./q07-hashmap-internals.md) | [Next: Q9 - ConcurrentHashMap](./q09-concurrenthashmap.md)

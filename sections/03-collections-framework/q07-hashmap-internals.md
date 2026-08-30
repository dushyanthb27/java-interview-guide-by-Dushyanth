# Q7: How Does HashMap Work Internally? (Deep Dive)

[Back to Collections Framework](./README.md) | **Difficulty**: 🟠 Hard

---

## Question
> Explain the internal working of HashMap in detail. How does it handle hashing, collisions, and resizing?

---

## Answer

### Internal Structure

HashMap stores key-value pairs in an **array of buckets** (called `table`). Each bucket can hold a linked list or a red-black tree of entries.

```java
// Simplified internal structure from OpenJDK
public class HashMap<K, V> extends AbstractMap<K, V>
        implements Map<K, V>, Cloneable, Serializable {

    // Constants
    static final int DEFAULT_INITIAL_CAPACITY = 16;      // Must be power of 2
    static final int MAXIMUM_CAPACITY = 1 << 30;         // 2^30
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final int TREEIFY_THRESHOLD = 8;              // List -> Tree
    static final int UNTREEIFY_THRESHOLD = 6;            // Tree -> List
    static final int MIN_TREEIFY_CAPACITY = 64;          // Min table size for treeify

    // Fields
    transient Node<K,V>[] table;          // The bucket array
    transient Set<Map.Entry<K,V>> entrySet;
    transient int size;                    // Number of key-value pairs
    transient int modCount;                // Structural modification count
    int threshold;                         // capacity * loadFactor
    final float loadFactor;

    // Node class (bucket entry)
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;       // cached hash value
        final K key;
        V value;
        Node<K,V> next;      // next node in bucket chain

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }

    // TreeNode for red-black tree buckets (Java 8+)
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;
        boolean red;
    }
}
```

### Hash Function - Spreading Bits

```java
// HashMap's internal hash function (NOT Object.hashCode() directly)
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// Why XOR with upper 16 bits?
// Bucket index = hash & (capacity - 1)
// For small capacities (e.g., 16), only lower 4 bits determine bucket
// XOR spreads influence of higher bits to lower bits, reducing collisions

// Example:
// key.hashCode()  = 0b 1010_1100_0011_0111_0000_0000_0000_0101
// h >>> 16        = 0b 0000_0000_0000_0000_1010_1100_0011_0111
// h ^ (h >>> 16)  = 0b 1010_1100_0011_0111_1010_1100_0011_0010
//                                          ^^^^^^^^^^^^^^^^ lower bits now mixed
```

### Bucket Index Calculation

```java
// Index = hash & (capacity - 1)
// This is equivalent to hash % capacity when capacity is a power of 2
// But bitwise AND is MUCH faster than modulo

// Example with capacity = 16 (binary: 10000)
// capacity - 1 = 15 (binary: 01111)

// hash = 0b...1010_0111
// index = 0b...1010_0111 & 0b0000_1111 = 0b0000_0111 = 7
// This key goes into bucket[7]

// That's why capacity MUST be a power of 2!
// HashMap rounds up any initial capacity to the next power of 2
// new HashMap<>(10) -> actual capacity = 16
// new HashMap<>(17) -> actual capacity = 32
```

### put() Operation - Step by Step

```java
// What happens when you call: map.put("key", "value")

// Step 1: Calculate hash
int hash = hash("key");  // key.hashCode() ^ (key.hashCode() >>> 16)

// Step 2: Calculate bucket index
int index = hash & (table.length - 1);

// Step 3: Check bucket
// Case A: Bucket is empty -> create new Node
if (table[index] == null) {
    table[index] = new Node<>(hash, "key", "value", null);
}

// Case B: Bucket has entries -> traverse chain
else {
    Node<K,V> node = table[index];
    while (node != null) {
        // If key already exists (same hash AND equals)
        if (node.hash == hash && node.key.equals("key")) {
            node.value = "value";  // Update value, return old value
            return oldValue;
        }
        node = node.next;
    }
    // Key not found in chain -> add new Node at end (Java 8: tail insertion)
    // Java 7 used head insertion (could cause infinite loop in concurrent resize!)
}

// Step 4: Check if resize needed
if (++size > threshold) {
    resize();  // Double capacity, rehash all entries
}
```

### Visual: put() Operation

```
Initial state (capacity=4):
  Bucket: [0]    [1]    [2]    [3]
           |      |      |      |
          null   null   null   null

After put("A", 1):  hash("A") & 3 = 1
  Bucket: [0]    [1]         [2]    [3]
           |      |           |      |
          null  [A:1]->null  null   null

After put("E", 2):  hash("E") & 3 = 1  (collision with "A"!)
  Bucket: [0]    [1]              [2]    [3]
           |      |                |      |
          null  [A:1]->[E:2]->null null   null

After put("B", 3):  hash("B") & 3 = 2
  Bucket: [0]    [1]              [2]         [3]
           |      |                |           |
          null  [A:1]->[E:2]     [B:3]->null  null
```

### Collision Handling: Linked List to Red-Black Tree

```java
// Java 8+ Treeification:
// When a single bucket has MORE THAN 8 entries (TREEIFY_THRESHOLD = 8)
// AND table capacity >= 64 (MIN_TREEIFY_CAPACITY)
// The linked list in that bucket converts to a Red-Black Tree

// Before treeification (linked list):
// bucket[5] -> [K1:V1] -> [K2:V2] -> [K3:V3] -> ... -> [K9:V9] -> null
// Lookup: O(n) - must traverse up to 9 nodes

// After treeification (red-black tree):
// bucket[5] ->    [K5:V5]
//                /        \
//           [K2:V2]      [K7:V7]
//           /    \        /    \
//       [K1:V1] [K3:V3] [K6:V6] [K9:V9]
// Lookup: O(log n) - balanced tree search

// When bucket shrinks below 6 entries (UNTREEIFY_THRESHOLD = 6)
// Tree converts back to linked list

// If table capacity < 64, HashMap resizes instead of treeifying
```

### Null Key Handling

```java
HashMap<String, Integer> map = new HashMap<>();

// ✅ HashMap allows ONE null key (stored at index 0)
map.put(null, 42);
map.get(null);           // 42
map.containsKey(null);   // true

// Internally: hash(null) returns 0
// So null key always goes to bucket[0]

// ✅ HashMap allows multiple null values
map.put("key1", null);
map.put("key2", null);
map.get("key1");         // null (but is the key present or absent?)
map.containsKey("key1"); // true - use this to check presence
```

### Resizing (Rehashing)

```java
// Triggered when: size > threshold (capacity * loadFactor)
// Default: resize when 13th element added (16 * 0.75 = 12)

// resize() operation:
// 1. Create new array with 2x capacity
// 2. For EACH entry in old array:
//    - Recalculate bucket index (hash & (newCapacity - 1))
//    - Place in new array
// 3. Replace old array with new array

// Java 8 optimization: entries in a bucket either stay at same index
// OR move to (oldIndex + oldCapacity)
// This avoids recalculating hash - just check one bit!

// Example: oldCapacity = 16 (binary: 10000), newCapacity = 32 (100000)
// Old index = hash & 0b01111
// New index = hash & 0b11111
// If bit 4 of hash is 0: stays at same index
// If bit 4 of hash is 1: moves to oldIndex + 16

// Resize progression: 16 -> 32 -> 64 -> 128 -> 256 -> ...
```

### Load Factor Deep Dive

```java
// Load factor = size / capacity
// Higher load factor: more elements per bucket, more collisions, less memory
// Lower load factor: fewer collisions, more memory, fewer resizes

// Default 0.75 is a good balance between time and space

HashMap<String, Integer> map;

// High load factor (0.9) - memory efficient, more collisions
map = new HashMap<>(16, 0.9f);
// threshold = 16 * 0.9 = 14 (resize at 15th element)

// Low load factor (0.5) - fewer collisions, more memory
map = new HashMap<>(16, 0.5f);
// threshold = 16 * 0.5 = 8 (resize at 9th element)

// ✅ Pre-sizing to avoid resizes
// If you expect 100 entries:
// capacity needed = 100 / 0.75 = 134 -> next power of 2 = 256
map = new HashMap<>(256);  // No resizing for up to 192 entries
// Or simpler: new HashMap<>(100 * 4 / 3 + 1)
```

### equals() and hashCode() Contract for Keys

```java
// CRITICAL: Keys in HashMap MUST follow the hashCode/equals contract

public class CacheKey {
    private final String module;
    private final int version;

    // ✅ Must use same fields for both methods
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        CacheKey that = (CacheKey) o;
        return version == that.version &&
               Objects.equals(module, that.module);
    }

    @Override
    public int hashCode() {
        return Objects.hash(module, version);
    }
}

// ❌ What happens with broken hashCode:
// Two equal keys get DIFFERENT hash codes
// They end up in DIFFERENT buckets
// map.put(key1, value) puts in bucket[3]
// map.get(key2) looks in bucket[7]  -- NOT FOUND!
// Even though key1.equals(key2) is true
```

### Java 7 vs Java 8 HashMap Changes

```java
// Java 7 HashMap:
// - Linked list only (no tree optimization)
// - Head insertion for collisions (newest first)
// - Infinite loop possible during concurrent resize
//   (due to head insertion reversing list order)

// Java 8+ HashMap:
// - Linked list + Red-Black tree (TREEIFY_THRESHOLD = 8)
// - Tail insertion for collisions (insertion order in bucket)
// - No infinite loop on concurrent resize (but still not thread-safe!)
// - Better hash function (XOR with upper bits)
// - Improved resize algorithm (entries stay or move by oldCapacity)
```

### Thread Safety Warning

```java
// ❌ HashMap is NOT thread-safe!
// Concurrent modifications can cause:
// - Lost updates (two puts overwrite each other)
// - Infinite loops (Java 7 resize)
// - ConcurrentModificationException (during iteration)
// - Corrupted internal state

// ✅ Thread-safe alternatives:
// 1. ConcurrentHashMap (best for most concurrent use cases)
Map<String, Integer> safe = new ConcurrentHashMap<>();

// 2. Collections.synchronizedMap (coarse-grained locking)
Map<String, Integer> synced = Collections.synchronizedMap(new HashMap<>());

// 3. Hashtable (legacy - avoid)
```

### Common Anti-Patterns

```java
// ❌ Using mutable keys
Map<List<String>, String> map = new HashMap<>();
List<String> key = new ArrayList<>(List.of("A"));
map.put(key, "value");
key.add("B");          // Hash code changes! Entry is LOST
map.get(key);          // null!

// ❌ Not pre-sizing when count is known
Map<String, String> map = new HashMap<>();  // will resize multiple times
for (int i = 0; i < 10000; i++) { map.put("key" + i, "val"); }

// ❌ Iterating with get(key) instead of entrySet
for (String key : map.keySet()) {
    String value = map.get(key);  // Extra hash lookup per entry!
}

// ❌ Checking containsKey before get
if (map.containsKey("key")) {
    String val = map.get("key");  // Double hash lookup!
}
```

### Best Practices

```java
// ✅ Pre-size when count is known
Map<String, String> map = new HashMap<>(expectedSize * 4 / 3 + 1);

// ✅ Use entrySet() for iteration
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    String key = entry.getKey();
    Integer value = entry.getValue();
}

// ✅ Use getOrDefault / computeIfAbsent
String val = map.getOrDefault("key", "default");
List<String> list = map.computeIfAbsent("key", k -> new ArrayList<>());

// ✅ Use immutable keys (String, Integer, enums)
Map<String, Object> safeMap = new HashMap<>();

// ✅ Use Map.of() for small immutable maps (Java 9+)
Map<String, Integer> config = Map.of("timeout", 30, "retries", 3);

// ✅ Use putIfAbsent for conditional insertion
map.putIfAbsent("key", expensiveComputation());

// ✅ Use merge for combining values
map.merge("key", 1, Integer::sum);  // Increment counter
```

---

## Performance Summary

| Operation | Average Case | Worst Case (all collisions) |
|-----------|-------------|----------------------------|
| `put(key, value)` | O(1) | O(log n)* |
| `get(key)` | O(1) | O(log n)* |
| `remove(key)` | O(1) | O(log n)* |
| `containsKey(key)` | O(1) | O(log n)* |
| `containsValue(val)` | O(n) | O(n) |
| Iteration | O(capacity + n) | O(capacity + n) |
| Resize | O(n) | O(n) |

\* O(n) in Java 7 (linked list only), O(log n) in Java 8+ (red-black tree)

---

## Interview Tips
- This is THE most asked collection question - know every detail
- Draw the bucket array + linked list + tree structure
- Explain the hash spreading function: `hash ^ (hash >>> 16)`
- Know TREEIFY_THRESHOLD (8), UNTREEIFY_THRESHOLD (6), MIN_TREEIFY_CAPACITY (64)
- Explain why capacity must be a power of 2 (for `hash & (n-1)` optimization)
- Know Java 7 vs Java 8 differences (head vs tail insertion, tree optimization)
- Emphasize the equals/hashCode contract for keys

---

[Prev: Q6 - LinkedHashSet and TreeSet](./q06-linkedhashset-treeset.md) | [Next: Q8 - LinkedHashMap and TreeMap](./q08-linkedhashmap-treemap.md)

# Q19: Thread-Safe Collection Options

[Back to Collections Framework](./README.md) | **Difficulty**: 🟠 Hard

---

## Question
> What thread-safe collection options are available in Java? When would you use each?

---

## Answer

### Overview of Thread-Safe Options

```
Thread-Safe Collection Options in Java:

1. Legacy Synchronized Collections:
   └── Vector, Hashtable, Stack (avoid in new code)

2. Synchronized Wrappers:
   └── Collections.synchronizedList/Set/Map()

3. Concurrent Collections (java.util.concurrent):
   ├── ConcurrentHashMap
   ├── ConcurrentLinkedQueue
   ├── ConcurrentLinkedDeque
   ├── ConcurrentSkipListMap
   └── ConcurrentSkipListSet

4. Copy-On-Write Collections:
   ├── CopyOnWriteArrayList
   └── CopyOnWriteArraySet

5. Blocking Queues:
   ├── ArrayBlockingQueue
   ├── LinkedBlockingQueue
   ├── PriorityBlockingQueue
   ├── DelayQueue
   ├── SynchronousQueue
   └── LinkedTransferQueue
```

### 1. Synchronized Wrappers

```java
// Collections.synchronizedXxx() - wraps any collection with synchronized methods
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
Set<String> syncSet = Collections.synchronizedSet(new HashSet<>());
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());

// How it works internally:
// Every method call acquires the same mutex lock
// class SynchronizedList<E> implements List<E> {
//     final Object mutex;
//     final List<E> list;
//
//     public E get(int index) {
//         synchronized (mutex) { return list.get(index); }
//     }
//     public boolean add(E e) {
//         synchronized (mutex) { return list.add(e); }
//     }
// }

// ⚠️ Iteration is NOT automatically synchronized!
synchronized (syncList) {  // Must manually synchronize!
    for (String s : syncList) {
        System.out.println(s);
    }
}

// ⚠️ Compound operations are NOT atomic!
// Check-then-act race condition:
if (!syncMap.containsKey("key")) {    // Lock acquired and released
    syncMap.put("key", value);         // Another thread may insert between!
}
// ✅ Fix: synchronize the compound operation
synchronized (syncMap) {
    if (!syncMap.containsKey("key")) {
        syncMap.put("key", value);
    }
}
```

### 2. ConcurrentHashMap

```java
// Best general-purpose thread-safe map
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// ✅ Lock-free reads (volatile reads)
map.get("key");           // No locking!

// ✅ Per-node/bucket locking for writes (CAS + synchronized)
map.put("key", 1);        // Only locks the specific bucket

// ✅ Atomic compound operations
map.putIfAbsent("key", 0);
map.computeIfAbsent("key", k -> expensiveCreate(k));
map.merge("counter", 1, Integer::sum);

// ❌ No null keys or values
// map.put(null, 1);      // NullPointerException
// map.put("key", null);  // NullPointerException

// ✅ Weakly consistent iterators (no ConcurrentModificationException)
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    // Safe to iterate while other threads modify the map
    // May or may not see concurrent modifications
}

// ✅ Concurrent Set creation
Set<String> concurrentSet = ConcurrentHashMap.newKeySet();
concurrentSet.add("item");
```

### 3. CopyOnWriteArrayList

```java
// Creates a new internal array copy on EVERY write operation
CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();

// Write operations (expensive - full array copy each time)
cowList.add("A");           // O(n) - creates new array
cowList.add("B");           // O(n)
cowList.set(0, "X");        // O(n)
cowList.remove("A");        // O(n)

// Read operations (fast - no locking, no copying)
cowList.get(0);             // O(1) - direct array access, lock-free
cowList.contains("X");      // O(n) - linear scan, lock-free
cowList.size();              // O(1) - lock-free

// ✅ Iteration is safe - works on snapshot
for (String s : cowList) {
    cowList.add("new");  // ✅ No ConcurrentModificationException!
    // Iterator sees the OLD snapshot (before "new" was added)
}

// ⚠️ Iterator does NOT support modification
Iterator<String> it = cowList.iterator();
it.next();
// it.remove();  // ❌ UnsupportedOperationException!

// IDEAL FOR: event listener lists, observer pattern
// - Listeners registered rarely (writes are rare)
// - Listeners notified frequently (reads are frequent)
List<EventListener> listeners = new CopyOnWriteArrayList<>();

public void addListener(EventListener listener) {
    listeners.add(listener);  // Rare write
}

public void fireEvent(Event event) {
    for (EventListener listener : listeners) {  // Frequent read
        listener.onEvent(event);  // Safe, lock-free iteration
    }
}
```

### 4. CopyOnWriteArraySet

```java
// Backed by CopyOnWriteArrayList (not HashMap!)
CopyOnWriteArraySet<String> cowSet = new CopyOnWriteArraySet<>();

// Same copy-on-write semantics
cowSet.add("A");            // O(n) - array copy + contains check
cowSet.remove("A");         // O(n) - array copy
cowSet.contains("A");       // O(n) - linear scan (not hash-based!)

// ⚠️ contains() is O(n), not O(1) like HashSet
// Only use for small sets or read-heavy scenarios
```

### 5. ConcurrentSkipListMap / ConcurrentSkipListSet

```java
// Thread-safe SORTED map/set using Skip List data structure
ConcurrentSkipListMap<String, Integer> skipMap = new ConcurrentSkipListMap<>();

// ✅ Thread-safe with O(log n) operations
skipMap.put("Charlie", 3);
skipMap.put("Alice", 1);
skipMap.put("Bob", 2);

// ✅ NavigableMap operations (like TreeMap, but thread-safe)
skipMap.firstKey();           // "Alice"
skipMap.lastKey();            // "Charlie"
skipMap.floorKey("Bob");      // "Bob"
skipMap.ceilingKey("Bobby");  // "Charlie"
skipMap.subMap("Alice", "Charlie"); // {Alice=1, Bob=2}

// ✅ Weakly consistent iterators
for (Map.Entry<String, Integer> e : skipMap.entrySet()) {
    // Safe concurrent iteration
}

// ConcurrentSkipListSet
ConcurrentSkipListSet<String> skipSet = new ConcurrentSkipListSet<>();
// Thread-safe sorted set with NavigableSet operations
```

### 6. Blocking Queues

```java
// ArrayBlockingQueue - bounded, array-based
BlockingQueue<String> abq = new ArrayBlockingQueue<>(100);
abq.put("item");    // Blocks if full
abq.take();         // Blocks if empty
abq.offer("item", 5, TimeUnit.SECONDS);  // Waits up to 5s
abq.poll(5, TimeUnit.SECONDS);           // Waits up to 5s

// LinkedBlockingQueue - optionally bounded, node-based
BlockingQueue<String> lbq = new LinkedBlockingQueue<>();      // Unbounded
BlockingQueue<String> lbqB = new LinkedBlockingQueue<>(100);  // Bounded

// Key difference: ArrayBlockingQueue uses single lock for both put/take
// LinkedBlockingQueue uses TWO locks (put lock + take lock) = better throughput

// PriorityBlockingQueue - unbounded, priority-ordered
BlockingQueue<Task> pbq = new PriorityBlockingQueue<>(11,
    Comparator.comparingInt(Task::getPriority));

// SynchronousQueue - zero capacity, direct handoff
BlockingQueue<String> sq = new SynchronousQueue<>();
// put() blocks until another thread calls take()
// Used in Executors.newCachedThreadPool()

// DelayQueue - elements available after a delay
BlockingQueue<DelayedTask> dq = new DelayQueue<>();
// Elements must implement Delayed interface
// poll() returns null if no elements have expired
// take() blocks until an element's delay has expired

// LinkedTransferQueue - combines LinkedBlockingQueue + SynchronousQueue
TransferQueue<String> tq = new LinkedTransferQueue<>();
tq.transfer("item");    // Blocks until a consumer takes it (like SynchronousQueue)
tq.put("item");          // Always succeeds (like LinkedBlockingQueue)
tq.tryTransfer("item");  // Returns false immediately if no waiting consumer
```

### Lock Striping (ConcurrentHashMap Concept)

```java
// Lock striping = dividing data into segments, each with its own lock
// Multiple threads can operate on different segments simultaneously

// Conceptual example of lock striping:
public class StripedMap<K, V> {
    private static final int NUM_LOCKS = 16;
    private final Object[] locks = new Object[NUM_LOCKS];
    private final Map<K, V>[] buckets;

    public StripedMap() {
        for (int i = 0; i < NUM_LOCKS; i++) {
            locks[i] = new Object();
        }
        // Initialize buckets...
    }

    private int getLockIndex(K key) {
        return Math.abs(key.hashCode() % NUM_LOCKS);
    }

    public V get(K key) {
        int lockIdx = getLockIndex(key);
        synchronized (locks[lockIdx]) {  // Only lock this stripe
            return buckets[lockIdx].get(key);
        }
    }
}

// ConcurrentHashMap (Java 8+) goes even further:
// - CAS for empty buckets (no lock at all)
// - synchronized on individual nodes (finest granularity)
```

### When to Use Which

```java
// ┌──────────────────────────────┬─────────────────────────────────┐
// │ Scenario                     │ Best Choice                     │
// ├──────────────────────────────┼─────────────────────────────────┤
// │ Concurrent key-value store   │ ConcurrentHashMap               │
// │ Concurrent sorted map        │ ConcurrentSkipListMap           │
// │ Read-heavy list, few writes  │ CopyOnWriteArrayList            │
// │ Read-heavy set, few writes   │ CopyOnWriteArraySet (small)     │
// │ Producer-consumer queue      │ LinkedBlockingQueue              │
// │ Bounded producer-consumer    │ ArrayBlockingQueue               │
// │ Priority-based processing    │ PriorityBlockingQueue            │
// │ Direct thread handoff        │ SynchronousQueue                 │
// │ Non-blocking concurrent queue│ ConcurrentLinkedQueue            │
// │ Non-blocking concurrent deque│ ConcurrentLinkedDeque            │
// │ Legacy API compatibility     │ Collections.synchronizedXxx()    │
// │ Concurrent sorted set        │ ConcurrentSkipListSet            │
// │ Delayed task processing      │ DelayQueue                       │
// └──────────────────────────────┴─────────────────────────────────┘
```

### Performance Under Contention

```java
// Low contention (few threads, infrequent access):
// - All options perform similarly
// - synchronized wrappers are fine
// - ConcurrentHashMap has slight overhead from volatile reads

// High contention (many threads, frequent access):
//
// WORST:  Hashtable / Collections.synchronizedMap
//         └── Single lock, all operations serialized
//         └── Throughput drops significantly
//
// BETTER: Collections.synchronizedMap with external sync
//         └── Still single lock, but more control
//
// BEST:   ConcurrentHashMap
//         └── Lock-free reads
//         └── Per-node locking for writes
//         └── CAS for empty buckets
//         └── Throughput scales with thread count

// Read-to-write ratio matters:
// Mostly reads -> CopyOnWriteArrayList outperforms synchronizedList
// Mostly writes -> synchronizedList outperforms CopyOnWriteArrayList
// Mixed -> ConcurrentHashMap (for maps), ConcurrentLinkedQueue (for queues)
```

### Common Anti-Patterns

```java
// ❌ Using Hashtable or Vector in new code
Hashtable<String, Integer> ht = new Hashtable<>();  // Legacy!

// ❌ Using synchronized wrappers when concurrent collections are available
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());
// ✅ Use ConcurrentHashMap instead

// ❌ Forgetting to synchronize iteration on synchronized wrappers
for (String s : syncList) { ... }  // NOT thread-safe!

// ❌ Using CopyOnWriteArrayList for write-heavy scenarios
// Each write copies the entire array - O(n) per write!

// ❌ Assuming ConcurrentHashMap.size() is exact under contention
// It returns an estimate

// ❌ Using putIfAbsent with expensive value creation
map.putIfAbsent(key, new ExpensiveObject());  // Object created even if key exists!
// ✅ Use computeIfAbsent instead
map.computeIfAbsent(key, k -> new ExpensiveObject());  // Lambda only called if absent
```

### Best Practices

```java
// ✅ Default concurrent map: ConcurrentHashMap
Map<String, Service> services = new ConcurrentHashMap<>();

// ✅ Use atomic operations for compound actions
services.computeIfAbsent("cache", k -> createCacheService());
services.merge("counter", 1, Integer::sum);

// ✅ Use CopyOnWriteArrayList for event listeners
List<Listener> listeners = new CopyOnWriteArrayList<>();

// ✅ Use BlockingQueue for producer-consumer
BlockingQueue<Task> taskQueue = new LinkedBlockingQueue<>(1000);

// ✅ Use ConcurrentSkipListMap when you need sorted + concurrent
NavigableMap<Instant, Event> timeline = new ConcurrentSkipListMap<>();

// ✅ Choose based on read/write ratio and contention level
// ✅ Pre-size concurrent collections when possible
// ✅ Use LongAdder instead of AtomicLong for high-contention counters
ConcurrentHashMap<String, LongAdder> counters = new ConcurrentHashMap<>();
counters.computeIfAbsent("hits", k -> new LongAdder()).increment();

// ✅ Prefer immutable collections for shared read-only data
List<String> config = List.of("option1", "option2", "option3");
// No synchronization needed - immutable is inherently thread-safe!
```

---

## Quick Reference

| Collection | Read | Write | Iterator | Null | Best For |
|------------|------|-------|----------|------|----------|
| synchronizedList | O(1) locked | O(1) locked | Must sync | Yes | Legacy compat |
| CopyOnWriteArrayList | O(1) free | O(n) copy | Snapshot | Yes | Read-heavy |
| ConcurrentHashMap | O(1) free | O(1) CAS/sync | Weakly consistent | No | General concurrent map |
| ConcurrentSkipListMap | O(log n) | O(log n) | Weakly consistent | No | Sorted concurrent map |
| LinkedBlockingQueue | O(1) locked | O(1) locked | Weakly consistent | No | Producer-consumer |
| ArrayBlockingQueue | O(1) locked | O(1) locked | Weakly consistent | No | Bounded queue |

---

## Interview Tips
- ConcurrentHashMap is the #1 choice for concurrent maps (know CAS + per-node sync)
- CopyOnWriteArrayList is ideal for event listeners (read-heavy, write-rare)
- BlockingQueue is essential for producer-consumer patterns
- Synchronized wrappers require manual sync during iteration
- Immutable collections are inherently thread-safe (cheapest option)
- Know the trade-offs: read/write ratio, contention level, ordering needs

---

[Prev: Q18 - Collection Performance](./q18-collection-performance.md) | [Next: Q20 - Custom Collections](./q20-custom-collections.md)

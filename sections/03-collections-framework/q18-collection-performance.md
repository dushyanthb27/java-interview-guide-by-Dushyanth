# Q18: Collection Performance Comparison

[Back to Collections Framework](./README.md) | **Difficulty**: 🟠 Hard

---

## Question
> Compare the performance characteristics of different collection implementations. How do you choose the right one?

---

## Answer

### Big-O Comparison: List Implementations

```
┌─────────────────────┬────────────┬────────────┬──────────┐
│ Operation           │ ArrayList  │ LinkedList │ Vector   │
├─────────────────────┼────────────┼────────────┼──────────┤
│ get(index)          │ O(1)       │ O(n)       │ O(1)     │
│ add(element) [end]  │ O(1)*      │ O(1)       │ O(1)*    │
│ add(index, element) │ O(n)       │ O(n)       │ O(n)     │
│ remove(index)       │ O(n)       │ O(n)       │ O(n)     │
│ remove(element)     │ O(n)       │ O(n)       │ O(n)     │
│ contains(element)   │ O(n)       │ O(n)       │ O(n)     │
│ indexOf(element)    │ O(n)       │ O(n)       │ O(n)     │
│ iterator.remove()   │ O(n)       │ O(1)       │ O(n)     │
│ size()              │ O(1)       │ O(1)       │ O(1)     │
└─────────────────────┴────────────┴────────────┴──────────┘
* Amortized O(1) - occasional O(n) resize
```

### Big-O Comparison: Set Implementations

```
┌─────────────────────┬──────────┬───────────────┬──────────┐
│ Operation           │ HashSet  │ LinkedHashSet │ TreeSet  │
├─────────────────────┼──────────┼───────────────┼──────────┤
│ add(element)        │ O(1)     │ O(1)          │ O(log n) │
│ remove(element)     │ O(1)     │ O(1)          │ O(log n) │
│ contains(element)   │ O(1)     │ O(1)          │ O(log n) │
│ first() / last()    │ N/A      │ N/A           │ O(log n) │
│ floor() / ceiling() │ N/A      │ N/A           │ O(log n) │
│ Iteration           │ O(cap)   │ O(n)          │ O(n)     │
│ size()              │ O(1)     │ O(1)          │ O(1)     │
└─────────────────────┴──────────┴───────────────┴──────────┘
Note: HashSet/LinkedHashSet average case; worst case O(log n) due to treeification
```

### Big-O Comparison: Map Implementations

```
┌─────────────────────┬──────────┬───────────────┬──────────┬───────────────────┐
│ Operation           │ HashMap  │ LinkedHashMap │ TreeMap  │ ConcurrentHashMap │
├─────────────────────┼──────────┼───────────────┼──────────┼───────────────────┤
│ put(key, value)     │ O(1)     │ O(1)          │ O(log n) │ O(1)              │
│ get(key)            │ O(1)     │ O(1)          │ O(log n) │ O(1)              │
│ remove(key)         │ O(1)     │ O(1)          │ O(log n) │ O(1)              │
│ containsKey(key)    │ O(1)     │ O(1)          │ O(log n) │ O(1)              │
│ containsValue(val)  │ O(n)     │ O(n)          │ O(n)     │ O(n)              │
│ firstKey/lastKey    │ N/A      │ O(1)          │ O(log n) │ N/A               │
│ Iteration           │ O(cap)   │ O(n)          │ O(n)     │ O(n)              │
│ size()              │ O(1)     │ O(1)          │ O(1)     │ ~O(1)**           │
└─────────────────────┴──────────┴───────────────┴──────────┴───────────────────┘
** ConcurrentHashMap.size() may be an estimate under contention
```

### Big-O Comparison: Queue/Deque Implementations

```
┌─────────────────────┬────────────┬────────────┬───────────────┐
│ Operation           │ ArrayDeque │ LinkedList │ PriorityQueue │
├─────────────────────┼────────────┼────────────┼───────────────┤
│ offer (enqueue)     │ O(1)*      │ O(1)       │ O(log n)      │
│ poll (dequeue)      │ O(1)       │ O(1)       │ O(log n)      │
│ peek                │ O(1)       │ O(1)       │ O(1)          │
│ push (stack)        │ O(1)*      │ O(1)       │ N/A           │
│ pop (stack)         │ O(1)       │ O(1)       │ N/A           │
│ contains            │ O(n)       │ O(n)       │ O(n)          │
│ remove(element)     │ O(n)       │ O(n)       │ O(n)          │
│ size()              │ O(1)       │ O(1)       │ O(1)          │
└─────────────────────┴────────────┴────────────┴───────────────┘
* Amortized O(1)
```

### Memory Overhead Comparison

```java
// Approximate memory per element (64-bit JVM, compressed oops):

// ArrayList:
//   ~4-8 bytes per slot (object reference in array)
//   + unused capacity overhead
//   Total: 4-8 bytes/element + wasted slots

// LinkedList:
//   ~40-48 bytes per element (Node: header + item + prev + next refs)
//   Total: ~40-48 bytes/element (5-10x more than ArrayList!)

// HashMap:
//   ~48-64 bytes per entry (Node: header + hash + key + value + next)
//   + bucket array overhead
//   Total: ~48-64 bytes/entry

// TreeMap:
//   ~56-72 bytes per entry (Entry: header + key + value + left + right + parent + color)
//   Total: ~56-72 bytes/entry

// HashSet:
//   Same as HashMap (backed by HashMap with dummy value)
//   ~48-64 bytes/element

// ArrayDeque:
//   ~4-8 bytes per slot (like ArrayList)
//   Total: 4-8 bytes/element + unused capacity

// PriorityQueue:
//   ~4-8 bytes per slot (array-based heap)
//   Total: 4-8 bytes/element + unused capacity
```

### Cache Performance and CPU Locality

```java
// Modern CPUs load data in cache lines (typically 64 bytes)
// Collections with contiguous memory benefit from spatial locality

// CACHE-FRIENDLY (contiguous memory):
// ArrayList  -> Object[] array, references are sequential
// ArrayDeque -> Object[] circular array
// PriorityQueue -> Object[] heap array
// HashMap    -> Node[] bucket array (partially)

// CACHE-UNFRIENDLY (scattered memory):
// LinkedList -> Nodes scattered across heap, pointer chasing
// TreeMap    -> Tree nodes scattered across heap
// TreeSet    -> Tree nodes scattered across heap

// Real-world impact: ArrayList iteration can be 5-10x faster
// than LinkedList iteration for large collections due to cache misses

// Benchmark example (approximate, depends on hardware):
// Sequential iteration of 1 million elements:
// ArrayList:    ~2ms
// LinkedList:   ~15ms (7-8x slower due to cache misses)
// ArrayDeque:   ~2ms
// HashSet:      ~5ms (hash table traversal)
// TreeSet:      ~10ms (tree traversal, scattered nodes)
```

### Benchmarking with JMH

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(1)
public class CollectionBenchmark {

    private ArrayList<Integer> arrayList;
    private LinkedList<Integer> linkedList;
    private HashSet<Integer> hashSet;
    private TreeSet<Integer> treeSet;
    private static final int SIZE = 100_000;

    @Setup
    public void setup() {
        arrayList = new ArrayList<>();
        linkedList = new LinkedList<>();
        hashSet = new HashSet<>();
        treeSet = new TreeSet<>();
        for (int i = 0; i < SIZE; i++) {
            arrayList.add(i);
            linkedList.add(i);
            hashSet.add(i);
            treeSet.add(i);
        }
    }

    @Benchmark
    public int arrayListGet() {
        return arrayList.get(SIZE / 2);   // O(1) direct access
    }

    @Benchmark
    public int linkedListGet() {
        return linkedList.get(SIZE / 2);  // O(n) traversal
    }

    @Benchmark
    public boolean hashSetContains() {
        return hashSet.contains(SIZE / 2);  // O(1) hash lookup
    }

    @Benchmark
    public boolean treeSetContains() {
        return treeSet.contains(SIZE / 2);  // O(log n) tree search
    }

    @Benchmark
    public int arrayListIteration() {
        int sum = 0;
        for (int i : arrayList) { sum += i; }
        return sum;
    }

    @Benchmark
    public int linkedListIteration() {
        int sum = 0;
        for (int i : linkedList) { sum += i; }
        return sum;
    }
}

// Running JMH:
// mvn clean install
// java -jar target/benchmarks.jar

// Typical results (ns/op, lower is better):
// arrayListGet:           ~5ns
// linkedListGet:          ~250,000ns (50,000x slower!)
// hashSetContains:        ~8ns
// treeSetContains:        ~80ns (10x slower than HashSet)
// arrayListIteration:     ~50,000ns
// linkedListIteration:    ~400,000ns (8x slower)
```

### Choosing the Right Collection

```java
// Decision tree:

// Do you need key-value pairs?
//   YES -> Do you need sorted keys?
//     YES -> TreeMap
//     NO  -> Do you need insertion order?
//       YES -> LinkedHashMap
//       NO  -> Do you need thread safety?
//         YES -> ConcurrentHashMap
//         NO  -> HashMap

// Do you need unique elements?
//   YES -> Do you need sorted order?
//     YES -> TreeSet
//     NO  -> Do you need insertion order?
//       YES -> LinkedHashSet
//       NO  -> HashSet

// Do you need ordered/indexed access?
//   YES -> ArrayList (default choice for List)

// Do you need stack/queue behavior?
//   YES -> ArrayDeque

// Do you need priority ordering?
//   YES -> PriorityQueue

// Do you need thread-safe list?
//   Read-heavy -> CopyOnWriteArrayList
//   Balanced   -> Collections.synchronizedList

// Do you need blocking queue?
//   YES -> LinkedBlockingQueue or ArrayBlockingQueue
```

### Performance Pitfalls

```java
// ❌ Pitfall 1: Using LinkedList for random access
for (int i = 0; i < linkedList.size(); i++) {
    linkedList.get(i);  // O(n) per call = O(n^2) total!
}
// ✅ Use ArrayList or iterator

// ❌ Pitfall 2: Not pre-sizing collections
Map<String, Integer> map = new HashMap<>();  // Default 16, will resize multiple times
for (int i = 0; i < 10000; i++) { map.put("key" + i, i); }
// ✅ Pre-size: new HashMap<>(10000 * 4 / 3 + 1)

// ❌ Pitfall 3: Unnecessary boxing
List<Integer> list = new ArrayList<>();
for (int i = 0; i < 1000000; i++) {
    list.add(i);  // Autoboxing int -> Integer for each element
}
// ✅ Use int[] array or IntStream for primitive operations

// ❌ Pitfall 4: Using containsKey + get instead of getOrDefault
if (map.containsKey(key)) {
    value = map.get(key);  // Two hash lookups!
}
// ✅ value = map.getOrDefault(key, defaultValue);  // Single lookup

// ❌ Pitfall 5: Iterating keySet + get instead of entrySet
for (String key : map.keySet()) {
    Integer value = map.get(key);  // Extra lookup per entry!
}
// ✅ for (Map.Entry<String, Integer> e : map.entrySet()) { ... }

// ❌ Pitfall 6: String concatenation in large loops
String result = "";
for (String s : list) { result += s; }  // O(n^2)!
// ✅ Use StringBuilder or Collectors.joining()
```

### Real-World Collection Selection

```java
// ✅ User lookup by ID -> HashMap<Long, User>
Map<Long, User> userCache = new HashMap<>(expectedUsers);

// ✅ Maintaining event listeners -> CopyOnWriteArrayList
List<EventListener> listeners = new CopyOnWriteArrayList<>();

// ✅ Task scheduling by priority -> PriorityQueue
Queue<Task> taskQueue = new PriorityQueue<>(Comparator.comparingInt(Task::getPriority));

// ✅ LRU cache -> LinkedHashMap with removeEldestEntry
Map<String, Data> lruCache = new LinkedHashMap<>(capacity, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > capacity;
    }
};

// ✅ Word frequency count -> HashMap with merge
Map<String, Integer> freq = new HashMap<>();
for (String word : words) {
    freq.merge(word, 1, Integer::sum);
}

// ✅ Sorted unique timestamps -> TreeSet
NavigableSet<Instant> timestamps = new TreeSet<>();

// ✅ Enum flags -> EnumSet (most efficient Set possible)
Set<Permission> perms = EnumSet.of(Permission.READ, Permission.WRITE);

// ✅ Enum-keyed map -> EnumMap
Map<DayOfWeek, List<Meeting>> schedule = new EnumMap<>(DayOfWeek.class);
```

### Common Anti-Patterns

```java
// ❌ Using TreeMap/TreeSet when order isn't needed
Map<String, Integer> map = new TreeMap<>();  // O(log n) when O(1) is available

// ❌ Using LinkedList "because insertions are O(1)"
// They're only O(1) at known positions - finding the position is O(n)

// ❌ Using synchronized collections when concurrent ones are available
Map<String, Integer> bad = Collections.synchronizedMap(new HashMap<>());
// ✅ Use ConcurrentHashMap instead

// ❌ Choosing collection based on insertion performance alone
// Consider read patterns, memory, cache locality, and thread safety
```

### Best Practices

```java
// ✅ Default choices:
// List  -> ArrayList
// Set   -> HashSet
// Map   -> HashMap
// Queue -> ArrayDeque
// Stack -> ArrayDeque

// ✅ Pre-size when you know the element count
// ✅ Use primitive arrays/streams to avoid boxing overhead
// ✅ Profile with JMH before optimizing (don't guess!)
// ✅ Consider cache locality (array-based > pointer-based)
// ✅ Use EnumSet/EnumMap for enum keys (specialized & fast)
// ✅ Use entrySet() to iterate maps, not keySet() + get()
```

---

## Interview Tips
- Know the Big-O for ALL operations across ALL major implementations
- Explain WHY ArrayList beats LinkedList (cache locality, not just Big-O)
- Mention memory overhead differences between implementations
- Know when HashMap degrades from O(1) to O(log n) (collisions + treeification)
- Be ready to recommend the right collection for a specific use case
- Mention JMH as the standard for Java microbenchmarking

---

[Prev: Q17 - Immutable Collections](./q17-immutable-collections.md) | [Next: Q19 - Thread-Safe Collections](./q19-thread-safe-collections.md)

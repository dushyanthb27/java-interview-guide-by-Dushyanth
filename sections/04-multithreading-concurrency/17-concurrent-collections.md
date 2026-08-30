# Q17: What Concurrent Collections Are Available?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What concurrent collections does Java provide? When should you use each one?

---

## Answer

### Overview of Concurrent Collections

```
┌──────────────────────────────────────────────────────────┐
│                java.util.concurrent                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Maps:        ConcurrentHashMap                          │
│               ConcurrentSkipListMap (sorted)             │
│                                                          │
│  Lists:       CopyOnWriteArrayList                       │
│                                                          │
│  Sets:        CopyOnWriteArraySet                        │
│               ConcurrentSkipListSet (sorted)             │
│                                                          │
│  Queues:      ConcurrentLinkedQueue                      │
│               ConcurrentLinkedDeque                      │
│                                                          │
│  Blocking     ArrayBlockingQueue (bounded)               │
│  Queues:      LinkedBlockingQueue (optionally bounded)   │
│               PriorityBlockingQueue                      │
│               DelayQueue                                 │
│               SynchronousQueue                           │
│               LinkedTransferQueue                        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### ConcurrentHashMap

```java
public class ConcurrentHashMapDemo {
    // ✅ Thread-safe, high-performance concurrent map
    private final ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

    public void basicOps() {
        // Standard operations (all thread-safe)
        map.put("key1", 1);
        map.get("key1");
        map.remove("key1");

        // Atomic compound operations
        map.putIfAbsent("key2", 2);        // Put only if key doesn't exist
        map.replace("key2", 2, 3);         // Replace if current value matches
        map.remove("key2", 3);             // Remove if value matches

        // Atomic compute operations (Java 8+)
        map.compute("counter", (key, val) -> val == null ? 1 : val + 1);
        map.computeIfAbsent("cache", key -> expensiveCompute(key));
        map.computeIfPresent("counter", (key, val) -> val + 1);
        map.merge("counter", 1, Integer::sum); // merge with sum function
    }

    // Bulk operations (Java 8+) - parallel by nature
    public void bulkOps() {
        map.put("a", 1); map.put("b", 2); map.put("c", 3);

        // forEach with parallelism threshold
        map.forEach(2, // parallelism threshold (parallelize if size > 2)
            (key, value) -> System.out.println(key + "=" + value));

        // search: find first matching entry
        String found = map.search(1, (key, value) -> value > 2 ? key : null);

        // reduce: aggregate all values
        int sum = map.reduce(1,
            (key, value) -> value,  // transformer
            Integer::sum);          // reducer

        System.out.println("Sum: " + sum);
    }

    private int expensiveCompute(String key) {
        return key.hashCode();
    }
}
```

**ConcurrentHashMap vs synchronized HashMap vs Hashtable:**

```
┌──────────────────┬─────────────────┬──────────────────┬──────────────┐
│ Feature          │ ConcurrentHMap  │ sync'd HashMap   │ Hashtable    │
├──────────────────┼─────────────────┼──────────────────┼──────────────┤
│ Null keys/values │ ❌ Neither      │ ✅ Both          │ ❌ Neither   │
│ Lock granularity │ Segment/bucket  │ Entire map       │ Entire map   │
│ Iterator         │ Weakly consist. │ Fail-fast        │ Fail-fast    │
│ Performance      │ ⭐⭐⭐         │ ⭐               │ ⭐           │
│ Compound atomic  │ ✅ compute etc  │ ❌               │ ❌           │
└──────────────────┴─────────────────┴──────────────────┴──────────────┘
```

### CopyOnWriteArrayList

```java
public class CopyOnWriteDemo {
    // Creates a new copy of the array on every write
    // ✅ Great for: read-heavy, write-rare scenarios (e.g., listener lists)
    private final CopyOnWriteArrayList<String> listeners = new CopyOnWriteArrayList<>();

    public void addListener(String listener) {
        listeners.add(listener); // Creates a new internal array (expensive!)
    }

    public void removeListener(String listener) {
        listeners.remove(listener);
    }

    // Reads are lock-free and never throw ConcurrentModificationException
    public void notifyAll(String event) {
        for (String listener : listeners) { // Safe iteration, never fails
            System.out.println("Notifying " + listener + ": " + event);
        }
    }

    // Iterator reflects the state at creation time (snapshot)
    public void iteratorDemo() {
        listeners.addAll(List.of("A", "B", "C"));
        Iterator<String> it = listeners.iterator(); // Snapshot

        listeners.add("D"); // Doesn't affect the iterator

        while (it.hasNext()) {
            System.out.println(it.next()); // Prints A, B, C (not D)
        }
        // it.remove(); // ❌ UnsupportedOperationException
    }
}
```

### BlockingQueue Implementations

```java
public class BlockingQueueDemo {

    // 1. ArrayBlockingQueue - bounded, backed by array
    BlockingQueue<String> arrayBQ = new ArrayBlockingQueue<>(100);

    // 2. LinkedBlockingQueue - optionally bounded, backed by linked nodes
    BlockingQueue<String> linkedBQ = new LinkedBlockingQueue<>(1000);
    BlockingQueue<String> unboundedBQ = new LinkedBlockingQueue<>(); // Caution!

    // 3. PriorityBlockingQueue - unbounded, priority ordering
    BlockingQueue<Task> priorityBQ = new PriorityBlockingQueue<>(10,
        Comparator.comparingInt(Task::priority));

    // 4. SynchronousQueue - zero capacity, direct handoff
    BlockingQueue<String> syncQueue = new SynchronousQueue<>();
    // put() blocks until take() is called (and vice versa)

    // 5. DelayQueue - elements available only after delay
    // DelayQueue<DelayedTask> delayQueue = new DelayQueue<>();

    record Task(String name, int priority) {}

    public void producerConsumer() throws InterruptedException {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);

        // Producer
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 100; i++) {
                    queue.put(i);  // Blocks if queue is full
                    System.out.println("Produced: " + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        // Consumer
        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    Integer item = queue.take(); // Blocks if queue is empty
                    System.out.println("Consumed: " + item);
                    Thread.sleep(50); // Process item
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
    }
}
```

### BlockingQueue Operations Summary

```
┌──────────────┬──────────┬───────────┬──────────┬─────────────┐
│ Operation    │ Throws   │ Returns   │ Blocks   │ Times out   │
│              │ Exception│ Special   │          │             │
├──────────────┼──────────┼───────────┼──────────┼─────────────┤
│ Insert       │ add(e)   │ offer(e)  │ put(e)   │ offer(e,t)  │
│ Remove       │ remove() │ poll()    │ take()   │ poll(t)     │
│ Examine      │ element()│ peek()    │  N/A     │  N/A        │
└──────────────┴──────────┴───────────┴──────────┴─────────────┘
```

### ConcurrentLinkedQueue

```java
public class ConcurrentLinkedQueueDemo {
    // Non-blocking, unbounded, thread-safe queue (CAS-based)
    private final ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();

    public void produce(String item) {
        queue.offer(item); // Never blocks, always succeeds
    }

    public String consume() {
        return queue.poll(); // Returns null if empty (never blocks)
    }

    // Use when: you need a non-blocking concurrent queue
    // Don't use when: you need blocking behavior (use BlockingQueue)
}
```

### Weakly Consistent Iterators

```java
public class WeakConsistencyDemo {
    public static void main(String[] args) {
        // Concurrent collections use "weakly consistent" iterators:
        // - Do NOT throw ConcurrentModificationException
        // - May or may not reflect concurrent modifications
        // - Guaranteed to traverse elements as they existed at construction

        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
        map.put("a", 1);
        map.put("b", 2);

        // Safe to modify during iteration
        for (Map.Entry<String, Integer> entry : map.entrySet()) {
            map.put("c", 3); // ✅ No exception (unlike HashMap)
            System.out.println(entry.getKey() + "=" + entry.getValue());
        }

        // Compare with fail-fast (HashMap)
        // Map<String, Integer> hashMap = new HashMap<>(map);
        // for (var entry : hashMap.entrySet()) {
        //     hashMap.put("d", 4); // ❌ ConcurrentModificationException!
        // }
    }
}
```

### When to Use What

```
┌────────────────────────────┬──────────────────────────────────┐
│ Use Case                   │ Best Collection                  │
├────────────────────────────┼──────────────────────────────────┤
│ Concurrent key-value store │ ConcurrentHashMap                │
│ Sorted concurrent map      │ ConcurrentSkipListMap            │
│ Read-heavy listener list   │ CopyOnWriteArrayList             │
│ Producer-consumer pattern  │ ArrayBlockingQueue               │
│ Unbounded producer-consumer│ LinkedBlockingQueue              │
│ Priority-based processing  │ PriorityBlockingQueue            │
│ Thread pool work queue     │ SynchronousQueue                 │
│ Delayed task execution     │ DelayQueue                       │
│ Non-blocking queue         │ ConcurrentLinkedQueue            │
│ Concurrent sorted set      │ ConcurrentSkipListSet            │
└────────────────────────────┴──────────────────────────────────┘
```

---

## Anti-Patterns

❌ **Using `Collections.synchronizedMap()` when `ConcurrentHashMap` fits:**
```java
Map<String, String> map = Collections.synchronizedMap(new HashMap<>()); // ❌ Coarse locking
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();      // ✅ Fine-grained
```

❌ **Check-then-act without atomic operations:**
```java
if (!map.containsKey("key")) {
    map.put("key", value); // ❌ Race condition!
}
map.putIfAbsent("key", value); // ✅ Atomic
```

❌ **Using CopyOnWriteArrayList for write-heavy workloads**

## Best Practices

✅ Use `ConcurrentHashMap.computeIfAbsent()` for thread-safe lazy caching  
✅ Use `BlockingQueue` for producer-consumer (not manual wait/notify)  
✅ Use `CopyOnWriteArrayList` only when reads vastly outnumber writes  
✅ Prefer `putIfAbsent`, `compute`, `merge` over check-then-act patterns  
✅ Size bounded queues appropriately to prevent memory issues

---

[← Parallel Streams](./q16-parallel-streams.md) | [Next: Fork/Join Framework →](./q18-fork-join-framework.md)

# Q12: What is Fail-Fast vs Fail-Safe Behavior?

[Back to Collections Framework](./README.md) | **Difficulty**: 🟠 Hard

---

## Question
> What is the difference between fail-fast and fail-safe iterators in Java collections?

---

## Answer

### Fail-Fast Iterators

Fail-fast iterators throw **ConcurrentModificationException** when the collection is structurally modified during iteration (except through the iterator's own methods).

```java
// ArrayList, HashMap, HashSet, LinkedList, etc. have fail-fast iterators

List<String> list = new ArrayList<>(List.of("A", "B", "C", "D"));

// ❌ ConcurrentModificationException!
for (String item : list) {
    if (item.equals("B")) {
        list.remove(item);  // Structural modification during iteration!
    }
}

// ❌ Also fails with explicit iterator
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    it.next();
    list.add("X");  // ConcurrentModificationException!
}
```

### How Fail-Fast Works: The modCount Mechanism

```java
// Internally, collections maintain a modCount (modification count)
// The iterator captures modCount at creation time (expectedModCount)
// Before each next()/remove(), iterator checks: modCount == expectedModCount

// Simplified internal implementation:
public class ArrayList<E> {
    protected transient int modCount = 0;  // Incremented on structural changes

    public boolean add(E e) {
        modCount++;              // Increment on add
        // ... add element ...
        return true;
    }

    public E remove(int index) {
        modCount++;              // Increment on remove
        // ... remove element ...
        return oldValue;
    }

    private class Itr implements Iterator<E> {
        int expectedModCount = modCount;  // Snapshot at iterator creation

        public E next() {
            checkForComodification();      // Check before every next()
            // ... return element ...
        }

        public void remove() {
            checkForComodification();
            // ... remove element ...
            expectedModCount = modCount;   // Sync after iterator's own remove
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
}
```

### What Triggers ConcurrentModificationException

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
Iterator<String> it = list.iterator();

// Structural modifications that increment modCount:
list.add("D");          // ✅ modCount++ (add)
list.remove("A");       // ✅ modCount++ (remove)
list.clear();           // ✅ modCount++ (clear)
list.addAll(otherList); // ✅ modCount++ (addAll)
list.set(0, "X");       // ❌ modCount NOT changed (not structural)
// set() replaces an element without changing structure

it.next();  // ConcurrentModificationException! (if any of the above ran)

// ✅ Iterator's own remove() is safe - it syncs expectedModCount
Iterator<String> it2 = list.iterator();
it2.next();
it2.remove();  // Safe! Updates expectedModCount internally
```

### Fail-Safe (Weakly Consistent) Iterators

Fail-safe iterators **never throw ConcurrentModificationException**. They work on a copy or snapshot of the collection, or provide weak consistency guarantees.

```java
// CopyOnWriteArrayList - iterates over a SNAPSHOT
CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>(List.of("A", "B", "C"));

Iterator<String> it = cowList.iterator();  // Snapshot taken here!
cowList.add("D");           // Modifies the list (creates new internal array)
cowList.remove("A");        // Modifies again

while (it.hasNext()) {
    System.out.println(it.next());  // Still prints: A, B, C (from snapshot)
}
// No exception! Iterator sees the old snapshot
// Current list is now: [B, C, D]

// ConcurrentHashMap - weakly consistent iterator
ConcurrentHashMap<String, Integer> chm = new ConcurrentHashMap<>();
chm.put("A", 1);
chm.put("B", 2);
chm.put("C", 3);

Iterator<String> chIt = chm.keySet().iterator();
chm.put("D", 4);      // Concurrent modification
chm.remove("A");       // Concurrent modification

while (chIt.hasNext()) {
    System.out.println(chIt.next());
    // May or may not see "D", may or may not see "A"
    // But NO exception and each element seen AT MOST once
}
```

### Copy-On-Write Collections

```java
// CopyOnWriteArrayList: creates a new copy of the array on EVERY write
CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();

// Write operations (expensive - O(n) each, full array copy)
cowList.add("A");       // Creates new array [A]
cowList.add("B");       // Creates new array [A, B]
cowList.set(0, "X");    // Creates new array [X, B]
cowList.remove("B");    // Creates new array [X]

// Read operations (fast - no locking, no copying)
String item = cowList.get(0);    // Direct array access, no lock
boolean has = cowList.contains("X");  // No lock

// Iteration (safe - works on snapshot)
for (String s : cowList) {
    cowList.add("Y");  // ✅ No exception! Iterator uses old snapshot
    System.out.println(s);
}

// ⚠️ Iterator does NOT support remove/set/add
Iterator<String> it = cowList.iterator();
it.next();
it.remove();  // ❌ UnsupportedOperationException!

// CopyOnWriteArraySet: backed by CopyOnWriteArrayList
Set<String> cowSet = new CopyOnWriteArraySet<>();
cowSet.add("A");
cowSet.add("B");
// Same copy-on-write semantics
```

### When to Use Each

```java
// ✅ Fail-fast (ArrayList, HashMap, etc.) - DEFAULT choice
// - Single-threaded access
// - Multi-threaded with external synchronization
// - Detects programming errors early
List<String> list = new ArrayList<>();

// ✅ CopyOnWriteArrayList - Read-heavy, few writes
// - Event listener lists (register once, notify many)
// - Configuration lists (set once, read often)
// - Observer pattern implementations
List<EventListener> listeners = new CopyOnWriteArrayList<>();

// ✅ ConcurrentHashMap - Concurrent read/write access
// - Shared caches
// - Concurrent counters
// - Thread-safe registries
ConcurrentHashMap<String, Service> registry = new ConcurrentHashMap<>();

// ✅ Collections.synchronizedList - Moderate concurrent access
// - Must externally synchronize iteration
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
synchronized (syncList) {
    for (String s : syncList) {
        // Safe iteration with external sync
    }
}
```

### Safe Removal Patterns

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C", "D", "E"));

// ✅ Pattern 1: Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("C")) {
        it.remove();
    }
}

// ✅ Pattern 2: removeIf() (Java 8+) - cleanest approach
list.removeIf(s -> s.equals("C"));

// ✅ Pattern 3: Collect and remove after iteration
List<String> toRemove = new ArrayList<>();
for (String s : list) {
    if (s.equals("C")) {
        toRemove.add(s);
    }
}
list.removeAll(toRemove);

// ✅ Pattern 4: Backward iteration with index (for indexed lists)
for (int i = list.size() - 1; i >= 0; i--) {
    if (list.get(i).equals("C")) {
        list.remove(i);
    }
}

// ✅ Pattern 5: Stream filter to new list
List<String> filtered = list.stream()
    .filter(s -> !s.equals("C"))
    .collect(Collectors.toList());
```

### Safe Removal from Maps

```java
Map<String, Integer> map = new HashMap<>();
map.put("A", 1); map.put("B", 2); map.put("C", 3);

// ❌ Fails!
for (String key : map.keySet()) {
    if (map.get(key) < 3) {
        map.remove(key);  // ConcurrentModificationException!
    }
}

// ✅ Use entrySet iterator
Iterator<Map.Entry<String, Integer>> it = map.entrySet().iterator();
while (it.hasNext()) {
    if (it.next().getValue() < 3) {
        it.remove();
    }
}

// ✅ Use removeIf on entrySet (Java 8+)
map.entrySet().removeIf(entry -> entry.getValue() < 3);

// ✅ Use Map.values().removeIf()
map.values().removeIf(value -> value < 3);
```

### Multi-Threaded Fail-Fast Scenario

```java
// Fail-fast also detects concurrent modification from OTHER threads
// (but this is a best-effort detection, not guaranteed)

List<String> sharedList = new ArrayList<>(List.of("A", "B", "C"));

// Thread 1: iterating
new Thread(() -> {
    for (String s : sharedList) {
        System.out.println(s);
        try { Thread.sleep(100); } catch (InterruptedException e) {}
    }
    // May throw ConcurrentModificationException!
}).start();

// Thread 2: modifying
new Thread(() -> {
    try { Thread.sleep(50); } catch (InterruptedException e) {}
    sharedList.add("D");  // Structural modification!
}).start();

// ⚠️ IMPORTANT: ConcurrentModificationException is NOT guaranteed
// in multi-threaded scenarios. modCount check is not atomic.
// It's a "best-effort" detection mechanism.
// For true thread safety, use concurrent collections!
```

### Common Anti-Patterns

```java
// ❌ Modifying collection in for-each loop
for (String s : list) {
    if (condition(s)) list.remove(s);
}

// ❌ Assuming ConcurrentModificationException means thread-safety issue
// It can happen in single-threaded code too!

// ❌ Using CopyOnWriteArrayList for write-heavy workloads
// Each write creates a full copy of the array - O(n) per write!

// ❌ Catching ConcurrentModificationException and continuing
try {
    for (String s : list) { list.remove(s); }
} catch (ConcurrentModificationException e) {
    // This is a BUG, not something to catch and ignore!
}

// ❌ Relying on fail-fast for program correctness
// It's a debugging aid, not a correctness guarantee
```

### Best Practices

```java
// ✅ Use removeIf() for conditional removal (Java 8+)
list.removeIf(item -> item.isExpired());

// ✅ Use Iterator.remove() for pre-Java-8 removal
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (shouldRemove(it.next())) it.remove();
}

// ✅ Use CopyOnWriteArrayList for event listener patterns
List<ChangeListener> listeners = new CopyOnWriteArrayList<>();

// ✅ Use ConcurrentHashMap for shared maps
Map<String, Object> cache = new ConcurrentHashMap<>();

// ✅ Treat ConcurrentModificationException as a bug to fix, not to catch

// ✅ Document thread-safety expectations in your APIs
/**
 * This collection is not thread-safe. Callers must synchronize
 * externally if accessed from multiple threads.
 */
```

---

## Quick Reference

| Collection | Iterator Type | Behavior on Modification |
|------------|--------------|-------------------------|
| ArrayList | Fail-fast | ConcurrentModificationException |
| LinkedList | Fail-fast | ConcurrentModificationException |
| HashMap | Fail-fast | ConcurrentModificationException |
| HashSet | Fail-fast | ConcurrentModificationException |
| TreeMap | Fail-fast | ConcurrentModificationException |
| CopyOnWriteArrayList | Snapshot | Iterates over old copy |
| CopyOnWriteArraySet | Snapshot | Iterates over old copy |
| ConcurrentHashMap | Weakly consistent | May/may not see changes |
| ConcurrentLinkedQueue | Weakly consistent | May/may not see changes |

---

## Interview Tips
- Fail-fast uses modCount to detect structural modifications
- Fail-safe either copies (CopyOnWrite) or is weakly consistent (Concurrent*)
- `set()` on ArrayList does NOT increment modCount (not structural)
- ConcurrentModificationException is best-effort, not guaranteed in multi-threaded code
- CopyOnWrite iterators don't support remove/set/add (UnsupportedOperationException)
- Know all safe removal patterns (Iterator.remove, removeIf, backward loop)

---

[Prev: Q11 - Iterator vs ListIterator](./q11-iterator-listiterator.md) | [Next: Q13 - Comparable vs Comparator](./q13-comparable-comparator.md)

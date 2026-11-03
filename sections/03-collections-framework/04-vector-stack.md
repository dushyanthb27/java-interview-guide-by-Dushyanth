# Vector and Stack

[Home](../../README.md) > [Collections Framework](./README.md) > Vector and Stack

---

## Q4 🟡 Are Vector and Stack Still Relevant?

**Question:** Are Vector and Stack still relevant in modern Java? What should be used instead?

### Answer

Vector and Stack are legacy classes from Java 1.0 that predate the Collections Framework (Java 1.2). They're still in the JDK for backward compatibility, but you should never use them in new code. 

Understanding why they're legacy and what to use instead is a common interview topic.

### Vector — The Synchronized ArrayList

Vector is essentially a synchronized ArrayList. Every method is synchronized. This makes it thread-safe but slow:

```java
// Vector internal structure (simplified)
public class Vector<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, Serializable {

    protected Object[] elementData;     // backing array
    protected int elementCount;         // number of elements
    protected int capacityIncrement;    // growth amount (0 = double)

    // EVERY method is synchronized
    public synchronized boolean add(E e) { ... }
    public synchronized E get(int index) { ... }
    public synchronized E remove(int index) { ... }
    public synchronized int size() { ... }
    public synchronized boolean contains(Object o) { ... }
}
```

### Key Differences — Vector vs ArrayList

```java
// 1. Synchronization
// Vector: Every method is synchronized (thread-safe but slow)
// ArrayList: Not synchronized (fast but not thread-safe)

// 2. Growth Strategy
// Vector: Doubles capacity (2x) or grows by capacityIncrement
// ArrayList: Grows by 50% (1.5x) - more memory efficient

// 3. Legacy Methods
Vector<String> v = new Vector<>();
v.addElement("A");       // Legacy method (use add() instead)
v.elementAt(0);          // Legacy method (use get() instead)
v.removeElementAt(0);    // Legacy method (use remove() instead)
Enumeration<String> e = v.elements();  // Legacy (use iterator() instead)

// 4. Initial Capacity
// Vector: default capacity = 10, can specify capacityIncrement
// ArrayList: default capacity = 10, grows by 1.5x
Vector<String> v = new Vector<>(20, 10);  // initial=20, increment=10
```

### Why Vector is Legacy

Vector has three fundamental problems that make it unsuitable for modern code:

```java
// ❌ Problem 1: Synchronization is too coarse-grained
// Every single method call acquires a lock, even when no contention exists
Vector<String> v = new Vector<>();
v.add("A");     // acquires lock
v.size();       // acquires lock
v.get(0);       // acquires lock
// Three separate lock acquisitions for simple operations!

// ❌ Problem 2: Compound operations are NOT thread-safe despite synchronization
Vector<String> v = new Vector<>();
// This is NOT thread-safe even with Vector!
if (!v.contains("item")) {   // Lock acquired and released
    v.add("item");            // Lock acquired and released
    // Another thread could add "item" between these two calls!
}

// ❌ Problem 3: Poor performance under contention
// Method-level synchronization causes thread contention
// All operations on the vector are serialized
```

### Stack — Extends Vector (Legacy)

Stack extends Vector, which is a textbook example of bad inheritance design:

```java
// Stack extends Vector (bad inheritance design)
public class Stack<E> extends Vector<E> {
    public E push(E item) { ... }     // synchronized (inherited)
    public E pop() { ... }            // synchronized
    public E peek() { ... }           // synchronized
    public boolean empty() { ... }    // synchronized
    public int search(Object o) { ... }
}

// ❌ Problem: Stack IS-A Vector, so you can do this:
Stack<String> stack = new Stack<>();
stack.push("A");
stack.push("B");
stack.add(0, "C");     // ❌ Violates stack contract! Inserting at bottom
stack.remove(1);        // ❌ Violates stack contract! Removing from middle
stack.get(0);           // ❌ Violates stack contract! Random access
// This is a classic violation of the Liskov Substitution Principle
```

### Modern Replacements

Here's what you should use instead:

```java
// ✅ Replacement for Vector -> ArrayList
List<String> list = new ArrayList<>();

// ✅ Replacement for Vector (when thread-safety needed)
// Option 1: Collections.synchronizedList
List<String> syncList = Collections.synchronizedList(new ArrayList<>());

// Option 2: CopyOnWriteArrayList (read-heavy scenarios)
List<String> cowList = new CopyOnWriteArrayList<>();

// ✅ Replacement for Stack -> ArrayDeque
Deque<String> stack = new ArrayDeque<>();
stack.push("A");       // addFirst
stack.push("B");       // addFirst
String top = stack.pop();    // removeFirst -> "B"
String peek = stack.peek();  // peekFirst -> "A"

// ✅ Replacement for Stack (when thread-safety needed)
// Option 1: ConcurrentLinkedDeque
Deque<String> concurrentStack = new ConcurrentLinkedDeque<>();

// Option 2: Synchronized wrapper
Deque<String> syncStack = Collections.synchronizedDeque(new ArrayDeque<>());
// Note: synchronizedDeque is not available; use explicit synchronization
```

### Thread Safety Trade-offs

If you need thread-safe collections, choose the right tool for your use case:

```java
// Approach 1: Collections.synchronizedList (similar to Vector)
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
// Must manually synchronize compound operations:
synchronized (syncList) {
    if (!syncList.contains("item")) {
        syncList.add("item");
    }
}
// Must synchronize during iteration:
synchronized (syncList) {
    for (String s : syncList) {
        process(s);
    }
}

// Approach 2: CopyOnWriteArrayList (read-heavy, few writes)
List<String> cowList = new CopyOnWriteArrayList<>();
cowList.add("item");     // Creates a new internal array copy (expensive)
cowList.get(0);          // No locking needed (reads from snapshot)
for (String s : cowList) {
    // Safe - iterates over snapshot, no ConcurrentModificationException
    process(s);
}

// Approach 3: Manual synchronization with ArrayList
List<String> list = new ArrayList<>();
ReadWriteLock lock = new ReentrantReadWriteLock();

// Write
lock.writeLock().lock();
try {
    list.add("item");
} finally {
    lock.writeLock().unlock();
}

// Read
lock.readLock().lock();
try {
    String item = list.get(0);
} finally {
    lock.readLock().unlock();
}
```

### ArrayDeque vs Stack vs LinkedList (as Stack)

ArrayDeque is the clear winner for stack operations:

```java
// Performance comparison for stack operations:
//
// ArrayDeque:
//   - Backed by resizable circular array
//   - No synchronization overhead
//   - Cache-friendly (contiguous memory)
//   - Cannot store null
//   - Fastest option

// Stack (extends Vector):
//   - Backed by synchronized resizable array
//   - Synchronization overhead on every operation
//   - Allows random access (violates stack contract)
//   - Can store null

// LinkedList (as Deque):
//   - Node-based, scattered memory
//   - No synchronization overhead
//   - Higher memory overhead (Node objects)
//   - Cache-unfriendly
//   - Can store null

// Benchmark results (relative):
// ArrayDeque.push/pop:   ~1x   (baseline)
// Stack.push/pop:        ~3-5x (synchronization overhead)
// LinkedList.push/pop:   ~2-3x (memory allocation + cache misses)
```

### Migration Guide

Here's how to migrate legacy code:

```java
// FROM: Legacy Vector code
Vector<String> names = new Vector<>();
names.addElement("Alice");
names.addElement("Bob");
Enumeration<String> e = names.elements();
while (e.hasMoreElements()) {
    System.out.println(e.nextElement());
}

// TO: Modern ArrayList code
List<String> names = new ArrayList<>();
names.add("Alice");
names.add("Bob");
for (String name : names) {
    System.out.println(name);
}
// Or: names.forEach(System.out::println);

// FROM: Legacy Stack code
Stack<Integer> stack = new Stack<>();
stack.push(1);
stack.push(2);
while (!stack.empty()) {
    System.out.println(stack.pop());
}

// TO: Modern ArrayDeque code
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);
stack.push(2);
while (!stack.isEmpty()) {
    System.out.println(stack.pop());
}
```

### When You Might Still Encounter Vector/Stack

You'll see these in legacy codebases and certain APIs:

```java
// 1. Legacy codebases (pre-Java 1.2 systems)
// 2. Certain Java APIs that return Vector
//    - Some JDBC drivers return Vector
//    - Swing's DefaultListModel uses Vector internally
// 3. Android development (some older Android APIs)
// 4. Interview questions specifically about legacy collections!

// ✅ If you encounter Vector in legacy code, you can wrap it:
Vector<String> legacyVector = getLegacyData();
List<String> modernList = new ArrayList<>(legacyVector);  // Copy to ArrayList
```

### Common Anti-Patterns

```java
// ❌ Using Vector "for thread safety"
Vector<String> data = new Vector<>();  // False sense of security

// ❌ Using Stack for stack operations
Stack<String> stack = new Stack<>();   // Exposes non-stack methods

// ❌ Assuming Vector's compound operations are atomic
Vector<String> v = new Vector<>();
// NOT thread-safe:
if (v.size() > 0) {        // Lock acquired/released
    v.remove(v.size() - 1); // Lock acquired/released - size may have changed!
}
```

### Best Practices

```java
// ✅ Use ArrayList as default List implementation
List<String> list = new ArrayList<>();

// ✅ Use ArrayDeque for stack/queue operations
Deque<String> stack = new ArrayDeque<>();
Queue<String> queue = new ArrayDeque<>();

// ✅ Use CopyOnWriteArrayList for concurrent read-heavy lists
List<String> listeners = new CopyOnWriteArrayList<>();

// ✅ Use ConcurrentLinkedDeque for concurrent stack needs
Deque<String> concurrentStack = new ConcurrentLinkedDeque<>();

// ✅ If thread-safety is needed, choose the right tool:
// - Few writes, many reads: CopyOnWriteArrayList
// - Balanced reads/writes: Collections.synchronizedList + external sync
// - High concurrency: Consider lock-free structures
```

> **🎯 Interviewer Insight:** Vector and Stack are LEGACY — never recommend them in new code. Explain WHY they're legacy (coarse-grained synchronization, design flaws). Know the replacements — ArrayList, ArrayDeque, CopyOnWriteArrayList. Stack extends Vector is a textbook example of bad inheritance (LSP violation). Compound operations on Vector are NOT atomic despite synchronization.

> **⚠️ Common Mistake:** Using Vector "for thread safety" without understanding that compound operations (check-then-act) are still not atomic. Vector's method-level synchronization gives a false sense of security.

> **🔧 Performance Tip:** Use ArrayDeque instead of Stack for stack operations — it's 3-5x faster due to no synchronization overhead and better cache locality. Use ArrayList instead of Vector unless you truly need synchronization, and even then, consider CopyOnWriteArrayList or manual synchronization.

> **⚡ Follow-up:** Why does Stack extending Vector violate the Liskov Substitution Principle?

> **⚡ Follow-up:** When would you choose CopyOnWriteArrayList over Collections.synchronizedList?

> **⚡ Follow-up:** How does the 2x growth factor of Vector compare to ArrayList's 1.5x factor in terms of memory efficiency?

> **⚡ Follow-up:** What are the performance implications of Vector's coarse-grained synchronization under high contention?

> **📚 Related:** [ArrayList Internals](./02-arraylist.md), [HashSet Internals](./05-hashset.md), [Thread-Safe Collections](./19-thread-safe-collections.md)

---

[← Previous: LinkedList Internals](./03-linkedlist.md) · [Next: HashSet Internals →](./05-hashset.md)

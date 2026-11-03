# Collection Hierarchy

[Home](../../README.md) > [Collections Framework](./README.md) > Collection Hierarchy

---

## Q1 🟢 Explain the Java Collection Hierarchy

**Question:** Explain the Java Collection hierarchy. What are the key interfaces and their contracts?

### Answer

The Java Collections Framework is built around a hierarchy of interfaces that define contracts for different types of data structures. At the root is `Iterable`, which enables for-each loop support. Everything extends from there — except `Map`, which lives in its own separate hierarchy.

Here's the complete structure:

```
                        Iterable<T>
                            |
                        Collection<T>
                       /      |       \
                     /        |         \
                 List<T>    Set<T>     Queue<T>
                  |          |    \        |
                  |          |     \    Deque<T>
                  |          |      \
              ArrayList   HashSet  SortedSet<T>
              LinkedList  LinkedHashSet  |
              Vector         |      NavigableSet<T>
                             |          |
                          TreeSet    TreeSet

        Map<K,V> (separate hierarchy - NOT a Collection)
           |         \           \
      HashMap    SortedMap<K,V>   Hashtable
      LinkedHashMap    |
           |     NavigableMap<K,V>
           |           |
        WeakHashMap  TreeMap
        ConcurrentHashMap
```

Starting from the top — `Iterable<T>` is the foundation that makes any collection work with enhanced for-loops:

```java
public interface Iterable<T> {
    Iterator<T> iterator();

    // Java 8+
    default void forEach(Consumer<? super T> action) { ... }
    default Spliterator<T> spliterator() { ... }
}
```

Next is `Collection<T>`, the base interface for all collections except Map. It defines the core operations that every collection must support:

```java
public interface Collection<T> extends Iterable<T> {
    // Basic operations
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    boolean add(T e);
    boolean remove(Object o);

    // Bulk operations
    boolean containsAll(Collection<?> c);
    boolean addAll(Collection<? extends T> c);
    boolean removeAll(Collection<?> c);
    boolean retainAll(Collection<?> c);
    void clear();

    // Array operations
    Object[] toArray();
    <T> T[] toArray(T[] a);

    // Java 8+
    default Stream<T> stream() { ... }
    default Stream<T> parallelStream() { ... }
    default boolean removeIf(Predicate<? super T> filter) { ... }
}
```

From here, the hierarchy splits into the three main collection types — `List`, `Set`, and `Queue`.

**List<T>** represents ordered sequences that allow duplicates and provide positional access:

```java
public interface List<T> extends Collection<T> {
    // Positional access
    T get(int index);
    T set(int index, T element);
    void add(int index, T element);
    T remove(int index);

    // Search
    int indexOf(Object o);
    int lastIndexOf(Object o);

    // Range view
    List<T> subList(int fromIndex, int toIndex);

    // List-specific iterator
    ListIterator<T> listIterator();
    ListIterator<T> listIterator(int index);

    // Java 9+
    static <E> List<E> of(E... elements) { ... }

    // Java 10+
    static <E> List<E> copyOf(Collection<? extends E> coll) { ... }
}
```

**Set<T>** represents collections with unique elements — no duplicates allowed based on `equals()`:

```java
public interface Set<T> extends Collection<T> {
    // Inherits Collection methods
    // Contract: no duplicate elements (based on equals())
    // At most one null element (implementation-dependent)

    // Java 9+
    static <E> Set<E> of(E... elements) { ... }
}
```

**Queue<T>** represents FIFO (typically) ordering with two sets of methods — one that throws exceptions on failure and one that returns special values:

```java
public interface Queue<T> extends Collection<T> {
    // Throws exception on failure
    boolean add(T e);
    T remove();
    T element();

    // Returns special value on failure
    boolean offer(T e);   // returns false
    T poll();              // returns null
    T peek();              // returns null
}
```

**Deque<T>** extends Queue to support double-ended operations — you can add and remove from both ends:

```java
public interface Deque<T> extends Queue<T> {
    void addFirst(T e);
    void addLast(T e);
    T removeFirst();
    T removeLast();
    T peekFirst();
    T peekLast();

    // Stack operations
    void push(T e);   // addFirst
    T pop();           // removeFirst
}
```

And finally, **Map<K,V>** — the odd one out. It doesn't extend Collection at all because it stores key-value pairs, not single elements:

```java
public interface Map<K, V> {
    V get(Object key);
    V put(K key, V value);
    V remove(Object key);
    boolean containsKey(Object key);
    boolean containsValue(Object value);
    int size();
    Set<K> keySet();
    Collection<V> values();
    Set<Map.Entry<K,V>> entrySet();

    // Java 8+
    default V getOrDefault(Object key, V defaultValue) { ... }
    default V putIfAbsent(K key, V value) { ... }
    default V computeIfAbsent(K key, Function<? super K, ? extends V> f) { ... }
    default V computeIfPresent(K key, BiFunction<? super K, ? super V, ? extends V> f) { ... }
    default void forEach(BiConsumer<? super K, ? super V> action) { ... }
    default V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> f) { ... }
}
```

### Sequenced Collections (Java 21)

Java 21 introduced the Sequenced Collections API to unify first/last element access across Lists, Sets, and Maps. Before this, each collection type had different methods for accessing the first and last elements.

This new API provides a consistent way to work with ordered collections:

```java
// New interfaces in Java 21
public interface SequencedCollection<E> extends Collection<E> {
    SequencedCollection<E> reversed();
    void addFirst(E e);
    void addLast(E e);
    E getFirst();
    E getLast();
    E removeFirst();
    E removeLast();
}

public interface SequencedSet<E> extends Set<E>, SequencedCollection<E> {
    SequencedSet<E> reversed();
}

public interface SequencedMap<K, V> extends Map<K, V> {
    SequencedMap<K, V> reversed();
    Map.Entry<K, V> firstEntry();
    Map.Entry<K, V> lastEntry();
    Map.Entry<K, V> pollFirstEntry();
    Map.Entry<K, V> pollLastEntry();
    V putFirst(K key, V value);
    V putLast(K key, V value);
    SequencedSet<K> sequencedKeySet();
    SequencedCollection<V> sequencedValues();
    SequencedSet<Map.Entry<K, V>> sequencedEntrySet();
}
```

Here's how it looks in practice:

```java
// Java 21 usage examples
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
String first = list.getFirst();  // "A"
String last  = list.getLast();   // "C"

List<String> reversed = list.reversed(); // ["C", "B", "A"] (view)

LinkedHashSet<String> set = new LinkedHashSet<>(Set.of("X", "Y", "Z"));
set.getFirst();   // first inserted element
set.getLast();    // last inserted element

LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
map.put("one", 1);
map.put("two", 2);
map.firstEntry();  // one=1
map.lastEntry();   // two=2
```

### Choosing the Right Collection

The hierarchy gives you options — here's a quick decision guide:

```java
// Need ordered, indexed access?           -> ArrayList
// Need fast insert/delete at both ends?   -> ArrayDeque
// Need unique elements?                   -> HashSet
// Need sorted unique elements?            -> TreeSet
// Need key-value lookup?                  -> HashMap
// Need sorted key-value pairs?            -> TreeMap
// Need insertion-order key-value?         -> LinkedHashMap
// Need thread-safe map?                   -> ConcurrentHashMap
// Need FIFO queue?                        -> ArrayDeque or LinkedList
// Need priority-based processing?         -> PriorityQueue
```

> **🎯 Interviewer Insight:** Draw the hierarchy diagram on a whiteboard — interviewers love visual explanations. Emphasize that Map is NOT part of the Collection hierarchy (this is a common confusion point). If the role targets modern Java, mention Java 21 Sequenced Collections — it shows you're staying current.

> **⚠️ Common Mistake:** Confusing Map with Collection. Map does NOT extend Collection — they are separate hierarchies. Map stores key-value pairs, while Collection stores single elements. This matters because you can't pass a Map to a method expecting a Collection.

> **🔧 Performance Tip:** Always "program to interface, not implementation." Declare variables as `List`, `Set`, or `Map` rather than `ArrayList`, `HashSet`, or `HashMap`. This makes it easy to swap implementations later without changing client code.

> **⚡ Follow-up:** Why doesn't Map extend Collection, and what are the implications of this design decision?

> **⚡ Follow-up:** How does the Sequenced Collection API in Java 21 simplify operations that previously required different methods across collection types?

> **⚡ Follow-up:** When would you choose a specific implementation (like LinkedList) over the interface type in your method signatures?

> **📚 Related:** [ArrayList Internals](./02-arraylist.md), [HashMap Internals](./07-hashmap-internals.md), [Collection Performance](./18-collection-performance.md)

---

[← Back to Section Index](./README.md) · [Next: ArrayList Internals →](./02-arraylist.md)

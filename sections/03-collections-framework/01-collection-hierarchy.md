# Q1: Explain the Java Collection Hierarchy

[Back to Collections Framework](./README.md) | **Difficulty**: 🟢 Easy

---

## Question
> Explain the Java Collection hierarchy. What are the key interfaces and their contracts?

---

## Answer

### The Complete Hierarchy

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

### Core Interfaces and Their Contracts

#### 1. Iterable\<T\> (java.lang)
The root interface that enables for-each loop support.

```java
public interface Iterable<T> {
    Iterator<T> iterator();

    // Java 8+
    default void forEach(Consumer<? super T> action) { ... }
    default Spliterator<T> spliterator() { ... }
}
```

#### 2. Collection\<T\>
The foundation interface for all collections (except Map).

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

#### 3. List\<T\> - Ordered, allows duplicates

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

#### 4. Set\<T\> - No duplicates

```java
public interface Set<T> extends Collection<T> {
    // Inherits Collection methods
    // Contract: no duplicate elements (based on equals())
    // At most one null element (implementation-dependent)

    // Java 9+
    static <E> Set<E> of(E... elements) { ... }
}
```

#### 5. Queue\<T\> - FIFO ordering (typically)

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

#### 6. Deque\<T\> - Double-ended queue

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

#### 7. Map\<K,V\> - Key-value pairs (NOT a Collection)

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

Java 21 introduced the **Sequenced Collections** API to unify encounter-order access.

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

### Anti-Patterns

```java
// ❌ Using raw types
List names = new ArrayList();       // No type safety

// ❌ Programming to implementation
ArrayList<String> list = new ArrayList<>();  // Coupled to ArrayList

// ❌ Confusing Map with Collection
// Map does NOT extend Collection - they are separate hierarchies
```

### Best Practices

```java
// ✅ Always use generics
List<String> names = new ArrayList<>();

// ✅ Program to interface
List<String> list = new ArrayList<>();
Set<Integer> set = new HashSet<>();
Map<String, Object> map = new HashMap<>();

// ✅ Use diamond operator (Java 7+)
Map<String, List<Integer>> map = new HashMap<>();

// ✅ Use factory methods (Java 9+)
List<String> immutable = List.of("A", "B", "C");
Set<Integer> immutableSet = Set.of(1, 2, 3);
Map<String, Integer> immutableMap = Map.of("key", 1);
```

---

## Interview Tips
- Draw the hierarchy diagram on a whiteboard - interviewers love visual explanations
- Emphasize that Map is NOT part of the Collection hierarchy
- Mention Java 21 Sequenced Collections if the role targets modern Java
- Always highlight "program to interface, not implementation"

---

[Next: Q2 - How does ArrayList work internally?](./q02-arraylist.md)

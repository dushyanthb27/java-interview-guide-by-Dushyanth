# Q20: How to Implement a Custom Collection?

[Back to Collections Framework](./README.md) | **Difficulty**: 🔴 Expert

---

## Question
> How do you implement a custom collection in Java? What abstract classes and interfaces should you use?

---

## Answer

### Abstract Skeleton Classes

Java provides **abstract skeleton implementations** that do most of the heavy lifting. You only need to implement a few methods.

```java
// Abstract class hierarchy:
// AbstractCollection<E> implements Collection<E>
//   └── AbstractList<E> implements List<E>
//       └── AbstractSequentialList<E> (for sequential access lists)
//   └── AbstractSet<E> implements Set<E>
//   └── AbstractQueue<E> implements Queue<E>
//
// AbstractMap<K,V> implements Map<K,V>
//
// To create a custom collection, extend the appropriate abstract class
// and implement the required abstract methods.
```

### Custom Immutable List (Extending AbstractList)

```java
// AbstractList requires you to implement: get(index) and size()
// All other List methods are derived from these two!

public class ImmutableArrayList<E> extends AbstractList<E> {
    private final Object[] elements;

    @SafeVarargs
    public ImmutableArrayList(E... elements) {
        this.elements = elements.clone();  // Defensive copy
    }

    @Override
    @SuppressWarnings("unchecked")
    public E get(int index) {
        if (index < 0 || index >= elements.length) {
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + elements.length);
        }
        return (E) elements[index];
    }

    @Override
    public int size() {
        return elements.length;
    }

    // Note: add(), set(), remove() will throw UnsupportedOperationException
    // because we didn't override them (AbstractList's default behavior)
}

// Usage:
ImmutableArrayList<String> list = new ImmutableArrayList<>("A", "B", "C");
list.get(1);          // "B"
list.size();           // 3
list.contains("A");   // true (inherited, uses get() and size())
list.indexOf("B");    // 1 (inherited)
list.iterator();      // Works! (inherited)
// list.add("D");     // ❌ UnsupportedOperationException
```

### Custom Mutable List (Extending AbstractList)

```java
// For a mutable list, additionally override: set(), add(), remove()

public class SimpleArrayList<E> extends AbstractList<E> {
    private Object[] data;
    private int size;

    public SimpleArrayList() {
        this.data = new Object[10];
        this.size = 0;
    }

    @Override
    @SuppressWarnings("unchecked")
    public E get(int index) {
        rangeCheck(index);
        return (E) data[index];
    }

    @Override
    public int size() {
        return size;
    }

    @Override
    public E set(int index, E element) {
        rangeCheck(index);
        @SuppressWarnings("unchecked")
        E oldValue = (E) data[index];
        data[index] = element;
        return oldValue;
    }

    @Override
    public void add(int index, E element) {
        rangeCheckForAdd(index);
        ensureCapacity(size + 1);
        System.arraycopy(data, index, data, index + 1, size - index);
        data[index] = element;
        size++;
        modCount++;  // Important for fail-fast iterators!
    }

    @Override
    public E remove(int index) {
        rangeCheck(index);
        @SuppressWarnings("unchecked")
        E oldValue = (E) data[index];
        int numMoved = size - index - 1;
        if (numMoved > 0) {
            System.arraycopy(data, index + 1, data, index, numMoved);
        }
        data[--size] = null;  // Help GC
        modCount++;  // Important for fail-fast iterators!
        return oldValue;
    }

    private void ensureCapacity(int minCapacity) {
        if (minCapacity > data.length) {
            int newCapacity = data.length + (data.length >> 1);  // 1.5x
            if (newCapacity < minCapacity) newCapacity = minCapacity;
            data = Arrays.copyOf(data, newCapacity);
        }
    }

    private void rangeCheck(int index) {
        if (index < 0 || index >= size)
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
    }

    private void rangeCheckForAdd(int index) {
        if (index < 0 || index > size)
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
    }
}

// Usage:
SimpleArrayList<String> list = new SimpleArrayList<>();
list.add("A");            // Uses add(size(), element) -> our add(int, E)
list.add("B");
list.add(1, "X");         // [A, X, B]
list.set(0, "Z");         // [Z, X, B]
list.remove(1);           // [Z, B]
list.contains("Z");       // true (inherited from AbstractCollection)
list.stream().count();    // 2 (inherited Stream support!)
```

### Custom HashMap Implementation

```java
public class SimpleHashMap<K, V> extends AbstractMap<K, V> {
    private static final int DEFAULT_CAPACITY = 16;
    private static final float LOAD_FACTOR = 0.75f;

    private Entry<K, V>[] buckets;
    private int size;

    static class Entry<K, V> implements Map.Entry<K, V> {
        final int hash;
        final K key;
        V value;
        Entry<K, V> next;

        Entry(int hash, K key, V value, Entry<K, V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        @Override public K getKey() { return key; }
        @Override public V getValue() { return value; }
        @Override
        public V setValue(V value) {
            V old = this.value;
            this.value = value;
            return old;
        }

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (!(o instanceof Map.Entry)) return false;
            Map.Entry<?, ?> e = (Map.Entry<?, ?>) o;
            return Objects.equals(key, e.getKey()) &&
                   Objects.equals(value, e.getValue());
        }

        @Override
        public int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }
    }

    @SuppressWarnings("unchecked")
    public SimpleHashMap() {
        buckets = new Entry[DEFAULT_CAPACITY];
        size = 0;
    }

    private int hash(Object key) {
        if (key == null) return 0;
        int h = key.hashCode();
        return h ^ (h >>> 16);  // Spread bits
    }

    private int bucketIndex(int hash) {
        return hash & (buckets.length - 1);
    }

    @Override
    public V put(K key, V value) {
        int hash = hash(key);
        int index = bucketIndex(hash);

        // Check if key already exists
        for (Entry<K, V> e = buckets[index]; e != null; e = e.next) {
            if (e.hash == hash && Objects.equals(e.key, key)) {
                V oldValue = e.value;
                e.value = value;
                return oldValue;
            }
        }

        // Add new entry at head of chain
        buckets[index] = new Entry<>(hash, key, value, buckets[index]);
        size++;

        // Resize if needed
        if (size > buckets.length * LOAD_FACTOR) {
            resize();
        }
        return null;
    }

    @Override
    public V get(Object key) {
        int hash = hash(key);
        int index = bucketIndex(hash);

        for (Entry<K, V> e = buckets[index]; e != null; e = e.next) {
            if (e.hash == hash && Objects.equals(e.key, key)) {
                return e.value;
            }
        }
        return null;
    }

    @Override
    public V remove(Object key) {
        int hash = hash(key);
        int index = bucketIndex(hash);

        Entry<K, V> prev = null;
        for (Entry<K, V> e = buckets[index]; e != null; prev = e, e = e.next) {
            if (e.hash == hash && Objects.equals(e.key, key)) {
                if (prev == null) {
                    buckets[index] = e.next;
                } else {
                    prev.next = e.next;
                }
                size--;
                return e.value;
            }
        }
        return null;
    }

    @Override
    public int size() {
        return size;
    }

    // AbstractMap requires entrySet() to be implemented
    @Override
    public Set<Map.Entry<K, V>> entrySet() {
        Set<Map.Entry<K, V>> entries = new HashSet<>();
        for (Entry<K, V> bucket : buckets) {
            for (Entry<K, V> e = bucket; e != null; e = e.next) {
                entries.add(e);
            }
        }
        return entries;
    }

    @SuppressWarnings("unchecked")
    private void resize() {
        Entry<K, V>[] oldBuckets = buckets;
        buckets = new Entry[oldBuckets.length * 2];
        size = 0;

        for (Entry<K, V> bucket : oldBuckets) {
            for (Entry<K, V> e = bucket; e != null; e = e.next) {
                put(e.key, e.value);
            }
        }
    }
}
```

### Implementing Iterable and Iterator

```java
// Making any class work with for-each loops
public class NumberRange implements Iterable<Integer> {
    private final int start;
    private final int end;

    public NumberRange(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    public Iterator<Integer> iterator() {
        return new NumberIterator();
    }

    private class NumberIterator implements Iterator<Integer> {
        private int current = start;

        @Override
        public boolean hasNext() {
            return current < end;
        }

        @Override
        public Integer next() {
            if (!hasNext()) {
                throw new NoSuchElementException();
            }
            return current++;
        }

        // Optional: support remove()
        @Override
        public void remove() {
            throw new UnsupportedOperationException("Range is immutable");
        }
    }

    // Also implement Spliterator for parallel stream support
    @Override
    public Spliterator<Integer> spliterator() {
        return Spliterators.spliterator(
            iterator(),
            end - start,
            Spliterator.ORDERED | Spliterator.SIZED | Spliterator.IMMUTABLE
        );
    }
}

// Usage:
NumberRange range = new NumberRange(1, 6);
for (int n : range) {
    System.out.println(n);  // 1, 2, 3, 4, 5
}

range.forEach(System.out::println);

// Stream support
long count = StreamSupport.stream(range.spliterator(), false)
    .filter(n -> n % 2 == 0)
    .count();  // 2 (2 and 4)
```

### Custom Immutable Set

```java
public class ImmutableHashSet<E> extends AbstractSet<E> {
    private final Object[] elements;
    private final int size;

    @SafeVarargs
    public ImmutableHashSet(E... elements) {
        // Remove duplicates while preserving order
        LinkedHashSet<E> set = new LinkedHashSet<>(Arrays.asList(elements));
        this.elements = set.toArray();
        this.size = set.size();
    }

    // AbstractSet requires: iterator() and size()
    @Override
    public Iterator<E> iterator() {
        return new Iterator<>() {
            private int index = 0;

            @Override
            public boolean hasNext() {
                return index < size;
            }

            @Override
            @SuppressWarnings("unchecked")
            public E next() {
                if (!hasNext()) throw new NoSuchElementException();
                return (E) elements[index++];
            }
        };
    }

    @Override
    public int size() {
        return size;
    }

    // Override for O(1) performance instead of inherited O(n)
    @Override
    public boolean contains(Object o) {
        for (Object element : elements) {
            if (Objects.equals(o, element)) return true;
        }
        return false;
    }

    // add(), remove() will throw UnsupportedOperationException (AbstractSet default)
}
```

### Design Considerations

```java
// 1. Choose the right abstract class:
//    - AbstractList: random access list (implement get + size)
//    - AbstractSequentialList: sequential access (implement listIterator + size)
//    - AbstractSet: set (implement iterator + size)
//    - AbstractMap: map (implement entrySet)
//    - AbstractQueue: queue (implement offer, poll, peek, size, iterator)

// 2. Immutable vs Mutable:
//    - Immutable: implement only get/iterator + size
//    - Mutable: also override add, set, remove

// 3. Performance optimizations:
//    - Override contains() for O(1) lookup in hash-based collections
//    - Override isEmpty() to avoid computing size()
//    - Override addAll() for bulk operations

// 4. Thread safety:
//    - Use synchronized blocks or concurrent data structures
//    - Update modCount for fail-fast iterator support

// 5. Serialization:
//    - Implement Serializable if needed
//    - Consider custom writeObject/readObject for efficiency

// 6. equals() and hashCode():
//    - AbstractList and AbstractSet provide correct implementations
//    - AbstractMap provides correct implementation based on entrySet
```

### Custom Bounded Collection

```java
// Example: A fixed-size circular buffer
public class CircularBuffer<E> extends AbstractCollection<E> implements Queue<E> {
    private final Object[] buffer;
    private int head = 0;
    private int tail = 0;
    private int count = 0;

    public CircularBuffer(int capacity) {
        this.buffer = new Object[capacity];
    }

    @Override
    public boolean offer(E e) {
        if (count == buffer.length) return false;  // Full
        buffer[tail] = e;
        tail = (tail + 1) % buffer.length;
        count++;
        return true;
    }

    @Override
    @SuppressWarnings("unchecked")
    public E poll() {
        if (count == 0) return null;  // Empty
        E element = (E) buffer[head];
        buffer[head] = null;  // Help GC
        head = (head + 1) % buffer.length;
        count--;
        return element;
    }

    @Override
    @SuppressWarnings("unchecked")
    public E peek() {
        return count == 0 ? null : (E) buffer[head];
    }

    @Override
    public int size() {
        return count;
    }

    @Override
    public Iterator<E> iterator() {
        return new Iterator<>() {
            private int index = 0;

            @Override
            public boolean hasNext() {
                return index < count;
            }

            @Override
            @SuppressWarnings("unchecked")
            public E next() {
                if (!hasNext()) throw new NoSuchElementException();
                E element = (E) buffer[(head + index) % buffer.length];
                index++;
                return element;
            }
        };
    }

    // Queue methods that throw exceptions (delegates to offer/poll/peek)
    @Override
    public boolean add(E e) {
        if (!offer(e)) throw new IllegalStateException("Buffer full");
        return true;
    }

    @Override
    public E remove() {
        E e = poll();
        if (e == null) throw new NoSuchElementException("Buffer empty");
        return e;
    }

    @Override
    public E element() {
        E e = peek();
        if (e == null) throw new NoSuchElementException("Buffer empty");
        return e;
    }
}

// Usage:
CircularBuffer<String> buffer = new CircularBuffer<>(3);
buffer.offer("A");  // [A, _, _]
buffer.offer("B");  // [A, B, _]
buffer.offer("C");  // [A, B, C]
buffer.offer("D");  // false - buffer full!
buffer.poll();       // "A" -> [_, B, C]
buffer.offer("D");  // [D, B, C] (wraps around)
```

### Common Anti-Patterns

```java
// ❌ Implementing Collection from scratch without using abstract classes
public class MyList<E> implements List<E> {
    // You'd need to implement 25+ methods!
    // AbstractList gives you most of them for free
}

// ❌ Forgetting to update modCount for structural modifications
public void add(int index, E element) {
    // ... add element ...
    // modCount++;  // ❌ Forgot this! Fail-fast iterators won't work properly
}

// ❌ Not implementing equals/hashCode for custom Map.Entry
// AbstractMap.equals() depends on entrySet() entries having correct equals/hashCode

// ❌ Returning a mutable entrySet() from an immutable map
// The entrySet() view should also be unmodifiable
```

### Best Practices

```java
// ✅ Extend the appropriate abstract class (don't implement interfaces directly)
// ✅ Implement the minimum required methods first, add optimizations later
// ✅ Update modCount on structural modifications for fail-fast support
// ✅ Override contains/isEmpty for performance when possible
// ✅ Throw UnsupportedOperationException for unsupported operations
// ✅ Implement Serializable if the collection needs serialization
// ✅ Use @SafeVarargs for varargs constructors with generic types
// ✅ Add null checks and bounds checks with clear error messages
// ✅ Provide custom Spliterator for efficient parallel stream support
// ✅ Document the contract (ordered? allows nulls? allows duplicates? thread-safe?)
```

---

## Summary: Abstract Class Method Requirements

| Abstract Class | Required Methods | Free Methods |
|---------------|-----------------|--------------|
| `AbstractList` | `get(int)`, `size()` | contains, indexOf, iterator, listIterator, subList, equals, hashCode, toString |
| `AbstractList` (mutable) | + `set(int,E)`, `add(int,E)`, `remove(int)` | All of the above + add, remove, clear, addAll |
| `AbstractSet` | `iterator()`, `size()` | contains, containsAll, equals, hashCode, removeAll, toString |
| `AbstractMap` | `entrySet()` | get, containsKey, containsValue, keySet, values, equals, hashCode, toString |
| `AbstractMap` (mutable) | + `put(K,V)` | All of the above + putAll, remove, clear |
| `AbstractQueue` | `offer(E)`, `poll()`, `peek()`, `size()`, `iterator()` | add, remove, element, clear, addAll |

---

## Interview Tips
- Know which abstract class to extend for each collection type
- AbstractList needs only get() + size() for a read-only list (2 methods give you 25+!)
- Always update modCount for fail-fast iterator support
- Custom HashMap implementation is a classic coding interview question
- Understand the trade-off between implementing minimal methods and overriding for performance
- Know how to implement Iterable to enable for-each support

---

[Prev: Q19 - Thread-Safe Collections](./q19-thread-safe-collections.md) | [Back to Collections Framework](./README.md)

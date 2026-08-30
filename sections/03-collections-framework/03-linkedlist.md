# Q3: How Does LinkedList Work and When to Use It?

[Back to Collections Framework](./README.md) | **Difficulty**: 🟡 Medium

---

## Question
> How does LinkedList work internally? When should you use it over ArrayList?

---

## Answer

### Internal Structure - Doubly-Linked List

LinkedList is a **doubly-linked list** where each element is wrapped in a `Node` object with pointers to both the previous and next nodes.

```java
// Simplified internal structure from OpenJDK
public class LinkedList<E> extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, Serializable {

    transient int size = 0;
    transient Node<E> first;  // pointer to head
    transient Node<E> last;   // pointer to tail

    private static class Node<E> {
        E item;         // the actual data
        Node<E> next;   // pointer to next node
        Node<E> prev;   // pointer to previous node

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
}
```

### Memory Layout Visualization

```
ArrayList (contiguous memory):
  [ref0 | ref1 | ref2 | ref3 | ref4 | null | null]
   ↓      ↓      ↓      ↓      ↓
  obj0   obj1   obj2   obj3   obj4

LinkedList (scattered memory):
  null ← [prev|"A"|next] ⇄ [prev|"B"|next] ⇄ [prev|"C"|next] → null
          ↑ first                                  ↑ last
```

### Memory Overhead Per Element

```java
// Each LinkedList Node costs:
//   - 16 bytes: object header (on 64-bit JVM with compressed oops)
//   -  8 bytes: item reference
//   -  8 bytes: next reference
//   -  8 bytes: prev reference
//   = 40 bytes per Node (minimum)
//   + padding to 8-byte boundary
//
// Compare ArrayList: ~4-8 bytes per element (just the reference in array)
//
// LinkedList uses 5-8x MORE memory per element than ArrayList!
```

### Key Operations

```java
LinkedList<String> list = new LinkedList<>();

// O(1) - Add at head or tail (just update pointers)
list.addFirst("A");     // Deque method
list.addLast("C");      // Deque method
list.add("D");          // Same as addLast

// O(1) - Remove from head or tail
String head = list.removeFirst();
String tail = list.removeLast();

// O(1) - Peek at head or tail
String first = list.peekFirst();
String last  = list.peekLast();

// O(n) - Random access by index (must traverse from nearest end)
String item = list.get(5);  // traverses from head or tail, whichever is closer

// O(n) - Search
boolean exists = list.contains("B");
int index = list.indexOf("B");
```

### How get(index) Works Internally

```java
// LinkedList optimizes by traversing from the CLOSER end
Node<E> node(int index) {
    if (index < (size >> 1)) {
        // Start from head, go forward
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        // Start from tail, go backward
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

### LinkedList as a Deque (Double-Ended Queue)

```java
LinkedList<String> deque = new LinkedList<>();

// Queue operations (FIFO)
deque.offer("A");       // add to tail
deque.offer("B");
String head = deque.poll();   // remove from head -> "A"

// Stack operations (LIFO)
deque.push("X");        // add to head (addFirst)
deque.push("Y");
String top = deque.pop();    // remove from head -> "Y"

// Deque operations
deque.offerFirst("start");
deque.offerLast("end");
deque.peekFirst();
deque.peekLast();
deque.pollFirst();
deque.pollLast();
```

### Cache Locality Problem

```java
// ArrayList: elements stored contiguously in memory
// CPU cache lines load adjacent array elements together
// Sequential iteration is FAST (cache-friendly)

// LinkedList: nodes scattered across heap
// Each node.next may point to a completely different memory region
// CPU cache misses on nearly every access (cache-unfriendly)

// Benchmark result (typical):
// ArrayList sequential iteration: ~1x (baseline)
// LinkedList sequential iteration: ~5-10x slower (cache misses)
```

### When LinkedList Beats ArrayList (Rare Cases)

```java
// Case 1: Frequent insertions/removals at BOTH ends (use as Deque)
// But ArrayDeque is usually better even here!
Deque<String> deque = new LinkedList<>();  // Works but ArrayDeque is faster
Deque<String> better = new ArrayDeque<>(); // Preferred!

// Case 2: Frequent insertions/removals at KNOWN positions during iteration
// When you already have an iterator positioned at the insertion point
ListIterator<String> it = list.listIterator();
while (it.hasNext()) {
    String val = it.next();
    if (someCondition(val)) {
        it.add("newElement");  // O(1) with LinkedList iterator
        it.remove();           // O(1) with LinkedList iterator
    }
}

// Case 3: Very large elements where you never need random access
// and constantly add/remove from the ends
// (even here, ArrayDeque is typically preferred)
```

### LinkedList vs ArrayList - Detailed Comparison

```java
// INSERTION at beginning:
// ArrayList:   O(n) - must shift ALL elements right
// LinkedList:  O(1) - just update head pointers
list.add(0, element);

// INSERTION at end:
// ArrayList:   O(1) amortized
// LinkedList:  O(1)
list.add(element);

// INSERTION at middle (by index):
// ArrayList:   O(n) - find position O(1) + shift O(n)
// LinkedList:  O(n) - find position O(n) + insert O(1)
// ArrayList still wins due to System.arraycopy and cache locality!

// RANDOM ACCESS:
// ArrayList:   O(1) - direct array index
// LinkedList:  O(n) - must traverse nodes
list.get(index);

// MEMORY:
// ArrayList:   compact (array of references + unused slots)
// LinkedList:  heavy (Node object per element with 3 references)

// ITERATION:
// ArrayList:   fast (sequential memory access, CPU cache friendly)
// LinkedList:  slow (pointer chasing, CPU cache unfriendly)
```

### Common Anti-Patterns

```java
// ❌ Using LinkedList for random access
LinkedList<String> list = new LinkedList<>();
// ... populate ...
for (int i = 0; i < list.size(); i++) {
    list.get(i);  // O(n) per call = O(n^2) total!
}

// ❌ Using LinkedList when ArrayDeque would suffice
LinkedList<String> queue = new LinkedList<>();  // More memory overhead

// ❌ Choosing LinkedList "because insertions are O(1)"
// Insertions are only O(1) when you ALREADY HAVE the node reference
// Finding the position is still O(n)!
list.add(500, "element");  // O(n) for LinkedList too!

// ❌ Using LinkedList for sorted insertion
// TreeSet/TreeMap is better for maintaining sorted order
```

### Best Practices

```java
// ✅ Default to ArrayList for most use cases
List<String> list = new ArrayList<>();

// ✅ Use ArrayDeque instead of LinkedList for queue/stack
Deque<String> stack = new ArrayDeque<>();
Queue<String> queue = new ArrayDeque<>();

// ✅ If you must use LinkedList, iterate with iterator (not index)
for (String item : linkedList) {  // Uses iterator internally - O(n) total
    process(item);
}

// ✅ Use LinkedList only when you need List + Deque interface together
// and frequently operate on both ends
LinkedList<Task> taskList = new LinkedList<>();
taskList.addFirst(urgentTask);   // Push to front
taskList.addLast(normalTask);    // Add to back
Task next = taskList.pollFirst(); // Process from front

// ✅ Consider LinkedList for ListIterator-heavy operations
// where you insert/remove during traversal at many positions
ListIterator<String> it = linkedList.listIterator();
while (it.hasNext()) {
    if (shouldInsertBefore(it.next())) {
        it.previous();
        it.add("new");
        it.next(); // skip past inserted element
    }
}
```

### Null Handling

```java
LinkedList<String> list = new LinkedList<>();
list.add(null);        // ✅ Allowed - LinkedList permits null elements
list.add(null);        // ✅ Multiple nulls allowed
list.contains(null);   // ✅ Returns true
list.remove(null);     // ✅ Removes first null
```

---

## Performance Summary

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| `addFirst(e)` / `addLast(e)` | O(1) | Pointer updates only |
| `add(e)` (at end) | O(1) | Same as addLast |
| `add(index, e)` | O(n) | Traversal to find position |
| `get(index)` | O(n) | Traversal from nearest end |
| `removeFirst()` / `removeLast()` | O(1) | Pointer updates only |
| `remove(index)` | O(n) | Traversal to find position |
| `contains(e)` | O(n) | Linear search |
| `iterator.remove()` | O(1) | Already at position |
| Memory per element | ~40 bytes | vs ~4-8 bytes for ArrayList |

---

## Interview Tips
- LinkedList is almost NEVER the right choice in practice
- ArrayList beats LinkedList in most real-world scenarios due to cache locality
- ArrayDeque should be preferred over LinkedList for queue/stack operations
- The only strong case for LinkedList: ListIterator-based insertion/removal during traversal
- Know the Node internal class structure (prev, item, next)

---

[Prev: Q2 - ArrayList](./q02-arraylist.md) | [Next: Q4 - Vector and Stack](./q04-vector-stack.md)

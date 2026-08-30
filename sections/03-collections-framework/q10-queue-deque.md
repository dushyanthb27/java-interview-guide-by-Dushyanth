# Q10: Explain Queue and Deque Implementations

[Back to Collections Framework](./README.md) | **Difficulty**: 🟡 Medium

---

## Question
> Explain the Queue and Deque interfaces and their key implementations.

---

## Answer

### Queue Interface - Two Sets of Methods

Queue provides two methods for each operation: one **throws exception** on failure, the other **returns a special value**.

```java
public interface Queue<E> extends Collection<E> {
    //                  Throws Exception    Returns Special Value
    // Insert:          add(e)              offer(e) -> false
    // Remove:          remove()            poll()   -> null
    // Examine:         element()           peek()   -> null
}

// Example
Queue<String> queue = new LinkedList<>();

// ✅ Prefer special-value methods (safer)
queue.offer("A");           // returns true/false
String head = queue.poll();  // returns null if empty
String peek = queue.peek();  // returns null if empty

// ⚠️ Exception-throwing methods
queue.add("A");              // throws IllegalStateException if full
String head = queue.remove(); // throws NoSuchElementException if empty
String peek = queue.element();// throws NoSuchElementException if empty
```

### Deque Interface - Double-Ended Queue

```java
public interface Deque<E> extends Queue<E> {
    //                     First Element (Head)              Last Element (Tail)
    //                  Exception    Special Value       Exception    Special Value
    // Insert:          addFirst(e)  offerFirst(e)       addLast(e)   offerLast(e)
    // Remove:          removeFirst() pollFirst()        removeLast()  pollLast()
    // Examine:         getFirst()   peekFirst()         getLast()     peekLast()

    // Stack operations
    void push(E e);        // equivalent to addFirst(e)
    E pop();               // equivalent to removeFirst()

    // Iteration
    Iterator<E> descendingIterator();
}
```

### PriorityQueue - Heap-Based Priority Queue

PriorityQueue is backed by a **binary min-heap** (array-based). Elements are ordered by natural ordering or a Comparator.

```java
// Default: natural ordering (min-heap)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(30);
minHeap.offer(10);
minHeap.offer(20);

minHeap.peek();   // 10 (smallest - NOT insertion order!)
minHeap.poll();   // 10
minHeap.poll();   // 20
minHeap.poll();   // 30

// Max-heap with reverse comparator
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.offer(30);
maxHeap.offer(10);
maxHeap.offer(20);
maxHeap.poll();   // 30 (largest first)

// Custom priority
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority)
              .thenComparing(Task::getCreatedAt)
);

// Internal binary heap array structure:
//        [10]                Array: [10, 30, 20]
//       /    \               Index:  0   1   2
//     [30]  [20]             Parent of i: (i-1)/2
//                            Left child of i: 2*i + 1
//                            Right child of i: 2*i + 2
```

### PriorityQueue Key Characteristics

```java
PriorityQueue<String> pq = new PriorityQueue<>();

// ✅ O(log n) insert and remove
pq.offer("Banana");    // O(log n) - sift up
pq.offer("Apple");
pq.offer("Cherry");
pq.poll();              // O(log n) - sift down -> "Apple"

// ✅ O(1) peek
pq.peek();              // O(1) -> "Banana"

// ❌ NOT ordered during iteration!
for (String s : pq) {
    // Does NOT iterate in priority order!
    // Internal heap order ≠ sorted order
}

// ✅ To get sorted order, poll() repeatedly
while (!pq.isEmpty()) {
    System.out.println(pq.poll());  // Prints in sorted order
}

// ❌ Does NOT allow null elements
pq.offer(null);  // NullPointerException

// O(n) for contains() and remove(Object)
pq.contains("Banana");  // O(n) linear scan
pq.remove("Banana");    // O(n) find + O(log n) restructure
```

### ArrayDeque - Resizable Circular Array

ArrayDeque is the **preferred implementation** for both Stack and Queue. It uses a circular array internally.

```java
// Circular array structure:
// Array: [D, E, _, _, _, A, B, C]
//              ↑ tail    ↑ head
// Logical order: A, B, C, D, E
// head wraps around to the beginning when it reaches the end

Deque<String> deque = new ArrayDeque<>();

// Queue operations (FIFO)
deque.offer("A");       // Add to tail
deque.offer("B");
deque.offer("C");
deque.poll();           // Remove from head -> "A"
// Queue: [B, C]

// Stack operations (LIFO)
deque.push("X");        // addFirst -> Add to head
deque.push("Y");
deque.pop();            // removeFirst -> "Y"
// Stack: [X, B, C]

// Double-ended operations
deque.offerFirst("start");
deque.offerLast("end");
deque.peekFirst();      // "start"
deque.peekLast();       // "end"
```

### ArrayDeque Internals

```java
// ArrayDeque internal structure (simplified)
public class ArrayDeque<E> extends AbstractCollection<E>
        implements Deque<E>, Cloneable, Serializable {

    transient Object[] elements;  // circular array
    transient int head;           // index of first element
    transient int tail;           // index past last element

    // Default initial capacity: 16 (must be power of 2)
    // Doubles when full (2x growth)

    // Circular indexing:
    // Next index: (i + 1) & (elements.length - 1)
    // Prev index: (i - 1) & (elements.length - 1)
    // Works because capacity is always power of 2
}

// ❌ ArrayDeque does NOT allow null elements
ArrayDeque<String> deque = new ArrayDeque<>();
deque.offer(null);  // NullPointerException

// Why? null is used as a sentinel value internally
```

### BlockingQueue - Producer-Consumer Pattern

```java
// BlockingQueue adds blocking operations:
//                  Blocks        Times out
// Insert:          put(e)        offer(e, time, unit)
// Remove:          take()        poll(time, unit)

// ✅ LinkedBlockingQueue (optionally bounded, linked nodes)
BlockingQueue<String> lbq = new LinkedBlockingQueue<>(100);  // capacity 100

// ✅ ArrayBlockingQueue (bounded, array-based)
BlockingQueue<String> abq = new ArrayBlockingQueue<>(100);

// ✅ PriorityBlockingQueue (unbounded, priority-ordered)
BlockingQueue<Task> pbq = new PriorityBlockingQueue<>();

// ✅ SynchronousQueue (zero capacity - direct handoff)
BlockingQueue<String> sq = new SynchronousQueue<>();

// ✅ DelayQueue (elements available after a delay)
BlockingQueue<DelayedTask> dq = new DelayQueue<>();
```

### Producer-Consumer Example

```java
// Thread-safe producer-consumer with BlockingQueue
BlockingQueue<String> queue = new LinkedBlockingQueue<>(10);

// Producer thread
Runnable producer = () -> {
    try {
        for (int i = 0; i < 100; i++) {
            String item = "Item-" + i;
            queue.put(item);  // Blocks if queue is full
            System.out.println("Produced: " + item);
        }
        queue.put("DONE");  // Poison pill
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
};

// Consumer thread
Runnable consumer = () -> {
    try {
        while (true) {
            String item = queue.take();  // Blocks if queue is empty
            if ("DONE".equals(item)) break;
            System.out.println("Consumed: " + item);
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
};

new Thread(producer).start();
new Thread(consumer).start();
```

### Choosing the Right Queue Implementation

```java
// Need FIFO queue (single-threaded)?       -> ArrayDeque
Queue<String> queue = new ArrayDeque<>();

// Need LIFO stack?                          -> ArrayDeque
Deque<String> stack = new ArrayDeque<>();

// Need priority ordering?                   -> PriorityQueue
Queue<Task> pq = new PriorityQueue<>(Comparator.comparingInt(Task::getPriority));

// Need blocking producer-consumer?          -> LinkedBlockingQueue/ArrayBlockingQueue
BlockingQueue<String> bq = new LinkedBlockingQueue<>(100);

// Need concurrent non-blocking queue?       -> ConcurrentLinkedQueue
Queue<String> clq = new ConcurrentLinkedQueue<>();

// Need concurrent non-blocking deque?       -> ConcurrentLinkedDeque
Deque<String> cld = new ConcurrentLinkedDeque<>();

// Need direct handoff between threads?      -> SynchronousQueue
BlockingQueue<String> sq = new SynchronousQueue<>();

// Need delayed processing?                  -> DelayQueue
BlockingQueue<Delayed> dq = new DelayQueue<>();
```

### Common Anti-Patterns

```java
// ❌ Using LinkedList as a Queue (use ArrayDeque instead)
Queue<String> queue = new LinkedList<>();  // Higher memory, cache-unfriendly

// ❌ Using Stack class (use ArrayDeque instead)
Stack<String> stack = new Stack<>();  // Legacy, synchronized

// ❌ Iterating PriorityQueue expecting sorted order
PriorityQueue<Integer> pq = new PriorityQueue<>(List.of(3, 1, 2));
for (int n : pq) {
    // NOT guaranteed sorted! Internal heap order ≠ sorted order
}

// ❌ Using offer/poll on ArrayDeque with null check confusion
ArrayDeque<String> deque = new ArrayDeque<>();
String item = deque.poll();  // Returns null if empty
// But ArrayDeque doesn't allow null elements, so null always means empty
// This is different from LinkedList where null could be an actual element

// ❌ Busy-waiting instead of using BlockingQueue
while (queue.isEmpty()) {
    Thread.sleep(10);  // Wasteful! Use BlockingQueue.take() instead
}
```

### Best Practices

```java
// ✅ Use ArrayDeque as default Queue and Stack implementation
Deque<String> stack = new ArrayDeque<>();
Queue<String> queue = new ArrayDeque<>();

// ✅ Use PriorityQueue for priority-based processing
PriorityQueue<Task> tasks = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority)
);

// ✅ Use BlockingQueue for producer-consumer patterns
BlockingQueue<Job> jobQueue = new LinkedBlockingQueue<>(1000);

// ✅ Use ConcurrentLinkedQueue for non-blocking concurrent access
Queue<Event> eventQueue = new ConcurrentLinkedQueue<>();

// ✅ Pre-size ArrayDeque when count is known
Deque<String> deque = new ArrayDeque<>(expectedSize);

// ✅ Use offer/poll/peek instead of add/remove/element
// (they return special values instead of throwing exceptions)
```

---

## Performance Summary

| Implementation | offer/add | poll/remove | peek | contains | Thread-Safe |
|---------------|-----------|-------------|------|----------|-------------|
| ArrayDeque | O(1)* | O(1) | O(1) | O(n) | No |
| LinkedList | O(1) | O(1) | O(1) | O(n) | No |
| PriorityQueue | O(log n) | O(log n) | O(1) | O(n) | No |
| ArrayBlockingQueue | O(1) | O(1) | O(1) | O(n) | Yes |
| LinkedBlockingQueue | O(1) | O(1) | O(1) | O(n) | Yes |
| ConcurrentLinkedQueue | O(1) | O(1) | O(1) | O(n) | Yes |

\* Amortized O(1) - occasional resize

---

## Interview Tips
- ArrayDeque is the go-to for both Queue and Stack (not LinkedList, not Stack)
- PriorityQueue iteration does NOT give sorted order - only poll() does
- Know the BlockingQueue methods: put/take (blocking), offer/poll (with timeout)
- Producer-consumer with BlockingQueue is a very common interview question
- ConcurrentLinkedQueue is lock-free (uses CAS), BlockingQueue uses locks
- ArrayDeque doesn't allow null; LinkedList does

---

[Prev: Q9 - ConcurrentHashMap](./q09-concurrenthashmap.md) | [Next: Q11 - Iterator vs ListIterator](./q11-iterator-listiterator.md)

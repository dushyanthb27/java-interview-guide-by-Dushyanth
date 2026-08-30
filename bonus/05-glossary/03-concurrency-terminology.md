# Concurrency Terminology

Comprehensive reference for threading, synchronization, and concurrent programming concepts. This glossary covers 100+ essential concurrency terms organized alphabetically.

---

## A

### Atomic Operation
**Category**: Concurrency

**Definition**: An operation that completes entirely without interruption, appearing to execute instantaneously from the perspective of other threads. Atomic operations are indivisible and cannot be partially observed by other threads.

**Practical Example**:
```java
// Non-atomic operation
int count = 0;
count++;  // Not atomic: read, increment, write

// Atomic operation using AtomicInteger
AtomicInteger atomicCount = new AtomicInteger(0);
atomicCount.incrementAndGet();  // Atomic operation

// Atomic operations
atomicCount.getAndSet(5);
atomicCount.compareAndSet(5, 10);
```

**Related Terms**: [Atomic Variable](#atomic-variable), [Race Condition](#race-condition), [Synchronization](#synchronization), [Volatile](#volatile)

**Interview Tip**: Understand why simple operations like `count++` are not atomic. Know about AtomicInteger, AtomicLong, and AtomicReference.

---

### Atomic Variable
**Category**: Concurrency

**Definition**: A variable that can be updated atomically without explicit synchronization. Java provides atomic variables in the `java.util.concurrent.atomic` package for lock-free thread-safe programming.

**Practical Example**:
```java
// Atomic variables
AtomicInteger counter = new AtomicInteger(0);
AtomicLong timestamp = new AtomicLong(System.currentTimeMillis());
AtomicReference<String> reference = new AtomicReference<>("initial");
AtomicBoolean flag = new AtomicBoolean(false);

// Atomic operations
counter.incrementAndGet();
counter.addAndGet(5);
counter.compareAndSet(5, 10);

// Lock-free counter
class LockFreeCounter {
    private AtomicInteger count = new AtomicInteger(0);
    
    public void increment() {
        count.incrementAndGet();
    }
    
    public int getCount() {
        return count.get();
    }
}
```

**Related Terms**: [Atomic Operation](#atomic-operation), [Lock-Free](#lock-free), [CAS](#cas), [Synchronization](#synchronization)

**Interview Tip**: Know when to use atomic variables instead of synchronized blocks. Understand the performance benefits of lock-free programming.

---

## B

### Barrier
**Category**: Concurrency Utilities

**Definition**: A synchronization mechanism that allows multiple threads to wait for each other to reach a common point before proceeding. Barriers are useful for coordinating parallel tasks.

**Practical Example**:
```java
// CyclicBarrier
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("All threads reached barrier");
});

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try {
            System.out.println(Thread.currentThread().getName() + " waiting");
            barrier.await();  // Wait for all threads
            System.out.println(Thread.currentThread().getName() + " proceeding");
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    }).start();
}

// CountDownLatch (one-time barrier)
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        System.out.println("Task started");
        latch.countDown();  // Signal completion
    }).start();
}
latch.await();  // Wait for all tasks
System.out.println("All tasks completed");
```

**Related Terms**: [CyclicBarrier](#cyclicbarrier), [CountDownLatch](#countdownlatch), [Synchronization](#synchronization), [Thread Coordination](#thread-coordination)

**Interview Tip**: Know the difference between CyclicBarrier and CountDownLatch. Understand use cases for each.

---

## C

### CAS (Compare-And-Swap)
**Category**: Concurrency

**Definition**: An atomic operation that compares the current value of a variable with an expected value and, if they match, swaps it with a new value. CAS is the foundation for lock-free programming and atomic variables.

**Practical Example**:
```java
// CAS operation conceptually
class CASExample {
    private int value = 0;
    
    // Simulated CAS
    public synchronized boolean compareAndSwap(int expected, int newValue) {
        if (value == expected) {
            value = newValue;
            return true;
        }
        return false;
    }
}

// Using AtomicInteger (uses CAS internally)
AtomicInteger atomic = new AtomicInteger(0);
boolean success = atomic.compareAndSet(0, 1);  // CAS operation
System.out.println(success);  // true if value was 0

// Retry loop with CAS
AtomicInteger counter = new AtomicInteger(0);
while (!counter.compareAndSet(counter.get(), counter.get() + 1)) {
    // Retry if CAS failed
}
```

**Related Terms**: [Atomic Operation](#atomic-operation), [Atomic Variable](#atomic-variable), [Lock-Free](#lock-free), [ABA Problem](#aba-problem)

**Interview Tip**: Understand how CAS enables lock-free programming. Know about the ABA problem and how to handle it.

---

### Concurrent Collection
**Category**: Collections

**Definition**: A thread-safe collection that allows multiple threads to access and modify it concurrently without explicit synchronization. Examples include ConcurrentHashMap, CopyOnWriteArrayList, and ConcurrentLinkedQueue.

**Practical Example**:
```java
// ConcurrentHashMap
Map<String, Integer> map = new ConcurrentHashMap<>();
map.put("key1", 1);
map.putIfAbsent("key2", 2);

// CopyOnWriteArrayList
List<String> list = new CopyOnWriteArrayList<>();
list.add("item1");
list.add("item2");

// ConcurrentLinkedQueue
Queue<Integer> queue = new ConcurrentLinkedQueue<>();
queue.offer(1);
queue.offer(2);
Integer value = queue.poll();

// Thread-safe iteration
for (String item : list) {
    System.out.println(item);
}

// Atomic operations
map.compute("key1", (k, v) -> v == null ? 1 : v + 1);
```

**Related Terms**: [Thread Safety](#thread-safety), [Synchronization](#synchronization), [ConcurrentHashMap](#concurrenthashmap), [CopyOnWriteArrayList](#copyonwritearraylist)

**Interview Tip**: Know the differences between synchronized collections and concurrent collections. Understand when to use each.

---

### ConcurrentHashMap
**Category**: Collections

**Definition**: A thread-safe hash map that uses segment-level locking (or bucket-level locking in Java 8+) instead of table-level locking, allowing multiple threads to read and write concurrently.

**Practical Example**:
```java
// ConcurrentHashMap
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Put operations
map.put("key1", 1);
map.putIfAbsent("key2", 2);

// Atomic operations
map.compute("key1", (k, v) -> v == null ? 1 : v + 1);
map.merge("key1", 1, Integer::sum);

// Concurrent iteration (safe)
map.forEach((k, v) -> System.out.println(k + ": " + v));

// Segment-based locking (Java 7)
// Bucket-based locking (Java 8+)

// Performance comparison
// ConcurrentHashMap: Better for high concurrency
// Hashtable: Synchronized, slower
// HashMap: Not thread-safe
```

**Related Terms**: [Concurrent Collection](#concurrent-collection), [HashMap](#hashmap), [Hashtable](#hashtable), [Thread Safety](#thread-safety)

**Interview Tip**: Understand the locking strategy of ConcurrentHashMap. Know why it's better than Hashtable for concurrent access.

---

### Condition Variable
**Category**: Synchronization

**Definition**: A mechanism that allows threads to wait for a specific condition to be true before proceeding. Condition variables are used with locks to coordinate thread execution.

**Practical Example**:
```java
// Using Condition with ReentrantLock
class ProducerConsumer {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Queue<Integer> queue = new LinkedList<>();
    
    public void produce(int value) throws InterruptedException {
        lock.lock();
        try {
            queue.offer(value);
            notEmpty.signalAll();  // Notify waiting threads
        } finally {
            lock.unlock();
        }
    }
    
    public int consume() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();  // Wait for condition
            }
            return queue.poll();
        } finally {
            lock.unlock();
        }
    }
}

// Using wait/notify
class ProducerConsumerSync {
    private Queue<Integer> queue = new LinkedList<>();
    
    public synchronized void produce(int value) {
        queue.offer(value);
        notifyAll();
    }
    
    public synchronized int consume() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();
        }
        return queue.poll();
    }
}
```

**Related Terms**: [Lock](#lock), [ReentrantLock](#reentrantlock), [Wait/Notify](#waitnotify), [Synchronization](#synchronization)

**Interview Tip**: Know the difference between wait/notify and condition variables. Understand spurious wakeups and the importance of while loops.

---

### CopyOnWriteArrayList
**Category**: Collections

**Definition**: A thread-safe list implementation that creates a copy of the underlying array whenever a modification occurs. It's optimized for scenarios with many reads and few writes.

**Practical Example**:
```java
// CopyOnWriteArrayList
List<String> list = new CopyOnWriteArrayList<>();
list.add("item1");
list.add("item2");
list.add("item3");

// Safe iteration (snapshot)
for (String item : list) {
    System.out.println(item);
}

// Modifications create a copy
list.remove("item1");  // Creates new array
list.set(0, "modified");  // Creates new array

// Performance characteristics
// Read: O(1) - fast
// Write: O(n) - slow (creates copy)
// Iteration: Safe, no ConcurrentModificationException

// Use case: Event listeners
class EventManager {
    private List<EventListener> listeners = new CopyOnWriteArrayList<>();
    
    public void addListener(EventListener listener) {
        listeners.add(listener);
    }
    
    public void fireEvent(Event event) {
        for (EventListener listener : listeners) {
            listener.onEvent(event);
        }
    }
}
```

**Related Terms**: [Concurrent Collection](#concurrent-collection), [Thread Safety](#thread-safety), [ArrayList](#arraylist), [Copy-On-Write](#copy-on-write)

**Interview Tip**: Know when to use CopyOnWriteArrayList. Understand the performance implications of copying on write.

---

### CyclicBarrier
**Category**: Concurrency Utilities

**Definition**: A synchronization barrier that allows a fixed number of threads to wait for each other to reach a common point. Unlike CountDownLatch, CyclicBarrier can be reused after all threads pass the barrier.

**Practical Example**:
```java
// CyclicBarrier
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("All threads reached barrier");
});

for (int i = 0; i < 3; i++) {
    final int threadId = i;
    new Thread(() -> {
        try {
            System.out.println("Thread " + threadId + " waiting");
            barrier.await();  // Wait for all threads
            System.out.println("Thread " + threadId + " proceeding");
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    }).start();
}

// Reusable barrier
for (int round = 0; round < 3; round++) {
    System.out.println("Round " + round);
    // Threads reach barrier and proceed
    // Barrier resets automatically
}

// Resetting barrier
barrier.reset();
```

**Related Terms**: [Barrier](#barrier), [CountDownLatch](#countdownlatch), [Thread Coordination](#thread-coordination), [Synchronization](#synchronization)

**Interview Tip**: Know the difference between CyclicBarrier and CountDownLatch. Understand when to use each.

---

## D

### Deadlock
**Category**: Concurrency Issues

**Definition**: A situation where two or more threads are blocked forever, waiting for each other to release resources. Deadlock occurs when circular wait conditions exist in resource acquisition.

**Practical Example**:
```java
// Deadlock scenario
class Account {
    private int balance;
    
    synchronized void transfer(Account other, int amount) {
        // Thread 1: locks this, waits for other
        // Thread 2: locks other, waits for this
        // DEADLOCK!
        this.balance -= amount;
        other.balance += amount;
    }
}

// Deadlock prevention - lock ordering
class SafeAccount {
    private int balance;
    private static final Object lock = new Object();
    
    void transfer(SafeAccount other, int amount) {
        Account first, second;
        if (System.identityHashCode(this) < System.identityHashCode(other)) {
            first = this;
            second = other;
        } else {
            first = other;
            second = this;
        }
        
        synchronized (first) {
            synchronized (second) {
                this.balance -= amount;
                other.balance += amount;
            }
        }
    }
}

// Deadlock prevention - timeout
ReentrantLock lock1 = new ReentrantLock();
ReentrantLock lock2 = new ReentrantLock();

try {
    if (lock1.tryLock(1, TimeUnit.SECONDS)) {
        try {
            if (lock2.tryLock(1, TimeUnit.SECONDS)) {
                try {
                    // Critical section
                } finally {
                    lock2.unlock();
                }
            }
        } finally {
            lock1.unlock();
        }
    }
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

**Related Terms**: [Synchronization](#synchronization), [Lock](#lock), [Race Condition](#race-condition), [Thread Safety](#thread-safety)

**Interview Tip**: Know the four conditions for deadlock and how to prevent them. Discuss strategies like lock ordering, timeouts, and resource hierarchy.

---

## E

### Executor
**Category**: Concurrency Utilities

**Definition**: An interface that abstracts the execution of asynchronous tasks. Executors decouple task submission from task execution, allowing for thread pooling and other execution strategies.

**Practical Example**:
```java
// Executor interface
Executor executor = new ThreadPoolExecutor(
    2,  // Core threads
    4,  // Max threads
    60, TimeUnit.SECONDS,  // Keep-alive time
    new LinkedBlockingQueue<>()
);

executor.execute(() -> System.out.println("Task executed"));

// ExecutorService (extends Executor)
ExecutorService service = Executors.newFixedThreadPool(2);

// Submit tasks
Future<Integer> future = service.submit(() -> {
    return 5 + 3;
});

Integer result = future.get();  // Wait for result

// Shutdown
service.shutdown();
service.awaitTermination(5, TimeUnit.SECONDS);

// Common factories
ExecutorService fixed = Executors.newFixedThreadPool(10);
ExecutorService cached = Executors.newCachedThreadPool();
ExecutorService single = Executors.newSingleThreadExecutor();
ExecutorService scheduled = Executors.newScheduledThreadPool(5);
```

**Related Terms**: [ExecutorService](#executorservice), [ThreadPool](#threadpool), [Future](#future), [Task](#task)

**Interview Tip**: Understand the Executor framework and its benefits. Know about different executor types and when to use each.

---

### ExecutorService
**Category**: Concurrency Utilities

**Definition**: An interface that extends Executor and provides methods for managing the lifecycle of executor threads and submitting tasks that return results (Futures).

**Practical Example**:
```java
// ExecutorService
ExecutorService service = Executors.newFixedThreadPool(3);

// Submit single task
Future<String> future = service.submit(() -> {
    Thread.sleep(1000);
    return "Result";
});

String result = future.get();  // Blocking wait

// Submit multiple tasks
List<Callable<Integer>> tasks = Arrays.asList(
    () -> 1 + 1,
    () -> 2 + 2,
    () -> 3 + 3
);

List<Future<Integer>> futures = service.invokeAll(tasks);

// Get all results
for (Future<Integer> f : futures) {
    System.out.println(f.get());
}

// Invoke any (returns first completed)
Integer firstResult = service.invokeAny(tasks);

// Shutdown
service.shutdown();
if (!service.awaitTermination(5, TimeUnit.SECONDS)) {
    service.shutdownNow();
}
```

**Related Terms**: [Executor](#executor), [ThreadPool](#threadpool), [Future](#future), [Callable](#callable)

**Interview Tip**: Know the lifecycle of ExecutorService. Understand the difference between shutdown() and shutdownNow().

---

## F

### Future
**Category**: Concurrency Utilities

**Definition**: An interface representing the result of an asynchronous computation. A Future allows you to check if the computation is complete, wait for its completion, and retrieve the result.

**Practical Example**:
```java
// Future
ExecutorService service = Executors.newFixedThreadPool(1);

Future<Integer> future = service.submit(() -> {
    Thread.sleep(2000);
    return 42;
});

// Check if done
if (!future.isDone()) {
    System.out.println("Still computing");
}

// Get result (blocking)
Integer result = future.get();

// Get result with timeout
try {
    Integer result2 = future.get(3, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    System.out.println("Computation took too long");
}

// Cancel task
boolean cancelled = future.cancel(true);  // true = interrupt if running

// CompletableFuture (Java 8+)
CompletableFuture<Integer> completable = CompletableFuture.supplyAsync(() -> {
    return 42;
});

completable.thenApply(n -> n * 2)
    .thenAccept(System.out::println);
```

**Related Terms**: [ExecutorService](#executorservice), [Callable](#callable), [CompletableFuture](#completablefuture), [Asynchronous](#asynchronous)

**Interview Tip**: Know the methods of Future interface. Understand the difference between Future and CompletableFuture.

---

## H

### Happens-Before
**Category**: Memory Visibility

**Definition**: A relationship that guarantees the visibility of memory operations across threads. If action A happens-before action B, then all effects of A are visible to B.

**Practical Example**:
```java
// Happens-before relationships

// 1. Program order
int x = 1;
int y = x + 1;  // y sees x = 1

// 2. Volatile writes/reads
volatile int flag = 0;
flag = 1;  // Write happens-before
int value = flag;  // Read

// 3. Synchronized blocks
synchronized (lock) {
    x = 1;  // Write
}
synchronized (lock) {
    int y = x;  // Read sees x = 1
}

// 4. Thread start
Thread thread = new Thread(() -> {
    // All actions before thread.start() happen-before
    // actions in the thread
});
thread.start();

// 5. Thread termination
thread.join();  // All actions in thread happen-before
// actions after join()

// 6. Volatile field access
volatile boolean ready = false;
volatile int value = 0;

// Writer thread
value = 42;
ready = true;

// Reader thread
if (ready) {
    int x = value;  // Guaranteed to see 42
}
```

**Related Terms**: [Memory Visibility](#memory-visibility), [Volatile](#volatile), [Synchronization](#synchronization), [Memory Barrier](#memory-barrier)

**Interview Tip**: Understand happens-before relationships and their importance for thread safety. Know the different sources of happens-before relationships.

---

## I

### Immutable Object
**Category**: Thread Safety

**Definition**: An object whose state cannot be changed after creation. Immutable objects are inherently thread-safe and can be safely shared between threads without synchronization.

**Practical Example**:
```java
// Immutable object
final class ImmutablePoint {
    private final int x;
    private final int y;
    
    public ImmutablePoint(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    public int getX() { return x; }
    public int getY() { return y; }
    
    public ImmutablePoint move(int dx, int dy) {
        return new ImmutablePoint(x + dx, y + dy);
    }
}

// Thread-safe without synchronization
ImmutablePoint point = new ImmutablePoint(0, 0);
// Multiple threads can safely read point

// Immutable collections
List<String> list = Collections.unmodifiableList(new ArrayList<>());
Map<String, Integer> map = Collections.unmodifiableMap(new HashMap<>());

// Java 9+ immutable collections
List<String> list2 = List.of("a", "b", "c");
Map<String, Integer> map2 = Map.of("key", 1);
```

**Related Terms**: [Thread Safety](#thread-safety), [Final](#final), [Immutability](#immutability), [Synchronization](#synchronization)

**Interview Tip**: Discuss the benefits of immutable objects for concurrency. Know how to create immutable classes.

---

## L

### Lock
**Category**: Synchronization

**Definition**: A synchronization mechanism that provides mutual exclusion, ensuring only one thread can access a resource at a time. Java provides explicit locks through the Lock interface in java.util.concurrent.locks.

**Practical Example**:
```java
// Lock interface
Lock lock = new ReentrantLock();

lock.lock();
try {
    // Critical section
    System.out.println("Protected code");
} finally {
    lock.unlock();
}

// Try lock with timeout
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // Critical section
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("Could not acquire lock");
}

// ReadWriteLock
ReadWriteLock rwLock = new ReentrantReadWriteLock();

// Multiple readers
rwLock.readLock().lock();
try {
    // Read operation
} finally {
    rwLock.readLock().unlock();
}

// Single writer
rwLock.writeLock().lock();
try {
    // Write operation
} finally {
    rwLock.writeLock().unlock();
}
```

**Related Terms**: [ReentrantLock](#reentrantlock), [ReadWriteLock](#readwritelock), [Synchronization](#synchronization), [Mutual Exclusion](#mutual-exclusion)

**Interview Tip**: Know the difference between Lock and synchronized. Understand when to use each. Know about fair locks and lock conditions.

---

### Lock-Free
**Category**: Concurrency

**Definition**: A programming technique that achieves thread safety without using locks. Lock-free algorithms use atomic operations (like CAS) to coordinate thread access without blocking.

**Practical Example**:
```java
// Lock-free counter
class LockFreeCounter {
    private AtomicInteger count = new AtomicInteger(0);
    
    public void increment() {
        count.incrementAndGet();
    }
    
    public int getCount() {
        return count.get();
    }
}

// Lock-free stack
class LockFreeStack<T> {
    private AtomicReference<Node<T>> head = new AtomicReference<>();
    
    static class Node<T> {
        final T value;
        Node<T> next;
        
        Node(T value) {
            this.value = value;
        }
    }
    
    public void push(T value) {
        Node<T> newHead = new Node<>(value);
        Node<T> oldHead;
        do {
            oldHead = head.get();
            newHead.next = oldHead;
        } while (!head.compareAndSet(oldHead, newHead));
    }
    
    public T pop() {
        Node<T> oldHead;
        Node<T> newHead;
        do {
            oldHead = head.get();
            if (oldHead == null) return null;
            newHead = oldHead.next;
        } while (!head.compareAndSet(oldHead, newHead));
        return oldHead.value;
    }
}

// Benefits
// - No blocking
// - Better scalability
// - No deadlock risk
```

**Related Terms**: [Atomic Operation](#atomic-operation), [CAS](#cas), [Synchronization](#synchronization), [Scalability](#scalability)

**Interview Tip**: Understand the benefits and challenges of lock-free programming. Know about the ABA problem and how to handle it.

---

## M

### Memory Barrier
**Category**: Memory Visibility

**Definition**: A CPU instruction that enforces ordering constraints on memory operations. Memory barriers ensure that memory operations are visible across threads and prevent reordering by the compiler or CPU.

**Practical Example**:
```java
// Memory barriers are implicit in Java

// Volatile field access creates memory barriers
volatile int flag = 0;
volatile int value = 0;

// Writer thread
value = 42;  // Store barrier
flag = 1;    // Store barrier

// Reader thread
if (flag == 1) {  // Load barrier
    int x = value;  // Load barrier - guaranteed to see 42
}

// Synchronized block creates memory barriers
synchronized (lock) {
    value = 42;  // Implicit store barrier
}

synchronized (lock) {
    int x = value;  // Implicit load barrier
}

// Explicit barriers (rare)
// VarHandle provides explicit control
VarHandle handle = MethodHandles.lookup()
    .findVarHandle(MyClass.class, "field", int.class);
handle.setRelease(obj, 42);  // Release barrier
int value = (int) handle.getAcquire(obj);  // Acquire barrier
```

**Related Terms**: [Volatile](#volatile), [Happens-Before](#happens-before), [Memory Visibility](#memory-visibility), [Synchronization](#synchronization)

**Interview Tip**: Understand the role of memory barriers in ensuring visibility. Know that Java abstracts away explicit barrier management.

---

### Memory Visibility
**Category**: Concurrency

**Definition**: The guarantee that changes made by one thread to shared variables are visible to other threads. Without proper synchronization, changes may not be visible due to caching and reordering.

**Practical Example**:
```java
// Without synchronization - visibility issue
class VisibilityProblem {
    private int value = 0;  // Not volatile
    
    void writer() {
        value = 42;  // May not be visible to reader
    }
    
    void reader() {
        int x = value;  // May see 0 even after writer sets 42
    }
}

// With volatile - guaranteed visibility
class VisibilitySolution {
    private volatile int value = 0;
    
    void writer() {
        value = 42;  // Visible to all readers
    }
    
    void reader() {
        int x = value;  // Always sees latest value
    }
}

// With synchronization - guaranteed visibility
class SynchronizedSolution {
    private int value = 0;
    
    synchronized void writer() {
        value = 42;
    }
    
    synchronized int reader() {
        return value;
    }
}

// Immutable objects - guaranteed visibility
class ImmutableSolution {
    private final int value;
    
    ImmutableSolution(int value) {
        this.value = value;
    }
    
    int getValue() {
        return value;  // Always safe
    }
}
```

**Related Terms**: [Volatile](#volatile), [Synchronization](#synchronization), [Happens-Before](#happens-before), [Memory Barrier](#memory-barrier)

**Interview Tip**: Understand the importance of memory visibility. Know the mechanisms that guarantee visibility: volatile, synchronized, and happens-before.

---

## M (continued)

### Mutex
**Category**: Synchronization

**Definition**: Short for "mutual exclusion", a synchronization mechanism that ensures only one thread can access a resource at a time. In Java, synchronized blocks and ReentrantLock provide mutex functionality.

**Practical Example**:
```java
// Mutex using synchronized
class Counter {
    private int count = 0;
    
    synchronized void increment() {
        count++;
    }
    
    synchronized int getCount() {
        return count;
    }
}

// Mutex using ReentrantLock
class CounterWithLock {
    private int count = 0;
    private Lock lock = new ReentrantLock();
    
    void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
    
    int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}

// Mutex with try-lock
class CounterWithTryLock {
    private int count = 0;
    private Lock lock = new ReentrantLock();
    
    boolean tryIncrement() {
        if (lock.tryLock()) {
            try {
                count++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }
}
```

**Related Terms**: [Lock](#lock), [Synchronization](#synchronization), [ReentrantLock](#reentrantlock), [Mutual Exclusion](#mutual-exclusion)

**Interview Tip**: Understand the concept of mutual exclusion. Know the difference between coarse-grained and fine-grained locking.

---

## R

### Race Condition
**Category**: Concurrency Issues

**Definition**: A situation where the behavior of a program depends on the relative timing of events in different threads. Race conditions occur when multiple threads access shared data without proper synchronization.

**Practical Example**:
```java
// Race condition
class UnsafeCounter {
    private int count = 0;
    
    void increment() {
        count++;  // Not atomic: read, increment, write
    }
    
    int getCount() {
        return count;
    }
}

// Race condition scenario
UnsafeCounter counter = new UnsafeCounter();
Thread t1 = new Thread(() -> {
    for (int i = 0; i < 1000; i++) {
        counter.increment();
    }
});
Thread t2 = new Thread(() -> {
    for (int i = 0; i < 1000; i++) {
        counter.increment();
    }
});
t1.start();
t2.start();
t1.join();
t2.join();
System.out.println(counter.getCount());  // May be < 2000

// Fix with synchronization
class SafeCounter {
    private int count = 0;
    
    synchronized void increment() {
        count++;
    }
    
    synchronized int getCount() {
        return count;
    }
}

// Fix with atomic variable
class AtomicCounter {
    private AtomicInteger count = new AtomicInteger(0);
    
    void increment() {
        count.incrementAndGet();
    }
    
    int getCount() {
        return count.get();
    }
}
```

**Related Terms**: [Synchronization](#synchronization), [Atomic Operation](#atomic-operation), [Thread Safety](#thread-safety), [Deadlock](#deadlock)

**Interview Tip**: Know how to identify and fix race conditions. Understand the importance of proper synchronization.

---

### ReentrantLock
**Category**: Synchronization

**Definition**: A reentrant mutual exclusion lock that can be acquired multiple times by the same thread. ReentrantLock provides more flexibility than synchronized blocks, including try-lock, fair locking, and condition variables.

**Practical Example**:
```java
// ReentrantLock
ReentrantLock lock = new ReentrantLock();

// Basic usage
lock.lock();
try {
    // Critical section
} finally {
    lock.unlock();
}

// Try lock with timeout
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // Critical section
    } finally {
        lock.unlock();
    }
}

// Reentrant behavior
class Reentrant {
    private ReentrantLock lock = new ReentrantLock();
    
    void outer() {
        lock.lock();
        try {
            System.out.println("Outer");
            inner();
        } finally {
            lock.unlock();
        }
    }
    
    void inner() {
        lock.lock();  // Same thread can acquire again
        try {
            System.out.println("Inner");
        } finally {
            lock.unlock();
        }
    }
}

// Fair lock
ReentrantLock fairLock = new ReentrantLock(true);

// Condition variables
Condition condition = lock.newCondition();
lock.lock();
try {
    while (!condition) {
        condition.await();
    }
    // Process
    condition.signalAll();
} finally {
    lock.unlock();
}
```

**Related Terms**: [Lock](#lock), [Synchronization](#synchronization), [Condition Variable](#condition-variable), [Fair Locking](#fair-locking)

**Interview Tip**: Know the advantages of ReentrantLock over synchronized. Understand reentrancy and fair locking.

---

## S

### Semaphore
**Category**: Concurrency Utilities

**Definition**: A synchronization primitive that maintains a count and allows threads to acquire and release permits. Semaphores can be used to control access to a limited number of resources.

**Practical Example**:
```java
// Semaphore
Semaphore semaphore = new Semaphore(3);  // 3 permits

// Acquire permit
semaphore.acquire();
try {
    // Use resource
} finally {
    semaphore.release();
}

// Try acquire with timeout
if (semaphore.tryAcquire(1, TimeUnit.SECONDS)) {
    try {
        // Use resource
    } finally {
        semaphore.release();
    }
}

// Connection pool using Semaphore
class ConnectionPool {
    private Semaphore semaphore;
    private List<Connection> connections;
    
    public ConnectionPool(int poolSize) {
        semaphore = new Semaphore(poolSize);
        connections = new ArrayList<>();
        for (int i = 0; i < poolSize; i++) {
            connections.add(new Connection());
        }
    }
    
    public Connection getConnection() throws InterruptedException {
        semaphore.acquire();
        synchronized (connections) {
            return connections.remove(0);
        }
    }
    
    public void releaseConnection(Connection conn) {
        synchronized (connections) {
            connections.add(conn);
        }
        semaphore.release();
    }
}

// Binary semaphore (mutex)
Semaphore mutex = new Semaphore(1);
```

**Related Terms**: [Synchronization](#synchronization), [Barrier](#barrier), [Resource Pool](#resource-pool), [Concurrency Utilities](#concurrency-utilities)

**Interview Tip**: Know the difference between Semaphore and Mutex. Understand use cases for semaphores.

---

### Synchronization
**Category**: Concurrency

**Definition**: The coordination of multiple threads to ensure correct behavior when accessing shared resources. Synchronization mechanisms include locks, semaphores, monitors, and atomic operations.

**Practical Example**:
```java
// Synchronized method
class Counter {
    private int count = 0;
    
    synchronized void increment() {
        count++;
    }
    
    synchronized int getCount() {
        return count;
    }
}

// Synchronized block
class Counter2 {
    private int count = 0;
    private Object lock = new Object();
    
    void increment() {
        synchronized (lock) {
            count++;
        }
    }
    
    int getCount() {
        synchronized (lock) {
            return count;
        }
    }
}

// Volatile (for visibility, not atomicity)
class Flag {
    private volatile boolean flag = false;
    
    void setFlag() {
        flag = true;
    }
    
    boolean getFlag() {
        return flag;
    }
}

// Atomic variables
class AtomicCounter {
    private AtomicInteger count = new AtomicInteger(0);
    
    void increment() {
        count.incrementAndGet();
    }
    
    int getCount() {
        return count.get();
    }
}

// Lock interface
class LockCounter {
    private int count = 0;
    private Lock lock = new ReentrantLock();
    
    void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```

**Related Terms**: [Lock](#lock), [Volatile](#volatile), [Atomic Operation](#atomic-operation), [Thread Safety](#thread-safety)

**Interview Tip**: Know the different synchronization mechanisms and when to use each. Understand the performance implications of synchronization.

---

## T

### Thread
**Category**: Concurrency

**Definition**: The smallest unit of execution within a process. Java supports multithreading, allowing multiple threads to execute concurrently within the same program.

**Practical Example**:
```java
// Creating threads
class MyThread extends Thread {
    public void run() {
        System.out.println("Thread running");
    }
}

MyThread thread = new MyThread();
thread.start();  // Start the thread

// Using Runnable
Thread thread2 = new Thread(() -> {
    System.out.println("Thread running");
});
thread2.start();

// Thread lifecycle
Thread thread3 = new Thread(() -> {
    System.out.println("Running");
});

thread3.start();  // NEW -> RUNNABLE
// Thread executes -> RUNNING
// Thread completes -> TERMINATED

// Thread states
Thread.State state = thread3.getState();

// Thread operations
Thread.sleep(1000);  // Sleep
thread3.join();  // Wait for completion
Thread.yield();  // Hint to scheduler
```

**Related Terms**: [Runnable](#runnable), [Concurrency](#concurrency), [Multithreading](#multithreading), [Thread Pool](#threadpool)

**Interview Tip**: Know the thread lifecycle and states. Understand when to extend Thread vs. implement Runnable.

---

### Thread Pool
**Category**: Concurrency Utilities

**Definition**: A collection of pre-created threads that are reused to execute multiple tasks. Thread pools improve performance by avoiding the overhead of creating and destroying threads.

**Practical Example**:
```java
// Creating thread pools
ExecutorService fixedPool = Executors.newFixedThreadPool(5);
ExecutorService cachedPool = Executors.newCachedThreadPool();
ExecutorService singlePool = Executors.newSingleThreadExecutor();
ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(5);

// Submitting tasks
fixedPool.execute(() -> System.out.println("Task"));

Future<Integer> future = fixedPool.submit(() -> 42);

// Shutdown
fixedPool.shutdown();
fixedPool.awaitTermination(5, TimeUnit.SECONDS);

// Custom thread pool
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,      // Core threads
    4,      // Max threads
    60,     // Keep-alive time
    TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100),  // Queue
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.AbortPolicy()  // Rejection policy
);
```

**Related Terms**: [Executor](#executor), [ExecutorService](#executorservice), [Thread](#thread), [Task](#task)

**Interview Tip**: Know the different types of thread pools and when to use each. Understand thread pool sizing and rejection policies.

---

## V

### Volatile
**Category**: Memory Visibility

**Definition**: A keyword that ensures visibility of changes to a variable across threads. Volatile variables are not cached in registers and are always read from and written to main memory.

**Practical Example**:
```java
// Volatile variable
volatile boolean flag = false;
volatile int value = 0;

// Writer thread
void writer() {
    value = 42;
    flag = true;  // Visible to all readers
}

// Reader thread
void reader() {
    if (flag) {
        int x = value;  // Guaranteed to see 42
    }
}

// Volatile vs. synchronized
class VolatileExample {
    private volatile int count = 0;
    
    void increment() {
        count++;  // NOT atomic, but visible
    }
}

class SynchronizedExample {
    private int count = 0;
    
    synchronized void increment() {
        count++;  // Atomic and visible
    }
}

// Double-checked locking with volatile
class Singleton {
    private static volatile Singleton instance;
    
    static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**Related Terms**: [Memory Visibility](#memory-visibility), [Synchronization](#synchronization), [Atomic Operation](#atomic-operation), [Happens-Before](#happens-before)

**Interview Tip**: Know the difference between volatile and synchronized. Understand that volatile guarantees visibility but not atomicity.

---

## W

### Wait/Notify
**Category**: Synchronization

**Definition**: Methods on the Object class that allow threads to communicate and coordinate. `wait()` causes a thread to release a lock and wait for a notification, while `notify()` wakes up a waiting thread.

**Practical Example**:
```java
// Wait/Notify pattern
class ProducerConsumer {
    private Queue<Integer> queue = new LinkedList<>();
    private int capacity = 10;
    
    public synchronized void produce(int value) throws InterruptedException {
        while (queue.size() == capacity) {
            wait();  // Wait for space
        }
        queue.add(value);
        notifyAll();  // Notify consumers
    }
    
    public synchronized int consume() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();  // Wait for items
        }
        int value = queue.poll();
        notifyAll();  // Notify producers
        return value;
    }
}

// Spurious wakeup protection
synchronized (lock) {
    while (!condition) {
        lock.wait();  // Always use while, not if
    }
    // Process
}

// Condition variables (preferred)
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();

lock.lock();
try {
    while (!condition) {
        condition.await();
    }
    // Process
    condition.signalAll();
} finally {
    lock.unlock();
}
```

**Related Terms**: [Condition Variable](#condition-variable), [Synchronization](#synchronization), [Monitor](#monitor), [Thread Coordination](#thread-coordination)

**Interview Tip**: Know the wait/notify pattern. Understand spurious wakeups and why while loops are necessary. Know when to use condition variables instead.

---

This glossary provides comprehensive coverage of Java concurrency terminology essential for senior-level interviews.

# Concurrency & Multithreading - Cheat Sheet

Complete guide to Java concurrency with patterns, synchronization, and best practices.

## 🧵 Thread Basics

### Creating Threads
```java
// Method 1: Extend Thread
class MyThread extends Thread {
    public void run() {
        System.out.println("Thread running");
    }
}
Thread t = new MyThread();
t.start();  // Calls run() in new thread

// Method 2: Implement Runnable (preferred)
Thread t = new Thread(() -> {
    System.out.println("Thread running");
});
t.start();

// Method 3: ExecutorService (best practice)
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(() -> {
    System.out.println("Task running");
});
executor.shutdown();
```

### Thread States
```
NEW → RUNNABLE → RUNNING → WAITING/BLOCKED → TERMINATED
                    ↑                           ↓
                    └───────────────────────────┘
```

### Thread Methods
```java
Thread t = new Thread(() -> { });

// Control
t.start();                      // Start thread
t.join();                       // Wait for completion
t.interrupt();                  // Request interruption
Thread.sleep(1000);             // Sleep 1 second
Thread.yield();                 // Hint to scheduler

// Information
t.getName();                    // Get thread name
t.getPriority();                // 1-10 (default 5)
t.isAlive();                    // Is thread running?
Thread.currentThread();         // Get current thread
Thread.activeCount();           // Active threads in group
```

## 🔒 Synchronization

### Synchronized Methods
```java
public class Counter {
    private int count = 0;
    
    // Synchronized method - locks on 'this'
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int getCount() {
        return count;
    }
    
    // Static synchronized - locks on class object
    public static synchronized void staticMethod() {
        // Only one thread at a time across all instances
    }
}
```

### Synchronized Blocks
```java
public class Counter {
    private int count = 0;
    private Object lock = new Object();
    
    public void increment() {
        synchronized (lock) {
            count++;  // Only this block is synchronized
        }
    }
    
    // Lock on specific object
    public void method() {
        synchronized (this) {
            // Lock on current instance
        }
        
        synchronized (Counter.class) {
            // Lock on class object
        }
    }
}
```

### Volatile Keyword
```java
public class Flag {
    private volatile boolean flag = false;  // Visibility guarantee
    
    public void setFlag() {
        flag = true;  // Immediately visible to other threads
    }
    
    public boolean getFlag() {
        return flag;  // Always reads from memory
    }
}

// volatile guarantees:
// - Visibility: Changes visible to all threads
// - Ordering: Prevents instruction reordering
// - NOT atomicity: flag++ is still not atomic
```

## 🔐 Locks (java.util.concurrent.locks)

### ReentrantLock
```java
import java.util.concurrent.locks.ReentrantLock;

public class Counter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();
    
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();  // Always unlock in finally
        }
    }
    
    // Try lock with timeout
    public boolean tryIncrement() {
        if (lock.tryLock(1, TimeUnit.SECONDS)) {
            try {
                count++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }
    
    // Interruptible lock
    public void interruptibleIncrement() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```

### ReadWriteLock
```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Cache {
    private Map<String, String> cache = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    public String get(String key) {
        lock.readLock().lock();
        try {
            return cache.get(key);  // Multiple readers allowed
        } finally {
            lock.readLock().unlock();
        }
    }
    
    public void put(String key, String value) {
        lock.writeLock().lock();
        try {
            cache.put(key, value);  // Exclusive write access
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

### Condition Variables
```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class BoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    public BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }
    
    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();  // Wait until not full
            }
            queue.offer(item);
            notEmpty.signalAll();  // Signal waiting consumers
        } finally {
            lock.unlock();
        }
    }
    
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();  // Wait until not empty
            }
            T item = queue.poll();
            notFull.signalAll();  // Signal waiting producers
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

## ⚛️ Atomic Classes

```java
import java.util.concurrent.atomic.*;

// AtomicInteger
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();                  // Atomic increment
counter.getAndIncrement();                  // Get then increment
counter.addAndGet(5);                       // Add atomically
counter.compareAndSet(0, 1);                // CAS operation

// AtomicLong
AtomicLong longCounter = new AtomicLong(0);
longCounter.incrementAndGet();

// AtomicBoolean
AtomicBoolean flag = new AtomicBoolean(false);
flag.set(true);
flag.getAndSet(false);

// AtomicReference
AtomicReference<String> ref = new AtomicReference<>("initial");
ref.set("new value");
ref.compareAndSet("new value", "another");

// AtomicIntegerArray
AtomicIntegerArray array = new AtomicIntegerArray(10);
array.incrementAndGet(0);

// Performance: Lock-free, uses CAS (Compare-And-Swap)
// Use case: Counters, flags, simple state management
```

## 🏊 Concurrent Collections

### ConcurrentHashMap
```java
Map<String, Integer> map = new ConcurrentHashMap<>();
map.put("key1", 100);
map.get("key1");
map.remove("key1");

// Atomic operations
map.putIfAbsent("key2", 200);
map.computeIfAbsent("key3", k -> 300);
map.computeIfPresent("key1", (k, v) -> v + 1);

// Iteration is weakly consistent
for (Map.Entry<String, Integer> entry : map.entrySet()) { }

// Performance: Segment-based locking (16 segments by default)
// Better than Collections.synchronizedMap()
```

### CopyOnWriteArrayList
```java
List<String> list = new CopyOnWriteArrayList<>();
list.add("item1");
list.get(0);
list.remove(0);

// Iteration is safe (snapshot)
for (String item : list) { }

// Performance: O(1) reads, O(n) writes (copies array)
// Use case: Read-heavy workloads
```

### ConcurrentLinkedQueue
```java
Queue<String> queue = new ConcurrentLinkedQueue<>();
queue.offer("item1");
queue.poll();
queue.peek();

// Lock-free, uses CAS
// Performance: O(1) operations
// Use case: Producer-consumer patterns
```

### BlockingQueue
```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

BlockingQueue<String> queue = new LinkedBlockingQueue<>(100);

// Blocking operations
queue.put("item");                          // Wait if full
queue.take();                               // Wait if empty
queue.offer("item", 1, TimeUnit.SECONDS);   // Timeout
queue.poll(1, TimeUnit.SECONDS);            // Timeout

// Non-blocking operations
queue.offer("item");                        // Return false if full
queue.poll();                               // Return null if empty

// Use case: Producer-consumer, thread pools
```

## 🔄 Synchronization Utilities

### CountDownLatch
```java
import java.util.concurrent.CountDownLatch;

CountDownLatch latch = new CountDownLatch(3);

// Worker threads
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        System.out.println("Working...");
        latch.countDown();  // Decrement count
    }).start();
}

latch.await();  // Wait for all to complete
System.out.println("All done!");

// Use case: Wait for multiple threads to complete
```

### CyclicBarrier
```java
import java.util.concurrent.CyclicBarrier;

CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("All threads reached barrier");
});

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        System.out.println("Thread waiting at barrier");
        barrier.await();  // Wait for all threads
        System.out.println("Thread continuing");
    }).start();
}

// Use case: Synchronize multiple threads at a point
// Reusable (unlike CountDownLatch)
```

### Semaphore
```java
import java.util.concurrent.Semaphore;

Semaphore semaphore = new Semaphore(3);  // 3 permits

for (int i = 0; i < 10; i++) {
    new Thread(() -> {
        try {
            semaphore.acquire();  // Wait for permit
            System.out.println("Using resource");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            semaphore.release();  // Release permit
        }
    }).start();
}

// Use case: Limit concurrent access to resource
```

### Phaser
```java
import java.util.concurrent.Phaser;

Phaser phaser = new Phaser(3);  // 3 parties

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        System.out.println("Phase 1");
        phaser.arriveAndAwaitAdvance();
        System.out.println("Phase 2");
        phaser.arriveAndAwaitAdvance();
    }).start();
}

// Use case: Multi-phase synchronization
```

## 🏭 ExecutorService & Thread Pools

### Basic Usage
```java
import java.util.concurrent.*;

// Create thread pool
ExecutorService executor = Executors.newFixedThreadPool(10);

// Submit tasks
executor.submit(() -> {
    System.out.println("Task 1");
});

Future<Integer> future = executor.submit(() -> {
    return 42;
});

Integer result = future.get();  // Wait for result
executor.shutdown();            // Stop accepting tasks
executor.awaitTermination(10, TimeUnit.SECONDS);  // Wait for completion
```

### Different Executor Types
```java
// Fixed thread pool
ExecutorService fixed = Executors.newFixedThreadPool(10);

// Single thread executor
ExecutorService single = Executors.newSingleThreadExecutor();

// Cached thread pool (unbounded)
ExecutorService cached = Executors.newCachedThreadPool();

// Scheduled executor
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(5);
scheduled.schedule(() -> { }, 1, TimeUnit.SECONDS);
scheduled.scheduleAtFixedRate(() -> { }, 0, 1, TimeUnit.SECONDS);

// Work stealing pool (ForkJoinPool)
ExecutorService workStealing = Executors.newWorkStealingPool();
```

### Future & Callable
```java
Callable<Integer> task = () -> {
    return 42;
};

Future<Integer> future = executor.submit(task);

try {
    Integer result = future.get();              // Wait for result
    Integer result2 = future.get(5, TimeUnit.SECONDS);  // With timeout
    boolean done = future.isDone();
    boolean cancelled = future.isCancelled();
    future.cancel(true);                        // Cancel task
} catch (InterruptedException | ExecutionException e) {
    // Handle exceptions
}
```

## 📊 Comparison Table

| Tool | Purpose | Reusable | Thread Count |
|------|---------|----------|--------------|
| CountDownLatch | Wait for N threads | No | N |
| CyclicBarrier | Synchronize N threads | Yes | N |
| Semaphore | Limit concurrent access | Yes | Variable |
| Phaser | Multi-phase sync | Yes | Variable |

## ⚠️ Common Pitfalls

1. **Forgetting to unlock**
   ```java
   // ❌ Wrong - may deadlock
   lock.lock();
   doSomething();
   lock.unlock();
   
   // ✅ Correct
   lock.lock();
   try {
       doSomething();
   } finally {
       lock.unlock();
   }
   ```

2. **Deadlock with multiple locks**
   ```java
   // ❌ Risk of deadlock - inconsistent lock order
   public void transfer(Account a, Account b) {
       synchronized (a) {
           synchronized (b) { }
       }
   }
   
   // ✅ Correct - consistent lock order
   public void transfer(Account a, Account b) {
       Account first = a.id < b.id ? a : b;
       Account second = a.id < b.id ? b : a;
       synchronized (first) {
           synchronized (second) { }
       }
   }
   ```

3. **Not handling InterruptedException**
   ```java
   // ❌ Wrong - swallows interruption
   try {
       Thread.sleep(1000);
   } catch (InterruptedException e) {
       // Ignored
   }
   
   // ✅ Correct
   try {
       Thread.sleep(1000);
   } catch (InterruptedException e) {
       Thread.currentThread().interrupt();
       // Handle or propagate
   }
   ```

## 🎓 Interview Tips

- **Know the difference**: synchronized vs Lock, volatile vs atomic
- **Understand trade-offs**: Performance vs simplicity
- **Avoid deadlocks**: Lock ordering, timeouts
- **Use high-level tools**: Prefer ExecutorService over raw threads
- **Test thoroughly**: Concurrency bugs are hard to reproduce

---

**Related**: See Performance Optimization cheat sheet for profiling concurrent code

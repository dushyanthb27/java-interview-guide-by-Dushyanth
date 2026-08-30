# Q12: What Are Condition Variables?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What are `Condition` variables? How do they differ from `wait()`/`notify()`? How do you use them?

---

## Answer

### Condition Interface

`Condition` is the `java.util.concurrent` replacement for `wait()`/`notify()`. It allows multiple wait-sets per lock, giving finer-grained thread coordination.

```
Object Monitor (wait/notify):        Condition (await/signal):
┌─────────────────────┐             ┌─────────────────────┐
│  synchronized(lock) │             │  lock.lock()        │
│                     │             │                     │
│  ONE wait-set       │             │  Condition notFull  │
│  ┌───────────────┐  │             │  ┌───────────────┐  │
│  │ wait()        │  │             │  │ await()       │  │
│  │ notify()      │  │             │  │ signal()      │  │
│  │ notifyAll()   │  │             │  └───────────────┘  │
│  └───────────────┘  │             │                     │
│                     │             │  Condition notEmpty │
│                     │             │  ┌───────────────┐  │
│                     │             │  │ await()       │  │
│                     │             │  │ signal()      │  │
│                     │             │  └───────────────┘  │
└─────────────────────┘             └─────────────────────┘
```

### Condition vs wait/notify

| Feature | wait/notify | Condition |
|---------|------------|-----------|
| Associated with | Object monitor | `Lock` |
| Multiple conditions | ❌ One per object | ✅ Multiple via `newCondition()` |
| Wait method | `wait()` | `await()` |
| Signal method | `notify()` / `notifyAll()` | `signal()` / `signalAll()` |
| Timed wait | `wait(ms)` | `await(time, unit)` |
| Deadline-based | ❌ | ✅ `awaitUntil(Date)` |
| Uninterruptible wait | ❌ | ✅ `awaitUninterruptibly()` |
| Nanos precision | ❌ | ✅ `awaitNanos(nanos)` |

### Basic Usage

```java
public class ConditionBasics {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();
    private boolean ready = false;

    // Thread A: Wait for condition
    public void waitForReady() throws InterruptedException {
        lock.lock();
        try {
            while (!ready) {            // Always use while loop (spurious wakeups)
                condition.await();       // Releases lock and waits
            }
            System.out.println("Condition met! Proceeding...");
        } finally {
            lock.unlock();
        }
    }

    // Thread B: Signal condition
    public void makeReady() {
        lock.lock();
        try {
            ready = true;
            condition.signal();          // Wake up one waiting thread
            // condition.signalAll();    // Wake up ALL waiting threads
        } finally {
            lock.unlock();
        }
    }
}
```

### Bounded Buffer (Classic Producer-Consumer)

```java
public class BoundedBuffer<T> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();   // Producers wait on this
    private final Condition notEmpty = lock.newCondition();  // Consumers wait on this

    private final Object[] buffer;
    private int count = 0;
    private int putIndex = 0;
    private int takeIndex = 0;

    public BoundedBuffer(int capacity) {
        buffer = new Object[capacity];
    }

    // Producer
    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            // Wait while buffer is full
            while (count == buffer.length) {
                System.out.println("Buffer full, producer waiting...");
                notFull.await();
            }

            buffer[putIndex] = item;
            putIndex = (putIndex + 1) % buffer.length;
            count++;

            System.out.println("Produced: " + item + " (count=" + count + ")");
            notEmpty.signal(); // Wake up ONE consumer
        } finally {
            lock.unlock();
        }
    }

    // Consumer
    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lock();
        try {
            // Wait while buffer is empty
            while (count == 0) {
                System.out.println("Buffer empty, consumer waiting...");
                notEmpty.await();
            }

            T item = (T) buffer[takeIndex];
            buffer[takeIndex] = null;
            takeIndex = (takeIndex + 1) % buffer.length;
            count--;

            System.out.println("Consumed: " + item + " (count=" + count + ")");
            notFull.signal(); // Wake up ONE producer
            return item;
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        BoundedBuffer<Integer> buffer = new BoundedBuffer<>(5);

        // Producer thread
        Thread producer = new Thread(() -> {
            for (int i = 0; i < 20; i++) {
                try {
                    buffer.put(i);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }, "Producer");

        // Consumer thread
        Thread consumer = new Thread(() -> {
            for (int i = 0; i < 20; i++) {
                try {
                    buffer.take();
                    Thread.sleep(100); // Simulate processing
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }, "Consumer");

        producer.start();
        consumer.start();
    }
}
```

### Multiple Conditions Example: Read/Write Coordinator

```java
public class ReadWriteCoordinator {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition canRead = lock.newCondition();
    private final Condition canWrite = lock.newCondition();

    private int activeReaders = 0;
    private boolean writerActive = false;
    private int waitingWriters = 0;

    // Writer-preference ReadWriteLock
    public void acquireReadLock() throws InterruptedException {
        lock.lock();
        try {
            while (writerActive || waitingWriters > 0) {
                canRead.await(); // Wait if writer is active or writers are waiting
            }
            activeReaders++;
        } finally {
            lock.unlock();
        }
    }

    public void releaseReadLock() {
        lock.lock();
        try {
            activeReaders--;
            if (activeReaders == 0) {
                canWrite.signal(); // Last reader signals waiting writer
            }
        } finally {
            lock.unlock();
        }
    }

    public void acquireWriteLock() throws InterruptedException {
        lock.lock();
        try {
            waitingWriters++;
            while (activeReaders > 0 || writerActive) {
                canWrite.await();
            }
            waitingWriters--;
            writerActive = true;
        } finally {
            lock.unlock();
        }
    }

    public void releaseWriteLock() {
        lock.lock();
        try {
            writerActive = false;
            canRead.signalAll(); // Wake all waiting readers
            canWrite.signal();   // Wake one waiting writer
        } finally {
            lock.unlock();
        }
    }
}
```

### Timed and Uninterruptible Await

```java
public class AdvancedConditions {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition dataAvailable = lock.newCondition();
    private String data = null;

    // Timed await: wait up to a maximum time
    public String waitForData(long timeoutMs) throws InterruptedException {
        lock.lock();
        try {
            long nanosRemaining = TimeUnit.MILLISECONDS.toNanos(timeoutMs);
            while (data == null) {
                if (nanosRemaining <= 0) {
                    return null; // Timeout expired
                }
                nanosRemaining = dataAvailable.awaitNanos(nanosRemaining);
            }
            return data;
        } finally {
            lock.unlock();
        }
    }

    // Deadline-based await
    public String waitUntilDeadline(Date deadline) throws InterruptedException {
        lock.lock();
        try {
            while (data == null) {
                if (!dataAvailable.awaitUntil(deadline)) {
                    return null; // Deadline passed
                }
            }
            return data;
        } finally {
            lock.unlock();
        }
    }

    // Uninterruptible await: cannot be interrupted
    public String waitUninterruptible() {
        lock.lock();
        try {
            while (data == null) {
                dataAvailable.awaitUninterruptibly();
                // Thread.interrupt() won't break this wait
            }
            return data;
        } finally {
            lock.unlock();
        }
    }

    public void provideData(String newData) {
        lock.lock();
        try {
            data = newData;
            dataAvailable.signalAll();
        } finally {
            lock.unlock();
        }
    }
}
```

### Comparison with wait/notify

```java
// Old way: wait/notify
public class OldStyleBuffer {
    private final List<Object> items = new ArrayList<>();
    private final int capacity = 10;

    public synchronized void put(Object item) throws InterruptedException {
        while (items.size() == capacity) {
            wait(); // ❌ Wakes up producers AND consumers
        }
        items.add(item);
        notifyAll(); // ❌ Must wake ALL because we can't target consumers only
    }

    public synchronized Object take() throws InterruptedException {
        while (items.isEmpty()) {
            wait();
        }
        Object item = items.remove(0);
        notifyAll(); // ❌ Must wake ALL because we can't target producers only
        return item;
    }
}

// New way: Condition
public class NewStyleBuffer {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    // ✅ signal() targets ONLY the right threads
    // ✅ More efficient than notifyAll()
}
```

---

## Anti-Patterns

❌ **Calling await() without holding the lock:**
```java
condition.await(); // ❌ IllegalMonitorStateException
```

❌ **Using if instead of while for condition checks:**
```java
if (!ready) { condition.await(); } // ❌ Spurious wakeups!
while (!ready) { condition.await(); } // ✅ Always loop
```

❌ **Forgetting to signal after state change:**
```java
lock.lock();
try {
    data = newData;
    // ❌ Forgot condition.signal()! Waiting threads never wake up
} finally { lock.unlock(); }
```

## Best Practices

✅ Always await in a `while` loop (guard against spurious wakeups)  
✅ Use separate `Condition` objects for different wait reasons  
✅ Use `signal()` (not `signalAll()`) when only one waiter needs to wake up  
✅ Always call `signal()`/`signalAll()` while holding the lock  
✅ Use `awaitNanos()` for timed waits to handle remaining time correctly  
✅ Consider `BlockingQueue` before implementing your own producer-consumer

---

[← ReadWriteLock](./q11-readwrite-lock.md) | [Next: Deadlock →](./q13-deadlock.md)

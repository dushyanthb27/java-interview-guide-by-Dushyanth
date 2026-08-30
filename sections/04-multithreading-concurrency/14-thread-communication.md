# Q14: How Does Thread Communication Work (wait/notify)?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
How do `wait()`, `notify()`, and `notifyAll()` work? How do threads communicate with each other?

---

## Answer

### Object Monitor Mechanism

Every Java object has an associated monitor with a **wait set** of threads:

```
┌────────────────────────────────────┐
│         Object Monitor             │
│                                    │
│  ┌──────────────┐   ┌──────────┐  │
│  │ Entry Set    │   │ Wait Set │  │
│  │              │   │          │  │
│  │ Thread B ──► │   │ Thread C │  │
│  │ Thread D ──► │   │ Thread E │  │
│  │              │   │          │  │
│  └──────┬───────┘   └────┬─────┘  │
│         │                │        │
│         ▼       notify() │        │
│  ┌──────────────┐   ◄────┘        │
│  │  Owner       │                 │
│  │  Thread A    │                 │
│  └──────────────┘                 │
└────────────────────────────────────┘

Entry Set: Threads waiting to acquire the monitor lock
Wait Set: Threads that called wait() and released the lock
Owner: Thread currently holding the monitor lock
```

### wait(), notify(), notifyAll()

```java
public class WaitNotifyBasics {
    private final Object monitor = new Object();
    private boolean dataReady = false;

    // Thread A: Wait for data
    public void waitForData() throws InterruptedException {
        synchronized (monitor) {
            // ✅ ALWAYS wait in a while loop (not if)
            while (!dataReady) {
                System.out.println(Thread.currentThread().getName() + " waiting...");
                monitor.wait(); // 1. Releases the monitor lock
                                // 2. Thread enters WAITING state
                                // 3. Moves to the wait set
                                // 4. When notified: re-acquires lock, resumes
            }
            System.out.println("Data is ready! Processing...");
        }
    }

    // Thread B: Produce data and notify
    public void produceData() {
        synchronized (monitor) {
            // Prepare data
            dataReady = true;
            System.out.println("Data produced, notifying...");

            monitor.notify();    // Wake up ONE waiting thread
            // monitor.notifyAll(); // Wake up ALL waiting threads

            // Note: The waiting thread doesn't resume until
            // this synchronized block exits (lock is released)
        }
    }
}
```

### Why Always Wait in a While Loop?

```java
// ❌ WRONG: Using if
synchronized (monitor) {
    if (!condition) {
        monitor.wait(); // What if condition is still false after wakeup?
    }
    // Assumes condition is true here - UNSAFE!
}

// ✅ CORRECT: Using while
synchronized (monitor) {
    while (!condition) {
        monitor.wait(); // Re-checks condition after each wakeup
    }
    // Condition is GUARANTEED to be true here
}
```

**Reasons for spurious wakeups:**
1. **Spurious wakeups** - JVM/OS may wake a thread without `notify()`
2. **notifyAll()** - Multiple threads wake up, but only one should proceed
3. **Condition changed** - Between `notify()` and lock reacquisition, another thread may have changed the state

### notify() vs notifyAll()

```java
public class NotifyComparison {
    private final Object lock = new Object();

    // notify(): Wakes up ONE arbitrary waiting thread
    // ✅ Use when: Any single waiter can handle the event
    // ⚠️ Risk: May wake wrong thread in multi-condition scenarios
    public void signalOne() {
        synchronized (lock) {
            lock.notify();
        }
    }

    // notifyAll(): Wakes up ALL waiting threads
    // ✅ Use when: Multiple threads waiting for different conditions
    // ❌ Downside: Thundering herd (all wake up, most go back to sleep)
    public void signalAll() {
        synchronized (lock) {
            lock.notifyAll();
        }
    }
}
```

### Why Must wait/notify Be in Synchronized Block?

```java
// ❌ Without synchronized - IllegalMonitorStateException!
public void broken() throws InterruptedException {
    Object monitor = new Object();
    monitor.wait();   // Throws IllegalMonitorStateException
    monitor.notify(); // Throws IllegalMonitorStateException
}

// WHY? Without synchronization, there's a race condition:
//
// Thread A:                      Thread B:
//   if (!condition) {
//                                  condition = true;
//                                  notify();  // Lost! A hasn't called wait() yet
//   wait(); ← WAITS FOREVER (missed the notification)
//
// With synchronization, the check-then-wait is atomic:
//
// Thread A:
//   synchronized (monitor) {
//     while (!condition) {
//       wait();   // Atomically releases lock and waits
//     }
//   }
```

### Producer-Consumer Pattern

```java
public class ProducerConsumer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int CAPACITY = 5;

    // Producer
    public void produce() throws InterruptedException {
        int value = 0;
        while (true) {
            synchronized (this) {
                while (queue.size() == CAPACITY) {
                    System.out.println("Queue full, producer waiting...");
                    wait();
                }
                queue.add(value);
                System.out.println("Produced: " + value);
                value++;
                notifyAll(); // Wake consumers
            }
            Thread.sleep(100);
        }
    }

    // Consumer
    public void consume() throws InterruptedException {
        while (true) {
            synchronized (this) {
                while (queue.isEmpty()) {
                    System.out.println("Queue empty, consumer waiting...");
                    wait();
                }
                int value = queue.poll();
                System.out.println("Consumed: " + value);
                notifyAll(); // Wake producers
            }
            Thread.sleep(200);
        }
    }

    public static void main(String[] args) {
        ProducerConsumer pc = new ProducerConsumer();

        new Thread(() -> {
            try { pc.produce(); } catch (InterruptedException e) {}
        }, "Producer").start();

        new Thread(() -> {
            try { pc.consume(); } catch (InterruptedException e) {}
        }, "Consumer-1").start();

        new Thread(() -> {
            try { pc.consume(); } catch (InterruptedException e) {}
        }, "Consumer-2").start();
    }
}
```

### Modern Alternatives

```java
public class ModernAlternatives {

    // ✅ BlockingQueue - preferred over wait/notify
    private final BlockingQueue<String> queue = new LinkedBlockingQueue<>(10);

    public void producer() throws InterruptedException {
        queue.put("item"); // Blocks if queue is full (no manual wait/notify!)
    }

    public String consumer() throws InterruptedException {
        return queue.take(); // Blocks if queue is empty
    }

    // ✅ Condition variables - more flexible than wait/notify
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition hasData = lock.newCondition();

    // ✅ CompletableFuture - for one-time async results
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "result");

    // ✅ CountDownLatch - for coordination
    CountDownLatch latch = new CountDownLatch(3);

    // ✅ Exchanger - for data exchange between two threads
    Exchanger<String> exchanger = new Exchanger<>();
}
```

### Thread Communication Summary

```
┌─────────────────────────┬──────────────────────────┐
│  Mechanism              │  Use Case                │
├─────────────────────────┼──────────────────────────┤
│  wait() / notify()      │  Legacy code, simple     │
│  Condition.await()      │  Multiple conditions     │
│  BlockingQueue          │  Producer-Consumer       │
│  CountDownLatch         │  Wait for N events       │
│  CyclicBarrier          │  Sync at meeting point   │
│  Semaphore              │  Resource pooling        │
│  Exchanger              │  Pair-wise data exchange │
│  CompletableFuture      │  Async result chaining   │
│  Phaser                 │  Dynamic barrier         │
└─────────────────────────┴──────────────────────────┘
```

---

## Anti-Patterns

❌ **Using `if` instead of `while` with `wait()`:**
```java
if (!condition) wait(); // ❌ Spurious wakeup risk
```

❌ **Calling wait/notify outside synchronized:**
```java
obj.wait(); // ❌ IllegalMonitorStateException
```

❌ **Using `Thread.sleep()` for coordination:**
```java
while (!ready) { Thread.sleep(100); } // ❌ Busy-waiting, use wait/notify
```

❌ **Using wait/notify when BlockingQueue would suffice**

## Best Practices

✅ Always call `wait()` in a `while` loop  
✅ Always call `wait()`/`notify()` inside `synchronized` on the same object  
✅ Prefer `notifyAll()` over `notify()` unless you're certain only one waiter exists  
✅ Use `BlockingQueue` for producer-consumer patterns  
✅ Use `Condition` variables for multiple wait conditions  
✅ Prefer higher-level concurrency utilities (java.util.concurrent)

---

[← Deadlock](./q13-deadlock.md) | [Next: CompletableFuture →](./q15-completable-future.md)

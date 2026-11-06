# Q1: What is a Thread and Why Do We Need Multithreading?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question

What is a thread? How does it differ from a process? Why do we need multithreading in Java, and how has modern Java (Java 17/21+) evolved thread models via Virtual Threads and Structured Concurrency?

---

## Answer

### Thread vs Process: Deep Architectural Comparison

```
┌────────────────────────────────────────────────────────────────────────┐
│                              PROCESS                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │ Platform     │  │ Platform     │  │ Virtual      │                  │
│  │ Thread 1     │  │ Thread 2     │  │ Thread N     │                  │
│  │              │  │              │  │ (ForkJoinPool│                  │
│  │ Call Stack   │  │ Call Stack   │  │ carrier)     │                  │
│  │ Registers    │  │ Registers    │  │ Stack Chunks │                  │
│  └──────────────┘  └──────────────┘  └──────────────┘                  │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                         SHARED HEAP & MEMORY                   │    │
│  │  Object Instances  │  Static Variables  │  Code Cache  │ Files   │    │
│  └────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────┘
```

| Feature | Process | Platform (OS) Thread | Virtual Thread (Java 21+) |
|---------|---------|----------------------|---------------------------|
| **Address Space** | Isolated virtual memory mapping | Shares process heap, private native stack (~1MB) | Shares process heap, stack chunks on heap (few KB) |
| **Creation Cost** | Heavy (OS fork/clone, page table duplication) | Expensive (OS kernel allocations, native thread control block) | Extremely lightweight (Java object allocation) |
| **Scheduling** | OS Kernel scheduler | OS Kernel scheduler (preemptive time-slicing) | JVM User-space scheduler (`ForkJoinPool`) |
| **Context Switch** | Very expensive (TLB flush, CR3 register reload, ring transition) | Expensive (OS kernel trap, register save/restore) | Extremely cheap (JVM-level user-space scheduling, no kernel trap) |
| **Scalability** | Tens to hundreds per machine | Thousands (limited by OS thread limits & memory) | Millions (limited only by available heap memory) |

---

### The Java Memory Model (JMM) & Thread Initialization

Understanding threads requires understanding how the Java Memory Model guarantees memory visibility across thread boundaries.

#### Happens-Before Relationship on Thread Start (`Thread.start()`)
When thread A spawns thread B via `threadB.start()`, the Java Language Specification (JLS §17.4.5) establishes a **happens-before** edge:
- **Prior actions**: Every action performed by thread A prior to invoking `threadB.start()` is **fully visible** to thread B when its `run()` method begins.
- **Thread termination (`Thread.join()`)**: If thread A calls `threadB.join()`, all actions performed by thread B prior to its termination are guaranteed to be visible to thread A upon successful return from `join()`.

```java
public class ThreadHappensBeforeDemo {
    private static int data = 0;
    private static boolean ready = false;

    public static void main(String[] args) throws InterruptedException {
        data = 42;
        ready = true;

        Thread worker = new Thread(() -> {
            // Because of thread.start() happens-before guarantee:
            // worker thread is guaranteed to see ready == true and data == 42
            if (ready) {
                System.out.println("Data: " + data); // Prints 42 reliably
            }
        });

        worker.start();
        worker.join(); // Guarantees worker's side effects are visible back to main
    }
}
```

---

### Modern Java 21+ Evolution: Virtual Threads

Historically, Java threads were 1:1 wrappers around OS kernel threads (`Platform Threads`). This created the **"Thread-per-request" wall**:
- OS threads consume ~1MB of native stack memory.
- Spawning 100,000 platform threads requires ~100GB of RAM just for stacks.
- Kernel context switching overhead limits throughput during high I/O concurrency.

Java 21 introduced **Virtual Threads** (JEP 444), establishing an **M:N scheduling model**:
- Millions of virtual threads (M) are multiplexed across a small pool of OS carrier threads (N), typically equal to the number of CPU cores.
- When a virtual thread performs blocking I/O (e.g., socket read, file channel, `LockSupport.park()`), the JVM unmounts the virtual thread from the carrier thread, freeing the carrier to execute another virtual thread.
- When the I/O completes, the scheduler remounts the virtual thread.

```java
// Java 21+ Scalable I/O Server using Virtual Threads
public class VirtualThreadServer {
    public static void main(String[] args) throws Exception {
        try (var serverSocket = new java.net.ServerSocket(8080)) {
            System.out.println("Virtual thread server listening on port 8080...");
            while (true) {
                var socket = serverSocket.accept();
                // Spawns a lightweight virtual thread per incoming connection
                Thread.ofVirtual().start(() -> handleClient(socket));
            }
        }
    }

    private static void handleClient(Socket socket) {
        try (socket;
             var reader = new java.io.BufferedReader(new java.io.InputStreamReader(socket.getInputStream()));
             var writer = new java.io.PrintWriter(socket.getOutputStream(), true)) {
            String request = reader.readLine();
            writer.println("HTTP/1.1 200 OK\r\n\r\nHello from Virtual Thread: " + Thread.currentThread());
        } catch (Exception e) {
            System.err.println("Client handling failed: " + e.getMessage());
        }
    }
}
```

---

### Context Switch Cost & CPU Cache Invalidation

When threads context-switch at the CPU level:
1. **CPU Registers**: Program counter, stack pointer, and general-purpose registers must be saved to memory and restored.
2. **L1/L2/L3 Cache Pollution**: Cache lines populated by Thread A's working set are evicted, causing cold cache misses when Thread B runs.
3. **TLB Flushes**: Virtual memory translation caches may incur invalidation penalties.
4. **Virtual Thread Advantage**: Virtual thread context switches occur entirely within JVM user-space memory (swapping stack chunk references on the heap), avoiding kernel traps, ring transitions, and TLB invalidations.

---

### User Threads vs Daemon Threads

| Feature | Platform (User) Thread | Daemon Thread |
|---------|-------------------------|---------------|
| **JVM Exit Condition** | JVM waits for all user threads to terminate | JVM exits immediately when all user threads terminate, killing daemons |
| **Default State** | `isDaemon() == false` | Inherits parent thread state |
| **Primary Use Case** | Business logic, request processing | Background maintenance (Garbage Collection, connection pool heartbeats, metrics flushing) |
| **Configuration** | Default | Must invoke `setDaemon(true)` **before** `start()` |

---

## ⚠️ Gotchas & Common Pitfalls

1. **Virtual Thread Pinning**: If a virtual thread executes code inside a `synchronized` block/method or performs native JNI calls, the JVM cannot unmount it from the carrier thread. The carrier thread becomes **pinned**, degrading scalability during I/O blockages. *(Solution: Migrate legacy `synchronized` blocks to `ReentrantLock` in code expected to run on virtual threads).*
2. **Platform Thread Exhaustion**: Spawning raw platform threads (`new Thread(...)`) for bursty workloads will trigger `OutOfMemoryError: native thread creation failed` due to native OS process limits and 1MB stack allocations.
3. **Swallowing InterruptedException**: Catching `InterruptedException` without restoring the interruption flag (`Thread.currentThread().interrupt()`) breaks task cancellation protocols.
4. **Assuming Thread Priority Guarantees**: Setting `Thread.MAX_PRIORITY` is merely a hint to the OS scheduler and provides zero guarantees of execution ordering.
5. **Memory Visibility Bugs without Synchronization**: Assuming CPU cores instantly see shared variable updates without a happens-before edge (volatile, synchronized, or atomic guarantees) leads to silent race conditions.

---

## Interviewer Expectations & Deep Mental Models

- **Explain M:N Scheduling**: Articulate how Java 21 virtual threads use a ForkJoinPool scheduler to map millions of virtual tasks onto core-bound carrier threads.
- **JMM Happens-Before**: Be ready to cite exact JLS rules for `Thread.start()` and `Thread.join()` visibility guarantees.
- **Virtual Thread Gotchas**: Know precisely why `synchronized` blocks pin virtual threads and how ReentrantLocks resolve this.

---

[Next: Thread Lifecycle →](./02-thread-lifecycle.md)

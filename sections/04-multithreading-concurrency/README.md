# Multithreading & Concurrency

[Home](../../README.md) > Multithreading & Concurrency

---

## Overview
Concurrency is arguably the most challenging topic in Java interviews. This section covers everything from basic thread creation to modern virtual threads.

## Prerequisites
- [Java Fundamentals](../01-java-fundamentals/README.md)
- [OOP Concepts](../02-oop-concepts/README.md)
- [Collections Framework](../03-collections-framework/README.md)

## Question Index
| # | Question | Difficulty | File |
|---|----------|------------|------|
| 1 | What is a thread and why do we need multithreading? | 🟢 | [01-thread-basics.md](./01-thread-basics.md) |
| 2 | Explain the thread lifecycle and states | 🟢 | [02-thread-lifecycle.md](./02-thread-lifecycle.md) |
| 3 | What are the different ways to create threads? | 🟢 | [03-creating-threads.md](./03-creating-threads.md) |
| 4 | Runnable vs Callable - what's the difference? | 🟡 | [04-runnable-vs-callable.md](./04-runnable-vs-callable.md) |
| 5 | How do thread pools work? | 🟡 | [05-thread-pools.md](./05-thread-pools.md) |
| 6 | Explain the Executor framework | 🟡 | [06-executor-framework.md](./06-executor-framework.md) |
| 7 | How does the synchronized keyword work? | 🟡 | [07-synchronized-keyword.md](./07-synchronized-keyword.md) |
| 8 | What is the volatile keyword? | 🟡 | [08-volatile-keyword.md](./08-volatile-keyword.md) |
| 9 | How do Atomic classes work? | 🟠 | [09-atomic-classes.md](./09-atomic-classes.md) |
| 10 | Explain ReentrantLock and its advantages | 🟠 | [10-reentrant-lock.md](./10-reentrant-lock.md) |
| 11 | How does ReadWriteLock work? | 🟠 | [11-readwrite-lock.md](./11-readwrite-lock.md) |
| 12 | What are Condition variables? | 🟠 | [12-condition-variables.md](./12-condition-variables.md) |
| 13 | What is deadlock and how to prevent it? | 🟠 | [13-deadlock.md](./13-deadlock.md) |
| 14 | How does thread communication work (wait/notify)? | 🟡 | [14-thread-communication.md](./14-thread-communication.md) |
| 15 | Explain CompletableFuture in depth | 🟠 | [15-completable-future.md](./15-completable-future.md) |
| 16 | When should you use parallel streams? | 🟠 | [16-parallel-streams.md](./16-parallel-streams.md) |
| 17 | What concurrent collections are available? | 🟡 | [17-concurrent-collections.md](./17-concurrent-collections.md) |
| 18 | How does the Fork/Join framework work? | 🟠 | [18-fork-join-framework.md](./18-fork-join-framework.md) |
| 19 | What is ThreadLocal and when to use it? | 🟡 | [19-thread-local.md](./19-thread-local.md) |
| 20 | What are common thread safety patterns? | 🟠 | [20-thread-safety-patterns.md](./20-thread-safety-patterns.md) |
| 21 | Explain Semaphore and CountDownLatch | 🟡 | [21-semaphore-countdownlatch.md](./21-semaphore-countdownlatch.md) |
| 22 | CyclicBarrier vs Phaser | 🟠 | [22-cyclic-barrier-phaser.md](./22-cyclic-barrier-phaser.md) |
| 23 | What are Virtual Threads (Project Loom)? | � | [23-virtual-threads.md](./23-virtual-threads.md) |

## Key Takeaways
- Thread safety requires visibility (volatile/synchronized) AND atomicity
- Prefer higher-level concurrency utilities over low-level wait/notify
- Virtual threads (Java 21) revolutionize high-throughput I/O-bound applications
- Deadlock prevention: consistent lock ordering, timeout-based acquisition
- CompletableFuture is the modern approach for asynchronous programming

## Related Sections
- [Collections Framework](../03-collections-framework/README.md)
- [JVM Internals](../07-jvm-internals/README.md)
- [Performance Optimization](../12-performance-optimization/README.md)

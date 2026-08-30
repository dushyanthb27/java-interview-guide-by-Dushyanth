# Multithreading & Concurrency

[Home](../../README.md) > Multithreading & Concurrency

---

## Overview
Concurrency is arguably the most challenging topic in Java interviews. This section covers everything from basic thread creation to modern virtual threads and structured concurrency.

## Prerequisites
- [Java Fundamentals](../01-java-fundamentals/README.md)
- [OOP Concepts](../02-oop-concepts/README.md)
- [Collections Framework](../03-collections-framework/README.md)

## Question Index
| # | Question | Difficulty | File |
|---|----------|------------|------|
| 1 | What is a thread and why do we need multithreading? | 🟢 | [q01](./q01-thread-basics.md) |
| 2 | Explain the thread lifecycle and states | 🟢 | [q02](./q02-thread-lifecycle.md) |
| 3 | What are the different ways to create threads? | 🟢 | [q03](./q03-creating-threads.md) |
| 4 | Runnable vs Callable - what's the difference? | 🟡 | [q04](./q04-runnable-vs-callable.md) |
| 5 | How do thread pools work? | 🟡 | [q05](./q05-thread-pools.md) |
| 6 | Explain the Executor framework | 🟡 | [q06](./q06-executor-framework.md) |
| 7 | How does the synchronized keyword work? | 🟡 | [q07](./q07-synchronized-keyword.md) |
| 8 | What is the volatile keyword? | 🟡 | [q08](./q08-volatile-keyword.md) |
| 9 | How do Atomic classes work? | 🟠 | [q09](./q09-atomic-classes.md) |
| 10 | Explain ReentrantLock and its advantages | 🟠 | [q10](./q10-reentrant-lock.md) |
| 11 | How does ReadWriteLock work? | 🟠 | [q11](./q11-readwrite-lock.md) |
| 12 | What are Condition variables? | 🟠 | [q12](./q12-condition-variables.md) |
| 13 | What is deadlock and how to prevent it? | 🟠 | [q13](./q13-deadlock.md) |
| 14 | How does thread communication work (wait/notify)? | 🟡 | [q14](./q14-thread-communication.md) |
| 15 | Explain CompletableFuture in depth | 🟠 | [q15](./q15-completable-future.md) |
| 16 | When should you use parallel streams? | 🟠 | [q16](./q16-parallel-streams.md) |
| 17 | What concurrent collections are available? | 🟡 | [q17](./q17-concurrent-collections.md) |
| 18 | How does the Fork/Join framework work? | 🟠 | [q18](./q18-fork-join-framework.md) |
| 19 | What is ThreadLocal and when to use it? | 🟡 | [q19](./q19-thread-local.md) |
| 20 | What are common thread safety patterns? | 🟠 | [q20](./q20-thread-safety-patterns.md) |
| 21 | Explain Semaphore and CountDownLatch | 🟡 | [q21](./q21-semaphore-countdownlatch.md) |
| 22 | CyclicBarrier vs Phaser | 🟠 | [q22](./q22-cyclic-barrier-phaser.md) |
| 23 | What are Virtual Threads (Project Loom)? | 🔴 | [q23](./q23-virtual-threads.md) |
| 24 | What is Structured Concurrency? | 🔴 | [q24](./q24-structured-concurrency.md) |
| 25 | Introduction to Reactive Programming in Java | 🔴 | [q25](./q25-reactive-basics.md) |

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

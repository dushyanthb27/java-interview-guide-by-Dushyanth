# Q22: CyclicBarrier vs Phaser

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
How do `CyclicBarrier` and `Phaser` work? How do they compare with `CountDownLatch`?

---

## Answer

### CyclicBarrier

A `CyclicBarrier` allows a set of threads to wait for each other at a common barrier point. Unlike `CountDownLatch`, it is **reusable** (cyclic).

```
Phase 1:                          Phase 2 (barrier resets):
Thread A ──────► BARRIER          Thread A ──────► BARRIER
Thread B ──────►   |              Thread B ──────►   |
Thread C ──────►   |              Thread C ──────►   |
                   ▼ (all arrive)                    ▼ (all arrive again)
              Continue!                         Continue!
```

```java
public class CyclicBarrierDemo {
    public static void main(String[] args) {
        int parties = 3;

        // Optional barrier action runs when all threads arrive
        CyclicBarrier barrier = new CyclicBarrier(parties, () -> {
            System.out.println("=== All threads reached barrier! Merging results... ===");
        });

        for (int i = 0; i < parties; i++) {
            final int threadId = i;
            new Thread(() -> {
                try {
                    // Phase 1
                    System.out.println("Thread " + threadId + ": Phase 1 work");
                    Thread.sleep(ThreadLocalRandom.current().nextInt(500, 1500));

                    System.out.println("Thread " + threadId + ": Waiting at barrier (Phase 1)");
                    barrier.await(); // Wait for all threads

                    // Phase 2 (barrier automatically resets!)
                    System.out.println("Thread " + threadId + ": Phase 2 work");
                    Thread.sleep(ThreadLocalRandom.current().nextInt(500, 1500));

                    System.out.println("Thread " + threadId + ": Waiting at barrier (Phase 2)");
                    barrier.await(); // Wait again (reusable!)

                    System.out.println("Thread " + threadId + ": All phases complete!");
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }, "Worker-" + i).start();
        }
    }
}
```

### CyclicBarrier: Parallel Matrix Computation

```java
public class ParallelMatrixComputation {
    private final double[][] matrix;
    private final int rowsPerWorker;
    private final CyclicBarrier barrier;
    private final int iterations;

    public ParallelMatrixComputation(int rows, int cols, int workers, int iterations) {
        this.matrix = new double[rows][cols];
        this.rowsPerWorker = rows / workers;
        this.iterations = iterations;

        this.barrier = new CyclicBarrier(workers, () -> {
            // Runs after each iteration when all workers are done
            System.out.println("Iteration complete. Checking convergence...");
        });
    }

    public void start(int workers) {
        for (int w = 0; w < workers; w++) {
            final int startRow = w * rowsPerWorker;
            final int endRow = (w == workers - 1) ? matrix.length : startRow + rowsPerWorker;

            new Thread(() -> {
                try {
                    for (int iter = 0; iter < iterations; iter++) {
                        // Compute my portion
                        computeRows(startRow, endRow);

                        // Wait for all workers to finish this iteration
                        barrier.await();
                        // All workers see each other's results before next iteration
                    }
                } catch (InterruptedException | BrokenBarrierException e) {
                    Thread.currentThread().interrupt();
                }
            }, "Worker-" + w).start();
        }
    }

    private void computeRows(int startRow, int endRow) {
        for (int r = startRow; r < endRow; r++) {
            for (int c = 0; c < matrix[r].length; c++) {
                matrix[r][c] = Math.random(); // Placeholder computation
            }
        }
    }
}
```

### BrokenBarrierException

```java
public class BrokenBarrierDemo {
    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(3);

        Thread t1 = new Thread(() -> {
            try {
                barrier.await();
            } catch (BrokenBarrierException e) {
                // Another thread was interrupted or timed out
                System.out.println("Barrier is broken!");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread t2 = new Thread(() -> {
            try {
                barrier.await(1, TimeUnit.SECONDS); // Timeout
            } catch (TimeoutException e) {
                System.out.println("Timed out waiting at barrier");
                // This BREAKS the barrier for all other waiting threads!
            } catch (InterruptedException | BrokenBarrierException e) {
                Thread.currentThread().interrupt();
            }
        });

        t1.start();
        t2.start();

        // Only 2 of 3 threads → t2 times out → barrier breaks → t1 gets BrokenBarrierException

        // Check and reset
        // barrier.isBroken() → true
        // barrier.reset()    → resets barrier (waiting threads get BrokenBarrierException)
    }
}
```

---

### Phaser

`Phaser` is the most flexible barrier, supporting **dynamic registration** and **multiple phases**.

```
Phase 0:                    Phase 1:                    Phase 2:
Thread A ──► arrive         Thread A ──► arrive         Thread A ──► arrive
Thread B ──► arrive         Thread B ──► arrive         Thread C ──► arrive (new!)
                            Thread C registered!        Thread B deregistered
All arrived → advance       All arrived → advance       All arrived → advance
```

```java
public class PhaserDemo {
    public static void main(String[] args) {
        // Register main thread + 3 worker threads
        Phaser phaser = new Phaser(1); // Register self (main thread)

        for (int i = 0; i < 3; i++) {
            phaser.register(); // Dynamically register each worker
            final int workerId = i;

            new Thread(() -> {
                try {
                    // Phase 0
                    System.out.println("Worker " + workerId + ": Phase 0");
                    Thread.sleep(ThreadLocalRandom.current().nextInt(500, 1500));
                    phaser.arriveAndAwaitAdvance(); // Wait for all

                    // Phase 1
                    System.out.println("Worker " + workerId + ": Phase 1");
                    Thread.sleep(ThreadLocalRandom.current().nextInt(500, 1500));
                    phaser.arriveAndAwaitAdvance(); // Wait for all

                    // Phase 2
                    System.out.println("Worker " + workerId + ": Phase 2");
                    Thread.sleep(ThreadLocalRandom.current().nextInt(500, 1500));
                    phaser.arriveAndDeregister(); // Done, deregister

                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }, "Worker-" + workerId).start();
        }

        // Main thread coordinates phases
        phaser.arriveAndAwaitAdvance(); // Wait for Phase 0
        System.out.println("=== Phase 0 complete ===");

        phaser.arriveAndAwaitAdvance(); // Wait for Phase 1
        System.out.println("=== Phase 1 complete ===");

        phaser.arriveAndDeregister();   // Main thread done
        System.out.println("=== Main deregistered ===");
    }
}
```

### Phaser: Controlling Termination

```java
public class PhaserTermination {
    public static void main(String[] args) {
        // Override onAdvance to control when the phaser terminates
        Phaser phaser = new Phaser() {
            @Override
            protected boolean onAdvance(int phase, int registeredParties) {
                System.out.println("Phase " + phase + " complete, parties: " + registeredParties);
                // Return true to terminate, false to continue
                return phase >= 2 || registeredParties == 0;
                // Terminate after phase 2 or when no parties remain
            }
        };

        phaser.register(); // Register main thread

        for (int i = 0; i < 3; i++) {
            phaser.register();
            final int id = i;
            new Thread(() -> {
                while (!phaser.isTerminated()) {
                    System.out.println("Worker " + id + ": Phase " + phaser.getPhase());
                    phaser.arriveAndAwaitAdvance();
                }
                System.out.println("Worker " + id + ": Phaser terminated!");
            }).start();
        }

        // Main participates in all phases
        while (!phaser.isTerminated()) {
            phaser.arriveAndAwaitAdvance();
        }
        System.out.println("All done!");
    }
}
```

### Phaser: Dynamic Task Management

```java
public class DynamicTaskPhaser {
    public static void main(String[] args) throws InterruptedException {
        Phaser phaser = new Phaser(1); // Main thread registered

        // Phase 0: Start with 2 workers
        for (int i = 0; i < 2; i++) {
            spawnWorker(phaser, "Initial-" + i);
        }
        phaser.arriveAndAwaitAdvance(); // Complete Phase 0

        // Phase 1: Add 3 more workers dynamically
        for (int i = 0; i < 3; i++) {
            spawnWorker(phaser, "Dynamic-" + i);
        }
        phaser.arriveAndAwaitAdvance(); // Complete Phase 1 (with 5 workers)

        phaser.arriveAndDeregister();
        System.out.println("Main done. Registered parties: " + phaser.getRegisteredParties());
    }

    static void spawnWorker(Phaser phaser, String name) {
        phaser.register(); // Dynamic registration!
        new Thread(() -> {
            System.out.println(name + " working in phase " + phaser.getPhase());
            try { Thread.sleep(500); } catch (InterruptedException e) {}
            phaser.arriveAndDeregister();
        }, name).start();
    }
}
```

### Comparison: CountDownLatch vs CyclicBarrier vs Phaser

```
┌──────────────────┬──────────────────┬──────────────────┬──────────────────┐
│ Feature          │ CountDownLatch   │ CyclicBarrier    │ Phaser           │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Reusable         │ ❌ One-time     │ ✅ Auto-reset    │ ✅ Auto-advance  │
│ Dynamic parties  │ ❌ Fixed        │ ❌ Fixed         │ ✅ register/     │
│                  │                  │                  │    deregister    │
│ Barrier action   │ ❌              │ ✅ Optional      │ ✅ onAdvance()   │
│ Phase tracking   │ ❌              │ ❌               │ ✅ getPhase()    │
│ Termination      │ N/A             │ Manual           │ ✅ onAdvance()   │
│ Count semantics  │ Event count     │ Thread count     │ Party count      │
│ Who waits        │ await() callers │ All parties      │ All parties      │
│ Who counts       │ countDown()     │ await()          │ arrive()         │
│ Thread can count │ ✅ countDown()  │ ❌ (must await)  │ ✅ arrive()      │
│  without waiting │                  │                  │                  │
│ Complexity       │ Simple          │ Moderate         │ Complex          │
│ Since            │ JDK 1.5         │ JDK 1.5          │ JDK 1.7          │
└──────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

### When to Use What

```
CountDownLatch: "Wait for N events to happen"
  → Service startup, test coordination, one-time gates

CyclicBarrier: "N threads must meet, then continue together (repeatedly)"
  → Iterative parallel algorithms, gaming synchronization

Phaser: "Flexible barrier with dynamic participants and phases"
  → Complex multi-phase workflows, fork-join-style coordination
```

---

## Anti-Patterns

❌ **Using CyclicBarrier when CountDownLatch suffices (simpler is better)**  
❌ **Not handling BrokenBarrierException in CyclicBarrier**  
❌ **Forgetting to deregister from Phaser (causes other parties to wait forever)**  
❌ **Using Phaser for simple scenarios where CountDownLatch works**

## Best Practices

✅ Use `CountDownLatch` for simple one-shot coordination  
✅ Use `CyclicBarrier` for iterative parallel algorithms  
✅ Use `Phaser` when parties need to dynamically join/leave  
✅ Always use `await(timeout)` variants to prevent indefinite blocking  
✅ Handle `BrokenBarrierException` gracefully  
✅ Call `arriveAndDeregister()` when a party is done permanently

---

[← Semaphore & CountDownLatch](./q21-semaphore-countdownlatch.md) | [Next: Virtual Threads →](./q23-virtual-threads.md)

# Q18: How Does the Fork/Join Framework Work?

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
How does the Fork/Join framework work? What is work-stealing? When should you use it?

---

## Answer

### Fork/Join Concept

The Fork/Join framework implements **divide and conquer** parallelism:

```
                    [Problem: Sort 1M items]
                           │
                    fork() │
                    ┌──────┴──────┐
                    │             │
            [Sort 500K]    [Sort 500K]
                │                │
         fork() │          fork()│
         ┌──────┴───┐    ┌──────┴───┐
         │          │    │          │
   [Sort 250K] [Sort 250K] [Sort 250K] [Sort 250K]
         │          │    │          │
         ... keep splitting until threshold ...
         │          │    │          │
         ▼          ▼    ▼          ▼
      [Sequential sort each small piece]
         │          │    │          │
         └────┬─────┘    └────┬─────┘
              │               │
         join() merge    join() merge
              │               │
              └───────┬───────┘
                      │
                 join() merge
                      │
               [Sorted 1M items]
```

### Work-Stealing Algorithm

```
Traditional thread pool:          Fork/Join work-stealing:
┌─────────┐                      ┌─────────┐
│ Thread 1│ ██████████           │ Thread 1│ ██████               
│ Thread 2│ ██                   │ Thread 2│ ██████  ◄─ steals    
│ Thread 3│ █████████            │ Thread 3│ ██████  ◄─ from T1   
│ Thread 4│ █                    │ Thread 4│ ██████  ◄─ steals    
└─────────┘                      └─────────┘
❌ Unbalanced workload            ✅ Balanced via stealing

Each thread has its own deque (double-ended queue):
- Owner pushes/pops from the TOP (LIFO)
- Idle threads STEAL from the BOTTOM (FIFO) of other threads
```

### RecursiveTask (Returns a Value)

```java
public class ParallelSum extends RecursiveTask<Long> {
    private static final int THRESHOLD = 10_000;
    private final long[] array;
    private final int start, end;

    public ParallelSum(long[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;

        // Base case: compute sequentially
        if (length <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            return sum;
        }

        // Recursive case: split and conquer
        int mid = start + length / 2;

        ParallelSum leftTask = new ParallelSum(array, start, mid);
        ParallelSum rightTask = new ParallelSum(array, mid, end);

        leftTask.fork();  // Submit left to thread pool (async)
        long rightResult = rightTask.compute(); // Compute right in current thread
        long leftResult = leftTask.join(); // Wait for left result

        return leftResult + rightResult;
    }

    public static void main(String[] args) {
        long[] array = LongStream.rangeClosed(1, 10_000_000).toArray();

        ForkJoinPool pool = new ForkJoinPool();
        ParallelSum task = new ParallelSum(array, 0, array.length);

        long result = pool.invoke(task);
        System.out.println("Sum: " + result);

        pool.shutdown();
    }
}
```

### RecursiveAction (No Return Value)

```java
public class ParallelMergeSort extends RecursiveAction {
    private static final int THRESHOLD = 1000;
    private final int[] array;
    private final int start, end;

    public ParallelMergeSort(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected void compute() {
        if (end - start <= THRESHOLD) {
            // Base case: use Arrays.sort for small segments
            Arrays.sort(array, start, end);
            return;
        }

        int mid = (start + end) / 2;

        // Fork two subtasks
        ParallelMergeSort left = new ParallelMergeSort(array, start, mid);
        ParallelMergeSort right = new ParallelMergeSort(array, mid, end);

        // Execute both in parallel
        invokeAll(left, right);

        // Merge results
        merge(array, start, mid, end);
    }

    private void merge(int[] arr, int start, int mid, int end) {
        int[] temp = new int[end - start];
        int i = start, j = mid, k = 0;

        while (i < mid && j < end) {
            if (arr[i] <= arr[j]) {
                temp[k++] = arr[i++];
            } else {
                temp[k++] = arr[j++];
            }
        }
        while (i < mid) temp[k++] = arr[i++];
        while (j < end) temp[k++] = arr[j++];

        System.arraycopy(temp, 0, arr, start, temp.length);
    }

    public static void main(String[] args) {
        int[] array = ThreadLocalRandom.current().ints(1_000_000).toArray();

        ForkJoinPool pool = ForkJoinPool.commonPool();
        pool.invoke(new ParallelMergeSort(array, 0, array.length));

        System.out.println("Sorted: " + isSorted(array));
    }

    static boolean isSorted(int[] arr) {
        for (int i = 1; i < arr.length; i++) {
            if (arr[i] < arr[i - 1]) return false;
        }
        return true;
    }
}
```

### ForkJoinPool Configuration

```java
public class ForkJoinPoolDemo {
    public static void main(String[] args) {
        // Common pool (shared across JVM)
        ForkJoinPool common = ForkJoinPool.commonPool();
        System.out.println("Common pool parallelism: " + common.getParallelism());

        // Custom pool
        ForkJoinPool custom = new ForkJoinPool(
            8,                          // parallelism
            ForkJoinPool.defaultForkJoinWorkerThreadFactory,
            null,                       // UncaughtExceptionHandler
            true                        // asyncMode (FIFO instead of LIFO)
        );

        // Pool statistics
        System.out.println("Active threads: " + custom.getActiveThreadCount());
        System.out.println("Queued tasks: " + custom.getQueuedTaskCount());
        System.out.println("Steal count: " + custom.getStealCount());
        System.out.println("Pool size: " + custom.getPoolSize());

        custom.shutdown();
    }
}
```

### Choosing the Right Threshold

```java
/*
 * Threshold selection guidelines:
 *
 * Too small threshold:
 *   - Too many tasks created → overhead dominates
 *   - Task creation and scheduling cost exceeds computation
 *
 * Too large threshold:
 *   - Not enough parallelism → cores sit idle
 *   - Unbalanced work distribution
 *
 * Rules of thumb:
 *   1. Sequential computation per task: 100-10,000 elements
 *   2. Total tasks ≈ 10 * number of processors
 *   3. Benchmark with different thresholds!
 *
 * Example benchmarking:
 */
public class ThresholdBenchmark {
    public static void benchmark(long[] array) {
        int[] thresholds = {100, 1000, 5000, 10000, 50000, 100000};
        ForkJoinPool pool = new ForkJoinPool();

        for (int threshold : thresholds) {
            long start = System.nanoTime();
            pool.invoke(new SumTask(array, 0, array.length, threshold));
            long elapsed = System.nanoTime() - start;
            System.out.printf("Threshold %6d: %d ms%n", threshold, elapsed / 1_000_000);
        }

        pool.shutdown();
    }

    static class SumTask extends RecursiveTask<Long> {
        private final long[] array;
        private final int start, end, threshold;

        SumTask(long[] array, int start, int end, int threshold) {
            this.array = array; this.start = start;
            this.end = end; this.threshold = threshold;
        }

        @Override
        protected Long compute() {
            if (end - start <= threshold) {
                long sum = 0;
                for (int i = start; i < end; i++) sum += array[i];
                return sum;
            }
            int mid = (start + end) / 2;
            SumTask left = new SumTask(array, start, mid, threshold);
            SumTask right = new SumTask(array, mid, end, threshold);
            left.fork();
            return right.compute() + left.join();
        }
    }
}
```

### Best Practice: fork() + compute() + join()

```java
// ❌ WRONG: fork both, join both
leftTask.fork();
rightTask.fork();    // Wastes the current thread!
left = leftTask.join();
right = rightTask.join();

// ✅ CORRECT: fork one, compute other in current thread
leftTask.fork();              // Submit to pool
right = rightTask.compute();  // Use current thread
left = leftTask.join();       // Get forked result

// ✅ ALSO CORRECT: invokeAll (for multiple tasks)
invokeAll(task1, task2, task3); // Framework optimizes execution
```

---

## Anti-Patterns

❌ **Forking both subtasks (wasting current thread):**
```java
left.fork();
right.fork();  // ❌ Current thread does nothing but wait
```

❌ **Using Fork/Join for I/O-bound tasks:**
```java
// ❌ Blocks ForkJoinPool threads → deadlock risk
protected String compute() {
    return callExternalAPI(); // I/O blocks the worker thread
}
```

❌ **Too small threshold (excessive task creation overhead)**

❌ **Calling join() before fork():**
```java
left.join();  // ❌ Task hasn't been forked yet!
left.fork();
```

## Best Practices

✅ Use Fork/Join for CPU-bound, recursive divide-and-conquer problems  
✅ Fork one subtask, compute the other in the current thread  
✅ Choose thresholds by benchmarking (typically 1K-10K elements)  
✅ Use `invokeAll()` when forking multiple tasks  
✅ Consider parallel streams first (they use Fork/Join internally)  
✅ Keep tasks independent (no shared mutable state between subtasks)

---

[← Concurrent Collections](./q17-concurrent-collections.md) | [Next: ThreadLocal →](./q19-thread-local.md)

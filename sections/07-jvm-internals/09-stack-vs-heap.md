# Q9: Stack vs Heap in JVM

## 📋 Question
Explain the difference between Stack and Heap memory in the JVM. What is stored where, and what errors are associated with each?

## Difficulty: 🟡 Medium

---

## 💡 Answer

The JVM divides memory into **Stack** (per-thread, method execution data) and **Heap** (shared, object storage). Understanding the distinction is critical for debugging memory issues and writing efficient Java code.

### Stack vs Heap — Overview

```
┌═══════════════════════════════════════════════════════════════════┐
║                         JVM MEMORY                                ║
║                                                                   ║
║  ┌─────────────────────────┐    ┌──────────────────────────────┐  ║
║  │     STACK (per thread)  │    │          HEAP (shared)        │  ║
║  │                         │    │                              │  ║
║  │  Thread 1  Thread 2     │    │   ┌────────────────────┐     │  ║
║  │  ┌──────┐  ┌──────┐    │    │   │  Young Generation  │     │  ║
║  │  │Frame3│  │Frame2│    │    │   │  ┌──────┐┌──┐┌──┐  │     │  ║
║  │  ├──────┤  ├──────┤    │    │   │  │ Eden ││S0││S1│  │     │  ║
║  │  │Frame2│  │Frame1│    │    │   │  └──────┘└──┘└──┘  │     │  ║
║  │  ├──────┤  └──────┘    │    │   └────────────────────┘     │  ║
║  │  │Frame1│              │    │   ┌────────────────────┐     │  ║
║  │  └──────┘              │    │   │  Old Generation    │     │  ║
║  │                         │    │   │  (long-lived objs) │     │  ║
║  │  ✅ Thread-private      │    │   └────────────────────┘     │  ║
║  │  ✅ LIFO (fast alloc)   │    │                              │  ║
║  │  ✅ Auto-cleanup        │    │   ✅ Shared across threads   │  ║
║  │  ❌ Fixed size           │    │   ✅ Dynamic sizing          │  ║
║  │  ❌ Only primitives &   │    │   ✅ Stores all objects       │  ║
║  │     references          │    │   ❌ GC overhead             │  ║
║  └─────────────────────────┘    └──────────────────────────────┘  ║
╚═══════════════════════════════════════════════════════════════════╝
```

### Detailed Comparison

| Feature | Stack | Heap |
|---------|-------|------|
| **Scope** | Thread-private | Shared across all threads |
| **Stores** | Local variables, method parameters, return addresses | Object instances, arrays |
| **Lifetime** | Method invocation (frame pushed/popped) | Until garbage collected |
| **Size** | Fixed (`-Xss`, default ~512KB-1MB) | Dynamic (`-Xms` to `-Xmx`) |
| **Speed** | Very fast (LIFO, pointer bump) | Slower (GC managed, allocation search) |
| **Order** | LIFO (Last In, First Out) | No specific order |
| **Thread Safety** | Inherently thread-safe (private) | Requires synchronization |
| **Error** | `StackOverflowError` | `OutOfMemoryError` |
| **Fragmentation** | None | Possible (depends on GC) |

### What Goes Where?

```java
public class StackVsHeapDemo {

    // Static field → Method Area (part of Heap conceptually)
    static String appName = "Demo";

    // Instance field → Heap (part of the object)
    private int id;

    public void process(String input) {
        // ┌──────────────────── STACK ──────────────────────┐
        // │  'input' (reference) → points to Heap            │
        // │  'x' (primitive int) → value directly on Stack   │
        // │  'name' (reference) → points to Heap             │
        // │  'list' (reference) → points to Heap             │
        // │  'arr' (reference) → points to Heap              │
        // └──────────────────────────────────────────────────┘

        int x = 42;                        // Primitive → STACK
        String name = "Alice";             // Reference on STACK, String on HEAP
        List<String> list = new ArrayList<>(); // Reference on STACK, ArrayList on HEAP
        int[] arr = new int[10];           // Reference on STACK, array on HEAP

        // ┌──────────────────── HEAP ───────────────────────┐
        // │  String "Alice" object                           │
        // │  ArrayList object                                │
        // │  int[10] array object                            │
        // │  String object pointed to by 'input'             │
        // └──────────────────────────────────────────────────┘
    }
}
```

### Stack Frames in Detail

```
When process("hello") is called:

JVM Stack (Thread-1):
┌──────────────────────────────────────┐
│  Frame: process()              ← TOP │
│  ┌──────────────────────────────┐    │
│  │ Local Variable Table         │    │
│  │  [0] this    → 0x1A2B (Heap)│    │
│  │  [1] input   → 0x3C4D (Heap)│    │
│  │  [2] x       = 42           │    │
│  │  [3] name    → 0x5E6F (Heap)│    │
│  │  [4] list    → 0x7890 (Heap)│    │
│  │  [5] arr     → 0xABCD (Heap)│    │
│  ├──────────────────────────────┤    │
│  │ Operand Stack                │    │
│  │  (used during computation)   │    │
│  ├──────────────────────────────┤    │
│  │ Frame Data                   │    │
│  │  (return address, etc.)      │    │
│  └──────────────────────────────┘    │
├──────────────────────────────────────┤
│  Frame: main()                       │
│  ┌──────────────────────────────┐    │
│  │ Local Variable Table         │    │
│  │  [0] args    → 0xEF01 (Heap)│    │
│  │  [1] demo    → 0x1A2B (Heap)│    │
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘

When process() returns:
  - Its frame is POPPED (all local vars gone)
  - Control returns to main() frame
  - Heap objects become eligible for GC if no references remain
```

### StackOverflowError

```java
// StackOverflowError occurs when the stack is full (too many frames)
// Most common cause: infinite or very deep recursion

public class StackOverflowDemo {

    // ❌ Infinite recursion — will overflow the stack
    public static int factorial(int n) {
        return n * factorial(n - 1);  // No base case!
    }
    // Each call adds a frame: factorial(5) → factorial(4) → ... → StackOverflowError

    // ✅ Fixed with base case
    public static int factorialFixed(int n) {
        if (n <= 1) return 1;          // Base case — stops recursion
        return n * factorialFixed(n - 1);
    }

    // ✅ Even better — iterative (no stack growth)
    public static int factorialIterative(int n) {
        int result = 1;
        for (int i = 2; i <= n; i++) {
            result *= i;
        }
        return result;
    }
}

// Default stack size: ~512KB to 1MB (platform dependent)
// Increase with: java -Xss2m MyApp   (2 MB stack per thread)
// ⚠️ More stack per thread = fewer threads possible (stack uses native memory)
```

### OutOfMemoryError (Heap)

```java
// OutOfMemoryError occurs when the heap cannot accommodate new objects

public class OOMDemo {

    // ❌ Creates objects until heap is exhausted
    public static void main(String[] args) {
        List<byte[]> leak = new ArrayList<>();
        while (true) {
            leak.add(new byte[1024 * 1024]);  // 1 MB each
            // Objects never become eligible for GC because 'leak' holds references
        }
        // java.lang.OutOfMemoryError: Java heap space
    }
}

// Heap sizing:
// java -Xms256m -Xmx2g MyApp
//       ↑ initial     ↑ maximum

// Debug OOM:
// java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/dump.hprof MyApp
```

### Common OOM Types

```
java.lang.OutOfMemoryError: Java heap space
  → Heap is full, GC can't reclaim enough
  → Fix: increase -Xmx, find memory leak

java.lang.OutOfMemoryError: Metaspace
  → Too many loaded classes (class metadata)
  → Fix: increase -XX:MaxMetaspaceSize, check for class loader leaks

java.lang.OutOfMemoryError: GC overhead limit exceeded
  → GC runs > 98% of time but recovers < 2% of heap
  → Fix: increase heap, fix memory leak

java.lang.OutOfMemoryError: unable to create new native thread
  → OS can't create more threads (each thread needs stack memory)
  → Fix: reduce -Xss (smaller stacks), increase OS thread limits

java.lang.StackOverflowError
  → Stack is full (NOT an OOM, but related)
  → Fix: fix recursion, increase -Xss
```

### Escape Analysis and Stack Allocation

The JIT compiler can allocate objects on the **stack** instead of the heap if escape analysis determines the object doesn't escape the method:

```java
public int sumCoordinates(int x1, int y1, int x2, int y2) {
    // Without escape analysis: Point allocated on HEAP
    // With escape analysis: Point may be allocated on STACK (or eliminated entirely)
    Point p1 = new Point(x1, y1);  // Does NOT escape this method
    Point p2 = new Point(x2, y2);  // Does NOT escape this method

    return p1.x + p1.y + p2.x + p2.y;
}

// After escape analysis + scalar replacement:
// The Point objects are replaced with their fields on the stack
// Effectively becomes:
public int sumCoordinates(int x1, int y1, int x2, int y2) {
    return x1 + y1 + x2 + y2;  // No heap allocation at all!
}

// Object ESCAPES if:
// 1. Returned from the method
// 2. Stored in a field (instance or static)
// 3. Passed to another method that stores it
// 4. Passed to a thread

// Enabled by default:
// -XX:+DoEscapeAnalysis
// -XX:+EliminateAllocations
```

### Memory Layout of a Java Object (on Heap)

```
┌─────────────────────────────────────────────────┐
│              JAVA OBJECT IN HEAP                 │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │  Object Header (12-16 bytes on 64-bit)   │    │
│  │  ┌──────────────────┐                    │    │
│  │  │  Mark Word (8B)  │ GC age, hash,      │    │
│  │  │                  │ lock state          │    │
│  │  ├──────────────────┤                    │    │
│  │  │  Class Ptr (4B)  │ Pointer to class    │    │
│  │  │  (compressed)    │ metadata            │    │
│  │  └──────────────────┘                    │    │
│  ├──────────────────────────────────────────┤    │
│  │  Instance Fields                         │    │
│  │  int x = 10        (4 bytes)             │    │
│  │  int y = 20        (4 bytes)             │    │
│  │  String name       (4 bytes, compressed)  │    │
│  ├──────────────────────────────────────────┤    │
│  │  Padding (alignment to 8 bytes)          │    │
│  └──────────────────────────────────────────┘    │
│                                                  │
│  Minimum object size: 16 bytes (header + padding)│
│  Even an empty new Object() takes 16 bytes!      │
└─────────────────────────────────────────────────┘
```

### ⚠️ Common Mistake
Thinking that primitives are always on the stack. **Primitives that are instance fields** are stored on the **heap** (as part of the object). Only **local primitive variables** within methods are on the stack. Similarly, thinking that `new` always allocates on the heap — with escape analysis, the JIT may optimize the allocation to the stack or eliminate it entirely.

### ⚡ Performance Tip
Each thread's stack consumes **native memory** (not heap). With 1000 threads at 1 MB each, that's 1 GB of native memory just for stacks. In thread-heavy applications, reduce stack size with `-Xss256k` (if you don't have deep call chains) or switch to virtual threads (Java 21) which use much smaller stacks that grow dynamically.

```bash
# Memory budget example:
# -Xmx4g (heap) + 1000 threads * 1MB (stacks) + Metaspace + other
# Total: 4 GB + 1 GB + 256 MB + ... ≈ 5.5 GB native memory needed

# Reduce stack size for many threads:
java -Xss256k -Xmx4g -jar myapp.jar
```

---

## 🎯 Interviewer Insight
This is a fundamental question that tests your understanding of JVM memory layout. Interviewers want to hear that the **stack is per-thread with primitives and references**, while the **heap is shared with objects**. Strong candidates explain stack frames, the difference between `StackOverflowError` and `OutOfMemoryError`, and mention escape analysis for stack allocation.

## ⚡ Follow-up Questions
1. Where are static variables stored?
2. What happens to the stack when a method returns?
3. Can an object be allocated on the stack?

## 📚 Related Topics
- [JVM Architecture](./q01-jvm-architecture.md)
- [GC Algorithms](./q05-gc-algorithms.md)
- [Metaspace](./q10-metaspace.md)
- [JVM Tuning Parameters](./q12-jvm-tuning-parameters.md)

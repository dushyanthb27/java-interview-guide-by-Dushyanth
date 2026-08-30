# Q1: How Does Stack vs Heap Memory Work in Java?

[← Back to Memory Management](./README.md)

---

## Overview

Java uses two primary memory regions — the **stack** and the **heap** — each serving a distinct purpose. Understanding their mechanics is fundamental to writing efficient, bug-free code.

---

## Stack Memory

The stack is a **per-thread**, LIFO (Last-In-First-Out) data structure that stores **method frames**.

### What lives on the stack?
- Local primitive variables (`int`, `boolean`, `double`, etc.)
- Local reference variables (the pointer, not the object)
- Method parameters
- Return addresses and partial results

### Stack frame lifecycle

```java
public class StackExample {

    public static void main(String[] args) {
        int result = add(3, 5);       // Frame 1: main
        System.out.println(result);
    }

    static int add(int a, int b) {    // Frame 2: add (pushed on top of main)
        int sum = a + b;              // 'a', 'b', 'sum' live on the stack
        return sum;                   // Frame 2 popped when method returns
    }
}
```

**Stack frame structure:**

```
┌──────────────────────────┐
│  Frame: add()            │  ← Top of stack (current)
│    a = 3                 │
│    b = 5                 │
│    sum = 8               │
├──────────────────────────┤
│  Frame: main()           │
│    args = <ref>          │
│    result = (pending)    │
├──────────────────────────┤
│  ... (thread bootstrap)  │
└──────────────────────────┘
```

### Configuring stack size

```bash
# Default is typically 512KB–1MB depending on OS/JVM
java -Xss512k MyApp

# Increase for deeply recursive applications
java -Xss2m MyApp
```

---

## Heap Memory

The heap is a **shared** memory region where all **objects and arrays** are allocated.

### What lives on the heap?
- All objects created with `new`
- All arrays (even `int[]`)
- Instance variables of objects
- Static variables (stored in the Class object on the heap)

### Heap structure (Generational)

```
┌─────────────────────────────────────────────────────┐
│                     HEAP                            │
├─────────────┬────────────┬──────────────────────────┤
│   Young Gen │   Old Gen  │     Metaspace            │
│  ┌────────┐ │            │  (class metadata,        │
│  │  Eden  │ │  Tenured   │   method bytecode)       │
│  ├────────┤ │  objects   │                          │
│  │  S0    │ │            │  Not part of heap but    │
│  ├────────┤ │            │  managed by JVM          │
│  │  S1    │ │            │                          │
│  └────────┘ │            │                          │
└─────────────┴────────────┴──────────────────────────┘
```

### Configuring heap size

```bash
java -Xms256m -Xmx2g MyApp    # Initial: 256MB, Max: 2GB
java -XX:NewRatio=2 MyApp      # Old:Young = 2:1
```

---

## What Goes Where — Detailed Breakdown

```java
public class MemoryAllocation {

    static int CONSTANT = 42;                  // Heap (in Class object)

    public void demonstrate() {
        int x = 10;                            // Stack (primitive)
        String name = "Alice";                 // Stack: reference; Heap: String object (pool)
        int[] numbers = {1, 2, 3};             // Stack: reference; Heap: array object
        Person person = new Person("Bob");     // Stack: reference; Heap: Person object

        calculate(x);                          // New stack frame
    }

    private int calculate(int value) {         // 'value' is a copy on the stack
        int result = value * 2;                // Stack
        return result;                         // Stack frame popped on return
    }
}
```

### Memory layout illustration

```
     STACK (Thread-1)              HEAP
   ┌─────────────────┐      ┌──────────────────────┐
   │ x = 10          │      │ String "Alice" [pool] │
   │ name ──────────────────▶│                      │
   │ numbers ───────────────▶│ int[] {1, 2, 3}      │
   │ person ────────────────▶│ Person { "Bob" }     │
   └─────────────────┘      └──────────────────────┘
```

---

## Escape Analysis and Stack Allocation

Modern JVMs can perform **escape analysis** to allocate objects on the stack when they don't escape the method scope.

```java
// ✅ JVM may stack-allocate this object (it doesn't escape)
public int sumCoordinates() {
    Point p = new Point(3, 4);   // JIT may replace with scalar values on stack
    return p.x + p.y;
}

// ❌ Object escapes — must be heap-allocated
public Point createPoint() {
    Point p = new Point(3, 4);
    return p;                    // Escapes method — heap allocation required
}
```

### Escape analysis optimizations

| Optimization          | Description                                        |
|-----------------------|----------------------------------------------------|
| Stack allocation      | Object placed on stack instead of heap              |
| Scalar replacement    | Object decomposed into its primitive fields         |
| Lock elision          | Synchronization removed for non-escaping objects    |

```bash
# Escape analysis is enabled by default (JDK 6u23+)
# Disable for benchmarking:
java -XX:-DoEscapeAnalysis MyApp
```

---

## StackOverflowError vs OutOfMemoryError

### StackOverflowError

```java
// ❌ Anti-pattern: unbounded recursion
public class StackOverflowDemo {
    public static void recurse() {
        recurse();   // No base case — infinite recursion
    }

    public static void main(String[] args) {
        recurse();   // Throws java.lang.StackOverflowError
    }
}
```

**Cause:** Stack frames exceed the thread's stack size (`-Xss`).

### OutOfMemoryError (Heap)

```java
// ❌ Anti-pattern: unbounded object creation
public class OOMDemo {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        while (true) {
            list.add(new byte[1_000_000]);   // 1MB per iteration
        }
        // Throws java.lang.OutOfMemoryError: Java heap space
    }
}
```

**Cause:** Heap space exhausted and GC cannot reclaim enough memory.

---

## Comparison Table

| Feature            | Stack                          | Heap                              |
|--------------------|--------------------------------|-----------------------------------|
| **Scope**          | Per-thread                     | Shared across all threads         |
| **Speed**          | Very fast (pointer bump)       | Slower (allocation + GC)          |
| **Size**           | Small (512KB–1MB default)      | Large (configurable, GBs)         |
| **Contents**       | Primitives, references, frames | Objects, arrays                   |
| **Lifecycle**      | Automatic (frame pop)          | Managed by Garbage Collector      |
| **Thread-safety**  | Inherently thread-safe         | Requires synchronization          |
| **Error**          | StackOverflowError             | OutOfMemoryError                  |
| **Configuring**    | `-Xss`                         | `-Xms`, `-Xmx`                   |

---

## Best Practices

```java
// ✅ Prefer local variables (stack) over fields (heap) when possible
public int compute(int[] data) {
    int sum = 0;                // Stack — fast and auto-cleaned
    for (int val : data) {
        sum += val;
    }
    return sum;
}

// ✅ Limit recursion depth or use iteration
public long factorial(int n) {
    long result = 1;
    for (int i = 2; i <= n; i++) {
        result *= i;            // Iterative — no stack overflow risk
    }
    return result;
}

// ❌ Don't rely on finalizers for resource cleanup
// ❌ Don't create unnecessary objects in hot loops
// ❌ Don't set -Xss too large (wastes memory per thread)
```

---

## Interview Tips
- Stack memory is always faster because it's just pointer arithmetic (push/pop)
- Primitives declared as local variables go on the stack; as fields they live on the heap inside the object
- Escape analysis is a JIT optimization — it only kicks in after warmup
- Each thread has its own stack but shares the heap with all other threads

---

[Next: Q2 - Object Lifecycle →](./q02-object-lifecycle.md)

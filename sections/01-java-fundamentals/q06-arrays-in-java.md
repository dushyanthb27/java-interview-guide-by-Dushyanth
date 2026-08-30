# Q6: How Do Arrays Work in Java?

## 📋 Question
How do arrays work in Java? Explain declaration, initialization, multi-dimensional arrays, and common array operations.

## Difficulty: 🟢 Easy

---

## 💡 Answer

Arrays in Java are **fixed-size, contiguous data structures** that store elements of the same type. Unlike C/C++, Java arrays are **objects** — they are allocated on the heap, have a `length` field, and perform **bounds checking** at runtime (throwing `ArrayIndexOutOfBoundsException` for invalid indices). Arrays are **covariant**: `String[]` is a subtype of `Object[]`, which can lead to runtime `ArrayStoreException` issues.

**Declaration and Initialization**: Arrays can be declared with the type before or after the variable name (`int[] arr` or `int arr[]`, though the former is preferred). They can be initialized with `new` (specifying size), with an initializer list, or anonymously.

**Multi-dimensional Arrays**: Java supports multi-dimensional arrays as "arrays of arrays." Unlike languages with true 2D memory layouts, Java's multi-dimensional arrays are **jagged** by default — each row can have a different length. This means a `int[][]` is an array of `int[]` references, each pointing to a separately allocated array.

**Key Difference — Array vs ArrayList**: Arrays have a **fixed size** set at creation, use `[]` syntax, and can hold primitives. `ArrayList` is dynamically resizable, uses generics (objects only), and provides richer APIs. Use arrays for performance-critical, fixed-size data; use `ArrayList` for flexibility.

### Code Example

```java
package com.interview.fundamentals;

import java.util.Arrays;
import java.util.List;

public class ArraysDemo {

    public static void main(String[] args) {

        // === DECLARATION AND INITIALIZATION ===

        // ✅ Declare and allocate (elements default to 0 for int)
        int[] numbers = new int[5];

        // ✅ Declare with initializer list
        int[] primes = {2, 3, 5, 7, 11};

        // ✅ Declare and initialize with new keyword
        int[] fibonacci = new int[]{1, 1, 2, 3, 5, 8, 13};

        // ✅ Anonymous array (useful for method arguments)
        printArray(new int[]{10, 20, 30});

        // ❌ Invalid: cannot specify size AND initializer
        // int[] bad = new int[3]{1, 2, 3};  // Compile error

        // === ARRAY PROPERTIES ===
        System.out.println("Length: " + primes.length);  // 5 (field, not method)
        System.out.println("First: " + primes[0]);       // 2
        System.out.println("Last: " + primes[primes.length - 1]);  // 11

        // === DEFAULT VALUES ===
        int[] intArr = new int[3];         // [0, 0, 0]
        boolean[] boolArr = new boolean[3]; // [false, false, false]
        String[] strArr = new String[3];    // [null, null, null]
        double[] dblArr = new double[3];    // [0.0, 0.0, 0.0]

        // === MULTI-DIMENSIONAL ARRAYS ===

        // ✅ Regular 2D array (3 rows x 4 columns)
        int[][] matrix = new int[3][4];
        matrix[0][0] = 1;
        matrix[2][3] = 99;

        // ✅ 2D array with initializer
        int[][] grid = {
            {1, 2, 3},
            {4, 5, 6},
            {7, 8, 9}
        };

        // ✅ Jagged array — rows have different lengths
        int[][] jagged = new int[3][];
        jagged[0] = new int[]{1, 2};          // 2 elements
        jagged[1] = new int[]{3, 4, 5, 6};    // 4 elements
        jagged[2] = new int[]{7};              // 1 element

        // Iterate jagged array safely
        for (int[] row : jagged) {
            for (int val : row) {
                System.out.print(val + " ");
            }
            System.out.println();
        }

        // === ARRAYS UTILITY CLASS ===
        arraysUtilityDemo();

        // === ARRAY COPYING ===
        arrayCopyDemo();
    }

    static void arraysUtilityDemo() {
        int[] data = {5, 2, 8, 1, 9, 3};

        // ✅ Sorting
        int[] sorted = Arrays.copyOf(data, data.length);
        Arrays.sort(sorted);
        System.out.println("Sorted: " + Arrays.toString(sorted));  // [1, 2, 3, 5, 8, 9]

        // ✅ Binary search (array MUST be sorted first)
        int index = Arrays.binarySearch(sorted, 8);
        System.out.println("Index of 8: " + index);  // 4

        // ✅ Filling
        int[] filled = new int[5];
        Arrays.fill(filled, 42);
        System.out.println("Filled: " + Arrays.toString(filled));  // [42, 42, 42, 42, 42]

        // ✅ Comparison
        int[] a = {1, 2, 3};
        int[] b = {1, 2, 3};
        System.out.println("a == b: " + (a == b));              // false (different references)
        System.out.println("equals: " + Arrays.equals(a, b));   // true  (same content)

        // ✅ Deep equals for multi-dimensional arrays
        int[][] m1 = {{1, 2}, {3, 4}};
        int[][] m2 = {{1, 2}, {3, 4}};
        System.out.println("deepEquals: " + Arrays.deepEquals(m1, m2));  // true

        // ✅ Convert to List (returns fixed-size List backed by the array)
        String[] names = {"Alice", "Bob", "Charlie"};
        List<String> nameList = Arrays.asList(names);
        // nameList.add("Dave");  // ❌ UnsupportedOperationException (fixed-size)
        nameList.set(0, "Alicia");  // ✅ Modification is reflected in original array

        // ✅ Truly mutable List (Java 9+ for immutable, ArrayList for mutable)
        List<String> mutableList = new java.util.ArrayList<>(Arrays.asList(names));
        mutableList.add("Dave");  // ✅ Works
    }

    static void arrayCopyDemo() {
        int[] source = {10, 20, 30, 40, 50};

        // ✅ Arrays.copyOf — copies to a new array (can resize)
        int[] copy1 = Arrays.copyOf(source, source.length);       // Full copy
        int[] copy2 = Arrays.copyOf(source, 3);                   // [10, 20, 30]
        int[] copy3 = Arrays.copyOf(source, 8);                   // [10, 20, 30, 40, 50, 0, 0, 0]

        // ✅ Arrays.copyOfRange — copies a sub-range
        int[] sub = Arrays.copyOfRange(source, 1, 4);             // [20, 30, 40]

        // ✅ System.arraycopy — fastest copy (native method)
        int[] dest = new int[5];
        System.arraycopy(source, 0, dest, 0, source.length);
        // Parameters: src, srcPos, dest, destPos, length

        // ❌ Shallow copy warning for object arrays
        String[] original = {"Hello", "World"};
        String[] shallowCopy = Arrays.copyOf(original, original.length);
        // Both arrays reference the same String objects
        // For mutable objects, changes to elements affect both arrays

        System.out.println("Copy: " + Arrays.toString(copy1));
        System.out.println("Sub:  " + Arrays.toString(sub));
    }

    static void printArray(int[] arr) {
        System.out.println(Arrays.toString(arr));
    }
}
```

### Array vs ArrayList Comparison

```
┌─────────────────────┬──────────────────┬──────────────────────┐
│ Feature             │ Array            │ ArrayList            │
├─────────────────────┼──────────────────┼──────────────────────┤
│ Size                │ Fixed            │ Dynamic (grows 50%)  │
│ Primitives          │ Yes (int[])      │ No (Integer only)    │
│ Type Safety         │ Runtime check    │ Compile-time generic │
│ Performance         │ Faster (direct)  │ Slightly slower      │
│ Syntax              │ arr[i]           │ list.get(i)          │
│ Length              │ arr.length       │ list.size()          │
│ Multi-dimensional   │ Yes (int[][])    │ List<List<Integer>>  │
│ Null Elements       │ Yes (obj arrays) │ Yes                  │
│ Memory              │ Contiguous       │ Contiguous (backing) │
│ API Richness        │ Arrays utility   │ Rich Collection API  │
└─────────────────────┴──────────────────┴──────────────────────┘
```

### ⚠️ Common Mistake
**Array covariance can cause runtime errors**: Since `String[]` is a subtype of `Object[]`, you can assign a `String[]` to an `Object[]` variable. But storing a non-String in that `Object[]` throws `ArrayStoreException` at runtime — the compiler cannot catch this. Generics (e.g., `List<String>`) are invariant and catch such errors at compile time.

```java
String[] strings = {"hello", "world"};
Object[] objects = strings;   // ✅ Compiles (covariance)
objects[0] = 42;              // ❌ ArrayStoreException at runtime!
```

### ⚡ Performance Tip
`System.arraycopy()` is the **fastest** way to copy arrays — it's a native method that uses memory-level block copying (similar to C's `memcpy`). For small arrays, the JIT may inline the copy entirely. `Arrays.copyOf()` internally calls `System.arraycopy()`. For sorting, `Arrays.sort()` uses **dual-pivot Quicksort** for primitives (O(n log n) average) and **TimSort** for objects (stable, O(n log n)). The choice of algorithm matters: TimSort is stable (preserves order of equal elements), while Quicksort is not.

### 🔧 JVM Internals
Arrays in Java are objects with a special **header** that includes the standard object header (mark word + klass pointer) plus a **length field** (4 bytes). This is why `arr.length` is O(1) — it reads a field, not a method call. Primitive arrays store values **contiguously** in memory, enabling CPU cache-friendly sequential access. Object arrays store **references** contiguously, but the objects themselves may be scattered across the heap. The JVM performs **bounds checking** on every array access — the JIT compiler can sometimes eliminate redundant bounds checks via **range check elimination** when it can prove the index is always valid (e.g., in a standard for loop iterating from 0 to `arr.length`).

---

## 🎯 Interviewer Insight
Interviewers check whether you understand the fundamental trade-offs between arrays and collections, and whether you know about array covariance (a classic Java design flaw). Demonstrating knowledge of the `Arrays` utility class, `System.arraycopy()`, and the difference between shallow and deep copies shows practical experience. Understanding jagged arrays vs true multi-dimensional arrays reveals your knowledge of Java's memory model.

## ⚡ Follow-up Questions
1. Why is array covariance considered a design flaw, and how do generics solve this?
2. What sorting algorithm does `Arrays.sort()` use for primitives vs objects, and why?
3. How does `Arrays.asList()` differ from `List.of()`, and what are the mutability implications?

## 📚 Related Topics
- [Data Types and Primitives](./q03-data-types-primitives.md)
- [Strings and String Pool](./q07-strings-and-string-pool.md)
- [Generics & Type System](../09-generics-type-system/README.md)

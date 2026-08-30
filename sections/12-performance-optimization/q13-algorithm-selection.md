# Algorithm Selection for Performance

[Home](../../README.md) > [Performance Optimization](./README.md) > Algorithm Selection

---

## Overview

Choosing the right algorithm often yields orders-of-magnitude performance improvements that no amount of micro-optimization can match. Understanding time and space complexity, and knowing when to trade one for the other, is essential for writing performant Java code.

---

## 1. Time Complexity Analysis

### Big-O Growth Rates

```
Input Size    O(1)    O(log n)   O(n)      O(n log n)   O(n^2)       O(2^n)
----------   ------  ---------  --------  -----------  -----------  ----------
10           1       3          10        33           100          1,024
100          1       7          100       664          10,000       1.27e30
1,000        1       10         1,000     9,966        1,000,000    ∞
10,000       1       13         10,000    132,877      100,000,000  ∞
1,000,000    1       20         1,000,000 19,931,569   10^12        ∞
```

```java
// O(1) - Constant time
public int getFirst(int[] arr) {
    return arr[0]; // same time regardless of array size
}

// O(log n) - Logarithmic
public int binarySearch(int[] sorted, int target) {
    int low = 0, high = sorted.length - 1;
    while (low <= high) {
        int mid = (low + high) >>> 1;
        if (sorted[mid] == target) return mid;
        else if (sorted[mid] < target) low = mid + 1;
        else high = mid - 1;
    }
    return -1; // halves search space each iteration
}

// O(n) - Linear
public int linearSearch(int[] arr, int target) {
    for (int i = 0; i < arr.length; i++) {
        if (arr[i] == target) return i;
    }
    return -1; // checks every element
}

// O(n log n) - Linearithmic
public void efficientSort(int[] arr) {
    Arrays.sort(arr); // dual-pivot quicksort for primitives
}

// O(n^2) - Quadratic
public void bubbleSort(int[] arr) {
    for (int i = 0; i < arr.length; i++) {
        for (int j = 0; j < arr.length - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
            }
        }
    }
}
```

---

## 2. Space-Time Trade-Offs

```java
// Trade-off 1: Memoization (use memory to save time)

// O(2^n) time, O(n) space - exponential without memoization
public int fibRecursive(int n) {
    if (n <= 1) return n;
    return fibRecursive(n - 1) + fibRecursive(n - 2);
}

// O(n) time, O(n) space - linear with memoization
public int fibMemoized(int n) {
    int[] memo = new int[n + 1];
    Arrays.fill(memo, -1);
    return fibHelper(n, memo);
}

private int fibHelper(int n, int[] memo) {
    if (n <= 1) return n;
    if (memo[n] != -1) return memo[n];
    memo[n] = fibHelper(n - 1, memo) + fibHelper(n - 2, memo);
    return memo[n];
}

// O(n) time, O(1) space - iterative (best)
public int fibIterative(int n) {
    if (n <= 1) return n;
    int prev2 = 0, prev1 = 1;
    for (int i = 2; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}

// Trade-off 2: Precomputation (build lookup table)
// Finding duplicates: O(n^2) with nested loops vs O(n) with HashSet
public boolean hasDuplicatesNaive(int[] arr) {       // O(n^2) time, O(1) space
    for (int i = 0; i < arr.length; i++)
        for (int j = i + 1; j < arr.length; j++)
            if (arr[i] == arr[j]) return true;
    return false;
}

public boolean hasDuplicatesOptimal(int[] arr) {     // O(n) time, O(n) space
    Set<Integer> seen = new HashSet<>(arr.length);
    for (int value : arr) {
        if (!seen.add(value)) return true;
    }
    return false;
}
```

---

## 3. Algorithm Choice Impact

### Real-World Example: Finding Intersection of Two Lists

```java
// Approach 1: Nested loops - O(n * m)
public List<Integer> intersectionNaive(List<Integer> a, List<Integer> b) {
    List<Integer> result = new ArrayList<>();
    for (int x : a) {
        for (int y : b) {
            if (x == y) { result.add(x); break; }
        }
    }
    return result;
}

// Approach 2: Sort + Two pointers - O(n log n + m log m)
public List<Integer> intersectionSorted(List<Integer> a, List<Integer> b) {
    Collections.sort(a);
    Collections.sort(b);
    List<Integer> result = new ArrayList<>();
    int i = 0, j = 0;
    while (i < a.size() && j < b.size()) {
        int cmp = Integer.compare(a.get(i), b.get(j));
        if (cmp == 0) {
            result.add(a.get(i));
            i++; j++;
        } else if (cmp < 0) i++;
        else j++;
    }
    return result;
}

// Approach 3: HashSet - O(n + m) time, O(n) space
public List<Integer> intersectionHash(List<Integer> a, List<Integer> b) {
    Set<Integer> setA = new HashSet<>(a);
    List<Integer> result = new ArrayList<>();
    for (int x : b) {
        if (setA.remove(x)) result.add(x);  // O(1) lookup
    }
    return result;
}
```

### Performance Impact

```
n = m = 100,000 elements:
  Nested loops (O(n*m)):         ~25 seconds
  Sort + two pointers:           ~35 milliseconds
  HashSet intersection:          ~12 milliseconds

  HashSet is 2000x faster than nested loops!
```

---

## 4. Sorting Algorithm Selection

```java
// Java's built-in sorting algorithms
// Arrays.sort() for primitives: Dual-Pivot Quicksort - O(n log n) avg
int[] primitives = {5, 3, 8, 1, 9};
Arrays.sort(primitives);

// Arrays.sort() for objects: TimSort - O(n log n), stable
String[] objects = {"banana", "apple", "cherry"};
Arrays.sort(objects);

// Arrays.parallelSort(): parallel merge sort - for large arrays
int[] large = new int[10_000_000];
Arrays.parallelSort(large); // uses ForkJoinPool
```

### When to Use Which Sort

| Algorithm        | Best Case | Average | Worst | Stable | Use Case                     |
|-----------------|-----------|---------|-------|--------|------------------------------|
| TimSort          | O(n)      | O(n log n)| O(n log n)| Yes | Objects (default)           |
| Dual-Pivot QS   | O(n log n)| O(n log n)| O(n^2)| No | Primitives (default)        |
| Counting Sort    | O(n+k)   | O(n+k)  | O(n+k)| Yes  | Small integer range          |
| Radix Sort       | O(nk)    | O(nk)   | O(nk) | Yes  | Fixed-length integers/strings|
| Parallel Sort    | -        | O(n log n / p)| - | No | Large arrays (>10K elements) |

```java
// Custom sorting for special cases

// Counting sort for small integer range (O(n+k))
public void countingSort(int[] arr, int maxVal) {
    int[] count = new int[maxVal + 1];
    for (int v : arr) count[v]++;
    int idx = 0;
    for (int i = 0; i <= maxVal; i++) {
        while (count[i]-- > 0) arr[idx++] = i;
    }
}

// Partial sort: find top K elements - O(n log k) instead of O(n log n)
public List<Integer> topK(int[] arr, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(k);
    for (int val : arr) {
        if (minHeap.size() < k) {
            minHeap.offer(val);
        } else if (val > minHeap.peek()) {
            minHeap.poll();
            minHeap.offer(val);
        }
    }
    return new ArrayList<>(minHeap);
}
// Full sort: O(n log n) = O(10^6 * 20) = 20M operations
// Top-K:     O(n log k) = O(10^6 * 10) = 10M operations (for k=1000)
```

---

## 5. Search Algorithm Optimization

```java
// Linear search: O(n) - use for unsorted/small data
public int linearSearch(List<String> list, String target) {
    for (int i = 0; i < list.size(); i++) {
        if (list.get(i).equals(target)) return i;
    }
    return -1;
}

// Binary search: O(log n) - requires sorted data
public int binarySearch(List<String> sortedList, String target) {
    return Collections.binarySearch(sortedList, target);
}

// Hash-based lookup: O(1) average - best for frequent lookups
public String hashLookup(Map<String, String> map, String key) {
    return map.get(key);
}

// Trie lookup: O(L) where L = key length - best for prefix matching
public class TrieNode {
    TrieNode[] children = new TrieNode[26];
    boolean isEnd;
}
```

### Search Performance (1 million elements, 100K lookups)

```
Linear search:    ~50,000 ms   (O(n) per lookup)
Binary search:    ~5 ms        (O(log n) per lookup, sorted data)
HashMap lookup:   ~3 ms        (O(1) per lookup)
```

---

## 6. Hash-Based vs Tree-Based Lookups

```java
// HashMap: O(1) average, O(log n) worst case (tree bins since Java 8)
Map<String, User> hashMap = new HashMap<>();
// Best for: exact key lookups, no ordering needed

// TreeMap: O(log n) guaranteed
Map<String, User> treeMap = new TreeMap<>();
// Best for: range queries, ordered iteration, navigable operations

// When to use TreeMap over HashMap:
NavigableMap<Integer, String> scores = new TreeMap<>();
scores.put(85, "Alice");
scores.put(92, "Bob");
scores.put(78, "Charlie");
scores.put(95, "Diana");

// Range query: all scores between 80-90
SortedMap<Integer, String> range = scores.subMap(80, 91);
// Result: {85=Alice, 92=Bob} - O(log n + k)

// Floor/ceiling operations
Map.Entry<Integer, String> justBelow90 = scores.floorEntry(90); // 85=Alice
Map.Entry<Integer, String> justAbove90 = scores.ceilingEntry(90); // 92=Bob
```

---

## 7. Algorithm Optimization Patterns

```java
// Pattern 1: Sliding Window - O(n) instead of O(n*k)
// Find maximum sum subarray of size k
public int maxSumSubarrayNaive(int[] arr, int k) { // O(n*k)
    int maxSum = Integer.MIN_VALUE;
    for (int i = 0; i <= arr.length - k; i++) {
        int sum = 0;
        for (int j = i; j < i + k; j++) sum += arr[j];
        maxSum = Math.max(maxSum, sum);
    }
    return maxSum;
}

public int maxSumSubarrayOptimal(int[] arr, int k) { // O(n)
    int windowSum = 0;
    for (int i = 0; i < k; i++) windowSum += arr[i];
    int maxSum = windowSum;
    for (int i = k; i < arr.length; i++) {
        windowSum += arr[i] - arr[i - k]; // slide window
        maxSum = Math.max(maxSum, windowSum);
    }
    return maxSum;
}

// Pattern 2: Two Pointers - O(n) instead of O(n^2)
// Check if sorted array has pair summing to target
public boolean hasPairSum(int[] sorted, int target) { // O(n)
    int left = 0, right = sorted.length - 1;
    while (left < right) {
        int sum = sorted[left] + sorted[right];
        if (sum == target) return true;
        else if (sum < target) left++;
        else right--;
    }
    return false;
}

// Pattern 3: Divide and Conquer
// Merge sort, quicksort, binary search
// Reduce O(n^2) to O(n log n) by splitting problem

// Pattern 4: Dynamic Programming
// Reduce O(2^n) to O(n) or O(n^2) by caching subproblems
```

---

## 8. Complexity Quick Reference

| Operation                  | Naive        | Optimal       | Key Technique      |
|---------------------------|--------------|---------------|--------------------|
| Finding duplicates         | O(n^2)       | O(n)          | HashSet            |
| Sorting                   | O(n^2)       | O(n log n)    | Merge/Quick sort   |
| Finding k-th largest      | O(n log n)   | O(n)          | QuickSelect        |
| Substring search          | O(n*m)       | O(n+m)        | KMP / Rabin-Karp   |
| Graph shortest path       | O(V^2)       | O(E log V)    | Dijkstra + heap    |
| Set intersection          | O(n*m)       | O(n+m)        | HashSet            |
| Max subarray sum          | O(n^3)       | O(n)          | Kadane's algorithm |
| Fibonacci                 | O(2^n)       | O(n) or O(1)  | DP / iterative     |
| Power (x^n)               | O(n)         | O(log n)      | Fast exponentiation|

---

## Interview Tips

- Algorithm choice typically matters 10-1000x more than micro-optimization
- Always state the time AND space complexity of your solution
- Hash-based structures give O(1) lookups but O(n) space and no ordering
- Tree-based structures give O(log n) lookups with ordering and range queries
- For top-K problems, use a min-heap of size K: O(n log k) instead of O(n log n)
- Sliding window and two-pointer techniques convert O(n^2) to O(n)
- Know when to trade space for time (memoization, precomputation, hash tables)

---

[<< Previous: Micro-Optimizations](./q12-micro-optimizations.md) | [Next: Data Structure Selection >>](./q14-data-structure-selection.md)

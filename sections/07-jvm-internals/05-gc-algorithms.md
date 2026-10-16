# Q5: What GC Algorithms Are Used?

## 📋 Question
What garbage collection algorithms does the JVM use? Explain generational collection, minor vs major GC, and how objects are promoted.

## Difficulty: 🟠 Hard

---

## 💡 Answer

The JVM uses several GC algorithms as building blocks — **Mark-Sweep**, **Mark-Compact**, and **Copying** — combined with a **generational collection** strategy based on the **weak generational hypothesis**: most objects die young.

### Core GC Algorithms

#### 1. Mark-Sweep

```
Before:                After Mark:            After Sweep:
┌───┬───┬───┬───┐     ┌───┬───┬───┬───┐     ┌───┬───┬───┬───┐
│ A │ B │ C │ D │     │ A✓│ B │ C✓│ D │     │ A │   │ C │   │
└───┴───┴───┴───┘     └───┴───┴───┴───┘     └───┴───┴───┴───┘
  ↑       ↑             ↑       ↑             ↑       ↑
 root    root          root    root          root    root

Mark:  Traverse from GC roots, mark all reachable objects
Sweep: Free all unmarked objects

⚠️ Problem: Memory fragmentation — free space is scattered
```

#### 2. Mark-Compact

```
Before:                After Mark:            After Compact:
┌───┬───┬───┬───┐     ┌───┬───┬───┬───┐     ┌───┬───┬───────┐
│ A │ B │ C │ D │     │ A✓│ B │ C✓│ D │     │ A │ C │ FREE  │
└───┴───┴───┴───┘     └───┴───┴───┴───┘     └───┴───┴───────┘
  ↑       ↑             ↑       ↑             ↑   ↑
 root    root          root    root          root root

Mark:    Same as Mark-Sweep
Compact: Move live objects to one end, update references

✅ No fragmentation — contiguous free space
⚠️ Slower — must move objects and update all references
```

#### 3. Copying (Semi-Space)

```
Before:                              After Copy:
┌─────────────┬─────────────┐        ┌─────────────┬─────────────┐
│  FROM Space │  TO Space   │        │  FROM Space │  TO Space   │
│ A  B  C  D  │  (empty)    │   →    │  (cleared)  │  A  C       │
│ ↑     ↑     │             │        │             │  ↑  ↑       │
│root  root   │             │        │             │ root root   │
└─────────────┴─────────────┘        └─────────────┴─────────────┘
                                     Then swap FROM ↔ TO

Copy live objects from FROM to TO, then swap spaces

✅ No fragmentation, very fast (just copy live objects)
⚠️ Wastes 50% of memory (one space always empty)
✅ Cost proportional to LIVE objects, not total heap
```

### Algorithm Comparison

| Algorithm | Fragmentation | Speed | Space Overhead | Used In |
|-----------|--------------|-------|----------------|---------|
| **Mark-Sweep** | Yes | Medium | None | CMS (Old Gen) |
| **Mark-Compact** | No | Slow | None | Serial/Parallel (Old Gen) |
| **Copying** | No | Fast | 50% | Young Generation |

### Generational Garbage Collection

```
┌════════════════════════════════════════════════════════════════┐
║                         JVM HEAP                               ║
║                                                                ║
║  ┌──────────────────────────────┐  ┌────────────────────────┐  ║
║  │      YOUNG GENERATION        │  │    OLD GENERATION      │  ║
║  │         (~1/3 of heap)       │  │     (~2/3 of heap)     │  ║
║  │                              │  │                        │  ║
║  │  ┌────────────────────────┐  │  │  Long-lived objects    │  ║
║  │  │       EDEN             │  │  │  that survived many    │  ║
║  │  │  (new objects here)    │  │  │  GC cycles             │  ║
║  │  │    ~80% of Young Gen   │  │  │                        │  ║
║  │  └────────────────────────┘  │  │  GC Algorithm:         │  ║
║  │  ┌──────────┐ ┌──────────┐   │  │  Mark-Compact or       │  ║
║  │  │    S0    │ │    S1    │   │  │  Mark-Sweep            │  ║
║  │  │(Survivor)│ │(Survivor)│   │  │                        │  ║
║  │  │  ~10%    │ │  ~10%    │   │  │                        │  ║
║  │  └──────────┘ └──────────┘   │  │                        │  ║
║  │                              │  │                        │  ║
║  │  GC Algorithm: Copying       │  │                        │  ║
║  └──────────────────────────────┘  └────────────────────────┘  ║
╚════════════════════════════════════════════════════════════════╝
```

### Object Lifecycle Through Generations

```
Step 1: Object created in EDEN
┌──────────────────┐  ┌────┐ ┌────┐
│  EDEN            │  │ S0 │ │ S1 │     OLD GEN
│  obj(age=0) ●    │  │    │ │    │     [            ]
└──────────────────┘  └────┘ └────┘

Step 2: Minor GC — Eden full, live objects copied to S0
┌──────────────────┐  ┌────┐ ┌────┐
│  EDEN            │  │ S0 │ │ S1 │     OLD GEN
│  (cleared)       │  │ ●  │ │    │     [            ]
└──────────────────┘  │age1│ └────┘
                      └────┘

Step 3: Next Minor GC — Eden + S0 survivors copied to S1
┌──────────────────┐  ┌────┐ ┌────┐
│  EDEN            │  │ S0 │ │ S1 │     OLD GEN
│  (cleared)       │  │    │ │ ●  │     [            ]
└──────────────────┘  └────┘ │age2│
                             └────┘

Step 4: After N GCs (age ≥ threshold) — promoted to Old Gen
┌──────────────────┐  ┌────┐ ┌────┐
│  EDEN            │  │ S0 │ │ S1 │     OLD GEN
│  (new objects)   │  │    │ │    │     [ ●(promoted) ]
└──────────────────┘  └────┘ └────┘
```

### Minor GC vs Major GC vs Full GC

```java
// Minor GC — collects Young Generation only
// - Triggered: Eden is full
// - Speed: Very fast (milliseconds) — most objects are dead
// - Algorithm: Copying (Eden + S0 → S1, then swap)
// - Stop-the-world: Short pause (all app threads paused)

// Major GC — collects Old Generation only
// - Triggered: Old Gen is running low on space
// - Speed: Slower (tens to hundreds of milliseconds)
// - Algorithm: Mark-Compact or Mark-Sweep
// - Stop-the-world: Longer pause

// Full GC — collects BOTH Young and Old Generations + Metaspace
// - Triggered: Old Gen full, Metaspace full, System.gc(), etc.
// - Speed: Slowest (can be seconds on large heaps)
// - Algorithm: Mark-Compact on entire heap
// - Stop-the-world: Longest pause — AVOID if possible
```

### GC Roots — Where Reachability Starts

```
GC Roots (starting points for mark phase):
├── Local variables on thread stacks
├── Active thread objects
├── Static fields of loaded classes
├── JNI references (native code references)
├── Monitor objects (synchronized locks)
├── JVM-internal references (class loaders, etc.)
└── Objects held by the finalizer queue
```

```java
public class GCRootsExample {
    // Static field — GC root
    private static List<Object> staticList = new ArrayList<>();

    public void method() {
        // Local variable — GC root (while method is active)
        Object localRef = new Object();

        // Object reachable from GC root → NOT collected
        staticList.add(new Object());

        // After method returns, localRef is popped from stack
        // If nothing else references the Object, it becomes eligible for GC
    }
}
```

### Object Aging and Promotion

```java
// Default tenuring threshold: 15 (for most collectors)
// Object age increments each time it survives a Minor GC

// JVM flags:
// -XX:MaxTenuringThreshold=15     (max age before promotion, range 0-15)
// -XX:+PrintTenuringDistribution  (see age distribution)

// Output of -XX:+PrintTenuringDistribution:
// Desired survivor size 1048576 bytes, new threshold 7 (max 15)
// - age   1:     524288 bytes,     524288 total
// - age   2:     262144 bytes,     786432 total
// - age   3:     131072 bytes,     917504 total
// Objects at age 7+ are promoted to Old Gen
```

### Card Table — Cross-Generational References

```
Problem: Old Gen object references Young Gen object
         Minor GC only scans Young Gen — would miss this reference!

Solution: CARD TABLE — a byte array tracking dirty regions of Old Gen

┌────────────────────────────────┐
│          OLD GEN               │
│  ┌──────┬──────┬──────┬──────┐ │
│  │Card 0│Card 1│Card 2│Card 3│ │  ← Each card = 512 bytes of heap
│  │      │  A──────────────┐  │ │
│  │      │      │      │   │  │ │
│  └──────┴──────┴──────┴───┼──┘ │
└───────────────────────────┼────┘
                            │ reference
┌───────────────────────────▼────┐
│          YOUNG GEN             │
│          obj B (target)        │
└────────────────────────────────┘

Card Table: [clean][DIRTY][clean][clean]
             Only Card 1 needs scanning during Minor GC

When Old Gen writes a reference to Young Gen:
  1. Write barrier marks the card as DIRTY
  2. During Minor GC, only DIRTY cards are scanned for roots
  3. This avoids scanning the entire Old Gen
```

```java
// Simulating cross-generational reference
public class CardTableDemo {
    static Object[] longLived;  // Will be in Old Gen after promotion

    public static void main(String[] args) {
        longLived = new Object[1000];   // Large array → likely Old Gen

        // This creates a cross-generational reference:
        // Old Gen array → Young Gen object
        longLived[0] = new Object();    // Card table entry marked dirty

        // During next Minor GC:
        // 1. Scan GC roots (stacks, static fields)
        // 2. Scan DIRTY cards in card table (find longLived[0] reference)
        // 3. Mark new Object() as live → survive collection
    }
}
```

### ⚠️ Common Mistake
Calling `System.gc()` explicitly. This is merely a **suggestion** to the JVM — it may be ignored entirely. Even when honored, it typically triggers a **Full GC** which causes the longest stop-the-world pause. Use `-XX:+DisableExplicitGC` in production to ignore `System.gc()` calls. Let the JVM manage GC timing.

### ⚡ Performance Tip
Tune the Young Generation size to ensure most short-lived objects die before reaching Old Gen. A properly-sized Young Gen (`-XX:NewRatio`, `-XX:NewSize`) reduces the frequency of expensive Major/Full GCs. Monitor the **promotion rate** — if too many objects are promoted to Old Gen, either increase Young Gen size or increase the tenuring threshold.

```bash
# Key tuning flags
-XX:NewRatio=2              # Old:Young = 2:1 (Young = 1/3 of heap)
-XX:SurvivorRatio=8         # Eden:S0:S1 = 8:1:1 (Eden = 80% of Young)
-XX:MaxTenuringThreshold=15 # Max age before promotion
-Xlog:gc*=info              # GC logging (Java 9+)
```

---

## 🎯 Interviewer Insight
Interviewers want you to explain the three core algorithms and then how they map to the generational heap. Strong candidates explain **why** copying is used for Young Gen (most objects die, so copying few survivors is cheap), **why** Mark-Compact is used for Old Gen (live object ratio is high), and the card table mechanism for cross-generational references. Mentioning GC roots demonstrates deep understanding.

## ⚡ Follow-up Questions
1. Why is the copying algorithm ideal for Young Generation, and what happens if the live object ratio is high?
2. What causes a Full GC, and how can you prevent it through sizing and tuning?
3. How does the card table work, and what is the overhead of maintaining it?
4. What is the difference between Minor GC, Major GC, and Full GC in terms of stop-the-world duration?
5. How does the tenuring threshold affect promotion rate, and when would you adjust it?
6. What are GC roots, and which types of objects serve as roots during the mark phase?
7. How does escape analysis affect garbage collection by allowing stack allocation?

## 📚 Related Topics
- [GC Collectors (G1, ZGC, Shenandoah)](./q06-gc-collectors.md)
- [JVM Tuning Parameters](./q12-jvm-tuning-parameters.md)
- [Stack vs Heap](./q09-stack-vs-heap.md)

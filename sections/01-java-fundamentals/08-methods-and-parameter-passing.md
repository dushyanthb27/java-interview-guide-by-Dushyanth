# Q8: How Does Parameter Passing Work in Java?

## 📋 Question
How does parameter passing work in Java? Is Java pass-by-value or pass-by-reference? Explain with examples for both primitives and objects.

## Difficulty: 🟡 Medium

---

## 💡 Answer

Java is **always pass-by-value**. There are no exceptions to this rule. However, the confusion arises because when you pass an object, you are passing the **value of the reference** (a copy of the pointer to the object), not the object itself and not a reference to the reference. This means:

- **Primitives**: A **copy of the value** is passed. Changing the parameter inside the method has **no effect** on the original variable.
- **Objects**: A **copy of the reference** (memory address) is passed. You **can modify the object's state** through the copied reference (since both references point to the same object), but you **cannot make the original reference point to a different object** by reassigning the parameter.

This is the single most misunderstood concept in Java. Many developers say "Java passes objects by reference" — this is **incorrect**. If Java were truly pass-by-reference, reassigning the parameter inside a method would change the caller's variable. It doesn't.

**The Litmus Test**: If you can write a `swap(a, b)` method that swaps the caller's variables, the language supports pass-by-reference. You **cannot** write such a method in Java — proving it is pass-by-value.

### Methods Overview
Beyond parameter passing, Java methods have important features: **method overloading** (same name, different parameter lists), **varargs** (variable-length arguments), and **method signature** (name + parameter types, used for overload resolution). Return type is NOT part of the method signature.

### Code Example

```java
package com.interview.fundamentals;

import java.util.ArrayList;
import java.util.List;

public class ParameterPassingDemo {

    public static void main(String[] args) {

        // === PRIMITIVE PASS-BY-VALUE ===
        int x = 10;
        modifyPrimitive(x);
        System.out.println("After modifyPrimitive: x = " + x);  // 10 (unchanged!)

        // === OBJECT REFERENCE PASS-BY-VALUE ===

        // Case 1: Modifying object STATE through the copied reference — WORKS
        List<String> list = new ArrayList<>();
        list.add("original");
        addToList(list);
        System.out.println("After addToList: " + list);  // [original, added inside method]
        // The method modified the SAME object both references point to

        // Case 2: Reassigning the reference inside method — DOES NOT affect caller
        StringBuilder sb = new StringBuilder("Hello");
        reassignReference(sb);
        System.out.println("After reassignReference: " + sb);  // "Hello" (unchanged!)
        // The method reassigned its LOCAL copy of the reference, not ours

        // === THE SWAP TEST (proves Java is NOT pass-by-reference) ===
        String a = "First";
        String b = "Second";
        swap(a, b);
        System.out.println("After swap: a=" + a + ", b=" + b);
        // a=First, b=Second — swap FAILED because Java is pass-by-value
    }

    // ❌ Cannot modify the caller's primitive — only a copy is received
    static void modifyPrimitive(int value) {
        value = 999;  // Modifies local copy only
        System.out.println("Inside method: value = " + value);  // 999
    }

    // ✅ Can modify the object's STATE through the copied reference
    static void addToList(List<String> list) {
        list.add("added inside method");  // Modifies the SAME object
        // Both the caller's reference and this parameter point to the same List
    }

    // ❌ Reassigning the parameter does NOT affect the caller's reference
    static void reassignReference(StringBuilder sb) {
        sb = new StringBuilder("Goodbye");  // Reassigns LOCAL copy only
        System.out.println("Inside method: " + sb);  // "Goodbye"
        // The caller's reference still points to the original "Hello" object
    }

    // ❌ Swap is impossible in Java — proves pass-by-value
    static void swap(String a, String b) {
        String temp = a;
        a = b;       // Reassigns local copy
        b = temp;    // Reassigns local copy
        // Caller's variables are completely unaffected
    }
}
```

### Visual Explanation

```
BEFORE calling addToList(list):

  Caller's stack         Heap
  ┌──────────┐      ┌────────────────────┐
  │ list ─────┼─────>│ ArrayList          │
  └──────────┘      │ ["original"]       │
                     └────────────────────┘

INSIDE addToList(list):  (parameter is a COPY of the reference)

  Caller's stack         Heap                    Method's stack
  ┌──────────┐      ┌────────────────────┐      ┌──────────┐
  │ list ─────┼─────>│ ArrayList          │<─────┼─ list    │
  └──────────┘      │ ["original"]       │      └──────────┘
                     └────────────────────┘
  Both references point to the SAME object → modifications are visible

INSIDE reassignReference(sb):  (reassignment only affects local copy)

  Caller's stack         Heap                    Method's stack
  ┌──────────┐      ┌────────────────────┐      ┌──────────┐
  │ sb ───────┼─────>│ StringBuilder      │      │ sb ──────┼──┐
  └──────────┘      │ "Hello"            │      └──────────┘  │
                     └────────────────────┘                     │
                     ┌────────────────────┐                     │
                     │ StringBuilder      │<────────────────────┘
                     │ "Goodbye"          │  (parameter points to NEW object)
                     └────────────────────┘
  Caller's reference is UNCHANGED — still points to "Hello"
```

### Varargs and Method Overloading

```java
package com.interview.fundamentals;

public class MethodFeaturesDemo {

    // === VARARGS (Variable-Length Arguments) ===
    // Must be the LAST parameter; only ONE varargs per method
    static int sum(int... numbers) {
        int total = 0;
        for (int n : numbers) {  // Varargs is actually an int[] array
            total += n;
        }
        return total;
    }

    // === METHOD OVERLOADING ===
    // Same name, different parameter lists (type, count, or order)
    // Return type is NOT part of the method signature

    static void print(String message) {
        System.out.println("String: " + message);
    }

    static void print(int number) {
        System.out.println("Int: " + number);
    }

    static void print(String message, int times) {
        for (int i = 0; i < times; i++) {
            System.out.println(message);
        }
    }

    // ❌ Cannot overload by return type alone — compile error
    // static int print(String message) { return 0; }  // Same signature as above

    public static void main(String[] args) {
        // ✅ Varargs invocation
        System.out.println(sum());           // 0 (empty array)
        System.out.println(sum(1, 2, 3));    // 6
        System.out.println(sum(10, 20));     // 30

        // ✅ Overloaded method resolution (compiler picks best match)
        print("Hello");         // Calls print(String)
        print(42);              // Calls print(int)
        print("Repeat", 3);    // Calls print(String, int)

        // ⚠️ Overloading with autoboxing and widening
        printNum(42);           // Calls printNum(int) — exact match
        printNum(42L);          // Calls printNum(long) — widening
    }

    // Overload resolution priority: exact match > widening > autoboxing > varargs
    static void printNum(int n)     { System.out.println("int: " + n); }
    static void printNum(long n)    { System.out.println("long: " + n); }
    static void printNum(Integer n) { System.out.println("Integer: " + n); }
}
```

### ⚠️ Common Mistake
**Believing Java passes objects by reference**: This misconception leads to bugs where developers expect reassigning a method parameter to affect the caller's variable. For example, a `resetUser(User user)` method that does `user = new User()` will NOT reset the caller's `user` variable — it only reassigns the local copy of the reference. To "reset" the object, you must modify its state (`user.setName(null)`) rather than reassigning the reference. If you need swap semantics, use a container object, an array, or return the new value.

### ⚡ Performance Tip
Varargs creates a **new array** on every invocation, which adds GC pressure in hot paths. For performance-critical methods, provide explicit overloads for common arities:

```java
// ✅ Optimized: no array allocation for common cases
static int sum(int a) { return a; }
static int sum(int a, int b) { return a + b; }
static int sum(int a, int b, int c) { return a + b + c; }
static int sum(int... numbers) { /* fallback for 4+ args */ }
// This is the pattern used by List.of(), Set.of(), Map.of() in Java 9+
```

### 🔧 JVM Internals
At the bytecode level, method parameters are stored in the **local variable table** of the method's stack frame. For primitives, the value is copied directly. For object references, the **reference value** (a pointer to the heap object, typically 4 bytes with compressed oops or 8 bytes otherwise) is copied. The `invokevirtual` bytecode instruction pushes arguments onto the operand stack before the call, and the callee pops them into its local variable table. Varargs compiles to an array creation (`anewarray` or `newarray`) followed by element stores (`aastore` or `iastore`) before the method call. Method overloading is resolved entirely at **compile time** — the bytecode contains the exact method descriptor (name + parameter types + return type), so the JVM performs no runtime resolution for overloaded methods (unlike overridden methods, which use dynamic dispatch).

---

## 🎯 Interviewer Insight
This is a **trick question** that separates well-prepared candidates from the rest. Interviewers want to hear you say "Java is always pass-by-value" and then demonstrate the nuance: object references are passed by value, so you can modify the object's state but cannot reassign the caller's reference. The swap example is the definitive proof. Candidates who confidently say "pass-by-reference for objects" immediately reveal a gap in understanding. Mentioning the method overloading resolution order (exact > widening > boxing > varargs) shows compiler-level knowledge.

## ⚡ Follow-up Questions
1. How would you implement a swap operation in Java without using a temp variable?
2. What is the method overloading resolution priority when multiple methods match?
3. Why can't you overload methods by return type alone?

## 📚 Related Topics
- [Data Types and Primitives](./q03-data-types-primitives.md)
- [Static vs Instance](./q10-static-vs-instance.md)
- [Object-Oriented Programming](../02-oop-concepts/README.md)

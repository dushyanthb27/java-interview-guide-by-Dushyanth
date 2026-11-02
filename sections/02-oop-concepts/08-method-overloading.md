# Method Overloading

[Home](../../README.md) > [Object-Oriented Programming](./README.md) > Method Overloading

---

## Q11 🟡 How Does Method Overloading Work?

**Question** — What is method overloading in Java? Explain the rules, type promotion, ambiguity issues, and how overloading interacts with varargs, autoboxing, and generics.

### Answer

**Method overloading** is compile-time polymorphism where multiple methods in the same class share the same name but differ in their **parameter lists** (number, types, or order of parameters). The compiler selects the most specific matching method based on the argument types.

**Rules for Overloading**

A valid overload must differ in at least one of — number of parameters, types of parameters, or order of parameter types. Return type alone does NOT make a valid overload.

```java
public class Printer {
    // Different number of parameters
    public void print(String msg) {
        System.out.println(msg);
    }

    public void print(String msg, int copies) {
        for (int i = 0; i < copies; i++) System.out.println(msg);
    }

    // Different parameter types
    public void print(int number) {
        System.out.println(number);
    }

    public void print(double number) {
        System.out.println(number);
    }

    // Different order of types
    public void print(String msg, int code) {
        System.out.printf("[%d] %s%n", code, msg);
    }

    public void print(int code, String msg) {
        System.out.printf("[%d] %s%n", code, msg);
    }

    // Return type alone does NOT make a valid overload
    // public int print(String msg) { return 0; } // COMPILE ERROR
}
```

**Method Resolution Order**

The compiler selects the most specific matching method in this priority — exact match, widening primitive conversion (byte → short → int → long → float → double), autoboxing/unboxing (int ↔ Integer), and finally varargs.

**Type Promotion (Widening)**

When no exact match exists, the compiler widens primitive types to find a compatible method.

```java
public class TypePromotion {
    public void process(int x)    { System.out.println("int: " + x); }
    public void process(long x)   { System.out.println("long: " + x); }
    public void process(double x) { System.out.println("double: " + x); }

    public static void main(String[] args) {
        var tp = new TypePromotion();

        tp.process(10);       // "int: 10"      → exact match
        tp.process(10L);      // "long: 10"     → exact match
        tp.process(10.0);     // "double: 10.0" → exact match

        byte b = 5;
        tp.process(b);        // "int: 5"       → byte widened to int

        short s = 100;
        tp.process(s);        // "int: 100"     → short widened to int

        float f = 3.14f;
        tp.process(f);        // "double: 3.14" → float widened to double
    }
}
```

Widening path — `byte → short → int → long → float → double` (with `char` also widening to `int`).

**Autoboxing Interaction**

When both primitive and wrapper overloads exist, exact match wins. If only the wrapper overload exists, autoboxing occurs.

```java
public class BoxingOverload {
    public void show(int x)       { System.out.println("primitive int"); }
    public void show(Integer x)   { System.out.println("Integer wrapper"); }
    public void show(Object x)    { System.out.println("Object"); }

    public static void main(String[] args) {
        var bo = new BoxingOverload();

        bo.show(42);                     // "primitive int" → exact match wins
        bo.show(Integer.valueOf(42));    // "Integer wrapper" → exact match
        bo.show((Object) 42);           // "Object" → explicit cast
    }
}

// Without the primitive overload:
public class BoxingOnly {
    public void show(Integer x)  { System.out.println("Integer"); }
    public void show(Object x)   { System.out.println("Object"); }

    public static void main(String[] args) {
        var bo = new BoxingOnly();
        bo.show(42); // "Integer" → autobox int to Integer (most specific)
    }
}
```

**Varargs Overloading**

Varargs (variable arity) is the least specific option. The compiler prefers exact matches over varargs.

```java
public class VarargsOverload {
    public void log(String msg) {
        System.out.println("Single: " + msg);
    }

    public void log(String msg, String... extras) {
        System.out.println("Varargs: " + msg + " " + Arrays.toString(extras));
    }

    public static void main(String[] args) {
        var v = new VarargsOverload();

        v.log("hello");                 // "Single: hello" → exact match preferred
        v.log("hello", "world");        // "Varargs: hello [world]"
        v.log("hello", "a", "b", "c"); // "Varargs: hello [a, b, c]"
    }
}
```

**Ambiguity Issues**

When multiple overload options are equally valid, the compiler reports an ambiguity error. This commonly happens with mixed primitive types.

```java
public class Ambiguous {
    // These cause ambiguity in certain calls
    public void calculate(int a, double b) {
        System.out.println("int, double");
    }

    public void calculate(double a, int b) {
        System.out.println("double, int");
    }

    public static void main(String[] args) {
        var amb = new Ambiguous();

        amb.calculate(10, 20.0);  // OK → "int, double" (exact match)
        amb.calculate(10.0, 20);  // OK → "double, int" (exact match)
        // amb.calculate(10, 20); // COMPILE ERROR: ambiguous!
        // Both (int→int, int→double) and (int→double, int→int) are equally valid
    }
}
```

**Overloading with Generics**

After type erasure, generic types become their bounds (usually Object). This means you cannot overload based solely on different generic type parameters.

```java
public class GenericOverload {
    // These are valid overloads (different erased types)
    public void process(List<String> list) {
        System.out.println("String list");
    }

    // COMPILE ERROR after type erasure, both become process(List)
    // public void process(List<Integer> list) {
    //     System.out.println("Integer list");
    // }

    // Workaround: use different method names or different parameter counts
    public void processIntegers(List<Integer> list) {
        System.out.println("Integer list");
    }
}
```

**Real-World Example — String.valueOf()**

The JDK uses overloading extensively for convenience. `String.valueOf()` is overloaded for every primitive type and Object.

```java
// The JDK uses overloading extensively
public class StringValueOfDemo {
    public static void main(String[] args) {
        // String.valueOf() is overloaded for every type
        String s1 = String.valueOf(42);           // valueOf(int)
        String s2 = String.valueOf(3.14);         // valueOf(double)
        String s3 = String.valueOf(true);         // valueOf(boolean)
        String s4 = String.valueOf('A');           // valueOf(char)
        String s5 = String.valueOf(new Object());  // valueOf(Object)
        String s6 = String.valueOf((Object) null); // "null" (no NPE)
    }
}
```

> **⚠️ Common Mistake** — Overloading with null argument causes ambiguity when multiple reference types are possible. Cast null to the desired type to resolve.

```java
// Overloading with null argument causes ambiguity
public class NullAmbiguity {
    public void test(String s) { System.out.println("String"); }
    public void test(Integer i) { System.out.println("Integer"); }

    public static void main(String[] args) {
        var na = new NullAmbiguity();
        // na.test(null);  // COMPILE ERROR: ambiguous

        // Fix: cast null to the desired type
        na.test((String) null);   // "String"
        na.test((Integer) null);  // "Integer"
    }
}
```

> **🔧 Performance Tip** — Overloaded methods are resolved at compile time, so there is no runtime dispatch overhead. However, avoid excessive overloading as it reduces code readability. Consider using the Builder pattern or method objects for many parameter combinations.

> **🎯 Interviewer Insight** — Interviewers test your understanding of the compiler's method resolution algorithm — exact match → widening → autoboxing → varargs. Tricky questions often involve ambiguity with `null` arguments or widening vs autoboxing priority.

> **⚡ Follow-up** — Can you overload `main(String[] args)` in Java?

> **⚡ Follow-up** — What is the difference between overloading and overriding?

> **⚡ Follow-up** — Why does widening take priority over autoboxing in method resolution?

> **📚 Related** — [Method Overriding](./09-method-overriding.md), [Polymorphism](./04-polymorphism.md), [Constructors](./02-constructors.md)

---

[← Previous: Interfaces vs Abstract Classes](./07-interfaces-vs-abstract-classes.md) · [Next: Method Overriding →](./09-method-overriding.md)

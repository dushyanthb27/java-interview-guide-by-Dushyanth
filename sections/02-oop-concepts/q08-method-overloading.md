# Q8: How Does Method Overloading Work?

## Question
What is method overloading in Java? Explain the rules, type promotion, ambiguity issues, and how overloading interacts with varargs, autoboxing, and generics.

## Difficulty: 🟡 Intermediate

---

## Answer

**Method overloading** is compile-time polymorphism where multiple methods in the same class share the same name but differ in their **parameter lists** (number, types, or order of parameters).

### Rules for Overloading

A valid overload must differ in at least one of:
1. **Number** of parameters
2. **Types** of parameters
3. **Order** of parameter types

```java
public class Printer {
    // ✅ Different number of parameters
    public void print(String msg) {
        System.out.println(msg);
    }

    public void print(String msg, int copies) {
        for (int i = 0; i < copies; i++) System.out.println(msg);
    }

    // ✅ Different parameter types
    public void print(int number) {
        System.out.println(number);
    }

    public void print(double number) {
        System.out.println(number);
    }

    // ✅ Different order of types
    public void print(String msg, int code) {
        System.out.printf("[%d] %s%n", code, msg);
    }

    public void print(int code, String msg) {
        System.out.printf("[%d] %s%n", code, msg);
    }

    // ❌ Return type alone does NOT make a valid overload
    // public int print(String msg) { return 0; } // COMPILE ERROR
}
```

### Method Resolution Order

The compiler selects the most specific matching method in this priority:

```
1. Exact match
2. Widening primitive conversion (byte → short → int → long → float → double)
3. Autoboxing/unboxing (int ↔ Integer)
4. Varargs
```

### Type Promotion (Widening)

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

```
Widening path:
byte → short → int → long → float → double
         char ↗
```

### Autoboxing Interaction

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

### Varargs Overloading

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

### Ambiguity Issues

```java
public class Ambiguous {
    // ❌ These cause ambiguity in certain calls
    public void calculate(int a, double b) {
        System.out.println("int, double");
    }

    public void calculate(double a, int b) {
        System.out.println("double, int");
    }

    public static void main(String[] args) {
        var amb = new Ambiguous();

        amb.calculate(10, 20.0);  // ✅ OK → "int, double" (exact match)
        amb.calculate(10.0, 20);  // ✅ OK → "double, int" (exact match)
        // amb.calculate(10, 20); // ❌ COMPILE ERROR: ambiguous!
        // Both (int→int, int→double) and (int→double, int→int) are equally valid
    }
}
```

### Overloading with Generics

```java
public class GenericOverload {
    // ✅ These are valid overloads (different erased types)
    public void process(List<String> list) {
        System.out.println("String list");
    }

    // ❌ COMPILE ERROR after type erasure, both become process(List)
    // public void process(List<Integer> list) {
    //     System.out.println("Integer list");
    // }

    // ✅ Workaround: use different method names or different parameter counts
    public void processIntegers(List<Integer> list) {
        System.out.println("Integer list");
    }
}
```

### Real-World Example: String.valueOf()

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

### Common Mistake

```java
// ❌ Overloading with null argument causes ambiguity
public class NullAmbiguity {
    public void test(String s) { System.out.println("String"); }
    public void test(Integer i) { System.out.println("Integer"); }

    public static void main(String[] args) {
        var na = new NullAmbiguity();
        // na.test(null);  // ❌ COMPILE ERROR: ambiguous

        // ✅ Fix: cast null to the desired type
        na.test((String) null);   // "String"
        na.test((Integer) null);  // "Integer"
    }
}
```

### Performance Tip
Overloaded methods are resolved at compile time, so there is no runtime dispatch overhead. However, avoid excessive overloading as it reduces code readability. Consider using the Builder pattern or method objects for many parameter combinations.

---

## Interviewer Insight
Interviewers test your understanding of the compiler's method resolution algorithm: exact match → widening → autoboxing → varargs. Tricky questions often involve ambiguity with `null` arguments or widening vs autoboxing priority.

## Follow-up Questions
1. Can you overload `main(String[] args)` in Java?
2. What is the difference between overloading and overriding?
3. Why does widening take priority over autoboxing in method resolution?

## Related Topics
- [Method Overriding](./q09-method-overriding.md)
- [Polymorphism](./q04-polymorphism.md)
- [Constructors](./q02-constructors.md)

# Q2: How Does Class Loading Work?

## 📋 Question
Explain the class loading mechanism in the JVM. What are the different phases, and when does class loading occur?

## Difficulty: 🟠 Hard

---

## 💡 Answer

Class loading in the JVM is a **lazy, on-demand process** that loads, links, and initializes classes as they are first referenced. The process consists of three major phases: **Loading**, **Linking**, and **Initialization**.

### Class Loading Lifecycle

```
  .class file (bytecode)
        │
        ▼
┌───────────────┐
│   LOADING     │  Read binary data from .class file
│               │  Create java.lang.Class object
└───────┬───────┘
        ▼
┌───────────────────────────────────────┐
│            LINKING                    │
│  ┌──────────────┐                    │
│  │ Verification │ Validate bytecode  │
│  └──────┬───────┘                    │
│         ▼                            │
│  ┌──────────────┐                    │
│  │ Preparation  │ Allocate static    │
│  │              │ fields (defaults)  │
│  └──────┬───────┘                    │
│         ▼                            │
│  ┌──────────────┐                    │
│  │ Resolution   │ Resolve symbolic   │
│  │  (optional)  │ references         │
│  └──────────────┘                    │
└───────────────┬───────────────────────┘
                ▼
┌───────────────┐
│INITIALIZATION │  Execute <clinit> —
│               │  static initializers
│               │  and static blocks
└───────────────┘
```

### Phase 1: Loading

During loading, the JVM performs three tasks:

1. **Finds** the binary representation (`.class` file) using a class loader
2. **Reads** the binary data and parses the class file format
3. **Creates** a `java.lang.Class` object in the Method Area to represent the class

```java
// When this class is first referenced, the JVM loads it
public class Employee {
    private String name;
    private int id;
}

// Loading is triggered by:
Employee emp = new Employee();    // 1. Creating an instance
Class<?> clazz = Employee.class;  // 2. Accessing .class literal
Class.forName("Employee");        // 3. Reflective loading
```

### Phase 2: Linking

Linking has three sub-phases:

#### 2a. Verification
Ensures bytecode is structurally correct and safe:
- **File format** — Magic number `0xCAFEBABE`, valid version
- **Metadata** — Class has a superclass (except Object), no circular inheritance
- **Bytecode** — Valid opcodes, correct stack usage, type safety
- **Symbolic references** — References can be resolved

```java
// The verifier catches illegal bytecode like:
// - Accessing private fields of another class directly
// - Stack overflow/underflow in operand stack
// - Type mismatches (pushing int, popping as reference)
// - Jumping to an instruction in the middle of another instruction
```

#### 2b. Preparation
Allocates memory for **static fields** and sets them to **default values** (not assigned values):

```java
public class Config {
    static int timeout = 30;       // Preparation: timeout = 0 (default)
                                   // Initialization: timeout = 30 (assigned)

    static final int MAX = 100;    // Compile-time constant: MAX = 100 immediately
                                   // (final + primitive/String literal = ConstantValue attribute)

    static String name = "App";    // Preparation: name = null
                                   // Initialization: name = "App"
}
```

| Type | Default Value |
|------|---------------|
| `int`, `short`, `byte` | `0` |
| `long` | `0L` |
| `float` | `0.0f` |
| `double` | `0.0d` |
| `boolean` | `false` |
| `char` | `'\u0000'` |
| `reference` | `null` |

#### 2c. Resolution
Replaces **symbolic references** in the constant pool with **direct references** (memory addresses):

```java
// In bytecode, method calls are symbolic references:
//   invokevirtual #4  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
//
// Resolution replaces #4 with the actual memory address of println()
//
// Resolution can be:
//   - Eager: resolved during linking
//   - Lazy: resolved when the reference is first used (more common)
```

### Phase 3: Initialization

Executes the class's **`<clinit>`** (class initialization) method, which contains:
- All static variable assignments
- All static initializer blocks — in **textual order**

```java
public class InitOrder {
    // These execute in order during <clinit>:

    static int x = 10;                    // 1. x = 10

    static {                              // 2. Static block
        System.out.println("x = " + x);  //    Prints: x = 10
        x = 20;                           //    x = 20
    }

    static int y = x + 5;                // 3. y = 25

    static {                              // 4. Static block
        System.out.println("y = " + y);  //    Prints: y = 25
    }
}
```

### Class Loading Triggers

A class is initialized **only** when one of these occurs (JLS §12.4.1):

```java
// ✅ Active use — TRIGGERS initialization
new MyClass();                        // 1. Instantiation
MyClass.staticField = 10;             // 2. Access static field (non-final)
MyClass.staticMethod();               // 3. Invoke static method
Class.forName("MyClass");             // 4. Reflective access
// 5. Subclass initialization triggers parent initialization
// 6. Main class of the application

// ❌ Passive use — does NOT trigger initialization
MyClass[] array = new MyClass[10];    // Array creation
int val = MyClass.COMPILE_CONSTANT;   // Compile-time constant (final + literal)
Class<?> c = MyClass.class;           // Class literal (loads but may not initialize)
```

### Lazy Loading in Action

```java
public class LazyLoadingDemo {
    public static void main(String[] args) {
        System.out.println("Main started");
        // HeavyClass is NOT loaded yet

        if (args.length > 0) {
            HeavyClass heavy = new HeavyClass(); // Loaded only if condition is true
        }

        System.out.println("Main ended");
    }
}

class HeavyClass {
    static {
        System.out.println("HeavyClass initialized!");
    }
}
// If no args: prints "Main started" then "Main ended" — HeavyClass never loads
```

### Static Initialization Order — Parent Before Child

```java
class Parent {
    static { System.out.println("Parent <clinit>"); }   // 1st
}

class Child extends Parent {
    static { System.out.println("Child <clinit>"); }    // 2nd
}

public class Main {
    public static void main(String[] args) {
        new Child();
        // Output:
        //   Parent <clinit>
        //   Child <clinit>
    }
}
```

### Class Loading Errors

```java
// ClassNotFoundException — CHECKED exception
// Thrown when: Class.forName() or ClassLoader.loadClass() can't find the class
try {
    Class.forName("com.example.NonExistent");
} catch (ClassNotFoundException e) {
    // The class was not found on the classpath at runtime
    System.err.println("Class not found: " + e.getMessage());
}

// NoClassDefFoundError — ERROR (unchecked)
// Thrown when: A class was available at compile time but missing at runtime
// OR when: Static initialization of a class fails
public class Fragile {
    static {
        if (true) throw new RuntimeException("Init failed!");
    }
}
// First access:  ExceptionInInitializerError (wraps RuntimeException)
// Subsequent:    NoClassDefFoundError (class marked as failed)
```

### Key Difference: ClassNotFoundException vs NoClassDefFoundError

| Aspect | ClassNotFoundException | NoClassDefFoundError |
|--------|----------------------|---------------------|
| **Type** | Checked Exception | Error (unchecked) |
| **When** | Explicit loading fails | Implicit loading fails |
| **Cause** | `Class.forName()`, `loadClass()` | Was in classpath at compile time, missing at runtime |
| **Recovery** | Catchable, recoverable | Usually fatal |
| **Example** | JDBC driver not on classpath | Dependency JAR removed after compilation |

### Observing Class Loading

```bash
# Print every class as it loads
java -verbose:class MyApp

# Output example:
# [Loaded java.lang.Object from /path/to/rt.jar]
# [Loaded java.lang.String from /path/to/rt.jar]
# [Loaded MyApp from file:/path/to/classes/]

# Java 9+ unified logging
java -Xlog:class+load=info MyApp
```

### ⚠️ Common Mistake
Assuming all classes are loaded at startup. The JVM loads classes **lazily** — only when they are first actively used. This is why you might not discover a missing class until a specific code path is executed in production. Always ensure your test coverage exercises all class references.

### ⚡ Performance Tip
Class loading involves I/O (reading `.class` files) and computation (verification). For applications with many classes (e.g., large Spring Boot apps), startup time can be dominated by class loading. **Application Class-Data Sharing (AppCDS)** in Java 10+ caches loaded classes in a shared archive, dramatically reducing startup time for subsequent launches.

```bash
# Create a class list
java -Xshare:off -XX:DumpLoadedClassList=classes.lst -jar myapp.jar

# Create shared archive
java -Xshare:dump -XX:SharedClassListFile=classes.lst -XX:SharedArchiveFile=app-cds.jsa -jar myapp.jar

# Run with shared archive (faster startup)
java -Xshare:on -XX:SharedArchiveFile=app-cds.jsa -jar myapp.jar
```

---

## 🎯 Interviewer Insight
Interviewers want to hear the three phases (Loading, Linking, Initialization) with the sub-phases of Linking. The best candidates explain lazy loading, the difference between `ClassNotFoundException` and `NoClassDefFoundError`, and can describe the static initialization order. Bonus points for mentioning `<clinit>` and the JLS active-use rules.

## ⚡ Follow-up Questions
1. What triggers the `<clinit>` method to execute?
2. Can a class be loaded more than once in the same JVM?
3. How does AppCDS improve startup time?

## 📚 Related Topics
- [Class Loaders](./q03-class-loaders.md)
- [JVM Architecture](./q01-jvm-architecture.md)
- [Metaspace](./q10-metaspace.md)

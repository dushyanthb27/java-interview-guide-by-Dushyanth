# Q3: What Are the Different Class Loaders?

## 📋 Question
What are the different types of class loaders in Java? Explain the parent delegation model and how to create custom class loaders.

## Difficulty: 🟠 Hard

---

## 💡 Answer

Java uses a **hierarchical class loader architecture** with a **parent delegation model** where each class loader delegates to its parent before attempting to load a class itself. This ensures core Java classes are always loaded by trusted class loaders.

### Class Loader Hierarchy

```
┌─────────────────────────────────────┐
│     Bootstrap Class Loader          │  ← Written in native code (C/C++)
│     (null in Java API)              │
│     Loads: java.lang.*, java.util.* │
│     Source: rt.jar / jrt:/java.base │
└──────────────┬──────────────────────┘
               │ parent
               ▼
┌─────────────────────────────────────┐
│   Platform Class Loader             │  ← Java 9+ (was Extension ClassLoader)
│   (sun.misc.Launcher$PlatformCL)    │
│   Loads: java.sql.*, javax.crypto.* │
│   Source: jrt:/java.sql, etc.       │
└──────────────┬──────────────────────┘
               │ parent
               ▼
┌─────────────────────────────────────┐
│   Application Class Loader          │  ← Also called System ClassLoader
│   (sun.misc.Launcher$AppCL)         │
│   Loads: Your application classes   │
│   Source: -classpath / -cp          │
└──────────────┬──────────────────────┘
               │ parent
               ▼
┌─────────────────────────────────────┐
│      Custom Class Loaders           │  ← User-defined (extends ClassLoader)
│      (e.g., Tomcat, OSGi)           │
│      Loads: Whatever you define     │
└─────────────────────────────────────┘
```

### The Three Built-in Class Loaders

```java
public class ClassLoaderDemo {
    public static void main(String[] args) {
        // 1. Bootstrap Class Loader — loads core Java classes
        //    Returns null because it's implemented in native code
        ClassLoader stringLoader = String.class.getClassLoader();
        System.out.println("String loader: " + stringLoader);    // null

        // 2. Platform Class Loader (Java 9+) — loads platform modules
        ClassLoader sqlLoader = java.sql.Driver.class.getClassLoader();
        System.out.println("SQL loader: " + sqlLoader);          // PlatformClassLoader

        // 3. Application Class Loader — loads application classes
        ClassLoader appLoader = ClassLoaderDemo.class.getClassLoader();
        System.out.println("App loader: " + appLoader);          // AppClassLoader

        // Walk the hierarchy
        ClassLoader current = appLoader;
        while (current != null) {
            System.out.println(current);
            current = current.getParent();
        }
        // Output:
        //   jdk.internal.loader.ClassLoaders$AppClassLoader@...
        //   jdk.internal.loader.ClassLoaders$PlatformClassLoader@...
        //   (null = Bootstrap — stops here)
    }
}
```

| Class Loader | Loads From | Java 8 Name | Java 9+ Name |
|-------------|-----------|-------------|---------------|
| **Bootstrap** | `rt.jar` / `jrt:/java.base` | Bootstrap | Bootstrap |
| **Extension** | `jre/lib/ext` | Extension (`ExtClassLoader`) | **Platform** (`PlatformClassLoader`) |
| **Application** | `-classpath` | System (`AppClassLoader`) | Application (`AppClassLoader`) |

### Parent Delegation Model

When a class loader receives a request to load a class, it follows this algorithm:

```
loadClass("com.example.MyService")
│
├── 1. Check if already loaded (findLoadedClass)
│   └── Found? → Return cached Class object
│
├── 2. Delegate to PARENT class loader
│   └── Parent delegates to ITS parent... up to Bootstrap
│       ├── Bootstrap: Can I load it? (core Java classes)
│       │   └── No → return to child
│       ├── Platform: Can I load it? (platform modules)
│       │   └── No → return to child
│       └── Application: Can I load it? (classpath)
│           └── Found! → Load and return
│
└── 3. If parent can't load → try to load it yourself (findClass)
    └── Not found? → throw ClassNotFoundException
```

```java
// Simplified implementation of parent delegation in java.lang.ClassLoader
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // Step 1: Check if already loaded
        Class<?> c = findLoadedClass(name);

        if (c == null) {
            try {
                // Step 2: Delegate to parent
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    // No parent → try bootstrap
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // Parent couldn't load it — that's expected
            }

            if (c == null) {
                // Step 3: Try to load it ourselves
                c = findClass(name);
            }
        }

        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

### Why Parent Delegation?

1. **Security** — Prevents malicious classes from replacing core Java classes (e.g., a fake `java.lang.String`)
2. **Uniqueness** — Ensures a class is loaded only once by the most authoritative loader
3. **Visibility** — Child loaders can see classes loaded by parents, but not vice versa

```java
// You CANNOT create a class named java.lang.String
// The Bootstrap class loader always loads it first
package java.lang;
public class String {
    // This will NEVER be used — Bootstrap loads the real String
    // Java 9+ also blocks this with module access checks
}
```

### Custom Class Loaders

Custom class loaders are used for **hot deployment**, **class isolation**, **encryption**, and **dynamic code loading**.

```java
import java.io.*;
import java.nio.file.*;

public class CustomClassLoader extends ClassLoader {

    private final String classDirectory;

    public CustomClassLoader(String classDirectory, ClassLoader parent) {
        super(parent);
        this.classDirectory = classDirectory;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            // Convert class name to file path
            String fileName = name.replace('.', File.separatorChar) + ".class";
            Path classFile = Path.of(classDirectory, fileName);

            // Read bytecode
            byte[] bytecode = Files.readAllBytes(classFile);

            // Define the class from bytecode
            return defineClass(name, bytecode, 0, bytecode.length);

        } catch (IOException e) {
            throw new ClassNotFoundException("Could not load " + name, e);
        }
    }
}

// Usage: Hot reloading
public class HotReloadDemo {
    public static void main(String[] args) throws Exception {
        while (true) {
            // Create a NEW class loader each time to reload the class
            CustomClassLoader loader = new CustomClassLoader("/path/to/classes", null);
            Class<?> clazz = loader.loadClass("com.example.Plugin");
            Object plugin = clazz.getDeclaredConstructor().newInstance();

            // Use reflection to invoke methods
            clazz.getMethod("execute").invoke(plugin);

            Thread.sleep(5000); // Wait, then reload with new loader
            // Old class loader and its classes become eligible for GC
        }
    }
}
```

### Use Cases for Custom Class Loaders

```
┌──────────────────────────────────────────────────────────────┐
│  Use Case              │  Description                        │
├──────────────────────────────────────────────────────────────┤
│  Hot Deployment        │  Reload classes without restarting   │
│  Class Isolation       │  Load different versions of same lib │
│  Encrypted Classes     │  Decrypt .class files before loading │
│  Network Loading       │  Load classes from remote servers    │
│  Plugin Systems        │  Load plugins dynamically at runtime │
│  Bytecode Generation   │  Define classes from generated bytes │
└──────────────────────────────────────────────────────────────┘
```

### Breaking Parent Delegation — Tomcat & OSGi

Some frameworks **intentionally break** parent delegation for class isolation:

```
Standard Delegation:         Tomcat's Model:
  Bootstrap                    Bootstrap
    ↑                            ↑
  Platform                     System
    ↑                            ↑
  Application                  Common
    ↑                          ↗    ↖
  Custom                   WebApp1  WebApp2
                          (each has own loader)
```

```java
// Tomcat's WebAppClassLoader inverts delegation:
// 1. Check local (WEB-INF/classes, WEB-INF/lib) FIRST
// 2. Only delegate to parent if not found locally
// This allows each webapp to have its own version of a library

// OSGi goes further — each bundle has its own class loader
// with explicit import/export declarations for packages
```

### Context Class Loader (Thread Context)

The **Thread Context ClassLoader** solves a limitation of parent delegation where parent loaders need to load classes visible only to child loaders (SPI pattern):

```java
public class ContextClassLoaderDemo {
    public static void main(String[] args) {
        // Get the context class loader (defaults to Application ClassLoader)
        ClassLoader contextCL = Thread.currentThread().getContextClassLoader();
        System.out.println("Context CL: " + contextCL);

        // JDBC uses context class loader for SPI driver loading
        // Bootstrap CL loads java.sql.DriverManager
        // But JDBC drivers are on the classpath (Application CL)
        // Context CL bridges this gap:
        //   DriverManager uses Thread.currentThread().getContextClassLoader()
        //   to find driver classes on the classpath

        // Set a custom context class loader
        ClassLoader custom = new CustomClassLoader("/plugins", contextCL);
        Thread.currentThread().setContextClassLoader(custom);

        // Now SPI-style lookups will use the custom loader
        // ServiceLoader uses the context class loader by default
    }
}
```

### Class Identity: ClassLoader + Fully Qualified Name

Two classes with the **same name** loaded by **different class loaders** are **different types**:

```java
CustomClassLoader loader1 = new CustomClassLoader("/path1", null);
CustomClassLoader loader2 = new CustomClassLoader("/path2", null);

Class<?> class1 = loader1.loadClass("com.example.Plugin");
Class<?> class2 = loader2.loadClass("com.example.Plugin");

System.out.println(class1 == class2);           // false — different classes!
System.out.println(class1.equals(class2));       // false

// This is why ClassCastException can occur even with same class name:
// Object from loader1 cannot be cast to type from loader2
```

### ⚠️ Common Mistake
Overriding `loadClass()` instead of `findClass()` in custom class loaders. You should override `findClass()` to preserve parent delegation. Override `loadClass()` only if you intentionally want to break delegation (like Tomcat does). Also, forgetting that class identity includes the class loader — two classes with the same name from different loaders are not assignment-compatible.

### ⚡ Performance Tip
Class loaders hold strong references to all classes they load, and each class holds a reference back to its class loader. When using custom class loaders for hot reloading, ensure the old class loader and ALL references to its classes are released — otherwise you get a **class loader leak**, which manifests as a Metaspace/PermGen `OutOfMemoryError` after repeated redeployments (common in Tomcat).

---

## 🎯 Interviewer Insight
Interviewers want to hear the three built-in loaders, parent delegation, and why it exists (security + uniqueness). Strong candidates explain when and why to break delegation (Tomcat, OSGi), the context class loader for SPI, and class identity rules. Be ready to explain `ClassCastException` due to class loader mismatch.

## ⚡ Follow-up Questions
1. Why does `String.class.getClassLoader()` return `null`?
2. How does Tomcat isolate web applications?
3. What is a class loader leak and how do you fix it?

## 📚 Related Topics
- [Class Loading Mechanism](./q02-class-loading-mechanism.md)
- [Metaspace](./q10-metaspace.md)
- [JVM Security](./q14-jvm-security.md)

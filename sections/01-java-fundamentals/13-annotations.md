# Q13: What Are Annotations and How Do They Work?

## 📋 Question
What are annotations in Java and how do they work? Explain built-in annotations, meta-annotations, custom annotations, and annotation processing.

## Difficulty: 🟠 Intermediate

---

## 💡 Answer

**Annotations** are a form of **metadata** that provide data about a program but are not part of the program's logic itself. Introduced in Java 5, annotations can be applied to classes, methods, fields, parameters, packages, and even other annotations. They serve three primary purposes:

1. **Compiler instructions**: `@Override`, `@SuppressWarnings`, `@Deprecated` — provide information to the compiler for checks and warnings.
2. **Build/deployment-time processing**: Annotation processors can generate code, XML files, or validate constraints during compilation (e.g., Lombok's `@Data`, MapStruct's `@Mapper`).
3. **Runtime processing**: Frameworks use reflection to read annotations at runtime and alter behavior (e.g., Spring's `@Autowired`, JPA's `@Entity`, JUnit's `@Test`).

### Built-in Annotations

| Annotation | Purpose |
|-----------|---------|
| `@Override` | Ensures the method overrides a superclass method — compile error if not |
| `@Deprecated` | Marks an element as deprecated — compiler warning on usage |
| `@SuppressWarnings` | Suppresses compiler warnings for the annotated element |
| `@FunctionalInterface` | Ensures an interface has exactly one abstract method |
| `@SafeVarargs` | Suppresses heap pollution warnings for varargs with generics |

### Meta-Annotations (Annotations on Annotations)

| Meta-Annotation | Purpose |
|-----------------|---------|
| `@Target` | Where the annotation can be applied (METHOD, FIELD, TYPE, PARAMETER, etc.) |
| `@Retention` | How long the annotation is retained (SOURCE, CLASS, RUNTIME) |
| `@Documented` | Include the annotation in Javadoc |
| `@Inherited` | Subclasses inherit the annotation from the parent class |
| `@Repeatable` | The annotation can be applied multiple times to the same element |

### Retention Policies

```
┌─────────────────────────────────────────────────────────────────────┐
│ RetentionPolicy.SOURCE                                              │
│   → Discarded by compiler. Not in .class file, not at runtime.     │
│   → Example: @Override, @SuppressWarnings                          │
│   → Use for: compile-time checks only                              │
├─────────────────────────────────────────────────────────────────────┤
│ RetentionPolicy.CLASS (default)                                     │
│   → Stored in .class file, but NOT available at runtime.           │
│   → Example: @NonNull (some static analysis tools)                 │
│   → Use for: bytecode analysis tools                               │
├─────────────────────────────────────────────────────────────────────┤
│ RetentionPolicy.RUNTIME                                             │
│   → Stored in .class file AND available via reflection at runtime. │
│   → Example: @Autowired, @Entity, @Test, @RestController           │
│   → Use for: framework-driven runtime behavior                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Code Example — Built-in Annotations

```java
package com.interview.fundamentals;

import java.util.ArrayList;
import java.util.List;

public class BuiltInAnnotationsDemo {

    // ✅ @Override — compiler verifies this actually overrides a superclass method
    @Override
    public String toString() {
        return "BuiltInAnnotationsDemo";
    }

    // ❌ Without @Override, a typo creates a NEW method silently
    // public String tostring() { ... }  // Oops! No compile error without @Override

    // ✅ @Deprecated — marks as deprecated with message and version info
    @Deprecated(since = "2.0", forRemoval = true)
    public void oldMethod() {
        System.out.println("Use newMethod() instead");
    }

    public void newMethod() {
        System.out.println("Modern implementation");
    }

    // ✅ @SuppressWarnings — suppresses specific compiler warnings
    @SuppressWarnings("unchecked")
    public void suppressedWarning() {
        List rawList = new ArrayList();  // Raw type warning suppressed
        rawList.add("test");
    }

    // ✅ @FunctionalInterface — ensures exactly ONE abstract method
    @FunctionalInterface
    interface Transformer<T, R> {
        R transform(T input);
        // Adding another abstract method → compile error
    }

    // ✅ @SafeVarargs — suppresses heap pollution warning
    @SafeVarargs
    static <T> List<T> asList(T... elements) {
        List<T> list = new ArrayList<>();
        for (T e : elements) list.add(e);
        return list;
    }
}
```

### Custom Annotations

```java
package com.interview.fundamentals;

import java.lang.annotation.*;
import java.lang.reflect.Method;

// === DEFINING A CUSTOM ANNOTATION ===

@Target(ElementType.METHOD)           // Can only be applied to methods
@Retention(RetentionPolicy.RUNTIME)   // Available at runtime via reflection
@Documented                            // Included in Javadoc
@interface RateLimit {
    int maxRequests() default 100;     // Annotation element with default value
    int windowSeconds() default 60;
    String message() default "Rate limit exceeded";
}

// === DEFINING A REPEATABLE ANNOTATION ===

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(Roles.class)              // Can be applied multiple times
@interface Role {
    String value();                    // Single element named "value" — shorthand syntax
}

// Container annotation for @Repeatable
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@interface Roles {
    Role[] value();
}

// === MARKER ANNOTATION (no elements) ===
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface Auditable { }

// === USING CUSTOM ANNOTATIONS ===

@Auditable
public class CustomAnnotationDemo {

    @RateLimit(maxRequests = 10, windowSeconds = 30, message = "Slow down!")
    public void searchApi() {
        System.out.println("Searching...");
    }

    @RateLimit  // Uses default values: 100 requests per 60 seconds
    public void listApi() {
        System.out.println("Listing...");
    }

    // ✅ Repeatable annotation — multiple @Role on one method
    @Role("ADMIN")
    @Role("MANAGER")
    public void deleteUser() {
        System.out.println("Deleting user...");
    }

    // === READING ANNOTATIONS AT RUNTIME VIA REFLECTION ===
    public static void main(String[] args) throws Exception {

        Class<?> clazz = CustomAnnotationDemo.class;

        // Check class-level annotation
        if (clazz.isAnnotationPresent(Auditable.class)) {
            System.out.println("Class is auditable ✅");
        }

        // Read method-level annotations
        for (Method method : clazz.getDeclaredMethods()) {
            // Process @RateLimit
            if (method.isAnnotationPresent(RateLimit.class)) {
                RateLimit rl = method.getAnnotation(RateLimit.class);
                System.out.printf("Method '%s': max=%d, window=%ds, msg='%s'%n",
                    method.getName(), rl.maxRequests(), rl.windowSeconds(), rl.message());
            }

            // Process @Role (repeatable)
            Role[] roles = method.getAnnotationsByType(Role.class);
            if (roles.length > 0) {
                System.out.printf("Method '%s' requires roles: ", method.getName());
                for (Role role : roles) {
                    System.out.print(role.value() + " ");
                }
                System.out.println();
            }
        }
    }
}
```

### Compile-Time Annotation Processing

```java
// Annotation processors run DURING compilation (javac plugin)
// They can generate source code, validate constraints, and produce errors

// Popular annotation processors:
// ─────────────────────────────────────────────
// Lombok          → @Data, @Builder, @Getter   → generates boilerplate
// MapStruct       → @Mapper                     → generates mapping code
// Dagger/Hilt     → @Inject, @Component         → generates DI code
// JPA Metamodel   → @Entity                     → generates metamodel classes
// AutoValue       → @AutoValue                  → generates value classes
// Immutables      → @Value.Immutable            → generates immutable classes

// Custom processor skeleton:
// @SupportedAnnotationTypes("com.example.MyAnnotation")
// @SupportedSourceVersion(SourceVersion.RELEASE_17)
// public class MyProcessor extends AbstractProcessor {
//     @Override
//     public boolean process(Set<? extends TypeElement> annotations,
//                           RoundEnvironment roundEnv) {
//         for (Element element : roundEnv.getElementsAnnotatedWith(MyAnnotation.class)) {
//             // Generate code, validate, or produce errors
//             processingEnv.getMessager().printMessage(
//                 Diagnostic.Kind.NOTE, "Processing: " + element);
//         }
//         return true;
//     }
// }
```

### Annotation Element Types

```java
// Annotation elements can be of these types ONLY:
// ─────────────────────────────────────────────
// 1. Primitive types (int, long, double, etc.)
// 2. String
// 3. Class<?>
// 4. Enum types
// 5. Another annotation type
// 6. Arrays of any of the above
//
// ❌ Cannot use: objects, collections, maps, custom classes

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@interface ComplexAnnotation {
    int count() default 0;                              // Primitive
    String name();                                       // String (required — no default)
    Class<?> handler() default Void.class;              // Class
    Season season() default Season.SPRING;              // Enum
    Deprecated deprecated() default @Deprecated;        // Another annotation
    String[] tags() default {};                          // Array of String
}
```

### ⚠️ Common Mistake
**Forgetting `@Retention(RUNTIME)` on custom annotations** used with reflection: The default retention is `CLASS`, which means the annotation is stored in the `.class` file but NOT available at runtime via `getAnnotation()`. If you intend to read the annotation at runtime (as most framework-style annotations do), you **must** specify `RetentionPolicy.RUNTIME`. Another mistake: not using `@Override` consistently — it catches typos in method names, incorrect parameter types, and methods that don't actually override anything (e.g., overriding a method removed in a library update).

### ⚡ Performance Tip
Reflection-based annotation reading (`getAnnotation()`, `isAnnotationPresent()`) has **overhead** — it involves native method calls and object creation. In performance-critical paths, **cache annotation metadata** during startup rather than reading it on every invocation. This is exactly what frameworks like Spring do: they scan annotations during application context initialization and cache the results. Compile-time annotation processors (Lombok, MapStruct) have **zero runtime cost** because all processing happens during compilation — the generated code runs at full speed with no reflection.

### 🔧 JVM Internals
Annotations with `RUNTIME` retention are stored in the class file's `RuntimeVisibleAnnotations` attribute. When you call `getAnnotation()`, the JVM parses this attribute and creates a **dynamic proxy** that implements the annotation interface — each element accessor method returns the stored value. Annotations with `CLASS` retention are stored in `RuntimeInvisibleAnnotations` and are available to bytecode analysis tools (like ASM, ByteBuddy) but not via the standard Reflection API. `SOURCE` retention annotations are discarded by `javac` and never appear in the class file. Annotation processing uses the `javax.annotation.processing` API and runs in **rounds** — each round may generate new source files that are then compiled and may trigger additional annotation processing rounds.

---

## 🎯 Interviewer Insight
Interviewers use this question to differentiate developers who merely use annotations from those who understand how they work. Strong candidates explain the three retention policies and their implications, can define a custom annotation with correct meta-annotations, and know how to read annotations via reflection. Mentioning compile-time annotation processing (and naming tools like Lombok or MapStruct) shows practical framework experience. Understanding that Spring's `@Autowired`, JPA's `@Entity`, and JUnit's `@Test` all rely on `RUNTIME` retention and reflection connects annotations to real-world frameworks.

## ⚡ Follow-up Questions
1. How does Spring use annotations like `@Component` and `@Autowired` internally?
2. What is the difference between compile-time annotation processing and runtime reflection?
3. How do repeatable annotations work under the hood?

## 📚 Related Topics
- [Records in Java 14+](./q14-records-java14.md)
- [Enums in Java](./q12-enums-in-java.md)
- [Java 8+ Features](../06-java8-plus-features/README.md)

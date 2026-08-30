# Q12: Repeating and Type Annotations

## 📋 Question
What are repeating annotations and type annotations in Java? Explain the @Repeatable meta-annotation, container annotations, and the TYPE_USE and TYPE_PARAMETER annotation targets.

## Difficulty: 🟡 Medium

---

## 💡 Answer

**Repeating annotations** (Java 8) allow the same annotation to be applied **multiple times** to the same declaration. **Type annotations** (Java 8) extend annotation targets to any **use of a type**, enabling tools to perform deeper static analysis like null-safety checking and taint analysis.

### Repeating Annotations

```java
package com.interview.java8;

import java.lang.annotation.*;

public class RepeatingAnnotationsDemo {

    // ═══════════════════════════════════════════════════
    // Step 1: Define the repeatable annotation with @Repeatable
    // ═══════════════════════════════════════════════════
    @Repeatable(Schedules.class)  // Points to container annotation
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    @interface Schedule {
        String dayOfWeek();
        String time();
    }

    // ═══════════════════════════════════════════════════
    // Step 2: Define the CONTAINER annotation (holds the array)
    // ═══════════════════════════════════════════════════
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    @interface Schedules {
        Schedule[] value();  // Must have a value() method returning array
    }

    // ═══════════════════════════════════════════════════
    // Step 3: Apply the annotation multiple times
    // ═══════════════════════════════════════════════════
    @Schedule(dayOfWeek = "Monday", time = "09:00")
    @Schedule(dayOfWeek = "Wednesday", time = "14:00")
    @Schedule(dayOfWeek = "Friday", time = "09:00")
    public void weeklyMeeting() {
        System.out.println("Meeting in progress...");
    }

    // Before Java 8, you had to use the container explicitly:
    // @Schedules({
    //     @Schedule(dayOfWeek = "Monday", time = "09:00"),
    //     @Schedule(dayOfWeek = "Wednesday", time = "14:00")
    // })

    public static void main(String[] args) throws Exception {
        var method = RepeatingAnnotationsDemo.class.getMethod("weeklyMeeting");

        // ✅ Reading repeating annotations
        Schedule[] schedules = method.getAnnotationsByType(Schedule.class);
        for (Schedule s : schedules) {
            System.out.println(s.dayOfWeek() + " at " + s.time());
        }
        // Monday at 09:00
        // Wednesday at 14:00
        // Friday at 09:00

        // ✅ Reading the container annotation
        Schedules container = method.getAnnotation(Schedules.class);
        System.out.println("Total schedules: " + container.value().length);  // 3
    }
}
```

### Real-World Repeating Annotation Example

```java
package com.interview.java8;

import java.lang.annotation.*;
import java.lang.reflect.Method;

public class PracticalRepeatingAnnotation {

    // ✅ Security role annotation — method requires multiple roles
    @Repeatable(Roles.class)
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.METHOD, ElementType.TYPE})
    @interface RequiresRole {
        String value();
    }

    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.METHOD, ElementType.TYPE})
    @interface Roles {
        RequiresRole[] value();
    }

    // ✅ Validation constraints
    @Repeatable(Constraints.class)
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.FIELD)
    @interface Constraint {
        String rule();
        String message() default "Validation failed";
    }

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.FIELD)
    @interface Constraints {
        Constraint[] value();
    }

    // ═══════════════════════════════════════════════════
    // Usage
    // ═══════════════════════════════════════════════════
    @RequiresRole("ADMIN")
    @RequiresRole("MANAGER")
    public void deleteUser(String userId) {
        System.out.println("Deleting user: " + userId);
    }

    static class UserForm {
        @Constraint(rule = "notEmpty", message = "Name is required")
        @Constraint(rule = "maxLength:50", message = "Name too long")
        @Constraint(rule = "pattern:[a-zA-Z ]+", message = "Invalid characters")
        String name;
    }

    public static void main(String[] args) throws Exception {
        Method method = PracticalRepeatingAnnotation.class.getMethod("deleteUser", String.class);
        RequiresRole[] roles = method.getAnnotationsByType(RequiresRole.class);
        System.out.println("Required roles:");
        for (RequiresRole role : roles) {
            System.out.println("  - " + role.value());
        }
    }
}
```

### Type Annotations (TYPE_USE, TYPE_PARAMETER)

```java
package com.interview.java8;

import java.lang.annotation.*;
import java.util.*;

public class TypeAnnotationsDemo {

    // ✅ Type annotation — can be applied wherever a TYPE is used
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE_USE)
    @interface NonNull {}

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE_USE)
    @interface Nullable {}

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE_USE)
    @interface Positive {}

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE_USE)
    @interface Immutable {}

    // ═══════════════════════════════════════════════════
    // TYPE_USE allows annotations in many more positions
    // ═══════════════════════════════════════════════════

    // On local variable types
    void method() {
        @NonNull String name = "Alice";
        @Nullable String middleName = null;
    }

    // On generic type arguments
    List<@NonNull String> names = new ArrayList<>();

    // On array components
    @NonNull String @Nullable [] array;  // Nullable array of NonNull strings

    // On return types
    @NonNull String getName() { return "Alice"; }

    // On exception types
    void risky() throws @Nullable Exception { }

    // On casts
    void castExample(Object obj) {
        String s = (@NonNull String) obj;
    }

    // On type parameters
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE_PARAMETER)
    @interface TypeConstraint {}

    class Container<@TypeConstraint T> {
        @NonNull T value;

        Container(@NonNull T value) {
            this.value = value;
        }
    }

    // On extends/implements clauses
    class OrderedList<T> extends @Immutable ArrayList<@NonNull T> {}
}
```

### Annotation Targets Summary

| Target | Where Applicable | Java Version |
|--------|-----------------|--------------|
| `TYPE` | Class, interface, enum, annotation, record | 1.5 |
| `FIELD` | Fields (including enum constants) | 1.5 |
| `METHOD` | Methods | 1.5 |
| `PARAMETER` | Method/constructor parameters | 1.5 |
| `CONSTRUCTOR` | Constructors | 1.5 |
| `LOCAL_VARIABLE` | Local variables | 1.5 |
| `ANNOTATION_TYPE` | Annotation type declarations | 1.5 |
| `PACKAGE` | Package declarations | 1.5 |
| `TYPE_PARAMETER` | Type parameter declarations | **8** |
| `TYPE_USE` | Any use of a type | **8** |
| `MODULE` | Module declarations | **9** |
| `RECORD_COMPONENT` | Record components | **16** |

### Annotation Processing

```java
package com.interview.java8;

import java.lang.annotation.*;
import java.lang.reflect.*;

public class AnnotationProcessingDemo {

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.TYPE_USE)
    @interface Validated {}

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    @interface Cacheable {
        int ttlSeconds() default 300;
    }

    @Cacheable(ttlSeconds = 60)
    public @Validated String fetchData() {
        return "data";
    }

    public static void main(String[] args) throws Exception {
        Method method = AnnotationProcessingDemo.class.getMethod("fetchData");

        // ✅ Check if annotation is present
        if (method.isAnnotationPresent(Cacheable.class)) {
            Cacheable cacheable = method.getAnnotation(Cacheable.class);
            System.out.println("Cache TTL: " + cacheable.ttlSeconds() + "s");
        }

        // ✅ Get all annotations on return type
        AnnotatedType returnType = method.getAnnotatedReturnType();
        for (Annotation a : returnType.getAnnotations()) {
            System.out.println("Return type annotation: " + a);
        }

        // ✅ Type annotations on generic types
        // AnnotatedParameterizedType for List<@NonNull String>
    }
}
```

### ⚠️ Common Mistake
**Forgetting the container annotation**: When creating a `@Repeatable` annotation, you **must** define a container annotation with a `value()` method that returns an array of the repeatable annotation. The container must have the same (or broader) `@Target` and `@Retention` as the repeatable annotation. Mismatched retention policies cause the repeating annotations to silently disappear at runtime — a debugging nightmare.

### ⚡ Performance Tip
Annotation processing via reflection (`getAnnotation()`, `getAnnotationsByType()`) has a cost — each call involves reading class metadata and creating annotation proxy objects. For hot paths, **cache annotation values** at startup rather than reading them on every invocation. Frameworks like Spring do this: they scan annotations once during context initialization and store the results in internal caches, avoiding repeated reflection overhead.

### 🔧 JVM Internals
Annotations are stored in the class file's **attributes section** (`RuntimeVisibleAnnotations`, `RuntimeInvisibleAnnotations`). Type annotations use a separate attribute `RuntimeVisibleTypeAnnotations` (added in Java 8 class file format). The `@Repeatable` mechanism is syntactic sugar — the compiler transforms multiple `@Schedule` annotations into a single `@Schedules` container annotation in the bytecode. At runtime, `getAnnotationsByType()` checks both the repeatable annotation directly and unwraps the container annotation transparently.

---

## 🎯 Interviewer Insight
Interviewers test whether you understand the **two-part setup** of repeating annotations (repeatable + container). For type annotations, the key insight is that `TYPE_USE` enables tools like **Checker Framework** and **NullAway** to perform null-safety analysis that catches bugs at compile time. Knowing the practical difference between `getAnnotation()` (returns container) and `getAnnotationsByType()` (unwraps repeats) demonstrates attention to API details.

## ⚡ Follow-up Questions
1. Can you use `@Repeatable` without defining a container annotation?
2. How do annotation processors work during compilation?
3. What's the difference between `getAnnotation` and `getAnnotationsByType`?

## 📚 Related Topics
- [Lambda Expressions](./q01-lambda-expressions.md)
- [Records](./q14-records-detail.md)
- [Sealed Classes](./q16-sealed-classes-detail.md)

# Q9: Explain Packages and Access Modifiers in Java

## 📋 Question
Explain packages and access modifiers in Java. How do they work together to control visibility and organize code?

## Difficulty: 🟢 Easy

---

## 💡 Answer

**Packages** are Java's mechanism for organizing classes into **namespaces**, preventing naming conflicts, and providing a logical structure for large codebases. They map directly to the directory structure on disk. Package names follow the **reverse domain name convention** (e.g., `com.company.project.module`) to ensure global uniqueness.

**Access modifiers** control the **visibility** of classes, interfaces, fields, methods, and constructors. Java has four access levels, from most restrictive to least: `private`, default (package-private, no keyword), `protected`, and `public`. These modifiers work hand-in-hand with packages to implement **encapsulation** — one of the four pillars of OOP.

### Package Conventions
- **Reverse domain**: `com.google.common.collect`, `org.apache.commons.lang3`
- **Lowercase only**: package names are always lowercase
- **No reserved words**: avoid `int`, `class`, etc. as package segments
- **Logical grouping**: group related classes (e.g., `model`, `service`, `repository`, `controller`)

### Import Statements
- **Single class import**: `import java.util.List;` (preferred — explicit)
- **Wildcard import**: `import java.util.*;` (imports all classes in package, not sub-packages)
- **Static import**: `import static java.lang.Math.PI;` or `import static java.util.Collections.*;`
- **`java.lang.*`** is automatically imported (String, Integer, System, etc.)

### Access Modifier Visibility Table

```
┌──────────────────────────────────────────────────────────────────────┐
│             Access Modifier Visibility Matrix                        │
├──────────────┬──────────┬──────────┬────────────┬────────────────────┤
│ Modifier     │ Class    │ Package  │ Subclass   │ World (everywhere) │
│              │ itself   │ (same)   │ (diff pkg) │                    │
├──────────────┼──────────┼──────────┼────────────┼────────────────────┤
│ public       │    ✅    │    ✅    │     ✅     │        ✅          │
│ protected    │    ✅    │    ✅    │     ✅     │        ❌          │
│ default (-)  │    ✅    │    ✅    │     ❌     │        ❌          │
│ private      │    ✅    │    ❌    │     ❌     │        ❌          │
└──────────────┴──────────┴──────────┴────────────┴────────────────────┘

Key:
  ✅ = Accessible
  ❌ = Not accessible
  default = "package-private" — no keyword is used
```

### Code Example

```java
// === File: com/interview/model/User.java ===
package com.interview.model;

public class User {

    // private — accessible ONLY within this class
    private long id;
    private String email;

    // default (package-private) — accessible within com.interview.model
    String internalNotes;

    // protected — accessible in this package + subclasses in other packages
    protected String displayName;

    // public — accessible everywhere
    public String username;

    // ✅ Constructor with access control
    public User(String username, String email) {
        this.username = username;
        this.email = email;
        this.id = System.nanoTime();
    }

    // ✅ Public getter — exposes private field in a controlled way
    public String getEmail() {
        return email;
    }

    // ✅ Private helper — internal implementation detail
    private void validateEmail(String email) {
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("Invalid email: " + email);
        }
    }

    // ✅ Protected method — available to subclasses
    protected void updateDisplayName(String name) {
        this.displayName = name;
    }

    // Default (package-private) method — available within the package only
    void setInternalNotes(String notes) {
        this.internalNotes = notes;
    }
}
```

```java
// === File: com/interview/model/AdminUser.java (SAME package) ===
package com.interview.model;

public class AdminUser extends User {

    public AdminUser(String username, String email) {
        super(username, email);
    }

    public void demonstrate() {
        // ✅ public — always accessible
        System.out.println(this.username);

        // ✅ protected — accessible (same package + subclass)
        this.updateDisplayName("Admin");

        // ✅ default — accessible (same package)
        this.setInternalNotes("admin notes");

        // ❌ private — NOT accessible (even in subclass)
        // System.out.println(this.id);      // Compile error
        // this.validateEmail("test");        // Compile error

        // ✅ Must use public accessor
        System.out.println(this.getEmail());
    }
}
```

```java
// === File: com/interview/service/UserService.java (DIFFERENT package) ===
package com.interview.service;

import com.interview.model.User;
// import com.interview.model.*;  // Wildcard import alternative

public class UserService {

    public void processUser(User user) {
        // ✅ public — accessible from any package
        System.out.println(user.username);
        System.out.println(user.getEmail());

        // ❌ protected — NOT accessible (different package, not a subclass)
        // user.updateDisplayName("test");  // Compile error

        // ❌ default — NOT accessible (different package)
        // user.setInternalNotes("test");   // Compile error

        // ❌ private — NOT accessible
        // user.validateEmail("test");       // Compile error
    }
}
```

### Static Imports

```java
package com.interview.fundamentals;

// ✅ Static imports — use static members without class prefix
import static java.lang.Math.PI;
import static java.lang.Math.sqrt;
import static java.lang.Math.pow;
import static java.util.Collections.unmodifiableList;

// ❌ Avoid wildcard static imports — reduces readability
// import static java.lang.Math.*;

import java.util.ArrayList;
import java.util.List;

public class StaticImportDemo {

    public static void main(String[] args) {
        // Without static import: Math.sqrt(Math.pow(3, 2) + Math.pow(4, 2))
        // With static import:
        double hypotenuse = sqrt(pow(3, 2) + pow(4, 2));
        System.out.println("Hypotenuse: " + hypotenuse);  // 5.0

        double area = PI * pow(5, 2);
        System.out.println("Circle area (r=5): " + area);  // ~78.54

        List<String> items = new ArrayList<>(List.of("a", "b"));
        List<String> readOnly = unmodifiableList(items);
    }
}
```

### Top-Level Class Access Rules

```java
// ✅ A .java file can have only ONE public top-level class
// The file name MUST match the public class name

// public class User { ... }         // File MUST be named User.java
// class InternalHelper { ... }       // ✅ Default access — same file is OK
// private class Secret { ... }       // ❌ Top-level classes cannot be private
// protected class Semi { ... }       // ❌ Top-level classes cannot be protected

// Only public and default (package-private) are valid for top-level classes
```

### ⚠️ Common Mistake
**Using `public` by default on everything**: Many developers make all classes, fields, and methods `public` out of habit or convenience. This violates encapsulation and creates tight coupling — changes to internal implementation details become breaking API changes. Follow the **principle of least privilege**: start with `private`, widen only when necessary. Fields should almost always be `private` with `public` getters/setters when external access is truly needed. Also, `protected` does NOT mean "subclass only" — it also grants access to **all classes in the same package**, which surprises many developers.

### ⚡ Performance Tip
Import statements have **zero runtime cost** — they are purely a compile-time convenience for the compiler to resolve short class names to fully qualified names. At the bytecode level, all class references use fully qualified names regardless of imports. Wildcard imports (`import java.util.*`) do NOT import more classes into memory — they simply tell the compiler to search the package for name resolution. However, wildcard imports can cause **ambiguity compile errors** if two packages contain a class with the same name (e.g., `java.util.List` vs `java.awt.List`), so specific imports are preferred.

### 🔧 JVM Internals
Access modifiers are enforced at **two levels**: the compiler checks access at compile time, and the JVM's **verifier** re-checks at class loading time. This means even if you bypass the compiler (e.g., by editing bytecode), the JVM will reject illegal access with `IllegalAccessError`. However, the **Reflection API** can bypass access checks using `setAccessible(true)` — this is used by frameworks like Spring and Hibernate to inject dependencies into private fields. Since Java 9, the **module system (JPMS)** adds a new layer of access control: even `public` classes are inaccessible from other modules unless explicitly `exports`-ed in `module-info.java`. This means packages can be "public within the module but hidden from the outside world" — a capability previously impossible.

---

## 🎯 Interviewer Insight
Interviewers test your understanding of encapsulation through access modifiers. They want you to explain the four levels and — critically — know that `protected` grants access to **both subclasses and the same package** (many candidates miss the package part). Being able to articulate why `private` should be the default choice and explaining the principle of least privilege shows design maturity. Mentioning Java 9 modules as an additional access layer demonstrates modern Java knowledge.

## ⚡ Follow-up Questions
1. What is the difference between `protected` access in Java vs C++?
2. Can a subclass in a different package access a `protected` member through a reference of the parent type (not `this` or `super`)?
3. How does the Java 9 module system add access control beyond packages?

## 📚 Related Topics
- [Static vs Instance](./q10-static-vs-instance.md)
- [Final Keyword](./q11-final-keyword.md)
- [Object-Oriented Programming](../02-oop-concepts/README.md)

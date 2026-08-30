# Q10: Sandboxing and the Module System

[Home](../../README.md) > [Java Security](./README.md) > Sandboxing & Modules

---

## Overview

With the deprecation of `SecurityManager` (JEP 411), the **Java Platform Module System (JPMS / Project Jigsaw)** has become the primary mechanism for enforcing encapsulation and access control at the code level. JPMS provides **strong encapsulation** that prevents unauthorized access to internal APIs at both compile time and runtime.

## Module System (JPMS) for Encapsulation

### Basic Module Declaration

```java
// module-info.java for a core library
module com.myapp.core {
    // Only these packages are accessible to other modules
    exports com.myapp.core.api;
    exports com.myapp.core.model;

    // Internal packages are NOT accessible from outside
    // com.myapp.core.internal  -- cannot be accessed
    // com.myapp.core.util      -- cannot be accessed
}
```

```java
// module-info.java for a web module that depends on core
module com.myapp.web {
    requires com.myapp.core;           // Compile and runtime dependency
    requires java.net.http;            // JDK module
    requires transitive java.logging;  // Passes dependency to consumers

    exports com.myapp.web.controller;
}
```

## Strong Encapsulation

```java
// Package structure:
// com.myapp.core/
//   +-- com.myapp.core.api/
//   |     +-- UserService.java          (exported - accessible)
//   +-- com.myapp.core.internal/
//   |     +-- UserRepository.java       (NOT exported - hidden)
//   +-- com.myapp.core.security/
//         +-- PasswordHasher.java       (NOT exported - hidden)

// ✅ Exported class - accessible from other modules
package com.myapp.core.api;

public class UserService {
    // Public API that other modules can use
    public User findUser(String id) {
        // Internally uses hidden classes
        return repository.findById(id);
    }
}

// ✅ Internal class - hidden from other modules
package com.myapp.core.internal;

// Even though this class is public, it cannot be accessed
// from outside the module because the package is not exported
public class UserRepository {
    public User findById(String id) {
        // Database access logic
        return null;
    }
}
```

```java
// What happens when you try to access internal classes:

// From another module (com.myapp.web):
// import com.myapp.core.internal.UserRepository;  // COMPILE ERROR!
// "Package com.myapp.core.internal is not accessible"

// Even via reflection at runtime:
try {
    Class<?> clazz = Class.forName("com.myapp.core.internal.UserRepository");
    clazz.getDeclaredConstructor().newInstance();
} catch (IllegalAccessException e) {
    // "module com.myapp.core does not export com.myapp.core.internal"
    System.err.println("Access denied: " + e.getMessage());
}
```

## exports and opens Directives

```java
module com.myapp.core {
    // Public API - accessible at compile time and runtime
    exports com.myapp.core.api;

    // Qualified export - only specific modules can access
    exports com.myapp.core.spi to com.myapp.plugins;

    // Open for reflection only (not compile-time access)
    // Needed for frameworks like Hibernate, Jackson, Spring
    opens com.myapp.core.model to com.fasterxml.jackson.databind;

    // Open for reflection to all modules
    opens com.myapp.core.dto;
}
```

### Directive Comparison

| Directive | Compile-Time Access | Runtime Access | Reflection |
|-----------|-------------------|----------------|------------|
| (not exported) | No | No | No |
| `exports pkg` | Yes | Yes | Public only |
| `exports pkg to M` | Yes (only M) | Yes (only M) | Public only (M) |
| `opens pkg` | No | No | Full (all modules) |
| `opens pkg to M` | No | No | Full (only M) |
| `open module` | N/A | N/A | Full (all packages) |

## Reflection Restrictions

```java
module com.myapp.core {
    exports com.myapp.core.api;
    // Internal packages are NOT opened for reflection
}

// From another module:
public class ReflectionDemo {

    // ❌ This will fail at runtime
    public void accessPrivateField() {
        try {
            Class<?> clazz = Class.forName("com.myapp.core.api.UserService");
            var field = clazz.getDeclaredField("repository");
            field.setAccessible(true);  // InaccessibleObjectException!
        } catch (Exception e) {
            // "Unable to make field private ... accessible:
            //  module com.myapp.core does not 'opens com.myapp.core.api'"
            System.err.println(e.getMessage());
        }
    }
}

// ✅ To allow framework reflection, open specific packages
module com.myapp.core {
    exports com.myapp.core.api;
    opens com.myapp.core.model to spring.core;  // Allow Spring reflection
    opens com.myapp.core.entity to hibernate.core;  // Allow Hibernate
}
```

## Module-Based Access Control

```java
// ✅ Service provider pattern for controlled extensibility
module com.myapp.core {
    exports com.myapp.core.api;

    // Declare a service interface
    uses com.myapp.core.api.PaymentProcessor;
}

module com.myapp.stripe {
    requires com.myapp.core;

    // Provide an implementation
    provides com.myapp.core.api.PaymentProcessor
        with com.myapp.stripe.StripePaymentProcessor;
}
```

```java
// Service interface (in com.myapp.core)
package com.myapp.core.api;

public interface PaymentProcessor {
    boolean processPayment(double amount, String currency);
    String getProviderName();
}

// Implementation (in com.myapp.stripe - internal class)
package com.myapp.stripe;

public class StripePaymentProcessor implements PaymentProcessor {
    @Override
    public boolean processPayment(double amount, String currency) {
        // Stripe API calls
        return true;
    }

    @Override
    public String getProviderName() { return "Stripe"; }
}

// Discovery and usage (in com.myapp.core or any requiring module)
import java.util.ServiceLoader;

public class PaymentService {
    public void processPayment(double amount) {
        // ✅ ServiceLoader discovers providers safely
        ServiceLoader<PaymentProcessor> processors =
            ServiceLoader.load(PaymentProcessor.class);

        for (PaymentProcessor processor : processors) {
            System.out.println("Found processor: " + processor.getProviderName());
            processor.processPayment(amount, "USD");
        }
    }
}
```

## Comparison: SecurityManager vs Module System

| Feature | SecurityManager (deprecated) | Module System (JPMS) |
|---------|------------------------------|---------------------|
| **Granularity** | Per-operation (file read, socket connect) | Per-package (export, open) |
| **Enforcement time** | Runtime only | Compile-time AND runtime |
| **Performance** | Stack walking on every check | Zero overhead after startup |
| **Configuration** | Policy files (error-prone) | module-info.java (type-safe) |
| **Scope** | JVM-wide | Per-module |
| **Reflection control** | Coarse (ReflectPermission) | Fine-grained (opens to specific modules) |
| **Maintenance** | High (every new API needed integration) | Low (module declarations are stable) |
| **Status** | Deprecated (JEP 411) | Active, recommended |

## Jigsaw Benefits for Security

```java
// 1. Reduced attack surface
//    Only exported packages are accessible - internal classes are hidden
//    even from reflection by default

// 2. Reliable configuration
//    Module dependencies are declared and checked at startup
module com.myapp.web {
    requires com.myapp.core;      // Fails at startup if module is missing
    requires java.sql;
}

// 3. Custom runtime images with jlink (minimal JRE)
// jlink --module-path $JAVA_HOME/jmods:mods \
//        --add-modules com.myapp.web \
//        --output custom-runtime
//
// Result: Minimal runtime containing ONLY the modules your app needs
// Reduces attack surface by removing unused JDK modules (e.g., java.rmi)

// 4. Integrity checking
//    JVM verifies module descriptors at startup
//    Split packages (same package in multiple modules) are detected and rejected
```

## Anti-Patterns and Best Practices

```java
// ❌ Anti-pattern: Using --add-opens to bypass encapsulation
// java --add-opens java.base/java.lang=ALL-UNNAMED MyApp
// This defeats the purpose of strong encapsulation!

// ✅ Best practice: Properly declare module dependencies
module com.myapp.web {
    requires com.myapp.core;
    opens com.myapp.web.dto to com.fasterxml.jackson.databind;
}

// ❌ Anti-pattern: Using 'open module' for everything
// open module com.myapp.core {  // Opens ALL packages to reflection
//     exports com.myapp.core.api;
// }

// ✅ Best practice: Open only specific packages to specific modules
module com.myapp.core {
    exports com.myapp.core.api;
    opens com.myapp.core.model to hibernate.core, spring.core;
}

// ❌ Anti-pattern: Exporting internal packages
// module com.myapp.core {
//     exports com.myapp.core.api;
//     exports com.myapp.core.internal;  // Defeats encapsulation!
// }

// ✅ Best practice: Export only public API packages
module com.myapp.core {
    exports com.myapp.core.api;      // Public API only
    // Internal packages remain hidden
}
```

## Migration Path from SecurityManager

```text
Step 1: Identify SecurityManager usage
   grep -r "SecurityManager\|checkPermission\|setSecurityManager" src/

Step 2: Replace runtime permission checks with module boundaries
   - File access control    --> module exports + OS permissions
   - Reflection control     --> module opens directives
   - Class loading control  --> module requires + ServiceLoader

Step 3: Use container-level isolation for operational security
   - Network restrictions   --> Container networking / firewall rules
   - File system access     --> Container volume mounts (read-only)
   - Process execution      --> Container capabilities (--cap-drop=ALL)

Step 4: Apply application-level authorization
   - Use Spring Security or custom authorization for business rules
   - Implement RBAC/ABAC at the service layer

Step 5: Test and remove SecurityManager references
   - Run with -Djava.security.manager=disallow (Java 18+)
   - Verify no SecurityManager dependency remains
```

## Interview Tips

| Point | Detail |
|-------|--------|
| **JPMS purpose** | Strong encapsulation at compile-time and runtime |
| **exports** | Makes a package accessible to other modules |
| **opens** | Allows deep reflection (for frameworks like Spring, Hibernate) |
| **vs SecurityManager** | Modules: compile-time + zero overhead; SM: runtime-only + slow |
| **jlink** | Creates minimal custom runtimes, reducing attack surface |
| **ServiceLoader** | Safe extensibility pattern for module-based plugin systems |
| **Migration** | Replace SM with modules + containers + app-level authorization |

---

[<< Previous: Q9 - Secure Coding Practices](./q09-secure-coding-practices.md) | [Back to Section Index](./README.md)

# Q2: How Do Permissions and Policies Work?

[Home](../../README.md) > [Java Security](./README.md) > Permissions & Policies

---

## Overview

Java's permission model uses **Permission** objects to represent access rights, **Policy** files to configure what code gets which permissions, and **ProtectionDomain** to associate permissions with code sources. While the SecurityManager is deprecated, understanding this model is essential for legacy systems.

## Core Permission Classes

```java
import java.security.*;
import java.io.FilePermission;
import java.net.SocketPermission;

public class PermissionExamples {

    public static void main(String[] args) {
        // File permissions
        Permission readFile = new FilePermission("/tmp/data.txt", "read");
        Permission writeFile = new FilePermission("/tmp/-", "read,write");  // Recursive
        Permission allFiles = new FilePermission("<<ALL FILES>>", "read");

        // Socket permissions
        Permission connectWeb = new SocketPermission("example.com:443", "connect");
        Permission anyPort = new SocketPermission("localhost:1024-", "listen,accept");

        // Runtime permissions
        Permission exitVM = new RuntimePermission("exitVM");
        Permission classLoader = new RuntimePermission("createClassLoader");
        Permission reflection = new RuntimePermission("accessDeclaredMembers");

        // Property permissions
        Permission readProp = new PropertyPermission("java.home", "read");
        Permission writeProp = new PropertyPermission("user.*", "read,write");

        // Check implication (does one permission include another?)
        FilePermission broad = new FilePermission("/tmp/-", "read,write");
        FilePermission narrow = new FilePermission("/tmp/data.txt", "read");
        System.out.println(broad.implies(narrow));  // true
    }
}
```

## Permission Hierarchy

```
java.security.Permission (abstract)
    |
    +-- java.security.BasicPermission
    |       +-- RuntimePermission
    |       +-- PropertyPermission
    |       +-- NetPermission
    |       +-- SecurityPermission
    |       +-- ReflectPermission
    |
    +-- java.io.FilePermission
    +-- java.net.SocketPermission
    +-- java.security.AllPermission
    +-- java.security.UnresolvedPermission
```

## Policy File Syntax

```
// my-app.policy

// Grant permissions to code from a specific location
grant codeBase "file:/opt/myapp/lib/*" {
    permission java.io.FilePermission "/opt/myapp/data/-", "read,write";
    permission java.net.SocketPermission "db-server:5432", "connect";
    permission java.util.PropertyPermission "app.*", "read";
};

// Grant permissions to signed code
grant signedBy "trustedVendor" {
    permission java.io.FilePermission "/tmp/-", "read,write,delete";
    permission java.lang.RuntimePermission "getenv.*";
};

// Grant permissions to code from a URL signed by a specific entity
grant codeBase "https://updates.example.com/*", signedBy "releaseTeam" {
    permission java.security.AllPermission;
};

// Grant to all code (use sparingly)
grant {
    permission java.util.PropertyPermission "java.version", "read";
    permission java.util.PropertyPermission "os.name", "read";
};
```

## Policy Configuration

```java
// Loading and using policies programmatically
public class PolicyDemo {

    public static void main(String[] args) {
        // Set policy file via system property
        System.setProperty("java.security.policy", "my-app.policy");

        // Refresh the policy
        Policy.getPolicy().refresh();

        // Inspect permissions for a CodeSource
        CodeSource cs = new CodeSource(
            new java.net.URL("file:/opt/myapp/lib/utils.jar"),
            (java.security.cert.Certificate[]) null
        );

        PermissionCollection perms = Policy.getPolicy().getPermissions(cs);

        // Enumerate granted permissions
        java.util.Enumeration<Permission> elements = perms.elements();
        while (elements.hasMoreElements()) {
            System.out.println("Granted: " + elements.nextElement());
        }
    }
}
```

## CodeSource and ProtectionDomain

```java
import java.security.*;
import java.net.URL;

public class ProtectionDomainDemo {

    public static void main(String[] args) throws Exception {
        // CodeSource identifies WHERE code came from
        URL location = new URL("file:/opt/myapp/app.jar");
        CodeSource codeSource = new CodeSource(location, (java.security.cert.Certificate[]) null);

        // ProtectionDomain binds CodeSource + Permissions
        PermissionCollection permissions = new Permissions();
        permissions.add(new FilePermission("/tmp/-", "read"));
        permissions.add(new PropertyPermission("app.*", "read"));

        ProtectionDomain domain = new ProtectionDomain(codeSource, permissions);

        // Check if domain has a specific permission
        Permission check = new FilePermission("/tmp/data.txt", "read");
        System.out.println("Has read access: " + domain.implies(check));  // true

        Permission denied = new FilePermission("/etc/passwd", "read");
        System.out.println("Has /etc access: " + domain.implies(denied)); // false
    }
}
```

## Custom Permission Class

```java
import java.security.BasicPermission;

// Custom permission for application-level authorization
public class OrderPermission extends BasicPermission {

    private String actions;

    public OrderPermission(String name) {
        super(name);
    }

    public OrderPermission(String name, String actions) {
        super(name);
        this.actions = actions;
    }

    @Override
    public String getActions() {
        return actions;
    }

    @Override
    public boolean implies(Permission permission) {
        if (!(permission instanceof OrderPermission)) return false;

        OrderPermission other = (OrderPermission) permission;

        // "admin" implies all order permissions
        if (this.getName().equals("admin")) return true;

        // Wildcard matching
        if (this.getName().endsWith("*")) {
            String prefix = this.getName().substring(0, this.getName().length() - 1);
            return other.getName().startsWith(prefix);
        }

        return this.getName().equals(other.getName());
    }
}

// Usage in policy file:
// grant {
//     permission com.myapp.OrderPermission "orders.view";
//     permission com.myapp.OrderPermission "orders.create";
// };
```

## Anti-Patterns and Best Practices

```java
public class PermissionPractices {

    // ❌ Anti-pattern: Granting AllPermission
    // grant { permission java.security.AllPermission; };
    // This defeats the entire purpose of the permission system

    // ❌ Anti-pattern: Overly broad file permissions
    // permission java.io.FilePermission "<<ALL FILES>>", "read,write,delete,execute";

    // ✅ Best practice: Principle of least privilege
    // permission java.io.FilePermission "/app/data/-", "read";
    // permission java.io.FilePermission "/app/logs/-", "read,write";

    // ❌ Anti-pattern: Catching and ignoring SecurityException
    public void unsafeAccess() {
        try {
            System.exit(0);
        } catch (SecurityException e) {
            // Silently ignoring - BAD!
        }
    }

    // ✅ Best practice: Handle SecurityException properly
    public void safeAccess() {
        try {
            Runtime.getRuntime().exec("cleanup.sh");
        } catch (SecurityException e) {
            logger.warn("Insufficient permissions for cleanup: {}", e.getMessage());
            // Fall back to a safe alternative
            performManualCleanup();
        }
    }
}
```

## Modern Alternatives to Policy-Based Permissions

```java
// ✅ Use application-level authorization instead of JVM policies
public class ModernAuthorizationService {

    private final Map<String, Set<String>> rolePermissions = Map.of(
        "ADMIN",  Set.of("read", "write", "delete", "admin"),
        "EDITOR", Set.of("read", "write"),
        "VIEWER", Set.of("read")
    );

    public boolean hasPermission(String role, String action) {
        return rolePermissions.getOrDefault(role, Set.of()).contains(action);
    }

    public void authorize(String role, String action) {
        if (!hasPermission(role, action)) {
            throw new SecurityException(
                "Role '" + role + "' lacks permission: " + action);
        }
    }
}

// ✅ Use module system for code-level access control
// module-info.java
// module com.myapp.data {
//     exports com.myapp.data.api;        // Public API only
//     opens com.myapp.data.model to hibernate.core;  // Targeted reflection
// }
```

## Quick Reference

| Concept | Purpose |
|---------|---------|
| `Permission` | Represents a specific access right |
| `PermissionCollection` | Group of permissions of the same type |
| `Permissions` | Heterogeneous collection of all permissions |
| `Policy` | Maps `CodeSource` to `PermissionCollection` |
| `CodeSource` | URL + certificates identifying code origin |
| `ProtectionDomain` | Binds `CodeSource` + `Permissions` to loaded classes |
| `implies()` | Checks if one permission encompasses another |

## Interview Tips

| Point | Detail |
|-------|--------|
| **Permission types** | FilePermission, SocketPermission, RuntimePermission, etc. |
| **Policy files** | `grant` blocks map code sources to permission sets |
| **ProtectionDomain** | Links code origin (CodeSource) to granted permissions |
| **implies()** | Hierarchical check - broad permissions encompass narrow ones |
| **Modern approach** | Use JPMS modules + app-level auth instead of JVM policies |

---

[<< Previous: Q1 - Security Manager](./q01-security-manager.md) | [Next: Q3 - Code Signing >>](./q03-code-signing.md)

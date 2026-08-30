# Q14: JVM Security Architecture

## 📋 Question
How does the JVM enforce security? Explain bytecode verification, the SecurityManager, the module system's security features, and the sandbox model.

## Difficulty: 🟡 Medium

---

## 💡 Answer

The JVM provides a **multi-layered security architecture** that protects against malicious code, unauthorized access, and system compromise. Security is enforced at class loading time (bytecode verification), runtime (access control), and through the module system (encapsulation).

### JVM Security Layers

```
┌═══════════════════════════════════════════════════════════════┐
║                  JVM SECURITY ARCHITECTURE                    ║
║                                                               ║
║  Layer 1: BYTECODE VERIFICATION (class loading time)          ║
║  ┌─────────────────────────────────────────────────────────┐  ║
║  │  • Structural validation of .class files                │  ║
║  │  • Type safety checks                                   │  ║
║  │  • Stack integrity verification                         │  ║
║  │  • Access control validation                            │  ║
║  └─────────────────────────────────────────────────────────┘  ║
║                                                               ║
║  Layer 2: CLASS LOADER SECURITY (runtime)                     ║
║  ┌─────────────────────────────────────────────────────────┐  ║
║  │  • Parent delegation prevents class spoofing             │  ║
║  │  • Namespace isolation between loaders                   │  ║
║  │  • Protection domains per code source                    │  ║
║  └─────────────────────────────────────────────────────────┘  ║
║                                                               ║
║  Layer 3: MODULE SYSTEM (Java 9+ — strong encapsulation)      ║
║  ┌─────────────────────────────────────────────────────────┐  ║
║  │  • Explicit exports/requires declarations                │  ║
║  │  • Deep reflection blocked by default                    │  ║
║  │  • Internal APIs inaccessible                            │  ║
║  └─────────────────────────────────────────────────────────┘  ║
║                                                               ║
║  Layer 4: SecurityManager (DEPRECATED Java 17, removed 24)    ║
║  ┌─────────────────────────────────────────────────────────┐  ║
║  │  • Runtime permission checks (file, network, etc.)       │  ║
║  │  • Policy-based access control                           │  ║
║  │  • ⚠️ Deprecated — do NOT use in new code                │  ║
║  └─────────────────────────────────────────────────────────┘  ║
╚═══════════════════════════════════════════════════════════════╝
```

### Layer 1: Bytecode Verification

The bytecode verifier runs during the **Linking → Verification** phase of class loading and ensures that the class file is safe to execute:

```
Bytecode Verification Checks:
┌───────────────────────────────────────────────────────────┐
│ 1. FILE FORMAT VALIDATION                                 │
│    ✓ Magic number is 0xCAFEBABE                           │
│    ✓ Class file version is supported                      │
│    ✓ Constant pool entries are well-formed                │
│                                                           │
│ 2. METADATA VALIDATION                                    │
│    ✓ Class has exactly one superclass (except Object)     │
│    ✓ Final classes are not extended                        │
│    ✓ Final methods are not overridden                      │
│    ✓ No circular class hierarchy                          │
│                                                           │
│ 3. BYTECODE VALIDATION                                    │
│    ✓ No stack overflow or underflow                       │
│    ✓ Local variable accesses are within bounds            │
│    ✓ Type safety: operations match operand types          │
│    ✓ Method arguments match parameter types               │
│    ✓ Object fields accessed after initialization          │
│    ✓ Jump targets are valid instruction boundaries        │
│    ✓ Return type matches method declaration               │
│                                                           │
│ 4. ACCESS CONTROL VALIDATION                              │
│    ✓ Private/protected/package access rules respected      │
│    ✓ Final fields not modified outside constructors       │
└───────────────────────────────────────────────────────────┘
```

```java
// What the verifier catches at class loading time:

// ❌ Type mismatch (caught by verifier)
// Bytecode tries to use int value as object reference
// → java.lang.VerifyError

// ❌ Stack manipulation violations
// Bytecode pops from empty operand stack
// → java.lang.VerifyError

// ❌ Illegal access
// Bytecode accesses private field of another class directly
// → java.lang.IllegalAccessError

// ❌ Incompatible class change
// Interface method called on non-implementing class
// → java.lang.IncompatibleClassChangeError
```

```java
// You can skip verification (NOT recommended):
// java -noverify MyApp          (Java 8)
// java -XX:-BytecodeVerification (deprecated)
// ⚠️ Disabling verification opens security vulnerabilities
//    and may crash the JVM with corrupted bytecode
```

### Layer 2: Class Loader Security

```java
// Parent delegation prevents malicious class replacement
// You CANNOT replace java.lang.String with your own:

// ❌ This will never be loaded by the JVM
package java.lang;
public class String {
    // Attacker's code — Bootstrap always loads real String first
}

// Why? Parent delegation:
// 1. Custom loader asks parent (App loader) to load java.lang.String
// 2. App loader asks parent (Platform loader)
// 3. Platform loader asks parent (Bootstrap loader)
// 4. Bootstrap finds the REAL java.lang.String → loads it
// 5. Custom loader never gets a chance to load its fake version

// Java 9+ module system adds additional protection:
// java.lang package is in java.base module
// Only the bootstrap loader can define classes in java.base
```

### Protection Domains

```java
// Every class has a ProtectionDomain that identifies:
// 1. WHERE the code came from (CodeSource — URL + certificates)
// 2. WHAT it's allowed to do (Permissions)

public class ProtectionDomainDemo {
    public static void main(String[] args) {
        // Check where this class was loaded from
        ProtectionDomain pd = ProtectionDomainDemo.class.getProtectionDomain();
        System.out.println("Code Source: " + pd.getCodeSource());
        System.out.println("Permissions: " + pd.getPermissions());
        System.out.println("Class Loader: " + pd.getClassLoader());

        // Core Java classes have AllPermission
        ProtectionDomain stringPD = String.class.getProtectionDomain();
        System.out.println("String permissions: " + stringPD.getPermissions());
    }
}
```

### Layer 3: Module System Security (Java 9+)

```java
// module-info.java — strong encapsulation
module com.myapp {
    requires java.base;            // Explicit dependency
    requires java.sql;             // Only accessible modules
    exports com.myapp.api;         // Only exported packages are accessible
    // com.myapp.internal is NOT accessible from outside
}

// Deep reflection is blocked by default:
// Field f = SomeClass.class.getDeclaredField("privateField");
// f.setAccessible(true);  // ❌ InaccessibleObjectException (if in a module)

// To allow reflective access, the module must explicitly open:
module com.myapp {
    opens com.myapp.model to com.fasterxml.jackson.databind; // Open to specific module
    opens com.myapp.entity to org.hibernate.core;            // For Hibernate reflection
}
```

```bash
# Common module system security messages:
# "module X does not export package Y to module Z"
# "Unable to make field accessible: module X does not opens package Y"

# Override with command-line flags (escape hatch):
--add-exports java.base/sun.nio.ch=ALL-UNNAMED
--add-opens java.base/java.lang=ALL-UNNAMED

# ⚠️ These flags weaken security — use only when necessary
# (e.g., legacy frameworks that need deep reflection)
```

### Layer 4: SecurityManager (Deprecated)

```java
// SecurityManager — runtime permission checks
// ⚠️ DEPRECATED in Java 17 (JEP 411)
// ⚠️ REMOVED for future versions
// Shown here for historical knowledge and legacy systems

// How it worked:
// java -Djava.security.manager -Djava.security.policy=my.policy MyApp

// my.policy file:
// grant codeBase "file:/app/lib/-" {
//     permission java.io.FilePermission "/data/-", "read,write";
//     permission java.net.SocketPermission "api.example.com:443", "connect";
// };

// In code, security checks looked like:
public void readFile(String path) {
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        sm.checkRead(path);  // Throws SecurityException if not allowed
    }
    // ... read file ...
}

// Why deprecated?
// 1. Rarely used correctly in practice
// 2. Huge maintenance burden on the JDK team
// 3. Performance overhead on every security-sensitive operation
// 4. Better alternatives exist (containers, OS-level security)
```

### Modern Security Best Practices

```java
// Instead of SecurityManager, use:

// 1. CONTAINER ISOLATION (Docker, Kubernetes)
//    - Process-level isolation
//    - Network policies
//    - Resource limits
//    - Read-only file systems

// 2. OS-LEVEL SECURITY
//    - Run JVM as non-root user
//    - SELinux / AppArmor policies
//    - Firewall rules

// 3. MODULE SYSTEM (Java 9+)
//    - Strong encapsulation of internal APIs
//    - Explicit dependency declarations
//    - Controlled reflective access

// 4. INPUT VALIDATION
public class SecureService {
    public User findUser(String id) {
        // ✅ Validate and sanitize inputs
        if (id == null || !id.matches("[a-zA-Z0-9-]+")) {
            throw new IllegalArgumentException("Invalid user ID");
        }
        return userRepository.findById(id);
    }
}

// 5. DEPENDENCY SECURITY
// Use tools like:
//   - OWASP Dependency-Check
//   - Snyk
//   - GitHub Dependabot
// To detect known vulnerabilities in dependencies
```

### Secure Coding with JVM Features

```java
// Use final and sealed classes to prevent malicious subclassing
public final class ImmutableConfig {
    private final String dbUrl;
    private final int timeout;

    public ImmutableConfig(String dbUrl, int timeout) {
        this.dbUrl = Objects.requireNonNull(dbUrl);
        this.timeout = timeout;
    }

    public String dbUrl() { return dbUrl; }
    public int timeout() { return timeout; }
}

// Use sealed classes (Java 17+) for controlled hierarchies
public sealed interface PaymentMethod
    permits CreditCard, BankTransfer, DigitalWallet {
    // Only these three implementations allowed
    // Prevents malicious implementations
}

// Use records for immutable data (Java 16+)
public record AuthToken(String token, Instant expiry) {
    // Automatically final, immutable fields
    // No way to create mutable subclass
}
```

### ⚠️ Common Mistake
Relying on the SecurityManager for application security. The SecurityManager is deprecated (Java 17) and will be removed. Modern Java security should rely on **module system encapsulation**, **container isolation**, **input validation**, and **OS-level security**. If you're maintaining a legacy system with SecurityManager, plan your migration path now.

### ⚡ Performance Tip
Bytecode verification adds ~10-15% to class loading time. For production systems where you trust the code source (your own builds), you can disable verification for specific packages. However, **never** disable verification for untrusted code. Java's AppCDS (Application Class-Data Sharing) pre-verifies classes in the shared archive, eliminating verification overhead on subsequent startups.

```bash
# Skip verification (ONLY for trusted code in performance-critical startups):
# ⚠️ NOT recommended for most applications
java -XX:-BytecodeVerificationRemote MyApp

# Better: Use AppCDS — classes are verified once and cached
java -XX:SharedArchiveFile=app-cds.jsa -jar myapp.jar
```

---

## 🎯 Interviewer Insight
Interviewers want to hear about bytecode verification, the class loader's parent delegation for security, and the module system's encapsulation. The fact that SecurityManager is deprecated is important — it shows you're current with JVM evolution. Strong candidates discuss modern security alternatives (containers, modules, OS-level security) rather than just the deprecated SecurityManager.

## ⚡ Follow-up Questions
1. What replaced SecurityManager in modern Java?
2. How does the module system prevent reflection attacks?
3. What is a VerifyError and when does it occur?

## 📚 Related Topics
- [Class Loaders](./q03-class-loaders.md)
- [Class Loading Mechanism](./q02-class-loading-mechanism.md)
- [Bytecode Understanding](./q08-bytecode-understanding.md)

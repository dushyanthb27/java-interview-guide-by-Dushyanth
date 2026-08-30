# Q9: What Are Secure Coding Best Practices?

[Home](../../README.md) > [Java Security](./README.md) > Secure Coding Practices

---

## Overview

Secure coding practices are defensive programming techniques that prevent vulnerabilities from being introduced into Java applications. These practices cover input validation, sensitive data handling, serialization security, immutability, and dependency management.

## Input Validation

```java
public class InputValidationPractices {

    // ❌ Anti-pattern: No validation on user input
    public void processOrder(String quantity, String price) {
        int qty = Integer.parseInt(quantity);     // NumberFormatException if malicious
        double prc = Double.parseDouble(price);   // No bounds checking
        createOrder(qty, prc);
    }

    // ✅ Best practice: Validate type, range, and format
    public void processOrderSafe(String quantity, String price) {
        // Validate quantity
        int qty;
        try {
            qty = Integer.parseInt(quantity);
        } catch (NumberFormatException e) {
            throw new IllegalArgumentException("Quantity must be a number");
        }
        if (qty < 1 || qty > 10_000) {
            throw new IllegalArgumentException("Quantity must be between 1 and 10000");
        }

        // Validate price
        double prc;
        try {
            prc = Double.parseDouble(price);
        } catch (NumberFormatException e) {
            throw new IllegalArgumentException("Price must be a number");
        }
        if (prc <= 0 || prc > 1_000_000) {
            throw new IllegalArgumentException("Price must be between 0 and 1000000");
        }

        createOrder(qty, prc);
    }

    // ✅ Best practice: Reject-first approach (whitelist validation)
    public String sanitizeUsername(String input) {
        if (input == null || input.isBlank()) {
            throw new IllegalArgumentException("Username cannot be empty");
        }
        if (input.length() > 50) {
            throw new IllegalArgumentException("Username too long");
        }
        if (!input.matches("^[a-zA-Z0-9_.-]+$")) {
            throw new IllegalArgumentException("Username contains invalid characters");
        }
        return input;
    }
}
```

## Output Encoding

```java
public class OutputEncodingPractices {

    // ❌ Anti-pattern: Direct interpolation in different contexts
    public String buildHtml(String userInput) {
        return "<div class=\"" + userInput + "\">" + userInput + "</div>";
    }

    // ✅ Best practice: Context-aware encoding
    public String buildHtmlSafe(String userInput) {
        String htmlEncoded = encodeForHtml(userInput);
        String attrEncoded = encodeForHtmlAttribute(userInput);
        return "<div class=\"" + attrEncoded + "\">" + htmlEncoded + "</div>";
    }

    // ✅ Encode for HTML body
    private String encodeForHtml(String input) {
        return input.replace("&", "&amp;")
                    .replace("<", "&lt;")
                    .replace(">", "&gt;")
                    .replace("\"", "&quot;")
                    .replace("'", "&#x27;");
    }

    // ✅ Encode for HTML attributes
    private String encodeForHtmlAttribute(String input) {
        StringBuilder sb = new StringBuilder();
        for (char c : input.toCharArray()) {
            if (Character.isLetterOrDigit(c)) {
                sb.append(c);
            } else {
                sb.append("&#x").append(Integer.toHexString(c)).append(";");
            }
        }
        return sb.toString();
    }

    // ✅ Recommended: Use OWASP Java Encoder library
    // Encode.forHtml(input);
    // Encode.forHtmlAttribute(input);
    // Encode.forJavaScript(input);
    // Encode.forCssString(input);
    // Encode.forUriComponent(input);
}
```

## Sensitive Data Handling: char[] vs String

```java
import java.util.Arrays;

public class SensitiveDataHandling {

    // ❌ Anti-pattern: Using String for passwords
    public boolean authenticateUnsafe(String password) {
        // String is immutable and stays in memory until GC
        // String is stored in the string pool - may persist indefinitely
        // Heap dumps and core dumps will contain the password
        return checkPassword(password);
    }

    // ✅ Best practice: Use char[] for passwords and clear after use
    public boolean authenticateSafe(char[] password) {
        try {
            return checkPassword(new String(password));  // Brief String use
        } finally {
            Arrays.fill(password, '\0');  // Overwrite immediately
        }
    }

    // ✅ Best practice: Sensitive configuration
    public class DatabaseConfig {
        private final char[] password;

        public DatabaseConfig(char[] password) {
            this.password = password.clone();  // Defensive copy
        }

        public void connect() {
            try {
                // Use password for connection
                establishConnection(new String(password));
            } finally {
                Arrays.fill(password, '\0');  // Clear after use
            }
        }
    }

    // ✅ Best practice: Use try-with-resources pattern for sensitive data
    public static class SensitiveString implements AutoCloseable {
        private char[] data;

        public SensitiveString(char[] data) {
            this.data = data.clone();
        }

        public char[] getData() {
            if (data == null) throw new IllegalStateException("Already cleared");
            return data;
        }

        @Override
        public void close() {
            if (data != null) {
                Arrays.fill(data, '\0');
                data = null;
            }
        }
    }

    // Usage:
    // try (SensitiveString pwd = new SensitiveString(passwordChars)) {
    //     authenticate(pwd.getData());
    // }  // Automatically cleared
}
```

## Least Privilege

```java
public class LeastPrivilegePractices {

    // ❌ Anti-pattern: Overly permissive API
    public class UserService {
        private final Connection dbConnection;  // Full DB access

        public void deleteUser(long userId) {
            // Any caller can delete any user - no checks
            executeSQL("DELETE FROM users WHERE id = " + userId);
        }
    }

    // ✅ Best practice: Restrict access based on role
    public class SecureUserService {
        private final Connection dbConnection;
        private final AuthorizationService authService;

        public void deleteUser(long userId, Principal caller) {
            // Verify caller has permission
            authService.requirePermission(caller, "user:delete");

            // Verify caller can act on this specific user
            if (!authService.canManageUser(caller, userId)) {
                throw new SecurityException("Cannot manage user: " + userId);
            }

            // Use parameterized query
            try (PreparedStatement ps = dbConnection.prepareStatement(
                    "DELETE FROM users WHERE id = ?")) {
                ps.setLong(1, userId);
                ps.executeUpdate();
            }
        }
    }

    // ✅ Best practice: Use minimal database permissions
    // CREATE USER app_readonly WITH PASSWORD 'xxx';
    // GRANT SELECT ON users, orders TO app_readonly;
    //
    // CREATE USER app_writer WITH PASSWORD 'yyy';
    // GRANT SELECT, INSERT, UPDATE ON users, orders TO app_writer;
    // -- No DELETE, no DROP, no schema changes
}
```

## Secure Defaults and Defense in Depth

```java
public class SecureDefaultsPractices {

    // ❌ Anti-pattern: Insecure default configuration
    public class InsecureServer {
        private boolean authRequired = false;    // Default: no auth
        private boolean tlsEnabled = false;       // Default: plaintext
        private int maxRequestSize = Integer.MAX_VALUE;  // No limit
    }

    // ✅ Best practice: Secure by default
    public class SecureServer {
        private boolean authRequired = true;         // Default: require auth
        private boolean tlsEnabled = true;            // Default: encrypted
        private int maxRequestSize = 10 * 1024 * 1024; // 10 MB limit
        private int maxHeaderSize = 8 * 1024;          // 8 KB limit
        private int requestTimeout = 30_000;           // 30 second timeout

        // Users must explicitly opt-out of security features
        public void disableAuth(String reason) {
            System.err.println("WARNING: Auth disabled - " + reason);
            this.authRequired = false;
        }
    }

    // ✅ Best practice: Defense in depth (multiple layers)
    public String processUserData(String input, Principal caller) {
        // Layer 1: Authentication
        if (caller == null) throw new SecurityException("Not authenticated");

        // Layer 2: Authorization
        if (!hasPermission(caller, "data:write")) {
            throw new SecurityException("Not authorized");
        }

        // Layer 3: Input validation
        if (input == null || input.length() > 1000) {
            throw new IllegalArgumentException("Invalid input length");
        }

        // Layer 4: Sanitization
        String sanitized = sanitize(input);

        // Layer 5: Parameterized storage
        saveToDatabase(sanitized);  // Using PreparedStatement internally

        // Layer 6: Audit logging
        auditLog("Data processed by " + caller.getName());

        return sanitized;
    }
}
```

## Serialization Security

```java
import java.io.*;

public class SerializationSecurity {

    // ❌ Anti-pattern: Deserializing untrusted data without filtering
    public Object deserializeUnsafe(byte[] data) throws Exception {
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(data));
        return ois.readObject();  // Can execute arbitrary code via gadget chains!
    }

    // ✅ Best practice: Use ObjectInputFilter (Java 9+)
    public Object deserializeSafe(byte[] data) throws Exception {
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(data));

        // Allow only specific classes
        ois.setObjectInputFilter(info -> {
            if (info.serialClass() != null) {
                String className = info.serialClass().getName();

                // Whitelist allowed classes
                if (className.startsWith("com.myapp.model.") ||
                    className.startsWith("java.lang.") ||
                    className.startsWith("java.util.")) {
                    return ObjectInputFilter.Status.ALLOWED;
                }
                return ObjectInputFilter.Status.REJECTED;
            }

            // Limit depth and array sizes
            if (info.depth() > 10) return ObjectInputFilter.Status.REJECTED;
            if (info.arrayLength() > 10_000) return ObjectInputFilter.Status.REJECTED;

            return ObjectInputFilter.Status.UNDECIDED;
        });

        return ois.readObject();
    }

    // ✅ Best practice: Prefer JSON/Protocol Buffers over Java serialization
    // ObjectMapper mapper = new ObjectMapper();
    // MyObject obj = mapper.readValue(jsonString, MyObject.class);

    // ✅ Best practice: Make classes non-serializable when not needed
    public class SecureService {
        // No implements Serializable - cannot be serialized

        // If serialization is forced by a parent class:
        private void readObject(ObjectInputStream ois) throws IOException {
            throw new InvalidObjectException("Deserialization not allowed");
        }
    }
}
```

## Immutability for Security

```java
import java.util.*;

public class ImmutabilitySecurity {

    // ❌ Anti-pattern: Mutable security-sensitive objects
    public class MutablePermissions {
        private List<String> roles;

        public MutablePermissions(List<String> roles) {
            this.roles = roles;  // Caller can modify the list externally!
        }

        public List<String> getRoles() {
            return roles;  // Caller can add roles after the fact!
        }
    }

    // ✅ Best practice: Immutable security-sensitive objects
    public class ImmutablePermissions {
        private final List<String> roles;

        public ImmutablePermissions(List<String> roles) {
            this.roles = List.copyOf(roles);  // Defensive copy + unmodifiable
        }

        public List<String> getRoles() {
            return roles;  // Already unmodifiable
        }
    }

    // ✅ Best practice: Use records for immutable data (Java 16+)
    public record UserCredentials(String username, Set<String> roles) {
        public UserCredentials {
            Objects.requireNonNull(username, "Username required");
            roles = Set.copyOf(roles);  // Defensive copy in compact constructor
        }
    }
}
```

## Dependency Scanning

```xml
<!-- ✅ Best practice: Use OWASP Dependency-Check in your build -->
<!-- pom.xml -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>9.0.9</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
    </configuration>
    <executions>
        <execution>
            <goals><goal>check</goal></goals>
        </execution>
    </executions>
</plugin>
```

```bash
# Run dependency vulnerability scan
mvn org.owasp:dependency-check-maven:check

# Gradle equivalent
# plugins { id 'org.owasp.dependencycheck' version '9.0.9' }
# gradle dependencyCheckAnalyze
```

## Secure Coding Checklist

```text
Input/Output:
  [x] Validate all input (type, length, range, format)
  [x] Use whitelist validation over blacklist
  [x] Encode output for the appropriate context
  [x] Use parameterized queries for all database operations

Data Protection:
  [x] Use char[] for passwords, clear after use
  [x] Use immutable objects for security-sensitive data
  [x] Minimize sensitive data in logs
  [x] Encrypt sensitive data at rest and in transit

Architecture:
  [x] Apply least privilege at all layers
  [x] Use secure defaults (opt-out, not opt-in)
  [x] Implement defense in depth
  [x] Use ObjectInputFilter for deserialization

Dependencies:
  [x] Scan for known vulnerabilities (OWASP Dependency-Check)
  [x] Keep dependencies updated
  [x] Remove unused dependencies
```

## Interview Tips

| Point | Detail |
|-------|--------|
| **Input validation** | Whitelist > blacklist; validate type, length, format, range |
| **Output encoding** | Context-specific: HTML body, HTML attribute, JS, CSS, URL |
| **char[] vs String** | char[] can be cleared; String is immutable and stays in memory |
| **Serialization** | Use ObjectInputFilter; prefer JSON over Java serialization |
| **Immutability** | Immutable objects prevent unauthorized modification |
| **Secure defaults** | Auth on, TLS on, limits set - require explicit opt-out |
| **Dependency scanning** | OWASP Dependency-Check catches known CVEs in libraries |

---

[<< Previous: Q8 - Preventing Injection](./q08-preventing-injection.md) | [Next: Q10 - Sandboxing & Modules >>](./q10-sandboxing-modules.md)

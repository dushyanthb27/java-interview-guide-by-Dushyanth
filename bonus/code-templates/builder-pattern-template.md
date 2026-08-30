# Builder Pattern Template

## Overview
Fluent API for constructing complex objects step-by-step, separating construction logic from representation.

## Use Cases
- Complex object construction
- Immutable object creation
- Configuration objects
- Fluent APIs
- Optional parameters
- Validation during build

## Implementation

### Simple Version
Basic builder with method chaining.

### Production Version
Advanced with validation and nested builders.

### Thread-Safe Version
Fully concurrent with proper synchronization.

## Code

```java
import java.util.*;
import java.util.concurrent.locks.ReentrantLock;

/**
 * SIMPLE VERSION: Basic Builder Pattern
 * - Simple fluent API
 * - Method chaining
 * - Immutable result
 */
public class SimpleBuilder {
    /**
     * Product class
     */
    public static class Person {
        private final String name;
        private final int age;
        private final String email;
        private final String phone;

        private Person(Builder builder) {
            this.name = builder.name;
            this.age = builder.age;
            this.email = builder.email;
            this.phone = builder.phone;
        }

        @Override
        public String toString() {
            return "Person{" +
                    "name='" + name + '\'' +
                    ", age=" + age +
                    ", email='" + email + '\'' +
                    ", phone='" + phone + '\'' +
                    '}';
        }

        // Getters
        public String getName() { return name; }
        public int getAge() { return age; }
        public String getEmail() { return email; }
        public String getPhone() { return phone; }
    }

    /**
     * Builder class with fluent API
     */
    public static class Builder {
        private String name;
        private int age;
        private String email;
        private String phone;

        public Builder name(String name) {
            this.name = name;
            return this;
        }

        public Builder age(int age) {
            this.age = age;
            return this;
        }

        public Builder email(String email) {
            this.email = email;
            return this;
        }

        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }

        /**
         * Build the Person object
         */
        public Person build() {
            return new Person(this);
        }
    }
}

/**
 * PRODUCTION VERSION: Advanced Builder with Validation
 * - Input validation
 * - Default values
 * - Nested builders
 * - Immutable collections
 * - Comprehensive error messages
 */
public class ProductionBuilder {
    /**
     * Product class with validation
     */
    public static class User {
        private final String username;
        private final String email;
        private final String password;
        private final Set<String> roles;
        private final Map<String, String> metadata;
        private final boolean active;
        private final long createdAt;

        private User(Builder builder) {
            this.username = builder.username;
            this.email = builder.email;
            this.password = builder.password;
            this.roles = Collections.unmodifiableSet(
                new HashSet<>(builder.roles));
            this.metadata = Collections.unmodifiableMap(
                new HashMap<>(builder.metadata));
            this.active = builder.active;
            this.createdAt = System.currentTimeMillis();
        }

        @Override
        public String toString() {
            return "User{" +
                    "username='" + username + '\'' +
                    ", email='" + email + '\'' +
                    ", roles=" + roles +
                    ", active=" + active +
                    '}';
        }

        // Getters
        public String getUsername() { return username; }
        public String getEmail() { return email; }
        public Set<String> getRoles() { return roles; }
        public Map<String, String> getMetadata() { return metadata; }
        public boolean isActive() { return active; }
        public long getCreatedAt() { return createdAt; }
    }

    /**
     * Advanced builder with validation
     */
    public static class Builder {
        private String username;
        private String email;
        private String password;
        private final Set<String> roles = new HashSet<>();
        private final Map<String, String> metadata = new HashMap<>();
        private boolean active = true;

        public Builder username(String username) {
            if (username == null || username.trim().isEmpty()) {
                throw new IllegalArgumentException("Username cannot be empty");
            }
            if (username.length() < 3) {
                throw new IllegalArgumentException(
                    "Username must be at least 3 characters");
            }
            this.username = username;
            return this;
        }

        public Builder email(String email) {
            if (email == null || !email.contains("@")) {
                throw new IllegalArgumentException("Invalid email format");
            }
            this.email = email;
            return this;
        }

        public Builder password(String password) {
            if (password == null || password.length() < 8) {
                throw new IllegalArgumentException(
                    "Password must be at least 8 characters");
            }
            this.password = password;
            return this;
        }

        public Builder addRole(String role) {
            if (role != null && !role.trim().isEmpty()) {
                this.roles.add(role);
            }
            return this;
        }

        public Builder addRoles(String... roles) {
            for (String role : roles) {
                addRole(role);
            }
            return this;
        }

        public Builder addMetadata(String key, String value) {
            if (key != null && value != null) {
                this.metadata.put(key, value);
            }
            return this;
        }

        public Builder active(boolean active) {
            this.active = active;
            return this;
        }

        /**
         * Build with validation
         */
        public User build() {
            // Validate required fields
            if (username == null) {
                throw new IllegalStateException("Username is required");
            }
            if (email == null) {
                throw new IllegalStateException("Email is required");
            }
            if (password == null) {
                throw new IllegalStateException("Password is required");
            }

            return new User(this);
        }

        /**
         * Build with default values
         */
        public User buildWithDefaults() {
            if (username == null) {
                username = "user_" + System.currentTimeMillis();
            }
            if (email == null) {
                email = username + "@example.com";
            }
            if (password == null) {
                password = UUID.randomUUID().toString();
            }
            return build();
        }
    }
}

/**
 * THREAD-SAFE VERSION: Concurrent Builder
 * - Thread-safe construction
 * - Immutable after build
 * - Reusable builder
 */
public class ThreadSafeBuilder {
    /**
     * Configuration object
     */
    public static class DatabaseConfig {
        private final String host;
        private final int port;
        private final String database;
        private final String username;
        private final String password;
        private final int poolSize;
        private final long connectionTimeout;
        private final boolean ssl;

        private DatabaseConfig(Builder builder) {
            this.host = builder.host;
            this.port = builder.port;
            this.database = builder.database;
            this.username = builder.username;
            this.password = builder.password;
            this.poolSize = builder.poolSize;
            this.connectionTimeout = builder.connectionTimeout;
            this.ssl = builder.ssl;
        }

        @Override
        public String toString() {
            return "DatabaseConfig{" +
                    "host='" + host + '\'' +
                    ", port=" + port +
                    ", database='" + database + '\'' +
                    ", poolSize=" + poolSize +
                    ", ssl=" + ssl +
                    '}';
        }

        // Getters
        public String getHost() { return host; }
        public int getPort() { return port; }
        public String getDatabase() { return database; }
        public String getUsername() { return username; }
        public String getPassword() { return password; }
        public int getPoolSize() { return poolSize; }
        public long getConnectionTimeout() { return connectionTimeout; }
        public boolean isSsl() { return ssl; }
    }

    /**
     * Thread-safe builder
     */
    public static class Builder {
        private final ReentrantLock lock = new ReentrantLock();
        private String host = "localhost";
        private int port = 5432;
        private String database;
        private String username;
        private String password;
        private int poolSize = 10;
        private long connectionTimeout = 30000;
        private boolean ssl = false;

        public Builder host(String host) {
            lock.lock();
            try {
                if (host == null || host.trim().isEmpty()) {
                    throw new IllegalArgumentException("Host cannot be empty");
                }
                this.host = host;
                return this;
            } finally {
                lock.unlock();
            }
        }

        public Builder port(int port) {
            lock.lock();
            try {
                if (port < 1 || port > 65535) {
                    throw new IllegalArgumentException(
                        "Port must be between 1 and 65535");
                }
                this.port = port;
                return this;
            } finally {
                lock.unlock();
            }
        }

        public Builder database(String database) {
            lock.lock();
            try {
                if (database == null || database.trim().isEmpty()) {
                    throw new IllegalArgumentException("Database cannot be empty");
                }
                this.database = database;
                return this;
            } finally {
                lock.unlock();
            }
        }

        public Builder username(String username) {
            lock.lock();
            try {
                if (username == null || username.trim().isEmpty()) {
                    throw new IllegalArgumentException("Username cannot be empty");
                }
                this.username = username;
                return this;
            } finally {
                lock.unlock();
            }
        }

        public Builder password(String password) {
            lock.lock();
            try {
                if (password == null || password.trim().isEmpty()) {
                    throw new IllegalArgumentException("Password cannot be empty");
                }
                this.password = password;
                return this;
            } finally {
                lock.unlock();
            }
        }

        public Builder poolSize(int poolSize) {
            lock.lock();
            try {
                if (poolSize < 1 || poolSize > 100) {
                    throw new IllegalArgumentException(
                        "Pool size must be between 1 and 100");
                }
                this.poolSize = poolSize;
                return this;
            } finally {
                lock.unlock();
            }
        }

        public Builder connectionTimeout(long timeoutMs) {
            lock.lock();
            try {
                if (timeoutMs < 1000) {
                    throw new IllegalArgumentException(
                        "Timeout must be at least 1000ms");
                }
                this.connectionTimeout = timeoutMs;
                return this;
            } finally {
                lock.unlock();
            }
        }

        public Builder ssl(boolean ssl) {
            lock.lock();
            try {
                this.ssl = ssl;
                return this;
            } finally {
                lock.unlock();
            }
        }

        /**
         * Build with validation
         */
        public DatabaseConfig build() {
            lock.lock();
            try {
                // Validate required fields
                if (database == null) {
                    throw new IllegalStateException("Database is required");
                }
                if (username == null) {
                    throw new IllegalStateException("Username is required");
                }
                if (password == null) {
                    throw new IllegalStateException("Password is required");
                }

                return new DatabaseConfig(this);
            } finally {
                lock.unlock();
            }
        }

        /**
         * Get current state (for debugging)
         */
        public String getState() {
            lock.lock();
            try {
                return "Builder{" +
                        "host='" + host + '\'' +
                        ", port=" + port +
                        ", database='" + database + '\'' +
                        ", poolSize=" + poolSize +
                        ", ssl=" + ssl +
                        '}';
            } finally {
                lock.unlock();
            }
        }
    }
}

/**
 * NESTED BUILDER PATTERN: For complex hierarchical objects
 */
public class NestedBuilder {
    /**
     * Complex configuration with nested objects
     */
    public static class ApplicationConfig {
        private final String appName;
        private final DatabaseConfig database;
        private final ServerConfig server;
        private final SecurityConfig security;

        private ApplicationConfig(Builder builder) {
            this.appName = builder.appName;
            this.database = builder.database;
            this.server = builder.server;
            this.security = builder.security;
        }

        public static class DatabaseConfig {
            public final String host;
            public final int port;
            public final String name;

            private DatabaseConfig(DatabaseBuilder builder) {
                this.host = builder.host;
                this.port = builder.port;
                this.name = builder.name;
            }

            public static class DatabaseBuilder {
                private String host = "localhost";
                private int port = 5432;
                private String name;

                public DatabaseBuilder host(String host) {
                    this.host = host;
                    return this;
                }

                public DatabaseBuilder port(int port) {
                    this.port = port;
                    return this;
                }

                public DatabaseBuilder name(String name) {
                    this.name = name;
                    return this;
                }

                public DatabaseConfig build() {
                    return new DatabaseConfig(this);
                }
            }
        }

        public static class ServerConfig {
            public final String host;
            public final int port;

            private ServerConfig(ServerBuilder builder) {
                this.host = builder.host;
                this.port = builder.port;
            }

            public static class ServerBuilder {
                private String host = "0.0.0.0";
                private int port = 8080;

                public ServerBuilder host(String host) {
                    this.host = host;
                    return this;
                }

                public ServerBuilder port(int port) {
                    this.port = port;
                    return this;
                }

                public ServerConfig build() {
                    return new ServerConfig(this);
                }
            }
        }

        public static class SecurityConfig {
            public final boolean enableSSL;
            public final String certificatePath;

            private SecurityConfig(SecurityBuilder builder) {
                this.enableSSL = builder.enableSSL;
                this.certificatePath = builder.certificatePath;
            }

            public static class SecurityBuilder {
                private boolean enableSSL = false;
                private String certificatePath;

                public SecurityBuilder enableSSL(boolean enable) {
                    this.enableSSL = enable;
                    return this;
                }

                public SecurityBuilder certificatePath(String path) {
                    this.certificatePath = path;
                    return this;
                }

                public SecurityConfig build() {
                    return new SecurityConfig(this);
                }
            }
        }

        @Override
        public String toString() {
            return "ApplicationConfig{" +
                    "appName='" + appName + '\'' +
                    ", database=" + database +
                    ", server=" + server +
                    ", security=" + security +
                    '}';
        }

        public static class Builder {
            private String appName;
            private DatabaseConfig database;
            private ServerConfig server;
            private SecurityConfig security;

            public Builder appName(String appName) {
                this.appName = appName;
                return this;
            }

            public Builder database(DatabaseConfig database) {
                this.database = database;
                return this;
            }

            public Builder server(ServerConfig server) {
                this.server = server;
                return this;
            }

            public Builder security(SecurityConfig security) {
                this.security = security;
                return this;
            }

            public ApplicationConfig build() {
                return new ApplicationConfig(this);
            }
        }
    }
}
```

## Testing

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

public class BuilderPatternTest {

    @Test
    public void testSimpleBuilder() {
        SimpleBuilder.Person person = new SimpleBuilder.Builder()
            .name("John Doe")
            .age(30)
            .email("john@example.com")
            .phone("555-1234")
            .build();

        assertEquals("John Doe", person.getName());
        assertEquals(30, person.getAge());
        assertEquals("john@example.com", person.getEmail());
    }

    @Test
    public void testProductionBuilderValidation() {
        assertThrows(IllegalArgumentException.class, () -> {
            new ProductionBuilder.Builder()
                .username("ab") // Too short
                .build();
        });

        assertThrows(IllegalArgumentException.class, () -> {
            new ProductionBuilder.Builder()
                .email("invalid-email") // Invalid format
                .build();
        });
    }

    @Test
    public void testProductionBuilderWithDefaults() {
        ProductionBuilder.User user = new ProductionBuilder.Builder()
            .buildWithDefaults();

        assertNotNull(user.getUsername());
        assertNotNull(user.getEmail());
        assertNotNull(user.getPassword());
    }

    @Test
    public void testProductionBuilderWithRoles() {
        ProductionBuilder.User user = new ProductionBuilder.Builder()
            .username("john_doe")
            .email("john@example.com")
            .password("SecurePassword123")
            .addRole("ADMIN")
            .addRoles("USER", "MODERATOR")
            .build();

        assertEquals(3, user.getRoles().size());
        assertTrue(user.getRoles().contains("ADMIN"));
    }

    @Test
    public void testThreadSafeBuilder() {
        ThreadSafeBuilder.Builder builder = new ThreadSafeBuilder.Builder()
            .host("db.example.com")
            .port(5432)
            .database("mydb")
            .username("admin")
            .password("secret123")
            .poolSize(20)
            .ssl(true);

        ThreadSafeBuilder.DatabaseConfig config = builder.build();

        assertEquals("db.example.com", config.getHost());
        assertEquals(5432, config.getPort());
        assertEquals(20, config.getPoolSize());
        assertTrue(config.isSsl());
    }

    @Test
    public void testNestedBuilder() {
        NestedBuilder.ApplicationConfig config = 
            new NestedBuilder.ApplicationConfig.Builder()
                .appName("MyApp")
                .database(new NestedBuilder.ApplicationConfig.DatabaseConfig
                    .DatabaseBuilder()
                    .host("localhost")
                    .port(5432)
                    .name("mydb")
                    .build())
                .server(new NestedBuilder.ApplicationConfig.ServerConfig
                    .ServerBuilder()
                    .host("0.0.0.0")
                    .port(8080)
                    .build())
                .security(new NestedBuilder.ApplicationConfig.SecurityConfig
                    .SecurityBuilder()
                    .enableSSL(true)
                    .certificatePath("/path/to/cert")
                    .build())
                .build();

        assertEquals("MyApp", config.appName);
        assertEquals("localhost", config.database.host);
        assertEquals(8080, config.server.port);
        assertTrue(config.security.enableSSL);
    }

    @Test
    public void testBuilderImmutability() {
        ProductionBuilder.User user = new ProductionBuilder.Builder()
            .username("john_doe")
            .email("john@example.com")
            .password("SecurePassword123")
            .addRole("ADMIN")
            .build();

        // Roles should be immutable
        assertThrows(UnsupportedOperationException.class, () -> {
            user.getRoles().add("USER");
        });
    }
}
```

## Performance Considerations

### Time Complexity
- **Each setter**: O(1)
- **build()**: O(n) where n = number of validations

### Space Complexity
- **O(n)** where n = number of fields

### Optimization Tips
1. **Lazy validation**: Validate only on build()
2. **Immutable collections**: Use Collections.unmodifiable*
3. **Thread-safe builders**: Use locks only when necessary
4. **Reusable builders**: Reset state for multiple builds
5. **Default values**: Reduce required parameters

## Common Pitfalls

1. **Mutable fields**: Builder should be mutable, product immutable
2. **No validation**: Invalid objects created
3. **Thread safety issues**: Concurrent builder access
4. **Memory leaks**: Holding references to large objects
5. **Circular dependencies**: In nested builders
6. **No error messages**: Validation errors unclear
7. **Inefficient copying**: Deep copying in build()

## Variations

1. **Telescoping Constructor**: Alternative to builder
2. **JavaBeans Pattern**: Setters with no-arg constructor
3. **Fluent Builder**: Method chaining for readability
4. **Immutable Builder**: Builder itself is immutable
5. **Copy Constructor**: Clone existing objects

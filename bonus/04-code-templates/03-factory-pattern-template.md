# Factory Pattern Template

## Overview
Object creation abstraction patterns that decouple object creation from usage, providing flexibility and maintainability.

## Use Cases
- Creating objects based on type/configuration
- Abstracting object creation logic
- Supporting multiple implementations
- Reducing coupling between classes
- Plugin systems
- Configuration-driven object creation

## Implementation

Multiple factory pattern variations.

## Code

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * SIMPLE VERSION: Simple Factory Pattern
 * - Static factory method
 * - Centralized object creation
 * - Not a true design pattern, but practical
 */
public class SimpleFactory {
    /**
     * Product interface
     */
    public interface Shape {
        void draw();
    }

    /**
     * Concrete implementations
     */
    public static class Circle implements Shape {
        @Override
        public void draw() {
            System.out.println("Drawing Circle");
        }
    }

    public static class Rectangle implements Shape {
        @Override
        public void draw() {
            System.out.println("Drawing Rectangle");
        }
    }

    public static class Triangle implements Shape {
        @Override
        public void draw() {
            System.out.println("Drawing Triangle");
        }
    }

    /**
     * Simple factory method
     */
    public static Shape createShape(String type) {
        return switch (type.toLowerCase()) {
            case "circle" -> new Circle();
            case "rectangle" -> new Rectangle();
            case "triangle" -> new Triangle();
            default -> throw new IllegalArgumentException("Unknown shape: " + type);
        };
    }
}

/**
 * PRODUCTION VERSION: Factory Method Pattern
 * - Abstract factory method in base class
 * - Concrete factories in subclasses
 * - Follows Open-Closed Principle
 */
public class FactoryMethodPattern {
    /**
     * Product interface
     */
    public interface Database {
        void connect();
        void disconnect();
        void query(String sql);
    }

    /**
     * Concrete products
     */
    public static class MySQLDatabase implements Database {
        @Override
        public void connect() {
            System.out.println("Connecting to MySQL");
        }

        @Override
        public void disconnect() {
            System.out.println("Disconnecting from MySQL");
        }

        @Override
        public void query(String sql) {
            System.out.println("Executing MySQL query: " + sql);
        }
    }

    public static class PostgreSQLDatabase implements Database {
        @Override
        public void connect() {
            System.out.println("Connecting to PostgreSQL");
        }

        @Override
        public void disconnect() {
            System.out.println("Disconnecting from PostgreSQL");
        }

        @Override
        public void query(String sql) {
            System.out.println("Executing PostgreSQL query: " + sql);
        }
    }

    public static class MongoDBDatabase implements Database {
        @Override
        public void connect() {
            System.out.println("Connecting to MongoDB");
        }

        @Override
        public void disconnect() {
            System.out.println("Disconnecting from MongoDB");
        }

        @Override
        public void query(String sql) {
            System.out.println("Executing MongoDB query: " + sql);
        }
    }

    /**
     * Abstract creator
     */
    public abstract static class DatabaseFactory {
        /**
         * Factory method - subclasses implement this
         */
        public abstract Database createDatabase();

        /**
         * Template method using factory method
         */
        public void initializeDatabase() {
            Database db = createDatabase();
            db.connect();
            db.query("SELECT * FROM users");
            db.disconnect();
        }
    }

    /**
     * Concrete factories
     */
    public static class MySQLFactory extends DatabaseFactory {
        @Override
        public Database createDatabase() {
            return new MySQLDatabase();
        }
    }

    public static class PostgreSQLFactory extends DatabaseFactory {
        @Override
        public Database createDatabase() {
            return new PostgreSQLDatabase();
        }
    }

    public static class MongoDBFactory extends DatabaseFactory {
        @Override
        public Database createDatabase() {
            return new MongoDBDatabase();
        }
    }
}

/**
 * ADVANCED VERSION: Abstract Factory Pattern
 * - Create families of related objects
 * - Ensure consistency between products
 * - Support multiple themes/configurations
 */
public class AbstractFactoryPattern {
    /**
     * Product families
     */
    public interface Button {
        void click();
    }

    public interface TextField {
        void setText(String text);
    }

    /**
     * Windows implementations
     */
    public static class WindowsButton implements Button {
        @Override
        public void click() {
            System.out.println("Windows Button clicked");
        }
    }

    public static class WindowsTextField implements TextField {
        @Override
        public void setText(String text) {
            System.out.println("Windows TextField: " + text);
        }
    }

    /**
     * Mac implementations
     */
    public static class MacButton implements Button {
        @Override
        public void click() {
            System.out.println("Mac Button clicked");
        }
    }

    public static class MacTextField implements TextField {
        @Override
        public void setText(String text) {
            System.out.println("Mac TextField: " + text);
        }
    }

    /**
     * Abstract factory
     */
    public interface UIFactory {
        Button createButton();
        TextField createTextField();
    }

    /**
     * Concrete factories
     */
    public static class WindowsUIFactory implements UIFactory {
        @Override
        public Button createButton() {
            return new WindowsButton();
        }

        @Override
        public TextField createTextField() {
            return new WindowsTextField();
        }
    }

    public static class MacUIFactory implements UIFactory {
        @Override
        public Button createButton() {
            return new MacButton();
        }

        @Override
        public TextField createTextField() {
            return new MacTextField();
        }
    }

    /**
     * Client code
     */
    public static class Application {
        private Button button;
        private TextField textField;

        public Application(UIFactory factory) {
            this.button = factory.createButton();
            this.textField = factory.createTextField();
        }

        public void render() {
            button.click();
            textField.setText("Hello World");
        }
    }
}

/**
 * REGISTRY FACTORY: Dynamic factory with registration
 * - Register factories at runtime
 * - Support for plugins
 * - Flexible and extensible
 */
public class RegistryFactory {
    /**
     * Product interface
     */
    public interface Logger {
        void log(String message);
    }

    /**
     * Concrete implementations
     */
    public static class ConsoleLogger implements Logger {
        @Override
        public void log(String message) {
            System.out.println("[CONSOLE] " + message);
        }
    }

    public static class FileLogger implements Logger {
        @Override
        public void log(String message) {
            System.out.println("[FILE] " + message);
        }
    }

    public static class DatabaseLogger implements Logger {
        @Override
        public void log(String message) {
            System.out.println("[DATABASE] " + message);
        }
    }

    /**
     * Factory with registry
     */
    public static class LoggerFactory {
        private static final Map<String, java.util.function.Supplier<Logger>> 
            registry = new ConcurrentHashMap<>();

        static {
            // Register default loggers
            register("console", ConsoleLogger::new);
            register("file", FileLogger::new);
            register("database", DatabaseLogger::new);
        }

        /**
         * Register a logger type
         */
        public static void register(String type, 
                                   java.util.function.Supplier<Logger> supplier) {
            registry.put(type.toLowerCase(), supplier);
        }

        /**
         * Create logger by type
         */
        public static Logger createLogger(String type) {
            java.util.function.Supplier<Logger> supplier = 
                registry.get(type.toLowerCase());
            if (supplier == null) {
                throw new IllegalArgumentException("Unknown logger type: " + type);
            }
            return supplier.get();
        }

        /**
         * Get all registered types
         */
        public static Set<String> getRegisteredTypes() {
            return new HashSet<>(registry.keySet());
        }
    }
}

/**
 * PARAMETERIZED FACTORY: Factory with configuration
 * - Pass configuration to factory
 * - Create customized objects
 * - Flexible object creation
 */
public class ParameterizedFactory {
    /**
     * Product interface
     */
    public interface DataSource {
        void connect();
        String getConnectionString();
    }

    /**
     * Concrete implementations
     */
    public static class JDBCDataSource implements DataSource {
        private final String host;
        private final int port;
        private final String database;

        public JDBCDataSource(String host, int port, String database) {
            this.host = host;
            this.port = port;
            this.database = database;
        }

        @Override
        public void connect() {
            System.out.println("Connecting to JDBC: " + getConnectionString());
        }

        @Override
        public String getConnectionString() {
            return "jdbc:mysql://" + host + ":" + port + "/" + database;
        }
    }

    public static class MongoDataSource implements DataSource {
        private final String host;
        private final int port;
        private final String database;

        public MongoDataSource(String host, int port, String database) {
            this.host = host;
            this.port = port;
            this.database = database;
        }

        @Override
        public void connect() {
            System.out.println("Connecting to MongoDB: " + getConnectionString());
        }

        @Override
        public String getConnectionString() {
            return "mongodb://" + host + ":" + port + "/" + database;
        }
    }

    /**
     * Configuration for factory
     */
    public static class DataSourceConfig {
        public final String type;
        public final String host;
        public final int port;
        public final String database;

        public DataSourceConfig(String type, String host, int port, 
                               String database) {
            this.type = type;
            this.host = host;
            this.port = port;
            this.database = database;
        }
    }

    /**
     * Parameterized factory
     */
    public static class DataSourceFactory {
        public static DataSource createDataSource(DataSourceConfig config) {
            return switch (config.type.toLowerCase()) {
                case "jdbc" -> new JDBCDataSource(config.host, config.port, 
                                                 config.database);
                case "mongo" -> new MongoDataSource(config.host, config.port, 
                                                   config.database);
                default -> throw new IllegalArgumentException(
                    "Unknown datasource type: " + config.type);
            };
        }
    }
}

/**
 * PROTOTYPE FACTORY: Clone-based factory
 * - Create objects by cloning prototypes
 * - Avoid expensive object creation
 * - Support for object customization
 */
public class PrototypeFactory {
    /**
     * Cloneable product
     */
    public static class Document implements Cloneable {
        private String title;
        private String content;
        private String author;

        public Document(String title, String content, String author) {
            this.title = title;
            this.content = content;
            this.author = author;
        }

        @Override
        public Document clone() {
            try {
                return (Document) super.clone();
            } catch (CloneNotSupportedException e) {
                throw new RuntimeException(e);
            }
        }

        public void setTitle(String title) {
            this.title = title;
        }

        public void setContent(String content) {
            this.content = content;
        }

        @Override
        public String toString() {
            return "Document{" +
                    "title='" + title + '\'' +
                    ", content='" + content + '\'' +
                    ", author='" + author + '\'' +
                    '}';
        }
    }

    /**
     * Prototype factory
     */
    public static class DocumentFactory {
        private final Map<String, Document> prototypes = new HashMap<>();

        public void registerPrototype(String name, Document prototype) {
            prototypes.put(name, prototype);
        }

        public Document createDocument(String prototypeName) {
            Document prototype = prototypes.get(prototypeName);
            if (prototype == null) {
                throw new IllegalArgumentException(
                    "Unknown prototype: " + prototypeName);
            }
            return prototype.clone();
        }
    }
}

/**
 * BUILDER FACTORY: Combine builder with factory
 * - Complex object creation
 * - Fluent configuration
 * - Validation during creation
 */
public class BuilderFactory {
    /**
     * Complex product
     */
    public static class HttpClient {
        private final String baseUrl;
        private final int timeout;
        private final boolean followRedirects;
        private final Map<String, String> defaultHeaders;

        private HttpClient(Builder builder) {
            this.baseUrl = builder.baseUrl;
            this.timeout = builder.timeout;
            this.followRedirects = builder.followRedirects;
            this.defaultHeaders = new HashMap<>(builder.defaultHeaders);
        }

        public void get(String path) {
            System.out.println("GET " + baseUrl + path);
        }

        @Override
        public String toString() {
            return "HttpClient{" +
                    "baseUrl='" + baseUrl + '\'' +
                    ", timeout=" + timeout +
                    ", followRedirects=" + followRedirects +
                    '}';
        }

        /**
         * Builder for HttpClient
         */
        public static class Builder {
            private String baseUrl;
            private int timeout = 30000;
            private boolean followRedirects = true;
            private final Map<String, String> defaultHeaders = new HashMap<>();

            public Builder baseUrl(String baseUrl) {
                this.baseUrl = baseUrl;
                return this;
            }

            public Builder timeout(int timeout) {
                this.timeout = timeout;
                return this;
            }

            public Builder followRedirects(boolean follow) {
                this.followRedirects = follow;
                return this;
            }

            public Builder addHeader(String key, String value) {
                defaultHeaders.put(key, value);
                return this;
            }

            public HttpClient build() {
                if (baseUrl == null || baseUrl.isEmpty()) {
                    throw new IllegalStateException("Base URL is required");
                }
                return new HttpClient(this);
            }
        }
    }

    /**
     * Factory using builder
     */
    public static class HttpClientFactory {
        public static HttpClient createDefaultClient(String baseUrl) {
            return new HttpClient.Builder()
                .baseUrl(baseUrl)
                .timeout(30000)
                .followRedirects(true)
                .addHeader("User-Agent", "MyApp/1.0")
                .build();
        }

        public static HttpClient createCustomClient(String baseUrl, 
                                                    int timeout,
                                                    boolean followRedirects) {
            return new HttpClient.Builder()
                .baseUrl(baseUrl)
                .timeout(timeout)
                .followRedirects(followRedirects)
                .build();
        }
    }
}
```

## Testing

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

public class FactoryPatternTest {

    @Test
    public void testSimpleFactory() {
        SimpleFactory.Shape circle = SimpleFactory.createShape("circle");
        SimpleFactory.Shape rectangle = SimpleFactory.createShape("rectangle");

        assertNotNull(circle);
        assertNotNull(rectangle);
        assertNotEquals(circle.getClass(), rectangle.getClass());
    }

    @Test
    public void testFactoryMethodPattern() {
        FactoryMethodPattern.DatabaseFactory mysqlFactory = 
            new FactoryMethodPattern.MySQLFactory();
        FactoryMethodPattern.DatabaseFactory postgresFactory = 
            new FactoryMethodPattern.PostgreSQLFactory();

        FactoryMethodPattern.Database mysql = mysqlFactory.createDatabase();
        FactoryMethodPattern.Database postgres = postgresFactory.createDatabase();

        assertNotNull(mysql);
        assertNotNull(postgres);
        assertNotEquals(mysql.getClass(), postgres.getClass());
    }

    @Test
    public void testAbstractFactory() {
        AbstractFactoryPattern.UIFactory windowsFactory = 
            new AbstractFactoryPattern.WindowsUIFactory();
        AbstractFactoryPattern.UIFactory macFactory = 
            new AbstractFactoryPattern.MacUIFactory();

        AbstractFactoryPattern.Application windowsApp = 
            new AbstractFactoryPattern.Application(windowsFactory);
        AbstractFactoryPattern.Application macApp = 
            new AbstractFactoryPattern.Application(macFactory);

        assertNotNull(windowsApp);
        assertNotNull(macApp);
    }

    @Test
    public void testRegistryFactory() {
        RegistryFactory.Logger console = 
            RegistryFactory.LoggerFactory.createLogger("console");
        RegistryFactory.Logger file = 
            RegistryFactory.LoggerFactory.createLogger("file");

        assertNotNull(console);
        assertNotNull(file);
        assertNotEquals(console.getClass(), file.getClass());
    }

    @Test
    public void testRegistryFactoryRegistration() {
        RegistryFactory.LoggerFactory.register("custom", 
            () -> new RegistryFactory.ConsoleLogger());

        assertTrue(RegistryFactory.LoggerFactory.getRegisteredTypes()
            .contains("custom"));
    }

    @Test
    public void testParameterizedFactory() {
        ParameterizedFactory.DataSourceConfig mysqlConfig = 
            new ParameterizedFactory.DataSourceConfig(
                "jdbc", "localhost", 3306, "mydb");
        ParameterizedFactory.DataSourceConfig mongoConfig = 
            new ParameterizedFactory.DataSourceConfig(
                "mongo", "localhost", 27017, "mydb");

        ParameterizedFactory.DataSource mysql = 
            ParameterizedFactory.DataSourceFactory.createDataSource(mysqlConfig);
        ParameterizedFactory.DataSource mongo = 
            ParameterizedFactory.DataSourceFactory.createDataSource(mongoConfig);

        assertTrue(mysql.getConnectionString().contains("mysql"));
        assertTrue(mongo.getConnectionString().contains("mongodb"));
    }

    @Test
    public void testPrototypeFactory() {
        PrototypeFactory.DocumentFactory factory = 
            new PrototypeFactory.DocumentFactory();

        PrototypeFactory.Document prototype = 
            new PrototypeFactory.Document("Template", "Content", "Author");
        factory.registerPrototype("template", prototype);

        PrototypeFactory.Document doc1 = 
            factory.createDocument("template");
        PrototypeFactory.Document doc2 = 
            factory.createDocument("template");

        assertNotSame(doc1, doc2);
        assertEquals(doc1.toString(), doc2.toString());
    }

    @Test
    public void testBuilderFactory() {
        BuilderFactory.HttpClient client = 
            BuilderFactory.HttpClientFactory.createDefaultClient(
                "https://api.example.com");

        assertNotNull(client);
        client.get("/users");
    }

    @Test
    public void testBuilderFactoryCustom() {
        BuilderFactory.HttpClient client = 
            BuilderFactory.HttpClientFactory.createCustomClient(
                "https://api.example.com", 60000, false);

        assertNotNull(client);
    }
}
```

## Performance Considerations

### Time Complexity
- **Simple Factory**: O(1) with switch/if-else
- **Registry Factory**: O(1) with HashMap lookup
- **Prototype Factory**: O(n) where n = object size (cloning)

### Space Complexity
- **O(1)** for most factories
- **O(n)** for registry factory (n = number of registered types)

### Optimization Tips
1. **Cache created objects**: Reuse instances when possible
2. **Lazy initialization**: Create objects only when needed
3. **Object pooling**: Reuse expensive objects
4. **Prototype cloning**: Faster than creating new objects
5. **Registry caching**: Cache factory lookups

## Common Pitfalls

1. **Overusing factories**: Not every object needs a factory
2. **Tight coupling**: Factory still coupled to concrete classes
3. **Memory leaks**: Registry holding references indefinitely
4. **Thread safety**: Not synchronizing registry access
5. **Clone issues**: Shallow copy instead of deep copy
6. **Configuration errors**: Invalid parameters to factory
7. **No validation**: Creating invalid objects

## Variations

1. **Static Factory Methods**: Simple and practical
2. **Factory Interface**: Implement factory as interface
3. **Generic Factory**: Type-safe factory with generics
4. **Lazy Factory**: Defer object creation
5. **Caching Factory**: Cache created objects
6. **Composite Factory**: Combine multiple factories

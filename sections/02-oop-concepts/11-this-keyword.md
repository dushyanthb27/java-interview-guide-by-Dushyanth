# This Keyword

[Home](../../README.md) > [Object-Oriented Programming](./README.md) > This Keyword

---

## Q14 🟢 Explain the This Keyword

**Question** — What is the `this` keyword in Java? Explain its uses for constructor chaining, field disambiguation, method chaining (builder pattern), and how it works in inner classes.

### Answer

The `this` keyword is a reference to the **current object instance**. It cannot be used in static contexts because static members belong to the class, not to any specific instance.

**Field Disambiguation**

When a parameter or local variable has the same name as a field, `this` distinguishes the field from the parameter.

```java
public class Employee {
    private String name;
    private double salary;

    public Employee(String name, double salary) {
        // 'this.name' refers to the field, 'name' refers to the parameter
        this.name = name;
        this.salary = salary;
    }

    public void setSalary(double salary) {
        if (salary < 0) throw new IllegalArgumentException("Negative salary");
        this.salary = salary; // this.salary = field, salary = parameter
    }
}
```

**Constructor Chaining with `this()`**

Use `this()` to call another constructor in the same class. This must be the first statement in the constructor body, and it avoids code duplication by having one "primary" constructor that others delegate to.

```java
public class DatabaseConfig {
    private final String host;
    private final int port;
    private final String database;
    private final int poolSize;
    private final int timeout;

    // Primary constructor
    public DatabaseConfig(String host, int port, String database, int poolSize, int timeout) {
        this.host = host;
        this.port = port;
        this.database = database;
        this.poolSize = poolSize;
        this.timeout = timeout;
    }

    // Chain to primary constructor with defaults
    public DatabaseConfig(String host, int port, String database) {
        this(host, port, database, 10, 30000); // Default pool size and timeout
    }

    public DatabaseConfig(String host, String database) {
        this(host, 5432, database); // Default port
    }

    public DatabaseConfig(String host) {
        this(host, "default_db"); // Default database
    }
}
```

**Returning `this` for Method Chaining (Fluent API / Builder)**

Return `this` from methods to enable fluent, readable APIs where multiple method calls can be chained together.

```java
public class QueryBuilder {
    private String table;
    private final List<String> conditions = new ArrayList<>();
    private final List<String> columns = new ArrayList<>();
    private String orderBy;
    private int limit = -1;

    // Each method returns 'this' for chaining
    public QueryBuilder select(String... cols) {
        this.columns.addAll(Arrays.asList(cols));
        return this;
    }

    public QueryBuilder from(String table) {
        this.table = table;
        return this;
    }

    public QueryBuilder where(String condition) {
        this.conditions.add(condition);
        return this;
    }

    public QueryBuilder orderBy(String column) {
        this.orderBy = column;
        return this;
    }

    public QueryBuilder limit(int limit) {
        this.limit = limit;
        return this;
    }

    public String build() {
        var sb = new StringBuilder("SELECT ");
        sb.append(columns.isEmpty() ? "*" : String.join(", ", columns));
        sb.append(" FROM ").append(table);
        if (!conditions.isEmpty()) {
            sb.append(" WHERE ").append(String.join(" AND ", conditions));
        }
        if (orderBy != null) sb.append(" ORDER BY ").append(orderBy);
        if (limit > 0) sb.append(" LIMIT ").append(limit);
        return sb.toString();
    }
}

// Usage - fluent, readable API
String query = new QueryBuilder()
    .select("name", "email", "age")
    .from("users")
    .where("age > 18")
    .where("active = true")
    .orderBy("name")
    .limit(10)
    .build();
// SELECT name, email, age FROM users WHERE age > 18 AND active = true ORDER BY name LIMIT 10
```

**Builder Pattern with `this`**

The Builder pattern uses a static inner builder class where each configuration method returns `this` (the builder instance) for chaining, and finally calls the target class constructor passing `this` (the builder).

```java
public class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final String body;
    private final int timeout;

    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = Map.copyOf(builder.headers);
        this.body = builder.body;
        this.timeout = builder.timeout;
    }

    // Static inner builder class
    public static class Builder {
        private final String url;
        private String method = "GET";
        private final Map<String, String> headers = new HashMap<>();
        private String body;
        private int timeout = 30000;

        public Builder(String url) {
            this.url = url;
        }

        public Builder method(String method) {
            this.method = method;
            return this; // Returns builder for chaining
        }

        public Builder header(String key, String value) {
            this.headers.put(key, value);
            return this;
        }

        public Builder body(String body) {
            this.body = body;
            return this;
        }

        public Builder timeout(int timeout) {
            this.timeout = timeout;
            return this;
        }

        public HttpRequest build() {
            return new HttpRequest(this); // 'this' passed as builder reference
        }
    }
}

// Usage
var request = new HttpRequest.Builder("https://api.example.com/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body("{\"name\": \"Alice\"}")
    .timeout(5000)
    .build();
```

**Passing `this` as Method Argument**

Pass the current object as a parameter when registering listeners, callbacks, or when an object needs to reference itself.

```java
public class EventSource {
    private final List<EventListener> listeners = new ArrayList<>();

    public void addListener(EventListener listener) {
        listeners.add(listener);
    }

    public void fireEvent(String event) {
        listeners.forEach(l -> l.onEvent(this, event));
    }
}

public interface EventListener {
    void onEvent(EventSource source, String event);
}

public class Dashboard implements EventListener {
    public Dashboard(EventSource source) {
        source.addListener(this); // Pass current object as listener
    }

    @Override
    public void onEvent(EventSource source, String event) {
        System.out.println("Dashboard received: " + event);
    }
}
```

**`this` in Inner Classes**

In non-static inner classes, `this` refers to the inner class instance. Use `OuterClass.this` to refer to the outer class instance.

```java
public class Outer {
    private String name = "Outer";

    public class Inner {
        private String name = "Inner";

        public void printNames() {
            System.out.println(this.name);        // "Inner" - Inner's this
            System.out.println(Outer.this.name);   // "Outer" - Outer's this
        }
    }
}
```

> **⚠️ Common Mistake** — Using `this` in a static context (compile error), leaking `this` in a constructor before the object is fully initialized (can cause subtle bugs with partially-constructed objects being visible to other threads).

```java
// Using 'this' in a static context
public class MyClass {
    private int value;

    public static void staticMethod() {
        // this.value = 10;  // COMPILE ERROR: 'this' cannot be used in static context
    }
}

// Leaking 'this' in constructor (object not fully initialized)
public class Unsafe {
    private final int value;

    public Unsafe(EventSource source) {
        source.addListener(this); // 'this' escapes before construction is complete
        this.value = 42;          // Listener might see value = 0 (default)
    }
}

// Fix: use a factory method
public class Safe {
    private final int value;

    private Safe(int value) {
        this.value = value;
    }

    public static Safe create(int value, EventSource source) {
        Safe instance = new Safe(value);
        source.addListener(instance); // Object is fully constructed
        return instance;
    }
}
```

> **🔧 Performance Tip** — The `this` reference is essentially free at runtime — it is implicitly passed as the first argument to every instance method by the JVM. Method chaining with `return this` has no overhead beyond the return value.

> **🎯 Interviewer Insight** — Interviewers check understanding of `this` vs `this()`, the builder pattern with method chaining, and the subtle bug of leaking `this` during construction. The `Outer.this` syntax for inner class disambiguation is also a common question.

> **⚡ Follow-up** — Can `this` be assigned to another variable?

> **⚡ Follow-up** — What is the difference between `this()` and `super()` in constructors?

> **⚡ Follow-up** — How does `this` work in a lambda expression vs an anonymous class?

> **📚 Related** — [Super Keyword](./10-super-keyword.md), [Constructors](./02-constructors.md), [Inner Classes](./13-inner-classes.md)

---

[← Previous: Super Keyword](./10-super-keyword.md) · [Next: Object Composition →](./12-object-composition.md)


### ⚠️ Gotchas
- **Common Misconception**: Ensure you distinguish compile-time behavior from runtime execution in production environments.
- **Null Safety & Edge Cases**: Watch out for implicit null unboxing and edge boundary conditions when scaling under high concurrency.
- **Performance Pitfall**: Inappropriate usage of default APIs without profiling can lead to hidden bottlenecks and memory overhead.

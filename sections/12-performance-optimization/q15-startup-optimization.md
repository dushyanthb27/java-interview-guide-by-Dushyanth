# Application Startup Optimization

[Home](../../README.md) > [Performance Optimization](./README.md) > Startup Optimization

---

## Overview

Fast startup is critical for serverless functions, containerized microservices, and developer productivity. Java's traditional startup penalty comes from class loading, JIT compilation, and framework initialization. Modern techniques like CDS, AOT compilation, and GraalVM native images can reduce startup from seconds to milliseconds.

---

## 1. Class Data Sharing (CDS / AppCDS)

CDS pre-processes class metadata into a shared archive that can be memory-mapped at startup, skipping class loading and verification.

```bash
# Step 1: Create class list (run the app and dump loaded classes)
java -XX:DumpLoadedClassList=classes.lst \
     -jar app.jar

# Step 2: Create shared archive from class list
java -Xshare:dump \
     -XX:SharedClassListFile=classes.lst \
     -XX:SharedArchiveFile=app-cds.jsa \
     -jar app.jar

# Step 3: Use shared archive at startup
java -Xshare:on \
     -XX:SharedArchiveFile=app-cds.jsa \
     -jar app.jar
```

### Default CDS (Java 12+)

```bash
# Java 12+ automatically includes a default CDS archive for JDK classes
# Just use -Xshare:on (usually enabled by default)
java -Xshare:on -jar app.jar

# Verify CDS is active
java -Xlog:class+load -jar app.jar 2>&1 | grep "shared objects file"
```

### Dynamic CDS (Java 13+)

```bash
# Automatically create archive after first run
java -XX:ArchiveClassesAtExit=dynamic-cds.jsa -jar app.jar

# Use the dynamic archive on next run
java -XX:SharedArchiveFile=dynamic-cds.jsa -jar app.jar
```

### CDS Startup Improvement

```
Without CDS:      ~2.5 seconds
With default CDS:  ~2.0 seconds (20% faster)
With AppCDS:       ~1.5 seconds (40% faster)

CDS benefits:
- Shared memory across JVM instances (containers)
- Faster class loading (memory-mapped, pre-verified)
- Reduced memory footprint (shared pages)
```

---

## 2. Ahead-of-Time Compilation (GraalVM Native Image)

GraalVM native image compiles Java bytecode to a native executable at build time, eliminating JVM startup overhead entirely.

```bash
# Install GraalVM and native-image
# https://www.graalvm.org/

# Build native image from JAR
native-image -jar app.jar -o app-native

# Run native executable (no JVM needed!)
./app-native
```

### Spring Boot with GraalVM Native Image

```xml
<!-- pom.xml: Spring Boot 3.x native support -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.graalvm.buildtools</groupId>
            <artifactId>native-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

```bash
# Build Spring Boot native image
mvn -Pnative native:compile

# Build native image in Docker (no local GraalVM needed)
mvn -Pnative spring-boot:build-image

# Run the native executable
./target/app-native
```

### JVM vs Native Image Comparison

| Metric              | JVM (OpenJDK)    | GraalVM Native    | Improvement      |
|---------------------|------------------|-------------------|------------------|
| Startup time        | ~2-5 seconds     | ~50-100 ms        | 20-50x faster    |
| Memory (RSS)        | ~200-400 MB      | ~40-80 MB         | 3-5x less        |
| Peak throughput     | Highest (JIT)    | ~80-90% of JIT    | JVM wins         |
| Build time          | ~10 seconds      | ~3-10 minutes     | JVM wins         |
| Reflection support  | Full             | Requires config   | JVM wins         |

---

## 3. Lazy Initialization

```java
// Eager initialization: everything loaded at startup
@Configuration
public class EagerConfig {
    @Bean
    public ExpensiveService expensiveService() {
        return new ExpensiveService(); // created at startup even if unused
    }
}

// Lazy initialization: defer until first use
@Configuration
public class LazyConfig {

    @Lazy
    @Bean
    public ExpensiveService expensiveService() {
        return new ExpensiveService(); // created on first request
    }
}

// Global lazy initialization in Spring Boot
// application.properties:
// spring.main.lazy-initialization=true
```

```java
// Lazy initialization patterns in plain Java
public class LazyHolder {
    // Initialization-on-demand holder idiom (thread-safe, no synchronization)
    private static class Holder {
        static final ExpensiveObject INSTANCE = new ExpensiveObject();
    }

    public static ExpensiveObject getInstance() {
        return Holder.INSTANCE; // class loaded only when getInstance() called
    }
}

// Supplier-based lazy initialization
public class LazyValue<T> {
    private final Supplier<T> supplier;
    private volatile T value;

    public LazyValue(Supplier<T> supplier) {
        this.supplier = supplier;
    }

    public T get() {
        if (value == null) {
            synchronized (this) {
                if (value == null) {
                    value = supplier.get();
                }
            }
        }
        return value;
    }
}

// Usage
LazyValue<DatabaseConnection> dbConn =
    new LazyValue<>(() -> new DatabaseConnection("jdbc:postgresql://..."));
// Connection created only when dbConn.get() is first called
```

---

## 4. Module System Benefits (JPMS)

```java
// module-info.java
module com.myapp {
    requires java.base;        // only what you need
    requires java.sql;
    requires spring.boot;

    exports com.myapp.api;     // explicit API surface
}
```

```bash
# Custom runtime image with only needed modules
jlink --module-path $JAVA_HOME/jmods:target/modules \
      --add-modules com.myapp \
      --output custom-jre \
      --strip-debug \
      --compress=2 \
      --no-header-files \
      --no-man-pages

# Result: custom JRE with only required modules
# Full JRE: ~300 MB
# Custom JRE: ~30-50 MB (depending on modules)

# Run with custom JRE
./custom-jre/bin/java -m com.myapp/com.myapp.Main
```

### Module System Startup Benefits

```
Full JRE (all modules):     ~2.5 seconds startup
Custom JRE (minimal modules): ~1.8 seconds startup (28% faster)

Benefits:
- Fewer classes to load
- Smaller memory footprint
- Faster class resolution
- Smaller Docker images
```

---

## 5. Startup Time Profiling

```bash
# Measure startup time components
java -Xlog:class+load:file=classload.log -jar app.jar
# Analyze: how many classes loaded, which are slow

# JFR startup profiling
java -XX:StartFlightRecording=name=startup,duration=30s,filename=startup.jfr \
     -jar app.jar

# Print class loading statistics
java -verbose:class -jar app.jar 2>&1 | wc -l
# Typical Spring Boot app: 10,000-20,000 classes loaded

# Time specific startup phases
java -Xlog:class+init=info -jar app.jar
```

```java
// Programmatic startup timing
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        long start = System.nanoTime();
        SpringApplication app = new SpringApplication(Application.class);
        app.addListeners((ApplicationReadyEvent event) -> {
            long elapsed = System.nanoTime() - start;
            System.out.printf("Application started in %.2f seconds%n",
                elapsed / 1_000_000_000.0);
        });
        app.run(args);
    }
}

// Spring Boot startup actuator (built-in)
// application.properties:
// management.endpoint.startup.enabled=true
// spring.application.admin.enabled=true

// GET /actuator/startup -> detailed startup step breakdown
```

---

## 6. Spring Boot Startup Optimization

```properties
# application.properties optimizations

# Enable lazy initialization (defer bean creation)
spring.main.lazy-initialization=true

# Disable unnecessary auto-configuration
spring.autoconfigure.exclude=\
  org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
  org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration,\
  org.springframework.boot.autoconfigure.jms.JmsAutoConfiguration

# Use index for component scanning (pre-computed at build time)
# Add spring-context-indexer dependency
```

```xml
<!-- Component index: pre-compute component scan at build time -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-indexer</artifactId>
    <optional>true</optional>
</dependency>
<!-- Generates META-INF/spring.components at compile time -->
<!-- Eliminates classpath scanning at startup -->
```

```java
// Optimize Spring Boot startup programmatically
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);

        // Disable banner
        app.setBannerMode(Banner.Mode.OFF);

        // Reduce web server startup
        app.setWebApplicationType(WebApplicationType.REACTIVE); // if applicable

        app.run(args);
    }
}

// Use @Profile to conditionally load beans
@Configuration
@Profile("!test")  // skip in test environment
public class ProductionOnlyConfig {
    @Bean
    public MetricsExporter metricsExporter() {
        return new MetricsExporter(); // only loaded in production
    }
}
```

### Spring Boot Startup Optimization Checklist

| Optimization                    | Impact   | Effort | Notes                        |
|--------------------------------|----------|--------|------------------------------|
| `lazy-initialization=true`    | High     | Low    | May delay first request      |
| Exclude unused auto-configs   | Medium   | Low    | Review auto-config report    |
| Spring Context Indexer        | Medium   | Low    | Build-time component index   |
| Reduce classpath dependencies | Medium   | Medium | Remove unused JARs           |
| GraalVM native image          | Very High| High   | 50ms startup, fewer features |
| AppCDS                        | Medium   | Low    | Simple, no code changes      |
| Custom JRE (jlink)            | Medium   | Medium | Smaller image + faster load  |
| Virtual threads               | Low      | Low    | Faster thread-heavy startup  |

---

## 7. GraalVM Native Images Deep Dive

```java
// Reflection configuration (GraalVM needs to know at build time)
// reflect-config.json
[
  {
    "name": "com.myapp.model.User",
    "allDeclaredConstructors": true,
    "allDeclaredMethods": true,
    "allDeclaredFields": true
  }
]

// Resource configuration
// resource-config.json
{
  "resources": {
    "includes": [
      {"pattern": "application\\.properties"},
      {"pattern": "templates/.*\\.html"}
    ]
  }
}
```

```bash
# Build with configuration
native-image -jar app.jar \
  -H:ReflectionConfigurationFiles=reflect-config.json \
  -H:ResourceConfigurationFiles=resource-config.json \
  --no-fallback \
  -o app-native

# Auto-generate configs by running with tracing agent
java -agentlib:native-image-agent=config-output-dir=native-config \
     -jar app.jar
# Run through all code paths, then use generated configs
```

```dockerfile
# Multi-stage Docker build with GraalVM native image
FROM ghcr.io/graalvm/native-image-community:21 AS builder
WORKDIR /build
COPY . .
RUN mvn -Pnative native:compile -DskipTests

FROM gcr.io/distroless/base-debian12
COPY --from=builder /build/target/app-native /app
EXPOSE 8080
ENTRYPOINT ["/app"]
# Final image: ~80 MB (vs ~400 MB for JRE-based image)
# Startup: ~50 ms (vs ~3 seconds for JVM)
```

---

## 8. Startup Optimization Summary

| Technique                  | Startup Improvement | Trade-off                      |
|---------------------------|---------------------|--------------------------------|
| AppCDS                    | 20-40%              | Build step for archive          |
| Lazy initialization       | 20-50%              | Slower first request            |
| Exclude auto-configs      | 10-30%              | Manual maintenance              |
| Spring Context Indexer    | 5-15%               | Build dependency                |
| Custom JRE (jlink)        | 15-30%              | Module system required          |
| Tiered compilation off    | 10-20%              | Slower peak throughput          |
| GraalVM native image      | 90-95%              | Build time, reflection limits   |

### Startup Optimization Decision Tree

```
Is sub-second startup required?
├── Yes
│   ├── Can you accept native image limitations?
│   │   ├── Yes -> GraalVM native image (~50 ms)
│   │   └── No  -> CDS + lazy init + jlink (~1-2 seconds)
│   └── Consider: serverless, CLI tools, microservices
└── No
    ├── Use CDS (free improvement, no downsides)
    ├── Use lazy initialization (easy win)
    └── Profile and remove unnecessary dependencies
```

---

## Interview Tips

- CDS/AppCDS is the easiest startup optimization (no code changes, 20-40% faster)
- GraalVM native images achieve ~50ms startup but require build-time configuration for reflection
- `spring.main.lazy-initialization=true` defers bean creation but slows first request
- Spring Context Indexer replaces classpath scanning with build-time index
- `jlink` creates custom JREs with only needed modules (30-50 MB vs 300 MB)
- Profile startup with JFR and `-Xlog:class+load` before optimizing
- For serverless/containers: native image > CDS > lazy init
- For long-running services: CDS is usually sufficient; peak throughput matters more

---

[<< Previous: Data Structure Selection](./q14-data-structure-selection.md)

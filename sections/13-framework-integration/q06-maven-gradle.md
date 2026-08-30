# Q6: Maven vs Gradle Build Tools 🟡

[Home](../../README.md) > [Framework Integration](./README.md) > Maven & Gradle

---

## Overview

**Maven** and **Gradle** are the two dominant build tools in the Java ecosystem. Maven follows a convention-over-configuration philosophy using XML. Gradle offers a programmable build using Groovy or Kotlin DSL with superior performance.

---

## Maven Build Lifecycle

Maven has three built-in lifecycles. The **default** lifecycle is the most important:

```
validate → compile → test → package → verify → install → deploy
```

```
┌───────────┐   ┌───────────┐   ┌──────┐   ┌─────────┐
│ validate  │──►│  compile  │──►│ test │──►│ package │
└───────────┘   └───────────┘   └──────┘   └────┬────┘
                                                 │
    ┌────────┐   ┌───────────┐   ┌───────┐       │
    │ deploy │◄──│  install  │◄──│verify │◄──────┘
    └────────┘   └───────────┘   └───────┘
```

| Phase | Description |
|-------|-------------|
| `validate` | Checks project structure is correct |
| `compile` | Compiles source code (`src/main/java`) |
| `test` | Runs unit tests (`src/test/java`) |
| `package` | Packages into JAR/WAR |
| `verify` | Runs integration tests |
| `install` | Installs to local `.m2` repository |
| `deploy` | Uploads to remote repository |

**Clean lifecycle:** `pre-clean → clean → post-clean`

```bash
# Running a phase executes all previous phases
mvn package    # runs: validate → compile → test → package
mvn clean install   # clean first, then full default lifecycle through install
```

---

## Maven POM Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- ✅ Project coordinates (GAV) -->
    <groupId>com.mycompany</groupId>
    <artifactId>my-application</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <!-- ✅ Parent POM (common for Spring Boot) -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <!-- ✅ Centralized version management -->
    <properties>
        <java.version>17</java.version>
        <lombok.version>1.18.30</lombok.version>
    </properties>

    <!-- ✅ Dependencies -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <!-- ✅ Build plugins -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Maven Dependency Scopes

| Scope | Compile CP | Test CP | Runtime CP | Transitive |
|-------|-----------|---------|------------|------------|
| `compile` (default) | ✅ | ✅ | ✅ | ✅ |
| `provided` | ✅ | ✅ | ❌ | ❌ |
| `runtime` | ❌ | ✅ | ✅ | ✅ |
| `test` | ❌ | ✅ | ❌ | ❌ |
| `system` | ✅ | ✅ | ❌ | ❌ |

```xml
<!-- ❌ Anti-pattern: test dependencies without test scope -->
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <scope>compile</scope>  <!-- ships JUnit in production JAR! -->
</dependency>

<!-- ✅ Best practice -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

---

## Gradle Build Script (Groovy DSL)

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

group = 'com.mycompany'
version = '1.0.0-SNAPSHOT'

java {
    sourceCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

---

## Gradle Build Script (Kotlin DSL)

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
}

group = "com.mycompany"
version = "1.0.0-SNAPSHOT"

java {
    sourceCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

tasks.test {
    useJUnitPlatform()
}
```

---

## Gradle Dependency Configurations

| Gradle | Maven Equivalent | Description |
|--------|-----------------|-------------|
| `implementation` | `compile` | Compile + runtime, not exposed to consumers |
| `api` | `compile` | Compile + runtime, exposed transitively |
| `compileOnly` | `provided` | Compile only, not in runtime |
| `runtimeOnly` | `runtime` | Runtime only |
| `testImplementation` | `test` | Test compile + runtime |
| `annotationProcessor` | — | Annotation processing at compile time |

---

## Custom Gradle Tasks

```groovy
// ✅ Gradle is programmable — define custom tasks
tasks.register('hello') {
    group = 'custom'
    description = 'Prints a greeting'
    doLast {
        println 'Hello from custom Gradle task!'
    }
}

// ✅ Task dependencies
tasks.register('generateDocs') {
    dependsOn 'compileJava'
    doLast {
        // generate documentation
    }
}
```

```bash
gradle hello
gradle generateDocs
```

---

## Comparison Table

| Feature | Maven | Gradle |
|---------|-------|--------|
| **Config format** | XML (`pom.xml`) | Groovy/Kotlin DSL |
| **Build model** | Lifecycle phases | Task graph (DAG) |
| **Performance** | Slower (no cache) | Faster (incremental + cache) |
| **Flexibility** | Convention-based | Highly programmable |
| **Learning curve** | Lower | Higher |
| **IDE support** | Excellent | Excellent |
| **Dependency mgmt** | `<dependencyManagement>` | `platform()` / BOM |
| **Multi-module** | Parent POM inheritance | `include` in settings |
| **Build cache** | ❌ | ✅ local + remote |
| **Daemon** | ❌ | ✅ persistent JVM |
| **Wrapper** | `mvnw` (Maven 3.7+) | `gradlew` (built-in) |
| **Android** | ❌ | ✅ official |
| **Market share** | Higher (enterprise) | Growing (modern projects) |

---

## Multi-Module Projects

### Maven

```xml
<!-- parent pom.xml -->
<modules>
    <module>core</module>
    <module>api</module>
    <module>service</module>
</modules>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.mycompany</groupId>
            <artifactId>core</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Gradle

```kotlin
// settings.gradle.kts
rootProject.name = "my-application"
include("core", "api", "service")

// service/build.gradle.kts
dependencies {
    implementation(project(":core"))
}
```

---

## Anti-Patterns vs Best Practices

```xml
<!-- ❌ Anti-pattern: hardcoded versions scattered across modules -->
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>32.1.3-jre</version>
</dependency>

<!-- ✅ Best practice: centralize in properties or dependencyManagement -->
<properties>
    <guava.version>32.1.3-jre</guava.version>
</properties>
```

```bash
# ❌ Anti-pattern: committing IDE-specific files
# ✅ Best practice: use wrapper for reproducible builds
./mvnw clean install     # Maven Wrapper
./gradlew build          # Gradle Wrapper
```

---

## Interview Tips

| Topic | Key Point |
|-------|-----------|
| Maven lifecycle | `validate → compile → test → package → verify → install → deploy` |
| GAV | groupId + artifactId + version uniquely identify an artifact |
| `implementation` vs `api` | `implementation` hides transitive deps; `api` exposes them |
| Why Gradle is faster | Incremental builds, build cache, daemon process |
| When to choose which | Maven for simple enterprise; Gradle for complex or Android projects |

---

[← Previous: Q5 - Mockito Basics](./q05-mockito-basics.md) | [Next: Q7 - Lombok →](./q07-lombok.md)

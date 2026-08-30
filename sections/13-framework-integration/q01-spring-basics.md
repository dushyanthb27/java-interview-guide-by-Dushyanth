# Q1: How Does Spring Framework Work? 🟡

[Home](../../README.md) > [Framework Integration](./README.md) > Spring Basics

---

## Overview

The Spring Framework is built around the **Inversion of Control (IoC)** principle. Instead of your code creating and managing objects, the Spring IoC container takes over that responsibility, assembling your application from loosely coupled components.

---

## Core Concepts

### 1. IoC Container

The IoC container is the heart of Spring. It reads configuration metadata and creates, wires, and manages beans.

```
┌─────────────────────────────────────────────┐
│            Spring IoC Container             │
│                                             │
│  Configuration ──► Container ──► Ready App  │
│  (XML/Anno/Java)    (creates     (fully     │
│                      & wires      wired     │
│                      beans)       beans)    │
└─────────────────────────────────────────────┘
```

**Two main container types:**

| Container | Interface | Use Case |
|-----------|-----------|----------|
| BeanFactory | `BeanFactory` | Lightweight, lazy initialization |
| ApplicationContext | `ApplicationContext` | Full-featured, eager initialization |

### 2. ApplicationContext

```java
// ✅ Using ApplicationContext (preferred)
ApplicationContext context =
    new AnnotationConfigApplicationContext(AppConfig.class);

UserService userService = context.getBean(UserService.class);
```

**Common ApplicationContext implementations:**

```java
// Annotation-based configuration
new AnnotationConfigApplicationContext(AppConfig.class);

// XML-based configuration
new ClassPathXmlApplicationContext("applicationContext.xml");

// Web application
new AnnotationConfigWebApplicationContext();
```

---

## Bean Lifecycle

```
Instantiation → Populate Properties → BeanNameAware → BeanFactoryAware
    → ApplicationContextAware → @PostConstruct → InitializingBean
    → Custom init-method → Bean Ready → @PreDestroy → DisposableBean
    → Custom destroy-method → Bean Destroyed
```

```java
@Component
public class OrderService {

    // ✅ Called after dependency injection is complete
    @PostConstruct
    public void init() {
        System.out.println("OrderService initialized");
    }

    // ✅ Called before bean is removed from container
    @PreDestroy
    public void cleanup() {
        System.out.println("OrderService destroyed");
    }
}
```

---

## Configuration Approaches

### 1. Java Configuration (Modern & Preferred)

```java
// ✅ Best practice — type-safe, refactor-friendly
@Configuration
public class AppConfig {

    @Bean
    public UserRepository userRepository() {
        return new JdbcUserRepository();
    }

    @Bean
    public UserService userService(UserRepository userRepository) {
        return new UserServiceImpl(userRepository);
    }
}
```

### 2. Annotation-Based (Component Scanning)

```java
// ✅ Clean and concise for most use cases
@Component          // Generic component
@Service            // Service layer
@Repository         // Data access layer
@Controller         // Web layer (Spring MVC)
@RestController     // REST API controller
```

```java
@Service
public class PaymentService {

    private final PaymentGateway gateway;

    @Autowired
    public PaymentService(PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

### 3. XML Configuration (Legacy)

```xml
<!-- ❌ Avoid for new projects — verbose and error-prone -->
<beans>
    <bean id="userRepo" class="com.app.JdbcUserRepository"/>
    <bean id="userService" class="com.app.UserServiceImpl">
        <constructor-arg ref="userRepo"/>
    </bean>
</beans>
```

---

## Component Scanning

```java
@Configuration
@ComponentScan(basePackages = "com.myapp")
public class AppConfig { }

// Spring Boot auto-scans from the main class package downward
@SpringBootApplication  // includes @ComponentScan
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

---

## Spring Boot Auto-Configuration

Spring Boot automatically configures beans based on classpath dependencies.

```java
// ✅ Spring Boot starter brings in auto-configured defaults
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

```yaml
# application.yml — override auto-configured defaults
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: secret
server:
  port: 8081
```

---

## Profiles

```java
@Configuration
@Profile("dev")
public class DevDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2).build();
    }
}

@Configuration
@Profile("prod")
public class ProdDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:mysql://prod-server:3306/mydb").build();
    }
}
```

```properties
# Activate a profile
spring.profiles.active=dev
```

---

## Properties & Externalized Configuration

```java
@Component
public class AppSettings {

    @Value("${app.name}")
    private String appName;

    @Value("${app.max-retries:3}")   // default value = 3
    private int maxRetries;
}

// ✅ Type-safe configuration properties (preferred)
@ConfigurationProperties(prefix = "app.mail")
public class MailProperties {
    private String host;
    private int port;
    private String from;
    // getters, setters
}
```

---

## Anti-Patterns vs Best Practices

```java
// ❌ Anti-pattern: creating dependencies manually inside Spring beans
@Service
public class OrderService {
    public void process() {
        PaymentGateway gateway = new StripeGateway(); // tightly coupled
    }
}

// ✅ Best practice: let Spring inject the dependency
@Service
public class OrderService {
    private final PaymentGateway gateway;

    public OrderService(PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

```java
// ❌ Anti-pattern: using ApplicationContext.getBean() in business code
@Service
public class ReportService {
    @Autowired
    private ApplicationContext context;

    public void generate() {
        // Service locator pattern — defeats the purpose of DI
        var printer = context.getBean(Printer.class);
    }
}

// ✅ Best practice: inject directly
@Service
public class ReportService {
    private final Printer printer;

    public ReportService(Printer printer) {
        this.printer = printer;
    }
}
```

---

## Interview Tips

| Topic | Key Point |
|-------|-----------|
| IoC vs DI | IoC is the principle; DI is the mechanism Spring uses to implement it |
| BeanFactory vs ApplicationContext | ApplicationContext adds i18n, events, AOP — always prefer it |
| @Component vs @Bean | @Component = class-level scan; @Bean = method-level factory in @Configuration |
| Spring Boot magic | `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| Profile use case | Swap implementations (H2 for dev, MySQL for prod) without code changes |

---

[Next: Q2 - Dependency Injection →](./q02-dependency-injection.md)

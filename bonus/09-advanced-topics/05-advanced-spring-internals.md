# Advanced Spring Internals Deep-Dive

## 📋 Question
Explain advanced Spring internals including the complete bean lifecycle, AOP proxy mechanisms (JDK dynamic proxy vs CGLIB), circular dependency resolution, and the ApplicationContext hierarchy. How does Spring handle proxy creation and method interception?

## Difficulty: 🔴 Expert

---

## 💡 Answer

Understanding Spring's internals is crucial for debugging complex issues, optimizing performance, and building custom extensions. This deep-dive covers the bean lifecycle, AOP proxy mechanisms, and advanced container features.

### Complete Bean Lifecycle

The Spring bean lifecycle is a complex process with multiple phases and extension points.

```java
// Complete bean lifecycle phases:
// 1. Instantiation (constructor)
// 2. Population of properties (setter injection)
// 3. BeanNameAware, BeanFactoryAware, etc.
// 4. BeanPostProcessor.beforeInitialization()
// 5. @PostConstruct / InitializingBean.afterPropertiesSet()
// 6. BeanPostProcessor.afterInitialization()
// 7. Bean ready for use
// 8. @PreDestroy / DisposableBean.destroy()
// 9. Bean destruction
```

**Demonstrating the complete lifecycle:**

```java
import org.springframework.beans.factory.*;
import org.springframework.context.*;
import org.springframework.beans.factory.config.*;
import javax.annotation.*;

@Component
public class LifecycleBean implements 
    BeanNameAware, BeanFactoryAware, ApplicationContextAware,
    InitializingBean, DisposableBean {
    
    private String name;
    private BeanFactory beanFactory;
    private ApplicationContext applicationContext;
    
    // 1. Constructor
    public LifecycleBean() {
        System.out.println("1. Constructor called");
    }
    
    // 2. Setter injection
    @Autowired
    public void setName(String name) {
        System.out.println("2. Setter injection: " + name);
        this.name = name;
    }
    
    // 3. Aware interfaces
    @Override
    public void setBeanName(String name) {
        System.out.println("3. BeanNameAware: " + name);
    }
    
    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        System.out.println("3. BeanFactoryAware");
        this.beanFactory = beanFactory;
    }
    
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        System.out.println("3. ApplicationContextAware");
        this.applicationContext = applicationContext;
    }
    
    // 4. BeanPostProcessor.beforeInitialization
    @Override
    public void afterPropertiesSet() {
        System.out.println("5. InitializingBean.afterPropertiesSet");
    }
    
    // 4. @PostConstruct
    @PostConstruct
    public void postConstruct() {
        System.out.println("5. @PostConstruct");
    }
    
    // 6. BeanPostProcessor.afterInitialization
    // (handled by container)
    
    // 7. Bean ready for use
    public void doWork() {
        System.out.println("7. Bean is ready and working");
    }
    
    // 8. @PreDestroy
    @PreDestroy
    public void preDestroy() {
        System.out.println("8. @PreDestroy");
    }
    
    // 8. DisposableBean.destroy
    @Override
    public void destroy() {
        System.out.println("8. DisposableBean.destroy");
    }
}
```

**Custom BeanPostProcessor:**

```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof LifecycleBean) {
            System.out.println("4. BeanPostProcessor.beforeInitialization: " + beanName);
        }
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof LifecycleBean) {
            System.out.println("6. BeanPostProcessor.afterInitialization: " + beanName);
        }
        return bean;
    }
}
```

### AOP Proxy Mechanisms

Spring AOP uses either **JDK dynamic proxies** (for interfaces) or **CGLIB proxies** (for classes).

**JDK Dynamic Proxy:**

```java
import java.lang.reflect.*;

// Target interface
public interface UserService {
    String getUsername(Long userId);
    void updateUser(Long userId, String username);
}

// Implementation
@Service
public class UserServiceImpl implements UserService {
    @Override
    public String getUsername(Long userId) {
        return "user-" + userId;
    }
    
    @Override
    public void updateUser(Long userId, String username) {
        System.out.println("Updating user: " + userId + " -> " + username);
    }
}

// Manual JDK dynamic proxy
public class ProxyFactory {
    public static UserService createProxy(UserService target) {
        return (UserService) Proxy.newProxyInstance(
            UserService.class.getClassLoader(),
            new Class<?>[] { UserService.class },
            new InvocationHandler() {
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    System.out.println("Before: " + method.getName());
                    Object result = method.invoke(target, args);
                    System.out.println("After: " + method.getName());
                    return result;
                }
            }
        );
    }
}

// Usage
UserService proxy = ProxyFactory.createProxy(new UserServiceImpl());
proxy.getUsername(1L);
```

**CGLIB Proxy:**

```java
import org.springframework.cglib.proxy.*;

// Target class (no interface)
@Service
public class OrderService {
    public void createOrder(String orderId) {
        System.out.println("Creating order: " + orderId);
    }
    
    public void cancelOrder(String orderId) {
        System.out.println("Cancelling order: " + orderId);
    }
}

// Manual CGLIB proxy
public class CglibProxyFactory {
    public static OrderService createProxy(OrderService target) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(OrderService.class);
        enhancer.setCallback(new MethodInterceptor() {
            @Override
            public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
                System.out.println("CGLIB Before: " + method.getName());
                Object result = proxy.invoke(target, args);
                System.out.println("CGLIB After: " + method.getName());
                return result;
            }
        });
        return (OrderService) enhancer.create();
    }
}

// Usage
OrderService proxy = CglibProxyFactory.createProxy(new OrderService());
proxy.createOrder("ORD-123");
```

**Spring AOP with @Aspect:**

```java
import org.aspectj.lang.*;
import org.aspectj.lang.annotation.*;

@Aspect
@Component
public class LoggingAspect {
    
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("Before: " + joinPoint.getSignature());
    }
    
    @After("execution(* com.example.service.*.*(..))")
    public void logAfter(JoinPoint joinPoint) {
        System.out.println("After: " + joinPoint.getSignature());
    }
    
    @Around("execution(* com.example.service.*.*(..))")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        System.out.println("Around before: " + joinPoint.getSignature());
        
        Object result = joinPoint.proceed();
        
        long duration = System.currentTimeMillis() - start;
        System.out.println("Around after: " + joinPoint.getSignature() + " took " + duration + "ms");
        
        return result;
    }
    
    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        System.out.println("AfterReturning: " + joinPoint.getSignature() + " returned " + result);
    }
    
    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))", throwing = "error")
    public void logAfterThrowing(JoinPoint joinPoint, Throwable error) {
        System.out.println("AfterThrowing: " + joinPoint.getSignature() + " threw " + error);
    }
}
```

**Proxy Selection Strategy:**

```java
// Spring's proxy selection logic:
// - If target implements at least one interface → JDK dynamic proxy
// - If target doesn't implement any interface → CGLIB proxy
// - Can force CGLIB with @EnableAspectJAutoProxy(proxyTargetClass = true)

@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true)  // Force CGLIB
public class AopConfig {
    // ...
}
```

### Circular Dependency Resolution

Spring resolves circular dependencies through **three-phase construction** and **early exposure** of beans.

```java
@Service
public class ServiceA {
    private final ServiceB serviceB;
    
    public ServiceA(ServiceB serviceB) {
        this.serviceB = serviceB;
        System.out.println("ServiceA constructed");
    }
    
    @PostConstruct
    public void init() {
        System.out.println("ServiceA initialized");
    }
}

@Service
public class ServiceB {
    private final ServiceA serviceA;
    
    public ServiceB(ServiceA serviceA) {
        this.serviceA = serviceA;
        System.out.println("ServiceB constructed");
    }
    
    @PostConstruct
    public void init() {
        System.out.println("ServiceB initialized");
    }
}

// Spring resolves this by:
// 1. Create ServiceA (instance only, no properties set)
// 2. Expose ServiceA reference early (before fully initialized)
// 3. Create ServiceB (needs ServiceA — uses early reference)
// 4. Inject ServiceB into ServiceA
// 5. Complete ServiceA initialization
// 6. Complete ServiceB initialization
```

**Circular dependency with @Async:**

```java
// ❌ This will fail — @Async creates a proxy, breaking circular resolution
@Service
public class ServiceA {
    @Autowired
    private ServiceB serviceB;
    
    @Async
    public void asyncMethod() {
        serviceB.method();
    }
}

@Service
public class ServiceB {
    @Autowired
    private ServiceA serviceA;
    
    public void method() {
        // ...
    }
}

// ✅ Solution: Use @Lazy
@Service
public class ServiceA {
    @Autowired
    @Lazy
    private ServiceB serviceB;
    
    @Async
    public void asyncMethod() {
        serviceB.method();
    }
}
```

### ApplicationContext Hierarchy

Spring supports hierarchical application contexts where child contexts can access beans from parent contexts.

```java
// Parent context
AnnotationConfigApplicationContext parentContext = 
    new AnnotationConfigApplicationContext(ParentConfig.class);

// Child context
AnnotationConfigApplicationContext childContext = 
    new AnnotationConfigApplicationContext();
childContext.setParent(parentContext);
childContext.register(ChildConfig.class);
childContext.refresh();

// Child can access parent beans
Object parentBean = childContext.getBean("parentBean");

// Parent cannot access child beans
// parentContext.getBean("childBean");  // NoSuchBeanDefinitionException
```

**Spring Boot context hierarchy:**

```
BootstrapContext (very early)
    ↓
ApplicationContext (main)
    ↓
ServletWebServerApplicationContext (for web apps)
    ↓
ApplicationContext for each @Configuration class
```

### Code Example: Complete Spring Internals Demo

```java
package com.interview.advanced.spring;

import org.springframework.*;
import org.springframework.context.*;
import org.springframework.beans.factory.*;
import org.springframework.beans.factory.config.*;
import org.springframework.stereotype.*;
import org.springframework.beans.factory.annotation.*;
import javax.annotation.*;

// === BEAN LIFECYCLE DEMO ===
@Component
public class DemoBean implements 
    BeanNameAware, BeanFactoryAware, ApplicationContextAware,
    InitializingBean, DisposableBean {
    
    private String value;
    
    public DemoBean() {
        System.out.println("1. Constructor");
    }
    
    @Autowired
    public void setValue(String value) {
        System.out.println("2. Setter: " + value);
        this.value = value;
    }
    
    @Override
    public void setBeanName(String name) {
        System.out.println("3. BeanNameAware: " + name);
    }
    
    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        System.out.println("3. BeanFactoryAware");
    }
    
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        System.out.println("3. ApplicationContextAware");
    }
    
    @PostConstruct
    public void postConstruct() {
        System.out.println("5. @PostConstruct");
    }
    
    @Override
    public void afterPropertiesSet() {
        System.out.println("5. InitializingBean.afterPropertiesSet");
    }
    
    public void doWork() {
        System.out.println("7. Bean working");
    }
    
    @PreDestroy
    public void preDestroy() {
        System.out.println("8. @PreDestroy");
    }
    
    @Override
    public void destroy() {
        System.out.println("8. DisposableBean.destroy");
    }
}

@Component
class CustomPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof DemoBean) {
            System.out.println("4. BeforeInitialization: " + beanName);
        }
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof DemoBean) {
            System.out.println("6. AfterInitialization: " + beanName);
        }
        return bean;
    }
}

// === CIRCULAR DEPENDENCY DEMO ===
@Component
class CircularA {
    private final CircularB circularB;
    
    public CircularA(CircularB circularB) {
        System.out.println("CircularA constructor");
        this.circularB = circularB;
    }
    
    @PostConstruct
    public void init() {
        System.out.println("CircularA init");
    }
}

@Component
class CircularB {
    private final CircularA circularA;
    
    public CircularB(CircularA circularA) {
        System.out.println("CircularB constructor");
        this.circularA = circularA;
    }
    
    @PostConstruct
    public void init() {
        System.out.println("CircularB init");
    }
}

// === CONFIGURATION ===
@Configuration
@ComponentScan
class SpringConfig {
    @Bean
    public String value() {
        return "demo-value";
    }
}

// === MAIN ===
public class SpringInternalsDemo {
    public static void main(String[] args) {
        System.out.println("=== Starting Spring Context ===");
        AnnotationConfigApplicationContext context = 
            new AnnotationConfigApplicationContext(SpringConfig.class);
        
        System.out.println("\n=== Context Started ===");
        DemoBean bean = context.getBean(DemoBean.class);
        bean.doWork();
        
        System.out.println("\n=== Shutting Down ===");
        context.close();
    }
}
```

### ⚠️ Common Mistakes

**1. Calling @Async methods from within the same class**

```java
@Service
public class OrderService {
    @Async
    public void asyncProcess() {
        // Async processing
    }
    
    public void processOrder() {
        // ❌ This won't be async — calls the method directly, bypassing proxy
        asyncProcess();
    }
}

// ✅ Solution: Inject self or move to another service
@Service
public class OrderService {
    @Autowired
    private OrderService self;  // Injects proxy
    
    @Async
    public void asyncProcess() {
        // Async processing
    }
    
    public void processOrder() {
        self.asyncProcess();  // Goes through proxy
    }
}
```

**2. @Transactional on private methods**

```java
@Service
public class UserService {
    // ❌ Won't work — private methods can't be proxied
    @Transactional
    private void privateTransactionalMethod() {
        // ...
    }
    
    // ✅ Use public or protected
    @Transactional
    public void publicTransactionalMethod() {
        // ...
    }
}
```

**3. Final classes with AOP**

```java
// ❌ Can't proxy final classes with CGLIB
@Service
public final class FinalService {
    public void method() {
        // ...
    }
}

// ✅ Use interfaces or make class non-final
@Service
public class ServiceImpl implements ServiceInterface {
    public void method() {
        // ...
    }
}
```

### ⚡ Performance Tip

**Proxy creation has overhead.** For high-performance scenarios:
- Use interface-based programming (JDK proxies are faster than CGLIB)
- Avoid unnecessary AOP for hot paths
- Consider compile-time weaving (AspectJ) for zero runtime overhead
- Use `@Lazy` to defer proxy creation until needed

### 🔧 Spring Internals

Spring uses **singleton scope** by default, storing bean instances in a `Map<String, Object>`. The `DefaultSingletonBeanRegistry` manages the singleton cache. Circular dependencies are resolved using a **three-level cache**:
1. `singletonObjects`: Fully initialized beans
2. `earlySingletonObjects`: Early-exposed beans (for circular deps)
3. `singletonFactories`: Object factories for creating early references

AOP proxies are created by `AbstractAutoProxyCreator`, which wraps the target bean in a proxy that intercepts method calls and applies advices.

---

## 🎯 Interviewer Insight

Spring internals questions test your understanding of the framework's architecture. Interviewers want to see that you:
- Know the complete bean lifecycle and extension points
- Understand proxy mechanisms (JDK vs CGLIB)
- Can troubleshoot circular dependency issues
- Understand the ApplicationContext hierarchy
- Know how AOP works under the hood

Strong candidates can explain the three-level cache for circular dependencies and the difference between `@PostConstruct` and `InitializingBean`.

## ⚡ Follow-up Questions

1. How does Spring's three-level cache resolve circular dependencies?
2. Why can't @Async methods be called from within the same class?
3. What is the difference between JDK dynamic proxy and CGLIB proxy?

## 📚 Related Topics

- [Dependency Injection](../../sections/13-framework-integration/q02-dependency-injection.md)
- [Spring Basics](../../sections/13-framework-integration/q01-spring-basics.md)
- [AOP Concepts](../../sections/13-framework-integration/README.md)

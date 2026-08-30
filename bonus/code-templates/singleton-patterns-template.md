# Singleton Patterns Template

## Overview
All singleton pattern variations with their trade-offs, thread-safety guarantees, and use cases.

## Use Cases
- Logger instances
- Configuration managers
- Database connections
- Thread pools
- Cache managers
- Resource managers

## Implementation

All major singleton variations with detailed explanations.

## Code

```java
import java.io.Serializable;
import java.lang.reflect.Constructor;

/**
 * PATTERN 1: Eager Initialization (Thread-Safe)
 * - Simplest approach
 * - Thread-safe by default
 * - Instance created at class loading time
 * - Drawback: Always created, even if never used
 */
public class EagerSingleton {
    // Instance created when class is loaded
    private static final EagerSingleton INSTANCE = new EagerSingleton();

    // Private constructor prevents instantiation
    private EagerSingleton() {
        // Prevent instantiation via reflection
        if (INSTANCE != null) {
            throw new IllegalStateException("Singleton already instantiated");
        }
    }

    public static EagerSingleton getInstance() {
        return INSTANCE;
    }

    public void doSomething() {
        System.out.println("EagerSingleton doing something");
    }
}

/**
 * PATTERN 2: Lazy Initialization (NOT Thread-Safe)
 * - Instance created on first use
 * - Not thread-safe
 * - Simple implementation
 * - Drawback: Race conditions in multi-threaded environment
 */
public class LazySingleton {
    private static LazySingleton instance;

    private LazySingleton() {
    }

    // NOT THREAD-SAFE - Do not use in production
    public static LazySingleton getInstance() {
        if (instance == null) {
            instance = new LazySingleton();
        }
        return instance;
    }

    public void doSomething() {
        System.out.println("LazySingleton doing something");
    }
}

/**
 * PATTERN 3: Synchronized Lazy Initialization (Thread-Safe)
 * - Lazy initialization with synchronization
 * - Thread-safe
 * - Drawback: Performance overhead from synchronization
 */
public class SynchronizedLazySingleton {
    private static SynchronizedLazySingleton instance;

    private SynchronizedLazySingleton() {
    }

    // Synchronized method - thread-safe but slow
    public static synchronized SynchronizedLazySingleton getInstance() {
        if (instance == null) {
            instance = new SynchronizedLazySingleton();
        }
        return instance;
    }

    public void doSomething() {
        System.out.println("SynchronizedLazySingleton doing something");
    }
}

/**
 * PATTERN 4: Double-Checked Locking (Thread-Safe, Optimized)
 * - Lazy initialization with minimal synchronization
 * - Thread-safe with good performance
 * - Requires volatile keyword
 * - Drawback: Complex, can be misunderstood
 */
public class DoubleCheckedLockingSingleton {
    // volatile ensures visibility across threads
    private static volatile DoubleCheckedLockingSingleton instance;

    private DoubleCheckedLockingSingleton() {
    }

    public static DoubleCheckedLockingSingleton getInstance() {
        // First check without synchronization
        if (instance == null) {
            synchronized (DoubleCheckedLockingSingleton.class) {
                // Second check with synchronization
                if (instance == null) {
                    instance = new DoubleCheckedLockingSingleton();
                }
            }
        }
        return instance;
    }

    public void doSomething() {
        System.out.println("DoubleCheckedLockingSingleton doing something");
    }
}

/**
 * PATTERN 5: Bill Pugh Singleton (Thread-Safe, Elegant)
 * - Uses class loader to ensure thread safety
 * - Lazy initialization
 * - No synchronization needed
 * - Best practice for most scenarios
 */
public class BillPughSingleton {
    // Private constructor
    private BillPughSingleton() {
    }

    /**
     * Static inner class is loaded only when getInstance() is called
     * Class loading is thread-safe by JVM
     */
    private static class SingletonHelper {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }

    public static BillPughSingleton getInstance() {
        return SingletonHelper.INSTANCE;
    }

    public void doSomething() {
        System.out.println("BillPughSingleton doing something");
    }
}

/**
 * PATTERN 6: Enum Singleton (Thread-Safe, Serialization-Safe)
 * - Best approach for thread-safety and serialization
 * - Prevents reflection attacks
 * - Handles serialization automatically
 * - Recommended by Joshua Bloch (Effective Java)
 */
public enum EnumSingleton {
    // Single instance
    INSTANCE;

    // Instance variables
    private String value;

    EnumSingleton() {
        this.value = "initialized";
    }

    public void doSomething() {
        System.out.println("EnumSingleton doing something");
    }

    public String getValue() {
        return value;
    }

    public void setValue(String value) {
        this.value = value;
    }
}

/**
 * PATTERN 7: Serialization-Safe Singleton
 * - Handles serialization/deserialization
 * - Prevents creation of new instance via serialization
 * - Implements Serializable
 */
public class SerializableSingleton implements Serializable {
    private static final long serialVersionUID = 1L;
    private static final SerializableSingleton INSTANCE = 
        new SerializableSingleton();

    private SerializableSingleton() {
    }

    public static SerializableSingleton getInstance() {
        return INSTANCE;
    }

    /**
     * Prevent deserialization from creating new instance
     */
    protected Object readResolve() {
        return INSTANCE;
    }

    public void doSomething() {
        System.out.println("SerializableSingleton doing something");
    }
}

/**
 * PATTERN 8: Reflection-Safe Singleton
 * - Prevents creation via reflection
 * - Throws exception if instantiation attempted via reflection
 */
public class ReflectionSafeSingleton {
    private static final ReflectionSafeSingleton INSTANCE = 
        new ReflectionSafeSingleton();
    private static boolean instantiated = false;

    private ReflectionSafeSingleton() {
        // Prevent instantiation via reflection
        if (instantiated) {
            throw new IllegalStateException(
                "Singleton already instantiated via reflection");
        }
        instantiated = true;
    }

    public static ReflectionSafeSingleton getInstance() {
        return INSTANCE;
    }

    public void doSomething() {
        System.out.println("ReflectionSafeSingleton doing something");
    }
}

/**
 * PATTERN 9: Thread-Safe Singleton with Holder Pattern
 * - Combines benefits of eager and lazy initialization
 * - Thread-safe without synchronization
 * - Efficient
 */
public class HolderPatternSingleton {
    private HolderPatternSingleton() {
    }

    /**
     * Holder class - loaded only when getInstance() is called
     */
    private static class Holder {
        static final HolderPatternSingleton INSTANCE = 
            new HolderPatternSingleton();
    }

    public static HolderPatternSingleton getInstance() {
        return Holder.INSTANCE;
    }

    public void doSomething() {
        System.out.println("HolderPatternSingleton doing something");
    }
}

/**
 * PATTERN 10: Parameterized Singleton
 * - Singleton with configuration parameters
 * - Useful for different singleton instances based on parameters
 */
public class ParameterizedSingleton {
    private static final java.util.Map<String, ParameterizedSingleton> instances = 
        new java.util.concurrent.ConcurrentHashMap<>();
    private final String config;

    private ParameterizedSingleton(String config) {
        this.config = config;
    }

    /**
     * Get singleton instance for specific configuration
     */
    public static ParameterizedSingleton getInstance(String config) {
        return instances.computeIfAbsent(config, 
            key -> new ParameterizedSingleton(key));
    }

    public String getConfig() {
        return config;
    }

    public void doSomething() {
        System.out.println("ParameterizedSingleton(" + config + 
                          ") doing something");
    }
}

/**
 * PATTERN 11: Thread-Local Singleton
 * - Different instance per thread
 * - Thread-safe by design
 * - Useful for thread-specific state
 */
public class ThreadLocalSingleton {
    private static final ThreadLocal<ThreadLocalSingleton> threadLocal = 
        ThreadLocal.withInitial(ThreadLocalSingleton::new);

    private final String threadId;

    private ThreadLocalSingleton() {
        this.threadId = Thread.currentThread().getName();
    }

    public static ThreadLocalSingleton getInstance() {
        return threadLocal.get();
    }

    public String getThreadId() {
        return threadId;
    }

    public void doSomething() {
        System.out.println("ThreadLocalSingleton(" + threadId + 
                          ") doing something");
    }

    /**
     * Clean up thread-local instance
     */
    public static void cleanup() {
        threadLocal.remove();
    }
}

/**
 * COMPARISON TABLE
 * 
 * Pattern                          | Thread-Safe | Lazy | Serializable | Reflection-Safe | Performance
 * --------------------------------|-------------|------|--------------|-----------------|------------
 * Eager Initialization            | Yes         | No   | No           | No              | Good
 * Lazy (Not Synchronized)         | No          | Yes  | No           | No              | Excellent
 * Synchronized Lazy               | Yes         | Yes  | No           | No              | Poor
 * Double-Checked Locking          | Yes         | Yes  | No           | No              | Good
 * Bill Pugh (Holder Pattern)      | Yes         | Yes  | No           | No              | Excellent
 * Enum Singleton                  | Yes         | Yes  | Yes          | Yes             | Excellent
 * Serializable Singleton          | Yes         | No   | Yes          | No              | Good
 * Reflection-Safe Singleton       | Yes         | No   | No           | Yes             | Good
 * Thread-Local Singleton          | Yes         | Yes  | No           | No              | Good
 * Parameterized Singleton         | Yes         | Yes  | No           | No              | Good
 */

/**
 * RECOMMENDATION:
 * 
 * 1. Use Enum Singleton (EnumSingleton) for most cases
 *    - Thread-safe
 *    - Serialization-safe
 *    - Reflection-safe
 *    - Simplest and most robust
 * 
 * 2. Use Bill Pugh Singleton (BillPughSingleton) if you need:
 *    - Lazy initialization
 *    - No enum overhead
 *    - Maximum performance
 * 
 * 3. Use Double-Checked Locking only if you need:
 *    - Lazy initialization
 *    - Explicit control
 *    - And understand the complexity
 * 
 * 4. Avoid Lazy Singleton - not thread-safe
 * 
 * 5. Avoid Synchronized Lazy - poor performance
 */
```

## Testing

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class SingletonPatternsTest {

    @Test
    public void testEagerSingleton() {
        EagerSingleton instance1 = EagerSingleton.getInstance();
        EagerSingleton instance2 = EagerSingleton.getInstance();

        assertSame(instance1, instance2);
    }

    @Test
    public void testBillPughSingleton() {
        BillPughSingleton instance1 = BillPughSingleton.getInstance();
        BillPughSingleton instance2 = BillPughSingleton.getInstance();

        assertSame(instance1, instance2);
    }

    @Test
    public void testEnumSingleton() {
        EnumSingleton instance1 = EnumSingleton.INSTANCE;
        EnumSingleton instance2 = EnumSingleton.INSTANCE;

        assertSame(instance1, instance2);
    }

    @Test
    public void testDoubleCheckedLockingSingleton() {
        DoubleCheckedLockingSingleton instance1 = 
            DoubleCheckedLockingSingleton.getInstance();
        DoubleCheckedLockingSingleton instance2 = 
            DoubleCheckedLockingSingleton.getInstance();

        assertSame(instance1, instance2);
    }

    @Test
    public void testConcurrentAccess() throws InterruptedException {
        int threadCount = 100;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        ConcurrentHashMap<BillPughSingleton, Integer> instances = 
            new ConcurrentHashMap<>();

        for (int i = 0; i < threadCount; i++) {
            executor.submit(() -> {
                BillPughSingleton instance = BillPughSingleton.getInstance();
                instances.putIfAbsent(instance, 1);
            });
        }

        executor.shutdown();
        assertTrue(executor.awaitTermination(10, TimeUnit.SECONDS));

        // Should only have one unique instance
        assertEquals(1, instances.size());
    }

    @Test
    public void testParameterizedSingleton() {
        ParameterizedSingleton instance1 = 
            ParameterizedSingleton.getInstance("config1");
        ParameterizedSingleton instance2 = 
            ParameterizedSingleton.getInstance("config1");
        ParameterizedSingleton instance3 = 
            ParameterizedSingleton.getInstance("config2");

        assertSame(instance1, instance2);
        assertNotSame(instance1, instance3);
    }

    @Test
    public void testThreadLocalSingleton() throws InterruptedException {
        String[] threadNames = new String[2];
        Thread thread1 = new Thread(() -> {
            ThreadLocalSingleton instance = ThreadLocalSingleton.getInstance();
            threadNames[0] = instance.getThreadId();
            ThreadLocalSingleton.cleanup();
        });

        Thread thread2 = new Thread(() -> {
            ThreadLocalSingleton instance = ThreadLocalSingleton.getInstance();
            threadNames[1] = instance.getThreadId();
            ThreadLocalSingleton.cleanup();
        });

        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();

        assertNotEquals(threadNames[0], threadNames[1]);
    }

    @Test
    public void testEnumSingletonSerialization() throws Exception {
        EnumSingleton instance1 = EnumSingleton.INSTANCE;
        instance1.setValue("test");

        // Simulate serialization/deserialization
        java.io.ByteArrayOutputStream baos = 
            new java.io.ByteArrayOutputStream();
        java.io.ObjectOutputStream oos = 
            new java.io.ObjectOutputStream(baos);
        oos.writeObject(instance1);
        oos.close();

        java.io.ByteArrayInputStream bais = 
            new java.io.ByteArrayInputStream(baos.toByteArray());
        java.io.ObjectInputStream ois = 
            new java.io.ObjectInputStream(bais);
        EnumSingleton instance2 = (EnumSingleton) ois.readObject();
        ois.close();

        // Should be same instance
        assertSame(instance1, instance2);
        assertEquals("test", instance2.getValue());
    }

    @Test
    public void testSerializableSingleton() throws Exception {
        SerializableSingleton instance1 = SerializableSingleton.getInstance();

        // Serialize
        java.io.ByteArrayOutputStream baos = 
            new java.io.ByteArrayOutputStream();
        java.io.ObjectOutputStream oos = 
            new java.io.ObjectOutputStream(baos);
        oos.writeObject(instance1);
        oos.close();

        // Deserialize
        java.io.ByteArrayInputStream bais = 
            new java.io.ByteArrayInputStream(baos.toByteArray());
        java.io.ObjectInputStream ois = 
            new java.io.ObjectInputStream(bais);
        SerializableSingleton instance2 = (SerializableSingleton) ois.readObject();
        ois.close();

        // Should be same instance
        assertSame(instance1, instance2);
    }

    @Test
    public void testReflectionSafeSingleton() {
        ReflectionSafeSingleton instance1 = 
            ReflectionSafeSingleton.getInstance();

        // Try to create via reflection - should fail
        assertThrows(IllegalStateException.class, () -> {
            try {
                Constructor<?> constructor = 
                    ReflectionSafeSingleton.class.getDeclaredConstructor();
                constructor.setAccessible(true);
                constructor.newInstance();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });
    }

    @Test
    public void testPerformanceComparison() {
        int iterations = 1000000;

        // Test Eager Singleton
        long start = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            EagerSingleton.getInstance();
        }
        long eagerTime = System.nanoTime() - start;

        // Test Bill Pugh Singleton
        start = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            BillPughSingleton.getInstance();
        }
        long billPughTime = System.nanoTime() - start;

        // Test Enum Singleton
        start = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            EnumSingleton.INSTANCE;
        }
        long enumTime = System.nanoTime() - start;

        System.out.println("Eager: " + eagerTime + "ns");
        System.out.println("Bill Pugh: " + billPughTime + "ns");
        System.out.println("Enum: " + enumTime + "ns");
    }
}
```

## Performance Considerations

### Time Complexity
- **getInstance()**: O(1) for all patterns

### Space Complexity
- **O(1)** for single instance
- **O(n)** for parameterized singleton (n = number of parameters)

### Performance Ranking (Best to Worst)
1. Enum Singleton - Fastest, no synchronization
2. Bill Pugh Singleton - Excellent, lazy with no overhead
3. Eager Singleton - Good, simple
4. Double-Checked Locking - Good, but complex
5. Synchronized Lazy - Poor, synchronization overhead

## Common Pitfalls

1. **Not thread-safe**: Using lazy singleton without synchronization
2. **Reflection attacks**: Creating new instances via reflection
3. **Serialization issues**: Deserialization creates new instance
4. **Clone attacks**: Implementing Cloneable without override
5. **Circular dependencies**: Singletons depending on each other
6. **Memory leaks**: Holding references in thread-local
7. **Testing difficulties**: Hard to mock singletons

## Variations

1. **Multiton Pattern**: Multiple named instances
2. **Registry Pattern**: Central registry of singletons
3. **Monostate Pattern**: All instances share state
4. **Object Pool**: Reusable pool of objects
5. **Service Locator**: Central location for services

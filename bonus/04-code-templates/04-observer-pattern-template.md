# Observer Pattern Template

## Overview
Event-driven architecture implementation where objects (observers) register to receive notifications from a subject when its state changes.

## Use Cases
- Event handling systems
- Model-View-Controller (MVC) frameworks
- Publish-Subscribe messaging
- Real-time notifications
- Reactive programming
- Decoupled event propagation

## Implementation

### Simple Version
Basic observer pattern with direct notifications.

### Production Version
Event bus with type-safe subscribers and async handling.

### Thread-Safe Version
Fully concurrent with proper synchronization.

## Code

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.Consumer;
import java.util.stream.Collectors;

/**
 * SIMPLE VERSION: Basic Observer Pattern
 * - Simple subject-observer relationship
 * - Direct notification
 * - Manual registration/unregistration
 */
public class SimpleObserver {
    /**
     * Observer interface
     */
    public interface Observer {
        void update(String message);
    }

    /**
     * Subject that notifies observers
     */
    public static class Subject {
        private final List<Observer> observers = new ArrayList<>();

        /**
         * Register observer
         */
        public void attach(Observer observer) {
            if (!observers.contains(observer)) {
                observers.add(observer);
            }
        }

        /**
         * Unregister observer
         */
        public void detach(Observer observer) {
            observers.remove(observer);
        }

        /**
         * Notify all observers
         */
        public void notifyObservers(String message) {
            for (Observer observer : observers) {
                observer.update(message);
            }
        }

        /**
         * Get observer count
         */
        public int getObserverCount() {
            return observers.size();
        }
    }

    /**
     * Concrete observer implementation
     */
    public static class ConcreteObserver implements Observer {
        private final String name;

        public ConcreteObserver(String name) {
            this.name = name;
        }

        @Override
        public void update(String message) {
            System.out.println(name + " received: " + message);
        }
    }
}

/**
 * PRODUCTION VERSION: Event Bus with Type-Safe Subscribers
 * - Generic event handling
 * - Type-safe subscribers
 * - Async event processing
 * - Exception handling
 * - Subscriber lifecycle management
 */
public class EventBus {
    private final Map<Class<?>, List<EventSubscriber<?>>> subscribers;
    private final ExecutorService executorService;
    private final boolean asyncMode;
    private final ExceptionHandler exceptionHandler;

    @FunctionalInterface
    public interface EventHandler<T> {
        void handle(T event) throws Exception;
    }

    @FunctionalInterface
    public interface ExceptionHandler {
        void handle(Exception e, Object event);
    }

    /**
     * Event subscriber wrapper
     */
    private static class EventSubscriber<T> {
        private final Class<T> eventType;
        private final EventHandler<T> handler;
        private final String subscriberId;

        EventSubscriber(Class<T> eventType, EventHandler<T> handler, 
                       String subscriberId) {
            this.eventType = eventType;
            this.handler = handler;
            this.subscriberId = subscriberId;
        }

        void handle(T event) throws Exception {
            handler.handle(event);
        }
    }

    public EventBus(boolean asyncMode, ExceptionHandler exceptionHandler) {
        this.subscribers = new ConcurrentHashMap<>();
        this.asyncMode = asyncMode;
        this.exceptionHandler = exceptionHandler;
        this.executorService = asyncMode ? 
            Executors.newFixedThreadPool(10) : null;
    }

    /**
     * Subscribe to events of specific type
     */
    public <T> void subscribe(Class<T> eventType, EventHandler<T> handler) {
        subscribe(eventType, handler, UUID.randomUUID().toString());
    }

    /**
     * Subscribe with custom subscriber ID
     */
    public <T> void subscribe(Class<T> eventType, EventHandler<T> handler, 
                             String subscriberId) {
        subscribers.computeIfAbsent(eventType, k -> 
            new CopyOnWriteArrayList<>())
            .add(new EventSubscriber<>(eventType, handler, subscriberId));
    }

    /**
     * Unsubscribe from events
     */
    public <T> void unsubscribe(Class<T> eventType, String subscriberId) {
        List<EventSubscriber<?>> subs = subscribers.get(eventType);
        if (subs != null) {
            subs.removeIf(s -> s.subscriberId.equals(subscriberId));
        }
    }

    /**
     * Publish event to all subscribers
     */
    @SuppressWarnings("unchecked")
    public <T> void publish(T event) {
        Class<?> eventType = event.getClass();
        List<EventSubscriber<?>> subs = subscribers.get(eventType);

        if (subs == null || subs.isEmpty()) {
            return;
        }

        for (EventSubscriber<?> sub : subs) {
            EventSubscriber<T> typedSub = (EventSubscriber<T>) sub;

            if (asyncMode) {
                executorService.submit(() -> {
                    try {
                        typedSub.handle(event);
                    } catch (Exception e) {
                        if (exceptionHandler != null) {
                            exceptionHandler.handle(e, event);
                        }
                    }
                });
            } else {
                try {
                    typedSub.handle(event);
                } catch (Exception e) {
                    if (exceptionHandler != null) {
                        exceptionHandler.handle(e, event);
                    }
                }
            }
        }
    }

    /**
     * Get subscriber count for event type
     */
    public int getSubscriberCount(Class<?> eventType) {
        List<EventSubscriber<?>> subs = subscribers.get(eventType);
        return subs != null ? subs.size() : 0;
    }

    /**
     * Shutdown event bus
     */
    public void shutdown() throws InterruptedException {
        if (executorService != null) {
            executorService.shutdown();
            if (!executorService.awaitTermination(10, TimeUnit.SECONDS)) {
                executorService.shutdownNow();
            }
        }
        subscribers.clear();
    }
}

/**
 * THREAD-SAFE VERSION: Advanced Event Bus
 * - Weak references to prevent memory leaks
 * - Event filtering
 * - Priority-based event handling
 * - Event history
 */
public class AdvancedEventBus {
    private final Map<Class<?>, List<WeakSubscriber<?>>> subscribers;
    private final Queue<Object> eventHistory;
    private final int maxHistorySize;
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private final ExecutorService executorService;
    private final boolean trackHistory;

    @FunctionalInterface
    public interface EventFilter<T> {
        boolean accept(T event);
    }

    /**
     * Weak reference subscriber
     */
    private static class WeakSubscriber<T> {
        private final WeakReference<Consumer<T>> handler;
        private final EventFilter<T> filter;
        private final int priority;
        private final String id;

        WeakSubscriber(Consumer<T> handler, EventFilter<T> filter, 
                      int priority, String id) {
            this.handler = new WeakReference<>(handler);
            this.filter = filter;
            this.priority = priority;
            this.id = id;
        }

        boolean isAlive() {
            return handler.get() != null;
        }

        void handle(T event) {
            Consumer<T> h = handler.get();
            if (h != null && (filter == null || filter.accept(event))) {
                h.accept(event);
            }
        }
    }

    public AdvancedEventBus(int maxHistorySize, boolean trackHistory) {
        this.subscribers = new ConcurrentHashMap<>();
        this.eventHistory = new ConcurrentLinkedQueue<>();
        this.maxHistorySize = maxHistorySize;
        this.trackHistory = trackHistory;
        this.executorService = Executors.newFixedThreadPool(10);
    }

    /**
     * Subscribe with filter and priority
     */
    public <T> String subscribe(Class<T> eventType, Consumer<T> handler,
                               EventFilter<T> filter, int priority) {
        lock.writeLock().lock();
        try {
            String id = UUID.randomUUID().toString();
            List<WeakSubscriber<?>> subs = subscribers.computeIfAbsent(eventType, 
                k -> new CopyOnWriteArrayList<>());

            WeakSubscriber<T> subscriber = new WeakSubscriber<>(handler, filter, 
                priority, id);
            subs.add(subscriber);

            // Sort by priority (higher first)
            subs.sort((a, b) -> Integer.compare(
                ((WeakSubscriber<?>) b).priority, 
                ((WeakSubscriber<?>) a).priority));

            return id;
        } finally {
            lock.writeLock().unlock();
        }
    }

    /**
     * Unsubscribe
     */
    public <T> void unsubscribe(Class<T> eventType, String subscriberId) {
        lock.writeLock().lock();
        try {
            List<WeakSubscriber<?>> subs = subscribers.get(eventType);
            if (subs != null) {
                subs.removeIf(s -> s.id.equals(subscriberId));
            }
        } finally {
            lock.writeLock().unlock();
        }
    }

    /**
     * Publish event
     */
    @SuppressWarnings("unchecked")
    public <T> void publish(T event) {
        // Track history
        if (trackHistory) {
            eventHistory.offer(event);
            if (eventHistory.size() > maxHistorySize) {
                eventHistory.poll();
            }
        }

        lock.readLock().lock();
        try {
            List<WeakSubscriber<?>> subs = subscribers.get(event.getClass());
            if (subs == null) {
                return;
            }

            // Remove dead subscribers
            subs.removeIf(s -> !s.isAlive());

            // Handle event
            for (WeakSubscriber<?> sub : subs) {
                WeakSubscriber<T> typedSub = (WeakSubscriber<T>) sub;
                executorService.submit(() -> typedSub.handle(event));
            }
        } finally {
            lock.readLock().unlock();
        }
    }

    /**
     * Get event history
     */
    public List<Object> getEventHistory() {
        return new ArrayList<>(eventHistory);
    }

    /**
     * Get subscriber count
     */
    public int getSubscriberCount(Class<?> eventType) {
        lock.readLock().lock();
        try {
            List<WeakSubscriber<?>> subs = subscribers.get(eventType);
            return subs != null ? subs.size() : 0;
        } finally {
            lock.readLock().unlock();
        }
    }

    /**
     * Shutdown
     */
    public void shutdown() throws InterruptedException {
        lock.writeLock().lock();
        try {
            subscribers.clear();
            eventHistory.clear();
            executorService.shutdown();
            if (!executorService.awaitTermination(10, TimeUnit.SECONDS)) {
                executorService.shutdownNow();
            }
        } finally {
            lock.writeLock().unlock();
        }
    }
}

/**
 * Example Event Classes
 */
public class Events {
    public static class UserLoginEvent {
        public final String userId;
        public final long timestamp;

        public UserLoginEvent(String userId) {
            this.userId = userId;
            this.timestamp = System.currentTimeMillis();
        }
    }

    public static class UserLogoutEvent {
        public final String userId;
        public final long timestamp;

        public UserLogoutEvent(String userId) {
            this.userId = userId;
            this.timestamp = System.currentTimeMillis();
        }
    }

    public static class DataChangedEvent {
        public final String dataId;
        public final Object newValue;

        public DataChangedEvent(String dataId, Object newValue) {
            this.dataId = dataId;
            this.newValue = newValue;
        }
    }
}
```

## Testing

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ObserverPatternTest {

    @Test
    public void testSimpleObserver() {
        SimpleObserver.Subject subject = new SimpleObserver.Subject();
        SimpleObserver.ConcreteObserver observer1 = 
            new SimpleObserver.ConcreteObserver("Observer1");
        SimpleObserver.ConcreteObserver observer2 = 
            new SimpleObserver.ConcreteObserver("Observer2");

        subject.attach(observer1);
        subject.attach(observer2);

        assertEquals(2, subject.getObserverCount());

        subject.notifyObservers("Test message");

        subject.detach(observer1);
        assertEquals(1, subject.getObserverCount());
    }

    @Test
    public void testEventBusSync() throws Exception {
        EventBus eventBus = new EventBus(false, null);
        AtomicInteger counter = new AtomicInteger(0);

        eventBus.subscribe(Events.UserLoginEvent.class, event -> {
            counter.incrementAndGet();
        });

        eventBus.publish(new Events.UserLoginEvent("user1"));
        eventBus.publish(new Events.UserLoginEvent("user2"));

        assertEquals(2, counter.get());
        eventBus.shutdown();
    }

    @Test
    public void testEventBusAsync() throws Exception {
        EventBus eventBus = new EventBus(true, (e, event) -> {
            System.err.println("Error handling event: " + e.getMessage());
        });

        AtomicInteger counter = new AtomicInteger(0);
        eventBus.subscribe(Events.UserLoginEvent.class, event -> {
            counter.incrementAndGet();
        });

        eventBus.publish(new Events.UserLoginEvent("user1"));
        eventBus.publish(new Events.UserLoginEvent("user2"));

        Thread.sleep(500); // Wait for async processing
        assertEquals(2, counter.get());
        eventBus.shutdown();
    }

    @Test
    public void testMultipleEventTypes() throws Exception {
        EventBus eventBus = new EventBus(false, null);
        AtomicInteger loginCount = new AtomicInteger(0);
        AtomicInteger logoutCount = new AtomicInteger(0);

        eventBus.subscribe(Events.UserLoginEvent.class, 
            event -> loginCount.incrementAndGet());
        eventBus.subscribe(Events.UserLogoutEvent.class, 
            event -> logoutCount.incrementAndGet());

        eventBus.publish(new Events.UserLoginEvent("user1"));
        eventBus.publish(new Events.UserLogoutEvent("user1"));
        eventBus.publish(new Events.UserLoginEvent("user2"));

        assertEquals(2, loginCount.get());
        assertEquals(1, logoutCount.get());
        eventBus.shutdown();
    }

    @Test
    public void testAdvancedEventBusWithFilter() throws Exception {
        AdvancedEventBus eventBus = new AdvancedEventBus(100, true);
        AtomicInteger counter = new AtomicInteger(0);

        // Subscribe with filter - only accept user1
        eventBus.subscribe(Events.UserLoginEvent.class, 
            event -> counter.incrementAndGet(),
            event -> "user1".equals(event.userId),
            1);

        eventBus.publish(new Events.UserLoginEvent("user1"));
        eventBus.publish(new Events.UserLoginEvent("user2"));
        eventBus.publish(new Events.UserLoginEvent("user1"));

        Thread.sleep(100);
        assertEquals(2, counter.get());
        eventBus.shutdown();
    }

    @Test
    public void testEventHistory() throws Exception {
        AdvancedEventBus eventBus = new AdvancedEventBus(10, true);

        eventBus.subscribe(Events.UserLoginEvent.class, event -> {});

        for (int i = 0; i < 5; i++) {
            eventBus.publish(new Events.UserLoginEvent("user" + i));
        }

        Thread.sleep(100);
        assertEquals(5, eventBus.getEventHistory().size());
        eventBus.shutdown();
    }

    @Test
    public void testConcurrentPublishing() throws Exception {
        EventBus eventBus = new EventBus(true, null);
        AtomicInteger counter = new AtomicInteger(0);

        eventBus.subscribe(Events.UserLoginEvent.class, 
            event -> counter.incrementAndGet());

        int threadCount = 10;
        int eventsPerThread = 100;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);

        for (int t = 0; t < threadCount; t++) {
            executor.submit(() -> {
                for (int i = 0; i < eventsPerThread; i++) {
                    eventBus.publish(new Events.UserLoginEvent("user"));
                }
            });
        }

        executor.shutdown();
        assertTrue(executor.awaitTermination(10, TimeUnit.SECONDS));
        Thread.sleep(500);

        assertEquals(threadCount * eventsPerThread, counter.get());
        eventBus.shutdown();
    }
}
```

## Performance Considerations

### Time Complexity
- **subscribe()**: O(n log n) due to sorting by priority
- **publish()**: O(n) where n = number of subscribers
- **unsubscribe()**: O(n)

### Space Complexity
- **O(n)** where n = number of subscribers

### Optimization Tips
1. **Weak references**: Prevent memory leaks from dead subscribers
2. **Event filtering**: Reduce unnecessary processing
3. **Priority-based handling**: Process important events first
4. **Async processing**: Non-blocking event handling
5. **Event history**: Limited size to prevent memory issues

## Common Pitfalls

1. **Memory leaks**: Subscribers not unregistering
2. **Circular dependencies**: Events triggering more events
3. **Exception handling**: Exceptions in one subscriber affecting others
4. **Ordering issues**: Relying on subscriber order
5. **Deadlocks**: Lock contention in high-concurrency scenarios
6. **Event loss**: Async processing without acknowledgment
7. **Type safety**: Casting issues with generic events

## Variations

1. **Weak Event Pattern**: Using weak references
2. **Priority-based Events**: Handle important events first
3. **Filtered Events**: Only notify interested subscribers
4. **Async Event Bus**: Non-blocking event processing
5. **Distributed Event Bus**: Across multiple servers
6. **Reactive Streams**: RxJava-style event handling

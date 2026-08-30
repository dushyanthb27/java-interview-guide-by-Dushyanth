# Q25: Introduction to Reactive Programming in Java

[← Back to Multithreading & Concurrency](./README.md)

---

## Question
What is Reactive Programming? How does the Reactive Streams specification work in Java?

---

## Answer

### What is Reactive Programming?

Reactive programming is a paradigm built around **asynchronous data streams** and **backpressure**.

```
Imperative (Pull-based):              Reactive (Push-based):
┌────────────┐                        ┌────────────┐
│  Consumer   │── request data ──►    │  Producer   │── pushes data ──►
│            │◄── get response ──     │            │                   
│            │── request data ──►    │            │── pushes data ──►
│            │◄── get response ──     │            │                   
└────────────┘                        └────────────┘
                                      Consumer controls flow via backpressure
```

### Reactive Streams Specification (JDK 9)

Four core interfaces in `java.util.concurrent.Flow`:

```java
// 1. Publisher - source of data
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}

// 2. Subscriber - consumer of data
public interface Subscriber<T> {
    void onSubscribe(Subscription subscription);  // Called first
    void onNext(T item);                          // Each data item
    void onError(Throwable throwable);            // On failure
    void onComplete();                            // When done
}

// 3. Subscription - link between publisher and subscriber
public interface Subscription {
    void request(long n);  // Request n items (BACKPRESSURE!)
    void cancel();         // Cancel the subscription
}

// 4. Processor - both publisher and subscriber (transformer)
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
```

### Data Flow Sequence

```
Publisher                    Subscriber
   │                            │
   │◄──── subscribe(sub) ──────│
   │                            │
   │──── onSubscribe(sub) ─────►│
   │                            │
   │◄──── request(3) ──────────│  ← Backpressure: "I can handle 3 items"
   │                            │
   │──── onNext(item1) ────────►│
   │──── onNext(item2) ────────►│
   │──── onNext(item3) ────────►│
   │                            │
   │◄──── request(2) ──────────│  ← "Send me 2 more"
   │                            │
   │──── onNext(item4) ────────►│
   │──── onNext(item5) ────────►│
   │                            │
   │──── onComplete() ─────────►│  ← No more data
```

### Java 9 Flow API Example

```java
import java.util.concurrent.Flow.*;
import java.util.concurrent.SubmissionPublisher;

public class FlowApiDemo {
    public static void main(String[] args) throws InterruptedException {

        // SubmissionPublisher is the JDK's built-in publisher
        try (SubmissionPublisher<String> publisher = new SubmissionPublisher<>()) {

            // Create a subscriber
            Subscriber<String> subscriber = new Subscriber<>() {
                private Subscription subscription;

                @Override
                public void onSubscribe(Subscription subscription) {
                    this.subscription = subscription;
                    subscription.request(1); // Request first item
                    System.out.println("Subscribed!");
                }

                @Override
                public void onNext(String item) {
                    System.out.println("Received: " + item);
                    subscription.request(1); // Request next item (backpressure)
                }

                @Override
                public void onError(Throwable throwable) {
                    System.err.println("Error: " + throwable.getMessage());
                }

                @Override
                public void onComplete() {
                    System.out.println("All items received!");
                }
            };

            // Subscribe
            publisher.subscribe(subscriber);

            // Publish items
            publisher.submit("Hello");
            publisher.submit("Reactive");
            publisher.submit("World");
        }
        // close() calls onComplete() on all subscribers

        Thread.sleep(1000); // Wait for async processing
    }
}
```

### Processor (Transform Stream)

```java
public class TransformProcessor extends SubmissionPublisher<String>
        implements Flow.Processor<String, String> {

    private Flow.Subscription subscription;
    private final Function<String, String> transformer;

    public TransformProcessor(Function<String, String> transformer) {
        this.transformer = transformer;
    }

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        subscription.request(1);
    }

    @Override
    public void onNext(String item) {
        submit(transformer.apply(item)); // Transform and publish downstream
        subscription.request(1);
    }

    @Override
    public void onError(Throwable throwable) {
        closeExceptionally(throwable);
    }

    @Override
    public void onComplete() {
        close();
    }

    public static void main(String[] args) throws InterruptedException {
        SubmissionPublisher<String> publisher = new SubmissionPublisher<>();
        TransformProcessor uppercaseProcessor = new TransformProcessor(String::toUpperCase);

        // Chain: publisher → processor → subscriber
        publisher.subscribe(uppercaseProcessor);
        uppercaseProcessor.subscribe(new SimpleSubscriber());

        publisher.submit("hello");
        publisher.submit("reactive");
        publisher.submit("streams");
        publisher.close();

        Thread.sleep(1000);
    }

    static class SimpleSubscriber implements Flow.Subscriber<String> {
        private Flow.Subscription subscription;

        @Override
        public void onSubscribe(Flow.Subscription subscription) {
            this.subscription = subscription;
            subscription.request(Long.MAX_VALUE); // Unlimited demand
        }

        @Override
        public void onNext(String item) {
            System.out.println("Final: " + item); // HELLO, REACTIVE, STREAMS
        }

        @Override
        public void onError(Throwable t) { t.printStackTrace(); }

        @Override
        public void onComplete() { System.out.println("Done!"); }
    }
}
```

### Backpressure Strategies

```
┌──────────────────┬──────────────────────────────────────┐
│ Strategy         │ Description                          │
├──────────────────┼──────────────────────────────────────┤
│ Request(n)       │ Pull n items at a time               │
│ Buffer           │ Buffer items when consumer is slow   │
│ Drop             │ Drop items when buffer is full       │
│ Latest           │ Keep only the latest item            │
│ Error            │ Signal error when overwhelmed        │
│ Request(MAX)     │ Unbounded demand (no backpressure)   │
└──────────────────┴──────────────────────────────────────┘
```

```java
// Controlled backpressure example
public class BackpressureSubscriber implements Flow.Subscriber<Integer> {
    private Flow.Subscription subscription;
    private final int batchSize = 10;
    private int count = 0;

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        subscription.request(batchSize); // Request first batch
    }

    @Override
    public void onNext(Integer item) {
        process(item);
        count++;

        if (count >= batchSize) {
            count = 0;
            subscription.request(batchSize); // Request next batch
        }
    }

    @Override
    public void onError(Throwable t) { t.printStackTrace(); }

    @Override
    public void onComplete() { System.out.println("Done"); }

    private void process(Integer item) {
        System.out.println("Processing: " + item);
    }
}
```

### Reactive Libraries Overview

```
┌──────────────────────────────────────────────────────────┐
│ Java Reactive Ecosystem                                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Specification: Reactive Streams (java.util.concurrent.  │
│                 Flow in JDK 9+)                          │
│                                                          │
│  Implementations:                                        │
│  ┌──────────────────┐  ┌──────────────────┐              │
│  │ Project Reactor   │  │ RxJava 3         │             │
│  │ (Spring WebFlux)  │  │ (Android/Java)   │             │
│  │                   │  │                   │             │
│  │ Mono<T>  (0..1)   │  │ Single<T> (1)    │             │
│  │ Flux<T>  (0..N)   │  │ Maybe<T>  (0..1) │             │
│  │                   │  │ Observable (0..N) │             │
│  └──────────────────┘  │ Flowable (0..N+BP)│             │
│                         └──────────────────┘              │
│                                                          │
│  Frameworks: Spring WebFlux, Vert.x, Micronaut           │
└──────────────────────────────────────────────────────────┘
```

### Reactor Quick Preview

```java
// Project Reactor (Spring WebFlux uses this)

// Mono: 0 or 1 element
Mono<String> mono = Mono.just("Hello")
    .map(String::toUpperCase)
    .flatMap(s -> Mono.just(s + " World"));

// Flux: 0 to N elements
Flux<Integer> flux = Flux.range(1, 100)
    .filter(n -> n % 2 == 0)
    .map(n -> n * n)
    .take(10)
    .subscribe(System.out::println);

// Error handling
Mono.just("data")
    .flatMap(this::riskyOperation)
    .onErrorResume(ex -> Mono.just("fallback"))
    .subscribe(System.out::println);

// Backpressure with Flux
Flux.range(1, 1000)
    .onBackpressureBuffer(256)   // Buffer up to 256 items
    .onBackpressureDrop()        // Drop if buffer full
    .subscribe(new BaseSubscriber<Integer>() {
        @Override
        protected void hookOnSubscribe(Subscription s) {
            request(10); // Start with 10
        }

        @Override
        protected void hookOnNext(Integer value) {
            process(value);
            request(1); // Request one more after processing
        }
    });
```

### Reactive vs Imperative

```java
// ❌ Imperative approach (thread-per-request)
public UserProfile getUserProfile(long userId) {
    User user = userRepository.findById(userId);         // Blocks
    List<Order> orders = orderService.getOrders(userId); // Blocks
    Address address = addressService.getAddress(userId); // Blocks
    return new UserProfile(user, orders, address);
    // Total time = sum of all calls
    // Thread blocked the entire time
}

// ✅ Reactive approach (non-blocking)
public Mono<UserProfile> getUserProfileReactive(long userId) {
    Mono<User> user = userRepository.findById(userId);
    Mono<List<Order>> orders = orderService.getOrders(userId);
    Mono<Address> address = addressService.getAddress(userId);

    return Mono.zip(user, orders, address)
        .map(tuple -> new UserProfile(tuple.getT1(), tuple.getT2(), tuple.getT3()));
    // Total time = max of all calls (parallel!)
    // No thread blocked
}
```

### When to Use Reactive

```
✅ USE Reactive when:
  • High concurrency with many I/O-bound operations
  • Streaming data (WebSocket, SSE, real-time feeds)
  • Microservices with many downstream calls
  • Need fine-grained backpressure control
  • Event-driven architectures

❌ AVOID Reactive when:
  • Simple CRUD applications (adds complexity)
  • CPU-bound processing
  • Team is not familiar with reactive paradigm
  • Existing codebase is fully imperative
  • Debugging simplicity is a priority

⚠️ CONSIDER Virtual Threads as an alternative:
  • Virtual threads offer similar throughput benefits
  • With simpler, sequential programming model
  • No need to learn reactive operators
  • Better debugging and stack traces
```

---

## Anti-Patterns

❌ **Blocking inside reactive pipelines:**
```java
Mono.just(data)
    .map(d -> blockingDatabaseCall(d)) // ❌ Blocks the event loop!
    .subscribe();
```

❌ **Subscribing inside a reactive chain (nested subscribe):**
```java
Mono.just(data)
    .flatMap(d -> {
        otherMono.subscribe(); // ❌ Fire and forget inside chain
        return Mono.just(d);
    });
```

❌ **Ignoring backpressure with `request(Long.MAX_VALUE)` on slow consumers**

❌ **Using reactive for simple synchronous workflows (over-engineering)**

## Best Practices

✅ Use backpressure (`request(n)`) to control data flow  
✅ Never block inside reactive pipelines  
✅ Use `subscribeOn`/`publishOn` to control threading  
✅ Handle errors at every stage of the pipeline  
✅ Consider virtual threads (Java 21) as a simpler alternative  
✅ Use reactive testing utilities (StepVerifier in Reactor)

---

[← Structured Concurrency](./q24-structured-concurrency.md) | [Back to Section Index](./README.md)

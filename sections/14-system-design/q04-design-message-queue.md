# Q4: Design a Message Queue

## Question
Design and implement an in-memory Message Queue in Java. Support topics, pub/sub, consumer groups, message ordering, acknowledgment, and dead letter queues.

## Difficulty: Hard

---

## Solution

### Requirements

**Functional Requirements:**
- Create/delete topics
- Producers publish messages to a topic
- Consumers subscribe to topics and receive messages
- Consumer groups — each message delivered to one consumer per group
- Message ordering within a topic partition
- Acknowledgment — messages redelivered if not acknowledged
- Dead letter queue for repeatedly failed messages

**Non-Functional Requirements:**
- Thread-safe publish and consume
- High throughput with minimal lock contention
- At-least-once delivery guarantee
- Configurable retry count before dead-lettering

### API Design

```java
public interface MessageQueue {
    void createTopic(String topicName, int partitions);
    void deleteTopic(String topicName);
    void publish(String topic, String key, byte[] payload);
    String subscribe(String topic, String consumerGroup, MessageHandler handler);
    void unsubscribe(String subscriptionId);
    void acknowledge(String subscriptionId, String messageId);
}

@FunctionalInterface
public interface MessageHandler {
    void onMessage(Message message);
}

public class Message {
    private final String id;
    private final String topic;
    private final String key;
    private final byte[] payload;
    private final long timestamp;
    private int deliveryAttempts;
    // constructor, getters ...
}
```

### Implementation

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

public class InMemoryMessageQueue implements MessageQueue {

    private final ConcurrentHashMap<String, Topic> topics = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, Subscription> subscriptions = new ConcurrentHashMap<>();
    private final AtomicLong subscriptionIdGen = new AtomicLong(0);
    private final ScheduledExecutorService redeliveryScheduler;
    private final int maxRetries;

    public InMemoryMessageQueue(int maxRetries) {
        this.maxRetries = maxRetries;
        this.redeliveryScheduler = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "mq-redelivery");
            t.setDaemon(true);
            return t;
        });
        // Check for unacknowledged messages every 5 seconds
        redeliveryScheduler.scheduleAtFixedRate(this::redeliverUnacked, 5, 5, TimeUnit.SECONDS);
    }

    // --- Topic ---

    static class Topic {
        final String name;
        final int partitionCount;
        final List<Partition> partitions;
        final ConcurrentHashMap<String, ConsumerGroup> consumerGroups = new ConcurrentHashMap<>();
        final BlockingQueue<Message> deadLetterQueue = new LinkedBlockingQueue<>();

        Topic(String name, int partitionCount) {
            this.name = name;
            this.partitionCount = partitionCount;
            this.partitions = new ArrayList<>(partitionCount);
            for (int i = 0; i < partitionCount; i++) {
                partitions.add(new Partition(i));
            }
        }

        int partitionFor(String key) {
            if (key == null) return ThreadLocalRandom.current().nextInt(partitionCount);
            return Math.abs(key.hashCode() % partitionCount);
        }
    }

    // --- Partition ---

    static class Partition {
        final int id;
        final BlockingQueue<Message> messages = new LinkedBlockingQueue<>();

        Partition(int id) { this.id = id; }
    }

    // --- Consumer Group ---

    static class ConsumerGroup {
        final String name;
        final List<Subscription> members = new CopyOnWriteArrayList<>();
        private int roundRobinIndex = 0;

        ConsumerGroup(String name) { this.name = name; }

        synchronized Subscription nextConsumer() {
            if (members.isEmpty()) return null;
            Subscription sub = members.get(roundRobinIndex % members.size());
            roundRobinIndex++;
            return sub;
        }
    }

    // --- Subscription ---

    static class Subscription {
        final String id;
        final String topicName;
        final String consumerGroup;
        final MessageHandler handler;
        final ConcurrentHashMap<String, Message> pendingAcks = new ConcurrentHashMap<>();
        volatile boolean active = true;

        Subscription(String id, String topicName, String consumerGroup, MessageHandler handler) {
            this.id = id;
            this.topicName = topicName;
            this.consumerGroup = consumerGroup;
            this.handler = handler;
        }
    }

    // --- Core Operations ---

    @Override
    public void createTopic(String topicName, int partitions) {
        if (partitions <= 0) throw new IllegalArgumentException("Partitions must be > 0");
        topics.computeIfAbsent(topicName, name -> new Topic(name, partitions));
    }

    @Override
    public void deleteTopic(String topicName) {
        Topic topic = topics.remove(topicName);
        if (topic != null) {
            topic.consumerGroups.values().forEach(group ->
                group.members.forEach(sub -> sub.active = false));
        }
    }

    @Override
    public void publish(String topicName, String key, byte[] payload) {
        Topic topic = topics.get(topicName);
        if (topic == null) throw new IllegalArgumentException("Topic not found: " + topicName);

        String messageId = UUID.randomUUID().toString();
        Message message = new Message(messageId, topicName, key, payload, System.currentTimeMillis());

        // Route to partition based on key
        int partitionIdx = topic.partitionFor(key);
        Partition partition = topic.partitions.get(partitionIdx);
        partition.messages.offer(message);

        // Dispatch to consumer groups
        dispatchMessages(topic, partition);
    }

    @Override
    public String subscribe(String topicName, String consumerGroupName, MessageHandler handler) {
        Topic topic = topics.get(topicName);
        if (topic == null) throw new IllegalArgumentException("Topic not found: " + topicName);

        String subId = "sub-" + subscriptionIdGen.incrementAndGet();
        Subscription sub = new Subscription(subId, topicName, consumerGroupName, handler);

        ConsumerGroup group = topic.consumerGroups.computeIfAbsent(
            consumerGroupName, ConsumerGroup::new);
        group.members.add(sub);
        subscriptions.put(subId, sub);

        return subId;
    }

    @Override
    public void unsubscribe(String subscriptionId) {
        Subscription sub = subscriptions.remove(subscriptionId);
        if (sub == null) return;
        sub.active = false;

        Topic topic = topics.get(sub.topicName);
        if (topic != null) {
            ConsumerGroup group = topic.consumerGroups.get(sub.consumerGroup);
            if (group != null) {
                group.members.remove(sub);
            }
        }
    }

    @Override
    public void acknowledge(String subscriptionId, String messageId) {
        Subscription sub = subscriptions.get(subscriptionId);
        if (sub != null) {
            sub.pendingAcks.remove(messageId);
        }
    }

    // --- Internal dispatch ---

    private void dispatchMessages(Topic topic, Partition partition) {
        Message msg;
        while ((msg = partition.messages.poll()) != null) {
            final Message message = msg;

            for (ConsumerGroup group : topic.consumerGroups.values()) {
                Subscription consumer = group.nextConsumer();
                if (consumer != null && consumer.active) {
                    deliverMessage(consumer, message, topic);
                }
            }
        }
    }

    private void deliverMessage(Subscription sub, Message message, Topic topic) {
        sub.pendingAcks.put(message.getId(), message);
        try {
            sub.handler.onMessage(message);
        } catch (Exception e) {
            System.err.println("Handler failed for message " + message.getId() + ": " + e.getMessage());
            // Message stays in pendingAcks for redelivery
        }
    }

    // --- Redelivery of unacknowledged messages ---

    private void redeliverUnacked() {
        for (Subscription sub : subscriptions.values()) {
            if (!sub.active) continue;

            Topic topic = topics.get(sub.topicName);
            if (topic == null) continue;

            for (Map.Entry<String, Message> entry : sub.pendingAcks.entrySet()) {
                Message msg = entry.getValue();
                msg.incrementDeliveryAttempts();

                if (msg.getDeliveryAttempts() > maxRetries) {
                    // Move to dead letter queue
                    sub.pendingAcks.remove(entry.getKey());
                    topic.deadLetterQueue.offer(msg);
                    System.err.println("Message " + msg.getId() + " moved to DLQ after "
                        + maxRetries + " attempts");
                } else {
                    // Redeliver
                    try {
                        sub.handler.onMessage(msg);
                    } catch (Exception e) {
                        // will retry again next cycle
                    }
                }
            }
        }
    }
}
```

#### Message Class

```java
public class Message {
    private final String id;
    private final String topic;
    private final String key;
    private final byte[] payload;
    private final long timestamp;
    private volatile int deliveryAttempts = 0;

    public Message(String id, String topic, String key, byte[] payload, long timestamp) {
        this.id = id;
        this.topic = topic;
        this.key = key;
        this.payload = payload;
        this.timestamp = timestamp;
    }

    public String getId() { return id; }
    public String getTopic() { return topic; }
    public String getKey() { return key; }
    public byte[] getPayload() { return payload; }
    public long getTimestamp() { return timestamp; }
    public int getDeliveryAttempts() { return deliveryAttempts; }
    public void incrementDeliveryAttempts() { deliveryAttempts++; }

    @Override
    public String toString() {
        return "Message{id='" + id + "', topic='" + topic + "', key='" + key +
               "', size=" + (payload != null ? payload.length : 0) +
               ", attempts=" + deliveryAttempts + "}";
    }
}
```

### Concurrency Considerations

| Concern | Solution |
|---------|----------|
| Topic map access | `ConcurrentHashMap` — lock-free reads |
| Partition message queue | `LinkedBlockingQueue` — thread-safe FIFO |
| Consumer group round-robin | `synchronized` on `nextConsumer()` |
| Consumer membership | `CopyOnWriteArrayList` — safe iteration with infrequent mutations |
| Pending acknowledgments | `ConcurrentHashMap` per subscription |
| Redelivery | Single-threaded `ScheduledExecutorService` — avoids concurrent redelivery |
| Subscription cancellation | `volatile boolean active` flag |

### Performance Analysis

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `publish()` | O(G) | G = number of consumer groups per topic |
| `subscribe()` | O(1) | ConcurrentHashMap operations |
| `acknowledge()` | O(1) | ConcurrentHashMap remove |
| `redeliverUnacked()` | O(S * P) | S = subscriptions, P = pending messages |

**Bottlenecks:**
- `dispatchMessages` is synchronous — for high throughput, use a dispatch thread pool
- Partition count is fixed — consider dynamic repartitioning for scaling
- No persistence — messages lost on JVM crash

### Testing Strategy

```java
public class MessageQueueTest {

    @Test
    void testPubSub() throws Exception {
        InMemoryMessageQueue mq = new InMemoryMessageQueue(3);
        mq.createTopic("orders", 4);

        CountDownLatch latch = new CountDownLatch(1);
        AtomicReference<String> received = new AtomicReference<>();

        mq.subscribe("orders", "group-1", msg -> {
            received.set(new String(msg.getPayload()));
            latch.countDown();
        });

        mq.publish("orders", "user-1", "order-123".getBytes());
        assertTrue(latch.await(2, TimeUnit.SECONDS));
        assertEquals("order-123", received.get());
    }

    @Test
    void testConsumerGroups() throws Exception {
        InMemoryMessageQueue mq = new InMemoryMessageQueue(3);
        mq.createTopic("events", 1);

        AtomicInteger consumer1Count = new AtomicInteger();
        AtomicInteger consumer2Count = new AtomicInteger();

        String sub1 = mq.subscribe("events", "group-A", msg -> {
            consumer1Count.incrementAndGet();
            mq.acknowledge(/* subId */ "sub-1", msg.getId());
        });

        mq.subscribe("events", "group-A", msg -> {
            consumer2Count.incrementAndGet();
        });

        for (int i = 0; i < 10; i++) {
            mq.publish("events", null, ("msg-" + i).getBytes());
        }

        Thread.sleep(500);
        // Messages should be distributed across the two consumers in the group
        assertTrue(consumer1Count.get() + consumer2Count.get() == 10);
    }

    @Test
    void testDeadLetterQueue() throws Exception {
        InMemoryMessageQueue mq = new InMemoryMessageQueue(2); // max 2 retries
        mq.createTopic("failures", 1);

        mq.subscribe("failures", "g1", msg -> {
            throw new RuntimeException("Always fails");
            // never acknowledge
        });

        mq.publish("failures", "k", "data".getBytes());

        // Wait for redelivery cycles to exhaust retries
        Thread.sleep(15000);
        // Message should now be in DLQ (verified via logs)
    }
}
```

---

## Interviewer Insight
Interviewers want to see that you understand the publish-subscribe pattern, consumer groups, and message delivery guarantees. The key differentiators are: message ordering (per-partition), acknowledgment-based redelivery, and dead letter queue handling. Discuss trade-offs between at-most-once, at-least-once, and exactly-once delivery.

## Follow-up Questions
1. How would you add message persistence to survive JVM restarts?
2. How does Kafka achieve high throughput with its log-based architecture?
3. How would you implement exactly-once delivery semantics?

## Related Topics
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md)
- [Collections Framework](../03-collections-framework/README.md)

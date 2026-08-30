# Q4: Design a Message Queue

## 📋 Question
Design a message queue system. Discuss the data structures, message ordering, and delivery guarantees.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Publish and subscribe to messages
- Message ordering guarantees
- At-least-once and at-most-once delivery
- Dead letter queue for failed messages
- Message persistence

**Non-Functional Requirements:**
- High throughput (1M+ messages/sec)
- Low latency (< 100ms)
- Horizontal scalability
- Fault tolerance

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Producers                             │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                 Message Broker (Kafka-style)             │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Topic/Exchange                                     │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        │  │
│  │  │ Partition│ │ Partition│ │ Partition│        │  │
│  │  │    0     │ │    1     │ │    2     │        │  │
│  │  └──────────┘ └──────────┘ └──────────┘        │  │
│  └──────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Consumer 1  │ │  Consumer 2  │ │  Consumer 3  │
│  (Consumer   │ │  (Consumer   │ │  (Consumer   │
│   Group A)    │ │   Group B)    │ │   Group C)    │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Message Structure

```java
public class Message {
    private final String id;
    private final String topic;
    private final byte[] payload;
    private final long timestamp;
    private final Map<String, String> headers;
    private final int partitionKey;

    public Message(String id, String topic, byte[] payload, 
                   int partitionKey, Map<String, String> headers) {
        this.id = id;
        this.topic = topic;
        this.payload = payload;
        this.partitionKey = partitionKey;
        this.headers = headers;
        this.timestamp = System.currentTimeMillis();
    }

    // Getters
    public String getId() { return id; }
    public String getTopic() { return topic; }
    public byte[] getPayload() { return payload; }
    public long getTimestamp() { return timestamp; }
    public Map<String, String> getHeaders() { return headers; }
    public int getPartitionKey() { return partitionKey; }
}
```

### Partitioning Strategy

```java
public class Partitioner {
    private final int numPartitions;

    public Partitioner(int numPartitions) {
        this.numPartitions = numPartitions;
    }

    public int partition(String key, int numPartitions) {
        if (key == null) {
            return (int) (Math.random() * numPartitions);
        }
        return Math.abs(key.hashCode() % numPartitions);
    }

    // Consistent hashing for better distribution
    public int consistentHashPartition(String key) {
        if (key == null) {
            return (int) (Math.random() * numPartitions);
        }
        
        // Use consistent hashing
        int hash = key.hashCode();
        return ((hash & Integer.MAX_VALUE) % numPartitions);
    }
}
```

### Consumer Group Management

```java
public class ConsumerGroup {
    private final String groupId;
    private final String topic;
    private final Map<Integer, ConsumerPartition> partitions;
    private final int numConsumers;

    public ConsumerGroup(String groupId, String topic, int numPartitions, int numConsumers) {
        this.groupId = groupId;
        this.topic = topic;
        this.numConsumers = numConsumers;
        this.partitions = new HashMap<>();
        
        // Assign partitions to consumers
        for (int i = 0; i < numPartitions; i++) {
            int consumerId = i % numConsumers;
            partitions.put(i, new ConsumerPartition(consumerId, i));
        }
    }

    public int getPartitionForConsumer(int consumerId) {
        return partitions.entrySet().stream()
            .filter(entry -> entry.getValue().getConsumerId() == consumerId)
            .map(Map.Entry::getKey)
            .findFirst()
            .orElse(-1);
    }

    static class ConsumerPartition {
        private final int consumerId;
        private final int partitionId;
        private long offset;

        ConsumerPartition(int consumerId, int partitionId) {
            this.consumerId = consumerId;
            this.partitionId = partitionId;
            this.offset = 0;
        }

        int getConsumerId() { return consumerId; }
        int getPartitionId() { return partitionId; }
        long getOffset() { return offset; }
        void setOffset(long offset) { this.offset = offset; }
    }
}
```

### Message Queue Implementation

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

public class MessageQueue {
    private final String topic;
    private final int numPartitions;
    private final List<Partition> partitions;
    private final Partitioner partitioner;
    private final Map<String, ConsumerGroup> consumerGroups;

    public MessageQueue(String topic, int numPartitions) {
        this.topic = topic;
        this.numPartitions = numPartitions;
        this.partitions = new ArrayList<>();
        this.partitioner = new Partitioner(numPartitions);
        this.consumerGroups = new ConcurrentHashMap<>();

        for (int i = 0; i < numPartitions; i++) {
            partitions.add(new Partition(i));
        }
    }

    public void publish(Message message) {
        int partitionIndex = partitioner.partition(
            String.valueOf(message.getPartitionKey()), 
            numPartitions
        );
        partitions.get(partitionIndex).append(message);
    }

    public void subscribe(String groupId, int numConsumers) {
        ConsumerGroup group = new ConsumerGroup(groupId, topic, numPartitions, numConsumers);
        consumerGroups.put(groupId, group);
    }

    public Message poll(String groupId, int consumerId) {
        ConsumerGroup group = consumerGroups.get(groupId);
        if (group == null) return null;

        int partitionId = group.getPartitionForConsumer(consumerId);
        if (partitionId == -1) return null;

        return partitions.get(partitionId).poll();
    }

    static class Partition {
        private final int partitionId;
        private final BlockingQueue<Message> messages;
        private final AtomicLong offset = new AtomicLong(0);

        Partition(int partitionId) {
            this.partitionId = partitionId;
            this.messages = new LinkedBlockingQueue<>(10000);
        }

        void append(Message message) {
            messages.offer(message);
            offset.incrementAndGet();
        }

        Message poll() {
            return messages.poll();
        }

        long getOffset() { return offset.get(); }
    }
}
```

### Delivery Guarantees

#### At-Least-Once Delivery

```java
public class AtLeastOnceConsumer {
    private final Set<String> processedMessages = new HashSet<>();
    private final MessageQueue queue;
    private final String groupId;
    private final int consumerId;

    public AtLeastOnceConsumer(MessageQueue queue, String groupId, int consumerId) {
        this.queue = queue;
        this.groupId = groupId;
        this.consumerId = consumerId;
    }

    public void consume() {
        Message message = queue.poll(groupId, consumerId);
        if (message != null) {
            if (!processedMessages.contains(message.getId())) {
                processMessage(message);
                processedMessages.add(message.getId());
            }
        }
    }

    private void processMessage(Message message) {
        // Process the message
        System.out.println("Processing: " + message.getId());
    }
}
```

#### At-Most-Once Delivery

```java
public class AtMostOnceConsumer {
    private final MessageQueue queue;
    private final String groupId;
    private final int consumerId;

    public AtMostOnceConsumer(MessageQueue queue, String groupId, int consumerId) {
        this.queue = queue;
        this.groupId = groupId;
        this.consumerId = consumerId;
    }

    public void consume() {
        Message message = queue.poll(groupId, consumerId);
        if (message != null) {
            processMessage(message);
            // No deduplication - process immediately
        }
    }

    private void processMessage(Message message) {
        System.out.println("Processing: " + message.getId());
    }
}
```

### Dead Letter Queue

```java
public class DeadLetterQueue {
    private final BlockingQueue<Message> failedMessages;
    private final int maxRetries;

    public DeadLetterQueue(int maxRetries) {
        this.failedMessages = new LinkedBlockingQueue<>(1000);
        this.maxRetries = maxRetries;
    }

    public void addToDLQ(Message message, int retryCount) {
        if (retryCount >= maxRetries) {
            failedMessages.offer(message);
            System.out.println("Message sent to DLQ: " + message.getId());
        }
    }

    public Message poll() {
        return failedMessages.poll();
    }

    public int size() {
        return failedMessages.size();
    }
}
```

### Performance Considerations

| Metric | Target | Optimization |
|--------|--------|--------------|
| Throughput | 1M+ msg/sec | Batch processing, async I/O |
| Latency | < 100ms | Zero-copy, efficient serialization |
| Ordering | Per partition | Partition by key |
| Durability | Configurable | Async disk writes, replication |

---

## Interviewer Insight
Interviewers want to see that you understand message queue design: partitioning, consumer groups, delivery guarantees, and ordering. Strong candidates discuss at-least-once vs at-most-once delivery, dead letter queues, and partitioning strategies.

## Follow-up Questions
1. How do you ensure message ordering in a distributed system?
2. What is the difference between at-least-once and exactly-once delivery?
3. How do you handle consumer failures?

## 📚 Related Topics
- [Thread Pool Design](./01-design-thread-pool.md)
- [System Design Principles](../12-performance-optimization/README.md)
- [Concurrency Patterns](../04-multithreading-concurrency/README.md)

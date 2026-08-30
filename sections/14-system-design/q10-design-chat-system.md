# Q10: Design a Chat System

## Question
Design and implement a Chat System in Java. Support user management, one-on-one messaging, group chat, message routing, message persistence, online/offline status, and thread-safe message delivery.

## Difficulty: Hard

---

## Solution

### Requirements

**Functional Requirements:**
- User registration and authentication
- Send one-on-one (direct) messages
- Create group chats and send group messages
- Message history retrieval (paginated)
- Online/offline presence tracking
- Offline message delivery (when user comes online)
- Message read receipts
- Typing indicators

**Non-Functional Requirements:**
- Thread-safe message routing and delivery
- Real-time delivery via event-based model (simulating WebSocket)
- Message ordering guaranteed within a conversation
- Scalable to thousands of concurrent users
- Persistent message storage

### API Design

```java
public interface ChatService {
    // User management
    User register(String username);
    void connect(String userId, MessageListener listener);
    void disconnect(String userId);
    boolean isOnline(String userId);

    // Messaging
    Message sendDirectMessage(String senderId, String recipientId, String content);
    Message sendGroupMessage(String senderId, String groupId, String content);
    List<Message> getMessageHistory(String conversationId, int page, int pageSize);

    // Group management
    Group createGroup(String name, String creatorId, List<String> memberIds);
    void addGroupMember(String groupId, String userId);
    void removeGroupMember(String groupId, String userId);

    // Status
    void markAsRead(String userId, String conversationId, String messageId);
    Set<String> getOnlineUsers();
}

@FunctionalInterface
public interface MessageListener {
    void onMessage(ChatEvent event);
}

public enum EventType {
    NEW_MESSAGE, MESSAGE_READ, USER_ONLINE, USER_OFFLINE, TYPING
}
```

### Core Models

```java
import java.util.*;
import java.util.concurrent.atomic.AtomicLong;

public class User {
    private final String id;
    private final String username;
    private final long createdAt;

    public User(String id, String username) {
        this.id = id;
        this.username = username;
        this.createdAt = System.currentTimeMillis();
    }

    public String getId() { return id; }
    public String getUsername() { return username; }
    public long getCreatedAt() { return createdAt; }
}

public class Message {
    private static final AtomicLong ID_GEN = new AtomicLong(0);

    private final String id;
    private final String conversationId;
    private final String senderId;
    private final String content;
    private final long timestamp;
    private final Set<String> readBy;

    public Message(String conversationId, String senderId, String content) {
        this.id = String.valueOf(ID_GEN.incrementAndGet());
        this.conversationId = conversationId;
        this.senderId = senderId;
        this.content = content;
        this.timestamp = System.currentTimeMillis();
        this.readBy = ConcurrentHashMap.newKeySet();
        this.readBy.add(senderId); // sender has implicitly read it
    }

    public String getId() { return id; }
    public String getConversationId() { return conversationId; }
    public String getSenderId() { return senderId; }
    public String getContent() { return content; }
    public long getTimestamp() { return timestamp; }
    public Set<String> getReadBy() { return Collections.unmodifiableSet(readBy); }
    public void markReadBy(String userId) { readBy.add(userId); }
}

public class Group {
    private final String id;
    private final String name;
    private final String creatorId;
    private final Set<String> memberIds;
    private final long createdAt;

    public Group(String id, String name, String creatorId, Collection<String> memberIds) {
        this.id = id;
        this.name = name;
        this.creatorId = creatorId;
        this.memberIds = ConcurrentHashMap.newKeySet();
        this.memberIds.addAll(memberIds);
        this.memberIds.add(creatorId);
        this.createdAt = System.currentTimeMillis();
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getCreatorId() { return creatorId; }
    public Set<String> getMemberIds() { return Collections.unmodifiableSet(memberIds); }
    public void addMember(String userId) { memberIds.add(userId); }
    public void removeMember(String userId) { memberIds.remove(userId); }
    public long getCreatedAt() { return createdAt; }
}

public class ChatEvent {
    private final EventType type;
    private final Message message;
    private final String userId;
    private final String conversationId;

    private ChatEvent(EventType type, Message message, String userId, String conversationId) {
        this.type = type;
        this.message = message;
        this.userId = userId;
        this.conversationId = conversationId;
    }

    public static ChatEvent newMessage(Message msg) {
        return new ChatEvent(EventType.NEW_MESSAGE, msg, msg.getSenderId(), msg.getConversationId());
    }

    public static ChatEvent userOnline(String userId) {
        return new ChatEvent(EventType.USER_ONLINE, null, userId, null);
    }

    public static ChatEvent userOffline(String userId) {
        return new ChatEvent(EventType.USER_OFFLINE, null, userId, null);
    }

    public static ChatEvent messageRead(String userId, String conversationId, String messageId) {
        return new ChatEvent(EventType.MESSAGE_READ, null, userId, conversationId);
    }

    public EventType getType() { return type; }
    public Message getMessage() { return message; }
    public String getUserId() { return userId; }
    public String getConversationId() { return conversationId; }
}
```

### Implementation

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

public class InMemoryChatService implements ChatService {

    // Registries
    private final ConcurrentHashMap<String, User> users = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, Group> groups = new ConcurrentHashMap<>();
    private final AtomicLong userIdGen = new AtomicLong(0);
    private final AtomicLong groupIdGen = new AtomicLong(0);

    // Online presence & message delivery
    private final ConcurrentHashMap<String, MessageListener> onlineUsers = new ConcurrentHashMap<>();

    // Message storage: conversationId -> ordered messages
    private final ConcurrentHashMap<String, CopyOnWriteArrayList<Message>> messageStore =
        new ConcurrentHashMap<>();

    // Offline message queue: userId -> pending events
    private final ConcurrentHashMap<String, ConcurrentLinkedQueue<ChatEvent>> offlineQueue =
        new ConcurrentHashMap<>();

    // Thread pool for async delivery
    private final ExecutorService deliveryExecutor =
        Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors(), r -> {
            Thread t = new Thread(r, "chat-delivery");
            t.setDaemon(true);
            return t;
        });

    // --- User Management ---

    @Override
    public User register(String username) {
        String userId = "user-" + userIdGen.incrementAndGet();
        User user = new User(userId, username);
        users.put(userId, user);
        offlineQueue.put(userId, new ConcurrentLinkedQueue<>());
        return user;
    }

    @Override
    public void connect(String userId, MessageListener listener) {
        if (!users.containsKey(userId)) {
            throw new IllegalArgumentException("User not found: " + userId);
        }

        onlineUsers.put(userId, listener);

        // Deliver queued offline messages
        ConcurrentLinkedQueue<ChatEvent> pending = offlineQueue.get(userId);
        if (pending != null) {
            ChatEvent event;
            while ((event = pending.poll()) != null) {
                deliverToUser(userId, event);
            }
        }

        // Broadcast online status
        broadcastPresence(ChatEvent.userOnline(userId), userId);
    }

    @Override
    public void disconnect(String userId) {
        onlineUsers.remove(userId);
        broadcastPresence(ChatEvent.userOffline(userId), userId);
    }

    @Override
    public boolean isOnline(String userId) {
        return onlineUsers.containsKey(userId);
    }

    @Override
    public Set<String> getOnlineUsers() {
        return Collections.unmodifiableSet(onlineUsers.keySet());
    }

    // --- Direct Messaging ---

    @Override
    public Message sendDirectMessage(String senderId, String recipientId, String content) {
        validateUser(senderId);
        validateUser(recipientId);

        String conversationId = getDirectConversationId(senderId, recipientId);
        Message message = new Message(conversationId, senderId, content);

        // Persist
        messageStore.computeIfAbsent(conversationId, k -> new CopyOnWriteArrayList<>()).add(message);

        // Deliver
        ChatEvent event = ChatEvent.newMessage(message);
        deliverToUser(recipientId, event);
        deliverToUser(senderId, event); // echo to sender for multi-device

        return message;
    }

    // --- Group Messaging ---

    @Override
    public Message sendGroupMessage(String senderId, String groupId, String content) {
        validateUser(senderId);
        Group group = groups.get(groupId);
        if (group == null) throw new IllegalArgumentException("Group not found: " + groupId);
        if (!group.getMemberIds().contains(senderId)) {
            throw new IllegalArgumentException("User is not a member of this group");
        }

        String conversationId = "group-" + groupId;
        Message message = new Message(conversationId, senderId, content);

        // Persist
        messageStore.computeIfAbsent(conversationId, k -> new CopyOnWriteArrayList<>()).add(message);

        // Deliver to all group members
        ChatEvent event = ChatEvent.newMessage(message);
        for (String memberId : group.getMemberIds()) {
            deliverToUser(memberId, event);
        }

        return message;
    }

    // --- Message History ---

    @Override
    public List<Message> getMessageHistory(String conversationId, int page, int pageSize) {
        CopyOnWriteArrayList<Message> messages = messageStore.get(conversationId);
        if (messages == null) return Collections.emptyList();

        int totalMessages = messages.size();
        int start = Math.max(0, totalMessages - (page * pageSize));
        int end = Math.max(0, totalMessages - ((page - 1) * pageSize));

        if (start >= end) return Collections.emptyList();

        List<Message> pageMessages = new ArrayList<>(messages.subList(start, end));
        Collections.reverse(pageMessages); // newest first
        return pageMessages;
    }

    // --- Group Management ---

    @Override
    public Group createGroup(String name, String creatorId, List<String> memberIds) {
        validateUser(creatorId);
        memberIds.forEach(this::validateUser);

        String groupId = "group-" + groupIdGen.incrementAndGet();
        Group group = new Group(groupId, name, creatorId, memberIds);
        groups.put(groupId, group);

        return group;
    }

    @Override
    public void addGroupMember(String groupId, String userId) {
        validateUser(userId);
        Group group = groups.get(groupId);
        if (group == null) throw new IllegalArgumentException("Group not found: " + groupId);
        group.addMember(userId);
    }

    @Override
    public void removeGroupMember(String groupId, String userId) {
        Group group = groups.get(groupId);
        if (group == null) throw new IllegalArgumentException("Group not found: " + groupId);
        group.removeMember(userId);
    }

    // --- Read Receipts ---

    @Override
    public void markAsRead(String userId, String conversationId, String messageId) {
        CopyOnWriteArrayList<Message> messages = messageStore.get(conversationId);
        if (messages == null) return;

        for (Message msg : messages) {
            if (msg.getId().equals(messageId)) {
                msg.markReadBy(userId);
                // Notify the sender about the read receipt
                ChatEvent readEvent = ChatEvent.messageRead(userId, conversationId, messageId);
                deliverToUser(msg.getSenderId(), readEvent);
                break;
            }
        }
    }

    // --- Internal Delivery ---

    private void deliverToUser(String userId, ChatEvent event) {
        MessageListener listener = onlineUsers.get(userId);

        if (listener != null) {
            // User is online — deliver asynchronously
            deliveryExecutor.submit(() -> {
                try {
                    listener.onMessage(event);
                } catch (Exception e) {
                    System.err.println("Delivery failed for " + userId + ": " + e.getMessage());
                }
            });
        } else {
            // User is offline — queue for later delivery
            ConcurrentLinkedQueue<ChatEvent> queue = offlineQueue.get(userId);
            if (queue != null) {
                queue.offer(event);
            }
        }
    }

    private void broadcastPresence(ChatEvent event, String excludeUserId) {
        for (String userId : onlineUsers.keySet()) {
            if (!userId.equals(excludeUserId)) {
                deliverToUser(userId, event);
            }
        }
    }

    private String getDirectConversationId(String userId1, String userId2) {
        // Deterministic conversation ID regardless of who initiates
        String first = userId1.compareTo(userId2) < 0 ? userId1 : userId2;
        String second = userId1.compareTo(userId2) < 0 ? userId2 : userId1;
        return "dm-" + first + "-" + second;
    }

    private void validateUser(String userId) {
        if (!users.containsKey(userId)) {
            throw new IllegalArgumentException("User not found: " + userId);
        }
    }
}
```

### Architecture Diagram

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Client A   │     │   Client B   │     │   Client C   │
│  (Listener)  │     │  (Listener)  │     │  (Offline)   │
└──────┬───────┘     └──────┬───────┘     └──────────────┘
       │                    │                      │
       ▼                    ▼                      │
┌──────────────────────────────────────────────────┤
│               ChatService                        │
│  ┌─────────────────────────────────────┐         │
│  │  onlineUsers: userId → Listener     │         │
│  │  messageStore: convId → [Messages]  │         │
│  │  offlineQueue: userId → Queue       │         │
│  └─────────────────────────────────────┘         │
│                                                  │
│  ┌──────────────────┐  ┌───────────────┐         │
│  │  Delivery Thread  │  │  Delivery     │◄────── │
│  │  Pool (async)     │  │  Thread Pool  │  (queued for
│  └──────────────────┘  └───────────────┘   offline user)
└──────────────────────────────────────────────────┘
```

### Concurrency Considerations

| Concern | Solution |
|---------|----------|
| User registry | `ConcurrentHashMap` — lock-free reads |
| Online presence | `ConcurrentHashMap` for `onlineUsers` map |
| Message storage | `CopyOnWriteArrayList` — thread-safe append + iteration |
| Message delivery | Async via `ExecutorService` — non-blocking for sender |
| Offline queue | `ConcurrentLinkedQueue` — lock-free enqueue/dequeue |
| Group membership | `ConcurrentHashMap.newKeySet()` — thread-safe set |
| Message ordering | Single-threaded per conversation (via ordered list) |
| Read receipts | `ConcurrentHashMap.newKeySet()` per message |
| Conversation ID | Deterministic (sorted user IDs) — avoids duplicates |

### Performance Analysis

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `sendDirectMessage` | O(1) | Store + deliver to 1-2 users |
| `sendGroupMessage` | O(M) | M = group members |
| `getMessageHistory` | O(pageSize) | Sublist of CopyOnWriteArrayList |
| `connect` | O(Q) | Q = queued offline messages |
| `markAsRead` | O(N) | N = messages in conversation (linear scan) |

**Bottlenecks & Improvements:**
- `CopyOnWriteArrayList` is expensive for writes at scale — replace with an append-only log or `ConcurrentLinkedDeque`
- Message history pagination should use an indexed structure or database
- Offline queue should be bounded to prevent memory leaks
- For production: use WebSocket connections, message broker (Kafka), and database persistence

### Testing Strategy

```java
public class ChatServiceTest {

    @Test
    void testDirectMessage() {
        InMemoryChatService chat = new InMemoryChatService();
        User alice = chat.register("Alice");
        User bob = chat.register("Bob");

        List<Message> received = Collections.synchronizedList(new ArrayList<>());

        chat.connect(bob.getId(), event -> {
            if (event.getType() == EventType.NEW_MESSAGE) {
                received.add(event.getMessage());
            }
        });

        chat.sendDirectMessage(alice.getId(), bob.getId(), "Hello Bob!");

        // Wait for async delivery
        await().atMost(2, TimeUnit.SECONDS).until(() -> received.size() == 1);
        assertEquals("Hello Bob!", received.get(0).getContent());
    }

    @Test
    void testGroupMessage() throws Exception {
        InMemoryChatService chat = new InMemoryChatService();
        User alice = chat.register("Alice");
        User bob = chat.register("Bob");
        User charlie = chat.register("Charlie");

        AtomicInteger bobCount = new AtomicInteger();
        AtomicInteger charlieCount = new AtomicInteger();

        chat.connect(bob.getId(), e -> { if (e.getType() == EventType.NEW_MESSAGE) bobCount.incrementAndGet(); });
        chat.connect(charlie.getId(), e -> { if (e.getType() == EventType.NEW_MESSAGE) charlieCount.incrementAndGet(); });

        Group group = chat.createGroup("Team", alice.getId(),
            List.of(bob.getId(), charlie.getId()));

        chat.connect(alice.getId(), e -> {}); // alice online
        chat.sendGroupMessage(alice.getId(), group.getId(), "Hello team!");

        Thread.sleep(500);
        assertTrue(bobCount.get() >= 1);
        assertTrue(charlieCount.get() >= 1);
    }

    @Test
    void testOfflineMessageDelivery() throws Exception {
        InMemoryChatService chat = new InMemoryChatService();
        User alice = chat.register("Alice");
        User bob = chat.register("Bob");

        // Bob is offline — message should be queued
        chat.sendDirectMessage(alice.getId(), bob.getId(), "Are you there?");

        List<Message> received = Collections.synchronizedList(new ArrayList<>());

        // Bob comes online — should receive queued message
        chat.connect(bob.getId(), event -> {
            if (event.getType() == EventType.NEW_MESSAGE) {
                received.add(event.getMessage());
            }
        });

        Thread.sleep(500);
        assertEquals(1, received.size());
        assertEquals("Are you there?", received.get(0).getContent());
    }

    @Test
    void testMessageHistory() {
        InMemoryChatService chat = new InMemoryChatService();
        User alice = chat.register("Alice");
        User bob = chat.register("Bob");

        for (int i = 1; i <= 25; i++) {
            chat.sendDirectMessage(alice.getId(), bob.getId(), "Message " + i);
        }

        String convId = "dm-" + (alice.getId().compareTo(bob.getId()) < 0
            ? alice.getId() + "-" + bob.getId()
            : bob.getId() + "-" + alice.getId());

        List<Message> page1 = chat.getMessageHistory(convId, 1, 10);
        assertEquals(10, page1.size());

        List<Message> page3 = chat.getMessageHistory(convId, 3, 10);
        assertEquals(5, page3.size()); // 25 - 20 = 5 remaining
    }

    @Test
    void testPresenceTracking() {
        InMemoryChatService chat = new InMemoryChatService();
        User alice = chat.register("Alice");

        assertFalse(chat.isOnline(alice.getId()));

        chat.connect(alice.getId(), e -> {});
        assertTrue(chat.isOnline(alice.getId()));

        chat.disconnect(alice.getId());
        assertFalse(chat.isOnline(alice.getId()));
    }

    @Test
    void testReadReceipts() throws Exception {
        InMemoryChatService chat = new InMemoryChatService();
        User alice = chat.register("Alice");
        User bob = chat.register("Bob");

        chat.connect(alice.getId(), e -> {});
        chat.connect(bob.getId(), e -> {});

        Message msg = chat.sendDirectMessage(alice.getId(), bob.getId(), "Read this");

        String convId = msg.getConversationId();
        chat.markAsRead(bob.getId(), convId, msg.getId());

        assertTrue(msg.getReadBy().contains(bob.getId()));
    }
}
```

---

## Interviewer Insight
This is a comprehensive system design question that tests multiple skills: data modeling (User, Message, Group), real-time communication patterns (pub/sub via listeners), offline handling, and concurrency. Interviewers look for clean separation between message routing and storage, awareness of delivery guarantees, and how you'd scale to production (WebSockets, message brokers, database persistence, horizontal scaling with consistent hashing).

## Follow-up Questions
1. How would you scale this to millions of concurrent users across multiple servers?
2. How would you implement end-to-end encryption for messages?
3. How would you handle message ordering in a distributed system with eventual consistency?

## Related Topics
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md)
- [Collections Framework](../03-collections-framework/README.md)
- [Java I/O & NIO](../08-java-io-nio/README.md)

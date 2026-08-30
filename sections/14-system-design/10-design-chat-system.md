# Q10: Design a Chat System

## 📋 Question
Design a real-time chat system. Discuss the data structures, message delivery, and scalability considerations.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Real-time messaging
- Group chats and direct messages
- Message persistence
- Online/offline status
- Read receipts
- Message history

**Non-Functional Requirements:**
- Low latency (< 200ms message delivery)
- High availability
- Scalable to millions of users
- Message ordering guarantees

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Client Apps (Web/Mobile)              │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                  Load Balancer (WebSocket)               │
└──────────────────────┬──────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Chat Server  │ │  Chat Server  │ │  Chat Server  │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │               │               │
       └───────────────┼───────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│              Message Broker (Kafka)                     │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│              Message Database (MongoDB)                  │
└─────────────────────────────────────────────────────────┘
```

### Message Structure

```java
public class ChatMessage {
    private final String messageId;
    private final String senderId;
    private final String receiverId; // null for group chats
    private final String groupId;    // null for direct messages
    private final String content;
    private final MessageType type;
    private final long timestamp;
    private final MessageStatus status;

    public ChatMessage(String messageId, String senderId, String receiverId, 
                        String groupId, String content, MessageType type) {
        this.messageId = messageId;
        this.senderId = senderId;
        this.receiverId = receiverId;
        this.groupId = groupId;
        this.content = content;
        this.type = type;
        this.timestamp = System.currentTimeMillis();
        this.status = MessageStatus.SENT;
    }

    // Getters
    public String getMessageId() { return messageId; }
    public String getSenderId() { return senderId; }
    public String getReceiverId() { return receiverId; }
    public String getGroupId() { return groupId; }
    public String getContent() { return content; }
    public MessageType getType() { return type; }
    public long getTimestamp() { return timestamp; }
    public MessageStatus getStatus() { return status; }

    public void setStatus(MessageStatus status) {
        this.status = status;
    }
}

public enum MessageType {
    TEXT, IMAGE, FILE, SYSTEM
}

public enum MessageStatus {
    SENT, DELIVERED, READ, FAILED
}
```

### WebSocket Handler

```java
import org.springframework.web.socket.*;
import org.springframework.web.socket.handler.TextWebSocketHandler;
import java.util.*;

public class ChatWebSocketHandler extends TextWebSocketHandler {
    private final Map<String, WebSocketSession> userSessions;
    private final Map<String, Set<String>> groupMembers;

    public ChatWebSocketHandler() {
        this.userSessions = new ConcurrentHashMap<>();
        this.groupMembers = new ConcurrentHashMap<>();
    }

    @Override
    public void afterConnectionEstablished(WebSocketSession session, Principal principal) throws Exception {
        String userId = principal.getName();
        userSessions.put(userId, session);
        System.out.println("User connected: " + userId);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        String payload = message.getPayload();
        // Parse and handle message
        ChatMessage chatMessage = parseMessage(payload);
        
        if (chatMessage.getGroupId() != null) {
            // Group message
            broadcastToGroup(chatMessage.getGroupId(), chatMessage);
        } else {
            // Direct message
            sendToUser(chatMessage.getReceiverId(), chatMessage);
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
        // Remove user from sessions
        userSessions.values().remove(session);
    }

    private void broadcastToGroup(String groupId, ChatMessage message) {
        Set<String> members = groupMembers.get(groupId);
        if (members != null) {
            for (String userId : members) {
                WebSocketSession session = userSessions.get(userId);
                if (session != null && session.isOpen()) {
                    session.sendMessage(new TextMessage(serializeMessage(message)));
                }
            }
        }
    }

    private void sendToUser(String userId, ChatMessage message) {
        WebSocketSession session = userSessions.get(userId);
        if (session != null && session.isOpen()) {
            session.sendMessage(new TextMessage(serializeMessage(message)));
        }
    }

    private String serializeMessage(ChatMessage message) {
        // Convert message to JSON
        return "{\"messageId\":\"" + message.getMessageId() + 
               "\",\"senderId\":\"" + message.getSenderId() + 
               "\",\"content\":\"" + message.getContent() + "\"}";
    }

    private ChatMessage parseMessage(String payload) {
        // Parse JSON to ChatMessage
        // Simplified for example
        return new ChatMessage(UUID.randomUUID().toString(), 
                                  "user1", "user2", null, 
                                  "Hello", MessageType.TEXT);
    }
}
```

### Message Persistence

```java
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.data.mongodb.core.query.Update;
import org.springframework.stereotype.Repository;
import java.util.*;

@Repository
public class ChatMessageRepository {
    private final MongoTemplate mongoTemplate;

    public ChatMessageRepository(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }

    public void saveMessage(ChatMessage message) {
        mongoTemplate.save(message);
    }

    public List<ChatMessage> getConversationHistory(String userId1, String userId2, int limit) {
        Query query = new Query();
        query.addCriteria(Criteria.where("senderId").is(userId1).and("receiverId").is(userId2))
               .or(Criteria.where("senderId").is(userId2).and("receiverId").is(userId1));
        query.with(Sort.by(Sort.Direction.DESC, "timestamp"));
        query.limit(limit);
        
        return mongoTemplate.find(query, ChatMessage.class);
    }

    public List<ChatMessage> getGroupHistory(String groupId, int limit) {
        Query query = new Query(Criteria.where("groupId").is(groupId));
        query.with(Sort.by(Sort.Direction.DESC, "timestamp"));
        query.limit(limit);
        
        return mongoTemplate.find(query, ChatMessage.class);
    }

    public void markAsRead(String messageId) {
        Query query = new Query(Criteria.where("messageId").is(messageId));
        Update update = Update.update("status", MessageStatus.READ);
        mongoTemplate.updateFirst(query, update, ChatMessage.class);
    }
}
```

### Online Status Management

```java
import java.util.*;
import java.util.concurrent.*;

public class UserStatusManager {
    private final Map<String, UserStatus> userStatuses;
    private final ScheduledExecutorService scheduler;

    public UserStatusManager() {
        this.userStatuses = new ConcurrentHashMap<>();
        this.scheduler = Executors.newSingleThreadScheduledExecutor();
        
        // Cleanup inactive users every minute
        scheduler.scheduleAtFixedRate(this::cleanupInactiveUsers, 1, 1, TimeUnit.MINUTES);
    }

    public void setUserOnline(String userId) {
        userStatuses.put(userId, new UserStatus(userId, true, System.currentTimeMillis()));
    }

    public void setUserOffline(String userId) {
        UserStatus status = userStatuses.get(userId);
        if (status != null) {
            status.setOnline(false);
            status.setLastSeen(System.currentTimeMillis());
        }
    }

    public boolean isUserOnline(String userId) {
        UserStatus status = userStatuses.get(userId);
        return status != null && status.isOnline();
    }

    public List<String> getOnlineUsers() {
        List<String> onlineUsers = new ArrayList<>();
        for (Map.Entry<String, UserStatus> entry : userStatuses.entrySet()) {
            if (entry.getValue().isOnline()) {
                onlineUsers.add(entry.getKey());
            }
        }
        return onlineUsers;
    }

    private void cleanupInactiveUsers() {
        long now = System.currentTimeMillis();
        long inactiveThreshold = TimeUnit.MINUTES.toMillis(5);
        
        userStatuses.entrySet().removeIf(entry -> {
            UserStatus status = entry.getValue();
            return !status.isOnline() && (now - status.getLastSeen() > inactiveThreshold);
        });
    }

    static class UserStatus {
        private final String userId;
        private boolean online;
        private long lastSeen;

        UserStatus(String userId, boolean online, long lastSeen) {
            this.userId = userId;
            this.online = online;
            this.lastSeen = lastSeen;
        }

        boolean isOnline() { return online; }
        void setOnline(boolean online) { this.online = online; }
        long getLastSeen() { return lastSeen; }
        void setLastSeen(long lastSeen) { this.lastSeen = lastSeen; }
    }
}
```

### Message Queue Integration

```java
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import java.util.concurrent.CompletableFuture;

public class MessageProducer {
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final String topicName;

    public MessageProducer(KafkaTemplate<String, String> kafkaTemplate, String topicName) {
        this.kafkaTemplate = kafkaTemplate;
        this.topicName = topicName;
    }

    public CompletableFuture<Void> sendMessage(ChatMessage message) {
        String key = message.getGroupId() != null ? message.getGroupId() : message.getReceiverId();
        String value = serializeMessage(message);
        
        return kafkaTemplate.send(topicName, key, value);
    }

    private String serializeMessage(ChatMessage message) {
        // Convert to JSON
        return "{\"messageId\":\"" + message.getMessageId + 
               "\",\"senderId\":\"" + message.getSenderId() + 
               "\",\"content\":\"" + message.getContent() + "\"}";
    }
}

public class MessageConsumer {
    private final ChatWebSocketHandler webSocketHandler;
    private final ChatMessageRepository messageRepository;

    public MessageConsumer(ChatWebSocketHandler webSocketHandler, 
                        ChatMessageRepository messageRepository) {
        this.webSocketHandler = webSocketHandler;
        this.messageRepository = messageRepository;
    }

    public void handleMessage(String messageJson) {
        ChatMessage message = parseMessage(messageJson);
        
        // Persist message
        messageRepository.saveMessage(message);
        
        // Deliver via WebSocket
        if (message.getGroupId() != null) {
            webSocketHandler.broadcastToGroup(message.getGroupId(), message);
        } else {
            webSocketHandler.sendToUser(message.getReceiverId(), message);
        }
    }

    private ChatMessage parseMessage(String json) {
        // Parse JSON to ChatMessage
        return new ChatMessage(UUID.randomUUID().toString(), 
                                  "user1", "user2", null, 
                                  "Hello", MessageType.TEXT);
    }
}
```

### Performance Considerations

| Aspect | Solution | Impact |
|--------|----------|--------|
| Message delivery | WebSocket for real-time | Low latency |
| Persistence | Async MongoDB writes | Non-blocking |
| Scalability | Kafka message broker | Horizontal scaling |
| Online status | In-memory with TTL cleanup | Low overhead |
| History retrieval | MongoDB with pagination | Efficient queries |

---

## Interviewer Insight
Interviewers want to see that you understand chat system design: real-time communication, message persistence, and scalability. Strong candidates discuss WebSocket, message queues, and database design for chat applications.

## Follow-up Questions
1. How do you handle message ordering in a distributed chat system?
2. What is the difference between direct messages and group chats in terms of architecture?
3. How do you implement read receipts in a chat system?

## 📚 Related Topics
- [Message Queue Design](./04-design-message-queue.md)
- [System Design Principles](../12-performance-optimization/README.md)
- [WebSocket Programming](../13-framework-integration/README.md)

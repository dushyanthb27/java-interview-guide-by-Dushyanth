# System Design Interview Guide

## Master System Design Interviews

System design interviews assess your ability to design scalable, reliable systems. This guide provides a structured approach to tackle any system design question.

## The DDSCA Framework

A proven 5-step approach to system design:

### Step 1: DEFINE Requirements (5-10 minutes)

**Functional Requirements:**
- What should the system do?
- What are the main features?
- What are the user interactions?

**Non-Functional Requirements:**
- How many users?
- How much data?
- Latency requirements?
- Availability requirements?
- Consistency requirements?

**Example: Design a URL Shortening Service**

```
Functional Requirements:
- Generate short URL from long URL
- Redirect short URL to long URL
- Track click statistics
- Expire URLs after certain time

Non-Functional Requirements:
- 100 million URLs per month
- 1 billion redirects per month
- 100ms latency for redirect
- 99.9% availability
- Consistency: eventual consistency acceptable
```

**Questions to Ask:**
1. "How many users are we expecting?"
2. "What's the read-to-write ratio?"
3. "What are the latency requirements?"
4. "How much data will we store?"
5. "What's the availability requirement?"
6. "Do we need strong or eventual consistency?"

### Step 2: DESIGN High-Level Architecture (10-15 minutes)

**Components to Consider:**
1. **Client Layer**
   - Web browser
   - Mobile app
   - API clients

2. **API Layer**
   - REST endpoints
   - Request validation
   - Rate limiting

3. **Business Logic Layer**
   - Core algorithms
   - Business rules
   - Data processing

4. **Data Layer**
   - Primary database
   - Cache
   - Message queue

5. **External Services**
   - Analytics
   - Notifications
   - Payment processing

**Example Architecture Diagram:**
```
┌─────────────────────────────────────────────────┐
│                    Clients                       │
│         (Web, Mobile, API Clients)              │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│              Load Balancer                       │
│         (Distribute traffic)                    │
└────────────────────┬────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────┐
│           API Servers (Multiple)                │
│    - Handle requests                           │
│    - Validate input                            │
│    - Rate limiting                             │
└────────────────────┬────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
    ┌───▼──┐    ┌───▼──┐    ┌───▼──┐
    │Cache │    │  DB  │    │Queue │
    │Redis │    │MySQL │    │Kafka │
    └──────┘    └──────┘    └──────┘
```

**Key Design Decisions:**
- Monolithic vs. Microservices
- Synchronous vs. Asynchronous
- Relational vs. NoSQL database
- Cache strategy
- Message queue for async processing

### Step 3: SCALE the Design (10-15 minutes)

**Scaling Strategies:**

1. **Horizontal Scaling**
   - Add more servers
   - Load balancing
   - Database replication

2. **Vertical Scaling**
   - Increase server capacity
   - Larger database
   - More memory

3. **Caching**
   - Cache frequently accessed data
   - Reduce database load
   - Improve response time

4. **Database Optimization**
   - Indexing
   - Partitioning
   - Read replicas

5. **Asynchronous Processing**
   - Message queues
   - Background jobs
   - Event-driven architecture

**Example: Scaling URL Shortening Service**

```
Challenge: 1 billion redirects per month
Solution:
1. Use CDN for static content
2. Cache hot URLs in Redis
3. Read replicas for database
4. Async logging to message queue
5. Horizontal scaling of API servers

Expected Performance:
- 99.9% of requests < 100ms
- Handles 10,000 requests/second
- 99.9% availability
```

### Step 4: CHOOSE Technologies (5-10 minutes)

**Database Selection:**

| Type | Use Case | Examples |
|------|----------|----------|
| Relational | Structured data, ACID | MySQL, PostgreSQL |
| NoSQL (Document) | Flexible schema | MongoDB, CouchDB |
| NoSQL (Key-Value) | Fast access | Redis, DynamoDB |
| NoSQL (Graph) | Relationships | Neo4j, ArangoDB |
| Time-Series | Metrics, logs | InfluxDB, Prometheus |

**Cache Selection:**
- **Redis:** In-memory, fast, supports data structures
- **Memcached:** Simple key-value, distributed
- **Local Cache:** Single server, no network overhead

**Message Queue Selection:**
- **Kafka:** High throughput, persistent, distributed
- **RabbitMQ:** Reliable, flexible routing
- **AWS SQS:** Managed, scalable, simple

**Example Technology Stack:**
```
Frontend: React, Vue.js
Backend: Java Spring Boot, Node.js
Database: MySQL (primary), PostgreSQL (analytics)
Cache: Redis
Message Queue: Kafka
Search: Elasticsearch
Monitoring: Prometheus, Grafana
Logging: ELK Stack
```

### Step 5: ANALYZE Trade-offs (5-10 minutes)

**Common Trade-offs:**

1. **Consistency vs. Availability**
   - Strong consistency: slower, more reliable
   - Eventual consistency: faster, may be stale

2. **Latency vs. Throughput**
   - Low latency: fewer requests per second
   - High throughput: higher latency

3. **Cost vs. Performance**
   - Expensive: better performance
   - Cheap: slower, more latency

4. **Complexity vs. Simplicity**
   - Simple: easier to maintain
   - Complex: better performance

**Example Trade-off Analysis:**
```
URL Shortening Service:

Option 1: Strong Consistency
- Pros: Always correct data
- Cons: Slower, more complex
- Use when: Financial transactions

Option 2: Eventual Consistency
- Pros: Faster, simpler
- Cons: May see stale data briefly
- Use when: URL shortening (acceptable)

Decision: Eventual consistency
Reasoning: Users can tolerate brief delays
```

## Common System Design Problems

### 1. URL Shortening Service (tinyurl.com)

**Requirements:**
- Convert long URL to short URL
- Redirect short URL to long URL
- Track statistics
- Expire URLs

**Key Components:**
- Hash function for URL encoding
- Database for URL mapping
- Cache for hot URLs
- Analytics service

**Challenges:**
- Collision handling
- Scalability
- Analytics accuracy

### 2. Social Media Feed

**Requirements:**
- Display posts from followed users
- Real-time updates
- Pagination
- Filtering and sorting

**Key Components:**
- User graph database
- Feed cache
- Message queue for updates
- Search engine

**Challenges:**
- Real-time updates
- Scalability
- Personalization

### 3. Distributed Cache

**Requirements:**
- Fast data access
- Distributed across servers
- Consistency
- Eviction policy

**Key Components:**
- Hash ring for distribution
- Replication strategy
- Eviction policy
- Consistency protocol

**Challenges:**
- Cache invalidation
- Replication
- Consistency

### 4. Rate Limiter

**Requirements:**
- Limit requests per user
- Different limits for different users
- Distributed rate limiting
- Accurate counting

**Key Components:**
- Token bucket algorithm
- Distributed counter
- Redis for state
- API gateway

**Challenges:**
- Distributed counting
- Race conditions
- Accuracy

### 5. Payment System

**Requirements:**
- Process payments
- Handle failures
- Ensure consistency
- Prevent fraud

**Key Components:**
- Payment gateway
- Transaction database
- Message queue
- Fraud detection

**Challenges:**
- Consistency
- Failure handling
- Fraud prevention

## Design Patterns

### Pattern 1: Load Balancing
```
Multiple servers behind load balancer
Distributes traffic evenly
Strategies: Round-robin, Least connections, IP hash
```

### Pattern 2: Caching
```
Cache frequently accessed data
Reduces database load
Strategies: LRU, LFU, TTL
```

### Pattern 3: Database Replication
```
Master-slave replication
Improves read performance
Provides redundancy
```

### Pattern 4: Sharding
```
Partition data across servers
Improves scalability
Challenges: Hotspots, rebalancing
```

### Pattern 5: Message Queue
```
Asynchronous processing
Decouples components
Improves reliability
```

### Pattern 6: Circuit Breaker
```
Prevents cascading failures
Fails fast
Allows recovery
```

## Estimation Techniques

### Traffic Estimation

**Example: URL Shortening Service**
```
100 million URLs per month
= 100M / (30 * 24 * 3600) = 38 URLs/second

1 billion redirects per month
= 1B / (30 * 24 * 3600) = 385 redirects/second

Peak traffic (10x average)
= 3,850 redirects/second
```

### Storage Estimation

**Example: URL Shortening Service**
```
100 million URLs per month
= 100M * 12 months = 1.2 billion URLs per year

Per URL storage:
- Original URL: 2KB
- Short URL: 10 bytes
- Metadata: 100 bytes
Total: ~2KB per URL

1.2B URLs * 2KB = 2.4TB per year
5 years = 12TB
```

### Bandwidth Estimation

**Example: URL Shortening Service**
```
385 redirects/second (average)
Response size: 1KB
Bandwidth: 385 * 1KB = 385KB/second
= 30MB/minute
= 1.8GB/hour
```

## Communication During Design

### What to Say:

1. **Clarify Requirements:**
   - "Let me confirm the requirements..."
   - "So we need to support X users?"
   - "The latency requirement is Y milliseconds?"

2. **Propose Architecture:**
   - "I would propose this architecture..."
   - "The main components would be..."
   - "This would handle the traffic because..."

3. **Discuss Trade-offs:**
   - "There's a trade-off between consistency and latency"
   - "We could use X, but that would impact Y"
   - "I chose this approach because..."

4. **Handle Feedback:**
   - "That's a good point, let me reconsider"
   - "I hadn't thought about that, let me adjust"
   - "How would you approach this?"

### What NOT to Say:

❌ "I don't know how to design this"
❌ "This is too complicated"
❌ "I've never done this before"
❌ Long silences without explanation
❌ Defensive responses to feedback
❌ "Let me just guess"

## Interview Scenarios

### Scenario 1: You Don't Know the Answer

**Response:**
```
"I haven't designed a system like this before, 
but let me think through it systematically.

First, let me clarify the requirements...
Then, I would propose this architecture...
The main components would be..."
```

**Key Points:**
- Don't panic
- Use the framework
- Think systematically
- Ask for feedback

### Scenario 2: Interviewer Challenges Your Design

**Response:**
```
"That's a good point. Let me reconsider.
If we need to handle X, then we might need to...
Would this approach work better?"
```

**Key Points:**
- Accept feedback gracefully
- Adjust your design
- Show flexibility
- Ask for confirmation

### Scenario 3: You Run Out of Time

**Response:**
```
"I see we're running short on time. 
Let me summarize the key components:
1. API servers with load balancing
2. Database with replication
3. Cache layer
4. Message queue for async processing

For optimization, we could also add..."
```

**Key Points:**
- Prioritize key components
- Summarize clearly
- Discuss improvements verbally
- Show you understand the full design

## Evaluation Criteria (Interviewer Perspective)

### Requirements Analysis (20%)
- [ ] Asks clarifying questions
- [ ] Identifies functional requirements
- [ ] Identifies non-functional requirements
- [ ] Estimates scale correctly

### Architecture Design (30%)
- [ ] Proposes reasonable architecture
- [ ] Identifies key components
- [ ] Explains component interactions
- [ ] Considers scalability

### Technical Knowledge (25%)
- [ ] Knows relevant technologies
- [ ] Understands trade-offs
- [ ] Discusses optimization strategies
- [ ] Shows depth of knowledge

### Communication (15%)
- [ ] Explains clearly
- [ ] Uses diagrams effectively
- [ ] Listens to feedback
- [ ] Asks good questions

### Problem-Solving (10%)
- [ ] Handles challenges
- [ ] Adjusts design based on feedback
- [ ] Shows flexibility
- [ ] Thinks systematically

## Common Mistakes to Avoid

❌ **Don't:**
- Jump to implementation details
- Ignore non-functional requirements
- Propose overly complex design
- Claim to know everything
- Ignore interviewer feedback
- Make unrealistic assumptions
- Forget about failure scenarios
- Over-engineer the solution

✅ **Do:**
- Clarify requirements first
- Consider scalability early
- Propose simple solutions first
- Admit knowledge gaps
- Accept feedback gracefully
- Make realistic assumptions
- Plan for failures
- Keep it simple

## Quick Reference Checklist

### Before Designing:
- [ ] Clarify all requirements
- [ ] Estimate scale
- [ ] Identify constraints
- [ ] Get interviewer approval

### While Designing:
- [ ] Propose high-level architecture
- [ ] Identify key components
- [ ] Discuss trade-offs
- [ ] Explain decisions

### After Designing:
- [ ] Review for completeness
- [ ] Discuss scalability
- [ ] Discuss failure scenarios
- [ ] Ask for feedback

## Resources for Practice

### Practice Problems:
1. Design a URL shortening service
2. Design a social media feed
3. Design a distributed cache
4. Design a rate limiter
5. Design a payment system
6. Design a search engine
7. Design a video streaming platform
8. Design a chat application

### Study Materials:
- "Designing Data-Intensive Applications" by Martin Kleppmann
- System Design Interview by Alex Xu
- High Scalability blog
- Architecture patterns documentation

## Final Tips

1. **Practice with a partner** - Get feedback on your design
2. **Draw diagrams** - Visualize your architecture
3. **Discuss trade-offs** - Show you understand complexity
4. **Ask questions** - Clarify requirements
5. **Stay flexible** - Adjust based on feedback
6. **Think systematically** - Use the framework
7. **Estimate accurately** - Show you understand scale
8. **Communicate clearly** - Explain your thinking

---

**Remember: System design interviews are about your approach and reasoning, not the perfect design. Show your thinking, accept feedback, and demonstrate your ability to design scalable systems.**

Last Updated: 2024

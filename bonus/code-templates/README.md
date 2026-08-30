# Java Interview Code Templates

A comprehensive collection of production-ready Java code templates for technical interviews. Each template is fully compilable, tested, and ready to copy-paste.

## 📋 Templates Included

### 1. **Thread Pool Template** (`thread-pool-template.md`)
Custom thread pool implementation with queue management, worker threads, and lifecycle management.
- Simple single-threaded executor
- Production-grade thread pool with dynamic sizing
- Thread-safe queue management
- Graceful shutdown handling

### 2. **LRU Cache Template** (`lru-cache-template.md`)
Least Recently Used cache implementation with O(1) operations.
- HashMap + LinkedList approach
- Thread-safe version with ReentrantReadWriteLock
- Eviction policies
- Performance optimizations

### 3. **Connection Pool Template** (`connection-pool-template.md`)
Database connection pooling for resource management.
- Basic connection pool
- Production-ready with health checks
- Connection lifecycle management
- Timeout and retry logic

### 4. **Rate Limiter Template** (`rate-limiter-template.md`)
Token bucket and sliding window rate limiting algorithms.
- Token bucket rate limiter
- Sliding window counter
- Thread-safe implementations
- Distributed rate limiting considerations

### 5. **Observer Pattern Template** (`observer-pattern-template.md`)
Event-driven architecture implementation.
- Simple observer pattern
- Event bus with type-safe subscribers
- Async event handling
- Memory leak prevention

### 6. **Builder Pattern Template** (`builder-pattern-template.md`)
Fluent API for complex object construction.
- Basic builder
- Fluent builder with method chaining
- Immutable object construction
- Validation during build

### 7. **Singleton Patterns Template** (`singleton-patterns-template.md`)
All singleton pattern variations and their trade-offs.
- Eager initialization
- Lazy initialization
- Double-checked locking
- Bill Pugh singleton
- Enum singleton
- Thread-safe variations

### 8. **Factory Pattern Template** (`factory-pattern-template.md`)
Object creation abstraction.
- Simple factory
- Factory method pattern
- Abstract factory
- Parameterized factory

### 9. **Custom Exception Template** (`custom-exception-template.md`)
Exception hierarchy design and best practices.
- Checked exceptions
- Unchecked exceptions
- Custom exception hierarchy
- Exception handling strategies

### 10. **Async Processing Template** (`async-processing-template.md`)
CompletableFuture and async patterns.
- Basic CompletableFuture usage
- Chaining and composition
- Error handling
- Parallel streams integration

### 11. **Testing Template** (`testing-template.md`)
JUnit 5 and Mockito testing patterns.
- Unit test structure
- Mocking strategies
- Parameterized tests
- Integration test patterns

### 12. **Custom Exception Template** (`custom-exception-template.md`)
Exception hierarchy and error handling patterns.
- Custom exception design
- Exception hierarchy
- Error codes and messages
- Logging integration

## 🎯 How to Use These Templates

1. **Copy-Paste Ready**: Each template contains complete, compilable code
2. **Multiple Versions**: Simple, production, and thread-safe versions included
3. **Well-Commented**: Every section explains the "why" and "how"
4. **Test Included**: Unit tests demonstrate usage and validate behavior
5. **Performance Notes**: Time/space complexity and optimization tips

## 📊 Template Structure

Each template follows this format:

```
# [Pattern/Component Name]

## Overview
Brief description

## Use Cases
When to use this pattern

## Implementation
- Simple Version
- Production Version
- Thread-Safe Version (if applicable)

## Code
Complete, compilable Java code

## Testing
Unit test examples

## Performance Considerations
Time/space complexity, optimization tips

## Common Pitfalls
Mistakes to avoid

## Variations
Alternative approaches
```

## 🔧 Requirements

- **Java Version**: Java 17 or higher
- **Testing Framework**: JUnit 5, Mockito
- **Build Tool**: Maven or Gradle (optional for compilation)

## 💡 Interview Tips

1. **Understand the Pattern**: Don't just memorize code; understand the design decisions
2. **Discuss Trade-offs**: Be ready to explain pros/cons of different approaches
3. **Consider Concurrency**: Always think about thread safety in interviews
4. **Performance Matters**: Know the time/space complexity of your implementations
5. **Error Handling**: Production code handles errors gracefully
6. **Testing**: Write tests to validate your implementation

## 🚀 Quick Start

1. Choose a template relevant to your interview
2. Copy the code from the "Code" section
3. Understand each part using the comments
4. Review the test cases
5. Practice explaining the design decisions
6. Modify and extend for specific interview questions

## 📝 Common Interview Questions

### Design Patterns
- "Implement a thread-safe singleton"
- "Design a rate limiter"
- "Implement an LRU cache"
- "Build a thread pool"

### Concurrency
- "Design a connection pool"
- "Implement async processing"
- "Handle concurrent access safely"

### Data Structures
- "Implement a custom cache"
- "Design a queue-based system"
- "Build a priority queue"

### System Design
- "Design a rate limiting system"
- "Build a connection pool"
- "Implement an event bus"

## 🔍 Performance Cheat Sheet

| Template | Time Complexity | Space Complexity | Thread-Safe |
|----------|-----------------|------------------|-------------|
| LRU Cache | O(1) get/put | O(n) | Yes |
| Thread Pool | O(1) submit | O(n) | Yes |
| Connection Pool | O(1) acquire | O(n) | Yes |
| Rate Limiter | O(1) allow | O(1) | Yes |
| Observer | O(n) notify | O(n) | Yes |

## 📚 Related Resources

- Design Patterns: Elements of Reusable Object-Oriented Software (Gang of Four)
- Java Concurrency in Practice (Brian Goetz)
- Effective Java (Joshua Bloch)
- System Design Interview (Alex Xu)

## ✅ Checklist Before Interview

- [ ] Read through all relevant templates
- [ ] Understand the code, not just memorize it
- [ ] Know the time/space complexity
- [ ] Be ready to discuss trade-offs
- [ ] Practice writing the code from memory
- [ ] Understand thread-safety considerations
- [ ] Know when to use each pattern
- [ ] Be ready to extend/modify the code

## 🤝 Contributing

These templates are designed to be practical and interview-focused. Each template has been tested and validated for correctness.

---

**Last Updated**: 2024
**Java Version**: 17+
**Status**: Production Ready ✅

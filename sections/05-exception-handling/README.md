# Exception Handling

[Home](../../README.md) > Exception Handling

---

## Overview
Proper exception handling is critical for building robust Java applications. This section covers the exception hierarchy, best practices, and advanced topics like handling exceptions in streams.

## Prerequisites
- [Java Fundamentals](../01-java-fundamentals/README.md)
- [OOP Concepts](../02-oop-concepts/README.md)

## Question Index
| # | Question | Difficulty | File |
|---|----------|------------|------|
| 1 | Explain the Java exception hierarchy | 🟢 | [q01](./q01-exception-hierarchy.md) |
| 2 | What's the difference between checked and unchecked exceptions? | 🟢 | [q02](./q02-checked-vs-unchecked.md) |
| 3 | How does try-catch-finally work? | 🟢 | [q03](./q03-try-catch-finally.md) |
| 4 | What is try-with-resources? | 🟡 | [q04](./q04-try-with-resources.md) |
| 5 | How to create custom exceptions? | 🟡 | [q05](./q05-custom-exceptions.md) |
| 6 | What are exception handling best practices? | 🟠 | [q06](./q06-exception-best-practices.md) |
| 7 | Error vs Exception - what's the difference? | 🟡 | [q07](./q07-error-vs-exception.md) |
| 8 | What is exception chaining? | 🟡 | [q08](./q08-exception-chaining.md) |
| 9 | What are suppressed exceptions? | 🟠 | [q09](./q09-suppressed-exceptions.md) |
| 10 | How to handle exceptions in Streams? | 🟠 | [q10](./q10-exceptions-in-streams.md) |
| 11 | What is the performance impact of exceptions? | 🟠 | [q11](./q11-exception-performance.md) |
| 12 | Multi-catch and rethrowing exceptions | 🟡 | [q12](./q12-multi-catch-rethrowing.md) |

## Key Takeaways
- Use checked exceptions for recoverable conditions, unchecked for programming errors
- Always use try-with-resources for AutoCloseable resources
- Never catch Throwable or Error in application code
- Exception creation is expensive due to stack trace filling
- Wrap checked exceptions in unchecked wrappers for stream operations

## Related Sections
- [Java I/O & NIO](../08-java-io-nio/README.md)
- [Java 8+ Features](../06-java8-plus-features/README.md)
- [Performance Optimization](../12-performance-optimization/README.md)

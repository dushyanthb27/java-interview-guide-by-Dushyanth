# Java Core Interview Guide 2026

[![Java](https://img.shields.io/badge/Java-17%2B-orange?style=for-the-badge&logo=openjdk)](https://openjdk.org/)
[![Questions](https://img.shields.io/badge/Questions-215-blue?style=for-the-badge)](.)
[![Year](https://img.shields.io/badge/Year-2026-green?style=for-the-badge)](.)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)

> A comprehensive Java interview preparation guide covering **215 questions** from Junior to Staff/Principal level, with in-depth answers, code examples, performance analysis, and real-world scenarios.

---

## Quick Start

1. **Pick your level** - Use the difficulty legend below to find questions matching your experience
2. **Follow the sections** - Sections are ordered from foundational to advanced
3. **Practice with code** - All examples compile and run on Java 17+
4. **Use cross-references** - Related topics are linked for deeper exploration
5. **Review interviewer insights** - Understand what interviewers are really looking for

---

## Difficulty Legend

| Emoji | Level | Target Role | Answer Depth |
|-------|-------|-------------|--------------|
| 🟢 | Junior | 0-2 years | Foundational, 8-12 lines with basic examples |
| 🟡 | Mid-Level | 2-5 years | Nuanced, 12-18 lines with trade-off analysis |
| 🟠 | Senior | 5-8 years | Trade-offs, 18-25 lines with architectural considerations |
| 🔴 | Staff/Principal | 8+ years | Architectural, 25-35 lines with system design implications |

---

## Complete Table of Contents

| # | Section | Questions | Key Topics |
|---|---------|-----------|------------|
| 01 | [Java Fundamentals](./sections/01-java-fundamentals/README.md) | 15 | JVM/JRE/JDK, data types, strings, records, sealed classes |
| 02 | [Object-Oriented Programming](./sections/02-oop-concepts/README.md) | 18 | Inheritance, polymorphism, interfaces, design patterns |
| 03 | [Collections Framework](./sections/03-collections-framework/README.md) | 20 | List, Set, Map, Queue, Streams, thread-safe collections |
| 04 | [Multithreading & Concurrency](./sections/04-multithreading-concurrency/README.md) | 25 | Threads, executors, locks, CompletableFuture, virtual threads |
| 05 | [Exception Handling](./sections/05-exception-handling/README.md) | 12 | Checked/unchecked, try-with-resources, best practices |
| 06 | [Java 8+ Features](./sections/06-java8-plus-features/README.md) | 18 | Lambdas, streams, Optional, records, pattern matching |
| 07 | [JVM Internals](./sections/07-jvm-internals/README.md) | 15 | Class loading, GC algorithms, JIT, memory model |
| 08 | [Java I/O & NIO](./sections/08-java-io-nio/README.md) | 12 | Streams, NIO channels, serialization, async I/O |
| 09 | [Generics & Type System](./sections/09-generics-type-system/README.md) | 15 | Type erasure, wildcards, bounded types, variance |
| 10 | [Java Memory Management](./sections/10-memory-management/README.md) | 12 | Stack/heap, references, memory leaks, profiling |
| 11 | [Java Security](./sections/11-java-security/README.md) | 10 | Cryptography, SSL/TLS, JAAS, secure coding |
| 12 | [Performance Optimization](./sections/12-performance-optimization/README.md) | 15 | JVM tuning, GC optimization, profiling, benchmarking |
| 13 | [Framework Integration](./sections/13-framework-integration/README.md) | 8 | Spring, Hibernate, JUnit, Maven/Gradle |
| 14 | [System Design (Bonus)](./sections/14-system-design/README.md) | 10 | Thread pool, cache, connection pool, rate limiter |
| 15 | [Behavioral Scenarios (Bonus)](./sections/15-behavioral-scenarios/README.md) | 10 | Leadership, debugging, code review, incident response |
| | **Total** | **215** | |

---

## How to Use This Repo

### For Interview Preparation
- **Week 1-2**: Sections 1-3 (Fundamentals, OOP, Collections)
- **Week 3-4**: Sections 4-6 (Concurrency, Exceptions, Java 8+ Features)
- **Week 5-6**: Sections 7-10 (JVM, I/O, Generics, Memory)
- **Week 7-8**: Sections 11-15 (Security, Performance, Frameworks, System Design, Behavioral)

### For Quick Review
- Focus on 🟢 and 🟡 questions for rapid refresher
- Read "Interviewer Insight" callouts for key talking points
- Review "Common Mistake" callouts to avoid pitfalls

### For Deep Dive
- Work through 🟠 and 🔴 questions for senior/staff roles
- Follow "Related Topics" links for comprehensive understanding
- Implement code examples and experiment with variations

---

## Answer Format Legend

| Callout | Purpose |
|---------|---------|
| 🎯 **Interviewer Insight** | What the interviewer is really looking for |
| ⚡ **Follow-up** | Expected follow-up questions |
| 📚 **Related** | Links to related topics |
| ⚠️ **Common Mistake** | Typical pitfalls to avoid |
| 💡 **Production Consideration** | Real-world production advice |
| 🔧 **JVM Internals** | Under-the-hood explanation |
| ⚡ **Performance Tip** | Optimization advice |

---

## Java Version Coverage

This guide covers features from **Java 8 through Java 21+**, with emphasis on:
- **Java 8**: Lambdas, Streams, Optional, Date/Time API
- **Java 11**: `var`, HTTP Client, String methods
- **Java 14**: Records, Switch Expressions
- **Java 15**: Sealed Classes, Text Blocks
- **Java 16**: Pattern Matching for `instanceof`
- **Java 17**: Sealed Classes (final), Pattern Matching
- **Java 19**: Virtual Threads (Preview)
- **Java 21**: Virtual Threads (GA), Pattern Matching for Switch, Sequenced Collections

---

## Bonus Content

In addition to the 215 interview questions, this guide includes comprehensive bonus materials to support your preparation:

### Quick Reference Materials
- **[Cheat Sheets](./bonus/cheat-sheets/README.md)** - 10 quick reference guides (Collections, Concurrency, Streams, JVM Flags, Testing, Design Patterns)
- **[Glossary](./bonus/glossary/README.md)** - 600+ Java terminology definitions with examples
- **[Code Templates](./bonus/code-templates/README.md)** - 13 production-ready code templates (Thread Pool, LRU Cache, Rate Limiter, Design Patterns)

### Learning Resources
- **[Resources](./bonus/resources/README.md)** - 250+ curated resources (books, courses, tools, communities, blogs)
- **[Study Plans](./bonus/study-plans/README.md)** - 10 structured study plans (2-week to 12-week, career progression)

### Interview Preparation
- **[Interview Tips](./bonus/interview-tips/README.md)** - 11 interview guides (Technical, Coding, System Design, Behavioral, Time Management)
- **[Mock Interview Schedule](./bonus/study-plans/mock-interview-schedule.md)** - Structured mock interview practice

### Advanced Topics (Senior/Staff Level)
- **[Advanced Topics](./bonus/advanced-topics/README.md)** - 10 deep dives (Generics, Concurrency, JVM Tuning, Microservices, Security)
- **[Case Studies](./bonus/case-studies/README.md)** - 10 real-world production scenarios (Migration, Performance, Security, Scaling)
- **[Architecture Patterns](./bonus/architecture-patterns/README.md)** - 10 architecture patterns (DDD, CQRS, Event Sourcing, Clean Architecture)
- **[Database Patterns](./bonus/database-patterns/README.md)** - 10 database patterns (Connection Pooling, Transactions, Sharding)
- **[Testing Strategies](./bonus/testing-strategies/README.md)** - 10 testing strategies (TDD, Mutation Testing, Chaos Engineering, Load Testing)

---

## Contributing

Contributions are welcome! Please follow these guidelines:
1. Maintain the existing formatting and structure
2. Include code examples that compile on Java 17+
3. Add appropriate difficulty level badges
4. Include interviewer insights and follow-up questions
5. Cross-reference related topics

---

## Author

**Dushyanth BR** - Java Core Interview Guide 2026

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

# Java Security

[Home](../../README.md) > Java Security

---

## Overview
Security is a critical aspect of Java development. This section covers Java's security architecture, cryptography, secure coding practices, and common vulnerability prevention.

## Prerequisites
- [Java Fundamentals](../01-java-fundamentals/README.md)
- [Exception Handling](../05-exception-handling/README.md)
- [Java I/O & NIO](../08-java-io-nio/README.md)

## Question Index
| # | Question | Difficulty | File |
|---|----------|------------|------|
| 1 | What is the Security Manager (deprecated)? | 🟡 | [q01](./q01-security-manager.md) |
| 2 | How do permissions and policies work? | 🟡 | [q02](./q02-permissions-policies.md) |
| 3 | What is code signing in Java? | 🟡 | [q03](./q03-code-signing.md) |
| 4 | Explain JAAS (Java Authentication and Authorization) | 🟠 | [q04](./q04-jaas.md) |
| 5 | How does Java cryptography work? | 🟠 | [q05](./q05-cryptography-basics.md) |
| 6 | What is SecureRandom and why is it important? | 🟡 | [q06](./q06-secure-random.md) |
| 7 | How does SSL/TLS work in Java? | 🟠 | [q07](./q07-ssl-tls.md) |
| 8 | How to prevent injection attacks? | 🟠 | [q08](./q08-preventing-injection.md) |
| 9 | What are secure coding best practices? | 🟠 | [q09](./q09-secure-coding-practices.md) |
| 10 | Sandboxing and the module system | 🟡 | [q10](./q10-sandboxing-modules.md) |

## Key Takeaways
- SecurityManager is deprecated since Java 17; modules provide modern access control
- Always use SecureRandom for security-sensitive random number generation
- Use PreparedStatement to prevent SQL injection
- Java Cryptography Architecture (JCA) provides pluggable security providers
- Input validation and output encoding are fundamental security practices

## Related Sections
- [JVM Internals](../07-jvm-internals/README.md)
- [Java I/O & NIO](../08-java-io-nio/README.md)
- [Performance Optimization](../12-performance-optimization/README.md)

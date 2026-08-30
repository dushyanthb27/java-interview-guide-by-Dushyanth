# Java I/O & NIO

[Home](../../README.md) > Java I/O & NIO

---

## Overview
Input/Output operations are fundamental to any application. This section covers traditional I/O streams, modern NIO channels, and advanced topics like memory-mapped files and asynchronous I/O.

## Prerequisites
- [Java Fundamentals](../01-java-fundamentals/README.md)
- [Exception Handling](../05-exception-handling/README.md)

## Question Index
| # | Question | Difficulty | File |
|---|----------|------------|------|
| 1 | Byte streams vs character streams | 🟢 | [q01](./q01-byte-vs-character-streams.md) |
| 2 | How does File I/O work in Java? | 🟢 | [q02](./q02-file-io.md) |
| 3 | BufferedReader and BufferedWriter usage | 🟢 | [q03](./q03-buffered-reader-writer.md) |
| 4 | How does serialization work? | 🟡 | [q04](./q04-serialization.md) |
| 5 | What is Externalizable? | 🟡 | [q05](./q05-externalizable.md) |
| 6 | NIO Channels explained | 🟠 | [q06](./q06-nio-channels.md) |
| 7 | NIO Buffers in depth | 🟠 | [q07](./q07-nio-buffers.md) |
| 8 | NIO Selectors for multiplexed I/O | 🔴 | [q08](./q08-nio-selectors.md) |
| 9 | Memory-mapped files | 🟠 | [q09](./q09-memory-mapped-files.md) |
| 10 | Path API and Files utility | 🟡 | [q10](./q10-path-api.md) |
| 11 | File watching with WatchService | 🟡 | [q11](./q11-file-watching.md) |
| 12 | Asynchronous I/O (AIO) | 🟠 | [q12](./q12-async-io.md) |

## Key Takeaways
- NIO provides non-blocking, channel-based I/O for high-performance applications
- Always use try-with-resources for I/O operations
- Serialization has security concerns; consider alternatives (JSON, Protocol Buffers)
- Memory-mapped files offer excellent performance for large file processing
- Path API (java.nio.file) is preferred over legacy java.io.File

## Related Sections
- [Exception Handling](../05-exception-handling/README.md)
- [Performance Optimization](../12-performance-optimization/README.md)
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md)

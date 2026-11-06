# Q6: NIO Channels Explained

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟠 Advanced

---

## Question
What are NIO Channels? How do they differ from traditional `java.io` streams, and how do components like `FileChannel`, `SocketChannel`, and zero-copy transfers operate?

---

## Answer

**Channels** are the foundational high-performance abstraction introduced in Java NIO (`java.nio.channels`). Unlike traditional streams (which are unidirectional, stream-oriented, and strictly blocking), channels are **bidirectional**, operate directly with memory **Buffers**, and support both **non-blocking I/O** and **multiplexed selectors**.

---

## Streams vs Channels

| Feature | Streams (`java.io`) | Channels (`java.nio`) |
|---------|---------------------|-----------------------|
| Direction | Unidirectional (either InputStream OR OutputStream) | Bidirectional (read AND write on the same channel) |
| Blocking Model | Always blocking | Can be configured as blocking or non-blocking |
| Data Unit | Byte-at-a-time or char-at-a-time | Bulk transfer via `ByteBuffer` |
| Multiplexing | Not supported | Supported via `Selector` (Epoll/Kqueue underlying) |
| Scatter / Gather | Not supported | Supported natively |
| Zero-Copy OS Transfers | Not supported | Supported via `transferTo()` / `transferFrom()` |
| File Locking | Limited/Native file locks | Fine-grained file and region locking (`FileLock`) |

---

## Channel Hierarchy

```
Channel (interface)
├── ReadableByteChannel
├── WritableByteChannel
├── ByteChannel (read + write)
│   ├── FileChannel           — file I/O and memory mapping
│   ├── SocketChannel         — TCP client socket
│   ├── ServerSocketChannel   — TCP server socket
│   └── DatagramChannel       — UDP datagram socket
└── AsynchronousChannel
    ├── AsynchronousFileChannel
    └── AsynchronousSocketChannel
```

---

## FileChannel Operations

```java
// ✅ Read from a file using FileChannel and ByteBuffer
try (FileChannel channel = FileChannel.open(
        Path.of("data.txt"), StandardOpenOption.READ)) {

    ByteBuffer buffer = ByteBuffer.allocate(1024);
    int bytesRead;
    while ((bytesRead = channel.read(buffer)) != -1) {
        buffer.flip();  // switch buffer from write mode to read mode
        while (buffer.hasRemaining()) {
            System.out.print((char) buffer.get());
        }
        buffer.clear();  // reset buffer for next write cycle
    }
}

// ✅ Random Access: read/write at precise file offsets
try (FileChannel channel = FileChannel.open(
        Path.of("data.bin"), StandardOpenOption.READ, StandardOpenOption.WRITE)) {

    ByteBuffer buffer = ByteBuffer.allocate(4);

    // Read 4 bytes starting exactly at position 100
    channel.read(buffer, 100);
    buffer.flip();
    int value = buffer.getInt();

    // Write 4 bytes at position 200 without altering channel position pointer
    buffer.clear();
    buffer.putInt(42);
    buffer.flip();
    channel.write(buffer, 200);
}
```

---

## Efficient File Copy: transferTo / transferFrom

`FileChannel` leverages OS-level zero-copy mechanisms (such as `sendfile` on Linux/Unix) to transfer data directly between kernel space file descriptors without bouncing through user-space JVM memory buffers.

```java
// ✅ Zero-copy file transfer with loop guard for partial transfers
public static void copyFile(Path source, Path target) throws IOException {
    try (FileChannel srcChannel = FileChannel.open(source, StandardOpenOption.READ);
         FileChannel destChannel = FileChannel.open(target,
             StandardOpenOption.CREATE, StandardOpenOption.WRITE, StandardOpenOption.TRUNCATE_EXISTING)) {

        long transferred = 0;
        long size = srcChannel.size();
        
        // transferTo may not transfer all bytes in a single system call
        while (transferred < size) {
            long count = srcChannel.transferTo(transferred, size - transferred, destChannel);
            if (count == 0) break;
            transferred += count;
        }
    }
}
```

---

## SocketChannel (Non-Blocking TCP Client)

```java
public static void nonBlockingClient() throws IOException {
    try (SocketChannel channel = SocketChannel.open()) {
        channel.configureBlocking(false);  // enable non-blocking mode

        // Initiation returns immediately; connection proceeds asynchronously
        boolean connected = channel.connect(new InetSocketAddress("localhost", 8080));

        if (!connected) {
            while (!channel.finishConnect()) {
                // Perform other background tasks while TCP handshake completes
                Thread.yield();
            }
        }

        // Send HTTP request
        ByteBuffer writeBuffer = ByteBuffer.wrap(
            "GET / HTTP/1.1\r\nHost: localhost\r\n\r\n".getBytes(StandardCharsets.UTF_8));
        while (writeBuffer.hasRemaining()) {
            channel.write(writeBuffer);
        }

        // Read response non-blockingly
        ByteBuffer readBuffer = ByteBuffer.allocate(4096);
        int bytesRead = channel.read(readBuffer);
        if (bytesRead > 0) {
            readBuffer.flip();
            System.out.println(StandardCharsets.UTF_8.decode(readBuffer));
        }
    }
}
```

---

## Scatter/Gather I/O

- **Scatter Read**: Reads a single channel payload across multiple buffers in sequence (ideal for structured packets with fixed headers and variable bodies).
- **Gather Write**: Writes multiple buffers into a single channel write operation atomically.

```java
// ✅ Scatter Read: Header (128 bytes) + Body (4096 bytes)
try (FileChannel channel = FileChannel.open(Path.of("packet.bin"), StandardOpenOption.READ)) {
    ByteBuffer headerBuffer = ByteBuffer.allocate(128);
    ByteBuffer bodyBuffer = ByteBuffer.allocate(4096);
    
    ByteBuffer[] buffers = { headerBuffer, bodyBuffer };
    
    long totalBytesRead = channel.read(buffers);  // fills header fully before filling body
    
    headerBuffer.flip();
    bodyBuffer.flip();
}
```

---

## File Locking

```java
// ✅ Exclusive file region locking across OS processes
try (FileChannel channel = FileChannel.open(
        Path.of("app.lock"), StandardOpenOption.CREATE, StandardOpenOption.WRITE);
     FileLock lock = channel.lock(0, Long.MAX_VALUE, false)) { // exclusive lock

    // Critical section: safely update state file
    ByteBuffer buf = ByteBuffer.wrap("ACTIVE_LOCK".getBytes(StandardCharsets.UTF_8));
    channel.write(buf, 0);
    
} // FileLock is AutoCloseable; closing channel or lock releases it automatically
```

---

## ⚠️ Gotchas

1. **`FileChannel` Does Not Support Non-Blocking Mode**: Developers often assume all NIO channels are non-blocking. However, `FileChannel` is always tied to blocking file system operations. Attempting to call `fileChannel.configureBlocking(false)` throws an `IllegalBlockingModeException`.
2. **Partial Transfers in `transferTo()`**: `FileChannel.transferTo()` is not guaranteed to transfer all requested bytes in a single invocation due to kernel buffer limits or OS limits (e.g., 2GB chunk ceilings). Failing to wrap `transferTo` in a `while` loop tracking total bytes transferred results in truncated files.
3. **Busy-Wait CPU Spikes in Non-Blocking Loops**: When configuring `SocketChannel` or `ServerSocketChannel` to non-blocking mode without a `Selector`, checking `.accept()` or `.read()` in a tight `while (true)` loop causes 100% CPU core utilization. Always use a `Selector` or appropriate backoff/sleep timers.
4. **Direct Buffer Memory Leaks and GC Pressure**: Allocating off-heap memory via `ByteBuffer.allocateDirect()` bypasses the JVM garbage collector heap. If direct buffers are allocated rapidly in high-throughput loops without explicit cleanup or relying on phantom references, native memory can leak, triggering native `OutOfMemoryError`.
5. **Selector Epoll Bug (Busy Polling / 100% CPU)**: On older Linux kernels and JDK versions, Java's NIO `Selector` implementation can suffer from the epoll null-dereference / busy-poll bug where `selector.select()` returns immediately with zero selected keys even when no events occurred, spinning the CPU indefinitely. Explicitly setting `-Djava.nio.channels.spi.SelectorProvider=sun.nio.ch.PollSelectorProvider` acts as a fallback mitigation.

---

[← Previous: Q5 - Externalizable](./q05-externalizable.md) | [Next: Q7 - NIO Buffers →](./q07-nio-buffers.md)

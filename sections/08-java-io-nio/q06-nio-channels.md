# Q6: NIO Channels Explained

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟠 Advanced

---

## Question
What are NIO Channels? How do they differ from traditional I/O streams?

---

## Answer

**Channels** are the core abstraction of Java NIO (`java.nio.channels`). Unlike streams (which are unidirectional and blocking), channels are **bidirectional**, can be **non-blocking**, and work with **Buffers** for data transfer.

---

## Streams vs Channels

| Feature | Streams (java.io) | Channels (java.nio) |
|---------|-------------------|---------------------|
| Direction | Unidirectional (read OR write) | Bidirectional (read AND write) |
| Blocking | Always blocking | Blocking or non-blocking |
| Data transfer | Byte/char at a time | Buffer-based (bulk transfer) |
| Scatter/Gather | Not supported | Supported |
| Memory-mapped I/O | Not supported | Supported |
| Selectors | Not supported | Supported (multiplexed I/O) |

---

## Channel Hierarchy

```
Channel (interface)
├── ReadableByteChannel
├── WritableByteChannel
├── ByteChannel (read + write)
│   ├── FileChannel           — file I/O
│   ├── SocketChannel         — TCP client
│   ├── ServerSocketChannel   — TCP server
│   └── DatagramChannel       — UDP
└── AsynchronousChannel
    ├── AsynchronousFileChannel
    └── AsynchronousSocketChannel
```

---

## FileChannel

```java
// ✅ Read from a file using FileChannel
try (FileChannel channel = FileChannel.open(
        Path.of("data.txt"), StandardOpenOption.READ)) {

    ByteBuffer buffer = ByteBuffer.allocate(1024);
    int bytesRead;
    while ((bytesRead = channel.read(buffer)) != -1) {
        buffer.flip();  // switch to read mode
        while (buffer.hasRemaining()) {
            System.out.print((char) buffer.get());
        }
        buffer.clear();  // switch back to write mode
    }
}

// ✅ Write to a file using FileChannel
try (FileChannel channel = FileChannel.open(
        Path.of("output.txt"),
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE,
        StandardOpenOption.TRUNCATE_EXISTING)) {

    String data = "Hello, NIO Channels!";
    ByteBuffer buffer = ByteBuffer.wrap(data.getBytes(StandardCharsets.UTF_8));
    channel.write(buffer);
}

// ✅ Random access: read/write at specific positions
try (FileChannel channel = FileChannel.open(
        Path.of("data.bin"), StandardOpenOption.READ, StandardOpenOption.WRITE)) {

    ByteBuffer buffer = ByteBuffer.allocate(4);

    // Read 4 bytes at position 100
    channel.read(buffer, 100);
    buffer.flip();
    int value = buffer.getInt();

    // Write 4 bytes at position 200
    buffer.clear();
    buffer.putInt(42);
    buffer.flip();
    channel.write(buffer, 200);
}
```

---

## Efficient File Copy: transferTo / transferFrom

```java
// ✅ Zero-copy file transfer (OS-level optimization)
public static void copyFile(Path source, Path target) throws IOException {
    try (FileChannel srcChannel = FileChannel.open(source, StandardOpenOption.READ);
         FileChannel destChannel = FileChannel.open(target,
             StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {

        long transferred = 0;
        long size = srcChannel.size();
        // transferTo may not transfer all bytes in one call
        while (transferred < size) {
            transferred += srcChannel.transferTo(transferred, size - transferred, destChannel);
        }
    }
}

// ✅ Alternative: transferFrom
public static void copyFileFrom(Path source, Path target) throws IOException {
    try (FileChannel srcChannel = FileChannel.open(source, StandardOpenOption.READ);
         FileChannel destChannel = FileChannel.open(target,
             StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {

        destChannel.transferFrom(srcChannel, 0, srcChannel.size());
    }
}
```

---

## SocketChannel (TCP Client)

```java
// ✅ Non-blocking TCP client
public static void tcpClient() throws IOException {
    try (SocketChannel channel = SocketChannel.open()) {
        channel.configureBlocking(false);  // non-blocking mode

        channel.connect(new InetSocketAddress("localhost", 8080));

        // Wait for connection to complete (non-blocking)
        while (!channel.finishConnect()) {
            // do other work while connecting...
        }

        // Send data
        ByteBuffer writeBuffer = ByteBuffer.wrap(
            "GET / HTTP/1.1\r\nHost: localhost\r\n\r\n".getBytes());
        while (writeBuffer.hasRemaining()) {
            channel.write(writeBuffer);
        }

        // Receive data
        ByteBuffer readBuffer = ByteBuffer.allocate(4096);
        int bytesRead = channel.read(readBuffer);
        if (bytesRead > 0) {
            readBuffer.flip();
            String response = StandardCharsets.UTF_8.decode(readBuffer).toString();
            System.out.println(response);
        }
    }
}
```

---

## ServerSocketChannel (TCP Server)

```java
// ✅ Simple non-blocking TCP server
public static void tcpServer() throws IOException {
    try (ServerSocketChannel serverChannel = ServerSocketChannel.open()) {
        serverChannel.bind(new InetSocketAddress(8080));
        serverChannel.configureBlocking(false);

        System.out.println("Server listening on port 8080");

        while (true) {
            SocketChannel clientChannel = serverChannel.accept();
            if (clientChannel != null) {
                // Handle client connection
                clientChannel.configureBlocking(false);
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                int read = clientChannel.read(buffer);
                if (read > 0) {
                    buffer.flip();
                    String request = StandardCharsets.UTF_8.decode(buffer).toString();
                    System.out.println("Received: " + request);

                    // Send response
                    ByteBuffer response = ByteBuffer.wrap(
                        "HTTP/1.1 200 OK\r\n\r\nHello!".getBytes());
                    clientChannel.write(response);
                }
                clientChannel.close();
            }
            Thread.sleep(100); // avoid busy-wait (use Selector instead)
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}
```

---

## DatagramChannel (UDP)

```java
// ✅ UDP sender
public static void udpSend() throws IOException {
    try (DatagramChannel channel = DatagramChannel.open()) {
        String message = "Hello, UDP!";
        ByteBuffer buffer = ByteBuffer.wrap(message.getBytes(StandardCharsets.UTF_8));
        channel.send(buffer, new InetSocketAddress("localhost", 9999));
    }
}

// ✅ UDP receiver
public static void udpReceive() throws IOException {
    try (DatagramChannel channel = DatagramChannel.open()) {
        channel.bind(new InetSocketAddress(9999));
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        SocketAddress sender = channel.receive(buffer);
        buffer.flip();
        String message = StandardCharsets.UTF_8.decode(buffer).toString();
        System.out.println("From " + sender + ": " + message);
    }
}
```

---

## Scatter/Gather I/O

**Scatter** reads from one channel into multiple buffers. **Gather** writes from multiple buffers into one channel.

```java
// ✅ Scatter read: read a message with fixed-length header + variable body
try (FileChannel channel = FileChannel.open(Path.of("message.bin"), StandardOpenOption.READ)) {

    ByteBuffer header = ByteBuffer.allocate(128);   // first 128 bytes → header
    ByteBuffer body = ByteBuffer.allocate(4096);     // rest → body

    ByteBuffer[] buffers = {header, body};
    long bytesRead = channel.read(buffers);  // scatter into multiple buffers

    header.flip();
    body.flip();
    // header contains first 128 bytes, body contains the rest
}

// ✅ Gather write: write header + body from separate buffers
try (FileChannel channel = FileChannel.open(Path.of("output.bin"),
        StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {

    ByteBuffer header = ByteBuffer.wrap(new byte[128]);
    ByteBuffer body = ByteBuffer.wrap("Body content".getBytes(StandardCharsets.UTF_8));

    ByteBuffer[] buffers = {header, body};
    long bytesWritten = channel.write(buffers);  // gather from multiple buffers
}
```

---

## Blocking vs Non-Blocking Mode

```java
// ✅ Blocking mode (default) — thread blocks until I/O completes
SocketChannel blocking = SocketChannel.open();
blocking.configureBlocking(true);  // default
blocking.read(buffer);  // blocks until data is available

// ✅ Non-blocking mode — returns immediately
SocketChannel nonBlocking = SocketChannel.open();
nonBlocking.configureBlocking(false);
int bytesRead = nonBlocking.read(buffer);
// bytesRead == 0 if no data available (does NOT block)
// bytesRead == -1 if connection closed

// Note: FileChannel does NOT support non-blocking mode
// FileChannel is always blocking
```

---

## File Locking

```java
// ✅ Exclusive file lock (prevents other processes from accessing)
try (FileChannel channel = FileChannel.open(
        Path.of("shared.dat"), StandardOpenOption.WRITE);
     FileLock lock = channel.lock()) {  // blocks until lock acquired

    ByteBuffer buffer = ByteBuffer.wrap("Exclusive data".getBytes());
    channel.write(buffer);
}  // lock released on close

// ✅ Try-lock (non-blocking)
try (FileChannel channel = FileChannel.open(
        Path.of("shared.dat"), StandardOpenOption.WRITE)) {

    FileLock lock = channel.tryLock();  // returns null if can't acquire
    if (lock != null) {
        try {
            // write safely
        } finally {
            lock.release();
        }
    } else {
        System.out.println("File is locked by another process");
    }
}
```

---

## Key Takeaways

1. **Channels are bidirectional** — a single `FileChannel` can read and write
2. **`transferTo`/`transferFrom`** enable zero-copy file transfers at the OS level
3. **Non-blocking mode** allows channels to return immediately when no data is available
4. **Scatter/Gather** enables reading/writing structured data with multiple buffers
5. **`FileChannel`** is always blocking; use `SocketChannel` for non-blocking network I/O
6. **File locking** prevents concurrent access from multiple processes

---

[← Previous: Q5 - Externalizable](./q05-externalizable.md) | [Next: Q7 - NIO Buffers →](./q07-nio-buffers.md)

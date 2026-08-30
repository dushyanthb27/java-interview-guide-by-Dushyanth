# Q12: Asynchronous I/O (AIO)

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟠 Advanced

---

## Question
How does asynchronous I/O work in Java? Compare Future-based and callback-based approaches, and explain when to use AIO vs NIO selectors.

---

## Answer

**Asynchronous I/O** (AIO, also called NIO.2 async channels, Java 7+) allows I/O operations to run in the background without blocking the calling thread. The OS performs the I/O and notifies the application when it's complete. Java provides two notification mechanisms: **Future-based** and **callback-based** (`CompletionHandler`).

---

## AIO vs NIO vs Traditional I/O

| Feature | Traditional I/O | NIO (Selectors) | AIO (Async) |
|---------|----------------|------------------|-------------|
| Blocking | Yes | No (non-blocking) | No (async) |
| Thread model | Thread per connection | Single thread + Selector | OS-managed threads |
| Pattern | Blocking | Reactor (event-driven) | Proactor (completion-driven) |
| Complexity | Low | High | Medium |
| Best for | Simple apps | Many connections, low throughput each | File I/O, moderate connections |

---

## Asynchronous Channel Types

```
AsynchronousChannel
├── AsynchronousFileChannel      — async file read/write
├── AsynchronousSocketChannel    — async TCP client
├── AsynchronousServerSocketChannel  — async TCP server
└── AsynchronousDatagramChannel  — async UDP (limited support)
```

---

## AsynchronousFileChannel

### Future-Based Approach

```java
// ✅ Async file read using Future
public static void asyncReadWithFuture() throws Exception {
    try (AsynchronousFileChannel channel = AsynchronousFileChannel.open(
            Path.of("data.txt"), StandardOpenOption.READ)) {

        ByteBuffer buffer = ByteBuffer.allocate(1024);

        // read() returns immediately with a Future
        Future<Integer> future = channel.read(buffer, 0);

        // Do other work while I/O is in progress...
        System.out.println("Reading in background...");

        // Block until I/O completes
        int bytesRead = future.get();  // or future.get(5, TimeUnit.SECONDS)
        System.out.println("Bytes read: " + bytesRead);

        buffer.flip();
        String content = StandardCharsets.UTF_8.decode(buffer).toString();
        System.out.println(content);
    }
}

// ✅ Async file write using Future
public static void asyncWriteWithFuture() throws Exception {
    try (AsynchronousFileChannel channel = AsynchronousFileChannel.open(
            Path.of("output.txt"),
            StandardOpenOption.CREATE,
            StandardOpenOption.WRITE,
            StandardOpenOption.TRUNCATE_EXISTING)) {

        ByteBuffer buffer = ByteBuffer.wrap(
            "Hello, Async I/O!".getBytes(StandardCharsets.UTF_8));

        Future<Integer> future = channel.write(buffer, 0);

        // Do other work...
        int bytesWritten = future.get();
        System.out.println("Bytes written: " + bytesWritten);
    }
}
```

### Callback-Based Approach (CompletionHandler)

```java
// ✅ Async file read using CompletionHandler
public static void asyncReadWithCallback() throws Exception {
    AsynchronousFileChannel channel = AsynchronousFileChannel.open(
        Path.of("data.txt"), StandardOpenOption.READ);

    ByteBuffer buffer = ByteBuffer.allocate(1024);

    // read() with CompletionHandler — no blocking at all
    channel.read(buffer, 0, buffer, new CompletionHandler<Integer, ByteBuffer>() {

        @Override
        public void completed(Integer bytesRead, ByteBuffer attachment) {
            System.out.println("Read completed: " + bytesRead + " bytes");
            attachment.flip();
            String content = StandardCharsets.UTF_8.decode(attachment).toString();
            System.out.println(content);
            try { channel.close(); } catch (IOException e) { /* handle */ }
        }

        @Override
        public void failed(Throwable exc, ByteBuffer attachment) {
            System.err.println("Read failed: " + exc.getMessage());
            try { channel.close(); } catch (IOException e) { /* handle */ }
        }
    });

    // Thread continues immediately — callback fires on I/O thread pool
    System.out.println("Read initiated, doing other work...");
    Thread.sleep(2000);  // keep main thread alive for callback
}

// ✅ Async file write using CompletionHandler
public static void asyncWriteWithCallback() throws Exception {
    AsynchronousFileChannel channel = AsynchronousFileChannel.open(
        Path.of("output.txt"),
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE);

    ByteBuffer buffer = ByteBuffer.wrap(
        "Async write content".getBytes(StandardCharsets.UTF_8));

    channel.write(buffer, 0, "write-op", new CompletionHandler<Integer, String>() {

        @Override
        public void completed(Integer bytesWritten, String attachment) {
            System.out.printf("'%s' completed: %d bytes written%n",
                attachment, bytesWritten);
            try { channel.close(); } catch (IOException e) { /* handle */ }
        }

        @Override
        public void failed(Throwable exc, String attachment) {
            System.err.printf("'%s' failed: %s%n", attachment, exc.getMessage());
            try { channel.close(); } catch (IOException e) { /* handle */ }
        }
    });
}
```

---

## AsynchronousSocketChannel (TCP Client)

```java
// ✅ Async TCP client
public class AsyncTcpClient {

    public static void main(String[] args) throws Exception {
        AsynchronousSocketChannel channel = AsynchronousSocketChannel.open();

        // ✅ Async connect
        Future<Void> connectFuture = channel.connect(
            new InetSocketAddress("localhost", 8080));
        connectFuture.get();  // wait for connection
        System.out.println("Connected!");

        // ✅ Async write
        ByteBuffer writeBuffer = ByteBuffer.wrap(
            "Hello, Server!".getBytes(StandardCharsets.UTF_8));
        Future<Integer> writeFuture = channel.write(writeBuffer);
        writeFuture.get();

        // ✅ Async read
        ByteBuffer readBuffer = ByteBuffer.allocate(4096);
        Future<Integer> readFuture = channel.read(readBuffer);
        int bytesRead = readFuture.get();

        readBuffer.flip();
        String response = StandardCharsets.UTF_8.decode(readBuffer).toString();
        System.out.println("Response: " + response);

        channel.close();
    }
}
```

---

## AsynchronousServerSocketChannel (TCP Server)

```java
// ✅ Async TCP server using CompletionHandler
public class AsyncTcpServer {

    public static void main(String[] args) throws Exception {
        AsynchronousServerSocketChannel server =
            AsynchronousServerSocketChannel.open()
                .bind(new InetSocketAddress(8080));

        System.out.println("Async server started on port 8080");

        // ✅ Accept connections asynchronously
        server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {

            @Override
            public void completed(AsynchronousSocketChannel client, Void attachment) {
                // Accept next connection immediately
                server.accept(null, this);

                // Handle current client
                handleClient(client);
            }

            @Override
            public void failed(Throwable exc, Void attachment) {
                System.err.println("Accept failed: " + exc.getMessage());
            }
        });

        // Keep server running
        Thread.currentThread().join();
    }

    private static void handleClient(AsynchronousSocketChannel client) {
        ByteBuffer buffer = ByteBuffer.allocate(4096);

        client.read(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>() {

            @Override
            public void completed(Integer bytesRead, ByteBuffer buf) {
                if (bytesRead == -1) {
                    try { client.close(); } catch (IOException e) { /* handle */ }
                    return;
                }

                buf.flip();
                String request = StandardCharsets.UTF_8.decode(buf).toString();
                System.out.println("Received: " + request);

                // Echo response
                ByteBuffer response = ByteBuffer.wrap(
                    ("Echo: " + request).getBytes(StandardCharsets.UTF_8));
                client.write(response, response,
                    new CompletionHandler<Integer, ByteBuffer>() {
                        @Override
                        public void completed(Integer result, ByteBuffer b) {
                            try { client.close(); } catch (IOException e) { /* handle */ }
                        }

                        @Override
                        public void failed(Throwable exc, ByteBuffer b) {
                            try { client.close(); } catch (IOException e) { /* handle */ }
                        }
                    });
            }

            @Override
            public void failed(Throwable exc, ByteBuffer buf) {
                try { client.close(); } catch (IOException e) { /* handle */ }
            }
        });
    }
}
```

---

## Proactor Pattern vs Reactor Pattern

```
Reactor Pattern (NIO Selectors):
  1. Selector detects "channel is READY for I/O"
  2. Application performs the I/O operation
  3. Application processes the data

  Thread: [wait for readiness] → [do I/O] → [process] → [wait...]

Proactor Pattern (AIO):
  1. Application initiates async I/O operation
  2. OS performs the I/O in the background
  3. OS notifies application when I/O is COMPLETE

  Thread: [initiate I/O] → [do other work] → [callback: process result]
```

---

## AsynchronousChannelGroup (Thread Pool)

```java
// ✅ Custom thread pool for async I/O
AsynchronousChannelGroup group = AsynchronousChannelGroup.withThreadPool(
    Executors.newFixedThreadPool(4));

// Use the group when opening channels
AsynchronousServerSocketChannel server =
    AsynchronousServerSocketChannel.open(group)
        .bind(new InetSocketAddress(8080));

AsynchronousSocketChannel client =
    AsynchronousSocketChannel.open(group);

// ✅ Shutdown the group when done
group.shutdown();                    // no new channels, existing ones finish
group.awaitTermination(10, TimeUnit.SECONDS);
group.shutdownNow();                 // force close all channels
```

---

## AIO vs NIO Selectors: When to Use Each

| Criteria | NIO Selectors | AIO (Async) |
|----------|--------------|-------------|
| Connection count | Very high (10K+) | Moderate (hundreds to thousands) |
| OS support | Excellent everywhere | Best on Windows (IOCP); Linux uses thread pool |
| File I/O | Not applicable | ✅ `AsynchronousFileChannel` |
| Control | Fine-grained (event loop) | Coarser (OS-managed) |
| Complexity | Higher (manual event loop) | Lower (Future/callback) |
| Frameworks | Netty, Vert.x, Java NIO | Built-in Java AIO |
| Performance (Linux) | Better (epoll) | Good (but uses thread pool, not true AIO) |
| Performance (Windows) | Good | Better (uses IOCP — true kernel AIO) |

---

## Common Anti-Patterns

```java
// ❌ Blocking immediately after async call (defeats the purpose)
Future<Integer> future = channel.read(buffer, 0);
int result = future.get();  // blocks — might as well use synchronous I/O

// ✅ Do meaningful work between initiation and get()
Future<Integer> future = channel.read(buffer, 0);
processOtherData();          // useful work
prepareNextRequest();        // useful work
int result = future.get();   // now block

// ❌ Callback hell — deeply nested CompletionHandlers
channel.read(buf, null, new CompletionHandler<>() {
    public void completed(Integer r, Void a) {
        channel.write(response, null, new CompletionHandler<>() {
            public void completed(Integer r2, Void a2) {
                channel.read(nextBuf, null, new CompletionHandler<>() {
                    // deeply nested — hard to read and maintain
                });
            }
            public void failed(Throwable e, Void a2) { }
        });
    }
    public void failed(Throwable e, Void a) { }
});

// ✅ Extract handlers into named classes or methods
channel.read(buf, null, new ReadHandler(channel));

// ❌ Not closing channels in failed() callback
public void failed(Throwable exc, ByteBuffer buf) {
    System.err.println("Error: " + exc);
    // channel not closed — resource leak!
}

// ✅ Always close channels on failure
public void failed(Throwable exc, ByteBuffer buf) {
    System.err.println("Error: " + exc);
    try { channel.close(); } catch (IOException e) { /* log */ }
}
```

---

## Key Takeaways

1. **AIO uses the Proactor pattern** — the OS performs I/O and notifies on completion
2. **Two notification styles**: `Future` (poll/block for result) and `CompletionHandler` (callback)
3. **`CompletionHandler`** is truly non-blocking — no thread is blocked waiting for I/O
4. **`AsynchronousFileChannel`** is the only way to do async file I/O in Java
5. **On Linux**, AIO uses a thread pool internally (not true kernel AIO); NIO selectors (epoll) are often better
6. **On Windows**, AIO uses IOCP (true kernel async I/O) — often outperforms selectors
7. **Avoid callback hell** — extract handlers into named classes or use `Future` with `CompletableFuture`
8. **Always close channels** in both `completed()` and `failed()` callbacks

---

[← Previous: Q11 - File Watching](./q11-file-watching.md) | [Back to Java I/O & NIO →](./README.md)

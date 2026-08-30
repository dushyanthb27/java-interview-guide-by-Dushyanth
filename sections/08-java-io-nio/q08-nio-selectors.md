# Q8: NIO Selectors for Multiplexed I/O

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🔴 Expert

---

## Question
How do NIO Selectors work? Explain the Reactor pattern and how to build a non-blocking server.

---

## Answer

A **Selector** allows a single thread to monitor multiple channels for I/O readiness events (readable, writable, connectable, acceptable). This enables building high-performance servers that handle thousands of connections without a thread per connection.

---

## Thread-Per-Connection vs Selector

```
Thread-Per-Connection (Traditional):
┌─────────┐    ┌──────────┐
│ Client 1 │───│ Thread 1  │   ← 10,000 clients = 10,000 threads
│ Client 2 │───│ Thread 2  │   ← Memory: ~1MB stack per thread
│ Client 3 │───│ Thread 3  │   ← Context switching overhead
│   ...     │   │   ...     │
└─────────┘    └──────────┘

Selector (NIO):
┌─────────┐
│ Client 1 │──┐
│ Client 2 │──┼── Selector ── Single Thread
│ Client 3 │──┘   (multiplexes all channels)
│   ...     │
└─────────┘        ← 10,000 clients = 1 thread (or a few)
```

---

## SelectionKey Operations

| Operation | Constant | Channel Type | Meaning |
|-----------|----------|-------------|---------|
| Accept | `SelectionKey.OP_ACCEPT` | `ServerSocketChannel` | New connection ready |
| Connect | `SelectionKey.OP_CONNECT` | `SocketChannel` | Connection established |
| Read | `SelectionKey.OP_READ` | `SocketChannel` | Data available to read |
| Write | `SelectionKey.OP_WRITE` | `SocketChannel` | Channel ready for writing |

---

## Core Selector API

```java
// ✅ Create a selector
Selector selector = Selector.open();

// ✅ Register a channel with the selector
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);  // MUST be non-blocking
serverChannel.bind(new InetSocketAddress(8080));
SelectionKey key = serverChannel.register(selector, SelectionKey.OP_ACCEPT);

// ✅ Register for multiple operations
channel.register(selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);

// ✅ Attach data to a key
channel.register(selector, SelectionKey.OP_READ, myAttachment);
// or
key.attach(myAttachment);
Object attachment = key.attachment();

// ✅ Select ready channels
int readyCount = selector.select();       // blocks until at least 1 ready
int readyCount = selector.select(1000);   // blocks up to 1000ms
int readyCount = selector.selectNow();    // non-blocking, returns immediately

// ✅ Get ready keys
Set<SelectionKey> selectedKeys = selector.selectedKeys();
```

---

## Complete Non-Blocking Echo Server

```java
public class NioEchoServer {

    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();

        // ✅ Open and configure server channel
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(8080));
        serverChannel.configureBlocking(false);
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("Echo server started on port 8080");
        ByteBuffer buffer = ByteBuffer.allocate(4096);

        // ✅ Event loop
        while (true) {
            selector.select();  // block until events arrive

            Iterator<SelectionKey> keyIterator = selector.selectedKeys().iterator();
            while (keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();
                keyIterator.remove();  // MUST remove — selector doesn't auto-remove

                try {
                    if (key.isAcceptable()) {
                        handleAccept(key, selector);
                    } else if (key.isReadable()) {
                        handleRead(key, buffer);
                    }
                } catch (IOException e) {
                    System.err.println("Error handling key: " + e.getMessage());
                    key.cancel();
                    key.channel().close();
                }
            }
        }
    }

    private static void handleAccept(SelectionKey key, Selector selector)
            throws IOException {
        ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
        SocketChannel clientChannel = serverChannel.accept();
        clientChannel.configureBlocking(false);
        clientChannel.register(selector, SelectionKey.OP_READ);
        System.out.println("Accepted: " + clientChannel.getRemoteAddress());
    }

    private static void handleRead(SelectionKey key, ByteBuffer buffer)
            throws IOException {
        SocketChannel clientChannel = (SocketChannel) key.channel();
        buffer.clear();
        int bytesRead = clientChannel.read(buffer);

        if (bytesRead == -1) {
            // Client closed connection
            System.out.println("Closed: " + clientChannel.getRemoteAddress());
            key.cancel();
            clientChannel.close();
            return;
        }

        // Echo back
        buffer.flip();
        while (buffer.hasRemaining()) {
            clientChannel.write(buffer);
        }
    }
}
```

---

## Reactor Pattern Implementation

The **Reactor Pattern** is a design pattern for handling service requests delivered concurrently to an application.

```java
public class Reactor implements Runnable {

    private final Selector selector;
    private final ServerSocketChannel serverChannel;

    public Reactor(int port) throws IOException {
        selector = Selector.open();
        serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(port));
        serverChannel.configureBlocking(false);

        // Register for ACCEPT events with an Acceptor handler
        SelectionKey key = serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        key.attach(new Acceptor());
    }

    @Override
    public void run() {
        try {
            while (!Thread.interrupted()) {
                selector.select();
                Set<SelectionKey> selectedKeys = selector.selectedKeys();
                for (SelectionKey key : selectedKeys) {
                    dispatch(key);
                }
                selectedKeys.clear();
            }
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    // ✅ Dispatch to the handler attached to the key
    private void dispatch(SelectionKey key) {
        Runnable handler = (Runnable) key.attachment();
        if (handler != null) {
            handler.run();
        }
    }

    // ✅ Acceptor handles new connections
    private class Acceptor implements Runnable {
        @Override
        public void run() {
            try {
                SocketChannel clientChannel = serverChannel.accept();
                if (clientChannel != null) {
                    new Handler(selector, clientChannel);
                }
            } catch (IOException e) {
                throw new UncheckedIOException(e);
            }
        }
    }

    // ✅ Handler processes I/O for a single connection
    private static class Handler implements Runnable {
        private final SocketChannel channel;
        private final SelectionKey key;
        private final ByteBuffer buffer = ByteBuffer.allocate(4096);
        private static final int READING = 0, WRITING = 1;
        private int state = READING;

        Handler(Selector selector, SocketChannel channel) throws IOException {
            this.channel = channel;
            channel.configureBlocking(false);
            key = channel.register(selector, SelectionKey.OP_READ);
            key.attach(this);
            selector.wakeup();  // wake selector for new registration
        }

        @Override
        public void run() {
            try {
                if (state == READING) read();
                else if (state == WRITING) write();
            } catch (IOException e) {
                closeChannel();
            }
        }

        private void read() throws IOException {
            buffer.clear();
            int bytesRead = channel.read(buffer);
            if (bytesRead == -1) {
                closeChannel();
                return;
            }
            buffer.flip();
            // Process data...
            state = WRITING;
            key.interestOps(SelectionKey.OP_WRITE);
        }

        private void write() throws IOException {
            channel.write(buffer);
            if (!buffer.hasRemaining()) {
                state = READING;
                key.interestOps(SelectionKey.OP_READ);
            }
        }

        private void closeChannel() {
            try {
                key.cancel();
                channel.close();
            } catch (IOException ignored) {}
        }
    }

    public static void main(String[] args) throws IOException {
        new Thread(new Reactor(8080)).start();
        System.out.println("Reactor server started on port 8080");
    }
}
```

---

## Multi-Threaded Reactor (Worker Pool)

```java
public class MultiThreadedReactor {

    private final Selector selector;
    private final ExecutorService workerPool;

    public MultiThreadedReactor(int port, int poolSize) throws IOException {
        selector = Selector.open();
        workerPool = Executors.newFixedThreadPool(poolSize);

        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(port));
        serverChannel.configureBlocking(false);
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
    }

    public void start() throws IOException {
        while (!Thread.currentThread().isInterrupted()) {
            selector.select();
            Iterator<SelectionKey> keys = selector.selectedKeys().iterator();

            while (keys.hasNext()) {
                SelectionKey key = keys.next();
                keys.remove();

                if (key.isAcceptable()) {
                    accept(key);
                } else if (key.isReadable()) {
                    // ✅ Offload processing to worker thread pool
                    key.interestOps(0);  // temporarily deregister
                    workerPool.submit(() -> processRead(key));
                }
            }
        }
    }

    private void accept(SelectionKey key) throws IOException {
        ServerSocketChannel server = (ServerSocketChannel) key.channel();
        SocketChannel client = server.accept();
        client.configureBlocking(false);
        client.register(selector, SelectionKey.OP_READ);
    }

    private void processRead(SelectionKey key) {
        SocketChannel channel = (SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(4096);
        try {
            int bytesRead = channel.read(buffer);
            if (bytesRead == -1) {
                key.cancel();
                channel.close();
                return;
            }
            buffer.flip();
            // Heavy processing on worker thread...
            channel.write(buffer);
            key.interestOps(SelectionKey.OP_READ);  // re-register
            selector.wakeup();
        } catch (IOException e) {
            try { key.cancel(); channel.close(); } catch (IOException ignored) {}
        }
    }
}
```

---

## Common Anti-Patterns

```java
// ❌ Forgetting to remove keys from selectedKeys
for (SelectionKey key : selector.selectedKeys()) {
    handle(key);
    // selectedKeys not cleared — same key processed again!
}

// ✅ Always remove processed keys
Iterator<SelectionKey> it = selector.selectedKeys().iterator();
while (it.hasNext()) {
    SelectionKey key = it.next();
    it.remove();  // remove BEFORE processing
    handle(key);
}

// ❌ Registering a blocking channel with a selector
SocketChannel channel = SocketChannel.open();
// channel.configureBlocking(false);  — forgot this!
channel.register(selector, SelectionKey.OP_READ);
// throws IllegalBlockingModeException

// ❌ Blocking I/O inside the selector thread
private void handleRead(SelectionKey key) {
    // database call or HTTP request here blocks the entire selector!
    database.query("SELECT ...");  // blocks all other channels
}

// ✅ Offload blocking work to a thread pool
workerPool.submit(() -> database.query("SELECT ..."));
```

---

## Key Takeaways

1. **Selectors** enable a single thread to handle thousands of connections (I/O multiplexing)
2. **Always remove processed keys** from `selectedKeys()` — the selector doesn't do it for you
3. **Channels must be non-blocking** to register with a selector
4. **Reactor pattern** separates I/O dispatching (reactor) from processing (handlers)
5. **Never block inside the selector thread** — offload heavy work to worker threads
6. **Use `selector.wakeup()`** to interrupt a blocking `select()` call from another thread

---

[← Previous: Q7 - NIO Buffers](./q07-nio-buffers.md) | [Next: Q9 - Memory-Mapped Files →](./q09-memory-mapped-files.md)

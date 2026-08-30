# I/O Performance Optimization

[Home](../../README.md) > [Performance Optimization](./README.md) > I/O Optimization

---

## Overview

I/O operations are often the biggest performance bottleneck in Java applications. Optimizing I/O involves reducing system calls, leveraging kernel-level optimizations, and choosing the right I/O model for your workload.

---

## 1. Buffered I/O

Every unbuffered read/write results in a system call. Buffering amortizes system call overhead.

```java
// BAD: Unbuffered - one system call per byte/character
try (FileReader reader = new FileReader("data.txt")) {
    int ch;
    while ((ch = reader.read()) != -1) {  // system call per character!
        process((char) ch);
    }
}

// GOOD: Buffered - one system call per buffer fill (default 8 KB)
try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        process(line);
    }
}

// BETTER: Larger buffer for big files
try (BufferedReader reader = new BufferedReader(
        new FileReader("bigdata.txt"), 64 * 1024)) { // 64 KB buffer
    String line;
    while ((line = reader.readLine()) != null) {
        process(line);
    }
}

// BEST (Java 8+): Files utility methods (internally buffered)
List<String> lines = Files.readAllLines(Path.of("data.txt"));
// Or streaming for large files:
try (Stream<String> lines = Files.lines(Path.of("bigdata.txt"))) {
    lines.filter(line -> line.contains("ERROR"))
         .forEach(System.out::println);
}
```

### Buffered I/O Performance Comparison

```
Reading 100 MB file:
  Unbuffered (byte-by-byte):     ~12,000 ms
  BufferedInputStream (8 KB):    ~120 ms     (100x faster)
  BufferedInputStream (64 KB):   ~80 ms      (150x faster)
  Files.readAllBytes():          ~60 ms      (200x faster)
```

```java
// Buffered binary I/O
try (BufferedOutputStream bos = new BufferedOutputStream(
        new FileOutputStream("output.dat"), 32 * 1024)) {
    for (int i = 0; i < 1_000_000; i++) {
        bos.write(intToBytes(i));
    }
} // flush happens automatically on close
```

---

## 2. NIO for Scalability

```java
import java.nio.*;
import java.nio.channels.*;
import java.nio.file.*;

// NIO Channel-based file reading (more control than streams)
public class NioFileReader {

    public static String readFile(Path path) throws IOException {
        try (FileChannel channel = FileChannel.open(path, StandardOpenOption.READ)) {
            ByteBuffer buffer = ByteBuffer.allocate((int) channel.size());
            channel.read(buffer);
            buffer.flip();
            return StandardCharsets.UTF_8.decode(buffer).toString();
        }
    }

    // Chunked reading for very large files
    public static void processLargeFile(Path path) throws IOException {
        try (FileChannel channel = FileChannel.open(path, StandardOpenOption.READ)) {
            ByteBuffer buffer = ByteBuffer.allocateDirect(64 * 1024); // direct buffer

            while (channel.read(buffer) != -1) {
                buffer.flip();
                processBuffer(buffer);
                buffer.clear();
            }
        }
    }
}

// NIO non-blocking server (scalable to thousands of connections)
public class NioServer {
    public void start(int port) throws IOException {
        Selector selector = Selector.open();
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(port));
        serverChannel.configureBlocking(false);
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            selector.select(); // blocks until events available
            Iterator<SelectionKey> keys = selector.selectedKeys().iterator();

            while (keys.hasNext()) {
                SelectionKey key = keys.next();
                keys.remove();

                if (key.isAcceptable()) {
                    SocketChannel client = serverChannel.accept();
                    client.configureBlocking(false);
                    client.register(selector, SelectionKey.OP_READ);
                } else if (key.isReadable()) {
                    handleRead(key);
                }
            }
        }
    }
}
```

---

## 3. Memory-Mapped Files

```java
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;

public class MemoryMappedFileDemo {

    // Map file directly into virtual memory
    // OS handles paging - no explicit read() calls needed
    public static long sumIntegers(Path path) throws IOException {
        try (FileChannel channel = FileChannel.open(path, StandardOpenOption.READ)) {
            long fileSize = channel.size();
            MappedByteBuffer buffer = channel.map(
                FileChannel.MapMode.READ_ONLY, 0, fileSize);

            long sum = 0;
            while (buffer.hasRemaining()) {
                sum += buffer.getInt();
            }
            return sum;
        }
    }

    // Memory-mapped for large file random access
    public static void randomAccess(Path path) throws IOException {
        try (FileChannel channel = FileChannel.open(path,
                StandardOpenOption.READ, StandardOpenOption.WRITE)) {

            MappedByteBuffer buffer = channel.map(
                FileChannel.MapMode.READ_WRITE, 0, channel.size());

            // Random access is O(1) - OS manages page cache
            buffer.position(1000);
            int value = buffer.getInt();

            buffer.position(5000);
            buffer.putInt(42);
            // Changes are automatically flushed to disk by OS
        }
    }
}
```

### When to Use Memory-Mapped Files

| Scenario                 | Memory-Mapped | Buffered I/O |
|--------------------------|---------------|--------------|
| Large file, random access| Best          | Slow (seeks) |
| Large file, sequential   | Good          | Good         |
| Small files (< 1 MB)     | Overhead      | Best         |
| Shared memory (IPC)      | Best          | N/A          |
| Frequently accessed data | Best          | OK           |

---

## 4. Zero-Copy Transfers

```java
// Traditional copy: Disk -> Kernel Buffer -> User Buffer -> Kernel Buffer -> Network
// Zero-copy:        Disk -> Kernel Buffer -> Network (no user-space copy)

public class ZeroCopyTransfer {

    // FileChannel.transferTo() uses OS-level zero-copy (sendfile on Linux)
    public static void sendFile(Path filePath, SocketChannel socketChannel)
            throws IOException {
        try (FileChannel fileChannel = FileChannel.open(filePath)) {
            long position = 0;
            long remaining = fileChannel.size();

            while (remaining > 0) {
                long transferred = fileChannel.transferTo(
                    position, remaining, socketChannel);
                position += transferred;
                remaining -= transferred;
            }
        }
    }

    // File-to-file zero-copy
    public static void copyFile(Path source, Path target) throws IOException {
        try (FileChannel srcChannel = FileChannel.open(source, StandardOpenOption.READ);
             FileChannel dstChannel = FileChannel.open(target,
                 StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {

            long size = srcChannel.size();
            srcChannel.transferTo(0, size, dstChannel);
        }
    }
}
```

### Zero-Copy Performance

```
Sending 1 GB file over network:
  Traditional (read + write): ~3.2 seconds  (2 copies through user space)
  transferTo() (zero-copy):   ~1.8 seconds  (kernel-only, no user copy)
  Improvement:                ~44% faster + less CPU usage
```

---

## 5. Asynchronous I/O (AIO)

```java
import java.nio.channels.AsynchronousFileChannel;
import java.nio.channels.CompletionHandler;

public class AsyncFileReader {

    // Callback-based async file reading
    public static void readAsync(Path path) throws IOException {
        AsynchronousFileChannel channel =
            AsynchronousFileChannel.open(path, StandardOpenOption.READ);

        ByteBuffer buffer = ByteBuffer.allocate(1024);

        channel.read(buffer, 0, buffer, new CompletionHandler<Integer, ByteBuffer>() {
            @Override
            public void completed(Integer bytesRead, ByteBuffer buf) {
                buf.flip();
                byte[] data = new byte[buf.remaining()];
                buf.get(data);
                System.out.println("Read " + bytesRead + " bytes");
                System.out.println(new String(data));
            }

            @Override
            public void failed(Throwable exc, ByteBuffer buf) {
                System.err.println("Read failed: " + exc.getMessage());
            }
        });
    }

    // Future-based async reading
    public static String readAsyncFuture(Path path) throws Exception {
        try (AsynchronousFileChannel channel =
                AsynchronousFileChannel.open(path, StandardOpenOption.READ)) {

            ByteBuffer buffer = ByteBuffer.allocate((int) channel.size());
            Future<Integer> result = channel.read(buffer, 0);

            // Do other work while reading...
            doOtherWork();

            // Block only when result is needed
            int bytesRead = result.get();
            buffer.flip();
            return StandardCharsets.UTF_8.decode(buffer).toString();
        }
    }
}
```

---

## 6. Compression

```java
import java.util.zip.*;

public class CompressionOptimization {

    // GZIP compression for I/O-bound scenarios
    // Trades CPU time for reduced I/O volume
    public static void writeCompressed(Path path, List<String> data)
            throws IOException {
        try (GZIPOutputStream gzos = new GZIPOutputStream(
                new BufferedOutputStream(
                    Files.newOutputStream(path)), 8192)) {
            for (String line : data) {
                gzos.write(line.getBytes(StandardCharsets.UTF_8));
                gzos.write('\n');
            }
        }
    }

    public static List<String> readCompressed(Path path) throws IOException {
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(
                    new GZIPInputStream(
                        new BufferedInputStream(
                            Files.newInputStream(path)))))) {
            return reader.lines().collect(Collectors.toList());
        }
    }
}
```

### Compression Trade-offs

```
1 GB text file:
  Uncompressed:  1000 MB, Read: 1.5s
  GZIP (level 1): 350 MB, Read: 1.8s  (CPU cost < I/O savings for slow disk)
  GZIP (level 6): 250 MB, Read: 2.5s  (better ratio, more CPU)
  GZIP (level 9): 240 MB, Read: 4.0s  (diminishing returns)
  
  Rule: Use compression when I/O bandwidth is the bottleneck (network, slow disk)
        Skip compression when CPU is the bottleneck (fast SSD, simple processing)
```

---

## 7. Connection Pooling

```java
// HTTP connection pooling with HttpClient (Java 11+)
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)          // multiplexing
    .connectTimeout(Duration.ofSeconds(10))
    .executor(Executors.newFixedThreadPool(10))   // shared thread pool
    .build();

// The HttpClient internally pools connections
// Reuse the same HttpClient instance across requests!

// BAD: Creating new client per request
public String fetchBad(String url) throws Exception {
    HttpClient client = HttpClient.newHttpClient(); // new pool each time!
    HttpRequest request = HttpRequest.newBuilder(URI.create(url)).build();
    return client.send(request, HttpResponse.BodyHandlers.ofString()).body();
}

// GOOD: Reuse shared client
private static final HttpClient SHARED_CLIENT = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(5))
    .build();

public String fetchGood(String url) throws Exception {
    HttpRequest request = HttpRequest.newBuilder(URI.create(url)).build();
    return SHARED_CLIENT.send(request, HttpResponse.BodyHandlers.ofString()).body();
}
```

---

## 8. I/O Optimization Summary

| Technique              | When to Use                         | Speedup     |
|------------------------|-------------------------------------|-------------|
| Buffered I/O           | Always (baseline optimization)      | 100x+       |
| Larger buffer sizes    | Large files, sequential access      | 1.5-2x      |
| NIO channels           | Network servers, many connections   | Scalability |
| Memory-mapped files    | Random access, large files          | 2-5x        |
| Zero-copy (transferTo) | File serving, file-to-socket        | 1.5-2x      |
| Async I/O              | Non-blocking architectures          | Throughput   |
| Compression            | I/O-bound, compressible data        | 2-4x        |
| Connection pooling     | HTTP/DB connections                 | 10-50x      |
| Direct ByteBuffers     | Long-lived buffers, native I/O      | 1.2-1.5x    |

---

## Interview Tips

- Always use buffered I/O - the single biggest I/O optimization
- `Files.lines()` for lazy, streaming line reading of large files
- Memory-mapped files excel at random access on large files
- `FileChannel.transferTo()` enables zero-copy file transfers
- Reuse `HttpClient` instances; they pool connections internally
- Compression helps when I/O is the bottleneck, not CPU
- NIO selectors allow one thread to manage thousands of connections
- Direct `ByteBuffer` for long-lived I/O buffers; heap buffers for short-lived

---

[<< Previous: String Optimization](./q06-string-optimization.md) | [Next: JDBC Optimization >>](./q08-jdbc-optimization.md)

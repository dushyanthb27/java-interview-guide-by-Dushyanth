# Q9: Memory-Mapped Files

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟠 Advanced

---

## Question
What are memory-mapped files in Java? How do they work and when should you use them?

---

## Answer

**Memory-mapped files** allow you to map a region of a file directly into memory, so it can be accessed as if it were a byte array. The OS handles reading from and writing to the disk transparently via virtual memory. This avoids explicit read/write system calls and can dramatically improve performance for large files.

---

## How It Works

```
Traditional I/O:
  Application → read() syscall → Kernel buffer → User buffer → Application

Memory-Mapped I/O:
  Application → Virtual memory (mapped to file) → Direct access
  (OS handles page faults and disk I/O transparently)
```

---

## MappedByteBuffer Basics

```java
// ✅ Map a file into memory for reading
try (FileChannel channel = FileChannel.open(Path.of("data.bin"), StandardOpenOption.READ)) {

    long fileSize = channel.size();
    MappedByteBuffer buffer = channel.map(
        FileChannel.MapMode.READ_ONLY,  // map mode
        0,                               // offset (start of file)
        fileSize                         // size to map
    );

    // Access file data like a ByteBuffer
    while (buffer.hasRemaining()) {
        byte b = buffer.get();
        // process byte
    }
}
```

---

## Map Modes

| Mode | Description | Requires |
|------|-------------|----------|
| `READ_ONLY` | Read-only access; writing throws `ReadOnlyBufferException` | `StandardOpenOption.READ` |
| `READ_WRITE` | Changes are written back to the file | `READ` + `WRITE` |
| `PRIVATE` | Copy-on-write; changes are local, not written to file | `READ` + `WRITE` |

```java
// ✅ READ_ONLY — safe, read-only view
try (FileChannel ch = FileChannel.open(Path.of("config.dat"), StandardOpenOption.READ)) {
    MappedByteBuffer readOnly = ch.map(FileChannel.MapMode.READ_ONLY, 0, ch.size());
    int value = readOnly.getInt(0);
    // readOnly.putInt(0, 42);  // ❌ ReadOnlyBufferException!
}

// ✅ READ_WRITE — changes are propagated to the file
try (FileChannel ch = FileChannel.open(Path.of("data.dat"),
        StandardOpenOption.READ, StandardOpenOption.WRITE)) {
    MappedByteBuffer readWrite = ch.map(FileChannel.MapMode.READ_WRITE, 0, ch.size());
    readWrite.putInt(0, 42);   // written to file (eventually)
    readWrite.force();          // flush changes to disk immediately
}

// ✅ PRIVATE (copy-on-write) — modifications are local only
try (FileChannel ch = FileChannel.open(Path.of("template.dat"),
        StandardOpenOption.READ, StandardOpenOption.WRITE)) {
    MappedByteBuffer privateBuf = ch.map(FileChannel.MapMode.PRIVATE, 0, ch.size());
    privateBuf.putInt(0, 99);  // only in-memory, file is unchanged
}
```

---

## Performance: Memory-Mapped vs Traditional I/O

```java
public class MappedFileBenchmark {

    private static final Path FILE = Path.of("test-100mb.dat");
    private static final int FILE_SIZE = 100 * 1024 * 1024; // 100 MB

    public static void main(String[] args) throws IOException {
        // Create test file
        createTestFile();

        // ❌ Traditional I/O: ~500ms for 100MB
        long start = System.nanoTime();
        traditionalRead();
        System.out.printf("Traditional I/O: %d ms%n",
            (System.nanoTime() - start) / 1_000_000);

        // ✅ Memory-mapped I/O: ~50ms for 100MB
        start = System.nanoTime();
        mappedRead();
        System.out.printf("Memory-mapped:   %d ms%n",
            (System.nanoTime() - start) / 1_000_000);

        Files.delete(FILE);
    }

    private static void traditionalRead() throws IOException {
        try (FileInputStream fis = new FileInputStream(FILE.toFile())) {
            byte[] buffer = new byte[8192];
            long sum = 0;
            int bytesRead;
            while ((bytesRead = fis.read(buffer)) != -1) {
                for (int i = 0; i < bytesRead; i++) {
                    sum += buffer[i];
                }
            }
        }
    }

    private static void mappedRead() throws IOException {
        try (FileChannel channel = FileChannel.open(FILE, StandardOpenOption.READ)) {
            MappedByteBuffer buffer = channel.map(
                FileChannel.MapMode.READ_ONLY, 0, channel.size());
            long sum = 0;
            while (buffer.hasRemaining()) {
                sum += buffer.get();
            }
        }
    }

    private static void createTestFile() throws IOException {
        byte[] data = new byte[FILE_SIZE];
        Arrays.fill(data, (byte) 1);
        Files.write(FILE, data);
    }
}
```

---

## Mapping Large Files (> 2GB)

`MappedByteBuffer` inherits from `ByteBuffer`, which uses `int` for positions — limiting a single mapping to ~2GB. Map in chunks for larger files.

```java
// ✅ Map a large file in chunks
public class LargeFileMapper {

    private static final long CHUNK_SIZE = 1L * 1024 * 1024 * 1024; // 1GB chunks

    public static void processLargeFile(Path path) throws IOException {
        try (FileChannel channel = FileChannel.open(path, StandardOpenOption.READ)) {
            long fileSize = channel.size();
            long offset = 0;

            while (offset < fileSize) {
                long remaining = fileSize - offset;
                long mapSize = Math.min(CHUNK_SIZE, remaining);

                MappedByteBuffer buffer = channel.map(
                    FileChannel.MapMode.READ_ONLY, offset, mapSize);

                processChunk(buffer);
                offset += mapSize;
            }
        }
    }

    private static void processChunk(MappedByteBuffer buffer) {
        while (buffer.hasRemaining()) {
            // process data
            buffer.get();
        }
    }
}
```

---

## Use Case: Shared Memory Between Processes

Memory-mapped files can be used for inter-process communication (IPC).

```java
// ✅ Process A: Writer
public class SharedMemoryWriter {
    public static void main(String[] args) throws Exception {
        Path sharedFile = Path.of("shared-memory.dat");
        try (FileChannel channel = FileChannel.open(sharedFile,
                StandardOpenOption.CREATE, StandardOpenOption.READ, StandardOpenOption.WRITE)) {

            MappedByteBuffer buffer = channel.map(
                FileChannel.MapMode.READ_WRITE, 0, 1024);

            for (int i = 0; i < 100; i++) {
                buffer.putInt(0, i);        // write counter at position 0
                buffer.force();              // flush to file
                Thread.sleep(100);
            }
        }
    }
}

// ✅ Process B: Reader
public class SharedMemoryReader {
    public static void main(String[] args) throws Exception {
        Path sharedFile = Path.of("shared-memory.dat");
        try (FileChannel channel = FileChannel.open(sharedFile, StandardOpenOption.READ)) {

            MappedByteBuffer buffer = channel.map(
                FileChannel.MapMode.READ_ONLY, 0, 1024);

            int lastValue = -1;
            while (true) {
                buffer.load();  // refresh from file
                int value = buffer.getInt(0);
                if (value != lastValue) {
                    System.out.println("Read: " + value);
                    lastValue = value;
                }
                Thread.sleep(50);
            }
        }
    }
}
```

---

## Use Case: Binary File Search

```java
// ✅ Search for a byte pattern in a large file using memory-mapping
public class BinarySearch {

    public static long findPattern(Path file, byte[] pattern) throws IOException {
        try (FileChannel channel = FileChannel.open(file, StandardOpenOption.READ)) {
            MappedByteBuffer buffer = channel.map(
                FileChannel.MapMode.READ_ONLY, 0, channel.size());

            for (int i = 0; i <= buffer.limit() - pattern.length; i++) {
                boolean match = true;
                for (int j = 0; j < pattern.length; j++) {
                    if (buffer.get(i + j) != pattern[j]) {
                        match = false;
                        break;
                    }
                }
                if (match) return i;
            }
            return -1; // not found
        }
    }
}
```

---

## Memory Management Considerations

```java
// ⚠️ MappedByteBuffer is NOT unmapped when the channel closes
// The mapping persists until the buffer is garbage collected

try (FileChannel channel = FileChannel.open(path, StandardOpenOption.READ)) {
    MappedByteBuffer buffer = channel.map(FileChannel.MapMode.READ_ONLY, 0, channel.size());
    // channel closes here, but buffer is STILL valid and mapped
}
// buffer is still mapped — the file may remain locked on Windows!

// ✅ Force cleanup (not guaranteed, but helps)
// There is no standard API to unmap a MappedByteBuffer
// The sun.misc.Cleaner approach is internal and non-portable

// ✅ Best practice: let GC handle it, avoid on Windows if file deletion needed
System.gc();  // hint to GC (not guaranteed)

// ✅ Java 19+ (Preview): MemorySegment from Foreign Function & Memory API
// Provides deterministic deallocation
```

---

## Common Anti-Patterns

```java
// ❌ Mapping very small files — overhead isn't worth it
MappedByteBuffer buf = channel.map(MapMode.READ_ONLY, 0, 100);
// For small files, traditional read is simpler and equally fast

// ❌ Forgetting force() for critical writes
buffer.putInt(0, importantValue);
// force() not called — data may be lost on crash!

// ✅ Call force() for durability
buffer.putInt(0, importantValue);
buffer.force();  // ensures data reaches disk

// ❌ Mapping beyond file size without extending
// Will work (file is extended) but may cause issues
```

---

## ⚠️ Gotchas

- **Windows File Locking (`AccessDeniedException` on Delete):** Unlike POSIX systems where an open file can be unlinked while descriptors remain active, Windows strictly locks memory-mapped files. Because `MappedByteBuffer` unmapping is non-deterministic (tied to JVM Garbage Collection), attempting to delete or overwrite a mapped file immediately after closing its `FileChannel` on Windows will frequently throw `AccessDeniedException` or `IOException`. Developers must either avoid deleting mapped files on Windows or use Java 19+ Foreign Function & Memory API (`MemorySegment`) for deterministic resource release.
- **JVM Crash on Accessing Truncated Mapped Files (`SIGSEGV`):** If a memory-mapped file is truncated externally (e.g., another process shrinks the file while your JVM has it mapped), attempting to read or write past the new file boundary results in a native segmentation fault (`SIGSEGV`), crashing the entire JVM process instead of throwing a clean Java exception.
- **Page Fault Latency / Disk Thrashing:** While memory-mapped I/O feels like accessing an in-memory array, the underlying data resides on disk. Accessing unmapped pages triggers synchronous OS page faults that block the calling thread while disk I/O occurs. For large files that exceed available RAM, random access across non-contiguous regions can cause severe disk thrashing and latency spikes.
- **JVM 2GB Limit per Mapping (`IllegalArgumentException`):** `MappedByteBuffer` extends `ByteBuffer`, which uses signed 32-bit integers (`int`) for indexing. Consequently, a single `channel.map()` call cannot exceed `Integer.MAX_VALUE` bytes (~2GB). Attempting to map a larger file in one go throws an `IllegalArgumentException`. Large files must be sliced and mapped in 1GB–2GB chunks.
- **Cache Coherency and `force()` Durability:** Writes to a `READ_WRITE` `MappedByteBuffer` modify virtual memory pages managed by the OS kernel. They are not guaranteed to be physically written to persistent storage immediately. If the OS crashes or loses power before the kernel flushes dirty pages, data is lost unless `buffer.force()` is explicitly invoked to force disk synchronization.

---

## Key Takeaways

1. **Memory-mapped files** map file contents directly into virtual memory — no explicit read/write syscalls
2. **`MappedByteBuffer`** is created via `FileChannel.map()` with three modes: `READ_ONLY`, `READ_WRITE`, `PRIVATE`
3. **Performance**: 5-10x faster than traditional I/O for large files (100MB+)
4. **Use `force()`** to ensure changes are flushed to disk for durability
5. **2GB limit** per mapping — use chunked mapping for larger files
6. **Unmapping is non-deterministic** — handled by GC, which can cause file-lock issues on Windows
7. **Best for**: large files, inter-process communication, read-heavy workloads

---

[← Previous: Q8 - NIO Selectors](./q08-nio-selectors.md) | [Next: Q10 - Path API →](./q10-path-api.md)

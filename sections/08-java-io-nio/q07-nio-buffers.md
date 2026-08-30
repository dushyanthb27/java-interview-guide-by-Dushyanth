# Q7: NIO Buffers In Depth

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟠 Advanced

---

## Question
How do NIO Buffers work? Explain the concepts of capacity, position, limit, and the key buffer operations.

---

## Answer

A **Buffer** is a fixed-size container that holds data for reading from or writing to a channel. All NIO I/O goes through buffers — channels read data *into* buffers and write data *from* buffers.

---

## Buffer Types

```
Buffer (abstract)
├── ByteBuffer          — most common, works with channels
├── CharBuffer          — for characters
├── ShortBuffer         — for short values
├── IntBuffer           — for int values
├── LongBuffer          — for long values
├── FloatBuffer         — for float values
└── DoubleBuffer        — for double values
```

---

## Buffer Properties: Capacity, Position, Limit, Mark

Every buffer has four core properties:

```
0 <= mark <= position <= limit <= capacity

┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ H │ e │ l │ l │ o │   │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7   8   9
                      ↑                   ↑
                   position            capacity
                   (& limit after flip)
```

| Property | Description |
|----------|-------------|
| **Capacity** | Maximum number of elements the buffer can hold (fixed at creation) |
| **Position** | Index of the next element to be read or written |
| **Limit** | First index that should NOT be read or written |
| **Mark** | A saved position that can be returned to via `reset()` |

---

## Buffer Lifecycle: Write → Flip → Read → Clear

```java
// ✅ Complete buffer lifecycle
ByteBuffer buffer = ByteBuffer.allocate(10);
// State: position=0, limit=10, capacity=10

// 1. WRITE data into buffer
buffer.put((byte) 'H');
buffer.put((byte) 'e');
buffer.put((byte) 'l');
buffer.put((byte) 'l');
buffer.put((byte) 'o');
// State: position=5, limit=10, capacity=10

// 2. FLIP — prepare for reading (sets limit=position, position=0)
buffer.flip();
// State: position=0, limit=5, capacity=10

// 3. READ data from buffer
while (buffer.hasRemaining()) {
    System.out.print((char) buffer.get());  // H e l l o
}
// State: position=5, limit=5, capacity=10

// 4. CLEAR — prepare for writing again (resets position=0, limit=capacity)
buffer.clear();
// State: position=0, limit=10, capacity=10
```

### Visual: Buffer Operations

```
After allocate(10):    [  |  |  |  |  |  |  |  |  |  ]
                        ↑ position                      ↑ limit=capacity

After put("Hello"):   [ H| e| l| l| o|  |  |  |  |  ]
                                          ↑ position    ↑ limit=capacity

After flip():          [ H| e| l| l| o|  |  |  |  |  ]
                        ↑ position       ↑ limit        ↑ capacity

After get() x5:        [ H| e| l| l| o|  |  |  |  |  ]
                                          ↑ pos=limit   ↑ capacity

After clear():         [ H| e| l| l| o|  |  |  |  |  ]
                        ↑ position                      ↑ limit=capacity
```

---

## Key Buffer Operations

### flip(), clear(), compact(), rewind()

```java
ByteBuffer buffer = ByteBuffer.allocate(10);

// flip() — prepare for reading: limit = position, position = 0
buffer.put("Hello".getBytes());
buffer.flip();
// Use after writing to buffer, before reading from buffer

// rewind() — re-read data: position = 0 (limit unchanged)
buffer.rewind();
// Use to re-read the same data

// clear() — prepare for writing: position = 0, limit = capacity
buffer.clear();
// Discards all data (logically, not physically)

// compact() — keep unread data, prepare for writing
buffer.put("Hello World".getBytes());
buffer.flip();
buffer.get(); // read 'H'
buffer.get(); // read 'e'
buffer.compact();
// Copies unread data ("llo World") to beginning
// position = 9 (after unread data), limit = capacity
// Ready for more writing while preserving unread data
```

### mark() and reset()

```java
ByteBuffer buffer = ByteBuffer.wrap("Hello, World!".getBytes());

buffer.get(); // H — position=1
buffer.get(); // e — position=2

buffer.mark();  // save position=2

buffer.get(); // l — position=3
buffer.get(); // l — position=4

buffer.reset();  // restore position=2

buffer.get(); // l again — position=3
```

---

## Buffer Allocation

### Heap Buffers vs Direct Buffers

```java
// ✅ Heap buffer — allocated in JVM heap memory
ByteBuffer heapBuffer = ByteBuffer.allocate(1024);
// Pros: faster allocation, managed by GC
// Cons: extra copy during I/O (heap → native → OS)

// ✅ Direct buffer — allocated in native (off-heap) memory
ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024);
// Pros: faster I/O (no extra copy — native → OS)
// Cons: slower allocation, not managed by GC, higher memory overhead

// ✅ Wrap an existing array
byte[] data = "Hello".getBytes(StandardCharsets.UTF_8);
ByteBuffer wrapped = ByteBuffer.wrap(data);
// Changes to buffer affect array and vice versa

// Check if direct
boolean isDirect = directBuffer.isDirect();  // true
boolean isHeap = heapBuffer.isDirect();       // false
```

### When to Use Direct Buffers

```java
// ✅ Good: Long-lived buffers used for heavy I/O (network servers)
public class NetworkServer {
    // Allocate once, reuse many times
    private final ByteBuffer ioBuffer = ByteBuffer.allocateDirect(65536);

    public void handleRequest(SocketChannel channel) throws IOException {
        ioBuffer.clear();
        channel.read(ioBuffer);
        ioBuffer.flip();
        channel.write(ioBuffer);
    }
}

// ❌ Bad: Short-lived buffers (allocation cost outweighs I/O benefit)
public void processMessage(String msg) {
    ByteBuffer buffer = ByteBuffer.allocateDirect(msg.length()); // wasteful!
    // Direct buffer allocation is expensive for short-lived objects
}
```

---

## Typed Buffer Access

```java
ByteBuffer buffer = ByteBuffer.allocate(256);

// ✅ Write typed data
buffer.putInt(42);           // 4 bytes
buffer.putLong(123456789L);  // 8 bytes
buffer.putDouble(3.14);      // 8 bytes
buffer.putChar('A');          // 2 bytes
buffer.putShort((short) 100);// 2 bytes

buffer.flip();

// ✅ Read typed data (same order!)
int i = buffer.getInt();
long l = buffer.getLong();
double d = buffer.getDouble();
char c = buffer.getChar();
short s = buffer.getShort();

// ✅ Byte order (endianness)
buffer.order(ByteOrder.BIG_ENDIAN);     // default — network byte order
buffer.order(ByteOrder.LITTLE_ENDIAN);  // x86 native order
buffer.order(ByteOrder.nativeOrder());  // platform-native
```

---

## Slice and Duplicate

```java
ByteBuffer original = ByteBuffer.allocate(10);
original.put("HelloWorld".getBytes());
original.flip();

// ✅ slice() — creates a new buffer sharing the remaining content
original.position(5);
ByteBuffer slice = original.slice();
// slice sees "World" — independent position/limit, shared data
// Changes to slice's DATA affect original and vice versa

// ✅ duplicate() — shares ALL content with independent position/limit
ByteBuffer dup = original.duplicate();
// dup sees same data as original but has its own position/limit/mark

// ✅ asReadOnlyBuffer() — read-only view
ByteBuffer readOnly = original.asReadOnlyBuffer();
// readOnly.put((byte) 'X');  // throws ReadOnlyBufferException
```

---

## View Buffers

```java
// ✅ Create typed views of a ByteBuffer
ByteBuffer byteBuffer = ByteBuffer.allocate(16);

// IntBuffer view — each int = 4 bytes
IntBuffer intView = byteBuffer.asIntBuffer();
intView.put(100);
intView.put(200);
intView.put(300);
intView.put(400);
// byteBuffer now contains these ints as bytes

// CharBuffer view — each char = 2 bytes
ByteBuffer charBytes = ByteBuffer.allocate(10);
CharBuffer charView = charBytes.asCharBuffer();
charView.put("Hello");
```

---

## Practical Example: Protocol Parser

```java
// ✅ Parse a simple binary protocol: [4-byte length][N-byte payload]
public class ProtocolParser {

    public static String readMessage(SocketChannel channel) throws IOException {
        // Read the 4-byte length header
        ByteBuffer headerBuffer = ByteBuffer.allocate(4);
        while (headerBuffer.hasRemaining()) {
            if (channel.read(headerBuffer) == -1) {
                throw new IOException("Connection closed");
            }
        }
        headerBuffer.flip();
        int payloadLength = headerBuffer.getInt();

        // Read the payload
        ByteBuffer payloadBuffer = ByteBuffer.allocate(payloadLength);
        while (payloadBuffer.hasRemaining()) {
            if (channel.read(payloadBuffer) == -1) {
                throw new IOException("Connection closed");
            }
        }
        payloadBuffer.flip();
        return StandardCharsets.UTF_8.decode(payloadBuffer).toString();
    }

    public static void writeMessage(SocketChannel channel, String message) throws IOException {
        byte[] payload = message.getBytes(StandardCharsets.UTF_8);

        ByteBuffer buffer = ByteBuffer.allocate(4 + payload.length);
        buffer.putInt(payload.length);     // 4-byte length header
        buffer.put(payload);                // N-byte payload
        buffer.flip();

        while (buffer.hasRemaining()) {
            channel.write(buffer);
        }
    }
}
```

---

## Common Anti-Patterns

```java
// ❌ Forgetting to flip before reading
ByteBuffer buffer = ByteBuffer.allocate(10);
buffer.put("Hi".getBytes());
byte b = buffer.get();  // reads at position 2, NOT position 0!
// ✅ Fix: buffer.flip() before reading

// ❌ Using clear() when compact() is needed
ByteBuffer buf = ByteBuffer.allocate(10);
buf.put("Hello".getBytes());
buf.flip();
buf.get(); buf.get(); // read "He"
buf.clear();  // discards unread "llo"!
// ✅ Fix: buf.compact() to keep unread data

// ❌ Allocating direct buffers in tight loops
for (int i = 0; i < 1000; i++) {
    ByteBuffer db = ByteBuffer.allocateDirect(1024); // expensive!
}
// ✅ Fix: allocate once, reuse with clear()
```

---

## Key Takeaways

1. **flip()** before reading, **clear()**/**compact()** before writing — this is the fundamental buffer lifecycle
2. **Direct buffers** are faster for I/O but expensive to allocate — reuse them for long-lived I/O operations
3. **Heap buffers** are fine for most use cases — faster allocation, GC-managed
4. **compact()** preserves unread data; **clear()** discards everything
5. **slice()** and **duplicate()** share underlying data but have independent positions
6. **Read/write order must match** when using typed access (`putInt`/`getInt`)

---

[← Previous: Q6 - NIO Channels](./q06-nio-channels.md) | [Next: Q8 - NIO Selectors →](./q08-nio-selectors.md)

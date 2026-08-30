# Q11: File Watching with WatchService

[← Back to Java I/O & NIO](./README.md) | Difficulty: 🟡 Medium

---

## Question
How does the `WatchService` API work in Java? What are its use cases and limitations?

---

## Answer

The **WatchService** API (`java.nio.file.WatchService`, Java 7+) monitors directories for changes — file creation, modification, and deletion. It uses native OS file-watching mechanisms (inotify on Linux, kqueue on macOS, ReadDirectoryChangesW on Windows) when available, falling back to polling.

---

## Core Components

| Component | Description |
|-----------|-------------|
| `WatchService` | The watcher instance; monitors registered directories |
| `WatchKey` | Represents a directory registration; holds pending events |
| `WatchEvent` | A single file-system event (create, modify, delete) |
| `WatchEvent.Kind` | The type of event |

### Event Kinds

| Kind | Constant | Description |
|------|----------|-------------|
| Create | `StandardWatchEventKinds.ENTRY_CREATE` | File/directory created |
| Modify | `StandardWatchEventKinds.ENTRY_MODIFY` | File/directory modified |
| Delete | `StandardWatchEventKinds.ENTRY_DELETE` | File/directory deleted |
| Overflow | `StandardWatchEventKinds.OVERFLOW` | Events may have been lost |

---

## Basic File Watcher

```java
public class BasicFileWatcher {

    public static void main(String[] args) throws IOException, InterruptedException {
        Path directory = Path.of("watched-dir");
        Files.createDirectories(directory);

        // ✅ Create a WatchService
        try (WatchService watchService = FileSystems.getDefault().newWatchService()) {

            // ✅ Register directory for specific events
            directory.register(watchService,
                StandardWatchEventKinds.ENTRY_CREATE,
                StandardWatchEventKinds.ENTRY_MODIFY,
                StandardWatchEventKinds.ENTRY_DELETE
            );

            System.out.println("Watching: " + directory.toAbsolutePath());

            // ✅ Event loop
            while (true) {
                WatchKey key = watchService.take();  // blocks until events arrive

                for (WatchEvent<?> event : key.pollEvents()) {
                    WatchEvent.Kind<?> kind = event.kind();

                    // Handle overflow
                    if (kind == StandardWatchEventKinds.OVERFLOW) {
                        System.out.println("Overflow — events may have been lost");
                        continue;
                    }

                    // The filename is the context of the event
                    @SuppressWarnings("unchecked")
                    WatchEvent<Path> pathEvent = (WatchEvent<Path>) event;
                    Path fileName = pathEvent.context();

                    System.out.printf("[%s] %s%n", kind.name(), fileName);
                }

                // ✅ MUST reset the key — otherwise no more events
                boolean valid = key.reset();
                if (!valid) {
                    System.out.println("Directory no longer accessible");
                    break;
                }
            }
        }
    }
}
```

**Output:**
```
Watching: /home/user/watched-dir
[ENTRY_CREATE] newfile.txt
[ENTRY_MODIFY] newfile.txt
[ENTRY_DELETE] old.log
```

---

## Polling vs Blocking

```java
// ✅ Blocking: take() — blocks until events arrive
WatchKey key = watchService.take();

// ✅ Timed wait: poll(timeout) — waits up to specified time
WatchKey key = watchService.poll(5, TimeUnit.SECONDS);
if (key == null) {
    System.out.println("No events in 5 seconds");
}

// ✅ Non-blocking: poll() — returns immediately
WatchKey key = watchService.poll();
if (key != null) {
    // process events
}
```

---

## Watching Multiple Directories

```java
public class MultiDirectoryWatcher {

    public static void main(String[] args) throws IOException, InterruptedException {
        WatchService watchService = FileSystems.getDefault().newWatchService();

        // ✅ Register multiple directories
        Map<WatchKey, Path> keyPathMap = new HashMap<>();

        List<Path> directories = List.of(
            Path.of("config"),
            Path.of("logs"),
            Path.of("data")
        );

        for (Path dir : directories) {
            Files.createDirectories(dir);
            WatchKey key = dir.register(watchService,
                StandardWatchEventKinds.ENTRY_CREATE,
                StandardWatchEventKinds.ENTRY_MODIFY,
                StandardWatchEventKinds.ENTRY_DELETE);
            keyPathMap.put(key, dir);
        }

        // ✅ Event loop — determine which directory each event belongs to
        while (true) {
            WatchKey key = watchService.take();
            Path directory = keyPathMap.get(key);

            for (WatchEvent<?> event : key.pollEvents()) {
                if (event.kind() == StandardWatchEventKinds.OVERFLOW) continue;

                @SuppressWarnings("unchecked")
                Path fileName = ((WatchEvent<Path>) event).context();
                Path fullPath = directory.resolve(fileName);

                System.out.printf("[%s] %s in %s%n",
                    event.kind().name(), fileName, directory);
            }

            if (!key.reset()) {
                keyPathMap.remove(key);
                if (keyPathMap.isEmpty()) break;
            }
        }

        watchService.close();
    }
}
```

---

## Recursive Directory Watching

WatchService only monitors a single directory level. For recursive watching, you must register each subdirectory.

```java
public class RecursiveWatcher {

    private final WatchService watchService;
    private final Map<WatchKey, Path> keyPathMap = new HashMap<>();

    public RecursiveWatcher(Path root) throws IOException {
        this.watchService = FileSystems.getDefault().newWatchService();
        registerRecursive(root);
    }

    // ✅ Register all subdirectories
    private void registerRecursive(Path root) throws IOException {
        try (Stream<Path> walk = Files.walk(root)) {
            walk.filter(Files::isDirectory)
                .forEach(dir -> {
                    try {
                        WatchKey key = dir.register(watchService,
                            StandardWatchEventKinds.ENTRY_CREATE,
                            StandardWatchEventKinds.ENTRY_MODIFY,
                            StandardWatchEventKinds.ENTRY_DELETE);
                        keyPathMap.put(key, dir);
                    } catch (IOException e) {
                        throw new UncheckedIOException(e);
                    }
                });
        }
    }

    public void watch() throws InterruptedException, IOException {
        while (true) {
            WatchKey key = watchService.take();
            Path directory = keyPathMap.get(key);

            for (WatchEvent<?> event : key.pollEvents()) {
                if (event.kind() == StandardWatchEventKinds.OVERFLOW) continue;

                @SuppressWarnings("unchecked")
                Path fileName = ((WatchEvent<Path>) event).context();
                Path fullPath = directory.resolve(fileName);

                System.out.printf("[%s] %s%n", event.kind().name(), fullPath);

                // ✅ Register newly created subdirectories
                if (event.kind() == StandardWatchEventKinds.ENTRY_CREATE
                        && Files.isDirectory(fullPath)) {
                    registerRecursive(fullPath);
                }
            }

            if (!key.reset()) {
                keyPathMap.remove(key);
                if (keyPathMap.isEmpty()) break;
            }
        }
    }
}
```

---

## Use Case: Hot Configuration Reload

```java
public class ConfigReloader {

    private volatile Properties config;
    private final Path configFile;

    public ConfigReloader(Path configFile) throws IOException {
        this.configFile = configFile;
        this.config = loadConfig();
        startWatching();
    }

    private Properties loadConfig() throws IOException {
        Properties props = new Properties();
        try (Reader reader = Files.newBufferedReader(configFile, StandardCharsets.UTF_8)) {
            props.load(reader);
        }
        return props;
    }

    // ✅ Watch for config changes and reload automatically
    private void startWatching() {
        Thread watchThread = new Thread(() -> {
            try (WatchService ws = FileSystems.getDefault().newWatchService()) {
                configFile.getParent().register(ws,
                    StandardWatchEventKinds.ENTRY_MODIFY);

                while (!Thread.currentThread().isInterrupted()) {
                    WatchKey key = ws.take();
                    for (WatchEvent<?> event : key.pollEvents()) {
                        @SuppressWarnings("unchecked")
                        Path changed = ((WatchEvent<Path>) event).context();
                        if (changed.equals(configFile.getFileName())) {
                            // Small delay — file may still be writing
                            Thread.sleep(100);
                            config = loadConfig();
                            System.out.println("Config reloaded!");
                        }
                    }
                    key.reset();
                }
            } catch (IOException | InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "config-watcher");
        watchThread.setDaemon(true);
        watchThread.start();
    }

    public String getProperty(String key) {
        return config.getProperty(key);
    }
}
```

---

## Limitations

| Limitation | Description |
|-----------|-------------|
| **Not recursive** | Only watches a single directory — must manually register subdirectories |
| **Platform-dependent behavior** | macOS uses polling (slow); Linux/Windows use native events |
| **Duplicate events** | Some OSes fire multiple MODIFY events for a single file save |
| **No file content** | Events only report file names, not what changed inside |
| **OVERFLOW possible** | Events can be lost if they arrive faster than consumed |
| **Latency on macOS** | macOS polling-based implementation has ~10s delay by default |

### Handling Duplicate Events

```java
// ✅ Debounce: ignore rapid-fire events for the same file
private final Map<Path, Long> lastEventTime = new ConcurrentHashMap<>();
private static final long DEBOUNCE_MS = 500;

private boolean shouldProcess(Path file) {
    long now = System.currentTimeMillis();
    Long lastTime = lastEventTime.get(file);
    if (lastTime != null && (now - lastTime) < DEBOUNCE_MS) {
        return false;  // skip duplicate
    }
    lastEventTime.put(file, now);
    return true;
}
```

---

## Key Takeaways

1. **`WatchService`** monitors directories for file create, modify, and delete events
2. **Always call `key.reset()`** after processing events — otherwise no more events are delivered
3. **Not recursive** by default — register subdirectories manually for recursive watching
4. **Handle `OVERFLOW`** events — they indicate events may have been lost
5. **Platform-dependent**: native on Linux/Windows, polling on macOS (slower)
6. **Debounce duplicate events** — some OSes fire multiple events for a single file change
7. **Common use cases**: hot config reload, file sync, build tool watchers, log monitoring

---

[← Previous: Q10 - Path API](./q10-path-api.md) | [Next: Q12 - Asynchronous I/O →](./q12-async-io.md)

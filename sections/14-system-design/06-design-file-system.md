# Q6: Design a File System

## 📋 Question
Design a distributed file system. Discuss the data structures, metadata management, and consistency guarantees.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Create, read, update, delete files
- Directory structure support
- File metadata (size, permissions, timestamps)
- Distributed storage across nodes
- File versioning

**Non-Functional Requirements:**
- High availability and durability
- Strong consistency
- Scalable to petabytes
- Low latency (< 50ms)

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Client Application                   │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                  Metadata Server (Name Node)             │
│  ┌──────────────────────────────────────────────────┐  │
│  │  File Metadata (Inode Table)                     │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        │  │
│  │  │ File A   │ │ File B   │ │ File C   │        │  │
│  │  └──────────┘ └──────────┘ └──────────┘        │  │
│  └──────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Storage Node│ │  Storage Node│ │  Storage Node│
│      1       │ │      2       │ │      3       │
└──────────────┘ └──────────────┘ └──────────────┘
```

### File Metadata Structure

```java
public class FileMetadata {
    private final String fileId;
    private final String fileName;
    private final long fileSize;
    private final String owner;
    private final Set<String> permissions;
    private final long createdTime;
    private final long modifiedTime;
    private final String checksum;
    private final List<BlockLocation> blockLocations;

    public FileMetadata(String fileId, String fileName, long fileSize, String owner) {
        this.fileId = fileId;
        this.fileName = fileName;
        this.fileSize = fileSize;
        this.owner = owner;
        this.permissions = new HashSet<>();
        this.createdTime = System.currentTimeMillis();
        this.modifiedTime = this.createdTime;
        this.checksum = "";
        this.blockLocations = new ArrayList<>();
    }

    // Getters
    public String getFileId() { return fileId; }
    public String getFileName() { return fileName; }
    public long getFileSize() { return fileSize; }
    public String getOwner() { return owner; }
    public Set<String> getPermissions() { return permissions; }
    public long getCreatedTime() { return createdTime; }
    public long getModifiedTime() { return modifiedTime; }
    public String getChecksum() { return checksum; }
    public List<BlockLocation> getBlockLocations() { return blockLocations; }

    public void addBlockLocation(BlockLocation location) {
        blockLocations.add(location);
    }

    public void setChecksum(String checksum) {
        this.checksum = checksum;
    }
}

public class BlockLocation {
    private final String storageNodeId;
    private final String blockId;
    private final long offset;
    private final long size;

    public BlockLocation(String storageNodeId, String blockId, long offset, long size) {
        this.storageNodeId = storageNodeId;
        this.blockId = blockId;
        this.offset = offset;
        this.size = size;
    }

    // Getters
    public String getStorageNodeId() { return storageNodeId; }
    public String getBlockId() { return blockId; }
    public long getOffset() { return offset; }
    public long getSize() { return size; }
}
```

### Metadata Server Implementation

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class MetadataServer {
    private final Map<String, FileMetadata> fileMetadata;
    private final Map<String, DirectoryMetadata> directoryMetadata;
    private final BlockAllocator blockAllocator;

    public MetadataServer(BlockAllocator blockAllocator) {
        this.fileMetadata = new ConcurrentHashMap<>();
        this.directoryMetadata = new ConcurrentHashMap<>();
        this.blockAllocator = blockAllocator;
    }

    public String createFile(String path, String owner, long fileSize) {
        String fileId = generateFileId();
        FileMetadata metadata = new FileMetadata(fileId, getFileName(path), fileSize, owner);
        
        // Allocate blocks
        List<BlockLocation> blocks = blockAllocator.allocateBlocks(fileSize);
        for (BlockLocation block : blocks) {
            metadata.addBlockLocation(block);
        }
        
        fileMetadata.put(fileId, metadata);
        return fileId;
    }

    public FileMetadata getFileMetadata(String fileId) {
        return fileMetadata.get(fileId);
    }

    public void updateFile(String fileId, byte[] newData) {
        FileMetadata metadata = fileMetadata.get(fileId);
        if (metadata != null) {
            // Reallocate blocks if size changed
            List<BlockLocation> newBlocks = blockAllocator.allocateBlocks(newData.length);
            metadata = new FileMetadata(fileId, metadata.getFileName(), newData.length, metadata.getOwner());
            for (BlockLocation block : newBlocks) {
                metadata.addBlockLocation(block);
            }
            fileMetadata.put(fileId, metadata);
        }
    }

    public void deleteFile(String fileId) {
        FileMetadata metadata = fileMetadata.remove(fileId);
        if (metadata != null) {
            // Free blocks
            for (BlockLocation block : metadata.getBlockLocations()) {
                blockAllocator.freeBlock(block);
            }
        }
    }

    private String generateFileId() {
        return UUID.randomUUID().toString();
    }

    private String getFileName(String path) {
        String[] parts = path.split("/");
        return parts[parts.length - 1];
    }
}

public class DirectoryMetadata {
    private final String path;
    private final Map<String, String> files; // fileName -> fileId
    private final Map<String, String> subdirectories;

    public DirectoryMetadata(String path) {
        this.path = path;
        this.files = new HashMap<>();
        this.subdirectories = new HashMap<>();
    }

    // Getters and methods
    public String getPath() { return path; }
    public Map<String, String> getFiles() { return files; }
    public Map<String, String> getSubdirectories() { return subdirectories; }
}
```

### Block Allocation

```java
import java.util.*;

public class BlockAllocator {
    private final Map<String, StorageNode> storageNodes;
    private final int blockSize;
    private final int replicationFactor;

    public BlockAllocator(List<StorageNode> storageNodes, int blockSize, int replicationFactor) {
        this.storageNodes = new HashMap<>();
        for (StorageNode node : storageNodes) {
            this.storageNodes.put(node.getNodeId(), node);
        }
        this.blockSize = blockSize;
        this.replicationFactor = replicationFactor;
    }

    public List<BlockLocation> allocateBlocks(long fileSize) {
        List<BlockLocation> blocks = new ArrayList<>();
        int numBlocks = (int) Math.ceil((double) fileSize / blockSize);
        
        for (int i = 0; i < numBlocks; i++) {
            // Choose storage nodes for replication
            List<StorageNode> selectedNodes = selectStorageNodes(replicationFactor);
            
            for (StorageNode node : selectedNodes) {
                String blockId = generateBlockId();
                BlockLocation location = new BlockLocation(
                    node.getNodeId(), 
                    blockId, 
                    i * blockSize, 
                    Math.min(blockSize, fileSize - i * blockSize)
                );
                blocks.add(location);
            }
        }
        return blocks;
    }

    public void freeBlock(BlockLocation location) {
        StorageNode node = storageNodes.get(location.getStorageNodeId());
        if (node != null) {
            node.freeBlock(location.getBlockId());
        }
    }

    private List<StorageNode> selectStorageNodes(int count) {
        List<StorageNode> nodes = new ArrayList<>(storageNodes.values());
        Collections.shuffle(nodes);
        return nodes.subList(0, Math.min(count, nodes.size()));
    }

    private String generateBlockId() {
        return UUID.randomUUID().toString();
    }
}

public class StorageNode {
    private final String nodeId;
    private final Set<String> allocatedBlocks;
    private final long capacity;
    private final long usedSpace;

    public StorageNode(String nodeId, long capacity) {
        this.nodeId = nodeId;
        this.allocatedBlocks = new HashSet<>();
        this.capacity = capacity;
        this.usedSpace = 0;
    }

    public void allocateBlock(String blockId) {
        allocatedBlocks.add(blockId);
    }

    public void freeBlock(String blockId) {
        allocatedBlocks.remove(blockId);
    }

    // Getters
    public String getNodeId() { return nodeId; }
    public Set<String> getAllocatedBlocks() { return allocatedBlocks; }
    public long getCapacity() { return capacity; }
    public long getUsedSpace() { return usedSpace; }
}
```

### File Operations

```java
public class FileOperations {
    private final MetadataServer metadataServer;
    private final StorageClient storageClient;

    public FileOperations(MetadataServer metadataServer, StorageClient storageClient) {
        this.metadataServer = metadataServer;
        this.storageClient = storageClient;
    }

    public void writeFile(String path, byte[] data, String owner) {
        String fileId = metadataServer.createFile(path, owner, data.length);
        FileMetadata metadata = metadataServer.getFileMetadata(fileId);
        
        // Write data to storage nodes
        for (BlockLocation block : metadata.getBlockLocations()) {
            int offset = (int) block.getOffset();
            int size = (int) block.getSize();
            byte[] blockData = Arrays.copyOfRange(data, offset, offset + size);
            storageClient.writeBlock(block.getStorageNodeId(), block.getBlockId(), blockData);
        }
    }

    public byte[] readFile(String fileId) {
        FileMetadata metadata = metadataServer.getFileMetadata(fileId);
        if (metadata == null) {
            throw new IllegalArgumentException("File not found: " + fileId);
        }

        byte[] data = new byte[(int) metadata.getFileSize()];
        
        for (BlockLocation block : metadata.getBlockLocations()) {
            byte[] blockData = storageClient.readBlock(block.getStorageNodeId(), block.getBlockId());
            int offset = (int) block.getOffset();
            System.arraycopy(blockData, 0, data, offset, blockData.length);
        }
        
        return data;
    }

    public void deleteFile(String fileId) {
        metadataServer.deleteFile(fileId);
    }
}
```

### Consistency Model

```java
public class FileConsistencyManager {
    private final MetadataServer metadataServer;
    private final WriteAheadLog writeAheadLog;

    public FileConsistencyManager(MetadataServer metadataServer, WriteAheadLog writeAheadLog) {
        this.metadataServer = metadataServer;
        this.writeAheadLog = writeAheadLog;
    }

    public void createFileWithConsistency(String path, String owner, long fileSize) {
        // Write to WAL first
        writeAheadLog.append(new FileOperation("CREATE", path, owner, fileSize));
        
        try {
            String fileId = metadataServer.createFile(path, owner, fileSize);
            // Operation successful, commit WAL
            writeAheadLog.commit();
        } catch (Exception e) {
            // Rollback
            writeAheadLog.rollback();
            throw e;
        }
    }
}

class FileOperation {
    final String operation;
    final String path;
    final String owner;
    final long fileSize;

    FileOperation(String operation, String path, String owner, long fileSize) {
        this.operation = operation;
        this.path = path;
        this.owner = owner;
        this.fileSize = fileSize;
    }
}
```

---

## Interviewer Insight
Interviewers want to see that you understand distributed file system design: metadata management, block allocation, consistency, and replication. Strong candidates discuss write-ahead logs, consistency models, and storage node selection strategies.

## Follow-up Questions
1. How do you handle file consistency across multiple nodes?
2. What is the difference between strong and eventual consistency in file systems?
3. How do you implement file versioning?

## 📚 Related Topics
- [Database Sharding](../../bonus/06-database-patterns/08-database-sharding.md)
- [System Design Principles](../12-performance-optimization/README.md)
- [Distributed Systems](../04-multithreading-concurrency/README.md)

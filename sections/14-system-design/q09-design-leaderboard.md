# Q9: Design a Leaderboard

## Question
Design and implement a Leaderboard system in Java. Support top-K queries, real-time score updates, concurrent access, and efficient ranking operations.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Add or update a player's score
- Get a player's rank and score
- Retrieve the top-K players
- Retrieve players around a given player's rank (neighborhood query)
- Remove a player from the leaderboard
- Support multiple leaderboards (e.g., daily, weekly, all-time)

**Non-Functional Requirements:**
- Thread-safe for concurrent score updates and reads
- O(log N) or better for updates and rank queries
- Efficient top-K retrieval
- Handle millions of players

### API Design

```java
public interface Leaderboard {
    void addScore(String playerId, long score);
    void updateScore(String playerId, long newScore);
    void incrementScore(String playerId, long delta);
    PlayerRank getRank(String playerId);
    List<PlayerRank> getTopK(int k);
    List<PlayerRank> getAroundPlayer(String playerId, int count);
    boolean removePlayer(String playerId);
    int size();
}

public class PlayerRank {
    private final String playerId;
    private final long score;
    private final int rank; // 1-based

    public PlayerRank(String playerId, long score, int rank) {
        this.playerId = playerId;
        this.score = score;
        this.rank = rank;
    }

    public String getPlayerId() { return playerId; }
    public long getScore() { return score; }
    public int getRank() { return rank; }

    @Override
    public String toString() {
        return "#" + rank + " " + playerId + " (score: " + score + ")";
    }
}
```

### Implementation 1: TreeMap-Based Leaderboard

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class TreeMapLeaderboard implements Leaderboard {

    // ScoreEntry: sorted by score descending, then by playerId for uniqueness
    private static class ScoreEntry implements Comparable<ScoreEntry> {
        final String playerId;
        final long score;

        ScoreEntry(String playerId, long score) {
            this.playerId = playerId;
            this.score = score;
        }

        @Override
        public int compareTo(ScoreEntry other) {
            // Higher score first
            int cmp = Long.compare(other.score, this.score);
            if (cmp != 0) return cmp;
            // Tie-break by playerId (alphabetical)
            return this.playerId.compareTo(other.playerId);
        }

        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (!(o instanceof ScoreEntry)) return false;
            ScoreEntry that = (ScoreEntry) o;
            return score == that.score && playerId.equals(that.playerId);
        }

        @Override
        public int hashCode() {
            return Objects.hash(playerId, score);
        }
    }

    private final TreeSet<ScoreEntry> sortedScores = new TreeSet<>();
    private final ConcurrentHashMap<String, Long> playerScores = new ConcurrentHashMap<>();
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    @Override
    public void addScore(String playerId, long score) {
        lock.writeLock().lock();
        try {
            Long existing = playerScores.get(playerId);
            if (existing != null) {
                sortedScores.remove(new ScoreEntry(playerId, existing));
            }
            playerScores.put(playerId, score);
            sortedScores.add(new ScoreEntry(playerId, score));
        } finally {
            lock.writeLock().unlock();
        }
    }

    @Override
    public void updateScore(String playerId, long newScore) {
        addScore(playerId, newScore); // same operation
    }

    @Override
    public void incrementScore(String playerId, long delta) {
        lock.writeLock().lock();
        try {
            long current = playerScores.getOrDefault(playerId, 0L);
            long newScore = current + delta;

            if (current != 0 || playerScores.containsKey(playerId)) {
                sortedScores.remove(new ScoreEntry(playerId, current));
            }
            playerScores.put(playerId, newScore);
            sortedScores.add(new ScoreEntry(playerId, newScore));
        } finally {
            lock.writeLock().unlock();
        }
    }

    @Override
    public PlayerRank getRank(String playerId) {
        lock.readLock().lock();
        try {
            Long score = playerScores.get(playerId);
            if (score == null) return null;

            ScoreEntry entry = new ScoreEntry(playerId, score);
            int rank = sortedScores.headSet(entry).size() + 1;
            return new PlayerRank(playerId, score, rank);
        } finally {
            lock.readLock().unlock();
        }
    }

    @Override
    public List<PlayerRank> getTopK(int k) {
        lock.readLock().lock();
        try {
            List<PlayerRank> result = new ArrayList<>();
            int rank = 1;
            for (ScoreEntry entry : sortedScores) {
                if (rank > k) break;
                result.add(new PlayerRank(entry.playerId, entry.score, rank));
                rank++;
            }
            return result;
        } finally {
            lock.readLock().unlock();
        }
    }

    @Override
    public List<PlayerRank> getAroundPlayer(String playerId, int count) {
        lock.readLock().lock();
        try {
            Long score = playerScores.get(playerId);
            if (score == null) return Collections.emptyList();

            ScoreEntry target = new ScoreEntry(playerId, score);
            int targetRank = sortedScores.headSet(target).size() + 1;

            // Get players above and below
            int startRank = Math.max(1, targetRank - count);
            int endRank = targetRank + count;

            List<PlayerRank> result = new ArrayList<>();
            int rank = 1;
            for (ScoreEntry entry : sortedScores) {
                if (rank > endRank) break;
                if (rank >= startRank) {
                    result.add(new PlayerRank(entry.playerId, entry.score, rank));
                }
                rank++;
            }
            return result;
        } finally {
            lock.readLock().unlock();
        }
    }

    @Override
    public boolean removePlayer(String playerId) {
        lock.writeLock().lock();
        try {
            Long score = playerScores.remove(playerId);
            if (score != null) {
                sortedScores.remove(new ScoreEntry(playerId, score));
                return true;
            }
            return false;
        } finally {
            lock.writeLock().unlock();
        }
    }

    @Override
    public int size() {
        lock.readLock().lock();
        try {
            return playerScores.size();
        } finally {
            lock.readLock().unlock();
        }
    }
}
```

### Implementation 2: PriorityQueue for Top-K Only

```java
import java.util.*;

public class TopKLeaderboard {

    private final int maxSize;
    private final PriorityQueue<PlayerRank> minHeap;
    private final Map<String, Long> playerScores = new HashMap<>();

    public TopKLeaderboard(int maxSize) {
        this.maxSize = maxSize;
        // Min-heap: lowest score at top (for easy eviction)
        this.minHeap = new PriorityQueue<>(Comparator.comparingLong(PlayerRank::getScore));
    }

    public synchronized void addScore(String playerId, long score) {
        playerScores.put(playerId, score);

        // Rebuild heap (PriorityQueue doesn't support efficient update)
        rebuildHeap();
    }

    public synchronized List<PlayerRank> getTopK() {
        List<PlayerRank> result = new ArrayList<>(minHeap);
        result.sort(Comparator.comparingLong(PlayerRank::getScore).reversed());

        // Assign ranks
        List<PlayerRank> ranked = new ArrayList<>();
        for (int i = 0; i < result.size(); i++) {
            PlayerRank pr = result.get(i);
            ranked.add(new PlayerRank(pr.getPlayerId(), pr.getScore(), i + 1));
        }
        return ranked;
    }

    private void rebuildHeap() {
        minHeap.clear();
        for (Map.Entry<String, Long> entry : playerScores.entrySet()) {
            PlayerRank pr = new PlayerRank(entry.getKey(), entry.getValue(), 0);
            if (minHeap.size() < maxSize) {
                minHeap.offer(pr);
            } else if (pr.getScore() > minHeap.peek().getScore()) {
                minHeap.poll();
                minHeap.offer(pr);
            }
        }
    }
}
```

### Implementation 3: ConcurrentSkipListSet (Lock-Free Alternative)

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentSkipListSet;
import java.util.concurrent.atomic.AtomicLong;
import java.util.*;

public class SkipListLeaderboard implements Leaderboard {

    private static class ScoreEntry implements Comparable<ScoreEntry> {
        final String playerId;
        final long score;

        ScoreEntry(String playerId, long score) {
            this.playerId = playerId;
            this.score = score;
        }

        @Override
        public int compareTo(ScoreEntry o) {
            int cmp = Long.compare(o.score, this.score); // descending
            return cmp != 0 ? cmp : this.playerId.compareTo(o.playerId);
        }

        @Override
        public boolean equals(Object o) {
            if (!(o instanceof ScoreEntry)) return false;
            ScoreEntry that = (ScoreEntry) o;
            return score == that.score && playerId.equals(that.playerId);
        }

        @Override
        public int hashCode() { return Objects.hash(playerId, score); }
    }

    // ConcurrentSkipListSet — lock-free sorted set
    private final ConcurrentSkipListSet<ScoreEntry> board = new ConcurrentSkipListSet<>();
    private final ConcurrentHashMap<String, Long> scores = new ConcurrentHashMap<>();

    @Override
    public void addScore(String playerId, long score) {
        scores.compute(playerId, (id, oldScore) -> {
            if (oldScore != null) {
                board.remove(new ScoreEntry(id, oldScore));
            }
            board.add(new ScoreEntry(id, score));
            return score;
        });
    }

    @Override
    public void updateScore(String playerId, long newScore) {
        addScore(playerId, newScore);
    }

    @Override
    public void incrementScore(String playerId, long delta) {
        scores.compute(playerId, (id, oldScore) -> {
            long current = oldScore != null ? oldScore : 0;
            if (oldScore != null) {
                board.remove(new ScoreEntry(id, current));
            }
            long newScore = current + delta;
            board.add(new ScoreEntry(id, newScore));
            return newScore;
        });
    }

    @Override
    public PlayerRank getRank(String playerId) {
        Long score = scores.get(playerId);
        if (score == null) return null;
        ScoreEntry target = new ScoreEntry(playerId, score);
        int rank = board.headSet(target).size() + 1;
        return new PlayerRank(playerId, score, rank);
    }

    @Override
    public List<PlayerRank> getTopK(int k) {
        List<PlayerRank> result = new ArrayList<>();
        int rank = 1;
        for (ScoreEntry entry : board) {
            if (rank > k) break;
            result.add(new PlayerRank(entry.playerId, entry.score, rank++));
        }
        return result;
    }

    @Override
    public List<PlayerRank> getAroundPlayer(String playerId, int count) {
        // Delegate to full iteration approach (similar to TreeMap version)
        Long score = scores.get(playerId);
        if (score == null) return Collections.emptyList();

        ScoreEntry target = new ScoreEntry(playerId, score);
        int targetRank = board.headSet(target).size() + 1;
        int start = Math.max(1, targetRank - count);
        int end = targetRank + count;

        List<PlayerRank> result = new ArrayList<>();
        int rank = 1;
        for (ScoreEntry entry : board) {
            if (rank > end) break;
            if (rank >= start) {
                result.add(new PlayerRank(entry.playerId, entry.score, rank));
            }
            rank++;
        }
        return result;
    }

    @Override
    public boolean removePlayer(String playerId) {
        Long score = scores.remove(playerId);
        if (score != null) {
            return board.remove(new ScoreEntry(playerId, score));
        }
        return false;
    }

    @Override
    public int size() { return scores.size(); }
}
```

### Approach Comparison

| Approach | Update | Top-K | getRank | Thread Safety |
|----------|--------|-------|---------|---------------|
| TreeMap/TreeSet | O(log N) | O(K) | O(N) headSet | ReadWriteLock |
| PriorityQueue | O(N) rebuild | O(K log K) | Not supported | synchronized |
| ConcurrentSkipListSet | O(log N) | O(K) | O(N) headSet | Lock-free |

### Concurrency Considerations

| Concern | Solution |
|---------|----------|
| Score update atomicity | `compute()` on `ConcurrentHashMap` for atomic read-modify-write |
| Sorted set consistency | Write lock (TreeMap) or `ConcurrentSkipListSet` (lock-free) |
| Rank calculation | `headSet().size()` — O(N) but consistent under read lock |
| Top-K snapshot | Iterate first K elements under read lock |
| Remove + re-add race | `compute()` ensures remove-old and add-new are atomic |

### Performance Analysis

| Operation | TreeMap | SkipList | Notes |
|-----------|---------|----------|-------|
| `addScore` | O(log N) | O(log N) | Both balanced structures |
| `getTopK` | O(K) | O(K) | Iterate first K elements |
| `getRank` | O(N) | O(N) | `headSet().size()` is linear |
| `removePlayer` | O(log N) | O(log N) | Standard remove |

**Optimization for O(log N) rank:** Use an augmented BST (order-statistic tree) or a Fenwick tree (BIT) indexed by score ranges.

### Testing Strategy

```java
public class LeaderboardTest {

    @Test
    void testAddAndGetTopK() {
        TreeMapLeaderboard lb = new TreeMapLeaderboard();
        lb.addScore("alice", 100);
        lb.addScore("bob", 250);
        lb.addScore("charlie", 175);

        List<PlayerRank> top = lb.getTopK(2);
        assertEquals(2, top.size());
        assertEquals("bob", top.get(0).getPlayerId());
        assertEquals("charlie", top.get(1).getPlayerId());
    }

    @Test
    void testGetRank() {
        TreeMapLeaderboard lb = new TreeMapLeaderboard();
        lb.addScore("a", 50);
        lb.addScore("b", 150);
        lb.addScore("c", 100);

        assertEquals(1, lb.getRank("b").getRank());
        assertEquals(2, lb.getRank("c").getRank());
        assertEquals(3, lb.getRank("a").getRank());
    }

    @Test
    void testIncrementScore() {
        TreeMapLeaderboard lb = new TreeMapLeaderboard();
        lb.addScore("player1", 10);
        lb.incrementScore("player1", 5);

        assertEquals(15, lb.getRank("player1").getScore());
    }

    @Test
    void testAroundPlayer() {
        TreeMapLeaderboard lb = new TreeMapLeaderboard();
        for (int i = 1; i <= 10; i++) {
            lb.addScore("p" + i, i * 10);
        }

        List<PlayerRank> around = lb.getAroundPlayer("p5", 2);
        assertTrue(around.size() >= 3); // p4, p5, p6 at minimum
    }

    @Test
    void testConcurrentUpdates() throws Exception {
        SkipListLeaderboard lb = new SkipListLeaderboard();
        ExecutorService exec = Executors.newFixedThreadPool(8);

        for (int i = 0; i < 1000; i++) {
            final String player = "p" + (i % 100);
            final long score = i;
            exec.submit(() -> lb.addScore(player, score));
        }

        exec.shutdown();
        exec.awaitTermination(10, TimeUnit.SECONDS);
        assertEquals(100, lb.size());
    }

    @Test
    void testRemovePlayer() {
        TreeMapLeaderboard lb = new TreeMapLeaderboard();
        lb.addScore("alice", 100);
        lb.addScore("bob", 200);

        assertTrue(lb.removePlayer("alice"));
        assertNull(lb.getRank("alice"));
        assertEquals(1, lb.size());
    }
}
```

---

## Interviewer Insight
Interviewers want to see your choice of data structure and its trade-offs. `TreeMap`/`TreeSet` is the standard answer — discuss why `HashMap` alone isn't enough (no ordering). The key insight is that `getRank()` is O(N) with TreeSet but can be O(log N) with an order-statistic tree. For production systems, discuss Redis sorted sets (`ZADD`, `ZRANK`, `ZRANGE`) which provide O(log N) for all operations.

## Follow-up Questions
1. How would you achieve O(log N) rank queries? (Hint: order-statistic tree or Fenwick tree)
2. How would you implement this with Redis sorted sets in a distributed environment?
3. How would you handle tied scores (same score, different rank)?

## Related Topics
- [Collections Framework](../03-collections-framework/README.md)
- [Multithreading & Concurrency](../04-multithreading-concurrency/README.md)

# Q9: Design a Leaderboard

## 📋 Question
Design a real-time leaderboard system for a game application. Discuss the data structures, ranking algorithms, and performance optimization.

## Difficulty: Medium

---

## Solution

### Requirements

**Functional Requirements:**
- Track player scores
- Real-time ranking updates
- Support top N and around me queries
- Handle score updates efficiently
- Support multiple leaderboards

**Non-Functional Requirements:**
- Low latency (< 100ms for updates)
- High throughput (10K+ updates/sec)
- Real-time consistency
- Scalable to millions of players

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Game Clients                          │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│                 Load Balancer                            │
└──────────────────────┬──────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Game Server  │ │  Game Server  │ │  Game Server  │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │               │               │
       └───────────────┼───────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│              Leaderboard Service                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Sorted Set (Redis ZSET)                         │  │
│  │  ┌────────────────────────────────────────────┐    │  │
│  │  │ Player1: 10000                               │    │  │
│  │  │ Player2: 9500                                │    │  │
│  │  │ Player3: 9200                                │    │  │
│  │  └────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│              Redis Cluster                            │
└─────────────────────────────────────────────────────────┘
```

### Data Structure

```java
public class PlayerScore {
    private final String playerId;
    private final String playerName;
    private long score;
    private final long timestamp;

    public PlayerScore(String playerId, String playerName, long score) {
        this.playerId = playerId;
        this.playerName = playerName;
        this.score = score;
        this.timestamp = System.currentTimeMillis();
    }

    // Getters
    public String getPlayerId() { return playerId; }
    public String getPlayerName() { return playerName; }
    public long getScore() { return score; }
    public long getTimestamp() { return timestamp; }

    public void setScore(long score) {
        this.score = score;
        this.timestamp = System.currentTimeMillis();
    }
}
```

### Leaderboard Service

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import java.util.*;

public class LeaderboardService {
    private final JedisPool jedisPool;
    private final String leaderboardKey;
    private final int topN;

    public LeaderboardService(JedisPool jedisPool, String leaderboardKey, int topN) {
        this.jedisPool = jedisPool;
        this_leaderboardKey = leaderboardKey;
        this.topN = topN;
    }

    public void updateScore(String playerId, String playerName, long score) {
        try (Jedis jedis = jedisPool.getResource()) {
            // Add or update player score
            jedis.zadd(leaderboardKey, score, playerId);
            
            // Store player name separately
            jedis.hset("player:" + playerId, "name", playerName);
            jedis.hset("player:" + playerId, "score", String.valueOf(score));
        }
    }

    public List<PlayerScore> getTopPlayers() {
        try (Jedis jedis = jedisPool.getResource()) {
            // Get top N players by score (descending)
            Set<redis.clients.jedis.Tuple> topScores = jedis.zrevrangeWithScores(leaderboardKey, 0, topN - 1);
            
            List<PlayerScore> players = new ArrayList<>();
            for (redis.clients.jedis.Tuple tuple : topScores) {
                String playerId = tuple.getElement();
                long score = (long) tuple.getScore();
                String playerName = jedis.hget("player:" + playerId, "name");
                players.add(new PlayerScore(playerId, playerName, score));
            }
            return players;
        }
    }

    public List<PlayerScore> getAroundMe(String playerId, int range) {
        try (Jedis jedis = jedisPool.getResource()) {
            // Get player's rank
            Long rank = jedis.zrevrank(leaderboardKey, playerId);
            if (rank == null) {
                return Collections.emptyList();
            }

            // Get players around this rank
            int start = Math.max(0, (int) (rank - range));
            int end = (int) (rank + range);
            
            Set<redis.clients.jedis.Tuple> aroundScores = jedis.zrevrangeWithScores(leaderboardKey, start, end);
            
            List<PlayerScore> players = new ArrayList<>();
            for (redis.clients.jedis.Tuple tuple : aroundScores) {
                String pid = tuple.getElement();
                long score = (long) tuple.getScore();
                String playerName = jedis.hget("player:" + pid, "name");
                players.add(new PlayerScore(pid, playerName, score));
            }
            return players;
        }
    }

    public int getPlayerRank(String playerId) {
        try (Jedis jedis = jedisPool.getResource()) {
            Long rank = jedis.zrevrank(leaderboardKey, playerId);
            return rank != null ? rank.intValue() + 1 : -1;
        }
    }

    public long getPlayerScore(String playerId) {
        try (Jedis jedis = jedisPool.getResource()) {
            Double score = jedis.zscore(leaderboardKey, playerId);
            return score != null ? score.longValue() : 0;
        }
    }
}
```

### Batch Score Updates

```java
import java.util.*;
import java.util.concurrent.*;

public class BatchScoreUpdater {
    private final LeaderboardService leaderboardService;
    private final BlockingQueue<ScoreUpdate> updateQueue;
    private final ExecutorService executor;

    public BatchScoreUpdater(LeaderboardService leaderboardService) {
        this.leaderboardService = leaderboardService;
        this.updateQueue = new LinkedBlockingQueue<>(10000);
        this.executor = Executors.newSingleThreadExecutor();
        this.executor.submit(this::processUpdates);
    }

    public void queueUpdate(String playerId, String playerName, long score) {
        updateQueue.offer(new ScoreUpdate(playerId, playerName, score));
    }

    private void processUpdates() {
        List<ScoreUpdate> batch = new ArrayList<>(100);
        
        while (true) {
            try {
                // Wait for updates or timeout
                ScoreUpdate update = updateQueue.poll(100, TimeUnit.MILLISECONDS);
                if (update != null) {
                    batch.add(update);
                    // Drain queue
                    updateQueue.drainTo(batch, 100);
                }
                
                if (!batch.isEmpty()) {
                    // Process batch
                    for (ScoreUpdate su : batch) {
                        leaderboardService.updateScore(su.playerId, su.playerName, su.score);
                    }
                    batch.clear();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }

    static class ScoreUpdate {
        final String playerId;
        final String playerName;
        final long score;

        ScoreUpdate(String playerId, String playerName, long score) {
            this.playerId = playerId;
            this.playerName = playerName;
            this.score = score;
        }
    }
}
```

### Time-based Leaderboard

```java
import java.util.*;

public class TimeBasedLeaderboard {
    private final Map<String, LeaderboardService> timeBasedLeaderboards;
    private final List<String> timeWindows;

    public TimeBasedLeaderboard(JedisPool jedisPool, List<String> timeWindows) {
        this.timeBasedLeaderboards = new HashMap<>();
        this.timeWindows = timeWindows;

        for (String window : timeWindows) {
            String key = "leaderboard:" + window;
            timeBasedLeaderboards.put(window, new LeaderboardService(jedisPool, key, 100));
        }
    }

    public void updateScore(String playerId, String playerName, long score, String timeWindow) {
        LeaderboardService leaderboard = timeBasedLeaderboards.get(timeWindow);
        if (leaderboard != null) {
            leaderboard.updateScore(playerId, playerName, score);
        }
    }

    public List<PlayerScore> getTopPlayers(String timeWindow) {
        LeaderboardService leaderboard = timeBasedLeaderboards.get(timeWindow);
        if (leaderboard != null) {
            return leaderboard.getTopPlayers();
        }
        return Collections.emptyList();
    }
}
```

### Performance Optimization

| Optimization | Description | Impact |
|--------------|-------------|--------|
| Batch updates | Process multiple updates together | Reduces Redis calls |
| Pipeline updates | Use Redis pipelining | Reduces network round-trips |
| Local cache | Cache top players in memory | Reduces Redis load |
| Asynchronous updates | Non-blocking score updates | Better throughput |
| Sharding | Shard by game region | Horizontal scalability |

---

## Interviewer Insight
Interviewers want to see that you understand leaderboard design: data structures (sorted sets), ranking algorithms, and performance optimization. Strong candidates discuss Redis ZSET, batch processing, and time-based leaderboards.

## Follow-up Questions
1. How do you handle ties in the leaderboard?
2. What is the difference between global and time-based leaderboards?
3. How do you optimize leaderboard performance for high write throughput?

## 📚 Related Topics
- [Cache Design](./02-design-cache.md)
- [Distributed Counter](./08-design-distributed-counter.md)
- [System Design Principles](../12-performance-optimization/README.md)

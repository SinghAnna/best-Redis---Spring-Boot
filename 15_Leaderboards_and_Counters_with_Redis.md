# Part 15: Leaderboards and Counters with Redis — Complete Deep Dive

> Redis Sorted Set (`ZSET`) real-time leaderboard, ranking, trending posts aur score-based systems ke liye best data structure hai. Counters ke liye Redis `INCR` atomic operations provide karta hai.

---

## 1. Sorted Set Kya Hai?

Sorted Set ek collection hai jisme:

```text
Member -> Unique
Score  -> Numeric ranking value
```

Example:

```text
leaderboard
  ├── user:102 -> 1500
  ├── user:101 -> 950
  └── user:103 -> 700
```

Redis `ZSET` members ko score ke according sorted order me maintain karta hai. Same score hone par members lexicographically order hote hain. [219]

### Important

```text
Member unique hota hai.
Score duplicate ho sakta hai.
```

---

## 2. Leaderboard Redis Keys

### Daily Leaderboard

```text
leaderboard:daily:2026-10-19
```

### Weekly Leaderboard

```text
leaderboard:weekly:2026-W42
```

### Monthly Leaderboard

```text
leaderboard:monthly:2026-10
```

### Game Leaderboard

```text
game:pubg:leaderboard:global
game:pubg:leaderboard:india
```

### Trending Posts

```text
trending:posts:2026-10-19
```

---

## 3. Core Sorted Set Commands

### `ZADD`

Member add karta hai ya existing member ka score update karta hai.

```bash
ZADD leaderboard 950 user:101
ZADD leaderboard 1500 user:102
ZADD leaderboard 700 user:103
```

`ZADD` sorted set me member add karta hai ya uska score update karta hai. [219]

### `ZINCRBY`

Existing score ko increment karta hai.

```bash
ZINCRBY leaderboard 50 user:101
```

`ZINCRBY` member ka score increment karta hai; member exist na kare to previous score 0 maan kar add karta hai. [220]

### `ZSCORE`

Member ka current score return karta hai.

```bash
ZSCORE leaderboard user:101
```

### `ZRANK`

Ascending rank return karta hai.

```bash
ZRANK leaderboard user:101
```

### `ZREVRANK`

Descending rank return karta hai.

```bash
ZREVRANK leaderboard user:101
```

### `ZREVRANGE`

Top N members return karta hai.

```bash
ZREVRANGE leaderboard 0 9 WITHSCORES
```

### `ZREMRANGEBYRANK`

Rank range se members remove karta hai.

```bash
ZREMRANGEBYRANK leaderboard 0 9
```

### `ZCOUNT`

Score range me members count karta hai.

```bash
ZCOUNT leaderboard 800 1200
```

---

## 4. Spring Data Redis ZSet Operations

| Redis Command | Spring Data Redis Method |
|---|---|
| `ZADD` | `opsForZSet().add()` |
| `ZINCRBY` | `opsForZSet().incrementScore()` |
| `ZSCORE` | `opsForZSet().score()` |
| `ZRANK` | `opsForZSet().rank()` |
| `ZREVRANK` | `opsForZSet().reverseRank()` |
| `ZREVRANGE` | `opsForZSet().reverseRange()` |
| `ZCOUNT` | `opsForZSet().count()` |
| `ZREM` | `opsForZSet().remove()` |

Spring Data Redis `ZSetOperations` sorted set operations ke liye `rank`, `reverseRank`, score-based range aur member operations provide karta hai. [218]

---

## 5. Leaderboard Service

```java
package com.example.leaderboard;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.LocalDate;
import java.util.Set;

@Service
@RequiredArgsConstructor
public class LeaderboardService {

    private final StringRedisTemplate redisTemplate;

    private static final String DAILY_LEADERBOARD_PREFIX =
            "game-service:leaderboard:daily:";

    public void addScore(Long userId, double score) {
        String key = dailyLeaderboardKey();

        redisTemplate.opsForZSet().incrementScore(
                key,
                userId.toString(),
                score
        );

        redisTemplate.expire(key, Duration.ofDays(2));
    }

    public Double getScore(Long userId) {
        String key = dailyLeaderboardKey();

        return redisTemplate.opsForZSet().score(
                key,
                userId.toString()
        );
    }

    public Long getRank(Long userId) {
        String key = dailyLeaderboardKey();

        return redisTemplate.opsForZSet().reverseRank(
                key,
                userId.toString()
        );
    }

    public Set<String> getTopPlayers(int limit) {
        String key = dailyLeaderboardKey();

        return redisTemplate.opsForZSet().reverseRange(
                key,
                0,
                limit - 1
        );
    }

    private String dailyLeaderboardKey() {
        return DAILY_LEADERBOARD_PREFIX + LocalDate.now();
    }
}
```

---

## 6. Leaderboard DTO

```java
package com.example.leaderboard;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class LeaderboardEntry {

    private Long userId;
    private String username;
    private Double score;
    private Long rank;
}
```

---

## 7. Leaderboard API

```java
package com.example.controller;

import com.example.leaderboard.LeaderboardEntry;
import com.example.leaderboard.LeaderboardService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Set;

@RestController
@RequestMapping("/api/leaderboard")
@RequiredArgsConstructor
public class LeaderboardController {

    private final LeaderboardService leaderboardService;

    @PostMapping("/users/{userId}/score")
    public ResponseEntity<String> addScore(
            @PathVariable Long userId,
            @RequestParam double score
    ) {
        leaderboardService.addScore(userId, score);
        return ResponseEntity.ok("Score added");
    }

    @GetMapping("/users/{userId}/rank")
    public ResponseEntity<Long> getRank(@PathVariable Long userId) {
        Long rank = leaderboardService.getRank(userId);

        if (rank == null) {
            return ResponseEntity.notFound().build();
        }

        return ResponseEntity.ok(rank + 1);
    }

    @GetMapping("/top")
    public ResponseEntity<List<LeaderboardEntry>> getTopPlayers(
            @RequestParam(defaultValue = "10") int limit
    ) {
        Set<String> userIds = leaderboardService.getTopPlayers(limit);

        List<LeaderboardEntry> entries = userIds.stream()
                .map(id -> LeaderboardEntry.builder()
                        .userId(Long.parseLong(id))
                        .score(leaderboardService.getScore(Long.parseLong(id)))
                        .rank(leaderboardService.getRank(Long.parseLong(id)) + 1)
                        .build()
                )
                .toList();

        return ResponseEntity.ok(entries);
    }
}
```

---

## 8. Pagination in Leaderboard

### Page 1

```text
Rank 1 to 10
```

```bash
ZREVRANGE leaderboard 0 9
```

### Page 2

```text
Rank 11 to 20
```

```bash
ZREVRANGE leaderboard 10 19
```

### Formula

```text
start = (page - 1) x pageSize
end = start + pageSize - 1
```

### Java Implementation

```java
public Set<String> getLeaderboardPage(int page, int pageSize) {
    int start = (page - 1) * pageSize;
    int end = start + pageSize - 1;

    return redisTemplate.opsForZSet().reverseRange(
            dailyLeaderboardKey(),
            start,
            end
    );
}
```

---

## 9. Tie-Breaking Problem

Agar do users ka same score ho:

```text
user:101 -> 1000
user:102 -> 1000
```

Redis default me same score wale members ko lexicographically order karta hai. [219]

### Business Requirement

```text
Jisne score pehle achieve kiya
wo upar rank ho.
```

### Solution: Composite Score

```text
Final Score = Primary Score + Time-Based Tie Breaker
```

Example:

```text
Primary score = 1000
Earlier timestamp = Higher tie-breaker value

Final score = 1000.999999
```

### Java Example

```java
public double calculateScore(long primaryScore, long achievedAtMillis) {
    long maxTimestamp = 9999999999999L;

    double tieBreaker =
            (maxTimestamp - achievedAtMillis) / 10000000000000.0;

    return primaryScore + tieBreaker;
}
```

### Redis Write

```java
double finalScore = calculateScore(
        primaryScore,
        System.currentTimeMillis()
);

redisTemplate.opsForZSet().add(
        leaderboardKey,
        userId.toString(),
        finalScore
);
```

Composite score me primary score integer part hota hai aur timestamp-derived fractional part tie-breaking karta hai. [223][224]

---

## 10. Like Counter

### Redis Command

```bash
INCR post:201:likes
```

### Java Service

```java
@Service
@RequiredArgsConstructor
public class LikeCounterService {

    private final StringRedisTemplate redisTemplate;

    public Long likePost(Long postId) {
        return redisTemplate.opsForValue().increment(
                "post-service:post:" + postId + ":likes"
        );
    }

    public Long unlikePost(Long postId) {
        return redisTemplate.opsForValue().decrement(
                "post-service:post:" + postId + ":likes"
        );
    }

    public Long getLikeCount(Long postId) {
        String value = redisTemplate.opsForValue().get(
                "post-service:post:" + postId + ":likes"
        );

        return value == null ? 0L : Long.parseLong(value);
    }
}
```

---

## 11. View Counter

### Redis Command

```bash
INCR video:301:views
```

### Java Service

```java
@Service
@RequiredArgsConstructor
public class ViewCounterService {

    private final StringRedisTemplate redisTemplate;

    public Long recordView(Long videoId) {
        String key = "video-service:video:" + videoId + ":views";

        Long views = redisTemplate.opsForValue().increment(key);

        redisTemplate.expire(key, Duration.ofDays(30));

        return views;
    }
}
```

---

## 12. Trending Posts Leaderboard

### Score Formula

```text
Trending Score =
(Likes x 3)
+ (Comments x 5)
+ (Shares x 8)
+ (Recent Activity Weight)
```

### Java Example

```java
@Service
@RequiredArgsConstructor
public class TrendingPostService {

    private final StringRedisTemplate redisTemplate;

    public void recordActivity(
            Long postId,
            long likes,
            long comments,
            long shares
    ) {
        double score =
                (likes * 3.0)
                        + (comments * 5.0)
                        + (shares * 8.0);

        redisTemplate.opsForZSet().incrementScore(
                "post-service:trending:posts",
                postId.toString(),
                score
        );

        redisTemplate.expire(
                "post-service:trending:posts",
                Duration.ofDays(1)
        );
    }

    public Set<String> getTrendingPosts(int limit) {
        return redisTemplate.opsForZSet().reverseRange(
                "post-service:trending:posts",
                0,
                limit - 1
        );
    }
}
```

---

## 13. Leaderboard Consistency

### Problem

Redis leaderboard fast hai, lekin agar Redis down ho jaye to score data loss ho sakta hai.

### Solution

```text
Redis = Real-time leaderboard
Database = Source of truth
Kafka = Score events
```

### Architecture

```text
Game Event
   |
   v
Spring Boot
   |
   +--> Redis ZSET update
   |
   +--> Kafka score event
   |
   v
Consumer -> PostgreSQL/MySQL update
```

### Recovery

```text
Redis restart hone par
DB se top players reload karo
aur ZSET rebuild karo.
```

---

## 14. Large Leaderboard Optimization

| Problem | Solution |
|---|---|
| Millions of users | Only top N in Redis, full ranking in DB |
| Frequent rank query | Cache user rank separately |
| Heavy write load | Kafka buffering |
| Same score ties | Composite score |
| Old leaderboard data | TTL / periodic cleanup |
| Global + regional leaderboard | Separate ZSET keys |

### Hybrid Design

```text
Redis:
- Top 10,000 players
- Real-time rank for active users

Database:
- Complete historical scores
- Accurate final rankings
```

---

## 15. Production Checklist

- [ ] Leaderboard key me time period hai.
- [ ] TTL set hai.
- [ ] Score update atomic hai.
- [ ] Tie-breaking rule defined hai.
- [ ] Pagination implemented hai.
- [ ] DB source of truth hai.
- [ ] Kafka event se durable score persistence hai.
- [ ] Redis failure recovery plan hai.
- [ ] Large leaderboard ke liye top-N strategy hai.
- [ ] Metrics monitor ho rahe hain.

---

## 16. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| `GET`/`SET` se leaderboard banana | Race condition | Sorted Set use karo |
| Score update non-atomic | Wrong ranking | `ZINCRBY` / `incrementScore` |
| No tie-breaking | Same score users ka order unclear | Composite score |
| No TTL | Old leaderboard memory me rah jayegi | Daily/weekly TTL |
| Redis ko permanent store samajhna | Data loss risk | DB source of truth |
| Full leaderboard Redis me | Memory pressure | Top-N Redis strategy |

---

## 17. Interview Questions

| Question | Answer |
|---|---|
| Leaderboard ke liye Redis kaunsa data type use hota hai? | Sorted Set (`ZSET`). |
| `ZADD` aur `ZINCRBY` me difference? | `ZADD` score set/update karta hai; `ZINCRBY` existing score increment karta hai. |
| User rank kaise nikalte hain? | `ZREVRANK` se descending rank. |
| Top 10 players kaise nikalte hain? | `ZREVRANGE leaderboard 0 9`. |
| Tie-breaking kaise handle karte hain? | Composite score: primary score + timestamp-based fractional score. |
| Like counter ke liye kaunsa command? | `INCR`. |
| Redis leaderboard durable kaise banate hain? | Score events Kafka me publish karke DB me persist karte hain. |
| Large leaderboard kaise scale karte hain? | Redis me top-N, DB me complete ranking. |

---

## 18. Final Mental Model

```text
ZADD       -> Set score
ZINCRBY    -> Increment score
ZSCORE     -> Get score
ZREVRANK   -> Get leaderboard rank
ZREVRANGE  -> Get top N
INCR       -> Atomic counter
TTL        -> Leaderboard lifecycle
Kafka/DB   -> Durable persistence
```

---

## Next Part

**Part 16: Redis Pub/Sub with Spring Boot — Complete Deep Dive**

Isme hum cover karenge:

- Pub/Sub kya hai
- Publisher, channel, subscriber
- Redis Pub/Sub vs Kafka
- Spring Boot publisher
- `RedisMessageListenerContainer`
- Message listener
- JSON serialization
- Delivery guarantees
- Use cases
- Limitations
- Production patterns
- Interview questions
# Part 2: Redis Data Types — Complete Deep Dive

> Redis sirf simple key-value store nahi hai. Ye ek **data structure server** hai. Sahi data type choose karna hi production Redis design ki sabse important skill hai.

---

## 1. Redis Data Types Overview

| Data Type | Structure | Main Use Case |
|---|---|---|
| `STRING` | Text, number, JSON, binary | Cache, token, counter |
| `HASH` | Field-value pairs | User profile, session |
| `LIST` | Ordered string sequence | Queue, recent activity |
| `SET` | Unique unordered strings | Followers, tags, permissions |
| `SORTED SET` | Unique strings with score | Leaderboard, ranking |
| `STREAM` | Append-only event log | Event processing, task queue |
| `BITMAP` | Bits | Online presence, feature flags |
| `HYPERLOGLOG` | Probabilistic set | Unique visitor count |
| `GEO` | Geospatial coordinates | Nearby users, delivery |

Redis lists insertion order me strings store karte hain, sets unique strings ka unordered collection hote hain, sorted sets score ke saath unique members rakhte hain, aur streams field-value pairs wali entries ka sequence hote hain. [42]

---

## 2. STRING

`STRING` Redis ka sabse basic data type hai. Ye sequence of bytes store karta hai — text, serialized object, counter value ya binary data. [28]

### Real Use Cases

- API response cache
- JWT / refresh token
- OTP
- Feature flag
- Counter
- JSON object cache
- Distributed lock

### Redis CLI

```bash
SET user:101:profile "{\"name\":\"Rahul\",\"city\":\"Bengaluru\"}"
GET user:101:profile

SET otp:101 "482913" EX 300
TTL otp:101

SET post:201:likes 0
INCR post:201:likes
GET post:201:likes
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class StringRedisService {

    private final StringRedisTemplate redisTemplate;

    public void cacheUserProfile(Long userId, String json) {
        redisTemplate.opsForValue().set(
                "user-service:user:" + userId + ":profile",
                json,
                Duration.ofMinutes(15)
        );
    }

    public String getUserProfile(Long userId) {
        return redisTemplate.opsForValue().get(
                "user-service:user:" + userId + ":profile"
        );
    }

    public Long incrementPostLikes(Long postId) {
        return redisTemplate.opsForValue().increment(
                "post-service:post:" + postId + ":likes"
        );
    }
}
```

### Production Tip

```text
Small JSON/String -> STRING
Multiple fields of one entity -> HASH
```

---

## 3. HASH

`HASH` ek key ke andar multiple field-value pairs store karta hai.

```text
user:101
  ├── name -> Rahul
  ├── city -> Bengaluru
  └── age  -> 22
```

### Real Use Cases

- User profile
- Session data
- Product details
- Configuration object
- Shopping cart

### Redis CLI

```bash
HSET user:101 name "Rahul" city "Bengaluru" age 22

HGET user:101 name
HGETALL user:101
HDEL user:101 age
HEXISTS user:101 city
HINCRBY user:101 loginCount 1
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class UserProfileRedisService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void saveUserProfile(Long userId, UserProfile profile) {
        String key = "user-service:user:" + userId + ":profile";

        redisTemplate.opsForHash().put(key, "name", profile.getName());
        redisTemplate.opsForHash().put(key, "email", profile.getEmail());
        redisTemplate.opsForHash().put(key, "city", profile.getCity());

        redisTemplate.expire(key, Duration.ofMinutes(30));
    }

    public Map<Object, Object> getUserProfile(Long userId) {
        String key = "user-service:user:" + userId + ":profile";
        return redisTemplate.opsForHash().entries(key);
    }

    public void incrementLoginCount(Long userId) {
        String key = "user-service:user:" + userId + ":profile";
        redisTemplate.opsForHash().increment(key, "loginCount", 1);
    }
}
```

### STRING vs HASH

| Scenario | Better Choice |
|---|---|
| Complete JSON object cache | `STRING` |
| Individual fields update karna | `HASH` |
| Partial read | `HASH` |
| Simple serialized DTO | `STRING` |
| User profile with frequent field updates | `HASH` |

---

## 4. LIST

`LIST` insertion order me strings ka sequence store karta hai.

```text
notifications:user:101
  ├── "You got a new follower"
  ├── "Your post was liked"
  └── "Your comment was replied"
```

### Real Use Cases

- Recent notifications
- Recent searches
- Simple task queue
- Timeline feed
- Chat message history

### Redis CLI

```bash
LPUSH notifications:user:101 "New follower"
RPUSH notifications:user:101 "Post liked"

LRANGE notifications:user:101 0 9
LLEN notifications:user:101
LPOP notifications:user:101
RPOP notifications:user:101
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class NotificationRedisService {

    private final StringRedisTemplate redisTemplate;

    public void addNotification(Long userId, String message) {
        String key = "user-service:user:" + userId + ":notifications";

        redisTemplate.opsForList().leftPush(key, message);
        redisTemplate.opsForList().trim(key, 0, 99);
        redisTemplate.expire(key, Duration.ofDays(7));
    }

    public List<String> getRecentNotifications(Long userId) {
        String key = "user-service:user:" + userId + ":notifications";
        return redisTemplate.opsForList().range(key, 0, 19);
    }
}
```

### Important Note

`LIST` simple FIFO/LIFO queue ke liye theek hai, lekin reliable distributed job queue ke liye Redis Streams ya proper message broker better hota hai.

---

## 5. SET

`SET` unique strings ka unordered collection hota hai.

```text
followers:user:101
  ├── 201
  ├── 202
  └── 203
```

### Real Use Cases

- Followers
- Following
- Liked posts
- User roles/permissions
- Tags
- Unique visitors
- Blacklisted tokens

### Redis CLI

```bash
SADD followers:user:101 201 202 203
SREM followers:user:101 202
SISMEMBER followers:user:101 201
SCARD followers:user:101
SMEMBERS followers:user:101

SINTER followers:user:101 followers:user:201
SUNION followers:user:101 followers:user:201
SDIFF followers:user:101 followers:user:201
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class FollowRedisService {

    private final StringRedisTemplate redisTemplate;

    public void follow(Long followerId, Long followingId) {
        redisTemplate.opsForSet().add(
                "user-service:user:" + followingId + ":followers",
                followerId.toString()
        );

        redisTemplate.opsForSet().add(
                "user-service:user:" + followerId + ":following",
                followingId.toString()
        );
    }

    public boolean isFollowing(Long followerId, Long followingId) {
        return Boolean.TRUE.equals(
                redisTemplate.opsForSet().isMember(
                        "user-service:user:" + followingId + ":followers",
                        followerId.toString()
                )
        );
    }

    public Long followerCount(Long userId) {
        return redisTemplate.opsForSet().size(
                "user-service:user:" + userId + ":followers"
        );
    }
}
```

### Set Operations

| Operation | Meaning | Example |
|---|---|---|
| `SINTER` | Common members | Mutual followers |
| `SUNION` | All unique members | Combined audience |
| `SDIFF` | Difference | Users followed by A but not B |

---

## 6. SORTED SET — ZSET

`SORTED SET` Redis ka sabse powerful data type hai jab ranking chahiye.

Har member unique hota hai, lekin uske saath ek numeric **score** hota hai. Redis members ko score ke according order me maintain karta hai; same score hone par lexicographical order use hota hai. [40]

```text
leaderboard
  ├── user:102 -> 1500
  ├── user:101 -> 950
  └── user:103 -> 700
```

### Real Use Cases

- Game leaderboard
- Trending posts
- Top sellers
- Most active users
- Priority queue
- Rate limiter sliding window
- Recent active users

### Redis CLI

```bash
ZADD leaderboard 950 user:101
ZADD leaderboard 1500 user:102
ZADD leaderboard 700 user:103

ZINCRBY leaderboard 50 user:101
ZSCORE leaderboard user:101
ZRANK leaderboard user:101
ZREVRANK leaderboard user:101

ZREVRANGE leaderboard 0 9 WITHSCORES
ZCOUNT leaderboard 800 1200
ZREM leaderboard user:103
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class LeaderboardRedisService {

    private final StringRedisTemplate redisTemplate;

    private static final String LEADERBOARD_KEY =
            "social-service:leaderboard:weekly";

    public void addScore(Long userId, double score) {
        redisTemplate.opsForZSet().incrementScore(
                LEADERBOARD_KEY,
                userId.toString(),
                score
        );
    }

    public Double getScore(Long userId) {
        return redisTemplate.opsForZSet().score(
                LEADERBOARD_KEY,
                userId.toString()
        );
    }

    public Long getRank(Long userId) {
        return redisTemplate.opsForZSet().reverseRank(
                LEADERBOARD_KEY,
                userId.toString()
        );
    }

    public Set<String> getTopUsers(int limit) {
        return redisTemplate.opsForZSet().reverseRange(
                LEADERBOARD_KEY,
                0,
                limit - 1
        );
    }
}
```

### Why Sorted Set Is Special

Redis documentation ke according `ZADD` score store karta hai, `ZRANGE` range retrieve karta hai, `ZRANGEBYSCORE` score range se members deta hai, aur `ZRANK` member ka rank deta hai. [36]

```text
ZADD     -> Add/update score
ZINCRBY  -> Increment score atomically
ZSCORE   -> Get score
ZRANK    -> Get ascending rank
ZREVRANK -> Get descending rank
ZREVRANGE-> Get top N users
```

---

## 7. STREAM

`STREAM` ek append-only log hai jo events ko unke occurrence order me record karta hai. Isme random access, consumer groups aur real-time event syndication support hota hai. [37]

```text
orders-stream
  ├── 1729260000000-0 -> orderId=1001, status=CREATED
  ├── 1729260001000-0 -> orderId=1002, status=CREATED
  └── 1729260002000-0 -> orderId=1003, status=CREATED
```

### Real Use Cases

- Order events
- User activity tracking
- Notification events
- Audit logs
- Sensor data
- Lightweight task queue
- Event sourcing

### Redis CLI

```bash
XADD orders-stream * orderId 1001 status CREATED amount 499

XLEN orders-stream

XRANGE orders-stream - +

XGROUP CREATE orders-stream order-processors 0

XREADGROUP GROUP order-processors consumer-1 COUNT 10 STREAMS orders-stream >

XACK orders-stream order-processors 1729260000000-0
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class OrderStreamProducer {

    private final RedisTemplate<String, Object> redisTemplate;

    public void publishOrderCreated(OrderCreatedEvent event) {
        Map<String, String> message = Map.of(
                "orderId", event.getOrderId().toString(),
                "userId", event.getUserId().toString(),
                "amount", event.getAmount().toString(),
                "status", "CREATED"
        );

        redisTemplate.opsForStream().add("order-service:orders", message);
    }
}
```

### Stream vs Pub/Sub

| Feature | Redis Stream | Redis Pub/Sub |
|---|---|---|
| Message persistence | Yes | No |
| Consumer groups | Yes | No |
| Acknowledgement | Yes | No |
| Replay | Possible | Not possible |
| Delivery guarantee | Better | At-most-once behavior |
| Use case | Task/event processing | Lightweight notifications |

Redis Streams consumer groups ke through distributed processing, acknowledgements aur fault-tolerant consumption support karte hain. [45]

---

## 8. BITMAP

`BITMAP` bits store karta hai. Ye memory-efficient hai jab sirf `true/false`, active/inactive, visited/not-visited type data chahiye.

### Real Use Cases

- Daily active users
- Online presence
- Feature flags
- Email opened/clicked
- Attendance tracking

### Redis CLI

```bash
SETBIT online:2026-10-18 101 1
GETBIT online:2026-10-18 101
BITCOUNT online:2026-10-18
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class PresenceRedisService {

    private final StringRedisTemplate redisTemplate;

    public void markUserOnline(Long userId) {
        String key = "presence-service:online:" + LocalDate.now();
        redisTemplate.opsForValue().setBit(key, userId, true);
        redisTemplate.expire(key, Duration.ofDays(2));
    }

    public boolean isUserOnline(Long userId) {
        String key = "presence-service:online:" + LocalDate.now();
        return Boolean.TRUE.equals(
                redisTemplate.opsForValue().getBit(key, userId)
        );
    }
}
```

---

## 9. HYPERLOGLOG

`HYPERLOGLOG` approximate unique count deta hai, bahut kam memory me.

Example: 10 million unique visitors count karne ke liye exact set me bahut memory lagegi, lekin HyperLogLog me memory almost constant rehti hai.

### Real Use Cases

- Unique website visitors
- Unique search terms
- Unique viewers of a video
- Unique users who performed an action

### Redis CLI

```bash
PFADD visitors:2026-10-18 user:101 user:102 user:103
PFCOUNT visitors:2026-10-18
PFMERGE visitors:weekly visitors:2026-10-17 visitors:2026-10-18
```

### Important

```text
HyperLogLog exact count nahi deta.
Ye approximate count deta hai, lekin memory bahut efficient hai.
```

---

## 10. GEO

`GEO` geospatial coordinates store karta hai aur nearby-location queries support karta hai.

### Real Use Cases

- Nearby delivery partners
- Nearby restaurants
- Nearby users
- Cab matching
- Store locator

### Redis CLI

```bash
GEOADD delivery-partners 77.5946 12.9716 partner:101
GEOADD delivery-partners 77.6101 12.9352 partner:102

GEOSEARCH delivery-partners FROMLONLAT 77.5946 12.9716
BYRADIUS 5 km ASC COUNT 10
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class NearbyPartnerService {

    private final StringRedisTemplate redisTemplate;

    public void addPartner(Long partnerId, double longitude, double latitude) {
        redisTemplate.opsForGeo().add(
                "delivery-service:partners",
                new Point(longitude, latitude),
                partnerId.toString()
        );
    }

    public List<String> findNearbyPartners(
            double longitude,
            double latitude,
            double radiusKm
    ) {
        Circle circle = new Circle(
                new Point(longitude, latitude),
                new Distance(radiusKm, Metrics.KILOMETERS)
        );

        RedisGeoCommands.GeoSearchCommandArgs args =
                RedisGeoCommands.GeoSearchCommandArgs.newGeoSearchArgs()
                        .includeDistance()
                        .sortAscending()
                        .limit(10);

        return redisTemplate.opsForGeo()
                .search("delivery-service:partners", circle, args)
                .getContent()
                .stream()
                .map(result -> result.getContent().getName())
                .toList();
    }
}
```

---

## 11. Data Type Selection Guide

| Requirement | Best Redis Type |
|---|---|
| Cache API response | `STRING` |
| Store object fields | `HASH` |
| Recent notifications | `LIST` |
| Unique followers | `SET` |
| Leaderboard | `SORTED SET` |
| Event processing | `STREAM` |
| Online status | `BITMAP` / `SET` |
| Unique visitor count | `HYPERLOGLOG` |
| Nearby search | `GEO` |
| Rate limiting | `STRING` / `SORTED SET` |
| Distributed lock | `STRING` with `SET NX PX` |

---

## 12. Production Key Design Examples

### User Service

```text
user-service:user:101:profile
user-service:user:101:followers
user-service:user:101:following
user-service:user:101:notifications
```

### Post Service

```text
post-service:post:201:details
post-service:post:201:likes
post-service:post:201:comments:recent
post-service:trending:posts
```

### Auth Service

```text
auth-service:refresh-token:{token}
auth-service:blacklist:{jti}
auth-service:login-attempts:{email}
```

### Order Service

```text
order-service:order:1001:details
order-service:orders:stream
order-service:idempotency:{requestId}
order-service:lock:order:1001
```

---

## 13. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Har cheez STRING me store karna | Field-level operations difficult | Structured data ke liye HASH |
| Followers ke liye LIST use karna | Duplicate possible, membership check slow | SET use karo |
| Leaderboard ke liye LIST use karna | Sorting manual karni padegi | SORTED SET use karo |
| Critical events ke liye Pub/Sub | Message loss possible | STREAM ya Kafka |
| Exact unique count ke liye SET | Memory zyada | Large-scale approximate counting me HyperLogLog |
| Large JSON repeatedly update karna | Network/memory overhead | HASH ya smaller keys |

---

## 14. Interview Questions

| Question | Answer |
|---|---|
| Redis STRING kab use karte hain? | Cache, token, counter, OTP, JSON object ke liye. |
| HASH aur STRING me difference? | HASH multiple field-value pairs store karta hai; STRING single value store karta hai. |
| SET aur LIST me difference? | SET unique unordered members rakhta hai; LIST ordered sequence rakhta hai aur duplicates allow karta hai. |
| Sorted Set kya hai? | Unique members with numeric score, score ke according sorted. |
| Leaderboard ke liye kaunsa type? | Sorted Set. |
| Redis Stream kya hai? | Append-only event log with consumer groups and acknowledgements. |
| Pub/Sub aur Stream me difference? | Pub/Sub fire-and-forget hai; Stream persistence, consumer groups aur ACK support karta hai. |
| HyperLogLog kab use karte hain? | Large dataset ka approximate unique count nikalne ke liye. |
| GEO kab use karte hain? | Nearby users, stores, delivery partners find karne ke liye. |

---

## 15. Final Mental Model

```text
STRING       -> single value
HASH         -> object with fields
LIST         -> ordered sequence
SET          -> unique values
SORTED SET   -> ranked values
STREAM       -> event log
BITMAP       -> true/false flags
HYPERLOGLOG  -> approximate unique count
GEO          -> location data
```

---

## Next Part

**Part 3: Redis Commands, TTL, Expiration and Transactions — Complete Deep Dive**

Isme hum cover karenge:

- Basic Redis commands
- TTL and expiration
- `EXPIRE`, `TTL`, `PERSIST`
- Atomic operations
- `INCR`, `DECR`
- Pipelining
- `MULTI`, `EXEC`, `DISCARD`
- Lua scripting
- `SCAN` vs `KEYS`
- Spring Boot Redis operations
- Production commands to avoid
- Interview questions
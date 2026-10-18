# Part 1: Introduction to Redis — Complete Deep Dive

> Redis ko samajhne ka best tarika: ye ek **super-fast, in-memory data structure server** hai jo normally cache ke liye use hota hai, lekin sessions, locks, rate limiting, queues, leaderboards aur real-time features ke liye bhi powerful hai.

---

## 1. Redis Kya Hai?

**Redis** ka full form hai:

> **Remote Dictionary Server**

Redis ek open-source, in-memory data structure store hai.

Simple words me:

```text
Redis = Fast in-memory key-value database + data structures + cache + messaging
```

Redis data ko primarily **RAM/memory** me store karta hai, isliye normal disk-based databases se bahut fast hota hai. Redis documentation ke according ye database, cache, message broker aur streaming engine ke roop me use ho sakta hai. [23]

### Basic Mental Model

```text
Application
    |
    | GET user:101
    v
Redis
    |
    +--> key: user:101
    |
    +--> value: JSON / String / Hash / List / Set
```

Example:

```bash
SET user:101 "Rahul"
GET user:101
```

Output:

```text
"Rahul"
```

---

## 2. Redis Itna Fast Kyun Hai?

Redis fast isliye hai kyunki:

| Reason | Explanation |
|---|---|
| In-memory storage | Data mostly RAM me rehta hai, disk access normal operation me nahi hota |
| Simple data structures | Strings, Hashes, Lists, Sets, Sorted Sets optimized hain |
| Single-threaded command execution | Commands sequentially execute hoti hain, isliye locking overhead kam hota hai |
| Efficient network protocol | RESP protocol lightweight hai |
| Pipelining | Multiple commands ek network round-trip me bhej sakte hain |
| Optimized C implementation | Redis C language me likha gaya hai |

### Important Clarification

Redis “single-threaded” hai, lekin iska matlab ye nahi ki Redis sirf ek thread use karta hai.

```text
Main thread:
- Client commands execute karta hai

Background threads:
- Persistence
- Replication
- Async operations
```

Normal command execution single-threaded hoti hai, isliye Redis operations predictable aur fast hote hain.

---

## 3. Redis vs Relational Database

| Feature | Redis | MySQL / PostgreSQL |
|---|---|---|
| Storage | Mostly RAM | Disk |
| Data model | Key-value + data structures | Tables, rows, columns |
| Speed | Extremely fast | Relatively slower for simple reads |
| Joins | Direct support nahi | Strong SQL joins |
| Transactions | Basic MULTI/EXEC, Lua scripts | Full ACID transactions |
| Query language | Commands | SQL |
| Best for | Cache, session, real-time data | Source of truth, complex relations |
| Durability | Optional/configurable | Strong by default |
| Scalability | Horizontal scaling via Cluster | Vertical + read replicas/sharding |

### Golden Rule

```text
Redis = Speed layer
Database = Source of truth
```

Production me generally:

```text
Client
  |
  v
Spring Boot API
  |
  +--> Redis Cache
  |
  +--> PostgreSQL / MySQL
```

---

## 4. Redis vs MongoDB

| Feature | Redis | MongoDB |
|---|---|---|
| Storage | In-memory first | Disk-based document store |
| Data model | Key-value and data structures | JSON-like documents |
| Best for | Caching, sessions, counters, real-time data | Flexible application data |
| Querying | Command-based | Rich document queries |
| Persistence | RDB/AOF configurable | Built-in durable storage |
| Use case | Low-latency operations | General-purpose application database |

### Simple Rule

```text
Agar data permanently important hai -> MongoDB / PostgreSQL
Agar data fast access ke liye chahiye -> Redis
```

---

## 5. Redis vs Kafka

Ye comparison bahut important hai kyunki dono event/data movement me use hote hain.

| Feature | Redis | Kafka |
|---|---|---|
| Primary role | Cache, data structures, low-latency operations | Distributed event log / streaming platform |
| Message retention | Configurable, memory-focused | Long-term durable log |
| Consumer groups | Redis Streams support karta hai | Strong native support |
| Replay | Limited | Strong replay support |
| Ordering | Per stream/partition | Per partition |
| Throughput | Very high for small, fast operations | Very high for large-scale event streaming |
| Best for | Cache, locks, rate limits, presence, lightweight streams | Event sourcing, analytics, microservice communication |

### Rule of Thumb

```text
Redis Streams  -> lightweight event flow
Kafka          -> mission-critical event backbone
```

Redis Streams consumer groups aur acknowledgements support karta hai, lekin Kafka generally long-term retention, replay aur large-scale event streaming ke liye stronger choice hai.

---

## 6. Redis Data Model

Redis me data hamesha key-value form me hota hai.

```text
KEY -> VALUE
```

Lekin value sirf plain string nahi hoti. Redis different data structures support karta hai.

### Example

```bash
SET name "Rahul"
```

```text
Key   -> name
Value -> "Rahul"
```

```bash
HSET user:101 name "Rahul" age 22
```

```text
Key   -> user:101
Value -> Hash
        name -> Rahul
        age  -> 22
```

---

## 7. Redis Core Data Structures

| Data Type | Example Use Case |
|---|---|
| `STRING` | Cache, token, counter, JSON object |
| `HASH` | User profile, session data |
| `LIST` | Simple queue, recent activity |
| `SET` | Unique followers, tags, permissions |
| `SORTED SET` | Leaderboard, trending posts |
| `STREAM` | Event stream, task queue |
| `BITMAP` | Online status, feature flags |
| `HYPERLOGLOG` | Unique visitor count |
| `GEO` | Nearby users, delivery locations |

### Real Examples

```bash
# Cache
SET product:101:details "{...}"

# User profile
HSET user:101 name "Rahul" city "Bengaluru"

# Followers
SADD user:101:followers 201 202 203

# Leaderboard
ZADD post:likes 120 post:1

# Rate limit
INCR rate-limit:user:101
```

---

## 8. Redis Real-World Use Cases

### 1. Caching

Sabse common use case.

```text
Client -> Spring Boot -> Redis
                           |
                           +--> Cache hit: return data
                           |
                           +--> Cache miss: fetch from DB, store in Redis
```

Redis documentation cache-aside pattern ko TTL-bounded staleness ke saath recommend karta hai. [21]

Example:

```java
@Cacheable(value = "userProfile", key = "#userId")
public UserProfile getUserProfile(Long userId) {
    return userRepository.findById(userId).orElseThrow();
}
```

### 2. Session Storage

```text
Authorization: Bearer JWT
        |
        +--> Redis me refresh token / session store
```

Use cases:

- Login session
- Refresh token
- Logout/token revocation
- Multi-device session management

### 3. Rate Limiting

```text
User request -> Redis INCR -> limit check
```

Example:

```text
Maximum 100 requests per minute per user
```

### 4. Distributed Lock

Microservices me jab multiple instances same resource update karne ki try karein:

```text
Instance A ---> lock:user:101
Instance B ---> lock:user:101
```

Sirf ek instance lock acquire kar sakega.

### 5. Leaderboard

Redis Sorted Set real-time leaderboard ke liye ideal hai. Redis documentation ke according sorted set me members entity ID hote hain aur score ranking value hota hai. [24]

```bash
ZADD leaderboard 950 user:101
ZADD leaderboard 1200 user:102
ZREVRANGE leaderboard 0 9
```

### 6. Pub/Sub

```text
Service A -> publish "order-created"
Service B -> subscribe "order-created"
```

Lekin Redis Pub/Sub me message delivery guarantee nahi hoti. Critical event processing ke liye Kafka better hota hai.

### 7. Redis Streams

```text
Producer -> XADD orders
Consumer Group -> XREADGROUP orders
Consumer -> XACK orders
```

Ye lightweight event processing ke liye use hota hai.

### 8. Counters

```bash
INCR post:101:likes
INCR video:201:views
```

### 9. Online Presence

```text
user:101 -> online
user:102 -> online
```

### 10. Feature Flags

```bash
SET feature:new-feed-enabled true
```

---

## 9. Redis Architecture

### Basic Architecture

```text
+----------------------+
|     Spring Boot      |
|      Application     |
+----------+-----------+
           |
           | Redis Protocol / RESP
           v
+----------------------+
|        Redis         |
|   In-Memory Store    |
+----------+-----------+
           |
           +--> RDB Snapshot
           |
           +--> AOF Log
```

### Components

| Component | Role |
|---|---|
| Redis Server | Data store and command executor |
| Redis Client | Spring Boot application / RedisTemplate |
| RESP | Redis serialization protocol |
| Persistence | RDB and/or AOF |
| Replication | Master-replica data copy |
| Sentinel | Monitoring and automatic failover |
| Cluster | Data sharding across multiple nodes |

Redis persistence me RDB point-in-time snapshots leta hai aur AOF har write operation ko log karta hai; dono ko combine bhi kiya ja sakta hai. [22]

---

## 10. Redis Request Flow

### Cache Hit Flow

```text
1. Client requests GET /users/101
2. Spring Boot checks Redis
3. Redis returns cached user data
4. Database call skipped
5. Response returned quickly
```

### Cache Miss Flow

```text
1. Client requests GET /users/101
2. Spring Boot checks Redis
3. Redis returns null
4. Spring Boot fetches data from PostgreSQL
5. Spring Boot stores data in Redis with TTL
6. Response returned to client
```

### Diagram

```text
        GET /users/101
              |
              v
       +-------------+
       | Spring Boot |
       +-------------+
              |
              v
       +-------------+
       |    Redis    |
       +-------------+
          |       |
     Cache Hit   Cache Miss
          |       |
          v       v
       Return   PostgreSQL
       Response      |
                     v
                Store in Redis
                     |
                     v
                  Return
```

---

## 11. Redis Persistence Overview

Redis by default in-memory hai, lekin production me persistence configure karna important hai.

| Persistence | Working | Benefit | Limitation |
|---|---|---|---|
| RDB | Point-in-time snapshot | Compact backup, fast restart | Last snapshot ke baad ka data loss ho sakta hai |
| AOF | Har write command log hota hai | Better durability | File size zyada ho sakti hai |
| RDB + AOF | Dono combined | Strong recovery strategy | Thoda zyada disk/CPU usage |

Redis RDB snapshots specified intervals par banata hai, jabki AOF har write operation ko append-only file me log karta hai. [25]

### Production Recommendation

```conf
appendonly yes
appendfsync everysec
```

Iska matlab approximately 1 second tak ka data loss possible ho sakta hai, lekin performance aur durability ka good balance milta hai.

---

## 12. Redis Kab Use Karna Chahiye?

Redis use karo jab:

- API response cache karna ho
- Database query result cache karna ho
- User session store karna ho
- Rate limiting implement karni ho
- Distributed lock chahiye
- Real-time leaderboard chahiye
- Online presence track karni ho
- Counter maintain karna ho
- Lightweight event stream chahiye
- Feature flag store karna ho

---

## 13. Redis Kab Avoid Karna Chahiye?

Redis avoid karo jab:

- Data permanently durable hona chahiye
- Complex relational queries chahiye
- Strong ACID transactions chahiye
- Large historical data store karna ho
- Complex joins required hain
- Long-term event retention chahiye
- Message delivery guarantee critical hai
- Data size RAM me fit nahi hota

### Important Production Rule

```text
Redis ko primary database mat banao
jab data loss business-critical ho.
```

---

## 14. Redis in Spring Boot Microservices

Typical microservice architecture:

```text
                     +----------------+
                     |   API Gateway  |
                     +--------+-------+
                              |
        +---------------------+---------------------+
        |                     |                     |
+-------v-------+     +-------v-------+     +-------v-------+
| User Service  |     | Post Service  |     | Feed Service  |
+-------+-------+     +-------+-------+     +-------+-------+
        |                     |                     |
        +----------+----------+----------+----------+
                   |                     |
                   v                     v
             +-----------+         +-----------+
             |   Redis   |         |   Kafka   |
             +-----------+         +-----------+
                   |
                   v
             +-----------+
             | PostgreSQL|
             +-----------+
```

### Redis Responsibilities

| Service | Redis Use |
|---|---|
| User Service | Profile cache, session, refresh token |
| Post Service | Post cache, like counter |
| Feed Service | Feed cache, trending posts |
| Auth Service | Token blacklist, refresh token |
| Notification Service | Pub/Sub, online presence |
| Order Service | Distributed lock, idempotency key |

---

## 15. Redis Key Naming Convention

Production me keys readable aur collision-free honi chahiye.

### Bad

```text
user101
cache
data
temp
```

### Good

```text
user-service:user:101:profile
post-service:post:201:likes
auth-service:refresh-token:abc123
rate-limit:user:101:2026-10-18T22
```

### Recommended Format

```text
<service-name>:<entity>:<id>:<attribute>
```

Example:

```text
user-service:user:101:profile
```

---

## 16. Redis Basic Commands

### String Commands

```bash
SET name "Rahul"
GET name
DEL name
EXISTS name
```

### TTL Commands

```bash
SET otp:101 "482913" EX 300
TTL otp:101
```

### Counter Commands

```bash
SET likes:101 0
INCR likes:101
GET likes:101
```

### Hash Commands

```bash
HSET user:101 name "Rahul" city "Bengaluru"
HGET user:101 name
HGETALL user:101
```

### Set Commands

```bash
SADD followers:101 201 202
SISMEMBER followers:101 201
SCARD followers:101
```

### Sorted Set Commands

```bash
ZADD leaderboard 100 user:101
ZINCRBY leaderboard 50 user:101
ZREVRANGE leaderboard 0 9 WITHSCORES
```

---

## 17. Redis Security Mindset

Redis by default network-exposed hone par dangerous ho sakta hai.

Production me:

- Password/authentication enable karo
- TLS use karo
- Redis ko public internet expose mat karo
- Private network/VPC use karo
- Dangerous commands restrict karo
- ACL users configure karo
- Backup aur monitoring rakho

---

## 18. Common Beginner Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Redis ko primary DB samajhna | Data loss risk | DB ko source of truth rakho |
| TTL set na karna | Memory full ho jayegi | Har cache key par TTL |
| `KEYS *` use karna | Production blocking | `SCAN` use karo |
| Java serialization use karna | Compatibility issues | JSON serialization |
| Cache invalidation ignore karna | Stale data | Write path me invalidation |
| Redis ko durable queue samajhna | Message loss possible | Requirement ke hisaab se Kafka/Streams choose karo |
| Security ignore karna | Unauthorized access | Auth, TLS, network isolation |

---

## 19. Redis Production Mindset

Production me Redis ke liye ye 5 cheezein always yaad rakho:

```text
1. TTL
2. Memory limit
3. Eviction policy
4. Security
5. Monitoring
```

### Simple Production Formula

```text
Redis = Fast temporary data layer
Database = Permanent source of truth
Kafka = Durable event backbone
```

---

## 20. Interview Questions

### Basic Questions

| Question | Answer |
|---|---|
| Redis kya hai? | Open-source in-memory data structure store used as cache, database, message broker and streaming engine. |
| Redis fast kyun hai? | Data RAM me rehta hai, optimized data structures aur efficient command execution hote hain. |
| Redis aur MySQL me difference? | Redis in-memory key-value/data-structure store hai; MySQL relational disk-based database hai. |
| Redis aur MongoDB me difference? | Redis low-latency in-memory store hai; MongoDB durable document database hai. |
| Redis aur Kafka me difference? | Redis cache/low-latency operations ke liye; Kafka durable event streaming ke liye. |

### Intermediate Questions

| Question | Answer |
|---|---|
| Redis persistence kya hai? | RDB snapshots aur AOF write logs ke through data ko disk par save karna. |
| RDB aur AOF me difference? | RDB point-in-time snapshot hai; AOF har write command log karta hai. |
| Redis cache me TTL kyun zaroori hai? | Memory control aur stale data avoid karne ke liye. |
| Redis single-threaded kyun hai? | Main command execution single-threaded hoti hai, jisse atomicity aur predictability milti hai. |
| Redis Pub/Sub reliable kyun nahi hai? | Subscriber offline ho to message miss ho sakta hai; delivery guarantee nahi hoti. |

### Scenario-Based Questions

| Scenario | Answer |
|---|---|
| API slow hai, DB load high hai | Redis cache-aside pattern use karo |
| User duplicate order create kar raha hai | Redis distributed lock ya idempotency key use karo |
| API abuse ho raha hai | Redis rate limiter use karo |
| Live cricket scoreboard chahiye | Redis Sorted Set use karo |
| User online/offline status chahiye | Redis Set, Hash ya Bitmap use karo |
| Microservices me event bhejne hain | Redis Streams for lightweight flow, Kafka for durable event backbone |

---

## 21. Final Mental Model

```text
Redis ek fast in-memory data structure server hai.

Ye cache ke liye best hai,
sessions ke liye useful hai,
rate limiting ke liye powerful hai,
distributed locks ke liye practical hai,
leaderboards ke liye ideal hai,
aur lightweight real-time event processing ke liye capable hai.

Lekin permanent, relational aur strongly durable data
ke liye PostgreSQL/MySQL better hai.

Large-scale durable event streaming ke liye Kafka better hai.
```

---

## Next Part

**Part 2: Redis Data Types — Complete Deep Dive**

Isme hum cover karenge:

- `STRING`
- `HASH`
- `LIST`
- `SET`
- `SORTED SET`
- `STREAM`
- `BITMAP`
- `HYPERLOGLOG`
- `GEO`
- Har data type ka internal use case
- Spring Boot + Java examples
- Real production scenarios
- Memory considerations
- Interview questions
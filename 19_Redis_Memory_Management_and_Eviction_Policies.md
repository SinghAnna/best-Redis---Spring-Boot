# Part 19: Redis Memory Management and Eviction Policies — Complete Deep Dive

> Redis in-memory database hai, isliye memory management production me sabse critical topic hai. Galat `maxmemory` ya eviction policy se data loss, write failures aur application outage ho sakta hai.

---

## 1. Redis Memory Model

Redis data ko memory me rakhta hai:

```text
Redis Process
     |
     +--> Dataset memory
     |
     +--> Client buffers
     |
     +--> Replication buffers
     |
     +--> AOF buffers
     |
     +--> Fragmentation overhead
```

Redis memory fragmentation ko physical memory RSS aur logical used memory ke ratio se calculate karta hai. [269]

---

## 2. Important Memory Metrics

```bash
INFO memory
```

Important fields:

| Field | Meaning |
|---|---|
| `used_memory` | Redis dataset + internal structures ka logical memory |
| `used_memory_rss` | Operating system level actual RAM usage |
| `used_memory_peak` | Maximum memory used till now |
| `mem_fragmentation_ratio` | RSS / used_memory |
| `maxmemory` | Configured memory limit |
| `maxmemory_policy` | Eviction policy |
| `evicted_keys` | Evicted keys count |

### Fragmentation Ratio

\[
\text{Fragmentation Ratio} =
\frac{\text{used_memory_rss}}{\text{used_memory}}
\]

| Ratio | Interpretation |
|---|---|
| Around 1.0 | Healthy |
| 1.2–1.5 | Normal/acceptable |
| 1.5–2.0 | Moderate fragmentation |
| Above 2.0 | High fragmentation, investigate |

Redis fragmentation ratio `used_memory_rss` ko `used_memory` se divide karke calculate hota hai. [269]

---

## 3. `maxmemory`

`maxmemory` Redis dataset ke liye memory cap set karta hai.

```conf
maxmemory 2gb
```

### Important

```text
maxmemory sirf dataset ke liye hota hai,
pure Redis process ki total RAM requirement ke liye nahi.
```

Redis ko replication buffers, client buffers aur fragmentation overhead ke liye extra memory chahiye hoti hai.

### Recommended Formula

```text
Machine RAM = 8 GB

Safe Redis maxmemory = 5–6 GB

Remaining memory:
- OS
- Redis process overhead
- Replication buffers
- AOF buffers
- Fragmentation
```

---

## 4. `maxmemory-policy`

Jab `maxmemory` limit reach ho jaye, to `maxmemory-policy` decide karta hai:

```text
Kaunsi keys evict hongi?
Ya naya write fail hoga?
```

```conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

Eviction policy determine karti hai ki memory limit reach hone par kya hoga — older data remove hoga ya new writes reject honge. [271]

---

## 5. Eviction Policies

| Policy | Eviction Scope | Behavior |
|---|---|---|
| `noeviction` | None | Memory full par write error |
| `allkeys-lru` | All keys | Least recently used key evict |
| `allkeys-lfu` | All keys | Least frequently used key evict |
| `allkeys-random` | All keys | Random key evict |
| `volatile-lru` | TTL keys | Least recently used TTL key evict |
| `volatile-lfu` | TTL keys | Least frequently used TTL key evict |
| `volatile-random` | TTL keys | Random TTL key evict |
| `volatile-ttl` | TTL keys | Shortest remaining TTL key evict |

`noeviction` policy me memory limit reach hone par new values save nahi hoti aur write error return hota hai. [271][275]

---

## 6. LRU vs LFU

### LRU — Least Recently Used

```text
Kaunsi key sabse purani access hui?
```

Example:

```text
Key A: 2 hours pehle access
Key B: 5 minutes pehle access

LRU evicts Key A
```

### LFU — Least Frequently Used

```text
Kaunsi key sabse kam baar access hui?
```

Example:

```text
Key A: 1 time accessed
Key B: 10,000 times accessed

LFU evicts Key A
```

### When to Use

| Workload | Better Policy |
|---|---|
| Recently accessed data important | LRU |
| Frequently accessed hot keys important | LFU |
| Viral/spiky traffic | LFU |
| Normal API cache | LRU |

`allkeys-lru` recently used keys retain karta hai, jabki `allkeys-lfu` frequently used keys retain karta hai. [271]

---

## 7. Policy Selection Guide

### Pure Cache

```conf
maxmemory-policy allkeys-lru
```

Use case:

```text
Redis sirf cache hai.
Koi bhi key evict ho sakti hai.
DB source of truth hai.
```

### Hot Key Heavy Workload

```conf
maxmemory-policy allkeys-lfu
```

Use case:

```text
Kuch keys bahut frequently access hoti hain.
```

### Mixed Persistent + Cache Data

```conf
maxmemory-policy volatile-lru
```

Use case:

```text
Kuch keys TTL wali hain,
kuch keys important persistent data hain.
```

### Rate Limiter / Short-Lived Keys

```conf
maxmemory-policy volatile-ttl
```

Use case:

```text
Short TTL keys ko pehle evict karna hai.
```

### Critical Data Store

```conf
maxmemory-policy noeviction
```

Use case:

```text
Data loss acceptable nahi hai.
Memory full hone par writes fail hongi.
```

`volatile-*` policies me agar koi TTL wali key available nahi hoti, to behavior `noeviction` jaisa ho jata hai. [275]

---

## 8. Key Expiration

Redis keys ko expire karne ke liye TTL set karte hain.

```bash
SET otp:101 "482913" EX 300
TTL otp:101
```

### Expiration Types

| Type | Working |
|---|---|
| Lazy / Passive expiration | Key access hone par expiry check hota hai |
| Active expiration | Redis periodically random TTL keys sample karke expire karta hai |

Lazy expiration me key access hone par expiry check hota hai; active expiration me Redis periodically random TTL keys sample karke expired keys delete karta hai. [274][280]

### Flow

```text
GET otp:101
     |
     v
Redis checks TTL
     |
     +--> Not expired -> Return value
     |
     +--> Expired -> Delete key -> Return nil
```

---

## 9. TTL Best Practices

### Rule 1: Cache Keys Must Have TTL

```java
redisTemplate.opsForValue().set(
        key,
        value,
        Duration.ofMinutes(15)
);
```

### Rule 2: TTL Jitter Use Karo

```java
long baseTtl = 1800;
long jitter = ThreadLocalRandom.current().nextLong(0, 300);

Duration ttl = Duration.ofSeconds(baseTtl + jitter);
```

### Rule 3: Business Data Ke Liye TTL Carefully Set Karo

| Data | TTL |
|---|---|
| OTP | 5 minutes |
| Session | 30 minutes |
| Refresh token | 7 days |
| Product cache | 30 minutes |
| Feed cache | 30 seconds |
| Rate limit | 1 minute |

---

## 10. Memory Optimization Techniques

### 1. Small DTO Cache Karo

```java
// Bad
cache entire entity graph

// Good
cache only required response DTO
```

### 2. Large Values Avoid Karo

```text
Bad:
Single Redis value = 10 MB

Good:
Value size = 10 KB–100 KB
```

### 3. Hash Me Fields Use Karo

Instead of multiple separate keys:

```text
user:101:name
user:101:email
user:101:city
```

Use:

```text
user:101:profile
  ├── name
  ├── email
  └── city
```

### 4. Short IDs Use Karo

```text
Bad:
user-service:user-profile-cache:101

Good:
user-service:user:101:profile
```

### 5. Compression

Large JSON values ke liye compression consider karo, lekin CPU trade-off measure karo.

### 6. TTL + Eviction Policy

```conf
maxmemory 4gb
maxmemory-policy allkeys-lru
```

---

## 11. Large Keys Problem

Large key ka matlab hai:

```text
Ek Redis key ke andar bahut zyada data
```

Example:

```text
followers:user:101
10 million members
```

### Problems

- Slow operations
- Network latency
- Redis blocking risk
- Memory spikes
- Cluster migration difficulty

### Solution

```text
Large set ko chunks me divide karo:

followers:user:101:0
followers:user:101:1
followers:user:101:2
```

---

## 12. Active Defragmentation

Memory fragmentation hone par Redis active defragmentation use kar sakta hai.

```conf
activedefrag yes

active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100
```

Active defragmentation background me values relocate karke scattered free memory ko consolidate karta hai. [273]

### Trade-off

```text
Memory reclaim hoti hai,
lekin CPU usage badh sakta hai.
```

Active defrag CPU cost ke saath fragmentation reduce karta hai; ise enable karne se pehle CPU capacity aur memory metrics monitor karo. [272][273]

---

## 13. Production Memory Configuration

### Cache-Only Redis

```conf
maxmemory 4gb
maxmemory-policy allkeys-lru

appendonly no
save ""
```

### Session + Cache Redis

```conf
maxmemory 4gb
maxmemory-policy volatile-lru

appendonly yes
appendfsync everysec
```

### Critical Data Redis

```conf
maxmemory 4gb
maxmemory-policy noeviction

appendonly yes
appendfsync everysec
```

### Docker Compose Example

```yaml
services:
  redis:
    image: redis:7.4-alpine
    command: >
      redis-server
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  redis_data:
```

---

## 14. Memory Monitoring Commands

### Memory Info

```bash
INFO memory
```

### Key Count

```bash
DBSIZE
```

### Big Keys Scan

```bash
redis-cli --bigkeys
```

### Memory Usage of a Key

```bash
MEMORY USAGE user:101:profile
```

### Eviction Stats

```bash
INFO stats
```

Important:

```text
evicted_keys
expired_keys
```

---

## 15. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| `maxmemory` set na karna | Host RAM exhaust | Memory cap set karo |
| `noeviction` with cache workload | Write errors | `allkeys-lru` |
| TTL na dena | Memory full | Har cache key TTL |
| Same TTL for all keys | Cache avalanche | TTL jitter |
| Very large values | Latency/memory spike | Small DTO/values |
| High fragmentation ignore karna | OOM / performance issue | Defrag + monitoring |
| `maxmemory` ko machine RAM ke barabar set karna | OOM kill | Headroom rakho |

---

## 16. Interview Questions

| Question | Answer |
|---|---|
| `maxmemory` kya hai? | Redis dataset ke liye memory limit. |
| `maxmemory-policy` kya decide karta hai? | Memory limit reach hone par eviction behavior. |
| LRU aur LFU me difference? | LRU least recently used evict karta hai; LFU least frequently used evict karta hai. |
| `noeviction` kab use karte hain? | Jab data loss acceptable nahi aur memory full hone par write fail hona chahiye. |
| Lazy expiration kya hai? | Key access hone par expiry check aur deletion. |
| Active expiration kya hai? | Redis periodically random TTL keys sample karke expire karta hai. |
| Memory fragmentation kya hai? | Physical RSS memory aur logical used memory ka gap. |
| Active defragmentation kya hai? | Background me memory relocate karke fragmentation reduce karna. |
| Large key problem kya hai? | Ek key me bahut zyada data hone se latency, blocking aur memory spike. |

---

## 17. Final Mental Model

```text
maxmemory          -> Memory limit
maxmemory-policy   -> Eviction behavior
TTL                -> Key lifecycle
LRU                -> Recency-based eviction
LFU                -> Frequency-based eviction
noeviction         -> No automatic eviction
Fragmentation      -> RSS vs used memory gap
Defrag             -> Memory cleanup
```

---

## Next Part

**Part 20: Redis Performance Optimization — Complete Deep Dive**

Isme hum cover karenge:

- Redis performance fundamentals
- Latency troubleshooting
- Pipelining
- Connection pooling
- Blocking commands
- `KEYS` vs `SCAN`
- Hot keys
- Big keys
- Slow log
- Client-side caching
- Redis benchmarking
- Production performance checklist
- Interview questions
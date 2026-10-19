# Part 20: Redis Performance Optimization — Complete Deep Dive

> Redis normally sub-millisecond latency deta hai, lekin wrong commands, large keys, hot keys, network round-trips ya memory pressure performance ko degrade kar sakte hain. Performance optimization ka goal hai: fast, predictable aur scalable Redis operations.

---

## 1. Redis Performance Fundamentals

Redis performance in factors par depend karti hai:

| Factor | Impact |
|---|---|
| Network round-trip time | Har command ka network latency |
| Command complexity | `O(1)` vs `O(N)` operations |
| Value size | Large payload = slow transfer |
| Number of keys | Large collections slow ho sakte hain |
| Memory pressure | Eviction, swapping, fragmentation |
| CPU | Lua scripts, large scans, heavy commands |
| Persistence | AOF fsync, RDB fork |
| Client connection model | Pooling vs many connections |

Redis latency troubleshooting me pehla step hota hai slow/blocking commands identify karna, jiske liye Slow Log use hota hai. [283]

---

## 2. Redis Command Complexity

| Command | Complexity | Production Note |
|---|---|---|
| `GET` | `O(1)` | Fast |
| `SET` | `O(1)` | Fast |
| `INCR` | `O(1)` | Fast |
| `HGET` | `O(1)` | Fast |
| `LPUSH` | `O(1)` | Fast |
| `SADD` | `O(1)` | Fast |
| `ZADD` | `O(log N)` | Generally fast |
| `LRANGE` | `O(S+N)` | Range size control karo |
| `SMEMBERS` | `O(N)` | Large set me risky |
| `HGETALL` | `O(N)` | Large hash me risky |
| `KEYS` | `O(N)` | Production me avoid |
| `SCAN` | Incremental | Safe alternative |
| `DEL` large collection | `O(N)` | Async delete consider karo |

Redis performance tuning guidance me slow commands, especially `EVALSHA`, `HGETALL`, `HMGET`, `MGET` aur `SCAN`, ko slow log me check karna recommend kiya jata hai. [285]

---

## 3. Network Round Trips

### Without Pipelining

```text
Client -> SET A
Client <- OK

Client -> SET B
Client <- OK

Client -> SET C
Client <- OK
```

Har command ka apna network round-trip.

### With Pipelining

```text
Client -> SET A, SET B, SET C
Client <- OK, OK, OK
```

Pipelining multiple commands ko ek saath bhejta hai bina har command ka response wait kiye, jisse performance improve hoti hai. [281]

### Formula

```text
Total Time =
Number of Commands x Network Round Trip Time
```

Example:

```text
1000 commands
1 ms RTT

Without pipelining = 1000 ms
With pipelining = Much lower
```

---

## 4. Spring Boot Pipelining

```java
package com.example.performance;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisCallback;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.nio.charset.StandardCharsets;
import java.util.List;

@Service
@RequiredArgsConstructor
public class PipelinePerformanceService {

    private final StringRedisTemplate redisTemplate;

    public void warmUpCache(List<String> keys) {
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (String key : keys) {
                byte[] keyBytes = key.getBytes(StandardCharsets.UTF_8);
                byte[] valueBytes = ("value-" + key).getBytes(StandardCharsets.UTF_8);

                connection.stringCommands().set(keyBytes, valueBytes);
            }

            return null;
        });
    }
}
```

### Batch Size

```text
Recommended:
500–1000 commands per pipeline batch
```

Bahut large pipeline Redis server par reply buffering memory pressure create kar sakti hai. [289]

---

## 5. Connection Pooling

Connection creation expensive operation hai. Pooling connections reuse karta hai.

```yaml
spring:
  data:
    redis:
      timeout: 1s
      connect-timeout: 1s
      lettuce:
        pool:
          enabled: true
          max-active: 32
          max-idle: 16
          min-idle: 4
          max-wait: 500ms
```

### Pool Sizing Rule

```text
Required Connections =
Average Redis Latency x Target Redis RPS
```

Example:

```text
Latency = 2 ms
Target RPS = 5000

Required = 0.002 x 5000 = 10 connections
```

Practical starting point:

```text
10–20 connections per instance
```

---

## 6. Slow Log

Slow Log Redis ke slow commands record karta hai.

### Configure Slow Log

```bash
CONFIG SET slowlog-log-slower-than 10000
CONFIG SET slowlog-max-len 256
```

Meaning:

```text
10,000 microseconds = 10 ms
```

### View Slow Log

```bash
SLOWLOG GET 10
```

### Reset Slow Log

```bash
SLOWLOG RESET
```

Slow Log threshold `slowlog-log-slower-than` se configure hota hai; lower threshold zyada slow commands capture karta hai. [285]

---

## 7. Latency Monitoring

### Redis CLI Latency Check

```bash
redis-cli --latency -h localhost -p 6379
```

### Latency Monitor

```bash
CONFIG SET latency-monitor-threshold 100
LATENCY HISTORY event
LATENCY LATEST
```

Redis latency monitoring slow commands, fork, AOF fsync, memory pressure aur network issues diagnose karne me help karta hai. [283]

---

## 8. Hot Key Problem

Hot key wo key hai jise bahut zyada requests hit karti hain.

```text
trending:post:501
```

Agar 100,000 requests/second sirf isi key par hain:

```text
Single Redis shard overloaded
CPU spike
Latency spike
```

### Detection

```bash
redis-cli --hotkeys
```

### Solutions

| Solution | Explanation |
|---|---|
| Local cache | Application memory me short TTL cache |
| Key sharding | `post:501:shard:0`, `post:501:shard:1` |
| Read replicas | Read traffic replicas par distribute karo |
| Client-side caching | Frequent reads locally cache karo |
| Rate limiting | Abusive traffic control karo |

---

## 9. Big Key Problem

Big key wo key hai jiska value bahut large ho.

```text
followers:user:101
10 million members
```

### Problems

```text
- Slow read/write
- High network transfer
- Redis blocking
- Memory spikes
- Cluster migration issue
```

### Detect Big Keys

```bash
redis-cli --bigkeys
```

### Memory Usage

```bash
MEMORY USAGE followers:user:101
```

### Solutions

| Problem | Solution |
|---|---|
| Large Set | Shard by hash/range |
| Large Hash | Split fields into multiple hashes |
| Large List | Trim old entries / chunking |
| Large JSON value | Smaller DTO / compression |
| Large Sorted Set | Time-based partitioning |

---

## 10. `KEYS` vs `SCAN`

### `KEYS` — Avoid

```bash
KEYS user:*
```

Problem:

```text
Poora keyspace scan
Redis block ho sakta hai
```

### `SCAN` — Safe

```bash
SCAN 0 MATCH user:* COUNT 500
```

### Spring Boot SCAN

```java
ScanOptions options = ScanOptions.scanOptions()
        .match("user-service:user:*")
        .count(500)
        .build();

try (Cursor<String> cursor = redisTemplate.scan(options)) {
    while (cursor.hasNext()) {
        String key = cursor.next();
        // Process key
    }
}
```

---

## 11. Client-Side Caching

Client-side caching me application apni local memory me Redis data cache karti hai. Redis server modified keys par invalidation messages bhejta hai. [284]

```text
Application Local Cache
        |
        +--> Redis
        |
        +--> Database
```

### Benefit

```text
Redis network call bhi skip ho sakta hai.
```

### Redis Support

Redis server-assisted client-side caching support karta hai; server track karta hai ki client ne kaunsi keys cache ki hain aur change hone par invalidation message bhejta hai. [284][293]

### Use When

- Same keys repeatedly read hoti hain
- Read-heavy workload hai
- Redis network latency matter karti hai
- Data change frequency controlled hai

### Caution

```text
Local cache memory limit rakho.
Invalidation handling verify karo.
Stale data tolerance define karo.
```

---

## 12. Redis Benchmarking

Redis `redis-benchmark` utility multiple clients simulate karke commands run karti hai. [282]

### Basic Benchmark

```bash
redis-benchmark -q -n 100000
```

### GET Benchmark

```bash
redis-benchmark -t GET -n 100000 -c 100
```

### SET Benchmark

```bash
redis-benchmark -t SET -n 100000 -c 100 -d 1024
```

### Custom Key Pattern

```bash
redis-benchmark \
  -t SET,GET \
  -n 100000 \
  -c 100 \
  -d 1024 \
  -r 1000000
```

### Important Metrics

| Metric | Meaning |
|---|---|
| Throughput | Requests per second |
| Latency | Response time |
| p99 latency | 99% requests is time me complete |
| Errors | Failed requests |

---

## 13. Persistence Performance Impact

| Persistence | Performance Impact |
|---|---|
| RDB | Fork overhead during snapshot |
| AOF `always` | High durability, high latency |
| AOF `everysec` | Balanced durability/performance |
| AOF `no` | Fast but risky |
| AOF rewrite | CPU/disk usage spike |

Production me `appendfsync everysec` generally durability aur performance ka balanced choice hota hai.

---

## 14. Performance Checklist

- [ ] `KEYS` use nahi ho raha.
- [ ] Large `HGETALL`, `SMEMBERS`, `LRANGE` avoid ho rahe hain.
- [ ] Pipelining bulk operations me use ho rahi hai.
- [ ] Connection pool tuned hai.
- [ ] Timeouts configured hain.
- [ ] Hot keys identify kiye hain.
- [ ] Big keys identify kiye hain.
- [ ] TTL sab cache keys par hai.
- [ ] Memory fragmentation monitor ho rahi hai.
- [ ] Slow log enabled hai.
- [ ] Latency monitoring enabled hai.
- [ ] Load testing/benchmarking done hai.

---

## 15. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| `KEYS *` production me | Redis block | `SCAN` |
| Large `HGETALL` | High latency | `HGET` / `HMGET` |
| One command per operation | Network overhead | Pipelining / batching |
| Huge pipeline | Memory pressure | 500–1000 batch |
| No pool sizing | Connection exhaustion | Load-based tuning |
| Ignoring slow log | Hidden latency issues | Regular slow log review |
| Large JSON caching | Network/memory overhead | Small DTO |
| Hot key ignore | Shard overload | Local cache/sharding |

---

## 16. Interview Questions

| Question | Answer |
|---|---|
| Redis performance optimize kaise karte hain? | Pipelining, pooling, small values, avoiding blocking commands, hot/big key control aur monitoring se. |
| Pipelining kya hai? | Multiple commands ko ek network round-trip me bhejna. |
| `KEYS` kyun avoid karte hain? | Ye poora keyspace scan karke Redis block kar sakta hai. |
| `SCAN` kaise kaam karta hai? | Cursor-based incremental iteration karta hai. |
| Hot key kya hai? | Bahut zyada traffic receive karne wali single key. |
| Big key kya hai? | Jiska value ya collection size bahut large ho. |
| Slow log kya hai? | Configured threshold se slow commands record karne wala Redis feature. |
| Client-side caching kya hai? | Application local memory me Redis data cache karna, server invalidation ke saath. |
| Redis benchmark kaise karte hain? | `redis-benchmark` utility se. |

---

## 17. Final Mental Model

```text
Fast Redis =
O(1) commands
+
Small values
+
Pipelining
+
Connection pooling
+
No blocking scans
+
Controlled hot/big keys
+
Monitoring
```

---

## Next Part

**Part 21: Redis Replication — Master-Replica Architecture — Complete Deep Dive**

Isme hum cover karenge:

- Replication kya hai
- Master-replica architecture
- Replication flow
- Read replicas
- Replication lag
- Read-your-writes problem
- Failover basics
- Replica promotion
- Spring Boot replication configuration
- Production topology
- Interview questions
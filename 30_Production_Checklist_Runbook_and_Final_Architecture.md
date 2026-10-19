# Part 30: Production Checklist, Runbook and Final Architecture — Complete Deep Dive

> Ye final part Redis + Spring Boot system ko production-ready banata hai. Isme hum complete checklist, incident runbook, capacity planning, disaster recovery aur final architecture cover karenge.

---

## 1. Production-Ready Redis Means

```text
Fast Redis
+ Safe Redis
+ Observable Redis
+ Resilient Redis
+ Recoverable Redis
+ Secure Redis
```

Ek production Redis system me sirf caching nahi hoti — TTL, memory limits, HA, monitoring, backups, security aur runbook sab defined hote hain.

---

## 2. Final Production Checklist

### Configuration

- [ ] Redis version pinned hai.
- [ ] `maxmemory` configured hai.
- [ ] `maxmemory-policy` workload ke hisaab se set hai.
- [ ] Har cache key ka TTL hai.
- [ ] TTL jitter use ho raha hai.
- [ ] Cache key prefix service-specific hai.
- [ ] JSON serializer use ho raha hai.
- [ ] Connection pool configured hai.
- [ ] Command timeout configured hai.
- [ ] Connect timeout configured hai.
- [ ] Graceful shutdown timeout set hai.

### High Availability

- [ ] Redis single point of failure nahi hai.
- [ ] Master-replica replication enabled hai.
- [ ] Sentinel ya managed Redis failover configured hai.
- [ ] Master aur replicas alag availability zones me hain.
- [ ] Failover testing done hai.
- [ ] Replication lag monitoring hai.
- [ ] Client Sentinel/Cluster discovery verified hai.

### Persistence and Backup

- [ ] RDB snapshot configured hai.
- [ ] AOF enabled hai.
- [ ] `appendfsync everysec` configured hai.
- [ ] Hybrid persistence enabled hai.
- [ ] Automated backups hain.
- [ ] Backups off-site/cross-region hain.
- [ ] Backup retention policy hai.
- [ ] Restore testing schedule me hai.
- [ ] Disaster recovery RTO/RPO defined hai.

### Security

- [ ] Redis publicly exposed nahi hai.
- [ ] Private subnet/VPC me hai.
- [ ] Firewall/security group configured hai.
- [ ] Authentication enabled hai.
- [ ] ACL least-privilege users hain.
- [ ] Default user restricted/disabled hai.
- [ ] TLS enabled hai.
- [ ] `FLUSHALL`, `FLUSHDB`, `CONFIG`, `DEBUG` restricted hain.
- [ ] Secrets secret manager me hain.
- [ ] Redis version patched hai.

### Monitoring

- [ ] Redis up/down alert hai.
- [ ] Memory usage dashboard hai.
- [ ] Fragmentation dashboard hai.
- [ ] Hit/miss ratio dashboard hai.
- [ ] Eviction alert hai.
- [ ] Latency dashboard hai.
- [ ] Slow log monitoring hai.
- [ ] Connected clients alert hai.
- [ ] Replication lag alert hai.
- [ ] Persistence failure alert hai.
- [ ] Application cache metrics hain.
- [ ] Grafana dashboards ready hain.
- [ ] On-call runbook ready hai.

### Application

- [ ] Cache-aside pattern implemented hai.
- [ ] Cache invalidation strategy hai.
- [ ] Redis failure fallback hai.
- [ ] Circuit breaker/resilience handling hai.
- [ ] Distributed locks me idempotency hai.
- [ ] Rate limiting implemented hai.
- [ ] Load testing done hai.
- [ ] Integration tests CI me hain.
- [ ] Testcontainers tests hain.
- [ ] Documentation updated hai.

---

## 3. Redis Runbook

Runbook ka matlab hai: incident aane par on-call engineer ko step-by-step pata ho ki kya check karna hai aur kaise fix karna hai.

### Runbook Template

```text
Incident:
Redis latency high / application slow

Impact:
API latency increased, DB load increased

First Checks:
1. Redis up hai?
2. Memory pressure hai?
3. Slow commands hain?
4. Evictions spike hue hain?
5. Replication lag hai?
6. Application connection pool exhausted hai?

Actions:
1. Metrics check karo
2. Slow log check karo
3. Hot/big keys identify karo
4. Redis memory/eviction policy review karo
5. Required rollback/scale action lo
```

---

## 4. Incident 1 — Redis Down

### Symptoms

```text
- /actuator/health DOWN
- Application errors
- Redis connection refused
- Cache miss spike
```

### Diagnosis

```bash
redis-cli -h <host> -p 6379 ping
kubectl get pods -n redis
kubectl logs redis-0 -n redis
```

### Actions

```text
1. Redis container/pod restart karo
2. Node/resource issue check karo
3. Application fallback verify karo
4. Sentinel failover verify karo
5. Incident timeline document karo
```

### Prevention

```text
- Health probes
- Automatic restart
- Sentinel/managed Redis
- DB fallback
- Alerts
```

---

## 5. Incident 2 — High Memory Usage

### Symptoms

```text
- used_memory near maxmemory
- Eviction spike
- Cache hit ratio drop
- Application latency increase
```

### Diagnosis

```bash
redis-cli INFO memory
redis-cli INFO stats
redis-cli --bigkeys
redis-cli MEMORY USAGE <key>
```

Important checks:

```text
used_memory
used_memory_rss
mem_fragmentation_ratio
maxmemory
evicted_keys
```

### Actions

```text
1. Eviction policy verify karo
2. TTL missing keys identify karo
3. Large keys identify karo
4. Unnecessary keys delete karo
5. maxmemory increase plan karo
6. Cluster sharding consider karo
```

### Prevention

```text
- Har cache key TTL
- TTL jitter
- Small DTO caching
- Eviction alerts
- Capacity planning
```

High memory incidents me `used_memory`, `maxmemory`, fragmentation ratio aur eviction count check karna pehla step hota hai. [421][422]

---

## 6. Incident 3 — High Latency

### Symptoms

```text
- Redis p99 latency spike
- API latency spike
- Slow log entries
- CPU spike
```

### Diagnosis

```bash
redis-cli SLOWLOG GET 25
redis-cli --latency -h <host> -p 6379
redis-cli INFO latencystats
redis-cli INFO memory
redis-cli INFO stats
```

### Common Causes

| Cause | Fix |
|---|---|
| `KEYS *` | `SCAN` use karo |
| Large `HGETALL` | `HGET`/`HMGET` use karo |
| Large values | DTO size reduce karo |
| Hot key | Local cache/sharding |
| AOF fsync pressure | `appendfsync` review |
| RDB fork | Snapshot schedule review |
| Connection pool exhaustion | Pool tuning |

Redis latency troubleshooting me slow log, network, memory pressure, blocking commands aur persistence operations check karne ko recommend kiya jata hai. [422][425]

---

## 7. Incident 4 — Replication Lag

### Symptoms

```text
- Replica stale data
- Read-your-writes issue
- Replication offset gap increasing
- Failover risk
```

### Diagnosis

```bash
redis-cli INFO replication
redis-cli -h <master> INFO stats
redis-cli -h <replica> INFO replication
```

### Check

```text
master_link_status
master_repl_offset
slave_repl_offset
master_last_io_seconds_ago
connected_slaves
```

### Actions

```text
1. Network latency check karo
2. Replica CPU/memory check karo
3. Heavy writes identify karo
4. Large keys identify karo
5. Replica scale/restart plan karo
6. Critical reads ko master par route karo
```

### Prevention

```text
- Replication lag alerts
- Multi-AZ topology
- Load testing
- Replica capacity monitoring
```

Replication lag ko master aur replica replication offset ke difference se quantify karte hain. [421][424]

---

## 8. Incident 5 — Cache Stampede

### Symptoms

```text
- Hot key expire
- DB load spike
- Multiple same DB queries
- API latency spike
```

### Diagnosis

```text
1. Kaunsi key expire hui?
2. Kitne concurrent requests aaye?
3. DB query rate check karo
4. Cache hit ratio drop check karo
```

### Actions

```text
1. Distributed lock lagao
2. `sync = true` enable karo
3. Stale-while-revalidate implement karo
4. TTL jitter add karo
5. Cache warm-up karo
```

### Prevention

```text
- Hot key monitoring
- Locking
- Probabilistic early refresh
- TTL jitter
```

---

## 9. Incident 6 — Accidental FLUSHALL

### Symptoms

```text
- Redis keys suddenly gone
- Cache hit ratio 0%
- DB load spike
- Application errors
```

### Immediate Actions

```text
1. Redis writes temporarily block karo
2. Backup availability check karo
3. Latest RDB/AOF backup identify karo
4. Isolated environment me restore test karo
5. Redis restore karo
6. Cache gradually warm-up karo
```

### Prevention

```text
- FLUSHALL ACL se deny karo
- rename-command/disable karo
- Backup automation
- Restore drills
- Change approval process
```

---

## 10. Capacity Planning

### Memory Capacity

```text
Required Redis Memory =
Average Value Size x Active Keys
+ Overhead
+ Fragmentation
+ Replication Buffers
+ Safety Margin
```

### Example

```text
Average cached profile = 2 KB
Active cached profiles = 1 million

Dataset memory = 2 KB x 1,000,000
              = 2 GB

Recommended maxmemory = 3 GB
Recommended machine RAM = 5–6 GB
```

### Connection Capacity

```text
Total Connections =
Application Instances x Max Pool Size
```

Example:

```text
20 instances x 32 connections
= 640 Redis connections
```

Redis server `maxclients` isse zyada hona chahiye.

### Throughput Capacity

```text
Required Redis RPS =
Application RPS x Redis Commands Per Request
```

Example:

```text
Application RPS = 2000
Redis commands per request = 3

Required Redis RPS = 6000
```

---

## 11. Scaling Strategy

| Problem | Scaling Solution |
|---|---|
| Read load high | Read replicas |
| Write load high | Redis Cluster / sharding |
| Memory full | Increase memory / Cluster |
| Hot key | Local cache / key sharding |
| Large dataset | Redis Cluster |
| High availability needed | Sentinel / managed Redis |
| Global users | Regional Redis / geo-replication |

---

## 12. Disaster Recovery Plan

### RTO and RPO

| Term | Meaning | Example |
|---|---|---|
| RTO | Recovery Time Objective | 5 minutes |
| RPO | Recovery Point Objective | 1 minute |

### DR Architecture

```text
Primary Region
     |
     +--> Redis Primary
     |
     +--> Replica
     |
     +--> RDB/AOF Backup
              |
              v
     Cross-Region Object Storage
              |
              v
     DR Region Restore
```

### DR Runbook

```text
1. Incident declare karo
2. Latest healthy backup identify karo
3. DR region me Redis provision karo
4. Backup restore karo
5. Application config update karo
6. Health checks verify karo
7. Cache warm-up karo
8. Traffic switch karo
9. Post-incident review karo
```

Backup restore ko isolated environment me validate karna aur production cutover se pehle health checks verify karna recommended hai. [426]

---

## 13. Final Architecture

```text
                                Internet
                                   |
                                   v
                            API Gateway
                                   |
        +--------------------------+--------------------------+
        |                          |                          |
+-------v--------+        +--------v--------+        +--------v--------+
| User Service   |        | Post Service    |        | Feed Service    |
+-------+--------+        +--------+--------+        +--------+--------+
        |                          |                          |
        +-----------+--------------+-----------+--------------+
                    |                          |
                    v                          v
             +-------------+           +-------------+
             |    Redis    |           |    Kafka    |
             +------+------+           +------+------+
                    |                         |
        +-----------+-----------+             |
        |           |           |             |
+-------v---+ +-----v-----+ +---v-------+ +---v--------+
| PostgreSQL| | Monitoring| | Backup    | | Consumers  |
+-----------+ +-----------+ +-----------+ +------------+
```

### Redis Responsibilities

```text
User profile cache
Followers/following Sets
Feed timeline Sorted Sets
Like counters
Trending leaderboard
Rate limiting
Refresh tokens
JWT blacklist
Online presence
Distributed locks
Cache invalidation events
```

---

## 14. Final Technology Stack

| Layer | Technology |
|---|---|
| Application | Java 17+ / Spring Boot |
| API | REST |
| Cache | Redis + Spring Data Redis |
| Database | PostgreSQL / MySQL |
| Messaging | Kafka |
| Locks | Redisson |
| Monitoring | Actuator + Micrometer + Prometheus + Grafana |
| Deployment | Docker / Kubernetes |
| Testing | JUnit, Mockito, Testcontainers, k6 |

---

## 15. Final Best Practices

```text
1. Redis ko cache/low-latency layer banao,
   source of truth mat banao.

2. Har cache key par TTL rakho.

3. TTL jitter use karo.

4. Cache key naming consistent rakho.

5. JSON serialization use karo.

6. Redis failure par DB fallback rakho.

7. Hot keys aur big keys monitor karo.

8. Replication lag monitor karo.

9. Backup restore regularly test karo.

10. Security ko afterthought mat banao.

11. Load testing production se pehle karo.

12. Runbook aur alerts ready rakho.
```

---

## 16. Interview Preparation

### Core Redis Questions

| Question | Key Answer |
|---|---|
| Redis kya hai? | In-memory data structure store used as cache, database, message broker and streaming engine. |
| Redis fast kyun hai? | In-memory storage, optimized data structures aur efficient command execution. |
| Redis vs MySQL? | Redis low-latency in-memory store hai; MySQL durable relational database hai. |
| Redis vs Kafka? | Redis low-latency operations/cache ke liye; Kafka durable event streaming ke liye. |
| Redis persistence kya hai? | RDB snapshots aur AOF logs se disk par data save karna. |
| Redis Cluster kya hai? | Data ko 16,384 hash slots me shard karne wala distributed Redis deployment. |

### Spring Boot Questions

| Question | Key Answer |
|---|---|
| Spring Data Redis kya hai? | Spring Boot me Redis integration provide karne wala project. |
| RedisTemplate kya hai? | Redis operations ka thread-safe high-level abstraction. |
| `@Cacheable` kya karta hai? | Method result cache karta hai aur cache hit par method skip karta hai. |
| Cache invalidation kaise karte hain? | TTL, write-through delete aur Kafka event-based invalidation se. |
| Redis failure handle kaise karte hain? | DB fallback, circuit breaker, timeouts aur graceful degradation se. |

### System Design Questions

| Question | Key Answer |
|---|---|
| Social media feed design? | Hybrid fan-out: normal users fan-out-on-write, celebrities fan-out-on-read. |
| Rate limiter design? | Redis fixed/sliding window ya token bucket with atomic Lua operations. |
| Distributed lock design? | `SET NX PX`, unique owner value, Lua unlock, watchdog aur idempotency. |
| Leaderboard design? | Redis Sorted Set with score, rank and pagination. |
| Session management design? | Short-lived JWT + Redis-backed revocable refresh tokens. |

---

## 17. Project Completion

Tumne complete Redis + Spring Boot roadmap cover kar liya:

```text
Part 1  -> Redis fundamentals
Part 2  -> Redis data types
Part 3  -> Commands, TTL, transactions, Lua
Part 4  -> Spring Data Redis setup
Part 5  -> Connection configuration
Part 6  -> RedisTemplate
Part 7  -> Serialization
Part 8  -> Spring Cache abstraction
Part 9  -> Cache annotations
Part 10 -> Redis cache configuration
Part 11 -> Cache invalidation
Part 12 -> Session management
Part 13 -> Distributed locks
Part 14 -> Rate limiting
Part 15 -> Leaderboards and counters
Part 16 -> Pub/Sub
Part 17 -> Redis Streams
Part 18 -> Persistence
Part 19 -> Memory management
Part 20 -> Performance optimization
Part 21 -> Replication
Part 22 -> Sentinel
Part 23 -> Redis Cluster
Part 24 -> High availability design
Part 25 -> Security
Part 26 -> Monitoring
Part 27 -> Docker/Kubernetes deployment
Part 28 -> Testing
Part 29 -> Social media project
Part 30 -> Production checklist and runbook
```

---

## 18. Final Mental Model

```text
Redis = Fast temporary data layer
Database = Permanent source of truth
Kafka = Durable event backbone
Monitoring = Early problem detection
Runbook = Fast incident recovery
Security = Trust boundary
Testing = Production confidence
```

---

## Congratulations

Tumhare paas ab ek complete, production-oriented **Redis + Spring Boot documentation repository** ban sakti hai.

Isme included hai:

- Redis fundamentals
- Spring Boot integration
- Caching patterns
- Distributed systems patterns
- Security
- HA and scaling
- Monitoring
- Deployment
- Testing
- Real-world project
- Production runbook
- Interview preparation
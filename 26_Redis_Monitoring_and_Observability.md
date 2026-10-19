# Part 26: Redis Monitoring and Observability — Complete Deep Dive

> Monitoring ka goal hai problem ko user notice karne se pehle detect karna. Redis monitoring me memory, latency, cache hit ratio, evictions, connections, replication aur persistence sab track karte hain.

---

## 1. Observability Kya Hai?

Observability ke 3 pillars:

| Pillar | Purpose |
|---|---|
| Metrics | Numbers se system health samajhna |
| Logs | Events aur errors trace karna |
| Traces | Request flow across services samajhna |

Redis ke liye primarily metrics aur logs important hote hain.

---

## 2. Redis INFO Command

```bash
INFO
```

Specific sections:

```bash
INFO server
INFO clients
INFO memory
INFO persistence
INFO stats
INFO replication
INFO cpu
INFO keyspace
INFO latencystats
```

Redis `INFO` command server, clients, memory, persistence, stats, replication aur latency statistics return karta hai. [363][374]

---

## 3. Memory Metrics

```bash
INFO memory
```

### Important Metrics

| Metric | Meaning | Alert Condition |
|---|---|---|
| `used_memory` | Redis dataset + internal structures | Rapid growth |
| `used_memory_rss` | OS-level actual RAM usage | `used_memory` se bahut zyada |
| `used_memory_peak` | Maximum memory used | Repeated high peaks |
| `maxmemory` | Configured limit | Near limit |
| `mem_fragmentation_ratio` | RSS / used memory | Above 1.5–2.0 |
| `mem_allocator` | Memory allocator | Usually jemalloc |

Redis `MEMORY STATS` command allocated memory, replication backlog, client overheads aur cluster links jaise memory metrics return karta hai. [365]

### Memory Usage Percentage

```text
Memory Usage % =
used_memory / maxmemory x 100
```

### Alert Examples

```text
Memory usage > 80% -> Warning
Memory usage > 90% -> Critical
Fragmentation ratio > 1.5 -> Warning
Fragmentation ratio > 2.0 -> Critical
```

---

## 4. Cache Hit and Miss Ratio

```bash
INFO stats
```

Important fields:

```text
keyspace_hits
keyspace_misses
evicted_keys
expired_keys
```

### Hit Ratio Formula

\[
\text{Hit Ratio} =
\frac{\text{keyspace_hits}}
{\text{keyspace_hits} + \text{keyspace_misses}}
\]

### Example

```text
Hits = 900
Misses = 100

Hit Ratio = 900 / 1000 = 90%
```

### Healthy Target

| Workload | Target Hit Ratio |
|---|---|
| Product catalog cache | 85–95% |
| User profile cache | 80–95% |
| Feed cache | 70–90% |
| Config cache | 90%+ |

Low hit ratio ka matlab:

```text
- TTL bahut short hai
- Cache key design galat hai
- Data change frequently ho raha hai
- Cache warm-up missing hai
- Wrong data cache ho raha hai
```

---

## 5. Eviction Metrics

```bash
INFO stats
```

```text
evicted_keys
expired_keys
```

### Meaning

| Metric | Meaning |
|---|---|
| `expired_keys` | TTL expire hone par delete hui keys |
| `evicted_keys` | Memory limit ke kaaran forcibly removed keys |

### Alert

```text
Sudden eviction spike -> Memory pressure
High eviction rate -> maxmemory/eviction policy review karo
```

Grafana Redis dashboards commonly operations/sec, connected clients, keys, expired/evicted keys, memory aur network metrics track karte hain. [366][368]

---

## 6. Client and Connection Metrics

```bash
INFO clients
```

Important fields:

```text
connected_clients
blocked_clients
maxclients
```

| Metric | Meaning | Alert |
|---|---|---|
| `connected_clients` | Active Redis connections | Near `maxclients` |
| `blocked_clients` | Blocking command wait kar rahe clients | Sudden spike |
| `maxclients` | Maximum allowed clients | Capacity limit |

### Connection Pool Metrics

Spring Boot/Micrometer se track karo:

```text
Redis pool active connections
Redis pool idle connections
Redis pool pending requests
Redis pool timeout count
```

---

## 7. Latency Metrics

```bash
INFO latencystats
```

Redis command-type-wise latency percentile distribution deta hai. [363]

### Important Latency Targets

| Metric | Target |
|---|---|
| Average latency | < 1 ms |
| p95 latency | < 5 ms |
| p99 latency | < 10 ms |
| Slow command threshold | 10 ms se zyada investigate |

### Slow Log

```bash
CONFIG SET slowlog-log-slower-than 10000
SLOWLOG GET 10
```

Slow log se pata chalta hai kaunse commands latency create kar rahe hain.

---

## 8. Replication Metrics

```bash
INFO replication
```

Master side:

```text
role:master
connected_slaves:2
master_repl_offset:123456789
```

Replica side:

```text
role:slave
master_link_status:up
slave_repl_offset:123456700
lag:0
```

### Replication Lag

```text
Lag =
master_repl_offset - replica_repl_offset
```

High lag ka matlab:

```text
Replica stale data de raha hai
Failover risk
Read-your-writes issue
```

Redis replication me replica lag aur master/replica offset gap monitor karna important hai. [375]

### Alerts

```text
master_link_status != up -> Critical
Replication lag increasing -> Warning/Critical
connected_slaves drop -> Critical
```

---

## 9. Persistence Metrics

```bash
INFO persistence
```

Important fields:

```text
rdb_last_save_time
rdb_changes_since_last_save
aof_enabled
aof_last_write_status
aof_last_bgrewrite_status
aof_current_size
```

| Metric | Alert |
|---|---|
| `aof_last_write_status` | Not `ok` -> Critical |
| `rdb_last_bgsave_status` | Not `ok` -> Critical |
| `aof_current_size` | Rapid growth -> Warning |
| `rdb_changes_since_last_save` | Very high -> Backup overdue |

---

## 10. Spring Boot Actuator Setup

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Spring Boot Actuator Prometheus metrics ko `/actuator/prometheus` endpoint par expose karta hai jab `micrometer-registry-prometheus` dependency hoti hai. [362]

### Configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  health:
    redis:
      enabled: true
  metrics:
    tags:
      application: user-service
```

### Health Endpoint

```bash
curl http://localhost:8080/actuator/health
```

Response:

```json
{
  "status": "UP",
  "components": {
    "redis": {
      "status": "UP"
    }
  }
}
```

Spring Boot Actuator ka `redis` health indicator Redis server up hai ya nahi check karta hai. [362][369]

---

## 11. Prometheus Configuration

### `prometheus.yml`

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "spring-boot-user-service"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets:
          - "user-service:8080"

  - job_name: "redis-exporter"
    static_configs:
      - targets:
          - "redis-exporter:9121"
```

Prometheus individual application instances ko scrape/poll karta hai. [362]

---

## 12. Redis Exporter

Redis exporter Redis ke `INFO` command output ko Prometheus metrics me expose karta hai. [372]

### Docker Compose

```yaml
services:
  redis:
    image: redis:7.4-alpine
    ports:
      - "6379:6379"

  redis-exporter:
    image: oliver006/redis_exporter:latest
    container_name: redis-exporter
    environment:
      REDIS_ADDR: "redis://redis:6379"
    ports:
      - "9121:9121"
    depends_on:
      - redis
```

### Prometheus Metrics

```text
redis_up
redis_connected_clients
redis_memory_used_bytes
redis_memory_max_bytes
redis_keyspace_hits_total
redis_keyspace_misses_total
redis_evicted_keys_total
redis_commands_processed_total
redis_slowlog_length
```

---

## 13. Grafana Dashboard Queries

### Redis Up

```promql
redis_up
```

### Memory Usage Percentage

```promql
redis_memory_used_bytes / redis_memory_max_bytes * 100
```

### Cache Hit Ratio

```promql
rate(redis_keyspace_hits_total[5m]) /
(
  rate(redis_keyspace_hits_total[5m])
  +
  rate(redis_keyspace_misses_total[5m])
) * 100
```

### Eviction Rate

```promql
rate(redis_evicted_keys_total[5m])
```

### Connected Clients

```promql
redis_connected_clients
```

### Operations Per Second

```promql
rate(redis_commands_processed_total[5m])
```

Grafana Redis dashboards me commonly hit rate, memory usage, connected clients, eviction rate aur operations throughput panels hote hain. [368][370]

---

## 14. Grafana Dashboard Panels

| Panel | Query | Alert |
|---|---|---|
| Redis Availability | `redis_up` | Value 0 |
| Memory Usage | `redis_memory_used_bytes / redis_memory_max_bytes * 100` | > 85% |
| Fragmentation | `redis_memory_used_rss_bytes / redis_memory_used_bytes` | > 1.5 |
| Cache Hit Ratio | Hit formula | < 70% |
| Evictions | `rate(redis_evicted_keys_total[5m])` | Sudden spike |
| Connected Clients | `redis_connected_clients` | Near maxclients |
| Commands/sec | `rate(redis_commands_processed_total[5m])` | Abnormal drop/spike |
| Replication Lag | `redis_replication_lag_seconds` | Increasing |

---

## 15. Alerting Rules

### Prometheus Alert Rules

```yaml
groups:
  - name: redis-alerts
    rules:
      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis is down"

      - alert: RedisHighMemoryUsage
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage above 85%"

      - alert: RedisHighEvictions
        expr: rate(redis_evicted_keys_total[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Redis eviction rate"

      - alert: RedisLowCacheHitRatio
        expr:
          rate(redis_keyspace_hits_total[5m]) /
          (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) < 0.7
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Redis cache hit ratio below 70%"

      - alert: RedisReplicationDown
        expr: redis_master_link_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis replica lost connection to master"
```

---

## 16. Application-Level Metrics

Spring Boot me custom cache metrics bhi add kar sakte ho.

```java
package com.example.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class CacheMetricsService {

    private final MeterRegistry meterRegistry;

    public void recordCacheHit(String cacheName) {
        meterRegistry
                .counter("cache_hit_total", "cache", cacheName)
                .increment();
    }

    public void recordCacheMiss(String cacheName) {
        meterRegistry
                .counter("cache_miss_total", "cache", cacheName)
                .increment();
    }

    public void recordRedisFailure(String operation) {
        meterRegistry
                .counter("redis_failure_total", "operation", operation)
                .increment();
    }
}
```

---

## 17. Production Observability Checklist

- [ ] Redis up/down alert hai.
- [ ] Memory usage dashboard hai.
- [ ] Fragmentation dashboard hai.
- [ ] Hit/miss ratio dashboard hai.
- [ ] Eviction alert hai.
- [ ] Connected clients alert hai.
- [ ] Latency monitoring hai.
- [ ] Slow log monitoring hai.
- [ ] Replication lag alert hai.
- [ ] Persistence failure alert hai.
- [ ] Application cache metrics hain.
- [ ] Redis failure counter hai.
- [ ] Grafana dashboard production me available hai.
- [ ] On-call runbook ready hai.

---

## 18. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Sirf Redis up/down monitor | Memory/latency issues late detect | Full metrics monitoring |
| Evictions ignore | Cache suddenly empty | Eviction alerts |
| Hit ratio ignore | DB load high | Hit ratio dashboard |
| Replication lag ignore | Stale reads/failover risk | Lag alerts |
| No slow log | Hidden slow commands | Slow log review |
| No dashboard | Incident debugging slow | Grafana dashboards |
| No alert runbook | On-call confusion | Clear remediation steps |

---

## 19. Interview Questions

| Question | Answer |
|---|---|
| Redis monitoring me kaunse metrics important hain? | Memory, latency, hit ratio, evictions, clients, replication aur persistence. |
| Cache hit ratio kaise calculate hota hai? | Hits / (Hits + Misses). |
| Eviction kya hai? | Memory limit reach hone par keys forcibly remove hona. |
| Replication lag kya hai? | Master write aur replica apply hone ke beech ka delay. |
| Slow log kya hai? | Configured threshold se slow commands record karne wala Redis feature. |
| Spring Boot Actuator Redis health check kya karta hai? | Redis server reachable/up hai ya nahi check karta hai. |
| Prometheus Redis metrics kaise collect karta hai? | Redis exporter ya Micrometer endpoint scrape karke. |
| Grafana me Redis dashboard me kya show karte hain? | Memory, hit ratio, clients, evictions, throughput aur replication. |

---

## 20. Final Mental Model

```text
Redis health =
Up/Down
+
Memory
+
Latency
+
Hit ratio
+
Evictions
+
Connections
+
Replication
+
Persistence
```

---

## Next Part

**Part 27: Redis Deployment with Docker and Kubernetes — Complete Deep Dive**

Isme hum cover karenge:

- Docker Redis deployment
- Docker Compose production setup
- Persistent volumes
- Health checks
- Resource limits
- Kubernetes Redis deployment
- StatefulSet
- PersistentVolumeClaim
- Kubernetes probes
- Helm chart overview
-
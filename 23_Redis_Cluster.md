# Part 23: Redis Cluster — Sharding, Hash Slots and Horizontal Scaling — Complete Deep Dive

> Redis Cluster horizontal scaling aur automatic failover dono deta hai. Data multiple master nodes par shard hota hai, aur har master ke replicas high availability ke liye hote hain.

---

## 1. Redis Cluster Kya Hai?

Redis Cluster multiple Redis nodes ka group hai jisme data automatically distribute hota hai.

```text
Redis Cluster
     |
     +--> Master 1 + Replica 1
     |
     +--> Master 2 + Replica 2
     |
     +--> Master 3 + Replica 3
```

### Main Benefits

| Benefit | Explanation |
|---|---|
| Horizontal scaling | Data multiple nodes me split hota hai |
| High availability | Master fail hone par replica promote hota hai |
| Automatic sharding | Keys hash slots me distribute hoti hain |
| Large dataset support | Single node memory limit cross kar sakte ho |
| Better throughput | Writes/reads multiple masters par distribute hote hain |

---

## 2. Hash Slots

Redis Cluster keyspace ko **16,384 hash slots** me divide karta hai.

```text
Slot 0
Slot 1
Slot 2
...
Slot 16383
```

### Slot Calculation

\[
\text{Slot} = \text{CRC16(key)} \mod 16384
\]

Example:

```text
key = user:101
CRC16(user:101) mod 16384 = 12345
```

To `user:101` slot `12345` me jayega.

Redis Cluster keys ko directly nodes par map nahi karta; pehle 16,384 slots me map karta hai, phir slot ranges nodes ko assign hote hain. [326][329]

---

## 3. Cluster Topology

```text
Master 1: Slots 0-5460
Master 2: Slots 5461-10922
Master 3: Slots 10923-16383
```

### With Replicas

```text
Master 1 -> Replica 1
Master 2 -> Replica 2
Master 3 -> Replica 3
```

### Architecture

```text
+------------+      +------------+      +------------+
| Master 1   |      | Master 2   |      | Master 3   |
| Slots 0-   |      | Slots      |      | Slots      |
| 5460       |      | 5461-10922 |      | 10923-16383|
+-----+------+      +-----+------+      +-----+------+
      |                   |                   |
+-----v------+      +-----v------+      +-----v------+
| Replica 1  |      | Replica 2  |      | Replica 3  |
+------------+      +------------+      +------------+
```

---

## 4. Minimum Cluster Size

Production me minimum recommended topology:

```text
3 Masters
3 Replicas
= 6 Nodes
```

### Why 3 Masters?

```text
1. Fault tolerance
2. Majority quorum
3. Failover capability
```

Agar 2 masters hain aur ek down ho jaye, to majority maintain karna difficult ho jata hai.

---

## 5. Key to Slot Mapping

### Single Key

```text
user:101 -> CRC16 -> Slot -> Master Node
```

### Example Keys

```text
user:101 -> Slot 12345 -> Master 2
user:102 -> Slot 5432  -> Master 1
post:201 -> Slot 9876  -> Master 3
```

---

## 6. Multi-Key Operations Problem

Cluster me multi-key operations tabhi kaam karte hain jab saari keys **same hash slot** me hon.

### Problem

```bash
MGET user:101 user:102
```

Agar dono keys different slots me hain:

```text
CROSSSLOT Keys in request don't hash to the same slot
```

Redis Cluster me multi-key commands ko saari keys same hash slot me honi chahiye, warna cross-slot error aata hai. [323][324]

---

## 7. Hash Tags

Hash tag key ke `{}` part ko hash karne ke liye use hota hai.

### Without Hash Tag

```text
user:101:profile -> Slot A
user:101:followers -> Slot B
```

### With Hash Tag

```text
{user:101}:profile -> Same Slot
{user:101}:followers -> Same Slot
```

Redis hash tags `{}` ke andar wale part ko hash karta hai, jisse related keys same slot me co-locate ho jati hain. [322][332]

### Example

```bash
SET {user:101}:profile "{...}"
SADD {user:101}:followers 201 202
MGET {user:101}:profile {user:101}:followers
```

### Important Rule

```text
Hash tag sirf related keys ke liye use karo.
Bahut bada hash tag ek shard ko hot bana sakta hai.
```

---

## 8. Hash Tag Best Practices

### Good

```text
{user:101}:profile
{user:101}:followers
{user:101}:feed
```

### Bad

```text
{user}:101:profile
{user}:102:profile
```

Problem:

```text
Saare users ek hi slot me chale jayenge
= Hot shard
```

Hash tags se related keys ko co-locate karo, lekin overly broad tag use karne se hot shard ban sakta hai. [330][332]

---

## 9. Spring Boot Cluster Configuration

### YAML Configuration

```yaml
spring:
  data:
    redis:
      password: ${REDIS_PASSWORD}
      cluster:
        nodes:
          - redis-node-1:6379
          - redis-node-2:6379
          - redis-node-3:6379
          - redis-node-4:6379
          - redis-node-5:6379
          - redis-node-6:6379
        max-redirects: 3
      lettuce:
        cluster:
          refresh:
            adaptive: true
            period: 30s
```

Spring Boot Lettuce cluster nodes ko accept karta hai, cluster topology discover karta hai aur `MOVED`/`ASK` redirects handle karta hai. [328][336]

### Properties

| Property | Meaning |
|---|---|
| `cluster.nodes` | Initial cluster node addresses |
| `max-redirects` | Maximum MOVED/ASK redirects |
| `lettuce.cluster.refresh.adaptive` | Topology changes par adaptive refresh |
| `lettuce.cluster.refresh.period` | Periodic topology refresh |

---

## 10. Java Cluster Configuration

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisClusterConfiguration;
import org.springframework.data.redis.connection.RedisPassword;
import org.springframework.data.redis.connection.lettuce.LettuceClientConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;

import java.time.Duration;
import java.util.List;

@Configuration
public class RedisClusterConfig {

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisClusterConfiguration clusterConfig =
                new RedisClusterConfiguration(
                        List.of(
                                "redis-node-1:6379",
                                "redis-node-2:6379",
                                "redis-node-3:6379",
                                "redis-node-4:6379",
                                "redis-node-5:6379",
                                "redis-node-6:6379"
                        )
                );

        clusterConfig.setPassword(
                RedisPassword.of(System.getenv("REDIS_PASSWORD"))
        );

        clusterConfig.setMaxRedirects(3);

        LettuceClientConfiguration clientConfig =
                LettuceClientConfiguration.builder()
                        .commandTimeout(Duration.ofSeconds(1))
                        .build();

        return new LettuceConnectionFactory(clusterConfig, clientConfig);
    }
}
```

Spring Data Redis cluster mode me automatic sharding karta hai aur multi-key commands ko same slot requirement enforce karta hai. [323][325]

---

## 11. Cluster Commands

### Cluster Info

```bash
CLUSTER INFO
```

### Cluster Nodes

```bash
CLUSTER NODES
```

### Cluster Slots

```bash
CLUSTER SLOTS
```

### Check Key Slot

```bash
CLUSTER KEYSLOT user:101
```

### Check Hash Tag Slot

```bash
CLUSTER KEYSLOT {user:101}:profile
CLUSTER KEYSLOT {user:101}:followers
```

Dono same slot return karenge.

---

## 12. MOVED and ASK Redirects

### MOVED Redirect

```text
Client -> Node 1
Node 1 -> Key belongs to Node 2
Client -> Node 2
```

Response:

```text
MOVED 12345 redis-node-2:6379
```

### ASK Redirect

Resharding ke dauran temporary redirect:

```text
ASK 12345 redis-node-2:6379
```

### Client Behavior

```text
MOVED -> Permanent slot ownership change
ASK   -> Temporary redirect during migration
```

---

## 13. Cluster Failover

### Master Failure

```text
Master 2 down
     |
     v
Cluster detects failure
     |
     v
Replica 2 promoted
     |
     v
Replica 2 becomes Master 2
     |
     v
Hash slots remain served
```

### Failover Flow

```text
1. Master unreachable
2. Majority masters agree
3. Replica promoted
4. Slot ownership updated
5. Clients redirected to new master
```

---

## 14. Cluster vs Sentinel

| Feature | Sentinel | Cluster |
|---|---|---|
| Main purpose | High availability | Sharding + high availability |
| Data distribution | No | Yes |
| Dataset size | Single master limit | Distributed across masters |
| Multi-key operations | Easy | Same slot required |
| Horizontal scaling | No | Yes |
| Complexity | Medium | High |
| Best for | Moderate dataset | Large dataset / high throughput |

---

## 15. Cluster Use Cases

| Use Case | Cluster Suitable? |
|---|---|
| Redis cache with huge dataset | Yes |
| High write throughput | Yes |
| Large leaderboard | Yes |
| Session store with moderate data | Usually Sentinel enough |
| Multi-key transactions-heavy workload | Careful design required |
| Simple small microservice cache | Sentinel / standalone enough |

---

## 16. Production Checklist

- [ ] Minimum 3 masters + 3 replicas.
- [ ] Nodes alag availability zones me hain.
- [ ] `max-redirects` configured hai.
- [ ] Topology refresh enabled hai.
- [ ] Hash tags carefully use kiye hain.
- [ ] Multi-key operations audited hain.
- [ ] Hot shard monitoring hai.
- [ ] Cluster resharding tested hai.
- [ ] Failover testing done hai.
- [ ] Persistence aur backup configured hai.

---

## 17. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Broad hash tag | Hot shard | Entity-specific hash tag |
| Cross-slot multi-key command | `CROSSSLOT` error | Hash tags / separate commands |
| 2-master cluster | Weak failover | Minimum 3 masters |
| No topology refresh | Stale cluster map | Adaptive refresh |
| No resharding test | Production migration issue | Test resharding |
| Cluster ko Sentinel replacement samajhna | Design mismatch | Requirement ke hisaab se choose karo |

---

## 18. Interview Questions

| Question | Answer |
|---|---|
| Redis Cluster kya hai? | Data ko multiple master nodes me shard karne wala distributed Redis deployment. |
| Hash slot kya hai? | Redis Cluster me 16,384 logical partitions me se ek. |
| Key ka slot kaise calculate hota hai? | `CRC16(key) mod 16384`. |
| Hash tag kya hai? | Key ke `{}` part ko hash karke related keys same slot me rakhna. |
| `MOVED` redirect kya hai? | Jab key dusre node ke slot me ho, client ko naye node par bhejna. |
| `ASK` redirect kya hai? | Resharding ke dauran temporary redirect. |
| Cluster me multi-key operation kab fail hota hai? | Jab keys different hash slots me hon. |
| Cluster aur Sentinel me difference? | Cluster sharding + HA deta hai; Sentinel sirf HA deta hai. |

---

## 19. Final Mental Model

```text
Key
 -> CRC16
    -> Hash Slot
       -> Master Node

Hash tag:
{user:101}:profile
{user:101}:followers
     |
     +--> Same slot
     |
     +--> Multi-key operations possible
```

---

## Next Part

**Part 24: Redis High Availability Design — Standalone, Replication, Sentinel, Cluster and Managed Redis — Complete Deep Dive**

Isme hum cover karenge:

- HA kya hai
- Standalone Redis
- Master-replica
- Sentinel
- Cluster
- Managed Redis options
- Disaster recovery
- Backup and restore
- Failure scenarios
- HA topology selection
- Production checklist
- Interview questions
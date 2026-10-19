# Part 21: Redis Replication — Master-Replica Architecture — Complete Deep Dive

> Redis replication data redundancy aur read scaling deta hai. Master writes handle karta hai, replicas data ki copy rakhte hain aur read traffic serve kar sakte hain. Lekin replication asynchronous hoti hai, isliye replication lag aur read-your-writes consistency samajhna zaroori hai.

---

## 1. Replication Kya Hai?

Replication ka matlab hai ek Redis master ka data dusre Redis replicas par copy rakhna.

```text
Master
  |
  +--> Replica 1
  |
  +--> Replica 2
```

Master writes accept karta hai aur har change replicas ko stream karta hai; replicas data copy maintain karte hain. [302]

### Basic Flow

```text
Client -> SET user:101:name "Rahul"
        |
        v
Master
        |
        +--> Replication stream
                |
                +--> Replica 1
                |
                +--> Replica 2
```

---

## 2. Master-Replica Architecture

```text
                 +----------------+
                 |     Master     |
                 |  Writes + Reads|
                 +--------+-------+
                          |
          +---------------+---------------+
          |                               |
+---------v---------+           +---------v---------+
|     Replica 1     |           |     Replica 2     |
|     Reads         |           |     Reads         |
+-------------------+           +-------------------+
```

### Responsibilities

| Node | Responsibility |
|---|---|
| Master | Writes, critical reads, replication source |
| Replica | Read traffic, data redundancy, failover candidate |

Redis replication data redundancy aur read scaling provide karta hai: master writes handle karta hai, replicas reads serve karte hain aur failover capability dete hain. [308]

---

## 3. Replication Setup

### Master Configuration

```conf
# redis-master.conf
port 6379
requirepass strong-password
appendonly yes
appendfsync everysec
```

### Replica Configuration

```conf
# redis-replica.conf
port 6380
requirepass strong-password
replicaof 127.0.0.1 6379
masterauth strong-password
replica-read-only yes
```

`replicaof` command Redis server ko kisi master ka replica banata hai. [296][297]

### Verify Replication

Master par:

```bash
INFO replication
```

Expected:

```text
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6380,state=online
slave1:ip=127.0.0.1,port=6381,state=online
```

Replica par:

```bash
INFO replication
```

Expected:

```text
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
```

---

## 4. Docker Compose Replication

```yaml
services:
  redis-master:
    image: redis:7.4-alpine
    container_name: redis-master
    command: >
      redis-server
      --requirepass strong-password
      --appendonly yes
      --appendfsync everysec
    ports:
      - "6379:6379"

  redis-replica-1:
    image: redis:7.4-alpine
    container_name: redis-replica-1
    command: >
      redis-server
      --requirepass strong-password
      --masterauth strong-password
      --replicaof redis-master 6379
      --replica-read-only yes
    depends_on:
      - redis-master
    ports:
      - "6380:6379"

  redis-replica-2:
    image: redis:7.4-alpine
    container_name: redis-replica-2
    command: >
      redis-server
      --requirepass strong-password
      --masterauth strong-password
      --replicaof redis-master 6379
      --replica-read-only yes
    depends_on:
      - redis-master
    ports:
      - "6381:6379"
```

---

## 5. Replication Flow

### Initial Sync

```text
1. Replica master se connect karta hai
2. Replica PSYNC request bhejta hai
3. Master RDB snapshot generate karta hai
4. Master snapshot replica ko bhejta hai
5. Replica snapshot load karta hai
6. Master buffered commands bhejta hai
7. Replica commands apply karta hai
8. Continuous replication start
```

### Ongoing Sync

```text
Master write
     |
     v
Replication backlog
     |
     v
Replica command stream
     |
     v
Replica applies change
```

---

## 6. Replication Lag

Replication lag wo delay hai jo master write aur replica apply hone ke beech hota hai.

```text
Master write at T1
Replica applies at T2

Replication lag = T2 - T1
```

Replication lag bytes me measure hota hai: master replication offset aur replica replication offset ka difference. [298]

### Example

```text
Master:
SET user:101:name "Rahul Sharma"

Replica after 200 ms:
GET user:101:name -> "Rahul"

After replication completes:
GET user:101:name -> "Rahul Sharma"
```

### Causes of Lag

| Cause | Explanation |
|---|---|
| Heavy write traffic | Master bahut zyada writes kar raha hai |
| Slow network | Master-replica network slow |
| Replica overloaded | Replica reads/CPU saturated |
| Large values | Bade payloads replicate hone me time lagta hai |
| Full resync | Replica reconnect/rebuild ho raha hai |

---

## 7. Read-Your-Writes Problem

Read-your-writes consistency ka matlab hai: user ne jo data write kiya, usi user ko next read me updated data milna chahiye. [301]

### Problem Example

```text
1. User profile update on Master
2. Application reads from Replica
3. Replica lag hai
4. User ko old profile dikhta hai
```

### Solution Options

| Solution | Explanation |
|---|---|
| Read from master | Critical reads master se karo |
| Session affinity | Same user ke reads temporarily same node par route karo |
| Wait for replication | Advanced: replication offset check |
| Short TTL cache | Updated data ko cache me rakho |
| Accept eventual consistency | Non-critical reads ke liye |

Agar application ko read-your-write consistency chahiye, to reads bhi primary/master endpoint se karni chahiye. [300]

---

## 8. Read Scaling

### Read Traffic Distribution

```text
Write requests -> Master
Read requests  -> Replicas
```

### Example

```text
80% reads -> 2 replicas
20% writes -> master
```

### Important

```text
Replicas read load reduce karte hain,
lekin write load reduce nahi karte.
```

---

## 9. Spring Boot Master-Replica Configuration

Spring Data Redis Lettuce ke through writes master par aur reads replicas par bhej sakta hai. [295]

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisPassword;
import org.springframework.data.redis.connection.RedisStaticMasterReplicaConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceClientConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.connection.lettuce.ReadFrom;

@Configuration
public class MasterReplicaRedisConfig {

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisStaticMasterReplicaConfiguration serverConfig =
                new RedisStaticMasterReplicaConfiguration(
                        "redis-master",
                        6379
                );

        serverConfig.addNode("redis-replica-1", 6379);
        serverConfig.addNode("redis-replica-2", 6379);

        serverConfig.setPassword(
                RedisPassword.of(System.getenv("REDIS_PASSWORD"))
        );

        LettuceClientConfiguration clientConfig =
                LettuceClientConfiguration.builder()
                        .readFrom(ReadFrom.REPLICA_PREFERRED)
                        .build();

        return new LettuceConnectionFactory(serverConfig, clientConfig);
    }
}
```

### `ReadFrom` Strategies

| Strategy | Meaning |
|---|---|
| `MASTER` | Always read from master |
| `REPLICA` | Always read from replica |
| `REPLICA_PREFERRED` | Replica preferred, fallback master |
| `MASTER_PREFERRED` | Master preferred, fallback replica |
| `ANY` | Any available node |
| `ANY_REPLICA` | Any available replica |

Lettuce master-replica topology dynamic ya static dono tarike se discover ho sakti hai. [307]

---

## 10. Read Strategy Selection

| Requirement | Recommended Strategy |
|---|---|
| Strong read-your-writes | `MASTER` |
| Read scaling | `REPLICA_PREFERRED` |
| Maximum availability | `ANY` |
| Read replicas preferred but master fallback | `REPLICA_PREFERRED` |
| Master preferred but replica fallback | `MASTER_PREFERRED` |

---

## 11. Failover Basics

### Manual Failover

```text
Master down
     |
     v
Promote replica:
REPLICAOF NO ONE
     |
     v
Replica becomes new master
```

`REPLICAOF NO ONE` replication band karke replica ko master bana deta hai. [296]

### Automatic Failover

Manual failover production me risky hota hai. Automatic failover ke liye **Redis Sentinel** use karte hain.

```text
Sentinel detects master failure
        |
        v
Promotes best replica
        |
        v
Clients discover new master
```

Sentinel failover me selected replica ko `REPLICAOF NO ONE` command bhejkar master promote karta hai. [299]

---

## 12. Replication Monitoring

```bash
INFO replication
```

Important metrics:

| Metric | Meaning |
|---|---|
| `master_repl_offset` | Master replication offset |
| `slave_repl_offset` | Replica replication offset |
| `master_link_status` | Replica-master connection status |
| `master_last_io_seconds_ago` | Last replication communication time |
| `connected_slaves` | Connected replica count |

### Lag Calculation

```text
Replication Lag =
master_repl_offset - replica_repl_offset
```

---

## 13. Production Replication Checklist

- [ ] Master aur replicas different availability zones me hain.
- [ ] Replica read-only hai.
- [ ] `masterauth` configured hai.
- [ ] Replication lag monitoring hai.
- [ ] Alerts configured hain.
- [ ] Failover strategy defined hai.
- [ ] Read-your-writes requirement clear hai.
- [ ] Backup aur persistence enabled hai.
- [ ] Network latency tested hai.
- [ ] Redis client read strategy tuned hai.

---

## 14. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Replica se critical read | Stale data possible | Master read |
| Replication lag ignore karna | Read-your-writes issue | Lag monitoring |
| Manual failover only | Slow recovery | Sentinel/managed Redis |
| Replica ko writable banana | Data divergence | `replica-read-only yes` |
| Master aur replica same AZ | Zone failure risk | Multi-AZ topology |
| No replication monitoring | Silent replication break | Alerts |

---

## 15. Interview Questions

| Question | Answer |
|---|---|
| Redis replication kya hai? | Master ka data replicas par copy rakhna. |
| Master aur replica me difference? | Master writes handle karta hai; replica reads aur redundancy handle karta hai. |
| Replication asynchronous kyun hoti hai? | Performance ke liye; master write ko replica acknowledgement ka wait nahi karta. |
| Replication lag kya hai? | Master write aur replica apply hone ke beech ka delay. |
| Read-your-writes problem kya hai? | Replica lag ki wajah se user ko apna recent write purana dikhna. |
| Read replicas kab use karte hain? | Read-heavy workload me master load distribute karne ke liye. |
| Manual failover kaise karte hain? | Replica par `REPLICAOF NO ONE` run karke. |
| Automatic failover ke liye kya use karte hain? | Redis Sentinel ya managed Redis service. |

---

## 16. Final Mental Model

```text
Master   -> Writes + critical reads
Replica  -> Reads + redundancy
Lag      -> Async replication cost
Failover -> Master failure recovery
Sentinel -> Automatic failover
Backup   -> Disaster recovery
```

---

## Next Part

**Part 22: Redis Sentinel — High Availability and Automatic Failover — Complete Deep Dive**

Isme hum cover karenge:

- Sentinel kya hai
- Sentinel responsibilities
- Master discovery
- Failure detection
- Quorum
- Automatic failover
- Sentinel configuration
- Spring Boot Sentinel setup
- Sentinel limitations
- Production topology
- Interview questions
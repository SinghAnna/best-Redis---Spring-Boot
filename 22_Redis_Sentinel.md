# Part 22: Redis Sentinel — High Availability and Automatic Failover — Complete Deep Dive

> Redis Sentinel Redis master-replica setup me automatic failover provide karta hai. Master down hone par Sentinel best replica ko promote karta hai aur clients ko naya master discover karne deta hai.

---

## 1. Sentinel Kya Hai?

**Redis Sentinel** ek separate process/system hai jo Redis deployment ko monitor karta hai.

```text
Sentinel 1
Sentinel 2
Sentinel 3
     |
     v
Redis Master + Replicas
```

Sentinel ka kaam:

```text
1. Master monitor karna
2. Replicas monitor karna
3. Failure detect karna
4. Automatic failover karna
5. Clients ko current master batana
```

Sentinel master ko monitor karta hai, replica ko promote karta hai, baaki replicas ko naye master ke saath reconfigure karta hai aur applications ko naye master ka address inform karta hai. [311]

---

## 2. Sentinel Responsibilities

| Responsibility | Explanation |
|---|---|
| Monitoring | Master aur replicas continuously check karta hai |
| Notification | Admin/API ko problems notify karta hai |
| Automatic failover | Master down hone par replica promote karta hai |
| Configuration provider | Clients ko current master address deta hai |
| Topology discovery | Master aur replicas automatically discover karta hai |

---

## 3. Sentinel Architecture

```text
+-------------+     +-------------+     +-------------+
| Sentinel 1  |     | Sentinel 2  |     | Sentinel 3  |
+------+------+     +------+------+     +------+------+
       |                   |                   |
       +---------+---------+---------+---------+
                 |                         |
                 v                         v
        +----------------+        +----------------+
        |  Redis Master  |        | Redis Replica  |
        +----------------+        +----------------+
```

Production me minimum **3 Sentinel processes** recommended hote hain, jisse quorum aur majority voting properly kaam kare. [317]

---

## 4. Why 3 Sentinels?

Agar sirf 1 Sentinel ho:

```text
Network issue
     |
     v
Sentinel thinks master is down
     |
     v
Wrong failover possible
```

Agar 3 Sentinels hain:

```text
Sentinel 1: Master down hai
Sentinel 2: Master down hai
Sentinel 3: Master reachable hai

Quorum: 2
Decision: Failover proceed
```

Sentinel quorum decide karta hai ki kitne Sentinels master unreachable hone par agree karein; actual failover ke liye Sentinel majority vote se leader elect hota hai. [311][317]

---

## 5. Quorum Kya Hai?

Quorum wo minimum Sentinel count hai jo agree kare ki master down hai.

```conf
sentinel monitor mymaster 127.0.0.1 6379 2
```

Meaning:

```text
mymaster master monitor karo
127.0.0.1:6379 par
quorum = 2
```

### Example

```text
3 Sentinels
Quorum = 2

Master down:
Sentinel 1 + Sentinel 2 agree
=> Failure detected
```

Quorum sirf failure detection ke liye hota hai; failover execute karne ke liye Sentinel processes ki majority vote chahiye hoti hai. [311]

---

## 6. Sentinel Configuration

### sentinel.conf

```conf
port 26379

sentinel monitor mymaster redis-master 6379 2
sentinel auth-pass mymaster strong-password
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

### Configuration Explanation

| Directive | Meaning |
|---|---|
| `sentinel monitor` | Master name, address aur quorum |
| `sentinel auth-pass` | Redis master password |
| `down-after-milliseconds` | Master unreachable hone ka timeout |
| `failover-timeout` | Failover retry/timeout window |
| `parallel-syncs` | Failover ke baad kitne replicas ek saath sync karein |

`down-after-milliseconds` batata hai ki instance kitne milliseconds unreachable rahe to use down mana jayega. [310][313]

---

## 7. Sentinel Failure Detection

### Subjective Down — SDOWN

```text
Ek Sentinel khud master ko unreachable samajhta hai
        |
        v
Subjective Down
```

### Objective Down — ODOWN

```text
Quorum Sentinels agree
        |
        v
Objective Down
```

### Failover Authorization

```text
Majority Sentinels vote
        |
        v
Leader Sentinel elected
        |
        v
Failover starts
```

Sentinel me pehle SDOWN hota hai, phir quorum agreement par ODOWN, aur failover execute karne ke liye majority-elected leader Sentinel chahiye. [311][317]

---

## 8. Automatic Failover Flow

```text
1. Master down
2. Sentinels detect failure
3. Quorum reached
4. Leader Sentinel elected
5. Best replica selected
6. Replica promoted:
   REPLICAOF NO ONE
7. Other replicas reconfigured
8. Clients informed about new master
```

### Diagram

```text
Master Down
     |
     v
Sentinels Detect
     |
     v
Quorum + Leader Election
     |
     v
Replica Promotion
     |
     v
New Master
     |
     v
Clients Reconnect
```

Sentinel failover me selected replica promote hota hai, baaki replicas naye master ke saath reconfigure hote hain aur clients ko naya master address milta hai. [311][320]

---

## 9. Docker Compose Sentinel Setup

```yaml
services:
  redis-master:
    image: redis:7.4-alpine
    container_name: redis-master
    command: >
      redis-server
      --requirepass strong-password
      --appendonly yes
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
    depends_on:
      - redis-master
    ports:
      - "6381:6379"

  sentinel-1:
    image: redis:7.4-alpine
    container_name: sentinel-1
    command: >
      redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./sentinel-1.conf:/etc/redis/sentinel.conf
    depends_on:
      - redis-master
      - redis-replica-1
      - redis-replica-2
    ports:
      - "26379:26379"

  sentinel-2:
    image: redis:7.4-alpine
    container_name: sentinel-2
    command: >
      redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./sentinel-2.conf:/etc/redis/sentinel.conf
    depends_on:
      - redis-master
      - redis-replica-1
      - redis-replica-2
    ports:
      - "26380:26379"

  sentinel-3:
    image: redis:7.4-alpine
    container_name: sentinel-3
    command: >
      redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./sentinel-3.conf:/etc/redis/sentinel.conf
    depends_on:
      - redis-master
      - redis-replica-1
      - redis-replica-2
    ports:
      - "26381:26379"
```

---

## 10. Sentinel Config Files

### `sentinel-1.conf`

```conf
port 26379

sentinel monitor mymaster redis-master 6379 2
sentinel auth-pass mymaster strong-password
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

### `sentinel-2.conf`

```conf
port 26379

sentinel monitor mymaster redis-master 6379 2
sentinel auth-pass mymaster strong-password
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

### `sentinel-3.conf`

```conf
port 26379

sentinel monitor mymaster redis-master 6379 2
sentinel auth-pass mymaster strong-password
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

---

## 11. Spring Boot Sentinel Configuration

Spring Boot me Sentinel use karte waqt direct Redis master host/port configure nahi karte; Sentinel nodes aur master name configure karte hain. [312][320]

```yaml
spring:
  data:
    redis:
      password: ${REDIS_PASSWORD}
      sentinel:
        master: mymaster
        nodes:
          - sentinel-1:26379
          - sentinel-2:26379
          - sentinel-3:26379
      lettuce:
        shutdown-timeout: 100ms
```

### With Sentinel Authentication

```yaml
spring:
  data:
    redis:
      password: ${REDIS_PASSWORD}
      sentinel:
        master: mymaster
        password: ${SENTINEL_PASSWORD}
        nodes:
          - sentinel-1:26379
          - sentinel-2:26379
          - sentinel-3:26379
```

Spring Boot Lettuce Sentinel nodes se current master discover karta hai; master fail hone par naya master automatically resolve hota hai. [312][319]

---

## 12. Sentinel Health Check

```bash
redis-cli -p 26379 SENTINEL master mymaster
```

Important fields:

| Field | Meaning |
|---|---|
| `ip` | Current master IP |
| `port` | Current master port |
| `flags` | Master state |
| `num-slaves` | Connected replicas |
| `num-other-sentinels` | Other known Sentinels |
| `quorum` | Configured quorum |

### Check Sentinels

```bash
redis-cli -p 26379 SENTINEL sentinels mymaster
```

### Check Replicas

```bash
redis-cli -p 26379 SENTINEL replicas mymaster
```

---

## 13. Manual Failover Test

```bash
redis-cli -p 26379 SENTINEL FAILOVER mymaster
```

### Test Steps

```text
1. Current master identify karo
2. Master container stop karo
3. Sentinel failover observe karo
4. New master verify karo
5. Spring Boot reconnect verify karo
```

```bash
docker stop redis-master
```

```bash
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
```

---

## 14. Sentinel Limitations

| Limitation | Explanation |
|---|---|
| No data sharding | Single master ka data limit |
| Failover me brief downtime | Promotion aur client discovery time |
| Async replication | Failover me data loss possible |
| Complex multi-master nahi | Ek master group per Sentinel setup |
| Client support required | Client ko Sentinel discovery support karna chahiye |

Sentinel high availability deta hai, lekin horizontal data sharding nahi karta; large dataset ke liye Redis Cluster consider karo. [310][311]

---

## 15. Sentinel vs Cluster

| Feature | Sentinel | Cluster |
|---|---|---|
| Main goal | High availability | Sharding + high availability |
| Data distribution | Single master dataset | Hash slots across nodes |
| Automatic failover | Yes | Yes |
| Horizontal scaling | No | Yes |
| Multi-key operations | Easier | Hash slot restrictions |
| Best for | Moderate dataset | Large dataset |

---

## 16. Production Checklist

- [ ] Minimum 3 Sentinel processes hain.
- [ ] Quorum 2 set hai.
- [ ] Master aur replicas alag availability zones me hain.
- [ ] Sentinel processes alag hosts/nodes me hain.
- [ ] `masterauth` configured hai.
- [ ] Sentinel authentication configured hai.
- [ ] `down-after-milliseconds` realistic hai.
- [ ] Failover testing done hai.
- [ ] Monitoring alerts hain.
- [ ] Client Sentinel discovery verify kiya hai.

---

## 17. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| 1 Sentinel only | Wrong failover risk | Minimum 3 Sentinels |
| Quorum galat set karna | Failover fail | Majority-based quorum |
| Same host par Sentinels | Host failure me saare Sentinels down | Separate nodes |
| `masterauth` missing | Replica sync fail | Password configure karo |
| Very low `down-after-milliseconds` | False failover | Network tolerance set karo |
| Very high timeout | Slow recovery | Balanced timeout |
| Failover test na karna | Production surprise | Regular failover drills |

---

## 18. Interview Questions

| Question | Answer |
|---|---|
| Redis Sentinel kya hai? | Redis master-replica setup ka monitoring aur automatic failover system. |
| Quorum kya hai? | Master down declare karne ke liye required Sentinel agreement count. |
| SDOWN aur ODOWN me difference? | SDOWN ek Sentinel ka subjective failure detection hai; ODOWN quorum agreement hai. |
| Sentinel failover kaise karta hai? | Best replica ko promote karke naya master banata hai. |
| `down-after-milliseconds` kya hai? | Master unreachable hone ke baad kitne milliseconds me use down mana jayega. |
| `parallel-syncs` kya hai? | Failover ke baad kitne replicas ek saath naye master se sync karenge. |
| Sentinel aur Cluster me difference? | Sentinel HA deta hai; Cluster sharding + HA deta hai. |
| Spring Boot Sentinel configuration me master host kyun nahi dete? | Sentinel current master dynamically discover karta hai. |

---

## 19. Final Mental Model

```text
Sentinel = Watchdog + Failover manager

Monitor -> Detect failure
Quorum  -> Agree master is down
Election -> Choose leader Sentinel
Promote -> Replica becomes master
Reconfigure -> Other replicas follow new master
Notify -> Clients discover new master
```

---

## Next Part

**Part 23: Redis Cluster — Sharding, Hash Slots and Horizontal Scaling — Complete Deep Dive**

Isme hum cover karenge:

- Redis Cluster kya hai
- Hash slots
- Cluster topology
- Data sharding
- Cluster vs Sentinel
- Multi-key operations
- Hash tags
- Spring Boot Cluster configuration
- Cluster failover
- Production architecture
- Interview questions
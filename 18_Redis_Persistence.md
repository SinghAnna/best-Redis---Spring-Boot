# Part 18: Redis Persistence — RDB, AOF and Hybrid Persistence — Complete Deep Dive

> Redis primarily in-memory hai, lekin production me persistence configure karna zaroori hai. Persistence decide karti hai ki Redis restart/crash ke baad kitna data recover hoga.

---

## 1. Persistence Kya Hai?

Persistence ka matlab hai Redis memory data ko disk par save karna.

```text
Redis Memory
     |
     +--> RDB Snapshot
     |
     +--> AOF Log
     |
     v
Disk / SSD
```

Redis persistence durable storage, jaise SSD, par data likhne ka process hai; RDB point-in-time snapshots banata hai aur AOF har write operation ko log karta hai. [258]

---

## 2. Persistence Kyun Zaroori Hai?

### Without Persistence

```text
Redis restart / crash
        |
        v
All data lost
```

### With Persistence

```text
Redis restart / crash
        |
        v
RDB/AOF file load
        |
        v
Data recovered
```

### Important Reality

```text
Persistence = Data loss window reduce karna,
not zero data loss guarantee.
```

---

## 3. RDB Persistence

**RDB** ka full form hai **Redis Database**.

RDB point-in-time snapshot banata hai.

```text
Redis memory at 10:00 PM
        |
        v
dump.rdb file
```

### How RDB Works

```text
1. Redis fork karta hai
2. Child process memory snapshot banata hai
3. Snapshot temporary file me likhta hai
4. File atomically rename hoti hai: dump.rdb
```

`BGSAVE` child process fork karke RDB temporary file me likhta hai aur complete hone par file ko atomically rename karta hai. [260]

### RDB Configuration

```conf
save 900 1
save 300 10
save 60 10000

dbfilename dump.rdb
dir /data
```

Meaning:

| Rule | Meaning |
|---|---|
| `save 900 1` | 900 seconds me 1 change -> snapshot |
| `save 300 10` | 300 seconds me 10 changes -> snapshot |
| `save 60 10000` | 60 seconds me 10,000 changes -> snapshot |

### Manual RDB Snapshot

```bash
SAVE
```

Blocking snapshot.

```bash
BGSAVE
```

Background snapshot — production me recommended.

### RDB Advantages

- Compact single-file backup
- Fast restart
- Lower CPU overhead
- Easy file copy/backup

### RDB Disadvantages

- Last snapshot ke baad ka data loss possible
- Large dataset par fork overhead
- Real-time durability nahi

---

## 4. AOF Persistence

**AOF** ka full form hai **Append Only File**.

AOF har write command ko append-only log me record karta hai.

```text
SET user:101:name "Rahul"
INCR post:201:likes
DEL user:102:profile
```

Restart par Redis in commands ko replay karke state rebuild karta hai. [260]

### AOF Configuration

```conf
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
dir /data
```

### AOF Write Flow

```text
Client command
     |
     v
Redis executes command
     |
     v
Command appended to AOF buffer
     |
     v
appendfsync policy
     |
     v
Disk write
```

---

## 5. `appendfsync` Options

| Option | Behavior | Durability | Performance |
|---|---|---|---|
| `always` | Har write ke baad fsync | Highest | Slowest |
| `everysec` | Har second fsync | Good | Balanced |
| `no` | OS decide karta hai | Lowest | Fastest |

`appendfsync always` me write client ko tabhi acknowledge hota hai jab AOF file disk par flush ho chuki ho. [258]

### `appendfsync always`

```conf
appendfsync always
```

```text
Har write disk par flush
Data loss almost zero
Latency zyada
```

### `appendfsync everysec`

```conf
appendfsync everysec
```

```text
Har second fsync
Approximately 1 second data loss possible
Best production default
```

### `appendfsync no`

```conf
appendfsync no
```

```text
OS kab flush karega, ye decide karega
Fast but risky
```

`everysec` me typical data-loss window 1 second tak hota hai, jabki `always` sabse safe lekin slowest hota hai. [260][267]

---

## 6. RDB vs AOF

| Feature | RDB | AOF |
|---|---|---|
| Storage format | Binary snapshot | Write command log |
| Data granularity | Point-in-time | Every write |
| Recovery speed | Fast | Comparatively slower |
| Backup size | Compact | Larger |
| Data loss window | Snapshot interval | `appendfsync` policy-based |
| CPU impact | Fork overhead | Continuous write overhead |
| Best for | Fast recovery, backups | Better durability |

RDB faster recovery aur compact backups deta hai, jabki AOF better durability aur latest recovery point deta hai. [260][264]

---

## 7. Hybrid Persistence

Hybrid persistence RDB snapshot aur AOF log dono combine karti hai.

```text
AOF File
├── RDB Snapshot (base data)
└── AOF Commands (incremental changes)
```

### Configuration

```conf
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
```

Hybrid persistence me AOF rewrite ke waqt current state RDB snapshot ke form me likha jata hai, aur uske baad ke incremental commands AOF format me append hote hain. [261][265]

### Benefits

```text
RDB jaisi fast restart speed
+
AOF jaisi better durability
```

Hybrid mode me restart speed RDB ke close hoti hai aur durability AOF jaisi milti hai. [263]

---

## 8. AOF Rewrite

AOF file time ke saath grow karti hai kyunki har write command append hota hai.

### Problem

```text
SET counter 1
INCR counter
INCR counter
INCR counter
INCR counter
```

Final value:

```text
5
```

AOF me unnecessarily 5 commands ho sakte hain.

### AOF Rewrite

```text
Purani AOF commands
        |
        v
Current memory state
        |
        v
Compact new AOF file
```

### Configuration

```conf
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

Meaning:

```text
AOF file last rewrite size se 100% badi ho jaye
aur minimum 64 MB ho
-> AOF rewrite trigger
```

---

## 9. Redis Restart Recovery

### RDB Only

```text
Redis starts
     |
     v
Load dump.rdb
     |
     v
Dataset restored
```

### AOF Only

```text
Redis starts
     |
     v
Load AOF file
     |
     v
Replay write commands
     |
     v
Dataset restored
```

### Hybrid Mode

```text
Redis starts
     |
     v
Load RDB preamble
     |
     v
Replay remaining AOF commands
     |
     v
Dataset restored
```

---

## 10. Persistence Configuration Examples

### Cache-Only Redis

```conf
save ""
appendonly no
```

Use case:

```text
Redis sirf cache hai.
Data loss acceptable hai.
DB source of truth hai.
```

### Session Store

```conf
appendonly yes
appendfsync everysec
save 900 1
save 300 10
save 60 10000
```

Use case:

```text
User sessions important hain.
1 second data loss acceptable.
```

### Critical Redis Data

```conf
appendonly yes
appendfsync always
save 900 1
save 300 10
save 60 10000
```

Use case:

```text
Financial, inventory ya critical workflow data.
Performance cost measure karna hoga.
```

---

## 11. Docker Persistence

### Docker Compose

```yaml
services:
  redis:
    image: redis:7.4-alpine
    container_name: redis-persistent
    command: >
      redis-server
      --appendonly yes
      --appendfsync everysec
      --aof-use-rdb-preamble yes
      --save 900 1
      --save 300 10
      --save 60 10000
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  redis_data:
```

### Verify Persistence Files

```bash
docker exec -it redis-persistent ls -lh /data
```

Expected files:

```text
appendonlydir/
dump.rdb
```

---

## 12. Backup Strategy

### RDB Backup

```bash
redis-cli BGSAVE
```

Wait for completion:

```bash
redis-cli LASTSAVE
```

Copy snapshot:

```bash
cp /data/dump.rdb /backup/dump-$(date +%F-%H%M).rdb
```

### AOF Backup

```bash
cp -r /data/appendonlydir /backup/aof-$(date +%F-%H%M)
```

### Production Backup Rules

```text
1. Regular automated backups lo
2. Backups ko different storage/account me rakho
3. Backup restore test karo
4. Backup encryption enable karo
5. Retention policy define karo
6. RDB + AOF dono use karo
```

RDB aur AOF dono ek saath run karna production me recommended approach hai — RDB fast recovery deta hai aur AOF operation-level data safety deta hai. [259][260]

---

## 13. Replication and Persistence

Replication persistence ka replacement nahi hai.

```text
Replication -> Node failure se recovery
Persistence -> Redis restart/crash se recovery
Backup       -> Disaster recovery
```

### Production Setup

```text
Primary Redis
   |
   +--> Replica 1
   |
   +--> Replica 2
   |
   +--> RDB + AOF persistence
   |
   +--> Scheduled backups
```

---

## 14. Data Loss Scenarios

| Scenario | RDB | AOF everysec | AOF always |
|---|---|---|---|
| Redis crash | Last snapshot ke baad data loss | Up to ~1 second loss | Almost no loss |
| Disk failure | Backup par depend | Backup par depend | Backup par depend |
| Accidental `FLUSHALL` | Snapshot se recover | AOF replay se bhi flushed state recover | Same issue |
| Replica failover | Replication lag ke hisaab se | Replication lag ke hisaab se | Replication lag ke hisaab se |

---

## 15. Persistence Monitoring

```bash
INFO persistence
```

Important fields:

| Field | Meaning |
|---|---|
| `rdb_last_save_time` | Last RDB snapshot time |
| `rdb_changes_since_last_save` | Last snapshot ke baad changes |
| `aof_enabled` | AOF enabled or not |
| `aof_last_write_status` | AOF write status |
| `aof_current_size` | Current AOF size |
| `aof_last_bgrewrite_status` | Last rewrite status |

---

## 16. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Cache Redis me bhi persistence off | Theek ho sakta hai, lekin intent clear hona chahiye | Cache-only config document karo |
| Session Redis me persistence off | User logout ho jayega | AOF everysec |
| Sirf RDB | Last snapshot ke baad data loss | RDB + AOF |
| `appendfsync always` blindly | Latency/throughput issue | Requirement measure karo |
| Backup test na karna | Restore time unknown | Regular restore drills |
| `FLUSHALL` allowed | Complete data loss | Command restriction |

---

## 17. Interview Questions

| Question | Answer |
|---|---|
| Redis persistence kya hai? | Redis memory data ko disk par save karne ka process. |
| RDB kya hai? | Point-in-time binary snapshot. |
| AOF kya hai? | Har write command ko append-only file me log karne wala mechanism. |
| RDB aur AOF me difference? | RDB snapshot-based hai; AOF command-log-based hai. |
| `appendfsync everysec` kya karta hai? | Har second AOF ko disk par fsync karta hai. |
| Hybrid persistence kya hai? | AOF file ke start me RDB snapshot aur baad me incremental AOF commands. |
| AOF rewrite kya hai? | AOF file ko current memory state ke compact form me rewrite karna. |
| Redis restart par data kaise recover hota hai? | RDB load ya AOF replay se. |
| Replication persistence ka replacement hai? | Nahi; dono alag failure scenarios handle karte hain. |

---

## 18. Final Mental Model

```text
RDB       -> Snapshot, fast recovery, compact backup
AOF       -> Write log, better durability
Hybrid    -> RDB base + AOF increments
everysec  -> Production default
always    -> Maximum durability, higher latency
Replica   -> Node failure recovery
Backup    -> Disaster recovery
```

---

## Next Part

**Part 19: Redis Memory Management and Eviction Policies — Complete Deep Dive**

Isme hum cover karenge:

- Redis memory model
- `maxmemory`
- Memory fragmentation
- Key expiration
- Lazy vs active expiration
- Eviction policies
- LRU vs LFU
- `volatile-*` policies
- `noeviction`
- Large keys
- Memory optimization
- Production memory configuration
- Interview questions
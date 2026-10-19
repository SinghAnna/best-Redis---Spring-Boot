# Part 24: Redis High Availability Design — Complete Deep Dive

> High Availability (HA) ka matlab hai: Redis node, availability zone ya datacenter fail hone par bhi application service available rahe. HA sirf replication nahi hai — persistence, backup, failover, monitoring aur recovery plan ka complete design hota hai.

---

## 1. HA Kya Hai?

High Availability ka target:

```text
Minimum downtime
+
Automatic recovery
+
Data safety
+
Predictable failover
```

### Availability Formula

\[
\text{Availability} =
\frac{\text{Uptime}}{\text{Uptime} + \text{Downtime}} \times 100
\]

Example:

```text
99.9% availability
= 8.76 hours downtime per year
```

---

## 2. Redis Failure Scenarios

| Failure | Impact | HA Solution |
|---|---|---|
| Redis process crash | Service unavailable | Restart, systemd/Kubernetes |
| Master node failure | Writes fail | Replica promotion / Sentinel / managed failover |
| Availability Zone failure | Region-level outage | Multi-AZ deployment |
| Disk failure | Persistence data loss | Replication + backups |
| Network partition | Split-brain risk | Quorum, Sentinel/Cluster |
| Accidental data deletion | Data loss | Backup + restricted commands |
| Redis memory full | Eviction/write errors | `maxmemory`, eviction policy, scaling |
| Application bug | Bad data cache | TTL, versioning, cache invalidation |

---

## 3. Redis Topology Options

| Topology | HA Level | Complexity | Best For |
|---|---|---|---|
| Standalone | Low | Low | Development, small internal tools |
| Master-Replica | Medium | Medium | Read scaling, basic redundancy |
| Sentinel | High | Medium | Automatic failover, moderate dataset |
| Cluster | High + Sharding | High | Large dataset, high throughput |
| Managed Redis | High | Low operational | Cloud production workloads |

Redis Sentinel quorum-based failover decision leta hai aur master failure par replica ko promote karta hai; Redis Cluster data distribution aur node failure tolerance dono deta hai. [342]

---

## 4. Standalone Redis

```text
Application -> Redis
```

### Pros

- Simple
- Low latency
- Easy debugging
- Low cost

### Cons

- Single point of failure
- No automatic failover
- Restart par downtime
- Persistence ke bina data loss risk

### When to Use

```text
Local development
Non-critical internal tools
Temporary cache
Low-traffic prototype
```

---

## 5. Master-Replica Replication

```text
Application -> Master
                |
                +--> Replica 1
                |
                +--> Replica 2
```

### Benefits

- Read scaling
- Data redundancy
- Replica-based backup
- Basic failure recovery

### Limitations

- Automatic failover nahi hota
- Replication lag possible
- Master failure par manual promotion chahiye

### When to Use

```text
Read-heavy workload
Manual failover acceptable
Moderate availability requirement
```

Redis replication master se replicas me data copy karta hai, jisse read scaling aur redundancy milti hai. [342]

---

## 6. Redis Sentinel HA

```text
Sentinel 1
Sentinel 2
Sentinel 3
     |
     v
Master + Replicas
```

### Benefits

- Automatic failover
- Master discovery
- Quorum-based failure detection
- Better availability than manual replication

### Limitations

- No horizontal sharding
- Single master dataset limit
- Failover ke dauran brief downtime
- Client Sentinel support required

### When to Use

```text
Production microservices
Moderate dataset
Automatic failover required
High availability required
```

Sentinel multiple Sentinel processes ke saath quorum-based failover decision leta hai, jisse split-brain risk reduce hota hai. [342]

---

## 7. Redis Cluster HA

```text
Master 1 + Replica 1
Master 2 + Replica 2
Master 3 + Replica 3
```

### Benefits

- Horizontal sharding
- Large dataset support
- High write throughput
- Automatic failover
- Multiple master nodes

### Limitations

- Hash slot complexity
- Multi-key operation restrictions
- Hash tags required
- Operational complexity zyada

### When to Use

```text
Large dataset
High write throughput
Data single node me fit nahi hota
Horizontal scaling required
```

Redis Cluster data ko multiple nodes me distribute karta hai aur subset nodes fail hone par bhi operations continue rakhne me help karta hai. [342]

---

## 8. Managed Redis Options

| Provider | Service | HA Features |
|---|---|---|
| AWS | ElastiCache for Redis | Multi-AZ, automatic failover, read replicas |
| GCP | Memorystore | Replication, HA options |
| Azure | Azure Cache for Redis | Zone redundancy, geo-replication |
| Redis Cloud | Redis Cloud | Managed HA, persistence, backups |
| Upstash | Upstash Redis | Serverless Redis, multi-region options |

AWS ElastiCache Multi-AZ replicas ko different Availability Zones me distribute karta hai aur primary failure par least replication lag wale replica ko automatically promote karta hai. [338][340]

### Managed Redis Benefits

```text
- Automated failover
- Patching
- Monitoring
- Backups
- Scaling
- Security
- Reduced operational burden
```

---

## 9. Persistence vs Replication vs Backup

Ye teen cheezein alag problems solve karti hain.

| Component | Problem Solved | Example |
|---|---|---|
| Persistence | Redis restart/crash recovery | RDB, AOF |
| Replication | Node failure recovery | Master-replica |
| Backup | Disaster recovery | S3/GCS snapshot |
| Sentinel/Cluster | Automatic failover | Replica promotion |

Backup ko primary host ke bahar aur blast radius ke outside rakhna chahiye, aur restore testing regular karni chahiye. [341]

### Important

```text
Replication backup ka replacement nahi hai.
Persistence backup ka replacement nahi hai.
```

---

## 10. Disaster Recovery Design

### RTO and RPO

| Term | Meaning |
|---|---|
| RTO | Recovery Time Objective — kitni der me service wapas aayegi |
| RPO | Recovery Point Objective — kitna data loss acceptable hai |

### Example

```text
RTO = 5 minutes
RPO = 1 minute
```

Meaning:

```text
Disaster ke baad service 5 minute me wapas aani chahiye.
Maximum 1 minute ka data loss acceptable hai.
```

### DR Architecture

```text
Primary Region
     |
     +--> Redis Primary
     |
     +--> Replica
     |
     +--> Scheduled RDB/AOF backup
                |
                v
        Cross-region object storage
                |
                v
        DR Region Restore
```

Redis Enterprise backup ko S3, Azure Blob ya Google Cloud Storage jaise external storage me snapshot kar sakta hai. [337]

---

## 11. Backup Strategy

### RDB Backup

```bash
redis-cli BGSAVE
```

### Copy Backup

```bash
cp /data/dump.rdb /backup/dump-$(date +%F-%H%M).rdb
```

### AOF Backup

```bash
cp -r /data/appendonlydir /backup/aof-$(date +%F-%H%M)
```

### Backup Rules

```text
1. Automated scheduled backups
2. Off-site/cross-region storage
3. Versioned backups
4. Encryption
5. Restore testing
6. Retention policy
7. Access control
```

---

## 12. Restore Process

### RDB Restore

```text
1. Redis stop karo
2. dump.rdb ko /data directory me rakho
3. Redis start karo
4. Dataset load verify karo
```

### AOF Restore

```text
1. Redis stop karo
2. AOF files restore karo
3. Redis start karo
4. AOF replay se data recover hoga
```

### Production Rule

```text
Restore hamesha isolated/staging environment me test karo,
direct production overwrite mat karo.
```

Restore ko isolated instance par validate karna aur tabhi production me apply karna safe practice hai. [343][346]

---

## 13. Multi-AZ Architecture

```text
Availability Zone A          Availability Zone B
+----------------+          +----------------+
| Redis Primary  |          | Redis Replica  |
+----------------+          +----------------+
        |                          |
        +---------- Replication ---+
```

### Benefits

```text
- Zone failure survive
- Automatic failover
- Lower downtime
- Better durability
```

AWS ElastiCache Multi-AZ me primary fail hone par replica automatically promote hota hai aur writes naye primary par resume ho jate hain. [339][344]

---

## 14. HA Topology Selection

| Requirement | Recommended Topology |
|---|---|
| Local development | Standalone |
| Small internal cache | Standalone / Master-Replica |
| Read-heavy production cache | Master-Replica / Managed Redis |
| Automatic failover required | Sentinel / Managed Redis |
| Large dataset | Redis Cluster |
| Very high write throughput | Redis Cluster |
| Minimal ops team | Managed Redis |
| Strict DR requirement | Multi-AZ + cross-region backup |

---

## 15. HA Monitoring

### Critical Metrics

| Metric | Alert Condition |
|---|---|
| `master_link_status` | Not `up` |
| Replication lag | Sudden increase |
| `connected_slaves` | Replica count drop |
| Sentinel quorum | Sentinel down |
| Redis memory | Near `maxmemory` |
| Evictions | Sudden spike |
| AOF write status | Error |
| Redis health endpoint | DOWN |

---

## 16. Production HA Checklist

- [ ] Redis single point of failure nahi hai.
- [ ] Master aur replica alag availability zones me hain.
- [ ] Automatic failover enabled hai.
- [ ] Replication lag monitoring hai.
- [ ] RDB + AOF persistence enabled hai.
- [ ] Automated backups hain.
- [ ] Backups cross-region/off-site hain.
- [ ] Restore testing schedule me hai.
- [ ] `FLUSHALL` restricted hai.
- [ ] Redis authentication/TLS enabled hai.
- [ ] Alerting configured hai.
- [ ] Failover drill regular hota hai.
- [ ] RTO/RPO defined hai.

---

## 17. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Sirf replication | Backup/DR missing | Replication + backup |
| Sirf backup | Downtime zyada | Replication + failover |
| Same AZ deployment | Zone failure risk | Multi-AZ |
| Restore test na karna | Recovery plan unproven | Regular restore drills |
| Persistence off for critical data | Data loss | AOF/RDB enable |
| Sentinel single node | Failover unreliable | Minimum 3 Sentinels |
| Cluster without need | Unnecessary complexity | Workload-based topology |

---

## 18. Interview Questions

| Question | Answer |
|---|---|
| Redis HA kya hai? | Redis failure ke baad bhi service available rakhne ka design. |
| Replication, persistence aur backup me difference? | Replication node failure, persistence restart recovery, backup disaster recovery ke liye hota hai. |
| Sentinel kab use karte hain? | Jab automatic failover chahiye ho bina sharding ke. |
| Cluster kab use karte hain? | Jab data sharding aur horizontal scaling chahiye ho. |
| Multi-AZ kya hai? | Redis nodes ko multiple availability zones me deploy karna. |
| RTO kya hai? | Disaster ke baad service recover hone ka target time. |
| RPO kya hai? | Maximum acceptable data loss window. |
| Managed Redis kyun use karte hain? | Failover, patching, backup aur monitoring operational burden reduce karne ke liye. |

---

## 19. Final Mental Model

```text
Replication  -> Node failure protection
Persistence  -> Restart/crash recovery
Backup       -> Disaster recovery
Sentinel     -> Automatic failover
Cluster      -> Sharding + failover
Multi-AZ     -> Zone failure protection
Monitoring   -> Early failure detection
```

---

## Next Part

**Part 25: Redis Security — Authentication, ACL, TLS, Network Isolation and Dangerous Commands — Complete Deep Dive**

Isme hum cover karenge:

- Redis security risks
- Authentication
- ACL users
- TLS encryption
- Network isolation
- VPC/private subnet
- Firewall rules
- Dangerous commands
- `FLUSHALL`, `CONFIG`, `DEBUG`
- Command renaming
- Secret
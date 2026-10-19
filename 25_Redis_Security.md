# Part 25: Redis Security — Authentication, ACL, TLS, Network Isolation and Dangerous Commands — Complete Deep Dive

> Redis security ka core principle hai: **Redis ko sirf trusted application network me expose karo, strong authentication use karo, least-privilege ACL lagao aur dangerous commands restrict karo.**

---

## 1. Redis Security Risks

| Risk | Impact |
|---|---|
| Public Redis exposure | Unauthorized access |
| No authentication | Anyone can read/write data |
| `FLUSHALL` access | Complete data deletion |
| `CONFIG` access | Server configuration change |
| `DEBUG` access | DoS/crash possible |
| No TLS | Credentials/data plaintext me travel |
| Weak passwords | Brute-force risk |
| Over-privileged ACL | Blast radius zyada |
| Public bind address | Internet-facing attack surface |

Redis documentation ke according untrusted access ko ACL, input validation aur controlled operations layer ke through mediate karna chahiye. [349]

---

## 2. Network Isolation

### Golden Rule

```text
Redis ko public internet par expose mat karo.
```

### Recommended Network Topology

```text
Internet
   |
   v
Load Balancer / API Gateway
   |
   v
Private Subnet: Spring Boot Services
   |
   v
Private Subnet: Redis
```

### Redis Configuration

```conf
bind 127.0.0.1 10.0.1.10
protected-mode yes
port 0
```

### Meaning

| Directive | Purpose |
|---|---|
| `bind` | Redis kis network interfaces par listen kare |
| `protected-mode yes` | Unauthorized external access block |
| `port 0` | Non-TLS port disable |

Redis ko loopback ya private network interface par bind karna chahiye; `0.0.0.0` par expose karna avoid karo. [360]

---

## 3. Firewall Rules

### Allow Only Application Servers

```bash
# Allow app subnet
sudo ufw allow from 10.0.1.0/24 to any port 6379

# Deny everything else on Redis port
sudo ufw deny 6379/tcp
```

### AWS Security Group Example

```text
Inbound Rule:
Protocol: TCP
Port: 6379
Source: 10.0.1.0/24  # Application subnet
```

### Best Practice

```text
Security group / firewall me
sirf application subnet/IP allow karo.
```

Firewall ko trusted IP/network tak access restrict karna chahiye. [357]

---

## 4. Authentication

### Basic Password

```conf
requirepass strong-production-password
```

### Spring Boot Configuration

```yaml
spring:
  data:
    redis:
      password: ${REDIS_PASSWORD}
```

### Strong Password Rules

```text
- Minimum 32 characters
- Random generated
- Secret manager me store
- Git me commit na karo
- Regular rotation policy
- Different password per environment
```

Redis documentation recommend karta hai ki Redis 6+ me ACL-based named users aur fine-grained permissions use karo. [349]

---

## 5. Redis ACL — Access Control List

Redis ACL named users, passwords, command permissions aur key access patterns control karta hai. [348]

### ACL Syntax

```text
user <username> on >password ~key-pattern +commands -commands
```

| Token | Meaning |
|---|---|
| `on` | User enabled |
| `off` | User disabled |
| `>password` | Password add |
| `~pattern` | Allowed key pattern |
| `+command` | Command allow |
| `-command` | Command deny |
| `+@category` | Command category allow |
| `-@category` | Command category deny |

---

## 6. Create Application User

### Least-Privilege App User

```bash
ACL SETUSER appuser on >StrongAppPassword ~app:* +@read +@write -@dangerous -@admin
```

Meaning:

```text
appuser enabled hai
Password: StrongAppPassword
Keys: sirf app:* access
Commands: read + write allowed
Dangerous/admin commands denied
```

### Admin User

```bash
ACL SETUSER admin on >StrongAdminPassword ~* +@all
```

### Default User Restrict

```bash
ACL SETUSER default off
```

### ACL Categories

| Category | Examples |
|---|---|
| `@read` | `GET`, `MGET`, `HGET` |
| `@write` | `SET`, `DEL`, `INCR` |
| `@admin` | `CONFIG`, `DEBUG`, `SHUTDOWN` |
| `@dangerous` | `FLUSHALL`, `FLUSHDB`, `KEYS` |
| `@connection` | `AUTH`, `PING`, `SELECT` |

---

## 7. ACL File Configuration

### redis.conf

```conf
aclfile /etc/redis/users.acl
```

### users.acl

```conf
user default off

user appuser on >StrongAppPassword ~app:* +@read +@write -@dangerous -@admin

user admin on >StrongAdminPassword ~* +@all
```

### Test ACL User

```bash
redis-cli --user appuser --pass StrongAppPassword

PING
SET app:user:101:profile "{...}"
GET app:user:101:profile

FLUSHALL
```

Expected:

```text
FLUSHALL permission denied
```

Redis ACL connections ko commands aur accessible keys dono ke level par limit kar sakta hai. [348]

---

## 8. Dangerous Commands

| Command | Risk | Production Action |
|---|---|---|
| `FLUSHALL` | Poora Redis data delete | Disable/restrict |
| `FLUSHDB` | Current DB delete | Disable/restrict |
| `CONFIG` | Server config read/change | Restrict/rename |
| `DEBUG` | Crash/DoS possible | Disable |
| `SHUTDOWN` | Redis shutdown | Admin-only |
| `KEYS` | Blocking scan | Avoid / restrict |
| `EVAL` | Lua execution | Controlled use |
| `REPLICAOF` | Replication topology change | Admin-only |
| `SAVE` / `BGSAVE` | Disk/CPU spike | Controlled use |

`FLUSHALL`, `FLUSHDB`, `CONFIG`, `DEBUG` aur `SHUTDOWN` production me restrict karna recommended hai. [352]

---

## 9. Disable Dangerous Commands

### Modern ACL Approach — Recommended

```bash
ACL SETUSER appuser -FLUSHALL -FLUSHDB -DEBUG -SHUTDOWN -CONFIG -REPLICAOF
```

### Legacy `rename-command` Approach

```conf
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command DEBUG ""
rename-command CONFIG ""
```

### Rename Instead of Disable

```conf
rename-command CONFIG "CONFIG_9f8b2c7a"
rename-command SHUTDOWN "SHUTDOWN_7d3e91b2"
```

`rename-command` dangerous commands ko disable ya secret name me rename kar sakta hai, lekin Redis 6+ me ACL preferred approach hai. [352][354]

---

## 10. TLS Encryption

TLS network traffic encrypt karta hai — password aur data plaintext me travel nahi karte.

### Redis TLS Configuration

```conf
port 0
tls-port 6380

tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key
tls-ca-cert-file /etc/redis/tls/ca.crt

tls-auth-clients yes
```

### Spring Boot TLS Configuration

```yaml
spring:
  data:
    redis:
      host: redis.internal.example.com
      port: 6380
      username: appuser
      password: ${REDIS_PASSWORD}
      ssl:
        enabled: true
```

Redis TLS by default mutual TLS support karta hai aur client certificate ko trusted CA ke against validate kar sakta hai. [350]

### TLS Rules

```text
1. TLS 1.2+ use karo
2. Valid certificate use karo
3. Private key securely store karo
4. Certificate rotation plan rakho
5. Non-TLS port disable karo
```

---

## 11. Secret Management

### Bad

```yaml
spring:
  data:
    redis:
      password: my-password-123
```

### Good

```yaml
spring:
  data:
    redis:
      password: ${REDIS_PASSWORD}
```

### Better

```text
AWS Secrets Manager
HashiCorp Vault
Kubernetes Secrets
GCP Secret Manager
Azure Key Vault
```

### Kubernetes Secret Example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
type: Opaque
stringData:
  REDIS_PASSWORD: strong-production-password
```

### Deployment Environment

```yaml
env:
  - name: REDIS_PASSWORD
    valueFrom:
      secretKeyRef:
        name: redis-secret
        key: REDIS_PASSWORD
```

---

## 12. Redis Security Architecture

```text
Internet
   |
   X  Blocked
   |
Private Subnet
   |
   +------------------+
   | Spring Boot Apps |
   +--------+---------+
            |
            | TLS + ACL user
            v
   +------------------+
   |      Redis       |
   | Private Network  |
   +------------------+
```

---

## 13. Security Hardening Checklist

- [ ] Redis public internet par exposed nahi hai.
- [ ] Redis private subnet/VPC me hai.
- [ ] Firewall/security group configured hai.
- [ ] `bind` public interface nahi hai.
- [ ] `protected-mode yes` hai.
- [ ] Strong authentication enabled hai.
- [ ] Default user disabled/restricted hai.
- [ ] ACL least-privilege users hain.
- [ ] `FLUSHALL`, `FLUSHDB`, `CONFIG`, `DEBUG` restricted hain.
- [ ] TLS enabled hai.
- [ ] Secrets secret manager me hain.
- [ ] Redis version patched hai.
- [ ] Audit logging/monitoring enabled hai.
- [ ] Security review periodically hota hai.

---

## 14. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Redis public IP par expose | Unauthorized access | Private subnet |
| No password | Anyone can access | ACL/password |
| Default user full access | Over-privileged access | Least-privilege ACL |
| TLS off | Credentials plaintext | TLS enable |
| Password Git me | Secret leak | Secret manager |
| `FLUSHALL` allowed | Accidental data wipe | ACL/rename/disable |
| Old Redis version | Known vulnerabilities | Regular patching |

---

## 15. Interview Questions

| Question | Answer |
|---|---|
| Redis ACL kya hai? | Named users, passwords, command permissions aur key-pattern access control ka feature. |
| Redis ko public internet par kyun expose nahi karte? | Unauthorized access, data deletion aur server compromise ka risk hota hai. |
| `FLUSHALL` kya karta hai? | Saare databases ki saari keys delete karta hai. |
| TLS Redis me kyun zaroori hai? | Password aur data ko network transit me encrypt karne ke liye. |
| Least privilege kya hai? | User/service ko sirf zaroori commands aur keys ka access dena. |
| `rename-command` kya karta hai? | Dangerous command ko disable ya secret name me rename karta hai. |
| Redis password kahan store karna chahiye? | Secret manager, Kubernetes Secret ya environment variable me. |
| `protected-mode` kya karta hai? | Unauthorized external access ko restrict karta hai. |

---

## 16. Final Mental Model

```text
Secure Redis =
Private network
+
Authentication
+
Least-privilege ACL
+
TLS
+
Restricted dangerous commands
+
Secret management
+
Monitoring
```

---

## Next Part

**Part 26: Redis Monitoring and Observability — Metrics, Actuator, Prometheus and Grafana — Complete Deep Dive**

Isme hum cover karenge:

- Monitoring kya hai
- Redis important metrics
- Memory metrics
- Latency metrics
- Hit/miss ratio
- Eviction metrics
- Replication metrics
- Spring Boot Actuator
- Micrometer
- Prometheus
- Grafana dashboard
- Alerting rules
- Production observability checklist
- Interview questions
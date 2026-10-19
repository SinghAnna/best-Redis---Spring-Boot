# Part 13: Distributed Locks with Redis — Complete Deep Dive

> Distributed lock ka use tab hota hai jab multiple application instances ya microservices same resource ko simultaneously modify na kar sakein. Redis iske liye fast, practical solution deta hai — lekin correctness ke rules samajhna zaroori hai.

---

## 1. Distributed Lock Kya Hai?

Normal Java lock sirf ek JVM ke andar kaam karta hai.

```java
synchronized (lockObject) {
    // Only one thread in this JVM
}
```

Lekin microservices me multiple instances hote hain:

```text
User Service Instance 1
User Service Instance 2
User Service Instance 3
```

Agar teeno instances same user profile update karein, to race condition ho sakti hai.

### Distributed Lock Flow

```text
Instance 1 -> lock:user:101 -> SUCCESS
Instance 2 -> lock:user:101 -> FAIL
Instance 3 -> lock:user:101 -> FAIL
```

Distributed lock multiple nodes/instances ke beech mutual exclusion provide karta hai.

---

## 2. Race Condition Example

### Problem

```text
Instance A:
1. Read wallet balance = 1000
2. Deduct 300
3. Save balance = 700

Instance B:
1. Read wallet balance = 1000
2. Deduct 500
3. Save balance = 500
```

Expected balance:

```text
1000 - 300 - 500 = 200
```

Actual balance:

```text
500
```

### Solution

```text
Instance A acquires lock
    -> Performs update
    -> Releases lock

Instance B waits / retries
```

---

## 3. Redis Lock Basic Pattern

Redis official pattern:

```bash
SET resource_name my_random_value NX PX 30000
```

- `resource_name` = lock key
- `my_random_value` = unique lock owner ID
- `NX` = set only if key does not exist
- `PX 30000` = expire after 30 seconds

Redis documentation ke according `SET ... NX PX` command lock acquire karta hai; `NX` key ko sirf tab set karta hai jab wo already exist na kare, aur `PX` expiry milliseconds me set karta hai. [189]

### Example

```bash
SET lock:order:1001 "instance-1-uuid" NX PX 10000
```

Response:

```text
OK -> Lock acquired
nil -> Lock already held
```

---

## 4. Why Unique Lock Value?

Agar value random/unique nahi hogi:

```text
Instance A:
SET lock:order:1001 "locked" NX PX 10000

Lock expires

Instance B:
SET lock:order:1001 "locked" NX PX 10000

Instance A:
DEL lock:order:1001

Problem:
Instance A ne Instance B ka lock delete kar diya.
```

### Correct Approach

```text
Lock value = UUID / unique owner ID

Delete lock only if value matches.
```

---

## 5. Safe Unlock With Lua

Lock delete karne se pehle verify karna chahiye ki lock usi owner ka hai.

```bash
EVAL "
if redis.call('GET', KEYS) == ARGV then
  return redis.call('DEL', KEYS)
else
  return 0
end
" 1 lock:order:1001 instance-1-uuid
```

Redis documentation ke according lock release ke liye Lua script use karni chahiye jo pehle value compare kare aur matching value hone par hi `DEL` kare. [189]

### Why Lua?

```text
GET + compare + DEL
= Atomic operation
```

---

## 6. Spring Boot Lock With `RedisTemplate`

```java
package com.example.lock;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.List;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class RedisLockService {

    private final RedisTemplate<String, Object> redisTemplate;

    private static final String UNLOCK_SCRIPT = """
            if redis.call('GET', KEYS) == ARGV then
                return redis.call('DEL', KEYS)
            else
                return 0
            end
            """;

    public String acquireLock(String lockKey, Duration ttl) {
        String ownerId = UUID.randomUUID().toString();

        Boolean acquired = redisTemplate.opsForValue()
                .setIfAbsent(lockKey, ownerId, ttl);

        if (Boolean.TRUE.equals(acquired)) {
            return ownerId;
        }

        return null;
    }

    public boolean releaseLock(String lockKey, String ownerId) {
        Long result = redisTemplate.execute(
                new DefaultRedisScript<>(UNLOCK_SCRIPT, Long.class),
                List.of(lockKey),
                ownerId
        );

        return result != null && result == 1L;
    }
}
```

---

## 7. Lock Usage Example

```java
package com.example.service;

import com.example.lock.RedisLockService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderService {

    private final RedisLockService redisLockService;
    private final OrderRepository orderRepository;

    public void processOrder(Long orderId) {
        String lockKey = "order-service:lock:order:" + orderId;

        String ownerId = redisLockService.acquireLock(
                lockKey,
                Duration.ofSeconds(10)
        );

        if (ownerId == null) {
            log.warn("Order {} is already being processed", orderId);
            return;
        }

        try {
            Order order = orderRepository.findById(orderId)
                    .orElseThrow();

            order.setStatus("PROCESSING");
            orderRepository.save(order);

        } finally {
            boolean released =
                    redisLockService.releaseLock(lockKey, ownerId);

            if (!released) {
                log.warn("Failed to release lock for order {}", orderId);
            }
        }
    }
}
```

---

## 8. Lock Expiry Problem

Agar lock ka TTL chhota ho aur business operation slow ho:

```text
Instance A:
1. Acquire lock, TTL = 5 seconds
2. Start heavy DB operation
3. TTL expires
4. Instance B acquires lock
5. Instance A still running
6. Both instances modify same resource
```

### Solutions

| Solution | Explanation |
|---|---|
| Sufficient lease time | Operation ka realistic maximum time set karo |
| Watchdog renewal | Lock TTL automatically extend karo |
| Fencing token | Resource update par version/token validate karo |
| Idempotency | Duplicate execution harmless banao |

---

## 9. Redisson Introduction

Redisson Java ke liye Redis client aur distributed data structures library hai. Iska `RLock` reentrant distributed lock implement karta hai. [192][194]

### Dependency

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.50.0</version>
</dependency>
```

### Basic Configuration

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379

redisson:
  single-server-config:
    address: "redis://localhost:6379"
```

---

## 10. Redisson `RLock`

```java
package com.example.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class RedissonOrderService {

    private final RedissonClient redissonClient;
    private final OrderRepository orderRepository;

    public void processOrder(Long orderId) {
        String lockKey = "order-service:lock:order:" + orderId;

        RLock lock = redissonClient.getLock(lockKey);

        boolean acquired = false;

        try {
            acquired = lock.tryLock(
                    2,
                    10,
                    TimeUnit.SECONDS
            );

            if (!acquired) {
                log.warn("Could not acquire lock for order {}", orderId);
                return;
            }

            Order order = orderRepository.findById(orderId)
                    .orElseThrow();

            order.setStatus("PROCESSING");
            orderRepository.save(order);

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Lock interrupted", e);
        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

### `tryLock` Parameters

```java
lock.tryLock(waitTime, leaseTime, unit);
```

| Parameter | Meaning |
|---|---|
| `waitTime` | Lock milne ke liye maximum wait |
| `leaseTime` | Lock automatically expire hone ka time |
| `unit` | Time unit |

---

## 11. Redisson Watchdog

Redisson watchdog lock TTL ko automatically renew karta hai jab tak owning JVM alive hai. Default watchdog timeout 30 seconds hota hai. [192][194]

### Watchdog Mode

```java
boolean acquired = lock.tryLock(2, TimeUnit.SECONDS);
```

Jab `leaseTime` pass nahi karte:

```text
Redisson watchdog
    |
    +--> Lock TTL renew
    |
    +--> JVM alive -> lock extended
    |
    +--> JVM crash -> renewals stop -> lock expires
```

### Fixed Lease Mode

```java
boolean acquired = lock.tryLock(
        2,
        10,
        TimeUnit.SECONDS
);
```

Jab explicit `leaseTime` pass karte hain:

```text
Watchdog renewal disabled
Lock exactly 10 seconds baad expire
```

Redisson me watchdog sirf tab chalta hai jab `leaseTime` pass na kiya jaye; explicit lease time dene par lock fixed duration ke baad expire hota hai. [194][201]

---

## 12. Watchdog vs Fixed Lease

| Feature | Watchdog Mode | Fixed Lease Mode |
|---|---|---|
| TTL renewal | Automatic | No |
| JVM crash behavior | Lock expires after renewal stops | Lock expires after lease time |
| Operation duration | Long-running operations ke liye better | Predictable short operations |
| Risk | Agar process hang ho to lock lamba hold ho sakta hai | Long operation me lock expire ho sakta hai |

---

## 13. Reentrant Lock

`RLock` reentrant hota hai — same thread same lock ko multiple baar acquire kar sakta hai.

```java
RLock lock = redissonClient.getLock("order-service:lock:order:1001");

lock.lock();

try {
    lock.lock(); // Same thread can acquire again

    // Business logic

} finally {
    lock.unlock();
    lock.unlock();
}
```

Redisson `RLock` reentrant lock implement karta hai; owning thread same resource ko multiple baar lock kar sakta hai aur counter track karta hai. [194]

---

## 14. Idempotency + Lock

Distributed lock duplicate execution reduce karta hai, lekin 100% guarantee nahi deta. Isliye critical operations me idempotency key bhi use karo.

### Example

```text
Client request:
Idempotency-Key: abc-123

Redis:
idempotency:abc-123 -> PROCESSING
```

### Java Example

```java
public OrderResponse createOrder(CreateOrderRequest request, String idempotencyKey) {
    String idempotencyRedisKey =
            "order-service:idempotency:" + idempotencyKey;

    Boolean isNew = redisTemplate.opsForValue()
            .setIfAbsent(
                    idempotencyRedisKey,
                    "PROCESSING",
                    Duration.ofMinutes(10)
            );

    if (!Boolean.TRUE.equals(isNew)) {
        throw new DuplicateRequestException("Duplicate order request");
    }

    // Create order
}
```

### Best Practice

```text
Lock = Reduce duplicate work
Idempotency = Guarantee duplicate-safe behavior
```

---

## 15. RedLock Algorithm

Redlock multiple independent Redis master nodes par lock acquire karne ka algorithm hai.

```text
Redis Master 1
Redis Master 2
Redis Master 3
Redis Master 4
Redis Master 5
```

Lock tab successful mana jata hai jab majority nodes, jaise 5 me se 3, lock grant karein. [202]

### Redlock Flow

```text
1. Client unique token generate karta hai
2. Har Redis master par SET NX PX try karta hai
3. Majority nodes se lock mila -> Lock acquired
4. Operation complete
5. Sab nodes se lock release
```

Redlock majority quorum use karta hai: N independent Redis masters me se N/2 + 1 nodes par lock acquire hona required hai. [202]

### Important Note

Redlock strong correctness guarantee dene ka claim karta hai, lekin distributed systems experts iske failure modes par debate karte hain. Critical correctness ke liye fencing token, database constraints, versioning ya stronger consensus system consider karo. [193]

---

## 16. Lock Best Practices

- [ ] Lock key business resource par based ho.
- [ ] Lock value unique owner ID ho.
- [ ] TTL compulsory ho.
- [ ] Unlock sirf owner hi kare.
- [ ] `tryLock` use karo, blocking `lock()` avoid karo.
- [ ] `finally` me unlock karo.
- [ ] `isHeldByCurrentThread()` check karo.
- [ ] Long operations ke liye watchdog use karo.
- [ ] Critical operations me idempotency key rakho.
- [ ] Lock ko business correctness ka single guarantee mat samjho.

---

## 17. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| `SETNX` without TTL | Lock kabhi release nahi hota | `SET NX PX` |
| Fixed lock value | Dusre owner ka lock delete ho sakta hai | UUID owner value |
| Direct `DEL` | Wrong lock release | Lua-based safe unlock |
| Very short TTL | Operation complete hone se pehle lock expire | Realistic lease/watchdog |
| Very long TTL | Instance crash par lock stuck | Watchdog / reasonable TTL |
| Lock ko 100% correctness samajhna | Edge-case duplicate execution | Idempotency + fencing |
| Blocking lock | Thread starvation | `tryLock` with timeout |

---

## 18. Interview Questions

| Question | Answer |
|---|---|
| Distributed lock kya hai? | Multiple application instances/nodes ke beech mutual exclusion provide karne wala mechanism. |
| Redis me simple lock kaise banate hain? | `SET lockKey uniqueValue NX PX ttl` se. |
| `NX` kya karta hai? | Key ko sirf tab set karta hai jab wo already exist na kare. |
| Lock value unique kyun hoti hai? | Taaki lock owner hi apna lock safely release kar sake. |
| Lock unlock ke liye Lua script kyun? | GET, compare aur DEL ko atomically execute karne ke liye. |
| Redisson watchdog kya hai? | Owning JVM alive hone par lock TTL automatically renew karne wala mechanism. |
| Watchdog kab disable hota hai? | Jab explicit `leaseTime` pass kiya jata hai. |
| Redlock kya hai? | Multiple independent Redis masters par majority quorum se lock acquire karne ka algorithm. |
| Lock ke saath idempotency kyun chahiye? | Lock expire/failover edge cases me duplicate execution safe banane ke liye. |

---

## 19. Final Mental Model

```text
SET NX PX        -> Acquire lock
Unique value     -> Lock ownership
Lua unlock       -> Safe release
TTL              -> Crash protection
Watchdog         -> Long operation protection
Idempotency      -> Duplicate-safe execution
Fencing token    -> Stronger correctness
```

---

## Next Part

**Part 14: Rate Limiting with Redis — Complete Deep Dive**

Isme hum cover karenge:

- Rate limiting kya hai
- Fixed window counter
- Sliding window log
- Sliding window counter
- Token bucket
- Leaky bucket
- Redis-based implementation
- Redisson rate limiter
- API/user/IP rate limiting
- Spring Boot filter implementation
- Production best practices
- Interview questions
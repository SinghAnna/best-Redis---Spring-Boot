# Part 3: Redis Commands, TTL, Expiration, Transactions and Lua Scripting — Complete Deep Dive

> Is part me hum Redis ke core commands, TTL, expiration, pipelining, transactions aur Lua scripting ko production perspective se cover karenge.

---

## 1. Redis Command Categories

| Category | Commands | Use Case |
|---|---|---|
| String | `SET`, `GET`, `DEL`, `INCR`, `DECR` | Cache, counter, token |
| Expiration | `EXPIRE`, `TTL`, `PERSIST`, `EXPIREAT` | TTL management |
| Batch | `MGET`, `MSET` | Multiple keys |
| Scan | `SCAN` | Safe key iteration |
| Transaction | `MULTI`, `EXEC`, `DISCARD`, `WATCH` | Atomic command group |
| Scripting | `EVAL`, `EVALSHA` | Atomic server-side logic |
| Server | `INFO`, `CLIENT LIST`, `SLOWLOG GET` | Debugging and monitoring |

---

## 2. Basic String Commands

### `SET`

```bash
SET user:101:name "Rahul"
```

TTL ke saath:

```bash
SET otp:101 "482913" EX 300
```

- `EX 300` = 300 seconds
- `PX 300000` = 300000 milliseconds
- `NX` = sirf tab set karo jab key exist na kare
- `XX` = sirf tab set karo jab key already exist kare
- `GET` = purani value return karo aur nayi value set karo

### Distributed Lock Example

```bash
SET lock:order:1001 "instance-1" NX PX 10000
```

Meaning:

```text
lock:order:1001 key ko sirf tab set karo
jab wo already exist na kare,
10 seconds ke TTL ke saath.
```

### `GET`

```bash
GET user:101:name
```

### `DEL`

```bash
DEL user:101:name
```

### `EXISTS`

```bash
EXISTS user:101:name
```

Return:

```text
1 -> key exists
0 -> key does not exist
```

---

## 3. TTL and Expiration

TTL ka matlab hai **Time To Live**. TTL ke baad key automatically delete ho jati hai.

### Set TTL

```bash
SET session:abc123 "user:101" EX 3600
```

### Check Remaining TTL

```bash
TTL session:abc123
```

Possible outputs:

| Value | Meaning |
|---|---|
| Positive number | Remaining seconds |
| `-1` | Key exists but no TTL |
| `-2` | Key does not exist |

### Remove TTL

```bash
PERSIST session:abc123
```

### Set TTL Later

```bash
SET user:101:profile "{...}"
EXPIRE user:101:profile 900
```

### Milliseconds TTL

```bash
PEXPIRE user:101:profile 900000
PTTL user:101:profile
```

### Expire at Specific Time

```bash
EXPIREAT user:101:profile 1760000000
```

---

## 4. TTL Production Rules

### Rule 1: Cache Keys Must Have TTL

```java
redisTemplate.opsForValue().set(key, value, Duration.ofMinutes(15));
```

### Rule 2: TTL Business Requirement Par Depend Kare

| Data | Recommended TTL |
|---|---|
| OTP | 5 minutes |
| JWT refresh token | 7 days |
| User profile cache | 10–30 minutes |
| Product catalog cache | 30 minutes–2 hours |
| Feed cache | 30 seconds–5 minutes |
| Rate limit window | 1 minute / 1 hour |
| Distributed lock | 5–30 seconds |

### Rule 3: Har Expiry Same Mat Rakho

Agar sabhi keys ek hi time par expire hongi, to Redis par sudden load aa sakta hai.

```text
Cache avalanche:
Bahut saari cached keys ek saath expire
-> DB par sudden heavy load
```

Solution:

```java
long baseTtl = 15;
long jitter = ThreadLocalRandom.current().nextLong(0, 3);

Duration ttl = Duration.ofMinutes(baseTtl + jitter);
```

---

## 5. Atomic Counter Commands

### `INCR`

```bash
SET post:201:likes 0
INCR post:201:likes
INCR post:201:likes
GET post:201:likes
```

`INCR` key ki integer value ko 1 se increment karta hai; agar key exist nahi karti to initial value 0 maan leta hai. [55]

### `INCRBY`

```bash
INCRBY post:201:likes 10
```

### `DECR`

```bash
DECR post:201:likes
```

### `DECRBY`

```bash
DECRBY post:201:likes 5
```

### Rate Limiting Example

```bash
INCR rate-limit:user:101:2026101822
EXPIRE rate-limit:user:101:2026101822 60
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class CounterRedisService {

    private final StringRedisTemplate redisTemplate;

    public Long incrementLikeCount(Long postId) {
        String key = "post-service:post:" + postId + ":likes";
        return redisTemplate.opsForValue().increment(key);
    }

    public Long incrementWithTtl(String key, Duration ttl) {
        Long count = redisTemplate.opsForValue().increment(key);

        if (count != null && count == 1L) {
            redisTemplate.expire(key, ttl);
        }

        return count;
    }
}
```

---

## 6. Batch Commands

### `MSET`

```bash
MSET user:101:name "Rahul" user:102:name "Amit"
```

### `MGET`

```bash
MGET user:101:name user:102:name
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class BatchRedisService {

    private final StringRedisTemplate redisTemplate;

    public void saveMultipleProfiles(Map<String, String> profiles) {
        redisTemplate.opsForValue().multiSet(profiles);
    }

    public List<String> getMultipleProfiles(List<String> keys) {
        return redisTemplate.opsForValue().multiGet(keys);
    }
}
```

---

## 7. Pipelining

Pipelining me multiple commands ek network round-trip me bheje jate hain.

### Without Pipelining

```text
Client -> SET A
Client -> SET B
Client -> SET C
```

Har command ke liye network round-trip.

### With Pipelining

```text
Client -> SET A, SET B, SET C
Redis  -> responses
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class PipelineRedisService {

    private final StringRedisTemplate redisTemplate;

    public void saveUsersInPipeline(List<UserProfile> users) {
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (UserProfile user : users) {
                byte[] key = ("user-service:user:" + user.getId() + ":profile")
                        .getBytes(StandardCharsets.UTF_8);

                byte[] value = user.getName().getBytes(StandardCharsets.UTF_8);

                connection.stringCommands().set(key, value);
            }
            return null;
        });
    }
}
```

### When to Use

- Bulk insert
- Bulk cache warm-up
- Bulk delete
- High-throughput write operations

### Caution

Bahut large pipeline memory pressure create kar sakti hai. Batch size 500–1000 commands rakho.

---

## 8. `KEYS` vs `SCAN`

### `KEYS` — Avoid in Production

```bash
KEYS user:*
```

Problem:

```text
KEYS saare matching keys ko ek saath scan karta hai.
Large dataset me Redis block ho sakta hai.
```

### `SCAN` — Production Safe

```bash
SCAN 0 MATCH user:* COUNT 100
```

Output:

```text
1) "342"
2) 1) "user:101"
   2) "user:102"
```

`SCAN` cursor-based iteration karta hai, isliye ek baar me poora keyspace scan nahi karta.

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class RedisScanService {

    private final StringRedisTemplate redisTemplate;

    public List<String> findUserKeys() {
        List<String> keys = new ArrayList<>();

        ScanOptions options = ScanOptions.scanOptions()
                .match("user-service:user:*")
                .count(500)
                .build();

        try (Cursor<String> cursor = redisTemplate.scan(options)) {
            while (cursor.hasNext()) {
                keys.add(cursor.next());
            }
        }

        return keys;
    }
}
```

### Golden Rule

```text
Production me KEYS use mat karo.
SCAN use karo.
```

---

## 9. Redis Transactions

Redis transactions `MULTI`, `EXEC`, `DISCARD` aur `WATCH` commands par based hain. `MULTI` ke baad commands queue hoti hain aur `EXEC` call hone par sab execute hoti hain; `DISCARD` transaction queue ko flush kar deta hai. [50]

### Basic Transaction

```bash
MULTI
SET order:1001:status "CREATED"
INCR user:101:order-count
EXEC
```

### Flow

```text
MULTI
  |
  +--> Commands queue me jaati hain
  |
  +--> EXEC
        |
        +--> Saari queued commands ek saath execute
```

### `DISCARD`

```bash
MULTI
SET order:1001:status "CREATED"
DISCARD
```

Is case me koi command execute nahi hogi.

### Transaction Guarantees

| Guarantee | Supported? | Explanation |
|---|---|---|
| Commands grouped | Yes | Commands queue hoti hain |
| Atomic execution | Yes | Commands sequentially execute hoti hain |
| Isolation | Yes | Transaction execution ke beech dusre client commands intervene nahi karte |
| Rollback | No | Runtime error aane par already executed commands rollback nahi hote |

Redis transactions full relational-database-style rollback provide nahi karte. [50]

---

## 10. `WATCH` — Optimistic Locking

`WATCH` ek key ko monitor karta hai. Agar `EXEC` se pehle watched key modify ho jaye, transaction abort ho jata hai. [50]

```bash
WATCH stock:product:101

GET stock:product:101

MULTI
DECR stock:product:101
EXEC
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class InventoryRedisService {

    private final StringRedisTemplate redisTemplate;

    public boolean reserveStock(Long productId, int quantity) {
        String stockKey = "inventory-service:stock:" + productId;

        return Boolean.TRUE.equals(redisTemplate.execute(new SessionCallback<>() {
            @SuppressWarnings("unchecked")
            @Override
            public Boolean execute(RedisOperations operations) {
                operations.watch(stockKey);

                String currentStock = (String) operations.opsForValue().get(stockKey);

                if (currentStock == null || Integer.parseInt(currentStock) < quantity) {
                    operations.unwatch();
                    return false;
                }

                operations.multi();
                operations.opsForValue().increment(stockKey, -quantity);
                return operations.exec() != null;
            }
        }));
    }
}
```

### Important

`WATCH` optimistic locking deta hai. High contention me retry logic chahiye hota hai.

---

## 11. Lua Scripting

Lua script Redis server par execute hoti hai, isliye multiple operations ko ek atomic unit me run kiya ja sakta hai. `EVAL` Lua script ko server-side execute karta hai; script me keys `KEYS` aur additional arguments `ARGV` ke through available hote hain. [51][54]

### Basic Lua Script

```bash
EVAL "return redis.call('GET', KEYS)" 1 user:101:name
```

### Atomic Increment and TTL

```bash
EVAL "
local count = redis.call('INCR', KEYS)
if count == 1 then
  redis.call('EXPIRE', KEYS, ARGV)
end
return count
" 1 rate-limit:user:101 60
```

### Why Lua Script?

```text
INCR + EXPIRE ko alag-alag bhejne me
dono commands ke beech failure ho sakta hai.

Lua script me dono operations
server par atomically execute hote hain.
```

### Spring Boot Lua Example

```java
@Service
@RequiredArgsConstructor
public class RateLimiterLuaService {

    private final StringRedisTemplate redisTemplate;

    private static final String RATE_LIMIT_SCRIPT = """
            local count = redis.call('INCR', KEYS)
            if count == 1 then
                redis.call('EXPIRE', KEYS, ARGV)
            end
            return count
            """;

    public boolean isAllowed(String clientId, int limit, long windowSeconds) {
        String key = "rate-limit:" + clientId;

        Long count = redisTemplate.execute(
                new DefaultRedisScript<>(RATE_LIMIT_SCRIPT, Long.class),
                List.of(key),
                String.valueOf(windowSeconds)
        );

        return count != null && count <= limit;
    }
}
```

### Lua Script Best Practices

- Script chhoti rakho.
- Script me business logic zyada mat likho.
- `KEYS` aur `ARGV` ka sahi use karo.
- Script ko version control me rakho.
- Long-running script Redis ko block kar sakti hai.
- Cluster me multi-key scripts ke liye hash tags ka dhyan rakho.

---

## 12. Important Redis Commands for Debugging

### Redis Info

```bash
INFO memory
INFO stats
INFO replication
INFO clients
```

### Slow Log

```bash
SLOWLOG GET 10
SLOWLOG LEN
SLOWLOG RESET
```

### Client Connections

```bash
CLIENT LIST
```

### Database Size

```bash
DBSIZE
```

### Delete Key

```bash
DEL user:101:profile
```

### Check Key Type

```bash
TYPE user:101:followers
```

---

## 13. Production Commands to Avoid

| Command | Why Avoid | Alternative |
|---|---|---|
| `KEYS *` | Redis block kar sakta hai | `SCAN` |
| `FLUSHALL` | Poora Redis data delete | Restricted ACL / controlled admin operation |
| `FLUSHDB` | Current DB delete | Controlled maintenance only |
| `CONFIG GET *` | Security risk | Restricted access |
| `DEBUG` commands | Performance/security risk | Disable in production |
| Long Lua scripts | Redis block ho sakta hai | Small atomic scripts |

---

## 14. TTL + Cache Pattern

### Cache-Aside Pattern

```text
1. Request aata hai
2. Redis check karo
3. Cache hit -> return cached value
4. Cache miss -> DB se fetch karo
5. Redis me TTL ke saath store karo
6. Response return karo
```

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class ProductCacheService {

    private final ProductRepository productRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    public ProductResponse getProduct(Long productId) {
        String key = "product-service:product:" + productId + ":details";

        ProductResponse cached =
                (ProductResponse) redisTemplate.opsForValue().get(key);

        if (cached != null) {
            return cached;
        }

        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new ResourceNotFoundException("Product not found"));

        ProductResponse response = ProductResponse.from(product);

        long jitter = ThreadLocalRandom.current().nextLong(0, 120);

        redisTemplate.opsForValue().set(
                key,
                response,
                Duration.ofSeconds(1800 + jitter)
        );

        return response;
    }
}
```

---

## 15. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| TTL set na karna | Memory leak / Redis full | Har cache key par TTL |
| Same TTL for all keys | Cache avalanche | TTL jitter use karo |
| `KEYS` in production | Redis blocking | `SCAN` |
| Transaction ko DB transaction samajhna | Rollback expectation galat | Redis transaction semantics samjho |
| Lua script me heavy logic | Redis block | Short atomic scripts |
| `INCR` ke baad TTL set na karna | Counter key kabhi expire nahi hogi | First increment par TTL set karo |
| Pipelining me huge batch | Memory pressure | 500–1000 commands per batch |

---

## 16. Interview Questions

| Question | Answer |
|---|---|
| TTL kya hai? | Key ke automatically expire hone ka time. |
| `TTL -1` ka matlab? | Key exist karti hai lekin uska TTL set nahi hai. |
| `TTL -2` ka matlab? | Key exist nahi karti. |
| `INCR` kya karta hai? | Integer value ko atomically 1 se increment karta hai. |
| Pipelining kya hai? | Multiple commands ko ek network round-trip me bhejna. |
| `KEYS` production me kyun avoid karte hain? | Ye large dataset me Redis ko block kar sakta hai. |
| Redis transaction rollback karta hai? | Nahi, Redis transactions full rollback provide nahi karte. |
| `WATCH` kya karta hai? | Optimistic locking; watched key change hone par transaction abort hota hai. |
| Lua script kyun use karte hain? | Multiple Redis operations ko server par atomically execute karne ke liye. |
| Cache avalanche kya hai? | Bahut saari keys ek saath expire hone se DB par sudden load aana. |

---

## 17. Final Mental Model

```text
SET/GET       -> Basic key-value operations
EXPIRE/TTL    -> Automatic expiration
INCR/DECR     -> Atomic counters
MSET/MGET     -> Batch operations
Pipelining    -> Network round-trips reduce
MULTI/EXEC    -> Grouped command execution
WATCH         -> Optimistic locking
Lua Script    -> Server-side atomic logic
SCAN          -> Safe production key iteration
```

---

## Next Part

**Part 4: Spring Data Redis Setup — Complete Deep Dive**

Isme hum cover karenge:

- Spring Data Redis kya hai
- Lettuce vs Jedis
- Maven dependencies
- Docker Redis setup
- Spring Boot Redis connection
- `RedisConnectionFactory`
- `RedisTemplate`
- `StringRedisTemplate`
- Health check
- Common connection errors
- Production configuration
- Interview questions
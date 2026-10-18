# Part 8: Spring Cache Abstraction — Complete Deep Dive

> Spring Cache abstraction ka goal hai: business code me caching logic daalne ki jagah annotations se cache behavior control karna. Redis is abstraction ke backend cache store ke roop me kaam karta hai.

---

## 1. Caching Kya Hai?

Caching ka matlab hai frequently used data ko fast storage me rakhna, taaki baar-baar slow database call na karna pade.

### Without Cache

```text
Client -> Spring Boot -> Database
Client -> Spring Boot -> Database
Client -> Spring Boot -> Database
```

### With Cache

```text
Client -> Spring Boot -> Redis -> Return

Only on cache miss:
Client -> Spring Boot -> Redis -> Database -> Redis -> Return
```

---

## 2. Cache Hit vs Cache Miss

### Cache Hit

```text
Request -> Redis -> Data found -> Return
```

Database call skip hoti hai.

### Cache Miss

```text
Request -> Redis -> Data not found
       -> Database
       -> Store in Redis
       -> Return
```

### Hit Ratio

\[
\text{Hit Ratio} =
\frac{\text{Cache Hits}}{\text{Cache Hits} + \text{Cache Misses}}
\]

Example:

```text
Total requests = 1000
Cache hits = 900
Cache misses = 100

Hit Ratio = 900 / 1000 = 90%
```

---

## 3. Spring Cache Abstraction

Spring Cache abstraction application code aur actual cache provider ke beech abstraction layer hai.

```text
Business Service
      |
      v
Spring Cache Abstraction
      |
      +--> RedisCacheManager
      |
      +--> CaffeineCacheManager
      |
      +--> ConcurrentMapCacheManager
      |
      v
Cache Store
```

Spring me `@Cacheable`, `@CachePut` aur `@CacheEvict` declarative caching annotations hain. [116]

### Enable Caching

```java
@SpringBootApplication
@EnableCaching
public class UserServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

---

## 4. Cache-Aside Pattern

Ye sabse common Redis caching pattern hai.

### Flow

```text
1. Application Redis check karta hai
2. Cache hit -> Redis se data return
3. Cache miss -> Database se data fetch
4. Application Redis me data store karta hai
5. Response return
```

### Diagram

```text
        GET /users/101
              |
              v
       +-------------+
       | Spring Boot |
       +-------------+
              |
              v
       +-------------+
       |    Redis    |
       +-------------+
          |       |
     Cache Hit   Cache Miss
          |       |
          v       v
        Return  PostgreSQL
                |
                v
           Store in Redis
```

Cache-aside me application cache aur database dono se directly communicate karta hai; cache miss hone par DB se fetch karke cache me write-back hota hai. [117]

### Spring Boot Example

```java
@Service
@RequiredArgsConstructor
public class UserProfileService {

    private final UserRepository userRepository;

    @Cacheable(
            value = "userProfile",
            key = "#userId",
            unless = "#result == null"
    )
    public UserProfileResponse getUserProfile(Long userId) {
        return userRepository.findById(userId)
                .map(UserProfileResponse::from)
                .orElse(null);
    }
}
```

### Best For

- Read-heavy APIs
- User profile
- Product details
- Configuration data
- Reference data

---

## 5. Read-Through Cache

Read-through cache me application directly cache se baat karta hai. Cache miss hone par cache layer khud database se data fetch karke cache me store karta hai.

```text
Application -> Cache -> Database
```

### Difference From Cache-Aside

| Feature | Cache-Aside | Read-Through |
|---|---|---|
| Cache miss handling | Application karta hai | Cache layer karta hai |
| Application complexity | Zyada | Kam |
| Control | Application ke paas | Cache provider ke paas |
| Spring Boot common usage | Yes | Less common |

---

## 6. Write-Through Cache

Write-through me data pehle cache me write hota hai, aur synchronously database me bhi write hota hai.

```text
Application
    |
    +--> Redis Cache
    |        |
    |        +--> Database
    |
    +--> Response
```

Write-through caching me cache aur database dono me synchronous update hota hai, jisse strong consistency milti hai. [117][120]

### Example

```java
@Service
@RequiredArgsConstructor
public class UserProfileWriteService {

    private final UserRepository userRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    public UserProfileResponse updateProfile(Long userId, UpdateProfileRequest request) {
        UserProfile user = userRepository.findById(userId)
                .orElseThrow();

        user.setName(request.getName());
        user.setCity(request.getCity());

        UserProfile saved = userRepository.save(user);

        UserProfileResponse response = UserProfileResponse.from(saved);

        redisTemplate.opsForValue().set(
                "user-service:user:" + userId + ":profile",
                response,
                Duration.ofMinutes(15)
        );

        return response;
    }
}
```

### Best For

- Profile updates
- Settings
- Inventory count
- Data where read-after-write consistency matters

### Trade-off

```text
Write latency badh sakti hai
kyunki cache aur DB dono update hote hain.
```

---

## 7. Write-Behind Cache

Write-behind ko **write-back cache** bhi kehte hain.

Isme data pehle cache me write hota hai, aur database me asynchronously write hota hai.

```text
Application -> Redis Cache -> Response

Redis Cache -> Async Writer -> Database
```

Write-behind caching write performance improve karta hai kyunki database write asynchronously hota hai, lekin data loss risk hota hai agar cache fail ho jaye. [120]

### Example Flow

```text
1. Client updates profile
2. Application Redis me data write karta hai
3. Client ko immediate response milta hai
4. Background worker DB me data persist karta hai
```

### Best For

- High write throughput
- Analytics counters
- View counts
- Non-critical temporary data

### Risk

```text
Redis crash hone par
jo writes abhi DB me persist nahi hui
wo loss ho sakti hain.
```

---

## 8. Caching Pattern Comparison

| Pattern | Read | Write | Consistency | Best For |
|---|---|---|---|---|
| Cache-Aside | App manages | App manages | Eventual, TTL-bounded | Read-heavy APIs |
| Read-Through | Cache manages | App manages | Eventual | Transparent caching |
| Write-Through | Cache-first read | Cache + DB sync | Strong | Critical updates |
| Write-Behind | Cache-first read | Cache first, DB async | Weak | High write throughput |

Cache-aside read-heavy applications ke liye excellent choice hai jab cache misses acceptable hain. [117]

---

## 9. Cache Penetration

Cache penetration tab hota hai jab aisa data request ho jo **database me exist hi nahi karta**.

### Example

```text
GET /users/999999999

Redis -> null
DB -> null
Redis -> null
DB -> null
DB -> null
```

Har request DB tak jayegi.

### Solution 1: Null Caching

```java
@Cacheable(
        value = "userProfile",
        key = "#userId",
        unless = "#result == null"
)
public UserProfileResponse getUserProfile(Long userId) {
    return userRepository.findById(userId)
            .map(UserProfileResponse::from)
            .orElse(null);
}
```

Agar null caching enable karo:

```yaml
spring:
  cache:
    redis:
      cache-null-values: true
```

Lekin null values ka TTL short rakho.

### Solution 2: Bloom Filter

```text
Request -> Bloom Filter
             |
             +--> Definitely not present -> Reject
             |
             +--> Maybe present -> Check Redis/DB
```

Bloom filter false positive de sakta hai, lekin false negative nahi deta.

---

## 10. Cache Stampede

Cache stampede tab hota hai jab ek **hot key expire** ho jaye aur usi time bahut saare requests DB par chale jayein.

### Example

```text
Viral post: post:501

Cache expired

1000 requests same time
    |
    +--> All 1000 requests hit DB
```

Isko **dogpile effect** bhi kehte hain.

### Solution 1: `sync = true`

```java
@Cacheable(
        value = "post",
        key = "#postId",
        sync = true
)
public PostResponse getPost(Long postId) {
    return postRepository.findById(postId)
            .map(PostResponse::from)
            .orElseThrow();
}
```

`sync = true` same key ke liye multiple threads ko method execute karne se rokhne ki koshish karta hai.

### Solution 2: Distributed Lock

```text
First request:
- Acquire lock
- Load from DB
- Store in Redis
- Release lock

Other requests:
- Wait briefly
- Read updated cache
```

### Solution 3: Stale-While-Revalidate

```text
Purana cached data return karo
aur background me fresh data load karo.
```

---

## 11. Cache Avalanche

Cache avalanche tab hota hai jab bahut saari cache keys **ek hi time par expire** ho jayein.

### Example

```text
10,000 product cache keys
TTL = 10:00:00 PM

At 10:00:00 PM
All keys expire together

Massive DB load
```

### Solution: TTL Jitter

```java
long baseTtlSeconds = 1800;
long jitterSeconds = ThreadLocalRandom.current().nextLong(0, 300);

Duration ttl = Duration.ofSeconds(baseTtlSeconds + jitterSeconds);

redisTemplate.opsForValue().set(key, value, ttl);
```

### Solution: Gradual Cache Warm-Up

```text
Application start hone par
saara cache ek saath populate mat karo.

Small batches me warm-up karo.
```

TTL jitter, staggered expiration aur gradual warm-up cache avalanche mitigate karne ke common tarike hain. [125][128]

---

## 12. Cache Breakdown

Cache breakdown ek specific hot key expire hone par hota hai.

| Problem | Scope | Cause |
|---|---|---|
| Cache penetration | Non-existing data | Invalid ID requests |
| Cache stampede | One hot key | Hot key expiry |
| Cache avalanche | Many keys | Simultaneous expiry |

### Prevention Summary

| Problem | Prevention |
|---|---|
| Penetration | Null caching, Bloom filter, request validation |
| Stampede | Locking, `sync = true`, stale-while-revalidate |
| Avalanche | TTL jitter, gradual warm-up, HA Redis |

---

## 13. Spring Cache Annotations Overview

| Annotation | Purpose |
|---|---|
| `@Cacheable` | Method result cache karta hai |
| `@CachePut` | Method hamesha execute karta hai aur result cache update karta hai |
| `@CacheEvict` | Cache entry remove karta hai |
| `@Caching` | Multiple cache annotations combine karta hai |
| `@CacheConfig` | Class-level common cache configuration |

Spring documentation ke according `@Cacheable` subsequent calls par method execution skip karke cached value return karta hai. [116]

---

## 14. `@Cacheable`

```java
@Cacheable(
        value = "userProfile",
        key = "#userId"
)
public UserProfileResponse getUserProfile(Long userId) {
    return userRepository.findById(userId)
            .map(UserProfileResponse::from)
            .orElseThrow();
}
```

### Flow

```text
1. Spring cache check karta hai
2. Cache hit -> method skip, cached value return
3. Cache miss -> method execute
4. Result cache me store
5. Result return
```

### Conditional Caching

```java
@Cacheable(
        value = "userProfile",
        key = "#userId",
        condition = "#userId > 0",
        unless = "#result == null"
)
public UserProfileResponse getUserProfile(Long userId) {
    return userRepository.findById(userId)
            .map(UserProfileResponse::from)
            .orElse(null);
}
```

### `condition` vs `unless`

| Attribute | Check Time | Use |
|---|---|---|
| `condition` | Method execution se pehle | Cache karna hai ya nahi |
| `unless` | Method execution ke baad | Result cache karna hai ya nahi |

---

## 15. `@CachePut`

`@CachePut` method ko hamesha execute karta hai aur method ka result cache me update karta hai. [116]

```java
@CachePut(
        value = "userProfile",
        key = "#result.id"
)
public UserProfileResponse updateProfile(
        Long userId,
        UpdateProfileRequest request
) {
    UserProfile user = userRepository.findById(userId)
            .orElseThrow();

    user.setName(request.getName());
    user.setCity(request.getCity());

    return UserProfileResponse.from(userRepository.save(user));
}
```

### Important

```text
@Cacheable -> Method skip ho sakta hai
@CachePut  -> Method hamesha execute hota hai
```

---

## 16. `@CacheEvict`

`@CacheEvict` cache entry remove karta hai. [116]

### Single Key Evict

```java
@CacheEvict(
        value = "userProfile",
        key = "#userId"
)
public void deleteUser(Long userId) {
    userRepository.deleteById(userId);
}
```

### All Entries Evict

```java
@CacheEvict(
        value = "userProfile",
        allEntries = true
)
public void refreshAllUserProfiles() {
    // Admin operation
}
```

### Before Invocation

```java
@CacheEvict(
        value = "userProfile",
        key = "#userId",
        beforeInvocation = true
)
public void deleteUser(Long userId) {
    userRepository.deleteById(userId);
}
```

---

## 17. `@Caching`

Jab ek method par multiple cache operations chahiye:

```java
@Caching(
        evict = {
                @CacheEvict(
                        value = "userProfile",
                        key = "#userId"
                ),
                @CacheEvict(
                        value = "userFeed",
                        key = "#userId"
                )
        }
)
public void updateUser(Long userId, UpdateProfileRequest request) {
    userProfileRepository.update(userId, request);
}
```

---

## 18. `@CacheConfig`

Class-level common configuration:

```java
@Service
@CacheConfig(cacheNames = "userProfile")
@RequiredArgsConstructor
public class UserProfileService {

    private final UserRepository userRepository;

    @Cacheable(key = "#userId")
    public UserProfileResponse getUserProfile(Long userId) {
        return userRepository.findById(userId)
                .map(UserProfileResponse::from)
                .orElseThrow();
    }

    @CacheEvict(key = "#userId")
    public void deleteUser(Long userId) {
        userRepository.deleteById(userId);
    }
}
```

---

## 19. Redis CacheManager Configuration

```java
package com.example.config;

import org.springframework.cache.CacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;
import java.util.HashMap;
import java.util.Map;

@Configuration
public class RedisCacheManagerConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration
                .defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .disableCachingNullValues()
                .serializeKeysWith(
                        RedisSerializationContext.SerializationPair
                                .fromSerializer(new StringRedisSerializer())
                )
                .serializeValuesWith(
                        RedisSerializationContext.SerializationPair
                                .fromSerializer(new GenericJackson2JsonRedisSerializer())
                );

        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();

        cacheConfigs.put(
                "userProfile",
                defaultConfig.entryTtl(Duration.ofMinutes(15))
        );

        cacheConfigs.put(
                "post",
                defaultConfig.entryTtl(Duration.ofMinutes(5))
        );

        cacheConfigs.put(
                "feed",
                defaultConfig.entryTtl(Duration.ofSeconds(30))
        );

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(defaultConfig)
                .withInitialCacheConfigurations(cacheConfigs)
                .transactionAware()
                .build();
    }
}
```

---

## 20. Redis Cache Key Format

Spring Cache Redis me key aise banti hai:

```text
<cache-name>::<key>
```

Example:

```text
userProfile::101
post::201
feed::101
```

Agar prefix enabled hai:

```text
app:user-service:userProfile::101
```

---

## 21. Production Caching Checklist

- [ ] Har cache entry ka TTL hai
- [ ] TTL jitter use ho raha hai
- [ ] Null caching strategy defined hai
- [ ] Hot keys identify kiye hain
- [ ] Stampede protection hai
- [ ] Cache invalidation write path me hai
- [ ] Cache key naming consistent hai
- [ ] Sensitive data cache nahi ho raha
- [ ] Cache metrics monitor ho rahe hain
- [ ] Redis failure fallback defined hai

---

## 22. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| TTL na dena | Memory full / stale data | TTL + jitter |
| Har update par `allEntries = true` | Unnecessary cache loss | Specific key evict |
| Null result cache na karna | Cache penetration | Short TTL null caching |
| Hot key par no locking | Cache stampede | `sync = true` / distributed lock |
| Same TTL for all keys | Cache avalanche | TTL jitter |
| Entity directly cache karna | Lazy loading / serialization issue | DTO cache karo |
| Cache ko source of truth samajhna | Data loss/stale data risk | DB source of truth |

---

## 23. Interview Questions

| Question | Answer |
|---|---|
| Cache hit kya hai? | Requested data cache me mil jana. |
| Cache miss kya hai? | Requested data cache me na milna. |
| Cache-aside pattern kya hai? | Application cache check karta hai; miss par DB se load karke cache me store karta hai. |
| Write-through kya hai? | Cache aur database dono synchronously update hote hain. |
| Write-behind kya hai? | Pehle cache update, database update asynchronously. |
| Cache stampede kya hai? | Ek hot key expire hone par bahut saare requests DB par chale jana. |
| Cache avalanche kya hai? | Bahut saari keys ek saath expire hone par DB par heavy load aana. |
| Cache penetration kya hai? | Aise non-existing keys ki repeated requests jo DB tak jati hain. |
| `@Cacheable` kya karta hai? | Method result ko cache karta hai aur subsequent calls par method skip kar sakta hai. |
| `@CachePut` kab use karte hain? | Jab method hamesha execute ho aur uska result cache me update ho. |
| `@CacheEvict` kab use karte hain? | Jab data change/delete hone par cache invalidation chahiye. |

---

## 24. Final Mental Model

```text
Read-heavy data      -> Cache-Aside
Strong consistency   -> Write-Through
High write volume    -> Write-Behind
Hot key              -> Locking / sync
Many key expiry      -> TTL jitter
Non-existing key     -> Null caching / Bloom filter
```

---

## Next Part

**Part 9: `@Cacheable`, `@CachePut`, `@CacheEvict` — Complete Deep Dive**

Isme hum cover karenge:

- Har annotation ka internal working
- SpEL expressions
- Cache keys
- `condition` vs `unless`
- `sync = true`
- `allEntries`
- `beforeInvocation`
- Multiple cache names
- Self-invocation problem
- Proxy working
- Real user/profile/post examples
- Common mistakes
- Interview questions
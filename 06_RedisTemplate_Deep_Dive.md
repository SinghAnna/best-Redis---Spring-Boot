# Part 6: RedisTemplate Deep Dive — Complete Deep Dive

> `RedisTemplate` Spring Data Redis ka central API hai. Iske through hum Redis ke saare data structures — String, Hash, List, Set, Sorted Set aur Stream — ko Java code se operate karte hain.

---

## 1. RedisTemplate Kya Hai?

`RedisTemplate<K, V>` ek helper class hai jo Redis data access simplify karta hai.

```text
K = Key type
V = Value type
```

Example:

```java
RedisTemplate<String, Object> redisTemplate;
```

Meaning:

```text
Key   -> String
Value -> Any Java object
```

`RedisTemplate` objects aur Redis binary data ke beech automatic serialization/deserialization karta hai. [91]

---

## 2. RedisTemplate Internal Flow

```text
Java Code
   |
   | redisTemplate.opsForValue().set(key, value)
   v
RedisTemplate
   |
   +--> Key Serializer
   |
   +--> Value Serializer
   |
   +--> RedisConnectionFactory
   |
   v
Lettuce / Jedis
   |
   v
Redis Server
```

### Read Flow

```text
Redis Server
   |
   v
Lettuce / Jedis
   |
   v
RedisTemplate
   |
   +--> Deserialize key
   |
   +--> Deserialize value
   |
   v
Java Object
```

---

## 3. RedisTemplate vs StringRedisTemplate

| Feature | `RedisTemplate<String, Object>` | `StringRedisTemplate` |
|---|---|---|
| Key type | String | String |
| Value type | Object | String |
| Serializer | Configurable | String serializer |
| Java object support | Yes | No |
| JSON object | Yes, with JSON serializer | Manual JSON conversion |
| Best for | DTOs, entities, complex objects | Tokens, counters, simple values |

### Example

```java
// RedisTemplate
redisTemplate.opsForValue().set("user:101", userProfile);

// StringRedisTemplate
stringRedisTemplate.opsForValue().set("user:101", "{\"name\":\"Rahul\"}");
```

---

## 4. RedisTemplate Configuration

```java
package com.example.config;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisTemplateConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory
    ) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        StringRedisSerializer keySerializer = new StringRedisSerializer();
        GenericJackson2JsonRedisSerializer valueSerializer =
                new GenericJackson2JsonRedisSerializer(objectMapper());

        template.setKeySerializer(keySerializer);
        template.setHashKeySerializer(keySerializer);

        template.setValueSerializer(valueSerializer);
        template.setHashValueSerializer(valueSerializer);

        template.afterPropertiesSet();

        return template;
    }

    private ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.activateDefaultTyping(
                LaissezFaireSubTypeValidator.instance,
                ObjectMapper.DefaultTyping.NON_FINAL,
                JsonTypeInfo.As.PROPERTY
        );
        return mapper;
    }
}
```

`RedisTemplate` default me Java-based serializer use karta hai, lekin `StringRedisSerializer`, Jackson JSON serializers aur other implementations use karke serialization change ki ja sakti hai. [89]

---

## 5. RedisTemplate Operations Interfaces

| Method | Redis Data Type | Main Use |
|---|---|---|
| `opsForValue()` | STRING | Cache, token, counter |
| `opsForHash()` | HASH | Object fields |
| `opsForList()` | LIST | Queue, recent items |
| `opsForSet()` | SET | Unique values |
| `opsForZSet()` | SORTED SET | Leaderboard |
| `opsForStream()` | STREAM | Event processing |
| `opsForGeo()` | GEO | Location data |
| `opsForHyperLogLog()` | HYPERLOGLOG | Unique count |

`RedisTemplate` `ValueOperations`, `HashOperations`, `ListOperations`, `SetOperations`, `ZSetOperations` jaise operation interfaces expose karta hai. [89]

---

## 6. `opsForValue()` — STRING Operations

### Save Value

```java
redisTemplate.opsForValue().set("user:101:name", "Rahul");
```

### Save With TTL

```java
redisTemplate.opsForValue().set(
        "user:101:profile",
        userProfile,
        Duration.ofMinutes(15)
);
```

### Get Value

```java
UserProfile profile =
        (UserProfile) redisTemplate.opsForValue().get("user:101:profile");
```

### Increment

```java
Long likes = redisTemplate.opsForValue().increment("post:201:likes");
```

### Increment By Value

```java
Long views = redisTemplate.opsForValue().increment("video:301:views", 5);
```

### Set If Absent

```java
Boolean acquired = redisTemplate.opsForValue()
        .setIfAbsent("lock:order:1001", "instance-1", Duration.ofSeconds(10));
```

### Delete

```java
redisTemplate.delete("user:101:profile");
```

---

## 7. Complete String Service

```java
package com.example.service;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Optional;

@Service
@RequiredArgsConstructor
public class StringCacheService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void save(String key, Object value, Duration ttl) {
        redisTemplate.opsForValue().set(key, value, ttl);
    }

    public <T> Optional<T> get(String key, Class<T> type) {
        Object value = redisTemplate.opsForValue().get(key);
        return Optional.ofNullable(type.cast(value));
    }

    public boolean delete(String key) {
        return Boolean.TRUE.equals(redisTemplate.delete(key));
    }

    public boolean exists(String key) {
        return Boolean.TRUE.equals(redisTemplate.hasKey(key));
    }

    public Long increment(String key) {
        return redisTemplate.opsForValue().increment(key);
    }

    public boolean acquireLock(String key, String ownerId, Duration ttl) {
        return Boolean.TRUE.equals(
                redisTemplate.opsForValue()
                        .setIfAbsent(key, ownerId, ttl)
        );
    }
}
```

---

## 8. `opsForHash()` — HASH Operations

### Save Fields

```java
String key = "user:101:profile";

redisTemplate.opsForHash().put(key, "name", "Rahul");
redisTemplate.opsForHash().put(key, "city", "Bengaluru");
redisTemplate.opsForHash().put(key, "age", 22);
```

### Save Multiple Fields

```java
Map<String, Object> fields = Map.of(
        "name", "Rahul",
        "city", "Bengaluru",
        "age", 22
);

redisTemplate.opsForHash().putAll(key, fields);
```

### Get One Field

```java
Object name = redisTemplate.opsForHash().get(key, "name");
```

### Get All Fields

```java
Map<Object, Object> profile =
        redisTemplate.opsForHash().entries(key);
```

### Delete Field

```java
redisTemplate.opsForHash().delete(key, "age");
```

### Increment Field

```java
redisTemplate.opsForHash().increment(key, "loginCount", 1);
```

---

## 9. Complete Hash Service

```java
package com.example.service;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Map;

@Service
@RequiredArgsConstructor
public class UserProfileHashService {

    private final RedisTemplate<String, Object> redisTemplate;

    private static final String KEY_PREFIX = "user-service:user:";

    public void saveProfile(Long userId, Map<String, Object> profile) {
        String key = KEY_PREFIX + userId + ":profile";

        redisTemplate.opsForHash().putAll(key, profile);
        redisTemplate.expire(key, Duration.ofMinutes(30));
    }

    public Map<Object, Object> getProfile(Long userId) {
        String key = KEY_PREFIX + userId + ":profile";
        return redisTemplate.opsForHash().entries(key);
    }

    public Object getField(Long userId, String field) {
        String key = KEY_PREFIX + userId + ":profile";
        return redisTemplate.opsForHash().get(key, field);
    }

    public void deleteField(Long userId, String field) {
        String key = KEY_PREFIX + userId + ":profile";
        redisTemplate.opsForHash().delete(key, field);
    }

    public Long incrementLoginCount(Long userId) {
        String key = KEY_PREFIX + userId + ":profile";
        return redisTemplate.opsForHash().increment(key, "loginCount", 1);
    }
}
```

---

## 10. `opsForList()` — LIST Operations

### Add From Left

```java
redisTemplate.opsForList().leftPush(
        "user:101:notifications",
        "You got a new follower"
);
```

### Add From Right

```java
redisTemplate.opsForList().rightPush(
        "task-queue",
        "send-email-task"
);
```

### Get Range

```java
List<Object> notifications = redisTemplate.opsForList()
        .range("user:101:notifications", 0, 19);
```

### Pop From Left

```java
Object task = redisTemplate.opsForList()
        .leftPop("task-queue");
```

### Trim List

```java
redisTemplate.opsForList().trim(
        "user:101:notifications",
        0,
        99
);
```

---

## 11. Complete List Service

```java
package com.example.service;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.List;

@Service
@RequiredArgsConstructor
public class NotificationRedisService {

    private final RedisTemplate<String, Object> redisTemplate;

    private static final String KEY_PREFIX = "user-service:user:";

    public void addNotification(Long userId, String message) {
        String key = KEY_PREFIX + userId + ":notifications";

        redisTemplate.opsForList().leftPush(key, message);
        redisTemplate.opsForList().trim(key, 0, 99);
        redisTemplate.expire(key, Duration.ofDays(7));
    }

    public List<Object> getRecentNotifications(Long userId) {
        String key = KEY_PREFIX + userId + ":notifications";
        return redisTemplate.opsForList().range(key, 0, 19);
    }

    public Long notificationCount(Long userId) {
        String key = KEY_PREFIX + userId + ":notifications";
        return redisTemplate.opsForList().size(key);
    }
}
```

---

## 12. `opsForSet()` — SET Operations

### Add Members

```java
redisTemplate.opsForSet().add(
        "user:101:followers",
        "201",
        "202",
        "203"
);
```

### Check Membership

```java
Boolean isFollower = redisTemplate.opsForSet()
        .isMember("user:101:followers", "201");
```

### Get Size

```java
Long followerCount = redisTemplate.opsForSet()
        .size("user:101:followers");
```

### Remove Member

```java
redisTemplate.opsForSet().remove(
        "user:101:followers",
        "202"
);
```

### Intersection

```java
Set<Object> mutualFollowers = redisTemplate.opsForSet()
        .intersect("user:101:followers", "user:201:followers");
```

---

## 13. Complete Set Service

```java
package com.example.service;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.Set;

@Service
@RequiredArgsConstructor
public class FollowRedisService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void follow(Long followerId, Long followingId) {
        redisTemplate.opsForSet().add(
                "user-service:user:" + followingId + ":followers",
                followerId.toString()
        );

        redisTemplate.opsForSet().add(
                "user-service:user:" + followerId + ":following",
                followingId.toString()
        );
    }

    public void unfollow(Long followerId, Long followingId) {
        redisTemplate.opsForSet().remove(
                "user-service:user:" + followingId + ":followers",
                followerId.toString()
        );

        redisTemplate.opsForSet().remove(
                "user-service:user:" + followerId + ":following",
                followingId.toString()
        );
    }

    public boolean isFollowing(Long followerId, Long followingId) {
        return Boolean.TRUE.equals(
                redisTemplate.opsForSet().isMember(
                        "user-service:user:" + followingId + ":followers",
                        followerId.toString()
                )
        );
    }

    public Long followerCount(Long userId) {
        return redisTemplate.opsForSet()
                .size("user-service:user:" + userId + ":followers");
    }

    public Set<Object> mutualFollowers(Long userA, Long userB) {
        return redisTemplate.opsForSet().intersect(
                "user-service:user:" + userA + ":following",
                "user-service:user:" + userB + ":followers"
        );
    }
}
```

---

## 14. `opsForZSet()` — SORTED SET Operations

### Add Member With Score

```java
redisTemplate.opsForZSet().add(
        "leaderboard",
        "user:101",
        950.0
);
```

### Increment Score

```java
Double newScore = redisTemplate.opsForZSet()
        .incrementScore("leaderboard", "user:101", 50.0);
```

### Get Score

```java
Double score = redisTemplate.opsForZSet()
        .score("leaderboard", "user:101");
```

### Get Rank

```java
Long rank = redisTemplate.opsForZSet()
        .rank("leaderboard", "user:101");
```

### Get Reverse Rank

```java
Long topRank = redisTemplate.opsForZSet()
        .reverseRank("leaderboard", "user:101");
```

### Get Top 10

```java
Set<Object> topUsers = redisTemplate.opsForZSet()
        .reverseRange("leaderboard", 0, 9);
```

---

## 15. Complete Leaderboard Service

```java
package com.example.service;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.Set;

@Service
@RequiredArgsConstructor
public class LeaderboardRedisService {

    private final RedisTemplate<String, Object> redisTemplate;

    private static final String LEADERBOARD_KEY =
            "social-service:leaderboard:weekly";

    public void addScore(Long userId, double score) {
        redisTemplate.opsForZSet().incrementScore(
                LEADERBOARD_KEY,
                userId.toString(),
                score
        );
    }

    public Double getScore(Long userId) {
        return redisTemplate.opsForZSet()
                .score(LEADERBOARD_KEY, userId.toString());
    }

    public Long getRank(Long userId) {
        return redisTemplate.opsForZSet()
                .reverseRank(LEADERBOARD_KEY, userId.toString());
    }

    public Set<Object> getTopUsers(int limit) {
        return redisTemplate.opsForZSet()
                .reverseRange(LEADERBOARD_KEY, 0, limit - 1);
    }

    public void removeUser(Long userId) {
        redisTemplate.opsForZSet()
                .remove(LEADERBOARD_KEY, userId.toString());
    }
}
```

---

## 16. `opsForStream()` — STREAM Operations

### Publish Event

```java
Map<String, String> event = Map.of(
        "orderId", "1001",
        "userId", "101",
        "status", "CREATED"
);

redisTemplate.opsForStream().add("order-service:orders", event);
```

### Read Events

```java
List<MapRecord<String, Object, Object>> records =
        redisTemplate.opsForStream()
                .read(StreamOffset.create("order-service:orders", ReadOffset.lastConsumed()));
```

### Acknowledge Event

```java
redisTemplate.opsForStream()
        .acknowledge("order-service:orders", "order-processors", recordId);
```

Redis Streams append-only log ki tarah events record karte hain aur consumer groups ke through distributed consumption support karte hain.

---

## 17. Pipelining With RedisTemplate

`executePipelined` multiple commands ko ek pipeline me run karta hai aur results return karta hai. `RedisCallback` ka return value `null` hona chahiye kyunki actual pipelined results template return karta hai. [90]

```java
package com.example.service;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisCallback;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.nio.charset.StandardCharsets;
import java.util.List;

@Service
@RequiredArgsConstructor
public class PipelineRedisService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void warmUpCache() {
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (int userId = 1; userId <= 1000; userId++) {
                String key = "user-service:user:" + userId + ":status";
                byte[] keyBytes = key.getBytes(StandardCharsets.UTF_8);
                byte[] valueBytes = "active".getBytes(StandardCharsets.UTF_8);

                connection.stringCommands().set(keyBytes, valueBytes);
            }

            return null;
        });
    }
}
```

---

## 18. TTL Operations

### Set TTL

```java
redisTemplate.expire("user:101:profile", Duration.ofMinutes(15));
```

### Get TTL

```java
Long ttlSeconds = redisTemplate.getExpire("user:101:profile");
```

### Remove TTL

```java
redisTemplate.persist("user:101:profile");
```

### Check Key Exists

```java
Boolean exists = redisTemplate.hasKey("user:101:profile");
```

---

## 19. Production Redis Service Template

```java
package com.example.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Optional;

@Slf4j
@Service
@RequiredArgsConstructor
public class RedisCacheService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void put(String key, Object value, Duration ttl) {
        try {
            redisTemplate.opsForValue().set(key, value, ttl);
        } catch (Exception ex) {
            log.warn("Redis put failed. key={}", key, ex);
        }
    }

    public <T> Optional<T> get(String key, Class<T> type) {
        try {
            Object value = redisTemplate.opsForValue().get(key);
            return Optional.ofNullable(type.cast(value));
        } catch (Exception ex) {
            log.warn("Redis get failed. key={}", key, ex);
            return Optional.empty();
        }
    }

    public boolean delete(String key) {
        try {
            return Boolean.TRUE.equals(redisTemplate.delete(key));
        } catch (Exception ex) {
            log.warn("Redis delete failed. key={}", key, ex);
            return false;
        }
    }

    public Long increment(String key, Duration ttl) {
        Long count = redisTemplate.opsForValue().increment(key);

        if (count != null && count == 1L) {
            redisTemplate.expire(key, ttl);
        }

        return count;
    }
}
```

---

## 20. Serializer Best Practices

| Serializer | Use Case | Caution |
|---|---|---|
| `StringRedisSerializer` | Keys, simple string values | Object store nahi kar sakta |
| `GenericJackson2JsonRedisSerializer` | General JSON object caching | Type metadata store hoti hai |
| `Jackson2JsonRedisSerializer` | Fixed known class | Class-specific |
| `JdkSerializationRedisSerializer` | Legacy Java-only apps | Cross-language compatibility issue |
| `GenericToStringSerializer` | Simple conversions | Limited use |

Production me mostly recommended approach:

```text
Key -> StringRedisSerializer
Value -> JSON serializer
```

---

## 21. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Default JDK serialization | Cross-service compatibility issue | JSON serializer |
| Wrong serializer on read/write | Deserialization exception | Same serializer both sides |
| Raw type casting | `ClassCastException` | DTO mapping / type-safe service |
| TTL set na karna | Memory leak | TTL with every cache write |
| Large object caching | Network/memory overhead | Smaller DTO cache karo |
| Business logic in Redis service | Tight coupling | Cache layer separate rakho |
| No exception handling | Redis outage app down | Fallback + logging |

---

## 22. Interview Questions

| Question | Answer |
|---|---|
| `RedisTemplate` kya hai? | Redis operations ke liye thread-safe, high-level Spring abstraction. |
| `RedisTemplate` aur `StringRedisTemplate` me difference? | `RedisTemplate` generic object values support karta hai; `StringRedisTemplate` String values ke liye hota hai. |
| `opsForValue()` kis Redis type ke liye hai? | STRING. |
| `opsForHash()` kis Redis type ke liye hai? | HASH. |
| `opsForZSet()` kab use karte hain? | Leaderboard, ranking aur score-based data ke liye. |
| Serializer kyun important hai? | Java object ko Redis binary data me convert karne aur wapas deserialize karne ke liye. |
| Default RedisTemplate serializer kya hai? | Java-based serialization. |
| Pipelining kya hai? | Multiple Redis commands ko ek network round-trip me bhejna. |
| Distributed lock `setIfAbsent` se kaise banate hain? | `SET key value NX PX ttl` ke equivalent se. |

---

## 23. Final Mental Model

```text
RedisTemplate = Java application ka Redis driver wrapper

opsForValue  -> STRING
opsForHash   -> HASH
opsForList   -> LIST
opsForSet    -> SET
opsForZSet   -> SORTED SET
opsForStream -> STREAM

Serializer   -> Java object <-> Redis bytes
TTL          -> Automatic expiration
Pipeline     -> Batch performance
```

---

## Next Part

**Part 7: Redis Serialization in Spring Boot — Complete Deep Dive**

Isme hum cover karenge:

- Serialization kya hai
- JDK serialization problems
- JSON serialization
- `StringRedisSerializer`
- `GenericJackson2JsonRedisSerializer`
- `Jackson2JsonRedisSerializer`
- `LocalDateTime` handling
- Polymorphic objects
- Backward compatibility
- Cache versioning
- Production serializer strategy
- Common serialization errors
- Interview questions
# Part 11: Cache Invalidation Strategies — Complete Deep Dive

> Cache invalidation production me sabse tricky part hai. TTL se memory control hota hai, lekin **data freshness** ke liye correct invalidation strategy chahiye hoti hai.

---

## 1. Cache Invalidation Kya Hai?

Cache invalidation ka matlab hai:

```text
Jab database me data change ho jaye,
to Redis me stored purana/stale data
remove ya refresh karna.
```

### Example

```text
Redis:
userProfile::101 -> "Rahul"

Database update:
user 101 name -> "Rahul Sharma"

Problem:
Redis still returns "Rahul"
= Stale cache
```

### Solution

```text
DB update ke baad
Redis key delete/update karo.
```

Cache invalidation data ko cache se remove karta hai jab wo valid ya useful nahi rehta; event-based invalidation me specific event trigger hone par cache invalidate hota hai. [159]

---

## 2. Stale Cache Problem

### Flow Without Invalidation

```text
1. GET /users/101
2. Redis returns old profile
3. User sees old name
4. DB has new name
5. User confused
```

### Flow With Invalidation

```text
1. Update profile in DB
2. Delete/update Redis cache
3. Next request loads fresh data
4. Cache repopulated
```

---

## 3. Invalidation vs Eviction

| Concept | Purpose | Trigger |
|---|---|---|
| Invalidation | Stale data remove karna | Data change |
| Eviction | Memory free karna | Memory limit / capacity |

Cache invalidation stale data remove karne ke liye hota hai; eviction memory capacity full hone par entries remove karne ke liye hota hai. [172]

---

## 4. TTL-Based Invalidation

TTL-based invalidation sabse simple strategy hai.

```java
redisTemplate.opsForValue().set(
        "user-service:user:101:profile",
        userProfile,
        Duration.ofMinutes(15)
);
```

### Flow

```text
Cache write with TTL 15 minutes
        |
        +--> 15 minutes later
        |
        +--> Redis deletes key
        |
        +--> Next request -> DB -> Fresh cache
```

### Advantages

- Simple
- No extra infrastructure
- Good for non-critical data
- Protects against missed invalidation events

### Disadvantages

- TTL expire hone tak stale data possible
- Critical data ke liye insufficient
- Sudden expiry se cache stampede ho sakta hai

TTL-based expiration me cached value predefined time ke baad expire ho jati hai, rather than data change par turant invalidate hona. [160]

### Best For

| Data | TTL Strategy |
|---|---|
| Product catalog | 30 minutes |
| User profile | 10–30 minutes |
| Trending posts | 1–5 minutes |
| Feed | 30 seconds–5 minutes |
| Config | 5–15 minutes |
| Non-critical analytics | 1–5 minutes |

---

## 5. Write-Through Invalidation

Write-through me DB update ke saath cache bhi update ya invalidate hota hai.

### Update Cache

```java
@Transactional
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
```

### Delete Cache

```java
@Transactional
public UserProfileResponse updateProfile(Long userId, UpdateProfileRequest request) {
    UserProfile user = userRepository.findById(userId)
            .orElseThrow();

    user.setName(request.getName());
    user.setCity(request.getCity());

    UserProfile saved = userRepository.save(user);

    redisTemplate.delete(
            "user-service:user:" + userId + ":profile"
    );

    return UserProfileResponse.from(saved);
}
```

### Delete vs Update Cache

| Strategy | Advantage | Risk |
|---|---|---|
| Update cache | Next read fast | Complex object me partial update risk |
| Delete cache | Simple, always fresh next read | Next request me cache miss |

### Recommended Default

```text
Delete cache on write.
Let next read repopulate it.
```

---

## 6. Delete Cache: Correct Order

### Recommended Order

```text
1. Update database
2. Delete cache
3. Commit transaction
```

### Why?

Agar pehle cache delete karo aur DB update fail ho jaye:

```text
Cache deleted
DB update failed
Old data still in DB
Next request reloads old data
```

Ye usually acceptable hai.

Lekin agar DB update ho jaye aur cache delete fail ho jaye:

```text
DB updated
Cache still old
Stale cache until TTL
```

Isliye:

```text
DB update -> Cache delete -> TTL as safety net
```

---

## 7. Event-Based Invalidation

Event-based invalidation me data change hone par event publish hota hai, aur interested services us event ko consume karke apni cache invalidate karti hain. [159][160]

### Architecture

```text
User Service
     |
     | DB update
     |
     +--> Kafka: user-updated
                 |
                 +--> Feed Service -> invalidate user feed cache
                 |
                 +--> Post Service -> invalidate author cache
                 |
                 +--> Notification Service -> invalidate user cache
```

### Why Kafka?

- Multiple microservices ko notify karna
- Reliable event delivery
- Replay capability
- Decoupled services
- Distributed invalidation

Event-based invalidation me services data change par events publish karti hain, aur dusre nodes/services us event ko listen karke cache invalidate karti hain. [160]

---

## 8. Kafka-Based Cache Invalidation Example

### Event DTO

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserUpdatedEvent {

    private Long userId;
    private String name;
    private String city;
    private LocalDateTime updatedAt;
}
```

### Producer

```java
package com.example.producer;

import com.example.event.UserUpdatedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class UserEventProducer {

    private final KafkaTemplate<String, UserUpdatedEvent> kafkaTemplate;

    private static final String TOPIC = "user-service.user-updated";

    public void publishUserUpdated(UserUpdatedEvent event) {
        kafkaTemplate.send(
                TOPIC,
                event.getUserId().toString(),
                event
        );

        log.info("Published user-updated event for userId={}", event.getUserId());
    }
}
```

### Consumer in Feed Service

```java
package com.example.consumer;

import com.example.event.UserUpdatedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class UserCacheInvalidationConsumer {

    private final RedisTemplate<String, Object> redisTemplate;

    @KafkaListener(
            topics = "user-service.user-updated",
            groupId = "feed-service-cache-invalidator"
    )
    public void handleUserUpdated(UserUpdatedEvent event) {
        String profileKey =
                "feed-service:user:" + event.getUserId() + ":profile";

        String feedKey =
                "feed-service:user:" + event.getUserId() + ":feed";

        redisTemplate.delete(profileKey);
        redisTemplate.delete(feedKey);

        log.info(
                "Invalidated cache for userId={}",
                event.getUserId()
        );
    }
}
```

### Flow

```text
User Service
    |
    +--> Update DB
    |
    +--> Publish Kafka event
             |
             v
       Feed Service Consumer
             |
             +--> Delete Redis keys
```

---

## 9. Redis Pub/Sub Invalidation

Redis Pub/Sub lightweight invalidation ke liye use ho sakta hai.

### Publisher

```java
redisTemplate.convertAndSend(
        "cache-invalidation",
        "user-service:user:101:profile"
);
```

### Subscriber

```java
@Component
@RequiredArgsConstructor
public class CacheInvalidationSubscriber {

    private final RedisTemplate<String, Object> redisTemplate;

    public void handleMessage(String message) {
        redisTemplate.delete(message);
    }
}
```

### Pub/Sub Limitation

```text
Agar subscriber offline ho,
message miss ho sakta hai.
```

Isliye critical invalidation ke liye Kafka better hai.

Redis Pub/Sub distributed systems me invalidation events broadcast kar sakta hai, lekin offline subscriber message miss kar sakta hai. [162]

---

## 10. Cache Versioning

Cache versioning me data delete karne ki jagah cache key ka version change kar dete hain.

### Example

```text
Old key:
user-service:v1:user:101:profile

New key:
user-service:v2:user:101:profile
```

### Flow

```text
1. Data update
2. Version increment: v1 -> v2
3. Application ab v2 key use karta hai
4. v1 keys TTL se expire
```

Versioned cache keys me data delete karne ki jagah incremented version ke saath nayi cache key create hoti hai. [160]

### Java Example

```java
@Service
@RequiredArgsConstructor
public class VersionedProfileService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final UserRepository userRepository;

    private static final String CURRENT_VERSION = "v2";

    public UserProfileResponse getProfile(Long userId) {
        String key = profileKey(userId);

        Object cached = redisTemplate.opsForValue().get(key);

        if (cached != null) {
            return (UserProfileResponse) cached;
        }

        UserProfileResponse profile = userRepository.findById(userId)
                .map(UserProfileResponse::from)
                .orElseThrow();

        redisTemplate.opsForValue().set(
                key,
                profile,
                Duration.ofMinutes(15)
        );

        return profile;
    }

    private String profileKey(Long userId) {
        return "user-service:" + CURRENT_VERSION + ":user:" + userId + ":profile";
    }
}
```

### Version Storage in Redis

```text
user-service:profile-version -> 2
```

### Version Increment

```java
redisTemplate.opsForValue().increment(
        "user-service:profile-version"
);
```

---

## 11. Tag-Based Invalidation

Tag-based invalidation me ek logical group ke multiple cache keys ko ek tag se associate karte hain.

### Example

```text
tag:user:101 -> 
    user:101:profile
    user:101:followers
    user:101:feed
    user:101:notifications
```

### Flow

```text
User 101 update
    |
    +--> Invalidate tag:user:101
    |
    +--> All related cache keys removed
```

### Redis Implementation

```java
public void addToTag(String tag, String cacheKey) {
    redisTemplate.opsForSet().add(tag, cacheKey);
}

public void invalidateTag(String tag) {
    Set<Object> keys = redisTemplate.opsForSet().members(tag);

    if (keys != null && !keys.isEmpty()) {
        redisTemplate.delete(keys);
    }

    redisTemplate.delete(tag);
}
```

---

## 12. Social Media Feed Invalidation

### Scenario

```text
User 101 ne post create kiya.
User 101 ke followers ki feed cache invalidate karni hai.
```

### Problem

```text
User 101 ke 1 million followers hain.

Kya 1 million feed keys delete karenge?
```

### Better Strategy

```text
Feed cache ko user-specific mat banao.

Feed key:
feed:user:201:page:0

Invalidation:
User 101 post create
    |
    +--> Followers ke feed keys selectively invalidate
```

### Practical Approach

```text
1. Feed cache TTL short rakho: 30 seconds
2. New post event publish karo
3. Active followers ke feed cache invalidate karo
4. Inactive followers ki cache TTL se expire hone do
```

### Kafka Consumer

```java
@KafkaListener(
        topics = "post-service.post-created",
        groupId = "feed-service-cache-invalidator"
)
public void handlePostCreated(PostCreatedEvent event) {
    List<Long> activeFollowerIds =
            followerService.getActiveFollowerIds(event.getAuthorId());

    List<String> keys = activeFollowerIds.stream()
            .map(followerId -> "feed-service:user:" + followerId + ":feed")
            .toList();

    redisTemplate.delete(keys);
}
```

---

## 13. Dual-Write Problem

Jab application DB aur cache dono ko update karta hai, to ordering aur failure dono important hote hain.

### Problem Scenario

```text
1. Update cache
2. Update DB fails
3. Cache has new data
4. DB has old data
= Inconsistency
```

### Better Order

```text
1. Update DB
2. Delete cache
3. TTL safety net
```

### Advanced Solution

```text
DB update
    |
    +--> Outbox table
    |
    +--> Kafka event
    |
    +--> Cache invalidation consumer
```

Dual-write problem me DB aur cache ke beech read-modify-write race aur ordering issues aa sakte hain. [164]

---

## 14. Outbox Pattern

### Flow

```text
1. DB transaction me:
   - Business data update
   - Outbox event insert

2. Outbox publisher:
   - Event Kafka me publish karta hai

3. Cache invalidator:
   - Kafka event consume karta hai
   - Redis cache delete karta hai
```

### Benefit

```text
DB commit aur event publish
ek consistent workflow me hote hain.
```

---

## 15. Stale-While-Revalidate

Is strategy me purana cached data turant return hota hai, aur background me fresh data load hota hai.

### Flow

```text
1. Cache entry near expiry
2. Return stale data immediately
3. Background refresh
4. Next request gets fresh data
```

### Benefit

- Low latency
- No user-facing cache miss delay
- DB load smooth

### Trade-off

```text
User ko thoda stale data mil sakta hai.
```

---

## 16. Invalidation Strategy Comparison

| Strategy | Freshness | Complexity | Scalability | Best For |
|---|---|---|---|---|
| TTL | Eventual | Low | High | Non-critical cache |
| Write-through delete | High | Medium | Medium | Important updates |
| Kafka event | High | High | High | Microservices |
| Redis Pub/Sub | High | Medium | Medium | Lightweight invalidation |
| Versioned keys | High | Medium | High | DTO/schema changes |
| Tag-based | High | High | Medium | Related data groups |
| Stale-while-revalidate | Medium | Medium | High | Low-latency reads |

Hybrid approach — TTL + event-driven invalidation — freshness aur fault tolerance dono deta hai. [166]

---

## 17. Production Invalidation Rules

```text
1. Har cache key ka TTL rakho.
2. DB update ke baad relevant cache delete karo.
3. Critical data ke liye event-based invalidation use karo.
4. Multiple services me Kafka use karo.
5. Cache versioning DTO changes ke liye use karo.
6. Related data ke liye tag-based invalidation use karo.
7. Massive invalidation avoid karo.
8. Stale data tolerance define karo.
9. Invalidation failures monitor karo.
10. TTL ko safety net ki tarah treat karo.
```

---

## 18. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Sirf TTL par depend karna | Critical data stale | TTL + event invalidation |
| Cache update before DB update | Inconsistency | DB first, then cache delete |
| Massive cache deletion | Redis/DB load | Selective invalidation |
| No TTL safety net | Permanent stale cache | TTL compulsory |
| Pub/Sub for critical events | Message loss possible | Kafka/Streams |
| DTO change without versioning | Deserialization failure | Cache version prefix |
| Invalidating unrelated keys | Low cache hit ratio | Dependency mapping |

---

## 19. Interview Questions

| Question | Answer |
|---|---|
| Cache invalidation kya hai? | Underlying data change hone par stale cache data remove/refresh karna. |
| Cache invalidation aur eviction me difference? | Invalidation data change se hota hai; eviction memory pressure se hota hai. |
| TTL-based invalidation kya hai? | Cache entry fixed time baad automatically expire hoti hai. |
| Write-through invalidation kya hai? | DB write ke saath cache update/delete karna. |
| Kafka-based invalidation kyun use karte hain? | Multiple microservices ko reliably notify karne ke liye. |
| Cache versioning kya hai? | Data change par cache key version increment karna, purani keys TTL se expire karwana. |
| Dual-write problem kya hai? | DB aur cache dono update karte waqt inconsistency/failure ordering issue. |
| Stale-while-revalidate kya hai? | Stale data turant return karna aur background me fresh data load karna. |
| Social media feed invalidation kaise karte hain? | Short TTL + selective invalidation of active followers’ feed keys. |

---

## 20. Final Mental Model

```text
TTL              -> Safety net
DB update        -> Source of truth
Cache delete     -> Immediate freshness
Kafka event      -> Distributed invalidation
Cache versioning -> Safe schema evolution
Tag invalidation -> Related data cleanup
```

---

## Next Part

**Part 12: User Session Management with Redis — Complete Deep Dive**

Isme hum cover karenge:

- Stateless vs stateful authentication
- JWT vs Redis sessions
- Spring Session Redis
- Refresh token storage
- Token revocation
- Logout implementation
- Multi-device session handling
- Session TTL
- Security best practices
- Production session architecture
- Interview questions
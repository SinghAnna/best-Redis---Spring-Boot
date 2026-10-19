# Part 29: Social Media Backend with Redis — Complete Production Project

> Is project me hum ek production-style **Social Media Backend** banayenge jisme Redis caching, sessions, rate limiting, leaderboards, online presence, Kafka-based cache invalidation aur distributed locks use honge.

---

## 1. Project Overview

### Features

| Feature | Redis Use |
|---|---|
| User profile | Profile cache |
| Follow/unfollow | Followers/following Sets |
| Create post | Feed fan-out |
| Home feed | Redis timeline cache |
| Like post | Atomic counter |
| Trending posts | Sorted Set leaderboard |
| Rate limiting | Redis counter/token bucket |
| Login/session | Refresh token in Redis |
| Online presence | Redis Set/Bitmap |
| Cache invalidation | Kafka events |
| Feed generation lock | Redisson distributed lock |

---

## 2. Architecture

```text
                         +----------------+
                         |   API Gateway  |
                         +--------+-------+
                                  |
          +-----------------------+-----------------------+
          |                       |                       |
+---------v---------+   +---------v---------+   +---------v---------+
|  User Service     |   |  Post Service     |   |  Feed Service     |
+---------+---------+   +---------+---------+   +---------+---------+
          |                       |                       |
          +-----------+-----------+-----------+-----------+
                      |                       |
                      v                       v
               +-------------+         +-------------+
               |    Redis    |         |    Kafka    |
               +------+------+         +------+------+
                      |                       |
                      v                       v
               +-------------+         +-------------+
               | PostgreSQL  |         |  Consumers  |
               +-------------+         +-------------+
```

### Redis Responsibilities

| Service | Redis Keys |
|---|---|
| User Service | Profile cache, followers, following, presence |
| Post Service | Post cache, like counters, trending posts |
| Feed Service | User timeline, feed cache |
| Auth Service | Refresh tokens, token blacklist |
| Notification Service | Pub/Sub notifications |

---

## 3. Feed Strategy

### Hybrid Fan-Out

```text
Normal user creates post
        |
        +--> Fan-out-on-write
        |    Post ID followers ke Redis timeline me push
        |
Celebrity creates post
        |
        +--> Fan-out-on-read
             Feed request time par posts fetch/merge
```

Normal users ke liye fan-out-on-write use karo, celebrity accounts ke liye fan-out-on-read — isse write amplification control hota hai. [408][412]

### Timeline Data Structure

```text
feed:user:201:timeline
  ├── post:1001 -> score: createdAt
  ├── post:1002 -> score: createdAt
  └── post:1003 -> score: createdAt
```

Redis Sorted Set use karo:

```text
ZADD feed:user:201:timeline <createdAtMillis> post:1001
```

Feed timeline ko Sorted Set me store karo, score publish timestamp rakho, aur recent posts ke liye `ZREVRANGE` use karo. [412]

---

## 4. Redis Key Design

### User Service

```text
user-service:user:{userId}:profile
user-service:user:{userId}:followers
user-service:user:{userId}:following
user-service:user:{userId}:presence
```

### Post Service

```text
post-service:post:{postId}:details
post-service:post:{postId}:likes
post-service:trending:posts
```

### Feed Service

```text
feed-service:user:{userId}:timeline
feed-service:user:{userId}:feed
```

### Auth Service

```text
auth-service:refresh-token:{tokenId}
auth-service:user:{userId}:refresh-tokens
auth-service:blacklist:{jti}
```

### Rate Limiting

```text
rate-limit:user:{userId}:endpoint:/posts
rate-limit:ip:{ip}:endpoint:/login
```

---

## 5. Docker Compose Setup

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: social-postgres
    environment:
      POSTGRES_DB: social_media
      POSTGRES_USER: social_user
      POSTGRES_PASSWORD: social_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7.4-alpine
    container_name: social-redis
    command: >
      redis-server
      --requirepass redis-password
      --appendonly yes
      --appendfsync everysec
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "redis-password", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  kafka:
    image: bitnami/kafka:3.7
    container_name: social-kafka
    environment:
      KAFKA_CFG_NODE_ID: 1
      KAFKA_CFG_PROCESS_ROLES: controller,broker
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      ALLOW_PLAINTEXT_LISTENER: "yes"
    ports:
      - "9092:9092"

volumes:
  postgres_data:
  redis_data:
```

---

## 6. Spring Boot Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>

    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson-spring-boot-starter</artifactId>
        <version>3.50.0</version>
    </dependency>

    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

---

## 7. Application Configuration

```yaml
spring:
  application:
    name: social-media-service

  datasource:
    url: jdbc:postgresql://localhost:5432/social_media
    username: social_user
    password: social_password

  jpa:
    hibernate:
      ddl-auto: update
    open-in-view: false

  data:
    redis:
      host: localhost
      port: 6379
      password: redis-password
      timeout: 1s
      connect-timeout: 1s
      lettuce:
        pool:
          enabled: true
          max-active: 32
          max-idle: 16
          min-idle: 4
          max-wait: 500ms

  cache:
    type: redis
    redis:
      time-to-live: 10m
      cache-null-values: false
      use-key-prefix: true
      key-prefix: "social-media:"

  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: social-media-consumer
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.example.social.event"

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  health:
    redis:
      enabled: true
```

---

## 8. Entities

### User

```java
package com.example.social.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "users")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String username;
    private String email;
    private String fullName;
    private String city;
}
```

### Post

```java
package com.example.social.entity;

import jakarta.persistence.*;
import lombok.*;

import java.time.LocalDateTime;

@Entity
@Table(name = "posts")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long authorId;
    private String content;

    private Long likeCount;

    private LocalDateTime createdAt;
}
```

### Follow

```java
package com.example.social.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "follows")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Follow {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long followerId;
    private Long followingId;
}
```

---

## 9. User Profile Caching

```java
package com.example.social.service;

import com.example.social.dto.UserProfileResponse;
import com.example.social.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class UserProfileService {

    private final UserRepository userRepository;

    @Cacheable(
            value = "userProfile",
            key = "#userId",
            unless = "#result == null",
            sync = true
    )
    public UserProfileResponse getProfile(Long userId) {
        return userRepository.findById(userId)
                .map(UserProfileResponse::from)
                .orElse(null);
    }
}
```

### Redis Key

```text
social-media:userProfile::101
```

---

## 10. Follow/Unfollow Service

```java
package com.example.social.service;

import com.example.social.repository.FollowRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class FollowService {

    private final FollowRepository followRepository;
    private final StringRedisTemplate redisTemplate;

    @Transactional
    public void follow(Long followerId, Long followingId) {
        followRepository.save(
                Follow.builder()
                        .followerId(followerId)
                        .followingId(followingId)
                        .build()
        );

        redisTemplate.opsForSet().add(
                "user-service:user:" + followingId + ":followers",
                followerId.toString()
        );

        redisTemplate.opsForSet().add(
                "user-service:user:" + followerId + ":following",
                followingId.toString()
        );
    }

    @Transactional
    public void unfollow(Long followerId, Long followingId) {
        followRepository.deleteByFollowerIdAndFollowingId(
                followerId,
                followingId
        );

        redisTemplate.opsForSet().remove(
                "user-service:user:" + followingId + ":followers",
                followerId.toString()
        );

        redisTemplate.opsForSet().remove(
                "user-service:user:" + followerId + ":following",
                followingId.toString()
        );
    }

    public Long followerCount(Long userId) {
        Long count = redisTemplate.opsForSet()
                .size("user-service:user:" + userId + ":followers");

        return count == null ? 0L : count;
    }
}
```

---

## 11. Post Creation and Feed Fan-Out

### Kafka Event

```java
package com.example.social.event;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PostCreatedEvent {

    private Long postId;
    private Long authorId;
    private LocalDateTime createdAt;
}
```

### Post Service

```java
package com.example.social.service;

import com.example.social.dto.CreatePostRequest;
import com.example.social.dto.PostResponse;
import com.example.social.entity.Post;
import com.example.social.event.PostCreatedEvent;
import com.example.social.repository.PostRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;

@Service
@RequiredArgsConstructor
public class PostService {

    private final PostRepository postRepository;
    private final KafkaTemplate<String, PostCreatedEvent> kafkaTemplate;

    private static final String POST_CREATED_TOPIC =
            "post-service.post-created";

    @Transactional
    public PostResponse createPost(Long authorId, CreatePostRequest request) {
        Post post = postRepository.save(
                Post.builder()
                        .authorId(authorId)
                        .content(request.getContent())
                        .likeCount(0L)
                        .createdAt(LocalDateTime.now())
                        .build()
        );

        PostCreatedEvent event = PostCreatedEvent.builder()
                .postId(post.getId())
                .authorId(authorId)
                .createdAt(post.getCreatedAt())
                .build();

        kafkaTemplate.send(
                POST_CREATED_TOPIC,
                post.getId().toString(),
                event
        );

        return PostResponse.from(post);
    }
}
```

### Feed Fan-Out Consumer

```java
package com.example.social.consumer;

import com.example.social.event.PostCreatedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.util.List;

@Slf4j
@Component
@RequiredArgsConstructor
public class PostCreatedFeedConsumer {

    private final StringRedisTemplate redisTemplate;
    private final FollowService followService;
    private final CelebrityService celebrityService;

    private static final String TIMELINE_PREFIX =
            "feed-service:user:";

    @KafkaListener(
            topics = "post-service.post-created",
            groupId = "feed-service-fanout"
    )
    public void handlePostCreated(PostCreatedEvent event) {
        if (celebrityService.isCelebrity(event.getAuthorId())) {
            log.info(
                    "Celebrity post. Skipping fan-out. postId={}",
                    event.getPostId()
            );
            return;
        }

        List<Long> followerIds =
                followService.getFollowerIds(event.getAuthorId());

        double score = event.getCreatedAt()
                .toInstant(java.time.ZoneOffset.UTC)
                .toEpochMilli();

        for (Long followerId : followerIds) {
            String timelineKey =
                    TIMELINE_PREFIX + followerId + ":timeline";

            redisTemplate.opsForZSet().add(
                    timelineKey,
                    "post:" + event.getPostId(),
                    score
            );

            redisTemplate.opsForZSet().removeRange(
                    timelineKey,
                    0,
                    -801
            );

            redisTemplate.expire(timelineKey, Duration.ofDays(7));
        }
    }
}
```

---

## 12. Feed Service

```java
package com.example.social.service;

import com.example.social.dto.PostResponse;
import com.example.social.repository.PostRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Set;

@Service
@RequiredArgsConstructor
public class FeedService {

    private final StringRedisTemplate redisTemplate;
    private final PostRepository postRepository;

    public List<PostResponse> getHomeFeed(Long userId, int page, int size) {
        String timelineKey =
                "feed-service:user:" + userId + ":timeline";

        int start = page * size;
        int end = start + size - 1;

        Set<String> postIds = redisTemplate.opsForZSet()
                .reverseRange(timelineKey, start, end);

        if (postIds == null || postIds.isEmpty()) {
            return List.of();
        }

        return postIds.stream()
                .map(postId -> postId.replace("post:", ""))
                .map(Long::parseLong)
                .map(postRepository::findById)
                .flatMap(java.util.Optional::stream)
                .map(PostResponse::from)
                .toList();
    }
}
```

---

## 13. Like Counter and Trending Posts

```java
package com.example.social.service;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class PostEngagementService {

    private final StringRedisTemplate redisTemplate;

    public Long likePost(Long postId) {
        Long likes = redisTemplate.opsForValue().increment(
                "post-service:post:" + postId + ":likes"
        );

        redisTemplate.opsForZSet().incrementScore(
                "post-service:trending:posts",
                postId.toString(),
                3.0
        );

        return likes;
    }

    public Long getLikeCount(Long postId) {
        String value = redisTemplate.opsForValue().get(
                "post-service:post:" + postId + ":likes"
        );

        return value == null ? 0L : Long.parseLong(value);
    }

    public void recordComment(Long postId) {
        redisTemplate.opsForZSet().incrementScore(
                "post-service:trending:posts",
                postId.toString(),
                5.0
        );
    }

    public void recordShare(Long postId) {
        redisTemplate.opsForZSet().incrementScore(
                "post-service:trending:posts",
                postId.toString(),
                8.0
        );
    }
}
```

---

## 14. Trending Posts API

```java
package com.example.social.controller;

import com.example.social.service.PostEngagementService;
import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Set;

@RestController
@RequestMapping("/api/trending")
@RequiredArgsConstructor
public class TrendingController {

    private final StringRedisTemplate redisTemplate;

    @GetMapping("/posts")
    public ResponseEntity<List<Long>> getTrendingPosts(
            @RequestParam(defaultValue = "10") int limit
    ) {
        Set<String> postIds = redisTemplate.opsForZSet()
                .reverseRange(
                        "post-service:trending:posts",
                        0,
                        limit - 1
                );

        if (postIds == null) {
            return ResponseEntity.ok(List.of());
        }

        List<Long> result = postIds.stream()
                .map(Long::parseLong)
                .toList();

        return ResponseEntity.ok(result);
    }
}
```

---

## 15. Rate Limiting

```java
package com.example.social.ratelimit;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;

@Service
@RequiredArgsConstructor
public class ApiRateLimiter {

    private final StringRedisTemplate redisTemplate;

    public boolean isAllowed(
            Long userId,
            String endpoint,
            int limit,
            Duration window
    ) {
        long windowNumber =
                Instant.now().toEpochMilli() / window.toMillis();

        String key =
                "rate-limit:user:" + userId + ":" + endpoint + ":" + windowNumber;

        Long count = redisTemplate.opsForValue().increment(key);

        if (count != null && count == 1L) {
            redisTemplate.expire(key, window);
        }

        return count != null && count <= limit;
    }
}
```

### Controller Usage

```java
@PostMapping
public ResponseEntity<PostResponse> createPost(
        @RequestHeader("X-User-Id") Long userId,
        @RequestBody CreatePostRequest request
) {
    boolean allowed = apiRateLimiter.isAllowed(
            userId,
            "create-post",
            20,
            Duration.ofMinutes(1)
    );

    if (!allowed) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
                .body(null);
    }

    return ResponseEntity.ok(postService.createPost(userId, request));
}
```

---

## 16. Online Presence

```java
package com.example.social.presence;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.LocalDate;

@Service
@RequiredArgsConstructor
public class PresenceService {

    private final StringRedisTemplate redisTemplate;

    public void markOnline(Long userId) {
        String key =
                "user-service:presence:" + LocalDate.now();

        redisTemplate.opsForValue().setBit(
                key,
                userId,
                true
        );

        redisTemplate.expire(key, Duration.ofDays(2));
    }

    public boolean isOnline(Long userId) {
        String key =
                "user-service:presence:" + LocalDate.now();

        return Boolean.TRUE.equals(
                redisTemplate.opsForValue().getBit(key, userId)
        );
    }
}
```

---

## 17. Feed Generation Lock

```java
package com.example.social.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class FeedGenerationService {

    private final RedissonClient redissonClient;
    private final FeedService feedService;

    public void regenerateFeed(Long userId) {
        String lockKey =
                "feed-service:lock:feed:" + userId;

        RLock lock = redissonClient.getLock(lockKey);

        boolean acquired = false;

        try {
            acquired = lock.tryLock(
                    2,
                    10,
                    TimeUnit.SECONDS
            );

            if (!acquired) {
                log.info("Feed generation already running for user {}", userId);
                return;
            }

            feedService.getHomeFeed(userId, 0, 20);

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Feed generation interrupted", e);
        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

---

## 18. Cache Invalidation

### User Updated Event

```java
@KafkaListener(
        topics = "user-service.user-updated",
        groupId = "social-media-cache-invalidator"
)
public void handleUserUpdated(UserUpdatedEvent event) {
    redisTemplate.delete(
            "social-media:userProfile::" + event.getUserId()
    );

    redisTemplate.delete(
            "feed-service:user:" + event.getUserId() + ":feed"
    );
}
```

### Post Updated Event

```java
@KafkaListener(
        topics = "post-service.post-updated",
        groupId = "social-media-cache-invalidator"
)
public void handlePostUpdated(PostUpdatedEvent event) {
    redisTemplate.delete(
            "social-media:post::" + event.getPostId()
    );

    redisTemplate.delete(
            "post-service:post:" + event.getPostId() + ":details"
    );
}
```

Cache-aside pattern me DB update ke baad cache invalidate karna recommended hota hai; event-driven invalidation multi-service consistency ke liye useful hai. [409][413]

---

## 19. API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/users/{userId}` | Get cached user profile |
| `POST` | `/api/users/{followerId}/follow/{followingId}` | Follow user |
| `DELETE` | `/api/users/{followerId}/follow/{followingId}` | Unfollow user |
| `POST` | `/api/posts` | Create post |
| `GET` | `/api/feed` | Get home feed |
| `POST` | `/api/posts/{postId}/like` | Like post |
| `GET` | `/api/trending/posts` | Get trending posts |
| `POST` | `/api/auth/login` | Login |
| `POST` | `/api/auth/refresh` | Refresh token |
| `POST` | `/api/auth/logout` | Logout |
| `GET` | `/actuator/health` | Health check |
| `GET` | `/actuator/prometheus` | Metrics |

---

## 20. Monitoring

### Redis Metrics

```text
redis_memory_used_bytes
redis_memory_max_bytes
redis_keyspace_hits_total
redis_keyspace_misses_total
redis_evicted_keys_total
redis_connected_clients
redis_commands_processed_total
```

### Application Metrics

```text
cache_hit_total
cache_miss_total
redis_failure_total
feed_generation_duration_seconds
post_created_events_total
rate_limit_blocked_total
```

### Grafana Panels

| Panel | Query |
|---|---|
| Cache Hit Ratio | `rate(cache_hit_total[5m]) / (rate(cache_hit_total[5m]) + rate(cache_miss_total[5m])) * 100` |
| Redis Memory Usage | `redis_memory_used_bytes / redis_memory_max_bytes * 100` |
| Feed Latency | `http_server_requests_seconds_bucket{uri="/api/feed"}` |
| Rate Limit Blocks | `rate(rate_limit_blocked_total[5m])` |
| Kafka Consumer Lag | `kafka_consumergroup_lag` |

---

## 21. Production Checklist

- [ ] Redis Cluster/managed Redis use ho raha hai.
- [ ] Har cache key ka TTL hai.
- [ ] Feed timeline max 800 posts tak trim hota hai.
- [ ] Celebrity posts fan-out-on-read se handle hote hain.
- [ ] Kafka consumer lag monitor ho raha hai.
- [ ] Redis failure par DB fallback hai.
- [ ] Rate limiting enabled hai.
- [ ] Refresh tokens revocable hain.
- [ ] Distributed locks me idempotency hai.
- [ ] Trending leaderboard ka TTL hai.
- [ ] Prometheus/Grafana dashboards hain.
- [ ] Load testing done hai.
- [ ] Backup and disaster recovery configured hai.

---

## 22. Interview Questions

| Question | Answer |
|---|---|
| Social media feed me Redis kaise use hota hai? | User timeline/feed caching, fan-out, trending posts aur engagement counters ke liye. |
| Fan-out-on-write kya hai? | Post create hone par usko followers ke Redis timelines me push karna. |
| Fan-out-on-read kya hai? | Celebrity posts ko feed request time par fetch/merge karna. |
| Feed timeline ke liye kaunsa Redis type? | Sorted Set, score ke roop me post creation timestamp. |
| Like count kaise maintain karte hain? | Redis `INCR` atomic counter se. |
| Trending posts kaise banate hain? | Like/comment/share activity se Sorted Set score increment karke. |
| Cache invalidation kaise hota hai? | DB update ke baad Kafka event publish karke related Redis keys delete karke. |
| Feed generation me distributed lock kyun? | Same user ke liye duplicate/expensive feed regeneration avoid karne ke liye. |

---

## 23. Final Mental Model

```text
Redis = Fast read/write layer
DB = Source of truth
Kafka = Event distribution
Sorted Set = Feed timeline + trending
Set = Followers/following
String = Counters, tokens, locks
Bitmap = Presence
Pub/Sub = Live notifications
```

---

## Next Part

**Part 30: Production Checklist, Runbook and Final Architecture — Complete Deep Dive**

Isme hum cover karenge:

- Complete production checklist
- Redis runbook
- Common incidents
- Incident response steps
- Failure scenarios
- Rollback strategy
- Capacity planning
- Security review
- Performance review
- Final architecture diagram
- Interview preparation
- Project completion
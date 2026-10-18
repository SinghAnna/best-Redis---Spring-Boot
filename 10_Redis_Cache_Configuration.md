# Part 10: Redis Cache Configuration — Complete Deep Dive

> `RedisCacheManager` decide karta hai ki Spring Cache Redis me kaise store hoga. Isme hum TTL, key prefix, null values, serializers, transaction-aware caching aur multiple cache regions configure karte hain.

---

## 1. Redis Cache Configuration Overview

Spring Boot me Redis cache configure karne ke do main tarike hain:

| Approach | Kab Use Karein |
|---|---|
| `application.yml` properties | Simple, uniform configuration |
| `RedisCacheManager` bean | Fine-grained, per-cache TTL and serializer control |

`RedisCacheConfiguration` immutable hota hai aur ye `RedisCache` behavior customize karta hai — null caching, key prefix computation aur binary serialization. [147]

---

## 2. Simple YAML Configuration

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 10m
      cache-null-values: false
      use-key-prefix: true
      key-prefix: "app:user-service:"
```

### Properties

| Property | Meaning |
|---|---|
| `spring.cache.type` | Cache provider type |
| `time-to-live` | Cache entry TTL |
| `cache-null-values` | Null result cache karna hai ya nahi |
| `use-key-prefix` | Key prefix apply karna hai ya nahi |
| `key-prefix` | Custom prefix |

By default Spring Data Redis cache entry ko `cacheName::key` format me store karta hai. [145][150]

### Example Key

```text
app:user-service:userProfile::101
```

---

## 3. RedisCacheManager Kya Hai?

`RedisCacheManager` Spring Cache abstraction ka Redis implementation manager hai.

```text
@Service
   |
   | @Cacheable
   v
Spring Cache Abstraction
   |
   v
RedisCacheManager
   |
   +--> RedisCache
   |
   +--> RedisCacheWriter
   |
   +--> RedisConnectionFactory
   |
   v
Redis Server
```

`RedisCacheManager` default cache configuration bhi support karta hai aur specific caches ke liye alag configuration bhi. [148]

---

## 4. Basic RedisCacheManager Bean

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

@Configuration
public class RedisCacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {

        RedisCacheConfiguration cacheConfiguration =
                RedisCacheConfiguration.defaultCacheConfig()
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

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(cacheConfiguration)
                .build();
    }
}
```

### Flow

```text
@Cacheable(value = "userProfile", key = "#userId")
        |
        v
RedisCacheManager
        |
        v
Default configuration
        |
        +--> TTL: 10 minutes
        +--> Null caching: disabled
        +--> Key serializer: String
        +--> Value serializer: JSON
```

---

## 5. Default Cache Configuration

```java
RedisCacheConfiguration.defaultCacheConfig()
```

Default behavior:

| Setting | Default |
|---|---|
| TTL | None |
| Null caching | Enabled |
| Key prefix | Enabled |
| Key format | `cacheName::key` |
| Serialization | Java-based serializer |

Spring Data Redis documentation ke according default me key cache name + `::` prefix ke saath store hoti hai, null caching enabled hoti hai, aur TTL none hota hai. [145]

### Production Rule

```text
Default TTL none hai,
isliye production me TTL explicitly set karna compulsory hai.
```

---

## 6. TTL Configuration

### Fixed TTL

```java
RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(15));
```

### Per-Cache TTL

```java
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
```

### TTL Reset Behavior

Spring Data Redis me TTL create ya update operation par set/reset hota hai. Agar entry TTL expire hone se pehle phir se write hoti hai, to uska timeout configured duration par reset ho jata hai. [145]

---

## 7. Complete Multi-Cache Configuration

```java
package com.example.config;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
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
public class MultiCacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {

        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration
                .defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .disableCachingNullValues()
                .prefixCacheNameWith("app:user-service:")
                .serializeKeysWith(
                        RedisSerializationContext.SerializationPair
                                .fromSerializer(new StringRedisSerializer())
                )
                .serializeValuesWith(
                        RedisSerializationContext.SerializationPair
                                .fromSerializer(
                                        new GenericJackson2JsonRedisSerializer(objectMapper())
                                )
                );

        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();

        cacheConfigs.put(
                "userProfile",
                defaultConfig.entryTtl(Duration.ofMinutes(15))
        );

        cacheConfigs.put(
                "followers",
                defaultConfig.entryTtl(Duration.ofMinutes(2))
        );

        cacheConfigs.put(
                "feed",
                defaultConfig.entryTtl(Duration.ofSeconds(30))
        );

        cacheConfigs.put(
                "product",
                defaultConfig.entryTtl(Duration.ofHours(1))
        );

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(defaultConfig)
                .withInitialCacheConfigurations(cacheConfigs)
                .transactionAware()
                .build();
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

`withInitialCacheConfigurations(Map)` specific caches ko default configuration se alag configuration dene ke liye use hota hai. [148]

---

## 8. Generated Redis Keys

### Cache Names and Keys

```java
@Cacheable(value = "userProfile", key = "#userId")
public UserProfileResponse getUserProfile(Long userId) { ... }
```

Generated key:

```text
app:user-service:userProfile::101
```

```java
@Cacheable(value = "feed", key = "#userId + ':' + #page")
public List<PostResponse> getUserFeed(Long userId, int page) { ... }
```

Generated key:

```text
app:user-service:feed::101:0
```

### Key Structure

```text
app:user-service:feed::101:0
└──────┬──────┘ └─┬─┘ └──┬──┘
   prefix     cache name  key
```

---

## 9. Key Prefix Options

### Static Prefix

```java
RedisCacheConfiguration.defaultCacheConfig()
        .prefixCacheNameWith("prod:");
```

Generated key:

```text
prod:userProfile::101
```

### Computed Prefix

```java
RedisCacheConfiguration.defaultCacheConfig()
        .computePrefixWith(
                cacheName -> "app:user-service:" + cacheName + ":"
        );
```

Generated key:

```text
app:user-service:userProfile:101
```

### Disable Prefix

```java
RedisCacheConfiguration.defaultCacheConfig()
        .disableKeyPrefix();
```

`RedisCacheConfiguration` static aur computed dono prefix support karta hai. [145]

---

## 10. Null Caching

### Disable Null Caching

```java
.disableCachingNullValues()
```

### Why Disable?

Agar user ID `999999` DB me exist nahi karta:

```text
Request -> Redis miss -> DB null
```

Agar null cache na ho:

```text
Har request DB tak jayegi
= Cache penetration
```

Agar null cache ho:

```text
Request -> Redis null cache hit
= DB protected
```

### Enable Null Caching

```java
RedisCacheConfiguration.defaultCacheConfig();
```

By default null caching enabled hoti hai. [145]

### Production Recommendation

```text
Null caching enable karo,
lekin null entries ka TTL short rakho.
```

Example:

```java
cacheConfigs.put(
        "userProfile",
        defaultConfig.entryTtl(Duration.ofMinutes(15))
);

cacheConfigs.put(
        "userProfileNull",
        defaultConfig.entryTtl(Duration.ofSeconds(60))
);
```

---

## 11. Transaction-Aware Caching

```java
RedisCacheManager.builder(connectionFactory)
        .transactionAware()
        .build();
```

### Meaning

Transaction commit hone ke baad cache operation perform hota hai.

### Without `transactionAware`

```text
DB transaction start
Cache update
DB rollback
Cache still updated -> stale cache
```

### With `transactionAware`

```text
DB transaction start
DB commit
Cache update
```

### Best Practice

```text
Jab cache update DB transaction ke saath sync karna ho,
tab transactionAware use karo.
```

---

## 12. Custom Cache Key Serialization

```java
.serializeKeysWith(
        RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer())
)
```

### Why String Key Serializer?

Readable keys:

```text
userProfile::101
```

Binary/JDK keys:

```text
\xac\xed\x00\x05t\x00\x0buserProfile
```

Production me readable keys debugging ke liye better hote hain.

---

## 13. Custom Value Serialization

```java
.serializeValuesWith(
        RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer())
)
```

### Options

| Serializer | Use Case |
|---|---|
| `GenericJackson2JsonRedisSerializer` | Multiple object types |
| `Jackson2JsonRedisSerializer<T>` | Fixed DTO type |
| `StringRedisSerializer` | Raw string values |
| JDK serializer | Avoid in production |

---

## 14. Dynamic TTL

Spring Data Redis `RedisCacheWriter.TtlFunction` ke through per-entry dynamic TTL support karta hai. [145]

### Concept

```text
Static TTL:
All entries -> 15 minutes

Dynamic TTL:
User profile -> 15 minutes
Null profile -> 1 minute
Premium user -> 30 minutes
```

### Practical Alternative

Alag cache names use karo:

```java
cacheConfigs.put(
        "userProfile",
        defaultConfig.entryTtl(Duration.ofMinutes(15))
);

cacheConfigs.put(
        "userProfileNull",
        defaultConfig.entryTtl(Duration.ofSeconds(60))
);
```

---

## 15. Cache Statistics

```yaml
spring:
  cache:
    redis:
      enable-statistics: true
```

### Useful Metrics

| Metric | Meaning |
|---|---|
| Cache hits | Cache se data mila |
| Cache misses | Cache me data nahi mila |
| Puts | Cache me entries write hui |
| Evictions | Cache entries remove hui |

---

## 16. Cache TTL Strategy

| Cache Name | Recommended TTL | Reason |
|---|---|---|
| `userProfile` | 10–30 minutes | Moderate change frequency |
| `product` | 30 minutes–2 hours | Relatively stable |
| `feed` | 30 seconds–5 minutes | Highly dynamic |
| `followers` | 1–5 minutes | Frequently changing |
| `config` | 5–15 minutes | Controlled updates |
| `nullEntity` | 30 seconds–2 minutes | Penetration protection |

---

## 17. Production Configuration Template

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 10m
      cache-null-values: false
      use-key-prefix: true
      key-prefix: "app:user-service:"
      enable-statistics: true
```

### Java Equivalent

```java
RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(10))
        .disableCachingNullValues()
        .prefixCacheNameWith("app:user-service:")
        .serializeKeysWith(
                RedisSerializationContext.SerializationPair
                        .fromSerializer(new StringRedisSerializer())
        )
        .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer())
        );
```

---

## 18. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| TTL set na karna | Entries kabhi expire nahi hongi | `entryTtl()` compulsory |
| Same TTL for all caches | Dynamic data stale / stable data unnecessarily expire | Per-cache TTL |
| Null caching disable without strategy | Cache penetration | Null cache with short TTL |
| No key prefix | Multiple services key collision | Service-specific prefix |
| Entity cache karna | Lazy loading / serialization issue | DTO cache karo |
| No transaction awareness | DB rollback ke baad stale cache | `transactionAware()` |
| Very long TTL | Stale data | Business TTL define karo |

---

## 19. Interview Questions

| Question | Answer |
|---|---|
| `RedisCacheManager` kya hai? | Spring Cache abstraction ka Redis-backed cache manager. |
| `RedisCacheConfiguration` kya control karta hai? | TTL, key prefix, null caching aur serialization. |
| Default Redis cache key format kya hai? | `cacheName::key`. |
| Default TTL kya hai? | None; explicitly set karna padta hai. |
| Null caching kab enable karni chahiye? | Cache penetration avoid karne ke liye, short TTL ke saath. |
| `transactionAware()` kya karta hai? | Cache operation ko DB transaction commit ke baad perform karta hai. |
| Per-cache TTL kaise set karte hain? | `withInitialCacheConfigurations()` ya `withCacheConfiguration()` se. |
| Key prefix kyun use karte hain? | Multiple services/environments me key collision avoid karne ke liye. |

---

## 20. Final Mental Model

```text
RedisCacheManager
    |
    +--> Default cache config
    |
    +--> Per-cache config
    |
    +--> TTL
    |
    +--> Key prefix
    |
    +--> Null caching
    |
    +--> Serialization
    |
    +--> Transaction awareness
```

---

## Next Part

**Part 11: Cache Invalidation Strategies — Complete Deep Dive**

Isme hum cover karenge:

- Cache invalidation kya hai
- TTL-based invalidation
- Write-through invalidation
- Event-based invalidation
- Kafka-based cache invalidation
- Cache versioning
- Stale cache handling
- Social media feed invalidation
- Distributed cache invalidation
- Production patterns
- Interview questions
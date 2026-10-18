# Part 4: Spring Data Redis Setup — Complete Deep Dive

> Is part me hum Spring Boot application ko Redis ke saath connect karenge — dependencies, Docker setup, Lettuce, connection factory, RedisTemplate, health check aur production configuration ke saath.

---

## 1. Spring Data Redis Kya Hai?

**Spring Data Redis** Spring ecosystem ka project hai jo Redis ko Spring Boot application me easily integrate karta hai.

Ye provide karta hai:

- Redis connection management
- `RedisTemplate`
- `StringRedisTemplate`
- Spring Cache integration
- Redis repositories
- Pub/Sub support
- Redis Streams support
- Lettuce and Jedis client support

Spring Data Redis low-level `RedisConnection` aur high-level `RedisTemplate` abstractions dono deta hai. [69]

### Simple Architecture

```text
Spring Boot Application
        |
        v
Spring Data Redis
        |
        +--> RedisTemplate
        |
        +--> RedisConnectionFactory
        |
        v
Lettuce / Jedis Client
        |
        v
Redis Server
```

---

## 2. Redis Client: Lettuce vs Jedis

| Feature | Lettuce | Jedis |
|---|---|---|
| Default in Spring Boot | Yes | No |
| Thread safety | Thread-safe | Thread-safe with pool |
| Connection model | Netty-based, async | Blocking, connection-pool based |
| Reactive support | Yes | Limited |
| Performance | Generally better for high concurrency | Simple and widely used |
| API style | Modern, async/sync/reactive | Traditional sync |
| Recommended for | Most Spring Boot production apps | Legacy/simple apps |

Spring Data Redis Lettuce ko Netty-based connector ke through support karta hai aur Jedis ko community-driven connector ke through support karta hai. [66]

### Recommendation

```text
Default choice: Lettuce
Legacy project: Jedis
High concurrency: Lettuce
Reactive application: Lettuce
```

---

## 3. Maven Dependencies

### Basic Redis Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

Yeh dependency Lettuce bhi include karti hai.

### Spring Cache Support

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

### Actuator for Health Check

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Lombok

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

### Complete Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
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
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>

    <dependency>
        <groupId>com.fasterxml.jackson.datatype</groupId>
        <artifactId>jackson-datatype-jsr310</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

---

## 4. Redis Docker Setup

### Simple Docker Command

```bash
docker run --name redis-local \
  -p 6379:6379 \
  -d redis:7.4-alpine
```

### Docker Compose

`docker-compose.yml`

```yaml
services:
  redis:
    image: redis:7.4-alpine
    container_name: redis-local
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: >
      redis-server
      --appendonly yes
      --appendfsync everysec
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  redis_data:
```

Run:

```bash
docker compose up -d
```

Check Redis:

```bash
docker exec -it redis-local redis-cli ping
```

Expected output:

```text
PONG
```

---

## 5. Spring Boot Redis Configuration

### Local Development

`application.yml`

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2s
```

### With Password

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: strong-password
      timeout: 2s
```

### Production Configuration

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      username: ${REDIS_USERNAME:}
      password: ${REDIS_PASSWORD}
      database: ${REDIS_DATABASE:0}
      timeout: 2s
      connect-timeout: 2s
      client-name: user-service
      lettuce:
        shutdown-timeout: 100ms
        pool:
          enabled: true
          max-active: 32
          max-idle: 16
          min-idle: 4
          max-wait: 500ms

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  health:
    redis:
      enabled: true
```

Spring Boot 3.x me Redis properties `spring.data.redis.*` namespace ke under aati hain. [72]

---

## 6. Environment Variables

### `.env`

```env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=strong-password
REDIS_DATABASE=0
```

### Docker Compose with Environment

```yaml
services:
  redis:
    image: redis:7.4-alpine
    container_name: redis-local
    environment:
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    ports:
      - "6379:6379"
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --appendonly yes
```

### Production Best Practice

```text
Password ko application.yml me hardcode mat karo.
Environment variable, secret manager ya vault use karo.
```

---

## 7. RedisConnectionFactory

`RedisConnectionFactory` Redis connection create karne ka core interface hai. Active `RedisConnection` objects isi factory se create hote hain. [68]

### Default Auto Configuration

Spring Boot automatically `LettuceConnectionFactory` create karta hai jab `spring-boot-starter-data-redis` dependency add hoti hai.

```text
spring-boot-starter-data-redis
        |
        +--> Lettuce dependency
        |
        +--> RedisAutoConfiguration
        |
        +--> LettuceConnectionFactory
        |
        +--> RedisTemplate
        |
        +--> StringRedisTemplate
```

### Manual Lettuce Connection Factory

```java
@Configuration
public class RedisConnectionConfig {

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config =
                new RedisStandaloneConfiguration("localhost", 6379);

        config.setPassword(RedisPassword.of("strong-password"));

        return new LettuceConnectionFactory(config);
    }
}
```

### With Connection Pool

```java
@Configuration
public class RedisConnectionConfig {

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration serverConfig =
                new RedisStandaloneConfiguration("localhost", 6379);

        serverConfig.setPassword(RedisPassword.of("strong-password"));

        LettuceClientConfiguration clientConfig = LettucePoolingClientConfiguration.builder()
                .commandTimeout(Duration.ofSeconds(2))
                .poolConfig(poolConfig())
                .build();

        return new LettuceConnectionFactory(serverConfig, clientConfig);
    }

    private GenericObjectPoolConfig<?> poolConfig() {
        GenericObjectPoolConfig<?> poolConfig = new GenericObjectPoolConfig<>();
        poolConfig.setMaxTotal(32);
        poolConfig.setMaxIdle(16);
        poolConfig.setMinIdle(4);
        poolConfig.setMaxWait(Duration.ofMillis(500));
        return poolConfig;
    }
}
```

---

## 8. RedisTemplate

`RedisTemplate` Redis operations ke liye high-level, thread-safe template hai. Ye connections acquire aur manage karta hai. [68]

### Default Auto-Configured Beans

Spring Boot automatically provide karta hai:

```java
RedisTemplate<Object, Object>
StringRedisTemplate
```

### `RedisTemplate` vs `StringRedisTemplate`

| Feature | `RedisTemplate` | `StringRedisTemplate` |
|---|---|---|
| Key type | Object | String |
| Value type | Object | String |
| Serialization | Configurable | String serializer |
| Best for | Custom objects | Simple strings, counters, tokens |
| JSON support | Custom serializer needed | Manual JSON conversion |

---

## 9. Basic RedisTemplate Configuration

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory
    ) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());

        template.setValueSerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new StringRedisSerializer());

        template.afterPropertiesSet();

        return template;
    }
}
```

### Problem

Agar value serializer `StringRedisSerializer` hai, to custom Java object directly store nahi kar sakte.

```java
redisTemplate.opsForValue().set(key, userProfile); // Not ideal
```

Isliye JSON serializer chahiye.

---

## 10. JSON Serializer Configuration

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
public class RedisJsonConfig {

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

---

## 11. First Redis Integration Test

### DTO

```java
package com.example.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserProfile {

    private Long id;
    private String name;
    private String email;
    private String city;
    private LocalDateTime createdAt;
}
```

### Service

```java
package com.example.service;

import com.example.dto.UserProfile;
import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Optional;

@Service
@RequiredArgsConstructor
public class UserProfileRedisService {

    private final RedisTemplate<String, Object> redisTemplate;

    private static final String KEY_PREFIX = "user-service:user:";

    public void saveProfile(UserProfile profile) {
        String key = KEY_PREFIX + profile.getId() + ":profile";

        redisTemplate.opsForValue().set(
                key,
                profile,
                Duration.ofMinutes(15)
        );
    }

    public Optional<UserProfile> getProfile(Long userId) {
        String key = KEY_PREFIX + userId + ":profile";

        Object value = redisTemplate.opsForValue().get(key);

        return Optional.ofNullable((UserProfile) value);
    }

    public void deleteProfile(Long userId) {
        redisTemplate.delete(KEY_PREFIX + userId + ":profile");
    }
}
```

### Controller

```java
package com.example.controller;

import com.example.dto.UserProfile;
import com.example.service.UserProfileRedisService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDateTime;

@RestController
@RequestMapping("/api/redis/users")
@RequiredArgsConstructor
public class UserProfileRedisController {

    private final UserProfileRedisService userProfileRedisService;

    @PostMapping("/{userId}")
    public ResponseEntity<String> saveProfile(@PathVariable Long userId) {
        UserProfile profile = UserProfile.builder()
                .id(userId)
                .name("Rahul")
                .email("rahul@example.com")
                .city("Bengaluru")
                .createdAt(LocalDateTime.now())
                .build();

        userProfileRedisService.saveProfile(profile);

        return ResponseEntity.ok("Profile cached successfully");
    }

    @GetMapping("/{userId}")
    public ResponseEntity<UserProfile> getProfile(@PathVariable Long userId) {
        return userProfileRedisService.getProfile(userId)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @DeleteMapping("/{userId}")
    public ResponseEntity<String> deleteProfile(@PathVariable Long userId) {
        userProfileRedisService.deleteProfile(userId);
        return ResponseEntity.ok("Profile removed from Redis");
    }
}
```

### Test Using curl

```bash
curl -X POST http://localhost:8080/api/redis/users/101
```

```bash
curl http://localhost:8080/api/redis/users/101
```

```bash
curl -X DELETE http://localhost:8080/api/redis/users/101
```

### Verify in Redis CLI

```bash
docker exec -it redis-local redis-cli

KEYS user-service:user:*
GET user-service:user:101:profile
TTL user-service:user:101:profile
```

---

## 12. Redis Health Check

Spring Boot Actuator Redis health indicator Redis data store ke liye health status return karta hai. [65]

### Enable Health Endpoint

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      show-details: always
```

### Check Health

```bash
curl http://localhost:8080/actuator/health
```

Healthy response:

```json
{
  "status": "UP",
  "components": {
    "redis": {
      "status": "UP"
    }
  }
}
```

Redis down hone par:

```json
{
  "status": "DOWN",
  "components": {
    "redis": {
      "status": "DOWN"
    }
  }
}
```

---

## 13. Common Connection Errors

### 1. `Unable to connect to Redis`

```text
Unable to connect to Redis
```

Possible reasons:

- Redis container running nahi hai
- Wrong host/port
- Firewall issue
- Docker network issue

Fix:

```bash
docker ps
docker exec -it redis-local redis-cli ping
```

### 2. `NOAUTH Authentication required`

```text
NOAUTH Authentication required
```

Reason:

```text
Redis password set hai,
lekin Spring Boot configuration me password missing hai.
```

Fix:

```yaml
spring:
  data:
    redis:
      password: ${REDIS_PASSWORD}
```

### 3. `WRONGPASS invalid username-password pair`

Reason:

```text
Galat password
```

Fix:

```bash
docker exec -it redis-local redis-cli
AUTH your-password
```

### 4. `RedisConnectionFailureException`

Possible reasons:

- Redis down
- Network timeout
- Connection pool exhausted
- DNS resolution issue

Fix:

```yaml
spring:
  data:
    redis:
      timeout: 2s
      connect-timeout: 2s
```

---

## 14. Production Setup Checklist

- [ ] Lettuce client use kar rahe ho
- [ ] Redis host/port environment variables se aa rahe hain
- [ ] Password secret manager/env variable me hai
- [ ] Command timeout set hai
- [ ] Connect timeout set hai
- [ ] Connection pool configured hai
- [ ] Redis health check enabled hai
- [ ] JSON serializer configured hai
- [ ] Key prefix use ho raha hai
- [ ] TTL strategy defined hai
- [ ] Redis publicly exposed nahi hai
- [ ] Monitoring metrics enabled hain

---

## 15. Interview Questions

| Question | Answer |
|---|---|
| Spring Data Redis kya hai? | Spring ecosystem ka project jo Redis ko Spring Boot me integrate karta hai. |
| Lettuce aur Jedis me difference? | Lettuce Netty-based, thread-safe aur reactive support deta hai; Jedis traditional connection-pool-based client hai. |
| Spring Boot default Redis client kaunsa hai? | Lettuce. |
| `RedisConnectionFactory` kya karta hai? | Redis connections create aur manage karta hai. |
| `RedisTemplate` kya hai? | Redis commands execute karne ke liye high-level, thread-safe Spring abstraction. |
| `StringRedisTemplate` kab use karte hain? | Jab key aur value dono simple string hon. |
| Redis health check kaise enable karte hain? | Spring Boot Actuator ke saath, `/actuator/health` endpoint se. |
| Redis password kahan store karna chahiye? | Environment variable, secret manager ya vault me. |
| Connection pool kyun configure karte hain? | High concurrency me connection exhaustion avoid karne ke liye. |

---

## 16. Final Mental Model

```text
Spring Boot
   |
   +--> Spring Data Redis
          |
          +--> RedisConnectionFactory
                 |
                 +--> Lettuce
                        |
                        +--> Redis Server

RedisTemplate
   |
   +--> RedisConnectionFactory
   |
   +--> Serializers
   |
   +--> Redis operations
```

---

## Next Part

**Part 5: Redis Connection Configuration — Complete Deep Dive**

Isme hum cover karenge:

- `spring.data.redis` properties
- Host, port, database index
- Password and username
- Timeout configuration
- Lettuce connection pool
- Pool sizing
- SSL/TLS configuration
- Environment-specific configuration
- `@ConfigurationProperties`
- Sentinel and Cluster connection basics
- Common production mistakes
- Interview questions
# Part 5: Redis Connection Configuration — Complete Deep Dive

> Redis connection configuration sirf host/port set karna nahi hai. Production me timeout, connection pool, TLS, database index, Sentinel/Cluster topology aur graceful failure handling sab important hote hain.

---

## 1. Connection Configuration Overview

Spring Boot me Redis connection mainly in properties se control hota hai:

```yaml
spring:
  data:
    redis:
      host
      port
      username
      password
      database
      timeout
      connect-timeout
      client-name
      lettuce
```

Spring Boot Lettuce ke through Redis Standalone, Sentinel aur Cluster connections support karta hai. [85]

---

## 2. Basic Standalone Configuration

### application.yml

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      database: 0
      timeout: 2s
      connect-timeout: 2s
```

### Meaning

| Property | Meaning |
|---|---|
| `host` | Redis server address |
| `port` | Redis port, default 6379 |
| `database` | Redis logical database index |
| `timeout` | Command timeout |
| `connect-timeout` | TCP connection establish karne ka timeout |

---

## 3. Redis Database Index

Redis me by default 16 logical databases hote hain:

```text
DB 0
DB 1
DB 2
...
DB 15
```

### Example

```yaml
spring:
  data:
    redis:
      database: 0
```

### CLI Me Database Switch

```bash
SELECT 1
```

### Production Recommendation

```text
Production me mostly database 0 use karo.
Alag-alag services ke liye logical database use karne ki jagah
key prefix use karo.
```

### Why?

- Cluster mode me multiple logical databases support nahi hote.
- Key prefix se ownership clear hoti hai.
- Monitoring aur debugging easier hoti hai.

---

## 4. Authentication Configuration

### Username + Password

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      username: ${REDIS_USERNAME:default}
      password: ${REDIS_PASSWORD}
```

### Only Password

```yaml
spring:
  data:
    redis:
      password: ${REDIS_PASSWORD}
```

### Environment Variables

```env
REDIS_HOST=redis.internal.example.com
REDIS_PORT=6379
REDIS_USERNAME=default
REDIS_PASSWORD=strong-secret
```

### Production Rule

```text
Password ko Git me commit mat karo.
Environment variable, Kubernetes Secret,
AWS Secrets Manager, Vault ya equivalent use karo.
```

---

## 5. Timeout Configuration

Timeouts production me sabse important hain.

### Types of Timeout

| Timeout | Meaning | Impact if Missing |
|---|---|---|
| Connect timeout | TCP connection establish hone ka maximum time | App hang ho sakti hai |
| Command timeout | Ek Redis command complete hone ka maximum time | Threads block ho sakte hain |
| Pool wait timeout | Pool se connection milne ka maximum wait time | Requests slow ya fail hongi |
| Shutdown timeout | Application shutdown par connection close ka wait time | Graceful shutdown affected |

### Recommended Configuration

```yaml
spring:
  data:
    redis:
      timeout: 1s
      connect-timeout: 1s
      lettuce:
        shutdown-timeout: 100ms
        pool:
          max-wait: 500ms
```

### Why Not Very High Timeout?

```text
Redis down hai
+ timeout 30s
= har request 30 second tak wait karegi
= thread pool exhaustion
= application down
```

### Better Approach

```text
Fast fail + fallback + circuit breaker
```

---

## 6. Lettuce Connection Pool

Lettuce by default multiplexing use karta hai, lekin blocking operations aur high-concurrency workloads ke liye connection pool configure karna useful hota hai. Redis client libraries connection pooling aur multiplexing dono approaches support karte hain. [76]

### YAML Configuration

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      timeout: 1s
      connect-timeout: 1s
      lettuce:
        shutdown-timeout: 100ms
        pool:
          enabled: true
          max-active: 32
          max-idle: 16
          min-idle: 4
          max-wait: 500ms
```

### Pool Properties

| Property | Meaning |
|---|---|
| `enabled` | Connection pool enable/disable |
| `max-active` | Maximum active connections |
| `max-idle` | Maximum idle connections |
| `min-idle` | Minimum idle connections |
| `max-wait` | Pool se connection milne ka maximum wait |

`max-active` pool ke simultaneously allocated connections ki hard limit set karta hai. [81]

---

## 7. Pool Sizing Strategy

Pool size blindly 100 ya 200 set mat karo.

### Factors

- Application instances
- Tomcat/Netty worker threads
- Redis command latency
- Expected RPS
- Redis server `maxclients`
- Blocking operations

### Simple Formula

```text
Required Connections =
(Average Redis Latency in seconds) x (Target Redis RPS)
```

Example:

```text
Average Redis latency = 2 ms = 0.002 seconds
Target Redis RPS = 5000

Required Connections = 0.002 x 5000 = 10
```

To ek application instance ke liye 10–20 connections reasonable ho sakte hain.

### Practical Starting Point

| Application Type | Starting Pool Size |
|---|---|
| Small microservice | 8–16 |
| Medium API service | 16–32 |
| High-throughput service | 32–64 |
| Batch-heavy service | Workload-based tuning |

### Important Formula for Total Connections

```text
Total Redis Connections =
Application Instances x Pool Size
```

Example:

```text
20 application instances
x 32 max-active connections
= 640 Redis connections
```

Redis server ke `maxclients` limit se compare karo.

---

## 8. Custom Lettuce Pool Configuration

```java
package com.example.config;

import org.apache.commons.pool2.impl.GenericObjectPoolConfig;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.RedisPassword;
import org.springframework.data.redis.connection.lettuce.LettuceClientConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.connection.lettuce.LettucePoolingClientConfiguration;

import java.time.Duration;

@Configuration
public class RedisPoolConfig {

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration serverConfig =
                new RedisStandaloneConfiguration(
                        System.getenv("REDIS_HOST"),
                        Integer.parseInt(System.getenv().getOrDefault("REDIS_PORT", "6379"))
                );

        serverConfig.setPassword(
                RedisPassword.of(System.getenv("REDIS_PASSWORD"))
        );

        LettuceClientConfiguration clientConfig =
                LettucePoolingClientConfiguration.builder()
                        .commandTimeout(Duration.ofSeconds(1))
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

## 9. SSL/TLS Configuration

Redis ko public network ya untrusted network me expose karte waqt TLS use karo.

### YAML Configuration

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: 6380
      username: ${REDIS_USERNAME}
      password: ${REDIS_PASSWORD}
      ssl:
        enabled: true
```

### Custom SSL Configuration

```java
@Bean
public LettuceConnectionFactory redisConnectionFactory() {
    RedisStandaloneConfiguration serverConfig =
            new RedisStandaloneConfiguration("redis.example.com", 6380);

    serverConfig.setPassword(RedisPassword.of(System.getenv("REDIS_PASSWORD")));

    LettuceClientConfiguration clientConfig =
            LettuceClientConfiguration.builder()
                    .useSsl()
                    .build();

    return new LettuceConnectionFactory(serverConfig, clientConfig);
}
```

### Production Security Rule

```text
Redis internet par expose ho to TLS compulsory hai.
Internal VPC me bhi TLS recommended hai for defense in depth.
```

---

## 10. Environment-Specific Configuration

### application.yml

```yaml
spring:
  profiles:
    active: local

***
spring:
  config:
    activate:
      on-profile: local
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2s

***
spring:
  config:
    activate:
      on-profile: dev
  data:
    redis:
      host: redis-dev.internal
      port: 6379
      password: ${REDIS_PASSWORD}
      timeout: 1s

***
spring:
  config:
    activate:
      on-profile: prod
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      username: ${REDIS_USERNAME}
      password: ${REDIS_PASSWORD}
      timeout: 1s
      connect-timeout: 1s
      ssl:
        enabled: true
      lettuce:
        pool:
          enabled: true
          max-active: 32
          max-idle: 16
          min-idle: 4
          max-wait: 500ms
```

### Run Profiles

```bash
# Local
./mvnw spring-boot:run -Dspring-boot.run.profiles=local

# Dev
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# Production
java -jar app.jar --spring.profiles.active=prod
```

---

## 11. Type-Safe Configuration with `@ConfigurationProperties`

```java
package com.example.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Data
@Component
@ConfigurationProperties(prefix = "app.redis")
public class RedisProperties {

    private String host = "localhost";
    private int port = 6379;
    private String username;
    private String password;
    private int database = 0;
    private Duration commandTimeout = Duration.ofSeconds(1);
    private Duration connectTimeout = Duration.ofSeconds(1);
    private Pool pool = new Pool();

    @Data
    public static class Pool {
        private boolean enabled = true;
        private int maxActive = 32;
        private int maxIdle = 16;
        private int minIdle = 4;
        private Duration maxWait = Duration.ofMillis(500);
    }
}
```

### application.yml

```yaml
app:
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}
    username: ${REDIS_USERNAME:default}
    password: ${REDIS_PASSWORD}
    database: 0
    command-timeout: 1s
    connect-timeout: 1s
    pool:
      enabled: true
      max-active: 32
      max-idle: 16
      min-idle: 4
      max-wait: 500ms
```

### Connection Factory Using Custom Properties

```java
package com.example.config;

import lombok.RequiredArgsConstructor;
import org.apache.commons.pool2.impl.GenericObjectPoolConfig;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisPassword;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceClientConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.connection.lettuce.LettucePoolingClientConfiguration;

@Configuration
@RequiredArgsConstructor
public class CustomRedisConfig {

    private final RedisProperties redisProperties;

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration serverConfig =
                new RedisStandaloneConfiguration(
                        redisProperties.getHost(),
                        redisProperties.getPort()
                );

        serverConfig.setDatabase(redisProperties.getDatabase());
        serverConfig.setUsername(redisProperties.getUsername());
        serverConfig.setPassword(RedisPassword.of(redisProperties.getPassword()));

        LettuceClientConfiguration clientConfig =
                LettucePoolingClientConfiguration.builder()
                        .commandTimeout(redisProperties.getCommandTimeout())
                        .poolConfig(poolConfig())
                        .build();

        return new LettuceConnectionFactory(serverConfig, clientConfig);
    }

    private GenericObjectPoolConfig<?> poolConfig() {
        GenericObjectPoolConfig<?> poolConfig = new GenericObjectPoolConfig<>();

        poolConfig.setMaxTotal(redisProperties.getPool().getMaxActive());
        poolConfig.setMaxIdle(redisProperties.getPool().getMaxIdle());
        poolConfig.setMinIdle(redisProperties.getPool().getMinIdle());
        poolConfig.setMaxWait(redisProperties.getPool().getMaxWait());

        return poolConfig;
    }
}
```

---

## 12. Redis Sentinel Configuration

Redis Sentinel automatic failover ke liye use hota hai.

```yaml
spring:
  data:
    redis:
      password: ${REDIS_PASSWORD}
      sentinel:
        master: mymaster
        nodes:
          - sentinel-1:26379
          - sentinel-2:26379
          - sentinel-3:26379
```

### Sentinel Flow

```text
Spring Boot
    |
    v
Sentinel Nodes
    |
    +--> Current master discover
    |
    v
Redis Master
    |
    +--> Redis Replicas
```

Sentinel master discovery, monitoring aur automatic failover handle karta hai.

---

## 13. Redis Cluster Configuration

```yaml
spring:
  data:
    redis:
      password: ${REDIS_PASSWORD}
      cluster:
        nodes:
          - redis-node-1:6379
          - redis-node-2:6379
          - redis-node-3:6379
          - redis-node-4:6379
          - redis-node-5:6379
          - redis-node-6:6379
        max-redirects: 3
```

### Cluster Notes

- Data hash slots me divide hota hai.
- Multi-key operations me same hash slot required hota hai.
- Hash tags use karo, jaise `{user:101}:profile` aur `{user:101}:followers`.
- Cluster mode me multiple logical databases supported nahi hote.

Lettuce Redis Cluster support ko `RedisClusterClient` ke through provide karta hai aur Spring Data Redis integration recommended approach hai. [83]

---

## 14. Graceful Failure Handling

Redis down hone par application completely down nahi honi chahiye.

### Cache Service with Fallback

```java
package com.example.service;

import com.example.dto.UserProfile;
import com.example.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Optional;

@Slf4j
@Service
@RequiredArgsConstructor
public class ResilientProfileService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final UserRepository userRepository;

    public UserProfile getProfile(Long userId) {
        String key = "user-service:user:" + userId + ":profile";

        try {
            Object cached = redisTemplate.opsForValue().get(key);

            if (cached != null) {
                return (UserProfile) cached;
            }
        } catch (Exception ex) {
            log.warn("Redis unavailable, falling back to database. key={}", key, ex);
        }

        UserProfile profile = userRepository.findById(userId)
                .map(UserProfile::from)
                .orElseThrow();

        try {
            redisTemplate.opsForValue().set(key, profile, Duration.ofMinutes(15));
        } catch (Exception ex) {
            log.warn("Failed to cache profile. key={}", key, ex);
        }

        return profile;
    }
}
```

### Golden Rule

```text
Redis cache failure should degrade performance,
not break the application.
```

---

## 15. Common Production Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| No timeout | Threads block ho sakte hain | Connect + command timeout set karo |
| Very large pool | Redis `maxclients` exhaustion | Instance count x pool size calculate karo |
| Very small pool | Connection wait / latency | Load test karke tune karo |
| Password in Git | Security breach | Secret manager/env variable |
| No TLS | Data interception risk | TLS enable karo |
| Database index per service | Cluster compatibility issue | Key prefix use karo |
| No fallback | Redis outage = app outage | DB fallback + circuit breaker |
| No monitoring | Connection exhaustion late detect | Pool and Redis metrics monitor karo |

---

## 16. Production Configuration Template

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      username: ${REDIS_USERNAME:default}
      password: ${REDIS_PASSWORD}
      database: 0
      timeout: 1s
      connect-timeout: 1s
      client-name: ${SPRING_APPLICATION_NAME:user-service}
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

---

## 17. Interview Questions

| Question | Answer |
|---|---|
| Redis connection pool kya hai? | Reusable Redis connections ka pool jo connection creation overhead kam karta hai. |
| `max-active` kya control karta hai? | Pool se simultaneously kitne active connections allocate ho sakte hain. |
| `max-wait` kya hai? | Pool se connection milne ke liye maximum wait time. |
| Connect timeout aur command timeout me difference? | Connect timeout TCP connection establish karne ka time hai; command timeout Redis command complete hone ka time hai. |
| Redis TLS kab enable karna chahiye? | Jab Redis untrusted/public network par accessible ho ya defense-in-depth chahiye. |
| Sentinel kya karta hai? | Redis master monitor karta hai aur failover handle karta hai. |
| Cluster me multi-key operation issue kyun hota hai? | Keys different hash slots par ho sakti hain. |
| Redis down hone par application kaise behave karni chahiye? | Degrade gracefully — DB fallback, circuit breaker aur alerts. |

---

## 18. Final Mental Model

```text
Redis connection configuration =
Host + Port + Auth
+ Timeouts
+ Pool sizing
+ TLS
+ Topology
+ Failure handling
```

---

## Next Part

**Part 6: RedisTemplate Deep Dive — Complete Deep Dive**

Isme hum cover karenge:

- `RedisTemplate` internal architecture
- `RedisTemplate` vs `StringRedisTemplate`
- `opsForValue()`
- `opsForHash()`
- `opsForList()`
- `opsForSet()`
- `opsForZSet()`
- `opsForStream()`
- Serializers
- Custom Redis service
- CRUD operations
- TTL handling
- Production patterns
- Interview questions
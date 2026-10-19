# Part 28: Redis Testing in Spring Boot — Complete Deep Dive

> Redis testing ka goal hai: cache logic, TTL, serialization, locks, rate limits aur failure handling ko reliably verify karna. Unit test me Redis mock karo; integration test me real Redis container use karo.

---

## 1. Redis Testing Strategy

| Test Type | Redis Required? | Purpose |
|---|---|---|
| Unit test | No — mock | Business logic and Redis interaction verify |
| Integration test | Yes — real Redis | Actual Redis behavior verify |
| Contract test | Yes | Serializer/key compatibility verify |
| Load test | Yes | Throughput, latency, pool behavior |
| Chaos test | Yes | Redis failure handling verify |

Testcontainers JUnit ke saath integrate hota hai aur tests run hone se pehle temporary Docker container start karta hai. [392]

---

## 2. Testing Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
```

---

## 3. Unit Testing Redis Service

### Service Under Test

```java
package com.example.service;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Optional;

@Service
@RequiredArgsConstructor
public class UserProfileCacheService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void cacheProfile(Long userId, UserProfile profile) {
        redisTemplate.opsForValue().set(
                "user-service:user:" + userId + ":profile",
                profile,
                Duration.ofMinutes(15)
        );
    }

    public Optional<UserProfile> getProfile(Long userId) {
        Object value = redisTemplate.opsForValue()
                .get("user-service:user:" + userId + ":profile");

        return Optional.ofNullable((UserProfile) value);
    }
}
```

### Unit Test

```java
package com.example.service;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ValueOperations;

import java.time.Duration;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class UserProfileCacheServiceTest {

    @Mock
    private RedisTemplate<String, Object> redisTemplate;

    @Mock
    private ValueOperations<String, Object> valueOperations;

    private UserProfileCacheService service;

    @BeforeEach
    void setUp() {
        service = new UserProfileCacheService(redisTemplate);
    }

    @Test
    void shouldCacheUserProfileWithTtl() {
        UserProfile profile = UserProfile.builder()
                .id(101L)
                .name("Rahul")
                .build();

        when(redisTemplate.opsForValue()).thenReturn(valueOperations);

        service.cacheProfile(101L, profile);

        verify(valueOperations).set(
                eq("user-service:user:101:profile"),
                eq(profile),
                eq(Duration.ofMinutes(15))
        );
    }

    @Test
    void shouldReturnProfileWhenPresent() {
        UserProfile profile = UserProfile.builder()
                .id(101L)
                .name("Rahul")
                .build();

        when(redisTemplate.opsForValue()).thenReturn(valueOperations);
        when(valueOperations.get("user-service:user:101:profile"))
                .thenReturn(profile);

        Optional<UserProfile> result = service.getProfile(101L);

        assertThat(result).isPresent();
        assertThat(result.get().getName()).isEqualTo("Rahul");
    }

    @Test
    void shouldReturnEmptyWhenProfileNotPresent() {
        when(redisTemplate.opsForValue()).thenReturn(valueOperations);
        when(valueOperations.get(any())).thenReturn(null);

        Optional<UserProfile> result = service.getProfile(101L);

        assertThat(result).isEmpty();
    }
}
```

Unit test me `RedisTemplate` aur `ValueOperations` mock karte hain, jisse test Redis server par depend nahi karta. [393][397]

---

## 4. Integration Testing With Testcontainers

### Testcontainers Setup

```java
package com.example.integration;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Testcontainers
class RedisIntegrationTest {

    @Container
    static GenericContainer<?> redis =
            new GenericContainer<>("redis:7.4-alpine")
                    .withExposedPorts(6379);

    @Autowired
    private StringRedisTemplate redisTemplate;

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add(
                "spring.data.redis.port",
                () -> redis.getMappedPort(6379)
        );
    }

    @Test
    void shouldStoreAndRetrieveValue() {
        redisTemplate.opsForValue().set(
                "test:key",
                "hello-redis",
                Duration.ofMinutes(5)
        );

        String value = redisTemplate.opsForValue().get("test:key");

        assertThat(value).isEqualTo("hello-redis");
    }
}
```

Testcontainers Redis container start karta hai, random host port expose karta hai aur test complete hone ke baad container cleanup karta hai. [395][399]

---

## 5. Cache Integration Test

```java
package com.example.integration;

import com.example.dto.UserProfile;
import com.example.service.UserProfileCacheService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Testcontainers
class UserProfileCacheIntegrationTest {

    @Container
    static GenericContainer<?> redis =
            new GenericContainer<>("redis:7.4-alpine")
                    .withExposedPorts(6379);

    @Autowired
    private UserProfileCacheService cacheService;

    @Autowired
    private StringRedisTemplate redisTemplate;

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add(
                "spring.data.redis.port",
                () -> redis.getMappedPort(6379)
        );
    }

    @Test
    void shouldCacheAndRetrieveProfile() {
        UserProfile profile = UserProfile.builder()
                .id(101L)
                .name("Rahul")
                .email("rahul@example.com")
                .build();

        cacheService.cacheProfile(101L, profile);

        UserProfile cached =
                cacheService.getProfile(101L).orElseThrow();

        assertThat(cached.getId()).isEqualTo(101L);
        assertThat(cached.getName()).isEqualTo("Rahul");
    }

    @Test
    void shouldExpireCachedProfile() throws InterruptedException {
        UserProfile profile = UserProfile.builder()
                .id(102L)
                .name("Amit")
                .build();

        cacheService.cacheProfile(102L, profile);

        String key = "user-service:user:102:profile";

        Long ttl = redisTemplate.getExpire(key);

        assertThat(ttl).isNotNull();
        assertThat(ttl).isGreaterThan(0);
    }
}
```

---

## 6. TTL Testing

### Service

```java
public void cacheOtp(String key, String otp) {
    redisTemplate.opsForValue().set(key, otp, Duration.ofSeconds(300));
}
```

### Integration Test

```java
@Test
void shouldSetTtlForOtp() {
    String key = "auth-service:otp:101";

    redisTemplate.opsForValue().set(
            key,
            "482913",
            Duration.ofSeconds(300)
    );

    Long ttl = redisTemplate.getExpire(key);

    assertThat(ttl).isNotNull();
    assertThat(ttl).isBetween(295L, 300L);
}
```

### Expiry Test

```java
@Test
void shouldExpireKeyAfterTtl() throws InterruptedException {
    String key = "test:short-ttl";

    redisTemplate.opsForValue().set(
            key,
            "temporary",
            Duration.ofMillis(500)
    );

    Thread.sleep(700);

    String value = redisTemplate.opsForValue().get(key);

    assertThat(value).isNull();
}
```

---

## 7. Distributed Lock Test

### Lock Service

```java
public boolean acquireLock(String key, String ownerId, Duration ttl) {
    return Boolean.TRUE.equals(
            redisTemplate.opsForValue()
                    .setIfAbsent(key, ownerId, ttl)
    );
}
```

### Integration Test

```java
@Test
void shouldAcquireLockOnlyOnce() {
    String lockKey = "test:lock:order:1001";

    boolean first = lockService.acquireLock(
            lockKey,
            "instance-1",
            Duration.ofSeconds(10)
    );

    boolean second = lockService.acquireLock(
            lockKey,
            "instance-2",
            Duration.ofSeconds(10)
    );

    assertThat(first).isTrue();
    assertThat(second).isFalse();
}
```

### Lock Release Test

```java
@Test
void shouldReleaseLockAndAllowNewOwner() {
    String lockKey = "test:lock:order:1002";

    String ownerId = "instance-1";

    assertThat(
            lockService.acquireLock(lockKey, ownerId, Duration.ofSeconds(10))
    ).isTrue();

    lockService.releaseLock(lockKey, ownerId);

    assertThat(
            lockService.acquireLock(
                    lockKey,
                    "instance-2",
                    Duration.ofSeconds(10)
            )
    ).isTrue();
}
```

---

## 8. Rate Limiter Test

```java
@Test
void shouldAllowRequestsUpToLimit() {
    String clientId = "user:101";

    for (int i = 1; i <= 5; i++) {
        assertThat(rateLimiter.isAllowed(clientId, 5, Duration.ofMinutes(1)))
                .isTrue();
    }

    assertThat(rateLimiter.isAllowed(clientId, 5, Duration.ofMinutes(1)))
            .isFalse();
}
```

---

## 9. Pub/Sub Integration Test

### Listener

```java
@Component
public class TestNotificationListener {

    private final List<String> receivedMessages = new CopyOnWriteArrayList<>();

    public void handleMessage(String message) {
        receivedMessages.add(message);
    }

    public List<String> getReceivedMessages() {
        return receivedMessages;
    }
}
```

### Configuration

```java
@Bean
public RedisMessageListenerContainer testListenerContainer(
        RedisConnectionFactory connectionFactory,
        TestNotificationListener listener
) {
    RedisMessageListenerContainer container =
            new RedisMessageListenerContainer();

    container.setConnectionFactory(connectionFactory);
    container.addMessageListener(
            new MessageListenerAdapter(listener, "handleMessage"),
            new ChannelTopic("test-notifications")
    );

    return container;
}
```

### Test

```java
@Test
void shouldReceivePublishedMessage() throws InterruptedException {
    redisTemplate.convertAndSend(
            "test-notifications",
            "hello-subscriber"
    );

    Awaitility.await()
            .atMost(Duration.ofSeconds(5))
            .untilAsserted(() ->
                    assertThat(listener.getReceivedMessages())
                            .contains("hello-subscriber")
            );
}
```

---

## 10. Redis Stream Test

### Producer Test

```java
@Test
void shouldAddEventToStream() {
    Map<String, String> event = Map.of(
            "orderId", "1001",
            "status", "CREATED"
    );

    RecordId recordId = redisTemplate.opsForStream()
            .add("test-orders", event);

    assertThat(recordId).isNotNull();

    List<MapRecord<String, Object, Object>> records =
            redisTemplate.opsForStream()
                    .range("test-orders", Range.unbounded());

    assertThat(records).isNotEmpty();
}
```

### Consumer Group Test

```java
@Test
void shouldCreateConsumerGroupAndReadMessages() {
    redisTemplate.opsForStream().createGroup(
            "test-orders",
            ReadOffset.from("0"),
            "test-group"
    );

    redisTemplate.opsForStream().add(
            "test-orders",
            Map.of("orderId", "1002")
    );

    List<MapRecord<String, Object, Object>> messages =
            redisTemplate.opsForStream()
                    .read(
                            Consumer.from("test-group", "consumer-1"),
                            StreamOffset.create(
                                    "test-orders",
                                    ReadOffset.lastConsumed()
                            )
                    );

    assertThat(messages).isNotEmpty();
}
```

---

## 11. Cache Failure Test

### Resilient Service

```java
public UserProfile getProfileWithFallback(Long userId) {
    try {
        UserProfile cached = (UserProfile) redisTemplate.opsForValue()
                .get("user-service:user:" + userId + ":profile");

        if (cached != null) {
            return cached;
        }
    } catch (Exception ex) {
        // Redis down: fallback to DB
    }

    return userRepository.findById(userId)
            .map(UserProfile::from)
            .orElseThrow();
}
```

### Unit Test

```java
@Test
void shouldFallbackToDatabaseWhenRedisFails() {
    when(redisTemplate.opsForValue())
            .thenThrow(new RedisConnectionFailureException("Redis down"));

    when(userRepository.findById(101L))
            .thenReturn(Optional.of(userProfile));

    UserProfile result = service.getProfileWithFallback(101L);

    assertThat(result.getId()).isEqualTo(101L);
}
```

---

## 12. Load Testing With k6

### `redis-load-test.js`

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 50 },
    { duration: '1m', target: 100 },
    { duration: '30s', target: 0 }
  ],
  thresholds: {
    http_req_duration: ['p(95)<300'],
    http_req_failed: ['rate<0.01']
  }
};

export default function () {
  const response = http.get('http://localhost:8080/api/users/101');

  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 300ms': (r) => r.timings.duration < 300
  });

  sleep(0.1);
}
```

### Run

```bash
k6 run redis-load-test.js
```

k6 ramping, soak aur spike tests support karta hai; normal load tests me commonly 50–200 virtual users aur capacity tests me 500–2000 VUs use hote hain. [402][405]

---

## 13. Load Test Metrics

| Metric | Target |
|---|---|
| p95 latency | < 300 ms |
| p99 latency | < 500 ms |
| Error rate | < 1% |
| Cache hit ratio | > 80% |
| Redis CPU | Stable |
| Redis memory | No continuous growth |
| Connection pool | No exhaustion |

---

## 14. Testing Checklist

- [ ] Redis service unit tests hain.
- [ ] RedisTemplate mocking covered hai.
- [ ] Testcontainers integration tests hain.
- [ ] Cache hit/miss tests hain.
- [ ] TTL/expiry tests hain.
- [ ] Serializer compatibility tests hain.
- [ ] Distributed lock tests hain.
- [ ] Rate limiter tests hain.
- [ ] Pub/Sub tests hain.
- [ ] Stream consumer/ACK tests hain.
- [ ] Redis failure fallback tests hain.
- [ ] Load tests hain.
- [ ] CI pipeline me integration tests hain.

---

## 15. Common Testing Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Sirf mock tests | Real Redis behavior untested | Testcontainers integration tests |
| Shared local Redis for tests | Test data pollution | Per-test container |
| TTL test me fixed sleep only | Flaky tests | Awaitility / bounded assertions |
| Serializer test missing | Production deserialization failure | DTO serialize/deserialize test |
| Lock test missing | Race condition undetected | Concurrent lock tests |
| No failure tests | Redis outage app down | Fallback/circuit breaker tests |
| No load test | Production capacity unknown | k6/JMeter/Gatling |

---

## 16. Interview Questions

| Question | Answer |
|---|---|
| Redis unit test aur integration test me difference? | Unit test Redis mock karta hai; integration test real Redis container use karta hai. |
| Testcontainers kya hai? | Tests ke liye temporary Docker containers start/cleanup karne wali Java library. |
| RedisTemplate kaise mock karte hain? | `RedisTemplate` aur `ValueOperations` ko Mockito se mock karke. |
| TTL test kaise karte hain? | `getExpire()` se TTL verify karke aur short TTL expiry test karke. |
| Distributed lock test kaise karte hain? | Pehla lock acquire, dusra reject, release ke baad naya lock acquire verify karke. |
| Pub/Sub test kaise karte hain? | Publisher message publish karo, Awaitility se subscriber received messages verify karo. |
| Redis Stream test kaise karte hain? | `XADD`, consumer group creation, read aur `XACK` verify karke. |
| Load testing kyun zaroori hai? | Connection pool, latency, throughput aur cache behavior production-like load me validate karne ke liye. |

---

## 17. Final Mental Model

```text
Unit Test:
Mock RedisTemplate
Verify business logic

Integration Test:
Real Redis via Testcontainers
Verify cache, TTL, lock, stream

Failure Test:
Redis down
Verify fallback

Load Test:
High concurrency
Verify latency, pool, throughput
```

---

## Next Part

**Part 29: Social Media Backend with Redis — Complete Production Project**

Isme hum ek complete production-style module banayenge:

- User profile caching
- Followers/following cache
- Feed caching
- Like count caching
- Trending posts leaderboard
- Rate limiting
- JWT refresh-token storage
- Online user presence
- Kafka-based cache invalidation
- Distributed lock for feed generation
- Monitoring and alerting
- Docker Compose setup
- Complete Spring Boot code structure
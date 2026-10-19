# Part 14: Rate Limiting with Redis — Complete Deep Dive

> Rate limiting API ko abuse, brute-force attacks aur overload se protect karta hai. Redis distributed rate limiting ke liye ideal hai kyunki saare application instances ek shared counter/state use karte hain.

---

## 1. Rate Limiting Kya Hai?

Rate limiting ka matlab hai:

```text
Ek user/IP/API key fixed time me
kitni requests bhej sakta hai.
```

### Example

```text
Maximum 100 requests per minute per user
```

### Without Rate Limiting

```text
Attacker -> 10,000 requests/second
Application -> Overloaded
Database -> Overloaded
```

### With Rate Limiting

```text
Attacker -> 10,000 requests/second
Redis -> First 100 allowed
Remaining -> 429 Too Many Requests
```

---

## 2. Rate Limit Key Design

Rate limit kis level par lagani hai, ye pehla decision hai.

| Level | Redis Key Example | Use Case |
|---|---|---|
| User | `rate-limit:user:101` | Logged-in APIs |
| IP | `rate-limit:ip:192.168.1.10` | Public APIs |
| API key | `rate-limit:api-key:abc123` | Partner APIs |
| Endpoint | `rate-limit:endpoint:/login` | Sensitive endpoint |
| Tenant | `rate-limit:tenant:company-1` | Multi-tenant SaaS |
| Combined | `rate-limit:user:101:endpoint:/orders` | Fine-grained control |

Redis rate limiter user, IP, API key, tenant ya model level par quotas enforce kar sakta hai. [204]

---

## 3. Rate Limiting Algorithms

| Algorithm | Memory | Accuracy | Burst Behavior | Best For |
|---|---|---|---|---|
| Fixed Window | Low | Approximate | Boundary burst possible | Simple API limits |
| Sliding Window Log | High | Exact | No burst | High-value APIs |
| Sliding Window Counter | Low | Near-exact | Smoothed | General APIs |
| Token Bucket | Low | High | Controlled burst | Bursty traffic |
| Leaky Bucket | Medium | High | Smooth output | Traffic shaping |

Redis documentation me fixed window, sliding window log, sliding window counter, token bucket aur leaky bucket — paanchon algorithms compare kiye gaye hain. [205]

---

## 4. Fixed Window Counter

Fixed window me time ko fixed intervals me divide karte hain.

```text
Window 1: 10:00:00 - 10:00:59
Window 2: 10:01:00 - 10:01:59
```

### Flow

```text
Request -> Current window calculate
        -> Redis INCR
        -> Count > limit? Reject : Allow
```

### Redis Commands

```bash
INCR rate-limit:user:101:202610191401
EXPIRE rate-limit:user:101:202610191401 60
```

### Problem: Boundary Burst

```text
Limit = 100 requests/minute

10:00:59 -> 100 requests allowed
10:01:00 -> New window, again 100 requests allowed

Within 2 seconds: 200 requests
```

Fixed window me window boundary par 2x burst possible hota hai. [205]

---

## 5. Fixed Window Spring Boot Implementation

```java
package com.example.ratelimit;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;

@Service
@RequiredArgsConstructor
public class FixedWindowRateLimiter {

    private final StringRedisTemplate redisTemplate;

    public boolean isAllowed(String clientId, int limit, Duration window) {
        long windowNumber =
                Instant.now().toEpochMilli() / window.toMillis();

        String key = "rate-limit:" + clientId + ":" + windowNumber;

        Long count = redisTemplate.opsForValue().increment(key);

        if (count != null && count == 1L) {
            redisTemplate.expire(key, window);
        }

        return count != null && count <= limit;
    }
}
```

### Controller Example

```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final FixedWindowRateLimiter rateLimiter;

    @GetMapping
    public ResponseEntity<String> getOrders(
            @RequestHeader("X-User-Id") Long userId
    ) {
        boolean allowed = rateLimiter.isAllowed(
                "user:" + userId,
                100,
                Duration.ofMinutes(1)
        );

        if (!allowed) {
            return ResponseEntity
                    .status(HttpStatus.TOO_MANY_REQUESTS)
                    .body("Rate limit exceeded");
        }

        return ResponseEntity.ok("Orders fetched");
    }
}
```

---

## 6. Sliding Window Log

Sliding window log me har request ka timestamp store hota hai.

```text
Current time: 10:01:30
Window: last 60 seconds

Valid timestamps:
10:00:31 to 10:01:30
```

### Redis Commands

```bash
ZADD rate-limit:user:101 1760000000000 request-id-1
ZREMRANGEBYSCORE rate-limit:user:101 0 1759999940000
ZCARD rate-limit:user:101
```

### Java Implementation

```java
package com.example.ratelimit;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class SlidingWindowLogRateLimiter {

    private final StringRedisTemplate redisTemplate;

    public boolean isAllowed(String clientId, int limit, Duration window) {
        String key = "rate-limit:sliding-log:" + clientId;

        long now = System.currentTimeMillis();
        long windowStart = now - window.toMillis();

        redisTemplate.opsForZSet()
                .removeRangeByScore(key, 0, windowStart);

        Long count = redisTemplate.opsForZSet().zCard(key);

        if (count != null && count >= limit) {
            return false;
        }

        redisTemplate.opsForZSet().add(
                key,
                UUID.randomUUID().toString(),
                now
        );

        redisTemplate.expire(key, window);

        return true;
    }
}
```

### Pros and Cons

| Feature | Details |
|---|---|
| Accuracy | Exact |
| Burst behavior | No boundary burst |
| Memory | High — har request timestamp store hota hai |
| Best for | High-value APIs, audit-sensitive flows |

Sliding window log exact rolling window enforce karta hai lekin har request ka timestamp store karne se memory usage zyada hota hai. [205]

---

## 7. Sliding Window Counter

Sliding window counter two fixed windows ka weighted average use karta hai.

```text
Allowed =
Previous window count x remaining previous window weight
+
Current window count
```

### Example

```text
Previous window: 80 requests
Current window elapsed: 30%
Previous window weight: 70%

Estimated recent requests:
80 x 0.7 + current count
```

### Java Implementation

```java
package com.example.ratelimit;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;

@Service
@RequiredArgsConstructor
public class SlidingWindowCounterRateLimiter {

    private final StringRedisTemplate redisTemplate;

    public boolean isAllowed(String clientId, int limit, Duration window) {
        long windowMillis = window.toMillis();
        long now = Instant.now().toEpochMilli();

        long currentWindow = now / windowMillis;
        long previousWindow = currentWindow - 1;

        String currentKey =
                "rate-limit:swc:" + clientId + ":" + currentWindow;

        String previousKey =
                "rate-limit:swc:" + clientId + ":" + previousWindow;

        Long currentCount = redisTemplate.opsForValue()
                .increment(currentKey);

        if (currentCount != null && currentCount == 1L) {
            redisTemplate.expire(currentKey, window);
        }

        String previousCountValue =
                redisTemplate.opsForValue().get(previousKey);

        long previousCount = previousCountValue == null
                ? 0
                : Long.parseLong(previousCountValue);

        double elapsedPercentage =
                (now % windowMillis) / (double) windowMillis;

        double previousWeight = 1 - elapsedPercentage;

        double estimatedCount =
                previousCount * previousWeight + currentCount;

        return estimatedCount <= limit;
    }
}
```

Sliding window counter two fixed windows ka weighted approximation use karta hai, jisse boundary burst smoothed hota hai aur memory usage low rehta hai. [205][212]

---

## 8. Token Bucket

Token bucket me tokens fixed rate par refill hote hain.

```text
Bucket capacity = 100 tokens
Refill rate = 10 tokens/second

Request -> 1 token consume
Tokens available? -> Allow
Bucket empty? -> Reject
```

### Benefits

- Short burst allow karta hai
- Long-term average rate control karta hai
- API-friendly behavior

Token bucket me tokens time ke saath refill hote hain, maximum capacity tak; har request ek token consume karti hai. [205]

### Redis Data Model

```text
rate-limit:token-bucket:user:101
  ├── tokens -> 75
  └── lastRefillTimestamp -> 1760000000000
```

### Lua Script

```lua
local key = KEYS
local capacity = tonumber(ARGV)
local refillRate = tonumber(ARGV)
local requestedTokens = tonumber(ARGV)
local now = tonumber(ARGV)

local bucket = redis.call('HMGET', key, 'tokens', 'lastRefill')
local tokens = tonumber(bucket)
local lastRefill = tonumber(bucket)

if tokens == nil then
    tokens = capacity
    lastRefill = now
end

local elapsed = math.max(0, now - lastRefill)
local refilledTokens = elapsed * refillRate

tokens = math.min(capacity, tokens + refilledTokens)

if tokens >= requestedTokens then
    tokens = tokens - requestedTokens
    redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
    redis.call('EXPIRE', key, 3600)
    return 1
else
    redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
    redis.call('EXPIRE', key, 3600)
    return 0
end
```

---

## 9. Token Bucket Spring Boot Service

```java
package com.example.ratelimit;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class TokenBucketRateLimiter {

    private final StringRedisTemplate redisTemplate;

    private static final String TOKEN_BUCKET_SCRIPT = """
            local key = KEYS
            local capacity = tonumber(ARGV)
            local refillRate = tonumber(ARGV)
            local requestedTokens = tonumber(ARGV)
            local now = tonumber(ARGV)

            local bucket = redis.call('HMGET', key, 'tokens', 'lastRefill')
            local tokens = tonumber(bucket)
            local lastRefill = tonumber(bucket)

            if tokens == nil then
                tokens = capacity
                lastRefill = now
            end

            local elapsed = math.max(0, now - lastRefill)
            local refilledTokens = elapsed * refillRate

            tokens = math.min(capacity, tokens + refilledTokens)

            if tokens >= requestedTokens then
                tokens = tokens - requestedTokens
                redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
                redis.call('EXPIRE', key, 3600)
                return 1
            else
                redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
                redis.call('EXPIRE', key, 3600)
                return 0
            end
            """;

    public boolean isAllowed(
            String clientId,
            int capacity,
            double refillRatePerSecond,
            int requestedTokens
    ) {
        String key = "rate-limit:token-bucket:" + clientId;

        Long result = redisTemplate.execute(
                new DefaultRedisScript<>(TOKEN_BUCKET_SCRIPT, Long.class),
                List.of(key),
                String.valueOf(capacity),
                String.valueOf(refillRatePerSecond),
                String.valueOf(requestedTokens),
                String.valueOf(System.currentTimeMillis())
        );

        return result != null && result == 1L;
    }
}
```

---

## 10. Redisson Rate Limiter

Redisson `RRateLimiter` distributed rate limiting ke liye ready-made abstraction deta hai. `RateType.OVERALL` hone par limit saare Redisson clients ke across shared hoti hai. [206]

### Dependency

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.50.0</version>
</dependency>
```

### Service

```java
package com.example.ratelimit;

import lombok.RequiredArgsConstructor;
import org.redisson.api.RRateLimiter;
import org.redisson.api.RateIntervalUnit;
import org.redisson.api.RateType;
import org.redisson.api.RedissonClient;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class RedissonRateLimiterService {

    private final RedissonClient redissonClient;

    public boolean isAllowed(String clientId) {
        String key = "rate-limit:redisson:" + clientId;

        RRateLimiter rateLimiter = redissonClient.getRateLimiter(key);

        rateLimiter.trySetRate(
                RateType.OVERALL,
                100,
                1,
                RateIntervalUnit.MINUTES
        );

        return rateLimiter.tryAcquire(1);
    }
}
```

### Rate Types

| Rate Type | Meaning |
|---|---|
| `OVERALL` | Limit saare instances me shared |
| `PER_CLIENT` | Har client instance ka apna limit |

`RateType.OVERALL` use karne par limit har Redisson client/JVM ke across shared hoti hai. [206]

---

## 11. Spring Boot Filter Rate Limiter

```java
package com.example.ratelimit;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.time.Duration;

@Component
@RequiredArgsConstructor
public class RateLimitFilter extends OncePerRequestFilter {

    private final FixedWindowRateLimiter rateLimiter;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        String clientId = resolveClientId(request);

        boolean allowed = rateLimiter.isAllowed(
                clientId,
                100,
                Duration.ofMinutes(1)
        );

        if (!allowed) {
            response.setStatus(429);
            response.setHeader("Retry-After", "60");
            response.getWriter().write("Too many requests");
            return;
        }

        filterChain.doFilter(request, response);
    }

    private String resolveClientId(HttpServletRequest request) {
        String userId = request.getHeader("X-User-Id");

        if (userId != null && !userId.isBlank()) {
            return "user:" + userId;
        }

        String apiKey = request.getHeader("X-API-Key");

        if (apiKey != null && !apiKey.isBlank()) {
            return "api-key:" + apiKey;
        }

        return "ip:" + request.getRemoteAddr();
    }
}
```

---

## 12. HTTP Response Headers

Rate limiting me client ko clear response dena chahiye.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1760000060
```

### Header Meanings

| Header | Meaning |
|---|---|
| `429` | Too many requests |
| `Retry-After` | Kitne seconds baad retry kare |
| `X-RateLimit-Limit` | Total allowed requests |
| `X-RateLimit-Remaining` | Remaining requests |
| `X-RateLimit-Reset` | Window reset time |

---

## 13. Rate Limiting Use Cases

| Use Case | Recommended Limit |
|---|---|
| Login API | 5 requests per minute per IP |
| OTP request | 3 requests per 10 minutes per user |
| Password reset | 3 requests per hour per email |
| Search API | 100 requests per minute per user |
| Public API | 1000 requests per hour per API key |
| Payment API | Strict per-user/per-tenant limit |

---

## 14. Production Best Practices

- [ ] Rate limit key me user/IP/API key identify karo.
- [ ] Sensitive endpoints par strict limits lagao.
- [ ] Lua script se atomic check-and-increment karo.
- [ ] `429` response with `Retry-After` return karo.
- [ ] Rate limit metrics monitor karo.
- [ ] Redis failure par graceful behavior define karo.
- [ ] Distributed deployment me shared Redis use karo.
- [ ] Different tiers ke liye alag limits rakho.
- [ ] Bot/abuse detection ke liye IP + user + API key combine karo.

---

## 15. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Local in-memory rate limiter | Multi-instance me limit bypass | Redis shared limiter |
| `INCR` and `EXPIRE` separately without atomicity | TTL miss ho sakta hai | Lua script |
| Sirf IP-based limit | NAT/proxy users unfairly blocked | User/API key + IP |
| Very strict limit | Legitimate users blocked | Tiered limits |
| No `429` response | Client retry behavior unclear | Proper HTTP response |
| No monitoring | Abuse late detect | Metrics + alerts |

---

## 16. Interview Questions

| Question | Answer |
|---|---|
| Rate limiting kya hai? | Fixed time window me requests ki count limit karna. |
| Fixed window rate limiter kya hai? | Fixed time interval me requests count karna aur limit cross hone par reject karna. |
| Fixed window ka problem kya hai? | Window boundary par 2x burst possible hota hai. |
| Sliding window log kya hai? | Har request timestamp store karke exact rolling window enforce karna. |
| Token bucket kya hai? | Tokens fixed rate par refill hote hain aur har request token consume karti hai. |
| Token bucket burst kyun allow karta hai? | Bucket me already accumulated tokens hote hain. |
| Redis rate limiter distributed kyun hai? | Saare application instances same Redis state use karte hain. |
| Redisson `RRateLimiter` kya hai? | Redis-backed distributed rate limiter abstraction. |
| Rate limit fail hone par kaunsa HTTP status return karna chahiye? | `429 Too Many Requests`. |

---

## 17. Final Mental Model

```text
Fixed window        -> Simple counter
Sliding window log  -> Exact but memory-heavy
Sliding counter     -> Approximate but efficient
Token bucket        -> Controlled burst
Leaky bucket        -> Smooth traffic

Redis               -> Shared distributed state
Lua                 -> Atomic check-and-update
429                 -> Client ko clear signal
```

---

## Next Part

**Part 15: Leaderboards and Counters with Redis — Complete Deep Dive**

Isme hum cover karenge:

- Sorted Set internals
- Leaderboard design
- `ZADD`, `ZINCRBY`, `ZSCORE`
- `ZRANK`, `ZREVRANK`
- `ZREVRANGE`
- Pagination
- Tie-breaking
- Real-time ranking
- Like/view counters
- Spring Boot leaderboard API
- Production best practices
- Interview questions
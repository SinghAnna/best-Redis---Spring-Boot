# Part 12: User Session Management with Redis — Complete Deep Dive

> Redis session management ka main benefit hai: session data ek shared, fast, external store me rehta hai. Isse multiple Spring Boot instances ke beech login state share hota hai aur logout/token revocation possible hota hai.

---

## 1. Stateless vs Stateful Authentication

### Stateless Authentication — JWT

```text
Client -> JWT -> API
API validates JWT signature
No server-side session required
```

### Stateful Authentication — Redis Session

```text
Client -> Session ID / Cookie -> API
API -> Redis -> Session data
```

### Comparison

| Feature | JWT | Redis Session |
|---|---|---|
| Server-side state | No | Yes |
| Instant logout | Difficult | Easy |
| Token revocation | Blacklist required | Delete session |
| Multi-instance support | Yes | Yes |
| Session metadata | Limited | Rich |
| DB/session lookup | No | Redis lookup |
| Best for | Microservices, APIs | Web apps, secure session flows |

---

## 2. Recommended Hybrid Model

Production me commonly hybrid model use hota hai:

```text
Access Token  -> Short-lived JWT
Refresh Token -> Redis-backed, revocable
Session Data  -> Redis
```

### Flow

```text
Login
  |
  +--> Access JWT: 15 minutes
  |
  +--> Refresh Token: 7 days, stored in Redis
  |
  +--> Session metadata: device, IP, user agent
```

Access token short-lived hota hai, jabki refresh token long-lived hota hai aur naya access token issue karne ke liye use hota hai. [183]

---

## 3. Spring Session Redis Setup

### Dependency

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### Configuration

```yaml
spring:
  session:
    store-type: redis
    redis:
      namespace: "spring:session:user-service"
      flush-mode: on-save
      save-mode: on-set-attribute
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD}
```

Spring Session Redis `HttpSession` ko Redis-backed implementation se replace karta hai; Spring Security ka `SecurityContext` bhi Redis me persist hota hai. [175]

---

## 4. Spring Session Example

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class SessionAuthController {

    @PostMapping("/login")
    public ResponseEntity<String> login(
            @RequestBody LoginRequest request,
            HttpSession session
    ) {
        // Validate user from database
        session.setAttribute("userId", 101L);
        session.setAttribute("username", request.getUsername());
        session.setAttribute("role", "USER");

        return ResponseEntity.ok("Logged in successfully");
    }

    @GetMapping("/me")
    public ResponseEntity<Map<String, Object>> me(HttpSession session) {
        Object userId = session.getAttribute("userId");

        if (userId == null) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }

        return ResponseEntity.ok(Map.of(
                "userId", userId,
                "username", session.getAttribute("username"),
                "role", session.getAttribute("role")
        ));
    }

    @PostMapping("/logout")
    public ResponseEntity<String> logout(HttpSession session) {
        session.invalidate();
        return ResponseEntity.ok("Logged out successfully");
    }
}
```

---

## 5. Redis Session Key Format

Spring Session Redis typically keys aise store karta hai:

```text
spring:session:user-service:sessions:{sessionId}
```

Example:

```text
spring:session:user-service:sessions:8f2b9c1e-4a3d-4f6a-9a1b-2c3d4e5f6a7b
```

### Session Data Example

```text
userId -> 101
username -> rahul
role -> USER
creationTime -> ...
lastAccessedTime -> ...
```

Spring Session `RedisSessionRepository` simple key-value structure use karta hai aur `spring.session.redis.namespace` se namespace customize hota hai. [177]

---

## 6. Session TTL

### Fixed TTL

```yaml
spring:
  session:
    redis:
      session-mapping-expiration: 30m
```

### Server Session Timeout

```yaml
server:
  servlet:
    session:
      timeout: 30m
```

### Sliding Expiration

Redis session me user activity hone par TTL refresh ho sakta hai. Matlab user active rehta rahega to session expire nahi hoga; inactivity ke specified time baad session expire ho jayega. [179]

```text
User active -> TTL refresh
User inactive for 30 minutes -> Session expires
```

---

## 7. Refresh Token Storage

### Recommended Design

```text
Refresh Token ID -> Redis
User ID -> Active Refresh Token IDs
```

### Redis Keys

```text
auth-service:refresh-token:{tokenId}
auth-service:user:{userId}:refresh-tokens
```

### Refresh Token Record

```text
auth-service:refresh-token:9f8b7c6d
  ├── userId -> 101
  ├── deviceId -> android-pixel-7
  ├── status -> ACTIVE
  ├── createdAt -> 2026-10-19T14:00:00
  └── expiresAt -> 2026-10-26T14:00:00
```

Refresh token ko server-side state me store karna usse revocable banata hai; logout ya revocation par Redis entry delete karne se naye access tokens issue hona band ho jate hain. [186]

---

## 8. Refresh Token Service

```java
package com.example.auth;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Set;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class RefreshTokenService {

    private final RedisTemplate<String, Object> redisTemplate;

    private static final Duration REFRESH_TOKEN_TTL =
            Duration.ofDays(7);

    public String createRefreshToken(Long userId, String deviceId) {
        String tokenId = UUID.randomUUID().toString();

        String tokenKey = "auth-service:refresh-token:" + tokenId;
        String userTokensKey = "auth-service:user:" + userId + ":refresh-tokens";

        RefreshTokenRecord record = RefreshTokenRecord.builder()
                .tokenId(tokenId)
                .userId(userId)
                .deviceId(deviceId)
                .status("ACTIVE")
                .build();

        redisTemplate.opsForValue().set(tokenKey, record, REFRESH_TOKEN_TTL);
        redisTemplate.opsForSet().add(userTokensKey, tokenId);
        redisTemplate.expire(userTokensKey, REFRESH_TOKEN_TTL);

        return tokenId;
    }

    public boolean isValidRefreshToken(String tokenId, Long userId) {
        String tokenKey = "auth-service:refresh-token:" + tokenId;

        Object value = redisTemplate.opsForValue().get(tokenKey);

        if (!(value instanceof RefreshTokenRecord record)) {
            return false;
        }

        return record.getUserId().equals(userId)
                && "ACTIVE".equals(record.getStatus());
    }

    public void revokeRefreshToken(String tokenId) {
        String tokenKey = "auth-service:refresh-token:" + tokenId;

        Object value = redisTemplate.opsForValue().get(tokenKey);

        if (value instanceof RefreshTokenRecord record) {
            record.setStatus("REVOKED");

            redisTemplate.opsForValue().set(
                    tokenKey,
                    record,
                    Duration.ofMinutes(10)
            );
        }
    }

    public void revokeAllUserTokens(Long userId) {
        String userTokensKey =
                "auth-service:user:" + userId + ":refresh-tokens";

        Set<Object> tokenIds =
                redisTemplate.opsForSet().members(userTokensKey);

        if (tokenIds != null) {
            for (Object tokenId : tokenIds) {
                revokeRefreshToken(tokenId.toString());
            }
        }

        redisTemplate.delete(userTokensKey);
    }
}
```

---

## 9. Refresh Token DTO

```java
package com.example.auth;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class RefreshTokenRecord {

    private String tokenId;
    private Long userId;
    private String deviceId;
    private String status;
}
```

---

## 10. JWT Access Token + Redis Refresh Token

### Login Flow

```text
1. User submits username/password
2. Application validates credentials
3. Application creates access JWT
4. Application creates refresh token
5. Refresh token stored in Redis
6. Access JWT returned to client
7. Refresh token returned in HttpOnly Secure Cookie
```

### Token Refresh Flow

```text
1. Access token expires
2. Client calls /auth/refresh
3. Server validates refresh token from Redis
4. Server issues new access JWT
5. Optional: rotate refresh token
6. Old refresh token invalidated
```

### Logout Flow

```text
1. Client calls /auth/logout
2. Server revokes refresh token in Redis
3. Optional: blacklist current access token JTI
4. Client clears token
```

Refresh token rotation me har refresh request par naya refresh token issue hota hai aur purana invalidate hota hai; reused token ko compromise indicator treat kiya jata hai. [180][182]

---

## 11. JWT Blacklist with Redis

JWT stateless hota hai, isliye logout ke baad bhi valid lag sakta hai jab tak expire na ho. Is problem ko solve karne ke liye token ka `jti` Redis blacklist me daal sakte hain.

### JWT Claims

```json
{
  "sub": "101",
  "jti": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "role": "USER",
  "iat": 1760000000,
  "exp": 1760000900
}
```

### Blacklist Key

```text
auth-service:blacklist:{jti}
```

### Blacklist Service

```java
package com.example.auth;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;

@Service
@RequiredArgsConstructor
public class JwtBlacklistService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void blacklistToken(String jti, Instant expiresAt) {
        Duration remainingTtl =
                Duration.between(Instant.now(), expiresAt);

        if (remainingTtl.isNegative() || remainingTtl.isZero()) {
            return;
        }

        redisTemplate.opsForValue().set(
                "auth-service:blacklist:" + jti,
                "REVOKED",
                remainingTtl
        );
    }

    public boolean isBlacklisted(String jti) {
        return Boolean.TRUE.equals(
                redisTemplate.hasKey("auth-service:blacklist:" + jti)
        );
    }
}
```

JWT revocation ke liye token ke `jti` ko Redis blacklist me store karna common pattern hai; blacklist entry ka TTL remaining token lifetime ke equal rakha jata hai. [181][186]

---

## 12. JWT Filter Example

```java
package com.example.auth;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtBlacklistService jwtBlacklistService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        String header = request.getHeader("Authorization");

        if (header == null || !header.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = header.substring(7);

        try {
            Claims claims = Jwts.parserBuilder()
                    .build()
                    .parseClaimsJws(token)
                    .getBody();

            String jti = claims.getId();
            Long userId = Long.parseLong(claims.getSubject());

            if (jwtBlacklistService.isBlacklisted(jti)) {
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                return;
            }

            UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                            userId,
                            null,
                            List.of()
                    );

            SecurityContextHolder.getContext()
                    .setAuthentication(authentication);

        } catch (Exception ex) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return;
        }

        filterChain.doFilter(request, response);
    }
}
```

---

## 13. Multi-Device Session Management

### Redis Data Model

```text
auth-service:user:101:refresh-tokens
  ├── token-1
  ├── token-2
  └── token-3

auth-service:refresh-token:token-1
  ├── userId -> 101
  ├── deviceId -> android-pixel-7
  └── status -> ACTIVE

auth-service:refresh-token:token-2
  ├── userId -> 101
  ├── deviceId -> macbook-chrome
  └── status -> ACTIVE
```

Multi-device session management me user ke saare active sessions Redis Set me track kiye jate hain, jisse per-device logout aur logout-all implement hota hai. [184]

### Get Active Devices

```java
public Set<Object> getActiveDevices(Long userId) {
    String userTokensKey =
            "auth-service:user:" + userId + ":refresh-tokens";

    Set<Object> tokenIds =
            redisTemplate.opsForSet().members(userTokensKey);

    if (tokenIds == null) {
        return Set.of();
    }

    return tokenIds.stream()
            .map(tokenId -> {
                String tokenKey =
                        "auth-service:refresh-token:" + tokenId;

                Object record =
                        redisTemplate.opsForValue().get(tokenKey);

                if (record instanceof RefreshTokenRecord refreshToken) {
                    return refreshToken.getDeviceId();
                }

                return null;
            })
            .filter(deviceId -> deviceId != null)
            .collect(java.util.stream.Collectors.toSet());
}
```

### Logout From One Device

```java
public void logoutDevice(Long userId, String tokenId) {
    revokeRefreshToken(tokenId);

    redisTemplate.opsForSet().remove(
            "auth-service:user:" + userId + ":refresh-tokens",
            tokenId
    );
}
```

### Logout From All Devices

```java
public void logoutAllDevices(Long userId) {
    revokeAllUserTokens(userId);
}
```

---

## 14. Session Security Best Practices

- [ ] Access token short rakho: 5–15 minutes.
- [ ] Refresh token revocable rakho.
- [ ] Refresh token rotation use karo.
- [ ] Refresh token ko HttpOnly, Secure, SameSite cookie me rakho.
- [ ] Refresh token raw form me Redis me store na karo; hash store karo.
- [ ] JWT me `jti` include karo.
- [ ] Logout par refresh token revoke karo.
- [ ] Password change par all sessions revoke karo.
- [ ] Suspicious login par session invalidation karo.
- [ ] Redis TLS aur authentication enable karo.

Refresh token ko client side secure cookie me rakhna, use-only-for-refresh rakhna aur rotation/revocation support karna recommended security practices hain. [183]

---

## 15. Session Architecture

```text
Client
  |
  | Access JWT + Refresh Cookie
  v
API Gateway
  |
  v
Auth Service
  |
  +--> Validate JWT
  |
  +--> Check Redis blacklist
  |
  +--> Issue/refresh tokens
  |
  v
Redis
  |
  +--> Refresh tokens
  |
  +--> Session data
  |
  +--> Token blacklist
  |
  +--> Device sessions
```

---

## 16. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Long-lived access JWT | Revocation difficult | 5–15 minute access token |
| Refresh token DB me plaintext | Security risk | Hashed/encrypted token |
| No refresh token rotation | Replay attack risk | Rotate on every refresh |
| No `jti` in JWT | Token blacklist difficult | Unique JWT ID |
| Logout par sirf client-side clear | Token still valid server-side | Redis revocation |
| Same session for all devices | Device-level logout impossible | Per-device session records |
| No TTL | Redis memory leak | Session/refresh token TTL |

---

## 17. Interview Questions

| Question | Answer |
|---|---|
| Redis session kya hai? | User session data ko Redis me store karna, multiple application instances ke beech share karne ke liye. |
| JWT aur Redis session me difference? | JWT stateless hota hai; Redis session server-side stateful hota hai. |
| JWT logout kaise implement karte hain? | JWT ke `jti` ko Redis blacklist me daal kar. |
| Refresh token revocation kaise hoti hai? | Redis me refresh token record delete/update karke. |
| Refresh token rotation kya hai? | Har refresh request par naya refresh token issue karna aur purana invalidate karna. |
| Multi-device session kaise track karte hain? | User ID ke saath active refresh token/session IDs Redis Set me store karke. |
| Sliding expiration kya hai? | User activity par session TTL refresh hona. |
| Spring Session Redis kya karta hai? | `HttpSession` ko Redis-backed implementation se replace karta hai. |

---

## 18. Final Mental Model

```text
Access JWT   -> Short-lived, stateless
Refresh token -> Long-lived, Redis-backed, revocable
Session data -> Redis
Blacklist    -> JWT revocation
Device set   -> Multi-device session control
```

---

## Next Part

**Part 13: Distributed Locks with Redis — Complete Deep Dive**

Isme hum cover karenge:

- Distributed lock kya hai
- Race conditions
- `SET NX PX`
- Lock expiry problem
- Lock ownership problem
- Redisson `RLock`
- `tryLock`
- Watchdog mechanism
- RedLock discussion
- Spring Boot implementation
- Idempotency keys
- Production best practices
- Interview questions
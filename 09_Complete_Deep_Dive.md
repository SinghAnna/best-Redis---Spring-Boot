# Part 9: `@Cacheable`, `@CachePut`, `@CacheEvict` — Complete Deep Dive

> Ye teen annotations Spring Cache abstraction ka core hain. Inka sahi use karne se DB load kam hota hai, response fast hota hai, aur stale cache problem control me rehti hai.

---

## 1. Annotation Overview

| Annotation | Method Execution | Cache Read | Cache Write | Cache Delete |
|---|---|---|---|---|
| `@Cacheable` | Cache hit par skip | Yes | On miss | No |
| `@CachePut` | Always | Optional | Always | No |
| `@CacheEvict` | Usually yes | No | No | Yes |

Spring documentation ke according `@Cacheable` cache population trigger karta hai, `@CacheEvict` eviction trigger karta hai, aur `@CachePut` method execution ko interfere kiye bina cache update karta hai. [116]

---

## 2. Enable Spring Cache

```java
@SpringBootApplication
@EnableCaching
public class UserServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

`@EnableCaching` ke bina ye annotations kaam nahi karenge.

---

## 3. `@Cacheable`

`@Cacheable` method result ko cache karta hai.

### Basic Example

```java
@Service
@RequiredArgsConstructor
public class UserProfileService {

    private final UserRepository userRepository;

    @Cacheable(
            value = "userProfile",
            key = "#userId"
    )
    public UserProfileResponse getUserProfile(Long userId) {
        return userRepository.findById(userId)
                .map(UserProfileResponse::from)
                .orElseThrow();
    }
}
```

### Flow

```text
getUserProfile(101)
        |
        v
Check Redis: userProfile::101
        |
        +--> Cache Hit -> Return cached value
        |
        +--> Cache Miss -> Call DB
                         -> Store result in Redis
                         -> Return result
```

### Redis Key

```text
userProfile::101
```

Agar global prefix configured hai:

```text
app:user-service:userProfile::101
```

---

## 4. `@Cacheable` Attributes

```java
@Cacheable(
        value = "userProfile",
        key = "#userId",
        condition = "#userId > 0",
        unless = "#result == null",
        sync = true
)
public UserProfileResponse getUserProfile(Long userId) {
    return userRepository.findById(userId)
            .map(UserProfileResponse::from)
            .orElse(null);
}
```

| Attribute | Purpose |
|---|---|
| `value` / `cacheNames` | Cache name |
| `key` | Custom cache key |
| `keyGenerator` | Custom key generation strategy |
| `condition` | Method call se pehle caching condition |
| `unless` | Method result ke baad caching veto |
| `sync` | Same key par concurrent method execution control |
| `cacheManager` | Custom CacheManager |
| `cacheResolver` | Custom cache resolver |

`@Cacheable` me default key method parameters se compute hoti hai; custom SpEL `key` attribute ya custom `KeyGenerator` bhi use kar sakte hain. [133]

---

## 5. `condition` vs `unless`

### `condition`

- Method execution **se pehle** evaluate hota hai.
- `false` hone par cache read/write dono skip hote hain.

```java
@Cacheable(
        value = "userProfile",
        key = "#userId",
        condition = "#userId > 0"
)
public UserProfileResponse getUserProfile(Long userId) {
    return userRepository.findById(userId)
            .map(UserProfileResponse::from)
            .orElseThrow();
}
```

### `unless`

- Method execution **ke baad** evaluate hota hai.
- `true` hone par result cache nahi hota.
- `#result` access kar sakta hai.

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

### Difference Table

| Attribute | Evaluation Time | Can Use `#result`? | If True |
|---|---|---|---|
| `condition` | Before method | No | Cache enabled |
| `unless` | After method | Yes | Result not cached |

`condition` method call se pehle evaluate hota hai, jabki `unless` method invocation ke baad evaluate hota hai. [136][138]

---

## 6. `sync = true`

`sync = true` same cache key par concurrent calls ko coordinate karta hai, jisse cache stampede kam hota hai.

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

### Without Sync

```text
Hot key expire

100 requests arrive
    |
    +--> 100 DB calls
```

### With Sync

```text
Hot key expire

100 requests arrive
    |
    +--> 1 request loads from DB
    |
    +--> Others wait / use updated cache
```

`sync = true` same key ke concurrent callers ko ek computation ka wait karne deta hai, jisse backend stampede avoid hota hai. [142]

---

## 7. SpEL Expressions

SpEL ka full form hai **Spring Expression Language**.

### Method Parameter

```java
@Cacheable(value = "userProfile", key = "#userId")
public UserProfileResponse getUserProfile(Long userId) {
    ...
}
```

### Positional Parameter

```java
@Cacheable(value = "userProfile", key = "#p0")
public UserProfileResponse getUserProfile(Long userId) {
    ...
}
```

### Multiple Parameters

```java
@Cacheable(
        value = "userFollowers",
        key = "#userId + ':' + #page"
)
public List<UserResponse> getFollowers(Long userId, int page) {
    ...
}
```

### Object Property

```java
@Cacheable(
        value = "userProfile",
        key = "#request.userId"
)
public UserProfileResponse getUserProfile(UserRequest request) {
    ...
}
```

### Method Result

```java
@CachePut(
        value = "userProfile",
        key = "#result.id"
)
public UserProfileResponse updateProfile(UpdateProfileRequest request) {
    ...
}
```

### Common SpEL Examples

| Expression | Meaning |
|---|---|
| `#userId` | Method parameter |
| `#p0` | First parameter |
| `#request.userId` | Object property |
| `#result.id` | Returned object property |
| `#userId + ':' + #page` | Composite key |
| `#root.methodName` | Method name |
| `#root.targetClass` | Target class |

---

## 8. `@CachePut`

`@CachePut` method ko hamesha execute karta hai aur returned value ko cache me update karta hai. [116][142]

### Example

```java
@Service
@RequiredArgsConstructor
public class UserProfileService {

    private final UserRepository userRepository;

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

        UserProfile saved = userRepository.save(user);

        return UserProfileResponse.from(saved);
    }
}
```

### Flow

```text
updateProfile(101, request)
        |
        v
Method always executes
        |
        v
DB update
        |
        v
Redis cache update: userProfile::101
        |
        v
Return updated response
```

### `@Cacheable` vs `@CachePut`

| Feature | `@Cacheable` | `@CachePut` |
|---|---|---|
| Cache hit par method skip | Yes | No |
| Method execution | Conditional | Always |
| Cache write | On cache miss | Always |
| Use case | Read operation | Write/update operation |

---

## 9. `@CacheEvict`

`@CacheEvict` cache entry remove karta hai.

### Single Key Eviction

```java
@CacheEvict(
        value = "userProfile",
        key = "#userId"
)
public void deleteUser(Long userId) {
    userRepository.deleteById(userId);
}
```

### All Entries Eviction

```java
@CacheEvict(
        value = "userProfile",
        allEntries = true
)
public void refreshAllUserProfiles() {
    // Admin operation
}
```

`allEntries = true` hone par cache ke andar ki saari entries remove hoti hain. [131][140]

### Multiple Cache Eviction

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

## 10. `beforeInvocation`

By default `@CacheEvict` method successful execution ke **baad** chalta hai.

```java
@CacheEvict(
        value = "userProfile",
        key = "#userId"
)
public void deleteUser(Long userId) {
    userRepository.deleteById(userId);
}
```

Agar method exception throw kare, to eviction nahi hogi.

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

`beforeInvocation = true` hone par eviction method invocation se pehle hoti hai, chahe method exception throw kare. [131]

### When to Use

| Scenario | Recommended |
|---|---|
| DB delete fail ho to stale cache na rahe | `beforeInvocation = true` |
| DB operation successful hone par hi cache clear karna | Default `false` |
| Cache refresh admin operation | `beforeInvocation = true` |

---

## 11. Self-Invocation Problem

Spring Cache proxy-based hota hai.

```java
@Service
public class UserService {

    @Cacheable(value = "userProfile", key = "#userId")
    public UserProfileResponse getUserProfile(Long userId) {
        return userRepository.findById(userId)
                .map(UserProfileResponse::from)
                .orElseThrow();
    }

    public UserProfileResponse loadUserProfile(Long userId) {
        // Self-invocation
        return getUserProfile(userId);
    }
}
```

### Problem

```text
loadUserProfile()
    |
    +--> this.getUserProfile()
            |
            +--> Proxy bypass
            |
            +--> Cache annotation ignore
```

Proxy mode me sirf external calls proxy ke through intercept hote hain; same class ke andar se method call karne par caching trigger nahi hoti. [132][135]

### Solution 1: Separate Service

```java
@Service
@RequiredArgsConstructor
public class UserProfileCacheService {

    @Cacheable(value = "userProfile", key = "#userId")
    public UserProfileResponse getUserProfile(Long userId) {
        return userRepository.findById(userId)
                .map(UserProfileResponse::from)
                .orElseThrow();
    }
}

@Service
@RequiredArgsConstructor
public class UserService {

    private final UserProfileCacheService userProfileCacheService;

    public UserProfileResponse loadUserProfile(Long userId) {
        return userProfileCacheService.getUserProfile(userId);
    }
}
```

### Solution 2: Inject Self Proxy

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final UserService self;

    @Cacheable(value = "userProfile", key = "#userId")
    public UserProfileResponse getUserProfile(Long userId) {
        return userRepository.findById(userId)
                .map(UserProfileResponse::from)
                .orElseThrow();
    }

    public UserProfileResponse loadUserProfile(Long userId) {
        return self.getUserProfile(userId);
    }
}
```

---

## 12. Complete Production Example

### DTO

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserProfileResponse {

    private Long id;
    private String name;
    private String email;
    private String city;

    public static UserProfileResponse from(UserProfile user) {
        return UserProfileResponse.builder()
                .id(user.getId())
                .name(user.getName())
                .email(user.getEmail())
                .city(user.getCity())
                .build();
    }
}
```

### Repository

```java
public interface UserRepository extends JpaRepository<UserProfile, Long> {
}
```

### Service

```java
package com.example.service;

import com.example.dto.UserProfileResponse;
import com.example.dto.UpdateProfileRequest;
import com.example.entity.UserProfile;
import com.example.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.cache.annotation.CacheConfig;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@CacheConfig(cacheNames = "userProfile")
@RequiredArgsConstructor
public class UserProfileService {

    private final UserRepository userRepository;

    @Cacheable(
            key = "#userId",
            unless = "#result == null",
            sync = true
    )
    public UserProfileResponse getUserProfile(Long userId) {
        return userRepository.findById(userId)
                .map(UserProfileResponse::from)
                .orElse(null);
    }

    @CachePut(key = "#result.id")
    @Transactional
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

    @CacheEvict(key = "#userId")
    @Transactional
    public void deleteUser(Long userId) {
        userRepository.deleteById(userId);
    }
}
```

---

## 13. Cache Key Design Best Practices

### Good

```text
userProfile::101
post::201
userFollowers::101:0
product::501
```

### Bad

```text
user
data
key1
temp
```

### Composite Key

```java
@Cacheable(
        value = "userFollowers",
        key = "#userId + ':' + #page"
)
public List<UserResponse> getFollowers(Long userId, int page) {
    ...
}
```

---

## 14. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| `@EnableCaching` missing | Annotations ignore | Enable caching |
| Self-invocation | Cache annotation bypass | Separate service / self proxy |
| Wrong SpEL key | Wrong cache entry | Key carefully design karo |
| `condition` aur `unless` confuse karna | Unexpected caching | Evaluation time samjho |
| Entity cache karna | Lazy loading / serialization issue | DTO cache karo |
| `allEntries = true` frequent use | Cache thanda ho jata hai | Specific key evict |
| No TTL | Stale data / memory issue | CacheManager TTL set karo |
| Exception par cache put | Invalid data cache | Exception handling / `unless` |

---

## 15. Interview Questions

| Question | Answer |
|---|---|
| `@Cacheable` kya karta hai? | Method result ko cache karta hai; cache hit par method skip hota hai. |
| `@CachePut` kya karta hai? | Method hamesha execute hota hai aur result cache me update hota hai. |
| `@CacheEvict` kya karta hai? | Cache entry ya poori cache remove karta hai. |
| `condition` aur `unless` me difference? | `condition` method se pehle, `unless` method ke baad evaluate hota hai. |
| `sync = true` kya karta hai? | Same key par concurrent cache miss requests ko coordinate karta hai. |
| `allEntries = true` kya karta hai? | Cache ke saare entries remove karta hai. |
| `beforeInvocation = true` kya karta hai? | Method se pehle cache eviction karta hai. |
| Self-invocation me cache kyun kaam nahi karta? | Spring proxy bypass hota hai. |
| SpEL kya hai? | Spring Expression Language, jisse method parameters aur result se cache key/condition banate hain. |

---

## 16. Final Mental Model

```text
@Cacheable
    -> Read data
    -> Cache hit? Return cache
    -> Cache miss? Execute method + store result

@CachePut
    -> Always execute method
    -> Update cache with result

@CacheEvict
    -> Remove cache entry
    -> Optional: before method execution
    -> Optional: remove all entries
```

---

## Next Part

**Part 10: Redis Cache Configuration — Complete Deep Dive**

Isme hum cover karenge:

- `RedisCacheManager`
- `RedisCacheConfiguration`
- Default TTL
- Cache-specific TTL
- Key prefix
- Null caching
- Transaction-aware caching
- Custom serialization
- Multiple cache regions
- Cache naming conventions
- Production configuration
- Interview questions
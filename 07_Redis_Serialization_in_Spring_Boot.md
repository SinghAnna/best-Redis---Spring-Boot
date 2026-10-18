# Part 7: Redis Serialization in Spring Boot — Complete Deep Dive

> Redis me data hamesha bytes ke form me store hota hai. Java object ko Redis me bhejne se pehle serialize karna padta hai, aur Redis se wapas aane par deserialize karna padta hai. Serializer galat choose kiya to cache corrupt, `ClassCastException`, ya cross-service compatibility issue aa sakta hai.

---

## 1. Serialization Kya Hai?

Serialization ka matlab:

```text
Java Object -> Bytes
```

Deserialization ka matlab:

```text
Bytes -> Java Object
```

### Redis Flow

```text
Java Object
   |
   | Serialize
   v
Byte Array
   |
   v
Redis Server

Redis Server
   |
   v
Byte Array
   |
   | Deserialize
   v
Java Object
```

---

## 2. Why Serialization Matters

Agar tumhare paas ye Java object hai:

```java
public class UserProfile {
    private Long id;
    private String name;
    private String email;
}
```

To Redis me ise store karne ke multiple tarike hain:

### Java Serialization

```text
Binary Java-specific format
```

### JSON Serialization

```json
{
  "id": 101,
  "name": "Rahul",
  "email": "rahul@example.com"
}
```

### String Serialization

```text
Rahul
```

Production me mostly **JSON serialization** best choice hoti hai.

---

## 3. RedisSerializer Interface

Spring Data Redis me `RedisSerializer` core interface hai.

```java
public interface RedisSerializer<T> {

    byte[] serialize(T value) throws SerializationException;

    T deserialize(byte[] bytes) throws SerializationException;
}
```

`RedisSerializer` objects ko byte arrays me aur byte arrays se objects me convert karta hai. [103]

### Common Implementations

| Serializer | Format | Use Case |
|---|---|---|
| `StringRedisSerializer` | UTF-8 String | Keys, tokens, simple values |
| `JdkSerializationRedisSerializer` | Java binary | Legacy Java-only apps |
| `GenericJackson2JsonRedisSerializer` | JSON with type info | Generic object caching |
| `Jackson2JsonRedisSerializer<T>` | JSON for fixed class | Known DTO type |
| `GenericToStringSerializer` | String conversion | Simple conversions |

---

## 4. JDK Serialization Problems

### Example

```java
template.setValueSerializer(
        RedisSerializer.java()
);
```

### Problems

| Problem | Explanation |
|---|---|
| Java-only format | Python, Node.js, Go service data read nahi kar sakti |
| Class compatibility issue | Class package/name change hone par deserialization fail |
| Security risk | Unsafe deserialization possible |
| Larger payload | JSON se zyada size ho sakta hai |
| Difficult debugging | Redis CLI me readable data nahi dikhta |

### Production Rule

```text
JDK serialization avoid karo.
JSON serialization use karo.
```

---

## 5. StringRedisSerializer

`StringRedisSerializer` simple `String` ko UTF-8 byte array me convert karta hai. [103]

### Configuration

```java
@Bean
public RedisTemplate<String, String> stringRedisTemplate(
        RedisConnectionFactory connectionFactory
) {
    RedisTemplate<String, String> template = new RedisTemplate<>();
    template.setConnectionFactory(connectionFactory);

    template.setKeySerializer(new StringRedisSerializer());
    template.setValueSerializer(new StringRedisSerializer());

    return template;
}
```

### Use Case

```java
stringRedisTemplate.opsForValue().set("user:101:name", "Rahul");
```

Redis me:

```text
user:101:name -> Rahul
```

### Best For

- Redis keys
- Tokens
- OTPs
- Counters
- Simple flags
- Raw JSON strings

---

## 6. GenericJackson2JsonRedisSerializer

`GenericJackson2JsonRedisSerializer` objects ko JSON me serialize karta hai aur dynamic typing use karta hai. Ye multiple Java object types ko handle kar sakta hai. [104]

### Configuration

```java
@Bean
public RedisTemplate<String, Object> redisTemplate(
        RedisConnectionFactory connectionFactory
) {
    RedisTemplate<String, Object> template = new RedisTemplate<>();
    template.setConnectionFactory(connectionFactory);

    template.setKeySerializer(new StringRedisSerializer());
    template.setValueSerializer(
            new GenericJackson2JsonRedisSerializer()
    );

    return template;
}
```

### Stored JSON Example

```json
{
  "@class": "com.example.dto.UserProfile",
  "id": 101,
  "name": "Rahul",
  "email": "rahul@example.com"
}
```

### Advantage

```text
Ek hi RedisTemplate se multiple object types cache kar sakte ho.
```

### Caution

Type information JSON me store hoti hai, isliye:

- Package rename hone par purani cache entries break ho sakti hain.
- Security ke liye deserialization allowlist better hoti hai.
- Cache versioning important hoti hai.

---

## 7. Full JSON RedisTemplate Configuration

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
public class RedisSerializerConfig {

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
        mapper.disable(
                com.fasterxml.jackson.databind.SerializationFeature.WRITE_DATES_AS_TIMESTAMPS
        );

        mapper.activateDefaultTyping(
                LaissezFaireSubTypeValidator.instance,
                ObjectMapper.DefaultTyping.NON_FINAL,
                JsonTypeInfo.As.PROPERTY
        );

        return mapper;
    }
}
```

`GenericJackson2JsonRedisSerializer` dynamic typing ke saath objects ko JSON me map karta hai; default typing enable karne se JSON payload me class information include hoti hai. [104][112]

---

## 8. `LocalDateTime` Serialization

`LocalDateTime` ko correctly serialize karne ke liye `JavaTimeModule` register karna hota hai. Jackson `LocalDateTime` handle karne ke liye `jackson-datatype-jsr310` module aur `JavaTimeModule` registration require karta hai. [108][109]

### Dependency

```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
</dependency>
```

### ObjectMapper

```java
ObjectMapper mapper = new ObjectMapper();
mapper.registerModule(new JavaTimeModule());
mapper.disable(
        SerializationFeature.WRITE_DATES_AS_TIMESTAMPS
);
```

### DTO

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserProfile {

    private Long id;
    private String name;
    private LocalDateTime createdAt;
}
```

### Stored JSON

```json
{
  "@class": "com.example.dto.UserProfile",
  "id": 101,
  "name": "Rahul",
  "createdAt": "2026-10-18T22:30:00"
}
```

---

## 9. Jackson2JsonRedisSerializer

`Jackson2JsonRedisSerializer<T>` ek specific class ke liye fixed JSON serializer hota hai.

### Configuration

```java
@Bean
public RedisTemplate<String, UserProfile> userProfileRedisTemplate(
        RedisConnectionFactory connectionFactory
) {
    RedisTemplate<String, UserProfile> template = new RedisTemplate<>();
    template.setConnectionFactory(connectionFactory);

    template.setKeySerializer(new StringRedisSerializer());

    ObjectMapper mapper = new ObjectMapper();
    mapper.registerModule(new JavaTimeModule());
    mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

    template.setValueSerializer(
            new Jackson2JsonRedisSerializer<>(mapper, UserProfile.class)
    );

    return template;
}
```

### Generic vs Fixed Serializer

| Feature | `GenericJackson2JsonRedisSerializer` | `Jackson2JsonRedisSerializer<T>` |
|---|---|---|
| Object type | Dynamic | Fixed |
| Type metadata | JSON me store hoti hai | Not required |
| Multiple DTO support | Yes | Ek template per type |
| Safety | Careful configuration needed | Safer for known type |
| Best for | Generic cache service | Specific DTO cache |

---

## 10. Cache Versioning

Jab DTO structure change hota hai, purani cache entries deserialize fail ho sakti hain.

### Problem

Purana object:

```json
{
  "id": 101,
  "name": "Rahul"
}
```

Naya DTO:

```java
public class UserProfile {
    private Long id;
    private String name;
    private String phoneNumber;
}
```

Agar `phoneNumber` primitive ho:

```java
private String phoneNumber;
```

To usually `null` milega.

Lekin agar type change ho:

```java
private int phoneNumber;
```

To deserialization fail ho sakti hai.

### Solution: Cache Version Prefix

```text
user-service:v1:user:101:profile
user-service:v2:user:101:profile
```

### Java Example

```java
private static final String CACHE_VERSION = "v2";

private String profileKey(Long userId) {
    return "user-service:" + CACHE_VERSION + ":user:" + userId + ":profile";
}
```

### Deployment Strategy

```text
1. New version key introduce karo
2. New data v2 key me cache karo
3. Purani v1 keys TTL se naturally expire hone do
4. Ya migration job se safely delete karo
```

---

## 11. Backward Compatibility Rules

| Change | Compatibility |
|---|---|
| New optional field add karna | Usually safe |
| Field remove karna | Usually safe if not required |
| Field type change karna | Risky |
| Field rename karna | Risky |
| Nested object type change | Risky |
| Package/class rename | Risky with default typing |
| Enum value add karna | Usually safe |
| Enum value remove karna | Risky |

### Safe DTO Evolution Example

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserProfile {

    private Long id;
    private String name;

    // New optional field
    private String phoneNumber;
}
```

---

## 12. Polymorphic Serialization

Agar cache me multiple subclasses store karne hain:

```java
public abstract class Notification {
    private Long id;
}

public class EmailNotification extends Notification {
    private String email;
}

public class SmsNotification extends Notification {
    private String phone;
}
```

To `GenericJackson2JsonRedisSerializer` + default typing useful hota hai.

### Stored JSON

```json
{
  "@class": "com.example.notification.EmailNotification",
  "id": 1,
  "email": "rahul@example.com"
}
```

### Security Recommendation

Production me unrestricted default typing risky ho sakta hai. Prefer karo:

- Fixed DTO serializer
- Explicit allowlist
- Separate cache keys per type
- Versioned cache keys

---

## 13. Common Serialization Errors

### 1. `ClassCastException`

```text
java.lang.ClassCastException:
class java.lang.String cannot be cast to class com.example.dto.UserProfile
```

Reason:

```text
Write time par ek serializer,
read time par dusra serializer.
```

Fix:

```text
Same serializer configuration for write and read.
```

### 2. `InvalidDefinitionException`

```text
Java 8 date/time type not supported by default
```

Fix:

```java
mapper.registerModule(new JavaTimeModule());
```

### 3. `SerializationException`

Reason:

```text
DTO me default constructor nahi hai
ya Jackson ko object create karne me problem hai.
```

Fix:

```java
@NoArgsConstructor
@AllArgsConstructor
```

### 4. Garbled / unreadable Redis value

Reason:

```text
JDK serialization ya wrong serializer
```

Fix:

```text
JSON serializer use karo.
```

---

## 14. Production Serializer Strategy

### Recommended Default

```text
Key   -> StringRedisSerializer
Value -> GenericJackson2JsonRedisSerializer
```

### For High-Security Systems

```text
Key   -> StringRedisSerializer
Value -> Jackson2JsonRedisSerializer<SpecificDTO>
```

### For Cross-Language Systems

```text
Key   -> StringRedisSerializer
Value -> Plain JSON without Java type metadata
```

### For High-Performance Internal Systems

```text
Protocol Buffers / Avro / Kryo
```

Lekin ye tabhi jab team aur ecosystem ready ho.

---

## 15. Complete Production Configuration

```java
package com.example.config;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.jsontype.impl.LaissezFaireSubTypeValidator;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class ProductionRedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory
    ) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        StringRedisSerializer keySerializer = new StringRedisSerializer();
        GenericJackson2JsonRedisSerializer valueSerializer =
                new GenericJackson2JsonRedisSerializer(redisObjectMapper());

        template.setKeySerializer(keySerializer);
        template.setHashKeySerializer(keySerializer);

        template.setValueSerializer(valueSerializer);
        template.setHashValueSerializer(valueSerializer);

        template.setEnableTransactionSupport(false);
        template.afterPropertiesSet();

        return template;
    }

    private ObjectMapper redisObjectMapper() {
        ObjectMapper mapper = new ObjectMapper();

        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

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

## 16. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| JDK serialization | Cross-service issue, security risk | JSON serializer |
| Different serializer for write/read | Deserialization failure | Consistent serializer |
| `LocalDateTime` without JavaTimeModule | Serialization error | `JavaTimeModule` register karo |
| Package rename without cache versioning | Old cache break | Versioned keys |
| Sensitive data in cache | Data leak risk | Encrypt ya sensitive fields exclude karo |
| Very large JSON value | Memory/network overhead | Small DTO cache karo |
| Polymorphic default typing blindly | Security risk | Allowlist/fixed serializer |

---

## 17. Interview Questions

| Question | Answer |
|---|---|
| Redis serialization kya hai? | Java object ko Redis-compatible bytes me convert karna. |
| `RedisSerializer` kya karta hai? | Object ko byte array me serialize aur byte array se deserialize karta hai. |
| JDK serialization kyun avoid karte hain? | Java-specific format, compatibility aur security issues ki wajah se. |
| JSON serialization kyun better hai? | Readable, cross-language compatible aur easier debugging. |
| `GenericJackson2JsonRedisSerializer` kya hai? | Dynamic typing ke saath objects ko JSON me serialize karne wala serializer. |
| `Jackson2JsonRedisSerializer` kab use karte hain? | Jab ek fixed DTO type cache karna ho. |
| `LocalDateTime` serialization kaise fix karte hain? | `JavaTimeModule` register karke. |
| Cache versioning kya hai? | DTO structure change hone par purani aur nayi cache entries ko alag version key me rakhna. |
| Polymorphic object cache kaise karte hain? | Generic JSON serializer with type metadata, lekin security carefully handle karke. |

---

## 18. Final Mental Model

```text
Redis stores bytes.

Java object
   -> Serializer
      -> Bytes
         -> Redis

Redis
   -> Bytes
      -> Deserializer
         -> Java object

Production default:
String key serializer
+
JSON value serializer
+
Versioned cache keys
```

---

## Next Part

**Part 8: Spring Cache Abstraction — Complete Deep Dive**

Isme hum cover karenge:

- Caching fundamentals
- Cache hit and cache miss
- Cache-aside pattern
- Read-through cache
- Write-through cache
- Write-behind cache
- Cache penetration
- Cache stampede
- Cache avalanche
- Spring Cache abstraction
- `@EnableCaching`
- CacheManager
- Redis cache integration
- Interview questions
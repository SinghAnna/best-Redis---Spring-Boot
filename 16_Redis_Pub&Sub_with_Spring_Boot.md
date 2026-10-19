# Part 16: Redis Pub/Sub with Spring Boot — Complete Deep Dive

> Redis Pub/Sub lightweight real-time messaging ke liye hai: notifications, chat updates, live dashboard events aur cache invalidation signals ke liye. Lekin ye durable message queue nahi hai.

---

## 1. Pub/Sub Kya Hai?

**Pub/Sub** ka full form hai:

```text
Publish / Subscribe
```

### Basic Model

```text
Publisher
   |
   | PUBLISH channel message
   v
Redis Channel
   |
   +--> Subscriber 1
   |
   +--> Subscriber 2
   |
   +--> Subscriber 3
```

Publisher message ko channel par bhejta hai, aur us channel ko subscribe karne wale saare active clients ko message deliver hota hai. [232]

---

## 2. Important Pub/Sub Terms

| Term | Meaning |
|---|---|
| Publisher | Message bhejne wala client |
| Subscriber | Message receive karne wala client |
| Channel | Message ka logical topic |
| Pattern subscription | Pattern-based channel subscription |
| Fire-and-forget | Message bhejne ke baad delivery tracking nahi |
| At-most-once | Message ek baar deliver ho sakta hai, ya miss ho sakta hai |

---

## 3. Redis Pub/Sub Delivery Guarantee

Redis Pub/Sub **at-most-once** delivery deta hai.

```text
Message published
    |
    +--> Active subscriber -> Message received
    |
    +--> Offline subscriber -> Message lost forever
```

Redis documentation ke according subscriber offline hone par published message permanently miss ho jata hai; Redis message ko dobara send nahi karta. [229][232]

### Important

```text
Redis Pub/Sub me:
- Message persistence nahi hoti
- Acknowledgement nahi hota
- Replay nahi hota
- Retry nahi hota
```

---

## 4. Pub/Sub vs Kafka

| Feature | Redis Pub/Sub | Kafka |
|---|---|---|
| Delivery | At-most-once | Configurable, usually at-least-once |
| Persistence | No | Yes |
| Replay | No | Yes |
| Acknowledgement | No | Yes |
| Consumer groups | No | Yes |
| Offline consumer | Message lost | Message retained |
| Best for | Real-time UI updates | Critical business events |

Redis Pub/Sub me message queue, storage ya history nahi hoti; guaranteed delivery, replay ya offline consumer support ke liye Redis Streams better choice hai. [236][242]

---

## 5. Redis CLI Pub/Sub

### Terminal 1 — Subscriber

```bash
SUBSCRIBE notifications
```

### Terminal 2 — Publisher

```bash
PUBLISH notifications "User 101 liked your post"
```

### Subscriber Output

```text
1) "message"
2) "notifications"
3) "User 101 liked your post"
```

### Pattern Subscription

```bash
PSUBSCRIBE user.*
```

```bash
PUBLISH user.created "User 101 registered"
PUBLISH user.updated "User 101 profile updated"
```

---

## 6. Spring Boot Pub/Sub Architecture

```text
Spring Boot Service A
        |
        | convertAndSend()
        v
Redis Channel
        |
        +--> Spring Boot Service B
        |
        +--> Spring Boot Service C
        |
        +--> Spring Boot Service D
```

Spring Data Redis me publisher `RedisTemplate.convertAndSend()` use karta hai, aur subscriber ke liye `RedisMessageListenerContainer` use hota hai. [230][234]

---

## 7. Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

---

## 8. Pub/Sub Configuration

```java
package com.example.pubsub.config;

import com.example.pubsub.listener.NotificationMessageListener;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.listener.ChannelTopic;
import org.springframework.data.redis.listener.RedisMessageListenerContainer;
import org.springframework.data.redis.listener.adapter.MessageListenerAdapter;

@Configuration
public class RedisPubSubConfig {

    public static final String NOTIFICATION_CHANNEL =
            "notification-service:notifications";

    @Bean
    public ChannelTopic notificationChannel() {
        return new ChannelTopic(NOTIFICATION_CHANNEL);
    }

    @Bean
    public MessageListenerAdapter notificationListenerAdapter(
            NotificationMessageListener listener
    ) {
        return new MessageListenerAdapter(listener, "handleMessage");
    }

    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer(
            RedisConnectionFactory connectionFactory,
            MessageListenerAdapter notificationListenerAdapter,
            ChannelTopic notificationChannel
    ) {
        RedisMessageListenerContainer container =
                new RedisMessageListenerContainer();

        container.setConnectionFactory(connectionFactory);
        container.addMessageListener(
                notificationListenerAdapter,
                notificationChannel
        );

        return container;
    }
}
```

`RedisMessageListenerContainer` subscriptions manage karta hai, messages asynchronously receive karta hai aur listener instances ko dispatch karta hai. [231][235]

---

## 9. Message DTO

```java
package com.example.pubsub.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class NotificationMessage {

    private Long userId;
    private String type;
    private String title;
    private String message;
    private LocalDateTime createdAt;
}
```

---

## 10. Publisher

```java
package com.example.pubsub.publisher;

import com.example.pubsub.config.RedisPubSubConfig;
import com.example.pubsub.dto.NotificationMessage;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class NotificationPublisher {

    private final StringRedisTemplate redisTemplate;
    private final ObjectMapper objectMapper;

    public void publish(NotificationMessage message) {
        try {
            String json = objectMapper.writeValueAsString(message);

            redisTemplate.convertAndSend(
                    RedisPubSubConfig.NOTIFICATION_CHANNEL,
                    json
            );

            log.info(
                    "Published notification for userId={}",
                    message.getUserId()
            );

        } catch (JsonProcessingException e) {
            throw new IllegalStateException(
                    "Failed to serialize notification message", e
            );
        }
    }
}
```

`convertAndSend()` given message ko configured topic/channel par publish karta hai. [234][237]

---

## 11. Subscriber

```java
package com.example.pubsub.listener;

import com.example.pubsub.dto.NotificationMessage;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class NotificationMessageListener {

    private final ObjectMapper objectMapper;

    public void handleMessage(String message) {
        try {
            NotificationMessage notification =
                    objectMapper.readValue(message, NotificationMessage.class);

            log.info(
                    "Received notification: userId={}, type={}, title={}",
                    notification.getUserId(),
                    notification.getType(),
                    notification.getTitle()
            );

            // Send push notification
            // Save notification in DB
            // Update websocket clients

        } catch (Exception e) {
            log.error("Failed to process notification message: {}", message, e);
        }
    }
}
```

`MessageListenerAdapter` normal POJO ko message listener bana sakta hai — matlab listener ko `MessageListener` interface implement karna zaroori nahi. [231][235]

---

## 12. Controller to Publish Notification

```java
package com.example.pubsub.controller;

import com.example.pubsub.dto.NotificationMessage;
import com.example.pubsub.publisher.NotificationPublisher;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDateTime;

@RestController
@RequestMapping("/api/notifications")
@RequiredArgsConstructor
public class NotificationController {

    private final NotificationPublisher notificationPublisher;

    @PostMapping
    public ResponseEntity<String> sendNotification(
            @RequestBody NotificationMessage request
    ) {
        NotificationMessage message = NotificationMessage.builder()
                .userId(request.getUserId())
                .type(request.getType())
                .title(request.getTitle())
                .message(request.getMessage())
                .createdAt(LocalDateTime.now())
                .build();

        notificationPublisher.publish(message);

        return ResponseEntity.ok("Notification published");
    }
}
```

---

## 13. Test Pub/Sub

### Start Application

```bash
./mvnw spring-boot:run
```

### Publish Notification

```bash
curl -X POST http://localhost:8080/api/notifications \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 101,
    "type": "LIKE",
    "title": "New Like",
    "message": "Rahul liked your post"
  }'
```

### Application Log

```text
Received notification: userId=101, type=LIKE, title=New Like
```

### Redis CLI Verification

```bash
docker exec -it redis-local redis-cli

SUBSCRIBE notification-service:notifications
```

Dusre terminal se:

```bash
PUBLISH notification-service:notifications '{"userId":101}'
```

---

## 14. Pattern Subscription Example

```java
@Bean
public RedisMessageListenerContainer redisMessageListenerContainer(
        RedisConnectionFactory connectionFactory,
        MessageListenerAdapter notificationListenerAdapter
) {
    RedisMessageListenerContainer container =
            new RedisMessageListenerContainer();

    container.setConnectionFactory(connectionFactory);

    container.addMessageListener(
            notificationListenerAdapter,
            new PatternTopic("notification-service:*")
    );

    return container;
}
```

### Matching Channels

```text
notification-service:notifications
notification-service:alerts
notification-service:system
```

---

## 15. JSON Serialization Best Practice

Pub/Sub me message ko JSON string bhejna better hota hai.

### Why?

```text
Multiple services alag languages use kar sakti hain:
Java, Node.js, Python, Go
```

### Recommended Message Format

```json
{
  "eventId": "d3b07384-d9a0-4c62-8f30-2b6f8a1f1d9a",
  "eventType": "POST_LIKED",
  "userId": 101,
  "postId": 201,
  "occurredAt": "2026-10-19T14:20:00Z"
}
```

---

## 16. Pub/Sub Use Cases

| Use Case | Example |
|---|---|
| Real-time notifications | “Your post got a new like” |
| Chat messages | Live chat room updates |
| Dashboard updates | Live order count |
| Cache invalidation | `user:101:profile` invalidate |
| Presence updates | User online/offline |
| Live score updates | Cricket / game score |
| WebSocket fan-out | Browser clients ko update |

---

## 17. Pub/Sub Limitations

| Limitation | Impact |
|---|---|
| No persistence | Redis restart par messages gone |
| No replay | Purana message retrieve nahi hota |
| No ACK | Subscriber ne process kiya ya nahi, pata nahi |
| Offline subscriber miss | Message permanently lost |
| No consumer groups | Load balancing manually manage karna padta hai |
| No retry | Processing failure par message lost |

Redis Pub/Sub at-most-once delivery deta hai aur message persistence, replay, acknowledgement ya retry provide nahi karta. [232][233]

---

## 18. When to Use Pub/Sub vs Streams vs Kafka

| Requirement | Best Choice |
|---|---|
| Live UI notification | Redis Pub/Sub |
| Chat message broadcast | Redis Pub/Sub |
| Cache invalidation signal | Redis Pub/Sub / Kafka |
| Order processing | Kafka / Redis Streams |
| Payment event | Kafka |
| Audit log | Kafka |
| Task queue with retry | Redis Streams / Kafka |
| Event replay | Kafka |
| Consumer groups | Redis Streams / Kafka |

---

## 19. Production Best Practices

- [ ] Message JSON format me bhejo.
- [ ] Message me `eventId` aur `eventType` rakho.
- [ ] Channel naming convention follow karo.
- [ ] Subscriber me exception handling karo.
- [ ] Poison message ko log karo.
- [ ] Redis connection failure handle karo.
- [ ] Critical events ke liye Kafka use karo.
- [ ] Pub/Sub ko durable queue mat samjho.
- [ ] Metrics add karo: published messages, received messages, failures.

---

## 20. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Pub/Sub ko durable queue samajhna | Message loss | Kafka/Streams |
| Subscriber offline hone ki expectation | Message miss | Streams/Kafka |
| No exception handling | Listener crash / silent failure | Try-catch + logging |
| Java serialization | Cross-service issue | JSON |
| No message ID | Duplicate/debug difficult | UUID event ID |
| Business-critical workflow Pub/Sub par | Data loss risk | Kafka |
| No monitoring | Silent message loss | Metrics + alerts |

---

## 21. Interview Questions

| Question | Answer |
|---|---|
| Redis Pub/Sub kya hai? | Publisher channel par message publish karta hai aur active subscribers receive karte hain. |
| Pub/Sub delivery guarantee kya hai? | At-most-once. |
| Offline subscriber message kaise handle karta hai? | Miss kar deta hai; message permanently lost hota hai. |
| Pub/Sub me persistence hoti hai? | Nahi. |
| Pub/Sub me acknowledgement hota hai? | Nahi. |
| Redis Pub/Sub vs Kafka? | Pub/Sub real-time fire-and-forget hai; Kafka durable, replayable event streaming hai. |
| Spring Data Redis me publisher kaise likhte hain? | `RedisTemplate.convertAndSend(channel, message)` se. |
| Spring Data Redis me subscriber kaise likhte hain? | `RedisMessageListenerContainer` aur `MessageListenerAdapter` se. |
| Pub/Sub kab avoid karna chahiye? | Jab message loss acceptable na ho. |

---

## 22. Final Mental Model

```text
Redis Pub/Sub = Live broadcast

Publisher -> Channel -> Active Subscribers

No storage
No replay
No ACK
No retry
No offline delivery

Critical events -> Kafka
Reliable task queue -> Redis Streams / Kafka
Live UI updates -> Redis Pub/Sub
```

---

## Next Part

**Part 17: Redis Streams with Spring Boot — Complete Deep Dive**

Isme hum cover karenge:

- Redis Streams kya hai
- Stream vs Pub/Sub
- Stream vs Kafka
- `XADD`
- `XREAD`
- Consumer groups
- `XREADGROUP`
- `XACK`
- Pending entries
- Message retry
- Dead letter handling
- Spring Boot Stream listener
- Event-driven microservices
- Interview questions
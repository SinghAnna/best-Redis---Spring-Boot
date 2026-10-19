# Part 17: Redis Streams with Spring Boot — Complete Deep Dive

> Redis Streams events ko append-only log me store karte hain. Pub/Sub fire-and-forget hota hai, jabki Streams me consumer groups, acknowledgement, pending messages aur recovery support hota hai.

---

## 1. Redis Streams Kya Hai?

Redis Stream ek append-only log hai jisme events ko occurrence order me store kiya jata hai.

```text
orders-stream
  ├── 1760000000000-0 -> orderId=1001, status=CREATED
  ├── 1760000001000-0 -> orderId=1002, status=CREATED
  └── 1760000002000-0 -> orderId=1003, status=CREATED
```

Har entry ka unique ID hota hai:

```text
<timestamp-milliseconds>-<sequence-number>
```

Example:

```text
1760000000000-0
```

Redis Stream append-only log jaisa behave karta hai, lekin consumer groups aur acknowledgements jaise operations bhi support karta hai. [254][256]

---

## 2. Stream vs Pub/Sub

| Feature | Redis Pub/Sub | Redis Streams |
|---|---|---|
| Message storage | No | Yes |
| Replay | No | Yes |
| Consumer groups | No | Yes |
| Acknowledgement | No | Yes |
| Pending messages | No | Yes |
| Offline consumer | Message lost | Message retained |
| Delivery semantics | At-most-once | At-least-once possible |
| Best for | Live UI updates | Reliable event processing |

Pub/Sub me offline subscriber message miss kar deta hai; Streams me message stream me rehta hai aur consumer group baad me process kar sakta hai. [229][252]

---

## 3. Stream vs Kafka

| Feature | Redis Streams | Kafka |
|---|---|---|
| Primary role | Lightweight event log | Distributed event streaming platform |
| Retention | Memory/config-based | Long-term disk-based log |
| Replay | Possible | Strong native replay |
| Consumer groups | Yes | Yes |
| Scaling | Redis Cluster | Kafka partitions |
| Ordering | Per stream | Per partition |
| Best for | Small/medium event flows | Large-scale event backbone |

---

## 4. Core Stream Commands

### `XADD`

Stream me event add karta hai.

```bash
XADD orders-stream * orderId 1001 userId 101 amount 499 status CREATED
```

`*` Redis ko automatically unique ID generate karne deta hai.

### `XLEN`

Stream me total entries count karta hai.

```bash
XLEN orders-stream
```

### `XRANGE`

Range se entries read karta hai.

```bash
XRANGE orders-stream - +
```

### `XREAD`

Simple stream read karta hai.

```bash
XREAD COUNT 10 STREAMS orders-stream 0
```

### `XGROUP CREATE`

Consumer group create karta hai.

```bash
XGROUP CREATE orders-stream order-processors 0
```

### `XREADGROUP`

Consumer group se message read karta hai.

```bash
XREADGROUP GROUP order-processors consumer-1 COUNT 10 STREAMS orders-stream >
```

### `XACK`

Message successfully process hone par acknowledge karta hai.

```bash
XACK orders-stream order-processors 1760000000000-0
```

`XREADGROUP` consumer groups support karta hai aur messages ko consumers me distribute karta hai; `XACK` processed message ko acknowledge karta hai. [245][255]

---

## 5. Consumer Group Model

```text
orders-stream
     |
     +--> Consumer Group: order-processors
              |
              +--> consumer-1 -> message 1, 4
              |
              +--> consumer-2 -> message 2, 5
              |
              +--> consumer-3 -> message 3, 6
```

### Important

```text
Ek consumer group ke andar
ek message sirf ek consumer ko milta hai.
```

### Multiple Consumer Groups

```text
orders-stream
     |
     +--> Group: email-service
     |       |
     |       +--> Receives every message
     |
     +--> Group: analytics-service
             |
             +--> Receives every message
```

Alag consumer groups same stream ke messages independently process kar sakte hain.

---

## 6. Pending Entries List — PEL

Jab consumer `XREADGROUP` se message leta hai, wo message **Pending Entries List** me add ho jata hai.

```text
Consumer reads message
        |
        v
Message enters PEL
        |
        +--> XACK -> Message removed from PEL
        |
        +--> No XACK -> Message remains pending
```

Consumer crash, hang ya deployment ke dauran message process karke `XACK` na kare to message PEL me stuck reh jata hai. [249][253]

---

## 7. Message Recovery — `XAUTOCLAIM`

Agar consumer fail ho jaye, to dusra consumer pending message claim kar sakta hai.

```bash
XAUTOCLAIM orders-stream order-processors consumer-2 60000 0-0 COUNT 10
```

Meaning:

```text
60,000 ms = 60 seconds se zyada idle pending messages
consumer-2 claim karega.
```

`XAUTOCLAIM` idle pending messages ko dusre consumer ko transfer karta hai. [249][253]

### Recovery Flow

```text
consumer-1 crash
        |
        v
Message remains in PEL
        |
        v
consumer-2 runs XAUTOCLAIM
        |
        v
Message ownership transferred
        |
        v
consumer-2 processes and ACKs
```

---

## 8. Retry and Dead Letter Queue

Har message ko infinite retry mat karo.

### Recommended Flow

```text
Message processing fail
        |
        +--> Retry 1
        |
        +--> Retry 2
        |
        +--> Retry 3
        |
        +--> Dead Letter Queue
```

### DLQ Stream

```text
orders-stream-dlq
```

### DLQ Message

```text
orderId -> 1001
originalId -> 1760000000000-0
failureReason -> Payment service unavailable
deliveryCount -> 3
failedAt -> 2026-10-19T14:30:00Z
```

Repeatedly failing messages ko delivery count track karke maximum retry threshold ke baad DLQ me route karna recommended pattern hai. [249][251]

---

## 9. Spring Boot Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

---

## 10. Stream Event DTO

```java
package com.example.stream.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderCreatedEvent {

    private String eventId;
    private Long orderId;
    private Long userId;
    private Double amount;
    private String status;
}
```

---

## 11. Stream Producer

```java
package com.example.stream.producer;

import com.example.stream.dto.OrderCreatedEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.connection.stream.RecordId;
import org.springframework.data.redis.connection.stream.StreamRecords;
import org.springframework.data.redis.connection.stream.StringRecord;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.Map;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderStreamProducer {

    private final StringRedisTemplate redisTemplate;
    private final ObjectMapper objectMapper;

    private static final String STREAM_KEY =
            "order-service:orders";

    public RecordId publishOrderCreated(OrderCreatedEvent event) {
        try {
            Map<String, String> fields = Map.of(
                    "eventId", event.getEventId(),
                    "orderId", event.getOrderId().toString(),
                    "userId", event.getUserId().toString(),
                    "amount", event.getAmount().toString(),
                    "status", event.getStatus(),
                    "payload", objectMapper.writeValueAsString(event)
            );

            StringRecord record = StreamRecords.newRecord()
                    .in(STREAM_KEY)
                    .ofStrings(fields);

            RecordId recordId = redisTemplate.opsForStream().add(record);

            log.info(
                    "Published order event. orderId={}, recordId={}",
                    event.getOrderId(),
                    recordId
            );

            return recordId;

        } catch (Exception e) {
            throw new IllegalStateException(
                    "Failed to publish order event", e
            );
        }
    }
}
```

---

## 12. Stream Consumer Configuration

```java
package com.example.stream.config;

import com.example.stream.listener.OrderCreatedListener;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.stream.Consumer;
import org.springframework.data.redis.connection.stream.MapReadOptions;
import org.springframework.data.redis.connection.stream.ReadOffset;
import org.springframework.data.redis.connection.stream.StreamOffset;
import org.springframework.data.redis.stream.StreamMessageListenerContainer;
import org.springframework.data.redis.stream.Subscription;

import java.time.Duration;

@Configuration
public class RedisStreamConfig {

    public static final String ORDER_STREAM =
            "order-service:orders";

    public static final String CONSUMER_GROUP =
            "order-processors";

    public static final String CONSUMER_NAME =
            "order-service-consumer-1";

    @Bean
    public StreamMessageListenerContainer<String, MapRecord<String, String, String>>
            streamMessageListenerContainer(
                    RedisConnectionFactory connectionFactory,
                    OrderCreatedListener listener
    ) {
        StreamMessageListenerContainer.StreamMessageListenerContainerOptions<String, MapRecord<String, String, String>> options =
                StreamMessageListenerContainer.StreamMessageListenerContainerOptions.builder()
                        .pollTimeout(Duration.ofSeconds(2))
                        .batchSize(10)
                        .errorHandler(error -> {
                            // Log and handle processing failure
                        })
                        .build();

        StreamMessageListenerContainer<String, MapRecord<String, String, String>> container =
                StreamMessageListenerContainer.create(connectionFactory, options);

        container.receive(
                Consumer.from(CONSUMER_GROUP, CONSUMER_NAME),
                StreamOffset.create(ORDER_STREAM, ReadOffset.lastConsumed()),
                listener
        );

        container.start();

        return container;
    }
}
```

Spring Data Redis `StreamMessageListenerContainer` stream subscribe karta hai, background polling karta hai aur incoming messages listener ko deliver karta hai. [244][246]

---

## 13. Stream Listener

```java
package com.example.stream.listener;

import com.example.stream.dto.OrderCreatedEvent;
import com.example.stream.service.OrderProcessingService;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.connection.stream.MapRecord;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.stream.StreamListener;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderCreatedListener
        implements StreamListener<String, MapRecord<String, String, String>> {

    private final OrderProcessingService orderProcessingService;
    private final StringRedisTemplate redisTemplate;
    private final ObjectMapper objectMapper;

    @Override
    public void onMessage(MapRecord<String, String, String> message) {
        try {
            String payload = message.getValue().get("payload");

            OrderCreatedEvent event =
                    objectMapper.readValue(payload, OrderCreatedEvent.class);

            orderProcessingService.processOrder(event);

            redisTemplate.opsForStream().acknowledge(
                    "order-service:orders",
                    "order-processors",
                    message.getId()
            );

            log.info(
                    "Processed and acknowledged order event. orderId={}",
                    event.getOrderId()
            );

        } catch (Exception e) {
            log.error(
                    "Failed to process stream message. id={}",
                    message.getId(),
                    e
            );

            // Do not acknowledge.
            // Message remains pending and can be reclaimed.
        }
    }
}
```

---

## 14. Consumer Group Initialization

Application startup par consumer group create karna important hai.

```java
package com.example.stream.config;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.ApplicationRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.stream.ReadOffset;
import org.springframework.data.redis.connection.stream.StreamOffset;
import org.springframework.data.redis.core.StreamOperations;
import org.springframework.data.redis.core.StringRedisTemplate;

@Slf4j
@Configuration
@RequiredArgsConstructor
public class RedisStreamGroupConfig {

    private final StringRedisTemplate redisTemplate;

    @Bean
    public ApplicationRunner createConsumerGroup() {
        return args -> {
            try {
                StreamOperations<String, Object, Object> streamOperations =
                        redisTemplate.opsForStream();

                streamOperations.createGroup(
                        "order-service:orders",
                        ReadOffset.from("0"),
                        "order-processors"
                );

                log.info("Created consumer group: order-processors");

            } catch (Exception e) {
                log.info(
                        "Consumer group already exists: {}",
                        e.getMessage()
                );
            }
        };
    }
}
```

---

## 15. Pending Message Recovery

```java
package com.example.stream.recovery;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.connection.stream.PendingMessages;
import org.springframework.data.redis.connection.stream.PendingMessagesSummary;
import org.springframework.data.redis.core.StreamOperations;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class PendingMessageRecoveryJob {

    private final StringRedisTemplate redisTemplate;

    private static final String STREAM_KEY =
            "order-service:orders";

    private static final String GROUP_NAME =
            "order-processors";

    @Scheduled(fixedDelay = 60000)
    public void recoverPendingMessages() {
        StreamOperations<String, Object, Object> streamOperations =
                redisTemplate.opsForStream();

        PendingMessagesSummary summary =
                streamOperations.pending(STREAM_KEY, GROUP_NAME);

        if (summary == null || summary.getTotalPendingMessages() == 0) {
            return;
        }

        log.info(
                "Pending messages found. total={}",
                summary.getTotalPendingMessages()
        );

        PendingMessages pendingMessages =
                streamOperations.pending(
                        STREAM_KEY,
                        GROUP_NAME,
                        org.springframework.data.domain.Range.unbounded(),
                        100L
                );

        pendingMessages.forEach(pendingMessage -> {
            log.warn(
                    "Pending message detected. id={}, consumer={}, elapsed={}",
                    pendingMessage.getId(),
                    pendingMessage.getConsumerName(),
                    pendingMessage.getElapsedTimeSinceLastDelivery()
            );

            // Claim and reprocess using XAUTOCLAIM / XCLAIM
        });
    }
}
```

Pending messages ko periodically inspect karna aur idle messages ko reclaim karna reliable stream processing ka important part hai. [249][254]

---

## 16. Stream Message Flow

### Happy Path

```text
Producer
   |
   | XADD
   v
Redis Stream
   |
   v
Consumer Group
   |
   v
Consumer reads message
   |
   v
Business processing
   |
   v
XACK
   |
   v
Message removed from PEL
```

### Failure Path

```text
Producer
   |
   | XADD
   v
Redis Stream
   |
   v
Consumer reads message
   |
   v
Processing fails
   |
   v
No XACK
   |
   v
Message remains in PEL
   |
   v
Another consumer claims it
   |
   v
Retry or DLQ
```

---

## 17. Stream Use Cases

| Use Case | Example |
|---|---|
| Order processing | Order created -> payment -> inventory |
| Notification pipeline | Post liked -> notification service |
| Audit logging | User action -> audit service |
| Activity tracking | User viewed post -> analytics |
| Task queue | Email sending, image processing |
| IoT events | Sensor readings |
| Event-driven microservices | Service-to-service events |

---

## 18. Stream Best Practices

- [ ] Event me unique `eventId` rakho.
- [ ] Payload JSON me serialize karo.
- [ ] Consumer group use karo.
- [ ] Message process hone ke baad `XACK` karo.
- [ ] Processing failure par ACK mat karo.
- [ ] Pending messages recovery job rakho.
- [ ] Retry limit define karo.
- [ ] DLQ implement karo.
- [ ] Stream length trim karo.
- [ ] Redis Streams ko long-term durable log mat samjho.

---

## 19. Stream Trimming

Stream unlimited grow ho sakti hai, isliye trimming important hai.

### Approximate Trimming

```bash
XTRIM orders-stream MAXLEN ~ 100000
```

### Exact Trimming

```bash
XTRIM orders-stream MAXLEN 100000
```

### Spring Boot Example

```java
redisTemplate.opsForStream().trim(
        "order-service:orders",
        100000,
        true
);
```

---

## 20. Common Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| Processing ke baad `XACK` na karna | Message pending reh jayega | Success par ACK |
| Failure par `XACK` karna | Message loss | Pending/DLQ flow |
| No consumer group | Load balancing difficult | Consumer group use karo |
| Infinite retry | Poison message loop | Retry limit + DLQ |
| Stream trim na karna | Memory growth | `XTRIM` |
| Streams ko Kafka replacement samajhna | Long retention/replay limitations | Requirement ke hisaab se choose karo |
| No idempotency | Duplicate processing | Event ID + idempotency check |

---

## 21. Interview Questions

| Question | Answer |
|---|---|
| Redis Stream kya hai? | Append-only event log jo consumer groups aur acknowledgements support karta hai. |
| Pub/Sub aur Stream me difference? | Pub/Sub fire-and-forget hai; Stream persistent, consumer-group-based aur ACK-supported hai. |
| `XADD` kya karta hai? | Stream me new event add karta hai. |
| `XREADGROUP` kya karta hai? | Consumer group ke through messages read/distribute karta hai. |
| `XACK` kya karta hai? | Successfully processed message ko acknowledge karta hai. |
| PEL kya hai? | Pending Entries List — messages jo read hue but ACK nahi hue. |
| `XAUTOCLAIM` kya karta hai? | Idle pending messages ko dusre consumer ko transfer karta hai. |
| DLQ kya hai? | Dead Letter Queue, jahan repeatedly failing messages route hote hain. |
| Stream aur Kafka me difference? | Streams lightweight Redis event log hain; Kafka large-scale durable event streaming platform hai. |

---

## 22. Final Mental Model

```text
XADD          -> Produce event
XREADGROUP    -> Consume event
XACK          -> Mark processed
PEL           -> Unacknowledged messages
XAUTOCLAIM    -> Recover stuck messages
DLQ           -> Handle poison messages
XTRIM         -> Control stream size
```

---

## Next Part

**Part 18: Redis Persistence — RDB, AOF and Hybrid Persistence — Complete Deep Dive**

Isme hum cover karenge:

- Persistence kya hai
- RDB snapshots
- AOF log
- RDB vs AOF
- `appendfsync` options
- Hybrid persistence
- Data loss scenarios
- Backup strategy
- Redis restart recovery
- Production persistence configuration
- Interview questions
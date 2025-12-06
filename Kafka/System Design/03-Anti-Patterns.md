# Anti-Patterns - What NOT to Do with Kafka

Learn from common mistakes that cause production outages, failed interviews, and architectural disasters.

---

## Table of Contents
1. [The "10 Million Consumers" Problem](#the-10-million-consumers-problem)
2. [Wrong Tool for the Job](#wrong-tool-for-the-job)
3. [Partition Key Mistakes](#partition-key-mistakes)
4. [Consumer Group Misconfigurations](#consumer-group-misconfigurations)
5. [Over-Engineering](#over-engineering)
6. [Under-Engineering](#under-engineering)
7. [Configuration Mistakes](#configuration-mistakes)

---

## The "10 Million Consumers" Problem

### The LinkedIn Post Scenario

**Interview Question:** "Design a live stream for 10 million users."

**Bad Answer:** "I'll use Kafka. The user devices will be the consumers."

**Why This Fails Immediately:**

This is the most common anti-pattern that shows fundamental misunderstanding of Kafka's architecture.

### The Math Behind the Failure

#### Scenario 1: One Consumer Group

```
Setup:
- Topic: live-stream
- Partitions: 1,000
- Users: 10,000,000
- Consumer Group: "viewers"
- Each user device = 1 consumer in the group

Kafka Rule: One partition → One consumer per group

Result:
- Active consumers: 1,000 (one per partition)
- Idle consumers: 9,999,000
- Users watching: 1,000 ✅
- Users seeing black screen: 9,999,000 ❌
```

**Why:**
- Kafka assigns each partition to exactly one consumer in a group
- Extra consumers sit idle
- 99.99% of users can't consume any data

#### Scenario 2: 10 Million Consumer Groups

```
Setup:
- Topic: live-stream
- Partitions: 1,000
- Consumer Groups: 10,000,000 (one per user)
- Each user = their own consumer group

Kafka Overhead per Consumer Group:
- Metadata: ~1 KB
- Heartbeat: Every 3 seconds
- Rebalancing coordination
- Offset commits

Total Overhead:
- Metadata: 10M × 1 KB = 10 GB
- Heartbeats: 3.3M requests/second
- Network: Saturated
- Coordinator: Overwhelmed

Result: CLUSTER CRASH 💥
```

**What Actually Happens:**

1. **Metadata Explosion**
   ```
   Each consumer group maintains:
   - Group membership
   - Partition assignments
   - Offset commits
   - Rebalancing state
   
   10M groups × 1 KB = 10 GB of metadata
   Coordinator can't handle this
   ```

2. **Heartbeat Storm**
   ```
   Each consumer sends heartbeat every 3 seconds
   10M consumers / 3 seconds = 3.3M heartbeats/second
   Network saturated
   Brokers overwhelmed
   ```

3. **Rebalancing Nightmare**
   ```
   Any consumer join/leave triggers rebalancing
   With 10M consumers, rebalancing never completes
   System in perpetual rebalancing state
   ```

4. **Security Nightmare**
   ```
   10M public devices directly connected to internal Kafka cluster
   No authentication/authorization at scale
   DDoS attack surface
   ```

### The Correct Architecture

**Live streaming is NOT a Kafka use case. Use CDN + WebSockets.**

```
Correct Architecture:

┌─────────────┐
│   Kafka     │ ← Internal event stream
│   Cluster   │    (player positions, game state)
└──────┬──────┘
       │
       ↓
┌─────────────┐
│  Stream     │ ← Aggregates/processes events
│  Processor  │    Generates video stream
└──────┬──────┘
       │
       ↓
┌─────────────┐
│    CDN      │ ← Distributes video
│  (Cloudflare│    Edge caching
│   Akamai)   │    Global distribution
└──────┬──────┘
       │
       ↓
┌─────────────┐
│  WebSocket  │ ← Real-time updates
│   Gateway   │    (scores, notifications)
└──────┬──────┘
       │
       ↓
┌─────────────┐
│ 10M Users   │ ← Consumers
└─────────────┘

Kafka consumers: ~10-100 (stream processors)
User-facing: CDN + WebSocket servers
```

**Why This Works:**

1. **Kafka handles internal events**
   - Player actions
   - Game state changes
   - Analytics events
   - Consumers: Stream processing applications (10-100)

2. **CDN handles video distribution**
   - Edge caching
   - Global distribution
   - Scales to billions of users
   - Optimized for video delivery

3. **WebSocket handles real-time updates**
   - Scores, notifications
   - Chat messages
   - Stateful connections
   - Horizontal scaling with load balancers

### Interview Red Flags

**Immediate Fail Indicators:**

❌ "Each user is a Kafka consumer"
❌ "One consumer group per user"
❌ "Kafka will stream video to browsers"
❌ "We'll use Kafka for request-response"
❌ "Kafka handles WebSocket connections"

**Correct Thinking:**

✅ "Kafka for internal event processing"
✅ "CDN for content distribution"
✅ "WebSocket for real-time updates"
✅ "Kafka consumers are backend services"
✅ "Scale consumers to match partitions"

---

## Wrong Tool for the Job

### When NOT to Use Kafka

#### 1. Request-Response Patterns

**Anti-Pattern:**
```java
// Using Kafka for API calls
public Order getOrder(String orderId) {
    // Send request to Kafka
    producer.send(new ProducerRecord<>("order-requests", orderId));
    
    // Wait for response from Kafka
    ConsumerRecords<String, Order> records = consumer.poll(Duration.ofSeconds(5));
    return records.iterator().next().value(); // ❌ WRONG
}
```

**Why It Fails:**
- Kafka is async, not sync
- High latency (50-100ms minimum)
- Complex correlation logic
- Wasted resources

**Correct Solution:**
```java
// Use REST/gRPC
public Order getOrder(String orderId) {
    return restTemplate.getForObject(
        "http://order-service/orders/" + orderId,
        Order.class
    ); // ✅ CORRECT
}
```

#### 2. Direct Client-to-Backend Streaming

**Anti-Pattern:**
```
Browser → Kafka → Backend
```

**Why It Fails:**
- Security: Exposing Kafka to internet
- Scalability: Kafka not designed for millions of direct connections
- Latency: Not optimized for low-latency streaming
- Protocol: Kafka protocol not browser-friendly

**Correct Solution:**
```
Browser → WebSocket/SSE → Backend → Kafka
```

#### 3. Small-Scale Pub/Sub

**Anti-Pattern:**
```
Use Case: 10 messages/second, 5 subscribers
Solution: Deploy 3-node Kafka cluster
```

**Why It's Overkill:**
- Kafka overhead: ZooKeeper/KRaft, replication, disk I/O
- Operational complexity
- Resource waste
- Cost

**Correct Solution:**
```
Use Redis Pub/Sub or RabbitMQ
- Simpler
- Lower latency
- Less overhead
- Easier operations
```

#### 4. Transactional Database

**Anti-Pattern:**
```java
// Using Kafka as database
public void updateUser(User user) {
    producer.send(new ProducerRecord<>("users", user.getId(), user));
}

public User getUser(String userId) {
    // Read entire topic to find user ❌
    consumer.seekToBeginning(partitions);
    while (true) {
        ConsumerRecords<String, User> records = consumer.poll(Duration.ofMillis(100));
        for (ConsumerRecord<String, User> record : records) {
            if (record.key().equals(userId)) {
                return record.value();
            }
        }
    }
}
```

**Why It Fails:**
- No random access
- No indexes
- No queries
- Retention limits

**Correct Solution:**
```java
// Use actual database
public User getUser(String userId) {
    return userRepository.findById(userId); // ✅
}

// Use Kafka for change events
public void updateUser(User user) {
    userRepository.save(user);
    producer.send(new ProducerRecord<>("user-changes", user.getId(), user));
}
```

#### 5. File Storage

**Anti-Pattern:**
```java
// Storing large files in Kafka
byte[] videoFile = Files.readAllBytes(Paths.get("video.mp4")); // 500 MB
producer.send(new ProducerRecord<>("videos", videoId, videoFile)); // ❌
```

**Why It Fails:**
- Message size limits (default 1 MB)
- Memory pressure
- Network saturation
- Inefficient storage

**Correct Solution:**
```java
// Store in object storage
String s3Url = s3Client.upload("video.mp4", videoFile);
producer.send(new ProducerRecord<>("video-uploaded", 
    videoId, 
    new VideoMetadata(videoId, s3Url)
)); // ✅
```

### Decision Matrix

| Use Case | Kafka | Better Alternative |
|----------|-------|-------------------|
| Event streaming | ✅ | - |
| Log aggregation | ✅ | - |
| CDC | ✅ | - |
| Request-response | ❌ | REST/gRPC |
| Direct client streaming | ❌ | WebSocket/SSE |
| Small pub/sub | ❌ | Redis/RabbitMQ |
| Database queries | ❌ | PostgreSQL/MySQL |
| File storage | ❌ | S3/HDFS |
| Real-time analytics | ✅ | - |
| Message queue | ⚠️ | RabbitMQ (better) |

---

## Partition Key Mistakes

### Anti-Pattern 1: No Partition Key

**Problem:**
```java
// Round-robin distribution
producer.send(new ProducerRecord<>("orders", null, order)); // ❌
```

**Impact:**
- No ordering guarantee
- Related events scattered across partitions
- Can't process related events together
- Difficult to maintain state

**Example Failure:**
```
User places order:
1. Order created → Partition 0
2. Payment processed → Partition 2
3. Order shipped → Partition 1

Consumer can't process in order!
Might see "shipped" before "created"
```

**Correct Solution:**
```java
// Use user ID as key
producer.send(new ProducerRecord<>("orders", order.getUserId(), order)); // ✅
// All orders for same user → same partition → ordered
```

### Anti-Pattern 2: Low-Cardinality Keys

**Problem:**
```java
// Using country as partition key
producer.send(new ProducerRecord<>("events", event.getCountry(), event)); // ❌
```

**Impact:**
```
Partitions: 10
Countries: 5 (US, UK, CA, AU, IN)

Distribution:
Partition 0: US events (60% of traffic) ← HOT PARTITION
Partition 1: UK events (15% of traffic)
Partition 2: CA events (10% of traffic)
Partition 3: AU events (8% of traffic)
Partition 4: IN events (7% of traffic)
Partition 5-9: IDLE

Result:
- Partition 0 overloaded
- Other partitions underutilized
- Poor parallelism
- Slow processing
```

**Correct Solution:**
```java
// Use high-cardinality key
producer.send(new ProducerRecord<>("events", event.getUserId(), event)); // ✅
// Millions of users → even distribution
```

### Anti-Pattern 3: Timestamp as Key

**Problem:**
```java
// Using timestamp as key
producer.send(new ProducerRecord<>("logs", 
    String.valueOf(System.currentTimeMillis()), 
    log)); // ❌
```

**Impact:**
- All messages at same millisecond → same partition
- Bursty traffic → hot partitions
- No ordering benefit (timestamps change)
- Poor distribution

**Correct Solution:**
```java
// Use entity ID as key
producer.send(new ProducerRecord<>("logs", 
    log.getServiceId(), 
    log)); // ✅
```

### Anti-Pattern 4: Composite Key Without Hashing

**Problem:**
```java
// Using concatenated string as key
String key = userId + "-" + sessionId; // ❌
producer.send(new ProducerRecord<>("events", key, event));

// Problem: String length varies, hash distribution poor
```

**Correct Solution:**
```java
// Hash the composite key
String key = userId + "-" + sessionId;
String hashedKey = DigestUtils.md5Hex(key); // ✅
producer.send(new ProducerRecord<>("events", hashedKey, event));
```

### Partition Key Best Practices

**Good Keys:**
- User ID (high cardinality)
- Session ID (high cardinality)
- Device ID (high cardinality)
- Transaction ID (high cardinality)

**Bad Keys:**
- Country (low cardinality)
- Date (low cardinality)
- Boolean flags (very low cardinality)
- Null (no ordering)

**Rule of Thumb:**
```
Cardinality should be >> Number of partitions

Good: 1M users, 10 partitions (100K:1 ratio)
Bad: 5 countries, 10 partitions (0.5:1 ratio)
```

---

## Consumer Group Misconfigurations

### Anti-Pattern 1: Not Setting Group ID

**Problem:**
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
// Missing: group.id
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props); // ❌
```

**Impact:**
- Each consumer gets unique auto-generated group ID
- No coordination between consumers
- All consumers read all partitions
- Duplicate processing

**Correct Solution:**
```java
props.put("group.id", "order-processors"); // ✅
```

### Anti-Pattern 2: Sharing Group ID Across Applications

**Problem:**
```
Application A: group.id = "processors"
Application B: group.id = "processors" // ❌ Same group!

Result:
- Kafka thinks they're same application
- Partitions split between A and B
- Both applications get incomplete data
```

**Correct Solution:**
```
Application A: group.id = "order-processors"
Application B: group.id = "analytics-processors"
```

### Anti-Pattern 3: Too Many Consumers

**Problem:**
```
Partitions: 10
Consumers: 100 // ❌

Result:
- 10 active consumers
- 90 idle consumers
- Wasted resources
- Slow rebalancing
```

**Correct Solution:**
```
Partitions: 10
Consumers: 10 (or fewer)

Or scale up partitions:
Partitions: 100
Consumers: 100
```

### Anti-Pattern 4: Not Handling Rebalancing

**Problem:**
```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processRecord(record); // Takes 10 seconds
    }
    consumer.commitSync(); // ❌ Might miss heartbeat
}
```

**Impact:**
- Processing takes too long
- Consumer misses heartbeat
- Kicked out of group
- Rebalancing triggered
- Perpetual rebalancing

**Correct Solution:**
```java
props.put("max.poll.interval.ms", "300000"); // 5 minutes
props.put("session.timeout.ms", "30000"); // 30 seconds

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
    }
    consumer.commitSync();
}

// Or process in batches
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    List<CompletableFuture<Void>> futures = new ArrayList<>();
    for (ConsumerRecord<String, String> record : records) {
        futures.add(CompletableFuture.runAsync(() -> processRecord(record)));
    }
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    consumer.commitSync();
}
```

### Anti-Pattern 5: Auto-Commit Without Understanding

**Problem:**
```java
props.put("enable.auto.commit", "true"); // ❌ Default
props.put("auto.commit.interval.ms", "5000");

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processRecord(record); // Might fail
    }
    // Offset auto-committed even if processing failed
}
```

**Impact:**
- Offsets committed before processing completes
- If processing fails, message lost
- No at-least-once guarantee

**Correct Solution:**
```java
props.put("enable.auto.commit", "false"); // ✅

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        try {
            processRecord(record);
            consumer.commitSync(); // Commit after successful processing
        } catch (Exception e) {
            // Handle error, don't commit
            log.error("Processing failed", e);
        }
    }
}
```

---

## Over-Engineering

### Anti-Pattern 1: Too Many Topics

**Problem:**
```
Creating separate topic for every entity:

user-created
user-updated
user-deleted
user-login
user-logout
user-password-changed
user-email-verified
... (100+ topics)
```

**Impact:**
- Operational nightmare
- Difficult to monitor
- Complex consumer logic
- Partition explosion

**Correct Solution:**
```
Use fewer topics with event types:

Topic: user-events
Events: UserCreated, UserUpdated, UserDeleted, etc.

Consumer filters by event type:
if (event.getType() == "UserCreated") {
    handleUserCreated(event);
}
```

### Anti-Pattern 2: Premature Partitioning

**Problem:**
```
Day 1: 1000 messages/day
Decision: Create 1000 partitions "for future scale" ❌
```

**Impact:**
- Unnecessary overhead
- Slow rebalancing
- Wasted resources
- Difficult to reduce later

**Correct Solution:**
```
Day 1: Start with 3-10 partitions
Monitor throughput
Scale up when needed
```

### Anti-Pattern 3: Complex Custom Serialization

**Problem:**
```java
// Custom binary format
class CustomSerializer implements Serializer<Order> {
    public byte[] serialize(String topic, Order order) {
        // 500 lines of custom serialization logic ❌
        // Custom compression
        // Custom encryption
        // Custom versioning
    }
}
```

**Impact:**
- Difficult to debug
- Hard to evolve schema
- Compatibility issues
- Maintenance burden

**Correct Solution:**
```java
// Use standard formats
// Avro, Protobuf, or JSON
class Order {
    @JsonProperty("orderId")
    private String orderId;
    // ...
}

// Or Avro with Schema Registry
```

---

## Under-Engineering

### Anti-Pattern 1: Single Partition

**Problem:**
```
Topic: orders
Partitions: 1 ❌
```

**Impact:**
- No parallelism
- Single consumer only
- Bottleneck
- No fault tolerance

**Correct Solution:**
```
Topic: orders
Partitions: 10+ (based on throughput)
```

### Anti-Pattern 2: No Replication

**Problem:**
```properties
replication.factor=1 ❌
```

**Impact:**
- Broker failure = data loss
- No fault tolerance
- Production outage

**Correct Solution:**
```properties
replication.factor=3 ✅
min.insync.replicas=2
```

### Anti-Pattern 3: No Monitoring

**Problem:**
```
Deploy Kafka
Hope it works ❌
```

**Impact:**
- Blind to issues
- Consumer lag unnoticed
- Broker failures undetected
- Performance degradation

**Correct Solution:**
```
Monitor:
- Consumer lag
- Broker disk usage
- Network throughput
- Rebalancing frequency
- Under-replicated partitions

Tools:
- Prometheus + Grafana
- Burrow (consumer lag)
- Kafka Manager
```

### Anti-Pattern 4: Default Configurations

**Problem:**
```properties
# Using all defaults ❌
```

**Impact:**
- Poor performance
- Inadequate durability
- Suboptimal throughput

**Correct Solution:**
```properties
# Producer
acks=all
retries=Integer.MAX_VALUE
max.in.flight.requests.per.connection=5
enable.idempotence=true

# Consumer
enable.auto.commit=false
max.poll.interval.ms=300000

# Broker
num.replica.fetchers=4
num.network.threads=8
num.io.threads=16
```

---

## Configuration Mistakes

### Anti-Pattern 1: Ignoring Message Size Limits

**Problem:**
```java
// Sending 10 MB message
byte[] largePayload = new byte[10 * 1024 * 1024];
producer.send(new ProducerRecord<>("topic", key, largePayload)); // ❌
// Fails: Message too large
```

**Default Limits:**
```properties
message.max.bytes=1048576 (1 MB)
```

**Correct Solution:**
```java
// Option 1: Increase limits (not recommended)
props.put("max.request.size", 10485760); // 10 MB

// Option 2: Store large data elsewhere (recommended)
String s3Url = uploadToS3(largePayload);
producer.send(new ProducerRecord<>("topic", key, s3Url)); // ✅
```

### Anti-Pattern 2: Mismatched Retention

**Problem:**
```properties
# Producer expects 30 days retention
# Broker configured for 7 days
log.retention.hours=168 ❌

# Consumer tries to read old data
# Data already deleted
```

**Correct Solution:**
```properties
# Align retention with requirements
log.retention.hours=720 # 30 days
log.retention.bytes=-1 # Unlimited (time-based only)
```

### Anti-Pattern 3: Inadequate Heap Size

**Problem:**
```bash
# Running Kafka with default heap
java -jar kafka.jar ❌
# Default: Usually too small
```

**Impact:**
- Frequent GC pauses
- Out of memory errors
- Poor performance

**Correct Solution:**
```bash
# Set appropriate heap size
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"
# 6 GB for typical broker
# Adjust based on workload
```

---

## Key Takeaways

### Interview Red Flags

1. ❌ "Each user is a Kafka consumer"
2. ❌ "I'll use Kafka for request-response"
3. ❌ "Kafka will stream video to browsers"
4. ❌ "One consumer group per user"
5. ❌ "Kafka is a database"

### Production Red Flags

1. ❌ Single partition topics
2. ❌ No replication
3. ❌ No monitoring
4. ❌ Default configurations
5. ❌ Low-cardinality partition keys

### Golden Rules

1. ✅ Kafka for internal event streaming, not user-facing
2. ✅ Consumer groups for backend services, not end users
3. ✅ High-cardinality partition keys
4. ✅ Replication factor = 3
5. ✅ Monitor everything

---

**Next:** [Scaling Patterns →](./02-Scaling-Patterns.md)

# Interview Guide - Kafka System Design

Master Kafka interviews with common questions, design scenarios, and expert answers.

---

## Table of Contents
1. [Common Interview Questions](#common-interview-questions)
2. [System Design Scenarios](#system-design-scenarios)
3. [Gotchas and Red Flags](#gotchas-and-red-flags)
4. [How to Approach Kafka Problems](#how-to-approach-kafka-problems)
5. [Sample Solutions](#sample-solutions)

---

## Common Interview Questions

### Conceptual Questions

#### Q1: Explain Kafka architecture in 2 minutes

**Good Answer:**
```
Kafka is a distributed event streaming platform with three main components:

1. Brokers: Servers that store and serve data
   - Organized into clusters
   - Handle reads/writes
   - Manage replication

2. Topics & Partitions:
   - Topics are logical channels
   - Partitions enable parallelism
   - Messages ordered within partition
   - Distributed across brokers

3. Producers & Consumers:
   - Producers write to topics
   - Consumers read from topics
   - Consumer groups enable scaling
   - One partition per consumer in a group

Key features:
- High throughput (millions of messages/sec)
- Durability (replication)
- Scalability (horizontal)
- Fault tolerance (automatic failover)
```

**Red Flags:**
- ❌ "Kafka is a message queue"
- ❌ "Kafka is a database"
- ❌ Not mentioning partitions
- ❌ Not mentioning consumer groups

---

#### Q2: What's the difference between Kafka and RabbitMQ?

**Good Answer:**
```
Architecture:
- Kafka: Distributed log, pull-based
- RabbitMQ: Message broker, push-based

Use Cases:
- Kafka: Event streaming, high throughput, replay
- RabbitMQ: Task queues, routing, low latency

Performance:
- Kafka: Millions of messages/sec
- RabbitMQ: Tens of thousands/sec

Message Retention:
- Kafka: Time-based (days/weeks)
- RabbitMQ: Until consumed

Ordering:
- Kafka: Per-partition ordering
- RabbitMQ: Per-queue ordering

Consumers:
- Kafka: Multiple independent consumer groups
- RabbitMQ: Competing consumers

Choose Kafka for:
- Event sourcing
- Log aggregation
- Stream processing
- High throughput

Choose RabbitMQ for:
- Task distribution
- Complex routing
- Request-response patterns
- Lower latency requirements
```

---

#### Q3: How does Kafka achieve high throughput?

**Good Answer:**
```
1. Sequential I/O:
   - Append-only log
   - Sequential disk writes (100 MB/s+)
   - Faster than random I/O

2. Zero-Copy:
   - Data transferred from disk to network
   - Bypasses application memory
   - Reduces CPU usage

3. Batching:
   - Producers batch messages
   - Reduces network overhead
   - Amortizes disk writes

4. Compression:
   - LZ4, Snappy, GZIP
   - Reduces network bandwidth
   - Reduces disk usage

5. Partitioning:
   - Parallel reads/writes
   - Scales horizontally
   - Load distribution

6. Page Cache:
   - OS caches frequently accessed data
   - Reduces disk I/O
   - Automatic memory management
```

---

#### Q4: Explain consumer groups and rebalancing

**Good Answer:**
```
Consumer Groups:
- Logical grouping of consumers
- Each partition consumed by one consumer in group
- Enables parallel processing
- Multiple groups can read same topic

Example:
Topic: orders (10 partitions)
Group A: 5 consumers → Each gets 2 partitions
Group B: 10 consumers → Each gets 1 partition
Group C: 3 consumers → 3 get 3-4 partitions, others get 3

Rebalancing:
- Triggered when:
  * Consumer joins/leaves
  * Consumer crashes
  * Partitions added
  
- Types:
  * Eager: Stop-the-world (all consumers stop)
  * Cooperative: Incremental (minimal disruption)

- Impact:
  * Brief processing pause
  * Partition reassignment
  * Offset commits

Best Practices:
- Use cooperative rebalancing (Kafka 2.4+)
- Implement graceful shutdown
- Monitor rebalancing frequency
- Tune session.timeout.ms and max.poll.interval.ms
```

---

#### Q5: What are the delivery guarantees in Kafka?

**Good Answer:**
```
Three levels:

1. At-Most-Once (may lose messages):
   Producer: acks=0 or acks=1, no retries
   Consumer: Commit before processing
   Use case: Metrics, logs (non-critical)

2. At-Least-Once (may duplicate):
   Producer: acks=all, retries enabled
   Consumer: Commit after processing
   Use case: Most production systems
   Handle: Idempotent processing

3. Exactly-Once (no loss, no duplicates):
   Producer: enable.idempotence=true, transactions
   Consumer: isolation.level=read_committed
   Use case: Financial transactions, payments
   Trade-off: ~20-30% throughput reduction

Implementation:
// Exactly-once
props.put("enable.idempotence", "true");
props.put("transactional.id", "tx-1");

producer.initTransactions();
producer.beginTransaction();
producer.send(record1);
producer.send(record2);
producer.commitTransaction();
```

---

### Design Questions

#### Q6: How would you partition a topic for a social media feed?

**Good Answer:**
```
Requirements Analysis:
- Need ordering per user's feed
- High volume (millions of posts/sec)
- Multiple consumers (timeline, notifications, analytics)

Partition Key Options:

Option 1: User ID (poster)
Key: posterId
Pro: All posts by user in same partition
Con: Celebrity users create hot partitions

Option 2: User ID (viewer) - WRONG
Key: viewerId
Problem: Each post viewed by many users
Result: Massive duplication

Option 3: Post ID - WRONG
Key: postId
Problem: No ordering guarantee
Result: Can't maintain timeline order

Recommended: User ID with salting for celebrities
```java
String key;
if (isHighVolumeUser(posterId)) {
    // Split celebrity posts across partitions
    int salt = hash(postId) % 10;
    key = posterId + "-" + salt;
} else {
    key = posterId;
}
```

Trade-offs:
- Regular users: Perfect ordering
- Celebrities: Distributed load, ordering within salt
- Consumers: Must aggregate for celebrity feeds

Partitions: Start with 50, scale to 500 based on throughput
Replication: 3 (for durability)
Retention: 7 days
```

---

#### Q7: Design a real-time analytics system using Kafka

**Good Answer:**
```
Architecture:

1. Data Ingestion:
   Producers → Kafka Topic: raw-events
   - Web servers, mobile apps, IoT devices
   - High throughput (100K events/sec)
   - Partitions: 50 (based on throughput)

2. Stream Processing:
   Kafka Streams / Flink
   - Windowed aggregations (1 min, 5 min, 1 hour)
   - Filtering, enrichment
   - Output → Kafka Topic: aggregated-metrics

3. Storage:
   Consumer → Time-series DB (InfluxDB, TimescaleDB)
   - For long-term storage
   - For complex queries

4. Real-time Dashboard:
   Consumer → WebSocket → Browser
   - Low latency updates
   - Live metrics

Data Flow:
Events → raw-events topic (50 partitions)
  ↓
Kafka Streams (50 instances)
  ↓
aggregated-metrics topic (10 partitions)
  ↓
├→ TimescaleDB (historical)
└→ WebSocket (real-time)

Key Decisions:
- Partition by: user_id or session_id
- Retention: 1 day (raw), 30 days (aggregated)
- Replication: 3
- Consumer groups: 
  * stream-processors (50 consumers)
  * db-writers (10 consumers)
  * websocket-publishers (5 consumers)

Scaling:
- Add partitions for more throughput
- Add stream processors for more parallelism
- Use Kafka Streams for stateful processing
```

---

#### Q8: How would you handle a failed consumer?

**Good Answer:**
```
Failure Scenarios & Solutions:

1. Consumer Crash:
   Detection: Heartbeat timeout (session.timeout.ms)
   Action: Automatic rebalancing
   Recovery: 
   - New consumer takes over partitions
   - Resumes from last committed offset
   - May reprocess some messages (at-least-once)

2. Processing Failure:
   Strategy: Dead Letter Queue (DLQ)
```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, String> record : records) {
        try {
            processRecord(record);
        } catch (RecoverableException e) {
            // Retry with exponential backoff
            retry(record, 3);
        } catch (UnrecoverableException e) {
            // Send to DLQ
            dlqProducer.send(new ProducerRecord<>("dlq", record.key(), record.value()));
            log.error("Sent to DLQ", e);
        }
    }
    
    consumer.commitSync();
}
```

3. Slow Consumer:
   Detection: Growing lag
   Solutions:
   - Scale up consumers (up to partition count)
   - Optimize processing logic
   - Increase max.poll.interval.ms
   - Use async processing

4. Poison Pill Message:
   Problem: One bad message blocks partition
   Solution:
```java
int consecutiveFailures = 0;
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, String> record : records) {
        try {
            processRecord(record);
            consecutiveFailures = 0;
        } catch (Exception e) {
            consecutiveFailures++;
            if (consecutiveFailures > 3) {
                // Skip poison pill
                dlqProducer.send(new ProducerRecord<>("dlq", record));
                consecutiveFailures = 0;
            }
        }
    }
    consumer.commitSync();
}
```

Monitoring:
- Consumer lag per partition
- Processing time (p99)
- Error rate
- DLQ size
```

---

## System Design Scenarios

### Scenario 1: Design Uber's Trip Events System

**Requirements:**
- 10M trips/day
- Real-time driver location updates (every 5 seconds)
- Multiple consumers (billing, analytics, notifications)
- 99.9% durability

**Solution:**

```
1. Topics:
   - trip-events: Trip lifecycle (created, started, completed)
   - location-updates: Driver locations
   - payment-events: Payment processing

2. Partitioning:
   trip-events: Partition by trip_id
   - Ensures ordering per trip
   - Partitions: 50 (200K trips/day per partition)
   
   location-updates: Partition by driver_id
   - Ensures ordering per driver
   - Partitions: 100 (high volume)
   
   payment-events: Partition by user_id
   - Ensures ordering per user
   - Partitions: 20

3. Consumer Groups:
   - billing-service: Reads trip-events, payment-events
   - analytics-service: Reads all topics
   - notification-service: Reads trip-events
   - driver-app: Reads location-updates (filtered by region)

4. Configuration:
   Producers:
   - acks=all (durability)
   - enable.idempotence=true
   - compression.type=lz4
   
   Brokers:
   - replication.factor=3
   - min.insync.replicas=2
   - retention: 7 days (trip-events), 1 day (location-updates)

5. Scaling:
   Current: 10M trips/day = 115 trips/sec
   Peak: 5x = 575 trips/sec
   Provision for: 10x = 1150 trips/sec
   
   Location updates: 1M drivers × 12 updates/min = 200K updates/sec
   Throughput: ~50 MB/sec
   
   Cluster: 10 brokers (5 MB/sec each)

6. Monitoring:
   - Consumer lag < 1000 messages
   - p99 latency < 100ms
   - Under-replicated partitions = 0
```

---

### Scenario 2: Design a Fraud Detection System

**Requirements:**
- Process 100K transactions/sec
- Detect fraud in < 100ms
- No false negatives (catch all fraud)
- Minimize false positives

**Solution:**

```
Architecture:

1. Ingestion:
   transactions topic (100 partitions)
   - Partition by: user_id
   - Ensures all user transactions in order
   - Enables stateful fraud detection

2. Stream Processing (Kafka Streams):
   - Window: 5 minutes
   - State: User transaction history
   - Rules:
     * Velocity checks (> 10 transactions/min)
     * Amount checks (> $10K)
     * Location checks (impossible travel)
     * Pattern matching (known fraud patterns)

3. Output:
   fraud-alerts topic (10 partitions)
   - High-risk transactions
   - Sent to manual review

   approved-transactions topic (100 partitions)
   - Low-risk transactions
   - Proceed to payment

4. Implementation:
```java
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Transaction> transactions = 
    builder.stream("transactions");

// Stateful processing
KTable<String, UserState> userState = transactions
    .groupByKey()
    .aggregate(
        UserState::new,
        (key, transaction, state) -> state.update(transaction),
        Materialized.as("user-state-store")
    );

// Fraud detection
KStream<String, Transaction> fraudulent = transactions
    .join(userState, (transaction, state) -> {
        return fraudDetector.check(transaction, state);
    })
    .filter((key, result) -> result.isFraudulent());

fraudulent.to("fraud-alerts");
```

5. Scaling:
   - 100K transactions/sec
   - 100 partitions = 1K transactions/sec per partition
   - 100 stream processors (one per partition)
   - Each processor maintains state for ~10K users

6. Latency Optimization:
   - RocksDB for state store (local SSD)
   - Batch processing (100 messages)
   - Async I/O for external calls
   - Target: p99 < 50ms

7. Exactly-Once Semantics:
   - Critical for fraud detection
   - enable.idempotence=true
   - Transactional processing
   - No duplicate fraud alerts
```

---

## Gotchas and Red Flags

### Interview Red Flags

**Immediate Fail:**
1. ❌ "Each user device is a Kafka consumer"
2. ❌ "I'll use Kafka for request-response"
3. ❌ "Kafka will stream video to browsers"
4. ❌ "One consumer group per user"
5. ❌ "Kafka is a database, I'll query it"

**Warning Signs:**
1. ⚠️ Not asking about scale (messages/sec, users)
2. ⚠️ Not discussing partition key strategy
3. ⚠️ Ignoring consumer lag
4. ⚠️ Not mentioning replication
5. ⚠️ Using default configurations

**Good Signals:**
1. ✅ Asks clarifying questions about scale
2. ✅ Discusses trade-offs (throughput vs latency)
3. ✅ Mentions monitoring and alerting
4. ✅ Considers failure scenarios
5. ✅ Talks about operational aspects

---

## How to Approach Kafka Problems

### Step-by-Step Framework

**1. Clarify Requirements (5 minutes)**
```
Ask:
- What's the scale? (messages/sec, data size)
- What's the latency requirement?
- What's the durability requirement?
- How many consumers?
- What's the retention period?
- What's the growth projection?
```

**2. Design Data Model (5 minutes)**
```
Decide:
- Topic structure (one vs multiple topics)
- Partition key (critical for ordering and distribution)
- Number of partitions (based on throughput)
- Message format (Avro, Protobuf, JSON)
```

**3. Design Architecture (10 minutes)**
```
Components:
- Producers (where, how many)
- Kafka cluster (brokers, partitions, replication)
- Consumers (groups, scaling strategy)
- Stream processing (if needed)
- Storage (if needed)

Draw diagram showing data flow
```

**4. Discuss Configuration (5 minutes)**
```
Producer:
- acks (0, 1, all)
- retries
- compression
- batching

Broker:
- replication.factor
- min.insync.replicas
- retention

Consumer:
- group.id
- enable.auto.commit
- max.poll.interval.ms
```

**5. Address Failure Scenarios (5 minutes)**
```
What if:
- Broker fails?
- Consumer crashes?
- Network partition?
- Disk full?
- Consumer lag grows?

Solutions for each
```

**6. Discuss Monitoring (3 minutes)**
```
Metrics:
- Consumer lag
- Throughput
- Latency
- Error rate
- Under-replicated partitions
```

**7. Discuss Scaling (2 minutes)**
```
How to scale:
- Add partitions
- Add consumers
- Add brokers
- Optimize configuration
```

---

## Sample Solutions

### Problem: Design a Log Aggregation System

**Requirements:**
- 1000 servers
- 10 GB/day per server
- Retain for 30 days
- Multiple consumers (search, analytics, alerting)

**Solution:**

```
1. Architecture:
   Servers → Filebeat → Kafka → Consumers

2. Topics:
   logs-app (application logs)
   logs-system (system logs)
   logs-security (security logs)

3. Partitioning:
   Partition by: server_id
   Partitions: 100 (10 servers per partition)
   Ensures logs from same server ordered

4. Throughput:
   Total: 10 TB/day = 115 MB/sec
   Per partition: 1.15 MB/sec
   Easily handled by Kafka

5. Storage:
   30 days × 10 TB = 300 TB
   Replication factor: 3
   Total: 900 TB
   Cluster: 10 brokers × 100 TB = 1 PB capacity

6. Consumers:
   - Elasticsearch: Full-text search
   - S3: Long-term storage
   - Alerting: Real-time monitoring

7. Configuration:
   Producer:
   - acks=1 (logs can tolerate some loss)
   - compression.type=gzip (high compression ratio)
   - batch.size=1MB
   
   Broker:
   - replication.factor=3
   - retention.ms=2592000000 (30 days)
   
   Consumer:
   - Elasticsearch: 10 consumers
   - S3: 5 consumers (batch writes)
   - Alerting: 20 consumers (low latency)
```

---

## Key Takeaways

### Must-Know Concepts
1. Partitions = parallelism
2. Consumer groups enable scaling
3. Replication provides durability
4. Partition key determines ordering
5. Kafka is not a database/queue/CDN

### Must-Ask Questions
1. What's the scale?
2. What's the latency requirement?
3. What's the durability requirement?
4. How many consumers?
5. What's the growth projection?

### Must-Discuss Topics
1. Partition strategy
2. Consumer group design
3. Failure handling
4. Monitoring approach
5. Scaling strategy

### Must-Avoid Mistakes
1. One consumer group per user
2. Using Kafka for request-response
3. Ignoring partition key design
4. Not discussing replication
5. Using default configurations

---

**Next:** [Performance Tuning →](./05-Performance-Tuning.md)

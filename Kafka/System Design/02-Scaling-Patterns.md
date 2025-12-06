# Scaling Patterns - Growing Kafka to Production Scale

Real-world patterns for scaling Kafka from prototype to billions of messages per day.

---

## Table of Contents
1. [Horizontal Scaling Strategies](#horizontal-scaling-strategies)
2. [Partition Key Design for Scale](#partition-key-design-for-scale)
3. [Consumer Group Scaling](#consumer-group-scaling)
4. [Multi-Datacenter Architectures](#multi-datacenter-architectures)
5. [Handling Hot Partitions](#handling-hot-partitions)
6. [Scaling Brokers](#scaling-brokers)

---

## Horizontal Scaling Strategies

### Scaling Producers

**Pattern 1: Partition-Aware Producers**

```java
// Scale producers by adding more instances
// Each producer can write to all partitions

Producer Instance 1 ─┐
Producer Instance 2 ─┼─→ Topic (10 partitions)
Producer Instance 3 ─┘

// No coordination needed
// Kafka handles distribution via partition key
```

**Configuration for High Throughput:**

```properties
# Batch messages for better throughput
batch.size=32768  # 32 KB (default: 16 KB)
linger.ms=10      # Wait 10ms to batch messages

# Compress messages
compression.type=lz4  # or snappy, gzip, zstd

# Buffer memory
buffer.memory=67108864  # 64 MB

# Multiple in-flight requests
max.in.flight.requests.per.connection=5

# Enable idempotence for safety
enable.idempotence=true
```

**Scaling Calculation:**

```
Single producer throughput: 50 MB/s
Target throughput: 500 MB/s
Producers needed: 500 / 50 = 10 producers

With batching and compression:
Single producer: 100 MB/s
Producers needed: 500 / 100 = 5 producers
```

### Scaling Consumers

**Pattern 1: Scale to Partition Count**

```
Topic: orders (20 partitions)

Scenario 1: 5 consumers
Consumer 1: Partitions 0-3   (4 partitions)
Consumer 2: Partitions 4-7   (4 partitions)
Consumer 3: Partitions 8-11  (4 partitions)
Consumer 4: Partitions 12-15 (4 partitions)
Consumer 5: Partitions 16-19 (4 partitions)

Scenario 2: 20 consumers (optimal)
Each consumer: 1 partition

Scenario 3: 40 consumers (waste)
20 active, 20 idle
```

**Pattern 2: Multi-Threaded Consumer**

```java
// Single consumer, multiple processing threads
public class MultiThreadedConsumer {
    private final KafkaConsumer<String, String> consumer;
    private final ExecutorService executor;
    
    public MultiThreadedConsumer(int threadCount) {
        this.consumer = new KafkaConsumer<>(props);
        this.executor = Executors.newFixedThreadPool(threadCount);
    }
    
    public void consume() {
        consumer.subscribe(Arrays.asList("orders"));
        
        while (true) {
            ConsumerRecords<String, String> records = 
                consumer.poll(Duration.ofMillis(100));
            
            // Process each partition's records in parallel
            for (TopicPartition partition : records.partitions()) {
                List<ConsumerRecord<String, String>> partitionRecords = 
                    records.records(partition);
                
                executor.submit(() -> {
                    for (ConsumerRecord<String, String> record : partitionRecords) {
                        processRecord(record);
                    }
                });
            }
            
            // Wait for all threads to complete before committing
            executor.awaitTermination(5, TimeUnit.SECONDS);
            consumer.commitSync();
        }
    }
}
```

**Pattern 3: Dynamic Consumer Scaling**

```java
// Kubernetes HPA (Horizontal Pod Autoscaler)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kafka-consumer
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kafka-consumer
  minReplicas: 3
  maxReplicas: 20  # Match partition count
  metrics:
  - type: External
    external:
      metric:
        name: kafka_consumer_lag
      target:
        type: Value
        value: "1000"  # Scale when lag > 1000
```

### Scaling Partitions

**When to Add Partitions:**

```
Current state:
- Partitions: 10
- Consumers: 10 (maxed out)
- Lag: Growing
- Throughput: Insufficient

Action: Increase partitions to 20

Result:
- Can add 10 more consumers
- Double the parallelism
- Reduce lag
```

**How to Add Partitions:**

```bash
# Add partitions to existing topic
kafka-topics --bootstrap-server localhost:9092 \
  --alter --topic orders \
  --partitions 20

# WARNING: Cannot decrease partition count
# Plan carefully before increasing
```

**Impact of Adding Partitions:**

```
Before (10 partitions):
Key "user123" → hash % 10 = Partition 3

After (20 partitions):
Key "user123" → hash % 20 = Partition 13

⚠️ Same key now goes to different partition!
⚠️ Ordering broken for existing keys
⚠️ Stateful consumers must rebuild state
```

**Safe Partition Scaling Strategy:**

```
1. Create new topic with more partitions
2. Dual-write to both topics temporarily
3. Migrate consumers to new topic
4. Deprecate old topic
5. Delete old topic after retention period
```

---

## Partition Key Design for Scale

### Pattern 1: Composite Keys

**Problem:** Need to partition by user but maintain ordering within user sessions.

```java
// Bad: Only user ID
String key = userId;  // All user events together

// Good: User ID + Session ID
String key = userId + "-" + sessionId;

// Better: Hash for even distribution
String key = DigestUtils.md5Hex(userId + "-" + sessionId);

producer.send(new ProducerRecord<>("events", key, event));
```

**Benefits:**
- Finer-grained partitioning
- Better distribution
- Maintains ordering within session
- Scales with concurrent sessions

### Pattern 2: Time-Based Bucketing

**Problem:** Time-series data with hot keys.

```java
// Bad: Only sensor ID
String key = sensorId;  // Hot sensors overload partitions

// Good: Sensor ID + Time bucket
long timeBucket = timestamp / (5 * 60 * 1000);  // 5-minute buckets
String key = sensorId + "-" + timeBucket;

producer.send(new ProducerRecord<>("sensor-data", key, reading));
```

**Benefits:**
- Distributes hot keys over time
- Enables time-based processing
- Prevents partition hotspots

### Pattern 3: Geographic Sharding

**Problem:** Global application with regional hotspots.

```java
// Bad: Only user ID
String key = userId;

// Good: Region + User ID
String region = getRegion(userId);  // US-EAST, EU-WEST, etc.
String key = region + "-" + userId;

producer.send(new ProducerRecord<>("user-events", key, event));
```

**Benefits:**
- Regional isolation
- Easier compliance (GDPR, data residency)
- Reduced cross-region traffic

### Pattern 4: Salting for Hot Keys

**Problem:** Celebrity users create hot partitions.

```java
// Bad: User ID only
String key = userId;  // Celebrity → hot partition

// Good: Add random salt for hot users
String key;
if (isHighVolumeUser(userId)) {
    int salt = random.nextInt(10);  // 10-way split
    key = userId + "-" + salt;
} else {
    key = userId;
}

producer.send(new ProducerRecord<>("user-events", key, event));
```

**Trade-offs:**
- ✅ Distributes hot keys
- ❌ Loses ordering for hot users
- ⚠️ Consumers must aggregate across salts

---

## Consumer Group Scaling

### Pattern 1: Independent Consumer Groups

```
Topic: user-events (10 partitions)

Consumer Group: analytics
├── Consumer 1: Partitions 0-4
└── Consumer 2: Partitions 5-9

Consumer Group: notifications
├── Consumer 1: Partitions 0-2
├── Consumer 2: Partitions 3-5
├── Consumer 3: Partitions 6-7
└── Consumer 4: Partitions 8-9

Consumer Group: audit
└── Consumer 1: Partitions 0-9

Each group processes independently
No interference between groups
```

### Pattern 2: Staged Processing

```
Stage 1: Fast Path (Low Latency)
Consumer Group: fast-processors
- 10 consumers
- Simple processing
- Low latency

Stage 2: Slow Path (Heavy Processing)
Consumer Group: slow-processors
- 5 consumers
- Complex processing
- Higher latency acceptable

Both read same topic, different groups
Fast path for real-time, slow path for analytics
```

### Pattern 3: Priority Consumers

```java
// High-priority consumer group
Properties highPriorityProps = new Properties();
highPriorityProps.put("group.id", "high-priority");
highPriorityProps.put("max.poll.records", 100);

// Low-priority consumer group
Properties lowPriorityProps = new Properties();
lowPriorityProps.put("group.id", "low-priority");
lowPriorityProps.put("max.poll.records", 500);

// High-priority processes smaller batches faster
// Low-priority processes larger batches for throughput
```

### Pattern 4: Canary Consumers

```
Production Consumer Group: order-processors
├── Consumer 1-9: Stable version
└── Consumer 10: Canary (new version)

Canary gets 10% of partitions
Monitor for errors
Gradual rollout if successful
```

---

## Multi-Datacenter Architectures

### Pattern 1: Active-Active (MirrorMaker 2)

```
Datacenter US-EAST:
┌─────────────────┐
│ Kafka Cluster 1 │
│ Topic: orders   │
└────────┬────────┘
         │
         ↓ MirrorMaker 2
         ↓
┌─────────────────┐
│ Kafka Cluster 2 │ ← Datacenter US-WEST
│ Topic: orders   │
└─────────────────┘

Both clusters accept writes
MirrorMaker 2 replicates bidirectionally
Conflict resolution at application level
```

**Configuration:**

```properties
# MirrorMaker 2 configuration
clusters = us-east, us-west
us-east.bootstrap.servers = east-broker:9092
us-west.bootstrap.servers = west-broker:9092

us-east->us-west.enabled = true
us-west->us-east.enabled = true

# Replication flow
us-east->us-west.topics = orders, users
us-west->us-east.topics = orders, users

# Prevent replication loops
replication.policy.separator = .
```

**Use Cases:**
- Global applications
- Disaster recovery
- Regional data residency

### Pattern 2: Active-Passive (Disaster Recovery)

```
Primary Datacenter:
┌─────────────────┐
│ Kafka Cluster 1 │ ← All writes
│ (Active)        │
└────────┬────────┘
         │
         ↓ MirrorMaker 2 (one-way)
         ↓
┌─────────────────┐
│ Kafka Cluster 2 │ ← Standby
│ (Passive)       │
└─────────────────┘

Passive cluster for disaster recovery
Failover manually or automatically
RPO: Minutes to hours
RTO: Minutes
```

### Pattern 3: Aggregation (Hub and Spoke)

```
Regional Clusters:
┌─────────┐  ┌─────────┐  ┌─────────┐
│ US-EAST │  │ US-WEST │  │ EU-WEST │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
                  ↓
         ┌────────────────┐
         │ Central Cluster│ ← Analytics
         │ (Aggregated)   │
         └────────────────┘

Regional clusters for local processing
Central cluster for global analytics
```

### Pattern 4: Stretch Clusters

```
Single Kafka cluster across multiple AZs:

Availability Zone 1:
├── Broker 1 (Leader for P0)
├── Broker 2 (Follower for P1)
└── Broker 3 (Follower for P2)

Availability Zone 2:
├── Broker 4 (Follower for P0)
├── Broker 5 (Leader for P1)
└── Broker 6 (Follower for P2)

Availability Zone 3:
├── Broker 7 (Follower for P0)
├── Broker 8 (Follower for P1)
└── Broker 9 (Leader for P2)

Benefits:
- Single cluster management
- Automatic failover
- Synchronous replication

Challenges:
- Network latency between AZs
- Requires low-latency network
```

---

## Handling Hot Partitions

### Problem: Identifying Hot Partitions

```bash
# Monitor partition sizes
kafka-log-dirs --bootstrap-server localhost:9092 \
  --describe --topic-list orders

# Monitor consumer lag per partition
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-processors \
  --describe

# Output shows uneven distribution:
PARTITION  LAG
0          100
1          150
2          50000  ← HOT PARTITION
3          120
```

### Solution 1: Repartition with Better Key

```java
// Before: Low-cardinality key
String key = order.getCountry();  // US gets 60% of traffic

// After: High-cardinality key
String key = order.getUserId();  // Millions of users
```

### Solution 2: Split Hot Keys

```java
// Identify hot keys
if (isHotKey(key)) {
    // Split across multiple partitions
    int salt = hash(key + timestamp) % 10;
    key = key + "-" + salt;
}

producer.send(new ProducerRecord<>("orders", key, order));

// Consumer must aggregate
Map<String, List<Order>> ordersByUser = new HashMap<>();
for (ConsumerRecord<String, Order> record : records) {
    String originalKey = extractOriginalKey(record.key());
    ordersByUser.computeIfAbsent(originalKey, k -> new ArrayList<>())
                .add(record.value());
}
```

### Solution 3: Dedicated Topic for Hot Keys

```java
// Route hot keys to separate topic
if (isHotKey(key)) {
    producer.send(new ProducerRecord<>("hot-orders", key, order));
} else {
    producer.send(new ProducerRecord<>("orders", key, order));
}

// Separate consumer group for hot topic
// More resources allocated to hot-orders
```

### Solution 4: Custom Partitioner

```java
public class HotKeyAwarePartitioner implements Partitioner {
    private final Set<String> hotKeys = loadHotKeys();
    
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        int numPartitions = cluster.partitionCountForTopic(topic);
        String keyStr = (String) key;
        
        if (hotKeys.contains(keyStr)) {
            // Distribute hot keys across all partitions
            return Math.abs(keyStr.hashCode() + System.nanoTime()) % numPartitions;
        } else {
            // Normal partitioning for regular keys
            return Math.abs(keyStr.hashCode()) % numPartitions;
        }
    }
}
```

---

## Scaling Brokers

### When to Add Brokers

**Indicators:**
1. **Disk Usage > 70%**
   ```bash
   df -h /data/kafka
   # Filesystem      Size  Used Avail Use%
   # /dev/sda1       1.0T  750G  250G  75%  ← Add brokers
   ```

2. **CPU Usage > 80%**
   ```bash
   top
   # %Cpu(s): 85.0 us  ← Add brokers
   ```

3. **Network Saturation**
   ```bash
   iftop
   # 950 Mb/s out of 1 Gb/s  ← Add brokers
   ```

4. **Partition Count > 4000 per broker**
   ```bash
   kafka-topics --bootstrap-server localhost:9092 --describe
   # Total partitions: 5000 across 1 broker  ← Add brokers
   ```

### How to Add Brokers

**Step 1: Add New Broker**

```bash
# Start new broker with unique broker.id
broker.id=4
log.dirs=/data/kafka

# Broker joins cluster automatically
```

**Step 2: Reassign Partitions**

```json
// reassignment.json
{
  "version": 1,
  "partitions": [
    {"topic": "orders", "partition": 0, "replicas": [1,2,4]},
    {"topic": "orders", "partition": 1, "replicas": [2,3,4]},
    {"topic": "orders", "partition": 2, "replicas": [3,4,1]}
  ]
}
```

```bash
# Execute reassignment
kafka-reassign-partitions --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute

# Monitor progress
kafka-reassign-partitions --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --verify
```

**Step 3: Rebalance Leaders**

```bash
# Trigger leader election to balance load
kafka-leader-election --bootstrap-server localhost:9092 \
  --election-type preferred \
  --all-topic-partitions
```

### Scaling Broker Resources

**Vertical Scaling:**

```
Small Broker:
- 8 CPU cores
- 32 GB RAM
- 1 TB SSD
- 1 Gbps network
- Handles: 100 MB/s, 1000 partitions

Medium Broker:
- 16 CPU cores
- 64 GB RAM
- 4 TB SSD
- 10 Gbps network
- Handles: 500 MB/s, 4000 partitions

Large Broker:
- 32 CPU cores
- 128 GB RAM
- 10 TB SSD
- 25 Gbps network
- Handles: 1 GB/s, 10000 partitions
```

**Horizontal Scaling:**

```
Cluster Growth:

Phase 1: 3 brokers
- Throughput: 300 MB/s
- Partitions: 3000

Phase 2: 10 brokers
- Throughput: 1 GB/s
- Partitions: 10000

Phase 3: 50 brokers
- Throughput: 5 GB/s
- Partitions: 50000

Phase 4: 100+ brokers
- Throughput: 10+ GB/s
- Partitions: 100000+
```

---

## Scaling Best Practices

### 1. Start Small, Scale Up

```
Day 1:
- 3 brokers
- 10 partitions per topic
- 3 consumers per group

Monitor and scale based on actual load
```

### 2. Plan for 3x Growth

```
Current: 100 MB/s
Plan for: 300 MB/s
Provision: 400 MB/s (buffer)
```

### 3. Monitor Key Metrics

```
- Consumer lag (per partition)
- Broker CPU/disk/network
- Partition count per broker
- Under-replicated partitions
- Request latency (p99)
```

### 4. Automate Scaling

```yaml
# Kubernetes example
apiVersion: v1
kind: Service
metadata:
  name: kafka-consumer
spec:
  replicas: 10  # Matches partition count
  
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kafka-consumer-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kafka-consumer
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: kafka_consumer_lag
      target:
        type: AverageValue
        averageValue: "1000"
```

### 5. Test at Scale

```
Load testing:
1. Generate 10x expected load
2. Measure latency, throughput
3. Identify bottlenecks
4. Scale accordingly
5. Repeat
```

---

**Next:** [Production Issues →](./04-Production-Issues.md)

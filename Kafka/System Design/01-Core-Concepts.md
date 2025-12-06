# Core Concepts - Apache Kafka

Understanding these fundamentals is critical for designing scalable Kafka systems and acing interviews.

---

## Table of Contents
1. [Topics, Partitions, and Offsets](#topics-partitions-and-offsets)
2. [Consumer Groups](#consumer-groups)
3. [Replication and ISR](#replication-and-isr)
4. [Brokers and Controllers](#brokers-and-controllers)
5. [ZooKeeper vs KRaft](#zookeeper-vs-kraft)
6. [Message Delivery Semantics](#message-delivery-semantics)

---

## Topics, Partitions, and Offsets

### Topics
A **topic** is a logical channel or category for messages. Think of it as a table in a database or a folder in a file system.

```
Topic: user-events
├── Partition 0
├── Partition 1
└── Partition 2
```

**Key Points:**
- Topics are multi-subscriber (many consumers can read)
- Topics are append-only logs
- Messages are retained based on time or size limits
- Topics can have multiple partitions for parallelism

### Partitions

**Partitions are the unit of parallelism in Kafka.**

```
Topic: orders (3 partitions)

Partition 0: [msg0] [msg3] [msg6] [msg9]  → offset: 0,1,2,3
Partition 1: [msg1] [msg4] [msg7] [msg10] → offset: 0,1,2,3
Partition 2: [msg2] [msg5] [msg8] [msg11] → offset: 0,1,2,3
```

**Critical Characteristics:**
1. **Ordering**: Messages within a partition are strictly ordered
2. **Parallelism**: Each partition can be consumed independently
3. **Distribution**: Partitions are distributed across brokers
4. **Immutability**: Messages cannot be modified once written

### Offsets

An **offset** is a unique sequential ID assigned to each message within a partition.

```
Partition 0:
Offset: 0    1    2    3    4    5    6
Data:  [A]  [B]  [C]  [D]  [E]  [F]  [G]
        ↑                        ↑
    Earliest                 Latest
```

**Offset Management:**
- Consumers track their position using offsets
- Offsets are committed to Kafka (or external store)
- Allows consumers to resume from last position
- Enables replay by resetting offsets

### Partition Assignment Strategy

**How messages get assigned to partitions:**

```java
// 1. With Key (Deterministic)
producer.send(new ProducerRecord<>("orders", userId, orderData));
// partition = hash(userId) % numPartitions
// Same userId always goes to same partition → Ordering guaranteed

// 2. Without Key (Round-robin)
producer.send(new ProducerRecord<>("logs", null, logData));
// partition = round-robin across partitions
// No ordering guarantee

// 3. Custom Partitioner
class CustomPartitioner implements Partitioner {
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        // Your custom logic
        if (isPremiumUser(key)) {
            return 0; // Premium users to partition 0
        }
        return hash(key) % (numPartitions - 1) + 1;
    }
}
```

### Choosing Number of Partitions

**Rule of thumb:**
```
Partitions = max(
    target_throughput / producer_throughput_per_partition,
    target_throughput / consumer_throughput_per_partition
)
```

**Example:**
- Target: 1 GB/s
- Producer throughput: 50 MB/s per partition
- Consumer throughput: 100 MB/s per partition
- Partitions needed: max(1000/50, 1000/100) = max(20, 10) = **20 partitions**

**Trade-offs:**

| More Partitions | Fewer Partitions |
|----------------|------------------|
| ✅ Higher parallelism | ✅ Lower overhead |
| ✅ More consumers possible | ✅ Faster rebalancing |
| ✅ Better throughput | ✅ Simpler management |
| ❌ More memory overhead | ❌ Limited parallelism |
| ❌ Slower rebalancing | ❌ Lower throughput |
| ❌ More file handles | ❌ Fewer consumers possible |

**Production Guidelines:**
- Start with 3-10 partitions per topic
- Don't exceed 4000 partitions per broker
- Don't exceed 200,000 partitions per cluster
- Plan for 2-3x growth

---

## Consumer Groups

**Consumer groups enable scalable message consumption.**

### How Consumer Groups Work

```
Topic: orders (4 partitions)

Consumer Group: order-processors
┌─────────────────────────────────────┐
│ Consumer 1 → Partition 0, 1         │
│ Consumer 2 → Partition 2, 3         │
└─────────────────────────────────────┘

Consumer Group: analytics
┌─────────────────────────────────────┐
│ Consumer 1 → Partition 0, 1, 2, 3   │
└─────────────────────────────────────┘
```

**Key Rules:**
1. Each partition is consumed by **exactly one consumer** within a group
2. Multiple groups can consume the same topic independently
3. If consumers > partitions, some consumers are idle
4. If partitions > consumers, some consumers handle multiple partitions

### Consumer Group Scenarios

#### Scenario 1: Perfect Balance
```
Partitions: 4
Consumers: 4
Result: Each consumer gets 1 partition ✅
```

#### Scenario 2: Under-provisioned
```
Partitions: 4
Consumers: 2
Result: Each consumer gets 2 partitions
Impact: Higher load per consumer, but works fine ✅
```

#### Scenario 3: Over-provisioned
```
Partitions: 4
Consumers: 10
Result: 4 consumers active, 6 consumers IDLE ⚠️
Impact: Wasted resources
```

#### Scenario 4: The LinkedIn Post Problem
```
Partitions: 1,000
Consumers: 10,000,000 (each user device)
Consumer Groups: 10,000,000 (one per user)
Result: CLUSTER CRASH 💥
```

**Why it crashes:**
- Each consumer group maintains metadata
- 10M groups = massive metadata overhead
- Coordinator overwhelmed
- Rebalancing becomes impossible
- Network saturated with heartbeats

### Rebalancing

**Rebalancing** redistributes partitions when consumers join/leave the group.

```
Initial State:
Consumer 1: [P0, P1]
Consumer 2: [P2, P3]

Consumer 3 joins → Rebalancing triggered

New State:
Consumer 1: [P0]
Consumer 2: [P1]
Consumer 3: [P2, P3]
```

**Rebalancing Protocols:**

1. **Eager Rebalancing** (Stop-the-world)
   - All consumers stop consuming
   - Partitions reassigned
   - Consumers resume
   - Downtime: 1-10 seconds typical

2. **Cooperative Rebalancing** (Incremental)
   - Only affected partitions reassigned
   - Other consumers keep running
   - Minimal disruption
   - Available in Kafka 2.4+

**Rebalancing Triggers:**
- Consumer joins the group
- Consumer leaves (graceful shutdown)
- Consumer crashes (heartbeat timeout)
- Consumer takes too long to process (max.poll.interval.ms exceeded)
- Partition count changes

**Production Best Practices:**
```properties
# Increase session timeout for slow consumers
session.timeout.ms=30000  # 30 seconds

# Increase max poll interval for heavy processing
max.poll.interval.ms=300000  # 5 minutes

# Enable cooperative rebalancing
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor

# Graceful shutdown
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    consumer.close(Duration.ofSeconds(10));
}));
```

### Consumer Lag

**Consumer lag** = Latest offset - Consumer's current offset

```
Partition 0:
Latest offset: 1000
Consumer offset: 950
Lag: 50 messages

Partition 1:
Latest offset: 2000
Consumer offset: 1800
Lag: 200 messages

Total lag: 250 messages
```

**Monitoring Lag:**
```bash
# Using kafka-consumer-groups
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-processors --describe

# Output:
GROUP           TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
order-processors orders  0          950             1000            50
order-processors orders  1          1800            2000            200
```

**Lag Thresholds:**
- **< 1000 messages**: Healthy ✅
- **1000-10000 messages**: Monitor ⚠️
- **> 10000 messages**: Action needed 🚨
- **Growing lag**: Scale up consumers immediately 💥

---

## Replication and ISR

### Replication Factor

**Replication** provides fault tolerance by maintaining copies of data across multiple brokers.

```
Topic: payments (replication-factor=3)

Partition 0:
├── Broker 1 (Leader) ← Producers write here
├── Broker 2 (Follower) ← Replicates from leader
└── Broker 3 (Follower) ← Replicates from leader
```

**Configuration:**
```properties
# Topic level
replication.factor=3

# Minimum in-sync replicas
min.insync.replicas=2
```

### Leader and Followers

**Leader:**
- Handles all reads and writes
- Maintains the authoritative copy
- Tracks follower replication progress

**Followers:**
- Replicate data from leader
- Can serve reads (if configured)
- Become leader if current leader fails

### In-Sync Replicas (ISR)

**ISR** = Set of replicas that are fully caught up with the leader.

```
Partition 0 (3 replicas):
Leader: Broker 1 (offset: 1000)
Follower 1: Broker 2 (offset: 1000) ← In ISR ✅
Follower 2: Broker 3 (offset: 950)  ← Out of ISR ❌

ISR = {Broker 1, Broker 2}
```

**A replica is removed from ISR if:**
- Falls behind by more than `replica.lag.time.max.ms` (default: 10s)
- Stops sending fetch requests

**Why ISR Matters:**
```properties
# Producer config
acks=all  # Wait for all ISR replicas to acknowledge

# If ISR = {Leader only} and min.insync.replicas=2
# → Producer will get NotEnoughReplicasException
```

### Durability Guarantees

**Producer `acks` Configuration:**

| acks | Behavior | Durability | Performance |
|------|----------|------------|-------------|
| 0 | Fire and forget | ❌ Lowest | ⚡ Fastest |
| 1 | Wait for leader | ⚠️ Medium | 🚀 Fast |
| all | Wait for all ISR | ✅ Highest | 🐢 Slower |

**Example Scenarios:**

```java
// Scenario 1: Maximum durability (financial transactions)
props.put("acks", "all");
props.put("min.insync.replicas", "2");
props.put("replication.factor", "3");
// Result: Data written to leader + 1 follower minimum

// Scenario 2: High throughput (logs, metrics)
props.put("acks", "1");
props.put("replication.factor", "3");
// Result: Data written to leader only, followers catch up async

// Scenario 3: Maximum throughput (non-critical data)
props.put("acks", "0");
props.put("replication.factor", "1");
// Result: Fire and forget, no durability guarantee
```

### Unclean Leader Election

**Scenario:** All ISR replicas are down. What happens?

```properties
# Option 1: Wait for ISR replica (default)
unclean.leader.election.enable=false
# Result: Topic unavailable until ISR replica recovers
# Pro: No data loss
# Con: Availability impact

# Option 2: Allow out-of-sync replica to become leader
unclean.leader.election.enable=true
# Result: Topic available immediately
# Pro: High availability
# Con: Potential data loss
```

**Production Recommendation:**
- Financial systems: `false` (consistency over availability)
- Logging systems: `true` (availability over consistency)
- Most systems: `false` (default)

---

## Brokers and Controllers

### Broker

A **broker** is a Kafka server that stores data and serves clients.

```
Kafka Cluster:
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Broker 1    │  │  Broker 2    │  │  Broker 3    │
│              │  │              │  │              │
│  Partition 0 │  │  Partition 1 │  │  Partition 2 │
│  Partition 3 │  │  Partition 4 │  │  Partition 5 │
└──────────────┘  └──────────────┘  └──────────────┘
```

**Broker Responsibilities:**
- Store partition data on disk
- Serve produce and fetch requests
- Replicate data from other brokers
- Maintain partition metadata
- Participate in leader elections

**Broker Configuration:**
```properties
# Unique broker ID
broker.id=1

# Data directories (use multiple for better I/O)
log.dirs=/data1/kafka,/data2/kafka,/data3/kafka

# Number of threads for network requests
num.network.threads=8

# Number of threads for I/O operations
num.io.threads=16

# Socket buffer sizes
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
```

### Controller

The **controller** is a special broker responsible for cluster-wide operations.

**Controller Responsibilities:**
1. **Partition Leader Election**: Elect new leaders when brokers fail
2. **Replica Management**: Add/remove replicas
3. **Topic Management**: Create/delete topics
4. **Metadata Propagation**: Push metadata updates to all brokers

**Controller Election:**
```
Initial State:
Broker 1: Controller ✅
Broker 2: Follower
Broker 3: Follower

Broker 1 fails:
Broker 2: Controller ✅ (elected via ZooKeeper/KRaft)
Broker 3: Follower
```

**Controller Failure Impact:**
- Brief unavailability (1-5 seconds)
- No data loss
- Automatic failover
- Clients may experience temporary errors

### Broker Failure Scenarios

#### Scenario 1: Non-Controller Broker Fails
```
Before:
Broker 1 (Controller): Partition 0 (Leader)
Broker 2: Partition 0 (Follower), Partition 1 (Leader)
Broker 3: Partition 0 (Follower), Partition 1 (Follower)

Broker 2 fails:

After:
Broker 1 (Controller): Partition 0 (Leader)
Broker 3: Partition 0 (Follower), Partition 1 (Leader) ← Promoted
```

**Impact:**
- Partition 1 gets new leader (Broker 3)
- Brief unavailability for Partition 1 (< 1 second)
- Partition 0 unaffected

#### Scenario 2: Controller Broker Fails
```
Before:
Broker 1 (Controller): Partition 0 (Leader)
Broker 2: Partition 0 (Follower)
Broker 3: Partition 0 (Follower)

Broker 1 fails:

After:
Broker 2 (Controller): Partition 0 (Leader) ← Promoted
Broker 3: Partition 0 (Follower)
```

**Impact:**
- New controller elected (Broker 2)
- New leader elected for Partition 0
- Brief cluster-wide unavailability (1-5 seconds)
- All operations resume normally

---

## ZooKeeper vs KRaft

### ZooKeeper (Legacy)

**ZooKeeper** is an external coordination service used by Kafka for:
- Controller election
- Cluster membership
- Topic configuration
- ACLs and quotas

```
Architecture:
┌─────────────┐
│  ZooKeeper  │ ← Coordination
│   Ensemble  │
└─────────────┘
      ↓
┌──────────────────────────────┐
│  Kafka Cluster               │
│  ┌────┐  ┌────┐  ┌────┐     │
│  │ B1 │  │ B2 │  │ B3 │     │
│  └────┘  └────┘  └────┘     │
└──────────────────────────────┘
```

**ZooKeeper Limitations:**
- Additional operational complexity
- Separate system to monitor and maintain
- Scalability bottleneck (metadata operations)
- Slower controller failover
- Complex upgrade process

### KRaft (Kafka Raft)

**KRaft** removes ZooKeeper dependency by implementing Raft consensus protocol within Kafka.

```
Architecture:
┌──────────────────────────────┐
│  Kafka Cluster (KRaft Mode)  │
│  ┌────┐  ┌────┐  ┌────┐     │
│  │ C1 │  │ C2 │  │ C3 │     │ ← Controller nodes
│  └────┘  └────┘  └────┘     │
│  ┌────┐  ┌────┐             │
│  │ B1 │  │ B2 │             │ ← Broker nodes
│  └────┘  └────┘             │
└──────────────────────────────┘
```

**KRaft Benefits:**
- Simpler architecture (no external dependency)
- Faster controller failover (< 1 second)
- Better scalability (millions of partitions)
- Simplified operations
- Faster metadata propagation

**Migration Status:**
- KRaft: Production-ready (Kafka 3.3+)
- ZooKeeper: Deprecated (will be removed in Kafka 4.0)
- Recommendation: Use KRaft for new clusters

**KRaft Configuration:**
```properties
# Controller node
process.roles=controller
node.id=1
controller.quorum.voters=1@localhost:9093,2@localhost:9094,3@localhost:9095

# Broker node
process.roles=broker
node.id=4

# Combined node (not recommended for production)
process.roles=broker,controller
node.id=1
```

---

## Message Delivery Semantics

### At-Most-Once Delivery

**Guarantee:** Messages may be lost but never duplicated.

```
Producer:
acks=0 or acks=1 without retries

Consumer:
1. Read message
2. Commit offset
3. Process message ← If crash here, message lost
```

**Use Cases:**
- Metrics collection
- Log aggregation
- Non-critical monitoring

### At-Least-Once Delivery

**Guarantee:** Messages never lost but may be duplicated.

```
Producer:
acks=all
retries=Integer.MAX_VALUE

Consumer:
1. Read message
2. Process message
3. Commit offset ← If crash before commit, message reprocessed
```

**Use Cases:**
- Most production systems
- Event processing
- Data pipelines

**Handling Duplicates:**
```java
// Idempotent processing
void processOrder(Order order) {
    if (orderRepository.exists(order.getId())) {
        return; // Already processed
    }
    orderRepository.save(order);
}
```

### Exactly-Once Semantics (EOS)

**Guarantee:** Messages delivered exactly once, even with failures.

**Producer Configuration:**
```java
props.put("enable.idempotence", "true");
props.put("transactional.id", "order-processor-1");

producer.initTransactions();
try {
    producer.beginTransaction();
    producer.send(record1);
    producer.send(record2);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

**Consumer Configuration:**
```java
props.put("isolation.level", "read_committed");
```

**How It Works:**
1. Producer gets unique PID (Producer ID)
2. Each message gets sequence number
3. Broker deduplicates based on PID + sequence
4. Transactions ensure atomic writes
5. Consumer only reads committed messages

**Use Cases:**
- Financial transactions
- Payment processing
- Inventory management
- Any system requiring strict consistency

**Performance Impact:**
- ~20-30% throughput reduction
- Slightly higher latency
- Worth it for critical data

---

## Key Takeaways

### For Interviews

1. **Partitions = Parallelism**: More partitions = more consumers possible
2. **Consumer Groups**: One partition per consumer in a group
3. **Ordering**: Only guaranteed within a partition
4. **Replication**: Use RF=3, min.insync.replicas=2 for production
5. **ISR**: Critical for durability guarantees

### For Production

1. **Start small**: 3-10 partitions, scale up as needed
2. **Monitor lag**: Alert on growing consumer lag
3. **Plan for failures**: Test broker failures regularly
4. **Use KRaft**: For new clusters (Kafka 3.3+)
5. **Choose semantics**: Match delivery guarantee to use case

### Common Mistakes

1. ❌ Too many partitions upfront
2. ❌ One consumer group per user
3. ❌ Ignoring consumer lag
4. ❌ Not testing rebalancing
5. ❌ Using default configurations

---

**Next:** [Scaling Patterns →](./02-Scaling-Patterns.md)

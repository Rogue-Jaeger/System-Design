# Production Issues & Solutions

Real-world incidents at scale, debugging techniques, and battle-tested solutions.

---

## Table of Contents
1. [Consumer Lag Explosion](#consumer-lag-explosion)
2. [Rebalancing Storms](#rebalancing-storms)
3. [Disk Space Exhaustion](#disk-space-exhaustion)
4. [Under-Replicated Partitions](#under-replicated-partitions)
5. [Memory Issues](#memory-issues)
6. [Network Saturation](#network-saturation)
7. [Monitoring and Alerting](#monitoring-and-alerting)

---

## Consumer Lag Explosion

### Incident: Lag Growing from 0 to 1M in 10 Minutes

**Symptoms:**
```
Time: 10:00 AM - Lag: 0
Time: 10:05 AM - Lag: 500,000
Time: 10:10 AM - Lag: 1,000,000
Time: 10:15 AM - Consumers stopped processing
```

**Root Causes:**

#### Cause 1: Slow Processing

```java
// Bad: Blocking I/O in consumer loop
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        // Slow database call (200ms each)
        database.save(record.value());  // ❌
    }
    consumer.commitSync();
}

// Problem:
// 100 messages × 200ms = 20 seconds per batch
// Producer rate: 1000 msg/s
// Consumer rate: 5 msg/s
// Lag grows at 995 msg/s
```

**Solution:**
```java
// Good: Batch processing
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    List<String> batch = new ArrayList<>();
    
    for (ConsumerRecord<String, String> record : records) {
        batch.add(record.value());
    }
    
    // Batch insert (10ms total)
    database.batchSave(batch);  // ✅
    consumer.commitSync();
}

// Result:
// 100 messages in 10ms
// Consumer rate: 10,000 msg/s
// Keeps up with producer
```

#### Cause 2: Rebalancing Loop

```
10:00 - Consumer A joins
10:01 - Rebalancing starts
10:02 - Consumer B times out (processing too long)
10:03 - Rebalancing starts again
10:04 - Consumer C times out
10:05 - Perpetual rebalancing, no processing
```

**Solution:**
```properties
# Increase timeouts
session.timeout.ms=30000  # 30 seconds (default: 10s)
max.poll.interval.ms=600000  # 10 minutes (default: 5 min)

# Enable cooperative rebalancing
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor

# Reduce batch size if processing is slow
max.poll.records=100  # (default: 500)
```

#### Cause 3: Downstream Service Outage

```
10:00 - External API goes down
10:01 - Consumers retry failed requests
10:02 - Retry timeouts accumulate
10:05 - All consumers blocked on retries
10:10 - Lag explodes
```

**Solution:**
```java
// Circuit breaker pattern
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("external-api");

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, String> record : records) {
        try {
            circuitBreaker.executeSupplier(() -> {
                return externalApi.call(record.value());
            });
        } catch (CallNotPermittedException e) {
            // Circuit open, send to DLQ
            dlqProducer.send(new ProducerRecord<>("dlq", record.key(), record.value()));
        }
    }
    
    consumer.commitSync();
}
```

### Debugging Consumer Lag

```bash
# Check lag per partition
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-processors \
  --describe

# Output:
GROUP           TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG     CONSUMER-ID
order-processors orders  0          1000            1000            0       consumer-1
order-processors orders  1          500             10000           9500    consumer-2  ← Problem
order-processors orders  2          1000            1000            0       consumer-3

# Check consumer metrics
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-processors \
  --describe --members --verbose

# Check processing time
# Add metrics to consumer code
Timer.Context context = processingTimer.time();
processRecord(record);
context.stop();
```

---

## Rebalancing Storms

### Incident: 50 Rebalances in 10 Minutes

**Symptoms:**
```
10:00 - Rebalancing triggered
10:01 - Rebalancing complete
10:02 - Rebalancing triggered again
10:03 - Rebalancing complete
... (repeats 50 times)
```

**Root Causes:**

#### Cause 1: Aggressive Timeouts

```properties
# Too aggressive
session.timeout.ms=6000  # 6 seconds
heartbeat.interval.ms=2000  # 2 seconds
max.poll.interval.ms=30000  # 30 seconds

# Network hiccup or GC pause → consumer kicked out
```

**Solution:**
```properties
# Production-ready
session.timeout.ms=30000  # 30 seconds
heartbeat.interval.ms=3000  # 3 seconds (session.timeout.ms / 10)
max.poll.interval.ms=600000  # 10 minutes
```

#### Cause 2: Kubernetes Pod Churn

```
10:00 - Deployment starts
10:01 - Old pods terminate → rebalancing
10:02 - New pods start → rebalancing
10:03 - Health checks fail → pods restart → rebalancing
10:05 - Perpetual churn
```

**Solution:**
```yaml
# Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-consumer
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1  # Add 1 pod at a time
      maxUnavailable: 0  # Don't kill pods until new ones ready
  template:
    spec:
      containers:
      - name: consumer
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 30"]  # Graceful shutdown
        livenessProbe:
          initialDelaySeconds: 60  # Wait for consumer to stabilize
          periodSeconds: 30
        readinessProbe:
          initialDelaySeconds: 30
          periodSeconds: 10
```

#### Cause 3: Uneven Partition Distribution

```
Before rebalancing:
Consumer 1: 5 partitions (fast)
Consumer 2: 5 partitions (fast)
Consumer 3: 1 partition (idle)

After rebalancing:
Consumer 1: 4 partitions
Consumer 2: 4 partitions
Consumer 3: 3 partitions

Consumer 3 can't keep up → times out → rebalancing again
```

**Solution:**
```properties
# Use sticky assignor
partition.assignment.strategy=org.apache.kafka.clients.consumer.StickyAssignor

# Or cooperative sticky
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

### Monitoring Rebalancing

```java
// Add rebalance listener
consumer.subscribe(Arrays.asList("orders"), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        log.warn("Rebalancing: Partitions revoked: {}", partitions);
        rebalanceCounter.increment();
        // Commit offsets before losing partitions
        consumer.commitSync();
    }
    
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        log.info("Rebalancing: Partitions assigned: {}", partitions);
    }
});
```

---

## Disk Space Exhaustion

### Incident: Brokers Running Out of Disk Space

**Symptoms:**
```bash
$ df -h /data/kafka
Filesystem      Size  Used Avail Use%
/dev/sda1       1.0T  980G   20G  98%  ← Critical!

# Kafka logs:
ERROR Disk full, cannot write to log
ERROR Producer request failed: NOT_ENOUGH_REPLICAS
```

**Root Causes:**

#### Cause 1: Retention Not Configured

```properties
# Default: 7 days retention
log.retention.hours=168

# Problem: High-volume topic
# 1 GB/hour × 168 hours = 168 GB per topic
# 10 topics = 1.68 TB
```

**Solution:**
```properties
# Set appropriate retention
log.retention.hours=24  # 1 day for high-volume topics
log.retention.hours=168  # 7 days for normal topics
log.retention.hours=720  # 30 days for audit logs

# Or size-based retention
log.retention.bytes=107374182400  # 100 GB per partition

# Enable log compaction for stateful topics
log.cleanup.policy=compact
```

#### Cause 2: Compaction Not Running

```bash
# Check compaction lag
kafka-log-dirs --bootstrap-server localhost:9092 \
  --describe --topic-list user-state

# Output shows:
# Partition 0: 500 GB (should be 50 GB after compaction)
```

**Solution:**
```properties
# Tune compaction
log.cleaner.threads=2  # More threads for compaction
log.cleaner.dedupe.buffer.size=134217728  # 128 MB
min.cleanable.dirty.ratio=0.5  # Compact when 50% dirty
```

#### Cause 3: Large Messages

```bash
# Find large messages
kafka-run-class kafka.tools.DumpLogSegments \
  --files /data/kafka/orders-0/00000000000000000000.log \
  --print-data-log | grep "payload size" | sort -k 4 -n | tail -10

# Output:
# offset: 1000 payload size: 50 MB  ← Problem!
```

**Solution:**
```java
// Store large data externally
String s3Url = s3Client.upload(largePayload);
producer.send(new ProducerRecord<>("orders", orderId, s3Url));  // ✅
```

### Monitoring Disk Space

```bash
# Alert when disk > 70%
df -h /data/kafka | awk 'NR==2 {print $5}' | sed 's/%//'

# Monitor per-topic disk usage
du -sh /data/kafka/*

# Set up automated alerts
if [ $(df /data/kafka | tail -1 | awk '{print $5}' | sed 's/%//') -gt 70 ]; then
    echo "ALERT: Disk usage > 70%"
fi
```

---

## Under-Replicated Partitions

### Incident: 50% of Partitions Under-Replicated

**Symptoms:**
```bash
$ kafka-topics --bootstrap-server localhost:9092 --describe --under-replicated-partitions

Topic: orders  Partition: 0  Leader: 1  Replicas: 1,2,3  Isr: 1  ← Missing 2,3
Topic: orders  Partition: 1  Leader: 2  Replicas: 2,3,1  Isr: 2  ← Missing 3,1
Topic: orders  Partition: 2  Leader: 3  Replicas: 3,1,2  Isr: 3  ← Missing 1,2
```

**Root Causes:**

#### Cause 1: Broker Overload

```bash
# Check broker CPU
top
# Broker 2: 100% CPU  ← Can't keep up with replication

# Check replication lag
kafka-replica-verification --broker-list localhost:9092 --topic-white-list 'orders'
```

**Solution:**
```properties
# Increase replication threads
num.replica.fetchers=8  # (default: 1)

# Increase network threads
num.network.threads=16  # (default: 3)

# Or add more brokers to distribute load
```

#### Cause 2: Network Issues

```bash
# Check network latency between brokers
ping broker-2.kafka.svc.cluster.local
# 100ms latency  ← Too high for replication

# Check network throughput
iperf3 -c broker-2.kafka.svc.cluster.local
# 100 Mbps  ← Saturated
```

**Solution:**
- Fix network issues
- Use faster network (10 Gbps)
- Place brokers in same AZ/datacenter

#### Cause 3: Disk I/O Bottleneck

```bash
# Check disk I/O
iostat -x 1

# Output:
# Device: sda
# %util: 100%  ← Disk saturated
# await: 500ms  ← High latency
```

**Solution:**
```
- Use SSDs instead of HDDs
- Use multiple disks (RAID or JBOD)
- Reduce retention to free up I/O
```

### Recovering from Under-Replication

```bash
# 1. Identify problem brokers
kafka-broker-api-versions --bootstrap-server localhost:9092

# 2. Check broker logs
tail -f /var/log/kafka/server.log | grep "ISR"

# 3. Restart slow broker (if needed)
systemctl restart kafka

# 4. Wait for replication to catch up
watch -n 1 'kafka-topics --bootstrap-server localhost:9092 --describe --under-replicated-partitions'

# 5. If broker is dead, reassign partitions
kafka-reassign-partitions --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json \
  --execute
```

---

## Memory Issues

### Incident: Kafka Broker OOM (Out of Memory)

**Symptoms:**
```
ERROR OutOfMemoryError: Java heap space
ERROR Broker shutting down
```

**Root Causes:**

#### Cause 1: Heap Too Small

```bash
# Check heap usage
jstat -gc <kafka-pid> 1000

# Output:
# Heap: 4 GB
# Used: 3.9 GB  ← Almost full
# GC time: 50% of CPU  ← Constant GC
```

**Solution:**
```bash
# Increase heap size
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"  # 6 GB heap

# Rule of thumb:
# Small broker: 4-6 GB
# Medium broker: 8-12 GB
# Large broker: 16-32 GB
# Don't exceed 32 GB (compressed pointers limit)
```

#### Cause 2: Too Many Partitions

```
Partitions per broker: 10,000
Memory per partition: ~1 MB
Total memory: 10 GB just for partitions
```

**Solution:**
```
- Reduce partition count
- Add more brokers
- Increase heap size
- Use log compaction to reduce memory
```

#### Cause 3: Large Messages

```java
// Producer sends 10 MB messages
byte[] largeMessage = new byte[10 * 1024 * 1024];
producer.send(new ProducerRecord<>("topic", largeMessage));

// Broker buffers messages in memory
// 1000 messages = 10 GB memory
```

**Solution:**
```properties
# Limit message size
message.max.bytes=1048576  # 1 MB
replica.fetch.max.bytes=1048576  # 1 MB

# Or store large data externally
```

### Monitoring Memory

```bash
# JVM memory
jstat -gcutil <kafka-pid> 1000

# Heap dump on OOM
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp"

# Analyze heap dump
jmap -heap <kafka-pid>
```

---

## Network Saturation

### Incident: Broker Network Saturated

**Symptoms:**
```bash
# Network usage
iftop
# 950 Mbps out of 1 Gbps  ← Saturated

# Producer errors
ERROR TimeoutException: Failed to send message
ERROR NetworkException: Connection reset
```

**Root Causes:**

#### Cause 1: Too Much Replication Traffic

```
Setup:
- Replication factor: 3
- Producer throughput: 500 MB/s
- Replication traffic: 500 MB/s × 2 = 1 GB/s
- Total: 1.5 GB/s  ← Exceeds 1 Gbps network
```

**Solution:**
```
- Upgrade to 10 Gbps network
- Reduce replication factor (not recommended)
- Add more brokers to distribute traffic
- Enable compression
```

#### Cause 2: No Compression

```java
// Without compression
props.put("compression.type", "none");  // ❌
// 1 GB/s uncompressed

// With compression
props.put("compression.type", "lz4");  // ✅
// 300 MB/s compressed (3x reduction)
```

#### Cause 3: Too Many Consumers

```
Setup:
- 100 consumer groups
- Each reads 100 MB/s
- Total: 10 GB/s  ← Exceeds network capacity
```

**Solution:**
```
- Reduce number of consumer groups
- Use consumer group sharing
- Add more brokers
- Use follower fetching (Kafka 2.4+)
```

### Monitoring Network

```bash
# Network throughput
sar -n DEV 1

# Network errors
netstat -s | grep -i error

# Per-broker network usage
kafka-run-class kafka.tools.JmxTool \
  --object-name kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec \
  --attributes OneMinuteRate
```

---

## Monitoring and Alerting

### Critical Metrics to Monitor

#### Broker Metrics

```
1. Under-replicated partitions
   Alert: > 0 for 5 minutes
   
2. Offline partitions
   Alert: > 0 immediately
   
3. Active controller count
   Alert: != 1
   
4. Disk usage
   Alert: > 70%
   
5. CPU usage
   Alert: > 80%
   
6. Network throughput
   Alert: > 80% of capacity
   
7. Request latency (p99)
   Alert: > 100ms
```

#### Consumer Metrics

```
1. Consumer lag
   Alert: > 10,000 messages
   
2. Lag growth rate
   Alert: Growing for 5 minutes
   
3. Rebalancing frequency
   Alert: > 5 per hour
   
4. Processing time (p99)
   Alert: > max.poll.interval.ms / 2
```

### Prometheus + Grafana Setup

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka-1:9090', 'kafka-2:9090', 'kafka-3:9090']
    
  - job_name: 'kafka-exporter'
    static_configs:
      - targets: ['kafka-exporter:9308']
```

```bash
# Deploy Kafka exporter
docker run -d \
  -p 9308:9308 \
  danielqsj/kafka-exporter \
  --kafka.server=kafka:9092
```

### Alert Rules

```yaml
# alerts.yml
groups:
  - name: kafka
    rules:
      - alert: ConsumerLagHigh
        expr: kafka_consumergroup_lag > 10000
        for: 5m
        annotations:
          summary: "Consumer lag high: {{ $labels.consumergroup }}"
          
      - alert: UnderReplicatedPartitions
        expr: kafka_server_replicamanager_underreplicatedpartitions > 0
        for: 5m
        annotations:
          summary: "Under-replicated partitions detected"
          
      - alert: OfflinePartitions
        expr: kafka_controller_kafkacontroller_offlinepartitionscount > 0
        for: 1m
        annotations:
          summary: "Offline partitions detected"
          
      - alert: DiskUsageHigh
        expr: (node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes > 0.7
        for: 5m
        annotations:
          summary: "Disk usage > 70%"
```

### Debugging Checklist

```
1. Check consumer lag
   kafka-consumer-groups --describe

2. Check under-replicated partitions
   kafka-topics --describe --under-replicated-partitions

3. Check broker logs
   tail -f /var/log/kafka/server.log

4. Check consumer logs
   tail -f /var/log/app/consumer.log

5. Check JVM metrics
   jstat -gcutil <pid>

6. Check disk space
   df -h /data/kafka

7. Check network
   iftop

8. Check broker metrics
   kafka-run-class kafka.tools.JmxTool

9. Check topic config
   kafka-configs --describe --entity-type topics --entity-name orders

10. Check consumer group state
    kafka-consumer-groups --describe --state
```

---

**Next:** [Use Cases →](./07-Use-Cases.md)

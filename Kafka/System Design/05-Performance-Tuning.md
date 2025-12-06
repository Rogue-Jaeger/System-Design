# Performance Tuning - Optimizing Kafka for Production

Comprehensive guide to tuning Kafka for maximum throughput and minimum latency.

---

## Producer Tuning

### Throughput Optimization

```properties
# Batching
batch.size=32768  # 32 KB (default: 16 KB)
linger.ms=10  # Wait 10ms to batch messages

# Compression
compression.type=lz4  # Fast compression (or snappy, gzip, zstd)

# Buffer memory
buffer.memory=67108864  # 64 MB

# In-flight requests
max.in.flight.requests.per.connection=5

# Enable idempotence
enable.idempotence=true
```

**Expected Results:**
- Without tuning: 50 MB/s per producer
- With tuning: 100-150 MB/s per producer

### Latency Optimization

```properties
# Minimal batching
linger.ms=0  # Send immediately
batch.size=1  # No batching

# No compression
compression.type=none

# Acks
acks=1  # Leader only (vs acks=all)
```

**Trade-offs:**
- Lower latency: ~5-10ms (vs 50-100ms)
- Lower throughput: ~30% reduction
- Less durability: Leader-only acks

---

## Consumer Tuning

### Throughput Optimization

```properties
# Fetch more data per request
fetch.min.bytes=1048576  # 1 MB
fetch.max.wait.ms=500  # Wait up to 500ms

# Process more records per poll
max.poll.records=500  # (default: 500)

# Larger receive buffer
receive.buffer.bytes=65536  # 64 KB
```

### Parallel Processing

```java
// Multi-threaded consumer
ExecutorService executor = Executors.newFixedThreadPool(10);

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    List<Future<?>> futures = new ArrayList<>();
    for (ConsumerRecord<String, String> record : records) {
        Future<?> future = executor.submit(() -> processRecord(record));
        futures.add(future);
    }
    
    // Wait for all to complete
    for (Future<?> future : futures) {
        future.get();
    }
    
    consumer.commitSync();
}
```

---

## Broker Tuning

### Core Configuration

```properties
# Network threads
num.network.threads=8  # (default: 3)

# I/O threads
num.io.threads=16  # (default: 8)

# Socket buffer sizes
socket.send.buffer.bytes=102400  # 100 KB
socket.receive.buffer.bytes=102400  # 100 KB

# Replication
num.replica.fetchers=4  # (default: 1)

# Log flush (let OS handle)
log.flush.interval.messages=Long.MAX_VALUE
log.flush.interval.ms=Long.MAX_VALUE
```

### JVM Tuning

```bash
# Heap size
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"

# GC tuning (G1GC)
export KAFKA_JVM_PERFORMANCE_OPTS="-XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M"

# GC logging
export KAFKA_GC_LOG_OPTS="-Xlog:gc*:file=/var/log/kafka/gc.log:time,tags:filecount=10,filesize=100M"
```

### OS Tuning

```bash
# File descriptors
ulimit -n 100000

# VM settings
sysctl -w vm.swappiness=1
sysctl -w vm.dirty_ratio=80
sysctl -w vm.dirty_background_ratio=5

# Network settings
sysctl -w net.core.rmem_max=134217728  # 128 MB
sysctl -w net.core.wmem_max=134217728  # 128 MB
sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"
```

---

## Disk Optimization

### RAID Configuration

```
Best: RAID 10 (striping + mirroring)
- High performance
- High reliability
- Expensive

Good: JBOD (Just a Bunch of Disks)
- Kafka handles replication
- Lower cost
- Good performance

Avoid: RAID 5/6
- Write penalty
- Slow rebuild
```

### Filesystem

```bash
# XFS recommended (better than ext4 for Kafka)
mkfs.xfs -f /dev/sdb
mount -o noatime,nodiratime /dev/sdb /data/kafka

# Add to /etc/fstab
/dev/sdb /data/kafka xfs noatime,nodiratime 0 0
```

---

## Monitoring Key Metrics

### Producer Metrics

```
- record-send-rate: Messages/sec
- record-error-rate: Errors/sec
- request-latency-avg: Average latency
- request-latency-max: Max latency
- buffer-available-bytes: Available buffer
```

### Consumer Metrics

```
- records-consumed-rate: Records/sec
- fetch-latency-avg: Fetch latency
- records-lag: Consumer lag
- commit-latency-avg: Commit latency
```

### Broker Metrics

```
- BytesInPerSec: Incoming bytes/sec
- BytesOutPerSec: Outgoing bytes/sec
- MessagesInPerSec: Messages/sec
- RequestsPerSec: Requests/sec
- UnderReplicatedPartitions: Count
- OfflinePartitionsCount: Count
```

---

## Benchmarking

### Producer Benchmark

```bash
kafka-producer-perf-test \
  --topic test \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props \
    bootstrap.servers=localhost:9092 \
    acks=all \
    compression.type=lz4 \
    batch.size=32768 \
    linger.ms=10

# Output:
# 100000 records sent, 50000 records/sec (48.83 MB/sec)
# Average latency: 25.5 ms
# Max latency: 150 ms
```

### Consumer Benchmark

```bash
kafka-consumer-perf-test \
  --bootstrap-server localhost:9092 \
  --topic test \
  --messages 1000000 \
  --threads 1

# Output:
# 100000 records consumed, 45000 records/sec (43.95 MB/sec)
# Average latency: 30.2 ms
```

---

**Next:** [Reliability Patterns →](./06-Reliability-Patterns.md)

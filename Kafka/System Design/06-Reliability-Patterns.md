# Reliability Patterns - Building Fault-Tolerant Systems

Patterns and practices for building reliable, fault-tolerant Kafka applications.

---

## Exactly-Once Semantics

### Producer Configuration

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("enable.idempotence", "true");  // Critical
props.put("transactional.id", "my-transactional-id");  // Unique per producer
props.put("acks", "all");
props.put("retries", Integer.MAX_VALUE);

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// Initialize transactions
producer.initTransactions();

try {
    producer.beginTransaction();
    
    producer.send(new ProducerRecord<>("topic1", "key1", "value1"));
    producer.send(new ProducerRecord<>("topic2", "key2", "value2"));
    
    producer.commitTransaction();
} catch (ProducerFencedException | OutOfSequenceException | AuthorizationException e) {
    // Fatal errors - close producer
    producer.close();
} catch (KafkaException e) {
    // Abort and retry
    producer.abortTransaction();
}
```

### Consumer Configuration

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "my-group");
props.put("isolation.level", "read_committed");  // Only read committed messages
props.put("enable.auto.commit", "false");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
```

---

## Idempotent Processing

### Database Deduplication

```java
@Transactional
public void processMessage(ConsumerRecord<String, Order> record) {
    String messageId = record.topic() + "-" + record.partition() + "-" + record.offset();
    
    // Check if already processed
    if (processedMessageRepository.exists(messageId)) {
        log.info("Message already processed: {}", messageId);
        return;
    }
    
    // Process message
    Order order = record.value();
    orderRepository.save(order);
    
    // Mark as processed
    processedMessageRepository.save(new ProcessedMessage(messageId, Instant.now()));
}
```

### Application-Level Deduplication

```java
// Use message key or unique ID
public void processOrder(Order order) {
    // Idempotent operation using unique order ID
    if (orderRepository.existsById(order.getId())) {
        // Update instead of insert
        orderRepository.update(order);
    } else {
        orderRepository.insert(order);
    }
}
```

---

## Failure Handling

### Retry with Exponential Backoff

```java
public class RetryableConsumer {
    private static final int MAX_RETRIES = 3;
    private static final long INITIAL_BACKOFF_MS = 1000;
    
    public void processWithRetry(ConsumerRecord<String, String> record) {
        int attempt = 0;
        long backoff = INITIAL_BACKOFF_MS;
        
        while (attempt < MAX_RETRIES) {
            try {
                processRecord(record);
                return;  // Success
            } catch (RecoverableException e) {
                attempt++;
                if (attempt >= MAX_RETRIES) {
                    sendToDLQ(record, e);
                    return;
                }
                
                log.warn("Retry attempt {} for record {}", attempt, record.offset());
                Thread.sleep(backoff);
                backoff *= 2;  // Exponential backoff
            } catch (UnrecoverableException e) {
                sendToDLQ(record, e);
                return;
            }
        }
    }
}
```

### Dead Letter Queue (DLQ)

```java
private final KafkaProducer<String, String> dlqProducer;

private void sendToDLQ(ConsumerRecord<String, String> record, Exception e) {
    ProducerRecord<String, String> dlqRecord = new ProducerRecord<>(
        "dlq-topic",
        record.key(),
        record.value()
    );
    
    // Add headers with error info
    dlqRecord.headers().add("original-topic", record.topic().getBytes());
    dlqRecord.headers().add("original-partition", String.valueOf(record.partition()).getBytes());
    dlqRecord.headers().add("original-offset", String.valueOf(record.offset()).getBytes());
    dlqRecord.headers().add("error-message", e.getMessage().getBytes());
    dlqRecord.headers().add("error-timestamp", String.valueOf(System.currentTimeMillis()).getBytes());
    
    dlqProducer.send(dlqRecord);
    log.error("Sent to DLQ: {}", record.offset(), e);
}
```

---

## Graceful Shutdown

### Producer Shutdown

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    log.info("Shutting down producer...");
    
    // Flush pending messages
    producer.flush();
    
    // Close producer (waits for in-flight requests)
    producer.close(Duration.ofSeconds(30));
    
    log.info("Producer shutdown complete");
}));
```

### Consumer Shutdown

```java
private final AtomicBoolean running = new AtomicBoolean(true);

Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    log.info("Shutting down consumer...");
    running.set(false);
}));

public void consume() {
    try {
        consumer.subscribe(Arrays.asList("orders"));
        
        while (running.get()) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
            
            for (ConsumerRecord<String, String> record : records) {
                processRecord(record);
            }
            
            consumer.commitSync();
        }
    } finally {
        // Commit final offsets
        consumer.commitSync();
        
        // Close consumer (triggers rebalancing)
        consumer.close(Duration.ofSeconds(30));
        
        log.info("Consumer shutdown complete");
    }
}
```

---

## Data Consistency

### Read-Your-Writes

```java
// Producer
RecordMetadata metadata = producer.send(record).get();  // Synchronous send
long offset = metadata.offset();

// Store offset with response
return new Response(orderId, offset);

// Consumer
public Order getOrder(String orderId, long minOffset) {
    // Wait until consumer has processed up to minOffset
    while (consumer.position(partition) < minOffset) {
        consumer.poll(Duration.ofMillis(100));
    }
    
    return orderRepository.findById(orderId);
}
```

### Transactional Outbox Pattern

```java
@Transactional
public void createOrder(Order order) {
    // 1. Save to database
    orderRepository.save(order);
    
    // 2. Save to outbox table (same transaction)
    OutboxEvent event = new OutboxEvent(
        UUID.randomUUID().toString(),
        "order-events",
        order.getId(),
        objectMapper.writeValueAsString(order)
    );
    outboxRepository.save(event);
    
    // Transaction commits atomically
}

// Separate process polls outbox and sends to Kafka
@Scheduled(fixedDelay = 1000)
public void publishOutboxEvents() {
    List<OutboxEvent> events = outboxRepository.findUnpublished();
    
    for (OutboxEvent event : events) {
        producer.send(new ProducerRecord<>(
            event.getTopic(),
            event.getKey(),
            event.getPayload()
        ));
        
        outboxRepository.markPublished(event.getId());
    }
}
```

---

## Circuit Breaker Pattern

```java
CircuitBreaker circuitBreaker = CircuitBreaker.of("external-api", CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .slidingWindowSize(10)
    .build());

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, String> record : records) {
        try {
            circuitBreaker.executeSupplier(() -> {
                return externalApi.call(record.value());
            });
        } catch (CallNotPermittedException e) {
            // Circuit open - send to retry topic
            retryProducer.send(new ProducerRecord<>("retry-topic", record.key(), record.value()));
        }
    }
    
    consumer.commitSync();
}
```

---

## Monitoring for Reliability

### Key Metrics

```
Producer:
- record-error-rate: Should be 0
- record-retry-rate: Monitor spikes
- request-latency-max: Watch for timeouts

Consumer:
- records-lag-max: Should be < threshold
- commit-latency-avg: Watch for slowness
- failed-rebalance-rate: Should be 0

Broker:
- under-replicated-partitions: Should be 0
- offline-partitions-count: Should be 0
- isr-shrinks-per-sec: Monitor for instability
```

### Alerts

```yaml
- alert: ConsumerLagCritical
  expr: kafka_consumergroup_lag > 100000
  for: 5m
  
- alert: UnderReplicatedPartitions
  expr: kafka_server_replicamanager_underreplicatedpartitions > 0
  for: 5m
  
- alert: OfflinePartitions
  expr: kafka_controller_kafkacontroller_offlinepartitionscount > 0
  for: 1m
```

---

**Next:** [Case Studies →](./08-Case-Studies.md)

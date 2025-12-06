# Use Cases & Patterns

Real-world applications of Kafka with implementation patterns and best practices.

---

## Table of Contents
1. [Event Sourcing](#event-sourcing)
2. [Change Data Capture (CDC)](#change-data-capture-cdc)
3. [Stream Processing](#stream-processing)
4. [Log Aggregation](#log-aggregation)
5. [Microservices Communication](#microservices-communication)
6. [Real-time Analytics](#real-time-analytics)

---

## Event Sourcing

### Pattern Overview

Store all changes to application state as a sequence of events.

**Benefits:**
- Complete audit trail
- Time travel (replay to any point)
- Event replay for debugging
- Multiple projections from same events

### Implementation

```java
// Event Store using Kafka
public class OrderEventStore {
    private final KafkaProducer<String, OrderEvent> producer;
    
    public void save(OrderEvent event) {
        ProducerRecord<String, OrderEvent> record = 
            new ProducerRecord<>(
                "order-events",
                event.getOrderId(),  // Key: ensures ordering per order
                event
            );
        
        producer.send(record);
    }
}

// Event Types
interface OrderEvent {
    String getOrderId();
    LocalDateTime getTimestamp();
}

class OrderCreated implements OrderEvent { }
class OrderPaid implements OrderEvent { }
class OrderShipped implements OrderEvent { }
class OrderCancelled implements OrderEvent { }

// Rebuild state from events
public Order rebuildOrder(String orderId) {
    Order order = new Order();
    
    consumer.assign(Arrays.asList(new TopicPartition("order-events", partition)));
    consumer.seekToBeginning(Arrays.asList(new TopicPartition("order-events", partition)));
    
    while (true) {
        ConsumerRecords<String, OrderEvent> records = consumer.poll(Duration.ofMillis(100));
        for (ConsumerRecord<String, OrderEvent> record : records) {
            if (record.key().equals(orderId)) {
                order.apply(record.value());
            }
        }
    }
    
    return order;
}
```

**Configuration:**
```properties
# Long retention for event store
log.retention.ms=-1  # Infinite retention
log.cleanup.policy=compact  # Keep latest per key

# Durability
acks=all
min.insync.replicas=2
replication.factor=3
```

**Use Cases:**
- Financial transactions
- Order management
- Inventory tracking
- User activity tracking

---

## Change Data Capture (CDC)

### Pattern Overview

Capture changes from databases and stream to Kafka for downstream processing.

### Implementation with Debezium

```yaml
# Debezium MySQL Connector
{
  "name": "mysql-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "secret",
    "database.server.id": "1",
    "database.server.name": "mydb",
    "table.include.list": "inventory.orders,inventory.customers",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "schema-changes"
  }
}
```

**Output to Kafka:**
```json
{
  "before": null,
  "after": {
    "id": 1001,
    "customer_id": 123,
    "total": 99.99,
    "status": "PENDING"
  },
  "source": {
    "version": "1.9.0",
    "connector": "mysql",
    "name": "mydb",
    "ts_ms": 1638360000000,
    "snapshot": "false",
    "db": "inventory",
    "table": "orders",
    "server_id": 1,
    "gtid": null,
    "file": "mysql-bin.000003",
    "pos": 154,
    "row": 0
  },
  "op": "c",  // c=create, u=update, d=delete
  "ts_ms": 1638360000123
}
```

**Consumer Processing:**
```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, String> record : records) {
        CDCEvent event = parseEvent(record.value());
        
        switch (event.getOp()) {
            case "c":  // Create
                elasticsearch.index(event.getAfter());
                cache.put(event.getKey(), event.getAfter());
                break;
            case "u":  // Update
                elasticsearch.update(event.getAfter());
                cache.put(event.getKey(), event.getAfter());
                break;
            case "d":  // Delete
                elasticsearch.delete(event.getBefore().getId());
                cache.remove(event.getKey());
                break;
        }
    }
    
    consumer.commitSync();
}
```

**Use Cases:**
- Database replication
- Cache invalidation
- Search index synchronization
- Data warehouse ETL
- Microservices data synchronization

---

## Stream Processing

### Pattern Overview

Process data in real-time as it flows through Kafka.

### Kafka Streams Example

```java
// Word count example
StreamsBuilder builder = new StreamsBuilder();

KStream<String, String> textLines = builder.stream("input-topic");

KTable<String, Long> wordCounts = textLines
    .flatMapValues(line -> Arrays.asList(line.toLowerCase().split("\\W+")))
    .groupBy((key, word) -> word)
    .count(Materialized.as("counts-store"));

wordCounts.toStream().to("output-topic", Produced.with(Serdes.String(), Serdes.Long()));

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

### Windowed Aggregations

```java
// 5-minute tumbling window
KStream<String, PageView> pageViews = builder.stream("page-views");

KTable<Windowed<String>, Long> viewCounts = pageViews
    .groupByKey()
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .count();

viewCounts.toStream()
    .map((windowedKey, count) -> {
        String key = windowedKey.key();
        long start = windowedKey.window().start();
        long end = windowedKey.window().end();
        return new KeyValue<>(key, new ViewCount(key, count, start, end));
    })
    .to("view-counts");
```

### Stateful Processing

```java
// Fraud detection with state
KStream<String, Transaction> transactions = builder.stream("transactions");

KTable<String, UserState> userState = transactions
    .groupByKey()
    .aggregate(
        UserState::new,
        (key, transaction, state) -> {
            state.addTransaction(transaction);
            if (state.getTransactionCount() > 10 && 
                state.getTotalAmount() > 10000) {
                state.setFraudulent(true);
            }
            return state;
        },
        Materialized.<String, UserState, KeyValueStore<Bytes, byte[]>>as("user-state")
            .withKeySerde(Serdes.String())
            .withValueSerde(new UserStateSerde())
    );

transactions
    .join(userState, (transaction, state) -> {
        if (state.isFraudulent()) {
            return new FraudAlert(transaction, state);
        }
        return null;
    })
    .filter((key, alert) -> alert != null)
    .to("fraud-alerts");
```

**Use Cases:**
- Real-time analytics
- Fraud detection
- Recommendation engines
- IoT data processing
- Clickstream analysis

---

## Log Aggregation

### Pattern Overview

Collect logs from multiple sources and centralize in Kafka.

### Architecture

```
Application Servers (1000+)
├── Filebeat → Kafka Topic: app-logs
├── Logstash → Kafka Topic: system-logs
└── Fluentd → Kafka Topic: container-logs

Kafka Cluster
├── app-logs (100 partitions)
├── system-logs (50 partitions)
└── container-logs (100 partitions)

Consumers
├── Elasticsearch (indexing)
├── S3 (archival)
├── Alerting Service (monitoring)
└── Analytics (processing)
```

### Filebeat Configuration

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/app/*.log
  fields:
    service: order-service
    environment: production

output.kafka:
  hosts: ["kafka1:9092", "kafka2:9092", "kafka3:9092"]
  topic: "app-logs"
  partition.round_robin:
    reachable_only: false
  compression: gzip
  max_message_bytes: 1000000
```

### Consumer for Elasticsearch

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    
    BulkRequest bulkRequest = new BulkRequest();
    
    for (ConsumerRecord<String, String> record : records) {
        LogEntry log = parseLog(record.value());
        
        IndexRequest indexRequest = new IndexRequest("logs-" + log.getDate())
            .id(log.getId())
            .source(log.toMap());
        
        bulkRequest.add(indexRequest);
    }
    
    if (bulkRequest.numberOfActions() > 0) {
        BulkResponse bulkResponse = esClient.bulk(bulkRequest, RequestOptions.DEFAULT);
        if (!bulkResponse.hasFailures()) {
            consumer.commitSync();
        }
    }
}
```

**Configuration:**
```properties
# High throughput
batch.size=65536  # 64 KB
linger.ms=100
compression.type=gzip

# Short retention (logs archived to S3)
log.retention.hours=24

# Many partitions for parallelism
num.partitions=100
```

**Use Cases:**
- Centralized logging
- Security monitoring
- Compliance auditing
- Performance analysis
- Troubleshooting

---

## Microservices Communication

### Pattern Overview

Use Kafka as event bus for asynchronous microservices communication.

### Architecture

```
Order Service
  ├→ Kafka: order-events
  
Payment Service
  ├← Kafka: order-events (consumes)
  └→ Kafka: payment-events (produces)
  
Inventory Service
  ├← Kafka: order-events (consumes)
  └→ Kafka: inventory-events (produces)
  
Notification Service
  ├← Kafka: order-events (consumes)
  ├← Kafka: payment-events (consumes)
  └→ Email/SMS
```

### Implementation

```java
// Order Service
@Service
public class OrderService {
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void createOrder(Order order) {
        // Save to database
        orderRepository.save(order);
        
        // Publish event
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getItems(),
            order.getTotal()
        );
        
        kafkaTemplate.send("order-events", order.getId(), event);
    }
}

// Payment Service
@Service
public class PaymentService {
    @KafkaListener(topics = "order-events", groupId = "payment-service")
    public void handleOrderEvent(OrderEvent event) {
        if (event instanceof OrderCreatedEvent) {
            processPayment((OrderCreatedEvent) event);
        }
    }
    
    private void processPayment(OrderCreatedEvent event) {
        // Process payment
        Payment payment = paymentGateway.charge(event.getTotal());
        
        // Publish payment event
        PaymentProcessedEvent paymentEvent = new PaymentProcessedEvent(
            event.getOrderId(),
            payment.getId(),
            payment.getStatus()
        );
        
        kafkaTemplate.send("payment-events", event.getOrderId(), paymentEvent);
    }
}

// Saga Pattern for distributed transactions
@Service
public class OrderSaga {
    @KafkaListener(topics = "payment-events", groupId = "order-saga")
    public void handlePaymentEvent(PaymentEvent event) {
        if (event instanceof PaymentFailedEvent) {
            // Compensating transaction
            OrderCancelledEvent cancelEvent = new OrderCancelledEvent(
                event.getOrderId(),
                "Payment failed"
            );
            kafkaTemplate.send("order-events", event.getOrderId(), cancelEvent);
        }
    }
}
```

**Benefits:**
- Loose coupling
- Async communication
- Event-driven architecture
- Scalability
- Resilience

**Use Cases:**
- E-commerce order processing
- Booking systems
- Payment processing
- Inventory management
- Notification systems

---

## Real-time Analytics

### Pattern Overview

Process and analyze data in real-time for immediate insights.

### Architecture

```
Data Sources
├── Web Events → Kafka: web-events
├── Mobile Events → Kafka: mobile-events
└── IoT Sensors → Kafka: sensor-data

Stream Processing (Kafka Streams / Flink)
├── Aggregations (1min, 5min, 1hour windows)
├── Filtering & Enrichment
└── Anomaly Detection

Output
├── Kafka: aggregated-metrics
├── TimescaleDB (time-series storage)
├── Redis (real-time cache)
└── WebSocket (live dashboard)
```

### Implementation

```java
// Real-time dashboard metrics
StreamsBuilder builder = new StreamsBuilder();

KStream<String, PageView> pageViews = builder.stream("page-views");

// 1-minute tumbling window
KTable<Windowed<String>, Long> pageViewCounts = pageViews
    .groupBy((key, pageView) -> pageView.getPage())
    .windowedBy(TimeWindows.of(Duration.ofMinutes(1)))
    .count();

// Top pages
KTable<String, Long> topPages = pageViewCounts
    .toStream()
    .map((windowedKey, count) -> new KeyValue<>(windowedKey.key(), count))
    .groupByKey()
    .reduce((aggValue, newValue) -> newValue);

// Real-time alerts
pageViewCounts
    .toStream()
    .filter((windowedKey, count) -> count > 10000)  // Threshold
    .map((windowedKey, count) -> {
        return new KeyValue<>(
            windowedKey.key(),
            new Alert("High traffic", windowedKey.key(), count)
        );
    })
    .to("alerts");
```

**Use Cases:**
- Website analytics
- IoT monitoring
- Financial trading
- Fraud detection
- Recommendation engines

---

**Next:** [Performance Tuning →](./05-Performance-Tuning.md)

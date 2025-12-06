# Case Studies - Kafka at Scale

Real-world implementations of Kafka at major tech companies.

---

## LinkedIn - Building Kafka

**Scale:**
- 7+ trillion messages/day
- 4000+ brokers
- 100,000+ topics
- Petabytes of data

**Use Cases:**
- Activity tracking
- Operational metrics
- Log aggregation
- Stream processing

**Key Learnings:**
1. **Partitioning Strategy**: Use high-cardinality keys (user_id, member_id)
2. **Monitoring**: Built Burrow for consumer lag monitoring
3. **Operations**: Automated partition reassignment and rebalancing
4. **Scaling**: Horizontal scaling with careful capacity planning

**Architecture:**
```
User Actions → Kafka → Stream Processing → Multiple Datastores
                 ↓
            Real-time Analytics
                 ↓
            Monitoring & Alerting
```

---

## Uber - Real-Time Data Platform

**Scale:**
- 1+ trillion messages/day
- Hundreds of Kafka clusters
- Thousands of topics
- Multi-datacenter deployment

**Use Cases:**
- Trip events
- Driver location updates
- Pricing calculations
- Fraud detection
- Real-time analytics

**Key Innovations:**
1. **uReplicator**: Improved MirrorMaker for cross-datacenter replication
2. **Chaperone**: End-to-end data auditing
3. **Kafka on Kubernetes**: Container orchestration for Kafka

**Challenges Solved:**
- **Hot Partitions**: Salting keys for high-volume drivers
- **Consumer Lag**: Auto-scaling consumers based on lag metrics
- **Multi-DC**: Active-active replication with conflict resolution

---

## Netflix - Keystone Real-Time Stream Processing

**Scale:**
- 700+ billion events/day
- Multi-region deployment
- Thousands of applications

**Use Cases:**
- Viewing activity tracking
- Recommendation engine
- A/B testing
- Operational monitoring
- Security and fraud detection

**Architecture:**
```
Client Events → Kafka → Flink/Spark → Elasticsearch/Cassandra
                  ↓
            Real-time ML Models
                  ↓
            Personalization
```

**Key Learnings:**
1. **Schema Evolution**: Use Avro with Schema Registry
2. **Backpressure**: Handle slow consumers gracefully
3. **Multi-tenancy**: Isolate workloads with separate clusters
4. **Cost Optimization**: Tiered storage for older data

---

## Airbnb - Real-Time Payment Processing

**Scale:**
- Millions of transactions/day
- Global deployment
- 99.99% availability requirement

**Use Cases:**
- Payment processing
- Booking events
- Pricing updates
- Fraud detection

**Reliability Patterns:**
1. **Exactly-Once**: Transactional producers and consumers
2. **Idempotency**: Database-level deduplication
3. **Circuit Breakers**: Protect against downstream failures
4. **Dead Letter Queues**: Handle poison pills

**Configuration:**
```properties
# Producer (maximum durability)
acks=all
enable.idempotence=true
transactional.id=payment-processor-1
retries=Integer.MAX_VALUE

# Broker
min.insync.replicas=2
replication.factor=3
unclean.leader.election.enable=false

# Consumer
isolation.level=read_committed
enable.auto.commit=false
```

---

## Shopify - Order Processing at Scale

**Scale:**
- Millions of orders/day
- Black Friday: 10x normal traffic
- Global merchants

**Use Cases:**
- Order events
- Inventory updates
- Payment processing
- Shipping notifications

**Scaling Strategy:**
1. **Auto-scaling**: Kubernetes HPA based on consumer lag
2. **Partition Planning**: Pre-scale for peak events
3. **Testing**: Load testing at 20x normal traffic
4. **Monitoring**: Real-time dashboards for Black Friday

**Black Friday Preparation:**
```
Normal: 10 partitions, 10 consumers
Black Friday: 100 partitions, 100 consumers
Pre-scale 1 week before
Gradual scale-down after event
```

---

## Cloudflare - Log Processing at Scale

**Scale:**
- 55+ million HTTP requests/second
- Petabytes of logs/day
- Global edge network

**Use Cases:**
- HTTP request logs
- Security events
- DDoS detection
- Analytics

**Architecture:**
```
Edge Servers (200+ locations)
  ↓
Kafka (Regional clusters)
  ↓
ClickHouse (Analytics)
S3 (Long-term storage)
```

**Optimizations:**
1. **Compression**: GZIP for 10x compression ratio
2. **Batching**: Large batches (1 MB) for throughput
3. **Regional Clusters**: Reduce cross-region traffic
4. **Tiered Storage**: Hot data in Kafka, cold in S3

---

## Common Patterns Across Companies

### 1. Multi-Datacenter Deployment
- Active-active for high availability
- MirrorMaker 2 for replication
- Regional clusters to reduce latency

### 2. Monitoring & Alerting
- Consumer lag monitoring (Burrow)
- Broker health metrics
- End-to-end latency tracking
- Automated alerting

### 3. Schema Management
- Avro/Protobuf with Schema Registry
- Backward/forward compatibility
- Version control for schemas

### 4. Auto-Scaling
- Kubernetes for container orchestration
- HPA based on consumer lag
- Pre-scaling for known events

### 5. Reliability
- Exactly-once semantics for critical data
- Dead letter queues
- Circuit breakers
- Graceful degradation

---

## Key Takeaways

### Start Simple
- Begin with 3 brokers, 10 partitions
- Scale based on actual metrics
- Don't over-engineer initially

### Monitor Everything
- Consumer lag is critical
- Track end-to-end latency
- Alert on anomalies

### Plan for Failures
- Test failure scenarios
- Implement retry logic
- Use dead letter queues

### Scale Horizontally
- Add brokers for throughput
- Add partitions for parallelism
- Add consumers to match partitions

### Optimize for Your Use Case
- Throughput vs latency trade-offs
- Durability vs performance
- Cost vs reliability

---

**End of Kafka System Design Guide**

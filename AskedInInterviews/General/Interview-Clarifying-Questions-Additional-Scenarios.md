# System Design Interview: Additional Scenarios
## Supplementary Question Sets for Data-Intensive, Financial, and Real-Time Systems

---

## Additional Scenario-Specific Question Sets

### Scenario 9: Design a Chat/Messaging System (e.g., WhatsApp, Slack)

#### Initial Clarification

**Functional Requirements**
- **"What type of messaging?"**
  - *Options*: 1-on-1, group chat, channels, or all?
  - *Follow-up*: "Do we need message history, search, or just real-time delivery?"
  - *Follow-up*: "Do we need read receipts, typing indicators, online status?"

- **"What media types?"**
  - *Options*: Text only, images, videos, files, voice messages
  - *Follow-up*: "What's the max file size?"
  - *Implications*: Storage, bandwidth, CDN requirements

- **"Do we need end-to-end encryption?"**
  - *Why it matters*: Affects architecture (can't do server-side processing on encrypted messages)
  - *Implications*: Key exchange, client-side encryption

**Scale Requirements**
- **"How many users?"**
  - *Example*: 1B users, 100M DAU
  - *Implications*: Massive scale

- **"How many messages per day?"**
  - *Example*: 50B messages/day
  - *Implications*: Write-heavy system

- **"What's the average group size?"**
  - *Example*: 1-on-1 (50%), small groups <10 (40%), large groups >100 (10%)
  - *Implications*: Fan-out strategy

- **"What's the message size distribution?"**
  - *Example*: Text (90%, <1KB), images (8%, 500KB), videos (2%, 10MB)
  - *Implications*: Storage, bandwidth

**Latency Requirements**
- **"What's the acceptable message delivery latency?"**
  - *Target*: <100ms for online users (real-time expectation)
  - *Follow-up*: "What about offline users? When they come online?"
  - *Implications*: WebSocket connections, push notifications

- **"Do messages need to be ordered?"**
  - *Why it matters*: Distributed systems can deliver out of order
  - *Implications*: Need sequence numbers or timestamps

**Consistency Requirements**
- **"Can we show messages out of order?"**
  - *Cost*: Confusing UX (reply appears before original message)
  - *Implications*: Need causal ordering

- **"What happens if a message fails to deliver?"**
  - *Options*: Retry, show error, queue for later
  - *Implications*: Reliability guarantees

- **"Can we lose messages?"**
  - *Answer*: No (critical for messaging)
  - *Implications*: Durable storage, acknowledgments

**Availability Requirements**
- **"What's the acceptable downtime?"**
  - *Target*: 99.99% (users expect messaging to always work)
  - *Implications*: Multi-region, redundancy

#### Design Implications

**Real-Time Communication**
- *Challenge*: Maintain persistent connections for 100M concurrent users
- *Options*:
  - **WebSocket**: Bidirectional, low latency
    - *Pro*: Real-time, efficient
    - *Con*: Connection management overhead, scaling challenge
  - **Long polling**: HTTP-based
    - *Pro*: Works through firewalls
    - *Con*: Higher latency, more overhead
  - **Server-Sent Events (SSE)**: One-way push
    - *Pro*: Simpler than WebSocket
    - *Con*: One-way only

**Message Delivery**
- *For online users*: Push via WebSocket
- *For offline users*: Store in queue, push notification, deliver when online

**Group Chat Fan-out**
- *Small groups (<10)*: Fan-out on write (write to each member's inbox)
- *Large groups (>100)*: Fan-out on read (query group messages when user opens chat)
- *Hybrid*: Use threshold (e.g., 50 members)

**Database Design**

**Users Table**
```sql
user_id (PK), username, phone, last_seen, status
```

**Messages Table**
```sql
message_id (PK), sender_id (FK), chat_id (FK), content, media_url, timestamp, sequence_number
```

**Chats Table**
```sql
chat_id (PK), type (1-on-1/group), created_at
```

**Chat Members Table**
```sql
chat_id (FK), user_id (FK), joined_at, last_read_message_id
PRIMARY KEY (chat_id, user_id)
```

**User Inbox (Denormalized)**
```sql
user_id (FK), message_id (FK), chat_id (FK), timestamp
PRIMARY KEY (user_id, timestamp, message_id)
```
- *Purpose*: Fast retrieval of user's messages
- *Storage*: NoSQL (Cassandra, DynamoDB) for scale

**Architecture**

```
User → WebSocket Gateway → Message Service → Message Queue (Kafka) → Delivery Service
                                ↓                                          ↓
                          Message DB                              User Inbox DB
                                                                         ↓
                                                              Push Notification Service
```

**Message Flow**
1. User A sends message to User B
2. Message service writes to Message DB
3. Message service publishes to Kafka
4. Delivery service consumes from Kafka
5. If User B online: Push via WebSocket
6. If User B offline: Write to inbox, send push notification
7. When User B comes online: Fetch from inbox

**Trade-offs to Discuss**
- *WebSocket vs polling*: Real-time vs simplicity
- *Fan-out on write vs read*: Write amplification vs read latency
- *End-to-end encryption*: Privacy vs features (can't do server-side search)
- *Message storage*: Keep forever vs archive old messages

**Edge Cases to Consider**
- *Message ordering*: Network delays can cause out-of-order delivery
- *Duplicate messages*: Network retries can cause duplicates (need idempotency)
- *Connection management*: Handle reconnections, network switches
- *Large groups*: Fan-out to 10K+ members (need batching, rate limiting)
- *Offline users*: Accumulate messages, deliver when online (pagination)

---

### Scenario 10: Design a Data Pipeline / ETL System

#### Initial Clarification

**Functional Requirements**
- **"What's the data flow?"**
  - *Example*: Extract from source DBs → Transform → Load into data warehouse
  - *Follow-up*: "What are the data sources? (databases, APIs, files, streams)"
  - *Follow-up*: "What transformations? (filtering, aggregation, enrichment, joins)"

- **"What's the output?"**
  - *Options*: Data warehouse, data lake, analytics DB, real-time dashboard
  - *Follow-up*: "What's the schema? Fixed or flexible?"

- **"Is it batch or streaming?"**
  - *Batch*: Process data periodically (hourly, daily)
  - *Streaming*: Process data in real-time (seconds)
  - *Hybrid*: Lambda architecture (both batch and streaming)

**Scale Requirements**
- **"How much data per day?"**
  - *Example*: 10TB/day
  - *Implications*: Storage, processing capacity

- **"How many data sources?"**
  - *Example*: 100 databases, 50 APIs, 20 file sources
  - *Implications*: Connector complexity

- **"What's the data growth rate?"**
  - *Example*: 50% YoY
  - *Implications*: Scalability planning

**Latency Requirements**
- **"What's the acceptable data freshness?"**
  - *Real-time*: <1 minute (streaming)
  - *Near real-time*: <15 minutes (micro-batch)
  - *Batch*: Hours to days
  - *Implications*: Architecture choice (Kafka vs batch jobs)

- **"What's the SLA for pipeline completion?"**
  - *Example*: "Daily batch must complete by 8am"
  - *Implications*: Retry strategy, monitoring

**Consistency Requirements**
- **"Can we have duplicate records?"**
  - *Cost*: Incorrect analytics, double-counting
  - *Implications*: Need deduplication (idempotency keys)

- **"Can we lose data?"**
  - *Answer*: No (data is critical)
  - *Implications*: Durable storage, acknowledgments, retries

- **"What happens if transformation fails?"**
  - *Options*: Retry, dead letter queue, alert
  - *Implications*: Error handling strategy

**Data Quality Requirements**
- **"What data validation is needed?"**
  - *Examples*: Schema validation, range checks, referential integrity
  - *Follow-up*: "What happens with invalid data? (reject, quarantine, fix)"

- **"Do we need data lineage?"**
  - *Why*: Track data origin, transformations applied
  - *Implications*: Metadata storage, audit trail

#### Design Implications

**Batch Processing**
- *Use case*: Large volumes, latency tolerance (hours)
- *Technologies*: Apache Spark, Apache Beam, AWS Glue
- *Pattern*: Extract → Stage → Transform → Load

**Streaming Processing**
- *Use case*: Real-time analytics, low latency (<1 min)
- *Technologies*: Apache Kafka, Apache Flink, AWS Kinesis
- *Pattern*: Continuous processing

**Lambda Architecture (Hybrid)**
- *Batch layer*: Process all historical data (slow, accurate)
- *Speed layer*: Process recent data (fast, approximate)
- *Serving layer*: Merge results from both layers
- *Trade-off*: Complexity vs completeness

**Database Design**

**Source Metadata**
```sql
source_id (PK), source_type, connection_string, schema, last_sync_timestamp
```

**Pipeline Runs**
```sql
run_id (PK), pipeline_id (FK), start_time, end_time, status, records_processed, records_failed
```

**Data Lineage**
```sql
record_id, source_id, transformations_applied, load_timestamp
```

**Architecture (Batch)**

```
Data Sources → Ingestion Service → Raw Data Lake (S3) → Spark Jobs → Transformed Data → Data Warehouse
                                                              ↓
                                                      Metadata Store
```

**Architecture (Streaming)**

```
Data Sources → Kafka → Stream Processor (Flink) → Output Sink (Data Warehouse, Cache, etc.)
                          ↓
                    State Store (RocksDB)
```

**Trade-offs to Discuss**
- *Batch vs streaming*: Latency vs complexity
- *Lambda vs Kappa*: Completeness vs simplicity
- *Schema on write vs read*: Data quality vs flexibility
- *Push vs pull*: Real-time vs control

**Edge Cases to Consider**
- *Schema evolution*: Source schema changes (need versioning)
- *Late-arriving data*: Data arrives after window closed (need watermarks)
- *Backfilling*: Reprocess historical data (need idempotency)
- *Data skew*: Some partitions much larger than others (need dynamic partitioning)
- *Exactly-once processing*: Prevent duplicates (need idempotency + transactions)

---

### Scenario 11: Design a Stock Trading Platform

#### Initial Clarification

**Functional Requirements**
- **"What trading features?"**
  - *Core*: Buy/sell stocks, view portfolio, real-time prices
  - *Advanced*: Limit orders, stop-loss, margin trading, options
  - *Follow-up*: "Do we need order matching, or integrate with exchange?"

- **"What markets?"**
  - *Example*: US stocks only, or global?
  - *Implications*: Data feeds, regulatory compliance

**Scale Requirements**
- **"How many users?"**
  - *Example*: 10M users, 1M DAU
  - *Implications*: Capacity planning

- **"How many trades per day?"**
  - *Example*: 10M trades/day
  - *Peak*: Market open/close (10x average)
  - *Implications*: Need to handle spikes

- **"How many price updates per second?"**
  - *Example*: 100K price updates/sec (all stocks)
  - *Implications*: Real-time data streaming

**Latency Requirements**
- **"What's the acceptable order execution latency?"**
  - *Target*: <100ms (users expect fast execution)
  - *Critical*: Must be deterministic (no race conditions)

- **"What's the acceptable price feed latency?"**
  - *Target*: <1 second (real-time prices)
  - *Implications*: WebSocket, streaming

**Consistency Requirements**
- **"Can we execute an order at stale price?"**
  - *Cost*: User gets wrong price (unacceptable, legal issues)
  - *Implications*: Strong consistency for order execution

- **"Can we show stale portfolio value?"**
  - *Tolerance*: Few seconds acceptable
  - *Implications*: Eventual consistency for portfolio view

- **"What happens if order fails mid-execution?"**
  - *Example*: Debit account but order not placed
  - *Implications*: Need distributed transactions (2PC or Saga)

**Regulatory Requirements**
- **"What compliance requirements?"**
  - *Examples*: SEC regulations, KYC (Know Your Customer), AML (Anti-Money Laundering)
  - *Implications*: Audit trails, identity verification, transaction monitoring

- **"What's the audit requirement?"**
  - *Answer*: Every trade must be auditable (immutable log)
  - *Implications*: Append-only ledger, event sourcing

**Availability Requirements**
- **"What's the acceptable downtime?"**
  - *Target*: 99.99% during market hours (critical)
  - *Implications*: Multi-region, redundancy

#### Design Implications

**Order Execution**
- *Challenge*: Ensure exactly-once execution (no duplicate trades)
- *Solution*: Idempotency keys, distributed transactions

**Price Feed**
- *Challenge*: Stream 100K price updates/sec to 1M users
- *Solution*: Pub-sub (Kafka), WebSocket, user subscribes to specific stocks

**Portfolio Calculation**
- *Challenge*: Calculate portfolio value in real-time
- *Solution*: Event sourcing (replay all trades) or materialized view (cache)

**Database Design**

**Users Table**
```sql
user_id (PK), name, email, kyc_status, account_balance
```

**Orders Table**
```sql
order_id (PK), user_id (FK), stock_symbol, order_type (market/limit), quantity, price, status (pending/executed/cancelled), created_at, executed_at
```

**Trades Table (Immutable)**
```sql
trade_id (PK), order_id (FK), user_id (FK), stock_symbol, quantity, price, timestamp
```

**Portfolio Table (Materialized View)**
```sql
user_id (FK), stock_symbol, quantity, avg_cost_basis
PRIMARY KEY (user_id, stock_symbol)
```

**Audit Log (Append-Only)**
```sql
event_id (PK), event_type, user_id, order_id, timestamp, details
```

**Architecture**

```
User → API Gateway → Order Service → Order Matching Engine → Exchange API
                          ↓                    ↓
                      Order DB            Trade DB (append-only)
                                               ↓
                                         Event Stream (Kafka)
                                               ↓
                                    Portfolio Service → Portfolio DB
                                               ↓
                                    Audit Service → Audit Log

Price Feed API → Kafka → WebSocket Gateway → Users
```

**Order Execution Flow**
1. User places order
2. Order service validates (sufficient balance, valid stock)
3. Order service writes to Order DB (status: pending)
4. Order matching engine matches order
5. Execute trade, write to Trade DB (immutable)
6. Publish event to Kafka
7. Portfolio service updates portfolio
8. Audit service logs transaction
9. Notify user

**Trade-offs to Discuss**
- *Latency vs consistency*: Fast execution vs correctness
- *Real-time vs batch*: Portfolio calculation (real-time expensive, batch cheaper)
- *Build vs buy*: Order matching engine (complex, consider third-party)
- *Sync vs async*: Order execution (sync for correctness, async for throughput)

**Edge Cases to Consider**
- *Insufficient funds*: Check balance before execution
- *Stock halted*: Trading suspended (reject orders)
- *Market closed*: Queue orders for next open
- *Partial fills*: Order partially executed (need to track)
- *Concurrent orders*: User places multiple orders (check total balance)

---

### Scenario 12: Design a Notification System

#### Initial Clarification

**Functional Requirements**
- **"What types of notifications?"**
  - *Channels*: Push (mobile), email, SMS, in-app
  - *Follow-up*: "Do users control preferences per channel?"
  - *Follow-up*: "Do we need notification history?"

- **"What triggers notifications?"**
  - *Examples*: User action, system event, scheduled (reminders)
  - *Follow-up*: "Do we need real-time or can we batch?"

- **"Do we need templating?"**
  - *Why*: Personalized messages (e.g., "Hi {name}, you have {count} new messages")
  - *Implications*: Template engine, variable substitution

**Scale Requirements**
- **"How many notifications per day?"**
  - *Example*: 1B notifications/day
  - *Peak*: 100K notifications/sec (e.g., breaking news)
  - *Implications*: Massive scale, need queuing

- **"How many users?"**
  - *Example*: 500M users
  - *Implications*: Storage for preferences

**Latency Requirements**
- **"What's the acceptable delivery latency?"**
  - *Critical*: <1 minute (e.g., security alerts)
  - *Normal*: <5 minutes (e.g., social updates)
  - *Low priority*: <1 hour (e.g., marketing)
  - *Implications*: Priority queues

- **"Do we need delivery guarantees?"**
  - *At least once*: May deliver duplicates
  - *Exactly once*: No duplicates (harder)
  - *Implications*: Idempotency, deduplication

**Consistency Requirements**
- **"Can we send duplicate notifications?"**
  - *Cost*: User annoyance
  - *Implications*: Need deduplication (idempotency keys)

- **"Can we miss notifications?"**
  - *Critical notifications*: No (security, payments)
  - *Non-critical*: Acceptable (social updates)
  - *Implications*: Retry strategy, dead letter queue

**Availability Requirements**
- **"What's the acceptable downtime?"**
  - *Target*: 99.9% (notifications are important but not critical)
  - *Implications*: Redundancy, failover

#### Design Implications

**Priority Queues**
- *High priority*: Security alerts, payment confirmations
- *Medium priority*: Social updates, messages
- *Low priority*: Marketing, newsletters
- *Implementation*: Separate Kafka topics or priority field

**Rate Limiting**
- *Per user*: Don't spam users (e.g., max 10 notifications/hour)
- *Per channel*: Respect provider limits (e.g., SMS rate limits)
- *Implementation*: Token bucket per user

**Deduplication**
- *Problem*: Same event triggers multiple notifications
- *Solution*: Idempotency key (hash of event + user + notification type)
- *Window*: Deduplicate within 24 hours

**Database Design**

**Notification Templates**
```sql
template_id (PK), name, channel, subject, body, variables
```

**User Preferences**
```sql
user_id (FK), channel, notification_type, enabled
PRIMARY KEY (user_id, channel, notification_type)
```

**Notification Log**
```sql
notification_id (PK), user_id (FK), template_id (FK), channel, status (sent/failed), sent_at, idempotency_key
```

**Architecture**

```
Event Source → API Gateway → Notification Service → Priority Queues (Kafka)
                                                          ↓
                                              Channel Workers (Push, Email, SMS)
                                                          ↓
                                              External Providers (FCM, SendGrid, Twilio)
                                                          ↓
                                              Notification Log DB
```

**Notification Flow**
1. Event occurs (e.g., new message)
2. Notification service receives event
3. Check user preferences (is this notification enabled?)
4. Check rate limits (has user received too many?)
5. Check deduplication (already sent?)
6. Render template with variables
7. Publish to appropriate priority queue
8. Channel worker consumes from queue
9. Send via external provider (FCM, SendGrid, Twilio)
10. Log result (sent/failed)
11. If failed, retry with exponential backoff

**Trade-offs to Discuss**
- *Push vs pull*: Push notifications (real-time) vs in-app polling (simpler)
- *Sync vs async*: Immediate send (latency) vs queue (throughput)
- *Batching*: Send individually (real-time) vs batch (efficient for email)
- *Retry strategy*: Aggressive retries (reliability) vs give up (avoid spam)

**Edge Cases to Consider**
- *User uninstalls app*: Push token invalid (need to handle gracefully)
- *Email bounces*: Mark email as invalid, don't retry
- *SMS failures*: Expensive, limit retries
- *Notification fatigue*: Too many notifications (need smart throttling)
- *Time zones*: Send at appropriate local time (need scheduling)

---

## Summary of Additional Scenarios

These 4 additional scenarios cover:

1. **Chat/Messaging System**: Real-time communication, WebSocket management, message ordering
2. **Data Pipeline/ETL**: Batch vs streaming, data quality, exactly-once processing
3. **Stock Trading Platform**: Financial transactions, strong consistency, regulatory compliance
4. **Notification System**: Multi-channel delivery, priority queues, rate limiting

Combined with the original 8 scenarios, you now have **12 comprehensive scenario-specific question sets** covering:
- Web applications (URL shortener, social feed, video streaming)
- Infrastructure (rate limiter, distributed KV store, leaderboard)
- Marketplace (ride-sharing, payments)
- Communication (chat/messaging, notifications)
- Data systems (data pipeline)
- Financial systems (stock trading)

This provides extensive coverage of common system design interview scenarios with deep, structured clarifying questions for each.

# System Design Interview: Mastering Clarifying Questions
## From Mid-Level to Senior/Staff Engineer

> **Core Principle**: Senior and Staff engineers don't pattern-match requirements to technologies. They model problems, quantify constraints, reason through trade-offs, and then select appropriate tools.

---

## Table of Contents

1. [The Fundamental Shift](#the-fundamental-shift)
2. [Universal Questions Framework](#universal-questions-framework)
3. [Requirement Dimensions Deep-Dive](#requirement-dimensions-deep-dive)
4. [Scenario-Specific Question Sets](#scenario-specific-question-sets)
5. [Communication & Interview Flow](#communication--interview-flow)
6. [Anti-Patterns & Red Flags](#anti-patterns--red-flags)
7. [Quantification & Estimation Frameworks](#quantification--estimation-frameworks)
8. [Trade-off Analysis Patterns](#trade-off-analysis-patterns)
9. [Real-World Constraints](#real-world-constraints)
10. [Case Studies: Good vs Bad Approaches](#case-studies-good-vs-bad-approaches)

---

## The Fundamental Shift

### What Separates Levels

**Mid-Level Engineer**
- Knows technologies and their basic use cases
- Can implement given designs
- Focuses on "what" (what components to use)

**Senior Engineer**
- Understands trade-offs deeply
- Can design systems independently
- Focuses on "why" (why this choice over alternatives)
- Quantifies requirements before designing

**Staff Engineer**
- Models problem spaces systematically
- Considers organizational and business constraints
- Focuses on "when" and "how" (when to introduce complexity, how to evolve systems)
- Thinks in terms of failure modes, edge cases, and long-term evolution

### The Interview Mindset

**Don't do this (Candidate 1 approach):**
- Hear "low latency" → say "Redis"
- Hear "high scale" → say "Kafka"
- Hear "rate limiting" → say "token bucket"

**Do this (Candidate 2 approach):**
- Hear "low latency" → ask "What's the target p95/p99? What's acceptable? What's the cost of missing it?"
- Hear "high scale" → ask "What's the QPS? Growth rate? Read/write ratio? Geographic distribution?"
- Hear "rate limiting" → ask "Per what dimension? How strict? What happens on violation? What's the cost of false positives?"

---

## Universal Questions Framework

These questions apply to **every** system design problem. Master this checklist and adapt it to each scenario.

### 1. Functional Requirements Clarification

#### Core Functionality
- **What are the primary user actions/workflows?**
  - *Why it matters*: Defines your API surface and data model
  - *Example*: For a ride-sharing app: "Is it just ride booking, or also driver management, payments, ratings?"
  - *Follow-up*: "Which of these is in scope for this interview?"

- **What are the must-have vs nice-to-have features?**
  - *Why it matters*: Prevents scope creep, focuses design effort
  - *Example*: "Is real-time location tracking must-have, or can we start with periodic updates?"

- **What does success look like for the end user?**
  - *Why it matters*: Reveals hidden requirements (UX expectations)
  - *Example*: "Should ride matching happen in <5 seconds, or is <30 seconds acceptable?"

#### Data & State
- **What data needs to be stored vs computed on-the-fly?**
  - *Why it matters*: Affects storage costs and query patterns
  - *Example*: "Do we pre-compute aggregates or calculate them on demand?"

- **What's the data retention policy?**
  - *Why it matters*: Impacts storage architecture and compliance
  - *Example*: "Do we keep ride history forever, or archive after 2 years?"

- **What data is immutable vs mutable?**
  - *Why it matters*: Influences database choice and caching strategy
  - *Example*: "Can users edit past reviews, or are they write-once?"

### 2. Scale & Performance Requirements

#### Traffic Patterns
- **What's the expected QPS (queries per second)?**
  - *Why it matters*: Baseline for capacity planning
  - *Follow-up questions*:
    - "Current vs 1-year vs 5-year projection?"
    - "Average vs peak?"
    - "What causes peaks? (time of day, events, seasonality)"

- **What's the read-to-write ratio?**
  - *Why it matters*: Determines if you optimize for reads (caching, read replicas) or writes (write-optimized storage)
  - *Example ranges*:
    - Social media feed: 100:1 (heavy read)
    - IoT sensor data: 1:100 (heavy write)
    - E-commerce: 10:1 (read-heavy)

- **What's the geographic distribution of users?**
  - *Why it matters*: Multi-region deployment, CDN strategy, data residency
  - *Example*: "Are users global, or concentrated in specific regions?"

#### Latency Requirements
- **What are the latency targets?**
  - *Critical question*: "What percentile matters? p50, p95, p99, p99.9?"
  - *Why it matters*: p50 vs p99 can mean 10x difference in architecture complexity
  - *Example*:
    - "p95 < 100ms" → aggressive caching, edge computing
    - "p99 < 500ms" → may need multi-region, sophisticated load balancing
    - "p50 < 1s" → simpler architecture, standard caching

- **What's the cost of missing latency targets?**
  - *Why it matters*: Determines how much complexity to introduce
  - *Example*: "Is this a hard SLA with financial penalties, or a soft goal?"

- **Which operations are latency-sensitive?**
  - *Why it matters*: Not all operations need same latency
  - *Example*: "Does search need <100ms, but report generation can take 5 seconds?"

#### Throughput & Capacity
- **What's the data volume?**
  - *Questions to ask*:
    - "How many total users/entities?"
    - "How much data per user/entity?"
    - "Growth rate?"
  - *Example calculation*: 100M users × 1KB profile × 3 years history = 300TB

- **What's the request size distribution?**
  - *Why it matters*: Affects bandwidth, serialization, caching strategy
  - *Example*: "Are requests mostly small (few KB) or large (MBs)?"

### 3. Consistency & Correctness Requirements

#### Consistency Model
- **What consistency guarantees do we need?**
  - *Options to discuss*:
    - **Strong consistency**: All reads see latest write (expensive, high latency)
    - **Eventual consistency**: Reads may be stale temporarily (cheaper, lower latency)
    - **Causal consistency**: Related operations maintain order
  - *Key question*: "What's the cost of showing stale data?"
  - *Example*:
    - Bank balance: Strong consistency required
    - Social media likes: Eventual consistency acceptable
    - Inventory (last item): Depends on business tolerance for overselling

- **Are there any operations that must be atomic?**
  - *Why it matters*: Determines transaction boundaries
  - *Example*: "Must payment and order creation happen atomically?"

- **What happens if we show inconsistent data?**
  - *Why it matters*: Quantifies the business impact
  - *Example*: "If a user sees 100 likes but it's actually 102, is that acceptable?"

#### Durability & Reliability
- **What's the acceptable data loss?**
  - *Key metric*: RPO (Recovery Point Objective)
  - *Example*: "Can we lose last 5 minutes of data, or must we guarantee zero loss?"

- **What's the acceptable downtime?**
  - *Key metric*: RTO (Recovery Time Objective)
  - *Example*: "Can the system be down for 1 hour, or must it be <1 minute?"

- **What's the target availability?**
  - *Common targets*:
    - 99.9% (three nines) = ~8.7 hours downtime/year
    - 99.99% (four nines) = ~52 minutes downtime/year
    - 99.999% (five nines) = ~5 minutes downtime/year
  - *Why it matters*: Each additional nine increases cost exponentially

### 4. Security & Compliance

#### Authentication & Authorization
- **Who are the users and what are their roles?**
  - *Why it matters*: Defines access control model
  - *Example*: "Is it single-tenant or multi-tenant? Role-based or attribute-based access?"

- **What data is sensitive?**
  - *Why it matters*: Encryption, audit logging, access controls
  - *Example*: "PII, financial data, health records?"

#### Compliance Requirements
- **What regulatory requirements apply?**
  - *Common frameworks*:
    - GDPR (EU data protection)
    - CCPA (California privacy)
    - PCI-DSS (payment card data)
    - HIPAA (healthcare)
    - SOC 2 (security controls)
  - *Implications*:
    - Data residency (must store in specific regions)
    - Right to deletion (must support data purging)
    - Audit trails (must log all access)
    - Encryption at rest and in transit

- **What's the data retention and deletion policy?**
  - *Why it matters*: Storage architecture, compliance
  - *Example*: "Must we support 'delete my account' with full data purge?"

### 5. Cost Constraints

- **What's the budget for infrastructure?**
  - *Why it matters*: Influences technology choices
  - *Example*: "Can we use managed services, or must we self-host?"

- **What's more expensive: infrastructure or engineering time?**
  - *Why it matters*: Build vs buy decisions
  - *Example*: "Should we build custom solution or use AWS/GCP managed services?"

- **What's the cost per user/transaction we're targeting?**
  - *Why it matters*: Determines optimization priorities
  - *Example*: "If storage costs $0.01/user/month, can we afford to cache aggressively?"

### 6. Operational Constraints

#### Team & Expertise
- **What's the team size and expertise?**
  - *Why it matters*: Operational complexity you can handle
  - *Example*: "2-person team → prefer managed services; 20-person team → can self-host"

- **What's the on-call burden tolerance?**
  - *Why it matters*: System complexity and operational overhead
  - *Example*: "Can we introduce Kubernetes, or is that too complex for current team?"

#### Existing Infrastructure
- **What's the existing tech stack?**
  - *Why it matters*: Integration points, team familiarity
  - *Example*: "Are we already on AWS, or cloud-agnostic?"

- **Is this greenfield or brownfield?**
  - *Why it matters*: Migration complexity, backward compatibility
  - *Example*: "Must we support existing APIs, or can we start fresh?"

#### Monitoring & Observability
- **What metrics matter most?**
  - *Why it matters*: Defines monitoring strategy
  - *Example*: "Latency, error rate, throughput, cost?"

- **What's the debugging/troubleshooting requirement?**
  - *Why it matters*: Logging, tracing, debugging tools
  - *Example*: "Must we trace individual requests across services?"

---

## Requirement Dimensions Deep-Dive

This section explores each requirement dimension in depth, with questions, trade-offs, and design implications.

### Dimension 1: Latency

#### The Latency Spectrum

**Ultra-low latency (<10ms p99)**
- *Use cases*: High-frequency trading, gaming, real-time collaboration
- *Implications*:
  - In-memory data stores (Redis, Memcached)
  - Edge computing / CDN
  - Co-location with users
  - Optimized serialization (Protocol Buffers, FlatBuffers)
  - Connection pooling, persistent connections
- *Cost*: Very high infrastructure cost, complex deployment

**Low latency (<100ms p99)**
- *Use cases*: Social media, e-commerce, search
- *Implications*:
  - Aggressive caching (multi-tier)
  - Read replicas close to users
  - Async processing where possible
  - Denormalized data for fast reads
- *Cost*: Moderate infrastructure cost, increased complexity

**Medium latency (<500ms p99)**
- *Use cases*: Analytics dashboards, reporting
- *Implications*:
  - Standard caching
  - Single-region deployment acceptable
  - Can use slower but cheaper storage
- *Cost*: Lower infrastructure cost

**High latency (>1s acceptable)**
- *Use cases*: Batch processing, background jobs, reports
- *Implications*:
  - Can use disk-based storage
  - Simpler architecture
  - Cost-optimized over performance
- *Cost*: Minimal infrastructure cost

#### Key Questions to Ask

1. **"What percentile matters for latency?"**
   - *Why*: p50 vs p99 can differ by 10-100x
   - *Example*: If p50=10ms but p99=500ms, you have a tail latency problem
   - *Design impact*: p99 requirements force you to handle outliers (slow queries, GC pauses, network hiccups)

2. **"Is latency measured end-to-end or per-service?"**
   - *Why*: Distributed systems accumulate latency
   - *Example*: If you have 5 services in chain, each with 20ms p99, total is 100ms p99
   - *Design impact*: May need to parallelize calls, use circuit breakers

3. **"What's the latency budget breakdown?"**
   - *Example breakdown for 100ms total*:
     - Network: 20ms
     - Load balancer: 5ms
     - Application logic: 30ms
     - Database query: 40ms
     - Serialization: 5ms
   - *Design impact*: Identifies bottlenecks to optimize

4. **"What causes latency spikes?"**
   - *Common causes*:
     - Cold cache (cache miss)
     - Database query variability
     - GC pauses
     - Network congestion
     - Resource contention
   - *Design impact*: Need strategies for each (cache warming, query optimization, GC tuning, etc.)

5. **"Can we trade latency for consistency?"**
   - *Example*: Read from cache (fast, stale) vs read from DB (slow, fresh)
   - *Design impact*: Determines caching strategy

#### Latency Optimization Patterns

**Pattern 1: Caching Hierarchy**
```
Request → L1 (in-process cache, <1ms)
       → L2 (Redis/Memcached, <5ms)
       → L3 (Read replica, <20ms)
       → L4 (Primary DB, <50ms)
```
- *When to use*: Read-heavy workloads with hot data
- *Trade-off*: Complexity vs performance

**Pattern 2: Denormalization**
- *Example*: Store user profile with every post (avoid join)
- *Trade-off*: Storage cost + update complexity vs read performance

**Pattern 3: Async Processing**
- *Example*: Return immediately, process in background
- *Trade-off*: Eventual consistency vs immediate feedback

**Pattern 4: Edge Computing**
- *Example*: Run logic at CDN edge (Cloudflare Workers, Lambda@Edge)
- *Trade-off*: Limited compute + cold starts vs proximity to users

**Pattern 5: Connection Pooling**
- *Example*: Reuse DB connections instead of creating new ones
- *Trade-off*: Connection limits vs connection overhead

### Dimension 2: Throughput & Scale

#### The Scale Spectrum

**Small scale (<100 QPS)**
- *Implications*: Single server, simple architecture, monolith acceptable
- *Database*: Single PostgreSQL/MySQL instance sufficient
- *Cost*: Very low

**Medium scale (100-10K QPS)**
- *Implications*: Load balancing, caching, read replicas
- *Database*: Primary + read replicas, or managed DB
- *Cost*: Moderate

**Large scale (10K-100K QPS)**
- *Implications*: Microservices, distributed caching, sharding
- *Database*: Sharded database, NoSQL for specific use cases
- *Cost*: High

**Massive scale (>100K QPS)**
- *Implications*: Multi-region, sophisticated sharding, custom infrastructure
- *Database*: Distributed databases (Cassandra, DynamoDB), custom solutions
- *Cost*: Very high

#### Key Questions to Ask

1. **"What's the growth trajectory?"**
   - *Why*: Determines if you design for current scale or future scale
   - *Example*: "Are we at 100 QPS growing to 10K in 6 months, or stable at 100?"
   - *Design impact*: May over-engineer for future, or plan for migration

2. **"What's the read-to-write ratio?"**
   - *Why*: Fundamentally different optimization strategies
   - *Read-heavy (100:1)*:
     - Aggressive caching
     - Read replicas
     - CDN for static content
     - Denormalized data
   - *Write-heavy (1:10)*:
     - Write-optimized storage (Cassandra, time-series DB)
     - Batch writes
     - Async processing
     - Sharding by write key

3. **"What's the traffic pattern?"**
   - *Steady*: Easier to provision, predictable costs
   - *Spiky*: Need auto-scaling, over-provisioning, or queuing
   - *Seasonal*: Plan for peak, scale down during off-peak
   - *Event-driven*: Extreme spikes (e.g., ticket sales, flash sales)
   - *Design impact*:
     - Spiky → need queues (Kafka, SQS) to smooth load
     - Event-driven → need rate limiting, queue-based architecture

4. **"What's the request size distribution?"**
   - *Why*: Affects bandwidth, serialization, caching
   - *Small requests (few KB)*: Can cache aggressively, low bandwidth
   - *Large requests (MBs)*: Need streaming, chunking, CDN
   - *Example*: "Are we serving thumbnails (10KB) or full videos (100MB)?"

5. **"What's the fan-out factor?"**
   - *Why*: One user request may trigger many internal requests
   - *Example*: "Loading a social media feed might query 100 posts, each with user data"
   - *Design impact*: Need batching, caching, or denormalization to avoid N+1 queries

#### Throughput Optimization Patterns

**Pattern 1: Horizontal Scaling**
- *Stateless services*: Add more servers behind load balancer
- *Stateful services*: Shard data, use consistent hashing
- *Trade-off*: Linear cost increase vs unlimited scale

**Pattern 2: Vertical Scaling**
- *Example*: Bigger server (more CPU/RAM)
- *Trade-off*: Limited by hardware, but simpler
- *When to use*: Until you hit hardware limits or cost becomes prohibitive

**Pattern 3: Caching**
- *Example*: Cache hot data in Redis
- *Trade-off*: Stale data vs reduced DB load
- *When to use*: Read-heavy workloads with hot data

**Pattern 4: Async Processing**
- *Example*: Queue writes, process in background
- *Trade-off*: Eventual consistency vs higher throughput
- *When to use*: Write-heavy workloads where immediate consistency not required

**Pattern 5: Batching**
- *Example*: Batch 100 writes into single DB transaction
- *Trade-off*: Latency vs throughput
- *When to use*: High write volume, latency tolerance

**Pattern 6: Sharding**
- *Example*: Partition data by user_id, route requests to specific shard
- *Trade-off*: Complexity vs scale
- *When to use*: Single DB can't handle load

### Dimension 3: Consistency

#### The Consistency Spectrum

**Strong Consistency (Linearizability)**
- *Guarantee*: All reads see the latest write
- *Use cases*: Financial transactions, inventory management, booking systems
- *Implications*:
  - Synchronous replication
  - Coordination overhead (consensus protocols like Raft, Paxos)
  - Higher latency
  - Lower availability (CAP theorem: choose consistency over availability)
- *Technologies*: PostgreSQL, MySQL with synchronous replication, Spanner, CockroachDB

**Sequential Consistency**
- *Guarantee*: Operations from same client are seen in order
- *Use cases*: Collaborative editing, messaging
- *Implications*: Easier to reason about than eventual consistency, but still allows some staleness

**Causal Consistency**
- *Guarantee*: Related operations maintain order (if A causes B, all see A before B)
- *Use cases*: Social media (comment after post), messaging threads
- *Implications*: More complex to implement, but better UX than eventual consistency

**Eventual Consistency**
- *Guarantee*: All replicas converge eventually (no guarantee on timing)
- *Use cases*: Social media likes, view counts, analytics
- *Implications*:
  - Async replication
  - High availability
  - Low latency
  - Must handle conflicts
- *Technologies*: DynamoDB, Cassandra, Riak

**No Consistency (Best Effort)**
- *Guarantee*: None
- *Use cases*: Metrics, logging, non-critical data
- *Implications*: Fire-and-forget, UDP-style

#### Key Questions to Ask

1. **"What's the cost of showing stale data?"**
   - *Why*: Quantifies business impact
   - *Examples*:
     - Bank balance: High cost (legal, trust) → strong consistency
     - Social media likes: Low cost (minor UX issue) → eventual consistency
     - Inventory (last item): Medium cost (overselling) → depends on tolerance
   - *Design impact*: Determines consistency model

2. **"What's the cost of rejecting a request due to consistency?"**
   - *Why*: Availability vs consistency trade-off
   - *Example*: "During network partition, do we reject writes (consistent) or accept them (available)?"
   - *Design impact*: CAP theorem choice

3. **"Are there different consistency requirements for different operations?"**
   - *Why*: Not all operations need same consistency
   - *Example*:
     - Read own writes: User must see their own updates immediately
     - Read others' writes: Can tolerate staleness
   - *Design impact*: Hybrid consistency model

4. **"How do we handle conflicts?"**
   - *Why*: Eventual consistency can lead to conflicts
   - *Strategies*:
     - Last-write-wins (simple, but loses data)
     - Version vectors (complex, but preserves data)
     - Application-level resolution (e.g., merge conflicts)
     - CRDTs (conflict-free replicated data types)
   - *Example*: "Two users edit same document simultaneously"

5. **"What's the acceptable staleness window?"**
   - *Why*: Defines replication lag tolerance
   - *Example*: "Can data be stale for 1 second, 1 minute, 1 hour?"
   - *Design impact*: Replication strategy, monitoring

#### Consistency Patterns & Trade-offs

**Pattern 1: Read-Your-Writes Consistency**
- *Guarantee*: User sees their own updates immediately
- *Implementation*:
  - Route user's reads to same replica that handled write
  - Use sticky sessions
  - Write to cache synchronously, replicate async
- *Trade-off*: Complexity vs UX

**Pattern 2: Monotonic Reads**
- *Guarantee*: User doesn't see data going backward in time
- *Implementation*: Route user to same replica
- *Trade-off*: Limits load balancing flexibility

**Pattern 3: Quorum Reads/Writes**
- *Guarantee*: Tunable consistency (W + R > N ensures strong consistency)
- *Example*: 3 replicas, write to 2, read from 2 → guaranteed to see latest
- *Trade-off*: Latency vs consistency
- *Technologies*: Cassandra, DynamoDB

**Pattern 4: Two-Phase Commit (2PC)**
- *Guarantee*: Atomic distributed transactions
- *Implementation*: Coordinator ensures all participants commit or abort
- *Trade-off*: Blocking protocol, coordinator is single point of failure
- *When to use*: Strong consistency across multiple databases

**Pattern 5: Saga Pattern**
- *Guarantee*: Eventual consistency with compensating transactions
- *Implementation*: Chain of local transactions, each with compensation
- *Trade-off*: Complexity vs availability
- *When to use*: Distributed transactions where 2PC too slow/unavailable

### Dimension 4: Availability & Reliability

#### The Availability Spectrum

**99% (two nines) = ~3.65 days downtime/year**
- *Acceptable for*: Internal tools, non-critical services
- *Architecture*: Single server, basic monitoring

**99.9% (three nines) = ~8.7 hours downtime/year**
- *Acceptable for*: Most web applications
- *Architecture*: Load balancing, health checks, automated restarts

**99.99% (four nines) = ~52 minutes downtime/year**
- *Acceptable for*: Business-critical applications
- *Architecture*: Multi-AZ, automated failover, redundancy

**99.999% (five nines) = ~5 minutes downtime/year**
- *Acceptable for*: Financial systems, healthcare, critical infrastructure
- *Architecture*: Multi-region, sophisticated failover, chaos engineering

#### Key Questions to Ask

1. **"What's the target availability (SLA)?"**
   - *Why*: Each nine increases cost exponentially
   - *Follow-up*: "What's the penalty for missing SLA?"
   - *Design impact*: Determines redundancy, failover complexity

2. **"What's the acceptable downtime (RTO)?"**
   - *RTO = Recovery Time Objective*
   - *Example*: "Can we be down for 1 hour, or must we recover in 1 minute?"
   - *Design impact*:
     - 1 hour → manual failover acceptable
     - 1 minute → automated failover required
     - 1 second → active-active multi-region

3. **"What's the acceptable data loss (RPO)?"**
   - *RPO = Recovery Point Objective*
   - *Example*: "Can we lose last 5 minutes of data, or must we guarantee zero loss?"
   - *Design impact*:
     - 5 minutes → async replication acceptable
     - Zero loss → synchronous replication required

4. **"What are the failure modes we must handle?"**
   - *Common failures*:
     - Server crash
     - Network partition
     - Data center outage
     - Region outage
     - Cascading failures
     - Correlated failures (e.g., software bug affecting all instances)
   - *Design impact*: Determines redundancy strategy

5. **"What's the blast radius of a failure?"**
   - *Why*: Limits impact of failures
   - *Example*: "If one shard fails, does entire system go down?"
   - *Design impact*: Isolation, bulkheads, circuit breakers

6. **"How do we detect failures?"**
   - *Strategies*:
     - Health checks (active monitoring)
     - Heartbeats (passive monitoring)
     - Synthetic transactions (end-to-end monitoring)
   - *Design impact*: Monitoring infrastructure

7. **"How do we recover from failures?"**
   - *Strategies*:
     - Restart (for transient failures)
     - Failover (to standby)
     - Rollback (for bad deployments)
     - Manual intervention (for complex failures)
   - *Design impact*: Automation, runbooks

#### Availability Patterns

**Pattern 1: Active-Passive Failover**
- *Setup*: Primary handles traffic, standby waits
- *Failover*: Detect failure, promote standby to primary
- *Trade-off*: Wastes standby capacity, but simpler
- *RTO*: Minutes to hours (depending on automation)

**Pattern 2: Active-Active Multi-Region**
- *Setup*: Multiple regions serve traffic simultaneously
- *Failover*: Route traffic away from failed region
- *Trade-off*: Higher cost, but better availability and performance
- *RTO*: Seconds to minutes

**Pattern 3: Circuit Breaker**
- *Purpose*: Prevent cascading failures
- *Implementation*: Detect failures, stop sending requests, retry after timeout
- *Trade-off*: Graceful degradation vs full functionality

**Pattern 4: Bulkhead**
- *Purpose*: Isolate failures
- *Implementation*: Separate thread pools, connection pools, or shards
- *Example*: "If payment service fails, search still works"

**Pattern 5: Retry with Exponential Backoff**
- *Purpose*: Handle transient failures
- *Implementation*: Retry failed requests with increasing delays
- *Trade-off*: Resilience vs increased load

**Pattern 6: Graceful Degradation**
- *Purpose*: Maintain core functionality during partial failures
- *Example*: "If recommendation service fails, show popular items instead"
- *Trade-off*: Reduced UX vs availability

### Dimension 5: Security

#### Key Questions to Ask

1. **"Who are the users and what are their roles?"**
   - *Why*: Defines access control model
   - *Options*:
     - Single-tenant vs multi-tenant
     - Role-based access control (RBAC)
     - Attribute-based access control (ABAC)
   - *Design impact*: Authentication, authorization architecture

2. **"What data is sensitive?"**
   - *Categories*:
     - PII (Personally Identifiable Information)
     - Financial data
     - Health records
     - Credentials, API keys
   - *Design impact*: Encryption, access controls, audit logging

3. **"What are the authentication requirements?"**
   - *Options*:
     - Username/password
     - OAuth/OIDC
     - Multi-factor authentication (MFA)
     - Certificate-based
   - *Design impact*: Identity provider, session management

4. **"What are the authorization requirements?"**
   - *Questions*:
     - "Who can access what?"
     - "Are permissions hierarchical?"
     - "Can permissions be delegated?"
   - *Design impact*: Authorization service, policy engine

5. **"What compliance requirements apply?"**
   - *Common frameworks*:
     - GDPR (EU): Right to deletion, data portability, consent
     - CCPA (California): Similar to GDPR
     - PCI-DSS (payments): Encryption, access controls, audit trails
     - HIPAA (healthcare): Encryption, access controls, audit trails
     - SOC 2: Security controls, availability, confidentiality
   - *Design impact*: Data residency, encryption, audit logging, data retention

6. **"What's the threat model?"**
   - *Common threats*:
     - Unauthorized access
     - Data breaches
     - DDoS attacks
     - Injection attacks (SQL, XSS, CSRF)
     - Man-in-the-middle attacks
   - *Design impact*: Security controls, WAF, rate limiting

7. **"What's the incident response plan?"**
   - *Questions*:
     - "How do we detect security incidents?"
     - "How do we respond?"
     - "What's the notification requirement?"
   - *Design impact*: Monitoring, alerting, runbooks

#### Security Patterns

**Pattern 1: Defense in Depth**
- *Principle*: Multiple layers of security
- *Example*:
  - Network layer: Firewall, VPC
  - Application layer: Authentication, authorization
  - Data layer: Encryption at rest
  - Transport layer: TLS
- *Trade-off*: Complexity vs security

**Pattern 2: Principle of Least Privilege**
- *Principle*: Grant minimum necessary permissions
- *Example*: Service accounts with read-only access
- *Trade-off*: Operational overhead vs security

**Pattern 3: Zero Trust**
- *Principle*: Never trust, always verify
- *Example*: Authenticate every request, even internal
- *Trade-off*: Latency vs security

**Pattern 4: Encryption**
- *At rest*: Encrypt data in storage
- *In transit*: TLS/SSL
- *In use*: Homomorphic encryption (rare, expensive)
- *Trade-off*: Performance vs security

**Pattern 5: Audit Logging**
- *Purpose*: Track all access and changes
- *Implementation*: Log who, what, when, where
- *Trade-off*: Storage cost vs compliance

### Dimension 6: Cost

#### Key Questions to Ask

1. **"What's the budget?"**
   - *Why*: Hard constraint on design
   - *Follow-up*: "Is it fixed or flexible?"
   - *Design impact*: Build vs buy, managed vs self-hosted

2. **"What's the cost per user/transaction target?"**
   - *Why*: Determines optimization priorities
   - *Example*: "If we target $0.01/user/month, can we afford to cache aggressively?"
   - *Design impact*: Storage, compute, bandwidth optimization

3. **"What's more expensive: infrastructure or engineering time?"**
   - *Why*: Build vs buy decision
   - *Example*:
     - Small team → use managed services (higher infra cost, lower eng cost)
     - Large team → self-host (lower infra cost, higher eng cost)
   - *Design impact*: Technology choices

4. **"What's the cost breakdown?"**
   - *Common categories*:
     - Compute (servers, containers, functions)
     - Storage (database, object storage, block storage)
     - Bandwidth (data transfer, CDN)
     - Managed services (RDS, ElastiCache, etc.)
     - Licenses (databases, monitoring tools)
   - *Why*: Identifies optimization opportunities

5. **"What's the cost of over-provisioning vs under-provisioning?"**
   - *Over-provisioning*: Wasted money, but handles spikes
   - *Under-provisioning*: Outages, poor UX, lost revenue
   - *Design impact*: Auto-scaling strategy

#### Cost Optimization Patterns

**Pattern 1: Right-Sizing**
- *Principle*: Use appropriately sized resources
- *Example*: Don't use 32-core server for 10% CPU utilization
- *Trade-off*: Optimization effort vs savings

**Pattern 2: Reserved Instances / Savings Plans**
- *Principle*: Commit to usage for discount
- *Example*: AWS Reserved Instances (up to 75% discount)
- *Trade-off*: Flexibility vs cost

**Pattern 3: Spot Instances**
- *Principle*: Use spare capacity at discount
- *Example*: AWS Spot Instances (up to 90% discount)
- *Trade-off*: Availability vs cost
- *When to use*: Batch processing, stateless workloads

**Pattern 4: Auto-Scaling**
- *Principle*: Scale up during peaks, scale down during off-peak
- *Trade-off*: Complexity vs cost

**Pattern 5: Tiered Storage**
- *Principle*: Hot data on fast storage, cold data on cheap storage
- *Example*: Recent data on SSD, old data on S3 Glacier
- *Trade-off*: Complexity vs cost

**Pattern 6: Caching**
- *Principle*: Reduce expensive operations (DB queries, API calls)
- *Trade-off*: Staleness vs cost

**Pattern 7: Compression**
- *Principle*: Reduce storage and bandwidth costs
- *Trade-off*: CPU vs storage/bandwidth

---

*Continued in Part 2...*

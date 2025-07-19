Amazon DynamoDB: 2022 Paper Summary & Analysis
==============================================

Paper Overview
---------------

A study of Amazon's managed NoSQL database service, focusing on its evolution to meet massive scale requirements while maintaining usability and cost-effectiveness.

-------------------------------

Paper: "Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service" (Elhemali et al., USENIX ATC 2022)

-------------------------------

System Overview
---------------

DynamoDB is Amazon's fully managed NoSQL database designed for applications that need to scale with predictable performance, minimal latency, and high availability. This paper describes how Amazon evolved DynamoDB to meet massive scale requirements while remaining highly usable and cost-effectiveness.

DynamoDB powers high-traffic systems like Alexa, Amazon.com, and AWS services such as Lambda, SageMaker, etc. It can serve **tens of millions of requests per second** (e.g., Prime Day 2021 hit 89.2M/sec), offering **sub-10ms latency** for reads and writes at scale.

High-Level Architecture
------------------------

```text
                            +---------------------+
                            |     Application     |
                            +----------+----------+
                                       |
                                       v
                            +----------+----------+
                            |   Request Routers   |  <- Handle auth, routing
                            +----------+----------+
                                       |
                                       v
    +-------------+     +-------------+     +-------------+
    |   MemDS     | <-> |  Metadata   | <-> |   AutoAdmin |
    | (Routing DB)|     |   Service   |     | (Control ops)
    +-------------+     +-------------+     +-------------+
                                       |
                          -----------------------------------
                          |          Storage Nodes           |
                          |   (Hold partitions and replicas) |
                          -----------------------------------
```

Key Concepts
-------------

- **Fully Managed**: Developers don’t need to worry about scaling, patching, provisioning, or recovery.

- **Elastic & Serverless**: Tables scale automatically with data size or demand.

- **Multi-Tenant**: Tables from many customers share hardware with enforced isolation.

- **Predictable Latency**: Thanks to smart partitioning, traffic control, and auto-scaling.

-------------------------------

Major Changes and Improvements
-------------------------------

1. **Shift from Provisioned to On-Demand Mode**: Previously users had to guess read/write capacity. Now DynamoDB scales automatically based on workload. Customers don’t need to over-provision.
2. **Global Admission Control (GAC)**: Replaced older per-partition throttling model. Now requests are smoothed globally across the system, reducing “hot partition” issues.
3. **Automatic Splitting Based on Access Patterns**: Partitions are split not just for size, but also when traffic spikes.
4. **Use of Log Replicas**: Improves availability and quick recovery from failures.
5. **MemDS for Metadata**: Custom in-memory datastore (like a distributed hash+range tree) for request routing and avoiding metadata bottlenecks.
6. **Continuous Verification**: Background scrub jobs revalidate data for corruption using logs and archived states.
7. **Read-Write Safe Deployments**: Supports rolling upgrades and safe protocol evolution without downtime.

-------------------------------

Issues Faced Before This Paper’s Design Changes
-----------------------------------------------

1. **Hot Partitions**:
   - Traffic often skewed to certain keys (e.g., most users access recent data).
   - Old system allocated throughput uniformly. Hot partitions would get throttled.

2. **Throughput Dilution**:
   - Splitting a large partition caused its throughput to be split too, leading to less performance per partition.
   - This caused customers to over-provision just to stay safe.

3. **Static Provisioning Pain**:
   - Users had to pre-guess read/write units (RCUs/WCUs).
   - Wrong guesses led to throttling or unused (wasted) capacity.

4. **Operational Complexity**:
   - Older Dynamo (before DynamoDB) was not managed. Teams had to become DB admins.

5. **Metadata Lookup Storms**:
   - Each router cached full routing tables. New routers booting up led to metadata spikes.
   - Cold starts risked overload and instability.

6. **Failure Recovery Was Slower**:
   - Only full replicas used earlier. Now they use “log replicas” which are much quicker to bring up.

7. **Complexity in Deployments**:
   - Rolling out new features in a distributed system without downtime was risky.
   - Protocol mismatches during upgrades could crash parts of the system.

-------------------------------

Current Limitations and Trade-offs
-----------------------------------

1. **Global Admission Control (GAC) is Ephemeral**:
   - Though resilient, GAC’s decisions are based on cached tokens & estimates.
   - Still reactive in nature for sudden spikes.

2. **Best-Effort Burst & Adaptive Capacity**:
   - Not hard guarantees—depends on available unused capacity.

3. **Complexity in Co-location**:
   - Even though bursting improves utilization, multiple customers on the same node can introduce noisy neighbor problems.

4. **Partition Splits Take Time**:
   - Even “smart splits” based on key distribution take minutes.
   - Workloads with extreme skews (e.g., a single hot item) still suffer.

5. **Grey Failures Still a Challenge**:
   - Network partitions or partial outages are hard to detect.
   - Improvements exist (e.g., health cross-checks), but not foolproof.

6. **MemDS Adds Write Pressure**:
   - Always contacting MemDS for routing can overload it during peak metadata updates.

-------------------------------

Implementation Details
-----------------------

1. **Partitioning Example**:

```text
   [table data] → [hashed by partition key] → [distributed across storage nodes]
```

2. **Global Admission Control (Token Bucket)**:

```text
   ┌────────────┐
   │ Requestor  │
   └─────┬──────┘
         ▼
   [Local Token Bucket]
         ▼
   [Requests Tokens from GAC Service]
         ▼
   [Storage Node]
```

3. **Write Flow with Log Replicas**:

```text
   Client → Request Router → Leader Replica → Log Replicas → Ack
```

-------------------------------

Conclusion
----------

This paper is a blueprint for how to build and operate a cloud-scale, fully managed NoSQL system with predictable performance and rich features. It highlights the **importance of designing for operations, not just architecture**. Amazon’s ability to mix formal verification, chaos testing, elastic scaling, and invisible maintenance is what lets DynamoDB continue to dominate in the cloud-native NoSQL space.

-------------------------------

Diagrams Section
-----------------

Here’s a detailed **text-based visual guide** for the architecture and flows described in the 2022 DynamoDB paper, designed for software engineers.

-------------------------------

System Architecture Diagram
----------------------------

```text
                            +---------------------+
                            |     Application     |
                            +----------+----------+
                                       |
                                       v
                            +----------+----------+
                            |   Request Routers   |  <- Handle auth, routing
                            +----------+----------+
                                       |
                                       v
    +-------------+     +-------------+     +-------------+
    |   MemDS     | <-> |  Metadata   | <-> |   AutoAdmin |
    | (Routing DB)|     |   Service   |     | (Control ops)
    +-------------+     +-------------+     +-------------+
                                       |
                          -----------------------------------
                          |          Storage Nodes           |
                          |   (Hold partitions and replicas) |
                          -----------------------------------
```

-------------------------------

Partitioning Model
-------------------

```text
    Table = Collection of items (key-value/document-style)

    Item keys → Partition Key (hashed) → maps to storage node

    Composite Key (Partition + Sort) allows ordered access

    Example:
    ┌───────────────┬──────────────┐
    │ Partition Key │ Sort Key     │
    ├───────────────┼──────────────┤
    │ customer_42   │ order_001    │
    │ customer_42   │ order_002    │
    │ customer_99   │ order_003    │
    └───────────────┴──────────────┘
```

-------------------------------

Replication and Write Path (Simplified Paxos)
---------------------------------------------

```text
      +--------------------+
      |    Client App      |
      +--------------------+
                |
                v
         [Request Router]
                |
                v
         [Leader Replica]  ← Only this handles strong writes
                |
     ┌──────────┴──────────┐
     ▼                     ▼
[Storage Replica]   [Log-Only Replica]
(B-tree + logs)     (only logs, faster repair)
```

- **Write flow**:

  - Leader generates Write-Ahead Log (WAL)
  - Sends to quorum (2 of 3 replicas)
  - ACKs client after quorum persists log
- **Log replicas** help quick heal on failure

-------------------------------

Global Admission Control (GAC) – Token Buckets
----------------------------------------------

```text
    Goal: Prevent overload from bursty workloads

          +----------------------+
          |   Application/Client |
          +----------+-----------+
                     |
                     v
         +------------------------+
         |  Request Router (local)|
         |  [Token Bucket Cache]  |
         +-----------+------------+
                     |
                     v
          +--------------------+
          |   GAC Service      |
          |   (Central Policy) |
          +--------------------+

- Routers "spend" tokens to allow requests
- Periodically replenish from GAC
- Ensures table-level fairness even if partition skew exists
```

-------------------------------

Handling Hot Partitions (Key Insight)
-------------------------------------

```text
Old Model:
-----------
- Static throughput per partition
- Hot partition = throttled, even if table has spare capacity

New Model:
-----------
- Bursting → Temporary use of unused capacity
- Adaptive Capacity → Reactively reallocates throughput
- Global Admission Control (GAC) → Proactively spreads load
- Split-for-consumption → Hot partitions are split using key access history

Example:
        Table with 1M keys
        80% requests → last 5% of keys
        -> GAC and splitting isolate the hotspots
```

-------------------------------

Durability Strategy
--------------------

```text
→ Write-Ahead Logs in 3 replicas (quorum)
→ Logs also archived to Amazon S3
→ Log Replicas help fast recovery

Extra Layers:
- Checksums for each WAL entry
- Scrubber process revalidates live data vs archived logs
- Formal verification using TLA+ for replication protocol

Backups:
- Full-table snapshots stored in S3
- Point-in-time recovery (up to 35 days)
```

-------------------------------

Metadata Routing using MemDS (Perkle Tree)
------------------------------------------

```text
Problem:
  - Earlier: Request router had to fetch & cache entire routing table
  - Caused cold-start pressure, cache failures

New Solution:
  - MemDS (in-memory metadata store)
  - Efficient range lookups with Perkle Tree (Patricia + Merkle)
  - Always-on traffic keeps MemDS warmed

Lookup:
    Key = "user_42"
    ↓ floor(key) operation in MemDS
    → Returns partition and storage node
```

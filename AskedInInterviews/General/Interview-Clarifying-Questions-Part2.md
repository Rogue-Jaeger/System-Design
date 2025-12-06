# System Design Interview: Clarifying Questions - Part 2
## Scenario-Specific Question Sets (Continued)

---

## Scenario-Specific Question Sets

### Scenario 1: Design a URL Shortener (e.g., bit.ly)

#### Initial Clarification

**Functional Requirements**
- **"What's the core functionality?"**
  - Shorten long URLs to short codes
  - Redirect short URLs to original URLs
  - *Follow-up*: "Do we need analytics (click tracking)?"
  - *Follow-up*: "Do we need custom short codes?"
  - *Follow-up*: "Do we need expiration?"

- **"What's the short code format?"**
  - *Options*: Alphanumeric (62 chars: a-z, A-Z, 0-9)
  - *Length*: 6 chars = 62^6 = 56 billion URLs
  - *Follow-up*: "Is 6 characters acceptable, or do we need shorter/longer?"

**Scale Requirements**
- **"How many URLs will we shorten per day?"**
  - *Example*: 100M URLs/day
  - *Implications*: Write-heavy system

- **"How many redirects per day?"**
  - *Example*: 10B redirects/day (100:1 read-to-write ratio)
  - *Implications*: Read-heavy, needs aggressive caching

- **"What's the expected QPS?"**
  - *Write*: 100M / 86400 = ~1,200 writes/sec
  - *Read*: 10B / 86400 = ~115,000 reads/sec
  - *Implications*: Need caching, read replicas, or CDN

**Latency Requirements**
- **"What's the acceptable redirect latency?"**
  - *Target*: <50ms p99 (users expect instant redirect)
  - *Implications*: Aggressive caching, CDN

**Data Requirements**
- **"What's the data retention policy?"**
  - *Options*: Forever, or expire after X years
  - *Implications*: Storage planning
  - *Calculation*: 100M URLs/day × 365 days × 5 years × 500 bytes = 9TB

- **"Can short codes be reused after expiration?"**
  - *Why*: Affects short code generation strategy

**Consistency Requirements**
- **"What happens if we generate duplicate short codes?"**
  - *Cost*: Collision, one URL overwrites another (unacceptable)
  - *Implications*: Need uniqueness guarantee

- **"Can we show stale redirects?"**
  - *Example*: URL updated, but cache still has old URL
  - *Cost*: User sees wrong page (depends on use case)
  - *Implications*: Cache invalidation strategy

**Analytics Requirements**
- **"Do we need click tracking?"**
  - *If yes*: "What metrics? (clicks, geography, referrer, device)"
  - *Implications*: Write to analytics DB on every redirect (adds latency)
  - *Trade-off*: Async write (eventual consistency) vs sync write (latency)

#### Design Implications

**Short Code Generation**
- *Option 1*: Hash (MD5, SHA) → take first 6 chars
  - *Pro*: Deterministic
  - *Con*: Collisions possible, need to check and retry
- *Option 2*: Counter (base62 encode)
  - *Pro*: No collisions
  - *Con*: Need distributed counter (coordination overhead)
- *Option 3*: Random + check for collision
  - *Pro*: Simple
  - *Con*: Collision probability increases with scale

**Database Choice**
- *Option 1*: SQL (PostgreSQL, MySQL)
  - *Pro*: ACID, familiar
  - *Con*: Harder to scale writes
  - *Schema*: `(short_code PK, long_url, created_at, expires_at)`
- *Option 2*: NoSQL (DynamoDB, Cassandra)
  - *Pro*: Scales writes easily
  - *Con*: Eventual consistency (may generate duplicate short codes)

**Caching Strategy**
- *Why*: 100:1 read-to-write ratio, caching essential
- *What to cache*: short_code → long_url mapping
- *Where*: Redis, Memcached, or CDN
- *TTL*: Long (hours to days) since URLs rarely change
- *Invalidation*: On URL update or deletion

**Architecture**
```
User → CDN (cache redirects) → Load Balancer → App Servers → Redis (cache) → Database
                                                           ↓
                                                    Analytics Queue (Kafka) → Analytics DB
```

**Trade-offs to Discuss**
- *Short code length*: Shorter = better UX, but fewer possible URLs
- *Caching*: Aggressive caching = low latency, but stale data risk
- *Analytics*: Sync write = accurate, but adds latency; Async = fast, but may lose data

---

### Scenario 2: Design a Rate Limiter

#### Initial Clarification

**Functional Requirements**
- **"What are we rate limiting?"**
  - *Options*: API requests, login attempts, messages sent
  - *Follow-up*: "Per what dimension? (user, IP, API key, endpoint)"

- **"What's the rate limit?"**
  - *Example*: 100 requests per minute per user
  - *Follow-up*: "Is it a hard limit or soft limit?"
  - *Follow-up*: "Do we have different limits for different users/tiers?"

- **"What happens when limit is exceeded?"**
  - *Options*:
    - Reject request (HTTP 429)
    - Queue request (delay)
    - Degrade service (lower quality response)
  - *Follow-up*: "Do we return how much quota is left?"

**Scale Requirements**
- **"How many requests per second?"**
  - *Example*: 1M requests/sec
  - *Implications*: Distributed rate limiting required

- **"How many users/entities to track?"**
  - *Example*: 100M users
  - *Implications*: Storage requirements

**Consistency Requirements**
- **"How strict is the rate limit?"**
  - *Strict*: Never exceed limit (strong consistency)
  - *Loose*: Occasional overages acceptable (eventual consistency)
  - *Why it matters*: Strict requires coordination (slow), loose allows local rate limiting (fast)

- **"What's the cost of false positives (blocking legitimate requests)?"**
  - *High cost*: User frustration, lost revenue
  - *Low cost*: Minor inconvenience
  - *Design impact*: How conservative to be

- **"What's the cost of false negatives (allowing excess requests)?"**
  - *High cost*: System overload, abuse
  - *Low cost*: Minor over-provisioning
  - *Design impact*: How aggressive to be

**Latency Requirements**
- **"What's the acceptable latency for rate limit check?"**
  - *Target*: <5ms (shouldn't add noticeable latency)
  - *Implications*: In-memory check, local cache

**Availability Requirements**
- **"What happens if rate limiter is down?"**
  - *Fail open*: Allow all requests (risk of abuse)
  - *Fail closed*: Block all requests (risk of outage)
  - *Design impact*: Redundancy, fallback strategy

#### Design Implications

**Algorithm Choice**

**Option 1: Token Bucket**
- *How it works*: Bucket holds tokens, each request consumes token, tokens refill at fixed rate
- *Pro*: Allows bursts, smooth rate limiting
- *Con*: Requires storing state (token count, last refill time)
- *Use case*: General purpose

**Option 2: Leaky Bucket**
- *How it works*: Requests enter queue, processed at fixed rate
- *Pro*: Smooth output rate
- *Con*: Requests may be delayed
- *Use case*: When you want to smooth traffic

**Option 3: Fixed Window**
- *How it works*: Count requests in fixed time window (e.g., 0-60 seconds)
- *Pro*: Simple, low memory
- *Con*: Burst at window boundary (2x limit possible)
- *Use case*: When simplicity matters more than accuracy

**Option 4: Sliding Window Log**
- *How it works*: Store timestamp of each request, count requests in sliding window
- *Pro*: Accurate
- *Con*: High memory (store all timestamps)
- *Use case*: When accuracy critical and scale is low

**Option 5: Sliding Window Counter**
- *How it works*: Hybrid of fixed window and sliding window
- *Pro*: Accurate, low memory
- *Con*: More complex
- *Use case*: Best balance of accuracy and efficiency

**Storage Choice**

**Option 1: In-Process (Local)**
- *Pro*: Fastest (<1ms)
- *Con*: Not accurate in distributed system (each server has own limit)
- *When to use*: Single server, or loose limits acceptable

**Option 2: Redis (Centralized)**
- *Pro*: Accurate, shared state
- *Con*: Network latency (~1-5ms), Redis is single point of failure
- *When to use*: Distributed system, strict limits

**Option 3: Hybrid (Local + Redis)**
- *How it works*: Local cache with short TTL, sync to Redis periodically
- *Pro*: Fast + accurate
- *Con*: Complex
- *When to use*: Distributed system, strict limits, low latency

**Architecture**

**Centralized Rate Limiter**
```
Request → Load Balancer → App Server → Rate Limiter Service (Redis) → Upstream Service
```
- *Pro*: Accurate, centralized control
- *Con*: Latency, single point of failure

**Distributed Rate Limiter**
```
Request → Load Balancer → App Server (local rate limiter) → Upstream Service
                                ↓
                          Sync to Redis (async)
```
- *Pro*: Low latency, no single point of failure
- *Con*: Less accurate (eventual consistency)

**Trade-offs to Discuss**
- *Accuracy vs latency*: Centralized (accurate, slow) vs local (fast, loose)
- *Fail open vs fail closed*: Availability vs security
- *Per-user vs per-IP*: Accuracy vs privacy (IP can be shared)
- *Fixed vs sliding window*: Simplicity vs accuracy

**Edge Cases to Consider**
- *Clock skew*: Servers have different times → inconsistent rate limiting
- *Race conditions*: Two requests check limit simultaneously, both pass
- *Distributed counters*: How to aggregate counts across servers
- *Quota reset*: When does the window reset? (midnight? rolling?)

---

### Scenario 3: Design a Ride-Sharing Service (e.g., Uber, Lyft)

#### Initial Clarification

**Functional Requirements**
- **"What's in scope?"**
  - *Core*: Ride booking, driver matching, real-time tracking
  - *Out of scope*: Payments, ratings, surge pricing (clarify)
  - *Follow-up*: "Do we need ride history, scheduled rides, ride sharing (multiple passengers)?"

- **"What's the matching algorithm?"**
  - *Simple*: Nearest driver
  - *Complex*: Optimize for ETA, driver rating, acceptance rate
  - *Follow-up*: "Do we need to consider traffic, driver direction?"

**Scale Requirements**
- **"How many users and drivers?"**
  - *Example*: 100M users, 1M drivers
  - *Implications*: Storage, indexing

- **"How many rides per day?"**
  - *Example*: 10M rides/day
  - *Implications*: Write load, storage growth

- **"What's the geographic distribution?"**
  - *Example*: Global, or specific cities
  - *Implications*: Multi-region deployment, data residency

- **"What's the peak QPS?"**
  - *Example*: 10M rides/day, peak 10x average = ~1,200 rides/sec
  - *Implications*: Capacity planning

**Latency Requirements**
- **"What's the acceptable matching time?"**
  - *Target*: <5 seconds (users expect fast matching)
  - *Implications*: Efficient geospatial search

- **"What's the acceptable location update frequency?"**
  - *Example*: Every 5 seconds
  - *Implications*: Write load (1M drivers × 12 updates/min = 200K writes/sec)

**Consistency Requirements**
- **"Can a driver be matched to multiple riders?"**
  - *Cost*: Double booking (unacceptable)
  - *Implications*: Need atomic driver assignment (strong consistency)

- **"Can we show stale driver locations?"**
  - *Cost*: Inaccurate ETA, poor UX
  - *Tolerance*: 5-10 seconds staleness acceptable
  - *Implications*: Eventual consistency for location updates

**Availability Requirements**
- **"What's the acceptable downtime?"**
  - *Target*: 99.99% (critical service)
  - *Implications*: Multi-region, automated failover

- **"What happens during partial outages?"**
  - *Example*: One city goes down
  - *Implications*: Isolation, graceful degradation

#### Design Implications

**Geospatial Indexing**
- *Problem*: Find nearby drivers efficiently
- *Options*:
  - **Geohash**: Encode lat/lng as string, use prefix matching
    - *Pro*: Simple, works with any DB
    - *Con*: Edge cases at geohash boundaries
  - **QuadTree**: Recursively divide map into quadrants
    - *Pro*: Efficient for sparse data
    - *Con*: Complex to implement, rebalancing needed
  - **Google S2**: Hierarchical geospatial indexing
    - *Pro*: Handles edge cases well
    - *Con*: Complex
  - **PostGIS**: PostgreSQL extension for geospatial queries
    - *Pro*: SQL-based, mature
    - *Con*: Harder to scale

**Matching Algorithm**
- *Simple*: Find nearest available driver
- *Complex*: Optimize for multiple factors
  - Distance to rider
  - Driver rating
  - Driver acceptance rate
  - Traffic conditions
  - Driver direction (heading towards rider)
- *Trade-off*: Simplicity vs optimal matching

**Real-Time Location Tracking**
- *Challenge*: 1M drivers × 12 updates/min = 200K writes/sec
- *Options*:
  - **WebSocket**: Persistent connection, low latency
    - *Pro*: Real-time, bidirectional
    - *Con*: Connection management overhead
  - **HTTP polling**: Client polls server periodically
    - *Pro*: Simple
    - *Con*: Inefficient, higher latency
  - **Server-Sent Events (SSE)**: Server pushes updates
    - *Pro*: Simpler than WebSocket
    - *Con*: One-way only

**Database Design**

**Users Table**
```sql
user_id (PK), name, phone, email, created_at
```

**Drivers Table**
```sql
driver_id (PK), name, phone, vehicle_info, rating, status (available/busy), current_location (lat, lng), last_updated
```

**Rides Table**
```sql
ride_id (PK), rider_id (FK), driver_id (FK), pickup_location, dropoff_location, status (requested/matched/in_progress/completed), requested_at, matched_at, completed_at
```

**Driver Locations (Time-Series)**
```sql
driver_id, timestamp, lat, lng
```
- *Storage*: Time-series DB (InfluxDB, TimescaleDB) or NoSQL (Cassandra)
- *Retention*: Keep recent locations (e.g., last 24 hours)

**Architecture**

```
Rider App → API Gateway → Ride Service → Matching Service → Driver Service
                                              ↓
                                      Geospatial Index (Redis/PostGIS)
                                              ↓
                                         Driver Locations DB

Driver App → WebSocket Gateway → Location Service → Driver Locations DB
                                                  ↓
                                          Geospatial Index (update)
```

**Matching Flow**
1. Rider requests ride (pickup location)
2. Matching service queries geospatial index for nearby available drivers
3. Rank drivers by distance, rating, etc.
4. Send ride request to top N drivers (in parallel or sequentially)
5. First driver to accept gets matched
6. Update driver status to "busy" (atomic operation to prevent double booking)
7. Notify rider of match

**Trade-offs to Discuss**
- *Push vs pull*: WebSocket (push) vs polling (pull) for location updates
- *Accuracy vs latency*: Precise geospatial search vs approximate (geohash)
- *Matching algorithm*: Simple (fast) vs complex (better UX)
- *Database choice*: SQL (ACID, complex queries) vs NoSQL (scale, performance)

**Edge Cases to Consider**
- *Driver cancellation*: Re-match rider
- *No drivers available*: Queue rider, notify when driver available
- *Network issues*: Handle stale location data, reconnection
- *Concurrent ride requests*: Prevent double booking (atomic driver assignment)
- *Geospatial edge cases*: Drivers near geohash boundaries, international date line

---

### Scenario 4: Design a Social Media Feed (e.g., Twitter, Facebook)

#### Initial Clarification

**Functional Requirements**
- **"What's in the feed?"**
  - *Options*: Posts from followed users, recommended posts, ads
  - *Follow-up*: "Is it chronological or algorithmic?"
  - *Follow-up*: "Do we need real-time updates (new posts appear live)?"

- **"What can users do?"**
  - *Core*: Post, follow, like, comment, share
  - *Follow-up*: "Do we need media (images, videos)?"
  - *Follow-up*: "Do we need hashtags, mentions, search?"

**Scale Requirements**
- **"How many users?"**
  - *Example*: 1B users, 100M DAU (daily active users)
  - *Implications*: Massive scale

- **"How many posts per day?"**
  - *Example*: 500M posts/day
  - *Implications*: Write load

- **"How many feed loads per day?"**
  - *Example*: 100M DAU × 20 feed loads/day = 2B feed loads/day
  - *Implications*: Read load (heavy read)

- **"What's the follower distribution?"**
  - *Example*: Power law (few users with millions of followers, most with <100)
  - *Implications*: Fanout strategy

**Latency Requirements**
- **"What's the acceptable feed load time?"**
  - *Target*: <500ms p99
  - *Implications*: Pre-computed feeds, caching

- **"What's the acceptable post latency?"**
  - *Target*: <1s (user expects immediate feedback)
  - *Implications*: Async fanout acceptable

**Consistency Requirements**
- **"Can we show stale feeds?"**
  - *Cost*: User doesn't see latest posts immediately
  - *Tolerance*: Few seconds staleness acceptable
  - *Implications*: Eventual consistency, caching

- **"Must users see their own posts immediately?"**
  - *Read-your-writes consistency*: Yes
  - *Implications*: Write to cache synchronously

**Availability Requirements**
- **"What's the acceptable downtime?"**
  - *Target*: 99.99% (critical service)
  - *Implications*: Multi-region, redundancy

#### Design Implications

**Feed Generation Strategy**

**Option 1: Fanout on Write (Push)**
- *How it works*: When user posts, write to all followers' feeds
- *Pro*: Fast read (feed pre-computed)
- *Con*: Slow write (if user has millions of followers), write amplification
- *Use case*: Users with few followers

**Option 2: Fanout on Read (Pull)**
- *How it works*: When user loads feed, query posts from all followed users
- *Pro*: Fast write
- *Con*: Slow read (must query many users)
- *Use case*: Users with many followers (celebrities)

**Option 3: Hybrid**
- *How it works*: Fanout on write for normal users, fanout on read for celebrities
- *Pro*: Best of both worlds
- *Con*: Complex
- *Use case*: Real-world systems (Twitter, Facebook)

**Ranking Algorithm**
- *Chronological*: Simple, but may miss important posts
- *Algorithmic*: Optimize for engagement (likes, comments, shares)
  - *Factors*: Recency, author relationship, post type, user interests
  - *Trade-off*: Complexity vs engagement

**Database Design**

**Users Table**
```sql
user_id (PK), username, name, bio, created_at
```

**Posts Table**
```sql
post_id (PK), user_id (FK), content, media_url, created_at
```

**Follows Table**
```sql
follower_id (FK), followee_id (FK), created_at
PRIMARY KEY (follower_id, followee_id)
```

**Feeds Table (Fanout on Write)**
```sql
user_id (FK), post_id (FK), created_at
PRIMARY KEY (user_id, created_at, post_id)
```
- *Sorted by created_at for pagination*
- *Stored in NoSQL (Cassandra, DynamoDB) for scale*

**Likes Table**
```sql
user_id (FK), post_id (FK), created_at
PRIMARY KEY (user_id, post_id)
```

**Architecture**

```
User → API Gateway → Feed Service → Feed Cache (Redis) → Feed DB (Cassandra)
                          ↓
                    Post Service → Post DB (PostgreSQL)
                          ↓
                   Fanout Service (Kafka) → Feed DB (write to followers' feeds)
```

**Posting Flow (Fanout on Write)**
1. User creates post
2. Post service writes to Post DB
3. Post service publishes event to Kafka
4. Fanout service consumes event
5. Fanout service queries Follows DB for followers
6. Fanout service writes post to each follower's feed (in Feed DB)
7. Fanout service invalidates feed cache for each follower

**Feed Loading Flow**
1. User requests feed
2. Feed service checks cache (Redis)
3. If cache hit, return cached feed
4. If cache miss, query Feed DB (sorted by created_at, paginated)
5. Enrich posts (fetch user info, like counts, etc.)
6. Cache feed
7. Return to user

**Trade-offs to Discuss**
- *Fanout on write vs read*: Write amplification vs read latency
- *Chronological vs algorithmic*: Simplicity vs engagement
- *Real-time updates*: WebSocket (complex) vs polling (simple)
- *Storage*: SQL (complex queries) vs NoSQL (scale)

**Edge Cases to Consider**
- *Celebrity posts*: Millions of followers → fanout on write too slow → use fanout on read
- *Deleted posts*: Remove from all feeds (expensive) or mark as deleted (lazy deletion)
- *Unfollowing*: Remove posts from feed or leave (lazy cleanup)
- *Pagination*: Cursor-based (stable) vs offset-based (simple but inconsistent)

---

### Scenario 5: Design a Payment System

#### Initial Clarification

**Functional Requirements**
- **"What payment methods?"**
  - *Options*: Credit card, debit card, bank transfer, digital wallet, cryptocurrency
  - *Follow-up*: "Do we process payments or use third-party (Stripe, PayPal)?"

- **"What's the payment flow?"**
  - *Options*: One-time payment, recurring subscription, marketplace (split payments)
  - *Follow-up*: "Do we need refunds, chargebacks, disputes?"

- **"What currencies?"**
  - *Example*: Single currency or multi-currency
  - *Implications*: Currency conversion, exchange rates

**Scale Requirements**
- **"How many transactions per day?"**
  - *Example*: 10M transactions/day
  - *Implications*: Throughput requirements

- **"What's the average transaction value?"**
  - *Example*: $50
  - *Implications*: Total money flow, fraud risk

- **"What's the peak QPS?"**
  - *Example*: 10M/day, peak 10x = ~1,200 TPS
  - *Implications*: Capacity planning

**Latency Requirements**
- **"What's the acceptable payment latency?"**
  - *Target*: <3 seconds (users expect fast checkout)
  - *Implications*: Async processing where possible

**Consistency Requirements**
- **"What happens if payment succeeds but order fails?"**
  - *Cost*: User charged but no order (unacceptable)
  - *Implications*: Need distributed transactions (2PC or Saga)

- **"Can we process duplicate payments?"**
  - *Cost*: User charged twice (unacceptable)
  - *Implications*: Idempotency required

- **"What's the acceptable data loss?"**
  - *Answer*: Zero (financial data)
  - *Implications*: Synchronous replication, strong consistency

**Security Requirements**
- **"What compliance requirements?"**
  - *PCI-DSS*: If storing card data
  - *Implications*: Encryption, tokenization, audit logging

- **"How do we prevent fraud?"**
  - *Strategies*: Velocity checks, anomaly detection, 3D Secure
  - *Implications*: Fraud detection service

**Availability Requirements**
- **"What's the acceptable downtime?"**
  - *Target*: 99.99% (financial system)
  - *Implications*: Multi-region, redundancy

- **"What happens during payment gateway outage?"**
  - *Options*: Retry, fallback to another gateway, queue
  - *Implications*: Resilience strategy

#### Design Implications

**Idempotency**
- *Problem*: Network failures may cause duplicate requests
- *Solution*: Idempotency key (client-generated UUID)
  - Store key in DB before processing
  - If key exists, return cached result
  - Prevents duplicate charges

**Distributed Transactions**
- *Problem*: Payment and order must succeed/fail together
- *Options*:
  - **Two-Phase Commit (2PC)**: Strong consistency, but blocking
  - **Saga Pattern**: Eventual consistency with compensating transactions
    - If payment succeeds but order fails, refund payment
  - *Trade-off*: Consistency vs availability

**Payment Flow (Saga Pattern)**
1. User initiates checkout
2. Reserve inventory (local transaction)
3. Charge payment (call payment gateway)
4. If payment succeeds, create order (local transaction)
5. If order fails, refund payment (compensating transaction)
6. Release inventory if payment or order fails

**Database Design**

**Users Table**
```sql
user_id (PK), name, email, created_at
```

**Payment Methods Table**
```sql
payment_method_id (PK), user_id (FK), type (card/bank), token (from payment gateway), created_at
```
- *Never store raw card numbers* (PCI-DSS compliance)
- *Use tokenization* (payment gateway returns token)

**Transactions Table**
```sql
transaction_id (PK), user_id (FK), amount, currency, status (pending/success/failed/refunded), idempotency_key (unique), created_at, updated_at
```

**Orders Table**
```sql
order_id (PK), user_id (FK), transaction_id (FK), items, total, status, created_at
```

**Architecture**

```
User → API Gateway → Order Service → Payment Service → Payment Gateway (Stripe/PayPal)
                          ↓                  ↓
                      Order DB          Transaction DB
                          ↓                  ↓
                    Inventory Service   Fraud Detection Service
```

**Trade-offs to Discuss**
- *2PC vs Saga*: Strong consistency vs availability
- *Sync vs async*: Immediate feedback vs higher throughput
- *Build vs buy*: Custom payment processing vs third-party (Stripe)
- *Fraud detection*: Real-time (latency) vs batch (delayed detection)

**Edge Cases to Consider**
- *Partial refunds*: User returns some items
- *Currency conversion*: Exchange rate changes between order and payment
- *Payment gateway timeout*: Unknown payment status (query gateway)
- *Concurrent payments*: User clicks "Pay" multiple times (idempotency)
- *Chargebacks*: User disputes payment (manual resolution)

---

### Scenario 6: Design a Video Streaming Service (e.g., YouTube, Netflix)

#### Initial Clarification

**Functional Requirements**
- **"What's the core functionality?"**
  - *Upload*: Users upload videos
  - *Streaming*: Users watch videos
  - *Follow-up*: "Do we need live streaming or just VOD (video on demand)?"
  - *Follow-up*: "Do we need recommendations, search, comments?"

- **"What video quality?"**
  - *Options*: SD (480p), HD (720p/1080p), 4K
  - *Follow-up*: "Adaptive bitrate streaming (adjust quality based on bandwidth)?"

**Scale Requirements**
- **"How many users?"**
  - *Example*: 1B users, 100M DAU
  - *Implications*: Massive scale

- **"How many videos uploaded per day?"**
  - *Example*: 500K videos/day
  - *Implications*: Storage, transcoding capacity

- **"How many video views per day?"**
  - *Example*: 5B views/day
  - *Implications*: Bandwidth, CDN costs

- **"What's the average video size?"**
  - *Example*: 100MB (10 min video at 720p)
  - *Implications*: Storage (500K × 100MB = 50TB/day)

**Latency Requirements**
- **"What's the acceptable video start time?"**
  - *Target*: <2 seconds (users expect instant playback)
  - *Implications*: CDN, adaptive streaming

- **"What's the acceptable buffering?"**
  - *Target*: <1% of playback time
  - *Implications*: CDN coverage, bandwidth

**Consistency Requirements**
- **"Can we show stale video metadata (views, likes)?"**
  - *Tolerance*: Yes, eventual consistency acceptable
  - *Implications*: Caching, async updates

- **"Must video be available immediately after upload?"**
  - *Answer*: No, transcoding takes time
  - *Implications*: Async processing

**Storage Requirements**
- **"What's the retention policy?"**
  - *Example*: Forever (YouTube), or delete unpopular videos
  - *Implications*: Storage costs

- **"Do we need multiple resolutions?"**
  - *Answer*: Yes (adaptive streaming)
  - *Implications*: Storage multiplier (3-5x for different resolutions)

**Availability Requirements**
- **"What's the acceptable downtime?"**
  - *Target*: 99.99%
  - *Implications*: Multi-region, CDN

#### Design Implications

**Video Upload Flow**
1. User uploads video to nearest server
2. Server stores raw video in object storage (S3)
3. Server publishes event to message queue (Kafka)
4. Transcoding service consumes event
5. Transcoding service transcodes video to multiple resolutions (480p, 720p, 1080p)
6. Transcoding service stores transcoded videos in object storage
7. Transcoding service updates video metadata (status = ready)
8. CDN pulls videos from object storage (lazy loading or pre-warming)

**Video Streaming**
- *Protocol*: HLS (HTTP Live Streaming) or DASH (Dynamic Adaptive Streaming over HTTP)
  - Video split into small chunks (2-10 seconds)
  - Client requests chunks based on bandwidth
  - Adaptive bitrate: Switch resolution mid-stream
- *CDN*: Serve videos from edge locations close to users
  - Reduces latency
  - Reduces origin server load
  - Reduces bandwidth costs

**Database Design**

**Users Table**
```sql
user_id (PK), username, name, email, created_at
```

**Videos Table**
```sql
video_id (PK), user_id (FK), title, description, duration, status (processing/ready), upload_date, view_count, like_count
```

**Video Files Table**
```sql
video_id (FK), resolution (480p/720p/1080p), file_url (S3), file_size
PRIMARY KEY (video_id, resolution)
```

**Views Table (Analytics)**
```sql
video_id, user_id, timestamp, watch_duration
```
- *Storage*: Time-series DB or data warehouse (BigQuery, Redshift)

**Architecture**

```
User → CDN (video delivery) → Origin (S3)

User → API Gateway → Video Service → Video DB
                          ↓
                    Upload Service → S3 (raw video)
                          ↓
                    Message Queue (Kafka)
                          ↓
                    Transcoding Service → S3 (transcoded videos)
                          ↓
                    Metadata Service → Video DB (update status)
```

**Trade-offs to Discuss**
- *Storage*: Multiple resolutions (better UX) vs storage cost
- *CDN*: Global coverage (low latency) vs cost
- *Transcoding*: Real-time (fast) vs batch (cheaper)
- *Recommendations*: Real-time (personalized) vs batch (stale)

**Edge Cases to Consider**
- *Upload failures*: Resume upload (chunked upload)
- *Transcoding failures*: Retry with exponential backoff
- *Copyright detection*: Scan videos for copyrighted content
- *Live streaming*: Different architecture (low latency, no transcoding)
- *Bandwidth throttling*: Limit upload/download speed for free users

---

### Scenario 7: Design a Distributed Key-Value Store (e.g., DynamoDB, Cassandra)

#### Initial Clarification

**Functional Requirements**
- **"What operations?"**
  - *Core*: GET, PUT, DELETE
  - *Advanced*: Range queries, secondary indexes, transactions

- **"What consistency model?"**
  - *Options*: Strong consistency, eventual consistency, tunable consistency
  - *Follow-up*: "Can users choose per-request?"

**Scale Requirements**
- **"How many keys?"**
  - *Example*: 1T (trillion) keys
  - *Implications*: Sharding required

- **"What's the average value size?"**
  - *Example*: 1KB
  - *Implications*: Storage (1T × 1KB = 1PB)

- **"What's the expected QPS?"**
  - *Example*: 10M QPS
  - *Implications*: Distributed system required

- **"What's the read-to-write ratio?"**
  - *Example*: 10:1
  - *Implications*: Optimize for reads (caching, read replicas)

**Latency Requirements**
- **"What's the acceptable latency?"**
  - *Target*: <10ms p99
  - *Implications*: In-memory caching, efficient data structures

**Consistency Requirements**
- **"Strong consistency or eventual consistency?"**
  - *Strong*: All reads see latest write (slower, less available)
  - *Eventual*: Reads may be stale (faster, more available)
  - *Trade-off*: CAP theorem (choose 2 of 3: Consistency, Availability, Partition tolerance)

**Availability Requirements**
- **"What's the target availability?"**
  - *Target*: 99.99%
  - *Implications*: Replication, multi-region

- **"What happens during network partition?"**
  - *CP system*: Reject writes (consistent but not available)
  - *AP system*: Accept writes (available but not consistent)
  - *Design impact*: CAP theorem choice

#### Design Implications

**Partitioning (Sharding)**
- *Problem*: Single server can't handle all data
- *Solution*: Partition data across multiple servers
- *Strategy*: Consistent hashing
  - Hash keys onto ring
  - Each server owns range of hash values
  - Adding/removing servers only affects neighbors (minimal data movement)

**Replication**
- *Purpose*: Fault tolerance, high availability
- *Strategy*: Each key replicated to N servers (N=3 typical)
  - Primary-replica: One primary, N-1 replicas
  - Multi-master: All replicas can accept writes

**Consistency Models**

**Strong Consistency (Quorum)**
- *Write*: Write to W replicas, wait for acknowledgment
- *Read*: Read from R replicas, return latest
- *Guarantee*: If W + R > N, guaranteed to see latest write
- *Example*: N=3, W=2, R=2 → strong consistency
- *Trade-off*: Higher latency, lower availability

**Eventual Consistency**
- *Write*: Write to one replica, replicate async
- *Read*: Read from one replica (may be stale)
- *Guarantee*: All replicas converge eventually
- *Trade-off*: Lower latency, higher availability, but stale reads

**Conflict Resolution**
- *Problem*: Concurrent writes to same key (in multi-master or eventual consistency)
- *Strategies*:
  - **Last-write-wins (LWW)**: Use timestamp, keep latest (simple, but loses data)
  - **Version vectors**: Track causality, detect conflicts (complex, but preserves data)
  - **Application-level resolution**: App decides how to merge (flexible, but complex)

**Database Design**

**Data Structure**
- *In-memory*: Hash table (key → value)
- *On-disk*: LSM tree (Log-Structured Merge tree)
  - Writes go to in-memory buffer (memtable)
  - When full, flush to disk (SSTable)
  - Reads check memtable, then SSTables (may need to merge)
  - Periodic compaction to merge SSTables
  - *Pro*: Fast writes (sequential I/O)
  - *Con*: Slower reads (may need to check multiple SSTables)

**Architecture**

```
Client → Load Balancer → Coordinator Node → Storage Nodes (sharded)
                              ↓
                        Consistent Hashing Ring
```

**Write Flow (Quorum)**
1. Client sends PUT(key, value)
2. Coordinator hashes key to determine storage nodes (N=3)
3. Coordinator sends write to all N nodes
4. Coordinator waits for W=2 acknowledgments
5. Coordinator returns success to client
6. Remaining node(s) replicate async

**Read Flow (Quorum)**
1. Client sends GET(key)
2. Coordinator hashes key to determine storage nodes (N=3)
3. Coordinator sends read to R=2 nodes
4. Coordinator compares versions, returns latest
5. If versions differ, coordinator triggers read repair (update stale replicas)

**Trade-offs to Discuss**
- *Consistency vs availability*: Strong consistency (CP) vs eventual consistency (AP)
- *Replication factor*: Higher N (more fault tolerance) vs cost
- *Quorum settings*: W+R>N (strong consistency) vs W+R≤N (eventual consistency)
- *Conflict resolution*: LWW (simple) vs version vectors (complex)

**Edge Cases to Consider**
- *Network partition*: Split-brain (multiple primaries)
- *Node failure*: Hinted handoff (temporary node stores writes for failed node)
- *Node recovery*: Anti-entropy (sync with other replicas)
- *Hot keys*: Some keys accessed much more than others (need caching or replication)
- *Range queries*: Consistent hashing doesn't support range queries (need secondary index)

---

### Scenario 8: Design a Real-Time Leaderboard (e.g., Gaming, Sports)

#### Initial Clarification

**Functional Requirements**
- **"What's the leaderboard scope?"**
  - *Options*: Global, regional, per-game, per-tournament
  - *Follow-up*: "Do we need multiple leaderboards (daily, weekly, all-time)?"

- **"What operations?"**
  - *Core*: Update score, get rank, get top N, get user's rank
  - *Follow-up*: "Do we need percentile (e.g., top 10%)?"

**Scale Requirements**
- **"How many users?"**
  - *Example*: 100M users
  - *Implications*: Storage, indexing

- **"How many score updates per second?"**
  - *Example*: 100K updates/sec
  - *Implications*: Write-heavy system

- **"How many leaderboard queries per second?"**
  - *Example*: 1M queries/sec
  - *Implications*: Read-heavy, needs caching

**Latency Requirements**
- **"What's the acceptable update latency?"**
  - *Target*: <100ms (users expect immediate feedback)
  - *Implications*: In-memory data structure

- **"What's the acceptable query latency?"**
  - *Target*: <50ms
  - *Implications*: Caching, efficient data structure

**Consistency Requirements**
- **"Can leaderboard be stale?"**
  - *Tolerance*: Few seconds staleness acceptable
  - *Implications*: Eventual consistency, caching

- **"What happens if two users have same score?"**
  - *Tie-breaking*: Timestamp (who scored first), user ID
  - *Implications*: Need secondary sort key

**Availability Requirements**
- **"What's the acceptable downtime?"**
  - *Target*: 99.9%
  - *Implications*: Replication, failover

#### Design Implications

**Data Structure**

**Option 1: Sorted Set (Redis ZSET)**
- *How it works*: Sorted set with score as sort key
- *Operations*:
  - `ZADD leaderboard score user_id` → O(log N) update
  - `ZREVRANGE leaderboard 0 9` → O(log N + M) get top 10
  - `ZREVRANK leaderboard user_id` → O(log N) get rank
- *Pro*: Simple, efficient, built-in
- *Con*: Single Redis instance limits scale (need sharding)

**Option 2: Skip List**
- *How it works*: Probabilistic data structure, multiple levels of linked lists
- *Operations*: O(log N) insert, delete, search
- *Pro*: Efficient, supports range queries
- *Con*: More complex than sorted set

**Option 3: B-tree**
- *How it works*: Balanced tree, sorted by score
- *Operations*: O(log N) insert, delete, search
- *Pro*: Disk-friendly (used in databases)
- *Con*: Slower than in-memory structures

**Sharding Strategy**

**Problem**: Single Redis instance can't handle 100M users

**Option 1: Shard by User ID**
- *How it works*: Hash user_id to determine shard
- *Pro*: Even distribution
- *Con*: Can't get global top N efficiently (need to query all shards and merge)

**Option 2: Shard by Score Range**
- *How it works*: Shard 1 has scores 0-1000, Shard 2 has 1000-2000, etc.
- *Pro*: Can get top N from single shard (top scores in highest shard)
- *Con*: Uneven distribution (most users in low score shards)

**Option 3: Hybrid (Approximate Leaderboard)**
- *How it works*: 
  - Shard by user ID for writes
  - Periodically aggregate top N from each shard
  - Cache aggregated top N
- *Pro*: Scales writes, fast reads
- *Con*: Approximate (not real-time)

**Database Design**

**Scores Table**
```sql
user_id (PK), score, timestamp
```

**Leaderboard Cache (Redis)**
```
ZSET leaderboard: (user_id, score)
```

**Architecture**

```
User → API Gateway → Leaderboard Service → Redis (sorted set)
                          ↓
                    Score DB (PostgreSQL, for persistence)
```

**Update Flow**
1. User completes game, earns score
2. Leaderboard service updates Redis (ZADD)
3. Leaderboard service writes to Score DB (async, for persistence)

**Query Flow**
1. User requests top 10
2. Leaderboard service queries Redis (ZREVRANGE)
3. Leaderboard service enriches with user info (name, avatar)
4. Return to user

**Trade-offs to Discuss**
- *Real-time vs approximate*: Exact leaderboard (complex) vs approximate (scalable)
- *Global vs sharded*: Single leaderboard (simple) vs sharded (scalable)
- *In-memory vs disk*: Redis (fast) vs database (durable)
- *Consistency*: Real-time updates (complex) vs batch updates (simple)

**Edge Cases to Consider**
- *Tie-breaking*: Multiple users with same score
- *Cheating*: Users manipulating scores (need validation)
- *Time-based leaderboards*: Daily/weekly leaderboards (need to reset)
- *Historical data*: Archive old leaderboards
- *Pagination*: Get ranks 100-110 (not just top N)

---

*Continued in Part 3...*

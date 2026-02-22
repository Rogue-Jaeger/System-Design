# System Design Interview: Clarifying Questions - Part 3
## Communication, Trade-offs, and Practical Frameworks

---

## Communication & Interview Flow

### The Art of Asking Questions

**Principle**: Questions should demonstrate structured thinking, not just gather information.

#### Opening the Interview

**Good Opening**
```
"Before I start designing, I'd like to clarify the requirements. I'll organize my questions 
into functional requirements, scale, performance, and constraints. Does that work?"
```

**Why it works**:
- Shows structured approach
- Sets expectations
- Demonstrates senior-level thinking

**Bad Opening**
```
"So... what exactly do you want me to build?"
```

**Why it fails**:
- Passive, waiting for interviewer to lead
- No structure
- Doesn't demonstrate thinking

#### Question Sequencing Strategy

**Phase 1: Scope Definition (2-3 minutes)**
- Clarify core functionality
- Identify must-have vs nice-to-have
- Define what's in/out of scope

**Example**:
```
"Let me make sure I understand the core problem. We're building a URL shortener that:
1. Takes long URLs and generates short codes
2. Redirects short codes to original URLs

Are these the two primary functions, or is there more in scope like analytics or custom URLs?"
```

**Phase 2: Scale & Performance (3-4 minutes)**
- Quantify traffic (QPS, users, data volume)
- Understand growth trajectory
- Identify latency requirements
- Determine read/write ratio

**Example**:
```
"Now let me understand the scale:
- How many URLs do we shorten per day?
- How many redirects per day?
- What's the expected latency for redirects?
- Is this a read-heavy or write-heavy system?"
```

**Phase 3: Constraints & Trade-offs (2-3 minutes)**
- Consistency requirements
- Availability targets
- Cost constraints
- Existing infrastructure

**Example**:
```
"A few more questions about constraints:
- If we generate duplicate short codes, what's the impact?
- Can we show stale redirects, or must they be immediately consistent?
- What's the acceptable downtime?
- Are we building on existing infrastructure or greenfield?"
```

**Phase 4: Validation (1 minute)**
- Summarize understanding
- Confirm assumptions
- Get buy-in before designing

**Example**:
```
"Let me summarize what I've understood:
- 100M URLs shortened per day, 10B redirects per day (100:1 read-heavy)
- Target latency: <50ms p99 for redirects
- Eventual consistency acceptable for analytics
- High availability required (99.99%)

Does that sound right? Anything I'm missing?"
```

### Communication Patterns

#### Pattern 1: Think Aloud

**Good**:
```
"I'm thinking about the short code generation. We have a few options:
1. Hash the URL - simple, but risk of collisions
2. Use a counter - no collisions, but need distributed coordination
3. Random generation - simple, but collision probability increases with scale

Given our scale of 100M URLs per day, I'm leaning towards option 2 with a distributed 
counter service. The coordination overhead is worth the guarantee of no collisions."
```

**Why it works**:
- Shows reasoning process
- Considers multiple options
- Explains trade-offs
- Makes a decision with justification

**Bad**:
```
"I'll use a counter for short codes."
```

**Why it fails**:
- No reasoning shown
- No alternatives considered
- No trade-off analysis

#### Pattern 2: Acknowledge Uncertainty

**Good**:
```
"I'm not certain about the exact collision probability for 6-character codes at this scale. 
Let me do a quick calculation: 62^6 = 56 billion possible codes. With 100M URLs per day, 
we'd use about 36 billion codes in a year. By the birthday paradox, collision probability 
becomes significant around 50% capacity. So we should be safe for several years, but we 
should monitor and potentially increase code length if needed."
```

**Why it works**:
- Honest about uncertainty
- Works through the problem
- Arrives at reasonable answer
- Identifies monitoring need

**Bad**:
```
"I don't know the collision probability."
```

**Why it fails**:
- Gives up without trying
- Doesn't demonstrate problem-solving

#### Pattern 3: Propose and Validate

**Good**:
```
"I'm thinking we should use Redis for caching the short code to URL mappings. Given the 
100:1 read-to-write ratio, caching will significantly reduce database load. We can use 
a TTL of 24 hours since URLs rarely change. Does that align with your expectations, or 
are there concerns about stale data I should consider?"
```

**Why it works**:
- Makes a proposal
- Explains reasoning
- Invites feedback
- Shows collaborative approach

**Bad**:
```
"We'll use Redis for caching."
```

**Why it fails**:
- No explanation
- No invitation for feedback
- Doesn't demonstrate thinking

#### Pattern 4: Quantify Everything

**Good**:
```
"Let me calculate the storage requirements:
- 100M URLs per day
- Average URL length: 200 characters = 200 bytes
- Short code: 6 characters = 6 bytes
- Metadata (timestamps, etc.): ~50 bytes
- Total per URL: ~256 bytes

Daily storage: 100M × 256 bytes = 25.6 GB
Annual storage: 25.6 GB × 365 = 9.3 TB
5-year storage: 9.3 TB × 5 = 46.5 TB

So we need to plan for ~50 TB of storage over 5 years, plus replication factor."
```

**Why it works**:
- Shows quantitative thinking
- Breaks down calculation
- Considers time horizon
- Accounts for replication

**Bad**:
```
"We'll need a lot of storage."
```

**Why it fails**:
- Vague, not quantified
- Doesn't demonstrate calculation skills

### Handling Pushback

#### Scenario: Interviewer Challenges Your Choice

**Interviewer**: "Why not use a hash function instead of a counter for short codes?"

**Good Response**:
```
"That's a great point. Let me reconsider. Hash functions have the advantage of being 
stateless - no coordination needed. The downside is collision risk. Let me think about 
the trade-offs:

Hash approach:
- Pro: No coordination overhead, simpler architecture
- Con: Must handle collisions (check DB, retry with different hash)
- Con: Collision probability increases with scale

Counter approach:
- Pro: Guaranteed no collisions
- Con: Need distributed counter (coordination overhead)
- Con: Single point of failure if counter service goes down

Actually, you're right. For this scale, the hash approach with collision handling might 
be simpler. We can use MD5 hash, take first 6 characters, and if collision occurs (rare), 
append a salt and rehash. The collision handling adds a bit of latency, but it's rare 
enough that it won't impact p99 latency significantly.

Does that sound reasonable?"
```

**Why it works**:
- Acknowledges the point
- Reconsiders thoughtfully
- Compares trade-offs
- Changes position with reasoning
- Invites further discussion

**Bad Response**:
```
"I think counter is better because it has no collisions."
```

**Why it fails**:
- Defensive, not open to feedback
- Doesn't reconsider
- Doesn't engage with the challenge

#### Scenario: Interviewer Says "I Don't Know"

**You**: "What's the acceptable latency for redirects?"

**Interviewer**: "I don't know. What do you think?"

**Good Response**:
```
"Okay, let me reason through this. For a URL shortener, users expect instant redirects - 
similar to clicking any link. Based on typical web performance standards:
- <100ms feels instant
- 100-300ms is noticeable but acceptable
- >300ms feels slow

I'd propose targeting <100ms p99 latency for redirects. This is aggressive but achievable 
with caching. If that's too aggressive, we can relax to <200ms, which would simplify the 
architecture (less aggressive caching needed).

Does <100ms p99 sound reasonable for this use case?"
```

**Why it works**:
- Takes ownership
- Provides reasoning
- Gives options
- Proposes a specific target
- Validates with interviewer

**Bad Response**:
```
"Um, I'm not sure either. Maybe 1 second?"
```

**Why it fails**:
- Doesn't take ownership
- No reasoning
- Arbitrary number

### Time Management

**Typical 45-minute Interview Breakdown**:

- **0-10 minutes**: Clarifying questions
- **10-30 minutes**: High-level design, deep dives
- **30-40 minutes**: Trade-offs, edge cases, scaling
- **40-45 minutes**: Wrap-up, questions

**Red Flags**:
- Spending >15 minutes on clarifying questions (too much)
- Spending <5 minutes on clarifying questions (too little)
- Not leaving time for trade-offs and scaling discussion

**Time Management Tips**:
1. **Set a timer mentally**: "I'll spend ~10 minutes on questions"
2. **Prioritize**: Ask most important questions first
3. **Batch questions**: Group related questions together
4. **Signal transitions**: "I think I have enough context. Let me start designing."

---

## Trade-off Analysis Patterns

### Framework: Structured Trade-off Analysis

For every major design decision, articulate:
1. **Options** (at least 2-3 alternatives)
2. **Pros/Cons** of each
3. **Context** (what makes one better than another)
4. **Decision** with justification
5. **Monitoring** (how to validate the decision)

### Common Trade-off Categories

#### Trade-off 1: Consistency vs Availability (CAP Theorem)

**Scenario**: Distributed database, network partition occurs

**Option A: Choose Consistency (CP)**
- **Pro**: All reads see latest write, no stale data
- **Con**: System becomes unavailable during partition
- **When to choose**: Financial transactions, inventory management
- **Example**: Bank balance must be consistent, even if it means rejecting requests

**Option B: Choose Availability (AP)**
- **Pro**: System remains available during partition
- **Con**: Reads may return stale data, must handle conflicts
- **When to choose**: Social media, analytics, non-critical data
- **Example**: Social media likes can be eventually consistent

**How to articulate**:
```
"We need to make a CAP theorem choice. Given that this is a social media feed, I'd 
prioritize availability over consistency. Users expect the app to always work, and 
seeing a like count that's off by a few is acceptable. We'll use eventual consistency 
with conflict resolution via last-write-wins. If this were a payment system, I'd make 
the opposite choice."
```

#### Trade-off 2: Latency vs Consistency

**Scenario**: Caching strategy

**Option A: Aggressive Caching (Low Latency)**
- **Pro**: <10ms p99 latency
- **Con**: Stale data (cache may be seconds/minutes behind)
- **When to choose**: Read-heavy, data changes infrequently
- **Example**: Product catalog (prices don't change often)

**Option B: Read-Through Cache (Balanced)**
- **Pro**: Moderate latency (~50ms), fresher data
- **Con**: More complex, cache invalidation needed
- **When to choose**: Data changes moderately
- **Example**: User profiles (updated occasionally)

**Option C: No Cache (Strong Consistency)**
- **Pro**: Always fresh data
- **Con**: High latency (>100ms), high DB load
- **When to choose**: Critical data that must be fresh
- **Example**: Bank balance

**How to articulate**:
```
"For the URL shortener, I'd use aggressive caching (Option A). URLs rarely change after 
creation, so stale data risk is low. We can cache short code → URL mappings in Redis 
with a 24-hour TTL. This will give us <10ms p99 latency and reduce DB load by 99%. 
The trade-off is if a URL is updated, users might see the old URL for up to 24 hours. 
We can mitigate this by invalidating cache on URL updates."
```

#### Trade-off 3: Normalization vs Denormalization

**Scenario**: Database schema design

**Option A: Normalized (3NF)**
- **Pro**: No data duplication, easy to update
- **Con**: Requires joins (slower reads), complex queries
- **When to choose**: Write-heavy, data changes frequently
- **Example**: Inventory system (stock levels change constantly)

**Option B: Denormalized**
- **Pro**: Fast reads (no joins), simple queries
- **Con**: Data duplication, complex updates (must update multiple places)
- **When to choose**: Read-heavy, data rarely changes
- **Example**: Social media feed (user profile duplicated with each post)

**How to articulate**:
```
"For the social media feed, I'd denormalize. We'll store user info (name, avatar) with 
each post. This means when loading a feed, we don't need to join with the users table - 
just read posts directly. The trade-off is if a user changes their name, we need to 
update all their posts. But profile changes are rare (maybe once a month), while feed 
loads happen millions of times per day. So the read optimization is worth the write 
complexity."
```

#### Trade-off 4: Synchronous vs Asynchronous Processing

**Scenario**: Processing user requests

**Option A: Synchronous**
- **Pro**: Immediate feedback, simpler to reason about
- **Con**: Higher latency, blocks user
- **When to choose**: User needs immediate result
- **Example**: Login (user must know if login succeeded)

**Option B: Asynchronous**
- **Pro**: Lower latency, higher throughput
- **Con**: Eventual consistency, more complex (need queues, workers)
- **When to choose**: User doesn't need immediate result
- **Example**: Video transcoding (user can wait)

**How to articulate**:
```
"For video upload, I'd use asynchronous processing. When a user uploads a video, we 
immediately return success and store the raw video in S3. Then we publish an event to 
Kafka, and transcoding workers process it in the background. This gives the user fast 
feedback (<1 second) instead of waiting for transcoding (minutes). The trade-off is the 
video isn't immediately available - it's in 'processing' state. We'll show a progress 
indicator to set expectations."
```

#### Trade-off 5: Build vs Buy

**Scenario**: Technology selection

**Option A: Build Custom**
- **Pro**: Full control, optimized for use case
- **Con**: High engineering cost, ongoing maintenance
- **When to choose**: Unique requirements, large team
- **Example**: Google building Bigtable (no existing solution met needs)

**Option B: Use Managed Service**
- **Pro**: Low engineering cost, battle-tested, managed operations
- **Con**: Less control, vendor lock-in, potentially higher infra cost
- **When to choose**: Standard requirements, small team
- **Example**: Using AWS RDS instead of self-hosting PostgreSQL

**How to articulate**:
```
"For the payment system, I'd use a third-party payment processor like Stripe rather than 
building custom. Here's why:
- PCI compliance is complex and expensive (audits, certifications)
- Payment processing has many edge cases (chargebacks, fraud, international)
- Stripe has solved these problems and is battle-tested
- Our team of 5 engineers should focus on core product, not payments

The trade-off is we pay Stripe 2.9% + $0.30 per transaction, and we're locked into their 
API. But the engineering cost savings and reduced risk are worth it. If we were processing 
billions in transactions, we might reconsider and build custom."
```

#### Trade-off 6: Monolith vs Microservices

**Scenario**: System architecture

**Option A: Monolith**
- **Pro**: Simple deployment, easy to develop, no network overhead
- **Con**: Hard to scale independently, tight coupling
- **When to choose**: Small team, early stage, unclear boundaries
- **Example**: Startup with 3 engineers

**Option B: Microservices**
- **Pro**: Independent scaling, team autonomy, fault isolation
- **Con**: Complex deployment, network overhead, distributed system challenges
- **When to choose**: Large team, clear boundaries, different scaling needs
- **Example**: Netflix with hundreds of services

**How to articulate**:
```
"For this ride-sharing system, I'd start with a modular monolith and plan for future 
microservices. Here's the reasoning:

Initial monolith with clear module boundaries:
- Ride module (booking, matching)
- Driver module (location tracking)
- User module (profiles, auth)
- Payment module (transactions)

This gives us:
- Fast development (no network calls, shared DB)
- Clear boundaries (can extract to microservices later)
- Simple deployment (one service)

As we scale, we can extract modules to microservices:
- Location tracking → separate service (high write load, different scaling needs)
- Payment → separate service (PCI compliance, fault isolation)

The trade-off is some refactoring later, but we avoid premature complexity."
```

---

## Quantification & Estimation Frameworks

### Back-of-the-Envelope Calculations

**Key Numbers to Memorize**:

**Time**:
- 1 second = 1,000 milliseconds (ms)
- 1 ms = 1,000 microseconds (μs)
- 1 day = 86,400 seconds (~100K seconds for estimation)

**Data**:
- 1 KB = 1,000 bytes (technically 1,024, but use 1,000 for estimation)
- 1 MB = 1,000 KB
- 1 GB = 1,000 MB
- 1 TB = 1,000 GB
- 1 PB = 1,000 TB

**Latency**:
- L1 cache: 0.5 ns
- L2 cache: 7 ns
- RAM: 100 ns
- SSD: 150 μs (0.15 ms)
- HDD: 10 ms
- Network within datacenter: 0.5 ms
- Network cross-region: 50-100 ms
- Network cross-continent: 150-300 ms

**Throughput**:
- Single server: ~10K QPS (simple reads)
- Database: ~1K QPS (complex queries)
- Redis: ~100K QPS (simple gets)
- Network: 1 Gbps = 125 MB/s

### Estimation Framework

#### Step 1: Clarify the Question

**Example**: "How much storage do we need?"

**Clarify**:
- For how long? (1 year, 5 years, forever?)
- Including replication? (3x for redundancy?)
- Including backups? (additional 2x?)

#### Step 2: Break Down the Problem

**Example**: Storage for URL shortener

**Break down**:
1. How many URLs per day?
2. How much data per URL?
3. How many days to store?
4. Replication factor?

#### Step 3: Make Reasonable Assumptions

**Example**:
- 100M URLs per day (given)
- 200 bytes per URL (assume average URL length)
- 5 years retention (assume)
- 3x replication (standard)

#### Step 4: Calculate

```
Daily storage:
100M URLs × 200 bytes = 20 GB

Annual storage:
20 GB × 365 days = 7.3 TB

5-year storage:
7.3 TB × 5 = 36.5 TB

With replication (3x):
36.5 TB × 3 = ~110 TB
```

#### Step 5: Sanity Check

**Ask yourself**:
- Does this number make sense?
- Is it too high or too low?
- What if I'm off by 2x? Does it change the design?

**Example**:
"110 TB over 5 years seems reasonable. Even if I'm off by 2x (220 TB), it's still 
manageable with modern storage. This doesn't fundamentally change the design."

### Common Estimation Patterns

#### Pattern 1: QPS Estimation

**Given**: 100M daily active users (DAU)

**Estimate QPS**:
```
Assume each user makes 20 requests per day
Total requests per day: 100M × 20 = 2B requests

Average QPS: 2B / 86,400 seconds = ~23K QPS

Peak QPS (assume 3x average): 23K × 3 = ~70K QPS
```

**Design implication**: Need to handle 70K QPS at peak

#### Pattern 2: Storage Estimation

**Given**: 10M videos uploaded per day

**Estimate storage**:
```
Assume average video size: 100 MB (10 min at 720p)
Daily storage: 10M × 100 MB = 1 PB

Annual storage: 1 PB × 365 = 365 PB

With multiple resolutions (480p, 720p, 1080p): 365 PB × 3 = ~1 EB (exabyte)
```

**Design implication**: Need petabyte-scale storage, likely object storage (S3)

#### Pattern 3: Bandwidth Estimation

**Given**: 5B video views per day

**Estimate bandwidth**:
```
Assume average video size: 50 MB (5 min at 720p)
Assume average watch time: 50% (2.5 min)
Effective data per view: 50 MB × 0.5 = 25 MB

Daily bandwidth: 5B × 25 MB = 125 PB

Bandwidth per second: 125 PB / 86,400 = ~1.4 TB/s
```

**Design implication**: Need CDN to distribute load globally

#### Pattern 4: Database Sizing

**Given**: 1B users, each with 100 posts

**Estimate database size**:
```
Users table:
1B users × 1 KB per user = 1 TB

Posts table:
1B users × 100 posts × 1 KB per post = 100 TB

Indexes (assume 50% of data size):
(1 TB + 100 TB) × 0.5 = 50.5 TB

Total: ~150 TB
```

**Design implication**: Need sharding, single database can't handle 150 TB

#### Pattern 5: Cache Sizing

**Given**: 100M daily active users, 80% of requests hit 20% of data

**Estimate cache size**:
```
Total data: 100 TB
Hot data (20%): 100 TB × 0.2 = 20 TB

If we cache hot data, we need 20 TB of cache.

But 20 TB of RAM is expensive. Let's cache even hotter data:
- 80% of requests hit 20% of data (20 TB)
- 50% of requests hit 5% of data (5 TB)
- 20% of requests hit 1% of data (1 TB)

Caching 1 TB (1% of data) would serve 20% of requests.
Caching 5 TB (5% of data) would serve 50% of requests.

Trade-off: Cost vs cache hit rate
```

**Design implication**: Start with 5 TB cache, monitor hit rate, adjust

---

## Anti-Patterns & Red Flags

### Anti-Pattern 1: Technology Name-Dropping

**Bad**:
```
"We'll use Kafka for the message queue, Redis for caching, Cassandra for the database, 
and Kubernetes for orchestration."
```

**Why it's bad**:
- No justification
- Sounds like buzzword bingo
- Doesn't demonstrate understanding

**Good**:
```
"For the message queue, we need to handle 100K messages per second with guaranteed delivery. 
Kafka is a good fit because:
- High throughput (millions of messages per second)
- Durable (persists messages to disk)
- Scalable (partitioning)

Alternatives like RabbitMQ or SQS could work, but Kafka's throughput and durability make 
it the best choice for this scale."
```

### Anti-Pattern 2: Over-Engineering

**Bad**:
```
"We'll use a microservices architecture with 20 services, each with its own database, 
deployed on Kubernetes with service mesh, API gateway, and event sourcing."
```

**Why it's bad**:
- Premature complexity
- Doesn't match scale
- Ignores team size and expertise

**Good**:
```
"Given we're at 1,000 QPS and have a team of 5 engineers, I'd start with a monolith. 
We can deploy it on a few servers behind a load balancer. This keeps things simple and 
lets us iterate quickly. As we scale to 10K+ QPS or the team grows to 20+ engineers, 
we can extract microservices for components with different scaling needs."
```

### Anti-Pattern 3: Ignoring Trade-offs

**Bad**:
```
"We'll use strong consistency because it's better."
```

**Why it's bad**:
- No acknowledgment of trade-offs
- Doesn't consider context
- Sounds naive

**Good**:
```
"We need to decide between strong consistency and eventual consistency. Strong consistency 
means all reads see the latest write, but it comes with higher latency and lower availability 
(CAP theorem). Eventual consistency is faster and more available, but reads may be stale.

For this social media feed, I'd choose eventual consistency. Users can tolerate seeing a 
like count that's off by a few, but they can't tolerate the app being down. If this were 
a payment system, I'd make the opposite choice."
```

### Anti-Pattern 4: Not Quantifying

**Bad**:
```
"We'll need a lot of servers to handle the load."
```

**Why it's bad**:
- Vague, not specific
- Doesn't demonstrate calculation skills
- Can't validate the design

**Good**:
```
"Let me calculate how many servers we need:
- Peak QPS: 70K
- Each server handles: 10K QPS
- Servers needed: 70K / 10K = 7 servers

With redundancy (3x for availability): 7 × 3 = 21 servers

So we need about 20-25 servers to handle peak load with redundancy."
```

### Anti-Pattern 5: Giving Up on Unknowns

**Bad**:
```
Interviewer: "What's the cache hit rate?"
Candidate: "I don't know."
```

**Why it's bad**:
- Doesn't attempt to reason through it
- Misses opportunity to demonstrate problem-solving

**Good**:
```
Interviewer: "What's the cache hit rate?"
Candidate: "I don't have the exact number, but let me reason through it. If we're caching 
the top 5% of data (hot data), and assuming a power law distribution where 80% of requests 
go to 20% of data, we'd expect a cache hit rate of around 60-70%. We should monitor this 
in production and adjust cache size accordingly. If hit rate is lower, we increase cache 
size. If it's higher, we might be over-provisioned."
```

### Anti-Pattern 6: Ignoring Failure Modes

**Bad**:
```
"We'll use Redis for caching. Done."
```

**Why it's bad**:
- Doesn't consider what happens when Redis fails
- No resilience strategy

**Good**:
```
"We'll use Redis for caching. If Redis goes down, we have two options:
1. Fail open: Bypass cache, query database directly (risk of DB overload)
2. Fail closed: Return error (risk of outage)

I'd choose fail open with rate limiting. We'll detect Redis failure, bypass cache, and 
rate limit database queries to prevent overload. We'll also have Redis in a primary-replica 
setup with automatic failover, so downtime should be minimal (<1 minute)."
```

---

## Real-World Constraints

### Constraint 1: Team Size & Expertise

**Small Team (2-5 engineers)**:
- **Prefer**: Managed services, monolith, simple architecture
- **Avoid**: Microservices, self-hosted infrastructure, complex systems
- **Reasoning**: Limited operational capacity, need to move fast

**Example**:
```
"Given the team size of 3 engineers, I'd use managed services wherever possible:
- AWS RDS for database (vs self-hosted PostgreSQL)
- AWS ElastiCache for caching (vs self-hosted Redis)
- AWS S3 for object storage
- Heroku or AWS Elastic Beanstalk for deployment (vs Kubernetes)

This minimizes operational burden and lets the team focus on product development."
```

**Large Team (50+ engineers)**:
- **Prefer**: Microservices, self-hosted infrastructure (if cost-effective), custom solutions
- **Avoid**: Monolith (hard to coordinate), over-reliance on managed services (expensive at scale)
- **Reasoning**: Can afford operational complexity, can optimize costs

### Constraint 2: Budget

**Tight Budget (Startup)**:
- **Prefer**: Open-source, self-hosted, spot instances, aggressive caching
- **Avoid**: Expensive managed services, over-provisioning
- **Reasoning**: Optimize for cost

**Example**:
```
"With a tight budget, I'd optimize for cost:
- Use spot instances for batch processing (90% discount)
- Self-host PostgreSQL on EC2 (vs RDS, saves 50%)
- Use aggressive caching to reduce database load (fewer DB instances needed)
- Use CloudFront CDN only for hot content (vs caching everything)

Trade-off: More operational burden, but significantly lower costs."
```

**Large Budget (Enterprise)**:
- **Prefer**: Managed services, over-provisioning for reliability, multi-region
- **Avoid**: Operational complexity, self-hosting
- **Reasoning**: Optimize for reliability and developer productivity

### Constraint 3: Time to Market

**Need to Launch Quickly (3 months)**:
- **Prefer**: Monolith, managed services, third-party APIs, MVP features only
- **Avoid**: Microservices, custom infrastructure, nice-to-have features
- **Reasoning**: Speed over perfection

**Example**:
```
"To launch in 3 months, I'd cut scope aggressively:
- MVP: Core features only (ride booking, basic matching)
- Defer: Advanced matching, surge pricing, ride sharing
- Use third-party: Stripe for payments, Twilio for SMS, Google Maps for routing
- Architecture: Monolith deployed on Heroku (fastest to production)

We can refactor later once we validate product-market fit."
```

**Long Timeline (1+ year)**:
- **Prefer**: Proper architecture, scalability planning, comprehensive features
- **Avoid**: Shortcuts that create tech debt
- **Reasoning**: Time to do it right

### Constraint 4: Existing Infrastructure

**Greenfield (New Project)**:
- **Freedom**: Choose any technology
- **Considerations**: Team expertise, cost, operational burden

**Brownfield (Existing System)**:
- **Constraints**: Must integrate with existing systems
- **Considerations**: Compatibility, migration path, backward compatibility

**Example**:
```
"Since we're integrating with an existing system that uses PostgreSQL and is deployed on 
AWS, I'd stick with that ecosystem:
- Use PostgreSQL for consistency (team knows it, existing tools work)
- Deploy on AWS (existing VPC, security groups, monitoring)
- Use existing authentication service (vs building new)

This minimizes integration complexity and leverages existing expertise."
```

---

*Continued in Part 4...*

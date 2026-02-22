# System Design Interview: Clarifying Questions - Part 4
## Case Studies and Final Summary

---

## Case Studies: Good vs Bad Approaches

### Case Study 1: Design a URL Shortener

#### Bad Interview (Candidate 1 Style)

**Interviewer**: "Design a URL shortener like bit.ly."

**Candidate**: "Okay, so we'll use Redis for caching, a database for storing URLs, and we'll hash the URLs to create short codes."

**Interviewer**: "How will you handle scale?"

**Candidate**: "We'll add more servers and use load balancing."

**Interviewer**: "What about collisions in the hash?"

**Candidate**: "We'll check if the short code exists and regenerate if there's a collision."

**Interviewer**: "How much storage do you need?"

**Candidate**: "A lot. Maybe a few terabytes."

**Why this failed**:
- No clarifying questions asked
- Vague, unquantified answers
- No trade-off analysis
- Pattern-matching (Redis, load balancing) without justification
- Doesn't demonstrate structured thinking

#### Good Interview (Candidate 2 Style)

**Interviewer**: "Design a URL shortener like bit.ly."

**Candidate**: "Great! Before I start, let me clarify the requirements. I'll organize my questions into functional requirements, scale, and constraints."

**Interviewer**: "Sounds good."

**Candidate**: "For functional requirements:
- Is the core functionality just shortening URLs and redirecting, or do we need analytics, custom URLs, or expiration?
- What's the short code format? Are 6 alphanumeric characters acceptable?"

**Interviewer**: "Core functionality only for now. 6 characters is fine."

**Candidate**: "For scale:
- How many URLs do we shorten per day?
- How many redirects per day?
- What's the expected latency for redirects?"

**Interviewer**: "Let's say 100 million URLs per day, 10 billion redirects per day. Latency should be under 100ms p99."

**Candidate**: "Got it. So that's a 100:1 read-to-write ratio, which is heavily read-optimized. And for constraints:
- Can we show stale redirects, or must they be immediately consistent?
- What's the acceptable downtime?"

**Interviewer**: "Eventual consistency is fine. We want 99.99% availability."

**Candidate**: "Perfect. Let me summarize: 100M writes/day, 10B reads/day (100:1 ratio), <100ms p99 latency, eventual consistency acceptable, 99.99% availability. Does that sound right?"

**Interviewer**: "Yes, that's correct."

**Candidate**: "Great. Let me start with capacity planning, then I'll design the system.

For storage:
- 100M URLs/day × 200 bytes/URL = 20 GB/day
- 5 years: 20 GB × 365 × 5 = 36.5 TB
- With 3x replication: ~110 TB

For QPS:
- Writes: 100M / 86,400 = ~1,200 writes/sec
- Reads: 10B / 86,400 = ~115,000 reads/sec

Given the 100:1 read-to-write ratio and <100ms latency requirement, caching is essential.

For the architecture, I'll use:
1. **Short code generation**: Base62-encoded counter
   - Why: Guaranteed no collisions, simple
   - Trade-off: Need distributed counter service, but worth it for collision-free guarantee
   
2. **Database**: PostgreSQL for URL storage
   - Schema: (short_code PK, long_url, created_at)
   - Why: ACID guarantees, familiar, sufficient for this scale
   
3. **Cache**: Redis for short_code → long_url mappings
   - Why: 115K reads/sec is too much for DB, Redis can handle it easily
   - TTL: 24 hours (URLs rarely change)
   - This will serve 99%+ of requests from cache
   
4. **CDN**: CloudFront in front of Redis
   - Why: Further reduce latency, handle geographic distribution
   - Serves static redirects from edge locations

The flow is:
- Write: User submits URL → Counter service generates short code → Write to DB → Write to cache
- Read: User clicks short URL → CDN (cache hit) → Redis (cache hit) → DB (cache miss)

For 99.99% availability, we'll have:
- Multi-AZ deployment
- Redis in primary-replica setup with automatic failover
- Database replication across AZs

Does this approach make sense? Any concerns?"

**Why this succeeded**:
- Structured clarifying questions
- Quantified everything (storage, QPS)
- Explained trade-offs for each decision
- Demonstrated deep thinking
- Invited feedback

---

### Case Study 2: Design a Rate Limiter

#### Bad Interview

**Interviewer**: "Design a rate limiter."

**Candidate**: "We'll use a token bucket algorithm and store the state in Redis."

**Interviewer**: "How do you handle distributed systems?"

**Candidate**: "We'll have all servers check Redis before allowing requests."

**Interviewer**: "What if Redis is down?"

**Candidate**: "We'll have a backup Redis."

**Why this failed**:
- No questions about requirements
- Didn't clarify what's being rate limited or the limits
- No discussion of consistency requirements
- Didn't consider trade-offs (centralized vs distributed)
- Didn't think through failure modes properly

#### Good Interview

**Interviewer**: "Design a rate limiter."

**Candidate**: "I'd like to clarify a few things first:
1. What are we rate limiting? API requests, login attempts, something else?
2. What's the rate limit? For example, 100 requests per minute per user?
3. Per what dimension? Per user, per IP, per API key?
4. What happens when the limit is exceeded? Reject, queue, or degrade?
5. How strict must the rate limit be? Can we tolerate occasional overages?"

**Interviewer**: "Let's say API requests, 100 requests per minute per user, and we reject with HTTP 429. We want it fairly strict but occasional overages are acceptable."

**Candidate**: "And for scale:
- How many requests per second total?
- How many users?"

**Interviewer**: "1 million requests per second, 100 million users."

**Candidate**: "Got it. One more question: what's the acceptable latency for the rate limit check? We don't want it to add significant overhead."

**Interviewer**: "Under 5ms would be ideal."

**Candidate**: "Perfect. So we have:
- 1M requests/sec to rate limit
- 100M users to track
- 100 requests/minute per user
- Fairly strict (occasional overages okay)
- <5ms latency for rate limit check

Given these requirements, I need to decide between centralized and distributed rate limiting.

**Option 1: Centralized (Redis)**
- Pro: Accurate, all servers see same state
- Con: Network latency (~1-5ms), Redis is single point of failure
- Con: 1M requests/sec all hitting Redis might be a bottleneck

**Option 2: Distributed (In-process)**
- Pro: Very fast (<1ms), no network calls
- Con: Inaccurate (each server has own limit), could allow 100 × num_servers requests
- Con: Not suitable for strict limits

**Option 3: Hybrid (In-process + Redis sync)**
- Pro: Fast (<1ms for most requests) + reasonably accurate
- How it works: Local counter with short TTL (e.g., 10 seconds), periodically sync to Redis
- Con: More complex

Given the requirements (fairly strict but occasional overages okay, <5ms latency), I'd use **Option 3: Hybrid**.

Here's the design:
1. Each server maintains local counters (in-memory hash map)
2. On each request, check local counter first (fast, <1ms)
3. If local counter allows, increment and allow request
4. Every 10 seconds, sync local counts to Redis (async)
5. If local counter is near limit, check Redis for accurate count (rare)

For the algorithm, I'll use **sliding window counter**:
- Why: Good balance of accuracy and memory efficiency
- How: Store count for current minute and previous minute, interpolate

For failure handling:
- If Redis is down: Fall back to local counters only (fail open)
- Why: Availability over strict enforcement
- Add monitoring to alert on Redis failures

Storage requirements:
- 100M users × 100 bytes per user (counter data) = 10 GB
- Easily fits in Redis memory

Does this approach address your requirements?"

**Why this succeeded**:
- Thorough clarifying questions
- Considered multiple options with trade-offs
- Made informed decision based on requirements
- Thought through failure modes
- Quantified storage needs

---

## Final Summary: The Clarifying Questions Checklist

### Pre-Design Checklist (Use this in every interview)

**Phase 1: Functional Requirements (2-3 minutes)**
- [ ] What are the core user actions/workflows?
- [ ] What's in scope vs out of scope?
- [ ] What does success look like for the end user?
- [ ] What data needs to be stored?
- [ ] What's the data retention policy?

**Phase 2: Scale & Performance (3-4 minutes)**
- [ ] How many users (total, DAU, MAU)?
- [ ] What's the expected QPS (average, peak)?
- [ ] What's the read-to-write ratio?
- [ ] What's the data volume (per user, total)?
- [ ] What's the geographic distribution?
- [ ] What are the latency targets (p50, p95, p99)?
- [ ] Which operations are latency-sensitive?

**Phase 3: Consistency & Reliability (2-3 minutes)**
- [ ] What consistency guarantees do we need?
- [ ] What's the cost of showing stale data?
- [ ] What operations must be atomic?
- [ ] What's the acceptable data loss (RPO)?
- [ ] What's the acceptable downtime (RTO)?
- [ ] What's the target availability (SLA)?

**Phase 4: Constraints (1-2 minutes)**
- [ ] What's the budget?
- [ ] What's the team size and expertise?
- [ ] What's the existing tech stack?
- [ ] Is this greenfield or brownfield?
- [ ] What compliance requirements apply?
- [ ] What's the timeline?

**Phase 5: Validation (1 minute)**
- [ ] Summarize understanding
- [ ] Confirm assumptions
- [ ] Get buy-in before designing

### The Mental Model: From Requirements to Design

```
Requirements → Quantification → Trade-offs → Design → Validation
```

**1. Requirements**: Ask clarifying questions (use checklist above)

**2. Quantification**: Calculate capacity needs
- Storage: data volume × retention × replication
- QPS: requests/day ÷ 86,400 × peak multiplier
- Bandwidth: data size × QPS
- Servers: QPS ÷ capacity per server

**3. Trade-offs**: For each major decision, consider:
- What are the options? (at least 2-3)
- What are the pros/cons of each?
- What does the context favor?
- What's the decision and why?

**4. Design**: Build the architecture
- Start simple (baseline)
- Layer in complexity only where needed
- Explain each component's purpose
- Draw diagrams

**5. Validation**: Check the design
- Does it meet requirements?
- What are the bottlenecks?
- What are the failure modes?
- How does it scale?

### Key Principles to Remember

**1. Quantify Everything**
- Don't say "a lot" - calculate the number
- Don't say "fast" - specify the latency target
- Don't say "many" - estimate the count

**2. Think in Trade-offs**
- Every decision has pros and cons
- Context determines the right choice
- Articulate why you chose option A over option B

**3. Start Simple, Add Complexity**
- Begin with a baseline (single server, monolith)
- Identify bottlenecks
- Add complexity only where needed (caching, sharding, etc.)

**4. Consider Failure Modes**
- What happens when X fails?
- How do we detect failures?
- How do we recover?
- What's the blast radius?

**5. Demonstrate Structured Thinking**
- Organize questions into categories
- Break down problems systematically
- Show your reasoning process
- Invite feedback

### Common Mistakes to Avoid

**❌ Don't do this**:
- Jump into design without clarifying requirements
- Pattern-match requirements to technologies (low latency → Redis)
- Give vague, unquantified answers ("we need a lot of storage")
- Ignore trade-offs ("we'll use strong consistency because it's better")
- Over-engineer for current scale (microservices for 100 QPS)
- Give up on unknowns ("I don't know")

**✅ Do this**:
- Ask structured clarifying questions
- Explain why you chose each technology (trade-offs)
- Quantify everything (storage, QPS, latency)
- Acknowledge trade-offs explicitly
- Match complexity to scale
- Reason through unknowns

### The Difference Between Levels

**Mid-Level**:
- Knows technologies
- Can implement given designs
- Focuses on "what"

**Senior**:
- Understands trade-offs
- Can design systems independently
- Focuses on "why"
- Quantifies requirements
- Considers failure modes

**Staff**:
- Models problem spaces systematically
- Considers organizational constraints
- Focuses on "when" and "how"
- Thinks in terms of evolution
- Balances technical and business concerns

### Interview Success Formula

```
Success = Structured Questions + Quantification + Trade-off Analysis + Clear Communication
```

**Structured Questions**: Use the checklist, organize by category

**Quantification**: Calculate storage, QPS, bandwidth, servers

**Trade-off Analysis**: For each decision, explain options, pros/cons, and choice

**Clear Communication**: Think aloud, invite feedback, summarize

### Practice Approach

**1. Memorize the Checklist**
- Practice asking these questions for every problem
- Internalize the categories (functional, scale, consistency, constraints)

**2. Practice Quantification**
- Do back-of-the-envelope calculations daily
- Memorize key numbers (1 day = 86,400 seconds, 1 GB = 1,000 MB)
- Practice estimating storage, QPS, bandwidth

**3. Study Trade-offs**
- For each technology, know pros/cons
- Understand when to use what (SQL vs NoSQL, cache vs no cache, etc.)
- Practice articulating trade-offs out loud

**4. Mock Interviews**
- Practice with peers or online platforms
- Record yourself and review
- Get feedback on communication style

**5. Learn from Real Systems**
- Read engineering blogs (Netflix, Uber, Airbnb, etc.)
- Understand why they made specific choices
- Note the trade-offs they considered

### Resources for Further Learning

**System Design Primers**:
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "System Design Interview" by Alex Xu
- High Scalability blog
- Engineering blogs from tech companies

**Practice Platforms**:
- Pramp (peer mock interviews)
- Interviewing.io (mock interviews with engineers)
- LeetCode System Design section
- Educative.io System Design courses

**Real-World Case Studies**:
- Netflix Tech Blog
- Uber Engineering Blog
- Airbnb Engineering Blog
- Facebook Engineering Blog
- AWS Architecture Blog

---

## Conclusion

Mastering system design interviews is not about memorizing architectures or technologies. It's about developing a structured approach to problem-solving:

1. **Ask the right questions** to understand requirements deeply
2. **Quantify everything** to make informed decisions
3. **Analyze trade-offs** to choose appropriate solutions
4. **Communicate clearly** to demonstrate your thinking process

The difference between mid-level and senior/staff engineers is not knowledge of more technologies—it's the ability to:
- Model problems systematically
- Quantify requirements before designing
- Reason through trade-offs explicitly
- Consider failure modes and edge cases
- Balance technical and business constraints

With AI becoming mainstream, the ability to name technologies (Redis, Kafka, etc.) is commoditized. What sets you apart is **structured reasoning under constraints**.

Use this guide as a reference. Practice the checklist until it becomes second nature. Focus on developing the mental model of moving from requirements → quantification → trade-offs → design.

Good luck with your interviews!

---

## Quick Reference Cards

### Card 1: Question Categories

**Functional**: What does the system do?
**Scale**: How much traffic/data?
**Performance**: How fast must it be?
**Consistency**: How correct must it be?
**Reliability**: How available must it be?
**Constraints**: What are the limits?

### Card 2: Key Numbers

**Time**:
- 1 day = 86,400 seconds (~100K for estimation)
- 1 ms = 1,000 μs

**Data**:
- 1 GB = 1,000 MB
- 1 TB = 1,000 GB

**Latency**:
- RAM: 100 ns
- SSD: 150 μs
- Network (datacenter): 0.5 ms
- Network (cross-region): 50-100 ms

**Throughput**:
- Single server: ~10K QPS
- Database: ~1K QPS
- Redis: ~100K QPS

### Card 3: Common Trade-offs

**Consistency vs Availability** (CAP)
**Latency vs Consistency** (cache staleness)
**Normalization vs Denormalization** (write vs read optimization)
**Sync vs Async** (latency vs complexity)
**Build vs Buy** (control vs cost)
**Monolith vs Microservices** (simplicity vs scalability)

### Card 4: Estimation Formulas

**Storage**: data_volume × retention × replication
**QPS**: requests_per_day ÷ 86,400 × peak_multiplier
**Bandwidth**: data_size × QPS
**Servers**: QPS ÷ capacity_per_server

### Card 5: Interview Flow

1. **Clarify** (10 min): Ask questions, quantify
2. **Design** (20 min): High-level architecture, deep dives
3. **Trade-offs** (10 min): Discuss alternatives, failure modes
4. **Wrap-up** (5 min): Summarize, answer questions

---

**End of Guide**

*This comprehensive guide covers everything you need to excel in system design interviews. Practice regularly, focus on structured thinking, and remember: it's not about memorizing solutions, it's about demonstrating how you think through complex problems.*

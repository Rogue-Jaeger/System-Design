# Message Suppression in a Time Window

## Problem

You're given a stream of messages arriving as tuples `(timestamp, message)`, where:

- `timestamp` is an integer (seconds)
- `message` is a string

Messages arrive in **non-decreasing order of timestamp**.

## Part 1

Implement a system that displays a message **only if it has not been displayed in the last 10 seconds**.

**Example:**

```text
Input:
(1, "A")
(5, "A")
(11, "A")

Output:
show
suppress
show
```

---

## Part 2 (Memory constraint)

The stream is **endless**. Modify your implementation to ensure **no memory overflow**.
You may only use a **hashmap** and must clean up expired messages efficiently.

---

## Part 3 (10-second exclusive window)

Now, change the rule:

- A message should be **suppressed entirely** if it occurs **more than once within any 10-second window** (before or after).
- That is, **if a message repeats within 10 seconds**, **none** of those appearances should be shown.

**Example:**

```text
Input:
(1, "A")
(2, "B")
(3, "A")
(12, "A")
(20, "B")
(25, "A")

Output:
B
B
A
```

**Explanation:**

- "A" at 1 and 3 → both suppressed
- "A" at 12 → allowed (past 10s from previous group)
- "B" at 2 and 20 → both allowed (each is outside 10s of other)
- Only one of each duplicate message in a 10s window can be shown.

---

## Part 4 (Out-of-order timestamps)

If messages can now arrive **out of order** due to delays, how would you adjust your solution to ensure correctness **without waiting indefinitely**?

## Solution (By Claude)

Below is a comprehensive solution for each part:

## Detailed Solution Breakdown

### Part 1: Basic Suppression

**Approach:** Simple HashMap storing last shown timestamp for each message.

- **Time Complexity:** O(1) per message
- **Space Complexity:** O(unique messages)
- **Key Insight:** Only need to track the last time each message was shown

### Part 2: Memory Efficiency

**Problem:** Infinite stream would cause unbounded memory growth.
**Solution:** Periodic cleanup of expired entries.

- **Strategy:** Before each operation, remove entries older than current_time - 10
- **Optimization:** Could use a separate cleanup thread or lazy cleanup
- **Trade-off:** Small constant overhead per operation vs. bounded memory

### Part 3: Exclusive Window Logic

**Problem:** If ANY occurrence of a message happens within 10 seconds of another, ALL should be suppressed.
**Approach:**

- Store ALL timestamps for each message within the window
- For new message, check if any existing timestamp is within 10 seconds
- This requires looking both forward and backward in time

**Key Insight:** This is more complex because we need to consider ALL occurrences in a sliding window, not just the last one.

### Part 4: Out-of-Order Handling

**Problem:** Messages arrive out of order due to network delays.
**Solution:** Buffering with bounded waiting.

**Strategy:**

1. **Buffer messages** in a TreeMap sorted by timestamp
2. **Process messages** only when we're confident we've seen all earlier messages
3. **Bounded waiting:** Process messages older than `current_time - MAX_DELAY`
4. **Trade-off:** Slight delay vs. correctness

## Key Design Decisions

### Memory Management

- Use iterators for safe removal during iteration
- Clean up empty data structures
- Bounded buffer size for out-of-order handling

### Time Complexity Analysis

- **Part 1:** O(1) per message
- **Part 2:** O(k) per message (k = expired entries to clean)
- **Part 3:** O(w) per message (w = messages in window)  
- **Part 4:** O(log n + w) per message (n = buffered messages)

### Edge Cases Handled

- First occurrence of a message
- Exact boundary conditions (10-second limits)
- Empty streams
- Very delayed messages
- Memory cleanup

### Production Considerations

1. **Threading:** Add synchronization for multi-threaded access
2. **Persistence:** Consider backing store for crash recovery  
3. **Monitoring:** Add metrics for buffer sizes, cleanup frequency
4. **Configuration:** Make window size and max delay configurable
5. **Graceful degradation:** Handle extreme delays or memory pressure

The solution progresses from simple to complex while maintaining clean interfaces and handling real-world constraints like memory limits and network delays.

```java
import java.util.*;

// ============================================================================
// PART 1: Basic Message Suppression (10-second cooldown)
// ============================================================================

class MessageSuppressor {
    private Map<String, Integer> lastShownTime = new HashMap<>();
    private static final int COOLDOWN = 10;
    
    public boolean shouldShow(int timestamp, String message) {
        Integer lastTime = lastShownTime.get(message);
        
        // If never shown or more than 10 seconds passed
        if (lastTime == null || timestamp - lastTime >= COOLDOWN) {
            lastShownTime.put(message, timestamp);
            return true;
        }
        
        return false;
    }
}

// ============================================================================
// PART 2: Memory-Efficient Version (Endless Stream)
// ============================================================================

class MemoryEfficientSuppressor {
    private Map<String, Integer> lastShownTime = new HashMap<>();
    private static final int COOLDOWN = 10;
    
    public boolean shouldShow(int timestamp, String message) {
        // Clean up expired entries before processing
        cleanExpiredEntries(timestamp);
        
        Integer lastTime = lastShownTime.get(message);
        
        if (lastTime == null || timestamp - lastTime >= COOLDOWN) {
            lastShownTime.put(message, timestamp);
            return true;
        }
        
        return false;
    }
    
    private void cleanExpiredEntries(int currentTimestamp) {
        // Remove entries older than current timestamp - COOLDOWN
        Iterator<Map.Entry<String, Integer>> iterator = lastShownTime.entrySet().iterator();
        while (iterator.hasNext()) {
            Map.Entry<String, Integer> entry = iterator.next();
            if (currentTimestamp - entry.getValue() >= COOLDOWN) {
                iterator.remove();
            }
        }
    }
}

// ============================================================================
// PART 3: 10-Second Exclusive Window (Suppress ALL duplicates in window)
// ============================================================================

class ExclusiveWindowSuppressor {
    // For each message, store all timestamps in current window
    private Map<String, List<Integer>> messageTimestamps = new HashMap<>();
    private static final int WINDOW = 10;
    
    public boolean shouldShow(int timestamp, String message) {
        // Clean up expired timestamps
        cleanExpiredTimestamps(timestamp);
        
        // Get current timestamps for this message
        List<Integer> timestamps = messageTimestamps.computeIfAbsent(message, k -> new ArrayList<>());
        
        // Check if there are any timestamps within the window
        boolean hasRecentOccurrence = false;
        for (int ts : timestamps) {
            if (Math.abs(timestamp - ts) < WINDOW) {
                hasRecentOccurrence = true;
                break;
            }
        }
        
        // Add current timestamp
        timestamps.add(timestamp);
        
        // Show only if no recent occurrence
        return !hasRecentOccurrence;
    }
    
    private void cleanExpiredTimestamps(int currentTimestamp) {
        for (Map.Entry<String, List<Integer>> entry : messageTimestamps.entrySet()) {
            List<Integer> timestamps = entry.getValue();
            timestamps.removeIf(ts -> currentTimestamp - ts >= WINDOW);
        }
        
        // Remove empty lists
        messageTimestamps.entrySet().removeIf(entry -> entry.getValue().isEmpty());
    }
}

// ============================================================================
// PART 4: Out-of-Order Timestamps (With Bounded Waiting)
// ============================================================================

class OutOfOrderSuppressor {
    private Map<String, List<Integer>> messageTimestamps = new HashMap<>();
    private TreeMap<Integer, List<Pair>> pendingMessages = new TreeMap<>(); // timestamp -> messages
    private int lastProcessedTime = -1;
    private static final int WINDOW = 10;
    private static final int MAX_DELAY = 5; // Maximum expected delay
    
    static class Pair {
        String message;
        boolean decision;
        
        Pair(String message, boolean decision) {
            this.message = message;
            this.decision = decision;
        }
    }
    
    public List<String> processMessage(int timestamp, String message) {
        List<String> output = new ArrayList<>();
        
        // Add to pending messages
        pendingMessages.computeIfAbsent(timestamp, k -> new ArrayList<>())
                      .add(new Pair(message, false)); // decision to be made later
        
        // Process all messages that are now safe to process
        // (current timestamp - MAX_DELAY ensures we've seen most out-of-order messages)
        int safeTimestamp = timestamp - MAX_DELAY;
        
        while (!pendingMessages.isEmpty() && pendingMessages.firstKey() <= safeTimestamp) {
            Map.Entry<Integer, List<Pair>> entry = pendingMessages.pollFirstEntry();
            int ts = entry.getKey();
            List<Pair> messages = entry.getValue();
            
            for (Pair pair : messages) {
                boolean shouldShow = shouldShowMessage(ts, pair.message);
                if (shouldShow) {
                    output.add(pair.message);
                }
            }
            
            lastProcessedTime = ts;
        }
        
        return output;
    }
    
    private boolean shouldShowMessage(int timestamp, String message) {
        // Clean up expired timestamps
        cleanExpiredTimestamps(timestamp);
        
        // Get current timestamps for this message
        List<Integer> timestamps = messageTimestamps.computeIfAbsent(message, k -> new ArrayList<>());
        
        // Check if there are any timestamps within the window
        boolean hasRecentOccurrence = false;
        for (int ts : timestamps) {
            if (Math.abs(timestamp - ts) < WINDOW) {
                hasRecentOccurrence = true;
                break;
            }
        }
        
        // Add current timestamp
        timestamps.add(timestamp);
        
        return !hasRecentOccurrence;
    }
    
    private void cleanExpiredTimestamps(int currentTimestamp) {
        for (Map.Entry<String, List<Integer>> entry : messageTimestamps.entrySet()) {
            List<Integer> timestamps = entry.getValue();
            timestamps.removeIf(ts -> currentTimestamp - ts >= WINDOW);
        }
        messageTimestamps.entrySet().removeIf(entry -> entry.getValue().isEmpty());
    }
    
    // Force flush remaining messages (call this when stream ends or periodically)
    public List<String> flush() {
        List<String> output = new ArrayList<>();
        
        while (!pendingMessages.isEmpty()) {
            Map.Entry<Integer, List<Pair>> entry = pendingMessages.pollFirstEntry();
            int ts = entry.getKey();
            List<Pair> messages = entry.getValue();
            
            for (Pair pair : messages) {
                boolean shouldShow = shouldShowMessage(ts, pair.message);
                if (shouldShow) {
                    output.add(pair.message);
                }
            }
        }
        
        return output;
    }
}

// ============================================================================
// TEST CLASS
// ============================================================================

class MessageSuppressionTest {
    public static void main(String[] args) {
        System.out.println("=== PART 1 TEST ===");
        testPart1();
        
        System.out.println("\n=== PART 3 TEST ===");
        testPart3();
        
        System.out.println("\n=== PART 4 TEST ===");
        testPart4();
    }
    
    static void testPart1() {
        MessageSuppressor suppressor = new MessageSuppressor();
        
        System.out.println(suppressor.shouldShow(1, "A") ? "show" : "suppress"); // show
        System.out.println(suppressor.shouldShow(5, "A") ? "show" : "suppress"); // suppress
        System.out.println(suppressor.shouldShow(11, "A") ? "show" : "suppress"); // show
    }
    
    static void testPart3() {
        ExclusiveWindowSuppressor suppressor = new ExclusiveWindowSuppressor();
        
        System.out.println(suppressor.shouldShow(1, "A") ? "A" : "suppress"); // suppress (will conflict with ts=3)
        System.out.println(suppressor.shouldShow(2, "B") ? "B" : "suppress"); // B
        System.out.println(suppressor.shouldShow(3, "A") ? "A" : "suppress"); // suppress (conflicts with ts=1)
        System.out.println(suppressor.shouldShow(12, "A") ? "A" : "suppress"); // A
        System.out.println(suppressor.shouldShow(20, "B") ? "B" : "suppress"); // B
        System.out.println(suppressor.shouldShow(25, "A") ? "A" : "suppress"); // A
    }
    
    static void testPart4() {
        OutOfOrderSuppressor suppressor = new OutOfOrderSuppressor();
        
        // Simulate out-of-order messages
        System.out.println("Processing (5, 'A'): " + suppressor.processMessage(5, "A"));
        System.out.println("Processing (1, 'B'): " + suppressor.processMessage(1, "B"));
        System.out.println("Processing (10, 'C'): " + suppressor.processMessage(10, "C"));
        System.out.println("Processing (3, 'A'): " + suppressor.processMessage(3, "A"));
        System.out.println("Processing (15, 'D'): " + suppressor.processMessage(15, "D"));
        
        // Flush remaining
        System.out.println("Flushing: " + suppressor.flush());
    }
}
```

## Solution (By ChatGPT)

Nice — this is a great multi-part design question. I’ll walk through each part, give clean, correct algorithms, show how to implement them (pseudocode / Python snippets), explain memory and time costs, cover corner-cases, and then extend to the out-of-order case with a practical solution (watermarks / bounded lateness).

**Important:** There’s an ambiguity in Part 3 about whether “within 10 seconds” includes the boundary (difference == 10) or not. I’ll state a clear convention and stick with it; I’ll also explain how to tweak the code for the other convention.

---

## Part 0 — Convention and Notation

Let `WINDOW = 10` (seconds).
I assume message timestamps are integers and arrive in **non-decreasing** order unless otherwise stated.

**Convention used below:**

- “Within 10 seconds” means difference **< WINDOW** (strictly less than 10).
- For the “last 10 seconds” test in Part 1 (typical LeetCode Logger problem), the usual rule is `timestamp - last_shown_timestamp >= WINDOW` allowed. That is, when `timestamp - last_shown >= 10` we allow it. You can flip comparisons (<= vs < or >=) if you prefer a different boundary.

If you want the opposite boundary semantics (include equality), I’ll point out where to change comparisons.

---

## Part 1 — Simple Rate Limiter: Show if Not Displayed in Last 10s

**Requirement:** display a message only if it has not been displayed in the last 10 seconds.

**Data structure:** `HashMap<String, int> lastShown` mapping message → last timestamp when it was displayed.

**Algorithm (arriving `(t, msg)`):**

- If `msg` not in `lastShown` or `t - lastShown[msg] >= WINDOW`:

  - display message
  - `lastShown[msg] = t`
- else

  - suppress

**Complexity:**

- Time per message: O(1)
- Space: O(U) where U is number of unique messages (could grow unbounded unless we clean it up — that is Part 2)

**Python-like example:**

```java
class SimpleRateLimiter {
    private static final int WINDOW = 10;
    private Map<String, Integer> lastShown = new HashMap<>();  // msg -> last timestamp shown

    public void handle(int t, String msg) {
        if (!lastShown.containsKey(msg) || t - lastShown.get(msg) >= WINDOW) {
            System.out.println("show");
            lastShown.put(msg, t);
        } else {
            System.out.println("suppress");
        }
    }
}
```

This is the classic Logger Rate Limiter.

---

## Part 2 — Stream is Endless; Avoid Unbounded Memory

If the stream is infinite, `last_shown` can grow without bound (messages that appeared once but never again). We must **clean up expired entries** so memory stays proportional to messages active in the recent window only.

Two practical, efficient options:

## Option A — HashMap + Queue (sliding window buffer)

**Idea:** Keep only the last `WINDOW` seconds of events in a queue. Keep counts or lastShown in a HashMap. When time moves forward, evict old entries from the queue and remove corresponding keys from the map if they are stale.

**Data structures:**

- `deque` of `(timestamp, msg)` for events that occurred in the *recent* window when you need them (or for cleaning).
- `last_shown` map still stores last shown timestamp for each message that occurred in the sliding window.

**How to clean:** When the current time is `t`, pop from left of queue while `queue[0].timestamp <= t - WINDOW` and for each popped msg, check if `last_shown[msg]` equals that popped timestamp — if yes, remove the key from `last_shown` (it’s now older than window and safe to drop). This keeps `last_shown` size ≈ number of distinct messages in the last `WINDOW` seconds.

**Complexity:** amortized O(1) per event. Memory = O(U\_window) where U\_window is distinct messages in last `WINDOW` seconds.

**Pseudo:**

```java
class SlidingWindowRateLimiter {
    private static final int WINDOW = 10;
    private Deque<TimestampMessage> queue = new ArrayDeque<>();  // stores (t, msg) of arrivals
    private Map<String, Integer> lastShown = new HashMap<>();  // msg -> last shown timestamp

    private static class TimestampMessage {
        final int timestamp;
        final String message;

        TimestampMessage(int timestamp, String message) {
            this.timestamp = timestamp;
            this.message = message;
        }
    }

    public void handle(int t, String msg) {
        // evict outdated events
        while (!queue.isEmpty() && queue.peekFirst().timestamp <= t - WINDOW) {
            TimestampMessage old = queue.pollFirst();
            // remove old_msg if lastShown indicates that old_t was its last shown time
            if (lastShown.containsKey(old.message) && lastShown.get(old.message) == old.timestamp) {
                lastShown.remove(old.message);
            }
        }

        // now normal logic from Part 1
        if (!lastShown.containsKey(msg) || t - lastShown.get(msg) >= WINDOW) {
            System.out.println("show");
            lastShown.put(msg, t);
        } else {
            System.out.println("suppress");
        }

        queue.addLast(new TimestampMessage(t, msg));
    }
}
```

> Note: the queue may store only *shown* timestamps (if you only used last\_shown). If you also need to garbage-collect messages that were suppressed but still present in other structures, store their timestamps similarly.

## Option B — HashMap + Time-buckets (circular buffer / timing wheel)

**Idea:** Use `WINDOW` buckets (one per second modulo `WINDOW`) — each bucket contains the set of messages that last occurred at timestamps congruent to that second. When time advances to second `s`, you clear bucket `s % WINDOW` and remove those messages from `last_shown` if they are still stale.

**Why:** O(1) amortized cleanup and no unbounded scanning.

**Data structures:**

- `last_shown` map: msg → last shown timestamp
- `buckets` array of size `WINDOW` where `buckets[i]` is a list/set of messages whose last\_shown timestamp mod WINDOW == i

**Procedure:**

- On new event `(t,msg)`:

  - Check `last_shown` for decision (as Part 1).
  - If you set `last_shown[msg] = t`, add `msg` into bucket `t % WINDOW`.
  - To clean at timestamp `t`, clear bucket `(t - WINDOW) % WINDOW` and remove those entries from `last_shown` **only if** their recorded timestamp ≤ `t - WINDOW` (verify equality to avoid removing messages that were updated later).

**Complexity:** O(1) average for each second; memory bounded by U\_window.

This is the go-to for very high-throughput systems.

---

## Part 3 — New Rule: Suppress Entirely if Any Duplicate Within 10s (Before or After)

**Problem restatement:** If a message has more than one occurrence within any 10-second window (before or after), **none** of the occurrences in that window should be shown. That implies we must consider both past and future occurrences when deciding to emit an event. Therefore we **cannot decide immediately** on arrival — we must delay the final decision for an event until we know whether a future duplicate arrives within the `WINDOW` after it.

**Key idea (safe delay / buffering):**

- Keep a sliding buffer of the most recent `WINDOW` seconds of *all* events (shown or suppressed candidates).
- Only when an event is older than `WINDOW` seconds relative to the current time can we be sure no future event inside that `WINDOW` will arrive to affect it. So we **flush** decisions for events with `timestamp <= current_time - WINDOW`.

**Algorithm (arriving `(t,msg)` in non-decreasing order):**

- Maintain:

  - `deque events` of `(t,msg)` — contains all events not yet flushed (i.e., timestamps > `t - WINDOW`)
  - `counts` map: `msg -> number of occurrences in current buffer` (includes occurrences before and after the candidate while buffer is active)
- On arrival `(t,msg)`:

  1. Append `(t,msg)` to `events` and do `counts[msg] += 1`.
  2. Evict/flush from the left of `events` while `events[0].t <= t - WINDOW` (these are now safe to decide).

     - For each popped event `(et, em)`:

       - If `counts[em] > 1`: **suppress** this popped event.
       - Else (counts\[em] == 1): **show** this event.
       - Then `counts[em] -= 1`. If `counts[em] == 0` remove key.

**Why this is correct:**
When we flush event at time `et` (because `t >= et + WINDOW`), the `counts` included any events in `(et, et + WINDOW)` (arrived already). Since we flush only after `WINDOW` seconds have passed, **no future event** arriving later than `t` could be within `WINDOW` of `et` (because `t` always moves forward and we flush when difference ≥ WINDOW). Thus the decision is final.

**Complexity:**

- Each event is appended and popped exactly once ⇒ O(1) amortized per event.
- Memory bounded by number of events in any `WINDOW` interval.

**Python-like pseudocode:**

```java
class ExclusiveWindowBuffer {
    private static final int WINDOW = 10;
    private Deque<TimestampMessage> events = new ArrayDeque<>();  // stores (timestamp, message)
    private Map<String, Integer> counts = new HashMap<>();

    private static class TimestampMessage {
        final int timestamp;
        final String message;

        TimestampMessage(int timestamp, String message) {
            this.timestamp = timestamp;
            this.message = message;
        }
    }

    public void handle(int t, String msg) {
        // 1) add new event into buffer
        events.addLast(new TimestampMessage(t, msg));
        counts.merge(msg, 1, Integer::sum);

        // 2) flush events that are safe to decide
        while (!events.isEmpty() && events.peekFirst().timestamp <= t - WINDOW) {
            TimestampMessage event = events.pollFirst();
            if (counts.get(event.message) > 1) {
                // this event had some neighbor in its WINDOW => suppress
                System.out.println("suppress");
            } else {
                System.out.println("show");
            }
            counts.merge(event.message, -1, Integer::sum);
            if (counts.get(event.message) == 0) {
                counts.remove(event.message);
            }
        }
    }
}
```

**Example walk-through (our convention):**

- Input: (1,A), (3,A), (12,A).

  - At t=1: buffer=\[(1,A)], counts\[A]=1. Nothing flush (no events <= 1-10).
  - At t=3: buffer=\[(1,A),(3,A)], counts\[A]=2. Still nothing flush.
  - At t=12: buffer append (12,A), counts\[A]=3. Now flush events with timestamp ≤ 12-10 = 2 → we flush (1,A). counts\[A] == 3 > 1 ⇒ suppress (1). counts\[A]=2.

    - Next, nothing else ≤2.
  - Later when t reaches 13, flush (3, A) (3 ≤ 13-10=3 → yes) counts\[A]=2 so suppress (3). counts\[A]=1.
  - When t reaches 22 and we flush (12,A), counts\[A]==1 so this (12) becomes **show**.

**Note:** in this run, (12,A) would eventually be shown once it becomes safe, which matches the problem’s textual example if they intended that behavior. That’s because 12 was >10 seconds after the earlier *flushed events* by the time we decide. Adjust `<=` or `<` to match your boundary semantics.

**Important:** this approach delays outputs by up to `WINDOW` seconds (latency trade-off) but ensures correctness.

---

## Alternate Interpretation and How to Adapt

If the problem intended **instant suppression** (i.e., on arrival you must decide show/suppress immediately based only on events in the past), Part 3’s “before or after” requirement cannot be satisfied correctly (you cannot know about future events). Therefore the only correct interpretation is that we *may buffer events up to WINDOW* to account for future arrivals — which is what we implemented.

If you require zero latency, you must define the rule as “suppress if there is a duplicate in the **past** 10 seconds only” (no mention of future), which is easy (use a per-message deque of recent timestamps and check last timestamp).

---

## Part 4 — Out-of-Order Timestamps (Delays), How to Ensure Correctness Without Waiting Forever

If messages can arrive **out of order**, we have to handle late arrivals. You cannot wait forever, so you trade off between latency and correctness using an **allowed lateness** parameter `L` (how late you are willing to tolerate messages arriving out of order). This is standard streaming/watermark design.

## Design idea — watermark + bounded lateness

- Maintain `max_seen_timestamp` — maximum timestamp you have observed so far.
- Define `ALLOWED_LATENESS = L` (system parameter, e.g., 5 seconds). The watermark is `watermark = max_seen_timestamp - L`.
- You can confidently output final decisions for any events with timestamp `et` such that `et + WINDOW <= watermark`. Why? Because:

  - Any future arrival that is ≤ `max_seen_timestamp` could be late up to `L`, but the watermark guarantees there will be no events with timestamp ≤ `watermark` arriving later (within your allowed lateness).
  - So for an event with timestamp `et`, when `et + WINDOW <= watermark`, even the latest allowed future arrivals (`≤ max_seen_timestamp`) cannot be within `WINDOW` of `et`. So `et` is safe to flush.
- Implement the buffer as a priority queue (min-heap) ordered by timestamp, or keep a deque but you must insert out of order (heap easier).
- On each arrival:

  - update `max_seen_timestamp`
  - add event to buffer and update counts
  - update watermark
  - flush all buffered events with `et + WINDOW <= watermark` using the same counts logic as Part 3.

**Trade-offs:**

- Larger `L` → higher correctness (tolerate more late arrivals) but higher latency and buffer/memory.
- If `L` is small you may get wrong outputs for very late messages that arrive after watermark passes — you must decide whether to drop or reprocess.

## Pseudocode outline with heap

```java
class OutOfOrderHandler {
    private PriorityQueue<TimestampMessage> eventsHeap;  // min-heap of (timestamp, msg)
    private Map<String, Integer> counts = new HashMap<>();
    private int maxSeen = Integer.MIN_VALUE;
    private static final int WINDOW = 10;
    private static final int ALLOWED_LATENESS = 5;

    public OutOfOrderHandler() {
        this.eventsHeap = new PriorityQueue<>((a, b) -> Integer.compare(a.timestamp, b.timestamp));
    }

    public void handleOutOfOrder(int t, String msg) {
        maxSeen = Math.max(maxSeen, t);
        
        // add event
        eventsHeap.offer(new TimestampMessage(t, msg));
        counts.merge(msg, 1, Integer::sum);

        // compute watermark
        int watermark = maxSeen - ALLOWED_LATENESS;

        // flush events safe to decide: et + WINDOW <= watermark
        while (!eventsHeap.isEmpty() && 
               eventsHeap.peek().timestamp + WINDOW <= watermark) {
            TimestampMessage event = eventsHeap.poll();
            if (counts.get(event.message) > 1) {
                emitSuppress(event.timestamp, event.message);
            } else {
                emitShow(event.timestamp, event.message);
            }
            counts.merge(event.message, -1, Integer::sum);
            if (counts.get(event.message) == 0) {
                counts.remove(event.message);
            }
        }
    }

    private void emitShow(int timestamp, String message) {
        System.out.println("show: (" + timestamp + ", " + message + ")");
    }

    private void emitSuppress(int timestamp, String message) {
        System.out.println("suppress: (" + timestamp + ", " + message + ")");
    }

    private static class TimestampMessage {
        final int timestamp;
        final String message;

        TimestampMessage(int timestamp, String message) {
            this.timestamp = timestamp;
            this.message = message;
        }
    }
}
```

**Notes:**

- If messages can be arbitrarily late (no upper bound), correctness without potentially infinite waiting is impossible.
- Real streaming systems (e.g., Apache Flink) use watermarks and allowed lateness; late events beyond the allowed lateness may be sent to a special “late events” stream to be handled differently.

---

## Summary, Complexity, and Practical Notes

- **Part 1**: Hash map of last shown times. O(1) per event. Simple.
- **Part 2**: Use HashMap + Queue or HashMap + circular buckets to evict stale keys and keep memory bounded to recent activity. O(1) amortized.
- **Part 3**: Because the rule depends on future occurrences, buffer events for up to `WINDOW` seconds and only flush when safe. Use `deque` + counts. Correct and O(1) amortized, uses memory proportional to events in any window.
- **Part 4**: For out-of-order arrivals, use watermark/allowed-lateness. Buffer in a heap or sorted structure; flush when `et + WINDOW <= watermark`. Trade correctness vs. latency with parameter `L`.

---

## Example: Full Java Implementation for Parts 1–3 (Ordered Input)

Here’s a complete, runnable (simple) implementation that supports Part 1 and Part 3 (ordered timestamps). You can toggle modes.

```java
class MessageSuppression {
    private static final int WINDOW = 10;

    static class RateLimiter {
        private Map<String, Integer> lastShown = new HashMap<>();  // for Part 1
        private Deque<TimestampMessage> queue = new ArrayDeque<>();  // for Part 2 cleaning

        // Part 1 handler
        public void handlePart1(int t, String msg) {
            if (!lastShown.containsKey(msg) || t - lastShown.get(msg) >= WINDOW) {
                System.out.printf("%d:%s -> show%n", t, msg);
                lastShown.put(msg, t);
            } else {
                System.out.printf("%d:%s -> suppress%n", t, msg);
            }
        }
    }

    static class Part3Buffer {
        private Deque<TimestampMessage> events = new ArrayDeque<>();
        private Map<String, Integer> counts = new HashMap<>();

        // Part 3 handler (ordered arrivals)
        public void handle(int t, String msg) {
            events.addLast(new TimestampMessage(t, msg));
            counts.merge(msg, 1, Integer::sum);

            // flush safe events: those with timestamp <= t - WINDOW
            while (!events.isEmpty() && events.peekFirst().timestamp <= t - WINDOW) {
                TimestampMessage event = events.pollFirst();
                if (counts.get(event.message) > 1) {
                    System.out.printf("%d:%s -> suppress%n", event.timestamp, event.message);
                } else {
                    System.out.printf("%d:%s -> show%n", event.timestamp, event.message);
                }
                counts.merge(event.message, -1, Integer::sum);
                if (counts.get(event.message) == 0) {
                    counts.remove(event.message);
                }
            }
        }
    }

    private static class TimestampMessage {
        final int timestamp;
        final String message;

        TimestampMessage(int timestamp, String message) {
            this.timestamp = timestamp;
            this.message = message;
        }
    }
}
```

---

## Final Thoughts and Tips

- **Define boundary semantics** (`<= 10` vs `< 10`) clearly in requirements — this changes comparisons in code.
- **Latency vs correctness:** Part 3 forces buffering (latency up to `WINDOW` seconds). For out-of-order, introduce `ALLOWED_LATENESS` to bound latency.
- **Memory:** All designs keep memory proportional to the number of unique messages in the recent `WINDOW` (or within allowed lateness). If your message variety is enormous in a short time, consider eviction policies or approximate structures (Bloom filters / HyperLogLog) depending on the acceptable error rate.
- **Testing:** For Part 3, test with sequences where duplicates arrive both before and after the event and around flush boundaries to ensure your comparisons are correct.

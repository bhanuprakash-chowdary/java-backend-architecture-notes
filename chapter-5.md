# Interview Survival Guide
## Chapter 5: HLD Advanced & Full Project Narratives
### (Weeks 18–22)

---

## PART 1 — ADVANCED HLD: TWITTER-LIKE NEWS FEED

---

### Design a Social News Feed (Twitter / LinkedIn style)

**Step 1 — Requirements Clarification:**

Functional:
- Users can post content (text, images, links)
- Users can follow other users
- Each user has a home feed: posts from people they follow, reverse-chronological
- Posts support likes and comments (read-heavy)
- Search by hashtag or keyword

Non-Functional:
- 500M users, 100M daily active
- 5M posts/day → ~58 writes/sec, ~500 writes/sec peak
- 200B feed reads/day → ~2.3M reads/sec
- Read:Write ratio ≈ 10,000:1 — heavily read-optimized
- Feed load latency < 200ms p99
- Eventual consistency acceptable (slight feed lag is fine)

**Step 2 — Capacity:**
```
Posts:   58 writes/sec avg, 500/sec peak
         1 post ≈ 1KB → 5M × 1KB = 5GB/day → 1.8TB/year

Follows: 100M users × 200 avg follows = 20B follow relationships
         20B × 16 bytes (uid pair) = 320GB → fits in distributed memory

Feed cache: 200M active users × 20 feed entries × 64 bytes = 256GB Redis
```

**Step 3 — Architecture: Fan-Out-on-Write vs Fan-Out-on-Read**

**The core design decision for a news feed:**

| Approach | How it works | When to use |
|----------|-------------|-------------|
| Fan-out-on-Write (Push) | On post, write to all followers' feed caches immediately | Most users — fast reads, write amplification |
| Fan-out-on-Read (Pull) | On feed load, merge posts from all followed accounts | Celebrity accounts (millions of followers) |
| Hybrid | Push for normal users, pull for celebrities | Production Twitter/Instagram approach |

**Why hybrid?**
A celebrity with 50M followers posting would trigger 50M cache writes — unacceptable write latency and cost. The hybrid: push to followers of normal users (<10K followers), pull-merge for celebrity posts at read time.

```
Write Path (Fan-Out-on-Write):

User posts → Post Service → Save to Posts DB (Cassandra)
                          → Fanout Service
                          → Fetch follower list (Cassandra)
                          → For each follower (if < 10K follows):
                                Write post_id to follower's feed cache (Redis sorted set, score = timestamp)
                          → Kafka: "post-created" event (for async fanout to large follower lists)

Read Path (Feed Assembly):

User requests feed → Feed Service
                   → Fetch from feed cache (Redis ZREVRANGE key 0 19) — top 20 entries
                   → For each post_id: fetch post details (in parallel, Redis/Cassandra)
                   → Merge with celebrity posts pulled directly (fan-out-on-read for celebrities)
                   → Return assembled feed
```

**Data model:**

```sql
-- Posts (Cassandra — write-heavy, time-series access pattern)
CREATE TABLE posts (
    post_id    UUID,
    user_id    UUID,
    content    TEXT,
    media_url  TEXT,
    created_at TIMESTAMP,
    like_count COUNTER,
    PRIMARY KEY (post_id)
);

-- Feeds (Redis — sorted set per user)
-- Key: "feed:{user_id}"
-- Member: "{post_id}"
-- Score: epoch_timestamp (for chronological ordering)
-- ZADD feed:{user_id} {timestamp} {post_id}
-- ZREVRANGE feed:{user_id} 0 19  → top 20 most recent

-- Follows (Cassandra — wide row per user)
CREATE TABLE follows (
    follower_id UUID,
    followed_id UUID,
    created_at  TIMESTAMP,
    PRIMARY KEY (follower_id, followed_id)
);

CREATE TABLE followers (  -- reverse lookup
    followed_id UUID,
    follower_id UUID,
    PRIMARY KEY (followed_id, follower_id)
);
```

**Feed cache management:**
```java
@Service
public class FeedCacheService {
    private static final int MAX_FEED_SIZE = 1000;  // keep last 1000 posts per user

    public void addToFeed(UUID followerId, UUID postId, long timestamp) {
        String key = "feed:" + followerId;
        redisTemplate.opsForZSet().add(key, postId.toString(), timestamp);
        // Trim to last 1000 — avoid unbounded growth
        redisTemplate.opsForZSet().removeRange(key, 0, -(MAX_FEED_SIZE + 1));
        redisTemplate.expire(key, Duration.ofDays(30));
    }

    public List<UUID> getTopFeed(UUID userId, int page, int size) {
        String key = "feed:" + userId;
        int start = page * size;
        Set<Object> postIds = redisTemplate.opsForZSet()
            .reverseRange(key, start, start + size - 1);
        return postIds.stream().map(id -> UUID.fromString((String) id)).toList();
    }
}
```

**Failure analysis:**
- **Redis feed cache miss:** Fall back to fan-out-on-read (hit Posts DB). Slower but correct. Rebuild cache asynchronously.
- **Fanout service lag:** Users see slightly stale feeds (seconds). Acceptable per requirements.
- **Posts DB (Cassandra) down:** All reads fail. Mitigation: read replicas (Cassandra replication factor 3), circuit breaker.

**Senior Signal:** "The critical insight here is the fan-out-on-write vs fan-out-on-read trade-off. This is the decision that defines the whole architecture. Pure push doesn't work for celebrities. Pure pull doesn't work for high read QPS. The hybrid is the industry-standard answer — and the interesting engineering is in the threshold (when do you switch from push to pull for a user?) and the merge algorithm at read time."

---

## PART 2 — ADVANCED HLD: DISTRIBUTED CACHE (REDIS CLONE)

---

### Design a Distributed In-Memory Cache

**Step 1 — Requirements:**

Functional:
- GET, SET, DELETE operations with string keys
- TTL support (per key expiration)
- Eviction when memory is full (LRU policy)

Non-Functional:
- < 1ms p99 latency (memory-only, no disk)
- 99.99% availability
- Horizontal scalability (add nodes without downtime)
- 1TB total data across cluster

**Step 2 — Architecture:**

```
Client
  │
  ▼
Client-Side Library (consistent hash routing)
  │
  ├── Node 1 (250GB, vnodes 0–250)
  ├── Node 2 (250GB, vnodes 251–500)
  ├── Node 3 (250GB, vnodes 501–750)
  └── Node 4 (250GB, vnodes 751–1000)

Each node:
  ├── Hash table (key → value + metadata)
  ├── LRU doubly-linked list (eviction ordering)
  ├── Expiry heap / wheel timer (TTL management)
  └── Replication (async to 1 replica node)
```

**Consistent Hashing:**
```java
public class ConsistentHashRing {
    private final TreeMap<Long, CacheNode> ring = new TreeMap<>();
    private static final int VIRTUAL_NODES = 150;  // per physical node
    private final MessageDigest md5;

    public void addNode(CacheNode node) {
        for (int i = 0; i < VIRTUAL_NODES; i++) {
            long hash = hash(node.getId() + "#" + i);
            ring.put(hash, node);
        }
    }

    public void removeNode(CacheNode node) {
        for (int i = 0; i < VIRTUAL_NODES; i++) {
            ring.remove(hash(node.getId() + "#" + i));
        }
    }

    public CacheNode getNode(String key) {
        if (ring.isEmpty()) throw new IllegalStateException("No nodes in ring");
        long hash = hash(key);
        Map.Entry<Long, CacheNode> entry = ring.ceilingEntry(hash);
        if (entry == null) entry = ring.firstEntry();  // wrap around
        return entry.getValue();
    }

    private long hash(String key) {
        byte[] digest = md5.digest(key.getBytes(StandardCharsets.UTF_8));
        // Use first 8 bytes as long
        return ((long)(digest[3] & 0xFF) << 24) | ((long)(digest[2] & 0xFF) << 16)
             | ((long)(digest[1] & 0xFF) << 8)  | ((long)(digest[0] & 0xFF));
    }
}
```

**LRU eviction:**
```
HashMap: O(1) key lookup → returns node in doubly-linked list
Doubly-linked list: most recently used at head, least recently used at tail

GET key:
  1. HashMap lookup → find node
  2. Move node to head of list
  3. Return value

SET key:
  1. If key exists: update value, move to head
  2. If memory full: remove tail node (LRU), remove from HashMap
  3. Insert new node at head, add to HashMap

DELETE key:
  1. HashMap lookup → find node
  2. Remove from list, remove from HashMap
```

**TTL — timing wheel:**
A timing wheel is a circular array of buckets, each representing a time slot (e.g., 1 second). Keys expiring in slot T go in bucket T % wheel_size. Each second, advance the pointer and expire all keys in the current bucket.

**Replication — async primary-replica:**
Each primary node asynchronously replicates writes to one replica. On primary failure, replica is promoted (leader election via Zookeeper or Raft). Data loss window: time since last replication ack (configurable, typically < 1 second).

---

## PART 3 — ADVANCED HLD: REAL-TIME LEADERBOARD

---

### Design a Real-Time Leaderboard (Gaming / Sales contest)

**Step 1 — Requirements:**

Functional:
- Update a user's score
- Get top N users (global leaderboard)
- Get a specific user's rank
- Get users around a specific rank (neighboring ranks)
- Real-time updates (< 5 second lag)

Non-Functional:
- 1M users, 10M score updates/day → ~115 updates/sec
- Leaderboard reads: 50M/day → ~580 reads/sec
- Top-N query < 10ms

**Architecture — Redis Sorted Set is the natural fit:**

Redis Sorted Set (`ZSET`) stores members with scores. Operations:
- `ZADD leaderboard {score} {userId}` — O(log N) insert/update
- `ZREVRANK leaderboard {userId}` — O(log N) get rank
- `ZREVRANGE leaderboard 0 99` — O(log N + K) get top 100
- `ZREVRANGEBYSCORE leaderboard +inf -inf LIMIT 0 10` — O(log N + K) range query

```java
@Service
public class LeaderboardService {
    private static final String GLOBAL_BOARD = "leaderboard:global";
    private final RedisTemplate<String, String> redisTemplate;

    public void updateScore(String userId, double scoreDelta) {
        // ZINCRBY is atomic increment — handles concurrent updates correctly
        redisTemplate.opsForZSet().incrementScore(GLOBAL_BOARD, userId, scoreDelta);
    }

    public List<LeaderboardEntry> getTopN(int n) {
        Set<ZSetOperations.TypedTuple<String>> topN = redisTemplate.opsForZSet()
            .reverseRangeWithScores(GLOBAL_BOARD, 0, n - 1);

        return topN.stream().map(tuple -> new LeaderboardEntry(
            tuple.getValue(),
            tuple.getScore(),
            getRank(tuple.getValue())
        )).toList();
    }

    public long getRank(String userId) {
        Long rank = redisTemplate.opsForZSet().reverseRank(GLOBAL_BOARD, userId);
        return rank != null ? rank + 1 : -1;  // 0-indexed → 1-indexed
    }

    public List<LeaderboardEntry> getNeighbors(String userId, int windowSize) {
        Long rank = redisTemplate.opsForZSet().reverseRank(GLOBAL_BOARD, userId);
        if (rank == null) return Collections.emptyList();
        long start = Math.max(0, rank - windowSize);
        long end = rank + windowSize;
        // Returns users above and below in rank
        return getRange(start, end);
    }
}
```

**Scaling beyond single Redis node:**

For 1M+ users, a single sorted set still performs at O(log N) — Redis handles 1M entries in a sorted set easily. For 100M+ users, shard by time bucket:

```
Daily leaderboard:   "leaderboard:2024-01-15"   (rolling)
Weekly leaderboard:  "leaderboard:week-2024-03"
Monthly leaderboard: "leaderboard:month-2024-01"
All-time leaderboard: "leaderboard:alltime"     (deduplicated via event sourcing)
```

**Real-time WebSocket push:**
```java
@Component
public class LeaderboardChangeListener {

    @EventListener
    @Async
    public void onScoreUpdate(ScoreUpdateEvent event) {
        // Check if rank changed significantly (top 100 threshold)
        long newRank = leaderboardService.getRank(event.getUserId());
        if (newRank <= 100) {
            // Push updated top-100 to all connected dashboard clients
            webSocketTemplate.convertAndSend(
                "/topic/leaderboard/top100",
                leaderboardService.getTopN(100)
            );
        }
        // Push rank update to the specific user
        webSocketTemplate.convertAndSendToUser(
            event.getUserId(),
            "/queue/my-rank",
            new RankUpdate(event.getUserId(), newRank, event.getNewScore())
        );
    }
}
```

---

## PART 4 — ADVANCED HLD: DISTRIBUTED JOB SCHEDULER

---

### Design a Distributed Job Scheduler (Cron-as-a-Service)

**Step 1 — Requirements:**

Functional:
- Schedule jobs with cron expressions (e.g., `0 9 * * MON-FRI`)
- One-time jobs (run at specific timestamp)
- Exactly-once execution (no missed runs, no double runs)
- Job history (last N executions, status, duration)
- Pause, resume, delete jobs

Non-Functional:
- 1M scheduled jobs
- ~100K jobs/day → ~1 job/sec average, ~10 jobs/sec peak
- Job trigger latency < 1 second from scheduled time
- High availability (scheduler itself cannot be a SPOF)

**Step 2 — The Core Challenges:**

1. **Exactly-once execution across multiple scheduler instances** — if two instances both think it's time to run job X, it runs twice.
2. **Efficient next-run computation** — with 1M jobs, checking all of them every second is infeasible.
3. **Single point of failure** — the scheduler must itself be highly available.

**Step 3 — Architecture:**

```
┌──────────────────────────────────────────────────────┐
│              Job Management API (stateless)           │
│  POST /jobs  GET /jobs/{id}  DELETE /jobs/{id}        │
└──────────────────────────┬───────────────────────────┘
                           │
                     ┌─────▼──────┐
                     │  Jobs DB   │
                     │ (Postgres) │
                     │ jobs table │
                     │ job_runs   │
                     └─────┬──────┘
                           │
         ┌─────────────────┼──────────────────┐
         ▼                 ▼                  ▼
   Scheduler Node 1  Scheduler Node 2  Scheduler Node 3
   (polls DB for     (polls DB for     (polls DB for
    due jobs via      due jobs via      due jobs via
    SELECT FOR        SELECT FOR        SELECT FOR
    UPDATE SKIP       UPDATE SKIP       UPDATE SKIP
    LOCKED)           LOCKED)           LOCKED)
         │
         ▼
   Kafka: "job-dispatch" topic
         │
         ▼
   Worker Pool (executes jobs, reports results)
```

**Exactly-once via `SELECT FOR UPDATE SKIP LOCKED`:**
```sql
-- Each scheduler instance runs this every second
WITH due_jobs AS (
    SELECT id, job_definition, next_run_at
    FROM scheduled_jobs
    WHERE next_run_at <= NOW()
      AND status = 'ACTIVE'
    ORDER BY next_run_at
    LIMIT 50                          -- claim batch of 50
    FOR UPDATE SKIP LOCKED            -- skip rows locked by other schedulers
)
UPDATE scheduled_jobs
SET status = 'RUNNING',
    last_claimed_at = NOW(),
    claimed_by = :instanceId,
    next_run_at = calculate_next_run(cron_expression, NOW())
FROM due_jobs
WHERE scheduled_jobs.id = due_jobs.id
RETURNING *;
```

`SKIP LOCKED` is the key: multiple scheduler instances each try to claim the same batch of due jobs, but each row can only be claimed by one instance. Other instances skip locked rows and move on. No double execution.

**Next-run computation:**
```java
public class CronNextRunCalculator {
    public Instant nextRun(String cronExpression, Instant from) {
        CronExpression cron = CronExpression.parse(cronExpression);
        // Use a cron library (quartz-scheduler or spring-cron)
        return cron.next(from);
    }
}

@Scheduled(fixedDelay = 1000)  // every second
@Transactional
public void pollAndDispatch() {
    List<ScheduledJob> dueJobs = jobRepository.claimDueJobs(instanceId, 50);

    for (ScheduledJob job : dueJobs) {
        JobDispatchMessage message = new JobDispatchMessage(
            job.getId(), job.getJobDefinition(), job.getExecutionContext()
        );
        kafkaTemplate.send("job-dispatch", job.getId().toString(), message);

        log.info("Dispatched job {} (type: {})", job.getId(), job.getJobType());
    }
}
```

**Job history and observability:**
```sql
CREATE TABLE job_runs (
    run_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id        UUID REFERENCES scheduled_jobs(id),
    started_at    TIMESTAMP NOT NULL,
    completed_at  TIMESTAMP,
    status        TEXT CHECK (status IN ('RUNNING', 'SUCCEEDED', 'FAILED', 'TIMED_OUT')),
    duration_ms   INTEGER,
    error_message TEXT,
    worker_id     TEXT
);

-- Index for job history queries
CREATE INDEX idx_job_runs_job_id_started ON job_runs(job_id, started_at DESC);
```

**Failure scenarios:**
- **Scheduler instance crashes after claiming but before dispatching:** Job row has `status='RUNNING'` with a stale `claimed_at`. A watchdog query: `UPDATE scheduled_jobs SET status='ACTIVE' WHERE status='RUNNING' AND claimed_at < NOW() - INTERVAL '5 minutes'` recovers stalled jobs.
- **Worker crashes during execution:** Job run entry has no `completed_at`. Dead-letter queue + alerting. Retry policy is job-type-specific.
- **DB down:** Schedulers fail gracefully. Kafka buffers already-dispatched jobs. Recovery: replay the dispatch after DB recovery.

---

## PART 5 — ADVANCED HLD: SEARCH AUTOCOMPLETE

---

### Design a Search Autocomplete / Typeahead System

**Step 1 — Requirements:**

Functional:
- As user types, return top 10 matching suggestions
- Results ranked by relevance (search frequency, recency)
- Support for prefix matching ("pay" → "payment", "paytm", "payout")
- Near real-time index updates (new queries indexed within 60 seconds)

Non-Functional:
- 1B searches/day → ~12K QPS
- Autocomplete latency < 100ms p99
- Corpus: 100M unique search terms

**Step 2 — Data Structures:**

**Trie (Prefix Tree):** Each node represents a character. Path from root to node = prefix. Each node also stores the top-K suggestions for that prefix.

```
Root
├── p
│   ├── pa
│   │   ├── pay → ["payment gateway", "payment status", "paytm"]
│   │   └── par
│   └── pi
└── s
    └── se
        └── sea
            └── search → ["search payments", "search partners"]
```

**Problem at scale:** A Trie with 100M terms takes ~10GB in memory. Updates require re-traversal from the changed node to root to update top-K lists. Distributed trie is complex.

**Practical approach — Redis Sorted Set per prefix:**
For each prefix of each search term, store the term in a sorted set with its frequency as score:

```
ZADD prefix:pay  1500 "payment gateway"
ZADD prefix:pay  1200 "payment status"
ZADD prefix:pay   900 "paytm"
ZADD prefix:paym  1500 "payment gateway"
ZADD prefix:paym  1200 "payment status"
...
```

Query: `ZREVRANGE prefix:{input} 0 9` → top 10 suggestions in O(log N + K).

**The storage concern:** Each term of length L generates L prefix keys. 100M terms, average length 10 = 1B keys. At ~100 bytes each = 100GB Redis.

**Mitigation:** Only store prefixes of length >= 2. Use compression. Shard across multiple Redis nodes by prefix hash.

**Full architecture:**

```
User types → Frontend debounce (50ms) → GET /autocomplete?q=pay
                                              │
                                    ┌─────────▼─────────┐
                                    │  Autocomplete Svc  │
                                    │  (stateless, 10    │
                                    │   instances)       │
                                    └─────────┬──────────┘
                                              │
                              ┌───────────────┼──────────────────┐
                              ▼               ▼                  ▼
                        Redis Cluster    Redis Cluster    Redis Cluster
                        (prefixes a-h)  (prefixes i-p)  (prefixes q-z)

Search event stream:
User completes search → Kafka "search-events"
                            │
                    ┌───────▼──────────┐
                    │  Aggregator Svc  │
                    │  (tumbling 5min  │
                    │   windows)       │
                    └───────┬──────────┘
                            │
                    Update Redis sorted sets with incremented frequency scores
```

```java
@RestController
public class AutocompleteController {

    @GetMapping("/autocomplete")
    public List<String> suggest(@RequestParam String q,
                                 @RequestParam(defaultValue = "10") int limit) {
        if (q == null || q.length() < 2) return Collections.emptyList();

        String prefix = q.toLowerCase().trim();
        String cacheKey = "prefix:" + prefix;

        Set<Object> suggestions = redisTemplate.opsForZSet()
            .reverseRange(cacheKey, 0, limit - 1);

        if (suggestions == null || suggestions.isEmpty()) {
            // Fall back to Elasticsearch for long-tail prefixes not in hot cache
            return elasticsearchService.prefixSearch(prefix, limit);
        }

        return suggestions.stream().map(Object::toString).toList();
    }
}

// Frequency updater
@Component
public class SearchFrequencyUpdater {

    @KafkaListener(topics = "search-events")
    public void handleSearchEvent(SearchEvent event) {
        String term = event.getQuery().toLowerCase().trim();
        // Update all prefix lengths for this term
        for (int i = 2; i <= Math.min(term.length(), 20); i++) {
            String prefix = term.substring(0, i);
            redisTemplate.opsForZSet()
                .incrementScore("prefix:" + prefix, term, 1.0);
            // Trim to top 100 per prefix (keep hot list small)
            redisTemplate.opsForZSet().removeRange("prefix:" + prefix, 0, -101);
        }
    }
}
```

---

## PART 6 — FINANCIAL RECONCILIATION SYSTEM DESIGN

---

### Design a Financial Reconciliation System

**Problem Statement:**
In our payment platform, money flows through multiple systems: our database records a payment as "completed," Razorpay's settlement report shows a different amount, and the partner's bank statement shows yet another. Reconciliation identifies and resolves these discrepancies.

**The Three Ledgers:**

```
Our System DB     ←→    Razorpay Settlement    ←→    Bank Statement
(internal record)        (payment processor)          (ground truth)
```

Discrepancy types:
1. **Missing from our DB, present in Razorpay:** Webhook was lost — need to replay
2. **Present in our DB, missing from Razorpay:** Ghost payment — investigate
3. **Amount mismatch:** Fee calculation differences — check gateway deduction rules
4. **Status mismatch:** Our DB says "completed," Razorpay says "reversed"

**Architecture:**

```
┌──────────────────────────────────────────────────────────────┐
│                   Data Ingestion Layer                        │
│                                                              │
│  Razorpay Settlement API → S3 (raw files) → Normalizer       │
│  Bank Statement (SFTP)   → S3 (raw files) → Normalizer       │
│  Our DB export           → S3 (snapshots) → Normalizer       │
└──────────────────────────────┬───────────────────────────────┘
                               │
                    ┌──────────▼────────────┐
                    │  Reconciliation Engine │
                    │  (comparison + match)  │
                    └──────────┬────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
       Matched records    Discrepancies    Exceptions
       (auto-close)       (auto-flag)      (manual review)
              │                │
              ▼                ▼
         Audit Trail      Discrepancy Queue
                          (Jira/email/dashboard)
```

**Reconciliation Engine:**

```java
@Service
public class ReconciliationEngine {

    public ReconciliationResult reconcile(ReconciliationBatch batch) {
        List<TransactionRecord> internal = batch.getInternalRecords();
        List<TransactionRecord> external = batch.getExternalRecords();  // Razorpay

        Map<String, TransactionRecord> internalByRef = internal.stream()
            .collect(Collectors.toMap(TransactionRecord::getPaymentReference, r -> r));

        Map<String, TransactionRecord> externalByRef = external.stream()
            .collect(Collectors.toMap(TransactionRecord::getPaymentReference, r -> r));

        List<MatchedRecord> matched = new ArrayList<>();
        List<Discrepancy> discrepancies = new ArrayList<>();

        // Check every internal record against external
        for (Map.Entry<String, TransactionRecord> entry : internalByRef.entrySet()) {
            String ref = entry.getKey();
            TransactionRecord internal_rec = entry.getValue();
            TransactionRecord external_rec = externalByRef.get(ref);

            if (external_rec == null) {
                discrepancies.add(Discrepancy.missingInExternal(internal_rec));
                continue;
            }

            if (!internal_rec.getAmount().equals(external_rec.getAmount())) {
                discrepancies.add(Discrepancy.amountMismatch(internal_rec, external_rec));
                continue;
            }

            if (!internal_rec.getStatus().equals(external_rec.getStatus())) {
                discrepancies.add(Discrepancy.statusMismatch(internal_rec, external_rec));
                continue;
            }

            matched.add(new MatchedRecord(internal_rec, external_rec));
        }

        // External records missing from internal (lost webhooks)
        for (String ref : externalByRef.keySet()) {
            if (!internalByRef.containsKey(ref)) {
                discrepancies.add(Discrepancy.missingInInternal(externalByRef.get(ref)));
            }
        }

        return new ReconciliationResult(matched, discrepancies);
    }
}
```

**Auto-resolution rules:**
```java
@Service
public class DiscrepancyResolver {

    public ResolutionAction resolve(Discrepancy discrepancy) {
        return switch (discrepancy.getType()) {

            case MISSING_IN_INTERNAL -> {
                // Lost webhook — replay from Razorpay data
                yield ResolutionAction.REPLAY_WEBHOOK(discrepancy.getExternalRecord());
            }

            case AMOUNT_MISMATCH -> {
                BigDecimal diff = discrepancy.getAmountDifference().abs();
                if (diff.compareTo(new BigDecimal("1.00")) <= 0) {
                    // < ₹1 difference = rounding/fee tolerance — auto-accept
                    yield ResolutionAction.AUTO_ACCEPT_WITH_NOTE("Fee rounding variance");
                } else {
                    yield ResolutionAction.ESCALATE_TO_FINANCE(discrepancy);
                }
            }

            case STATUS_MISMATCH -> {
                // External (Razorpay) is ground truth — update internal status
                yield ResolutionAction.SYNC_STATUS_FROM_EXTERNAL(discrepancy);
            }

            case MISSING_IN_EXTERNAL ->
                ResolutionAction.ESCALATE_TO_FINANCE(discrepancy);  // potential fraud
        };
    }
}
```

**Scheduling — daily reconciliation at 2 AM:**
```java
@Scheduled(cron = "0 0 2 * * *")  // 2 AM daily
public void runDailyReconciliation() {
    LocalDate yesterday = LocalDate.now().minusDays(1);
    
    ReconciliationBatch batch = dataIngestionService.fetchBatch(yesterday);
    ReconciliationResult result = reconciliationEngine.reconcile(batch);
    
    result.getDiscrepancies().forEach(discrepancy -> {
        ResolutionAction action = discrepancyResolver.resolve(discrepancy);
        action.execute();
        discrepancyAuditRepository.save(new DiscrepancyAuditEntry(discrepancy, action));
    });
    
    reconciliationReportService.generateAndEmail(result, financeTeamEmails);
    
    metrics.gauge("reconciliation.discrepancy.count", result.getDiscrepancies().size());
    metrics.gauge("reconciliation.match.rate",
        (double) result.getMatched().size() / result.getTotalRecords() * 100);
}
```

**Production insights from your project:**
"We run this at 2 AM daily. In 8 months, our match rate has been consistently 99.7%. The 0.3% discrepancies are almost always one of three things: (1) Razorpay settled a payment after midnight on day boundaries — resolved by running a 3-day rolling window reconciliation in addition to the daily snapshot, (2) Network retries that created duplicate webhook entries we hadn't idempotently handled at the time — the 5-layer idempotency system now prevents these, (3) Currency rounding for UPI micropayments below ₹1 — resolved by a tolerance band in the amount comparison."

---

## PART 7 — RESUME BULLET REWRITING WITH STAR FORMULA

---

### Before & After: Resume Bullets Transformed

The rule for Senior resume bullets: **Impact first, then mechanism.** Lead with the outcome, follow with the technical approach.

---

**Original (weak):**
> "Developed webhook processing system using Kafka and Redis"

**Rewritten (strong):**
> "Architected 5-layer idempotent webhook pipeline (Redis deduplication + Kafka buffer + DB unique constraint + state machine) that processed 50,000+ Razorpay callbacks with zero duplicate credits over 8 months"

**Why it's better:** Quantifies scale, names the specific problem solved (idempotency, not just 'webhook processing'), includes the outcome (zero duplicate credits), and shows system design depth.

---

**Original (weak):**
> "Worked on multi-tenant authentication for ThingsBoard platform"

**Rewritten (strong):**
> "Designed 4-layer multi-tenant auth system (custom AuthenticationProvider + JWT tenant claims + ThreadLocal context propagation + Hibernate row-level filter) enabling complete data isolation for 200+ enterprise tenants on shared infrastructure"

**Why it's better:** Describes the architecture layers, quantifies the tenant count, names the specific outcome (data isolation), shows both security and system design thinking.

---

**Original (weak):**
> "Fixed race condition in credit limit feature"

**Rewritten (strong):**
> "Diagnosed and resolved concurrent credit limit race condition under high lead assignment load; implemented progression from pessimistic locking to optimistic @Version to Redis Lua atomic decrement — reducing DB lock contention by ~80% while maintaining zero over-credit incidents"

**Why it's better:** Names the evolution of the solution (not just the final state), quantifies the improvement, and demonstrates that you understood the trade-offs at each step.

---

**Original (weak):**
> "Upgraded the application from Java 11 to Java 17"

**Rewritten (strong):**
> "Led zero-downtime Java 11 → 17 migration for production IoT platform serving 200 tenants; resolved 3 breaking changes (Mockito reflection, Jackson internal API usage, sun.misc.Unsafe replacement with VarHandle), achieving 15% startup time reduction and 12% memory improvement"

**Why it's better:** Proves the migration was non-trivial (names specific blockers), quantifies the post-migration improvements, and shows ownership (led, not just participated).

---

**Original (weak):**
> "Implemented caching using Redis for performance improvement"

**Rewritten (strong):**
> "Designed multi-level caching strategy (L1 Caffeine 30s TTL + L2 Redis 15min TTL + staggered jitter) with probabilistic early expiration to eliminate cache stampede; reduced p99 partner configuration lookup from 45ms to < 2ms under sustained load"

**Why it's better:** Names the specific problem solved (stampede prevention), describes the full solution architecture, quantifies the improvement with specific numbers.

---

**Original (weak):**
> "Designed payment reconciliation system"

**Rewritten (strong):**
> "Built automated daily reconciliation engine comparing 10,000+ transactions across three ledgers (internal DB, Razorpay settlement, bank statement); implemented auto-resolution for 4 discrepancy types maintaining 99.7% match rate with < ₹1 tolerance band for rounding variance"

**Why it's better:** Quantifies the data volume, names the complexity (three ledgers), shows the engineering precision (auto-resolution rules, tolerance band).

---

### Interview Verbal Bridge Formula

When your resume bullet comes up, use this formula to expand it verbally:

**Template:**
> "That bullet represents [the business problem]. The technical challenge was [the core constraint that made it hard]. I solved it by [your specific architectural decision], and the key trade-off I made was [what you gave up vs what you gained]. The outcome was [measurable result]."

**Example:**
> "That bullet represents the fact that payment webhooks from Razorpay are unreliable — they arrive out of order, get retried, and any duplicate processing means double-crediting a partner's account. The core challenge was: how do you handle at-least-once delivery from an external system when the consequence of processing twice is financial. I solved it with a 5-layer defense — each layer catches what slips through the previous one. The key trade-off was using Kafka over the Outbox Pattern for the buffer layer — Kafka is operationally simpler but introduces a tiny window of event loss if both the DB write and Kafka publish fail simultaneously. We accepted that trade-off because Razorpay retries for 24 hours, giving us a recovery window. The outcome was zero duplicate credits over 8 months of production on 50,000+ events."

---

## QUICK-REFERENCE — Senior Filter Questions (Chapter 5)

**Q: Fan-out-on-write vs fan-out-on-read — which is faster to read?**
Fan-out-on-write (push): reads are instant (just read pre-built feed). Fan-out-on-read (pull): reads require merging from multiple sources at query time — slower, especially with many followees. Instagram, Twitter use hybrid: push for normal users, pull for celebrities. The threshold is typically ~10K–50K followers.

**Q: Why is `SKIP LOCKED` better than retrying a regular `SELECT FOR UPDATE` for job schedulers?**
`SELECT FOR UPDATE` (without SKIP LOCKED) blocks: all scheduler instances queue up waiting for the same rows, causing contention. `SKIP LOCKED` immediately skips rows locked by other transactions and grabs the next available ones. Result: N scheduler instances can claim N non-overlapping batches simultaneously with no blocking — horizontal scaling of the scheduler itself.

**Q: What's the difference between a Trie and a Redis sorted set for autocomplete? When is each better?**
Trie: memory-efficient for common prefixes, O(prefix_length) lookup, natural for prefix matching, hard to distribute. Redis sorted set per prefix: simple, horizontally shardable, natively handles scoring (frequency), but O(average_term_length) storage multiplier. Use Redis sorted sets in production — operational simplicity and shardability outweigh the memory overhead for most scales. Trie is the right answer for an in-process autocomplete (mobile app, embedded search).

**Q: How do you handle time zones in a scheduled job system?**
Store all scheduled times in UTC. Store the user's expressed cron expression with their timezone metadata separately. At dispatch time, convert: `next_run_utc = cron.nextRun(expression, timezone).toInstant()`. Handle DST transitions carefully — a job scheduled for 2:30 AM may not fire or may fire twice during spring/fall clock changes. The safest approach: use the timezone's UTC offset at scheduling time and re-evaluate on DST boundaries.

**Q: What's the difference between reconciliation and idempotency?**
Idempotency prevents duplicates from entering the system in the first place (same-request multiple times = same result). Reconciliation detects and corrects discrepancies that occurred across system boundaries — where idempotency controls don't span (our DB vs external payment processor's records). They're complementary: idempotency is proactive defense, reconciliation is retroactive detection.

---

---

## PART 8 — DEPLOYMENT STRATEGIES

---

### How do you deploy without downtime? Walk through blue-green, canary, and rolling deployments.

**The answer:**

"There are four main strategies. The right choice depends on your risk tolerance, infrastructure cost, and whether your changes are backward-compatible.

---

**Rolling Deployment (default in Kubernetes):**
Replace instances one by one. At any point, some instances run the old version, some run the new version. Traffic is distributed across both.

```yaml
# Kubernetes Deployment strategy
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # allow 1 extra pod above desired count during rollout
      maxUnavailable: 0  # never take a pod down before new one is healthy
```

```
Before: [v1] [v1] [v1] [v1]
Step 1: [v1] [v1] [v1] [v2]  ← new pod healthy, old pod removed
Step 2: [v1] [v1] [v2] [v2]
Step 3: [v1] [v2] [v2] [v2]
After:  [v2] [v2] [v2] [v2]
```

Pro: No extra infrastructure needed, gradual rollout.
Con: Both versions live simultaneously — your DB schema changes must be backward-compatible with both v1 and v2 (expand-contract pattern from Flyway section).
Rollback: `kubectl rollout undo deployment/payment-api`

---

**Blue-Green Deployment:**
Two identical environments: Blue (live) and Green (new version). Switch all traffic atomically when Green is verified.

```
Load Balancer
    │
    ├── Blue (v1, live)    ← 100% traffic
    └── Green (v2, idle)   ← 0% traffic (being prepared)

After testing Green:
    ├── Blue (v1, idle)    ← 0% traffic (kept for instant rollback)
    └── Green (v2, live)   ← 100% traffic
```

```bash
# AWS ALB weighted target groups
aws elbv2 modify-rule \
  --rule-arn $RULE_ARN \
  --actions Type=forward,ForwardConfig='{
    "TargetGroups": [
      {"TargetGroupArn": "$GREEN_TG", "Weight": 100},
      {"TargetGroupArn": "$BLUE_TG",  "Weight": 0}
    ]
  }'
```

Pro: Instant rollback (switch back to Blue immediately). Zero-downtime deployment. Full production validation before switch.
Con: Double infrastructure cost (two full environments). DB schema changes: both environments must share the DB, so schema must be compatible with both versions simultaneously.

---

**Canary Deployment:**
Gradually shift traffic from old to new — 1% → 5% → 25% → 50% → 100%. Monitor error rates and latency at each step before proceeding.

```
Load Balancer
    ├── v1 (stable)   ← 95% traffic
    └── v2 (canary)   ← 5% traffic

Monitor: error rate, p99 latency, business metrics (payment success rate)
If metrics are good → increase to 25%
If metrics degrade → rollback to 0%
```

```yaml
# Kubernetes with Argo Rollouts (canary controller)
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5     # 5% to canary
        - pause: {duration: 10m}  # wait 10 min, check metrics
        - setWeight: 25
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {}        # manual approval to proceed to 100%
      analysis:
        templates:
          - templateName: payment-success-rate
        startingStep: 1
        args:
          - name: error-rate-threshold
            value: "0.01"  # rollback if error rate > 1%
```

Pro: Real user validation with minimal blast radius. Fine-grained control.
Con: Complex routing configuration. Requires monitoring infrastructure to make automated go/no-go decisions.

---

**Feature Flags (decouple deploy from release):**
Deploy the code to 100% of instances but control who sees the new feature via a flag. Allows instant kill switch without redeployment.

```java
@Service
public class PaymentService {

    private final FeatureFlagService flags;

    public PaymentResult processPayment(Payment payment) {
        if (flags.isEnabled("new-fraud-detection", payment.getTenantId())) {
            return newFraudDetectionFlow(payment);  // new code path
        }
        return legacyPaymentFlow(payment);  // old code path
    }
}

// Feature flag service backed by Redis or LaunchDarkly
@Service
public class FeatureFlagService {
    public boolean isEnabled(String flag, String tenantId) {
        // Check: global toggle, then tenant-specific override, then percentage rollout
        String key = "ff:" + flag + ":" + tenantId;
        String override = redisTemplate.opsForValue().get(key);
        if (override != null) return Boolean.parseBoolean(override);

        // Default: percentage-based rollout
        FlagConfig config = flagConfigRepository.findByName(flag);
        int hash = Math.abs(tenantId.hashCode() % 100);
        return hash < config.getRolloutPercentage();
    }
}
```

**Combine strategies:**
Production-grade deployments often combine: rolling deploy (zero downtime, no extra infra) + feature flags (control feature exposure independently of deploy). Blue-green is used for major, high-risk releases where instant rollback is non-negotiable.

**Zero-downtime deployment requirements:**
A deploy is zero-downtime only if ALL of these hold:
1. Application is stateless (no in-memory session)
2. DB schema is backward-compatible with the previous version (old app reads new schema without error)
3. Graceful shutdown is configured (in-flight requests complete)
4. Health checks are wired to readiness probe (new pod only receives traffic after it's ready)
5. At least 2 replicas (rolling update doesn't kill all at once)

---

> **What's Next — Chapter 6:** Full Behavioral & Mock Interview Simulation — additional STAR stories, the complete mock interview loop protocol, company-specific research framework, and DSA coding screen patterns.

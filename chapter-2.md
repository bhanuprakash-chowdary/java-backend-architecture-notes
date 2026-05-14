# Interview Survival Guide
## Chapter 2: Concurrency, Databases & Messaging
### (Weeks 7, 8, 14 & 15)

---

## PART 1 — JAVA MULTITHREADING & CONCURRENCY

---

### Q1. `synchronized` vs `ReentrantLock` vs `StampedLock` — when do you use each?

**The answer:**

"These three mechanisms exist on a spectrum of capability vs complexity. I'll walk through each and when to choose it.

**`synchronized`:**
The simplest option. Built into Java, no imports needed. Automatically releases on exit (normal or exception). Cannot be interrupted, cannot try-lock with timeout, uses an intrinsic monitor on the object.

```java
public class Counter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int get() {
        return count;
    }
}
```

**When to use `synchronized`:** Simple mutual exclusion with short critical sections, no need for try-lock or timed-lock, no complex lock ordering requirements.

**`ReentrantLock`:**
Explicit lock/unlock. Supports: `tryLock(timeout)`, `lockInterruptibly()`, `newCondition()` for fine-grained wait/notify, and fairness policy (FIFO waiting queue).

```java
public class BoundedQueue<T> {
    private final ReentrantLock lock = new ReentrantLock(true);  // fair
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final Queue<T> queue;
    private final int capacity;

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();  // release lock and wait
            }
            queue.add(item);
            notEmpty.signal();  // wake one waiting consumer
        } finally {
            lock.unlock();  // ALWAYS in finally
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }
            T item = queue.poll();
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

**When to use `ReentrantLock`:** Need `tryLock()` to avoid deadlocks, need timed locking, need multiple `Condition` objects (separate wait queues for producers vs consumers), need fair ordering.

**`StampedLock` (Java 8+):**
Optimized for read-heavy workloads. Three modes: write lock (exclusive), read lock (shared), and **optimistic read** (no lock at all — just validates after reading).

```java
public class Point {
    private double x, y;
    private final StampedLock lock = new StampedLock();

    // Write — exclusive
    public void move(double dx, double dy) {
        long stamp = lock.writeLock();
        try { x += dx; y += dy; }
        finally { lock.unlockWrite(stamp); }
    }

    // Optimistic read — try without locking, validate, fall back
    public double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();  // no lock acquired
        double currentX = x, currentY = y;
        if (!lock.validate(stamp)) {         // was there a write during our read?
            stamp = lock.readLock();         // fall back to real read lock
            try { currentX = x; currentY = y; }
            finally { lock.unlockRead(stamp); }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

**When to use `StampedLock`:** High-read, low-write contention (e.g., configuration cache, routing tables). Note: `StampedLock` is NOT reentrant — a thread that holds it cannot acquire it again. Never use in code with re-entrant patterns.

**Decision table:**

| Scenario | Use |
|----------|-----|
| Simple counter, shared state | `synchronized` |
| Need tryLock(), Condition | `ReentrantLock` |
| High read/low write, performance-critical | `StampedLock` |
| No locking at all (single writer) | `AtomicReference`, `CopyOnWriteArrayList` |

---

### Q2. `volatile`, `AtomicInteger`, and CAS — what's the difference?

**The answer:**

"These solve different problems in concurrent programming.

**`volatile`** — provides **visibility**, not atomicity.
It guarantees that reads and writes go directly to main memory, not CPU caches. Every write by any thread is immediately visible to all other threads.

```java
public class ServerState {
    private volatile boolean running = true;  // visibility guarantee
    
    public void shutdown() {
        running = false;  // write immediately visible to all threads
    }
    
    public void run() {
        while (running) {  // reads fresh value every time
            processRequest();
        }
    }
}
```

**`volatile` is NOT enough for compound actions:**
```java
private volatile int counter = 0;

// BROKEN — read-modify-write is not atomic even with volatile
counter++;  // 3 operations: read, increment, write — not atomic
```

**`AtomicInteger`** — provides **atomicity + visibility** via CAS.
```java
private final AtomicInteger counter = new AtomicInteger(0);

counter.incrementAndGet();          // atomic increment
counter.compareAndSet(5, 10);       // atomic: if current == 5, set to 10
int prev = counter.getAndAdd(5);    // atomic: return current, add 5
```

**CAS (Compare-And-Swap):**
CAS is a CPU-level atomic instruction: `CAS(address, expected, newValue)` — atomically: if `*address == expected`, set `*address = newValue` and return true; otherwise return false.

Java's `AtomicInteger.compareAndSet()` maps directly to the CPU's `LOCK CMPXCHG` instruction.

```java
// What incrementAndGet() conceptually does:
public int incrementAndGet() {
    int current, next;
    do {
        current = get();       // read current value
        next = current + 1;    // compute new value
    } while (!compareAndSet(current, next));  // retry if current changed under us
    return next;
}
```

**ABA Problem:**
CAS only compares values, not history. If a value changes A→B→A between your read and CAS, your CAS succeeds thinking nothing changed — but something did.

```
Thread 1 reads: counter = 5
Thread 2 changes: 5 → 10 → 5
Thread 1 CAS: (expected=5, new=6) → succeeds! But the intermediate state was lost.
```

Fix: `AtomicStampedReference<V>` — pairs the value with a version stamp. CAS checks both value AND stamp.

```java
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(5, 0);
int[] stampHolder = new int[1];
int val = ref.get(stampHolder);
ref.compareAndSet(val, val + 1, stampHolder[0], stampHolder[0] + 1);
// Only succeeds if both value and stamp match
```

---

### Q3. How do you size a `ThreadPoolExecutor`? What happens when the queue is full?

**The answer:**

"A `ThreadPoolExecutor` has 7 parameters. Understanding all of them is a Senior filter:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    corePoolSize,          // threads always alive (even idle)
    maximumPoolSize,       // max threads (capped here)
    keepAliveTime,         // idle time before non-core threads die
    timeUnit,
    workQueue,             // task buffer between acceptance and execution
    threadFactory,         // how new threads are created (naming, daemon, priority)
    rejectionHandler       // what to do when queue AND max threads are full
);
```

**Execution flow (critical — many get this wrong):**
1. If running threads < `corePoolSize`: start new thread
2. If running threads >= `corePoolSize` AND queue not full: enqueue task
3. If queue full AND running threads < `maximumPoolSize`: start new thread
4. If queue full AND running threads == `maximumPoolSize`: apply `RejectionPolicy`

**Note the order:** New threads above core are only created when the queue is FULL. This surprises people who expect core → extra threads → queue. It's core → queue → extra threads → rejection.

**Queue types:**
- `LinkedBlockingQueue(n)`: bounded, back-pressure when full
- `LinkedBlockingQueue()` (unbounded): never triggers rejection or max threads — effectively `maxPoolSize == corePoolSize`
- `SynchronousQueue`: zero capacity — hands off directly to a thread; if no free thread, goes to rejection. Used in `Executors.newCachedThreadPool()`
- `PriorityBlockingQueue`: tasks processed by priority, not FIFO

**Rejection Policies:**
- `AbortPolicy` (default): throws `RejectedExecutionException`
- `CallerRunsPolicy`: the calling thread executes the task (natural back-pressure)
- `DiscardPolicy`: silently drops the task
- `DiscardOldestPolicy`: drops the oldest queued task, retries current

**Sizing formulas:**
- **CPU-bound tasks:** `N_threads = N_cores + 1` (the +1 handles page faults)
- **I/O-bound tasks:** `N_threads = N_cores × (1 + wait_time / cpu_time)`
  - Example: If each DB call takes 100ms and CPU processing takes 10ms:
    `N_threads = 8 × (1 + 100/10) = 8 × 11 = 88 threads per node`
- **Mixed workloads:** Profile first. Use Micrometer to measure thread pool utilization: `executor.queue.size` and `executor.active`.

```java
// Production-quality thread pool setup for payment processing
ThreadPoolExecutor paymentExecutor = new ThreadPoolExecutor(
    10,                            // core: always-live threads for burst absorption
    50,                            // max: cap at 50 to protect DB connection pool
    60, TimeUnit.SECONDS,          // idle threads above core die after 60s
    new LinkedBlockingQueue<>(500), // bounded queue — 500 tasks max
    new ThreadFactory() {
        private final AtomicInteger id = new AtomicInteger();
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "payment-worker-" + id.incrementAndGet());
            t.setDaemon(false);
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy()  // back-pressure: caller processes if full
);
```

---

### Q4. Explain `CompletableFuture`. How did you use it in your payment service?

**The answer:**

"CompletableFuture is Java 8's implementation of a promise/future with a fluent, non-blocking composition API. Unlike `Future`, you can chain transformations, combine multiple futures, and handle errors declaratively.

**Core operations:**

```java
// Creating
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> callApi(), executor);

// Transforming (synchronous transformation, runs in completing thread)
CompletableFuture<Integer> length = cf.thenApply(s -> s.length());

// Chaining (async step that itself returns a future — flatMap)
CompletableFuture<Payment> payment = cf.thenCompose(id -> fetchPaymentAsync(id));

// Combining two independent futures
CompletableFuture<Result> combined = cf1.thenCombine(cf2, (r1, r2) -> merge(r1, r2));

// Waiting for all
CompletableFuture<Void> all = CompletableFuture.allOf(cf1, cf2, cf3);

// Waiting for first
CompletableFuture<Object> first = CompletableFuture.anyOf(cf1, cf2, cf3);

// Error handling (exceptionally = map for exceptions)
CompletableFuture<String> safe = cf.exceptionally(ex -> "default-value");

// Handle both success and failure in one place
CompletableFuture<String> handled = cf.handle((result, ex) -> {
    if (ex != null) return "error: " + ex.getMessage();
    return result.toUpperCase();
});
```

**Production example — parallel payment verification:**

```java
@Service
public class PaymentVerificationService {

    public PaymentVerificationResult verifyPayment(String paymentId) {
        // Run three checks in parallel
        CompletableFuture<FraudResult> fraudCheck = CompletableFuture
            .supplyAsync(() -> fraudService.check(paymentId), fraudExecutor)
            .orTimeout(2, TimeUnit.SECONDS)
            .exceptionally(ex -> FraudResult.unknown(ex));

        CompletableFuture<KycResult> kycCheck = CompletableFuture
            .supplyAsync(() -> kycService.check(paymentId), kycExecutor)
            .orTimeout(3, TimeUnit.SECONDS)
            .exceptionally(ex -> KycResult.pending());

        CompletableFuture<LimitResult> limitCheck = CompletableFuture
            .supplyAsync(() -> limitService.check(paymentId), limitExecutor);

        // Combine all three results
        return CompletableFuture.allOf(fraudCheck, kycCheck, limitCheck)
            .thenApply(v -> new PaymentVerificationResult(
                fraudCheck.join(),   // .join() safe here — already completed
                kycCheck.join(),
                limitCheck.join()
            ))
            .get(5, TimeUnit.SECONDS);  // overall timeout
    }
}
```

**Senior Signal:** "The key mistake people make with CompletableFuture is calling `.get()` on individual futures before all are submitted — that defeats the parallelism. Always submit all async operations first, then use `allOf()` to wait for completion. Also: never use the default ForkJoinPool for I/O-bound work — always pass a dedicated executor with I/O-sized thread count."

---

### Q5. `CountDownLatch`, `CyclicBarrier`, and `Semaphore` — when do you use each?

**The answer:**

"These are three different synchronization primitives in `java.util.concurrent`, each solving a different coordination problem.

**`CountDownLatch` — one-time gate:**
Initialize with a count N. N threads/operations call `countDown()`. Other threads block on `await()` until count reaches 0. Cannot be reset.

```java
// Use case: wait for N services to start before accepting traffic
CountDownLatch startupLatch = new CountDownLatch(3);

executor.submit(() -> { initDatabase(); startupLatch.countDown(); });
executor.submit(() -> { initCache(); startupLatch.countDown(); });
executor.submit(() -> { initKafka(); startupLatch.countDown(); });

startupLatch.await(30, TimeUnit.SECONDS);  // wait for all 3 to finish
server.start();  // safe to start now
```

**`CyclicBarrier` — reusable sync point:**
N threads each call `await()` and block until all N have called it. Once all arrive, they're all released and the barrier resets (reusable). Optional `Runnable` action fires when barrier trips.

```java
// Use case: batch processing phases — all workers must complete phase 1 before any starts phase 2
CyclicBarrier barrier = new CyclicBarrier(4, () -> {
    log.info("All workers completed phase — moving to next phase");
});

// Each worker thread:
processPhase1(myChunk);
barrier.await();  // wait for all 4 workers
processPhase2(myChunk);
barrier.await();  // wait for all 4 workers again
```

**`Semaphore` — bounded access control:**
Maintains N permits. `acquire()` takes a permit (blocks if none available). `release()` returns a permit. Used to limit concurrent access to a resource.

```java
// Use case: limit concurrent DB connections / API calls
Semaphore dbSemaphore = new Semaphore(10);  // max 10 concurrent queries

public List<Payment> queryPayments(String tenantId) {
    dbSemaphore.acquire();  // blocks if 10 queries already running
    try {
        return paymentRepository.findByTenantId(tenantId);
    } finally {
        dbSemaphore.release();  // always release
    }
}
```

**Quick decision:**

| Problem | Tool |
|---------|------|
| Wait for N events to complete (one-time) | `CountDownLatch` |
| All N threads reach a checkpoint before any continues | `CyclicBarrier` |
| Limit concurrent access to a resource | `Semaphore` |
| Coordinate producer-consumer with capacity | `BlockingQueue` |

---

## PART 2 — THE CREDIT LIMIT RACE CONDITION

---

### Q6. Tell me about a complex concurrency problem you solved in production. Walk me through the full solution.

**Your exact conversational answer:**

"Sure. This is actually the most interesting concurrency problem I've worked on, and it had direct financial consequences if we got it wrong.

**The Context:**
We were building a credit facility feature for partners in our lead assignment system. Each partner had a credit limit — a cap on how much outstanding credit they could carry at any time. Whenever a new lead was assigned to a partner, we'd deduct from their available credit. When a lead was settled or rejected, we'd return it.

**The Problem:**
Under high load, we'd have multiple leads being assigned to the same partner concurrently. The naive implementation had a classic race condition:

```java
// BROKEN CODE — race condition
@Transactional
public void assignLead(UUID partnerId, Lead lead) {
    Partner partner = partnerRepository.findById(partnerId).get();
    
    if (partner.getAvailableCredit() >= lead.getCreditRequired()) {
        // Thread A and Thread B both read availableCredit = 100
        // Thread A: 100 - 40 = 60
        // Thread B: 100 - 40 = 60
        // Both pass the check. Both subtract. Final credit = 60, not 20.
        partner.setAvailableCredit(partner.getAvailableCredit() - lead.getCreditRequired());
        partnerRepository.save(partner);
        leadRepository.assignToPartner(lead, partner);
    } else {
        throw new InsufficientCreditException();
    }
}
```

Both threads read the same `availableCredit`, both pass the check, both subtract — you've now over-extended by one lead's worth of credit. In a fintech context, this is real money.

**Solution 1 — Pessimistic Locking (SELECT FOR UPDATE):**

```java
@Transactional
public void assignLead(UUID partnerId, Lead lead) {
    // Lock the partner row — other threads block here
    Partner partner = partnerRepository.findByIdForUpdate(partnerId);
    
    if (partner.getAvailableCredit() >= lead.getCreditRequired()) {
        partner.deductCredit(lead.getCreditRequired());
        partnerRepository.save(partner);
        leadRepository.assignToPartner(lead, partner);
    } else {
        throw new InsufficientCreditException();
    }
    // Lock released when @Transactional commits
}

// Repository
@Query("SELECT p FROM Partner p WHERE p.id = :id FOR UPDATE")
Partner findByIdForUpdate(@Param("id") UUID id);
```

This works and is correct. But under high concurrency, threads queue up waiting for the lock — throughput degrades linearly with the number of concurrent requests.

**Solution 2 — Optimistic Locking (@Version):**

```java
@Entity
public class Partner {
    @Id UUID id;
    BigDecimal availableCredit;
    
    @Version
    Long version;  // Hibernate manages this automatically
}

@Transactional
public void assignLead(UUID partnerId, Lead lead) {
    Partner partner = partnerRepository.findById(partnerId).get();  // no lock

    if (partner.getAvailableCredit() < lead.getCreditRequired()) {
        throw new InsufficientCreditException();
    }
    
    partner.deductCredit(lead.getCreditRequired());
    partnerRepository.save(partner);
    // Hibernate generates: UPDATE partner SET credit=?, version=v+1 WHERE id=? AND version=v
    // If version changed (another thread committed first), throws ObjectOptimisticLockingFailureException
}
```

Optimistic locking has no blocking, but under high contention you get many retries. I added a retry wrapper:

```java
@Retryable(value = ObjectOptimisticLockingFailureException.class, 
           maxAttempts = 3,
           backoff = @Backoff(delay = 50, multiplier = 2))
public void assignLeadWithRetry(UUID partnerId, Lead lead) {
    assignLead(partnerId, lead);
}
```

**Solution 3 — Redis Lua Script (the scale path):**

When we needed to scale beyond single-node and the database was becoming a bottleneck, I moved to an atomic Redis Lua script:

```lua
-- Atomic credit check + deduct
local key = KEYS[1]          -- "partner:{id}:credit"
local required = tonumber(ARGV[1])

local available = tonumber(redis.call('GET', key))
if available == nil then
    return -1  -- key not initialized
end

if available < required then
    return 0   -- insufficient credit
end

redis.call('DECRBY', key, required)
return 1  -- success
```

```java
private static final String CREDIT_DEDUCT_SCRIPT = "..."; // Lua above
private final RedisScript<Long> deductScript = new DefaultRedisScript<>(CREDIT_DEDUCT_SCRIPT, Long.class);

public void assignLead(UUID partnerId, Lead lead) {
    String creditKey = "partner:" + partnerId + ":credit";
    Long result = redisTemplate.execute(deductScript, 
        List.of(creditKey), 
        String.valueOf(lead.getCreditRequired())
    );
    
    if (result == 0) throw new InsufficientCreditException();
    if (result == -1) {
        // Redis key not found — fall back to DB
        assignLeadWithPessimisticLock(partnerId, lead);
        return;
    }
    
    // Asynchronously sync to DB
    leadRepository.assignToPartner(lead, partnerId);
    // Note: Redis is source of truth for real-time; DB is durable record
}
```

**Trade-off summary I give in interviews:**
- Pessimistic lock: correct, simple, doesn't scale horizontally
- Optimistic lock: correct, good for low-to-medium contention, retry storms under high contention
- Redis Lua: highest throughput, handles multi-node, but Redis is a soft state — you need periodic reconciliation against DB

We shipped pessimistic locking first (correct, simple, testable), measured in production, and migrated to Redis only after we could show the DB lock was a bottleneck."

---

## PART 3 — KAFKA DELIVERY GUARANTEES

---

### Q7. Explain Kafka delivery guarantees. How did you implement exactly-once semantics for webhook processing?

**The answer:**

"Kafka has three delivery semantics, and choosing the right one depends on what failure modes you're protecting against.

**At-most-once:**
Producer sends without acknowledgment (`acks=0`) OR consumer commits offset before processing. If the process crashes after commit but before processing — message is lost, never reprocessed.

```java
// Producer — fire and forget
props.put("acks", "0");  // don't wait for acknowledgment
```

**At-least-once:**
Producer uses `acks=all` (wait for all ISR replicas). Consumer processes THEN commits offset. If crash after processing but before commit — message is reprocessed. Duplicate delivery possible.

```java
// Producer — at least once
props.put("acks", "all");
props.put("retries", "3");
props.put("enable.idempotence", "false");

// Consumer — manual commit after processing
@KafkaListener(topics = "payments")
public void handle(ConsumerRecord<String, WebhookEvent> record) {
    processWebhook(record.value());  // process FIRST
    kafkaTemplate.commitOffset(record);  // then commit
}
```

**Exactly-once (EOS):**
Requires both idempotent producer AND transactional semantics across produce + consume.

```java
// Idempotent producer — deduplicates producer retries
props.put("enable.idempotence", "true");
props.put("transactional.id", "payment-processor-1");

// Each producer gets a PID (Producer ID) + sequence number per partition.
// Broker rejects duplicates with the same PID + sequence.
```

**The 5-layer idempotent webhook architecture from my project:**

In our Razorpay integration, webhooks arrive out-of-order (NEFT can arrive before the RTGS confirmation, or the same payment event fires twice). Here's the defense-in-depth we built:

```
Layer 1: Immediate HTTP 200 ACK
    └── Razorpay retries if it doesn't get 200 within 5s
        → Always ACK immediately, process async

Layer 2: Redis SETNX deduplication
    └── Key = "webhook:{event_id}", TTL = 24 hours
        → If SETNX returns 0, event already processing/processed → skip

Layer 3: Kafka buffer (at-least-once producer)
    └── Durable, ordered queue for processing
        → Even if consumer crashes, reprocessing is safe (Layer 2 catches dupes)

Layer 4: DB unique constraint
    └── UNIQUE INDEX on (payment_reference_id, event_type)
        → Last-resort deduplication if Redis TTL expired
        → INSERT ... ON CONFLICT DO NOTHING

Layer 5: Payment State Machine
    └── Valid transitions: PENDING → PROCESSING → COMPLETED/FAILED
        → COMPLETED → COMPLETED = no-op, not an error
        → Prevents double-credit even if all other layers fail
```

```java
@Component
public class RazorpayWebhookHandler {

    @PostMapping("/webhooks/razorpay")
    public ResponseEntity<Void> receiveWebhook(@RequestBody WebhookPayload payload,
                                                @RequestHeader("X-Razorpay-Signature") String sig) {
        // Layer 1: Verify signature + immediate ACK
        webhookSignatureValidator.validate(payload, sig);
        kafkaTemplate.send("webhook-events", payload.getEventId(), payload);
        return ResponseEntity.ok().build();  // ACK immediately
    }
}

@KafkaListener(topics = "webhook-events", groupId = "webhook-processor")
public class WebhookConsumer {

    public void consume(WebhookPayload payload) {
        String dedupKey = "webhook:" + payload.getEventId();

        // Layer 2: Redis SETNX
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(dedupKey, "processing", Duration.ofHours(24));
        if (Boolean.FALSE.equals(acquired)) {
            log.info("Duplicate webhook event {}, skipping", payload.getEventId());
            return;
        }

        try {
            processWebhookEvent(payload);  // Layers 4 + 5 inside this
            redisTemplate.opsForValue().set(dedupKey, "processed");
        } catch (Exception e) {
            redisTemplate.delete(dedupKey);  // allow retry
            throw e;
        }
    }

    @Transactional
    private void processWebhookEvent(WebhookPayload payload) {
        // Layer 4: DB unique constraint
        webhookEventRepository.insertIfNotExists(payload);  // ON CONFLICT DO NOTHING

        // Layer 5: State machine
        Payment payment = paymentRepository.findByReference(payload.getPaymentReference());
        paymentStateMachine.transition(payment, payload.getEventType());
    }
}

// Payment State Machine
public class PaymentStateMachine {
    private static final Map<PaymentStatus, Set<PaymentStatus>> VALID_TRANSITIONS = Map.of(
        PENDING,    Set.of(PROCESSING),
        PROCESSING, Set.of(COMPLETED, FAILED),
        COMPLETED,  Set.of(),   // terminal
        FAILED,     Set.of(PENDING)  // allow retry
    );

    public void transition(Payment payment, EventType event) {
        PaymentStatus target = eventToStatus(event);
        if (!VALID_TRANSITIONS.get(payment.getStatus()).contains(target)) {
            log.warn("Invalid transition {} → {} for payment {}, ignoring",
                     payment.getStatus(), target, payment.getId());
            return;  // no-op, not an error — idempotent
        }
        payment.setStatus(target);
        paymentRepository.save(payment);
    }
}
```

---

## PART 4 — REDIS CACHING PATTERNS

---

### Q8. Explain the three core caching strategies. When do you use each?

**The answer:**

"The three strategies are Cache-Aside, Write-Through, and Write-Behind. Each has different consistency and performance trade-offs.

**Cache-Aside (Lazy Loading):**
The application manages the cache. On read: check cache → if miss, load from DB → put in cache. Writes go directly to DB, cache entry invalidated or updated.

```java
public Partner getPartner(UUID partnerId) {
    String key = "partner:" + partnerId;

    // Check cache
    Partner cached = (Partner) redisTemplate.opsForValue().get(key);
    if (cached != null) return cached;

    // Cache miss — load from DB
    Partner partner = partnerRepository.findById(partnerId)
        .orElseThrow(() -> new NotFoundException("Partner " + partnerId));

    // Populate cache
    redisTemplate.opsForValue().set(key, partner, Duration.ofMinutes(15));
    return partner;
}

public void updatePartner(UUID partnerId, PartnerUpdate update) {
    partnerRepository.update(partnerId, update);
    redisTemplate.delete("partner:" + partnerId);  // invalidate — next read re-fetches
}
```

Pros: Only caches data that's actually read (no wasted space). DB is always the source of truth.
Cons: First read after miss or invalidation is slow (cache miss). Potential for stale data between write and invalidation (race condition).

**Write-Through:**
Every write goes to cache AND DB synchronously in the same operation.

```java
public void savePartner(Partner partner) {
    partnerRepository.save(partner);  // write to DB
    redisTemplate.opsForValue().set(
        "partner:" + partner.getId(), partner, Duration.ofMinutes(15)
    );  // write to cache immediately
}
```

Pros: Cache always consistent with DB (strong consistency). No stale reads.
Cons: Every write hits both DB and cache — write latency increases. Data written may never be read (wasted cache space).

**Write-Behind (Write-Back):**
Writes go to cache immediately (fast ACK to caller), then asynchronously flushed to DB.

```java
public void updateCreditLimit(UUID partnerId, BigDecimal newLimit) {
    // Write to cache immediately — fast response
    redisTemplate.opsForValue().set("credit:" + partnerId, newLimit);
    // Queue async flush to DB
    dbFlushQueue.offer(new CreditUpdate(partnerId, newLimit));
}

@Scheduled(fixedDelay = 1000)
public void flushToDatabase() {
    CreditUpdate update;
    while ((update = dbFlushQueue.poll()) != null) {
        partnerRepository.updateCredit(update.partnerId(), update.newLimit());
    }
}
```

Pros: Lowest write latency. Absorbs write bursts (queue smooths them out).
Cons: Risk of data loss if cache fails before flush. Complexity — need to handle flush failures, ordering.

**Your Project Angle:** "In the payment gateway, we used Cache-Aside for partner configuration data (read-heavy, rarely changes, tolerate brief staleness). For credit limits, we started with Write-Through because correctness was critical, then moved to a Redis-atomic model when we needed sub-millisecond reads during high-concurrency lead assignment."

---

### Q9. What is a cache stampede and how do you prevent it?

**The answer:**

"A cache stampede (also called thundering herd) happens when a popular cache key expires and thousands of concurrent requests all experience a cache miss simultaneously, all try to recompute the value from the DB, and overwhelm it.

**Timeline:**
```
T=0: key 'partner:config' expires
T=1ms: 1000 concurrent requests all miss the cache
T=2ms: 1000 requests all hit the DB with SELECT for partner config
T=3ms: DB collapses under 1000x normal load
```

**Prevention Strategy 1 — Distributed Mutex (Redis SETNX):**
Only one thread recomputes; others wait.

```java
public Partner getPartner(UUID partnerId) {
    String cacheKey = "partner:" + partnerId;
    String lockKey = "lock:partner:" + partnerId;

    Partner cached = cache.get(cacheKey);
    if (cached != null) return cached;

    // Try to acquire recomputation lock
    Boolean acquired = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "locked", Duration.ofSeconds(10));

    if (Boolean.TRUE.equals(acquired)) {
        try {
            Partner partner = partnerRepository.findById(partnerId).get();
            cache.set(cacheKey, partner, Duration.ofMinutes(15));
            return partner;
        } finally {
            redisTemplate.delete(lockKey);
        }
    } else {
        // Another thread is recomputing — wait briefly and retry
        Thread.sleep(50);
        return getPartner(partnerId);  // retry — now cache should be warm
    }
}
```

**Prevention Strategy 2 — Probabilistic Early Expiration (XFetch Algorithm):**
Proactively refresh the cache slightly before expiry — no thundering herd because only one "early" request triggers refresh.

```java
public <T> T getWithProbabilisticRefresh(String key, Supplier<T> loader,
                                          Duration ttl, double beta) {
    CachedValue<T> entry = cache.getWithExpiry(key);
    
    if (entry != null) {
        double remainingTtlMs = entry.getExpiryMs() - System.currentTimeMillis();
        // XFetch: refresh if remaining TTL < beta * recomputation_time * random()
        double recomputeTimeMs = 100; // estimated DB query time
        if (remainingTtlMs > beta * recomputeTimeMs * -Math.log(Math.random())) {
            return entry.getValue();  // cache hit, no refresh needed
        }
    }

    // Refresh
    T value = loader.get();
    cache.setWithExpiry(key, value, ttl);
    return value;
}
```

**Prevention Strategy 3 — Staggered TTLs:**
Add a small random jitter to TTL so popular keys don't all expire simultaneously.

```java
private Duration staggeredTtl(Duration baseTtl) {
    long jitterMs = (long) (Math.random() * 60_000);  // ±60 seconds jitter
    return baseTtl.plusMillis(jitterMs);
}

cache.set(key, value, staggeredTtl(Duration.ofMinutes(15)));
```

**Multi-level cache (L1 Caffeine + L2 Redis):**

```java
@Service
public class MultiLevelCacheService {
    private final Cache<String, Object> l1Cache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(30, TimeUnit.SECONDS)  // L1: very short TTL, in-process
        .build();
    private final RedisTemplate<String, Object> redisTemplate;

    public Object get(String key, Supplier<Object> loader) {
        // Check L1 (no network, nanoseconds)
        Object l1 = l1Cache.getIfPresent(key);
        if (l1 != null) return l1;

        // Check L2 (network, microseconds)
        Object l2 = redisTemplate.opsForValue().get(key);
        if (l2 != null) {
            l1Cache.put(key, l2);  // backfill L1
            return l2;
        }

        // Load from DB (milliseconds)
        Object value = loader.get();
        redisTemplate.opsForValue().set(key, value, Duration.ofMinutes(15));
        l1Cache.put(key, value);
        return value;
    }
}
```

---

## PART 5 — DATABASE INTERNALS & LOCKING

---

### Q10. Explain B-Tree vs B+Tree. Why do databases use B+Trees for indexes?

**The answer:**

"Both are self-balancing tree data structures, but they differ in how data is distributed across nodes.

**B-Tree:**
Every node (internal and leaf) can store both keys AND data pointers. The data pointer associated with a key lives wherever that key appears in the tree — internal nodes or leaves.

**B+Tree:**
Only leaf nodes contain data pointers. Internal nodes contain only keys for routing. All leaf nodes are linked in a doubly-linked list.

```
B+Tree for InnoDB primary key index:

         [30 | 70]           ← internal node: keys only, for routing
        /     |     \
   [10|20] [40|60] [80|90]   ← leaf nodes: keys + row pointers
      ↔        ↔       ↔     ← leaf nodes linked for range scans
```

**Why databases chose B+Trees over B-Trees:**

1. **Range queries are O(log n + k):** B+Tree range scan: traverse to the start key (O(log n)), then follow the leaf linked list to collect k results (O(k)). In a B-Tree, there's no linked list — you'd need an in-order traversal touching internal nodes too.

2. **Higher fanout:** Internal nodes contain only keys, not data — they're smaller. A disk page that fits 100 B-Tree entries might fit 1000 B+Tree internal entries (keys only, no data). Higher fanout = shallower tree = fewer disk reads to reach any leaf. For a 1TB table: B+Tree height ≈ 3-4 levels, regardless of table size.

3. **Sequential reads for full table scans:** Leaf linked list allows sequential scan without random I/O. Critical for OLAP / reporting queries.

4. **Consistent depth:** B+Tree guarantees all data at leaf level = consistent access time regardless of key value. In B-Tree, a frequently-accessed key near the root is faster than one buried in a leaf.

**The rule of thumb:**
Index read = O(log n) to find the first key + O(k) for range scan.
For a table with 1 billion rows, B+Tree height ≈ log_1000(10^9) ≈ 3 levels.

---

### Q11. Walk me through your index strategy for the payment system.

**The answer:**

"I'll cover the index types and the specific decisions we made.

**Index types:**

```sql
-- Single column: equality or range on one column
CREATE INDEX idx_payments_status ON payments(status);

-- Composite: multiple columns, order matters
-- Rule: equality predicates first, range predicates last, ORDER BY columns last
CREATE INDEX idx_payments_tenant_status_created 
ON payments(tenant_id, status, created_at);
-- Satisfies: WHERE tenant_id = ? AND status = ? ORDER BY created_at DESC
-- Uses the full index. If columns were reversed, range on created_at kills use of rest.

-- Covering index: includes all columns needed — no table lookup
CREATE INDEX idx_payments_covering 
ON payments(tenant_id, status) INCLUDE (amount, reference_id);
-- Query: SELECT amount, reference_id WHERE tenant_id = ? AND status = ?
-- Answered entirely from index, no heap access

-- Partial index: smaller index for common query patterns
CREATE INDEX idx_pending_payments 
ON payments(created_at) WHERE status = 'PENDING';
-- Only indexes PENDING rows — much smaller for a table with mostly COMPLETED rows

-- Functional index: on expression
CREATE INDEX idx_payments_date ON payments(DATE(created_at));
-- Supports: WHERE DATE(created_at) = '2024-01-15'
```

**Multi-tenant index strategy:**
All queries in a multi-tenant system are scoped by `tenant_id`. Make it the first column in every composite index:

```sql
-- Payment lookup by partner
CREATE INDEX idx_payments_tenant_partner 
ON payments(tenant_id, partner_id, status, created_at DESC);

-- Webhook event deduplication  
CREATE UNIQUE INDEX idx_webhook_dedup 
ON webhook_events(payment_reference_id, event_type);
-- This is our Layer 4 idempotency constraint
```

**EXPLAIN ANALYZE output patterns to watch:**

| Pattern | Meaning | Fix |
|---------|---------|-----|
| `Seq Scan` on large table | No usable index | Add appropriate index |
| `Index Scan` | Good — using index | - |
| `Bitmap Index Scan` | Range query across many rows | Normal for large ranges |
| `rows=10000` but actual `10` | Stale statistics | `ANALYZE table` |
| `Index Scan` but slow | Over-fetching with large OFFSET | Switch to cursor pagination |

---

### Q12. Pessimistic vs Optimistic Locking — trade-offs and when to use each.

**The answer:**

"These are two fundamentally different strategies for handling concurrent writes to the same data.

**Pessimistic Locking:**
Acquire a lock before reading, hold it through the write. Other threads block until the lock is released.

```java
// PostgreSQL / MySQL: SELECT ... FOR UPDATE
@Query(value = "SELECT * FROM partner WHERE id = :id FOR UPDATE", nativeQuery = true)
Partner findByIdForUpdate(@Param("id") UUID id);

// SKIP LOCKED — skip rows already locked (for job queue patterns)
@Query(value = "SELECT * FROM leads WHERE status = 'PENDING' LIMIT 10 FOR UPDATE SKIP LOCKED",
       nativeQuery = true)
List<Lead> findPendingLeadsForProcessing();
```

When to use: High contention (multiple threads frequently competing for the same row), critical sections where you cannot afford ANY lost updates (financial deductions), or when the read-modify-write cycle is long enough that retries would be expensive.

**Optimistic Locking:**
Read without locking. On write, verify the data hasn't changed (via `@Version` stamp). If it has, the write fails — caller retries.

```java
@Entity
public class Partner {
    @Version
    Long version;  // Hibernate auto-increments on every UPDATE
}

// Hibernate generates:
// UPDATE partner SET credit = ?, version = 5 WHERE id = ? AND version = 4
// If version = 4 is no longer true → 0 rows updated → ObjectOptimisticLockingFailureException
```

When to use: Low-to-medium contention (most of the time, writes don't conflict), read-heavy data (optimize for the common read path, pay retry cost only on rare write conflicts).

**Thundering Retry Storm Problem:**
Under high contention, optimistic locking causes a retry storm — all threads fail simultaneously, all retry simultaneously, all fail again. Add jitter:

```java
@Retryable(
    value = ObjectOptimisticLockingFailureException.class,
    maxAttempts = 5,
    backoff = @Backoff(delay = 50, multiplier = 1.5, random = true)
)
public void updateCredit(UUID partnerId, BigDecimal amount) {
    // Spring will retry up to 5 times with increasing jittered delay
}
```

**Deadlock prevention with pessimistic locking:**
Always acquire multiple locks in the same order across all code paths:

```java
// ALWAYS lock partner before lead — consistent order everywhere
// Thread A: lock partner(1) → lock lead(100)
// Thread B: lock partner(2) → lock lead(200)
// No deadlock possible if order is always entity type: Partner → Lead → Payment
```

---

### Q13. Explain database sharding strategies. What are the cross-shard challenges?

**The answer:**

"Sharding splits data across multiple database nodes to scale writes and storage beyond a single node.

**Strategy 1 — Range Sharding:**
Partition by value range. Shard 1: IDs 1–1M, Shard 2: IDs 1M–2M, etc.
- Pro: Range queries efficient (all data in a range on same shard), easy to add new shards at the end.
- Con: Hot spots — if most recent data is heavily accessed, the highest-range shard gets all the load.

**Strategy 2 — Hash Sharding:**
`shard = hash(key) % num_shards`
- Pro: Even distribution, no hot spots.
- Con: Range queries require scatter-gather (hit all shards, merge results). Adding a shard requires rehashing ALL data — very expensive.

**Strategy 3 — Consistent Hashing (Cassandra, DynamoDB):**
Keys and nodes are placed on a hash ring. A key maps to the first node clockwise from its hash position.
- Pro: Adding/removing a node only moves ~K/N keys (K = total keys, N = nodes), not all keys.
- Con: More complex to implement; requires virtual nodes (vnodes) for even distribution.

```
Hash ring: 0 ────── Node A (pos 25) ────── Node B (pos 50) ────── Node C (pos 75) ────── 0
Key "payment-123" hashes to pos 35 → assigned to Node B (next clockwise)
```

```sql
-- Cassandra CQL example — partition key determines shard
CREATE TABLE payments (
    tenant_id UUID,
    payment_id UUID,
    amount DECIMAL,
    status TEXT,
    PRIMARY KEY ((tenant_id), payment_id)  -- (partition key), clustering columns
);
-- All payments for a tenant_id go to the same shard
-- Queries by tenant_id are single-shard (efficient)
-- Queries across all tenants = scatter-gather (expensive)
```

**Cross-shard challenges:**

1. **Cross-shard transactions:** No ACID across shards without distributed coordination (2PC, Saga). Most systems accept eventual consistency.

2. **Cross-shard queries:** `SELECT * FROM payments WHERE amount > 1000` must be sent to all shards and results merged. Use a separate read model (Elasticsearch, aggregated DB) for reporting queries.

3. **Global unique IDs:** Auto-increment doesn't work across shards (duplicates). Use: Snowflake IDs (timestamp + worker ID + sequence), UUIDs (globally unique but random → bad for B+Tree index locality), or sequence service.

4. **Schema changes:** Must be applied to all shards — coordinate carefully.

**Your Project Angle:** "In our multi-tenant payment system, we used Cassandra for webhook event storage — partitioned by `tenant_id`. This gives us single-node lookup for all queries scoped to a tenant. For reporting across tenants, we stream events to Elasticsearch. The key insight is: design your partition key around your primary access pattern."

---

## QUICK-REFERENCE — Senior Filter Questions

**Q: What is `ThreadLocal` and what's the memory leak risk?**
`ThreadLocal` gives each thread its own independent copy of a variable. Used in Spring for `SecurityContextHolder`, Hibernate's `EntityManager`, and our `TenantContextHolder`. The leak: thread pool threads are never destroyed — if you set a `ThreadLocal` and never remove it, the value persists forever. Always call `remove()` in a `finally` block after the request completes.

**Q: What is happens-before in Java's memory model?**
The happens-before relationship guarantees that memory writes in one operation are visible to subsequent operations. Key rules: (1) program order within a thread, (2) `synchronized` block unlock happens-before the next lock, (3) `volatile` write happens-before the next read of the same variable, (4) `Thread.start()` happens-before the started thread's code, (5) `Thread.join()` completes happens-before the code after `join()`.

**Q: Why is `ForkJoinPool.commonPool()` dangerous for blocking I/O?**
The common pool is sized to `CPU cores - 1`. If your CompletableFuture tasks do I/O (DB calls, HTTP requests) and block, you consume all pool threads, starving other async operations and creating deadlocks. Always pass a dedicated `Executor` with I/O-appropriate sizing to `CompletableFuture.supplyAsync(task, executor)`.

**Q: `ConcurrentHashMap` vs `Collections.synchronizedMap()`?**
`synchronizedMap` wraps every method with a single mutex — one lock for the entire map. `ConcurrentHashMap` uses segment-level locking (Java 7) or CAS + synchronized per bucket (Java 8+) — concurrent readers never block, concurrent writers on different buckets don't block each other. Use `ConcurrentHashMap` for any concurrent access. Never use `synchronizedMap` in performance-sensitive code.

**Q: What is the difference between `Runnable` and `Callable`?**
`Runnable.run()` returns `void` and cannot throw checked exceptions. `Callable<V>.call()` returns a value `V` and can throw checked exceptions. `ExecutorService.submit(callable)` returns a `Future<V>`. Use `Callable` when you need the result or need to propagate exceptions from async code.

---

---

## PART 6 — DATABASE PRODUCTION TOPICS

---

### Q14. How do you configure and tune HikariCP? What happens when the pool is exhausted?

**The answer:**

"HikariCP is Spring Boot's default connection pool since 2.0. It manages a pool of JDBC connections so the application doesn't pay the TCP handshake + authentication cost on every query.

**Key configuration properties:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # max connections held open
      minimum-idle: 5              # kept alive even when idle
      connection-timeout: 30000    # ms to wait for a connection before exception
      idle-timeout: 600000         # ms before idle connection is closed (10 min)
      max-lifetime: 1800000        # ms max connection lifetime (30 min) — rotate before DB kills it
      pool-name: PaymentHikariPool
      leak-detection-threshold: 60000  # ms — log stack trace if connection held this long
```

**Pool sizing — the formula:**
HikariCP's own recommendation (based on queuing theory):
```
pool_size = (core_count × 2) + effective_spindle_count
```
For an 8-core server with SSDs (1 spindle):
`pool_size = (8 × 2) + 1 = 17 ≈ 20`

The critical constraint: `maximum-pool-size × app_instances ≤ DB max_connections`
- PostgreSQL default max_connections = 100
- If you have 3 app instances × 20 pool size = 60 connections → fine
- 5 instances × 20 = 100 → saturates the DB entirely, nothing left for admin queries

**What happens when the pool is exhausted:**
All 20 connections are in use. A new request calls `DataSource.getConnection()`. It waits for `connection-timeout` ms (default 30s). If no connection is freed in that window: `SQLTransientConnectionException: Unable to acquire JDBC Connection`.

**Diagnosing pool exhaustion:**
```java
// Micrometer metrics (auto-exposed if HikariCP + Micrometer on classpath)
hikaricp_connections_active   // connections currently in use
hikaricp_connections_pending  // threads waiting for a connection
hikaricp_connections_timeout_total  // total timeout events

// Alert rule: if hikaricp_connections_pending > 0 for 60s → pool too small
```

**Common causes of exhaustion:**
1. Slow queries holding connections too long (fix: query optimization, read replicas)
2. N+1 problem creating too many sequential DB calls per request
3. `@Transactional` method doing I/O (HTTP call, Kafka) while holding a DB connection
4. `leak-detection-threshold` firing means a connection is never returned — usually a missing `close()` in non-JPA code

**Your Project Angle:** 'In the payment gateway, we hit pool exhaustion during a Razorpay API outage. Each payment request was: open DB transaction, call Razorpay (blocked for 30s timeout), commit. With 20 concurrent requests, all 20 DB connections were held hostage waiting for a network call. Fix: moved the Razorpay call outside the `@Transactional` boundary — get the DB connection only when you actually need to write, not for the full request duration.'"

---

### Q15. How do you manage database schema changes in production? Explain Flyway.

**The answer:**

"Database migrations are version-controlled SQL scripts that evolve the schema incrementally. Flyway is the de facto standard with Spring Boot.

**How Flyway works:**
1. On startup, Flyway scans `classpath:db/migration/` for SQL files matching `V{version}__{description}.sql`
2. It checks the `flyway_schema_history` table in the DB to see which migrations have already run
3. It applies any pending migrations in version order, inside a transaction (if the DB supports transactional DDL — PostgreSQL does, MySQL partially)
4. If any migration fails, it marks it as failed and refuses to start the app until resolved

```
db/migration/
├── V1__create_payments_table.sql
├── V2__add_partner_credit_column.sql
├── V3__add_webhook_events_table.sql
├── V4__add_tenant_id_index.sql
└── V5__add_payment_status_enum.sql
```

```sql
-- V3__add_webhook_events_table.sql
CREATE TABLE webhook_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_ref     VARCHAR(100) NOT NULL,
    event_type      VARCHAR(50)  NOT NULL,
    payload         JSONB,
    processed_at    TIMESTAMP,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_webhook_dedup ON webhook_events(payment_ref, event_type);
```

**Zero-downtime migration patterns — critical for production:**

The hard part is that during deployment, both old and new app versions run simultaneously (rolling deploy). The DB schema must work for both.

**Pattern 1 — Adding a nullable column (safe):**
```sql
-- V4__add_notes_column.sql
ALTER TABLE payments ADD COLUMN notes TEXT;  -- nullable by default → safe, no backfill needed
```

**Pattern 2 — Adding a NOT NULL column (multi-step):**
```sql
-- Step 1: Add nullable (deploy V4 — old app works, new app writes the column)
ALTER TABLE payments ADD COLUMN fraud_score DECIMAL(5,2);

-- Step 2: Backfill existing rows (deploy V5)
UPDATE payments SET fraud_score = 0.0 WHERE fraud_score IS NULL;

-- Step 3: Add NOT NULL constraint (deploy V6 — all rows now have a value)
ALTER TABLE payments ALTER COLUMN fraud_score SET NOT NULL;
```

**Pattern 3 — Renaming a column (expand-contract):**
```sql
-- Step 1: Add new column, dual-write from app
ALTER TABLE payments ADD COLUMN partner_id UUID;
-- App writes to BOTH old_partner_id AND partner_id

-- Step 2: Backfill old rows
UPDATE payments SET partner_id = old_partner_id WHERE partner_id IS NULL;

-- Step 3: Switch reads to new column (app no longer reads old_partner_id)
-- Step 4: Drop old column (safe — nobody reads it anymore)
ALTER TABLE payments DROP COLUMN old_partner_id;
```

**Pattern 4 — Adding an index without blocking writes (PostgreSQL):**
```sql
-- CONCURRENT index build — doesn't hold an exclusive lock on the table
-- Takes longer but never blocks reads or writes
CREATE INDEX CONCURRENTLY idx_payments_tenant_status
ON payments(tenant_id, status);
-- Note: Flyway wraps migrations in transactions by default.
-- CONCURRENTLY cannot run inside a transaction — disable with:
-- flyway.mixed=true and use -- flyway:executeInTransaction=false annotation
```

**Spring Boot + Flyway auto-configuration:**
```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true  # for existing DBs with no flyway history yet
    validate-on-migrate: true  # fail if a previously-applied migration's checksum changed
```

**Senior Signal:** 'The most dangerous mistake with migrations is modifying a script that has already been applied to production. Flyway stores a checksum of each applied migration — if you edit a script that already ran, Flyway throws `FlywayValidateException` and refuses to start. Always create a new migration file for every change, no matter how small.'"

---

## PART 7 — ADVANCED KAFKA PATTERNS

---

### Q16. Explain Kafka consumer groups, partition rebalancing, and what causes processing pauses.

**The answer:**

"A consumer group is a set of consumers that collectively read from a topic. Kafka assigns each partition to exactly one consumer within a group at any time. This is how Kafka achieves parallelism: a topic with 6 partitions can be consumed by up to 6 consumers in parallel (within one group).

```
Topic: payment-events (6 partitions)

Consumer Group A (payment-processor):
  Consumer 1 → Partitions 0, 1
  Consumer 2 → Partitions 2, 3
  Consumer 3 → Partitions 4, 5

Consumer Group B (audit-logger):
  Consumer 1 → Partitions 0, 1, 2  (independent — separate offset tracking)
  Consumer 2 → Partitions 3, 4, 5
```

**Rebalancing — what it is and why it causes pauses:**
When a consumer joins or leaves a group, Kafka must redistribute partitions. During rebalancing:
1. ALL consumers in the group stop processing (stop-the-world by default)
2. Group coordinator reassigns partitions
3. Consumers resume from their last committed offset

Causes of rebalance:
- New consumer instance started (scale-out)
- Consumer crashed or was stopped
- Consumer didn't poll within `max.poll.interval.ms` (default 5 min) — treated as dead

**The slow consumer rebalance trap:**
```java
// DANGEROUS — processing 500 records takes too long
@KafkaListener(topics = "payment-events")
public void consume(List<ConsumerRecord<String, Payment>> records) {
    for (ConsumerRecord<String, Payment> record : records) {
        callExternalService(record.value());  // each call takes 2 seconds
    }
    // 500 records × 2s = 1000s >> max.poll.interval.ms (300s default)
    // Kafka thinks consumer is dead → rebalance → messages reprocessed
}
```

**Fix — process async, commit manually:**
```java
@KafkaListener(topics = "payment-events",
               containerFactory = "batchFactory")
public void consume(List<ConsumerRecord<String, Payment>> records,
                    Acknowledgment ack) {
    List<CompletableFuture<Void>> futures = records.stream()
        .map(r -> CompletableFuture.runAsync(() -> process(r.value()), executor))
        .toList();

    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    ack.acknowledge();  // commit only after all processed
}
```

**Cooperative Incremental Rebalancing (Kafka 2.4+):**
The default `EagerRebalanceProtocol` revokes ALL partitions from ALL consumers, then reassigns. `CooperativeStickyAssignor` only reassigns partitions that are actually moving — others continue processing. Configure with:
```yaml
spring:
  kafka:
    consumer:
      properties:
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

**Hot partition — what it is and how to fix it:**
A partition receives disproportionate traffic because the partition key is skewed:
```java
// BAD partition key — one tenant sending 80% of messages
kafkaTemplate.send("payments", payment.getTenantId(), payment);
// If tenant "enterprise-A" sends 10K msg/sec and others send 100 msg/sec,
// enterprise-A's partition consumer is overwhelmed, others are idle.

// FIX 1 — add random suffix to spread within a tenant
String key = payment.getTenantId() + "-" + (payment.getPaymentId().hashCode() % 10);

// FIX 2 — use null key (round-robin across all partitions)
kafkaTemplate.send("payments", null, payment);
// Trade-off: lose ordering guarantees within a tenant
```

---

### Q17. What is a Dead Letter Queue? How do you implement it for Kafka?

**The answer:**

"A Dead Letter Queue (DLQ) is a destination for messages that cannot be processed successfully after a configured number of retries. Without a DLQ, a poison-pill message (malformed, causes an exception on every attempt) blocks the consumer indefinitely — it retries forever and the partition makes no progress.

**Kafka has no native DLQ — you implement it:**

```java
@KafkaListener(topics = "payment-events", groupId = "payment-processor")
public void consume(ConsumerRecord<String, String> record) {
    try {
        paymentService.process(parsePayment(record.value()));
    } catch (NonRetryableException e) {
        // Immediately send to DLQ — no retry
        sendToDlq(record, e);
    } catch (RetryableException e) {
        // Will be retried by Spring Kafka's retry mechanism
        throw e;
    }
}

private void sendToDlq(ConsumerRecord<String, String> original, Exception cause) {
    ProducerRecord<String, String> dlqRecord = new ProducerRecord<>(
        "payment-events.DLQ",   // convention: original-topic + .DLQ
        original.key(),
        original.value()
    );
    // Add failure metadata as headers
    dlqRecord.headers()
        .add("x-original-topic", original.topic().getBytes())
        .add("x-original-partition", String.valueOf(original.partition()).getBytes())
        .add("x-original-offset", String.valueOf(original.offset()).getBytes())
        .add("x-failure-reason", cause.getMessage().getBytes())
        .add("x-failure-timestamp", Instant.now().toString().getBytes());

    kafkaTemplate.send(dlqRecord);
    log.error("Message sent to DLQ — key: {}, reason: {}", original.key(), cause.getMessage());
}
```

**Spring Kafka built-in DLQ support:**
```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, String> template) {
    // After 3 retries with backoff, send to DLQ automatically
    DeadLetterPublishingRecoverer recoverer =
        new DeadLetterPublishingRecoverer(template,
            (record, ex) -> new TopicPartition(record.topic() + ".DLQ", record.partition())
        );

    ExponentialBackOffWithMaxRetries backoff = new ExponentialBackOffWithMaxRetries(3);
    backoff.setInitialInterval(1000L);
    backoff.setMultiplier(2.0);

    return new DefaultErrorHandler(recoverer, backoff);
}
```

**DLQ monitoring — a growing DLQ is an incident:**
```java
// Alert: if DLQ consumer lag > 0 for more than 5 minutes → page on-call
// Prometheus rule:
// kafka_consumer_group_lag{topic=~".*\\.DLQ"} > 0 for 5m
```

**Reprocessing from DLQ:**
Once the root cause is fixed, replay DLQ messages. Because the original consumer is idempotent (your 5-layer system), replaying is safe:
```bash
# Kafka CLI: reset DLQ consumer group offset to earliest, then restart consumer
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --group dlq-reprocessor \
  --topic payment-events.DLQ \
  --reset-offsets --to-earliest --execute
```

---

## PART 8 — ADVANCED REDIS PATTERNS

---

### Q18. Redis Sentinel vs Redis Cluster — what's the difference? When do you use each?

**The answer:**

"**Redis Sentinel** is a high-availability solution for a single Redis primary. It monitors the primary and replicas, automatically promotes a replica to primary on failure (failover), and provides service discovery (clients ask Sentinel for the current primary address).

```
            ┌── Sentinel 1 ─┐
Client ──→  ├── Sentinel 2 ─┤ ──→ Primary (writes)
            └── Sentinel 3 ─┘         │
                                  Replica 1 (reads)
                                  Replica 2 (reads)
```

- Data is replicated but NOT sharded — all data fits on one primary node
- Failover time: typically 10–30 seconds (Sentinel quorum election)
- Use when: your data fits in one node's memory, you need HA without sharding complexity

**Redis Cluster** shards data across multiple primaries. Each node holds a subset of the 16,384 hash slots. Every node has a replica for HA.

```
Primary A (slots 0-5460)       Primary B (slots 5461-10922)     Primary C (slots 10923-16383)
    └── Replica A                    └── Replica B                    └── Replica C
```

- Horizontal scaling: add primaries to handle more data / more write throughput
- Client must be cluster-aware (routes `GET key` to the correct shard)
- Some operations are restricted: `MGET` across different slots requires multiple calls
- `KEYS *` is per-node only

**When to use each:**

| Need | Use |
|------|-----|
| HA only, data fits in one node | Sentinel |
| Data exceeds single node memory | Cluster |
| Need multi-key atomic ops across all data | Sentinel |
| Need horizontal write scaling | Cluster |
| Operational simplicity | Sentinel |

**Your project context:** 'In our payment platform, we use Sentinel. Our Redis stores credit limits and rate-limit counters — a few GB total. We need HA (automatic failover if primary dies) but not sharding. Redis Cluster would add cross-slot coordination overhead for our Lua scripts (credit deduction accesses one key — fine on Cluster, but partition awareness would complicate client setup).'"

---

### Q19. Redis Pub/Sub vs Redis Streams — when do you use each?

**The answer:**

"**Redis Pub/Sub** is fire-and-forget broadcast messaging. Publishers push messages to a channel; all current subscribers receive them. No persistence, no acknowledgement, no replay.

```java
// Publisher
redisTemplate.convertAndSend("partner-events", "LEAD_ASSIGNED:" + partnerId);

// Subscriber (Spring listener)
@Component
public class PartnerEventSubscriber implements MessageListener {
    @Override
    public void onMessage(Message message, byte[] pattern) {
        String event = new String(message.getBody());
        // process event
    }
}
```

Critical limitation: **if a subscriber is offline, it misses the message permanently.** There is no persistence.

**Redis Streams** is a persistent, ordered log — like Kafka, but inside Redis. Messages are appended to the stream, read by consumer groups, and acknowledged explicitly. Unacknowledged messages can be re-delivered.

```java
// Producer — append to stream
Map<String, String> fields = Map.of(
    "partnerId", partnerId,
    "event", "LEAD_ASSIGNED",
    "leadId", leadId
);
redisTemplate.opsForStream().add("partner-stream", fields);

// Consumer group — persistent, acknowledgement-based
@Scheduled(fixedDelay = 100)
public void processStream() {
    List<MapRecord<String, String, String>> records = redisTemplate.opsForStream()
        .read(Consumer.from("assignment-group", "consumer-1"),
              StreamReadOptions.empty().count(10),
              StreamOffset.create("partner-stream", ReadOffset.lastConsumed()));

    for (MapRecord<String, String, String> record : records) {
        processEvent(record.getValue());
        redisTemplate.opsForStream().acknowledge("partner-stream",
            "assignment-group", record.getId());  // explicit ack
    }
}
```

**Decision table:**

| | Pub/Sub | Streams |
|--|---------|---------|
| Persistence | No | Yes (configurable retention) |
| Missed messages | Lost | Redelivered (pending entries) |
| Consumer groups | No | Yes |
| Replay | No | Yes |
| Use case | Real-time notification (loss ok) | Reliable event processing |

**Your Project Angle:** 'In the WebSocket lead assignment system, we use Redis Pub/Sub to relay real-time push notifications across app instances — if a client is offline when the event fires, they reconnect and poll their pending assignment queue (DB-backed). The Pub/Sub is intentionally fire-and-forget for the live push. Reliability comes from the DB-backed queue, not from Pub/Sub itself.'"

---

### Q20. Redis persistence — RDB vs AOF. Why does this matter for your credit limit design?

**The answer:**

"Redis offers two persistence mechanisms, and the choice directly impacts the durability guarantee you can claim about Redis-stored data.

**RDB (Redis Database Backup) — periodic snapshots:**
Redis forks the process and writes the entire dataset to disk at configured intervals (`save 900 1` = save if 1 key changed in 900s). Fast restart, compact file. Data loss window: up to the snapshot interval (minutes).

**AOF (Append-Only File) — write-ahead log:**
Every write command is appended to a file. On restart, Redis replays the file to reconstruct state. Three fsync policies:
- `appendfsync always` — fsync after every write. Durability: 0 data loss. Throughput: ~10K writes/sec
- `appendfsync everysec` — fsync every 1 second (default). Data loss: max 1 second of writes
- `appendfsync no` — let the OS decide. Fastest, least durable

**Hybrid (recommended for production):**
```
appendonly yes
appendfsync everysec
aof-rewrite-incremental-fsync yes
save 900 1    # RDB as a safety net
save 300 10
```

**Why this matters for your credit limit design:**

In Ch2, we store credit limits in Redis and use a Lua script for atomic deduction. The question an interviewer WILL ask: 'What happens if Redis crashes? Do you lose the credit limit state?'

Answer: 'With AOF + `everysec`, we lose at most 1 second of writes. This means up to 1 second of credit deductions could be lost. We handle this two ways: (1) periodic reconciliation — a scheduled job every 5 minutes compares Redis credit state with the DB's authoritative record and resynchronizes. (2) On Redis startup from AOF, the credit limits are replayed. The 1-second gap is acceptable because the DB is always the source of truth — Redis is a performance cache for the hot path, not the only record.'"

---

> **What's Next — Chapter 3:** Low-Level Design — SOLID principles, all design patterns with production code, and full architectural narratives for your three resume projects.

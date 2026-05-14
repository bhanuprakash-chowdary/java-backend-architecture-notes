# Interview Survival Guide
## Chapter 4: HLD Frameworks, Distributed Systems & Behavioral Mastery
### (Weeks 13, 16, 17, 18 & 24)

---

## PART 1 — THE HLD INTERVIEW FRAMEWORK

---

### The Universal 5-Step HLD Framework

Use this exact structure for every HLD question. Never skip a step. The interviewer is evaluating your process as much as your solution.

---

**Step 1: Requirements Clarification (5 minutes)**

*Verbal script to use:*
"Before I start designing, let me clarify requirements to make sure we're solving the right problem."

**Functional requirements (what the system does):**
- Who are the users? What are the primary use cases?
- What are the exact operations: read, write, update, delete?
- Any real-time requirements? (push vs pull)
- Multi-tenancy? Geographic distribution?

**Non-functional requirements (how well it does it):**
- Scale: QPS for reads and writes? (critical for architecture decisions)
- Latency SLA: p99 < Xms? Real-time?
- Availability: 99.9% (8.7 hrs/yr downtime) vs 99.99% (52 min/yr)?
- Consistency: strong vs eventual acceptable?
- Data retention: how long? GDPR considerations?

**Senior Signal:** "I distinguish between read-heavy vs write-heavy workloads upfront — this drives almost every subsequent architectural decision. A read-heavy system (10:1 read/write ratio) optimizes for caching and read replicas. A write-heavy system needs different sharding and write-path design."

---

**Step 2: Capacity Estimation (3 minutes)**

Back-of-envelope math. Shows you can reason about scale.

*Example — Payment Platform:*
```
Transactions: 10,000/day → ~0.1 TPS average, ~10 TPS peak (10x)

Write path:
  - 1 transaction = 5 DB writes (payment + audit + webhook + status + notification)
  - Peak write QPS = 10 × 5 = 50 writes/sec (well within single Postgres)

Read path:
  - Dashboard: 1,000 concurrent users × 5 reads/min = ~83 reads/sec
  - Status polling: 500 webhook senders × 1 poll/30s = ~17 reads/sec

Storage:
  - 1 payment record ≈ 1KB
  - 10,000/day × 365 days × 3 years = 11M records ≈ 11GB
  - With indexes and audit logs: ~50GB — easily fits on a single Postgres node

Bandwidth:
  - Webhook payloads: 10 events/sec × 2KB = 20KB/sec
  - Dashboard API: 83 reads/sec × 5KB = 415KB/sec ≈ 0.5 MB/s — negligible
```

**What this tells us:** Single-region Postgres can handle writes and reads. No sharding needed at this scale. Focus on reliability and correctness over horizontal scaling.

---

**Step 3: High-Level Architecture (10 minutes)**

Draw the boxes and arrows. Name every component.

```
Internet
    │
    ▼
Load Balancer (ALB)
    │
    ├──────────────────────────────────────┐
    ▼                                      ▼
API Gateway (auth, rate-limit)         Webhook Receiver (stateless)
    │                                      │
    ▼                                      ▼
Payment Service Cluster               Kafka (webhook-events topic)
(Spring Boot, 3 instances)                 │
    │                                      ▼
    ├── PostgreSQL Primary         Webhook Consumer Cluster
    │   (writes)                   (at-least-once processing)
    │                                      │
    ├── Redis Cache                        ▼
    │   (sessions, rate limits,    Payment State Machine
    │    credit limits)            (stateless, DB-backed)
    │
    └── PostgreSQL Replicas (reads)
```

*What to say:* "I'll start with the core components and identify the critical paths — the sequences of operations that must complete for the system to function correctly. Then we'll deep-dive into the most interesting ones."

---

**Step 4: Deep Dive into Critical Components (15 minutes)**

Interviewers will ask you to go deeper on one or two components. Pre-identify the most interesting ones and be ready. Common deep-dives:

- **The write path:** How does a payment flow from API to DB?
- **The failure scenario:** What happens when a component fails?
- **The scale bottleneck:** Where does this design break at 100x load?
- **The data model:** What does the DB schema look like?

---

**Step 5: Failure Analysis & Trade-offs (5 minutes)**

Close every design with: "Let me walk through what happens when components fail."

For each critical component, state:
- What fails when it goes down
- How you detect it (health check, circuit breaker)
- What the user experiences (error vs degraded vs invisible)
- How you recover (timeout, fallback, retry, dead letter queue)

---

## PART 2 — DISTRIBUTED SYSTEMS FUNDAMENTALS

---

### Q1. Explain the CAP Theorem precisely. What does it mean in practice?

**The answer:**

"CAP Theorem states that a distributed system can guarantee at most two of three properties simultaneously during a network partition:

- **Consistency (C):** Every read receives the most recent write or an error. All nodes see the same data at the same time.
- **Availability (A):** Every request receives a response (not an error), though it may not contain the most recent data.
- **Partition Tolerance (P):** The system continues operating even when network partitions (message loss/delay between nodes) occur.

**The key insight: P is not optional.**
Network partitions happen in any distributed system — cables fail, network switches drop packets, data center links saturate. P is a reality of distributed systems, not a choice. The real choice is: when a partition occurs, do you choose **Consistency** (refuse requests to maintain data integrity) or **Availability** (serve potentially stale data)?

**Real system mapping:**

| System | Choice | Reasoning |
|--------|--------|-----------|
| PostgreSQL (single node) | CA | No partition possible on one machine |
| PostgreSQL with synchronous replicas | CP | On partition, replica rejects reads to stay consistent |
| MySQL with async replicas | AP | Replica serves reads even if it's behind — stale but available |
| Cassandra | AP | Configurable, defaults to AP (eventual consistency) |
| Zookeeper / etcd | CP | Leader election requires quorum — sacrifices availability for correctness |
| DynamoDB | AP by default | Configurable to CP with `ConsistentRead=true` |

**For our payment system:**
Writes must be CP — we cannot show a payment as 'pending' on one node and 'completed' on another. A customer double-submitting a payment would get incorrect state. Reads can be AP — dashboard data can be 1-2 seconds stale; the user doesn't notice.

**PACELC extension:**
CAP only covers partition scenarios. PACELC (PAC-ELC) extends this: Even when the system is running normally (no partition), there's still a trade-off between **Latency** and **Consistency**. Synchronous replication = consistent but slower (synchronous wait for all replicas). Asynchronous replication = faster writes but potentially stale reads. Most production systems optimize for low latency (async) and accept eventual consistency during normal operation."

---

### Q2. ACID vs BASE — what's the implication for microservices?

**The answer:**

"ACID describes transactional guarantees in traditional relational databases:
- **Atomicity:** All operations in a transaction succeed or all roll back.
- **Consistency:** Transaction brings DB from one valid state to another.
- **Isolation:** Concurrent transactions don't interfere with each other.
- **Durability:** Committed transactions survive system failures.

BASE describes the relaxed guarantees typical in distributed / NoSQL systems:
- **Basically Available:** The system is available most of the time, with possible partial failures.
- **Soft state:** Data can change over time even without explicit updates (due to replication propagation).
- **Eventually Consistent:** All replicas converge to the same value — just not immediately.

**The critical microservices implication:**

In a monolith, `@Transactional` gives you ACID across multiple DB operations:
```java
// In a monolith — one DB, one transaction
@Transactional
public void transferFunds(Account from, Account to, BigDecimal amount) {
    from.debit(amount);
    to.credit(amount);
    // Both succeed or both roll back — ACID
}
```

In microservices, each service owns its database. There is no distributed `@Transactional`:
```java
// In microservices — TWO databases, no ACID guarantee
public void transferFunds(String fromAccountId, String toAccountId, BigDecimal amount) {
    accountService.debit(fromAccountId, amount);  // Service A, DB A
    walletService.credit(toAccountId, amount);    // Service B, DB B
    // If walletService crashes: debit happened, credit did not — money lost!
}
```

**Solutions:** Saga Pattern (choreography or orchestration) for eventual consistency across services, Outbox Pattern to guarantee at-least-once delivery without dual-write risk. These replace ACID with carefully designed compensation and idempotency."

---

### Q3. Eventual Consistency Patterns — what are the guarantees between 'eventual' and 'strong'?

**The answer:**

"Eventual consistency is not a single guarantee — there's a spectrum.

**Read-Your-Writes Consistency:** After you write data, your subsequent reads will see that write. Other users may not. Useful for: user profile updates (you immediately see your own change).

**Monotonic Reads:** If a user has read data at version N, all subsequent reads return data at version >= N. They never see older data. Prevents: seeing a feed where a post disappears and reappears.

**Causal Consistency:** If operation B causally depends on operation A (e.g., replying to a post), B is never visible without A. Prevents: seeing a reply before the original post.

**Strong Eventual Consistency (CRDTs):** All replicas that have received the same set of updates converge to the same state, without needing coordination. Data structures like counters, sets, and last-write-wins registers can be CRDT-designed. Used in: collaborative editing (Google Docs), shopping carts, distributed counters.

**In our payment system:**
- Write path: Strong consistency (PostgreSQL, synchronous commit)
- Read path for dashboards: Monotonic reads via Redis cache with version tracking
- Event propagation to partners: Read-Your-Writes (partner sees their own lead assignment immediately, even if the replica hasn't propagated)"

---

## PART 3 — DISTRIBUTED DESIGN PATTERNS

---

### Q4. Two-Phase Commit — why don't microservices use it?

**The answer:**

"Two-Phase Commit (2PC) is a distributed protocol for achieving atomic commits across multiple resources.

**Protocol:**
1. **Prepare phase:** Coordinator asks all participants: 'Can you commit?' Each participant executes the transaction locally, writes to a redo log, and responds 'Yes' or 'No'.
2. **Commit phase:** If ALL said 'Yes' → coordinator sends 'Commit' to all. If ANY said 'No' → coordinator sends 'Abort' to all.

**Why 2PC fails for microservices:**

1. **Blocking:** If the coordinator crashes after Phase 1 but before Phase 2, all participants are stuck holding their locks, waiting forever. No participant can decide alone — only the coordinator knows the global outcome.

2. **Tight coupling:** Every service must implement the 2PC protocol. They're now coupled at the protocol level — adding a service requires it to support 2PC.

3. **Performance:** All participants must hold locks and resources for the duration of both phases. With network latency between services, this significantly increases transaction duration.

4. **Not cloud-native:** Most cloud databases and message queues don't support the XA/2PC protocol. AWS RDS, DynamoDB, Kafka — none support cross-resource 2PC.

**The alternative: Saga Pattern** — break the distributed transaction into a sequence of local transactions, each with a compensating transaction (undo) for rollback."

---

### Q5. Saga Pattern — Choreography vs Orchestration with your payment example.

**The answer:**

"The Saga Pattern achieves distributed consistency without a global transaction, by breaking operations into a sequence of local transactions with compensating actions for rollback.

**Saga Choreography — event-driven, no central coordinator:**

Each service publishes an event, the next service listens and reacts:

```java
// Service 1: OrderService
@KafkaListener(topics = "order-placement-requested")
public void handleOrderPlaced(OrderRequest request) {
    Order order = orderRepository.save(new Order(request));
    eventPublisher.publish("order-created", new OrderCreatedEvent(order.getId()));
    // No direct call to next service
}

// Service 2: InventoryService
@KafkaListener(topics = "order-created")
public void handleOrderCreated(OrderCreatedEvent event) {
    try {
        inventoryService.reserve(event.getOrderId());
        eventPublisher.publish("inventory-reserved", new InventoryReservedEvent(event.getOrderId()));
    } catch (InsufficientStockException e) {
        // Publish failure event — triggers compensation in OrderService
        eventPublisher.publish("inventory-reservation-failed",
            new InventoryReservationFailedEvent(event.getOrderId()));
    }
}

// Service 1: OrderService — handles compensation
@KafkaListener(topics = "inventory-reservation-failed")
public void handleInventoryFailed(InventoryReservationFailedEvent event) {
    Order order = orderRepository.findById(event.getOrderId()).get();
    order.setStatus(OrderStatus.FAILED);
    orderRepository.save(order);
    eventPublisher.publish("order-cancelled", new OrderCancelledEvent(event.getOrderId()));
}
```

**Pros:** Loose coupling, no single point of failure, services are independent.
**Cons:** Hard to trace the overall saga flow, business logic scattered across services, eventual consistency is hard to reason about.

**Saga Orchestration — explicit coordinator:**

A dedicated orchestrator drives the saga:

```java
@Entity
public class SagaExecution {
    @Id UUID sagaId;
    String sagaType;      // "PAYMENT_PROCESSING"
    String currentStep;   // "INVENTORY_RESERVE"
    SagaStatus status;    // RUNNING, COMPENSATING, COMPLETED, FAILED
    String payload;       // JSON state
    
    @Version Long version;  // for optimistic locking
}

@Service
public class PaymentSagaOrchestrator {

    @Transactional
    public void execute(PaymentRequest request) {
        SagaExecution saga = sagaRepository.save(new SagaExecution("PAYMENT", request));

        try {
            // Step 1: Validate credit limit
            saga.setCurrentStep("CREDIT_CHECK");
            creditService.validateAndReserve(request.getPartnerId(), request.getAmount());

            // Step 2: Create payment record
            saga.setCurrentStep("PAYMENT_CREATE");
            Payment payment = paymentService.create(request);

            // Step 3: Initiate disbursement
            saga.setCurrentStep("DISBURSE");
            disbursementService.initiate(payment);

            saga.setStatus(SagaStatus.COMPLETED);
        } catch (Exception e) {
            saga.setStatus(SagaStatus.COMPENSATING);
            compensate(saga, request);
            throw e;
        } finally {
            sagaRepository.save(saga);
        }
    }

    private void compensate(SagaExecution saga, PaymentRequest request) {
        switch (saga.getCurrentStep()) {
            case "DISBURSE" -> paymentService.cancel(request);  // fall through
            case "PAYMENT_CREATE" -> creditService.release(request.getPartnerId(), request.getAmount());
            case "CREDIT_CHECK" -> {}  // nothing to undo
        }
    }
}
```

**Pros:** Clear audit trail, easy to reason about overall flow, centralized error handling.
**Cons:** Orchestrator can become a God class, single point of failure, tighter coupling.

**When to choose:**
- Choreography: simple sagas with few steps, teams want independence, eventual consistency is acceptable
- Orchestration: complex business flows with many steps, need audit trail, SLA-critical operations

---

### Q6. Outbox Pattern — solving the dual-write problem.

**The answer:**

"The dual-write problem: in a microservices system, you need to update your DB AND publish an event to Kafka atomically. If you do them separately, you can fail between the two:

```java
// BROKEN — not atomic
@Transactional
public void processPayment(Payment payment) {
    paymentRepository.save(payment);  // DB commit
    // Crash here → DB has the payment, Kafka has nothing
    kafkaTemplate.send("payments", payment);  // Kafka publish
}
```

The Outbox Pattern solves this with a single DB transaction:

```java
// FIXED — single transaction
@Transactional
public void processPayment(Payment payment) {
    paymentRepository.save(payment);
    
    // Write event to outbox table — same transaction as the payment
    OutboxEvent outboxEvent = OutboxEvent.builder()
        .aggregateType("PAYMENT")
        .aggregateId(payment.getId().toString())
        .eventType("PAYMENT_PROCESSED")
        .payload(objectMapper.writeValueAsString(payment))
        .build();
    outboxRepository.save(outboxEvent);
    // If either save fails, BOTH roll back — atomicity guaranteed
}

// Outbox entity
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id @GeneratedValue UUID id;
    String aggregateType;
    String aggregateId;
    String eventType;
    String payload;
    boolean published = false;
    Instant createdAt;
}
```

**The relay (background publisher):**
```java
@Scheduled(fixedDelay = 1000)  // poll every second
@Transactional
public void relayOutboxEvents() {
    List<OutboxEvent> unpublished = outboxRepository.findByPublishedFalse();
    
    for (OutboxEvent event : unpublished) {
        try {
            kafkaTemplate.send("payments", event.getAggregateId(), event.getPayload())
                         .get(5, TimeUnit.SECONDS);  // synchronous send for reliability
            event.setPublished(true);
            outboxRepository.save(event);
        } catch (Exception e) {
            log.error("Failed to relay outbox event {}: {}", event.getId(), e.getMessage());
            // Will retry on next poll
        }
    }
}
```

**Improvement — Transactional Outbox with CDC (Change Data Capture):**
Instead of polling, use Debezium to stream DB changes directly from PostgreSQL WAL (Write-Ahead Log) to Kafka. Zero latency, no polling, no DB load. Debezium monitors the outbox table and publishes every INSERT to Kafka automatically."

---

### Q7. CQRS — Command Query Responsibility Segregation.

**The answer:**

"CQRS separates the write model (commands that change state) from the read model (queries that return data).

**Why this matters in practice:**
Write models are optimized for correctness — normalized, ACID, with business invariants. Read models are optimized for query performance — denormalized, possibly cached, aggregated. These goals conflict. CQRS resolves the conflict by having two separate models.

```java
// COMMAND side — normalized, ACID, business logic
@Entity
public class Payment {
    @Id UUID id;
    UUID tenantId;
    UUID partnerId;
    BigDecimal amount;
    PaymentStatus status;
    @OneToMany(mappedBy = "payment") List<PaymentEvent> events;
}

// Command handler
@Service
public class PaymentCommandService {
    @Transactional
    public void processPayment(ProcessPaymentCommand cmd) {
        Payment payment = new Payment(cmd);
        paymentRepository.save(payment);
        
        // Publish event to update read model
        eventPublisher.publish(new PaymentCreatedEvent(payment));
    }
}

// QUERY side — denormalized, pre-aggregated for dashboard performance
// Could be in a separate table, or Elasticsearch, or a materialized view
@Entity
@Table(name = "payment_dashboard_view")
public class PaymentDashboardEntry {
    @Id UUID paymentId;
    String tenantName;        // denormalized — no join needed
    String partnerName;       // denormalized
    BigDecimal amount;
    String statusDisplay;
    String formattedDate;     // pre-formatted
    int transactionCount;     // pre-aggregated
}

// Event handler updates the read model when write model changes
@Component
public class PaymentDashboardProjection {
    @EventListener
    public void on(PaymentCreatedEvent event) {
        Partner partner = partnerRepository.findById(event.getPartnerId()).get();
        Tenant tenant = tenantRepository.findById(event.getTenantId()).get();
        
        PaymentDashboardEntry entry = new PaymentDashboardEntry();
        entry.setTenantName(tenant.getName());      // denormalize at write time
        entry.setPartnerName(partner.getName());     // not at read time
        entry.setFormattedDate(format(event.getCreatedAt()));
        dashboardRepository.save(entry);
    }
}

// QUERY service — trivially fast, no joins
@Service
public class PaymentQueryService {
    public Page<PaymentDashboardEntry> getDashboard(UUID tenantId, Pageable pageable) {
        return dashboardRepository.findByTenantId(tenantId, pageable);
        // Single table query, no joins, pre-aggregated — sub-millisecond
    }
}
```

**Trade-offs:**
- Pro: Read queries are extremely fast (no joins, pre-computed)
- Pro: Write and read models can scale independently
- Con: Eventual consistency — read model lags behind write model by milliseconds to seconds
- Con: Increased complexity — two models to maintain
- Use when: Complex domain with heavy read load, reporting requirements that strain the write DB, GDPR (delete from write model, read model eventually tombstones)"

---

## PART 4 — API DESIGN & SERVICE PATTERNS

---

### Q8. Design a production-quality REST API for the payment system.

**The answer:**

"There are five dimensions of good API design: resource naming, HTTP semantics, idempotency, versioning, and pagination.

**Resource naming — nouns, not verbs:**
```
POST /api/v1/payments              — create payment
GET  /api/v1/payments/{id}         — get payment
GET  /api/v1/payments?tenantId=X   — list payments (filter)
POST /api/v1/payments/{id}/cancel  — action on resource (noun/verb hybrid for actions)
GET  /api/v1/partners/{id}/credit  — sub-resource
```

**Precise HTTP status codes:**
| Status | When to use |
|--------|-------------|
| 200 OK | GET success, PUT/PATCH success with body |
| 201 Created | POST created a resource; include `Location: /payments/123` header |
| 202 Accepted | Async operation started; include status polling endpoint |
| 204 No Content | DELETE success, PUT success with no body |
| 400 Bad Request | Validation failed (include error details) |
| 401 Unauthorized | No valid auth |
| 403 Forbidden | Authenticated but no permission (different from 401!) |
| 404 Not Found | Resource doesn't exist |
| 409 Conflict | Resource state conflict (payment already processed) |
| 422 Unprocessable Entity | Business rule violation (insufficient credit) |
| 429 Too Many Requests | Rate limited; include `Retry-After` header |
| 503 Service Unavailable | Downstream dependency down; include `Retry-After` |

**Idempotency keys:**
```http
POST /api/v1/payments
Idempotency-Key: client-generated-uuid-here
Content-Type: application/json

{ "amount": 5000, "partnerId": "...", "currency": "INR" }
```

```java
@PostMapping("/payments")
public ResponseEntity<Payment> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @Valid @RequestBody CreatePaymentRequest request) {
    
    // Check if this request was already processed
    Optional<Payment> existing = idempotencyStore.find(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(existing.get());  // return same response as first time
    }
    
    Payment payment = paymentService.create(request);
    idempotencyStore.store(idempotencyKey, payment, Duration.ofDays(1));
    return ResponseEntity.status(201).body(payment);
}
```

**Cursor-based pagination (superior to offset for large datasets):**
```java
@GetMapping("/payments")
public PageResponse<Payment> getPayments(
        @RequestParam UUID tenantId,
        @RequestParam(required = false) String cursor,   // opaque cursor
        @RequestParam(defaultValue = "20") int limit) {

    UUID afterId = cursor != null ? decodeCursor(cursor) : null;
    
    List<Payment> payments = paymentRepository.findByTenantId(
        tenantId, afterId, limit + 1  // fetch one extra to detect hasMore
    );
    
    boolean hasMore = payments.size() > limit;
    if (hasMore) payments = payments.subList(0, limit);
    
    String nextCursor = hasMore ? encodeCursor(payments.get(limit - 1).getId()) : null;
    return new PageResponse<>(payments, nextCursor, hasMore);
}
```

**API versioning strategy:**
```
URL versioning:   /api/v1/payments  → /api/v2/payments  (visible, cacheable)
Header versioning: Accept: application/vnd.myapp.v2+json  (clean URLs, harder to test)
```
Hierarchy of backward-compatible changes (don't require version bump):
1. Add optional request fields
2. Add response fields
3. Add new endpoints
4. Change error messages (not codes)

Changes that require a version bump:
1. Remove or rename fields
2. Change field types
3. Change required fields to optional or vice versa

---

### Q9. Observability — correlation IDs, metrics, and distributed tracing.

**The answer:**

"Observability is how you understand what's happening inside your system without modifying it. Three pillars: metrics, logs, and traces.

**Correlation IDs:**
```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String correlationId = Optional
            .ofNullable(request.getHeader("X-Correlation-ID"))
            .orElse(UUID.randomUUID().toString());

        MDC.put("correlationId", correlationId);
        MDC.put("tenantId", extractTenantId(request));
        response.setHeader("X-Correlation-ID", correlationId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();  // always clear ThreadLocal
        }
    }
}
```

Logback pattern that includes correlation ID in every log line:
```xml
<pattern>%d{ISO8601} [%thread] %-5level [%X{correlationId}] [%X{tenantId}] %logger{36} - %msg%n</pattern>
```

**Micrometer metrics:**
```java
@Service
public class PaymentService {
    private final MeterRegistry registry;

    private final Counter paymentCounter;
    private final Counter paymentFailureCounter;
    private final Timer paymentTimer;
    private final Gauge activePaymentsGauge;

    public PaymentService(MeterRegistry registry) {
        this.registry = registry;
        this.paymentCounter = Counter.builder("payments.processed")
            .description("Total payments processed")
            .tag("environment", "production")
            .register(registry);

        this.paymentFailureCounter = Counter.builder("payments.failed")
            .register(registry);

        this.paymentTimer = Timer.builder("payments.processing.duration")
            .description("Payment processing latency")
            .publishPercentiles(0.5, 0.95, 0.99)  // p50, p95, p99
            .register(registry);

        this.activePaymentsGauge = Gauge.builder("payments.active", 
            paymentRepository::countByStatus, v -> v.doubleValue())
            .register(registry);
    }

    public Payment processPayment(CreatePaymentRequest request) {
        return paymentTimer.record(() -> {
            try {
                Payment result = doProcess(request);
                paymentCounter.increment();
                return result;
            } catch (Exception e) {
                paymentFailureCounter.increment();
                registry.counter("payments.failed", "reason", e.getClass().getSimpleName())
                        .increment();
                throw e;
            }
        });
    }
}
```

**Histogram for latency distribution:**
```java
// Histogram for payment amount distribution
DistributionSummary.builder("payments.amount")
    .publishPercentiles(0.5, 0.75, 0.9, 0.95, 0.99)
    .publishPercentileHistogram()
    .register(registry)
    .record(payment.getAmount().doubleValue());
```

**Key dashboard metrics to monitor:**
- `payments.processed` rate (success throughput)
- `payments.processing.duration` p99 (latency SLA)
- `payments.failed` rate by reason (failure categorization)
- `jvm.gc.pause` histogram (GC pressure)
- `hikaricp.connections.active` (DB connection pool saturation)
- `kafka.consumer.lag` (Kafka processing falling behind)

---

## PART 5 — FULL HLD DESIGN: MULTI-TENANT PAYMENT PLATFORM

---

### Design a Multi-Tenant Payment Platform

**Step 1 — Requirements:**

Functional:
- Process NEFT/RTGS/UPI payment transactions
- Track payment status in real-time
- Receive and process webhooks from Razorpay
- Dashboard for tenants to view their payments
- Partner credit limit management

Non-Functional:
- 10,000 transactions/day (peak: 50 TPS)
- P99 latency < 500ms for payment API
- Webhooks processed within 30 seconds of receipt
- 99.9% availability
- Multi-tenant: data isolation between tenants
- Compliance: payment data retained 7 years

**Step 2 — Capacity:**
(covered in Part 1 above — single Postgres handles this scale)

**Step 3 — Architecture:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Client Applications                        │
│             (Partner Dashboard, Admin Panel, Mobile Apps)           │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ HTTPS
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS Application Load Balancer                    │
└──────────────────┬───────────────────────────┬──────────────────────┘
                   │                           │
          ┌────────▼────────┐        ┌─────────▼──────────┐
          │   API Gateway   │        │  Webhook Receiver  │
          │  (auth, rate    │        │  (stateless)       │
          │   limit, route) │        │  3 instances       │
          └────────┬────────┘        └─────────┬──────────┘
                   │                           │
          ┌────────▼────────┐                  │
          │  Payment Svc    │        ┌──────────▼──────────┐
          │  (Spring Boot)  │        │    Kafka Cluster    │
          │  3 instances    │        │  (webhook-events)   │
          └────────┬────────┘        └──────────┬──────────┘
                   │                            │
        ┌──────────┼──────────┐      ┌──────────▼──────────┐
        │          │          │      │   Webhook Consumer  │
        ▼          ▼          ▼      │   (3 instances)     │
   PostgreSQL  Redis Cache  S3      └──────────┬──────────┘
   Primary     (sessions,   (docs,             │
   (writes)    credit,      reports)           ▼
               rate limits)          Payment State Machine
        │                            + Outbox Table
        ▼                            + DB Writer
   PostgreSQL
   Replicas ×2
   (reads)
```

**Step 4 — Critical Path Deep Dives:**

**Webhook Processing Path:**
```
Razorpay → POST /webhooks/razorpay
           ↓
    Signature validation (HMAC-SHA256)
           ↓
    Kafka produce (async, at-least-once)
           ↓
    HTTP 200 ACK returned immediately
           ↓ (async)
    Kafka consumer receives event
           ↓
    Redis SETNX deduplication
           ↓
    DB unique constraint check (outbox)
           ↓
    State machine transition
           ↓
    Credit release (if applicable)
           ↓
    WebSocket push to dashboard
```

**Credit Limit Check Path (hot path — < 5ms required):**
```
Lead Assignment Request
           ↓
    Redis: EVAL lua_credit_deduct KEYS[partner:X:credit] ARGV[amount]
           ↓ (atomic)
    Available? → Yes: proceed to assign
                 No: throw InsufficientCreditException
           ↓
    DB write (async outbox): persist the credit deduction
           ↓
    Periodic reconciliation: Redis vs DB credit balance
```

**Step 5 — Failure Analysis:**

| Component Failure | User Impact | Detection | Recovery |
|-------------------|-------------|-----------|----------|
| Single API instance | None | ALB health check, auto-remove | Auto-scaled, 2 remain |
| PostgreSQL Primary | Writes fail (reads ok from replicas) | RDS monitoring | Automatic failover to replica (< 60s with RDS Multi-AZ) |
| Redis failure | Credit checks fall back to DB (slow but correct) | Health check endpoint | Circuit breaker: bypass Redis, use DB pessimistic lock |
| Kafka partition leader | Webhook buffering delayed | Kafka ISR monitoring | Automatic leader election (< 30s) |
| Razorpay API down | New payment initiation fails | HTTP error rate alert | Queue payments, retry with exponential backoff |

---

## PART 6 — BEHAVIORAL & STAR METHOD MASTERY

---

### STAR Framework

**Structure:** Situation (15%) → Task (10%) → **Action (60%)** → Result (15%)

The interviewer cares most about **Action** — specifically what *you* did, the technical choices you made, and how you thought through trade-offs. Front-load your answer with the action: "I designed / I implemented / I proposed / I pushed back on..."

---

### STAR Story 1: Technical Disagreement — Outbox Pattern vs Direct Kafka Write

**Situation:** 3 months into the payment gateway project, we were implementing the webhook processing pipeline. My tech lead proposed direct Kafka writes from the API layer — write to DB, then write to Kafka in the same controller.

**Task:** I believed this had a subtle but critical flaw — a partial failure between the two writes would lose webhooks silently. I needed to convince the team and implement the correct solution.

**Action:** I built a demonstration first. I wrote a test that paused execution between the DB write and the Kafka write and crashed the process. The result: payment in DB, no webhook event in Kafka. I then explained the dual-write problem in a 15-minute architecture review — drew the failure scenarios on a whiteboard, calculated the probability (assuming 99.9% uptime for each write, you get 0.001% of events lost, which at 10,000/day = 10 lost events/day). Then I proposed and implemented the Outbox Pattern: a single DB transaction writing both the payment and the outbox event, with a separate relay process publishing from outbox to Kafka. The relay is idempotent — if it fails, it retries without data loss.

**Result:** The team adopted the Outbox Pattern. In 8 months of production, we had zero webhook events lost. We specifically caught a scenario where our Kafka cluster was briefly unavailable — the outbox queue accumulated 847 events and drained within 2 minutes of recovery. With the original design, those events would have been silently dropped.

---

### STAR Story 2: Deadline vs Technical Correctness — WebSocket Feature Flag

**Situation:** We were 2 weeks from the deadline for the lead assignment system launch. The WebSocket real-time updates feature had a known bug: on page reload, the client reconnected but didn't receive the pending assignments buffered during the disconnection window. The PM wanted to ship as-is.

**Task:** I had to decide: ship with a known UX gap, delay the launch, or find a middle path.

**Action:** I analyzed the actual failure scenario. The bug only manifested during a specific window: leads assigned while the page was loading (< 3 seconds). I quantified the frequency using staging load tests — it affected approximately 2% of lead assignments. I proposed a feature flag approach: ship the real-time push feature, but fall back to a polling mechanism (every 30 seconds) when the WebSocket connection is restored. This reduced the 2% of missed leads to effectively 0% (worst case: 30-second delay instead of real-time). I implemented the fallback in 4 hours, and we shipped on schedule. I documented the known limitation and added it to the technical debt backlog with a clear reproduction scenario.

**Result:** We shipped on time. The fallback worked correctly in production. Three weeks later, during a lower-pressure sprint, I fixed the underlying WebSocket reconnection bug and removed the polling fallback. The PM appreciated the pragmatic approach — we didn't hold up the launch for a 2% edge case.

---

### STAR Story 3: Mentoring — Race Condition Mental Model

**Situation:** A junior developer on the team was implementing a feature that updated partner status from multiple concurrent webhook events. They submitted a PR with a race condition similar to the credit limit bug I'd previously fixed.

**Task:** This was a teaching moment, not just a code review comment.

**Action:** Instead of just adding a comment "use SELECT FOR UPDATE here," I scheduled a 30-minute pair session. I used the analogy of two people reading the same bank balance at the same moment. I showed them the actual race condition with a simple test: two threads updating the same partner object, asserting the final state. The test was flaky — sometimes it passed, sometimes it failed. I explained that a test that's sometimes green is worse than one that's always red, because flakiness hides the bug. Then I walked through the three solutions: pessimistic lock, optimistic lock, and atomic Redis operation. We discussed the trade-offs — not just "use this tool" but "here's when each is appropriate." They implemented the fix themselves, chose optimistic locking correctly (low contention scenario), and added a retry wrapper.

**Result:** The junior developer became the team's go-to person for concurrency questions over the next two months. In their next performance review, they were cited for "strong technical judgment on data consistency problems."

---

### STAR Story 4: Production Incident — 40 Duplicate Webhook Credits

**Situation:** At 2:30 AM on a Monday, our alerting fired: "Payment credits anomaly — 40 partners received double credit in the last 10 minutes."

**Task:** As the on-call engineer, I had to stop the bleeding immediately, diagnose the root cause, and implement a fix.

**Action:**

*Immediate (0-15 minutes):* I disabled the webhook endpoint temporarily (switched to 503 response, which triggers Razorpay's retry queue — safe because they buffer for 24 hours). This stopped new duplicates.

*Diagnosis (15-60 minutes):* Searched logs with the correlation ID from the first duplicate. Found the pattern: our Redis SETNX keys for those specific events had unusually short TTLs — the webhook event IDs contained a timestamp component, and our key generation had a bug: `"webhook:" + eventId.substring(0, 8)` — we were truncating the event ID, causing hash collisions between different events from the same 15-minute window.

*Fix (60-120 minutes):* Fixed the key generation (use full event ID), deployed, re-enabled the webhook endpoint.

*Remediation:* For the 40 affected partners, I wrote a SQL query to identify the duplicate credits, built a compensating transaction that reversed the excess, and ran it in a staging-identical environment first. Applied to production with a transaction and manual verification.

*Systemic fix:* Added a monitoring alert for "duplicate credit detection" — if two credit events reference the same payment reference within 24 hours, alert immediately, don't wait for financial discrepancy reports.

**Result:** Total impact: 40 duplicate credits (₹180,000 excess credits) — all reversed within 4 hours of incident detection. Wrote a detailed post-mortem with the root cause, timeline, and 4 prevention actions. Three of the four actions were implemented in the next sprint.

---

### STAR Story 5: Ambiguity — Java 17 Migration

**Situation:** Our ThingsBoard platform was running on Java 11, but our parent company announced that all services must migrate to Java 17 within 6 months for compliance with their security policy. No technical lead had done this migration for ThingsBoard before. No runbook existed.

**Task:** Plan and execute the migration without disrupting the IoT platform serving 200 active tenants.

**Action:** I started by reading the Java 12-17 release notes and identifying breaking changes relevant to our codebase. Key concerns: (1) removed APIs (sun.misc.*, javax.xml.* moved to jakarta.*), (2) stronger encapsulation (illegal reflective access now errors), (3) new record types and text blocks (beneficial, not breaking). I ran the migration in a fork branch first, using the `--add-opens` JVM flags to diagnose illegal access warnings. Found 3 issues: a Mockito version incompatibility, a Jackson version that used internal reflection, and our own code that used `sun.misc.Unsafe` for performance. Fixed each: updated Mockito to 4.x, Jackson to 2.14+, replaced the Unsafe usage with `VarHandle`. I also updated the Docker base image from `openjdk:11` to `eclipse-temurin:17-jre-alpine` and updated the Spring Boot parent POM to 3.x (which requires Jakarta EE 9 namespace). Created a migration runbook documenting all the steps and the specific error signatures to look for. Ran a full regression test in staging for 2 weeks before production deployment.

**Result:** Zero-downtime migration. Deployed on a Sunday morning with a 4-hour maintenance window as a precaution — the actual migration took 45 minutes with no rollback needed. Post-migration: startup time improved by 15% (G1GC improvements in Java 17), memory usage reduced by ~12% (improved String encoding for ASCII). The runbook was adopted as the company standard for Java 17 migrations.

---

## PART 7 — CRITICAL BACKEND TOPICS

---

### Q10. How do you debug a production performance problem?

**The answer — 4-level framework:**

"I approach production performance problems in four levels, going from coarse to fine:

**Level 1 — Metrics dashboard (first 5 minutes):**
- Is the issue latency (slower responses) or throughput (fewer requests processed) or error rate?
- Which endpoint/service? Single service or cascading?
- When did it start? What changed? (deployment, traffic spike, cron job, external)
- CPU, memory, disk I/O normal? Thread pool saturation? DB connection pool?

**Level 2 — JVM profiling (if JVM is the bottleneck):**
- Attach `async-profiler` or enable JFR (Java Flight Recorder): `jcmd <pid> JFR.start`
- Look for: excessive GC (GC pauses eating CPU), thread contention (blocked waiting for locks), hot methods (tight loops, serialization overhead)
- Common culprits: synchronized method under high concurrency, N+1 queries inside a tight loop, large object allocations triggering frequent GC

**Level 3 — Database diagnosis:**
```sql
-- Active queries taking too long
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle' AND (now() - pg_stat_activity.query_start) > interval '5 seconds';

-- Lock waits
SELECT blocked.pid, blocking.pid, blocked.query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));

-- Missing indexes
SELECT schemaname, tablename, seq_scan, idx_scan,
       seq_scan - idx_scan AS diff
FROM pg_stat_user_tables
ORDER BY diff DESC;
```

**Level 4 — Infrastructure:**
- Network latency between services (unexpected spikes indicate DNS resolution issues or cross-AZ traffic)
- Disk I/O (Postgres WAL writes saturating disk — observe `pg_stat_bgwriter`)
- External dependencies (third-party API response times — check `payment.gateway.duration` histogram)"

---

### Q11. Security — JWT best practices, SQL injection prevention, OWASP Top 10.

**The answer:**

"Security is not optional in a fintech backend. Here's how I approach the OWASP Top 10 in practice:

**A01: Broken Access Control** — In ThingsBoard, we enforce access control at 3 layers: Spring Security (endpoint-level), Service layer (`@PreAuthorize` checks), and DB layer (Hibernate `@Filter` for tenant isolation). Even if one layer fails, the others catch it.

**A02: Cryptographic Failures** — Never store plaintext passwords (`BCryptPasswordEncoder`, cost factor 12+). JWT signed with RS256 (asymmetric) — public key for verification, private key for signing, never shared. Secrets in AWS Secrets Manager, not in environment variables or code.

**A03: Injection** — Spring Data JPA parameterized queries prevent SQL injection by default. Never use string concatenation in JPQL. For native queries, always use `@Param`:
```java
// SAFE
@Query("SELECT p FROM Payment p WHERE p.tenantId = :tenantId AND p.status = :status")
List<Payment> findBy(@Param("tenantId") UUID tenantId, @Param("status") String status);

// DANGEROUS — never do this
@Query("SELECT p FROM Payment p WHERE p.status = '" + status + "'")  // SQL injection
```

**A07: Identification and Authentication Failures:**
JWT best practices:
- Short expiry (15 min for access tokens, 7 days for refresh tokens)
- Store refresh tokens in DB with revocation capability
- Rotate refresh tokens on each use (sliding window)
- Never put sensitive data in JWT payload (it's Base64, not encrypted)
- Validate `iss`, `aud`, `exp` claims on every request

**A09: Security Logging and Monitoring:**
Log all auth events (login, logout, token refresh, failed auth) with correlation ID, IP, user agent. Alert on: brute force patterns (> 5 failed logins in 60 seconds), unusual geo-location, admin actions on production data."

---

## PART 8 — MANAGERIAL ROUND QUESTION BANK

---

### "Tell me about yourself and why you're ready for a Senior role."

"I'm a backend engineer with 4.5 years of experience primarily in Java, Spring Boot, and distributed systems. I've spent the last 2 years on a fintech platform where I owned the payment processing backend end-to-end — from the API design through the webhook processing pipeline to the data model.

What I think makes me ready for Senior is not just the years, but the nature of the problems I've worked on. I've debugged a production incident where 40 partners received duplicate credits — that required understanding the entire system, from the Redis key generation bug to the compensating transaction to the systemic monitoring fix. I've led architectural decisions — advocating for the Outbox Pattern when the simpler approach had a hidden failure mode. And I've mentored junior developers on concurrency patterns — explaining not just what to use but why.

The next step for me is taking broader technical ownership — leading a team's architecture decisions, mentoring more systematically, and driving the technical roadmap rather than contributing to it."

---

### "Where do you see yourself in 3-5 years?"

"In 3-5 years, I see myself as a Staff or Principal Engineer. That means I'm not just writing code — I'm defining the technical direction for a system or a team. I'm the person who thinks about: what are the architectural risks we're not seeing? Which technical debt will limit us in 18 months? How do we build a system that the team can operate at 2 AM without me?

I want to get there by deepening in distributed systems and platform engineering, and by becoming more effective at communicating technical decisions to non-technical stakeholders. I've noticed that the most impactful engineers I've worked with are the ones who can translate a complex architectural trade-off into a business decision in one sentence."

---

### "What's your biggest weakness?"

"My biggest weakness is that I sometimes want to solve a problem perfectly before I communicate that it exists. In the Java 17 migration, I spent two weeks diagnosing all the issues before I brought the full picture to my manager. In hindsight, I should have flagged the scope and the timeline risk at the end of week one, not week three. I'm actively working on this — I now do weekly async updates even on work that isn't complete, framed as 'here's what I know, here's what I'm still investigating, here's my current confidence in the timeline.' The goal is to give stakeholders enough information to make decisions without requiring me to have all the answers first."

---

### "Describe a time you had to say no to a technical request."

"A product manager asked us to add a 'bulk payment upload' feature with a 2-week turnaround. The initial ask was: accept a CSV, process all payments synchronously in the request-response cycle. I pushed back on the synchronous approach — a 10,000-row CSV at 100ms per payment would take 1,000 seconds. Users would time out.

I explained the trade-off: we can do synchronous processing now (2 weeks, but breaks for large files), or async processing (3 weeks, but scalable). I proposed a middle path: accept the CSV, immediately return a job ID, process async, and provide a status polling endpoint. The PM agreed once they understood the user experience difference. We delivered in 3.5 weeks. The async approach handled CSVs with up to 50,000 rows without issues. Had we shipped the synchronous version, we would have had to rewrite it anyway after the first large upload broke."

---

### "How do you handle criticism of your technical decisions?"

"I try to separate two questions: is the criticism technically correct, and is it delivered constructively? I focus on the first one and don't let the second one determine my response.

When my tech lead criticized my initial webhook deduplication implementation as 'over-engineered,' I stepped back and reviewed my design from their perspective. They were right that Layer 3 (Kafka buffer) added operational complexity for a scale we hadn't reached yet. I explained my rationale — Razorpay's retry window and the need for replay capability — and we had a productive discussion. We kept the Kafka layer but simplified the consumer by removing a retry queue that was genuinely redundant given the existing layers. The final design was better for having both perspectives.

If I receive criticism that I believe is technically wrong, I present my reasoning with evidence — not defensiveness. I've been wrong before; starting from 'let me understand your concern' rather than 'you're wrong' almost always leads to a better outcome."

---

### "Describe your leadership style."

"I lead by making my thinking visible. When I make a technical decision, I write down the alternatives I considered and why I ruled them out. This does two things: it forces me to think rigorously, and it gives team members context to challenge me if they see something I missed.

I also believe in giving people problems, not solutions. When a junior developer came to me with 'how do I prevent duplicate orders?', my instinct was to say 'use a unique constraint.' Instead, I asked them to spend 30 minutes researching idempotency patterns and come back with three options. They came back with the unique constraint, an idempotency key approach, and an optimistic locking approach — with pros and cons for each. That took more time up front, but they genuinely learned the design space rather than just implementing a recipe.

I think the best thing I can do as a Senior is leave the team more capable than I found it."

---

## QUICK-REFERENCE — Senior Filter Questions (Chapter 4)

**Q: Message queue vs event stream — what's the difference?**
Message queue (RabbitMQ, SQS): message consumed → deleted. Point-to-point. Each message goes to one consumer. Event stream (Kafka): messages retained for configurable duration. Multiple consumer groups can independently replay the same stream. Use message queues for task distribution; use event streams for event sourcing, CQRS read model updates, audit logs, fan-out to multiple services.

**Q: What's the difference between horizontal and vertical scaling, and when does each fail?**
Vertical scaling (bigger machine) fails at hardware limits and introduces single points of failure. Horizontal scaling (more machines) requires stateless services, distributed coordination, and consistent hashing for data. Horizontal scaling fails when you hit shared state bottlenecks: the database (shard or use a distributed DB), session state (externalize to Redis), distributed locks (use Redis Redlock or Zookeeper).

**Q: Synchronous vs asynchronous communication — when do you choose each?**
Synchronous (REST, gRPC): when you need an immediate response, when the caller cannot proceed without the result, when the operation is short-lived. Asynchronous (Kafka, SQS): when the receiver can work independently, when you need to decouple throughputs (producer is faster than consumer), when you need durability and replay capability, when the operation can fail and should be retried without blocking the caller.

**Q: How do you prevent cascade failures in microservices?**
Four patterns, typically combined:
1. **Circuit Breaker** — stop calling a failing service, fast-fail until it recovers
2. **Bulkhead** — isolate resources per downstream service (separate thread pool per dependency — one dependency exhausting its pool doesn't starve others)
3. **Timeout** — every outbound call has a timeout; never wait indefinitely
4. **Fallback** — when a call fails, return cached data, a default response, or a graceful error instead of propagating the failure

---

---

## PART 9 — ADDITIONAL DISTRIBUTED PATTERNS

---

### Q13. Explain Event Sourcing. How does it differ from CQRS?

**The answer:**

"Event Sourcing and CQRS are often mentioned together but are independent patterns. CQRS separates read and write models. Event Sourcing changes what the write model *stores* — instead of storing current state, you store the sequence of events that produced that state.

**Traditional (state-based) storage:**
```
payments table:
| id  | status    | amount | updated_at |
| P1  | COMPLETED | 5000   | 2024-01-15 |
```
You only know the current state. You lost the history: was it PENDING → PROCESSING → COMPLETED? Did it ever fail and retry?

**Event Sourcing — store every event, derive state:**
```
payment_events table (append-only):
| event_id | payment_id | event_type          | payload              | timestamp  |
| E1       | P1         | PAYMENT_INITIATED    | {amount: 5000, ...}  | 10:00:00   |
| E2       | P1         | PAYMENT_PROCESSING   | {gateway: razorpay}  | 10:00:01   |
| E3       | P1         | GATEWAY_FAILED       | {reason: timeout}    | 10:00:31   |
| E4       | P1         | PAYMENT_RETRIED      | {}                   | 10:01:00   |
| E5       | P1         | PAYMENT_COMPLETED    | {txn_id: TXN-456}    | 10:01:02   |
```
To get current state: replay E1 → E5. The history is never lost.

**Implementation:**
```java
// Event — immutable record of something that happened
public record PaymentInitiatedEvent(UUID paymentId, BigDecimal amount,
                                     String currency, Instant occurredAt) {}

// Aggregate — reconstructed by replaying events
public class PaymentAggregate {
    private UUID id;
    private PaymentStatus status;
    private BigDecimal amount;
    private List<Object> uncommittedEvents = new ArrayList<>();

    // Reconstitute from event history
    public static PaymentAggregate reconstitute(List<Object> events) {
        PaymentAggregate agg = new PaymentAggregate();
        events.forEach(agg::apply);
        return agg;
    }

    // Business command → validates + raises event
    public void initiate(BigDecimal amount, String currency) {
        if (this.status != null) throw new IllegalStateException("Already initiated");
        apply(new PaymentInitiatedEvent(id, amount, currency, Instant.now()));
    }

    // Apply event — mutates state, no validation here
    private void apply(Object event) {
        switch (event) {
            case PaymentInitiatedEvent e -> { this.status = PENDING; this.amount = e.amount(); }
            case PaymentCompletedEvent e -> { this.status = COMPLETED; }
            case PaymentFailedEvent e    -> { this.status = FAILED; }
            default -> {}
        }
        uncommittedEvents.add(event);
    }
}

// Repository: load by replaying events
@Service
public class PaymentEventRepository {
    public PaymentAggregate load(UUID paymentId) {
        List<Object> events = eventStore.findByPaymentId(paymentId);
        return PaymentAggregate.reconstitute(events);
    }

    public void save(PaymentAggregate agg) {
        eventStore.appendAll(agg.getUncommittedEvents());
        agg.clearUncommittedEvents();
    }
}
```

**Snapshots — avoid replaying 10,000 events:**
```java
// Every N events, save a snapshot of current state
if (events.size() % 100 == 0) {
    snapshotStore.save(new PaymentSnapshot(paymentId, agg.getStatus(), agg.getAmount(), events.size()));
}

// On load: start from snapshot, replay only events after it
PaymentSnapshot snapshot = snapshotStore.findLatest(paymentId);
List<Object> recentEvents = eventStore.findAfterVersion(paymentId, snapshot.version());
PaymentAggregate agg = PaymentAggregate.reconstitute(snapshot, recentEvents);
```

**Event Sourcing vs CQRS — the relationship:**
- CQRS: separate read/write models. Can be used without Event Sourcing.
- Event Sourcing: store events instead of state. The event log naturally becomes the write model. The read model is built by projecting events.
- Together: events drive both the write model (event store) and the read model (projections update Elasticsearch/dashboard DB).

**When NOT to use Event Sourcing:**
- Simple CRUD apps with no audit requirements
- High-volume, low-complexity data (sensor readings every millisecond → event log becomes enormous fast)
- Teams without experience — the operational overhead (event versioning, schema evolution) is significant"

---

### Q14. gRPC vs REST — when do you choose each?

**The answer:**

"REST and gRPC solve the same problem — service-to-service communication — with different trade-offs.

**REST (HTTP/JSON):**
- Human-readable, easy to debug with curl/Postman
- Universally supported — browsers, mobile, any HTTP client
- Schema optional (OpenAPI/Swagger)
- Stateless, cacheable

**gRPC (HTTP/2 + Protocol Buffers):**
- Binary serialization — 5–10x smaller payload than JSON for same data
- Strongly typed contracts via `.proto` files — compiler catches breaking changes
- HTTP/2 native: multiplexing, header compression, bidirectional streaming
- Streaming support: unary, server streaming, client streaming, bidirectional

```protobuf
// payment.proto
syntax = \"proto3\";

service PaymentService {
  rpc GetPayment(GetPaymentRequest) returns (Payment);
  rpc StreamPaymentEvents(StreamRequest) returns (stream PaymentEvent);  // server streaming
  rpc ProcessBatch(stream Payment) returns (BatchResult);  // client streaming
}

message Payment {
  string id = 1;
  double amount = 2;
  string status = 3;
  string tenant_id = 4;
}
```

```java
// gRPC server implementation
@GrpcService
public class PaymentGrpcService extends PaymentServiceGrpc.PaymentServiceImplBase {

    @Override
    public void getPayment(GetPaymentRequest request,
                           StreamObserver<Payment> responseObserver) {
        Payment payment = paymentRepository.findById(request.getId())
            .map(this::toProto)
            .orElseThrow();
        responseObserver.onNext(payment);
        responseObserver.onCompleted();
    }

    @Override
    public void streamPaymentEvents(StreamRequest request,
                                    StreamObserver<PaymentEvent> observer) {
        // Push events as they happen
        eventSubscription.subscribe(event -> {
            observer.onNext(toProto(event));
        });
    }
}
```

**Decision table:**

| Scenario | Use |
|----------|-----|
| Public API (browsers, partners, mobile) | REST — universal support |
| Internal microservice-to-microservice | gRPC — performance, type safety |
| Streaming / real-time event push | gRPC streaming |
| Human-debuggable, simple CRUD | REST |
| High-throughput, low-latency internal | gRPC |
| Polyglot services (Java + Python + Go) | gRPC — `.proto` as shared contract |

**Senior Signal:** 'In practice, most production systems use both. External-facing APIs are REST (clients are browsers or third-party apps that need JSON). Internal service communication, especially for high-frequency calls like telemetry ingestion or ML inference, uses gRPC. The `.proto` file becomes the service contract enforced at compile time — much stronger than OpenAPI for internal teams.'"

---

### Q15. Long Polling vs Server-Sent Events vs WebSocket — exact comparison.

**The answer:**

"These are three patterns for pushing data from server to client, each with different capabilities and trade-offs.

**Long Polling:**
Client sends a request. Server holds it open until it has data (or timeout). Client immediately re-polls after receiving a response.

```
Client → GET /notifications (held open)
Server ← 200 {event: "lead_assigned"}  (data ready, responds immediately)
Client → GET /notifications (immediately re-polls)
```

```java
@GetMapping("/notifications/poll")
public DeferredResult<List<Notification>> poll(
        @RequestParam String userId,
        @RequestParam(defaultValue = "30000") long timeout) {

    DeferredResult<List<Notification>> result = new DeferredResult<>(timeout, List.of());
    pendingRequests.put(userId, result);

    // If event arrives while holding:
    // pendingRequests.get(userId).setResult(notifications);

    return result;
}
```
Pro: Works everywhere, no special protocol. Con: High connection overhead, server holds many open connections.

**Server-Sent Events (SSE):**
Single long-lived HTTP connection. Server pushes events to client. Client uses `EventSource` API (browser-native). One-directional (server → client only). Auto-reconnects on disconnect.

```java
@GetMapping(value = "/notifications/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamNotifications(@RequestParam String userId) {
    SseEmitter emitter = new SseEmitter(Long.MAX_VALUE);
    sseEmitters.put(userId, emitter);

    emitter.onCompletion(() -> sseEmitters.remove(userId));
    emitter.onTimeout(() -> sseEmitters.remove(userId));

    return emitter;
}

// Pushing an event:
public void notifyUser(String userId, Notification notification) {
    SseEmitter emitter = sseEmitters.get(userId);
    if (emitter != null) {
        emitter.send(SseEmitter.event()
            .name("notification")
            .data(notification, MediaType.APPLICATION_JSON)
            .id(notification.getId()));
    }
}
```
Pro: Simple, works over HTTP/1.1, browser auto-reconnects, text/UTF-8. Con: Server → client only, no binary data.

**WebSocket:**
Bidirectional, full-duplex connection. HTTP upgrade handshake, then raw TCP frames. Both sides can send at any time.
Pro: True bidirectional, low overhead per message (no HTTP headers after handshake), binary support. Con: More complex (need session management, heartbeats, reconnect logic), stateful (complicates scaling — need Redis Pub/Sub relay as in your project).

**Comparison table:**

| | Long Polling | SSE | WebSocket |
|-|-------------|-----|-----------|
| Direction | Both | Server → Client | Bidirectional |
| Protocol | HTTP | HTTP | WS (upgraded HTTP) |
| Browser reconnect | Manual | Automatic | Manual |
| Binary data | Via Base64 | No | Yes |
| Load balancer friendly | Yes | Yes | Needs sticky or Redis |
| Use case | Simple push, legacy browsers | News feeds, notifications | Chat, real-time collab, gaming |

**Your project:** 'We use WebSocket for lead assignment notifications because partners need to receive assignments in < 500ms and also send acknowledgements back (bidirectional). SSE would cover the receive side, but the bidirectional need drove the WebSocket choice.'"

---

### Q16. Explain OAuth2 and OIDC. How does Spring Security integrate with them?

**The answer:**

"OAuth2 is an authorization framework. It allows a third-party app to access resources on behalf of a user, without the user giving their password to the third-party app. OIDC (OpenID Connect) is a thin identity layer on top of OAuth2 — it adds authentication (who is the user?) to OAuth2's authorization (what can they access?).

**The four OAuth2 grant types:**

**Authorization Code Flow (web apps — most common):**
```
User → App → Auth Server: 'I want to log in'
Auth Server → User: 'Please log in'
User → Auth Server: credentials
Auth Server → App: authorization_code
App → Auth Server: exchange code for tokens
Auth Server → App: access_token + refresh_token + id_token (OIDC)
App → Resource Server: Bearer access_token
Resource Server: validates token → returns data
```

**Authorization Code + PKCE (mobile/SPA — no client secret):**
Same as above, but the app generates a `code_verifier` (random string) and sends a `code_challenge` (SHA-256 hash) with the initial request. On token exchange, it proves it knows the original `code_verifier`. Prevents authorization code interception attacks.

**Client Credentials (machine-to-machine, no user involved):**
```
Service A → Auth Server: client_id + client_secret
Auth Server → Service A: access_token (no user context)
Service A → Service B: Bearer access_token
```
Use for: microservice-to-microservice API calls, backend jobs.

**JWT vs Opaque Tokens:**
- JWT: self-contained, resource server validates locally (no call to auth server). Pro: fast. Con: cannot be revoked until expiry. Mitigation: short expiry (15 min).
- Opaque token: just a random string. Resource server must call auth server introspection endpoint to validate. Pro: revocable instantly. Con: adds auth server as a hot path dependency.

**Spring Security OAuth2 Resource Server:**
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.mycompany.com
          # Spring auto-downloads JWKS (public keys) from issuer's well-known endpoint
```

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtConverter()))
            )
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/v1/admin/**").hasAuthority("SCOPE_admin")
                .requestMatchers("/api/v1/payments/**").hasAuthority("SCOPE_payments:write")
                .anyRequest().authenticated()
            )
            .build();
    }

    @Bean
    public JwtAuthenticationConverter jwtConverter() {
        JwtGrantedAuthoritiesConverter converter = new JwtGrantedAuthoritiesConverter();
        converter.setAuthoritiesClaimName("scope");
        converter.setAuthorityPrefix("SCOPE_");

        JwtAuthenticationConverter jwtConverter = new JwtAuthenticationConverter();
        jwtConverter.setJwtGrantedAuthoritiesConverter(converter);
        return jwtConverter;
    }
}
```

**Refresh token rotation:**
```
Access token: short-lived (15 min)
Refresh token: long-lived (7 days), stored server-side with revocation flag

On refresh:
  App → Auth Server: refresh_token
  Auth Server: invalidate old refresh_token, issue new access_token + new refresh_token
  // Rotation: if old refresh token is used again → FRAUD SIGNAL → revoke all tokens for user
```

**Scopes vs Roles:**
- Scopes (OAuth2): represent what the token holder is allowed to do — coarse-grained API access (`payments:read`, `payments:write`, `admin`)
- Roles (your app): represent who the user is — fine-grained business authorization (`PARTNER`, `TENANT_ADMIN`, `SUPERADMIN`)
Both are needed: scopes at the API gateway level, roles inside the application for business logic."

---

## PART 10 — PRODUCTION READINESS

---

### Q17. Spring Boot Actuator — what is it and what do you expose in production?

**The answer:**

"Spring Boot Actuator exposes operational endpoints over HTTP (and JMX) for monitoring and managing a running application. It's what connects your app to Kubernetes health probes, Prometheus scraping, and ops tooling.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus  # only expose what you need
  endpoint:
    health:
      show-details: when-authorized  # hide details from unauthenticated requests
      probes:
        enabled: true  # enables /actuator/health/liveness and /actuator/health/readiness
  metrics:
    export:
      prometheus:
        enabled: true  # /actuator/prometheus — Prometheus scrapes this
```

**Key endpoints:**

| Endpoint | Purpose |
|----------|---------|
| `/actuator/health` | Overall health status (UP/DOWN) |
| `/actuator/health/liveness` | Is the JVM alive? (restart if DOWN) |
| `/actuator/health/readiness` | Ready to receive traffic? (remove from LB if DOWN) |
| `/actuator/metrics` | List all Micrometer metric names |
| `/actuator/prometheus` | Prometheus-format metrics dump |
| `/actuator/info` | App version, git commit, build info |
| `/actuator/env` | (NEVER in prod — exposes config properties) |

**Liveness vs Readiness — the critical K8s distinction:**
```yaml
# Kubernetes probe config
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30   # wait for JVM warmup
  periodSeconds: 10
  failureThreshold: 3        # restart pod after 3 failures

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3        # remove from Service endpoints after 3 failures
```

- **Liveness failure** → Pod is restarted (JVM deadlock, OOM)
- **Readiness failure** → Pod removed from load balancer (DB connection lost, Kafka not ready), but NOT restarted

**Custom health indicator:**
```java
@Component
public class KafkaHealthIndicator implements HealthIndicator {

    private final KafkaAdmin kafkaAdmin;

    @Override
    public Health health() {
        try {
            kafkaAdmin.describeTopics("payment-events");
            return Health.up()
                .withDetail("topic", "payment-events reachable")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
            // This makes /actuator/health/readiness DOWN
            // → K8s removes pod from load balancer until Kafka recovers
        }
    }
}
```

**Securing Actuator — never expose to the internet:**
```java
@Bean
public SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
    return http
        .requestMatcher(EndpointRequest.toAnyEndpoint())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(EndpointRequest.to("health", "info")).permitAll()
            .anyRequest().hasRole("ACTUATOR_ADMIN")  // internal monitoring only
        )
        .build();
}
```"

---

### Q18. How do you configure CORS in Spring Boot? What is CORS actually protecting?

**The answer:**

"CORS (Cross-Origin Resource Sharing) is a **browser security policy** — it prevents JavaScript running on `evil.com` from making API calls to `mybank.com` and reading the response. It is NOT a server security feature — it does nothing against non-browser clients (curl, Postman, server-to-server calls).

**How it works:**
1. Browser makes a cross-origin request (e.g., `app.myco.com` → `api.myco.com`)
2. For non-simple requests (POST, PUT, custom headers), browser sends a preflight `OPTIONS` first
3. Server responds with `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, etc.
4. Browser checks the response and either allows or blocks the actual request

**Configuring CORS in Spring:**

```java
// Option 1 — Global config (recommended for APIs)
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(
                "https://app.mycompany.com",
                "https://partner-portal.mycompany.com"
            )
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("Content-Type", "Authorization", "Idempotency-Key")
            .allowCredentials(true)
            .maxAge(3600);  // preflight cache duration (1 hour)
    }
}

// Option 2 — Per controller (fine-grained)
@RestController
@CrossOrigin(origins = "https://app.mycompany.com", maxAge = 3600)
public class PaymentController { }

// NEVER in production:
.allowedOrigins("*")  // allows ANY origin — defeats the purpose
// And if you use allowedOrigins("*") with allowCredentials(true) → Spring throws an error
// (correctly — these two settings together are a security hole)
```

**Security implications:**
- CORS does NOT prevent CSRF (a user on `evil.com` can still trick a browser into making state-changing requests via form submission — CORS only blocks reading the response, not sending the request)
- Use CORS + CSRF protection together for browser-based apps
- For stateless JWT APIs: CSRF is less of a concern (no cookies), but CORS still matters for response data protection"

---

### Q19. What is graceful shutdown and how do you implement it in Spring Boot?

**The answer:**

"Graceful shutdown means: when the process receives a termination signal, it stops accepting new requests, waits for in-flight requests to complete, then exits cleanly. Without it, abrupt termination kills in-flight requests mid-processing — users get 500 errors, DB transactions are rolled back, and Kafka offsets may not be committed.

**Spring Boot built-in graceful shutdown (2.3+):**
```yaml
server:
  shutdown: graceful  # default is 'immediate'

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # max time to wait for in-flight requests
```

When `SIGTERM` is received:
1. Spring stops accepting new HTTP requests (returns 503 to load balancer)
2. Waits up to 30s for in-flight requests to complete
3. Fires `@PreDestroy` methods and `DisposableBean.destroy()` on all beans
4. JVM exits

**What happens in Kubernetes:**
```
kubectl delete pod payment-api-xxx
  → K8s sends SIGTERM to pod
  → K8s removes pod from Service endpoints (stops new traffic)
  → Spring: graceful shutdown starts (30s window)
  → K8s waits terminationGracePeriodSeconds (default 30s)
  → If pod still running: K8s sends SIGKILL (force kill)
```

```yaml
# Kubernetes Deployment — align with Spring's timeout
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60  # > Spring's 30s (buffer for JVM overhead)
      containers:
        - name: payment-api
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]
                # 5s sleep BEFORE SIGTERM — gives K8s time to remove pod from
                # load balancer before Spring stops accepting requests
                # Prevents race condition: traffic still routed to pod after SIGTERM
```

**Handling Kafka consumer shutdown:**
```java
@KafkaListener(topics = "payment-events")
public void consume(ConsumerRecord<String, Payment> record) {
    // Spring Kafka registers a shutdown hook:
    // - Stops polling on SIGTERM
    // - Waits for current message processing to complete
    // - Commits offsets
    // - Closes consumer
    processPayment(record.value());
}
```

**Senior Signal:** 'The non-obvious part is the `preStop` sleep. Without it, there's a race condition: K8s sends SIGTERM to stop the pod AND simultaneously removes it from the Service endpoint list — but the endpoint list propagation takes a few seconds to reach all load balancers. During those seconds, the pod is receiving SIGTERM (stopping) but still getting traffic. The `preStop` sleep introduces a short delay before the SIGTERM lands, giving the endpoint removal time to propagate.'"

---

> **What's Next — Chapter 5:** HLD Advanced & Full Project Narratives — Twitter Feed, Distributed Cache, Real-Time Leaderboard, Job Scheduler, Search Autocomplete, Financial Reconciliation System, resume bullet refinement, and deployment strategies.

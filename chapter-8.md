# Interview Survival Guide
## Chapter 8: Spring WebFlux, Reactive Programming & HTTP Client Patterns
### (Modern async I/O — when and how to use it)

---

## PART 1 — WHY REACTIVE? THE THREAD-PER-REQUEST PROBLEM

---

### Q1. What problem does reactive programming solve? Why can't you just use more threads?

**The answer:**

"Spring MVC (the traditional model) is built on the Servlet API's thread-per-request model: one thread handles one request, from receipt to response. That thread blocks whenever it waits for I/O — a DB query, an HTTP call to a partner service, a Redis read.

**The thread-per-request bottleneck:**
```
Request arrives → Thread A allocated
  → query DB (25ms wait) → Thread A BLOCKED (CPU idle)
  → DB returns           → Thread A ACTIVE (2ms)
  → call Razorpay (150ms wait) → Thread A BLOCKED
  → Razorpay returns     → Thread A ACTIVE (5ms)
  → Response sent        → Thread A released

Thread A was alive for 182ms but did useful work for only 7ms (3.8% utilisation)
```

Under high load, all 200 Tomcat threads are blocked on I/O. New requests queue. Latency spikes.

**The reactive solution — non-blocking I/O:**
```
Request arrives → Event loop thread handles it
  → query DB → thread RELEASED back to pool (callback registered)
  → DB returns → event loop picks up callback, CONTINUES on any free thread
  → call Razorpay → thread RELEASED again
  → Razorpay returns → CONTINUES
  → Response sent

1 thread handles hundreds of in-flight I/O operations simultaneously
```

**When reactive wins:**
- High-concurrency, I/O-bound workloads (many concurrent requests, each doing external calls)
- Streaming data (real-time feeds, SSE, WebSocket at scale)
- API gateway / proxy (fan-out to many downstream services)

**When reactive is NOT the answer:**
- CPU-bound work (reactive doesn't help — you're compute-limited, not I/O-limited)
- Simple CRUD apps with low concurrency — thread-per-request is simpler, debuggable, well-understood
- Blocking dependencies you can't change (JDBC is blocking — WebFlux + JDBC defeats the purpose; use R2DBC)
- Teams unfamiliar with reactive — debugging async stack traces is significantly harder

**Senior Signal:** 'The key question before going reactive is: are you actually I/O-bound at a scale where thread exhaustion is the bottleneck? For a service handling 1000 req/sec with 200 Tomcat threads — each request taking 100ms — that's 200 concurrent requests = 200 threads. Completely fine. Reactive starts mattering at 10,000+ concurrent requests with high I/O wait. Most backend services never need it. The right answer is always: profile first.'"

---

## PART 2 — PROJECT REACTOR: MONO AND FLUX

---

### Q2. Explain `Mono` and `Flux`. What are the core operators?

**The answer:**

"Project Reactor is the reactive library underlying Spring WebFlux. It provides two core types:

- **`Mono<T>`** — represents 0 or 1 asynchronous value (like `CompletableFuture<Optional<T>>`)
- **`Flux<T>`** — represents 0 to N asynchronous values (an async stream)

**Creating:**
```java
// Mono
Mono<Payment> payment = Mono.just(new Payment());       // eager — value ready
Mono<Payment> lazy = Mono.fromSupplier(() -> findPayment()); // lazy — supplier called on subscribe
Mono<Payment> fromFuture = Mono.fromFuture(asyncFetch());
Mono<Void> empty = Mono.empty();
Mono<Payment> error = Mono.error(new PaymentNotFoundException("PAY-001"));

// Flux
Flux<Payment> fromList = Flux.fromIterable(paymentList);
Flux<Long> ticker = Flux.interval(Duration.ofSeconds(1));  // emits 0,1,2,3... every second
Flux<Payment> range = Flux.range(1, 10).map(i -> new Payment(i));
```

**Transforming operators:**
```java
Mono<PaymentResponse> pipeline = paymentRepository.findById(paymentId)  // Mono<Payment>

    // map: synchronous transform (like Stream.map)
    .map(payment -> enrichWithMetadata(payment))

    // flatMap: async transform — input produces a Mono, flattens it
    .flatMap(payment -> partnerService.getCreditInfo(payment.getPartnerId())
        .map(credit -> new PaymentWithCredit(payment, credit)))

    // filter: keep only if condition holds (Mono becomes empty if false)
    .filter(pwc -> pwc.getCreditAvailable().compareTo(pwc.getPayment().getAmount()) >= 0)

    // switchIfEmpty: provide a fallback if Mono is empty
    .switchIfEmpty(Mono.error(new InsufficientCreditException()))

    // map to response DTO
    .map(PaymentResponse::from);
```

**Error handling:**
```java
Mono<Payment> resilient = paymentMono
    .onErrorReturn(RazorpayException.class, Payment.failed())  // fallback value on error
    .onErrorResume(PartnerServiceException.class,
        ex -> cachedPartnerService.getCredit(partnerId))       // fallback to cache
    .onErrorMap(IOException.class,
        ex -> new PaymentProcessingException("Network error", ex));  // translate exception
```

**Combining:**
```java
// zip: combine two Monos when both complete
Mono<PaymentResult> combined = Mono.zip(
    fraudService.check(paymentId),     // Mono<FraudResult>
    kycService.check(paymentId),       // Mono<KycResult>
    (fraud, kyc) -> combine(fraud, kyc)
);

// Flux mergeWith: merge two streams
Flux<Event> allEvents = paymentEvents.mergeWith(refundEvents);

// Flux flatMap: transform each element into an async stream, merge results
Flux<PaymentResult> results = paymentIds.flux()
    .flatMap(id -> processPaymentAsync(id), 10);  // max 10 concurrent
```

**Backpressure:**
```java
Flux<Payment> highVolumeStream = paymentEventStream();

highVolumeStream
    .onBackpressureBuffer(1000,          // buffer up to 1000 if consumer is slow
        dropped -> log.warn("Dropped: {}", dropped),
        BufferOverflowStrategy.DROP_OLDEST)
    .subscribe(
        payment -> processSlowly(payment),
        error -> log.error("Stream error", error)
    );
```

---

## PART 3 — SPRING WEBFLUX IN PRACTICE

---

### Q3. Show me a WebFlux controller and explain how it differs from Spring MVC.

**The answer:**

"WebFlux controllers look almost identical to MVC controllers — the difference is return types: `Mono<T>` and `Flux<T>` instead of `T` and `List<T>`.

```java
// Spring MVC (blocking)
@RestController
public class PaymentController {

    @GetMapping("/api/v1/payments/{id}")
    public Payment getPayment(@PathVariable UUID id) {
        return paymentService.findById(id);  // blocks thread while fetching from DB
    }

    @GetMapping("/api/v1/payments")
    public List<Payment> getPayments(@RequestParam UUID tenantId) {
        return paymentService.findByTenant(tenantId);
    }
}

// Spring WebFlux (non-blocking)
@RestController
public class ReactivePaymentController {

    @GetMapping("/api/v1/payments/{id}")
    public Mono<Payment> getPayment(@PathVariable UUID id) {
        return paymentService.findById(id);  // returns immediately, pipeline executes async
    }

    @GetMapping("/api/v1/payments")
    public Flux<Payment> getPayments(@RequestParam UUID tenantId) {
        return paymentService.findByTenant(tenantId);  // streamed to client as data arrives
    }

    // SSE streaming with Flux
    @GetMapping(value = "/api/v1/payments/stream",
                produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Payment> streamPayments(@RequestParam UUID tenantId) {
        return paymentEventService.streamForTenant(tenantId)  // infinite Flux
            .delayElements(Duration.ofMillis(100));  // backpressure-aware
    }
}
```

**Service layer — reactive all the way down:**
```java
@Service
public class ReactivePaymentService {

    private final R2dbcPaymentRepository repository;  // non-blocking DB driver
    private final WebClient partnerWebClient;

    // Parallel external calls with reactive
    public Mono<PaymentResult> processPayment(Payment payment) {
        Mono<FraudResult> fraudCheck = webClient.get()
            .uri("/fraud/check/{id}", payment.getId())
            .retrieve()
            .bodyToMono(FraudResult.class)
            .timeout(Duration.ofMillis(500))
            .onErrorReturn(FraudResult.unknown());

        Mono<CreditResult> creditCheck = webClient.get()
            .uri("/partners/{id}/credit", payment.getPartnerId())
            .retrieve()
            .bodyToMono(CreditResult.class)
            .timeout(Duration.ofMillis(500));

        return Mono.zip(fraudCheck, creditCheck)
            .flatMap(tuple -> {
                if (tuple.getT1().isHighRisk() || !tuple.getT2().hasSufficientCredit()) {
                    return Mono.error(new PaymentDeclinedException());
                }
                return repository.save(payment);
            })
            .map(PaymentResult::success);
    }
}
```

**The blocking code trap — the most common WebFlux mistake:**
```java
// BROKEN — blocking call inside reactive chain
@GetMapping("/payments/{id}")
public Mono<Payment> getPayment(@PathVariable UUID id) {
    return Mono.fromCallable(() -> jdbcTemplate.queryForObject(...));
    // jdbcTemplate.queryForObject() BLOCKS the event loop thread
    // All other requests stall while waiting for the DB
}

// FIX — wrap blocking calls explicitly with a bounded scheduler
@GetMapping("/payments/{id}")
public Mono<Payment> getPayment(@PathVariable UUID id) {
    return Mono.fromCallable(() -> jdbcTemplate.queryForObject(...))
        .subscribeOn(Schedulers.boundedElastic());  // runs on a separate blocking-safe thread pool
}

// BEST — use a non-blocking driver (R2DBC for DB, WebClient for HTTP)
@GetMapping("/payments/{id}")
public Mono<Payment> getPayment(@PathVariable UUID id) {
    return r2dbcPaymentRepository.findById(id);  // truly non-blocking
}
```

---

## PART 4 — WEBCLIENT VS RESTTEMPLATE VS FEIGN

---

### Q4. When do you use `WebClient`, `RestTemplate`, and Feign? What's being deprecated?

**The answer:**

"Spring provides three HTTP client abstractions. The landscape is shifting, and knowing the current recommendation matters.

**`RestTemplate` — blocking, synchronous (being deprecated):**
```java
// Traditional Spring MVC apps
RestTemplate restTemplate = new RestTemplateBuilder()
    .rootUri("http://partner-service")
    .setConnectTimeout(Duration.ofMillis(500))
    .setReadTimeout(Duration.ofMillis(2000))
    .build();

PartnerInfo partner = restTemplate.getForObject(
    "/api/v1/partners/{id}", PartnerInfo.class, partnerId
);
```
Spring 6.0 officially deprecated `RestTemplate`. It still works, but new code should use `WebClient`.

**`WebClient` — non-blocking, works in both MVC and WebFlux:**
```java
// The current recommendation — use in both MVC and WebFlux apps
@Bean
public WebClient partnerWebClient() {
    return WebClient.builder()
        .baseUrl("http://partner-service")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .filter(retryFilter())
        .filter(loggingFilter())
        .codecs(configurer ->
            configurer.defaultCodecs().maxInMemorySize(2 * 1024 * 1024))  // 2MB limit
        .build();
}

// In Spring MVC (blocking), call .block() to get the value synchronously:
PartnerInfo partner = webClient.get()
    .uri("/api/v1/partners/{id}", partnerId)
    .retrieve()
    .bodyToMono(PartnerInfo.class)
    .timeout(Duration.ofMillis(2000))
    .block();

// In Spring WebFlux (non-blocking), return the Mono:
Mono<PartnerInfo> partnerMono = webClient.get()
    .uri("/api/v1/partners/{id}", partnerId)
    .retrieve()
    .bodyToMono(PartnerInfo.class);
```

**Adding retry and circuit breaker to WebClient:**
```java
// Resilience4j + WebClient
@Bean
public WebClient resilientWebClient(CircuitBreakerRegistry registry) {
    CircuitBreaker cb = registry.circuitBreaker("partner-service");

    ExchangeFilterFunction retryFilter = (request, next) ->
        next.exchange(request)
            .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                .filter(ex -> ex instanceof WebClientRequestException));

    return WebClient.builder()
        .baseUrl("http://partner-service")
        .filter(retryFilter)
        .filter((req, next) ->
            Mono.fromCallable(() -> cb.executeSupplier(() -> next.exchange(req).block()))
        )
        .build();
}
```

**`Feign` — declarative HTTP client (Spring Cloud OpenFeign):**
```java
// Declare the client as an interface — Spring generates the implementation
@FeignClient(name = "partner-service",
             fallbackFactory = PartnerClientFallbackFactory.class)
public interface PartnerClient {

    @GetMapping("/api/v1/partners/{id}")
    PartnerInfo getPartner(@PathVariable("id") UUID partnerId);

    @PostMapping("/api/v1/partners/{id}/credit/deduct")
    CreditResult deductCredit(@PathVariable("id") UUID partnerId,
                               @RequestBody CreditDeductRequest request);
}

// Fallback factory for circuit breaker
@Component
public class PartnerClientFallbackFactory implements FallbackFactory<PartnerClient> {
    @Override
    public PartnerClient create(Throwable cause) {
        return new PartnerClient() {
            public PartnerInfo getPartner(UUID id) {
                log.error("partner-service unavailable: {}", cause.getMessage());
                return PartnerInfo.unknown();
            }
            public CreditResult deductCredit(UUID id, CreditDeductRequest req) {
                throw new PartnerServiceUnavailableException(cause);
            }
        };
    }
}
```

**When to use what:**

| Scenario | Use |
|----------|-----|
| New Spring MVC code | `WebClient` with `.block()` |
| New Spring WebFlux code | `WebClient` (reactive chain) |
| Legacy code / won't change | `RestTemplate` (works, but no new features) |
| Declarative interface, microservices | Feign (Spring Cloud OpenFeign) |
| Simple in-service HTTP | `RestClient` (Spring 6.1 — new blocking API, cleaner than RestTemplate) |

---

## PART 5 — REACTIVE WITH R2DBC

---

### Q5. What is R2DBC? When would you use it instead of JPA?

**The answer:**

"R2DBC (Reactive Relational Database Connectivity) is a non-blocking, reactive database driver for relational databases. JPA/JDBC are fundamentally blocking — a JDBC `executeQuery()` call blocks the calling thread until the DB responds. This defeats the purpose of WebFlux if you pair it with JDBC.

**JPA / JDBC in WebFlux — the problem:**
```java
// This defeats non-blocking WebFlux entirely
@GetMapping("/payments/{id}")
public Mono<Payment> getPayment(@PathVariable UUID id) {
    return Mono.fromCallable(() -> jpaRepository.findById(id))  // BLOCKS a thread
        .subscribeOn(Schedulers.boundedElastic());  // workaround: offload to thread pool
    // You're back to thread-per-request, just with extra complexity
}
```

**R2DBC — truly non-blocking:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>io.r2dbc</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
</dependency>
```

```java
// Repository — Spring Data R2DBC
@Repository
public interface PaymentR2dbcRepository extends ReactiveCrudRepository<Payment, UUID> {
    Flux<Payment> findByTenantIdAndStatus(UUID tenantId, PaymentStatus status);

    @Query("SELECT * FROM payments WHERE tenant_id = :tenantId ORDER BY created_at DESC LIMIT 20")
    Flux<Payment> findLatestByTenant(@Param("tenantId") UUID tenantId);
}

// Service — fully non-blocking
@Service
public class ReactivePaymentService {

    public Flux<Payment> getRecentPayments(UUID tenantId) {
        return paymentRepository.findLatestByTenant(tenantId)
            .filter(p -> p.getStatus() != PaymentStatus.CANCELLED)
            .map(this::enrichWithDisplayFields);
    }

    public Mono<Payment> createPayment(CreatePaymentRequest request) {
        return paymentRepository.save(new Payment(request))
            .flatMap(payment ->
                kafkaReactiveTemplate.send("payment-events", payment.getId().toString(),
                    new PaymentCreatedEvent(payment))
                .thenReturn(payment)
            );
    }
}
```

**JPA vs R2DBC — decision table:**

| | JPA / JDBC | R2DBC |
|-|-----------|-------|
| Programming model | Imperative, blocking | Reactive, non-blocking |
| Feature richness | Full (lazy loading, L1/L2 cache, JPQL) | Basic (no lazy loading, no L2 cache) |
| Transactions | `@Transactional` | `@Transactional` (reactive) |
| Tooling maturity | Very mature | Newer, less tooling |
| Use with | Spring MVC, batch processing | Spring WebFlux |
| Learning curve | Moderate | High |

**When to use R2DBC:**
- You're already committed to Spring WebFlux
- You have genuinely high I/O concurrency (10K+ concurrent DB connections would exhaust JDBC pool)
- You're building a streaming pipeline where the DB is the source

**When to stay with JPA:**
- Spring MVC apps (no benefit from R2DBC)
- Complex domain model (JPA relationship management, JPQL, criteria API)
- Reporting queries with complex joins
- When the team doesn't have reactive experience

**Senior Signal:** 'The honest answer about reactive in most backend services: unless you have a measured thread exhaustion problem or a genuine streaming use case, Spring MVC + JPA is the right choice. It's simpler, the team understands it, and the tooling (Hibernate, JPQL, `@Transactional`) is far richer. I would choose WebFlux + R2DBC specifically for: an API gateway service (high concurrent fan-out), a real-time event streaming endpoint, or if we were targeting 10K+ concurrent connections per instance.'"

---

## PART 6 — REACTIVE TESTING

---

### Q6. How do you test reactive code?

**The answer:**

```java
// StepVerifier — the standard reactive testing tool
@Test
void shouldReturnPaymentSuccessfully() {
    Payment expected = buildPayment("PAY-001");
    when(repository.findById(any())).thenReturn(Mono.just(expected));

    StepVerifier.create(paymentService.findById(UUID.randomUUID()))
        .expectNext(expected)         // assert each emitted element
        .verifyComplete();            // assert stream completes normally
}

@Test
void shouldEmitMultiplePayments() {
    List<Payment> payments = List.of(buildPayment("P1"), buildPayment("P2"), buildPayment("P3"));
    when(repository.findByTenantId(any())).thenReturn(Flux.fromIterable(payments));

    StepVerifier.create(paymentService.findByTenant(UUID.randomUUID()))
        .expectNextCount(3)
        .verifyComplete();
}

@Test
void shouldPropagateErrorWhenNotFound() {
    when(repository.findById(any())).thenReturn(Mono.empty());

    StepVerifier.create(paymentService.findById(UUID.randomUUID()))
        .expectError(PaymentNotFoundException.class)
        .verify();
}

@Test
void shouldHandleStreamingWithVirtualTime() {
    // Test time-based operators without actually waiting
    StepVerifier.withVirtualTime(() ->
        Flux.interval(Duration.ofMinutes(1)).take(3)
    )
    .expectSubscription()
    .thenAwait(Duration.ofMinutes(3))  // virtual time — no real wait
    .expectNextCount(3)
    .verifyComplete();
}
```

---

## QUICK-REFERENCE — Chapter 8 Senior Filter Questions

**Q: What is the difference between `map` and `flatMap` in reactive?**
`map` is a synchronous transformation: `Mono<A>` → `Mono<B>` where the function `A → B` is synchronous. `flatMap` is for asynchronous transformations: `Mono<A>` → `Mono<B>` where the function `A → Mono<B>` returns another async publisher. If you have a function that itself returns a `Mono`, you must use `flatMap` (otherwise you get `Mono<Mono<B>>`). Same relationship as `map` vs `flatMap` in Java Streams.

**Q: What is `subscribeOn` vs `publishOn`?**
`subscribeOn` changes the thread pool used for the upstream source (where data originates). `publishOn` changes the thread pool for operators below it in the chain. For wrapping blocking calls: `Mono.fromCallable(blockingCall).subscribeOn(Schedulers.boundedElastic())` — the blocking call runs on a dedicated elastic thread pool, not the event loop.

**Q: What is `Schedulers.boundedElastic()` and when do you use it?**
A thread pool designed specifically for wrapping blocking I/O in reactive code. It creates threads on demand up to a bounded limit (default: 10 × CPU cores), queues tasks if all threads are busy, and recycles idle threads after 60 seconds. Use it with `subscribeOn()` when you have no reactive-native alternative (legacy JDBC, file I/O, third-party blocking SDK).

**Q: Can `@Transactional` work with WebFlux?**
Yes, but requires R2DBC (not JDBC). Spring's `@Transactional` annotation works with reactive types when you use `spring-data-r2dbc` — it uses `ReactiveTransactionManager` under the hood. The standard `@Transactional` with JDBC is fundamentally incompatible with reactive pipelines because JDBC blocks.

**Q: What is backpressure and why does it matter?**
Backpressure is the mechanism by which a slow consumer signals an upstream producer to slow down or buffer. Without backpressure, a fast producer overwhelms a slow consumer — unbounded memory growth and eventually OOM. In Project Reactor, backpressure is handled via the `request(n)` protocol: a subscriber tells the publisher how many items it can handle. If a source doesn't support backpressure (e.g., a hot event stream), you handle it with `onBackpressureBuffer`, `onBackpressureDrop`, or `onBackpressureLatest`.

---

> **Guide is complete.** You now have 8 comprehensive chapters covering every topic area a Senior Backend Engineer interview tests — Java Core, Spring Internals, Testing, Concurrency, Databases, Messaging, LLD, HLD, Behavioral, Microservices Infrastructure, and Reactive Programming.

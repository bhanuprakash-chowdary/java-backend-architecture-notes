# Summary — Interview Survival Guide Navigation

Full section-level navigation index for all chapters.

---

## Chapter 1 — Java Core & Spring Internals Deep Dive

### Part 1 — JVM Architecture & Memory Model
- Q1. Walk me through the JVM memory model. Where does each type of data live?
- Q2. Explain Garbage Collection. How does G1GC work and how would you tune it?
- Q3. Explain the String Pool. Why is `new String("abc")` different from `"abc"`?
- Q4. Explain the ClassLoader hierarchy and delegation model.

### Part 2 — Java Type System & Functional Features
- Q5. Explain Java Generics, type erasure, and the wildcard rules.
- Q6. Explain Java Functional Interfaces and how to build a retry utility with `Supplier<T>`.

### Part 3 — Spring Core Internals
- Q7. What happens when a Spring Boot application starts?
- Q8. Explain the Spring Bean lifecycle. What is a `BeanPostProcessor`?
- Q9. Explain Spring AOP. Why does `@Transactional` fail on self-invocation?
- Q10. How does Spring Boot Auto-configuration work?

### Part 4 — Spring Security & JPA
- Q11. Walk me through the Spring Security filter chain.
- Q12. What is the N+1 problem in JPA? How do you diagnose and fix it?
- Q13. Explain `@Transactional` propagation and isolation levels.

### Quick Reference
- Senior Filter Questions — Chapter 1

---

## Chapter 2 — Concurrency, Databases & Messaging

### Part 1 — Java Multithreading & Concurrency
- Q1. `synchronized` vs `ReentrantLock` vs `StampedLock`
- Q2. `volatile`, `AtomicInteger`, and CAS
- Q3. How do you size a `ThreadPoolExecutor`?
- Q4. Explain `CompletableFuture`
- Q5. `CountDownLatch`, `Semaphore`, and `CyclicBarrier`

### Part 2 — The Credit Limit Race Condition
- Q6. The concurrent credit limit race condition — full solution walkthrough

### Part 3 — Kafka Delivery Guarantees
- Q7. Kafka delivery guarantees — at-most-once, at-least-once, exactly-once

### Part 4 — Redis Caching Patterns
- Q8. Core caching strategies — Cache-Aside, Write-Through, Write-Behind
- Q9. Cache stampede prevention

### Part 5 — Database Internals & Locking
- Q10. B-Tree vs B+Tree
- Q11. Index types and strategy
- Q12. Pessimistic vs Optimistic Locking
- Q13. Database sharding strategies

### Quick Reference
- Senior Filter Questions — Chapter 2

---

## Chapter 3 — Low-Level Design & Project Narratives

### Part 1 — SOLID Principles in Production
- Q1. Walk me through SOLID with examples from a real payment system

### Part 2 — Creational Design Patterns
- Q2. Builder Pattern — when and why
- Q3. Factory Method + Abstract Factory
- Q4. Singleton — thread-safe implementations

### Part 3 — Structural Design Patterns
- Q5. Decorator Pattern — layered behavior
- Q6. Chain of Responsibility — validation pipelines
- Q7. Template Method — base processor

### Part 4 — Behavioral Design Patterns
- Q8. State Pattern — loan application state machine
- Q9. Strategy Pattern — lead assignment
- Q10. Observer Pattern — Spring ApplicationEvent
- Q11. Circuit Breaker — manual + Resilience4j

### Part 5 — Full LLD Whiteboard Problems
- Rate Limiter — Token Bucket + Redis Lua distributed version
- Notification System — async delivery, retry, channel factory
- Parking Lot — entities, strategy, observer

### Part 6 — Project Architectural Narratives
- Narrative 1: The 5-Layer Razorpay Idempotent Webhook Architecture
- Narrative 2: ThingsBoard Multi-Tenant Authentication
- Narrative 3: WebSocket Lead Assignment System

### Quick Reference
- Senior Filter Questions — Chapter 3

---

## Chapter 4 — HLD Frameworks, Distributed Systems & Behavioral Mastery

### Part 1 — The HLD Interview Framework
- Universal 5-Step Framework with verbal scripts

### Part 2 — Distributed Systems Fundamentals
- Q1. CAP Theorem — precise definitions and real system mapping
- Q2. ACID vs BASE — microservices implication
- Q3. Eventual Consistency Patterns

### Part 3 — Distributed Design Patterns
- Q4. Two-Phase Commit — protocol and failure modes
- Q5. Saga Pattern — Choreography vs Orchestration
- Q6. Outbox Pattern — solving the dual-write problem
- Q7. CQRS — write model vs read model

### Part 4 — API Design & Service Patterns
- Q8. REST design — idempotency keys, versioning, cursor pagination
- Q9. Observability — correlation IDs, Micrometer metrics

### Part 5 — Full HLD Design
- Multi-Tenant Payment Platform — full architecture with failure analysis

### Part 6 — Behavioral & STAR Method Mastery
- STAR Story 1: Technical disagreement (Outbox Pattern)
- STAR Story 2: Deadline vs correctness (WebSocket feature flag)
- STAR Story 3: Mentoring (race condition mental model)
- STAR Story 4: Production incident (40 duplicate webhook credits)
- STAR Story 5: Ambiguity (Java 17 migration)

### Part 7 — Critical Backend Topics
- Q10. Performance debugging framework
- Q11. Security — JWT, SQL injection, OWASP Top 10
- Q12. API versioning and backward compatibility

### Part 8 — Managerial Round Question Bank

---

## Chapter 5 — HLD Advanced & Full Project Narratives

### Part 1 — Advanced HLD: Twitter-like News Feed
### Part 2 — Advanced HLD: Distributed Cache (Redis Clone)
### Part 3 — Advanced HLD: Real-Time Leaderboard
### Part 4 — Advanced HLD: Distributed Job Scheduler
### Part 5 — Advanced HLD: Search Autocomplete
### Part 6 — Financial Reconciliation System Design
### Part 7 — Resume Bullet Rewriting with STAR Formula

---

## Chapter 1 Additions (Part 5 & 6)
- Java Streams API (filter/map/flatMap/collect, groupingBy, partitioningBy, parallel streams)
- Optional (correct usage, anti-patterns, map/flatMap/orElseGet)
- Spring Testing (MockMvc, @WebMvcTest, @DataJpaTest, TestContainers, @Mock vs @MockBean)

## Chapter 2 Additions (Parts 6–8)
- HikariCP connection pool tuning (sizing formula, exhaustion diagnosis)
- Database migrations with Flyway (versioning, zero-downtime patterns)
- Advanced Kafka (consumer groups, rebalancing, cooperative incremental, hot partitions)
- Dead Letter Queue pattern (implementation, monitoring, reprocessing)
- Redis Sentinel vs Cluster, Pub/Sub vs Streams, RDB vs AOF persistence

## Chapter 4 Additions (Parts 9–10)
- Event Sourcing (append-only log, aggregates, snapshots, vs CQRS)
- gRPC vs REST (proto3, streaming types, when to use each)
- Long polling vs SSE vs WebSocket (full comparison table)
- OAuth2 / OIDC (Authorization Code + PKCE, Client Credentials, JWT vs opaque tokens)
- Spring Boot Actuator (health probes, Prometheus, securing endpoints)
- CORS (what it protects, Spring configuration, security implications)
- Graceful Shutdown (Spring config, K8s SIGTERM flow, preStop hook race condition)

## Chapter 5 Additions (Part 8)
- Deployment strategies (Rolling, Blue-Green, Canary, Feature Flags)
- Zero-downtime deployment requirements checklist

## Chapter 6 — Behavioral Mastery, DSA Patterns & Mock Interview Simulation

### Part 1 — DSA Coding Screen Patterns
- Two Pointers — Container With Most Water, 3Sum
- Sliding Window — Longest Substring, Minimum Window Substring
- Binary Search — Rotated Array, Binary Search on Answer
- Trees — Validate BST, LCA, Maximum Path Sum
- Graphs — Number of Islands, Topological Sort, Union-Find
- Dynamic Programming — Kadane's, LCS, Knapsack, Coin Change
- Heap / Priority Queue — Top K, K-th Largest, Merge K Lists

### Part 2 — Additional STAR Stories
- Story 6: Handling Ambiguous Requirements (real-time definition conflict)
- Story 7: Proactively Preventing a Production Outage (bulk API thread exhaustion)
- Story 8: Delivering Under Pressure (48-hour tenant isolation fix before demo)
- Story 9: Disagreeing with a Senior Decision (microservices vs modular monolith)

### Part 3 — Company-Specific Research Framework
- 5-step research protocol (Engineering blog, LinkedIn, Glassdoor, Product, Questions)
- Company Research Template (fill-in)
- Questions to ask per round type

### Part 4 — Full Mock Interview Simulation
- Round 1: Coding Screen (fraud detection sliding window problem)
- Round 2: System Design (real-time fraud detection system)
- Round 3: Java Deep Dive (verbatim Q&A)
- Round 4: Behavioral / Managerial (failure, staying current, hardest concept)

### Part 5 — The 30-Day Interview Sprint Plan
- Week-by-week schedule with daily activities
- Day-before checklist
- During-interview mental anchors

### Quick Reference — Final Cheat Sheet
- Java Senior Filter answers
- Spring Senior Filter answers
- System Design Senior Filter answers

---

## Chapter 7 — Microservices Infrastructure

### Part 1 — Service Discovery (Eureka, Kubernetes Service)
### Part 2 — API Gateway (Spring Cloud Gateway, filters, rate limiting)
### Part 3 — Config Management (Spring Cloud Config, secrets, dynamic refresh)
### Part 4 — Distributed Tracing (OpenTelemetry, spans, trace propagation, Jaeger)
### Part 5 — Docker (layer cache, multi-stage builds, non-root user, .dockerignore)
### Part 6 — Kubernetes (Pod, Deployment, Service, Ingress, ConfigMap, HPA, kubectl)
### Part 7 — Inter-Service Communication (sync vs async, resilience patterns, dependency chain)

---

## Chapter 8 — Spring WebFlux & Reactive Programming

### Part 1 — Why Reactive? Thread-per-request problem
### Part 2 — Project Reactor: Mono, Flux, core operators
### Part 3 — Spring WebFlux controllers, reactive service layer, blocking code trap
### Part 4 — WebClient vs RestTemplate vs Feign (deprecation landscape)
### Part 5 — R2DBC: non-blocking DB access, JPA vs R2DBC decision
### Part 6 — Reactive testing with StepVerifier

---

## Chapter 9 — Remaining Spring Essentials & Java 17

### Part 1 — Global Exception Handling with `@ControllerAdvice`
- `@RestControllerAdvice` vs `@ControllerAdvice`
- `ResponseEntityExceptionHandler` — inherited framework exception mappings
- Structured `ErrorResponse` record with traceId
- Custom business exception hierarchy (AppException, ResourceNotFoundException, BusinessException)
- Handling `MethodArgumentNotValidException` with field-level error list

### Part 2 — Bean Validation
- Built-in constraints (@NotNull, @NotBlank, @Positive, @Size, @Email, @Pattern)
- `@Valid` on `@RequestBody` — triggers `MethodArgumentNotValidException`
- Custom `ConstraintValidator<A, T>` — class-level cross-field rules
- `@Validated` vs `@Valid` — method-level validation on `@Service` beans
- `ConstraintViolationException` handling in `@ExceptionHandler`
- Validation Groups (OnCreate, OnUpdate) with `@Validated(Group.class)`

### Part 3 — Spring Profiles
- `application-{profile}.properties` resolution order
- `@Profile` on `@Configuration` beans — environment-specific wiring
- Activating profiles: properties / JVM system property / K8s env var
- `@ActiveProfiles` in tests
- `@ConditionalOnProperty` for feature flag–driven bean creation

### Part 4 — Java 17 New Features
- Records: immutable data carriers, compact canonical constructor, Jackson integration
- Sealed classes: exhaustive subtype hierarchies, compiler-enforced switch completeness
- Pattern matching `instanceof`: bind variable, guard conditions
- Text blocks: multi-line strings, SQL templates, JSON templates
- Switch expressions: arrow syntax, `yield`, exhaustive enum switches
- Helpful NullPointerExceptions (Java 14+)

### Part 5 — `@Async` Deep Dive
- `@EnableAsync` setup, valid return types (void / Future / CompletableFuture)
- The proxy trap — self-invocation breaks `@Async` (same as `@Transactional`)
- `AsyncUncaughtExceptionHandler` — handling exceptions from void async methods
- Custom `ThreadPoolTaskExecutor` — never use default `SimpleAsyncTaskExecutor`
- Multiple named executors for I/O vs CPU workloads
- `@Async` + `@Transactional` interaction — new thread = new transaction context
- `@TransactionalEventListener(phase = AFTER_COMMIT)` — post-commit async pattern

### Quick Reference
- Senior Filter Questions — Chapter 9

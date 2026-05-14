# Interview Survival Guide
## Chapter 3: Low-Level Design & Project Narratives
### (Weeks 9–12 & Week 20)

---

## PART 1 — SOLID PRINCIPLES IN PRODUCTION

---

### Q1. Walk me through SOLID with concrete examples from a payment system.

**The answer:**

"SOLID isn't theory for me — I've applied it directly to our payment codebase. Let me walk through each principle with a before/after from real code.

**S — Single Responsibility Principle:**
A class should have one reason to change.

```java
// BEFORE — God class doing everything
public class PaymentService {
    public void processPayment(Payment payment) { /* validate + process */ }
    public void sendEmail(Payment payment) { /* email logic */ }
    public void saveToDatabase(Payment payment) { /* persistence */ }
    public void calculateFee(Payment payment) { /* fee logic */ }
    public String generateReport(List<Payment> payments) { /* reporting */ }
}
// Reason to change: business logic changes, email templates change, DB schema changes,
// fee calculation rules change, report format changes — ALL change this one class.

// AFTER — each class has one job
public class PaymentProcessor { /* validation + processing only */ }
public class PaymentNotificationService { /* email/SMS notifications */ }
public class PaymentRepository { /* persistence */ }
public class FeeCalculator { /* fee strategy */ }
public class PaymentReportingService { /* reporting */ }
```

**O — Open/Closed Principle:**
Open for extension, closed for modification.

```java
// BEFORE — every new gateway requires changing existing code
public class PaymentGatewayService {
    public void process(Payment payment, String gatewayType) {
        if ("RAZORPAY".equals(gatewayType)) { /* razorpay logic */ }
        else if ("STRIPE".equals(gatewayType)) { /* stripe logic */ }
        else if ("PAYTM".equals(gatewayType)) { /* paytm logic */ }
        // Adding PayU requires modifying this class
    }
}

// AFTER — add new gateways without touching existing code
public interface PaymentGateway {
    PaymentResult process(Payment payment);
    String getGatewayType();
}

@Component
public class RazorpayGateway implements PaymentGateway {
    public PaymentResult process(Payment payment) { /* Razorpay API */ }
    public String getGatewayType() { return "RAZORPAY"; }
}

@Component
public class StripeGateway implements PaymentGateway {
    public PaymentResult process(Payment payment) { /* Stripe API */ }
    public String getGatewayType() { return "STRIPE"; }
}

// Factory wires it up — adding PayU = add one new class, zero changes elsewhere
@Service
public class PaymentGatewayFactory {
    private final Map<String, PaymentGateway> gateways;
    
    public PaymentGatewayFactory(List<PaymentGateway> all) {
        this.gateways = all.stream()
            .collect(Collectors.toMap(PaymentGateway::getGatewayType, g -> g));
    }
    
    public PaymentGateway getGateway(String type) {
        return Optional.ofNullable(gateways.get(type))
            .orElseThrow(() -> new UnsupportedGatewayException(type));
    }
}
```

**L — Liskov Substitution Principle:**
Subtypes must be substitutable for their base type without breaking the program.

```java
// VIOLATION
public interface NotificationSender {
    void send(Notification notification);
    List<DeliveryStatus> getDeliveryHistory();  // not all channels support history
}

public class SmsNotificationSender implements NotificationSender {
    public void send(Notification n) { /* SMS */ }
    public List<DeliveryStatus> getDeliveryHistory() {
        throw new UnsupportedOperationException();  // SMS doesn't track history!
    }
}
// Code that calls getDeliveryHistory() expecting it to work is broken with SMS sender.

// FIX — segregate the interface
public interface NotificationSender {
    void send(Notification notification);
}
public interface TrackableNotificationSender extends NotificationSender {
    List<DeliveryStatus> getDeliveryHistory();
}

public class EmailNotificationSender implements TrackableNotificationSender {
    public void send(Notification n) { /* email */ }
    public List<DeliveryStatus> getDeliveryHistory() { /* email history */ }
}

public class SmsNotificationSender implements NotificationSender {
    public void send(Notification n) { /* SMS — no history */ }
    // No getDeliveryHistory() — no LSP violation
}
```

**I — Interface Segregation Principle:**
Clients should not be forced to depend on interfaces they don't use.

```java
// FAT INTERFACE — forces all implementors to handle everything
public interface ReportGenerator {
    byte[] generatePdf();
    byte[] generateExcel();
    String generateCsv();
    void emailReport(String recipient);
    void scheduleReport(Schedule schedule);
}

// SEGREGATED INTERFACES
public interface PdfReportGenerator { byte[] generatePdf(); }
public interface ExcelReportGenerator { byte[] generateExcel(); }
public interface ReportEmailer { void emailReport(String recipient); }
public interface ReportScheduler { void scheduleReport(Schedule schedule); }

// Classes implement only what they support
public class PaymentSummaryReporter implements PdfReportGenerator, ExcelReportGenerator {
    public byte[] generatePdf() { /* PDF logic */ }
    public byte[] generateExcel() { /* Excel logic */ }
}
```

**D — Dependency Inversion Principle:**
High-level modules should not depend on low-level modules. Both should depend on abstractions.

```java
// VIOLATION — high-level service directly instantiates low-level
public class PaymentProcessor {
    private final MySqlPaymentRepository repository = new MySqlPaymentRepository();  // coupled to MySQL
    // If we switch to PostgreSQL, we change PaymentProcessor — wrong!
}

// CORRECT — depend on abstraction, inject the implementation
public class PaymentProcessor {
    private final PaymentRepository repository;  // interface

    public PaymentProcessor(PaymentRepository repository) {  // constructor injection
        this.repository = repository;
    }
}
// Spring injects MySqlPaymentRepository at runtime
// Tests inject MockPaymentRepository
// Switch to Postgres = new class implements interface, no change to PaymentProcessor
```

---

## PART 2 — CREATIONAL DESIGN PATTERNS

---

### Q2. Builder Pattern — when and why?

**The answer:**

"Use Builder when: (1) an object has many optional parameters, (2) some parameters are mutually exclusive, or (3) you need to validate cross-field invariants at construction time.

```java
public class WebhookProcessingRequest {
    // Required
    private final String eventId;
    private final String eventType;
    private final String paymentReference;
    
    // Optional
    private final int maxRetries;
    private final Duration timeout;
    private final boolean idempotencyEnabled;
    private final String callbackUrl;

    private WebhookProcessingRequest(Builder builder) {
        this.eventId = builder.eventId;
        this.eventType = builder.eventType;
        this.paymentReference = builder.paymentReference;
        this.maxRetries = builder.maxRetries;
        this.timeout = builder.timeout;
        this.idempotencyEnabled = builder.idempotencyEnabled;
        this.callbackUrl = builder.callbackUrl;
    }

    public static class Builder {
        // Required — set in constructor
        private final String eventId;
        private final String eventType;
        private final String paymentReference;
        
        // Optional — defaults
        private int maxRetries = 3;
        private Duration timeout = Duration.ofSeconds(30);
        private boolean idempotencyEnabled = true;
        private String callbackUrl = null;

        public Builder(String eventId, String eventType, String paymentReference) {
            this.eventId = Objects.requireNonNull(eventId);
            this.eventType = Objects.requireNonNull(eventType);
            this.paymentReference = Objects.requireNonNull(paymentReference);
        }

        public Builder maxRetries(int maxRetries) {
            if (maxRetries < 1 || maxRetries > 10) throw new IllegalArgumentException("maxRetries must be 1-10");
            this.maxRetries = maxRetries;
            return this;
        }

        public Builder timeout(Duration timeout) {
            this.timeout = Objects.requireNonNull(timeout);
            return this;
        }

        public Builder disableIdempotency() {
            this.idempotencyEnabled = false;
            return this;
        }

        public Builder callbackUrl(String url) {
            this.callbackUrl = url;
            return this;
        }

        public WebhookProcessingRequest build() {
            // Cross-field validation
            if (!idempotencyEnabled && callbackUrl != null) {
                throw new IllegalStateException("callbackUrl requires idempotency to be enabled");
            }
            return new WebhookProcessingRequest(this);
        }
    }
}

// Usage
WebhookProcessingRequest request = new WebhookProcessingRequest.Builder(
    "evt_123", "payment.captured", "pay_456"
)
.maxRetries(5)
.timeout(Duration.ofSeconds(10))
.callbackUrl("https://api.myapp.com/webhook-processed")
.build();
```

---

### Q3. Factory Method + Abstract Factory — how are they different?

**The answer:**

"**Factory Method:** Defines an interface for creating an object, but lets subclasses decide which class to instantiate. One factory creates one product.

```java
// Factory Method — each subclass creates its own product type
public abstract class NotificationFactory {
    public abstract NotificationSender createSender();  // factory method

    public void sendNotification(Notification notification) {
        NotificationSender sender = createSender();  // uses the factory method
        sender.send(notification);
    }
}

public class EmailNotificationFactory extends NotificationFactory {
    @Override
    public NotificationSender createSender() {
        return new EmailNotificationSender(emailConfig);
    }
}

public class SmsNotificationFactory extends NotificationFactory {
    @Override
    public NotificationSender createSender() {
        return new SmsNotificationSender(smsConfig);
    }
}
```

**Abstract Factory:** Creates families of related objects. One factory creates multiple related products that must be used together.

```java
// Abstract Factory — creates a family of related objects
public interface PaymentGatewayFactory {
    PaymentProcessor createProcessor();
    WebhookValidator createWebhookValidator();
    RefundHandler createRefundHandler();
}

// Razorpay family — all Razorpay-specific implementations
public class RazorpayGatewayFactory implements PaymentGatewayFactory {
    public PaymentProcessor createProcessor() { return new RazorpayProcessor(); }
    public WebhookValidator createWebhookValidator() { return new RazorpayWebhookValidator(); }
    public RefundHandler createRefundHandler() { return new RazorpayRefundHandler(); }
}

// Stripe family — all Stripe-specific implementations
public class StripeGatewayFactory implements PaymentGatewayFactory {
    public PaymentProcessor createProcessor() { return new StripeProcessor(); }
    public WebhookValidator createWebhookValidator() { return new StripeWebhookValidator(); }
    public RefundHandler createRefundHandler() { return new StripeRefundHandler(); }
}

// Usage — entire family is consistent
PaymentGatewayFactory factory = new RazorpayGatewayFactory();
PaymentProcessor processor = factory.createProcessor();
WebhookValidator validator = factory.createWebhookValidator();
// All three are Razorpay — no mixing of Razorpay processor with Stripe validator
```

---

### Q4. Singleton — thread-safe implementations. Which one is best?

**The answer:**

"There are three idioms. Only two are correct in multithreaded Java.

**Double-Checked Locking (correct with `volatile`):**
```java
public class DatabaseConnectionPool {
    private static volatile DatabaseConnectionPool instance;  // volatile is REQUIRED

    private DatabaseConnectionPool() {
        // initialize pool
    }

    public static DatabaseConnectionPool getInstance() {
        if (instance == null) {                    // first check — no sync
            synchronized (DatabaseConnectionPool.class) {
                if (instance == null) {            // second check — with sync
                    instance = new DatabaseConnectionPool();
                }
            }
        }
        return instance;
    }
}
// volatile prevents CPU instruction reordering during construction.
// Without volatile, another thread might see a partially-constructed object.
```

**Initialization-on-Demand Holder (the cleanest, recommended):**
```java
public class DatabaseConnectionPool {
    private DatabaseConnectionPool() { }

    private static class Holder {
        // Inner class not loaded until getInstance() is called
        // JVM guarantees class initialization is thread-safe
        static final DatabaseConnectionPool INSTANCE = new DatabaseConnectionPool();
    }

    public static DatabaseConnectionPool getInstance() {
        return Holder.INSTANCE;
    }
}
// No synchronization needed — class loading is inherently thread-safe.
// Lazy initialization — Holder class not loaded until first call to getInstance().
// This is the preferred idiom.
```

**Enum Singleton (bulletproof against serialization and reflection):**
```java
public enum ConfigurationManager {
    INSTANCE;

    private final Map<String, String> config = new HashMap<>();

    public String get(String key) { return config.get(key); }
    public void set(String key, String value) { config.put(key, value); }
}
// Java enum guarantees: only one instance per JVM, safe from serialization,
// safe from reflection. Use when the singleton semantics need to be absolute.
```

**Senior Signal:** "The initialization-on-demand holder is the standard answer. The enum approach is the most robust but feels unnatural for non-enum semantics. The key insight about double-checked locking is that `volatile` is required — without it, the Java memory model permits the JIT to publish a reference to a partially-constructed object."

---

## PART 3 — STRUCTURAL DESIGN PATTERNS

---

### Q5. Decorator Pattern — how did you use it in the payment gateway?

**The answer:**

"Decorator adds behavior to an object at runtime by wrapping it, without changing its interface and without subclassing.

In our payment gateway, the core gateway interface is `PaymentGateway`. We needed to add logging, retry logic, and metrics — three concerns orthogonal to the business logic. Instead of subclassing (which causes a class explosion: LoggingRazorpay, RetryingRazorpay, MetricsRazorpay, LoggingRetryingMetricsRazorpay...), we used decorator:

```java
public interface PaymentGateway {
    PaymentResult process(Payment payment);
    RefundResult refund(String paymentId, BigDecimal amount);
}

// Core implementation — pure business logic, no cross-cutting concerns
public class RazorpayGateway implements PaymentGateway {
    public PaymentResult process(Payment payment) { /* Razorpay API call */ }
    public RefundResult refund(String paymentId, BigDecimal amount) { /* Razorpay refund */ }
}

// Logging decorator
public class LoggingPaymentGateway implements PaymentGateway {
    private final PaymentGateway delegate;

    public LoggingPaymentGateway(PaymentGateway delegate) {
        this.delegate = delegate;
    }

    public PaymentResult process(Payment payment) {
        log.info("Processing payment {} via {}", payment.getId(), delegate.getClass().getSimpleName());
        long start = System.currentTimeMillis();
        try {
            PaymentResult result = delegate.process(payment);
            log.info("Payment {} processed in {}ms: {}", payment.getId(),
                     System.currentTimeMillis() - start, result.getStatus());
            return result;
        } catch (Exception e) {
            log.error("Payment {} failed after {}ms: {}", payment.getId(),
                      System.currentTimeMillis() - start, e.getMessage());
            throw e;
        }
    }

    public RefundResult refund(String paymentId, BigDecimal amount) {
        log.info("Initiating refund for payment {}", paymentId);
        return delegate.refund(paymentId, amount);
    }
}

// Retry decorator
public class RetryingPaymentGateway implements PaymentGateway {
    private final PaymentGateway delegate;
    private final int maxRetries;

    public PaymentResult process(Payment payment) {
        return RetryExecutor.executeWithRetry(
            () -> delegate.process(payment),
            maxRetries, 500L,
            TransientGatewayException.class
        );
    }

    public RefundResult refund(String paymentId, BigDecimal amount) {
        return delegate.refund(paymentId, amount);  // refunds are not retried
    }
}

// Metrics decorator
public class MetricsPaymentGateway implements PaymentGateway {
    private final PaymentGateway delegate;
    private final MeterRegistry registry;

    public PaymentResult process(Payment payment) {
        return Timer.builder("payment.gateway.process")
            .tag("gateway", delegate.getClass().getSimpleName())
            .register(registry)
            .record(() -> delegate.process(payment));
    }
}

// Composition — wrap in desired order
PaymentGateway gateway = new MetricsPaymentGateway(
    new LoggingPaymentGateway(
        new RetryingPaymentGateway(
            new RazorpayGateway(config),
            3  // retries
        )
    ),
    meterRegistry
);
// Call order: Metrics → Logging → Retry → RazorpayGateway
```

---

### Q6. Chain of Responsibility — validation pipeline.

**The answer:**

"Chain of Responsibility passes a request through a chain of handlers. Each handler either processes the request or passes it to the next handler.

```java
public abstract class PaymentValidator {
    private PaymentValidator next;

    public PaymentValidator setNext(PaymentValidator next) {
        this.next = next;
        return next;
    }

    public final ValidationResult validate(Payment payment) {
        ValidationResult result = doValidate(payment);
        if (!result.isValid()) return result;  // fail fast
        if (next != null) return next.validate(payment);
        return ValidationResult.valid();
    }

    protected abstract ValidationResult doValidate(Payment payment);
}

public class AmountValidator extends PaymentValidator {
    protected ValidationResult doValidate(Payment payment) {
        if (payment.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            return ValidationResult.invalid("Amount must be positive");
        }
        if (payment.getAmount().compareTo(new BigDecimal("1000000")) > 0) {
            return ValidationResult.invalid("Amount exceeds maximum allowed");
        }
        return ValidationResult.valid();
    }
}

public class BeneficiaryValidator extends PaymentValidator {
    protected ValidationResult doValidate(Payment payment) {
        if (!beneficiaryService.exists(payment.getBeneficiaryId())) {
            return ValidationResult.invalid("Beneficiary not found");
        }
        return ValidationResult.valid();
    }
}

public class FraudValidator extends PaymentValidator {
    protected ValidationResult doValidate(Payment payment) {
        FraudScore score = fraudService.score(payment);
        if (score.getRisk() > 0.8) {
            return ValidationResult.invalid("High fraud risk: " + score.getReason());
        }
        return ValidationResult.valid();
    }
}

// Wire the chain
PaymentValidator chain = new AmountValidator();
chain.setNext(new BeneficiaryValidator())
     .setNext(new FraudValidator());
     // .setNext(new KycValidator()) — add more without changing existing

ValidationResult result = chain.validate(payment);
```

---

### Q7. Template Method — base webhook processor.

**The answer:**

"Template Method defines the skeleton of an algorithm in a base class, deferring specific steps to subclasses.

```java
public abstract class BaseWebhookProcessor {

    // Template method — defines the algorithm skeleton
    public final void process(WebhookPayload payload) {
        validateSignature(payload);     // Step 1 — common to all
        WebhookEvent event = parse(payload);  // Step 2 — provider-specific
        deduplicate(event);             // Step 3 — common to all
        handleEvent(event);             // Step 4 — provider-specific
        acknowledge(payload);           // Step 5 — common to all
    }

    // Common steps — implemented in base class
    private void validateSignature(WebhookPayload payload) {
        if (!getSignatureValidator().validate(payload)) {
            throw new InvalidSignatureException(payload.getSource());
        }
    }

    private void deduplicate(WebhookEvent event) {
        String key = "webhook:" + event.getId();
        if (!redisTemplate.opsForValue().setIfAbsent(key, "1", Duration.ofHours(24))) {
            throw new DuplicateEventException(event.getId());
        }
    }

    private void acknowledge(WebhookPayload payload) {
        log.info("Webhook {} processed successfully", payload.getEventId());
    }

    // Provider-specific steps — must be implemented by subclasses
    protected abstract SignatureValidator getSignatureValidator();
    protected abstract WebhookEvent parse(WebhookPayload payload);
    protected abstract void handleEvent(WebhookEvent event);
}

// Razorpay-specific implementation
public class RazorpayWebhookProcessor extends BaseWebhookProcessor {

    @Override
    protected SignatureValidator getSignatureValidator() {
        return new RazorpaySignatureValidator(razorpaySecret);
    }

    @Override
    protected WebhookEvent parse(WebhookPayload payload) {
        RazorpayEvent razorpayEvent = objectMapper.readValue(payload.getBody(), RazorpayEvent.class);
        return razorpayEventMapper.toWebhookEvent(razorpayEvent);
    }

    @Override
    protected void handleEvent(WebhookEvent event) {
        switch (event.getType()) {
            case "payment.captured" -> paymentService.capture(event);
            case "payment.failed"   -> paymentService.markFailed(event);
            case "refund.processed" -> refundService.complete(event);
            default -> log.warn("Unhandled Razorpay event type: {}", event.getType());
        }
    }
}
```

---

### Q8. State Pattern — loan application state machine.

**The answer:**

"State Pattern allows an object to change its behavior when its internal state changes. The object will appear to change its class.

```java
public interface LoanApplicationState {
    void submit(LoanApplication application);
    void approve(LoanApplication application);
    void reject(LoanApplication application);
    void disburse(LoanApplication application);
    String getStateName();
}

public class DraftState implements LoanApplicationState {
    public void submit(LoanApplication app) {
        app.setState(new PendingReviewState());
        app.setSubmittedAt(Instant.now());
        eventPublisher.publishEvent(new LoanSubmittedEvent(app));
    }
    public void approve(LoanApplication app) { throw new InvalidStateTransitionException("Cannot approve a draft"); }
    public void reject(LoanApplication app) { throw new InvalidStateTransitionException("Cannot reject a draft"); }
    public void disburse(LoanApplication app) { throw new InvalidStateTransitionException("Cannot disburse a draft"); }
    public String getStateName() { return "DRAFT"; }
}

public class PendingReviewState implements LoanApplicationState {
    public void submit(LoanApplication app) { throw new InvalidStateTransitionException("Already submitted"); }
    public void approve(LoanApplication app) {
        app.setState(new ApprovedState());
        app.setApprovedAt(Instant.now());
        notificationService.notifyApproval(app);
    }
    public void reject(LoanApplication app) {
        app.setState(new RejectedState());
        app.setRejectedAt(Instant.now());
        notificationService.notifyRejection(app);
    }
    public void disburse(LoanApplication app) { throw new InvalidStateTransitionException("Not approved yet"); }
    public String getStateName() { return "PENDING_REVIEW"; }
}

public class ApprovedState implements LoanApplicationState {
    public void disburse(LoanApplication app) {
        app.setState(new DisbursedState());
        disbursementService.initiate(app);
    }
    // other transitions throw InvalidStateTransitionException
    public String getStateName() { return "APPROVED"; }
}

// LoanApplication delegates behavior to current state
public class LoanApplication {
    private LoanApplicationState state = new DraftState();
    
    public void submit() { state.submit(this); }
    public void approve() { state.approve(this); }
    public void reject() { state.reject(this); }
    public void disburse() { state.disburse(this); }
    
    void setState(LoanApplicationState state) { this.state = state; }
    public String getStatus() { return state.getStateName(); }
}
```

---

### Q9. Circuit Breaker — manual implementation + Resilience4j.

**The answer:**

"Circuit Breaker prevents cascading failures when a downstream service is unhealthy. It has three states: CLOSED (normal), OPEN (fast-fail), HALF_OPEN (test recovery).

```java
// Manual implementation to show the concept
public class CircuitBreaker {
    private enum State { CLOSED, OPEN, HALF_OPEN }
    
    private volatile State state = State.CLOSED;
    private final AtomicInteger failureCount = new AtomicInteger(0);
    private volatile Instant openedAt;
    
    private final int failureThreshold;
    private final Duration waitDuration;

    public <T> T execute(Supplier<T> operation) {
        if (state == State.OPEN) {
            if (Duration.between(openedAt, Instant.now()).compareTo(waitDuration) > 0) {
                state = State.HALF_OPEN;  // try recovery
            } else {
                throw new CircuitOpenException("Circuit breaker is OPEN");
            }
        }
        
        try {
            T result = operation.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            throw e;
        }
    }

    private synchronized void onSuccess() {
        failureCount.set(0);
        state = State.CLOSED;
    }

    private synchronized void onFailure() {
        int failures = failureCount.incrementAndGet();
        if (failures >= failureThreshold) {
            state = State.OPEN;
            openedAt = Instant.now();
        }
    }
}

// Resilience4j (production-ready)
@Bean
public CircuitBreaker razorpayCircuitBreaker(CircuitBreakerRegistry registry) {
    CircuitBreakerConfig config = CircuitBreakerConfig.custom()
        .failureRateThreshold(50)              // open when 50% of calls fail
        .waitDurationInOpenState(Duration.ofSeconds(30))
        .permittedNumberOfCallsInHalfOpenState(3)  // 3 test calls in half-open
        .slidingWindowSize(10)                 // evaluate last 10 calls
        .recordExceptions(IOException.class, TimeoutException.class)
        .build();
    return registry.circuitBreaker("razorpay", config);
}

@Service
public class RazorpayService {
    private final CircuitBreaker circuitBreaker;

    public PaymentResult processPayment(Payment payment) {
        return circuitBreaker.executeSupplier(() -> razorpayClient.process(payment));
    }
}
```

---

## PART 4 — BEHAVIORAL DESIGN PATTERNS

---

### Q10. Strategy Pattern — lead assignment.

**The answer:**

"Strategy defines a family of algorithms, encapsulates each, and makes them interchangeable.

```java
public interface LeadAssignmentStrategy {
    Partner assignLead(Lead lead, List<Partner> availablePartners);
    String getStrategyName();
}

// Round-robin assignment
@Component
public class RoundRobinStrategy implements LeadAssignmentStrategy {
    private final AtomicInteger counter = new AtomicInteger(0);

    public Partner assignLead(Lead lead, List<Partner> partners) {
        int index = counter.getAndIncrement() % partners.size();
        return partners.get(index);
    }

    public String getStrategyName() { return "ROUND_ROBIN"; }
}

// Score-based assignment (highest credit score gets the lead)
@Component
public class ScoreBasedStrategy implements LeadAssignmentStrategy {
    public Partner assignLead(Lead lead, List<Partner> partners) {
        return partners.stream()
            .filter(p -> p.getAvailableCredit().compareTo(lead.getCreditRequired()) >= 0)
            .max(Comparator.comparing(Partner::getCreditScore))
            .orElseThrow(() -> new NoAvailablePartnerException(lead.getId()));
    }

    public String getStrategyName() { return "SCORE_BASED"; }
}

// Geographic assignment
@Component  
public class GeographicStrategy implements LeadAssignmentStrategy {
    public Partner assignLead(Lead lead, List<Partner> partners) {
        return partners.stream()
            .filter(p -> p.getServiceableStates().contains(lead.getState()))
            .min(Comparator.comparing(p -> distanceTo(p, lead.getLocation())))
            .orElseThrow(() -> new NoAvailablePartnerException(lead.getId()));
    }

    public String getStrategyName() { return "GEOGRAPHIC"; }
}

// Context — uses the strategy
@Service
public class LeadAssignmentService {
    private final Map<String, LeadAssignmentStrategy> strategies;
    
    public LeadAssignmentService(List<LeadAssignmentStrategy> all) {
        this.strategies = all.stream()
            .collect(Collectors.toMap(LeadAssignmentStrategy::getStrategyName, s -> s));
    }

    public Assignment assignLead(Lead lead) {
        String strategyName = tenantConfigService.getAssignmentStrategy(lead.getTenantId());
        LeadAssignmentStrategy strategy = strategies.get(strategyName);
        List<Partner> available = partnerRepository.findAvailableForLead(lead);
        Partner partner = strategy.assignLead(lead, available);
        return assignmentRepository.save(new Assignment(lead, partner));
    }
}
```

---

### Q11. Observer Pattern — Spring ApplicationEvent.

**The answer:**

"Observer defines a one-to-many dependency — when one object changes state, all its dependents are notified automatically.

Spring's ApplicationEvent system is an Observer implementation built into the framework.

```java
// Event (the notification payload)
public class PaymentCompletedEvent extends ApplicationEvent {
    private final Payment payment;
    private final String tenantId;

    public PaymentCompletedEvent(Object source, Payment payment, String tenantId) {
        super(source);
        this.payment = payment;
        this.tenantId = tenantId;
    }
    // getters
}

// Publisher (the subject)
@Service
public class PaymentService {
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public void completePayment(String paymentId) {
        Payment payment = paymentRepository.findById(paymentId).get();
        payment.setStatus(PaymentStatus.COMPLETED);
        paymentRepository.save(payment);
        
        // Publish event — observers notified after TX commits
        eventPublisher.publishEvent(
            new PaymentCompletedEvent(this, payment, payment.getTenantId())
        );
    }
}

// Observers (listeners)
@Component
public class PaymentNotificationListener {
    @EventListener
    @Async  // runs in separate thread — non-blocking
    public void onPaymentCompleted(PaymentCompletedEvent event) {
        notificationService.sendPaymentConfirmation(event.getPayment());
    }
}

@Component
public class PartnerCreditReleaseListener {
    @EventListener
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    // Only fires after the publishing transaction commits — prevents premature credit release
    public void onPaymentCompleted(PaymentCompletedEvent event) {
        creditService.releaseCredit(event.getPayment().getPartnerId(),
                                    event.getPayment().getAmount());
    }
}

@Component
public class AuditListener {
    @EventListener
    public void onPaymentCompleted(PaymentCompletedEvent event) {
        auditService.record(AuditEntry.paymentCompleted(event.getPayment()));
    }
}
```

**Senior Signal:** "`@TransactionalEventListener` with `AFTER_COMMIT` is the critical detail. If you use `@EventListener` inside a `@Transactional` method, the event fires before the transaction commits. If the transaction rolls back, you've already sent a payment confirmation email — bad. `@TransactionalEventListener(AFTER_COMMIT)` only fires after the transaction successfully commits."

---

## PART 5 — FULL LLD WHITEBOARD PROBLEMS

---

### Rate Limiter — Token Bucket Algorithm

**System Requirement:** Limit API requests to N requests per second per tenant.

**Algorithm Selection:**
- Token Bucket: burst allowed (bucket can fill up), smooth average rate. Best for public APIs.
- Fixed Window Counter: simple, but burst at window boundaries. Avoid.
- Sliding Window Log: exact, memory-intensive (log every request). Use for strict SLAs.
- Leaky Bucket: constant output rate, rejects bursts. Good for protecting downstream.

**Token Bucket algorithm:**
- Bucket holds at most `capacity` tokens.
- Tokens added at rate `refillRate` tokens/second.
- Each request consumes 1 token.
- If no tokens available — request rejected (429 Too Many Requests).

```java
// In-memory token bucket (single node)
public class TokenBucket {
    private final int capacity;
    private final double refillRatePerMs;  // tokens per millisecond
    private double tokens;
    private long lastRefillTimestamp;

    public TokenBucket(int capacity, int refillRatePerSecond) {
        this.capacity = capacity;
        this.refillRatePerMs = refillRatePerSecond / 1000.0;
        this.tokens = capacity;
        this.lastRefillTimestamp = System.currentTimeMillis();
    }

    public synchronized boolean tryConsume() {
        refill();
        if (tokens >= 1) {
            tokens--;
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.currentTimeMillis();
        double tokensToAdd = (now - lastRefillTimestamp) * refillRatePerMs;
        tokens = Math.min(capacity, tokens + tokensToAdd);
        lastRefillTimestamp = now;
    }
}
```

**Distributed Redis Lua rate limiter:**
```lua
-- KEYS[1] = rate limit key (e.g., "rl:{tenantId}:{endpoint}")
-- ARGV[1] = capacity, ARGV[2] = refill rate/sec, ARGV[3] = current timestamp (ms)
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2]) / 1000  -- per ms
local now = tonumber(ARGV[3])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Refill
local elapsed = now - last_refill
tokens = math.min(capacity, tokens + elapsed * refill_rate)

if tokens >= 1 then
    tokens = tokens - 1
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, 3600)
    return 1  -- allowed
else
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    return 0  -- rate limited
end
```

```java
// Spring filter integration
@Component
@Order(1)
public class RateLimitingFilter extends OncePerRequestFilter {
    private final RedisScript<Long> rateLimitScript;
    private final RedisTemplate<String, String> redisTemplate;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse resp,
                                    FilterChain chain) throws ServletException, IOException {
        String tenantId = extractTenantId(req);
        String key = "rl:" + tenantId + ":" + req.getRequestURI();

        Long allowed = redisTemplate.execute(rateLimitScript,
            List.of(key),
            "100",   // capacity: 100 requests
            "50",    // refill: 50 req/sec
            String.valueOf(System.currentTimeMillis())
        );

        if (Long.valueOf(0L).equals(allowed)) {
            resp.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            resp.getWriter().write("{\"error\":\"rate_limit_exceeded\"}");
            return;
        }

        chain.doFilter(req, resp);
    }
}
```

---

### Notification System — Async, Multi-Channel, Retry

**Requirements:** Send notifications (payment confirm, lead assign, OTP) via email, SMS, push. Async delivery with retry on failure.

```java
// Channel interface — Strategy pattern
public interface NotificationChannel {
    boolean send(NotificationRequest request);
    ChannelType getChannelType();
}

// Channel implementations
@Component
public class EmailChannel implements NotificationChannel {
    public boolean send(NotificationRequest request) {
        return javaMailSender.send(buildMimeMessage(request));
    }
    public ChannelType getChannelType() { return ChannelType.EMAIL; }
}

@Component
public class SmsChannel implements NotificationChannel {
    public boolean send(NotificationRequest request) {
        return smsProvider.send(request.getPhone(), request.getMessage());
    }
    public ChannelType getChannelType() { return ChannelType.SMS; }
}

// Template rendering — Template Method
public abstract class NotificationTemplate {
    public final String render(Map<String, Object> variables) {
        String rawTemplate = loadTemplate();
        return processVariables(rawTemplate, variables);
    }

    protected abstract String loadTemplate();

    private String processVariables(String template, Map<String, Object> vars) {
        // Thymeleaf/Mustache template engine
        return templateEngine.process(template, new Context(Locale.ENGLISH, vars));
    }
}

// Orchestration with retry
@Service
public class NotificationService {
    private final Map<ChannelType, NotificationChannel> channels;

    @Async("notificationExecutor")
    @Retryable(value = NotificationDeliveryException.class,
               maxAttempts = 3,
               backoff = @Backoff(delay = 1000, multiplier = 2))
    public CompletableFuture<DeliveryResult> send(NotificationRequest request) {
        List<ChannelType> preferredChannels = 
            userPreferenceService.getPreferredChannels(request.getUserId());

        for (ChannelType channelType : preferredChannels) {
            NotificationChannel channel = channels.get(channelType);
            try {
                boolean sent = channel.send(request);
                if (sent) {
                    return CompletableFuture.completedFuture(
                        DeliveryResult.success(channelType)
                    );
                }
            } catch (Exception e) {
                log.warn("Channel {} failed for user {}: {}", channelType,
                         request.getUserId(), e.getMessage());
            }
        }
        throw new NotificationDeliveryException("All channels failed for " + request.getUserId());
    }

    @Recover
    public CompletableFuture<DeliveryResult> onAllRetriesExhausted(
            NotificationDeliveryException ex, NotificationRequest request) {
        failedNotificationRepository.save(request);  // for manual retry / dead letter
        log.error("Notification permanently failed for user {}: {}", 
                  request.getUserId(), ex.getMessage());
        return CompletableFuture.completedFuture(DeliveryResult.failed(ex));
    }
}
```

---

### Parking Lot — LLD

**Entities:**
```java
public enum VehicleType { MOTORCYCLE, CAR, BUS }
public enum SpotType { SMALL, MEDIUM, LARGE }

public class Vehicle {
    private String licensePlate;
    private VehicleType type;
}

public class ParkingSpot {
    private String spotId;
    private SpotType type;
    private boolean occupied;
    
    public boolean canFit(VehicleType vehicleType) {
        return switch (vehicleType) {
            case MOTORCYCLE -> true;  // fits any spot
            case CAR -> type == SpotType.MEDIUM || type == SpotType.LARGE;
            case BUS -> type == SpotType.LARGE;
        };
    }
}

public class ParkingTicket {
    private String ticketId;
    private Vehicle vehicle;
    private ParkingSpot spot;
    private Instant entryTime;
    private Instant exitTime;
    private BigDecimal fee;
}
```

**Fee Strategy:**
```java
public interface FeeStrategy {
    BigDecimal calculate(Duration duration, VehicleType vehicleType);
}

public class HourlyFeeStrategy implements FeeStrategy {
    private static final Map<VehicleType, BigDecimal> HOURLY_RATES = Map.of(
        VehicleType.MOTORCYCLE, new BigDecimal("20"),
        VehicleType.CAR, new BigDecimal("40"),
        VehicleType.BUS, new BigDecimal("100")
    );

    public BigDecimal calculate(Duration duration, VehicleType vehicleType) {
        long hours = Math.max(1, duration.toHours());  // minimum 1 hour
        return HOURLY_RATES.get(vehicleType).multiply(BigDecimal.valueOf(hours));
    }
}
```

**Spot Assignment + Observer:**
```java
public interface SpotAvailabilityListener {
    void onSpotOccupied(ParkingSpot spot);
    void onSpotFreed(ParkingSpot spot);
}

@Service
public class ParkingLotService {
    private final List<SpotAvailabilityListener> listeners = new CopyOnWriteArrayList<>();
    private final FeeStrategy feeStrategy;
    private final Map<SpotType, Queue<ParkingSpot>> availableSpots;

    public ParkingTicket park(Vehicle vehicle) {
        SpotType required = resolveSpotType(vehicle.getType());
        ParkingSpot spot = availableSpots.get(required).poll();
        if (spot == null) throw new ParkingFullException(required);
        
        spot.setOccupied(true);
        listeners.forEach(l -> l.onSpotOccupied(spot));  // notify observers
        
        ParkingTicket ticket = new ParkingTicket(vehicle, spot, Instant.now());
        return ticketRepository.save(ticket);
    }

    public BigDecimal checkout(String ticketId) {
        ParkingTicket ticket = ticketRepository.findById(ticketId).get();
        ticket.setExitTime(Instant.now());
        
        Duration duration = Duration.between(ticket.getEntryTime(), ticket.getExitTime());
        BigDecimal fee = feeStrategy.calculate(duration, ticket.getVehicle().getType());
        ticket.setFee(fee);
        
        ticket.getSpot().setOccupied(false);
        availableSpots.get(ticket.getSpot().getType()).offer(ticket.getSpot());
        listeners.forEach(l -> l.onSpotFreed(ticket.getSpot()));  // notify observers
        
        return fee;
    }
}
```

---

## PART 6 — PROJECT ARCHITECTURAL NARRATIVES

---

### Narrative 1: The 5-Layer Razorpay Idempotent Webhook Architecture

**Exact interview delivery:**

"Let me walk you through the most complex system I've designed — our idempotent webhook processor for Razorpay Smart Collect. The core problem is that payment webhooks are unreliable: they arrive out of order, they get retried by the provider, and a single missed duplicate can mean crediting a partner twice. In fintech, that's a direct financial loss.

I designed a 5-layer defense-in-depth approach:

**Layer 1 — Immediate HTTP ACK.** Razorpay's retry logic triggers if it doesn't get a 200 within 5 seconds. So we always ACK immediately upon receipt, before any processing. The payload goes to Kafka in the same controller method.

**Layer 2 — Redis SETNX deduplication.** The Kafka consumer's first action is `SETNX webhook:{event_id} 1 EX 86400`. If the key already exists, this event is a duplicate — we log it and skip. This handles the 99% case where Razorpay retries or our consumer processes twice.

**Layer 3 — Kafka as durable buffer.** Kafka gives us replay capability. If the consumer crashes mid-processing, Kafka's at-least-once guarantee means the event re-delivers. Layer 2 catches the re-delivery.

**Layer 4 — DB unique constraint.** `UNIQUE INDEX on (payment_reference_id, event_type)`. This is our last-resort deduplication — catches events where the Redis TTL has expired (after 24 hours). We use `INSERT ... ON CONFLICT DO NOTHING` so the operation is always safe.

**Layer 5 — Payment State Machine.** Even if all four layers fail simultaneously, the state machine refuses invalid transitions. `COMPLETED → COMPLETED` is a no-op. You literally cannot double-credit a payment because the state machine's transition table doesn't allow it.

The result: in 8 months of production, we processed over 50,000 webhook events with zero duplicate credits. We had 3 incidents where Razorpay sent duplicate events 25 minutes apart — all caught by Layer 4 (Redis TTL was 24 hours, so Layer 2 also would have caught it, but Layer 4 was the safety net).

The trade-off I'd discuss in a system design round: we chose Kafka over the Outbox Pattern for the buffer layer. Kafka is simpler operationally but introduces a soft guarantee — if both the DB write and the Kafka publish fail atomically, we could lose an event. The Outbox Pattern would give us true atomicity. We accepted this trade-off because our upstream (Razorpay) retries for 24 hours, giving us recovery time."

---

### Narrative 2: ThingsBoard Multi-Tenant Authentication

**Exact interview delivery:**

"ThingsBoard is an open-source IoT platform. We customized it for multi-tenancy — each enterprise customer is a tenant, with their own users, devices, and data. The core challenge was: how do you enforce tenant isolation at every layer — authentication, authorization, and data access?

I implemented a 4-layer approach:

**Layer 1 — TenantAuthenticationProvider.** When a user logs in, we don't just validate their credentials — we validate that they belong to the tenant they're claiming to access. Custom `AuthenticationProvider` that loads the user, checks their `tenant_id` against the request's tenant identifier, and builds a `TenantAwareAuthenticationToken` with the tenant context.

**Layer 2 — JWT with tenant_id claim.** The issued JWT embeds the `tenant_id`. Every subsequent request carries the tenant context without a DB lookup. The token is signed — tenants cannot forge a different `tenant_id`.

**Layer 3 — `JwtAuthenticationFilter` + `TenantContextHolder`.** Our custom `OncePerRequestFilter` validates the JWT, extracts the `tenant_id` claim, and stores it in a `ThreadLocal`-backed `TenantContextHolder`. Every service method can call `TenantContextHolder.getCurrentTenant()` without needing it passed as a parameter.

**Layer 4 — Hibernate `@Filter` for row-level isolation.** All entity queries automatically append `WHERE tenant_id = :currentTenantId`. We implemented `TenantIdentifierResolver` to read from `TenantContextHolder`. The `@Filter` is enabled per-request by an interceptor.

The trade-off I chose: row-level multi-tenancy vs schema-per-tenant. Schema-per-tenant gives stronger isolation and easier tenant-specific customization, but requires N database connections (one per schema per tenant), more complex migrations, and operational overhead. Row-level is simpler, scales to many tenants on shared infrastructure, and was the right choice for our deployment model."

---

### Narrative 3: WebSocket Lead Assignment System

**Exact interview delivery:**

"The lead assignment system had an interesting distributed systems problem. We use WebSockets to push real-time updates to partners — when a lead is assigned to you, you see it in your dashboard within 500ms. The challenge: we run multiple application instances. A lead assigned on Instance 1 needs to push the WebSocket message to the partner's browser, which might be connected to Instance 3.

**The multi-instance problem:**
WebSocket connections are stateful — they're TCP connections to a specific server instance. You can't push a message to a client from a server it's not connected to.

**Solution: Redis Pub/Sub as cross-instance relay.**
When we assign a lead: (1) Save the assignment to DB, (2) Publish to Redis channel `partner:{partnerId}:leads`. Every app instance subscribes to Redis. The instance that holds the partner's WebSocket connection receives the Redis message and pushes it to the browser.

```
Instance 1 (assigns lead) → Redis publish → All instances receive
Instance 3 (holds partner's WS) → gets Redis message → pushes to browser
```

**The retry queue:**
What if the partner is offline? We put the assignment in a `PendingAssignment` queue in the DB. On partner reconnect, we flush their pending queue. On timeout (partner offline for 4 hours), we reassign to next partner.

**The credit limit race condition:**
During assignment, we check and deduct the partner's credit limit. As discussed in the concurrency chapter — we used pessimistic locking (`SELECT FOR UPDATE`) here because assignment throughput was high enough that optimistic locking caused retry storms.

**Scale improvements I'd propose:**
The current Redis Pub/Sub approach doesn't scale to 100K concurrent WebSocket connections — it requires every message to fan out to all instances. The improvement: a session registry (Redis Hash) mapping `partner_id → instance_id`. Only the correct instance subscribes to the partner-specific channel. This eliminates cross-instance fan-out."

---

## QUICK-REFERENCE — Senior Filter Questions

**Q: Strategy vs State — how do you tell them apart?**
Strategy: the algorithm is chosen externally and doesn't change the object's identity. State: the behavior changes as the object transitions through states — the object "becomes" different. In Strategy, `context.setStrategy(new X())` is an external decision. In State, `context.setState(new X())` is triggered by an internal transition.

**Q: Why use Decorator instead of inheritance for adding behavior?**
Inheritance is static — you must decide at compile time which behaviors to combine. Decorator is dynamic — you compose behaviors at runtime. More importantly, inheritance for cross-cutting concerns causes a class explosion: `LoggingRetryingMetricsRazorpayGateway` is not a maintainable hierarchy.

**Q: How do you test code that uses Strategy pattern?**
Inject mock strategies. `new PaymentProcessor(mockGateway)` — pure constructor injection makes testing trivial. Compare to: `new PaymentProcessor()` where the gateway is created internally via `new RazorpayGateway()` — untestable without reflection.

**Q: When should you NOT use design patterns?**
When the code is simple enough that the pattern adds overhead without benefit. A class that processes one type of payment, never needs extension, and will be replaced in 6 months doesn't need an Abstract Factory. Premature abstraction is as harmful as no abstraction — it adds complexity without solving a current problem. Apply patterns when you see the actual need (multiple implementations, behavior that varies, complex object construction), not preemptively.

---

> **What's Next — Chapter 4:** HLD Frameworks, Distributed Systems & Behavioral Mastery — CAP theorem, Saga/Outbox/CQRS patterns, the 5-step HLD framework, complete STAR stories, and the full Multi-Tenant Payment Platform HLD.

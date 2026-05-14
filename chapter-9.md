# Chapter 9 — Remaining Spring Essentials & Java 17

---

## Part 1 — Global Exception Handling with `@ControllerAdvice`

### Q1. How do you implement production-grade global exception handling in Spring Boot?

**The naive approach that every mid-level candidate gives:**

Put try-catch blocks everywhere, or return `null` on errors, or let exceptions bubble up to a 500.

**The Senior answer — three layers of handling:**

---

**Layer 1: `@RestControllerAdvice` — centralized handler**

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // Your custom business exceptions
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {
        log.warn("Resource not found: {} at {}", ex.getMessage(), request.getRequestURI());
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse.of("RESOURCE_NOT_FOUND", ex.getMessage(), request));
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(
            BusinessException ex, HttpServletRequest request) {
        log.error("Business rule violation: {}", ex.getMessage());
        return ResponseEntity
            .status(HttpStatus.UNPROCESSABLE_ENTITY)
            .body(ErrorResponse.of("BUSINESS_ERROR", ex.getMessage(), request));
    }

    // Bean validation failures — @Valid on @RequestBody
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex,
            HttpHeaders headers, HttpStatusCode status, WebRequest request) {
        
        List<FieldError> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(e -> new FieldError(e.getField(), e.getDefaultMessage()))
            .toList();
        
        ValidationErrorResponse body = new ValidationErrorResponse(
            "VALIDATION_FAILED", "Request validation failed", fieldErrors);
        return ResponseEntity.badRequest().body(body);
    }

    // Access denied — wrong role
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(
            AccessDeniedException ex, HttpServletRequest request) {
        return ResponseEntity
            .status(HttpStatus.FORBIDDEN)
            .body(ErrorResponse.of("ACCESS_DENIED", "Insufficient permissions", request));
    }

    // Catch-all — never expose internal details
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(
            Exception ex, HttpServletRequest request) {
        log.error("Unhandled exception at {}: ", request.getRequestURI(), ex);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse.of("INTERNAL_ERROR", "An unexpected error occurred", request));
    }
}
```

---

**Layer 2: Structured error response model**

```java
public record ErrorResponse(
    String code,
    String message,
    String path,
    Instant timestamp,
    String traceId
) {
    public static ErrorResponse of(String code, String message, HttpServletRequest request) {
        return new ErrorResponse(
            code,
            message,
            request.getRequestURI(),
            Instant.now(),
            MDC.get("traceId")    // correlates to distributed trace
        );
    }
}

public record ValidationErrorResponse(
    String code,
    String message,
    List<FieldError> errors
) {
    public record FieldError(String field, String reason) {}
}
```

---

**Layer 3: Custom business exception hierarchy**

```java
// Base — carries HTTP status semantics
public abstract class AppException extends RuntimeException {
    private final String errorCode;
    public AppException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
    public String getErrorCode() { return errorCode; }
}

public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String resource, Object id) {
        super("RESOURCE_NOT_FOUND", resource + " not found with id: " + id);
    }
}

public class BusinessException extends AppException {
    public BusinessException(String code, String message) {
        super(code, message);
    }
}

public class ConflictException extends AppException {
    public ConflictException(String message) {
        super("CONFLICT", message);
    }
}
```

---

**`@RestControllerAdvice` vs `@ControllerAdvice`:**

| Annotation | Equivalent To | Use When |
|---|---|---|
| `@ControllerAdvice` | `@Component` + advice for controllers | MVC apps returning views |
| `@RestControllerAdvice` | `@ControllerAdvice` + `@ResponseBody` | REST APIs (JSON response) |

---

**`ResponseEntityExceptionHandler` — what it gives you:**

Spring's built-in handler already maps framework exceptions to proper HTTP statuses. Extending it means you inherit those mappings and can override specific ones:

- `MethodArgumentNotValidException` → 400
- `HttpMessageNotReadableException` → 400 (malformed JSON)
- `NoHandlerFoundException` → 404
- `HttpRequestMethodNotSupportedException` → 405
- `HttpMediaTypeNotAcceptableException` → 406

Override `handleExceptionInternal` if you want to wrap ALL framework errors in your standard `ErrorResponse` shape.

---

**Senior Signal:** The `traceId` in the error response body is the differentiator — it links the user-facing error to the exact log line in Kibana/Splunk. Combined with MDC propagation through your filter chain, every error is fully traceable across microservices. "We had a production incident where we correlated a 422 response the customer screenshotted to the exact Kafka message that triggered the double-write — only possible because of the traceId in our error body."

---

## Part 2 — Bean Validation

### Q2. How do you implement request validation and custom business rules with Bean Validation?

---

**Built-in constraints (memorize these):**

```java
public record CreatePaymentRequest(
    @NotNull(message = "Amount is required")
    @Positive(message = "Amount must be positive")
    BigDecimal amount,

    @NotBlank(message = "Currency must not be blank")
    @Size(min = 3, max = 3, message = "Currency must be 3-letter ISO code")
    String currency,

    @NotNull
    @Valid          // cascade validation into nested object
    BeneficiaryDetails beneficiary,

    @Email(message = "Invalid email format")
    String notificationEmail,

    @Pattern(regexp = "^[A-Z]{2}[0-9]{2}[A-Z0-9]{4}[0-9]{7}([A-Z0-9]?){0,16}$",
             message = "Invalid IBAN format")
    String iban
) {}

public record BeneficiaryDetails(
    @NotBlank String name,
    @NotBlank @Size(min = 9, max = 18) String accountNumber
) {}
```

**Triggering validation in the controller:**

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResponse> create(
        @Valid @RequestBody CreatePaymentRequest request) {   // @Valid triggers validation
    return ResponseEntity.ok(paymentService.create(request));
}
```

When `@Valid` fails, Spring throws `MethodArgumentNotValidException` — caught by your `@RestControllerAdvice`.

---

**Custom `ConstraintValidator` — the Senior differentiator:**

Built-in annotations cover field-level rules. Real business rules (e.g., "SWIFT code is required for international transfers") need custom validators.

```java
// Step 1: Define the annotation
@Target({ElementType.TYPE})        // class-level constraint
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = InternationalTransferValidator.class)
public @interface ValidInternationalTransfer {
    String message() default "SWIFT code required for international transfers";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Step 2: Implement ConstraintValidator<Annotation, TargetType>
public class InternationalTransferValidator
        implements ConstraintValidator<ValidInternationalTransfer, CreatePaymentRequest> {

    @Override
    public boolean isValid(CreatePaymentRequest request, ConstraintValidatorContext ctx) {
        if (request == null) return true;   // null handled by @NotNull separately
        
        boolean isInternational = !"INR".equals(request.currency());
        boolean hasSwift = request.swiftCode() != null && !request.swiftCode().isBlank();
        
        if (isInternational && !hasSwift) {
            // Customize the error message to point to the specific field
            ctx.disableDefaultConstraintViolation();
            ctx.buildConstraintViolationWithTemplate("SWIFT code required for non-INR transfers")
               .addPropertyNode("swiftCode")
               .addConstraintViolation();
            return false;
        }
        return true;
    }
}

// Step 3: Apply on the class
@ValidInternationalTransfer
public record CreatePaymentRequest(
    BigDecimal amount,
    String currency,
    String swiftCode,
    ...
) {}
```

---

**`@Validated` vs `@Valid` — a critical distinction:**

| | `@Valid` (Jakarta) | `@Validated` (Spring) |
|---|---|---|
| Source | Jakarta EE | Spring Framework |
| Location | Method parameters | Class or method |
| Validation groups | Not supported | Supported |
| Method-level validation | No | Yes (with AOP proxy) |

**Method-level validation with `@Validated`:**

```java
@Service
@Validated      // enables Spring's AOP-based method validation
public class PaymentService {

    public PaymentResponse process(@Valid @NotNull ProcessRequest request) {
        // Spring validates before entering this method
        // throws ConstraintViolationException (NOT MethodArgumentNotValidException)
        ...
    }
}
```

**Catch `ConstraintViolationException` in `@ExceptionHandler`:**

```java
@ExceptionHandler(ConstraintViolationException.class)
public ResponseEntity<ValidationErrorResponse> handleConstraintViolation(
        ConstraintViolationException ex, HttpServletRequest request) {
    
    List<FieldError> errors = ex.getConstraintViolations().stream()
        .map(v -> new FieldError(
            v.getPropertyPath().toString(),
            v.getMessage()))
        .toList();
    
    return ResponseEntity.badRequest()
        .body(new ValidationErrorResponse("VALIDATION_FAILED", "Validation failed", errors));
}
```

---

**Validation Groups — for different rules on create vs update:**

```java
public interface OnCreate {}
public interface OnUpdate {}

public class UserRequest {
    @NotNull(groups = OnCreate.class)   // required only on create
    String password;
    
    @NotNull(groups = {OnCreate.class, OnUpdate.class})
    String email;
}

@PostMapping
public ResponseEntity<Void> create(@Validated(OnCreate.class) @RequestBody UserRequest req) { ... }

@PutMapping("/{id}")
public ResponseEntity<Void> update(@Validated(OnUpdate.class) @RequestBody UserRequest req) { ... }
```

---

**Senior Signal:** "In the payment project, we had a business rule where the beneficiary account validation logic changed depending on the payment rail — NEFT required 11-digit IFSC, RTGS required the same but above a minimum amount, UPI required a VPA pattern. We implemented a `PaymentRailValidator` that injected `PaymentRailConfig` from the database — custom validators are Spring beans and support `@Autowired` injection. This let us keep all validation declarative and centralized rather than scattered across service methods."

---

## Part 3 — Spring Profiles

### Q3. How do you manage environment-specific configuration with Spring Profiles?

---

**The problem profiles solve:**

You need different database URLs, different Kafka brokers, different feature flags, and different external API endpoints across local / dev / staging / production — without if-else in code.

---

**Core mechanics:**

```
application.properties           ← always loaded (base config)
application-local.properties     ← loaded only when profile = local
application-dev.properties       ← loaded only when profile = dev
application-staging.properties   ← loaded only when profile = staging
application-prod.properties      ← loaded only when profile = prod
```

```properties
# application.properties
spring.datasource.hikari.maximum-pool-size=10
app.feature.new-payment-flow=false

# application-prod.properties
spring.datasource.url=jdbc:postgresql://prod-rds.internal:5432/payments
spring.datasource.hikari.maximum-pool-size=50
app.feature.new-payment-flow=true
spring.kafka.bootstrap-servers=kafka-cluster.prod.internal:9092
```

---

**Profile-specific beans — `@Profile`:**

```java
// Only created when profile = local or test
@Configuration
@Profile({"local", "test"})
public class LocalDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("schema.sql")
            .addScript("test-data.sql")
            .build();
    }
}

// Production uses RDS
@Configuration
@Profile("prod")
public class ProdDataSourceConfig {
    @Bean
    public DataSource dataSource(@Value("${spring.datasource.url}") String url, ...) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setMaximumPoolSize(50);
        return new HikariDataSource(config);
    }
}
```

---

**Activating profiles — three ways:**

```properties
# 1. In application.properties (rarely used — defeats the purpose)
spring.profiles.active=local
```

```bash
# 2. JVM system property (CI/CD pipeline)
java -Dspring.profiles.active=prod -jar app.jar
```

```yaml
# 3. Kubernetes environment variable (production standard)
# deployment.yaml
spec:
  containers:
    - name: payment-service
      env:
        - name: SPRING_PROFILES_ACTIVE
          value: prod
        - name: SPRING_DATASOURCE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
```

---

**Profile ordering — property override precedence:**

Spring loads `application.properties` first, then the active profile properties. Profile-specific values override base values. If multiple profiles are active (`SPRING_PROFILES_ACTIVE=prod,feature-x`), last one wins for conflicting keys.

---

**`@ActiveProfiles` in tests:**

```java
@SpringBootTest
@ActiveProfiles("test")     // activates application-test.properties
class PaymentServiceIntegrationTest {
    // uses H2 in-memory, mock Kafka, etc.
}

@WebMvcTest(PaymentController.class)
@ActiveProfiles("test")
class PaymentControllerTest {
    @MockBean PaymentService paymentService;
}
```

---

**`@ConditionalOnProperty` — feature flags without profiles:**

```java
// Bean exists only when app.feature.new-payment-flow=true
@Service
@ConditionalOnProperty(name = "app.feature.new-payment-flow", havingValue = "true")
public class NewPaymentFlowService implements PaymentFlowService {
    ...
}

// Fallback when flag is false or missing
@Service
@ConditionalOnProperty(name = "app.feature.new-payment-flow",
                       havingValue = "false", matchIfMissing = true)
public class LegacyPaymentFlowService implements PaymentFlowService {
    ...
}
```

This gives you runtime feature toggling without redeployment when combined with Spring Cloud Config + actuator refresh.

---

**Your Project Angle:** "In the ThingsBoard customization, we needed different authentication providers for local testing (bypass JWT) vs production (full JWT validation). We used `@Profile("local")` on a permissive security config that allowed all requests, and the production `SecurityConfig` was the default bean active for all other profiles. This let developers run locally without needing a valid JWT while production remained locked down — zero if-else in security code."

---

## Part 4 — Java 17 New Features

### Q4. What are the key Java 17 features and how do they apply to production backend code?

Java 17 is the current LTS at most companies. These features come up in Senior interviews because they signal you're current.

---

**1. Records — immutable data carriers**

```java
// Before Java 16 — 25 lines of boilerplate
public final class PaymentEvent {
    private final String paymentId;
    private final BigDecimal amount;
    private final Instant timestamp;
    
    public PaymentEvent(String paymentId, BigDecimal amount, Instant timestamp) {
        this.paymentId = paymentId;
        this.amount = amount;
        this.timestamp = timestamp;
    }
    // + getters, equals, hashCode, toString
}

// Java 16+ Record — 1 line, same semantics
public record PaymentEvent(String paymentId, BigDecimal amount, Instant timestamp) {}
// Compiler generates: constructor, accessors (no get prefix), equals, hashCode, toString
// All fields are final — immutable by default

// Compact canonical constructor for validation
public record Money(BigDecimal amount, String currency) {
    public Money {  // compact constructor — no parameter list
        if (amount == null || amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount must be non-negative");
        if (currency == null || currency.length() != 3)
            throw new IllegalArgumentException("Currency must be 3-char ISO code");
    }
}

// Records work with Jackson (with jackson-module-parameter-names or @JsonCreator)
// Records work as DTOs, event payloads, value objects
// Records CANNOT extend classes (implicit final), CAN implement interfaces
```

**Where records shine in backend:**
- DTO / request-response objects (immutable, no accidental mutation)
- Event payloads on Kafka (serialized with Jackson)
- Value objects in domain model (Money, AccountId, TenantId)
- Configuration properties classes

---

**2. Sealed Classes — exhaustive hierarchies**

```java
// Sealed: defines the complete set of subclasses at compile time
public sealed interface PaymentResult
    permits PaymentResult.Success, PaymentResult.Failure, PaymentResult.Pending {}

public record Success(String transactionId, BigDecimal amount) implements PaymentResult {}
public record Failure(String errorCode, String message)       implements PaymentResult {}
public record Pending(String referenceId, Instant expectedAt) implements PaymentResult {}

// Exhaustive switch — compiler error if you miss a case
String message = switch (result) {
    case Success s  -> "Payment " + s.transactionId() + " confirmed";
    case Failure f  -> "Failed: " + f.message();
    case Pending p  -> "Awaiting confirmation by " + p.expectedAt();
    // No default needed — compiler knows all cases are covered
};
```

**Why Sealed + Pattern Matching is powerful:**

Before sealed classes, you'd use `instanceof` chains that were fragile — adding a new subtype silently broke handling. Now the compiler enforces completeness. This is the functional programming algebraic data type (sum type) in Java.

---

**3. Pattern Matching for `instanceof`**

```java
// Before Java 16
if (event instanceof PaymentEvent) {
    PaymentEvent pe = (PaymentEvent) event;   // explicit cast
    process(pe.paymentId());
}

// Java 16+ pattern matching
if (event instanceof PaymentEvent pe) {       // bind variable in condition
    process(pe.paymentId());
}

// Combine with && guard
if (event instanceof PaymentEvent pe && pe.amount().compareTo(BigDecimal.TEN) > 0) {
    processLargePayment(pe);
}

// Pattern matching in switch (Java 21 — but good to know direction)
String result = switch (event) {
    case PaymentEvent pe when pe.amount().signum() > 0 -> "credit " + pe.amount();
    case RefundEvent re  -> "refund " + re.originalId();
    default              -> "unknown event";
};
```

---

**4. Text Blocks**

```java
// Before — JSON template in Java was a nightmare
String json = "{\n" +
              "  \"paymentId\": \"" + id + "\",\n" +
              "  \"amount\": " + amount + "\n" +
              "}";

// Java 15+ text block — preserves indentation, handles escaping
String json = """
        {
          "paymentId": "%s",
          "amount": %s
        }
        """.formatted(id, amount);

// Practical uses:
// - SQL queries in tests
String query = """
        SELECT p.id, p.amount, p.status
        FROM payments p
        JOIN beneficiaries b ON b.id = p.beneficiary_id
        WHERE p.tenant_id = :tenantId
          AND p.status = :status
        ORDER BY p.created_at DESC
        LIMIT :limit
        """;

// - HTML templates in tests
// - Kafka message templates in integration tests
// - Multi-line error messages
```

---

**5. Switch Expressions (Java 14+)**

```java
// Old switch — fall-through, mutable, no return value
String label;
switch (status) {
    case PENDING: label = "Awaiting"; break;
    case SUCCESS: label = "Confirmed"; break;
    default:      label = "Unknown";
}

// Switch expression — exhaustive, no fall-through, returns value
String label = switch (status) {
    case PENDING -> "Awaiting";
    case SUCCESS -> "Confirmed";
    case FAILED  -> "Rejected";
    // If status is an enum, compiler checks all cases covered — no default needed
};

// Multi-statement case
String label = switch (status) {
    case PENDING -> {
        log.debug("Payment still pending");
        yield "Awaiting";   // yield replaces return inside switch expression
    }
    case SUCCESS -> "Confirmed";
    case FAILED  -> "Rejected";
};
```

---

**6. NullPointerException improvements (Java 14+)**

Helpful NPEs — the JVM now tells you WHICH field/variable was null:

```
// Old:
Cannot invoke "String.length()" because "str" is null

// New (Java 14+ with -XX:+ShowCodeDetailsInExceptionMessages, default in 17):
Cannot invoke "com.example.Payment.getId()" because the return value of
"com.example.PaymentRepository.findById(String)" is null
```

This alone saves significant debugging time in production.

---

**7. Records + Jackson integration:**

```java
// Add to pom.xml:
// jackson-module-parameter-names (preserves constructor param names)

// Or use @JsonCreator explicitly:
public record PaymentRequest(
    @JsonProperty("payment_id") String paymentId,
    @JsonProperty("amount")     BigDecimal amount
) {}

// Spring Boot auto-configures Jackson to handle records if you have:
// spring-boot-starter-web (includes jackson-databind 2.12+)
```

---

**Senior Signal:** "In the Java 17 migration on ThingsBoard, records became our standard for all API DTOs — they eliminated about 40% of the DTO boilerplate in the codebase and made the objects genuinely immutable, which removed an entire class of bugs where service code was mutating request objects. Sealed classes were used for the event hierarchy in our async processing pipeline, and the compiler-enforced exhaustiveness caught two missing cases in our Kafka consumer switch that would have silently dropped events."

---

## Part 5 — `@Async` Deep Dive

### Q5. How does `@Async` work, what are the traps, and how do you use it safely in production?

---

**What `@Async` does:**

Marks a method to execute on a separate thread from the calling thread. Spring wraps the call in a proxy and submits it to a `TaskExecutor`.

```java
@SpringBootApplication
@EnableAsync    // Step 1: enable async support
public class Application { ... }

@Service
public class NotificationService {

    @Async           // Step 2: mark method async
    public void sendEmail(String to, String subject, String body) {
        // runs on a separate thread from an executor pool
        emailClient.send(to, subject, body);
        // return type is void — fire-and-forget
    }

    @Async
    public CompletableFuture<Boolean> sendEmailWithResult(String to, String subject, String body) {
        // return CompletableFuture to allow caller to await result
        boolean sent = emailClient.send(to, subject, body);
        return CompletableFuture.completedFuture(sent);
    }
}

// Caller
notificationService.sendEmail(user.email(), "Welcome", body);  // returns immediately
// email is sent asynchronously — caller thread continues
```

---

**Valid return types for `@Async`:**

| Return Type | Behavior |
|---|---|
| `void` | Fire-and-forget, exceptions swallowed by default |
| `Future<T>` | Caller can call `.get()` to block and retrieve |
| `CompletableFuture<T>` | Full async pipeline, `.thenApply()`, `.exceptionally()` |
| `ListenableFuture<T>` | Spring-specific, deprecated — don't use |

---

**The Proxy Trap — same as `@Transactional`:**

`@Async` is implemented via Spring AOP proxy. The same self-invocation problem applies:

```java
@Service
public class PaymentService {

    @Async
    public void processAsync(String id) { ... }

    public void trigger(String id) {
        this.processAsync(id);   // BROKEN — calling through 'this', not the proxy
        // processAsync runs SYNCHRONOUSLY
    }
}

// Fix 1: Inject self
@Service
public class PaymentService {
    @Autowired
    private PaymentService self;    // Spring injects the proxy

    public void trigger(String id) {
        self.processAsync(id);      // goes through the proxy, runs async
    }
}

// Fix 2: Extract to separate bean
@Service
public class PaymentAsyncBridge {
    @Async
    public void processAsync(String id) { ... }
}

@Service
public class PaymentService {
    @Autowired
    private PaymentAsyncBridge asyncBridge;

    public void trigger(String id) {
        asyncBridge.processAsync(id);   // different bean = proxy invocation
    }
}
```

---

**Exception handling for void `@Async` methods:**

Exceptions from `void` async methods are silently swallowed unless you configure an `AsyncUncaughtExceptionHandler`:

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (throwable, method, params) -> {
            log.error("Async exception in method [{}] with params {}: {}",
                method.getName(), Arrays.toString(params), throwable.getMessage(), throwable);
            // optionally: alert, metrics increment, dead-letter queue
        };
    }
}
```

For `CompletableFuture<T>` return types, handle exceptions with `.exceptionally()` on the caller side — the `AsyncUncaughtExceptionHandler` is NOT invoked for those.

---

**Custom executor — production-grade configuration:**

The default executor is `SimpleAsyncTaskExecutor` — it creates a new thread for EVERY call, no pooling. Never use this in production.

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-notification-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

**Multiple executors for different workloads:**

```java
@Configuration
public class AsyncExecutorConfig {

    // Heavy I/O — email sending, file uploads
    @Bean("ioExecutor")
    public Executor ioExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(20);
        exec.setMaxPoolSize(100);
        exec.setQueueCapacity(500);
        exec.setThreadNamePrefix("io-async-");
        exec.initialize();
        return exec;
    }

    // CPU work — report generation, data transformation
    @Bean("cpuExecutor")
    public Executor cpuExecutor() {
        int cores = Runtime.getRuntime().availableProcessors();
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(cores);
        exec.setMaxPoolSize(cores * 2);
        exec.setQueueCapacity(50);
        exec.setThreadNamePrefix("cpu-async-");
        exec.initialize();
        return exec;
    }
}

// Use by name
@Async("ioExecutor")
public void sendEmail(...) { ... }

@Async("cpuExecutor")
public CompletableFuture<ReportData> generateReport(...) { ... }
```

---

**`@Async` + `@Transactional` interaction:**

`@Async` submits to a new thread. `@Transactional` context is thread-local. Therefore, the async method does NOT inherit the caller's transaction — it starts a fresh transaction (or no transaction if not annotated).

```java
@Transactional
public void createPaymentAndNotify(Payment payment) {
    paymentRepository.save(payment);
    // payment not yet committed — it's in this thread's transaction
    
    notificationService.sendEmailAsync(payment.getId());
    // async thread starts — if it queries for the payment, it may NOT find it yet
    // (depends on isolation level and whether the calling transaction committed)
    
    // Fix: pass data directly to the async method instead of re-querying
    notificationService.sendEmailAsync(payment.getUserEmail(), payment.getAmount());
}
```

---

**Production Pattern — event-driven async (cleaner than `@Async`):**

For decoupled async processing, prefer Spring `ApplicationEvent` + `@TransactionalEventListener` over `@Async`:

```java
// After the transaction commits, fire the event
@Service
public class PaymentService {
    @Autowired ApplicationEventPublisher publisher;

    @Transactional
    public void create(Payment payment) {
        paymentRepository.save(payment);
        publisher.publishEvent(new PaymentCreatedEvent(payment.getId(), payment.getUserEmail()));
        // event fires AFTER transaction commits — data is visible to all listeners
    }
}

@Component
public class PaymentNotificationListener {

    @Async("ioExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onPaymentCreated(PaymentCreatedEvent event) {
        // runs async AFTER transaction committed — payment is fully saved
        emailService.sendConfirmation(event.userEmail(), event.paymentId());
    }
}
```

This pattern eliminates the transaction-visibility race condition.

---

**Quick Reference — `@Async` checklist:**

- `@EnableAsync` on your `@Configuration` class — or main class
- Never call `@Async` methods from within the same bean (self-invocation bypasses proxy)
- Always configure a custom `ThreadPoolTaskExecutor` — never use the default
- For `void` methods: configure `AsyncUncaughtExceptionHandler`
- For `CompletableFuture` methods: use `.exceptionally()` on the caller
- `@Async` + `@Transactional` on same method starts a new transaction, not continuation
- Prefer `@TransactionalEventListener` over `@Async` when you need post-commit guarantees

---

**Senior Signal:** "In the payment system, we had a bug where async email notifications were querying the payment table immediately after the creating transaction submitted the async task — but before the transaction committed. Some users got emails saying their payment was pending when it was actually confirmed. We fixed it with `@TransactionalEventListener(phase = AFTER_COMMIT)` — the notification job now fires only after the transaction is durably committed, eliminating the race condition entirely."

---

## Quick Reference — Chapter 9 Senior Filter Questions

**Q: What's the difference between `@Valid` and `@Validated`?**
`@Valid` is Jakarta EE — triggers cascade validation on `@RequestBody`, used in controllers. `@Validated` is Spring — enables AOP-based method-level validation on service beans, supports validation groups.

**Q: Where does `ConstraintViolationException` come from vs `MethodArgumentNotValidException`?**
`MethodArgumentNotValidException` — `@Valid` fails on a controller `@RequestBody`. `ConstraintViolationException` — `@Validated` fails on a `@Service` method parameter. Both need separate `@ExceptionHandler` mappings.

**Q: Why does `@Async` fail when called from the same class?**
`@Async` works via Spring's AOP proxy. Internal calls (`this.method()`) bypass the proxy and execute synchronously on the calling thread. Fix by injecting self or extracting to a separate Spring bean.

**Q: What is a Java Record and what can't it do?**
Record is a transparent, immutable data carrier — compiler generates constructor, accessors, `equals`, `hashCode`, `toString`. Cannot extend classes (implicitly `final`), cannot have mutable fields (`final` by definition), cannot add non-static fields beyond the record components.

**Q: When would you use a Sealed class?**
When you have a fixed set of subtypes and want the compiler to enforce exhaustive handling in switch expressions. Classic use: result types (`Success | Failure | Pending`), domain events, command/query objects. Eliminates the `default` fallback that silently ignores new cases.

**Q: How do you activate a Spring profile in Kubernetes?**
Set environment variable `SPRING_PROFILES_ACTIVE=prod` in the deployment YAML. Secrets (DB password, API keys) come from `secretKeyRef` — never bake them into profile properties files committed to git.

**Q: What happens to exceptions in a void `@Async` method?**
Silently swallowed by default. You must implement `AsyncUncaughtExceptionHandler` via `AsyncConfigurer` to log, alert, or re-queue them.

**Q: How do you make `@Async` respect transaction commit order?**
Use `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)` instead of `@Async` directly. This fires the async listener only after the calling transaction durably commits, ensuring the async work sees consistent data.

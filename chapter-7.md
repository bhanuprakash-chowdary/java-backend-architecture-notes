# Interview Survival Guide
## Chapter 7: Microservices Infrastructure
### (Service Discovery · API Gateway · Config Management · Distributed Tracing · Docker · Kubernetes)

---

## PART 1 — SERVICE DISCOVERY

---

### Q1. How do microservices find each other? Explain service discovery.

**The answer:**

"In a monolith, your DB connection string is a fixed config value. In microservices, service instances come and go — new pods spin up, old ones die, IPs change. Service discovery is the mechanism that lets Service A find a live instance of Service B at runtime.

**Client-Side Discovery (Eureka — Netflix OSS):**
Each service registers itself with a registry on startup. When Service A needs to call Service B, it queries the registry for a list of Service B instances and picks one (client-side load balancing via Ribbon/Spring Cloud LoadBalancer).

```
Service A → Eureka Registry: 'Where is payment-service?'
Eureka → Service A: [192.168.1.10:8080, 192.168.1.11:8080, 192.168.1.12:8080]
Service A → 192.168.1.11:8080: actual request (client picks instance)
```

```yaml
# application.yml — Eureka client config
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10    # heartbeat every 10s
    lease-expiration-duration-in-seconds: 30 # remove after 30s no heartbeat
```

```java
// Using @LoadBalanced RestTemplate — Spring Cloud auto-resolves service name
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// In service code:
ResponseEntity<Partner> response = restTemplate.getForEntity(
    "http://partner-service/api/v1/partners/{id}",  // 'partner-service' is the Eureka name
    Partner.class,
    partnerId
);
// Spring Cloud LoadBalancer resolves 'partner-service' to an actual IP via Eureka
```

**Server-Side Discovery (AWS ALB, Kubernetes Service):**
Clients call a stable virtual IP/hostname. A load balancer or proxy resolves to a live instance server-side. Client knows nothing about discovery — it just calls `http://payment-service/api/v1/payments`.

```
Service A → http://payment-service (stable DNS name)
           → Kubernetes Service (kube-proxy/iptables)
           → One of: [pod1, pod2, pod3]
```

Kubernetes uses server-side discovery natively — every `Service` object gets a DNS entry and load-balances across its Pods. This is the standard in Kubernetes deployments, making Eureka unnecessary.

**Health checking and self-healing:**
Eureka: service de-registers itself on shutdown, or gets removed after 3 missed heartbeats (30s).
Kubernetes: if a Pod's readiness probe fails, it's removed from the Service endpoints automatically.

**Senior Signal:** 'In a Kubernetes-native deployment, you don't need Eureka — Kubernetes Services provide service discovery out of the box. Eureka becomes relevant only in hybrid deployments (some services on K8s, some on VMs) or if you need client-side discovery with custom load-balancing logic (e.g., tenant-aware routing).'"

---

## PART 2 — API GATEWAY

---

### Q2. What is an API Gateway? What does Spring Cloud Gateway do?

**The answer:**

"An API Gateway is the single entry point for all client requests. It sits in front of all microservices and handles cross-cutting concerns that would otherwise be duplicated in every service: authentication, rate limiting, routing, SSL termination, request/response transformation, and observability.

**Without an API Gateway:**
```
Client → auth check in Service A → actual work
Client → auth check in Service B → actual work
Client → auth check in Service C → actual work
// Auth logic duplicated in every service
// Each service exposed directly to the internet
```

**With an API Gateway:**
```
Client → API Gateway (auth, rate limit, route) → Service A
                                               → Service B
                                               → Service C
// Auth happens once, services trust the gateway
// Services are not directly exposed
```

**Spring Cloud Gateway configuration:**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: payment-service
          uri: lb://payment-service        # 'lb://' = load-balanced via discovery
          predicates:
            - Path=/api/v1/payments/**
          filters:
            - StripPrefix=0
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100   # tokens/sec
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@tenantKeyResolver}"

        - id: partner-service
          uri: lb://partner-service
          predicates:
            - Path=/api/v1/partners/**
            - Header=X-Tenant-Id, .+     # only route if header present
```

```java
// Custom filter — add correlation ID and validate JWT at gateway level
@Component
public class AuthGatewayFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = extractBearerToken(exchange.getRequest());

        if (token == null) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        try {
            JwtClaims claims = jwtValidator.validate(token);

            // Add validated claims as headers to downstream services
            ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-Tenant-Id", claims.get("tenantId").toString())
                .header("X-Correlation-Id", UUID.randomUUID().toString())
                .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());
        } catch (JwtException e) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
    }

    @Override
    public int getOrder() { return -1; }  // run before all other filters
}

// Tenant-based rate limiter key
@Bean
public KeyResolver tenantKeyResolver() {
    return exchange -> Mono.justOrEmpty(
        exchange.getRequest().getHeaders().getFirst("X-Tenant-Id")
    ).defaultIfEmpty("anonymous");
}
```

**API Gateway vs Load Balancer:**

| | Load Balancer (ALB/NLB) | API Gateway |
|-|------------------------|-------------|
| Layer | L4 (TCP) or L7 (HTTP) | L7 (HTTP, application-aware) |
| Auth | No | Yes |
| Rate limiting | No | Yes |
| Request transformation | No | Yes |
| Business routing rules | No | Yes |
| Example | AWS ALB | Spring Cloud Gateway, Kong, AWS API Gateway |

**Senior Signal:** 'In production, both are used together. The internet-facing Load Balancer (ALB) handles TLS termination and distributes across API Gateway instances. The API Gateway handles authentication, rate limiting, and service routing. Individual microservices are only reachable through the gateway — never directly from the internet.'"

---

## PART 3 — CONFIG MANAGEMENT

---

### Q3. How do you manage configuration across microservices? Explain Spring Cloud Config.

**The answer:**

"In a microservices system, each service has its own `application.properties`. Managing these individually becomes painful: environment-specific configs (dev/staging/prod), secrets, and shared config values are duplicated or hard-coded. Spring Cloud Config Server solves this with a central configuration store.

**Architecture:**
```
Git Repo (config store)
  ├── application.yml          (shared by all services)
  ├── payment-service.yml      (payment-service specific)
  ├── payment-service-prod.yml (payment-service, production env)
  └── partner-service.yml

Config Server (Spring Boot app)
  ↑ reads from Git Repo

payment-service → Config Server: 'Give me config for payment-service, profile=prod'
Config Server → payment-service: merged config (application.yml + payment-service-prod.yml)
```

```yaml
# Config Server application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          search-paths: '{application}'
          default-label: main
          clone-on-start: true
          timeout: 5
```

```yaml
# Client service bootstrap.yml (loaded before application.yml)
spring:
  application:
    name: payment-service    # determines which config files are fetched
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true          # fail startup if config server unreachable
      retry:
        max-attempts: 6
```

**Dynamic config refresh (without restart):**
```java
@RestController
@RefreshScope  // re-created when /actuator/refresh is called
public class PaymentController {

    @Value("${payment.max-amount:100000}")
    private BigDecimal maxPaymentAmount;  // updated on /actuator/refresh
}
```

When config changes in Git → trigger `/actuator/refresh` endpoint (or use Spring Cloud Bus + Kafka/RabbitMQ for broadcast to all instances).

**Secrets management — never store secrets in Git:**
```yaml
# Config Server encrypting sensitive values
# Store in Git as: {cipher}AQA...encrypted-value...
spring:
  datasource:
    password: '{cipher}AQABMzQ1...'  # encrypted in Git, decrypted by Config Server

# Config Server decrypts using a symmetric key or asymmetric key pair
encrypt:
  key: ${ENCRYPT_KEY}  # injected from environment variable, not Git
```

**Production alternative — AWS Secrets Manager / Vault:**
For true secrets (DB passwords, API keys), use a dedicated secrets store:
```java
// Spring Cloud AWS auto-integration
@Value("${/myapp/prod/razorpay-api-key}")
private String razorpayApiKey;
// Value is fetched from AWS Secrets Manager at startup (not stored in Config Server)
```

**Senior Signal:** 'Spring Cloud Config is good for application configuration. For secrets — anything that provides access to other systems — use a dedicated secret manager (AWS Secrets Manager, HashiCorp Vault). The separation matters: config changes are low-risk (wrong `max-retries` value), secret changes are high-risk (exposed API key). Different teams should have different access levels to each.'"

---

## PART 4 — DISTRIBUTED TRACING

---

### Q4. What is distributed tracing? How does it differ from logging with correlation IDs?

**The answer:**

"Correlation IDs (Chapter 4) solve the problem of tracing a request through a single service — all log lines for request X share the same correlation ID. Distributed tracing solves a harder problem: tracing a request across multiple services, including timing information for each hop.

**The problem correlation IDs don't solve:**
```
Request PAY-001:
  payment-service: [correlationId=abc] Processing payment → 450ms
  partner-service: [correlationId=???] Validating credit  → ???ms
  fraud-service:   [correlationId=???] Checking fraud     → ???ms
```
The correlation ID doesn't propagate automatically across service boundaries. You don't know which partner-service call belongs to which payment-service request. And you have no timing breakdown.

**Distributed tracing: Spans + Traces:**
- **Trace**: the entire end-to-end flow for one request, identified by a `traceId`
- **Span**: a single unit of work within a trace (one service call, one DB query). Each span has a `spanId`, a `parentSpanId`, a start time, and a duration.

```
Trace: traceId=abc123
  ├── Span: payment-service.processPayment (root span)    450ms
  │     ├── Span: DB insert payment                        5ms
  │     ├── Span: partner-service.validateCredit          200ms  ← child span
  │     │     └── Span: DB select partner credit            3ms
  │     └── Span: fraud-service.checkFraud               240ms  ← child span
  │           └── Span: Redis lookup fraud score            1ms
  └── (total end-to-end: 450ms)
```

**OpenTelemetry + Spring Boot:**
OpenTelemetry is the vendor-neutral standard. Spring Boot 3 includes Micrometer Tracing which integrates with OpenTelemetry.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 0.1    # trace 10% of requests (100% in dev, 1–10% in prod)
  otlp:
    tracing:
      endpoint: http://jaeger:4318/v1/traces  # Jaeger or Zipkin collector
```

**Trace context propagation across services:**
Spring automatically injects `traceparent` header into outbound HTTP calls and Kafka messages when using instrumented `RestTemplate`, `WebClient`, or `KafkaTemplate`:
```
// Outbound HTTP request headers (automatic):
traceparent: 00-abc123traceId-spanId456-01
```

The downstream service reads this header and creates a child span under the same trace.

**Manual span creation:**
```java
@Service
public class PaymentService {

    private final Tracer tracer;

    public PaymentResult processPayment(Payment payment) {
        Span span = tracer.nextSpan().name("payment.process")
            .tag("payment.id", payment.getId().toString())
            .tag("payment.amount", payment.getAmount().toString())
            .start();

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            return doProcess(payment);
        } catch (Exception e) {
            span.tag("error", e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

**Jaeger UI — what it gives you:**
- Find all spans for a specific `traceId`
- See which service is the bottleneck (longest span)
- See dependency graph between services
- Compare traces: 'why was this request 2x slower than the median?'

**Sampling strategy:**
100% sampling = enormous storage cost + performance overhead. Production typical: 1% random sampling + 100% sampling for error traces (head-based vs tail-based sampling).

```java
// Always sample errors (tail-based sampling)
@Bean
public Sampler sampler() {
    return (traceId, parentId, operationName, tags) -> {
        if (tags.containsKey("error")) return SamplingFlags.SAMPLED;  // always trace errors
        return SamplingFlags.NOT_SAMPLED;  // skip 99% of successful requests
    };
}
```"

---

## PART 5 — DOCKER

---

### Q5. Explain Docker fundamentals. What is a layer cache and how do you write a production Dockerfile?

**The answer:**

"Docker packages an application and its dependencies into a portable, isolated container. The core concept: a container is an isolated process with its own filesystem (from an image), networking, and resource limits — sharing the host OS kernel.

**Image layers and layer cache:**
A Docker image is a stack of read-only layers. Each instruction in a Dockerfile creates a new layer. When you rebuild, Docker reuses cached layers for instructions that haven't changed.

```dockerfile
# INEFFICIENT — copies source code before installing dependencies
FROM eclipse-temurin:17-jre-alpine
COPY . /app                    # Layer 1: entire source (invalidates on any file change)
RUN mvn clean package          # Layer 2: always rebuilds dependencies

# EFFICIENT — dependencies layer is cached unless pom.xml changes
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY pom.xml .                 # Layer 1: only pom.xml (rarely changes)
RUN mvn dependency:go-offline  # Layer 2: cached as long as pom.xml unchanged
COPY src ./src                 # Layer 3: source code (changes often)
RUN mvn package -DskipTests    # Layer 4: rebuild only what changed
```

**Multi-stage build — keep the final image small:**
```dockerfile
# Stage 1: build (uses full JDK + Maven)
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: runtime (uses only JRE — much smaller)
FROM eclipse-temurin:17-jre-alpine AS runtime
WORKDIR /app

# Non-root user — security best practice
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copy only the JAR from the build stage
COPY --from=builder /app/target/payment-service-*.jar app.jar

EXPOSE 8080

# Use exec form (not shell form) — signals go directly to JVM, not sh
ENTRYPOINT ["java",
  "-XX:MaxRAMPercentage=75.0",    # use 75% of container memory limit
  "-XX:+UseG1GC",
  "-Dfile.encoding=UTF-8",
  "-jar", "app.jar"]
```

**Key Dockerfile best practices:**
- Use specific image tags (`eclipse-temurin:17.0.9-jre-alpine`), never `latest`
- Non-root user in production containers
- `.dockerignore` file to exclude `target/`, `.git/`, `*.md` from build context
- `ENTRYPOINT` exec form (`["java", ...]`) so SIGTERM reaches the JVM directly
- `-XX:MaxRAMPercentage=75.0` instead of `-Xmx` — adapts to the container's memory limit automatically

```
# .dockerignore
target/
.git/
.idea/
*.md
*.log
```

**Container resource limits (always set in production):**
```yaml
# Kubernetes
resources:
  requests:
    memory: "512Mi"    # minimum guaranteed
    cpu: "250m"        # 0.25 CPU cores
  limits:
    memory: "1Gi"      # hard cap — OOMKilled if exceeded
    cpu: "1000m"       # throttled if exceeded (not killed)
```

Without memory limits, a memory leak in one container can OOM the entire node."

---

## PART 6 — KUBERNETES

---

### Q6. Explain the core Kubernetes objects. How would you deploy the payment service?

**The answer:**

"Kubernetes (K8s) is a container orchestration platform — it manages deploying, scaling, and maintaining containerized applications. Think of it as an OS for distributed systems.

**Core objects:**

**Pod** — the smallest deployable unit. One or more containers sharing network and storage.
```yaml
# You rarely create Pods directly — Deployments manage them
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: payment-api
      image: myrepo/payment-service:v1.2.3
      ports:
        - containerPort: 8080
```

**Deployment** — declarative Pod management. Defines desired state; K8s reconciles actual state to match.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: payment-api
          image: myrepo/payment-service:v1.2.3
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: production
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: payment-db-secret
                  key: password
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

**Service** — stable DNS name + virtual IP for a set of Pods. Survives Pod restarts and IP changes.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-api
spec:
  selector:
    app: payment-api
  ports:
    - protocol: TCP
      port: 80           # service port (what callers use)
      targetPort: 8080   # container port
  type: ClusterIP        # internal only; use LoadBalancer or Ingress for external
```

**Ingress** — HTTP routing from outside the cluster.
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-ingress
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  rules:
    - host: api.mycompany.com
      http:
        paths:
          - path: /api/v1/payments
            pathType: Prefix
            backend:
              service:
                name: payment-api
                port:
                  number: 80
```

**ConfigMap and Secret:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-config
data:
  SPRING_PROFILES_ACTIVE: production
  LOG_LEVEL: INFO

---
apiVersion: v1
kind: Secret
metadata:
  name: payment-db-secret
type: Opaque
data:
  password: base64encodedpassword==  # base64, NOT encryption — use Sealed Secrets or Vault
```

**Horizontal Pod Autoscaler:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # scale up when CPU > 70%
```

**Common `kubectl` commands for interviews:**
```bash
kubectl get pods -n production                          # list pods
kubectl describe pod payment-api-xxx -n production      # pod details + events
kubectl logs payment-api-xxx -n production --tail=100   # pod logs
kubectl rollout status deployment/payment-api           # watch rollout progress
kubectl rollout undo deployment/payment-api             # rollback to previous
kubectl exec -it payment-api-xxx -- /bin/sh             # shell into running pod
kubectl top pods -n production                          # CPU/memory usage
```

**Senior Signal:** 'In production, you never create K8s manifests from scratch — you use Helm charts or Kustomize for templating. You also never edit secrets as base64 in YAML files — you use Sealed Secrets (encrypted in Git) or pull from AWS Secrets Manager at runtime. The YAML above is for understanding the objects, not for copy-pasting to production.'"

---

## PART 7 — INTER-SERVICE COMMUNICATION PATTERNS

---

### Q7. How do microservices communicate? When do you choose synchronous vs asynchronous?

**The answer:**

"Every inter-service call is either synchronous (caller blocks and waits for response) or asynchronous (caller publishes an event and continues).

**Synchronous communication — REST or gRPC:**
```java
// Service A (PaymentService) calling Service B (PartnerService) synchronously
@Service
public class PaymentService {

    private final WebClient webClient;

    public PartnerCreditInfo getCreditInfo(UUID partnerId) {
        return webClient.get()
            .uri("http://partner-service/api/v1/partners/{id}/credit", partnerId)
            .header("X-Tenant-Id", TenantContextHolder.getCurrentTenant().toString())
            .retrieve()
            .onStatus(HttpStatus::is5xxServerError, response ->
                Mono.error(new PartnerServiceUnavailableException()))
            .bodyToMono(PartnerCreditInfo.class)
            .timeout(Duration.ofMillis(500))   // always set timeouts
            .block();
    }
}
```

**Resilience for synchronous calls — the full stack:**
```java
@Bean
public WebClient resilientWebClient(CircuitBreakerRegistry cbRegistry) {
    CircuitBreaker cb = cbRegistry.circuitBreaker("partner-service");

    return WebClient.builder()
        .filter((request, next) ->
            Mono.fromCallable(() -> cb.executeCallable(() -> next.exchange(request).block()))
                .onErrorMap(CallNotPermittedException.class,
                    e -> new PartnerServiceUnavailableException("Circuit open"))
        )
        .build();
}
```

**Asynchronous communication — Kafka:**
```java
// Service A (LeadService) publishes event, does NOT wait for PartnerService response
@Service
public class LeadAssignmentService {

    public void assignLead(Lead lead) {
        // Do local work
        leadRepository.save(lead);

        // Publish event — fire and forget
        kafkaTemplate.send("lead-assigned", lead.getPartnerId().toString(),
            new LeadAssignedEvent(lead.getId(), lead.getPartnerId(), Instant.now()));

        // Return immediately — PartnerService processes asynchronously
    }
}

// PartnerService consumes the event independently
@KafkaListener(topics = "lead-assigned")
public void onLeadAssigned(LeadAssignedEvent event) {
    partnerCreditService.deductCredit(event.getPartnerId(), event.getCreditRequired());
    notificationService.notifyPartner(event);
}
```

**Decision framework:**

| Scenario | Communication | Reason |
|----------|--------------|--------|
| Need result to continue (credit check before assign) | Synchronous | Response required |
| Independent side effect (send email after payment) | Asynchronous | Decouple, email failure shouldn't fail payment |
| Real-time user-facing (< 500ms SLA) | Synchronous | Async adds latency |
| Batch or background processing | Asynchronous | Throughput over latency |
| Multiple consumers need same event | Asynchronous (Kafka) | Fan-out naturally |
| Strict ordering required | Synchronous or Kafka (same partition) | Guarantee ordering |

**The synchronous dependency chain problem:**
```
Request → Service A → Service B → Service C → Service D
```
If Service D has 99.9% availability: 0.999^4 = 99.6% end-to-end availability.
The longer the synchronous chain, the lower the availability. Break long chains with async communication or caching."

---

## QUICK-REFERENCE — Chapter 7 Senior Filter Questions

**Q: What is a service mesh (Istio)? Do you need it?**
A service mesh adds a sidecar proxy (Envoy) to every pod. The sidecar handles: mTLS between services, circuit breaking, retries, load balancing, and distributed tracing — without any code changes. Use it when: you need strong security (mTLS everywhere), multi-language teams (each team's service gets resilience for free), or when centralized traffic policy is needed. Don't use it for simple architectures — the operational complexity is significant.

**Q: What is the difference between a Kubernetes Service and an Ingress?**
A Service provides stable internal DNS and load-balancing within the cluster (`ClusterIP`). An Ingress handles external HTTP/HTTPS traffic routing into the cluster — host-based routing, path-based routing, TLS termination. An Ingress controller (NGINX, Traefik, ALB Controller) is needed to implement the Ingress resource.

**Q: How do you handle database access in microservices? Can two services share a DB?**
Each service should own its database (Database-per-Service pattern). Sharing a DB creates tight coupling — Service A's schema change can break Service B. If cross-service data access is needed, use the API (call Service B's endpoint) or events (subscribe to Service B's events). Shared DB is acceptable temporarily during a monolith decomposition, but is an anti-pattern for greenfield microservices.

**Q: What happens to in-flight requests during a Kubernetes rolling update?**
With `maxUnavailable: 0` and `preStop: sleep 5`, the flow is: (1) new pod starts and passes readiness probe, (2) new pod added to Service endpoints, (3) old pod receives SIGTERM (after preStop sleep delay), (4) old pod stops accepting new connections, (5) old pod finishes in-flight requests within `terminationGracePeriodSeconds`, (6) old pod exits. No request is dropped if all four conditions hold: `graceful shutdown`, `readiness probe`, `preStop hook`, `terminationGracePeriodSeconds > Spring shutdown timeout`.

---

> **What's Next — Chapter 8:** Spring WebFlux & Reactive Programming — non-blocking I/O model, Project Reactor, Mono/Flux, backpressure, and when reactive beats imperative.

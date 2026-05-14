# Interview Survival Guide
## Chapter 1: Java Core & Spring Internals Deep Dive
### (Weeks 1, 2, 5 & 6)

---

## PART 1 — JVM ARCHITECTURE & MEMORY MODEL

---

### Q1. Walk me through the JVM memory model. Where does each type of data live?

**The answer interviewers want:**

"The JVM divides memory into several runtime data areas, each serving a specific purpose.

**Heap** is the largest area — shared across all threads. All object instances and arrays live here. It's divided into Young Generation (Eden + Survivor S0/S1) and Old Generation. G1GC also uses the concept of regions instead of fixed contiguous spaces.

**Stack** — one per thread. Holds stack frames: local variables, method parameters, partial results, and the return address. Each method invocation creates a new frame. When a method returns, the frame is popped. StackOverflowError happens when the stack depth exceeds the limit (deep recursion).

**Metaspace** (replaced PermGen in Java 8) — stores class metadata: bytecode, method signatures, constant pools. Unlike PermGen, Metaspace grows dynamically using native memory. You set `-XX:MaxMetaspaceSize` to cap it; by default, it's unbounded. Metaspace OutOfMemoryError is the result of class loader leaks — typically in app servers where you redeploy and the old ClassLoader never gets GC'd.

**Code Cache** — stores JIT-compiled native code. When the JIT compiles a hot method, the native binary goes here. `-XX:ReservedCodeCacheSize` controls it. If it fills up, the JVM deoptimizes and falls back to interpreted mode — sudden performance degradation at production load.

**PC Register** — one per thread, holds the address of the currently executing bytecode instruction. For native methods, it's undefined.

**Native Method Stack** — supports native (JNI) method calls."

**Senior Signal:** "The non-obvious point is Metaspace. Most candidates know Heap and Stack. The interesting failure modes are: (1) Metaspace OOM in long-running servers due to ClassLoader leaks — not fixable with heap tuning, requires diagnosing which loader is leaking. (2) Code Cache saturation causing JIT deoptimization — you see this as a sudden throughput drop under production load after a warm-up period."

```
JVM Memory Layout
─────────────────────────────────────────────────────
 Thread 1 Stack  │  Thread 2 Stack  │  Thread N Stack
─────────────────────────────────────────────────────
              Heap (Shared)
  ┌─────────────────────────────────────────┐
  │  Young Gen: Eden | S0 | S1              │
  │  Old Gen (Tenured)                      │
  └─────────────────────────────────────────┘
─────────────────────────────────────────────────────
  Metaspace (Native Memory — class metadata)
  Code Cache (JIT-compiled native code)
─────────────────────────────────────────────────────
```

---

### Q2. Explain Garbage Collection. How does G1GC work, and how would you tune it for a high-throughput payment service?

**The answer:**

"There are two main GC algorithms worth knowing deeply for Senior interviews: G1GC (default since Java 9) and ZGC (low-latency, Java 15+).

**G1GC (Garbage First Garbage Collector):**
G1 divides the heap into equal-sized regions (typically 1–32 MB each). Regions are dynamically assigned as Eden, Survivor, Old, or Humongous (for large objects).

**G1 GC Phases:**

1. **Young GC** — triggered when Eden fills up. Stop-the-world. Copies live objects from Eden to Survivor regions, promotes objects that have survived enough collections to Old. Very fast — typically < 50ms.

2. **Concurrent Marking** — runs concurrently with the application. Marks all reachable objects from GC roots. Three phases: Initial Mark (STW, very brief), Concurrent Mark (parallel with app threads), Remark (STW, finalizes marking).

3. **Mixed GC** — after concurrent marking, G1 knows which Old regions have the most garbage. It selects the most 'garbage-first' regions and collects them alongside the next Young GC. This is the key insight of G1 — you don't collect all of Old at once; you pick the profitable regions.

4. **Full GC** — last resort. Stop-the-world, compacts entire heap. G1 tries hard to avoid this. If you're seeing Full GCs, your application is generating garbage faster than G1 can collect.

**Tuning for a payment service:**

```bash
# Target max pause time — G1 tunes itself toward this goal
-XX:MaxGCPauseMillis=200

# Heap sizes — avoid dynamic resizing at runtime
-Xms4g -Xmx4g

# G1 region size — increase for large object workloads
-XX:G1HeapRegionSize=16m

# Concurrent GC threads — more = faster marking but more CPU contention
-XX:ConcGCThreads=4

# Reserve heap space to avoid Full GC under sudden allocation bursts
-XX:G1ReservePercent=20

# GC logging for production diagnosis
-Xlog:gc*:file=/var/log/app/gc.log:time,level,tags:filecount=5,filesize=20m
```

**What to watch in production:**
- `gc.pause` histogram — if 99th percentile > 500ms, investigate
- `jvm.gc.memory.allocated` vs `jvm.gc.memory.promoted` ratio — high promotion rate → Old Gen pressure
- Full GC count — should be 0 in steady state; any Full GC is an incident

**ZGC (Java 15+):** Concurrent, region-based, sub-millisecond pauses. Pause time does not scale with heap size — 10ms max even on 100GB heap. Trade-off: higher throughput overhead (~15% more CPU). Use ZGC when latency SLAs are strict (payment APIs), stick with G1 when you need raw throughput (batch processors)."

---

### Q3. Explain the String Pool. Why is `new String("abc")` different from `"abc"`?

**The answer:**

"The String Pool (interned string pool) is a special area in the Heap (since Java 7 — it was in PermGen before) that caches String literals to avoid duplicate allocations.

**`"abc"` (string literal):**
When the JVM loads a class, it processes the constant pool. Any string literal encountered gets interned — looked up in the String Pool, and if not present, added. Every reference to `"abc"` anywhere in the codebase points to the same String object in the pool.

**`new String("abc")`:**
Always creates a new String object in the regular Heap, bypassing the pool. So `new String("abc") == new String("abc")` is `false` — two different object references.

```java
String a = "abc";          // pool reference
String b = "abc";          // same pool reference
String c = new String("abc"); // new heap object

System.out.println(a == b);           // true  — same pool object
System.out.println(a == c);           // false — different objects
System.out.println(a.equals(c));      // true  — same content
System.out.println(a == c.intern());  // true  — intern() returns pool reference
```

**`String.intern()`:** Checks if the string exists in the pool; if yes, returns the pool reference; if no, adds it and returns the reference. Calling `intern()` on a heap string effectively moves it to the pool (or returns the existing pool entry).

**When this matters in practice:**
1. **String comparison bugs:** Using `==` instead of `.equals()` is a classic bug. `==` compares references; `.equals()` compares content.
2. **Memory with large datasets:** If you're storing millions of String records (e.g., enum-like status codes), interning them can save significant heap.
3. **StringBuilder:** Use `StringBuilder` for string concatenation in loops — string concatenation with `+` inside a loop creates O(n²) intermediate objects."

---

### Q4. Explain the ClassLoader hierarchy and the delegation model.

**The answer:**

"Java uses a hierarchical ClassLoader model with parent delegation, which is central to how the JVM ensures class uniqueness and provides isolation.

**Three built-in ClassLoaders:**

1. **Bootstrap ClassLoader** — loads `java.*` classes from the JDK's rt.jar (Java 8) or the module path (Java 9+). Implemented in native code, not a Java class. Parent of everything — has no parent itself.

2. **Extension ClassLoader** (Platform ClassLoader in Java 9+) — loads classes from `jre/lib/ext`. Parent: Bootstrap.

3. **Application ClassLoader** (System ClassLoader) — loads classes from the application's classpath (`-cp` or `CLASSPATH`). Parent: Extension. This is what loads your code.

**The Parent Delegation Model:**
When a ClassLoader receives a load request for a class:
1. First, delegate to the parent ClassLoader.
2. If the parent can load it, use that.
3. Only if the parent cannot find it, attempt to load it yourself.

```
load("com.myapp.PaymentService")
    → AppClassLoader → delegates to ExtClassLoader
        → ExtClassLoader → delegates to BootstrapClassLoader
            → BootstrapClassLoader: not found (it's not a java.* class)
        → ExtClassLoader: not found
    → AppClassLoader: found in classpath → loads it
```

**Why this matters:**
- Prevents malicious code from replacing `java.lang.String` — even if you put a fake `java.lang.String` in your classpath, the Bootstrap ClassLoader loads the real one first.
- Ensures class identity: `com.myapp.PaymentService` loaded by AppClassLoader is the same class everywhere.

**ClassLoader leaks (common production issue):**
In app servers (Tomcat, JBoss), each web app gets its own ClassLoader. On undeploy/redeploy, if any thread or static reference holds a reference to an old ClassLoader, it can't be GC'd — Metaspace grows unboundedly. Symptoms: Metaspace OOM after several redeployments.

**Custom ClassLoaders:**
Used for plugin systems, hot-reload, sandboxing. You override `findClass()` to load bytecode from custom sources (DB, network, encrypted jars).

```java
public class PluginClassLoader extends ClassLoader {
    private final Path pluginJar;

    public PluginClassLoader(Path pluginJar, ClassLoader parent) {
        super(parent);  // parent delegation wired in
        this.pluginJar = pluginJar;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classBytes = loadBytesFromJar(name);
        return defineClass(name, classBytes, 0, classBytes.length);
    }
}
```

**Your Project Angle:** "In ThingsBoard, we had to ensure tenant-specific customizations were loaded in isolation. Understanding ClassLoader delegation was critical when we hit a Metaspace OOM during development — the issue was a static reference in a Spring bean holding onto an old ApplicationContext, preventing its ClassLoader from being collected."

---

## PART 2 — JAVA TYPE SYSTEM & FUNCTIONAL FEATURES

---

### Q5. Explain Java Generics, type erasure, and the wildcard rules (PECS).

**The answer:**

"Java Generics provide compile-time type safety through parameterized types. The critical implementation detail is **type erasure**: generics exist only at compile time — after compilation, all type parameters are replaced with their bounds (usually `Object`). The bytecode has no generic type information.

**Type Erasure consequences:**
```java
List<String> strings = new ArrayList<>();
List<Integer> integers = new ArrayList<>();
// At runtime, both are just List — you cannot do:
// strings.getClass() != integers.getClass() // FALSE — same class at runtime

// Cannot create generic arrays:
// T[] arr = new T[10]; // compile error

// Cannot use instanceof with generics:
// if (obj instanceof List<String>) // compile error
```

**Bounded wildcards — the PECS rule:**

PECS = **Producer Extends, Consumer Super**

- `? extends T` — the collection is a **Producer** of T. You can read T from it, but you cannot add to it (because you don't know the exact subtype).
- `? super T` — the collection is a **Consumer** of T. You can add T to it, but reading gives you `Object` (because you don't know the exact supertype).

```java
// PRODUCER — read from source, extends
public static <T> void copy(List<? extends T> source, List<T> dest) {
    for (T item : source) {  // can READ
        dest.add(item);
    }
    // source.add(something); // COMPILE ERROR — can't add to ? extends
}

// CONSUMER — write to destination, super  
public static <T> void fill(List<? super T> dest, T value, int count) {
    for (int i = 0; i < count; i++) {
        dest.add(value);  // can ADD
    }
    // T item = dest.get(0); // COMPILE ERROR — reading gives Object, not T
}

// Practical example
List<Integer> ints = List.of(1, 2, 3);
List<Number> numbers = new ArrayList<>();
copy(ints, numbers);   // Integer extends Number — valid producer
fill(numbers, 42, 3);  // Integer super? No — Number super Integer — valid consumer
```

**Generic methods vs generic classes:**
```java
// Generic class — type parameter defined at class level
public class Pair<A, B> {
    private final A first;
    private final B second;
    public Pair(A first, B second) { this.first = first; this.second = second; }
}

// Generic method — type parameter defined at method level
public <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
```

**Senior Signal:** "Type erasure is why `List<String>` and `List<Integer>` share a single `.class` file. This is different from C++ templates which generate separate code for each type. The trade-off: Java generics have no runtime overhead for type checking (it's all compile time) but you can't specialize generic code for primitives — that's why `List<int>` doesn't exist, only `List<Integer>`. Project Valhalla is addressing this with value types and specialized generics."

---

### Q6. Explain Java Functional Interfaces and build a retry utility using `Supplier<T>`.

**The answer:**

"A Functional Interface has exactly one abstract method. The `@FunctionalInterface` annotation enforces this at compile time. Lambda expressions and method references are syntactic sugar for anonymous implementations of functional interfaces.

**Core functional interfaces:**

| Interface | Signature | Use Case |
|-----------|-----------|----------|
| `Function<T,R>` | `R apply(T t)` | Transform input to output |
| `Supplier<T>` | `T get()` | Produce a value (lazy factory) |
| `Consumer<T>` | `void accept(T t)` | Side effect on input |
| `Predicate<T>` | `boolean test(T t)` | Test a condition |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Two inputs, one output |
| `UnaryOperator<T>` | `T apply(T t)` | Transform same type |
| `Runnable` | `void run()` | No input, no output |

**Composing functions:**
```java
Function<String, String> trim = String::trim;
Function<String, String> uppercase = String::toUpperCase;
Function<String, Integer> length = String::length;

// andThen: f.andThen(g) = g(f(x))
Function<String, String> trimAndUpper = trim.andThen(uppercase);
// compose: f.compose(g) = f(g(x))
Function<String, Integer> trimAndLength = length.compose(trim);

System.out.println(trimAndUpper.apply("  hello  ")); // "HELLO"
System.out.println(trimAndLength.apply("  hello  ")); // 5
```

**Production Retry Executor using `Supplier<T>`:**

```java
public class RetryExecutor {

    public static <T> T executeWithRetry(
            Supplier<T> operation,
            int maxAttempts,
            long initialDelayMs,
            Class<? extends Exception>... retryableExceptions) {

        Exception lastException = null;
        long delay = initialDelayMs;

        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                return operation.get();  // execute the supplied operation
            } catch (Exception e) {
                if (!isRetryable(e, retryableExceptions)) {
                    throw new RuntimeException("Non-retryable exception on attempt " + attempt, e);
                }
                lastException = e;
                if (attempt < maxAttempts) {
                    log.warn("Attempt {}/{} failed: {}. Retrying in {}ms",
                             attempt, maxAttempts, e.getMessage(), delay);
                    sleepUninterruptibly(delay);
                    delay = delay * 2;  // exponential backoff
                }
            }
        }
        throw new RuntimeException("All " + maxAttempts + " attempts failed", lastException);
    }

    private static boolean isRetryable(Exception e, Class<? extends Exception>[] retryableExceptions) {
        for (Class<? extends Exception> retryable : retryableExceptions) {
            if (retryable.isAssignableFrom(e.getClass())) return true;
        }
        return false;
    }

    private static void sleepUninterruptibly(long millis) {
        try { Thread.sleep(millis); }
        catch (InterruptedException ie) { Thread.currentThread().interrupt(); }
    }
}

// Usage — call Razorpay API with retry
Payment payment = RetryExecutor.executeWithRetry(
    () -> razorpayClient.payments().fetch(paymentId),  // Supplier<Payment>
    3,     // max attempts
    500L,  // initial delay 500ms, then 1000ms, then 2000ms
    RazorpayApiException.class, ConnectException.class
);
```

**Why `Supplier<T>` fits here perfectly:** A `Supplier<T>` wraps a value-producing computation without executing it immediately. This is exactly what retry needs — you want to describe the operation once and execute it multiple times. Compare to `Callable<T>` which is the same but checked (`throws Exception`) — `Supplier<T>` requires wrapping checked exceptions."

---

## PART 3 — SPRING CORE INTERNALS

---

### Q7. What happens when a Spring Boot application starts? Walk through the entire startup sequence.

**The answer:**

"I'll walk through exactly what happens from `main()` to the application being ready to serve requests.

```
SpringApplication.run(MyApp.class, args)
    │
    ├── Create SpringApplication instance
    │     ├── Determine application type (SERVLET / REACTIVE / NONE)
    │     ├── Load ApplicationContextInitializers from spring.factories
    │     └── Load ApplicationListeners from spring.factories
    │
    ├── Run listeners: ApplicationStartingEvent
    │
    ├── Prepare Environment
    │     ├── Create ConfigurableEnvironment
    │     ├── Load property sources (application.properties, env vars, command line)
    │     └── Publish EnvironmentPreparedEvent
    │
    ├── Create ApplicationContext
    │     └── (AnnotationConfigServletWebServerApplicationContext for web apps)
    │
    ├── Prepare Context
    │     ├── Apply ApplicationContextInitializers
    │     ├── Register primary source (your @SpringBootApplication class)
    │     └── Load bean definitions via @ComponentScan, @Import, @Bean
    │
    ├── refreshContext() ← THE CORE STEP
    │     ├── invokeBeanFactoryPostProcessors()
    │     │     └── ConfigurationClassPostProcessor runs @ComponentScan
    │     ├── registerBeanPostProcessors()
    │     ├── initMessageSource()
    │     ├── initApplicationEventMulticaster()
    │     ├── onRefresh() ← creates embedded Tomcat/Undertow/Jetty
    │     ├── registerListeners()
    │     └── finishBeanFactoryInitialization() ← instantiates ALL singleton beans
    │
    ├── ApplicationStartedEvent
    ├── Call ApplicationRunners and CommandLineRunners
    └── ApplicationReadyEvent ← app is live
```

**`finishBeanFactoryInitialization()` is the heavyweight step.** This is where every singleton bean is instantiated (unless `@Lazy`). If your startup is slow, this is where to profile.

**Auto-configuration integration:**
During `invokeBeanFactoryPostProcessors()`, `ConfigurationClassPostProcessor` processes all `@Configuration` classes, including auto-configurations loaded from `META-INF/spring.factories` (Spring Boot 2.x) or `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 3.x). Each auto-configuration class is evaluated against its `@ConditionalOn*` conditions — only configurations whose conditions pass contribute beans.

**Your Project Angle:** "When we upgraded ThingsBoard to Java 17 and Spring Boot 3.x, understanding this startup sequence was essential. The auto-configuration loading mechanism changed between 2.x and 3.x — `spring.factories` was deprecated for a new imports file format. Tracking down which auto-configurations were failing helped us fix the startup errors."

---

### Q8. Explain the Spring Bean lifecycle. What is a `BeanPostProcessor`?

**The answer:**

"The Spring Bean lifecycle has several well-defined phases:

```
1. DEFINITION    — BeanDefinition loaded (class, scope, dependencies, metadata)
2. INSTANTIATION — Constructor called (Spring uses reflection)
3. POPULATION    — @Autowired dependencies injected (field/setter/constructor)
4. INITIALIZATION:
     a. @PostConstruct method called
     b. InitializingBean.afterPropertiesSet() (if implemented)
     c. @Bean(initMethod="...") custom init method
5. IN USE         — Bean serves requests
6. DESTRUCTION:
     a. @PreDestroy method called
     b. DisposableBean.destroy() (if implemented)
     c. @Bean(destroyMethod="...") custom destroy method
```

**BeanPostProcessor:**
A `BeanPostProcessor` intercepts beans after instantiation and dependency injection but before they're put into service. It has two hook methods:

```java
public interface BeanPostProcessor {
    // Called before @PostConstruct — before initialization
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    // Called after @PostConstruct — after initialization, before InUse
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

The `postProcessAfterInitialization` method can **replace the bean with a proxy**. This is exactly how AOP works: `AnnotationAwareAspectJAutoProxyCreator` (a BeanPostProcessor) wraps beans with `@Transactional`, `@Aspect`, `@Cacheable`, etc. in a proxy. The proxy, not the original bean, goes into the context.

**Key BeanPostProcessors you must know:**

| BeanPostProcessor | What it does |
|-------------------|--------------|
| `AutowiredAnnotationBeanPostProcessor` | Processes `@Autowired`, `@Value`, `@Inject` |
| `CommonAnnotationBeanPostProcessor` | Processes `@PostConstruct`, `@PreDestroy`, `@Resource` |
| `AnnotationAwareAspectJAutoProxyCreator` | Wraps beans in AOP proxies for `@Transactional`, `@Async`, `@Cacheable` |
| `PersistenceAnnotationBeanPostProcessor` | Processes `@PersistenceContext`, `@PersistenceUnit` |

**Custom BeanPostProcessor example:**
```java
@Component
public class AuditLoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean.getClass().isAnnotationPresent(AuditLogged.class)) {
            log.info("Bean {} is audit-logged — wrapping with proxy", beanName);
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                new AuditLoggingHandler(bean)
            );
        }
        return bean;  // return original bean if no wrapping needed
    }
}
```

---

### Q9. Explain Spring AOP. Why does `@Transactional` fail on self-invocation? How do you fix it?

**The answer:**

"Spring AOP works by creating proxies. When you mark a bean with `@Transactional` (or any AOP advice), Spring's `AnnotationAwareAspectJAutoProxyCreator` (a BeanPostProcessor) replaces the bean in the context with a proxy. All calls to the bean go through the proxy, which applies the advice (transaction management, caching, etc.) before/after delegating to the real object.

**JDK Dynamic Proxy vs CGLIB:**

| Aspect | JDK Dynamic Proxy | CGLIB |
|--------|------------------|-------|
| Mechanism | Implements the target's interfaces | Subclasses the target class |
| Requirement | Target must implement at least one interface | No interface required |
| Default in Spring | Yes, if interface available | Yes, if no interface (`proxyTargetClass=true`) |
| `final` methods/classes | Cannot proxy | Cannot proxy (cannot subclass) |

**Self-invocation problem:**
```java
@Service
public class OrderService {

    @Transactional  // marks the method on the PROXY
    public void processOrder(Order order) {
        // ... transaction-aware code
    }

    public void processOrders(List<Order> orders) {
        for (Order o : orders) {
            this.processOrder(o);  // DIRECT CALL — bypasses proxy!
            // 'this' refers to the real object, not the proxy
            // @Transactional has NO EFFECT here
        }
    }
}
```

When `processOrders()` calls `this.processOrder()`, it's a direct Java method call on the underlying object. The proxy is bypassed entirely. The transaction never starts.

**Three solutions:**

**Option 1: Self-injection (inject the proxy into itself)**
```java
@Service
public class OrderService {
    
    @Autowired
    private OrderService self;  // Spring injects the proxy, not 'this'

    public void processOrders(List<Order> orders) {
        for (Order o : orders) {
            self.processOrder(o);  // goes through the proxy — @Transactional works
        }
    }

    @Transactional
    public void processOrder(Order order) { /* ... */ }
}
```

**Option 2: Get proxy from ApplicationContext**
```java
@Service
public class OrderService implements ApplicationContextAware {
    
    private ApplicationContext context;

    public void processOrders(List<Order> orders) {
        OrderService proxy = context.getBean(OrderService.class);
        orders.forEach(proxy::processOrder);
    }

    @Transactional
    public void processOrder(Order order) { /* ... */ }
}
```

**Option 3: Extract into separate bean (cleanest)**
```java
@Service
public class OrderProcessor {
    @Transactional
    public void processOrder(Order order) { /* ... */ }
}

@Service
public class OrderService {
    @Autowired
    private OrderProcessor processor;  // separate bean = separate proxy

    public void processOrders(List<Order> orders) {
        orders.forEach(processor::processOrder);  // always goes through proxy
    }
}
```

**Senior Signal:** "Option 3 is the correct architectural answer. Self-injection (Option 1) is a code smell — it implies a violation of the Single Responsibility Principle. If a class needs a proxy to itself, it's probably doing too much. The real fix is to extract the transactional logic into its own class, which also makes testing much easier."

---

### Q10. How does Spring Boot Auto-configuration work? How would you write a custom starter?

**The answer:**

"Auto-configuration is Spring Boot's mechanism for applying configuration automatically based on what's on the classpath and what beans are already defined.

**The mechanism:**

1. `@EnableAutoConfiguration` (part of `@SpringBootApplication`) triggers `AutoConfigurationImportSelector`.
2. `AutoConfigurationImportSelector` reads the candidate auto-configuration class names from:
   - `META-INF/spring.factories` (Spring Boot 2.x): `org.springframework.boot.autoconfigure.EnableAutoConfiguration=...`
   - `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 3.x)
3. Each candidate class is evaluated against its `@ConditionalOn*` conditions. Conditions determine whether the configuration applies.
4. Configurations that pass all conditions contribute their `@Bean` definitions.

**Key `@Conditional` annotations:**

| Annotation | Condition |
|------------|-----------|
| `@ConditionalOnClass(Foo.class)` | Only if `Foo` is on the classpath |
| `@ConditionalOnMissingBean(Bar.class)` | Only if no `Bar` bean is already defined |
| `@ConditionalOnProperty("app.feature.enabled=true")` | Only if the property is set |
| `@ConditionalOnBean(DataSource.class)` | Only if a `DataSource` bean exists |
| `@ConditionalOnWebApplication` | Only in a web application context |

**Writing a custom starter (e.g., `audit-log-spring-boot-starter`):**

**Structure:**
```
audit-log-spring-boot-starter/
├── pom.xml
└── src/main/
    ├── java/com/myco/audit/
    │   ├── AuditLogAutoConfiguration.java
    │   ├── AuditLogProperties.java
    │   └── AuditLogService.java
    └── resources/META-INF/spring/
        └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

```java
@ConfigurationProperties(prefix = "audit")
public class AuditLogProperties {
    private boolean enabled = true;
    private String destination = "database";
    // getters/setters
}

@Configuration
@ConditionalOnClass(AuditLogService.class)
@ConditionalOnProperty(prefix = "audit", name = "enabled", havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(AuditLogProperties.class)
public class AuditLogAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean  // don't override if user defined their own
    public AuditLogService auditLogService(AuditLogProperties props) {
        return new AuditLogService(props.getDestination());
    }
}
```

```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.myco.audit.AuditLogAutoConfiguration
```

**The consuming app:** Just adds `audit-log-spring-boot-starter` to its `pom.xml`. The `AuditLogService` bean is available without any configuration. The app can customize via `audit.destination=kafka` in `application.properties`, or override entirely by defining their own `AuditLogService` bean (`@ConditionalOnMissingBean` prevents the starter's bean from being created)."

---

## PART 4 — SPRING SECURITY & JPA

---

### Q11. Walk me through the Spring Security filter chain. How did you customize it for ThingsBoard multi-tenant auth?

**The answer:**

"Spring Security integrates with the Servlet container through a `DelegatingFilterProxy`. Here's the full chain:

```
HTTP Request
    │
    ▼
DelegatingFilterProxy  (registered with Servlet container)
    │  (delegates to Spring-managed bean)
    ▼
FilterChainProxy  (the real Spring Security entry point)
    │  (holds a list of SecurityFilterChain instances)
    ▼
SecurityFilterChain (matches /api/**)
    │
    ├── SecurityContextPersistenceFilter  (restore/save SecurityContext)
    ├── UsernamePasswordAuthenticationFilter  (form login)
    ├── BasicAuthenticationFilter  (HTTP Basic)
    ├── BearerTokenAuthenticationFilter / JwtAuthenticationFilter (JWT)
    ├── ExceptionTranslationFilter  (converts AuthenticationException → 401/403)
    └── FilterSecurityInterceptor  (authorization — access control decisions)
```

**Custom JWT filter for multi-tenant ThingsBoard:**

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenValidator tokenValidator;
    private final TenantContextHolder tenantContextHolder;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String token = extractBearerToken(request);
        if (token != null) {
            try {
                JwtClaims claims = tokenValidator.validate(token);
                // Extract tenant context from JWT claims
                UUID tenantId = UUID.fromString(claims.get("tenantId").toString());
                tenantContextHolder.setCurrentTenant(tenantId);

                // Set authentication in SecurityContext
                TenantAwareAuthenticationToken auth = new TenantAwareAuthenticationToken(
                    claims.getSubject(), tenantId, claims.getRoles()
                );
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (JwtException e) {
                // Don't set authentication — let ExceptionTranslationFilter handle 401
                log.debug("Invalid JWT: {}", e.getMessage());
            }
        }

        try {
            filterChain.doFilter(request, response);
        } finally {
            tenantContextHolder.clear();  // always clear ThreadLocal after request
        }
    }

    private String extractBearerToken(HttpServletRequest request) {
        String header = request.getHeader(HttpHeaders.AUTHORIZATION);
        if (header != null && header.startsWith("Bearer ")) {
            return header.substring(7);
        }
        return null;
    }
}
```

**Multi-tenant SecurityFilterChain configuration:**
```java
@Configuration
@EnableWebSecurity
public class MultiTenantSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http,
                                                    JwtAuthenticationFilter jwtFilter) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("TENANT_ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

**Hibernate @Filter for tenant data isolation:**
```java
@Entity
@FilterDef(name = "tenantFilter",
           parameters = @ParamDef(name = "tenantId", type = UUID.class))
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
public class Device {
    @Id UUID id;
    @Column UUID tenantId;
    // ...
}

// Activated per-request in a filter/interceptor:
Session session = entityManager.unwrap(Session.class);
session.enableFilter("tenantFilter").setParameter("tenantId", currentTenantId);
```

---

### Q12. What is the N+1 problem in JPA? How do you diagnose and fix it?

**The answer:**

"The N+1 problem is when fetching N parent entities triggers N additional queries to fetch their associated children — total N+1 queries instead of 1 or 2.

**Classic example:**
```java
// Entity
@Entity
public class Order {
    @Id Long id;
    @OneToMany(fetch = FetchType.LAZY)  // lazy by default
    List<OrderItem> items;
}

// Service
List<Order> orders = orderRepository.findAll();  // 1 query: SELECT * FROM orders
for (Order order : orders) {
    order.getItems().size();  // N queries: SELECT * FROM order_items WHERE order_id = ?
}
// Total: 1 + N queries
```

**Diagnosis:**
Add to `application.properties`:
```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql=TRACE
```
Or use `Hibernate Statistics` / `p6spy` for production-safe query counting.

**Fix 1 — JOIN FETCH (JPQL):**
```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findWithItemsByStatus(@Param("status") OrderStatus status);
// Result: 1 query with JOIN — all data loaded
```

**Fix 2 — @EntityGraph:**
```java
@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findByStatus(OrderStatus status);
// Spring Data JPA generates JOIN FETCH automatically
```

**Fix 3 — DTO Projection (most efficient for read-heavy APIs):**
```java
public interface OrderSummary {
    Long getId();
    String getStatus();
    int getItemCount();
}

@Query("SELECT o.id as id, o.status as status, SIZE(o.items) as itemCount FROM Order o")
List<OrderSummary> findOrderSummaries();
// Spring Data maps result directly to interface — no N+1, minimal data transfer
```

**The Cartesian Product trap with multiple collections:**
```java
// DON'T do this — causes Cartesian product
@Query("SELECT o FROM Order o JOIN FETCH o.items JOIN FETCH o.payments")
// If order has 3 items and 2 payments: 3×2 = 6 rows returned per order
```
**Fix:** Use `@BatchSize` or separate queries for the second collection.

---

### Q13. Explain `@Transactional` propagation and isolation levels.

**The answer:**

"These are two orthogonal dimensions of transaction behavior.

**Propagation** — what happens when a transactional method is called from within another transaction:

| Propagation | Behavior |
|-------------|----------|
| `REQUIRED` (default) | Join existing TX if present; start new if not |
| `REQUIRES_NEW` | Always start a new TX; suspend existing if any |
| `NESTED` | Create a savepoint within existing TX; on failure, roll back to savepoint only |
| `SUPPORTS` | Join if TX exists; run non-transactionally if not |
| `NOT_SUPPORTED` | Suspend existing TX; run non-transactionally |
| `MANDATORY` | Must join existing TX; throw exception if none |
| `NEVER` | Must run without TX; throw exception if one exists |

**Practical example — audit log that must always commit:**
```java
@Service
public class PaymentService {
    
    @Transactional  // REQUIRED — starts TX
    public void processPayment(Payment payment) {
        paymentRepository.save(payment);
        try {
            externalGatewayService.charge(payment);  // might throw
        } catch (GatewayException e) {
            throw new PaymentException(e);
            // TX rolls back — payment not saved
        } finally {
            auditService.log(payment);  // MUST record even if payment fails
        }
    }
}

@Service
public class AuditService {
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)  // always its own TX
    public void log(Payment payment) {
        // This TX commits independently — even if outer TX rolls back
        auditRepository.save(new AuditEntry(payment));
    }
}
```

**Isolation Levels** — control visibility of uncommitted data across concurrent transactions:

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Use Case |
|-------|-----------|---------------------|--------------|----------|
| `READ_UNCOMMITTED` | Yes | Yes | Yes | Never in production |
| `READ_COMMITTED` | No | Yes | Yes | Default in most DBs |
| `REPEATABLE_READ` | No | No | Yes | MySQL default |
| `SERIALIZABLE` | No | No | No | Max isolation, lowest throughput |

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void finalizeTransfer(UUID accountId, BigDecimal amount) {
    // Full serialization — prevents all concurrency anomalies
    // Appropriate for financial settlement operations
}

@Transactional(isolation = Isolation.READ_COMMITTED)
public List<Payment> findPendingPayments() {
    // Default — reads only committed data
    // May see different results if re-queried in same TX (non-repeatable read)
}
```

**Senior Signal:** "In practice, never change the isolation level from `READ_COMMITTED` unless you have a specific, measured concurrency problem. Higher isolation = more locking = less throughput. The right tool for concurrent financial operations is usually pessimistic locking (`SELECT FOR UPDATE`) or optimistic locking (`@Version`) applied to specific rows, not blanket `SERIALIZABLE` isolation."

---

## QUICK-REFERENCE — Senior Filter Questions

These are the short, sharp questions that interviewers use to screen quickly. Know each of these cold:

**Q: What's the difference between `BeanFactory` and `ApplicationContext`?**
`BeanFactory` is the basic IoC container — lazy initialization, no AOP, no events. `ApplicationContext` extends it with eager singleton initialization, AOP support, event publishing, internationalization, and integration with web scopes. You always use `ApplicationContext` in Spring apps; `BeanFactory` is only mentioned in theory.

**Q: What happens when you have two beans of the same type? (`NoUniqueBeanDefinitionException`)**
Spring can't decide which one to inject. Fix: mark one as `@Primary`, or use `@Qualifier("beanName")` at the injection point, or use `@Autowired Map<String, PaymentGateway>` to inject all of them keyed by bean name.

**Q: Can you use field injection `@Autowired` on a private field?**
Yes. Spring uses reflection (`field.setAccessible(true)`). But it's bad practice: makes beans impossible to instantiate in unit tests without a Spring context, hides dependencies, and prevents immutability. Constructor injection is the correct approach — explicit, testable, enforces required dependencies.

**Q: What is `@Lazy` and when should you use it?**
`@Lazy` defers bean instantiation until first use, instead of at startup. Use it to: (1) break circular dependencies as a last resort, (2) speed up startup in tests where you only need a subset of beans, (3) initialize expensive optional beans on-demand. Don't use it as a performance shortcut — it just moves startup cost to first request.

**Q: How do you resolve circular dependencies in Spring?**
Constructor injection with circular dependencies is unresolvable — Spring will throw `BeanCurrentlyInCreationException`. Solutions: (1) Refactor — usually the circular dependency indicates a design problem (two services doing too much). (2) Use setter/field injection for one side — Spring can inject via setter after construction. (3) `@Lazy` on one side. (4) Extract the shared dependency into a third bean.

**Q: What is `@Conditional` and where is it used in the framework?**
`@Conditional` is the extension point for conditional bean registration. The whole auto-configuration system is built on it — `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, etc. Custom conditions implement `Condition.matches()` and can inspect the environment, classpath, and existing bean definitions.

---

---

## PART 5 — JAVA STREAMS API & OPTIONAL

---

### Q14. Walk me through the Java Streams API. What are the most important operations?

**The answer:**

"The Streams API, introduced in Java 8, provides a declarative, functional pipeline for processing collections. A stream is a sequence of elements supporting sequential or parallel aggregate operations. Streams are lazy — intermediate operations don't execute until a terminal operation is called.

**Three categories of operations:**

```
Source → [Intermediate ops (lazy)] → Terminal op (triggers execution)
```

**Intermediate operations (return a new Stream — lazy):**

```java
List<Payment> payments = paymentRepository.findAll();

// filter — keep elements matching predicate
List<Payment> completed = payments.stream()
    .filter(p -> p.getStatus() == PaymentStatus.COMPLETED)
    .collect(Collectors.toList());

// map — transform each element
List<String> references = payments.stream()
    .map(Payment::getReference)       // method reference
    .collect(Collectors.toList());

// flatMap — flatten nested streams (one-to-many mapping)
List<OrderItem> allItems = orders.stream()
    .flatMap(order -> order.getItems().stream())  // Order → List<Item> → flattened
    .collect(Collectors.toList());

// distinct, sorted, limit, skip
List<String> topTenantIds = payments.stream()
    .map(Payment::getTenantId)
    .distinct()
    .sorted()
    .limit(10)
    .collect(Collectors.toList());

// peek — for debugging (doesn't change stream, side-effect only)
payments.stream()
    .peek(p -> log.debug("Processing payment: {}", p.getId()))
    .filter(p -> p.getAmount().compareTo(threshold) > 0)
    .collect(Collectors.toList());
```

**Terminal operations (trigger execution, consume the stream):**

```java
// collect — most versatile terminal op
List<Payment> list = stream.collect(Collectors.toList());
Set<String> set = stream.collect(Collectors.toSet());
String joined = stream.collect(Collectors.joining(", "));

// reduce — fold all elements into one
BigDecimal total = payments.stream()
    .map(Payment::getAmount)
    .reduce(BigDecimal.ZERO, BigDecimal::add);

// count, sum, min, max
long count = payments.stream().filter(p -> p.getStatus() == FAILED).count();
Optional<Payment> largest = payments.stream()
    .max(Comparator.comparing(Payment::getAmount));

// forEach — side effect terminal op
payments.stream().forEach(paymentService::process);

// anyMatch, allMatch, noneMatch — short-circuit
boolean hasFailed = payments.stream().anyMatch(p -> p.getStatus() == FAILED);
boolean allCompleted = payments.stream().allMatch(p -> p.getStatus() == COMPLETED);
```

**Key Collectors:**

```java
// groupingBy — partition into map by key
Map<PaymentStatus, List<Payment>> byStatus = payments.stream()
    .collect(Collectors.groupingBy(Payment::getStatus));

// groupingBy with downstream collector
Map<String, Long> countByTenant = payments.stream()
    .collect(Collectors.groupingBy(Payment::getTenantId, Collectors.counting()));

Map<String, BigDecimal> totalByPartner = payments.stream()
    .collect(Collectors.groupingBy(
        Payment::getPartnerId,
        Collectors.reducing(BigDecimal.ZERO, Payment::getAmount, BigDecimal::add)
    ));

// toMap — explicit key/value extraction
Map<UUID, Payment> paymentsById = payments.stream()
    .collect(Collectors.toMap(Payment::getId, p -> p));

// toMap with merge function (handle duplicate keys)
Map<String, BigDecimal> maxAmountByPartner = payments.stream()
    .collect(Collectors.toMap(
        Payment::getPartnerId,
        Payment::getAmount,
        BigDecimal::max  // merge: keep the larger amount if key collision
    ));

// partitioningBy — splits into true/false map
Map<Boolean, List<Payment>> partitioned = payments.stream()
    .collect(Collectors.partitioningBy(p -> p.getAmount().compareTo(threshold) > 0));
// partitioned.get(true) = above threshold
// partitioned.get(false) = at or below threshold
```

**Parallel Streams:**
```java
// Use for CPU-bound, stateless operations on large datasets
long count = hugeList.parallelStream()
    .filter(p -> heavyComputation(p))
    .count();

// DON'T use parallel streams for:
// 1. I/O operations (Kafka, DB) — threads block, defeats parallelism
// 2. Shared mutable state — race conditions
// 3. Small lists — thread coordination overhead costs more than gained
// 4. Operations with side effects (forEach with shared list)

// The underlying ForkJoinPool.commonPool() is shared — I/O-bound tasks
// can starve other parallel streams. Always measure before using.
```

**Senior Signal:** 'The most important interview point about streams is that they are lazy and single-use. Once a terminal operation is called, the stream is consumed — you cannot reuse it. If you need to iterate twice, collect to a List first. The second most important point is that parallel streams use `ForkJoinPool.commonPool()` — never use them for I/O-bound work without a custom pool.'"

---

### Q15. Explain `Optional`. When should you use it and when should you not?

**The answer:**

"`Optional<T>` is a container that may or may not hold a non-null value. It was introduced in Java 8 to replace `null` return values and reduce `NullPointerException` risk in method return types.

**Correct usage — as a return type for value-absent scenarios:**

```java
// Good — method that may not find a result
public Optional<Payment> findByReference(String reference) {
    return paymentRepository.findByReference(reference);
}

// Consuming Optional safely
Optional<Payment> payment = findByReference("PAY-123");

// Pattern 1 — orElse (always evaluates the default, even if value present)
Payment result = payment.orElse(new Payment());

// Pattern 2 — orElseGet (lazy evaluation — preferred for expensive defaults)
Payment result = payment.orElseGet(() -> createDefaultPayment());

// Pattern 3 — orElseThrow (most common in service layer)
Payment result = payment.orElseThrow(() ->
    new PaymentNotFoundException("PAY-123"));

// Pattern 4 — ifPresent (side effect only when value exists)
payment.ifPresent(p -> notificationService.sendReceipt(p));

// Pattern 5 — map / flatMap (transform if present)
Optional<String> partnerName = payment
    .map(Payment::getPartnerId)
    .flatMap(partnerService::findById)  // findById returns Optional<Partner>
    .map(Partner::getName);

// Pattern 6 — filter (narrow down)
Optional<Payment> largePay = payment
    .filter(p -> p.getAmount().compareTo(new BigDecimal("10000")) > 0);
```

**What NOT to do with Optional:**

```java
// NEVER use Optional as a field — it's not Serializable
public class Payment {
    private Optional<String> notes;  // WRONG — use String notes = null instead
}

// NEVER use Optional as a method parameter
public void process(Optional<Payment> payment) { }  // WRONG
// Callers can pass Optional.empty() OR null — you've made things worse

// NEVER call .get() without checking isPresent() first
Optional<Payment> opt = findByReference("X");
Payment p = opt.get();  // throws NoSuchElementException if empty — worse than NPE

// NEVER use Optional just to avoid null in intermediate logic
Optional<String> name = Optional.ofNullable(payment)
    .map(Payment::getTenantId);  // just check null directly in this case
```

**Senior Signal:** 'Optional is designed for method return types — specifically for cases where the absence of a value is a meaningful, expected outcome (not an error). The key distinction: if a result not being found is an error (you expected it to exist), throw an exception. If absence is a normal case (search that may find nothing), return Optional. Never use it as a field or parameter — that's not what the API designers intended.'"

---

## PART 6 — SPRING TESTING DEEP DIVE

---

### Q16. How do you test a Spring Boot application? Explain the testing pyramid and key annotations.

**The answer:**

"Testing a Spring Boot application follows the testing pyramid: many unit tests at the bottom (fast, no Spring context), fewer integration tests in the middle, and a small number of end-to-end tests at the top.

**Layer 1 — Pure unit tests (no Spring context, fastest):**

```java
// Service logic tested with Mockito — no @SpringBootTest needed
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock
    private PaymentRepository paymentRepository;

    @Mock
    private RazorpayGateway razorpayGateway;

    @InjectMocks
    private PaymentService paymentService;  // real instance, mocked deps injected

    @Test
    void shouldThrowWhenInsufficientCredit() {
        Partner partner = new Partner();
        partner.setAvailableCredit(new BigDecimal("100"));

        when(paymentRepository.findPartner(any())).thenReturn(Optional.of(partner));

        assertThrows(InsufficientCreditException.class,
            () -> paymentService.processPayment(new Payment(new BigDecimal("500")))
        );

        verify(paymentRepository, never()).save(any());  // ensure no save on failure
    }

    @Test
    void shouldCaptureArgumentOnSuccess() {
        // ArgumentCaptor — inspect what was actually passed to a mock
        ArgumentCaptor<Payment> captor = ArgumentCaptor.forClass(Payment.class);
        when(razorpayGateway.process(any())).thenReturn(PaymentResult.success("pay_123"));

        paymentService.processPayment(buildValidPayment());

        verify(paymentRepository).save(captor.capture());
        assertEquals(PaymentStatus.COMPLETED, captor.getValue().getStatus());
    }
}
```

**Layer 2 — Slice tests (partial Spring context, medium speed):**

Spring Boot provides test slices — they load only the layers relevant to what you're testing:

```java
// @WebMvcTest — loads ONLY web layer (controllers, filters, security config)
// No service beans, no repository beans
@WebMvcTest(PaymentController.class)
class PaymentControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean  // @MockBean = Mockito mock registered in Spring context
    private PaymentService paymentService;

    @Test
    void shouldReturn201OnValidPayment() throws Exception {
        when(paymentService.create(any())).thenReturn(buildPayment("PAY-001"));

        mockMvc.perform(post("/api/v1/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .header("Idempotency-Key", UUID.randomUUID().toString())
                .content("""
                    {"amount": 5000, "partnerId": "partner-1", "currency": "INR"}
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value("PAY-001"))
            .andExpect(header().exists("Location"));
    }

    @Test
    void shouldReturn400OnMissingAmount() throws Exception {
        mockMvc.perform(post("/api/v1/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"partnerId\": \"partner-1\"}"))  // missing amount
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors[0].field").value("amount"));
    }
}
```

```java
// @DataJpaTest — loads ONLY JPA layer (repositories, entity validation)
// Uses in-memory H2 by default (or TestContainers for real DB)
@DataJpaTest
class PaymentRepositoryTest {

    @Autowired
    private PaymentRepository paymentRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void shouldFindPendingPaymentsByTenant() {
        Payment p1 = entityManager.persist(buildPayment("t1", PENDING));
        Payment p2 = entityManager.persist(buildPayment("t1", COMPLETED));
        Payment p3 = entityManager.persist(buildPayment("t2", PENDING));
        entityManager.flush();

        List<Payment> result = paymentRepository.findByTenantIdAndStatus("t1", PENDING);

        assertThat(result).hasSize(1).containsExactly(p1);
    }
}
```

**Layer 3 — Integration tests with TestContainers (real DB/Redis/Kafka):**

TestContainers spins up real Docker containers per test class. This catches issues that H2 in-memory DB silently hides (PostgreSQL-specific SQL syntax, constraint behaviour, index usage).

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class PaymentIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("payments_test")
        .withUsername("test")
        .withPassword("test");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldProcessPaymentEndToEnd() {
        CreatePaymentRequest req = new CreatePaymentRequest(
            new BigDecimal("5000"), "partner-1", "INR"
        );
        ResponseEntity<Payment> response = restTemplate.postForEntity(
            "/api/v1/payments", req, Payment.class
        );

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getStatus()).isEqualTo(PaymentStatus.PROCESSING);
    }
}
```

**`@Mock` vs `@MockBean` — the critical difference:**

| | `@Mock` (Mockito) | `@MockBean` (Spring) |
|-|-------------------|----------------------|
| Context | No Spring context | Requires Spring context |
| Use with | `@ExtendWith(MockitoExtension.class)` | `@WebMvcTest`, `@SpringBootTest` |
| Effect | Creates Mockito mock | Creates mock AND replaces Spring bean |
| Speed | Very fast | Slower (context startup) |

**Testing `@Transactional` rollback:**
```java
@SpringBootTest
@Transactional  // rolls back after each test — no cleanup needed
class PaymentTransactionTest {

    @Test
    void shouldRollbackOnGatewayFailure() {
        // When gateway throws, payment should not be saved
        assertThrows(GatewayException.class, () ->
            paymentService.processWithFailingGateway(buildPayment())
        );
        // @Transactional on test rolls back automatically
        assertThat(paymentRepository.count()).isZero();
    }
}
```

**Senior Signal:** 'The most common mistake I see is overusing `@SpringBootTest` for everything — it loads the full context every time, making tests slow. The rule: test business logic with plain Mockito (`@ExtendWith(MockitoExtension.class)`), test controller validation with `@WebMvcTest`, test queries with `@DataJpaTest` + TestContainers, and reserve `@SpringBootTest` for true end-to-end flows. This keeps the test suite fast enough to run on every commit.'"

---

> **What's Next — Chapter 2:** Java Multithreading & Concurrency Deep Dive — synchronized, ReentrantLock, ThreadPools, CompletableFuture, Kafka delivery guarantees, Redis caching patterns, the full credit limit race condition story, plus advanced Kafka (DLQ, consumer rebalancing), advanced Redis (Sentinel vs Cluster, Streams), HikariCP tuning, and database migrations.

# Spring Core and Spring Boot — In-Depth Enterprise Guide

A practical guide for experienced Java developers, with concepts, production patterns, realistic enterprise cases, interview questions, and hands-on exercises.

> Version note: Examples use modern Spring Boot 3.x conventions and Java 17+. Spring Boot 3 uses `jakarta.*` APIs. Older Spring Boot 2 projects commonly use `javax.*`. Do not mix the namespaces. Check the Spring Boot, Spring Framework, Java, and dependency versions used by your project.

## 1. Spring ecosystem overview

### What Spring solves
Spring provides infrastructure for building Java applications: object creation and wiring, lifecycle management, configuration, transactions, data access integration, web applications, security integration, messaging, observability, and testing.

### Spring Framework vs Spring Boot
| Area | Spring Framework | Spring Boot |
|---|---|---|
| Main purpose | Core container and application frameworks | Opinionated way to build and run Spring applications |
| Configuration | Explicit configuration is common | Auto-configuration plus explicit overrides |
| Dependencies | Select and manage dependencies | Starter dependencies simplify dependency selection |
| Runtime | Can be deployed to an external server | Commonly runs with embedded server |
| Operations | Configure operational features yourself | Actuator and production integrations |
| Flexibility | Highly flexible | Uses conventions while retaining Spring flexibility |

Spring Boot does not replace Spring Framework. It builds on it.

### Main modules to recognize
- Spring Core / Beans: IoC container, dependency injection, bean lifecycle.
- Spring Context: application context, events, resources, environment.
- Spring AOP: cross-cutting concerns such as transactions and method interception.
- Spring Expression Language (SpEL).
- Spring JDBC and transaction management.
- Spring ORM integration.
- Spring MVC and WebFlux.
- Spring Test.
- Spring Boot: auto-configuration, starters, externalized configuration, Actuator.

## 2. IoC and Dependency Injection

### Inversion of Control (IoC)
Without IoC, application code constructs and manages its dependencies directly. With IoC, the Spring container creates objects, injects dependencies, and manages configured bean lifecycles.

```java
public class OrderService {
    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}
```

The service declares what it needs; the container supplies the repository implementation.

### Dependency Injection styles
1. Constructor injection — preferred for required dependencies.
2. Setter injection — useful for optional or reconfigurable dependencies.
3. Field injection — generally avoid; hides dependencies and complicates unit testing.

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;

    public OrderService(OrderRepository orderRepository,
                        PaymentGateway paymentGateway) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
    }
}
```

Benefits of constructor injection:
- Required dependencies are explicit.
- Fields can be `final`.
- Unit tests can instantiate the class without Spring.
- Missing dependencies fail during application startup.

### @Component, @Service, @Repository, @Controller
All are stereotype annotations that make classes eligible for component scanning.
- `@Component`: general-purpose Spring-managed component.
- `@Service`: service/business logic role.
- `@Repository`: persistence role; also participates in persistence exception translation.
- `@Controller`: MVC controller, usually returns views.
- `@RestController`: `@Controller` plus response-body semantics.

These annotations communicate intent; they do not create separate IoC containers.

### @Autowired, @Qualifier, @Primary
With a single constructor, Spring can inject its dependencies without `@Autowired`.

When multiple implementations exist:

```java
public interface NotificationSender {
    void send(String destination, String message);
}

@Component("emailSender")
class EmailNotificationSender implements NotificationSender {
    public void send(String destination, String message) { }
}

@Component("smsSender")
class SmsNotificationSender implements NotificationSender {
    public void send(String destination, String message) { }
}

@Service
class AlertService {
    private final NotificationSender sender;

    AlertService(@Qualifier("emailSender") NotificationSender sender) {
        this.sender = sender;
    }
}
```

Use `@Primary` to define a default candidate. Use `@Qualifier` when a specific implementation is required.

## 3. Bean configuration and lifecycle

### Defining beans
Component scanning:

```java
@Configuration
@ComponentScan("com.example")
class AppConfig { }
```

Java configuration:

```java
@Configuration
class PaymentConfig {
    @Bean
    PaymentGateway paymentGateway(PaymentClient client) {
        return new HttpPaymentGateway(client);
    }
}
```

Use `@Bean` for third-party classes, factory methods, or explicit construction. Prefer component scanning for application-owned components when appropriate.

### Bean lifecycle (simplified)
1. Instantiate bean.
2. Populate dependencies.
3. Apply aware callbacks and bean post-processors.
4. Run initialization callbacks.
5. Bean is available for use.
6. On context shutdown, run destruction callbacks for applicable scopes.

```java
@Component
class CacheWarmup {
    @PostConstruct
    void initialize() {
        // Validate configuration or warm a small local cache.
    }

    @PreDestroy
    void shutdown() {
        // Release resources owned by this bean.
    }
}
```

Avoid long, fragile network operations in `@PostConstruct`; they can make startup unreliable. Prefer explicit startup orchestration with clear failure and timeout behavior.

### Bean scopes
- `singleton` (default): one bean instance per Spring IoC container.
- `prototype`: a new instance each time the container is asked for one.
- `request`: one instance per HTTP request in a web application.
- `session`: one instance per HTTP session.
- `application`: one instance per ServletContext.
- `websocket`: one instance per WebSocket lifecycle.

Important: Spring singleton does not mean one instance per JVM. It means one instance per container. Singleton beans should generally be stateless or safely thread-safe. Do not store request-specific mutable data in singleton fields.

### Circular dependencies
A circular dependency exists when A requires B and B requires A. Constructor-based cycles fail during creation. Do not solve this by blindly switching to field injection or lazy injection. Refactor responsibilities, introduce a coordinator, or publish events where appropriate.

## 4. Spring AOP and proxy behavior

AOP separates cross-cutting behavior from business logic. Common uses include transactions, method security, metrics, auditing, and retry behavior.

Key terms:
- Aspect: module containing cross-cutting behavior.
- Advice: action to run (before, after, around, etc.).
- Join point: execution point where advice can apply.
- Pointcut: expression selecting join points.
- Proxy: object through which the intercepted call is made.

Spring AOP commonly uses JDK dynamic proxies for interface-based proxying or class-based proxies. Proxy-based advice has important limitations.

### Self-invocation trap
Calling an advised method from another method on the same object bypasses the proxy:

```java
@Service
class ImportService {
    public void importFile() {
        saveBatch(); // self-invocation: @Transactional may not be applied here
    }

    @Transactional
    public void saveBatch() {
        // database writes
    }
}
```

Move `saveBatch` to another Spring bean, or redesign the transaction boundary. Do not assume an annotation is active merely because it appears on a method.

### Custom aspect example
```java
@Aspect
@Component
class ExecutionLoggingAspect {
    @Around("within(com.example.application..*)")
    Object logExecution(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.nanoTime();
        try {
            return joinPoint.proceed();
        } finally {
            long elapsedMs = (System.nanoTime() - start) / 1_000_000;
            // Send structured timing data to a logger/metrics system.
        }
    }
}
```

Avoid logging secrets, personal data, payment details, or full request bodies.

## 5. Spring events and application context

Use application events for decoupling within one application process. They are not a durable message broker.

```java
public record ProductPublishedEvent(Long productId) {}

@Service
class ProductService {
    private final ApplicationEventPublisher publisher;

    ProductService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    void publish(Long productId) {
        // Persist product state first, according to the use case.
        publisher.publishEvent(new ProductPublishedEvent(productId));
    }
}

@Component
class ProductAuditListener {
    @EventListener
    void on(ProductPublishedEvent event) {
        // Record local audit activity.
    }
}
```

Use `@TransactionalEventListener(phase = AFTER_COMMIT)` when a listener should run only after a transaction commits. For reliable cross-service publication, use a durable broker and consider the transactional outbox pattern; an in-memory event is not a substitute for Kafka or another broker.

## 6. Spring Boot foundations

### @SpringBootApplication
Combines:
- `@Configuration`
- `@EnableAutoConfiguration`
- `@ComponentScan`

Place the main application class in a root package above the application's components so component scanning can discover them.

```java
@SpringBootApplication
public class CatalogApplication {
    public static void main(String[] args) {
        SpringApplication.run(CatalogApplication.class, args);
    }
}
```

### Auto-configuration
Spring Boot configures beans based on:
- Classes available on the classpath.
- Existing beans.
- Configuration properties.
- Conditional annotations.

Common conditions include:
- `@ConditionalOnClass`
- `@ConditionalOnMissingBean`
- `@ConditionalOnProperty`
- `@ConditionalOnBean`

Auto-configuration is not magic: inspect the condition evaluation report and application startup logs when a bean is unexpectedly present or absent.

### Starters
Starters provide curated dependency sets, for example:
- `spring-boot-starter-web`
- `spring-boot-starter-validation`
- `spring-boot-starter-data-jpa`
- `spring-boot-starter-security`
- `spring-boot-starter-actuator`
- `spring-boot-starter-test`

Use the Spring Boot dependency management/BOM. Avoid independently pinning Spring Framework modules unless there is a specific, tested reason.

## 7. Externalized configuration and profiles

Example `application.yml`:

```yaml
server:
  port: 8080

spring:
  application:
    name: catalog-service
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

catalog:
  import:
    max-batch-size: 500
```

Bind related settings with `@ConfigurationProperties`:

```java
@ConfigurationProperties(prefix = "catalog.import")
@Validated
public record ImportProperties(
        @Min(1) @Max(5000) int maxBatchSize) { }
```

Enable scanning with `@ConfigurationPropertiesScan` or register the properties class explicitly.

Profiles:
- `application-local.yml`
- `application-test.yml`
- `application-prod.yml`

Activate with `SPRING_PROFILES_ACTIVE=prod` or deployment configuration. Profiles select environment-specific configuration; they are not a secure secret store.

Configuration precedence can vary by property source. Treat environment variables and deployment-provided configuration as authoritative in production. Never commit passwords, API keys, signing keys, or private certificates.

## 8. Building REST APIs with Spring Boot

### Layered structure
A common structure:

```text
com.example.catalog
  CatalogApplication.java
  controller/
  service/
  repository/
  domain/
  dto/
  mapper/
  config/
  exception/
```

Typical request flow:

```text
HTTP request
  -> Controller
  -> Validation
  -> Service / use-case
  -> Repository
  -> Database
  -> DTO response
```

Keep controllers thin. Put business rules in application/domain services. Do not expose JPA entities as public API contracts.

### Controller and DTO example
```java
public record CreateProductRequest(
        @NotBlank String sku,
        @NotBlank String name,
        @NotNull Long categoryId) { }

public record ProductResponse(
        Long id, String sku, String name, String status) { }

@RestController
@RequestMapping("/api/v1/products")
class ProductController {
    private final ProductService service;

    ProductController(ProductService service) {
        this.service = service;
    }

    @PostMapping
    ResponseEntity<ProductResponse> create(
            @Valid @RequestBody CreateProductRequest request) {
        ProductResponse response = service.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

### Validation
Use Bean Validation annotations such as `@NotBlank`, `@NotNull`, `@Size`, `@Email`, `@Min`, and custom constraints. Validate at API boundaries, but enforce critical business invariants again in the service/domain and database.

### Global exception handling
```java
@RestControllerAdvice
class ApiExceptionHandler {
    @ExceptionHandler(ProductNotFoundException.class)
    ResponseEntity<ProblemDetail> handleNotFound(ProductNotFoundException ex) {
        ProblemDetail problem =
            ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Product not found");
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(problem);
    }
}
```

Use consistent error codes and safe messages. Do not return stack traces or internal SQL details to clients.

### API design checklist
- Use resource-oriented URLs and correct HTTP methods/status codes.
- Validate input and return consistent errors.
- Add pagination, sorting, and filtering for large result sets.
- Define idempotency behavior for retryable commands.
- Version APIs deliberately; avoid breaking existing consumers.
- Set request size, timeout, and rate limits at appropriate layers.
- Document APIs with OpenAPI where useful.
- Do not expose internal entity fields or database identifiers unnecessarily.

## 9. Transactions in Spring

`@Transactional` is typically implemented through proxies and transaction infrastructure.

```java
@Service
class OrderService {
    private final OrderRepository orders;
    private final InventoryService inventory;

    OrderService(OrderRepository orders, InventoryService inventory) {
        this.orders = orders;
        this.inventory = inventory;
    }

    @Transactional
    public OrderResponse placeOrder(CreateOrderCommand command) {
        // Validate and reserve inventory within the intended transaction design.
        // Persist order and its lines.
        return ...;
    }
}
```

### Propagation
- `REQUIRED`: join existing transaction or create one.
- `REQUIRES_NEW`: suspend current transaction and create a new one.
- `SUPPORTS`: join if one exists; otherwise run without one.
- `MANDATORY`: require an existing transaction.
- `NOT_SUPPORTED`: suspend current transaction.
- `NEVER`: fail if a transaction exists.
- `NESTED`: use a nested transaction/savepoint when supported by the transaction manager.

### Isolation
- `READ_UNCOMMITTED`
- `READ_COMMITTED`
- `REPEATABLE_READ`
- `SERIALIZABLE`

Actual behavior depends on the database and transaction manager. Understand dirty reads, non-repeatable reads, phantom reads, and the database's implementation.

### Rollback behavior
By default, Spring rolls back for unchecked exceptions (`RuntimeException` and `Error`), not every checked exception. Configure `rollbackFor` when needed:

```java
@Transactional(rollbackFor = ImportValidationException.class)
public void importProducts() throws ImportValidationException {
    // ...
}
```

Do not catch and suppress exceptions if the transaction should roll back. Remember that proxy boundaries and self-invocation affect whether transaction advice is applied.

## 10. Persistence integration: Spring Data JPA

### Repository
```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    Optional<Product> findBySku(String sku);
    Page<Product> findByStatus(ProductStatus status, Pageable pageable);
    boolean existsBySku(String sku);
}
```

### Service responsibilities
- Enforce SKU uniqueness and business rules.
- Coordinate repository operations within transaction boundaries.
- Convert entities to response DTOs.
- Avoid returning lazy entity graphs directly from controllers.
- Handle concurrent writes with database constraints and suitable locking/versioning.

### N+1 and fetch strategy
A page of products followed by one query per product for category/attributes can cause N+1 queries. Consider:
- DTO projections.
- Fetch joins for suitable bounded queries.
- `@EntityGraph`.
- Batch fetching.
- Explicit read models for complex screens.

Do not make every relationship eager. Large eager graphs can create worse performance and memory behavior.

### Pagination and sorting
```java
Pageable pageable = PageRequest.of(
    pageNumber, pageSize, Sort.by("name").ascending());
Page<Product> result = repository.findAll(pageable);
```

Validate maximum page size and use stable sorting. For very large or frequently changing datasets, keyset pagination may be more efficient than offset pagination.

## 11. Spring Security fundamentals

Security is not just adding a dependency. Define authentication, authorization, identity propagation, and operational controls.

Core concepts:
- Authentication: who is the caller?
- Authorization: what may the caller do?
- Security filter chain: processes web requests.
- Password hashing: use a password encoder such as BCrypt/Argon2; never store plaintext passwords.
- Method security: protect service operations as well as endpoints.
- OAuth2 Resource Server: validate access tokens, commonly JWTs.
- CORS: browser cross-origin policy; not authentication.
- CSRF: relevant to browser sessions/cookie-based authentication; assess based on the actual authentication model.

Modern configuration style:

```java
@Configuration
@EnableMethodSecurity
class SecurityConfig {
    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt())
            .build();
    }
}
```

This is a starting point, not a complete production security policy. Configure issuer/audience validation, CORS, CSRF, token claims-to-authority mapping, TLS, and endpoint access according to the system.

## 12. Async processing and scheduling

### @Async
```java
@Configuration
@EnableAsync
class AsyncConfig {
    @Bean
    Executor applicationTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(24);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("catalog-async-");
        executor.initialize();
        return executor;
    }
}
```

```java
@Service
class ExportService {
    @Async
    public CompletableFuture<ExportResult> generateExport(Long jobId) {
        return CompletableFuture.completedFuture(runExport(jobId));
    }
}
```

Configure bounded queues, rejection policy, metrics, timeouts, and error handling. Do not use unbounded thread creation. `@Async` is proxy-based and has a self-invocation limitation.

### Scheduling
```java
@Configuration
@EnableScheduling
class SchedulingConfig { }

@Component
class ReconciliationJob {
    @Scheduled(cron = "0 0/15 * * * *")
    void reconcile() {
        // Run a bounded, observable reconciliation batch.
    }
}
```

In a multi-instance deployment, a scheduled method may run on every instance. Use leader election, a distributed lock, or an external scheduler when the job must run once globally. Make jobs idempotent and observable.

## 13. Spring Boot Actuator and production readiness

Actuator provides operational endpoints and health/metrics integration.

Common endpoints:
- `/actuator/health`
- `/actuator/info`
- `/actuator/metrics`
- `/actuator/prometheus` when the relevant integration is configured.

Production checklist:
- Expose only required endpoints; secure sensitive endpoints.
- Configure liveness and readiness probes appropriately.
- Add structured logs with correlation/trace identifiers.
- Publish useful metrics: request rate, errors, latency, pool saturation, queue depth, JVM memory, GC, DB connection pool.
- Add distributed tracing and propagate trace context across HTTP/Kafka.
- Set graceful shutdown and request/drain timeouts.
- Set resource requests/limits and JVM container memory settings.
- Use TLS and secret management.
- Test startup and shutdown behavior.

Health checks should distinguish whether the process is alive from whether it is ready to receive traffic. Avoid making liveness fail just because a temporary downstream dependency is unavailable.

## 14. Testing Spring applications

### Unit test
Test business logic without starting Spring:

```java
@ExtendWith(MockitoExtension.class)
class ProductServiceTest {
    @Mock ProductRepository repository;
    @InjectMocks ProductService service;

    @Test
    void rejectsDuplicateSku() {
        // Arrange repository behavior.
        // Assert the service rejects the duplicate.
    }
}
```

### Spring slice tests
- `@WebMvcTest`: MVC/controller slice.
- `@DataJpaTest`: JPA repository slice.
- `@JsonTest`: JSON serialization slice.

### Integration tests
`@SpringBootTest` loads the application context. Use Testcontainers for realistic database/broker integration tests when practical. Avoid relying exclusively on H2 for behavior that differs from the production database.

Test:
- Validation and HTTP error contracts.
- Transaction rollback.
- Database constraints and concurrency.
- Security roles/claims.
- Kafka retry and duplicate delivery behavior.
- Migrations against the target database engine.

## 15. Enterprise case study A — Product Information Management (PIM)

### Business scenario
A merchandising platform manages products, SKUs, brands, categories, attributes, costs, selling prices, and channel-specific publication. Multiple users and batch imports update product data. Changes are published to downstream catalog/search systems.

### Suggested architecture
```text
React UI / External Clients
        |
   API Gateway / Ingress
        |
 Spring Boot Catalog API
   |       |         |
Controller Service  Validation
          |
   Domain / Use Cases
      |          |
 Product Repo  Outbox Repo
      |          |
   Relational DB |
                 v
          Outbox Publisher
                 |
               Kafka
                 |
     Search / Channel / Pricing Consumers
```

### Suggested domain model
- Product: identity, SKU, name, status, brand, category.
- ProductAttributeValue: product, attribute definition, value, locale/channel.
- Category: hierarchy and category metadata.
- Brand: brand identity and display metadata.
- ProductChannel: publication state per sales channel.
- ProductAudit: who changed what and when.
- OutboxEvent: event ID, aggregate ID, event type, payload, creation time, publish state.

### API examples
```text
POST   /api/v1/products
GET    /api/v1/products/{id}
GET    /api/v1/products?categoryId=10&status=ACTIVE&page=0&size=50
PATCH  /api/v1/products/{id}
POST   /api/v1/products/{id}/publish
POST   /api/v1/product-imports
GET    /api/v1/product-imports/{jobId}
```

### Enterprise concerns and design
1. Duplicate SKU: enforce a unique database constraint; translate violations to a stable conflict response. A pre-check alone is unsafe under concurrent requests.
2. Dynamic attributes: validate values against attribute definitions and product-type rules. Consider normalized attribute tables or a carefully governed JSON strategy; decide based on query/reporting needs.
3. Bulk imports: validate in chunks, persist job status, record row-level errors, use bounded batches, and make retries idempotent.
4. Product publishing: write product state and outbox record in one database transaction; publish asynchronously and mark delivery state. Consumers should be idempotent.
5. Concurrent edits: use `@Version` optimistic locking and return a conflict when a stale edit is submitted.
6. Audit: record actor, timestamp, action, and changed fields while protecting sensitive data.
7. Channel-specific data: model channel/locale-specific values explicitly; do not overwrite global product values accidentally.
8. Large category trees: avoid loading the full hierarchy for every request; use targeted queries and cache carefully.

### Example optimistic locking
```java
@Entity
class Product {
    @Id @GeneratedValue
    private Long id;

    @Version
    private long version;

    @Column(nullable = false, unique = true)
    private String sku;
}
```

### Example outbox transaction (conceptual)
```java
@Transactional
public void publishProduct(Long productId) {
    Product product = productRepository.findById(productId)
        .orElseThrow(ProductNotFoundException::new);
    product.markPublished();

    outboxRepository.save(OutboxEvent.productPublished(
        product.getId(), product.getVersion()));
}
```

A separate publisher reads unpublished outbox rows, sends events to Kafka, and marks them published. Design for retries and duplicate delivery; database commit and Kafka publish are not one ordinary local transaction.

## 16. Enterprise case study B — Order placement and inventory

### Scenario
A customer places an order containing multiple items. Inventory must be reserved, payment initiated, and order status updated.

### Important design decisions
- Do not hold a database transaction open while waiting on a slow external payment API.
- Use explicit order states such as `PENDING`, `RESERVATION_CONFIRMED`, `PAYMENT_PENDING`, `CONFIRMED`, `FAILED`, `CANCELLED`.
- Use idempotency keys for client retries.
- Use conditional stock updates or optimistic/pessimistic locking to prevent overselling.
- Model payment and inventory as separate service boundaries if they are independently deployed.
- Use a saga/process manager for multi-service coordination and compensating actions.
- Reconcile uncertain payment outcomes rather than assuming a timeout means payment failed.

### Example conditional update
```sql
UPDATE INVENTORY
SET AVAILABLE_QTY = AVAILABLE_QTY - :quantity
WHERE SKU = :sku
  AND AVAILABLE_QTY >= :quantity;
```

Check the affected row count. Zero means insufficient stock or a concurrent change; handle it explicitly.

## 17. Enterprise case study C — Bulk import and scheduled reconciliation

Scenario: thousands or millions of product rows arrive from suppliers.

Design:
- Upload file to object storage; persist import job metadata.
- Validate file format, schema, file size, and authorization.
- Process bounded chunks; do not load the whole file into heap.
- Use staging tables for large imports.
- Track `jobId`, row number, status, error code, and retry count.
- Make row processing idempotent.
- Apply back-pressure and limit concurrent jobs.
- Expose progress and downloadable error report.
- Schedule reconciliation for stuck jobs; use distributed locking in multi-instance deployments.

Spring options may include Spring Batch for restartable, chunk-oriented jobs. Choose it when job complexity and operational needs justify it rather than using it automatically for every small task.

## 18. Enterprise case study D — Multi-tenant SaaS

Questions to answer before implementation:
- Tenant identification: subdomain, token claim, header, or gateway context?
- Isolation: shared tables with tenant key, schema per tenant, or database per tenant?
- How are tenant IDs propagated to asynchronous messages?
- How are caches partitioned by tenant?
- Can support staff access tenant data, and how is that audited?
- How are migrations rolled out safely across tenants?

Never trust a tenant ID supplied only in an arbitrary request body. Derive and validate tenant context from authenticated identity and trusted routing. Add tenant predicates/constraints to every relevant data path.

## 19. Common production problems and diagnostic approach

| Symptom | Likely areas to inspect |
|---|---|
| Bean not found | Component scan, profile, conditional configuration, missing dependency |
| Multiple candidate beans | `@Qualifier`, `@Primary`, bean definitions |
| Circular dependency | Responsibility boundaries and dependency graph |
| Transaction not applied | Proxy boundary, self-invocation, method visibility, transaction manager |
| `LazyInitializationException` | Query boundary, DTO projection, fetch plan |
| Slow API | SQL count, N+1, indexes, pool saturation, external calls, serialization |
| Connection pool exhausted | Leaked connections, long transactions, pool sizing, slow queries |
| High memory | Large result sets, unbounded queues, retained references, oversized caches |
| Duplicate processing | At-least-once delivery, missing idempotency, retry design |
| App starts but is not ready | Downstream readiness, migrations, warmup, readiness probe design |
| Works locally, fails in prod | Profile/config drift, secrets, network policy, DB differences, clock/timezone |

A reliable debugging sequence:
1. Capture the exact symptom, timestamp, request/job ID, and affected environment.
2. Check recent deployment/configuration changes.
3. Inspect structured logs and trace IDs.
4. Inspect metrics: latency, errors, JVM, thread pools, connection pools, queue depth.
5. Inspect SQL and query plans where relevant.
6. Reproduce with production-like versions and data shape.
7. Fix the root cause and add a regression test/alert.

## 20. Spring Core and Boot interview questions

### Core concepts
1. What is IoC, and how does DI implement it?
2. Why is constructor injection generally preferred?
3. Difference between `BeanFactory` and `ApplicationContext`?
4. What is component scanning?
5. Difference between `@Component`, `@Service`, and `@Repository`?
6. How does Spring resolve multiple beans of one type?
7. Explain bean scopes and singleton thread-safety.
8. Explain bean lifecycle and post-processors.
9. What causes circular dependencies?
10. What is the difference between `@Bean` and `@Component`?
11. What is AOP, and how do Spring proxies work?
12. Why can self-invocation bypass `@Transactional`?
13. How do application events differ from Kafka events?

### Spring Boot
14. What does `@SpringBootApplication` combine?
15. How does auto-configuration work?
16. What are starter dependencies?
17. How do you debug an auto-configuration condition?
18. How do profiles and externalized configuration work?
19. How do you bind and validate custom properties?
20. What does Actuator provide?
21. How do readiness and liveness differ?
22. How do you secure Actuator endpoints?

### Transactions and persistence
23. How does `@Transactional` work internally?
24. Explain propagation and isolation.
25. What is the default rollback rule?
26. Why might a transaction annotation appear ineffective?
27. How do you prevent lost updates?
28. How do you diagnose N+1?
29. When would you use optimistic vs pessimistic locking?
30. Why is a database unique constraint necessary even if the service checks first?

### Enterprise architecture
31. How would you design a reliable product publication workflow?
32. How do you avoid losing an event after a database commit?
33. How do you make Kafka consumers idempotent?
34. How do you process a 2 GB supplier file safely?
35. How would you prevent overselling inventory?
36. How do you handle a payment timeout with an unknown outcome?
37. How do you design tenant isolation?
38. How do you operate a Spring Boot service in Kubernetes?
39. What metrics would you alert on?
40. How would you investigate a sudden increase in API latency?

## 21. Hands-on exercises

### Beginner to intermediate
1. Build a Spring application with constructor injection and two implementations of an interface.
2. Define a third-party client using `@Bean`.
3. Create a custom `@ConfigurationProperties` class with validation.
4. Create REST endpoints with request/response DTOs.
5. Add Bean Validation and global exception handling.
6. Add JPA repositories with pagination and sorting.
7. Write unit tests with JUnit and Mockito.
8. Add `@WebMvcTest` tests for controller behavior.
9. Add a transaction and demonstrate rollback.
10. Create an application event and an after-commit listener.

### Senior / enterprise
11. Build a product API with SKU uniqueness and optimistic locking.
12. Implement a bulk import job with chunking and row-level errors.
13. Implement an outbox table and Kafka publisher.
14. Make a Kafka consumer idempotent using an inbox/deduplication table.
15. Implement inventory reservation with a conditional SQL update.
16. Add role-based endpoint and method security.
17. Add Actuator, Prometheus metrics, and correlation IDs.
18. Build a Testcontainers integration test against the production database engine.
19. Add a distributed lock to a scheduled reconciliation job.
20. Load-test an endpoint and identify a connection-pool or N+1 bottleneck.

## 22. Suggested 21-day study plan

| Days | Focus | Practical output |
|---|---|---|
| 1–3 | IoC, DI, beans, scopes, lifecycle | Small Core container application |
| 4–5 | AOP, proxies, events | Logging aspect and event listener |
| 6–8 | Boot, starters, auto-config, profiles | Configurable REST service |
| 9–11 | MVC, validation, exceptions, API design | Product CRUD API |
| 12–13 | Transactions, JPA integration | Transactional order/product workflow |
| 14–15 | Security | JWT resource server and role rules |
| 16–17 | Async, scheduling, Actuator | Import job and health/metrics |
| 18–19 | Testing | Unit, slice, integration tests |
| 20 | Outbox, Kafka, idempotency | Reliable product publication |
| 21 | Capstone and interview review | Deployable PIM-style service |

## 23. Capstone project — Enterprise Product Catalog Service

Build a Spring Boot service with:
- Product, SKU, brand, category, attribute, and channel models.
- REST APIs with validation, pagination, and consistent errors.
- JPA persistence, migrations, indexes, and database constraints.
- Role-based authorization.
- Optimistic locking for product edits.
- Bulk import jobs with chunking and row-level error reporting.
- Transactional outbox and Kafka publication.
- Idempotent event consumer.
- Actuator, metrics, structured logs, tracing, and health probes.
- Unit, MVC slice, repository, and Testcontainers integration tests.
- Docker image and CI pipeline.

### Definition of done
- Concurrent requests cannot create duplicate SKUs.
- A stale update returns a controlled conflict.
- A committed product publication is eventually published to Kafka.
- Duplicate Kafka delivery does not duplicate downstream effects.
- Import processing is bounded, restartable, and observable.
- Sensitive operational endpoints are protected.
- Automated tests cover key failure paths.

---

## Further official references
- Spring Framework reference: https://docs.spring.io/spring-framework/reference/
- Spring Boot reference: https://docs.spring.io/spring-boot/reference/
- Spring Data JPA reference: https://docs.spring.io/spring-data/jpa/reference/
- Spring Security reference: https://docs.spring.io/spring-security/reference/
- Spring Batch reference: https://docs.spring.io/spring-batch/reference/

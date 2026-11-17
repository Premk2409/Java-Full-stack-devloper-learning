# Microservices Architecture and Design Patterns — In-Depth Enterprise Guide

A practical guide for experienced Java/Spring Boot developers. It covers service boundaries, distributed systems, communication, data ownership, resilience, deployment, observability, security, common design patterns, enterprise scenarios, interview questions, and hands-on exercises.

> **Version context:** Examples use Java 17+ and Spring Boot 3.x conventions. Microservice architecture is independent of Spring; Spring Boot is one implementation option. Use patterns only when they solve a real problem—distributed systems add operational and consistency costs.

## 1. What are microservices?

Microservices architecture structures a system as a set of independently deployable services. Each service owns a focused business capability and exposes a well-defined contract.

Typical characteristics:
- Business-capability-oriented boundaries.
- Independent deployment and scaling.
- Explicit APIs or messaging contracts.
- Data ownership per service.
- Fault isolation and operational autonomy.
- Automation for testing, deployment, monitoring, and recovery.

A service is not automatically a microservice because it has its own Spring Boot application or database schema. The boundary must represent a coherent business capability and have a clear ownership model.

### Monolith vs modular monolith vs microservices

| Architecture | Characteristics | Good fit |
|---|---|---|
| Monolith | One deployable application; modules may be tightly coupled | Small teams, simpler domains, fast initial delivery |
| Modular monolith | One deployable application with enforced internal module boundaries | Many systems that need modularity without distributed-system overhead |
| Microservices | Multiple independently deployable services with network boundaries | Multiple teams, independently evolving capabilities, distinct scaling/reliability needs |

**Important:** Start with a modular monolith when independent deployment is not yet valuable. Splitting too early creates network calls, data consistency problems, deployment complexity, and more failure modes.

## 2. How to identify service boundaries

Do not split services based only on technical layers such as controller, service, and repository. Split around business capabilities and domain boundaries.

### Domain-Driven Design concepts
- **Domain:** The business problem area.
- **Subdomain:** A distinct part of the business domain.
- **Bounded Context:** A boundary within which a domain model and its language have consistent meaning.
- **Ubiquitous Language:** Shared terms used by developers and domain experts.
- **Context Map:** How bounded contexts relate and integrate.
- **Aggregate:** A consistency boundary for domain changes.
- **Domain Event:** A fact that has happened in the domain.

### Example: enterprise merchandising / PIM platform

Potential business capabilities:

```text
Merchandising Platform
  ├── Product Catalog
  ├── Category Management
  ├── Pricing
  ├── Inventory
  ├── Supplier Integration
  ├── Product Import
  ├── Channel Publishing
  └── Search Indexing
```

Possible service boundaries (not mandatory):
- **Product Catalog Service:** product identity, SKU, product lifecycle and core product data.
- **Category Service:** category hierarchy, category rules and assignments.
- **Pricing Service:** price lists, effective dates, currency and pricing rules.
- **Inventory Service:** available quantity and reservations.
- **Supplier Integration Service:** supplier-specific mapping and inbound integration.
- **Import Service:** file intake, validation, job status and batch orchestration.
- **Publishing Service:** channel publication workflow and delivery status.
- **Search Service:** search-oriented read model and indexing.

Avoid splitting every table into a separate service. A service should own a cohesive capability and the data needed to enforce its invariants.

### Boundary assessment questions
1. Does this capability have a distinct business purpose?
2. Can its data and rules be owned by one team?
3. Does it need independent deployment or scaling?
4. Can its API/event contract remain stable?
5. Can it operate when another service is temporarily unavailable?
6. Are its consistency requirements understood?
7. Is the operational cost justified?

## 3. Reference architecture

```text
 Web / Mobile / Partner Clients
              |
       CDN / WAF / Gateway
              |
       API Gateway / BFF
        /      |       \
       v       v        v
 Product    Pricing   Inventory
 Service    Service    Service
    |          |          |
 Product DB  Price DB  Inventory DB
    |          |          |
    +------ Event Broker -+
               |
        Domain Events
               |
     +---------+----------+
     |                    |
 Publishing Service   Search Indexer
     |                    |
 Channel APIs        Search Database
```

### Common infrastructure components
- API Gateway or ingress.
- Identity provider and authorization.
- Service discovery or platform DNS.
- Synchronous HTTP/gRPC communication.
- Event broker such as Kafka.
- Database per service.
- Centralized logs, metrics and traces.
- CI/CD and container orchestration.
- Secrets/configuration management.

Do not add every infrastructure component by default. Select based on deployment environment and system requirements.

## 4. Communication patterns

### Synchronous communication
Common options:
- REST over HTTP.
- gRPC.
- Internal service-to-service HTTP clients.

Use synchronous calls when the caller needs an immediate answer and the dependency can meet the latency and availability requirements.

Example with Spring's `RestClient`:

```java
@Service
class PricingClient {
    private final RestClient restClient;

    PricingClient(RestClient.Builder builder) {
        this.restClient = builder
            .baseUrl("http://pricing-service")
            .build();
    }

    PriceResponse getPrice(String sku) {
        return restClient.get()
            .uri("/api/v1/prices/{sku}", sku)
            .retrieve()
            .body(PriceResponse.class);
    }
}
```

In production, configure connection and response timeouts, authentication, error mapping, metrics, and resilience. Do not rely on default or infinite timeouts.

### Asynchronous communication
Common options:
- Kafka.
- RabbitMQ.
- Cloud queues and event buses.

Use asynchronous messaging when the producer should not wait for every consumer, when workloads need buffering, or when independent consumers need the same event.

Example event:

```json
{
  "eventId": "evt-123",
  "eventType": "ProductPublished",
  "eventVersion": 1,
  "occurredAt": "2026-11-17T10:15:30Z",
  "productId": 501,
  "sku": "SKU-1001"
}
```

A production event contract should define schema evolution, compatibility, ownership, partition key, delivery semantics, retention, and personally identifiable information rules.

### Request-response vs event notification
- Request-response: caller asks a service to perform or retrieve something and expects a reply.
- Event notification: publisher announces a fact; subscribers decide what to do.
- Event-carried state transfer: event contains enough data for a consumer to update its local read model.

Do not use an event to disguise a synchronous dependency when the caller must know the result immediately.

## 5. Database per service and data ownership

Each microservice should own its data. Other services access it through a contract, not by querying its tables directly.

### Why shared databases create coupling
- A schema change can break multiple services.
- Teams coordinate deployments.
- Business rules are bypassed by direct SQL.
- Service boundaries become nominal rather than real.
- Independent scaling and technology choices become harder.

A database server or cluster may be shared operationally, but schemas/tables and access rights should preserve ownership boundaries.

### Cross-service queries
Options:
1. API composition: call multiple services and combine results.
2. CQRS/read model: build a query-optimized projection from events.
3. Data replication: maintain a local copy of selected data with clear freshness semantics.
4. Analytics platform: use a warehouse/lakehouse for reporting rather than querying operational databases.

Do not make a distributed join across service databases a normal application design.

## 6. Distributed transactions and consistency

A local database transaction cannot normally atomically commit changes across independent services and a message broker.

### Eventual consistency
A change in one service propagates asynchronously. Other services may temporarily show older state.

Design for:
- Clear state transitions.
- Retry and duplicate handling.
- Reconciliation.
- User-visible pending states.
- Monitoring of event lag and failed messages.

### Saga pattern
A saga coordinates a business process through local transactions and compensating actions.

Example: order placement:

```text
Create Order (PENDING)
       |
       v
Reserve Inventory
   | success
   v
Authorize Payment
   | success
   v
Confirm Order
```

Failure path:

```text
Payment authorization fails
       |
       v
Release Inventory Reservation
       |
       v
Mark Order PAYMENT_FAILED
```

Compensation is a new business action, not a database rollback of a completed remote transaction. For example, a refund is not identical to erasing a payment.

### Saga styles
- **Choreography:** services react to events and publish subsequent events.
  - Pros: fewer central coordinators.
  - Cons: process flow can become difficult to understand across many services.
- **Orchestration:** a process manager/saga orchestrator directs steps.
  - Pros: workflow is explicit and observable.
  - Cons: orchestrator can become overly central or business logic can accumulate there.

Use a saga when a business process spans multiple independently owned services and requires coordinated recovery.

## 7. Transactional Outbox pattern

Problem: application updates its database, then publishes an event. If the database commit succeeds but publishing fails, downstream systems never hear about the change.

Solution: write the business update and an outbox record in the same local database transaction. A separate publisher relays outbox records to the broker.

```java
@Transactional
public void publishProduct(Long productId) {
    Product product = productRepository.findById(productId)
        .orElseThrow(ProductNotFoundException::new);

    product.publish();

    outboxRepository.save(OutboxEvent.productPublished(
        product.getId(), product.getVersion()));
}
```

Publisher responsibilities:
- Read unpublished records in bounded batches.
- Publish with a stable event ID.
- Retry transient failures.
- Mark successful records.
- Handle poison messages and operational replay.
- Clean up/archive old records safely.

The outbox prevents the database-write/event-intent gap. It does not by itself guarantee exactly-once business effects at consumers. Consumers must tolerate duplicate delivery.

## 8. Idempotency and delivery semantics

Messaging systems commonly provide at-least-once delivery in practical configurations. A consumer may receive the same event more than once.

### Idempotent consumer
Use a stable event ID and record processed IDs in the same local transaction as the business effect.

```text
BEGIN
  if event_id already processed:
      skip business effect
  else:
      apply business effect
      insert event_id into processed_events
COMMIT
```

The deduplication record and business change must commit atomically in the consumer's database.

### Idempotent HTTP commands
For payment/order APIs, accept an idempotency key and persist:
- Key.
- Caller/tenant scope.
- Request hash.
- Processing state.
- Result or response reference.

Reject reuse of the same key with a different request body. Define key expiry and concurrency behavior.

### Delivery terms
- At-most-once: message may be lost, but is not redelivered.
- At-least-once: message may be redelivered; consumer must be idempotent.
- Exactly-once: often limited to a defined broker/transaction boundary; do not interpret it as end-to-end exactly-once business execution across arbitrary databases and external APIs.

## 9. Resilience patterns

### Timeout
Every network call needs an explicit deadline/timeout. A timeout should be based on the end-to-end latency budget, not chosen arbitrarily.

### Retry
Retry only transient failures. Use:
- Bounded attempts.
- Exponential backoff.
- Jitter.
- Retry budgets.
- Idempotency.
- Clear distinction between retryable and permanent errors.

Avoid retry storms. Do not retry validation failures or most client errors.

### Circuit breaker
Stops repeatedly calling a dependency that is failing. It moves between closed, open, and half-open states.

### Bulkhead
Isolates resource pools so one slow dependency does not consume all threads/connections.

### Rate limiting
Protects a service from excessive requests. Apply per user, tenant, client, or resource where appropriate.

### Fallback
A fallback may return cached/stale data or a controlled degraded response. Never fabricate a successful result for a critical operation such as payment authorization or inventory reservation.

### Resilience4j example (illustrative)
```java
@CircuitBreaker(name = "pricing", fallbackMethod = "fallbackPrice")
@Retry(name = "pricing")
public PriceResponse getPrice(String sku) {
    return pricingClient.getPrice(sku);
}

private PriceResponse fallbackPrice(String sku, Throwable cause) {
    throw new PricingUnavailableException("Pricing is temporarily unavailable");
}
```

Configure the policies explicitly. Annotation order, proxy behavior, exception classification, and fallback signatures matter. Test the actual configuration under failure.

## 10. API Gateway and Backend for Frontend (BFF)

### API Gateway responsibilities
- Routing.
- TLS termination (depending on platform).
- Authentication integration.
- Rate limiting.
- Request size limits.
- Basic request/response policies.
- Observability and correlation.

Avoid putting core business workflows and domain rules in the gateway.

### BFF
A Backend for Frontend provides a client-specific API composition layer, for example one BFF for web and another for mobile. It can reduce client round trips and tailor response shapes.

Do not create a BFF for every screen. Keep ownership and business logic in domain services.

## 11. Service discovery and load balancing

Service discovery lets callers find service instances. In Kubernetes, Services and DNS commonly provide stable service names while the platform routes to healthy pods.

Other environments may use a registry or platform-specific discovery.

Consider:
- Health/readiness integration.
- Connection pooling.
- Load balancing.
- DNS caching/TTL.
- Graceful shutdown and connection draining.
- Client-side vs server-side load balancing.

Avoid hard-coded pod IPs or instance addresses.

## 12. API and event versioning

### REST API evolution
Prefer additive, backward-compatible changes:
- Add optional fields.
- Avoid changing field meanings.
- Avoid removing fields before consumers migrate.
- Define null/missing/default semantics.
- Version only when compatibility cannot be maintained.

### Event schema evolution
- Include event type and schema version.
- Use a schema registry or governed schema repository when useful.
- Ensure consumers can handle compatible changes.
- Do not reuse a field with a new meaning.
- Consider replay: old events may be consumed by newer code.

Contract testing helps verify provider/consumer compatibility.

## 13. Observability for distributed systems

Three core pillars:
1. **Logs:** discrete events with structured fields.
2. **Metrics:** numeric measurements over time.
3. **Traces:** end-to-end path across service boundaries.

Use correlation/trace context across HTTP and messaging. Do not generate a new unrelated trace at every hop.

Useful metrics:
- Request rate, error rate, latency percentiles.
- Dependency latency and failure rate.
- Kafka consumer lag and processing failures.
- Queue depth and oldest message age.
- DB connection pool active/idle/pending.
- JVM heap, GC pauses, thread count.
- Circuit breaker state.
- Business metrics: import failures, publication delay, order failure rate.

### SLI, SLO, SLA
- SLI: measured indicator (e.g., successful request percentage).
- SLO: target for the indicator over a time window.
- SLA: contractual commitment, often with consequences.

Alert on user impact and actionable symptoms, not every noisy metric.

## 14. Security in microservices

Key practices:
- Central identity provider.
- OAuth 2.0 / OpenID Connect where appropriate.
- JWT validation with issuer, audience, expiry, and signature checks.
- Service-to-service identity; do not trust network location alone.
- Least-privilege authorization.
- TLS/mTLS where threat model requires it.
- Secret manager and rotation.
- Audit sensitive operations.
- Validate tenant context and ownership.
- Protect internal admin/Actuator endpoints.
- Avoid logging access tokens, passwords, personal data, or payment information.

A gateway authenticating a user does not automatically make every downstream call secure. Define how identity and authorization context are propagated and validated.

## 15. Deployment and platform patterns

### Containerization
Package each service as an immutable container image. Keep configuration and secrets outside the image.

### Kubernetes concepts
- Deployment: desired replicas and rollout.
- Pod: running workload unit.
- Service: stable network endpoint.
- ConfigMap: non-secret configuration.
- Secret: sensitive configuration (use platform controls and external secret management as appropriate).
- Ingress/Gateway: external HTTP entry.
- HPA: horizontal pod autoscaling.
- Readiness probe: whether the instance should receive traffic.
- Liveness probe: whether the process should be restarted.

### Deployment strategies
- Rolling deployment.
- Blue/green.
- Canary.
- Feature flags for controlled exposure.

Use backward-compatible database and event changes during rolling deployments. A service rollback may not reverse an irreversible data migration.

## 16. Design patterns — catalog

Patterns are reusable approaches, not rules. Choose based on the problem and trade-offs.

### Creational patterns
| Pattern | Intent | Enterprise example |
|---|---|---|
| Factory Method | Delegate object creation to a subtype/factory method | Choose supplier-specific parser |
| Abstract Factory | Create related families of objects | Create region-specific payment clients and validators |
| Builder | Construct complex objects step by step | Build a search request with optional filters |
| Singleton | One shared instance | Spring singleton-scoped stateless service (container-managed) |
| Prototype | New instance per request from container | Stateful helper object when genuinely needed |

### Structural patterns
| Pattern | Intent | Enterprise example |
|---|---|---|
| Adapter | Convert one interface to another | Wrap supplier API formats behind one interface |
| Facade | Simplify access to a subsystem | Catalog facade coordinates product, price and category reads |
| Decorator | Add behavior without changing wrapped class | Add metrics/retry/logging around a client |
| Proxy | Control access to an object | Spring transaction/security proxy |
| Composite | Treat individual and grouped objects uniformly | Category tree |
| Bridge | Separate abstraction from implementation | Notification abstraction with email/SMS providers |

### Behavioral patterns
| Pattern | Intent | Enterprise example |
|---|---|---|
| Strategy | Interchange algorithms | Pricing calculation by channel |
| Template Method | Define workflow skeleton; customize steps | Supplier import pipeline |
| Observer / Pub-Sub | Notify interested subscribers | Product changed event |
| Command | Encapsulate an operation | Submit/cancel import job |
| Chain of Responsibility | Pass request through handlers | Validation pipeline |
| State | Behavior depends on lifecycle state | Order or import job state |
| Specification | Encapsulate business predicates | Product eligibility rules |
| Mediator | Centralize interactions among components | Workflow coordinator |
| Iterator | Traverse collection without exposing representation | Iterate paged import records |

### Enterprise / architectural patterns
| Pattern | Intent | Example |
|---|---|---|
| Repository | Abstract persistence operations | ProductRepository |
| Unit of Work | Track changes for one transaction | JPA persistence context |
| DTO | Transfer data across boundaries | ProductResponse |
| Dependency Injection | Supply dependencies externally | Spring constructor injection |
| MVC | Separate model, view, controller concerns | Spring MVC API |
| CQRS | Separate command and query models | Product update model vs search projection |
| Event Sourcing | Store state changes as event sequence | Audit/history-centric domain (only when justified) |
| Saga | Coordinate distributed business transaction | Order/inventory/payment |
| Outbox | Persist event intent with business update | Reliable product publication |
| Strangler Fig | Gradually replace legacy system | Migrate legacy catalog modules one capability at a time |
| Anti-Corruption Layer | Protect domain model from external model | Supplier-specific API adapter |
| Bulkhead | Isolate failure/resource usage | Separate pools for pricing and supplier calls |
| Circuit Breaker | Stop calls to failing dependency | Protect catalog from pricing outage |

## 17. Applying patterns in Java and Spring

### Strategy pattern — pricing rules
```java
public interface PricingStrategy {
    Money calculate(Product product, PricingContext context);
}

@Component("standardPricing")
class StandardPricingStrategy implements PricingStrategy {
    public Money calculate(Product product, PricingContext context) {
        return product.basePrice();
    }
}

@Component("promotionPricing")
class PromotionPricingStrategy implements PricingStrategy {
    public Money calculate(Product product, PricingContext context) {
        return product.basePrice().subtract(context.discount());
    }
}

@Service
class PricingService {
    private final Map<String, PricingStrategy> strategies;

    PricingService(Map<String, PricingStrategy> strategies) {
        this.strategies = strategies;
    }

    Money calculate(String strategyName, Product product, PricingContext context) {
        PricingStrategy strategy = strategies.get(strategyName);
        if (strategy == null) {
            throw new IllegalArgumentException("Unknown pricing strategy");
        }
        return strategy.calculate(product, context);
    }
}
```

In a real system, avoid accepting arbitrary internal bean names from untrusted clients. Map validated business configuration to known strategies.

### Factory pattern — supplier file parser
```java
public interface SupplierFileParser {
    ParsedSupplierFile parse(InputStream input);
}

@Component
class SupplierParserFactory {
    private final Map<String, SupplierFileParser> parsers;

    SupplierParserFactory(Map<String, SupplierFileParser> parsers) {
        this.parsers = parsers;
    }

    SupplierFileParser forSupplier(String supplierCode) {
        SupplierFileParser parser = parsers.get(supplierCode);
        if (parser == null) {
            throw new UnsupportedSupplierException(supplierCode);
        }
        return parser;
    }
}
```

### Adapter pattern — external supplier API
```java
public interface SupplierCatalog {
    SupplierProduct fetchProduct(String supplierSku);
}

@Component
class AcmeSupplierAdapter implements SupplierCatalog {
    private final AcmeHttpClient client;

    AcmeSupplierAdapter(AcmeHttpClient client) {
        this.client = client;
    }

    public SupplierProduct fetchProduct(String supplierSku) {
        AcmeProductDto dto = client.getProduct(supplierSku);
        return new SupplierProduct(dto.code(), dto.title(), dto.amount());
    }
}
```

Keep vendor-specific DTOs and error models outside the core domain.

### Decorator pattern — client instrumentation
```java
class TimingSupplierCatalog implements SupplierCatalog {
    private final SupplierCatalog delegate;
    private final MeterRegistry registry;

    TimingSupplierCatalog(SupplierCatalog delegate, MeterRegistry registry) {
        this.delegate = delegate;
        this.registry = registry;
    }

    public SupplierProduct fetchProduct(String sku) {
        return Timer.builder("supplier.fetch")
            .register(registry)
            .record(() -> delegate.fetchProduct(sku));
    }
}
```

Spring AOP can implement similar cross-cutting behavior, but an explicit decorator is useful when behavior is part of the application's composition and should be visible in the type structure.

### State pattern — import job lifecycle
Possible states:
`RECEIVED -> VALIDATING -> PROCESSING -> COMPLETED`
with failure/cancellation transitions such as:
`VALIDATING -> FAILED`, `PROCESSING -> PARTIALLY_FAILED`, `PROCESSING -> CANCELLED`.

Define allowed transitions centrally. Do not let arbitrary API requests set a job to any state.

## 18. Enterprise case study — PIM microservices

### Business requirements
A retailer manages product master data, category assignments, product attributes, supplier feeds, prices, and publishing to multiple channels.

### Possible bounded contexts
| Context | Owns | Example operations |
|---|---|---|
| Product Catalog | Product identity, SKU, core attributes | Create/update product |
| Category Management | Category tree and classification rules | Assign product category |
| Pricing | Price lists and effective pricing | Calculate channel price |
| Supplier Integration | Supplier contracts and mapping | Ingest supplier feed |
| Import Orchestration | Import job and row processing status | Start/retry import |
| Publishing | Channel delivery workflow | Publish product |
| Search | Search projection and indexing | Search catalog |

These are candidate boundaries. Validate them with team ownership, transaction needs, data volume, and release independence before creating separate deployables.

### Product update flow
1. Client calls Product Catalog API.
2. Catalog validates the command and saves product changes.
3. Catalog writes a `ProductUpdated` outbox record in the same transaction.
4. Outbox publisher sends event to Kafka.
5. Search consumer updates its search projection.
6. Publishing consumer evaluates channel eligibility.
7. Publishing service sends eligible content to channel adapters.
8. Consumers record event IDs and handle duplicates.

### Example event contract
```json
{
  "eventId": "e7f9...",
  "eventType": "ProductUpdated",
  "eventVersion": 2,
  "occurredAt": "2026-11-17T10:15:30Z",
  "aggregateId": "501",
  "sku": "SKU-1001",
  "changedFields": ["name", "brandId"]
}
```

For events, include only data needed by consumers. Decide whether consumers fetch full details through an API or use event-carried state. Avoid publishing sensitive or unnecessary product/supplier data.

### Key failure scenarios
- Kafka unavailable: outbox accumulates; alert on oldest unpublished age and backlog.
- Duplicate event: consumer deduplicates by event ID.
- Out-of-order events: use aggregate version/sequence and ignore or reconcile stale updates.
- Channel API down: retry with backoff, then dead-letter/operational review.
- Invalid supplier row: persist row-level error; do not fail the entire file unless business policy requires it.
- Category service unavailable: decide whether product update can proceed with a pending validation state or must fail synchronously.
- Search projection delayed: expose freshness/lag and do not treat search as the source of truth.

## 19. Common microservices anti-patterns

1. **Distributed monolith:** services must be deployed together and make many synchronous calls.
2. **Shared database ownership:** multiple services update the same tables.
3. **Chatty services:** many small network calls for one user action.
4. **Nano-services:** service boundaries are too small to own a meaningful capability.
5. **God service:** one service owns nearly all business logic.
6. **Synchronous call chains:** one request depends on a long chain of services.
7. **No timeouts:** threads and connections remain occupied during outages.
8. **Blind retries:** amplify failures and duplicate non-idempotent operations.
9. **No observability:** impossible to trace one request across services.
10. **Shared domain library:** services cannot evolve independently because internal models are tightly coupled.
11. **Direct database reads:** consumers bypass the owning service's contract.
12. **Premature CQRS/event sourcing:** complexity without a clear business or scale requirement.
13. **One cluster/database per tiny service:** operational overhead exceeds value.
14. **Ignoring data ownership:** no team is accountable for data quality and schema evolution.

## 20. Choosing between synchronous and asynchronous communication

| Need | Prefer | Important caveat |
|---|---|---|
| Immediate validation or lookup | Synchronous API | Set timeout and define failure behavior |
| Long-running import | Async job + status API | Persist job state and support retries |
| Notify multiple downstream systems | Event | Consumers must be idempotent |
| Strong local consistency | One service/local transaction | Keep invariants within an ownership boundary |
| Cross-service workflow | Saga/process manager | Model compensation and timeouts |
| Search/reporting view | Read model/projection | Accept and expose eventual consistency |
| Critical payment result | Explicit payment workflow/API | Never fake success through fallback |

## 21. Interview questions

### Architecture and boundaries
1. What makes a service a microservice?
2. When would you choose a modular monolith?
3. How do you identify bounded contexts?
4. How do you decide whether two capabilities belong in one service?
5. What is a distributed monolith?
6. What does database-per-service mean?
7. How would you migrate a legacy monolith incrementally?

### Communication and data
8. REST vs gRPC vs messaging?
9. When should you use Kafka instead of REST?
10. What is eventual consistency?
11. How do you implement a transactional outbox?
12. What is the difference between outbox and CDC?
13. How do you handle duplicate messages?
14. How do you handle out-of-order events?
15. How do you evolve an event schema safely?
16. How do you implement cross-service queries?

### Resilience and workflow
17. Explain timeout, retry, circuit breaker, and bulkhead.
18. Why can retries make an outage worse?
19. What is a saga?
20. Choreography vs orchestration?
21. What is a compensating transaction?
22. How do you handle an unknown payment outcome?
23. How do you prevent overselling inventory?
24. How do you handle poison messages?

### Patterns and implementation
25. Explain Strategy, Factory, Adapter, Decorator, and Facade.
26. Where do you use Repository and DTO patterns?
27. What is CQRS? When is it unnecessary?
28. Explain Strangler Fig and Anti-Corruption Layer.
29. How does Spring implement transactions with proxies?
30. How would you make a scheduled job safe across multiple replicas?

### Operations and security
31. How do you trace a request across services?
32. What are SLI, SLO, and SLA?
33. Liveness vs readiness?
34. How do services authenticate one another?
35. How do you deploy without breaking old consumers?
36. What would you monitor for Kafka consumers?
37. How do you secure tenant data?
38. How do you investigate cascading failures?

## 22. Hands-on exercises

### Foundation
1. Convert a layered application into a modular monolith with explicit module boundaries.
2. Define bounded contexts for an online retail/PIM domain.
3. Create Product Catalog and Pricing Spring Boot services.
4. Implement REST communication with timeouts and error mapping.
5. Add service authentication and role-based authorization.
6. Add OpenAPI contracts and provider/consumer tests.

### Messaging and resilience
7. Publish product events to Kafka.
8. Implement an outbox publisher.
9. Implement an idempotent consumer.
10. Simulate duplicate and out-of-order messages.
11. Add circuit breaker, retry, timeout, and bulkhead policies.
12. Create a dead-letter handling and replay procedure.

### Distributed workflows
13. Implement an order saga: reserve inventory, authorize payment, confirm order.
14. Implement compensation when payment fails.
15. Handle payment timeout with status inquiry/reconciliation.
16. Build a long-running import workflow with job status.
17. Build a search projection from product events.
18. Add event schema compatibility tests.

### Operations
19. Containerize each service and run locally with a database and Kafka.
20. Deploy to Kubernetes with readiness/liveness probes.
21. Add trace propagation, structured logs, and metrics.
22. Load-test the system and diagnose one bottleneck.
23. Perform a rolling deployment with backward-compatible schema changes.
24. Document a production incident runbook.

## 23. Suggested 30-day learning plan

| Days | Focus | Deliverable |
|---|---|---|
| 1–3 | Monolith, modular monolith, microservice trade-offs | Architecture comparison |
| 4–6 | DDD, bounded contexts, service boundaries | PIM context map |
| 7–9 | REST, gRPC, gateway, BFF | Two-service synchronous flow |
| 10–12 | Kafka, event contracts, schema evolution | Product event pipeline |
| 13–15 | Database ownership, outbox, idempotency | Reliable event publication |
| 16–18 | Saga, eventual consistency, CQRS | Order workflow |
| 19–21 | Timeout, retry, circuit breaker, bulkhead | Resilience test suite |
| 22–24 | Security, observability, tracing | Secured observable services |
| 25–27 | Docker, Kubernetes, CI/CD | Deployable environment |
| 28–30 | Capstone, failure testing, interview review | End-to-end enterprise system |

## 24. Capstone — Retail PIM Microservices Platform

Build a small but production-minded platform:

**Services**
- Product Catalog
- Pricing
- Inventory
- Import Orchestrator
- Publishing
- Search Projection

**Technology options**
- Java 17 or 21.
- Spring Boot 3.x.
- Spring Data JPA.
- PostgreSQL or the database used by the target enterprise.
- Kafka.
- Resilience4j.
- Spring Security OAuth2 Resource Server.
- Docker Compose for local development.
- Kubernetes for deployment practice.
- OpenTelemetry and Micrometer/Actuator.
- JUnit, Mockito, Testcontainers, contract testing.

**Acceptance criteria**
- Each service owns its data.
- Product updates publish through an outbox.
- Consumers are idempotent.
- Search is eventually consistent and its lag is observable.
- Pricing outages do not exhaust Catalog resources.
- Import jobs are restartable and report row-level errors.
- APIs and event schemas evolve compatibly.
- Trace IDs connect a user request to downstream processing.
- The platform can be deployed and rolled back safely.

---

## Further reading — official documentation
- Spring Boot: https://docs.spring.io/spring-boot/reference/
- Spring Cloud: https://docs.spring.io/spring-cloud/reference/
- Spring for Apache Kafka: https://docs.spring.io/spring-kafka/reference/
- Resilience4j: https://resilience4j.readme.io/
- Apache Kafka documentation: https://kafka.apache.org/documentation/
- Kubernetes documentation: https://kubernetes.io/docs/
- OpenTelemetry documentation: https://opentelemetry.io/docs/

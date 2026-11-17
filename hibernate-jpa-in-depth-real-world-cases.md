# Hibernate & JPA -- In-Depth Guide with Real-World Work Cases

**For Java 8, 11, 17 and 21 \| Spring Boot \| Microservices \| 7+ Years
Experience**

This guide focuses on enterprise Hibernate usage: product/catalog
systems, order management, inventory, customer data, batch imports, REST
APIs and microservices.

> Version note: Spring Boot 2 commonly uses `javax.persistence`; Spring
> Boot 3+ uses `jakarta.persistence`. Use the namespace and Hibernate
> version managed by your project's Spring Boot BOM. Do not mix the two
> APIs.

------------------------------------------------------------------------

# Part 1 -- Fundamentals

## 1. ORM, JPA, Hibernate and Spring Data JPA

Object-Relational Mapping maps Java objects to relational database
tables.

-   **JPA / Jakarta Persistence:** specification, annotations and APIs
    such as `@Entity`, `EntityManager` and JPQL.
-   **Hibernate ORM:** implementation of JPA plus Hibernate-specific
    features.
-   **Spring Data JPA:** repository abstraction that reduces
    boilerplate; it does not replace Hibernate or JPA.

``` text
REST Controller
      |
Service Layer (@Transactional)
      |
Spring Data Repository / EntityManager
      |
JPA API
      |
Hibernate ORM
      |
JDBC
      |
Oracle / PostgreSQL / MySQL
```

## 2. Entity basics

``` java
@Entity
@Table(name = "CUSTOMER")
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "CUSTOMER_ID")
    private Long id;

    @Column(name = "CUSTOMER_NAME", nullable = false, length = 120)
    private String name;

    @Column(name = "EMAIL", unique = true, length = 180)
    private String email;

    protected Customer() { }

    public Customer(String name, String email) {
        this.name = name;
        this.email = email;
    }
}
```

Important: - Every entity needs an identifier. - JPA requires a
no-argument constructor, which may be protected. - Prefer non-final
entity classes and persistent fields for portable proxy/enhancement
behavior. - Database constraints should exist in the schema, not only in
Java annotations. - Use `BigDecimal` for money, not `double`.

## 3. Entity lifecycle

  State                  Meaning
  ---------------------- --------------------------------------------------
  Transient / new        Not associated with a persistence context
  Managed / persistent   Tracked by current persistence context
  Detached               Has persistent identity but is no longer tracked
  Removed                Marked for deletion

``` java
Customer c = new Customer("Ravi", "ravi@example.com"); // transient
entityManager.persist(c);  // managed
entityManager.flush();     // synchronize with database
entityManager.detach(c);   // detached
```

`merge()` returns a managed instance. The object passed to `merge()`
does not itself become managed.

## 4. Persistence context and first-level cache

A persistence context tracks managed entities, maintains one managed
instance per entity type/id within that context, performs dirty checking
and queues changes until flush.

``` java
Customer a = entityManager.find(Customer.class, 10L);
Customer b = entityManager.find(Customer.class, 10L);
System.out.println(a == b); // true in the same persistence context
```

### Dirty checking

``` java
@Transactional
public void renameCustomer(Long id, String newName) {
    Customer customer = repository.findById(id)
        .orElseThrow(EntityNotFoundException::new);

    customer.changeName(newName);
    // Hibernate detects the change and updates at flush/commit.
}
```

For a managed entity, calling `save()` after every field update is
generally unnecessary.

------------------------------------------------------------------------

# Part 2 -- Mapping Entities

## 5. Primary key generation

  -------------------------------------------------------------------------
  Strategy                Meaning                   Practical consideration
  ----------------------- ------------------------- -----------------------
  IDENTITY                Database                  Insert timing can limit
                          identity/auto-increment   batching

  SEQUENCE                Database sequence         Often suitable for
                                                    Oracle/PostgreSQL

  TABLE                   Uses a table to generate  Usually more overhead
                          IDs                       

  AUTO                    Provider chooses          Depends on
                                                    database/provider
  -------------------------------------------------------------------------

Oracle-style sequence mapping:

``` java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq_gen")
@SequenceGenerator(
    name = "product_seq_gen",
    sequenceName = "PRODUCT_SEQ",
    allocationSize = 50
)
private Long id;
```

Align sequence configuration with the database sequence and Hibernate
optimizer strategy.

## 6. Column, enum and date mapping

``` java
@Column(name = "PRODUCT_CODE", nullable = false, length = 40, unique = true)
private String productCode;

@Column(name = "UNIT_PRICE", precision = 12, scale = 2)
private BigDecimal unitPrice;

@Enumerated(EnumType.STRING)
@Column(name = "ORDER_STATUS")
private OrderStatus status;
```

Prefer `EnumType.STRING`; `ORDINAL` can silently change meaning if enum
order changes.

Use Java Time types deliberately: - `LocalDate`: calendar date. -
`LocalDateTime`: local date-time without zone. - `Instant`: point on UTC
timeline. - `OffsetDateTime`: date-time with offset.

Define timezone policy and database column types explicitly.

## 7. Embeddable value objects

``` java
@Embeddable
public class Address {
    private String line1;
    private String city;
    private String postalCode;
}

@Entity
public class Customer {
    @Id
    private Long id;

    @Embedded
    private Address address;
}
```

Use `@AttributeOverrides` when embedded columns need different names.

## 8. Relationships

### One-to-many / many-to-one -- Order and OrderItem

``` java
@Entity
@Table(name = "CUSTOMER_ORDER")
class CustomerOrder {
    @Id
    @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "order",
               cascade = CascadeType.ALL,
               orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
}

@Entity
@Table(name = "ORDER_ITEM")
class OrderItem {
    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "ORDER_ID", nullable = false)
    private CustomerOrder order;

    private String sku;
    private int quantity;

    void setOrder(CustomerOrder order) {
        this.order = order;
    }
}
```

Key concepts: - `@ManyToOne` normally owns the foreign key. - `mappedBy`
identifies the inverse side. - Update both sides in helper methods. -
`orphanRemoval = true` deletes a child disassociated from its parent at
flush. - Cascade only when the child's lifecycle belongs to the parent.

Do not cascade removal from a shared Product or Brand to unrelated
parent records.

### One-to-one

A one-to-one relationship generally needs a unique constraint on its
foreign key. Decide the owning side and consider lazy loading/proxy
behavior.

### Many-to-many

``` java
@ManyToMany
@JoinTable(
    name = "PRODUCT_TAG",
    joinColumns = @JoinColumn(name = "PRODUCT_ID"),
    inverseJoinColumns = @JoinColumn(name = "TAG_ID")
)
private Set<Tag> tags = new HashSet<>();
```

If the join table needs attributes such as created date, source, status
or sequence, model it as an explicit entity with two `@ManyToOne`
relationships.

## 9. Cascade and orphan removal

  Cascade   Effect
  --------- --------------------
  PERSIST   Persist propagates
  MERGE     Merge propagates
  REMOVE    Remove propagates
  REFRESH   Refresh propagates
  DETACH    Detach propagates
  ALL       All above

`orphanRemoval` is distinct from cascade remove: it removes a child when
disassociated from the parent relationship.

OrderItem may be owned by Order; Brand is shared master data and should
generally not be removed when a Product is deleted.

## 10. Fetch types and lazy loading

JPA defaults: - `@ManyToOne`, `@OneToOne`: EAGER. - `@OneToMany`,
`@ManyToMany`: LAZY.

EAGER does not guarantee one SQL join. For many application mappings,
prefer LAZY associations and fetch what a use case needs through fetch
joins, entity graphs or projections.

Accessing a lazy association after the persistence context closes can
cause `LazyInitializationException`.

------------------------------------------------------------------------

# Part 3 -- Queries and Repositories

## 11. Spring Data JPA repository

``` java
public interface ProductRepository extends JpaRepository<Product, Long> {
    Optional<Product> findByProductCode(String productCode);

    List<Product> findByDepartmentIdAndActiveTrue(Long departmentId);

    Page<Product> findByBrandId(Long brandId, Pageable pageable);
}
```

Derived query names suit simple conditions. Use JPQL, Criteria API or
native SQL for more complex cases.

## 12. JPQL

JPQL uses entity names and entity attributes, not physical table/column
names.

``` java
@Query("""
    select p
    from Product p
    where p.department.id = :departmentId
      and p.active = true
""")
List<Product> findActiveProducts(Long departmentId);
```

## 13. Native SQL

``` java
@Query(
    value = "SELECT * FROM PRODUCT WHERE DEPARTMENT_ID = :departmentId",
    nativeQuery = true
)
List<Product> findProductsNative(Long departmentId);
```

Use native SQL for database-specific features, legacy schemas or
measured tuning needs. It trades portability for control.

## 14. Criteria API

``` java
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<Product> cq = cb.createQuery(Product.class);
Root<Product> root = cq.from(Product.class);

List<Predicate> predicates = new ArrayList<>();
predicates.add(cb.isTrue(root.get("active")));
predicates.add(cb.equal(root.get("brand").get("id"), brandId));

cq.select(root).where(predicates.toArray(new Predicate[0]));
List<Product> result = entityManager.createQuery(cq).getResultList();
```

For complex dynamic search, use a reusable query builder/specification
rather than one enormous Criteria method.

## 15. Pagination and projections

``` java
Pageable pageable = PageRequest.of(
    0, 50, Sort.by(Sort.Direction.ASC, "productCode")
);
Page<Product> page = productRepository.findAll(pageable);
```

-   Use deterministic sorting with a unique tie-breaker.
-   Large offset pagination can become expensive.
-   Consider keyset/seek pagination for deep traversal.
-   `Page` often runs a count query; use `Slice` when total count is not
    needed.

Projection example:

``` java
public interface ProductSummary {
    Long getId();
    String getProductCode();
    String getName();
}

List<ProductSummary> findByDepartmentId(Long departmentId);
```

Use DTOs/projections when the endpoint needs only selected fields.

------------------------------------------------------------------------

# Part 4 -- Transactions and Concurrency

## 16. @Transactional

``` java
@Service
class OrderService {
    private final OrderRepository orderRepository;
    private final InventoryRepository inventoryRepository;

    @Transactional
    public Long placeOrder(CreateOrderRequest request) {
        // Validate request
        // Reserve inventory
        // Create order and lines
        // Persist changes atomically in this database
        return orderRepository.save(buildOrder(request)).getId();
    }
}
```

External HTTP calls and Kafka publication are not automatically part of
the database transaction.

Common attributes: - `propagation` - `isolation` - `readOnly` -
`timeout` - `rollbackFor` - `noRollbackFor`

Spring's default rollback behavior is generally for unchecked exceptions
(`RuntimeException` and `Error`), not every checked exception.

### Propagation

  Mode            Meaning
  --------------- -------------------------------------------
  REQUIRED        Join existing or create one
  REQUIRES_NEW    Suspend current and create a new one
  SUPPORTS        Join if present
  MANDATORY       Must have existing transaction
  NOT_SUPPORTED   Run without transaction
  NEVER           Fail if transaction exists
  NESTED          Savepoint/nested behavior where supported

Use `REQUIRES_NEW` carefully because it commits independently.

### Self-invocation

With Spring's default proxy-based transaction management, calling a
`@Transactional` method from another method on the same object usually
bypasses the proxy. The expected transaction boundary may not apply.
Prefer a separate Spring bean or explicit transaction design.

## 17. Flush vs commit

-   `flush()` synchronizes pending persistence-context changes with the
    database.
-   `commit()` completes the transaction.
-   Flush is not commit; flushed changes can still be rolled back.

Hibernate may flush automatically before certain queries and at
transaction commit.

## 18. save vs saveAndFlush

-   `save()` persists or merges depending on entity newness/state.
-   `saveAndFlush()` also triggers a flush.
-   Neither alone means the enclosing transaction committed.

For managed entities, dirty checking normally handles updates without
calling `save()` again.

## 19. Optimistic locking

Real case: two users edit the same product concurrently.

``` java
@Entity
class Product {
    @Id
    private Long id;

    @Version
    private Long version;

    private String description;
}
```

Hibernate checks the version during update. If another transaction
updated the row first, the later update can fail with an optimistic
locking exception.

Handle by returning a conflict, reloading and asking the user to retry,
or applying a domain-specific merge.

## 20. Pessimistic locking

Real case: two checkouts attempt to reserve the last unit.

``` java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select i from Inventory i where i.productId = :productId")
Optional<Inventory> findForUpdate(Long productId);
```

The database obtains a write lock on the row (exact behavior depends on
database/query). Keep transactions short and consider lock timeouts and
deadlocks.

An alternative is a conditional atomic update:

``` java
@Modifying
@Query("""
    update Inventory i
       set i.available = i.available - :quantity
     where i.productId = :productId
       and i.available >= :quantity
""")
int reserve(Long productId, int quantity);
```

One affected row means reservation succeeded; zero means stock was
insufficient or no row matched. Validate positive quantity. Bulk updates
bypass normal dirty checking for affected managed entities; clear or
refresh stale state as appropriate.

------------------------------------------------------------------------

# Part 5 -- Real-World Enterprise Work Cases

## 21. Case: Product Information Management (PIM)

Business requirements: - Product master data. - Brand, category,
department. - Dynamic product attributes. - Product publishing to sales
channels.

Possible model:

``` text
Brand 1 ---- * Product
Category 1 ---- * Product
Product 1 ---- * ProductAttributeValue
AttributeDefinition 1 ---- * ProductAttributeValue
```

``` java
@Entity
class Product {
    @Id
    private Long id;

    @Column(unique = true, nullable = false)
    private String sku;

    @ManyToOne(fetch = FetchType.LAZY)
    private Brand brand;

    @ManyToOne(fetch = FetchType.LAZY)
    private Category category;

    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL,
               orphanRemoval = true)
    private List<ProductAttributeValue> attributes = new ArrayList<>();
}
```

Design decisions: 1. Dynamic attributes may fit a child table or
carefully designed JSON model. 2. Frequently filtered attributes need
indexing and query design. 3. Brand is shared master data; do not
cascade delete it from Product. 4. Use endpoint-specific fetch plans or
projections rather than loading every attribute. 5. Use an outbox if
product updates and event publication must be coordinated reliably.

## 22. Case: Order and Order Items

Requirements: - Create order with multiple lines. - Calculate totals. -
Save order and lines atomically. - Update order status.

Design: - Order is aggregate root. - OrderItem lifecycle belongs to
Order. - Cascade persist may be appropriate. - Validate
quantities/totals in domain/service layer. - Keep entity graph out of
REST response. - Store SKU, description and unit-price snapshots on
order lines so historical orders do not change when catalog data
changes.

## 23. Case: Concurrent inventory reservation

Problem: many customers buy the same SKU at once.

Options: - Pessimistic row lock. - Optimistic versioning with retry. -
Atomic conditional SQL update.

Choose based on contention, throughput and business semantics. Add
integration tests with concurrent transactions. A simple read-then-write
without locking or a conditional update can oversell.

## 24. Case: Large product import

Requirements: hundreds of thousands of rows without exhausting memory.

Approach: 1. Read records in chunks. 2. Validate DTOs. 3. Persist
bounded batches. 4. Flush and clear persistence context. 5. Record
row-level errors. 6. Use chunked transactions. 7. Consider JDBC/bulk
loading for extreme throughput.

``` java
@Transactional
public void importBatch(List<Product> products) {
    int batchSize = 500;

    for (int i = 0; i < products.size(); i++) {
        entityManager.persist(products.get(i));

        if ((i + 1) % batchSize == 0) {
            entityManager.flush();
            entityManager.clear();
        }
    }

    entityManager.flush();
    entityManager.clear();
}
```

This assumes new entities and one transaction. For very large imports,
use chunked transactions. After `clear()`, previously managed objects
are detached.

Typical properties to evaluate:

``` properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
```

Batching depends on identifier strategy, driver support and SQL shape.
IDENTITY generation can limit insert batching. Verify actual behavior.

## 25. Case: LazyInitializationException in REST API

Cause: - Controller returns an entity. - JSON serialization accesses a
lazy association after the persistence context closes.

Better: - Map to DTO within transaction. - Fetch the needed association
with join/entity graph. - Use projection for list endpoints. - Do not
make every association EAGER to hide the problem.

## 26. Case: N+1 query problem

Example: - One query loads 100 orders. - Accessing each order's customer
triggers another query. - Total: 1 + 100 queries.

Fix options: - JPQL fetch join. - `@EntityGraph`. - DTO projection. -
Batch fetching.

``` java
@Query("""
    select o
      from CustomerOrder o
      join fetch o.customer
     where o.status = :status
""")
List<CustomerOrder> findOrdersWithCustomer(OrderStatus status);
```

Caution: - Multiple to-many fetches can multiply result rows or trigger
Hibernate limitations. - Collection fetch joins with pagination are
dangerous. - `distinct` may deduplicate entity results but does not
remove SQL row multiplication.

## 27. Case: Duplicate SKU under load

A pre-check is not enough:

``` java
if (!repository.existsByProductCode(code)) {
    repository.save(product);
}
```

Two requests can both pass the check before either inserts.

Correct approach: - Unique database constraint. - Optional pre-check for
friendly validation. - Translate unique-constraint exceptions. - Use
database upsert only if its semantics match the business requirement.

## 28. Case: Database update succeeds but Kafka event is missing

This is the dual-write problem.

Transactional outbox: 1. In one DB transaction, update Product and
insert OutboxEvent. 2. A publisher reads unpublished rows. 3. Publish
event to Kafka. 4. Mark published or track retry state. 5. Consumers are
idempotent because delivery can repeat.

Kafka publication is not automatically part of the ordinary JPA database
transaction.

------------------------------------------------------------------------

# Part 6 -- Performance and Troubleshooting

## 29. Performance checklist

-   Select only needed columns.
-   Avoid unnecessary EAGER relationships.
-   Detect and fix N+1 queries.
-   Add indexes based on actual query patterns.
-   Paginate large results.
-   Use JDBC batching when supported.
-   Avoid huge entity graphs and unbounded persistence contexts.
-   Use read-only transaction hints where appropriate.
-   Measure SQL, execution plans and connection-pool waits.
-   Avoid remote API calls inside long DB transactions.

## 30. First-level vs second-level cache

-   First-level cache belongs to the persistence context and is enabled
    for its lifetime.
-   Second-level cache is shared across persistence contexts for
    configured entity/collection regions.
-   Query cache stores query result information and depends on cache
    invalidation.

Use second-level cache for frequently read, infrequently changed data
with a clear invalidation strategy. Avoid caching volatile
inventory/order state without a strong reason.

## 31. Bulk update/delete

``` java
@Modifying
@Query("""
    update Product p
       set p.active = false
     where p.endDate < :today
""")
int deactivateExpired(LocalDate today);
```

Bulk JPQL DML: - Executes directly against the database. - Does not
update already managed instances. - Does not run normal per-entity
lifecycle callbacks/cascades. - Can leave persistence-context state
stale.

Clear/refresh deliberately and define transaction boundaries.

## 32. Common exceptions

  ---------------------------------------------------------------------------------
  Exception/problem                 Typical cause           Inspect
  --------------------------------- ----------------------- -----------------------
  LazyInitializationException       Lazy association        Transaction, DTO, fetch
                                    accessed outside        plan
                                    session                 

  NonUniqueResultException          Query expected one row  Query uniqueness
                                    but got multiple        

  OptimisticLockException           Version changed         Conflict/retry policy
                                    concurrently            

  ConstraintViolationException      DB constraint violated  Data and schema

  TransientPropertyValueException   Unsaved transient       Persist order/cascade
                                    association             

  EntityNotFoundException           Referenced entity       Referential integrity
                                    missing                 

  MultipleBagFetchException         Multiple bag            Fetch plan/collection
                                    collections fetched     mapping
                                    together                

  SQLGrammarException               Invalid SQL/schema      Generated SQL and
                                    mismatch                dialect

  StaleStateException               Unexpected row          Concurrent
                                    count/state             changes/mapping
  ---------------------------------------------------------------------------------

Names/wrappers vary by Hibernate and Spring versions. Inspect root cause
and SQL state.

## 33. SQL logging

Local development only:

``` properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
```

Bind-value logging is version-specific and can expose sensitive data. Do
not enable verbose bind logs in production without security review.

Slow endpoint investigation: 1. Capture latency and request size. 2.
Count SQL statements per request. 3. Identify N+1. 4. Inspect slow SQL
and execution plan. 5. Check indexes and row counts. 6. Check
connection-pool wait. 7. Review transaction duration and lock waits. 8.
Measure after each change.

------------------------------------------------------------------------

# Part 7 -- Testing

## 34. Repository tests

``` java
@DataJpaTest
class ProductRepositoryTest {
    @Autowired ProductRepository repository;

    @Test
    void findsProductByCode() {
        Product product = new Product("SKU-100");
        repository.saveAndFlush(product);

        assertTrue(repository.findByProductCode("SKU-100").isPresent());
    }
}
```

For Oracle-specific SQL, sequences, constraints, native queries or
migrations, test against the actual database engine or a compatible
container. H2 is not a perfect Oracle substitute.

Test: - Dirty checking. - Owning-side foreign key updates. - Cascade and
orphan removal. - Optimistic locking. - Lazy loading/fetch plans. -
Transaction rollback.

------------------------------------------------------------------------

# Part 8 -- Senior Interview Preparation

## 35. Senior-level interview questions

### Fundamentals

1.  JPA vs Hibernate vs Spring Data JPA?
2.  What is a persistence context?
3.  Explain entity lifecycle states.
4.  What is dirty checking?
5.  First-level vs second-level cache?
6.  Flush vs commit?
7.  `persist()` vs `merge()`?
8.  `save()` vs `saveAndFlush()`?

### Mapping

9.  Owning side and `mappedBy`?
10. ManyToOne vs OneToMany?
11. Cascade vs orphan removal?
12. Why use LAZY fetching?
13. Why is EAGER not the same as JOIN FETCH?
14. When should ManyToMany become a join entity?
15. STRING vs ORDINAL enum mapping?
16. How do you map composite keys?
17. How do you map embeddable value objects?

### Transactions and concurrency

18. Explain `@Transactional` propagation.
19. What is self-invocation?
20. Checked vs unchecked exception rollback?
21. Optimistic vs pessimistic locking?
22. What is `@Version`?
23. How do you prevent overselling inventory?
24. Does a database transaction include Kafka publish?
25. Explain transactional outbox.

### Performance

26. Explain N+1.
27. Fetch join vs EntityGraph vs DTO projection?
28. Why can collection fetch joins break pagination?
29. How do you enable JDBC batching?
30. Why can IDENTITY limit insert batching?
31. What do flush and clear do in batch processing?
32. How do bulk updates affect persistence context?
33. When should second-level cache be used?
34. How do you investigate a slow Hibernate endpoint?

## 36. Practical coding exercises

1.  Map Customer and Address bidirectionally.
2.  Map Order and OrderItem with cascade/orphan removal.
3.  Prevent accidental cascade deletion of shared Brand.
4.  Write JPQL for active products by department.
5.  Build paginated product search with deterministic sorting.
6.  Return DTO projection for product listing.
7.  Reproduce and fix N+1.
8.  Reproduce LazyInitializationException and fix it without global
    EAGER.
9.  Implement optimistic locking for product edits.
10. Prevent negative stock with conditional update or pessimistic lock.
11. Import 100,000 products in bounded batches.
12. Implement outbox persistence and publisher flow.
13. Handle duplicate SKU safely with unique constraint.
14. Test orphan removal.
15. Compare DTO projection with full entity loading.

## 37. Real-world scenario interview prompts

**A. Product listing takes 8 seconds:** inspect SQL count, N+1, fetch
plan, indexes, pagination, execution plan and projections.

**B. Duplicate SKU under load:** explain why an `exists` check is
insufficient, how a unique DB constraint solves the race and how to
translate the exception.

**C. Inventory becomes negative:** explain concurrent checkout,
conditional update, optimistic/pessimistic locking, transaction
boundaries and retry.

**D. Batch import consumes all memory:** explain chunking, flush/clear,
JDBC batching, bounded transactions and bulk loading.

**E. Product DB update succeeds but Kafka event is missing:** explain
dual-write and transactional outbox with idempotent consumers.

------------------------------------------------------------------------

# Part 9 -- Learning Plan and Capstone

## 38. 21-day learning plan

  Days     Topic                                Hands-on task
  -------- ------------------------------------ ------------------------------
  1--2     ORM, JPA, lifecycle                  Customer entity
  3--4     IDs, columns, embeddables, enums     Product mapping
  5--6     Relationships and cascade            Order and OrderItem
  7        Persistence context/dirty checking   Update managed entity
  8--9     JPQL, native SQL, Criteria           Product search
  10       Pagination and DTOs                  Product listing API
  11--12   Transactions and flush               Order transaction
  13       Optimistic/pessimistic locking       Concurrent inventory
  14--15   Fetch plans and N+1                  Diagnose query count
  16       Batch processing                     Chunked product import
  17       Cache and bulk DML                   Test stale state
  18       Outbox and Kafka                     Reliable event publishing
  19       Testing                              Repository integration tests
  20       Troubleshooting                      Diagnose slow endpoint
  21       Revision/project                     End-to-end PIM/order module

## 39. Capstone -- Product Catalog and Order Service

### Product Catalog

-   Product, Brand, Category and ProductAttribute entities.
-   Unique SKU and DB constraints.
-   Dynamic search, filters and pagination.
-   DTO projections.
-   Batch import.
-   Audit fields and optimistic locking.
-   Product-change outbox events.

### Order and Inventory

-   Order and OrderItem aggregate.
-   Inventory reservation with concurrency control.
-   Transactional order creation.
-   Snapshot SKU/name/price in order lines.
-   Idempotency for repeated requests.
-   Integration tests for transaction and locking behavior.

### Non-functional requirements

-   Capture SQL and query counts.
-   Prevent N+1 on key endpoints.
-   Add indexes based on query patterns.
-   Use bounded batch processing.
-   Measure endpoint latency.
-   Document transaction and event-consistency boundaries.

## 40. Official references

-   Jakarta Persistence: https://jakarta.ee/specifications/persistence/
-   Hibernate ORM: https://hibernate.org/orm/documentation/
-   Spring Data JPA: https://docs.spring.io/spring-data/jpa/reference/
-   Spring transactions:
    https://docs.spring.io/spring-framework/reference/data-access/transaction.html
-   Testcontainers: https://java.testcontainers.org/

**Senior developer priority:** Understand entity lifecycle and
persistence context, relationship ownership, transaction boundaries,
fetch plans, N+1 and slow SQL, concurrent updates, batch imports and
database/Kafka consistency. Hibernate expertise is not just annotations;
it is understanding the SQL and transaction behavior those annotations
produce.

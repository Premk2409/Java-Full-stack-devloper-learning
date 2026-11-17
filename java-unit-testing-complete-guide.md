# Java Unit Testing -- Complete Guide

**For Java 8/11/17/21, Spring Boot and Senior Full Stack Developers**

## 1. What is unit testing?

Unit testing verifies a small unit of application behavior in isolation.
A unit is often a method or class, but the real focus is a meaningful
behavior boundary.

  -----------------------------------------------------------------------
  Test level              Purpose                 Common tools
  ----------------------- ----------------------- -----------------------
  Unit                    Test business logic in  JUnit 5, Mockito
                          isolation               

  Integration             Test components working Spring Test,
                          together                Testcontainers

  API                     Test HTTP endpoints     MockMvc, REST Assured

  End-to-end              Test a complete         Playwright, Selenium
                          business flow           

  Performance             Test latency and        JMeter, Gatling, k6
                          throughput              
  -----------------------------------------------------------------------

A good unit test is fast, deterministic, independent, readable and
focused.

## 2. AAA pattern -- Arrange, Act, Assert

1.  Arrange: create input and dependencies.
2.  Act: call the behavior.
3.  Assert: verify the result.

``` java
@Test
void calculatesDiscount() {
    // Arrange
    DiscountService service = new DiscountService();

    // Act
    BigDecimal result = service.calculate(
        new BigDecimal("1000"), CustomerType.PREMIUM
    );

    // Assert
    assertEquals(new BigDecimal("900.00"), result);
}
```

The same idea is often described as Given--When--Then.

Test observable behavior: return values, state changes, exceptions and
important interactions. Avoid testing private methods directly.

## 3. JUnit 5

JUnit 5 has three main parts: - JUnit Platform: test execution
infrastructure. - JUnit Jupiter: programming and extension model for
JUnit 5. - JUnit Vintage: support for older JUnit 3/4 tests on the
platform.

### Maven dependencies

Prefer versions managed by your Spring Boot dependency BOM where
possible.

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

This starter commonly brings JUnit Jupiter, Mockito and Spring Test
support.

### Basic test

``` java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {
    @Test
    void addsTwoNumbers() {
        Calculator calculator = new Calculator();
        assertEquals(30, calculator.add(10, 20));
    }
}
```

### Important annotations

  Annotation             Purpose
  ---------------------- ----------------------------
  `@Test`                Test method
  `@DisplayName`         Human-readable test name
  `@BeforeEach`          Runs before each test
  `@AfterEach`           Runs after each test
  `@BeforeAll`           Runs once before all tests
  `@AfterAll`            Runs once after all tests
  `@Disabled`            Disables a test
  `@Nested`              Groups related tests
  `@Tag`                 Categorizes tests
  `@RepeatedTest`        Repeats a test
  `@ParameterizedTest`   Runs with multiple inputs
  `@TestFactory`         Creates dynamic tests

`@BeforeAll` and `@AfterAll` are static by default unless the test
lifecycle is configured as `PER_CLASS`.

### Assertions

``` java
assertEquals(expected, actual);
assertNotEquals(expected, actual);
assertTrue(condition);
assertFalse(condition);
assertNull(value);
assertNotNull(value);
assertSame(expectedReference, actualReference);
assertArrayEquals(expected, actual);
```

Grouped assertions:

``` java
assertAll(
    () -> assertEquals("ORD-1", order.getId()),
    () -> assertEquals(OrderStatus.CREATED, order.getStatus())
);
```

Exception testing:

``` java
OrderNotFoundException exception = assertThrows(
    OrderNotFoundException.class,
    () -> service.findById("missing")
);
assertEquals("Order not found", exception.getMessage());
```

Other useful assertions include `assertDoesNotThrow`, `assertTimeout`
and `assertInstanceOf`.

## 4. Parameterized tests

Use one test with multiple inputs.

``` java
@ParameterizedTest
@ValueSource(strings = {"", " ", "\t"})
void rejectsBlankNames(String name) {
    assertThrows(
        IllegalArgumentException.class,
        () -> validator.validateName(name)
    );
}
```

Common sources: - `@ValueSource` - `@NullSource` - `@EmptySource` -
`@NullAndEmptySource` - `@EnumSource` - `@CsvSource` -
`@CsvFileSource` - `@MethodSource`

CSV example:

``` java
@ParameterizedTest
@CsvSource({
    "100, 10, 90",
    "200, 20, 180",
    "50, 5, 45"
})
void calculatesDiscount(int amount, int discount, int expected) {
    assertEquals(expected, calculator.discount(amount, discount));
}
```

## 5. Mockito -- Mocking dependencies

Mockito lets a unit test isolate the class under test from external
collaborators such as repositories, payment clients, email services and
HTTP clients.

### Mock and inject dependencies

``` java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock
    OrderRepository repository;

    @Mock
    PaymentClient paymentClient;

    @InjectMocks
    OrderService service;
}
```

-   `@Mock`: creates a mock.
-   `@InjectMocks`: creates the object under test and injects available
    mocks.
-   `MockitoExtension`: initializes Mockito and integrates it with JUnit
    Jupiter.

Prefer constructor injection in production code.

### Stubbing

``` java
when(repository.findById("ORD-1"))
    .thenReturn(Optional.of(order));

when(client.charge("ORD-1"))
    .thenThrow(new PaymentException("Unavailable"));
```

Other options include `thenAnswer`, `thenThrow`, `doReturn` and
`doThrow`.

### Verify interactions

``` java
verify(repository).save(any(Order.class));
verify(paymentClient).charge("ORD-1");
verify(paymentClient, never()).refund(any());
verify(paymentClient, times(1)).charge("ORD-1");
```

Verify interactions when they are part of the behavior, not every
internal call.

### Argument matchers

Common matchers: - `any()`, `anyString()`, `anyLong()` - `eq(value)` -
`isNull()`, `notNull()` - `argThat(predicate)`

If one argument uses a matcher, use matchers for all arguments in that
invocation.

``` java
verify(client).send(eq("orders"), any(OrderEvent.class));
```

### ArgumentCaptor

Use a captor to inspect an object passed to a collaborator.

``` java
@Captor
ArgumentCaptor<Order> captor;

@Test
void savesOrderAsCreated() {
    service.createOrder(request);

    verify(repository).save(captor.capture());
    assertEquals(OrderStatus.CREATED, captor.getValue().getStatus());
}
```

### Mock vs Spy

  -----------------------------------------------------------------------
  Mock                                Spy
  ----------------------------------- -----------------------------------
  Methods return configured/default   Real methods execute by default
  values                              

  Good for external collaborators     Wraps a real object

  Interactions can be verified        Selected methods can be stubbed
  -----------------------------------------------------------------------

Use spies sparingly. Prefer simpler class design over partial mocking.

## 6. Test doubles

  Type    Meaning
  ------- ---------------------------------------------------------------
  Dummy   Passed but not used
  Stub    Returns prepared responses
  Mock    Records/verifies interactions
  Spy     Wraps a real implementation
  Fake    Lightweight working implementation, e.g. in-memory repository

A fake repository does not prove that the real database mapping or SQL
works. Add integration tests for that.

## 7. Complete service unit test example

Assume `OrderService` depends on `OrderRepository` and `PaymentClient`.

``` java
public class OrderService {
    private final OrderRepository repository;
    private final PaymentClient paymentClient;

    public OrderService(OrderRepository repository, PaymentClient paymentClient) {
        this.repository = repository;
        this.paymentClient = paymentClient;
    }

    public Order pay(String orderId) {
        Order order = repository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        if (order.getStatus() != OrderStatus.CREATED) {
            throw new IllegalStateException("Order is not payable");
        }

        paymentClient.charge(orderId);
        order.setStatus(OrderStatus.PAID);
        return repository.save(order);
    }
}
```

Test:

``` java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repository;
    @Mock PaymentClient paymentClient;
    @InjectMocks OrderService service;

    @Test
    void paysCreatedOrder() {
        Order order = new Order("ORD-1", OrderStatus.CREATED);
        when(repository.findById("ORD-1")).thenReturn(Optional.of(order));
        when(repository.save(any(Order.class)))
            .thenAnswer(invocation -> invocation.getArgument(0));

        Order result = service.pay("ORD-1");

        assertEquals(OrderStatus.PAID, result.getStatus());
        verify(paymentClient).charge("ORD-1");
        verify(repository).save(order);
    }

    @Test
    void doesNotChargeWhenOrderIsMissing() {
        when(repository.findById("missing")).thenReturn(Optional.empty());

        assertThrows(
            OrderNotFoundException.class,
            () -> service.pay("missing")
        );

        verifyNoInteractions(paymentClient);
        verify(repository, never()).save(any());
    }

    @Test
    void doesNotChargeAlreadyPaidOrder() {
        Order order = new Order("ORD-2", OrderStatus.PAID);
        when(repository.findById("ORD-2")).thenReturn(Optional.of(order));

        assertThrows(
            IllegalStateException.class,
            () -> service.pay("ORD-2")
        );

        verifyNoInteractions(paymentClient);
        verify(repository, never()).save(any());
    }
}
```

Add tests for payment failure, persistence failure and other
business-specific rules.

## 8. Spring Boot testing

Spring Boot provides test slices so you do not need to load the whole
application for every test.

  Annotation          Purpose
  ------------------- ------------------------------------
  `@SpringBootTest`   Loads the full application context
  `@WebMvcTest`       MVC/controller slice
  `@DataJpaTest`      JPA repository slice
  `@JsonTest`         JSON serialization/deserialization
  `@RestClientTest`   REST client testing

### Full context

``` java
@SpringBootTest
class ApplicationContextTest {
    @Test
    void contextLoads() {
    }
}
```

Use this for application wiring and broader integration scenarios. It is
slower than a plain unit test.

### Controller test with MockMvc

``` java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mockMvc;

    @MockBean
    OrderService orderService;

    @Test
    void returnsOrder() throws Exception {
        when(orderService.findById("ORD-1"))
            .thenReturn(new OrderDto("ORD-1"));

        mockMvc.perform(get("/orders/ORD-1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value("ORD-1"));
    }
}
```

`@MockBean` is used in many existing Spring Boot versions. Newer Spring
versions may provide `@MockitoBean`; use the annotation supported by the
project's dependency versions.

### Repository test

``` java
@DataJpaTest
class OrderRepositoryTest {
    @Autowired OrderRepository repository;

    @Test
    void savesAndFindsOrder() {
        repository.save(new Order("ORD-1", OrderStatus.CREATED));

        Optional<Order> found = repository.findById("ORD-1");

        assertTrue(found.isPresent());
        assertEquals(OrderStatus.CREATED, found.get().getStatus());
    }
}
```

Slice tests may use an embedded database and roll back transactions by
default. Use the actual database engine when testing vendor-specific
SQL, constraints or migrations.

## 9. MockMvc and REST API testing

MockMvc exercises Spring MVC request handling without starting a real
HTTP server.

``` java
mockMvc.perform(
        post("/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"customerId\":\"CUS-1\"}")
    )
    .andExpect(status().isCreated())
    .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON));
```

Test: - HTTP status and headers. - JSON fields and serialization. -
Validation errors. - Authentication and authorization. - Error response
format. - Request mapping and content types.

For a real server test, use
`@SpringBootTest(webEnvironment = RANDOM_PORT)` and an HTTP client.

## 10. Integration testing and Testcontainers

Integration tests verify that real components work together.

Examples: - Repository + PostgreSQL. - Kafka producer + broker +
consumer. - REST client + mock HTTP server. - Spring application +
database migrations.

Testcontainers starts disposable dependencies in containers.

``` java
@Testcontainers
@SpringBootTest
class OrderRepositoryIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16");

    // Configure the application to use this container.
    // Then test the real repository/database interaction.
}
```

The exact connection configuration depends on Spring Boot and
Testcontainers versions. Testcontainers can also run Kafka-compatible
brokers.

Use integration tests for SQL dialects, indexes, constraints,
transactions, migrations, serialization, broker behavior and other
external boundaries.

## 11. Testing asynchronous and concurrent code

-   Prefer latches, barriers and futures for coordination.
-   Use bounded timeouts.
-   Avoid arbitrary `Thread.sleep()` as the main synchronization
    mechanism.
-   Shut down executors.
-   Assert observable outcomes, not internal scheduling.

``` java
@Test
void completesAsyncTask() throws Exception {
    CompletableFuture<String> future =
        CompletableFuture.supplyAsync(() -> "done");

    assertEquals("done", future.get(2, TimeUnit.SECONDS));
}
```

For Kafka, use a broker container when verifying serialization,
partitioning, offset commits, retries or DLT behavior.

## 12. Code coverage and quality

Coverage measures which parts of code were executed. Common measures: -
Line coverage. - Branch coverage. - Instruction coverage. - Method
coverage.

Coverage is not proof of correctness. A test suite can have high line
coverage but weak assertions.

### JaCoCo Maven plugin

``` xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Commands:

``` bash
mvn clean test
mvn verify
```

Report is commonly generated at `target/site/jacoco/index.html`.

Other quality tools: - SonarQube. - Checkstyle / Spotless. - SpotBugs. -
PMD. - JaCoCo. - JMH for microbenchmarks.

## 13. Test data and test design

### Boundary-value testing

For an accepted amount range of 1 to 10,000, test: - 1: minimum valid. -
0: below minimum. - 10,000: maximum valid. - 10,001: above maximum. -
Null/missing value, if applicable.

### Equivalence partitioning

Group inputs expected to behave similarly and test representative values
from each group.

### Negative testing

Test nulls, empty values, invalid formats, missing records, duplicate
requests, unauthorized access, timeouts and dependency failures.

### Good test data practices

-   Use small, readable fixtures.
-   Avoid production data.
-   Keep test data explicit.
-   Use builders for complex objects.
-   Keep tests independent.
-   Clean up external resources.
-   Avoid shared mutable static test data.

### Test naming

Use descriptive names such as:

``` text
methodName_condition_expectedResult
```

Example: `pay_whenOrderIsAlreadyPaid_throwsIllegalStateException`.

## 14. Test pyramid and test strategy

``` text
          /\
         /E2E\          Few, broad, slower
        /------\
       /Integr. \       Some, real boundaries
      /----------\
     / Unit Tests \     Many, fast, focused
    /--------------\
```

A balanced suite usually has many fast unit tests, fewer integration
tests and a smaller number of end-to-end tests. Do not follow a fixed
ratio blindly; choose based on risk and architecture.

### Example strategy for a Spring Boot microservice

  Layer          What to test
  -------------- ---------------------------------------------
  Domain/model   Business invariants and value objects
  Service        Business rules using mocks/fakes
  Controller     Validation, HTTP mapping and errors
  Repository     Queries, mappings and constraints
  Messaging      Serialization, event handling, retries, DLT
  Security       Authentication, roles and access control
  End-to-end     Critical business workflows

## 15. CI/CD test automation

``` text
Developer pushes code
        |
        v
Compile and static analysis
        |
        v
Unit tests
        |
        v
Coverage report
        |
        v
Integration tests
        |
        v
Package application
        |
        v
Security/dependency checks
        |
        v
Deploy to test environment
        |
        v
Smoke / API / E2E tests
        |
        v
Promote or release
```

Useful commands:

``` bash
mvn clean test
mvn verify
mvn package
```

Common CI tools: Jenkins, GitHub Actions, GitLab CI and Azure DevOps.

Keep unit tests fast enough for every pull request. Run slower
integration and end-to-end tests at suitable pipeline stages.

## 16. Common mistakes

  -----------------------------------------------------------------------
  Mistake                             Better approach
  ----------------------------------- -----------------------------------
  Testing private methods directly    Test behavior through public APIs

  Mocking every dependency            Mock meaningful external
                                      collaborators

  Testing only happy paths            Add boundary and failure cases

  Weak assertions                     Assert observable behavior

  Arbitrary sleeps                    Coordinate with latches/futures

  Shared mutable state                Keep tests isolated

  Full Spring context for every test  Use plain unit tests or test slices

  Over-verifying calls                Verify meaningful interactions

  Depending on test order             Make tests independent

  Treating coverage as quality        Review cases and assertions
  -----------------------------------------------------------------------

## 17. Senior-level interview questions

1.  Unit testing vs integration testing?
2.  What is AAA?
3.  JUnit 4 vs JUnit 5?
4.  `@BeforeEach` vs `@BeforeAll`?
5.  `assertThrows` vs `assertDoesNotThrow`?
6.  What is a parameterized test?
7.  Mock vs stub vs spy vs fake?
8.  `@Mock` vs `@InjectMocks`?
9.  `when().thenReturn()` vs `doReturn().when()`?
10. When do you use `ArgumentCaptor`?
11. `@SpringBootTest` vs `@WebMvcTest`?
12. What is `@DataJpaTest`?
13. How do you test REST controllers?
14. How do you test a query against a real database?
15. What is Testcontainers?
16. Does 100% coverage guarantee quality?
17. How do you test exception scenarios?
18. How do you test asynchronous code?
19. How do you test Kafka producers and consumers?
20. How do you prevent flaky tests?
21. How do you test a service calling multiple external APIs?
22. How do you test transaction rollback?
23. How do you structure tests in microservices?
24. How do you run tests in Jenkins/GitHub Actions?
25. What should be mocked?

## 18. Practical exercises

### Exercise 1 -- Calculator

Test add, subtract, divide and division by zero.

### Exercise 2 -- Employee service

Test find, not found, create, duplicate and invalid data.

### Exercise 3 -- Order service

Test order creation, totals, discounts, invalid state transitions,
payment failures and notification behavior.

### Exercise 4 -- REST controller

Test success, validation failure, not found, JSON response, unauthorized
and forbidden cases.

### Exercise 5 -- Repository

Use Testcontainers with PostgreSQL to test save/retrieve, custom query,
unique constraint and rollback.

### Exercise 6 -- Kafka

Use a Kafka-compatible Testcontainer to test publishing, consuming,
deserialization, invalid messages, retries and dead-letter behavior.

## 19. Suggested 14-day learning plan

  Day   Focus                                     Practice
  ----- ----------------------------------------- ------------------------------
  1     Testing fundamentals, AAA, test pyramid   Simple unit tests
  2     JUnit 5 annotations/assertions            Validation tests
  3     Parameterized and exception tests         Boundary cases
  4     Mockito mocks and stubs                   Service with dependencies
  5     Verification and ArgumentCaptor           Verify saved entities/events
  6     Test doubles and clean test design        Refactor brittle tests
  7     Spring Boot test slices                   Controller test
  8     MockMvc and API testing                   REST endpoint tests
  9     JPA testing                               Repository tests
  10    Testcontainers                            PostgreSQL integration tests
  11    Async and Kafka testing                   Event flow tests
  12    JaCoCo and static analysis                Coverage report
  13    CI/CD automation                          Add tests to pipeline
  14    Revision and interview practice           Tested order service

## 20. Recommended references

-   JUnit 5: https://junit.org/junit5/docs/current/user-guide/
-   Mockito:
    https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html
-   Spring Boot testing:
    https://docs.spring.io/spring-boot/reference/testing/
-   Testcontainers: https://java.testcontainers.org/
-   JaCoCo: https://www.jacoco.org/jacoco/

**Senior developer priority:** Become strong at focused JUnit 5 tests,
appropriate Mockito usage, choosing the right Spring Boot test slice,
testing real database and messaging boundaries with Testcontainers, and
integrating reliable tests into CI/CD. The goal is confidence in
business behavior, not simply a high coverage percentage.

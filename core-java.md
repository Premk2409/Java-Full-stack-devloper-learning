# Core Java -- Complete Study Guide

**For Senior Java / Full Stack Developers \| Java 8, 11, 17 and 21**

A structured reference covering Java fundamentals, object-oriented
programming, language features, collections, generics, exceptions, I/O,
concurrency, JVM internals, modern Java features, testing, performance
and interview preparation.

------------------------------------------------------------------------

## 1. Java Platform Fundamentals

### 1.1 JDK, JRE and JVM

-   **JDK (Java Development Kit):** Tools for developing Java
    applications, including the compiler (`javac`), runtime, debugger
    and other utilities.
-   **JRE (Java Runtime Environment):** Runtime libraries and components
    needed to run Java applications.
-   **JVM (Java Virtual Machine):** Executes Java bytecode and manages
    runtime services such as memory, garbage collection and JIT
    compilation.

Typical flow:

``` text
.java source
    |
    | javac
    v
.class bytecode
    |
    | JVM class loader
    v
Bytecode verification and linking
    |
    v
Execution engine (interpreter + JIT)
    |
    v
Operating system / hardware
```

### 1.2 Java characteristics

-   Statically typed and strongly typed.
-   Object-oriented, with primitive types as well as objects.
-   Platform-independent at the bytecode level, provided a compatible
    JVM exists.
-   Automatic memory management through garbage collection.
-   Supports multithreading and concurrency APIs.
-   Rich standard library and ecosystem.
-   Supports functional-style programming through lambdas and streams.

### 1.3 Compile and run

``` bash
javac Hello.java
java Hello
```

Useful tools:

``` bash
java -version
javac -version
javap -c MyClass
jcmd <pid> VM.version
jstack <pid>
jmap -heap <pid>
```

------------------------------------------------------------------------

## 2. Data Types, Variables and Operators

### 2.1 Primitive types

  Type                                  Size Typical use
  ----------- ------------------------------ ---------------------------------
  `byte`                               8-bit Small integer
  `short`                             16-bit Integer
  `int`                               32-bit General integer
  `long`                              64-bit Large integer
  `float`                             32-bit Single-precision floating point
  `double`                            64-bit Double-precision floating point
  `char`                              16-bit UTF-16 code unit
  `boolean`     JVM-dependent representation `true` / `false`

Use `long` for large integer ranges and `double` for general
floating-point calculations. For financial calculations, prefer
`BigDecimal` with an explicit rounding policy.

### 2.2 Reference types

Reference variables refer to objects. Examples include classes, arrays,
interfaces, enums and records.

``` java
String name = "Java";
int[] numbers = {1, 2, 3};
```

A reference can be `null`, meaning it refers to no object.

### 2.3 Variables

-   Local variables: declared inside methods or blocks; must be
    definitely assigned before use.
-   Instance variables: belong to an object and receive default values.
-   Static variables: belong to the class.
-   Parameters: receive values when a method or constructor is invoked.

### 2.4 Type conversion

-   Widening conversion: generally automatic, such as `int` to `long`.
-   Narrowing conversion: requires an explicit cast and may lose
    information.
-   Boxing/unboxing: conversion between primitives and wrapper classes.

``` java
int count = 10;
long total = count;       // widening
int value = (int) total;  // narrowing
Integer boxed = count;    // autoboxing
int unboxed = boxed;      // unboxing
```

Beware of unboxing `null`, which causes `NullPointerException`.

### 2.5 Operators

Know arithmetic, relational, logical, assignment, bitwise, shift,
ternary and `instanceof` operators.

-   `&&` and `||` short-circuit.
-   `&` and `|` can evaluate both boolean operands and also operate on
    integral bits.
-   `==` compares primitive values or reference identity.
-   `equals()` is generally used for logical object equality.

### 2.6 `var`

Local-variable type inference was introduced in Java 10.

``` java
var message = "Hello"; // inferred as String
```

`var` is not a dynamic type. The compiler infers a fixed static type. It
is for local variables with initializers, not fields or method
parameters.

------------------------------------------------------------------------

## 3. Control Flow

### 3.1 Conditional statements

``` java
if (score >= 90) {
    System.out.println("A");
} else if (score >= 75) {
    System.out.println("B");
} else {
    System.out.println("C");
}
```

### 3.2 Loops

``` java
for (int i = 0; i < 5; i++) { }
while (condition) { }
do { } while (condition);
```

Enhanced for loop:

``` java
for (String item : items) {
    System.out.println(item);
}
```

### 3.3 `switch`

Traditional switch statements support case labels and fall-through
unless terminated appropriately.

Modern switch expressions:

``` java
String category = switch (score) {
    case 90, 100 -> "Excellent";
    case 70, 80 -> "Good";
    default -> "Other";
};
```

Use `yield` when a switch-expression arm needs a block and must return a
value.

------------------------------------------------------------------------

## 4. Object-Oriented Programming

### 4.1 Class and object

A class defines state and behavior. An object is an instance of a class.

``` java
class Employee {
    private final String name;

    Employee(String name) {
        this.name = name;
    }

    String getName() {
        return name;
    }
}
```

### 4.2 Four core OOP principles

1.  **Encapsulation:** Keep state private and expose controlled
    operations.
2.  **Abstraction:** Expose essential behavior while hiding
    implementation details.
3.  **Inheritance:** Reuse and specialize behavior through an `is-a`
    relationship.
4.  **Polymorphism:** Use a common supertype while runtime objects
    provide different implementations.

Prefer composition over inheritance when the relationship is not
genuinely `is-a` or when composition gives more flexible behavior.

### 4.3 Constructors

-   Have the same name as the class.
-   Do not have a return type.
-   Can be overloaded.
-   Are not inherited.
-   A constructor can call another constructor using `this(...)`, or a
    superclass constructor using `super(...)`.
-   Constructor chaining must occur first, subject to
    Java-version-specific constructor-body rules.

### 4.4 `this` and `super`

-   `this` refers to the current object.
-   `this(...)` calls another constructor in the same class.
-   `super` accesses superclass members.
-   `super(...)` invokes a superclass constructor.

### 4.5 Method overloading and overriding

**Overloading** - Same method name, different parameter list. - Resolved
at compile time. - Return type alone cannot distinguish overloads.

**Overriding** - Subclass supplies an implementation of an inherited
instance method. - Resolved dynamically at runtime. - Cannot reduce
visibility. - Cannot broaden checked exceptions beyond what the
overridden method permits. - `@Override` helps the compiler verify
intent.

Static methods are hidden, not overridden. Private methods are not
overridden. Constructors are not overridden.

### 4.6 `static`, `final` and `abstract`

-   `static`: member belongs to the class.
-   `final` variable: can be assigned once.
-   `final` method: cannot be overridden.
-   `final` class: cannot be subclassed.
-   `abstract` class: cannot be instantiated directly and may declare
    abstract methods.

A final reference cannot be reassigned, but the referenced object may
still be mutable.

### 4.7 Access modifiers

  -----------------------------------------------------------------------------
  Modifier              Same class   Same package    Subclass in Other packages
                                                         another 
                                                         package 
  ----------------- -------------- -------------- -------------- --------------
  `private`                    Yes             No             No             No

  package-private              Yes            Yes        Through             No
                                                     inheritance 
                                                           rules 

  `protected`                  Yes            Yes      Yes, with             No
                                                    access rules 

  `public`                     Yes            Yes            Yes            Yes
  -----------------------------------------------------------------------------

### 4.8 Abstract class vs interface

  -----------------------------------------------------------------------
  Abstract class                      Interface
  ----------------------------------- -----------------------------------
  Can hold instance state             Fields are implicitly
                                      `public static final`

  Can have constructors               No constructors

  Can have abstract and concrete      Can have abstract, default, static
  methods                             and private methods

  A class extends one class           A class can implement multiple
                                      interfaces

  Useful for shared base              Useful for contracts and
  state/behavior                      capabilities
  -----------------------------------------------------------------------

### 4.9 Object class methods

Know the contracts and typical uses of:

-   `equals()`
-   `hashCode()`
-   `toString()`
-   `getClass()`
-   `clone()` (often avoided in favor of copy constructors/factories)
-   `wait()`, `notify()`, `notifyAll()`

If two objects are equal according to `equals()`, they must return the
same `hashCode()`.

------------------------------------------------------------------------

## 5. Strings and Immutability

### 5.1 `String`

`String` is immutable. Operations that appear to modify a string create
another string.

``` java
String first = "Java";
String second = first.concat(" Developer");
```

### 5.2 String pool

String literals can be interned and shared.

``` java
String a = "hello";
String b = "hello";
System.out.println(a == b);       // commonly true due to pooling
System.out.println(a.equals(b));  // true: content equality
```

Do not use `==` to compare string contents.

### 5.3 `StringBuilder` and `StringBuffer`

  -----------------------------------------------------------------------
  String                  StringBuilder           StringBuffer
  ----------------------- ----------------------- -----------------------
  Immutable               Mutable                 Mutable

  Concatenation creates   Efficient repeated      Synchronized methods
  new strings             modification            

  Safe to share due to    Not thread-safe         Individual methods are
  immutability                                    synchronized
  -----------------------------------------------------------------------

Use `StringBuilder` for repeated concatenation in a single-threaded
context.

### 5.4 Why immutability is useful

-   Safe sharing across threads.
-   Stable hash codes for map keys.
-   Easier reasoning and caching.
-   Reduced risk of unexpected state changes.

Create immutable classes by using private final fields, controlled
construction, no mutating setters and defensive copies of mutable
inputs/outputs.

------------------------------------------------------------------------

## 6. Arrays, Wrapper Classes and Enums

### 6.1 Arrays

Arrays have fixed length and are indexed from zero.

``` java
int[] values = new int[3];
int[] scores = {10, 20, 30};
```

Know `Arrays.sort`, `Arrays.binarySearch`, `Arrays.copyOf`,
`Arrays.equals` and `Arrays.asList` (the latter returns a fixed-size
list backed by the array for object arrays).

### 6.2 Wrapper classes

`Integer`, `Long`, `Double`, `Boolean`, `Character`, etc. wrap primitive
values.

Important: - Wrapper objects are immutable. - Use `.equals()` for value
comparison. - Unboxing `null` throws `NullPointerException`. - Wrapper
caching exists for certain values, but never rely on reference identity.

### 6.3 Enums

Enums represent a fixed set of constants and can contain fields,
constructors and methods.

``` java
enum OrderStatus {
    CREATED, PAID, SHIPPED, CANCELLED
}
```

Prefer enums over arbitrary strings for closed sets of values.

------------------------------------------------------------------------

## 7. Packages, Imports and Classpath

-   Packages organize related types and help control access.
-   `import` lets source code refer to types by simple name.
-   The classpath tells the JVM/compiler where to find classes and JARs.
-   The module system introduced in Java 9 adds explicit module
    boundaries.
-   Understand package naming conventions and avoid the default package
    in application code.

Useful commands:

``` bash
jar tf application.jar
java -cp "lib/*:app.jar" com.example.Main
```

On Windows, classpath entries use `;` instead of `:`.

------------------------------------------------------------------------

## 8. Exception Handling

### 8.1 Exception hierarchy

``` text
Throwable
 |-- Error
 |-- Exception
      |-- RuntimeException
      |-- Other checked exceptions
```

-   **Error:** Serious JVM or environment problems; generally not
    handled as normal application failures.
-   **Checked exception:** Must be caught or declared.
-   **Unchecked exception:** `RuntimeException` and its subclasses; not
    required to be declared or caught.

### 8.2 `try`, `catch`, `finally`

``` java
try {
    int result = 10 / value;
} catch (ArithmeticException ex) {
    // Handle expected failure
} finally {
    // Cleanup, if needed
}
```

`finally` normally executes whether or not an exception is thrown, but
not in every possible termination scenario (for example, abrupt JVM
termination).

### 8.3 `throw` vs `throws`

-   `throw`: actually throws an exception object.
-   `throws`: declares exceptions that a method may propagate.

### 8.4 Custom exceptions

``` java
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(String orderId) {
        super("Order not found: " + orderId);
    }
}
```

### 8.5 Try-with-resources

Use for resources implementing `AutoCloseable`.

``` java
try (var reader = Files.newBufferedReader(path)) {
    return reader.readLine();
}
```

Resources close automatically. If both the body and close operation
fail, close failures can be recorded as suppressed exceptions.

### 8.6 Best practices

-   Catch the most specific exception you can handle.
-   Do not swallow exceptions silently.
-   Preserve the cause when wrapping an exception.
-   Do not use exceptions for normal control flow.
-   Log at an appropriate layer; avoid logging and rethrowing the same
    failure at every layer.
-   Do not expose sensitive internal details to API clients.
-   Define meaningful domain exceptions.

------------------------------------------------------------------------

## 9. Collections Framework

### 9.1 Main hierarchy

``` text
Iterable
  |
Collection
  |-- List
  |    |-- ArrayList
  |    |-- LinkedList
  |    |-- Vector (legacy)
  |
  |-- Set
  |    |-- HashSet
  |    |-- LinkedHashSet
  |    |-- TreeSet
  |
  |-- Queue
       |-- PriorityQueue
       |-- Deque
            |-- ArrayDeque
            |-- LinkedList

Map (separate from Collection)
  |-- HashMap
  |-- LinkedHashMap
  |-- TreeMap
  |-- Hashtable (legacy)
  |-- ConcurrentHashMap
```

### 9.2 List

-   `ArrayList`: resizable array; fast indexed access; middle
    insert/remove may shift elements.
-   `LinkedList`: doubly linked list; indexed access requires traversal;
    supports deque operations.
-   `Vector`: legacy synchronized list; generally prefer modern
    alternatives.

### 9.3 Set

-   `HashSet`: unique elements, no guaranteed iteration order.
-   `LinkedHashSet`: preserves insertion order.
-   `TreeSet`: sorted order; uses natural ordering or a comparator.

### 9.4 Map

-   `HashMap`: key-value map; permits one null key and null values; no
    guaranteed iteration order.
-   `LinkedHashMap`: predictable encounter order (insertion order by
    default; can be configured for access order).
-   `TreeMap`: sorted keys.
-   `Hashtable`: legacy synchronized map; disallows null keys and
    values.
-   `ConcurrentHashMap`: concurrent access; disallows null keys and
    values.

### 9.5 Queue and Deque

-   `Queue`: commonly FIFO.
-   `Deque`: supports insertion/removal at both ends.
-   `PriorityQueue`: head is determined by priority ordering, not
    insertion order.
-   `ArrayDeque`: efficient general-purpose deque; disallows null
    elements.

### 9.6 HashMap internals

Important concepts: - Uses hash codes to locate buckets. - Uses
`equals()` to distinguish keys with matching bucket/hash. - Collisions
are handled within a bucket; modern implementations can treeify
sufficiently large buckets under applicable conditions. - Resizing
occurs as the map grows beyond its threshold. - It is not thread-safe. -
Mutable keys are dangerous: changing fields used by
`equals()`/`hashCode()` after insertion can make entries difficult to
find.

### 9.7 Comparable vs Comparator

``` java
class Employee implements Comparable<Employee> {
    private int id;

    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.id, other.id);
    }
}
```

``` java
Comparator<Employee> byName =
    Comparator.comparing(Employee::getName);
```

-   `Comparable`: natural ordering implemented by the type.
-   `Comparator`: external/custom ordering; multiple comparators can
    exist.

### 9.8 Fail-fast and weakly consistent iteration

Many standard collection iterators are fail-fast on best-effort basis
when structural modification occurs outside the iterator. Concurrent
collections often provide weakly consistent iterators. Fail-fast
behavior is not a thread-safety guarantee.

### 9.9 Choosing a collection

  Requirement                        Typical choice
  ---------------------------------- -----------------------
  General indexed list               `ArrayList`
  Unique elements                    `HashSet`
  Unique elements, insertion order   `LinkedHashSet`
  Sorted set/map                     `TreeSet` / `TreeMap`
  Key-value lookup                   `HashMap`
  Concurrent key-value access        `ConcurrentHashMap`
  FIFO queue                         `ArrayDeque`
  Priority processing                `PriorityQueue`

------------------------------------------------------------------------

## 10. Generics

Generics provide compile-time type safety and reduce casts.

``` java
List<String> names = new ArrayList<>();
names.add("Asha");
```

### 10.1 Generic classes and methods

``` java
class Box<T> {
    private T value;

    public void set(T value) { this.value = value; }
    public T get() { return value; }
}

static <T> T first(List<T> values) {
    return values.get(0);
}
```

### 10.2 Bounded type parameters

``` java
<T extends Number>
```

A type parameter can be constrained by a class or interface bound.
Multiple interface bounds are possible.

### 10.3 Wildcards and PECS

-   `? extends T`: producer of `T`; read values as `T`, but generally
    cannot add a specific `T`.
-   `? super T`: consumer of `T`; can add `T`, but reads are generally
    typed as `Object`.

**PECS:** Producer Extends, Consumer Super.

``` java
void copy(List<? extends Number> source,
          List<? super Number> destination) {
    for (Number n : source) {
        destination.add(n);
    }
}
```

### 10.4 Type erasure

Generic type information is mostly erased at runtime. Consequences
include: - Cannot normally create `new T()` or `new T[]`. - Cannot use
primitive types as generic type arguments; use wrappers. - Cannot
overload methods solely by generic parameterizations that erase to the
same signature. - Runtime checks cannot generally distinguish
`List<String>` from `List<Integer>`.

------------------------------------------------------------------------

## 11. `equals()`, `hashCode()` and Object Contracts

### 11.1 `equals()` contract

It should be: - Reflexive - Symmetric - Transitive - Consistent - False
for comparison with `null`

### 11.2 `hashCode()` contract

-   Equal objects must have equal hash codes.
-   Unequal objects may have the same hash code.
-   Hash code should remain stable while equality-relevant state remains
    unchanged.

When overriding `equals()`, also override `hashCode()`.

``` java
@Override
public boolean equals(Object obj) {
    if (this == obj) return true;
    if (!(obj instanceof Employee other)) return false;
    return id == other.id;
}

@Override
public int hashCode() {
    return Integer.hashCode(id);
}
```

This pattern uses modern pattern matching for `instanceof`; use a
traditional cast on older Java versions.

------------------------------------------------------------------------

## 12. Java 8 Functional Programming

### 12.1 Functional interfaces

A functional interface has exactly one abstract method. It may have
default and static methods.

Common interfaces:

  Interface             Method         Purpose
  --------------------- -------------- ----------------------
  `Predicate<T>`        `test(T)`      Boolean condition
  `Function<T,R>`       `apply(T)`     Transform value
  `Consumer<T>`         `accept(T)`    Consume value
  `Supplier<T>`         `get()`        Supply value
  `UnaryOperator<T>`    `apply(T)`     T to T
  `BinaryOperator<T>`   `apply(T,T)`   Combine two T values

### 12.2 Lambda expressions

``` java
Predicate<Integer> even = n -> n % 2 == 0;
Function<String, Integer> length = String::length;
```

### 12.3 Method references

Forms include: - Static method: `ClassName::staticMethod` - Instance
method on a particular object: `object::method` - Instance method on an
arbitrary object of a type: `String::toLowerCase` - Constructor:
`ArrayList::new`

### 12.4 Optional

`Optional<T>` represents a value that may be absent.

``` java
Optional<String> name = Optional.ofNullable(findName());

String result = name.orElse("Unknown");
```

Useful methods: `of`, `ofNullable`, `empty`, `isPresent`, `ifPresent`,
`map`, `flatMap`, `filter`, `orElse`, `orElseGet`, `orElseThrow`.

Use `orElseGet` when the fallback is expensive or should only be
evaluated when empty. Avoid using `Optional` as a universal replacement
for every nullable field or method parameter.

------------------------------------------------------------------------

## 13. Stream API

Streams support declarative processing of data.

### 13.1 Stream pipeline

``` text
Source -> Intermediate operations -> Terminal operation
```

Intermediate operations are generally lazy. A terminal operation
triggers evaluation.

### 13.2 Common operations

-   Intermediate: `filter`, `map`, `flatMap`, `distinct`, `sorted`,
    `peek`, `limit`, `skip`.
-   Terminal: `collect`, `toList`, `forEach`, `count`, `reduce`, `min`,
    `max`, `anyMatch`, `allMatch`, `noneMatch`, `findFirst`, `findAny`.

### 13.3 Examples

``` java
List<String> names = employees.stream()
    .filter(e -> e.getSalary() > 50000)
    .map(Employee::getName)
    .sorted()
    .toList();
```

Grouping:

``` java
Map<String, List<Employee>> byDepartment =
    employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

Counting:

``` java
Map<String, Long> countByDepartment =
    employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.counting()
        ));
```

Flattening:

``` java
List<String> allSkills = employees.stream()
    .flatMap(e -> e.getSkills().stream())
    .distinct()
    .toList();
```

### 13.4 `map()` vs `flatMap()`

-   `map`: one input becomes one output.
-   `flatMap`: one input can become a stream of outputs; nested streams
    are flattened.

### 13.5 `reduce()` vs `collect()`

-   `reduce`: combines values into a result, often immutable or
    associative aggregation.
-   `collect`: mutable reduction into containers such as lists, sets or
    maps.

### 13.6 Parallel streams

Parallel streams use fork/join infrastructure and can help with suitable
CPU-bound, independent workloads. They are not automatically faster.

Avoid: - Shared mutable state. - Blocking I/O in common-pool tasks. -
Assuming encounter order without checking the operation. - Using
parallelism without measuring performance.

------------------------------------------------------------------------

## 14. Date and Time API

Java 8 introduced `java.time`.

  Type                  Purpose
  --------------------- -------------------------------
  `LocalDate`           Date without time zone
  `LocalTime`           Time without date
  `LocalDateTime`       Date and time without zone
  `Instant`             Point on UTC timeline
  `ZonedDateTime`       Date and time with zone
  `OffsetDateTime`      Date and time with UTC offset
  `Duration`            Time-based amount
  `Period`              Date-based amount
  `DateTimeFormatter`   Parsing and formatting

``` java
LocalDate today = LocalDate.now();
Instant now = Instant.now();

ZonedDateTime indiaTime =
    ZonedDateTime.now(ZoneId.of("Asia/Kolkata"));

String formatted =
    today.format(DateTimeFormatter.ISO_DATE);
```

For timestamps exchanged across services, prefer a clearly defined
instant/offset representation. Do not assume the server's default time
zone.

------------------------------------------------------------------------

## 15. I/O, NIO and File Handling

### 15.1 Classic I/O

-   Byte streams: `InputStream`, `OutputStream`.
-   Character streams: `Reader`, `Writer`.
-   Buffered streams reduce repeated low-level I/O operations.
-   Use try-with-resources to close streams.

### 15.2 NIO.2

`Path` and `Files` provide modern file APIs.

``` java
Path path = Path.of("data.txt");

Files.writeString(path, "Hello Java");
String content = Files.readString(path);
```

Other topics: - `Files.copy`, `move`, `delete`, `exists`, `walk`. -
`DirectoryStream`. - `FileChannel`. - `ByteBuffer`. - `WatchService`. -
`AsynchronousFileChannel`.

### 15.3 Serialization

Java object serialization uses `Serializable` and object streams. It has
compatibility and security risks. Avoid deserializing untrusted data;
for service communication, prefer explicit formats such as JSON, Avro or
Protocol Buffers.

------------------------------------------------------------------------

## 16. Multithreading and Concurrency

This is a major topic for senior Java interviews and production systems.

### 16.1 Process vs thread

-   Process: independent executing program with its own address space.
-   Thread: execution path within a process; threads share process
    memory.

### 16.2 Creating threads

``` java
Thread thread = new Thread(() -> System.out.println("Running"));
thread.start();
```

Calling `run()` directly does not start a new thread; `start()` does.

Prefer task submission to an executor over creating unmanaged threads
for every task.

### 16.3 Thread lifecycle

Common states: - NEW - RUNNABLE - BLOCKED - WAITING - TIMED_WAITING -
TERMINATED

`RUNNABLE` includes a thread that may be executing or ready to execute.

### 16.4 Runnable vs Callable

-   `Runnable`: no result; cannot declare checked exceptions in its
    method signature.
-   `Callable<V>`: returns a value and can throw checked exceptions.

### 16.5 ExecutorService

``` java
ExecutorService executor = Executors.newFixedThreadPool(4);

try {
    Future<Integer> future =
        executor.submit(() -> 10 + 20);

    Integer result = future.get();
} finally {
    executor.shutdown();
}
```

In production, choose a bounded and appropriately sized executor, define
a rejection policy, and manage its lifecycle. Avoid unbounded task
queues where overload can exhaust memory.

### 16.6 Synchronization

``` java
synchronized void increment() {
    count++;
}
```

A synchronized instance method locks on `this`; a static synchronized
method locks on the `Class` object.

Synchronization provides mutual exclusion and memory visibility under
the Java Memory Model.

### 16.7 `volatile`

``` java
private volatile boolean running = true;
```

`volatile` provides visibility and ordering guarantees for reads/writes
of that variable. It does not make compound operations such as `count++`
atomic.

### 16.8 Atomic classes

`AtomicInteger`, `AtomicLong`, `AtomicBoolean` and atomic reference
classes support atomic operations.

``` java
AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();
```

### 16.9 Locks

-   `ReentrantLock`: explicit lock/unlock, try-lock, interruptible
    acquisition and optional fairness.
-   `ReadWriteLock`: separate read and write locks.
-   `StampedLock`: optimistic and read/write locking modes; use
    carefully.

Always release explicit locks in `finally`.

### 16.10 wait/notify

These methods belong to `Object` and must be called while owning that
object's monitor.

Use a loop to re-check the condition because wakeups may be spurious and
the condition may no longer hold.

``` java
synchronized (lock) {
    while (!ready) {
        lock.wait();
    }
    consume();
}
```

Prefer higher-level concurrency utilities for most application code.

### 16.11 Concurrent utilities

Know: - `ConcurrentHashMap` - `BlockingQueue` - `ArrayBlockingQueue` -
`LinkedBlockingQueue` - `CountDownLatch` - `CyclicBarrier` -
`Semaphore` - `Phaser` - `Exchanger` - `CompletableFuture` -
`ForkJoinPool` - `ThreadLocal`

### 16.12 CompletableFuture

``` java
CompletableFuture<String> future =
    CompletableFuture.supplyAsync(() -> loadData())
        .thenApply(String::toUpperCase)
        .exceptionally(ex -> "FALLBACK");
```

Important methods: - `thenApply`: transform result. - `thenCompose`:
chain another asynchronous stage. - `thenCombine`: combine independent
stages. - `allOf`: wait for multiple futures. - `anyOf`: complete when
any supplied future completes. - `exceptionally`, `handle`,
`whenComplete`: error/result handling.

Use explicit executors for blocking work. Understand the difference
between synchronous and `Async` continuations.

### 16.13 Deadlock, livelock and starvation

-   Deadlock: threads wait indefinitely for each other's locks.
-   Livelock: threads remain active but make no useful progress.
-   Starvation: a thread is repeatedly denied needed resources or CPU
    time.

Reduce risk through consistent lock ordering, short critical sections,
timeouts and higher-level concurrency constructs.

### 16.14 Java Memory Model

Understand: - Heap and thread stacks. - Visibility and atomicity. -
Happens-before relationship. - Safe publication. - Data races. -
Final-field semantics. - Synchronization and volatile guarantees.

------------------------------------------------------------------------

## 17. JVM Internals

### 17.1 Runtime memory areas

  Area                  Purpose
  --------------------- ---------------------------------------------
  Heap                  Objects and arrays
  Java stacks           Frames for method calls and local variables
  Metaspace             Class metadata in native memory
  PC register           Current instruction position per thread
  Native method stack   Support for native execution

The exact implementation and memory layout depend on the JVM.

### 17.2 Heap generations

Many JVM garbage collectors organize heap space using generational
concepts: - Young generation / young collections. - Old generation / old
collections. - Survivor spaces in collectors that use them.

Do not assume every collector uses exactly the same physical layout.

### 17.3 Garbage collection

Garbage collection reclaims memory occupied by objects that are no
longer reachable.

Know these collectors: - Serial GC - Parallel GC - G1 GC - ZGC -
Shenandoah (availability depends on JVM distribution/version)

Important concepts: - Reachability and GC roots. - Stop-the-world
pauses. - Throughput vs latency. - Allocation rate. - Heap sizing. -
Promotion and remembered sets (collector-dependent).

### 17.4 Memory leaks in Java

A Java memory leak occurs when objects remain reachable even though the
application no longer needs them.

Common causes: - Static collections that grow indefinitely. - Unbounded
caches. - Listeners not removed. - `ThreadLocal` values not cleaned up
in pooled threads. - Retained references in long-lived objects. -
Unbounded queues.

### 17.5 JIT compiler

The JVM can compile frequently executed bytecode into native machine
code. The JIT uses runtime profiling and optimizations such as inlining
and escape analysis, depending on the JVM.

### 17.6 Class loading

Typical phases: 1. Loading 2. Linking: verification, preparation and
resolution 3. Initialization

Know bootstrap, platform and application class loaders, parent
delegation, classpath/module-path behavior and classloader leaks.

------------------------------------------------------------------------

## 18. Java 9--21 Language and Platform Evolution

### Java 9

-   Module system (Project Jigsaw).
-   JShell.
-   Collection factory methods such as `List.of`.
-   Private interface methods.
-   Try-with-resources can use effectively final existing variables.

### Java 10

-   Local variable type inference with `var`.

### Java 11

-   Standard HTTP Client API.
-   New String methods such as `isBlank`, `lines`, `strip`, `repeat`.
-   `Files.readString` and `Files.writeString`.
-   Single-file source-code launch.

### Java 14

-   Switch expressions finalized.
-   Helpful NullPointerException messages (JVM feature).
-   Records were previewed in this release.

### Java 15

-   Text blocks finalized.
-   Sealed classes were previewed.

### Java 16

-   Records finalized.
-   Pattern matching for `instanceof` finalized.
-   `Stream.toList()`.

### Java 17 (LTS)

-   Sealed classes finalized.
-   Strong encapsulation of JDK internals.
-   Pattern matching for `switch` was preview.

### Java 18

-   Simple web server.
-   UTF-8 as the default charset.
-   Javadoc code snippets.

### Java 19

-   Virtual threads, structured concurrency and record patterns were
    preview features.

### Java 20

-   Further previews of virtual threads, structured concurrency and
    record patterns.

### Java 21 (LTS)

-   Virtual threads finalized.
-   Record patterns finalized.
-   Pattern matching for `switch` finalized.
-   Sequenced collections.
-   Generational ZGC.
-   String templates were previewed, not finalized in Java 21.

### Records

Records are concise data-carrier classes.

``` java
public record Customer(Long id, String name) {}
```

Records are implicitly final and provide component accessors, canonical
construction, `equals`, `hashCode` and `toString`. Their component
fields are final, but referenced objects are not automatically deeply
immutable.

### Sealed classes

``` java
public sealed interface Payment permits CardPayment, UpiPayment {
}

public final class CardPayment implements Payment {}
public final class UpiPayment implements Payment {}
```

Sealed types restrict which types may extend or implement them.

### Pattern matching

``` java
if (obj instanceof String text) {
    System.out.println(text.length());
}
```

### Virtual threads

Virtual threads are lightweight threads managed by the JVM. They are
useful for high-concurrency, mostly blocking workloads.

``` java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<String> result = executor.submit(() -> callService());
    System.out.println(result.get());
}
```

Virtual threads do not make CPU-bound work faster and do not remove the
need for database connection limits, downstream rate limits or
backpressure.

------------------------------------------------------------------------

## 19. Annotations and Reflection

### 19.1 Built-in annotations

-   `@Override`
-   `@Deprecated`
-   `@SuppressWarnings`
-   `@FunctionalInterface`
-   `@SafeVarargs`

### 19.2 Custom annotations

``` java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Audited {
    String value() default "";
}
```

Retention: - `SOURCE` - `CLASS` - `RUNTIME`

Targets specify where an annotation may be used.

### 19.3 Reflection

Reflection allows runtime inspection and interaction with classes,
fields, methods and constructors.

``` java
Class<?> type = Class.forName("com.example.Customer");
for (var method : type.getDeclaredMethods()) {
    System.out.println(method.getName());
}
```

Use reflection thoughtfully because it can reduce compile-time safety,
complicate debugging and interact with module access restrictions.

------------------------------------------------------------------------

## 20. Design Principles and Common Design Patterns

### 20.1 SOLID

1.  **Single Responsibility:** A class should have one cohesive reason
    to change.
2.  **Open/Closed:** Open for extension, closed for modification.
3.  **Liskov Substitution:** Subtypes should be usable where their base
    type is expected.
4.  **Interface Segregation:** Prefer focused interfaces over broad
    ones.
5.  **Dependency Inversion:** High-level policy should depend on
    abstractions, not concrete low-level details.

### 20.2 Creational patterns

-   Singleton: one controlled instance; consider enum singleton or
    dependency injection.
-   Factory Method: delegate object creation to a method/subclass.
-   Abstract Factory: create families of related objects.
-   Builder: construct complex objects step by step.
-   Prototype: create objects by copying an existing instance.

### 20.3 Structural patterns

-   Adapter: make incompatible interfaces work together.
-   Decorator: add behavior by wrapping an object.
-   Facade: provide a simpler interface to a subsystem.
-   Proxy: control access to another object.
-   Composite: represent part-whole hierarchies.
-   Bridge: separate abstraction from implementation.

### 20.4 Behavioral patterns

-   Strategy: interchangeable algorithms.
-   Observer: notify subscribers about state/events.
-   Template Method: define algorithm skeleton with customizable steps.
-   Command: encapsulate a request as an object.
-   Chain of Responsibility: pass a request through handlers.
-   State: vary behavior based on internal state.
-   Iterator: traverse a collection.
-   Mediator: centralize communication between objects.

### 20.5 Enterprise patterns to know

-   Dependency Injection
-   Repository
-   Unit of Work
-   DTO
-   Service Layer
-   Data Mapper
-   Circuit Breaker
-   Retry and Timeout
-   Bulkhead
-   Saga
-   Transactional Outbox
-   Idempotent Consumer

------------------------------------------------------------------------

## 21. JDBC and Database Connectivity

Core Java developers should understand JDBC even when using Spring Data
JPA.

### JDBC workflow

1.  Obtain a connection.
2.  Create a prepared statement.
3.  Bind parameters.
4.  Execute query/update.
5.  Process the result set.
6.  Commit/rollback as appropriate.
7.  Close resources.

``` java
String sql = "SELECT id, name FROM employee WHERE id = ?";

try (Connection connection = dataSource.getConnection();
     PreparedStatement statement = connection.prepareStatement(sql)) {

    statement.setLong(1, employeeId);

    try (ResultSet rs = statement.executeQuery()) {
        if (rs.next()) {
            String name = rs.getString("name");
        }
    }
}
```

Important topics: - `DriverManager` and `DataSource`. - `Connection`,
`Statement`, `PreparedStatement`, `CallableStatement`. - `ResultSet`. -
Transactions, auto-commit, commit and rollback. - Batch updates. -
Connection pooling. - SQL injection prevention through parameter
binding.

------------------------------------------------------------------------

## 22. Networking and HTTP Basics

Understand these even when using Spring Boot: - TCP vs UDP. - DNS. -
TLS/HTTPS. - HTTP methods and status codes. - Headers, cookies and
content types. - JSON serialization. - Sockets and `java.net`. -
Timeouts and connection pooling. - REST principles.

Java 11's `HttpClient` supports synchronous and asynchronous HTTP calls.

``` java
HttpClient client = HttpClient.newHttpClient();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://example.com"))
    .GET()
    .build();

HttpResponse<String> response =
    client.send(request, HttpResponse.BodyHandlers.ofString());
```

Always configure suitable timeouts and handle network failures
explicitly in production.

------------------------------------------------------------------------

## 23. Testing Core Java

### 23.1 Unit testing

JUnit 5 concepts: - `@Test` - `@BeforeEach` - `@AfterEach` -
`@ParameterizedTest` - Assertions - Exception assertions

``` java
@Test
void addsNumbers() {
    assertEquals(5, calculator.add(2, 3));
}
```

### 23.2 Mocking

Mockito concepts: - `@Mock` - `@InjectMocks` -
`when(...).thenReturn(...)` - `verify(...)` - `ArgumentCaptor`

Mock external dependencies, not every object. Prefer testing meaningful
behavior rather than implementation details.

### 23.3 Other quality tools

-   Maven / Gradle.
-   Checkstyle / Spotless.
-   SpotBugs.
-   PMD.
-   SonarQube.
-   JaCoCo.
-   JMH for microbenchmarks.

------------------------------------------------------------------------

## 24. Performance and Troubleshooting

### 24.1 Common performance issues

-   Excessive object allocation.
-   Inefficient collection choices.
-   N+1 database access patterns.
-   Too many threads or poorly sized pools.
-   Blocking operations in shared executors.
-   Unbounded queues and caches.
-   Excessive logging.
-   Large payloads and unnecessary serialization.
-   Poor network/database timeout settings.

### 24.2 Useful diagnostic tools

  Tool                   Purpose
  ---------------------- -----------------------------------
  `jcmd`                 JVM diagnostics and commands
  `jstack`               Thread dump
  `jmap`                 Heap information/dump
  Java Flight Recorder   Runtime profiling and events
  Java Mission Control   Analyze recordings
  VisualVM               JVM monitoring and profiling
  `jstat`                JVM statistics
  JMH                    Reliable microbenchmark framework

### 24.3 Troubleshooting sequence

1.  Identify the symptom and affected time window.
2.  Check application logs and error rates.
3.  Inspect CPU, memory, GC and thread metrics.
4.  Capture thread dumps for hangs/high CPU.
5.  Capture and analyze heap dumps for memory growth.
6.  Check database, network and downstream latency.
7.  Form a hypothesis and reproduce where possible.
8.  Measure before and after the change.

Do not optimize based only on intuition; measure representative
workloads.

------------------------------------------------------------------------

## 25. Security Fundamentals

-   Never hard-code passwords, API keys or private keys.
-   Use a secret manager or secure runtime configuration.
-   Validate untrusted input.
-   Use parameterized SQL.
-   Avoid unsafe Java native/object deserialization of untrusted data.
-   Use TLS for sensitive network communication.
-   Apply least privilege.
-   Keep dependencies patched and scan for vulnerabilities.
-   Avoid logging secrets, tokens and sensitive personal data.
-   Use secure random number generation for security-sensitive tokens.
-   Understand cryptographic APIs and use established libraries rather
    than inventing algorithms.

------------------------------------------------------------------------

## 26. Core Java Interview Checklist -- 7 Years Experience

### Language and OOP

-   [ ] JDK vs JRE vs JVM
-   [ ] Primitive vs reference types
-   [ ] `==` vs `equals()`
-   [ ] `equals()` and `hashCode()` contract
-   [ ] Overloading vs overriding
-   [ ] Abstract class vs interface
-   [ ] `final`, `finally`, `finalize` (and why finalization is
    deprecated)
-   [ ] Immutability and String pool
-   [ ] Access modifiers
-   [ ] Composition vs inheritance

### Collections and Generics

-   [ ] ArrayList vs LinkedList
-   [ ] HashMap internals and collision handling
-   [ ] HashMap vs ConcurrentHashMap
-   [ ] HashSet vs TreeSet
-   [ ] Comparable vs Comparator
-   [ ] Fail-fast iterators
-   [ ] Generics and type erasure
-   [ ] PECS wildcard rule

### Java 8+

-   [ ] Functional interfaces
-   [ ] Lambda and method references
-   [ ] Optional
-   [ ] Stream API
-   [ ] `map()` vs `flatMap()`
-   [ ] `reduce()` vs `collect()`
-   [ ] Parallel streams
-   [ ] Date/Time API

### Concurrency

-   [ ] Thread lifecycle
-   [ ] Runnable vs Callable
-   [ ] ExecutorService and thread pools
-   [ ] synchronized vs volatile
-   [ ] Atomic classes
-   [ ] Locks and concurrent collections
-   [ ] wait/notify
-   [ ] CompletableFuture
-   [ ] Deadlock, livelock and starvation
-   [ ] Java Memory Model and happens-before
-   [ ] Virtual threads in Java 21

### JVM and operations

-   [ ] Heap, stack and metaspace
-   [ ] Garbage collection
-   [ ] Memory leaks
-   [ ] Class loading
-   [ ] JIT compiler
-   [ ] Thread dumps and heap dumps
-   [ ] JFR and profiling
-   [ ] JVM tuning basics

### Modern Java

-   [ ] Modules and `var`
-   [ ] HTTP Client
-   [ ] Records
-   [ ] Sealed classes
-   [ ] Pattern matching
-   [ ] Switch expressions
-   [ ] Text blocks
-   [ ] Virtual threads

### Design and engineering

-   [ ] SOLID
-   [ ] Creational, structural and behavioral patterns
-   [ ] JDBC and transactions
-   [ ] Unit testing
-   [ ] Performance profiling
-   [ ] Secure coding

------------------------------------------------------------------------

## 27. Suggested 30-Day Core Java Revision Plan

  -----------------------------------------------------------------------
  Days                    Focus                   Practical exercise
  ----------------------- ----------------------- -----------------------
  1--3                    Syntax, types, control  Employee and payment
                          flow, OOP               domain model

  4--6                    Strings, immutability,  Validation and custom
                          exceptions              exceptions

  7--10                   Collections and         In-memory employee
                          generics                search/index

  11--13                  Lambdas, Optional,      Group, sort and
                          Streams                 aggregate employee data

  14--16                  I/O, Date/Time, JDBC    CSV import and database
                                                  access

  17--21                  Threads and concurrency Bounded
                                                  producer-consumer
                                                  application

  22--24                  JVM and garbage         Capture thread/heap
                          collection              diagnostics

  25--27                  Java 11--21 features    Refactor examples using
                                                  records, switch and
                                                  virtual threads

  28--30                  Patterns, testing and   Build and test a small
                          revision                order-processing
                                                  service
  -----------------------------------------------------------------------

## 28. Recommended Practice Project

Build a **Core Java Order Processing Application** before moving into
Spring Boot.

Requirements: 1. Define customer, order, product and payment domain
classes. 2. Use enums for order/payment status. 3. Use immutable value
objects where appropriate. 4. Use collections and generics for in-memory
repositories. 5. Use custom exceptions for domain failures. 6. Use
Streams for reporting and aggregation. 7. Use JDBC to persist orders. 8.
Use an ExecutorService for independent background tasks. 9. Add unit
tests. 10. Profile and inspect the application.

This project connects language fundamentals, collections, exceptions,
concurrency, database access, design principles and testing.

------------------------------------------------------------------------

## 29. Recommended Official References

-   Java Documentation: https://docs.oracle.com/en/java/
-   Java Language Specification: https://docs.oracle.com/javase/specs/
-   Java API Documentation:
    https://docs.oracle.com/en/java/javase/21/docs/api/
-   Dev.java learning resources: https://dev.java/learn/
-   OpenJDK: https://openjdk.org/

**Study advice:** For a 7-year Java developer, do not spend all your
time memorizing syntax. Prioritize collection internals, concurrency,
JVM troubleshooting, Java 8+ functional programming, modern Java
features, API contracts and production problem-solving. Be able to
explain trade-offs and demonstrate them in code.

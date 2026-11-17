# Java Collections Framework and Stream API -- In-Depth Guide with Problems

**For Java 8, 11, 17 and 21 \| Senior Java / Spring Boot Developers**

This guide combines collection internals, data-structure selection,
complexity, equality and ordering contracts, generics, iterators,
concurrency, Stream API operations, collectors, performance, practical
coding problems and interview preparation.

------------------------------------------------------------------------

# Part I -- Java Collections Framework

## 1. What is the Collections Framework?

The Java Collections Framework (JCF) provides interfaces,
implementations and algorithms for storing and processing groups of
objects.

Main benefits: - Standard interfaces and reusable implementations. -
Common algorithms such as sorting and searching. - Type safety through
generics. - Interoperability between APIs. - Concurrent implementations
for multithreaded applications.

The framework is mainly in `java.util`; concurrent collections are in
`java.util.concurrent`.

### Collection hierarchy overview

``` text
Iterable
  |
Collection
  |-------------------|------------------|
 List                Set               Queue
  |                   |                  |
ArrayList          HashSet           PriorityQueue
LinkedList         LinkedHashSet     ArrayDeque
Vector             TreeSet           BlockingQueue*
Stack
```

`Map` is part of the Collections Framework but does **not** extend
`Collection`.

``` text
Map
 |------------------|--------------------|
HashMap          LinkedHashMap         SortedMap
Hashtable        ConcurrentHashMap       |
                                      NavigableMap
                                          |
                                        TreeMap
```

------------------------------------------------------------------------

## 2. Collection interfaces

  ----------------------------------------------------------------------------
  Interface        Main purpose                Duplicates Ordering
  ---------------- ---------------- --------------------- --------------------
  `List`           Indexed sequence               Allowed Usually
                                                          insertion/index
                                                          order

  `Set`            Unique elements         Not allowed by Depends on
                                        equality contract implementation

  `Queue`          Elements waiting       Usually allowed Depends on
                   for processing                         implementation

  `Deque`          Double-ended                   Allowed Head and tail
                   queue                                  operations

  `Map`            Key-value                  Keys unique Depends on
                   associations                           implementation

  `SortedSet`      Sorted unique              Not allowed Comparator/natural
                   elements                               order

  `NavigableSet`   Sorted set with            Not allowed Sorted
                   navigation                             

  `SortedMap`      Sorted key-value           Keys unique Sorted keys
                   map                                    

  `NavigableMap`   Sorted map with            Keys unique Sorted keys
                   navigation                             
  ----------------------------------------------------------------------------

### Common method families

Collection: - `add`, `remove`, `contains`, `size`, `isEmpty`, `clear` -
`addAll`, `removeAll`, `retainAll`, `containsAll` - `iterator`,
`toArray`, `stream`, `parallelStream`

List: - `get`, `set`, `add(index, value)`, `remove(index)` - `indexOf`,
`lastIndexOf`, `subList`

Queue: - `add` / `offer` - `remove` / `poll` - `element` / `peek`

Map: - `put`, `get`, `remove`, `containsKey`, `containsValue` -
`putIfAbsent`, `getOrDefault`, `replace` - `compute`, `computeIfAbsent`,
`computeIfPresent`, `merge` - `keySet`, `values`, `entrySet`

------------------------------------------------------------------------

## 3. List implementations

### 3.1 ArrayList

`ArrayList` is a resizable array.

``` java
List<String> names = new ArrayList<>();
names.add("Asha");
names.add("Ravi");
names.add("Asha");

String first = names.get(0);
```

Characteristics: - Maintains insertion/index order. - Allows duplicates
and null. - Fast random access. - Inserting/removing in the middle
shifts later elements. - Not thread-safe by default.

Typical time complexity:

  Operation                                    Complexity
  -------------------------------------- ----------------
  `get(index)` / `set(index)`                        O(1)
  Append                                   Amortized O(1)
  Insert/remove at beginning or middle               O(n)
  Search by value                                    O(n)
  Iteration                                          O(n)

**Capacity vs size** - Size is the number of elements stored. - Capacity
is the backing array space available before growth is needed. - Growth
policy is an implementation detail; do not write code that depends on an
exact capacity increment.

Use `ArrayList` as the default list for most ordinary application use
cases.

### 3.2 LinkedList

`LinkedList` is a doubly linked list and implements both `List` and
`Deque`.

``` java
Deque<String> deque = new LinkedList<>();
deque.addFirst("first");
deque.addLast("last");
String value = deque.removeFirst();
```

Characteristics: - Allows duplicates and null. - Indexed access is
O(n). - Adding/removing at a known end is O(1). - Higher memory overhead
per element than an array-backed list. - Not thread-safe.

Do not choose `LinkedList` merely because frequent insertion/removal is
expected. If locating the insertion point requires indexed traversal,
that traversal is O(n). For queue/stack behavior, `ArrayDeque` is often
a better default.

### 3.3 Vector and Stack

-   `Vector` is a legacy synchronized resizable array.
-   `Stack` extends `Vector` and is a legacy stack implementation.
-   Prefer `ArrayList` for list behavior and `ArrayDeque` for stack
    behavior.

``` java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(10);
stack.push(20);
int top = stack.pop();
```

------------------------------------------------------------------------

## 4. Set implementations

### 4.1 HashSet

`HashSet` stores unique elements using hash-based lookup. It is backed
by a `HashMap`.

``` java
Set<String> skills = new HashSet<>();
skills.add("Java");
skills.add("Spring");
skills.add("Java"); // duplicate ignored
```

Characteristics: - No guaranteed iteration order. - Allows one null
element. - Average expected O(1) add, remove and contains with good hash
distribution. - Not thread-safe. - Correct behavior depends on valid
`equals()` and `hashCode()` implementations.

### 4.2 LinkedHashSet

-   Maintains insertion iteration order.
-   Hash-based operations are expected O(1).
-   Allows one null.
-   Useful when uniqueness and predictable insertion order are both
    required.

### 4.3 TreeSet

`TreeSet` stores unique elements in sorted order, typically using a
red-black tree through its backing `TreeMap`.

``` java
Set<Integer> values = new TreeSet<>();
values.add(40);
values.add(10);
values.add(30);
// Iteration: 10, 30, 40
```

Characteristics: - O(log n) add, remove and contains. - Natural ordering
or supplied `Comparator`. - Null is generally not accepted with natural
ordering. - Uniqueness is based on comparison returning zero, which
should be consistent with equals for well-behaved Set semantics.

### Set comparison

  Feature                HashSet         LinkedHashSet     TreeSet
  ---------------------- --------------- ----------------- -----------------------------------
  Ordering               No guarantee    Insertion order   Sorted
  Typical add/search     Expected O(1)   Expected O(1)     O(log n)
  Null                   One allowed     One allowed       Usually not with natural ordering
  Implementation basis   HashMap         LinkedHashMap     TreeMap

------------------------------------------------------------------------

## 5. Map implementations -- very important

### 5.1 HashMap

`HashMap` stores key-value pairs. Keys are unique according to
`equals()` and `hashCode()`.

``` java
Map<String, Integer> scores = new HashMap<>();
scores.put("Ravi", 90);
scores.put("Asha", 95);
scores.put("Ravi", 98); // replaces old value
```

Characteristics: - Allows one null key and multiple null values. - No
iteration-order guarantee. - Expected O(1) get/put with good hash
distribution. - Not thread-safe. - One key maps to one current value.

### How HashMap works internally

At a high level: 1. Calls `hashCode()` on the key. 2. Applies a
hash-spreading operation. 3. Uses the hash and table capacity to
identify a bucket. 4. Checks keys in that bucket using hash and
equality. 5. Inserts or updates the mapping. 6. Resizes when the size
crosses the threshold.

A bucket can contain multiple entries due to hash collisions. Modern JDK
implementations can use a linked structure and may treeify heavily
populated buckets under implementation-specific conditions.

**Important:** exact implementation details can vary by JDK. Understand
the concepts; do not rely on private implementation constants in
application code.

### Collision

Two unequal keys can have the same hash code. A collision is valid.
`equals()` distinguishes keys within the same hash bucket.

### Why are equals() and hashCode() both important?

Contract: - If `a.equals(b)` is true, `a.hashCode() == b.hashCode()`
must also be true. - Equal objects must have equal hash codes. - Unequal
objects may have the same hash code. - Equality and hash code should
remain stable while an object is used as a map key.

If a key's hash-relevant fields are mutated after insertion,
lookup/removal may fail.

### Example key class

``` java
final class EmployeeKey {
    private final long id;

    EmployeeKey(long id) {
        this.id = id;
    }

    @Override
    public boolean equals(Object other) {
        if (this == other) return true;
        if (!(other instanceof EmployeeKey)) return false;
        EmployeeKey that = (EmployeeKey) other;
        return id == that.id;
    }

    @Override
    public int hashCode() {
        return Long.hashCode(id);
    }
}
```

### 5.2 LinkedHashMap

-   Predictable iteration order: insertion order by default.
-   Can be configured for access order.
-   Useful for caches and predictable map iteration.
-   Hash-based expected O(1) operations.

### 5.3 TreeMap

-   Sorts keys by natural order or comparator.
-   O(log n) get, put and remove.
-   Implements `NavigableMap`.
-   Null keys are generally not accepted with natural ordering.

Useful navigation methods: - `firstKey`, `lastKey` - `lowerKey`,
`floorKey` - `ceilingKey`, `higherKey` - `headMap`, `tailMap`, `subMap`

### 5.4 Hashtable

Legacy synchronized map: - Does not allow null keys or values. - Prefer
`ConcurrentHashMap` for concurrent access in modern applications.

### Map comparison

  -----------------------------------------------------------------------------------
  Feature        HashMap        LinkedHashMap      TreeMap        ConcurrentHashMap
  -------------- -------------- ------------------ -------------- -------------------
  Ordering       None           Insertion/access   Sorted keys    None guaranteed
                 guaranteed     order                             

  Typical        Expected O(1)  Expected O(1)      O(log n)       Expected O(1)
  get/put                                                         

  Null key/value One null key;  Same               Null key       Neither null keys
                 null values                       generally      nor values
                                                   unsupported    
                                                   with natural   
                                                   ordering       

  Thread-safe    No             No                 No             Yes, concurrent
                                                                  operations
  -----------------------------------------------------------------------------------

------------------------------------------------------------------------

## 6. Queue and Deque

### Queue method pairs

  Throws exception   Returns special value
  ------------------ -----------------------
  `add(e)`           `offer(e)`
  `remove()`         `poll()`
  `element()`        `peek()`

-   `offer` returns false if insertion cannot be made.
-   `poll` returns null if empty.
-   `peek` returns null if empty.
-   `remove` and `element` throw when empty.

### ArrayDeque

``` java
Deque<String> deque = new ArrayDeque<>();
deque.offerLast("A");
deque.offerLast("B");
String first = deque.pollFirst();
```

-   Resizable array-backed deque.
-   Efficient operations at both ends.
-   Does not allow null.
-   Not thread-safe.
-   Usually preferred over `Stack` for stack behavior and over
    `LinkedList` for ordinary queue/deque behavior.

### PriorityQueue

``` java
Queue<Integer> queue = new PriorityQueue<>();
queue.offer(40);
queue.offer(10);
queue.offer(30);

int smallest = queue.poll(); // 10
```

-   Head is the least element under natural ordering by default.
-   `offer`/`poll`: O(log n).
-   `peek`: O(1).
-   Iteration is not guaranteed to be sorted.
-   Does not allow null.
-   Not thread-safe.

### BlockingQueue

Used for producer-consumer workflows: - `ArrayBlockingQueue`: bounded. -
`LinkedBlockingQueue`: optionally bounded. - `PriorityBlockingQueue`:
priority ordering, logically unbounded. - `DelayQueue`: elements become
available after delay. - `SynchronousQueue`: direct handoff, no internal
capacity.

------------------------------------------------------------------------

## 7. Comparable and Comparator

### Comparable -- natural ordering

``` java
class Employee implements Comparable<Employee> {
    private final int id;

    Employee(int id) {
        this.id = id;
    }

    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.id, other.id);
    }
}
```

### Comparator -- external/custom ordering

``` java
Comparator<Employee> bySalary =
    Comparator.comparing(Employee::getSalary);

Comparator<Employee> bySalaryThenName =
    Comparator.comparing(Employee::getSalary)
              .thenComparing(Employee::getName);
```

Useful methods: - `comparing` - `comparingInt`, `comparingLong`,
`comparingDouble` - `thenComparing` - `reversed` - `nullsFirst`,
`nullsLast`

Avoid subtraction in comparators:

``` java
// Risk of integer overflow
(a, b) -> a.getAge() - b.getAge();

// Safer
Comparator.comparingInt(Employee::getAge);
```

For `TreeSet` and `TreeMap`, comparator equality (`compare(a,b) == 0`)
determines whether keys/elements are treated as the same sorted
key/element. Keep this consistent with `equals()` where practical.

------------------------------------------------------------------------

## 8. Iterator, ListIterator and fail-fast behavior

### Iterator

``` java
Iterator<String> iterator = names.iterator();
while (iterator.hasNext()) {
    String name = iterator.next();
    if (name.isBlank()) {
        iterator.remove();
    }
}
```

Use `Iterator.remove()` for supported removal during iteration.

### ListIterator

Supports: - Forward and backward traversal. - `add`, `set` and
`remove`. - Index information.

### Fail-fast

Many standard collection iterators are fail-fast and may throw
`ConcurrentModificationException` when structural modifications occur
outside the iterator during iteration.

Fail-fast behavior is best-effort bug detection, not a thread-safety
guarantee. Do not rely on it for correctness.

### Safe removal

``` java
list.removeIf(String::isBlank);
```

For concurrent iteration, choose a suitable concurrent collection and
understand its iterator semantics. For example, CopyOnWriteArrayList
iterators are snapshot-based.

------------------------------------------------------------------------

## 9. Generics and wildcards

Generics provide compile-time type safety.

``` java
List<String> names = new ArrayList<>();
names.add("Ravi");
// names.add(10); // compile-time error
```

### Invariance

`List<Integer>` is not a subtype of `List<Number>`.

### PECS rule

**Producer Extends, Consumer Super**

-   `? extends T`: useful when reading values as T.
-   `? super T`: useful when adding T values.

``` java
static double sum(List<? extends Number> values) {
    double total = 0;
    for (Number value : values) total += value.doubleValue();
    return total;
}

static void addDefaults(List<? super Integer> values) {
    values.add(10);
    values.add(20);
}
```

You can safely read from `? extends T` as T, but generally cannot add a
specific T (except null). With `? super T`, you can add T values, but
reads are only safely typed as Object.

### Type erasure

Generic type parameters are mostly erased at runtime. This is why you
cannot write `new T()` or `new List<String>[10]` directly.

------------------------------------------------------------------------

## 10. Collections utility methods

`Collections` provides algorithms and wrappers:

``` java
Collections.sort(list);
Collections.reverse(list);
Collections.shuffle(list);
Collections.frequency(list, "Java");
Collections.max(list);
Collections.min(list);
Collections.unmodifiableList(list);
Collections.synchronizedList(new ArrayList<>());
```

Important distinctions: - `Collections.unmodifiableList(list)` is an
unmodifiable **view**; changes to the backing list can still be
visible. - `List.of(...)` creates an unmodifiable list (Java 9+),
rejects nulls and is not a mutable list. - `Arrays.asList(array)`
returns a fixed-size list backed by the array; `set` works, but
add/remove do not. - `new ArrayList<>(Arrays.asList(array))` creates a
resizable copy.

------------------------------------------------------------------------

## 11. Complexity reference

Typical expected complexities; exact behavior depends on implementation
and workload.

  ---------------------------------------------------------------------------------
  Structure         Access/search             Add          Remove Notes
  --------------- --------------- --------------- --------------- -----------------
  ArrayList           Index O(1),          Append     Middle O(n) General-purpose
                      search O(n)  amortized O(1)                 list

  LinkedList              O(n) by       Ends O(1)       Ends O(1) Node overhead
                     index/search                                 

  HashSet           Expected O(1)   Expected O(1)   Expected O(1) Hash quality
                                                                  matters

  TreeSet                O(log n)        O(log n)        O(log n) Sorted

  HashMap           Expected O(1)   Expected O(1)   Expected O(1) Hash quality
                                                                  matters

  TreeMap                O(log n)        O(log n)        O(log n) Sorted keys

  PriorityQueue         Peek O(1)        O(log n)   Poll O(log n) Head has priority

  ArrayDeque            Ends O(1)       Ends O(1)       Ends O(1) Stack/queue
                        amortized       amortized       amortized 
  ---------------------------------------------------------------------------------

------------------------------------------------------------------------

# Part II -- Stream API

## 12. What is the Stream API?

Introduced in Java 8, Stream API supports declarative processing of data
sequences.

A stream: - Does not store data. - Usually processes data from a
collection, array, generator or I/O source. - Uses intermediate and
terminal operations. - Is consumed once. - Can be sequential or
parallel.

``` java
List<String> names = List.of("Ravi", "Asha", "Rahul");

List<String> result = names.stream()
    .filter(name -> name.startsWith("R"))
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

### Stream pipeline

``` text
Source -> Intermediate operations -> Terminal operation
List      filter -> map -> sorted      collect
```

### Intermediate vs terminal operations

  Intermediate (lazy)   Terminal (consumes stream)
  --------------------- -------------------------------------
  `filter`              `collect`
  `map`                 `toList`
  `flatMap`             `forEach`
  `distinct`            `count`
  `sorted`              `reduce`
  `limit`, `skip`       `min`, `max`
  `peek`                `findFirst`, `findAny`
  `mapToInt`            `anyMatch`, `allMatch`, `noneMatch`

After a terminal operation, the stream cannot be reused.

------------------------------------------------------------------------

## 13. Creating streams

``` java
list.stream();
list.parallelStream();

Stream.of("A", "B", "C");
Arrays.stream(new int[] {1, 2, 3});

Stream.iterate(1, n -> n + 1).limit(5);
Stream.generate(Math::random).limit(5);

IntStream.range(1, 5);       // 1,2,3,4
IntStream.rangeClosed(1, 5); // 1,2,3,4,5
```

Primitive streams: - `IntStream` - `LongStream` - `DoubleStream`

They avoid some boxing overhead and provide numeric operations such as
`sum`, `average`, `min`, `max` and `summaryStatistics`.

------------------------------------------------------------------------

## 14. Core intermediate operations

### filter

Keeps elements matching a predicate.

``` java
List<Integer> even = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());
```

### map

Transforms one element into one element.

``` java
List<String> upper = names.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

### flatMap

Transforms each element into a stream and flattens the streams into one
stream.

``` java
List<String> words = sentences.stream()
    .flatMap(sentence -> Arrays.stream(sentence.split(" ")))
    .collect(Collectors.toList());
```

### distinct

Removes duplicates using equality.

``` java
List<Integer> unique = numbers.stream()
    .distinct()
    .collect(Collectors.toList());
```

### sorted

``` java
employees.stream()
    .sorted(Comparator.comparing(Employee::getSalary).reversed())
    .collect(Collectors.toList());
```

### limit and skip

``` java
numbers.stream().skip(5).limit(10).collect(Collectors.toList());
```

### peek

Useful mainly for debugging/inspection; do not rely on it for required
side effects because pipeline execution and optimization may mean an
action is not run in the way you expect.

------------------------------------------------------------------------

## 15. Terminal operations

### forEach

``` java
names.stream().forEach(System.out::println);
```

`forEach` does not promise encounter order for parallel streams. Use
`forEachOrdered` when encounter order matters.

### count, min, max

``` java
long count = numbers.stream().filter(n -> n > 10).count();

Optional<Integer> max = numbers.stream().max(Integer::compareTo);
```

### findFirst and findAny

Return `Optional<T>`. - `findFirst` respects encounter order when one
exists. - `findAny` may return any element and can be more flexible for
parallel execution.

### Matching

``` java
boolean any = numbers.stream().anyMatch(n -> n > 100);
boolean all = numbers.stream().allMatch(n -> n > 0);
boolean none = numbers.stream().noneMatch(n -> n < 0);
```

Be aware: `allMatch` returns true for an empty stream; `anyMatch`
returns false; `noneMatch` returns true.

------------------------------------------------------------------------

## 16. Collectors -- most important for interviews

### toList / toSet

``` java
List<String> list = stream.collect(Collectors.toList());
Set<String> set = stream.collect(Collectors.toSet());
```

`Collectors.toList()` does not guarantee a specific concrete list type
or mutability contract. In Java 16+, `Stream.toList()` returns an
unmodifiable list.

### toMap

``` java
Map<Long, String> namesById = employees.stream()
    .collect(Collectors.toMap(
        Employee::getId,
        Employee::getName
    ));
```

If keys can repeat, provide a merge function:

``` java
Map<String, Integer> totalByDepartment = employees.stream()
    .collect(Collectors.toMap(
        Employee::getDepartment,
        Employee::getSalary,
        Integer::sum
    ));
```

### groupingBy

``` java
Map<String, List<Employee>> byDepartment = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));
```

### counting

``` java
Map<String, Long> countByDepartment = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()
    ));
```

### summing and averaging

``` java
Map<String, Integer> salaryByDepartment = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.summingInt(Employee::getSalary)
    ));
```

### averaging

``` java
Map<String, Double> averageSalary = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingInt(Employee::getSalary)
    ));
```

### maxBy and minBy

``` java
Map<String, Optional<Employee>> highestByDepartment =
    employees.stream().collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.maxBy(Comparator.comparingInt(Employee::getSalary))
    ));
```

### mapping

``` java
Map<String, Set<String>> employeeNamesByDepartment =
    employees.stream().collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.toSet())
    ));
```

### partitioningBy

Partitions into `true` and `false` groups:

``` java
Map<Boolean, List<Employee>> partition =
    employees.stream().collect(Collectors.partitioningBy(
        e -> e.getSalary() >= 100000
    ));
```

### joining

``` java
String names = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", "));
```

### summarizing

``` java
IntSummaryStatistics stats = employees.stream()
    .collect(Collectors.summarizingInt(Employee::getSalary));

long count = stats.getCount();
long sum = stats.getSum();
double average = stats.getAverage();
int min = stats.getMin();
int max = stats.getMax();
```

### collectingAndThen

``` java
List<String> immutable = names.stream()
    .collect(Collectors.collectingAndThen(
        Collectors.toList(),
        Collections::unmodifiableList
    ));
```

------------------------------------------------------------------------

## 17. reduce

`reduce` combines stream elements into one result.

``` java
int sum = numbers.stream().reduce(0, Integer::sum);
```

Other examples:

``` java
Optional<Integer> maximum =
    numbers.stream().reduce(Integer::max);

String joined = words.stream().reduce("", String::concat);
```

Prefer built-in operations such as `sum`, `count`, `max` and
`Collectors.joining` when they express the intent more clearly.

For parallel reduction, the identity and accumulator/combiner must obey
the required associativity and identity rules. Do not use mutable shared
state inside `reduce`.

------------------------------------------------------------------------

## 18. Optional with streams

Methods: - `of`, `ofNullable`, `empty` - `isPresent`, `ifPresent` -
`orElse`, `orElseGet`, `orElseThrow` - `map`, `flatMap`, `filter`

``` java
String name = employees.stream()
    .filter(e -> e.getId() == 10)
    .findFirst()
    .map(Employee::getName)
    .orElse("Unknown");
```

`orElse(value)` evaluates the fallback expression eagerly.
`orElseGet(supplier)` evaluates it only when the Optional is empty.

------------------------------------------------------------------------

## 19. Primitive streams and boxing

``` java
int total = employees.stream()
    .mapToInt(Employee::getSalary)
    .sum();
```

`mapToInt`, `mapToLong` and `mapToDouble` produce primitive streams. Use
them for numeric processing to avoid unnecessary boxing/unboxing.

Convert back: - `boxed()` - `mapToObj(...)`

------------------------------------------------------------------------

## 20. Parallel streams

``` java
long count = numbers.parallelStream()
    .filter(n -> n % 2 == 0)
    .count();
```

Parallel streams can help for sufficiently large, CPU-bound, independent
workloads. They can hurt performance when: - Data sets are small. - Work
per element is cheap. - Operations block on I/O. - Shared mutable state
is used. - Splitting/combining overhead is high. - Ordering constraints
limit parallelism.

Avoid side effects:

``` java
// Unsafe pattern
List<Integer> output = new ArrayList<>();
numbers.parallelStream().forEach(output::add);
```

Prefer collectors:

``` java
List<Integer> output = numbers.parallelStream()
    .filter(n -> n > 10)
    .collect(Collectors.toList());
```

Do not assume parallel streams use an application-specific executor.
Their common-pool behavior should be considered when used in server
applications.

------------------------------------------------------------------------

# Part III -- Practical Coding Problems

Use these sample models for the employee exercises:

``` java
class Employee {
    private final int id;
    private final String name;
    private final String department;
    private final int age;
    private final int salary;

    // Constructor and getters omitted for brevity
}
```

Assume `employees` is a `List<Employee>`.

## 21. Collection coding problems

### Problem 1 -- Remove duplicates while preserving order

Input:

``` text
[Java, SQL, Java, Spring, SQL, AWS]
```

Expected:

``` text
[Java, SQL, Spring, AWS]
```

Hint: `LinkedHashSet`.

### Problem 2 -- Find duplicate elements in a list

Input:

``` text
[1, 2, 3, 2, 4, 1, 5]
```

Expected duplicates:

``` text
[1, 2]
```

Try solving with: 1. Two `HashSet` instances. 2. `Map<T, Long>`
frequency counting. 3. Streams.

### Problem 3 -- Character frequency

Input:

``` text
"programming"
```

Expected counts:

``` text
p=1, r=2, o=1, g=2, a=1, m=2, i=1, n=1
```

Hint: `Map<Character, Integer>` and `merge`.

### Problem 4 -- First non-repeated character

Input:

``` text
"swiss"
```

Expected:

``` text
'w'
```

Hint: `LinkedHashMap` to preserve first-seen order.

### Problem 5 -- Sort employees by salary descending, then name ascending

Hint:

``` java
Comparator.comparingInt(Employee::getSalary)
          .reversed()
          .thenComparing(Employee::getName)
```

### Problem 6 -- Find the second-highest distinct number

Input:

``` text
[10, 20, 20, 30, 40, 40]
```

Expected:

``` text
30
```

Handle lists with fewer than two distinct values.

### Problem 7 -- Merge two maps by summing duplicate keys

Input:

``` text
Map 1: {Java=10, SQL=5}
Map 2: {Java=7, AWS=3}
```

Expected:

``` text
{Java=17, SQL=5, AWS=3}
```

Hint: `Map.merge`.

### Problem 8 -- Group words by length

Input:

``` text
["Java", "Go", "Spring", "AI", "Python"]
```

Expected:

``` text
2 -> [Go, AI]
4 -> [Java]
6 -> [Spring, Python]
```

### Problem 9 -- Implement an LRU cache

Requirements: - Fixed capacity. - `get(key)` returns value and marks key
recently used. - `put(key,value)` inserts/updates. - Evict least
recently used entry when capacity is exceeded.

Hint: `LinkedHashMap` with access order, or implement a HashMap + doubly
linked list for O(1) operations.

### Problem 10 -- Sort a map by values

Input:

``` text
{A=30, B=10, C=20}
```

Expected entries ordered by value:

``` text
B=10, C=20, A=30
```

Hint: stream `entrySet()` and comparator.

------------------------------------------------------------------------

## 22. Stream API coding problems

### Problem 11 -- Filter even numbers and square them

Input:

``` text
[1, 2, 3, 4, 5, 6]
```

Expected:

``` text
[4, 16, 36]
```

### Problem 12 -- Find names beginning with A

Input:

``` text
["Asha", "Ravi", "Anil", "John"]
```

Expected:

``` text
["Asha", "Anil"]
```

### Problem 13 -- Convert list of strings to uppercase

Use `map`.

### Problem 14 -- Flatten a list of lists

Input:

``` text
[[1, 2], [3, 4], [5]]
```

Expected:

``` text
[1, 2, 3, 4, 5]
```

Hint: `flatMap`.

### Problem 15 -- Find the second-highest distinct number

Use `distinct`, `sorted` and `skip`, or solve with a single-pass
approach.

### Problem 16 -- Find the longest string

Return an `Optional<String>` and handle an empty input.

### Problem 17 -- Count each word

Input:

``` text
["java", "sql", "java", "aws", "sql", "java"]
```

Expected:

``` text
java=3, sql=2, aws=1
```

Hint: `groupingBy(identity(), counting())`.

### Problem 18 -- Partition numbers into even and odd

Use `partitioningBy`.

### Problem 19 -- Join employee names with commas

Use `Collectors.joining(", ")`.

### Problem 20 -- Find the highest-paid employee in each department

Return:

``` text
Map<String, Optional<Employee>>
```

Hint: `groupingBy` + `maxBy`.

### Problem 21 -- Average salary by department

Return:

``` text
Map<String, Double>
```

Hint: `groupingBy` + `averagingInt`.

### Problem 22 -- Department-wise employee names

Return:

``` text
Map<String, List<String>>
```

Hint: `groupingBy` + `mapping`.

### Problem 23 -- Find the top three highest-paid employees

Use sorting and limiting. Decide how ties should be handled.

### Problem 24 -- Find employees whose salary is above department average

Approach: 1. Calculate average salary per department. 2. Filter
employees using the matching department average. 3. Return the
qualifying employees.

### Problem 25 -- Find the most frequent element

Use a frequency map and select the maximum count. Define tie behavior.

------------------------------------------------------------------------

## 23. Advanced Stream and collection problems

### Problem 26 -- Transaction summary

Given transactions with account ID, type and amount: - Group by
account. - Calculate total credit and debit. - Calculate net balance. -
Return accounts sorted by net balance descending.

### Problem 27 -- Multi-level grouping

Group employees by department, then by age band.

Expected conceptual structure:

``` text
Department -> Age band -> Employees
```

Hint: nested `groupingBy`.

### Problem 28 -- Find top N values efficiently

Compare: - Full sorting: O(n log n). - Bounded priority queue: O(n log
k) for top k.

Implement top 5 salaries without sorting the entire data set.

### Problem 29 -- Merge overlapping intervals

Input:

``` text
[[1,3], [2,6], [8,10], [9,12]]
```

Expected:

``` text
[[1,6], [8,12]]
```

Hint: sort by interval start, then scan and merge.

### Problem 30 -- Build an index

Given a list of products, build:

``` text
Map<Category, Map<Brand, List<Product>>>
```

Use nested grouping or explicit loops. Compare readability.

------------------------------------------------------------------------

## 24. Solutions and reference approaches

### Solution 1 -- Remove duplicates preserving order

``` java
List<String> result = new ArrayList<>(new LinkedHashSet<>(input));
```

### Solution 2 -- Find duplicates

``` java
Set<Integer> seen = new HashSet<>();
Set<Integer> duplicates = new LinkedHashSet<>();

for (Integer value : input) {
    if (!seen.add(value)) {
        duplicates.add(value);
    }
}
```

### Solution 3 -- Character frequency

``` java
Map<Character, Long> frequency = text.chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(
        Function.identity(),
        LinkedHashMap::new,
        Collectors.counting()
    ));
```

### Solution 4 -- First non-repeated character

``` java
Map<Character, Long> frequency = text.chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(
        Function.identity(),
        LinkedHashMap::new,
        Collectors.counting()
    ));

Optional<Character> firstUnique = frequency.entrySet().stream()
    .filter(e -> e.getValue() == 1)
    .map(Map.Entry::getKey)
    .findFirst();
```

### Solution 5 -- Employee sorting

``` java
List<Employee> sorted = employees.stream()
    .sorted(Comparator.comparingInt(Employee::getSalary)
        .reversed()
        .thenComparing(Employee::getName))
    .collect(Collectors.toList());
```

### Solution 6 -- Second-highest distinct number

``` java
Optional<Integer> secondHighest = numbers.stream()
    .distinct()
    .sorted(Comparator.reverseOrder())
    .skip(1)
    .findFirst();
```

### Solution 7 -- Merge maps

``` java
Map<String, Integer> merged = new HashMap<>(first);

second.forEach((key, value) ->
    merged.merge(key, value, Integer::sum)
);
```

### Solution 8 -- Group by string length

``` java
Map<Integer, List<String>> grouped = words.stream()
    .collect(Collectors.groupingBy(String::length));
```

### Solution 9 -- LRU cache using LinkedHashMap

``` java
class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    LruCache(int capacity) {
        super(capacity, 0.75f, true); // access order
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

This simple implementation is not thread-safe. Synchronize access or use
a production caching library for concurrent production use.

### Solution 10 -- Sort map by values

``` java
List<Map.Entry<String, Integer>> sortedEntries = map.entrySet().stream()
    .sorted(Map.Entry.comparingByValue())
    .collect(Collectors.toList());
```

### Solution 11 -- Even squares

``` java
List<Integer> result = numbers.stream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * n)
    .collect(Collectors.toList());
```

### Solution 14 -- Flatten nested lists

``` java
List<Integer> flattened = nested.stream()
    .flatMap(Collection::stream)
    .collect(Collectors.toList());
```

### Solution 17 -- Word counts

``` java
Map<String, Long> counts = words.stream()
    .collect(Collectors.groupingBy(
        Function.identity(),
        Collectors.counting()
    ));
```

### Solution 18 -- Even/odd partition

``` java
Map<Boolean, List<Integer>> partitioned = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));
```

### Solution 20 -- Highest-paid employee per department

``` java
Map<String, Optional<Employee>> highest = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.maxBy(Comparator.comparingInt(Employee::getSalary))
    ));
```

### Solution 21 -- Average salary by department

``` java
Map<String, Double> averages = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingInt(Employee::getSalary)
    ));
```

### Solution 22 -- Names by department

``` java
Map<String, List<String>> namesByDepartment = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.toList())
    ));
```

### Solution 23 -- Top three employees

``` java
List<Employee> topThree = employees.stream()
    .sorted(Comparator.comparingInt(Employee::getSalary).reversed())
    .limit(3)
    .collect(Collectors.toList());
```

This returns at most three employees, not necessarily three distinct
salary levels.

### Solution 24 -- Above department average

``` java
Map<String, Double> averageByDepartment = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingInt(Employee::getSalary)
    ));

List<Employee> aboveAverage = employees.stream()
    .filter(e -> e.getSalary() > averageByDepartment.get(e.getDepartment()))
    .collect(Collectors.toList());
```

### Solution 25 -- Most frequent element

``` java
Optional<Map.Entry<String, Long>> mostFrequent = words.stream()
    .collect(Collectors.groupingBy(
        Function.identity(),
        Collectors.counting()
    ))
    .entrySet()
    .stream()
    .max(Map.Entry.comparingByValue());
```

If multiple elements tie, define a secondary comparator to make the
result deterministic.

------------------------------------------------------------------------

# Part IV -- Best Practices and Interview Preparation

## 25. Choosing the right collection

  Requirement                       Good starting choice
  --------------------------------- ------------------------
  General-purpose indexed list      `ArrayList`
  Unique values, order irrelevant   `HashSet`
  Unique values, insertion order    `LinkedHashSet`
  Unique sorted values              `TreeSet`
  Key-value lookup                  `HashMap`
  Predictable map iteration         `LinkedHashMap`
  Sorted keys and range queries     `TreeMap`
  Stack or deque                    `ArrayDeque`
  Priority scheduling               `PriorityQueue`
  Concurrent key-value access       `ConcurrentHashMap`
  Producer-consumer                 `BlockingQueue`
  Read-mostly list                  `CopyOnWriteArrayList`

Choose based on required ordering, uniqueness, concurrency, null
behavior, memory and workload---not habit.

## 26. Common mistakes

  -----------------------------------------------------------------------
  Mistake                             Better approach
  ----------------------------------- -----------------------------------
  Assuming HashMap preserves          Use LinkedHashMap
  insertion order                     

  Using a mutable object as a map key Use stable immutable key fields

  Overriding equals without hashCode  Implement both contracts

  Using LinkedList for every          Consider locating cost and
  insertion-heavy case                ArrayDeque

  Assuming TreeSet uses equals for    Understand comparator result zero
  uniqueness                          

  Modifying a list during enhanced    Use iterator removal or removeIf
  for loop                            

  Reusing a consumed stream           Create a new stream from the source

  Using peek for required business    Use explicit terminal processing
  side effects                        

  Using toMap with duplicate keys and Supply a merge function
  no merge function                   

  Mutating shared state in parallel   Use collectors or concurrency-safe
  streams                             designs

  Using parallel streams for blocking Use an explicit concurrency
  I/O                                 strategy

  Assuming Stream.toList() is mutable It returns an unmodifiable list
  -----------------------------------------------------------------------

## 27. Senior-level interview questions

### Collections

1.  Explain the Collection hierarchy. Why is Map separate?
2.  ArrayList vs LinkedList?
3.  HashSet vs LinkedHashSet vs TreeSet?
4.  HashMap internal working?
5.  How are collisions handled?
6.  What happens when two keys have the same hash code?
7.  Why must equals and hashCode be consistent?
8.  What happens if a mutable key is changed after insertion?
9.  HashMap vs ConcurrentHashMap?
10. HashMap vs LinkedHashMap vs TreeMap?
11. How does TreeMap maintain order?
12. Comparable vs Comparator?
13. What is fail-fast behavior?
14. Iterator vs ListIterator?
15. Arrays.asList vs List.of vs new ArrayList?
16. Why is ArrayDeque often preferred over Stack?
17. What is CopyOnWriteArrayList?
18. How does BlockingQueue support producer-consumer?
19. Explain PECS in generics.
20. What is type erasure?

### Stream API

21. What is a stream and how is it different from a collection?
22. Intermediate vs terminal operations?
23. Why are intermediate operations lazy?
24. map vs flatMap?
25. filter vs map?
26. findFirst vs findAny?
27. anyMatch vs allMatch vs noneMatch?
28. reduce vs collect?
29. groupingBy vs partitioningBy?
30. How do you handle duplicate keys in toMap?
31. What is a downstream collector?
32. What is the difference between Stream.toList and Collectors.toList?
33. Why should streams avoid shared mutable state?
34. When should parallel streams be avoided?
35. How do primitive streams reduce boxing?
36. Can a stream be reused after a terminal operation?
37. What are short-circuiting operations?
38. What is the role of encounter order?

## 28. Suggested 21-day learning plan

  Days     Focus                                Practice
  -------- ------------------------------------ ------------------------------------
  1--2     Collection hierarchy and List        ArrayList/LinkedList problems
  3--4     Set and Map                          Duplicates, frequency, HashMap
  5--6     HashMap internals and equality       Custom key class
  7        Queue, Deque and PriorityQueue       Stack and scheduling problems
  8        Comparable, Comparator and sorting   Employee sorting
  9        Iterators and fail-fast              Safe removal
  10       Generics and PECS                    Generic utility methods
  11       Stream fundamentals                  filter/map/flatMap
  12       Terminal operations and Optional     Search and matching
  13--14   Collectors                           grouping/counting/mapping
  15       reduce and primitive streams         Numeric problems
  16       Parallel streams                     Benchmark and side-effect pitfalls
  17--18   Coding problems                      Problems 1--20
  19       Advanced problems                    LRU, top K, interval merge
  20       Interview revision                   Explain internals and complexity
  21       Mini project                         Employee analytics/reporting

## 29. Mini project -- Employee Analytics

Build a Java application that: 1. Stores employee records in
collections. 2. Supports lookup by employee ID. 3. Rejects duplicate
IDs. 4. Groups employees by department. 5. Calculates department-wise
headcount and average salary. 6. Finds highest-paid employee in each
department. 7. Lists top five salaries. 8. Finds employees earning above
their department average. 9. Sorts by salary and name. 10. Exports a
department summary.

Use `Map` for indexing, `Set` for uniqueness, `List` for ordered data
and Stream collectors for reporting. Add unit tests for empty input,
duplicate IDs, null fields and salary ties.

## 30. Official references

-   Java Collections Framework:
    https://docs.oracle.com/javase/8/docs/technotes/guides/collections/
-   Java 21 Collection API:
    https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/package-summary.html
-   Stream API:
    https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/package-summary.html
-   Collectors API:
    https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Collectors.html
-   Java tutorials: https://dev.java/learn/

**Senior developer priority:** Be able to select the right collection
from requirements, explain HashMap and equality contracts, reason about
time complexity, use generics safely, write readable stream pipelines,
master grouping/partitioning/toMap collectors, and recognize when
ordinary streams or parallel streams are the wrong tool.

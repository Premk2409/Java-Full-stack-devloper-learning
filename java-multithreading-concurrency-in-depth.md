# Java Multithreading and Concurrency -- In-Depth Guide

**For Java 8, 11, 17 and 21 \| Senior Java / Spring Boot Developers**

## 1. Processes, threads and concurrency

A **process** is an executing program with its own address space and
resources. A **thread** is an execution path within a process. Threads
share process heap/resources, while each thread has its own stack and
execution state.

-   **Concurrency:** Multiple tasks make progress over overlapping
    periods.
-   **Parallelism:** Multiple tasks execute simultaneously on different
    CPU cores.

Use concurrency for request handling, independent tasks and overlapping
I/O waits. It introduces race conditions, visibility issues, deadlocks
and resource contention.

## 2. Creating threads

### Extending Thread

``` java
class Worker extends Thread {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName());
    }
}
new Worker().start();
```

### Implementing Runnable

``` java
Runnable task = () -> System.out.println("Working");
new Thread(task).start();
```

### Runnable vs Callable

  -----------------------------------------------------------------------
  Runnable                            Callable
  ----------------------------------- -----------------------------------
  `run()` returns no result           `call()` returns a result

  Cannot declare checked exceptions   Can throw checked exceptions
  in signature                        

  Can be passed to Thread/executor    Commonly submitted to executor

  No result by itself                 Result retrieved through Future
  -----------------------------------------------------------------------

`start()` starts a new thread; direct `run()` invocation is an ordinary
method call. A Thread object can only be started once.

## 3. Thread lifecycle

  State           Meaning
  --------------- -----------------------------------------
  NEW             Created, not started
  RUNNABLE        Executing or eligible to execute
  BLOCKED         Waiting to acquire an intrinsic monitor
  WAITING         Waiting indefinitely for another action
  TIMED_WAITING   Waiting for a bounded time
  TERMINATED      Execution completed

Java does not expose a separate RUNNING state. `sleep()` normally causes
TIMED_WAITING; `wait()` without timeout causes WAITING; waiting to enter
a synchronized block can cause BLOCKED.

## 4. Important Thread methods

  Method                   Purpose
  ------------------------ ----------------------------------------------------
  `start()`                Starts a new thread
  `run()`                  Task body
  `sleep()`                Pauses current thread; does not release held locks
  `join()`                 Waits for another thread
  `interrupt()`            Requests cooperative interruption
  `isInterrupted()`        Checks interrupt status without clearing it
  `Thread.interrupted()`   Checks and clears current thread's status
  `currentThread()`        Returns current thread
  `yield()`                Scheduling hint; no guarantee

Interruption is a cancellation signal, not forced termination.

``` java
try {
    Thread.sleep(1000);
} catch (InterruptedException ex) {
    Thread.currentThread().interrupt();
    return;
}
```

If a method cannot propagate `InterruptedException`, restore the
interrupt status. Avoid deprecated `stop()`, `suspend()` and `resume()`.

## 5. Race conditions and thread safety

A race condition occurs when results depend on the timing of concurrent
operations.

``` java
class Counter {
    int count;
    void increment() {
        count++; // read-modify-write; not atomic
    }
}
```

Two threads can read the same value and overwrite each other's updates.

Solutions include: - `synchronized` - `Lock` - Atomic classes - Thread
confinement - Immutable state - Concurrent collections - Message passing
/ queues

Thread safety means the object behaves correctly under concurrent access
according to its contract.

## 6. synchronized and intrinsic locks

``` java
class SafeCounter {
    private int count;

    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }
}
```

-   A synchronized instance method locks `this`.
-   A static synchronized method locks the `Class` object.
-   A synchronized block can lock a specific private lock object.
-   Only one thread at a time owns a particular monitor.
-   Exiting a synchronized block releases the monitor.
-   Unlocking a monitor happens-before a subsequent lock of the same
    monitor.

Use private lock objects. Avoid synchronizing on public objects or
string literals. Keep critical sections short.

## 7. volatile and visibility

`volatile` provides visibility and ordering guarantees for reads/writes
of that variable.

``` java
class Worker {
    private volatile boolean running = true;

    void stop() {
        running = false;
    }

    void work() {
        while (running) {
            doWork();
        }
    }
}
```

`volatile` does **not** make compound operations atomic:

``` java
volatile int count;
count++; // still not atomic
```

Use `AtomicInteger`, synchronization or a lock for read-modify-write
operations.

  -----------------------------------------------------------------------
  volatile                            synchronized
  ----------------------------------- -----------------------------------
  Visibility and ordering for         Mutual exclusion and visibility
  variable                            

  No compound-operation atomicity     Protects critical sections

  No monitor lock                     Acquires monitor
  -----------------------------------------------------------------------

## 8. Java Memory Model and happens-before

The Java Memory Model defines visibility and ordering rules across
threads.

Important concepts: - **Atomicity:** operation happens indivisibly. -
**Visibility:** one thread observes another thread's writes. -
**Ordering:** required operation ordering is respected. - **Data race:**
conflicting accesses without a suitable happens-before ordering, with at
least one write.

Happens-before examples: 1. Program order within a thread. 2. Unlock of
a monitor happens-before a subsequent lock of the same monitor. 3. A
volatile write happens-before a subsequent read of that variable. 4.
Calling `Thread.start()` happens-before actions in the started thread.
5. Actions in a thread happen-before another thread successfully returns
from joining it. 6. Happens-before is transitive.

Safe publication mechanisms include static initialization, volatile
references, correctly constructed final fields, synchronized handoff,
concurrent collections and executor submission mechanisms with
documented guarantees.

## 9. Atomic variables and CAS

Common types: `AtomicInteger`, `AtomicLong`, `AtomicBoolean`,
`AtomicReference`, `LongAdder`, `LongAccumulator`.

``` java
AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();
counter.addAndGet(5);

boolean updated = counter.compareAndSet(10, 20);
```

CAS (Compare-And-Set) updates only if the current value equals the
expected value. It does not automatically make a multi-step business
operation atomic.

-   `AtomicInteger`: single atomic value.
-   `LongAdder`: often better for high-contention statistics; `sum()` is
    not an atomic snapshot during concurrent updates.

## 10. Explicit locks

``` java
class Account {
    private final Lock lock = new ReentrantLock();
    private int balance;

    void deposit(int amount) {
        lock.lock();
        try {
            balance += amount;
        } finally {
            lock.unlock();
        }
    }
}
```

`ReentrantLock` supports `tryLock`, timed acquisition, interruptible
acquisition, optional fairness and multiple Conditions. Always unlock in
`finally`.

  ReentrantLock               synchronized
  --------------------------- ---------------------------------
  Explicit lock/unlock        Automatic monitor management
  Timed/try lock              No timed monitor acquisition
  Interruptible acquisition   Monitor entry not interruptible
  Optional fairness           No fairness guarantee
  Multiple Conditions         `wait`/`notify` on monitor

Other locks: - `ReadWriteLock`: multiple readers or one writer. -
`ReentrantReadWriteLock`: common implementation. - `StampedLock`:
read/write and optimistic reads; not reentrant. Validate optimistic
reads.

## 11. wait, notify and notifyAll

These `Object` methods must be called while owning the object's monitor.

-   `wait()` releases that monitor and waits.
-   `notify()` wakes one waiter; it does not immediately release the
    monitor.
-   `notifyAll()` wakes all waiters; they compete to reacquire the
    monitor.

Always test the condition in a loop:

``` java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
    process();
}
```

A loop handles spurious wakeups and another thread changing the
condition before the awakened thread reacquires the lock. Prefer
`BlockingQueue` for producer-consumer workflows.

## 12. Deadlock, livelock and starvation

-   **Deadlock:** threads wait indefinitely for locks held by each
    other.
-   **Livelock:** threads remain active but make no useful progress.
-   **Starvation:** a thread repeatedly fails to obtain CPU time or a
    needed resource.

Prevent deadlocks with consistent lock ordering, short critical
sections, fewer nested locks, timed `tryLock` where appropriate and
avoiding unknown external calls while holding locks.

## 13. Executor framework

Executors separate task submission from thread management.

``` java
ExecutorService executor = Executors.newFixedThreadPool(4);
try {
    executor.submit(() -> processOrder());
} finally {
    executor.shutdown();
}
```

Important APIs: `execute`, `submit`, `invokeAll`, `invokeAny`,
`shutdown`, `shutdownNow`, `awaitTermination`.

-   `shutdown()` stops accepting new tasks but lets submitted tasks
    finish.
-   `shutdownNow()` requests interruption and returns tasks that never
    commenced; it cannot guarantee immediate termination.

Graceful shutdown:

``` java
executor.shutdown();
try {
    if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
        executor.shutdownNow();
    }
} catch (InterruptedException ex) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

### Common executor factories

  -----------------------------------------------------------------------
  Executor                Behavior                Risk/consideration
  ----------------------- ----------------------- -----------------------
  Fixed thread pool       Fixed workers           Unbounded queue

  Single thread executor  One worker              Unbounded queue

  Cached thread pool      Grows as needed         Can grow very large

  Scheduled pool          Delayed/periodic tasks  Scheduling semantics

  Work-stealing pool      Fork/join work stealing Suitable for
                                                  decomposable tasks
  -----------------------------------------------------------------------

For production, configure `ThreadPoolExecutor` deliberately:

``` java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4, 8, 60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(200),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

Parameters: core/max pool size, keep-alive time, work queue, thread
factory and rejection handler.

Rejection policies: - `AbortPolicy`: throws
`RejectedExecutionException`. - `CallerRunsPolicy`: submitting thread
executes task (unless shut down). - `DiscardPolicy`: silently
discards. - `DiscardOldestPolicy`: discards oldest queued task and
retries.

Silent discard is dangerous for business-critical work. Bounded queues
help apply backpressure.

## 14. Callable, Future and CompletableFuture

``` java
ExecutorService executor = Executors.newFixedThreadPool(2);
try {
    Future<Integer> future = executor.submit(() -> 100 + 200);
    Integer result = future.get(2, TimeUnit.SECONDS);
} finally {
    executor.shutdown();
}
```

`Future.get()` blocks until completion or timeout. Cancellation is
cooperative.

### CompletableFuture

``` java
CompletableFuture<String> future =
    CompletableFuture.supplyAsync(() -> loadCustomer(), ioExecutor)
        .thenApply(Customer::getName)
        .thenApply(String::toUpperCase)
        .exceptionally(ex -> "UNKNOWN");
```

  Method            Purpose
  ----------------- ---------------------------------------
  `supplyAsync`     Async task returning a value
  `runAsync`        Async task without result
  `thenApply`       Transform result
  `thenAccept`      Consume result
  `thenRun`         Next action without result
  `thenCompose`     Chain another async stage and flatten
  `thenCombine`     Combine independent stages
  `allOf`           Complete when all complete
  `anyOf`           Complete when any completes
  `exceptionally`   Fallback on failure
  `handle`          Handle success or failure
  `whenComplete`    Observe completion

`thenApply` transforms `T -> U`; `thenCompose` chains
`T -> CompletionStage<U>`. Use explicit executors for blocking I/O.
Default async methods commonly use the common ForkJoinPool, which may be
unsuitable for blocking work.

## 15. ForkJoinPool and parallel streams

Fork/join splits work into smaller tasks and combines results using work
stealing.

-   `RecursiveTask<V>` returns a result.
-   `RecursiveAction` returns no result.

``` java
class SumTask extends RecursiveTask<Long> {
    private final long[] values;
    private final int start, end;

    SumTask(long[] values, int start, int end) {
        this.values = values;
        this.start = start;
        this.end = end;
    }

    protected Long compute() {
        if (end - start <= 1000) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += values[i];
            return sum;
        }
        int mid = (start + end) / 2;
        SumTask left = new SumTask(values, start, mid);
        SumTask right = new SumTask(values, mid, end);
        left.fork();
        return right.compute() + left.join();
    }
}
```

Parallel streams commonly use fork/join infrastructure. They are not
automatically faster. Avoid shared mutable state and blocking I/O in
parallel stream pipelines.

## 16. Concurrent collections

  Collection                Characteristics
  ------------------------- --------------------------------------------
  `ConcurrentHashMap`       Concurrent map; no null keys/values
  `CopyOnWriteArrayList`    Cheap reads; writes copy backing array
  `BlockingQueue`           Thread-safe queue with blocking operations
  `ArrayBlockingQueue`      Bounded array-backed queue
  `LinkedBlockingQueue`     Optionally bounded linked queue
  `PriorityBlockingQueue`   Priority queue; logically unbounded
  `ConcurrentLinkedQueue`   Non-blocking concurrent queue
  `ConcurrentSkipListMap`   Concurrent sorted map

Use atomic map operations:

``` java
ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();
counts.merge("orders", 1, Integer::sum);
```

Other useful operations: `putIfAbsent`, `compute`, `computeIfAbsent`,
`computeIfPresent`. Separate map calls do not automatically form one
atomic business transaction.

### BlockingQueue

``` java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(100);

// Producer
queue.put("ORDER-1");

// Consumer
String orderId = queue.take();
```

`put` waits if full; `take` waits if empty. A bounded queue provides
backpressure.

## 17. ThreadLocal

A ThreadLocal stores a separate value for each thread.

``` java
private static final ThreadLocal<String> REQUEST_ID = new ThreadLocal<>();

void handle(String requestId) {
    try {
        REQUEST_ID.set(requestId);
        process();
    } finally {
        REQUEST_ID.remove();
    }
}
```

Thread pools reuse threads, so values can remain attached to pooled
threads. Always remove values when finished. ThreadLocal values do not
automatically propagate to arbitrary async tasks or virtual threads.

## 18. Producer-consumer pattern

``` text
Producer(s) -> Bounded BlockingQueue -> Consumer workers
```

It decouples production and processing rates, buffers work and supports
backpressure. Define capacity, shutdown behavior, retry/failure handling
and overload policy.

## 19. Java 8--21 concurrency evolution

-   **Java 8:** CompletableFuture, lambdas, functional APIs and parallel
    streams.
-   **Java 9:** Flow API and SubmissionPublisher.
-   **Java 11:** HTTP Client with synchronous/asynchronous calls;
    CompletableFuture timeout and delayed-executor enhancements.
-   **Java 17:** Modern JVM improvements; records and sealed classes can
    model task/result data.
-   **Java 21:** Virtual threads finalized; structured concurrency
    remained preview in Java 21.

### Virtual threads (Java 21)

Virtual threads are lightweight threads scheduled by the JVM over
carrier platform threads. They are useful for high-concurrency, mostly
blocking workloads.

``` java
try (ExecutorService executor =
         Executors.newVirtualThreadPerTaskExecutor()) {
    Future<String> future =
        executor.submit(() -> callRemoteService());
    System.out.println(future.get());
}
```

  -----------------------------------------------------------------------
  Platform thread                     Virtual thread
  ----------------------------------- -----------------------------------
  Typically maps to OS thread         JVM schedules over carrier threads

  More expensive at very high counts  Lightweight for many blocking tasks

  Useful for CPU work and bounded     Useful for thread-per-task blocking
  pools                               workloads
  -----------------------------------------------------------------------

Virtual thread guidance: - Do not pool virtual threads merely to limit
their number. - Use semaphores or resource pools to limit scarce
resources such as DB connections. - They do not speed up CPU-bound
work. - They do not remove downstream rate limits. - Check the target
JDK's behavior and diagnostics for monitor-related pinning. - Structured
concurrency was preview in Java 21; verify the target JDK before using
it.

## 20. Thread-pool sizing and performance

For CPU-heavy work, a pool near available processors is a starting
point, not a universal rule:

``` java
int processors = Runtime.getRuntime().availableProcessors();
```

I/O-bound work may need more concurrent tasks, but pool size must
respect database connections, downstream capacity and memory.

Monitor: - Active and total thread count. - Queue depth. - Task wait and
execution time. - Rejected task count. - CPU and memory. - Downstream
connection pool usage. - Request latency and throughput.

Avoid unbounded queues: overload can become memory exhaustion and high
latency.

## 21. Spring Boot concurrency

### @Async

``` java
@Configuration
@EnableAsync
class AsyncConfig {
    @Bean
    Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("app-async-");
        executor.initialize();
        return executor;
    }
}
```

``` java
@Service
class NotificationService {
    @Async
    public CompletableFuture<Void> sendAsync(String message) {
        send(message);
        return CompletableFuture.completedFuture(null);
    }
}
```

Important: - Proxy-based `@Async` generally does not apply to
self-invocation within the same bean. - Configure executor capacity and
rejection behavior. - Handle async exceptions. - Caller transaction and
ThreadLocal context do not automatically propagate.

### @Scheduled

Scheduled tasks may overlap depending on configuration and duration. If
multiple application replicas run a task that must execute once
globally, design distributed coordination.

### Transactions and threads

Spring transactions are commonly thread-bound. A new thread does not
automatically join the caller's transaction. Define transaction
boundaries explicitly.

### Singleton beans

Spring services are commonly singleton-scoped. Avoid mutable
request-specific instance fields in singleton beans unless correctly
coordinated.

## 22. Thread dump and production troubleshooting

Capture a thread dump:

``` bash
jstack <pid> > thread-dump.txt
```

Or:

``` bash
jcmd <pid> Thread.print
```

Other diagnostics:

``` bash
jcmd <pid> VM.version
jcmd <pid> JFR.start
jstat -gcutil <pid> 1000
```

Look for: - BLOCKED threads. - Long-running WAITING states. - Repeated
stack traces. - Deadlock reports. - Many workers blocked on downstream
calls. - Saturated pools and growing queues.

Troubleshooting sequence: 1. Identify affected request/time window. 2.
Check CPU, memory and application metrics. 3. Capture multiple thread
dumps several seconds apart. 4. Inspect lock contention and blocked I/O.
5. Check executor queues and rejection counts. 6. Check DB/downstream
connection pools. 7. Reproduce under controlled load. 8. Fix and compare
measurements.

Use Java Flight Recorder and Java Mission Control where available.

## 23. Common mistakes

  -----------------------------------------------------------------------
  Mistake                             Better approach
  ----------------------------------- -----------------------------------
  Calling `run()` expecting a new     Call `start()` or submit task
  thread                              

  Creating unlimited threads          Use bounded executors or virtual
                                      threads with resource limits

  Assuming volatile makes `count++`   Use atomics or locks
  safe                                

  Forgetting unlock                   Unlock in `finally`

  Ignoring interrupts                 Propagate or restore interrupt
                                      status

  Using sleep for coordination        Use synchronization primitives

  Unbounded task queues               Bound queues and define overload
                                      behavior

  Sharing mutable state unnecessarily Prefer immutability/confinement

  Holding locks during remote calls   Keep critical sections short

  Forgetting executor shutdown        Manage lifecycle

  Assuming transactions cross threads Define transaction boundaries

  Assuming virtual threads remove     Limit DB/downstream resources
  bottlenecks                         
  -----------------------------------------------------------------------

## 24. Senior-level interview questions

### Fundamentals

1.  Process vs thread?
2.  Concurrency vs parallelism?
3.  `start()` vs `run()`?
4.  Explain Java thread states.
5.  Runnable vs Callable?
6.  What does interrupt do?
7.  Why is Thread.stop unsafe?

### Memory model and synchronization

8.  What is a race condition?
9.  How does synchronized work?
10. Instance vs static synchronized method?
11. What does volatile guarantee?
12. Why is volatile count++ unsafe?
13. Explain happens-before and safe publication.
14. What is CAS?
15. AtomicInteger vs LongAdder?

### Locks and coordination

16. synchronized vs ReentrantLock?
17. Deadlock vs livelock vs starvation?
18. How do wait/notify/notifyAll work?
19. Why call wait in a loop?
20. ReadWriteLock vs StampedLock?

### Executors and async

21. execute vs submit?
22. shutdown vs shutdownNow?
23. How does ThreadPoolExecutor work?
24. Why are unbounded queues risky?
25. Explain rejection policies.
26. Future vs CompletableFuture?
27. thenApply vs thenCompose?
28. allOf vs anyOf?
29. What is work stealing?

### Production and modern Java

30. HashMap vs ConcurrentHashMap?
31. What is BlockingQueue?
32. ThreadLocal and memory leak risk?
33. How do you size a pool?
34. How do you diagnose deadlocks?
35. How does Spring @Async work?
36. Do Spring transactions propagate to new threads?
37. Platform vs virtual threads?
38. Are virtual threads good for CPU-heavy work?
39. How do you implement backpressure?

## 25. Practical coding exercises

1.  Two threads print odd/even numbers in order.
2.  Thread-safe counter with synchronized.
3.  Same counter using AtomicInteger.
4.  Producer-consumer using BlockingQueue.
5.  Fixed-size executor with rejection handling.
6.  Callable/Future for independent calculations.
7.  Parallel service calls using CompletableFuture.
8.  Demonstrate deadlock, capture dump and fix lock order.
9.  Bounded task queue with backpressure.
10. Java 21 virtual-thread example with simulated blocking calls.
11. Concurrent frequency counter using ConcurrentHashMap.merge.
12. Test interruption and graceful shutdown.

Odd/even exercise requirements: - Odd worker prints 1, 3, 5... - Even
worker prints 2, 4, 6... - Stop at a maximum. - No busy waiting. -
Handle interruption correctly. - First solve with wait/notifyAll, then
with a higher-level utility.

## 26. 21-day learning plan

  ------------------------------------------------------------------------------------
  Days                    Topics                               Practice
  ----------------------- ------------------------------------ -----------------------
  1--2                    Thread basics/lifecycle              Compare start and run

  3--4                    Race conditions/synchronized         Safe and unsafe
                                                               counters

  5--6                    volatile/JMM                         Visibility and
                                                               happens-before

  7--8                    Atomics/CAS                          Atomic counter and
                                                               frequency map

  9--10                   Locks/wait/notify/deadlocks          Producer-consumer lab

  11--12                  ExecutorService/ThreadPoolExecutor   Bounded executor

  13--14                  Future/CompletableFuture             Parallel service calls

  15--16                  Concurrent collections/ThreadLocal   Concurrent cache and
                                                               cleanup

  17                      ForkJoinPool/parallel streams        Recursive computation

  18                      Spring @Async/transactions           Async notification
                                                               service

  19                      Virtual threads                      Blocking I/O simulation

  20                      Thread dumps/performance             Diagnose stuck executor

  21                      Revision/project                     Concurrent order
                                                               processor
  ------------------------------------------------------------------------------------

## 27. Practical project: Concurrent Order Processing

Build a small Spring Boot application: 1. Accept orders through REST. 2.
Validate each order. 3. Submit independent tasks to a configured
executor. 4. Use bounded queues and explicit rejection policy. 5.
Protect shared order state. 6. Call independent payment/notification
services concurrently only where business rules allow. 7. Implement
idempotency. 8. Handle timeouts and interruption. 9. Expose
active-thread, queue-depth and failure metrics. 10. Add unit and
integration tests. 11. Capture a thread dump under load. 12. Compare
platform-thread and Java 21 virtual-thread approaches for an I/O-bound
workload.

Concurrency must preserve business invariants; do not parallelize
payment and inventory actions without considering consistency and
compensation rules.

## 28. Official references

-   Java Concurrency Tutorial:
    https://docs.oracle.com/javase/tutorial/essential/concurrency/
-   `java.util.concurrent` API:
    https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html
-   Java Language Specification: https://docs.oracle.com/javase/specs/
-   OpenJDK: https://openjdk.org/
-   Java Flight Recorder:
    https://docs.oracle.com/en/java/javase/21/jfapi/

**Senior developer priority:** Master the Java Memory Model,
happens-before, synchronization, locks, executor design,
CompletableFuture, concurrent collections, pool saturation,
interruption, deadlock diagnosis and Java 21 virtual threads. The
production skill is controlling concurrency while preserving
correctness, capacity and graceful failure behavior.

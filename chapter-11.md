# Chapter 11: Concurrency

Threads allow multiple activities to proceed concurrently. Concurrent programming is harder than single-threaded programming, because more things can go wrong, and failures can be hard to reproduce. You can’t avoid concurrency. It is inherent in the platform and a requirement if you are to obtain good performance from multicore processors, which are now ubiquitous. This chapter contains advice to help you write clear, correct, well-documented concurrent programs.

## Item 78: Synchronize access to shared mutable data

The `synchronized` keyword ensures that only a single thread can execute a method or block at one time. Many programmers think of synchronization solely as a means of mutual exclusion, to prevent an object from being seen in an inconsistent state by one thread while it’s being modified by another. In this view, an object is created in a consistent state (Item 17) and locked by the methods that access it. These methods observe the state and optionally cause a state transition, transforming the object from one consistent state to another. Proper use of synchronization guarantees that no method will ever observe the object in an inconsistent state.

This view is correct, but it’s only half the story. Without synchronization, one thread’s changes might not be visible to other threads. Not only does synchronization prevent threads from observing an object in an inconsistent state, but it ensures that each thread entering a synchronized method or block sees the effects of all previous modifications that were guarded by the same lock.

- Reading or writing a variable is atomic unless the variable is of type `long` or `double`, i.e. reading a variable other than a `long` or `double` is guaranteed to return *some* value that was stored into that variable by some thread, even if multiple threads modify the variable concurrently and without synchronization. While the language specification guarantees that a thread will not see an arbitrary value when reading a field, it does not guarantee that a value written by one thread will be visible to another. __Synchronization is required for reliable communication between threads as well as for mutual exclusion__

- __Do not use `Thread.stop`__. It is inherently unsafe — its use can result in data corruption. A recommended way to stop one thread from another is to have the first thread poll a `boolean` field that is initially `false` but can be set to `true` by the second thread to indicate that the first thread is to stop itself.

```java
// Properly synchronized cooperative thread termination
public class StopThread {
	private static boolean stopRequested;

	private static synchronized void requestStop() {
		stopRequested = true;
	}

	private static synchronized boolean stopRequested() {
		return stopRequested;
	}

	public static void main(String[] args)
		throws InterruptedException {
		Thread backgroundThread = new Thread(() -> {
			int i = 0;
			while (!stopRequested())
				i++;
		});
		backgroundThread.start();
		
		TimeUnit.SECONDS.sleep(1);
		requestStop(); //terminates background thread after 1 sec
	}
}
```

The actions of the synchronized methods in `StopThread` would be atomic even without synchronization. In other words, the synchronization on these methods is used solely for its communication effects, not for mutual exclusion.

- __Synchronization is not guaranteed to work unless both read and
write operations are synchronized__

- Declaring variables `volatile`. While the volatile modifier performs no mutual exclusion, it guarantees that any thread that reads the field will see the most recently written value:

```java
// Cooperative thread termination with a volatile field
public class StopThread {
	private static volatile boolean stopRequested;
	
	public static void main(String[] args)
		throws InterruptedException {
		Thread backgroundThread = new Thread(() -> {
			int i = 0;
			while (!stopRequested)
				i++;
		});
		backgroundThread.start();
		
		TimeUnit.SECONDS.sleep(1);
		stopRequested = true;
	}
}
```

- __The increment operator (++) is not atomic__. It performs two operations on a field: first it reads the value, and then it writes back a new value, equal to the old value plus one. If a second thread reads the field between the time a thread reads the old value and writes back a new one, the second thread will see the same value as the first and return the same serial number. This is a safety failure: the program computes the wrong results. Wrap it in a `synchronized` method, ensuring that multiple invocations won’t be interleaved and that each invocation of the method will see the effects of all previous invocations.

- `java.util.concurrent.atomic` provides primitives for lock-free, thread-safe programming on single variables. While `volatile` provides only the communication effects of synchronization, this package also provides atomicity

- The best way to avoid the problems discussed in this item is not to share mutable data. Either share immutable data (Item 17) or don’t share at all. In other words, __confine mutable data to a single thread__. If you adopt this policy, it is important to document it so that the policy is maintained as your program evolves. It is also important to have a deep understanding of the frameworks and libraries you’re using because they may introduce threads that you are unaware of.

__When multiple threads share mutable data, each thread that reads or writes the data must perform synchronization__. Without synchronization, there is no guarantee that one thread’s changes will be visible to another. The penalties for failing to synchronize shared mutable data are liveness and safety failures. These failures are among the most difficult to debug. They can be intermittent and timing-dependent, and program behavior can vary radically from one VM to another. If you need only inter-thread communication, and not mutual exclusion, the `volatile` modifier is an acceptable form of synchronization, but it can be tricky to use correctly.

## Item 79: Avoid excessive synchronization

Item 78 warns of the dangers of insufficient synchronization. This item concerns the opposite problem. Depending on the situation, excessive synchronization can cause reduced performance, deadlock, or even nondeterministic behavior.

- __To avoid liveness and safety failures (i.e. deadlock and data corruption), never cede control to the client within a synchronized method or block__. In other words, inside a synchronized region, do not invoke a method that is designed to be overridden, or one provided by a client in the form of a function object (Item 24). From the perspective of the class with the synchronized region, such methods are alien. The class has no knowledge of what the method does and has no control over it. Depending on what an alien method does, calling it from a synchronized region can cause exceptions, deadlocks, or data corruption.
	- It is usually not too hard to fix this sort of problem by moving alien method invocations out of synchronized blocks
	- The libraries provide a concurrent collection (Item 81) known as `CopyOnWriteArrayList` that is tailor-made for this purpose. This `List` implementation is a variant of `ArrayList` in which all modification operations are implemented by making a fresh copy of the entire underlying array. Because the internal array is never modified, iteration requires no locking and is very fast. For most uses, the performance of `CopyOnWriteArrayList` would be atrocious, but it’s perfect for observer lists, which are rarely modified and often traversed.
- As a rule, __you should do as little work as possible inside synchronized regions__. Obtain the lock, examine the shared data, transform it as necessary, and drop the lock. If you must perform some time-consuming activity, find a way to move it out of the synchronized region without violating the guidelines in Item 78.

More generally, try to limit the amount of work that you do from within synchronized regions. When you are designing a mutable class, think about whether it should do its own synchronization. In the modern multicore era, it is more important than ever not to synchronize excessively. Synchronize your class internally only if there is a good reason to do so, and document your decision clearly ([Item 82](chapter-11.md#item-82-document-thread-safety)).

## Item 80: Prefer executors and tasks to threads

You should generally use Executor Framework, which is a flexible interface-based task execution facility contained in `java.util.concurrent`.

It allows you to create a work queue for asynchronous processing by a background thread:
```java
ExecutorService exec = Executors.newSingleThreadExecutor();
```

Here is how to submit a runnable for execution:
```java
exec.execute(runnable);
```

And here is how to tell the executor to terminate gracefully (if you fail to do this, it is likely that your VM will not exit):
```java
exec.shutdown();
```

You can do many more things with an executor service.
- You can wait for a particular task to complete (with the `get` method, as shown in Item 79, page 319) 
- You can wait for any or all of a collection of tasks to complete (using the `invokeAny` or `invokeAll` methods) 
- You can wait for the executor service to terminate (using the `awaitTermination` method) 
- You can retrieve the results of tasks one by one as they complete (using an `ExecutorCompletionService`) 
- You can schedule tasks to run at a particular time or to run periodically (using a `ScheduledThreadPoolExecutor`), and so on.
- If you want more than one thread to process requests from the queue, simply call a different static factory that creates a different kind of executor service called a *thread pool*. You can create a thread pool with a fixed or variable number of threads.
- The java.util.concurrent.Executors class contains static factories that provide most of the executors you’ll ever need. If, however, you want some thing out of the ordinary, you can use the `ThreadPoolExecutor` class directly. This class lets you configure nearly every aspect of a thread pool’s operation.

- For a small program, or a lightly loaded server, `Executors.newCachedThreadPool` is generally a good choice because it demands no configuration and generally “does the right thing.” But a cached thread pool is not a good choice for a heavily loaded production server! In a cached thread pool, submitted tasks are not queued but immediately handed off to a thread for execution. If no threads are available, a new one is created. If a server is so heavily loaded that all of its CPUs are fully utilized and more tasks arrive, more threads will be created, which will only make matters worse. Therefore, in a heavily loaded production server, you are much better off using `Executors.newFixedThreadPool`, which gives you a pool with a fixed number of threads, or using the `ThreadPoolExecutor` class directly, for maximum control.

- You should generally refrain from working directly with threads. When you work directly with threads, a Thread serves as both a unit of work and the mechanism for executing it. In the executor framework, the unit of work and the execution mechanism are separate. The key abstraction is the unit of work, which is the task. There are two kinds of tasks: `Runnable` and its close cousin, `Callable` (which is like `Runnable`, except that it returns a value and can throw arbitrary exceptions). The general mechanism for executing tasks is the executor service. If you think in terms of tasks and let an executor service execute them for you, you gain the flexibility to select an appropriate execution policy to meet your needs and to change the policy if your needs change. In essence, the Executor Framework does for execution what the Collections Framework did for aggregation.

- In Java 7, the Executor Framework was extended to support fork-join tasks, which are run by a special kind of executor service known as a fork-join pool. A fork-join task, represented by a `ForkJoinTask` instance, may be split up into smaller subtasks, and the threads comprising a `ForkJoinPool` not only process these tasks but “steal” tasks from one another to ensure that all threads remain busy, resulting in higher CPU utilization, higher throughput, and lower latency. Writing and tuning fork-join tasks is tricky. Parallel streams (Item 48) are written atop fork join pools and allow you to take advantage of their performance benefits with little effort, assuming they are appropriate for the task at hand.

- Reference *Java Concurrency in Practice* for a more complete treatment of the Executor Framework.

## Item 81: Prefer concurrency utilities to wait and notify

Since Java 5, the platform has provided higher-level concurrency utilities that do the sorts of things you formerly had to hand-code atop `wait` and `notify`. __Given the difficulty of using `wait` and `notify` correctly, you should use the higher-level concurrency utilities instead__.

The higher-level utilities in java.util.concurrent fall into three categories: 1) the Executor Framework, which was covered briefly in Item 80; 2) concurrent collections; and 3) synchronizers. Concurrent collections and synchronizers are covered briefly in this item.

- The concurrent collections are high-performance concurrent implementations of standard collection interfaces such as `List`, `Queue`, and `Map`. To provide high concurrency, these implementations manage their own synchronization internally (Item 79). Therefore, it is impossible to exclude concurrent activity from a concurrent collection; locking it will only slow the program.
 	- Because you can’t exclude concurrent activity on concurrent collections, you can’t atomically compose method invocations on them either. Therefore, concurrent collection interfaces were outfitted with state-dependent modify operations, which combine several primitives into a single atomic operation. These operations proved sufficiently useful on concurrent collections that they were added to the corresponding collection interfaces in Java 8, using default methods (Item 21). For example, `Map`’s `putIfAbsent(key, value)` method inserts a mapping for a key if none was present and returns the previous value associated with the key, or `null` if there was none. This makes it easy to implement thread-safe canonicalizing maps.
 	- Concurrent collections make synchronized collections largely obsolete. For example, __use ConcurrentHashMap in preference to Collections.synchronizedMap__. Simply replacing synchronized maps with concurrent maps can dramatically increase the performance of concurrent applications.
 	- Some of the collection interfaces were extended with blocking operations, which wait (or block) until they can be successfully performed. For example, `BlockingQueue` extends `Queue` and adds several methods, including take, which removes and returns the head element from the queue, waiting if the queue is empty. This allows blocking queues to be used for work queues (also known as producer-consumer queues), to which one or more producer threads enqueue work items and from which one or more consumer threads dequeue and process items as they become available. As you’d expect, most `ExecutorService` implementations, including `ThreadPoolExecutor`, use a `BlockingQueue` (Item 80).

- Synchronizers are objects that enable threads to wait for one another, allowing them to coordinate their activities. The most commonly used synchronizers are `CountDownLatch` and `Semaphore`. Less commonly used are `CyclicBarrier` and `Exchanger`. The most powerful synchronizer is `Phaser`.
	- Countdown latches are single-use barriers that allow one or more threads to wait for one or more other threads to do something. The sole constructor for `CountDownLatch` takes an `int` that is the number of times the `countDown` method must be invoked on the latch before all waiting threads are allowed to proceed.
		- can be used to time things (see book for example)
		- __For interval timing, always use System.nanoTime rather than System.currentTimeMillis__. `System.nanoTime` is both more accurate and more precise and is unaffected by adjustments to the system’s real-time clock. Finally, note that the code in this example won’t yield accurate timings unless action does a fair amount of work, say a second or more. Accurate microbenchmarking is notoriously hard and is best done with the aid of a specialized framework such as jmh.

- The wait method is used to make a thread wait for some condition. It must be invoked inside a synchronized region that locks the object on which it is invoked. Here is the standard idiom for using the wait method:
```java
// The standard idiom for using the wait method
synchronized (obj) {
	while (<condition does not hold>)
		obj.wait(); // (Releases lock, and reacquires on wakeup)
	... // Perform action appropriate to condition
}
```

__Always use the wait loop idiom to invoke the wait method; never invoke it outside of a loop__. The loop serves to test the condition before and after waiting.

- A related issue is whether to use `notify` or `notifyAll` to wake waiting threads. (Recall that `notify` wakes a single waiting thread, assuming such a thread exists, and `notifyAll` wakes all waiting threads.) It is sometimes said that you should always use `notifyAll`. This is reasonable, conservative advice. It will always yield correct results because it guarantees that you’ll wake the threads that need to be awakened. You may wake some other threads, too, but this won’t affect the correctness of your program. These threads will check the condition for which they’re waiting and, finding it false, will continue waiting.

- Just as placing the `wait` invocation in a loop protects against accidental or malicious notifications on a publicly accessible object, using `notifyAll` in place of `notify` protects against accidental or malicious waits by an unrelated thread. Such waits could otherwise “swallow” a critical notification, leaving its intended recipient waiting indefinitely.

Using `wait` and `notify` directly is like programming in “concurrency assembly language,” as compared to the higher-level language provided by `java.util.concurrent`. __There is seldom, if ever, a reason to use `wait` and `notify` in new code__. If you maintain code that uses `wait` and `notify`, make sure that it always invokes `wait` from within a `while` loop using the standard idiom. The `notifyAll` method should generally be used in preference to `notify`. If `notify` is used, great care must be taken to ensure liveness.

## Item 82: Document thread safety

- In normal operation, Javadoc does not include the `synchronized` modifier in its output, and with good reason. The presence of the `synchronized` modifier in a method declaration is an implementation detail, not a part of its API. It does not reliably indicate that a method is thread-safe.

- To enable safe concurrent use, a class must clearly document what level of thread safety it supports. The following list summarizes levels of thread safety. It is not exhaustive but covers the common cases:
	- Immutable — Instances of this class appear constant. No external synchronization is necessary. Examples include String, Long, and BigInteger (Item 17).
	- Unconditionally thread-safe — Instances of this class are mutable, but the class has sufficient internal synchronization that its instances can be used concurrently without the need for any external synchronization. Examples include AtomicLong and ConcurrentHashMap.
	- Conditionally thread-safe — Like unconditionally thread-safe, except that some methods require external synchronization for safe concurrent use. Examples include the collections returned by the Collections.synchronized wrappers, whose iterators require external synchronization.
		- You must indicate which invocation sequences require external synchronization, and which lock (or in rare cases, locks) must be acquired to execute these sequences. Typically it is the lock on the instance itself, but there are exceptions. For example, the documentation for `Collections.synchronizedMap` says this: 
		It is imperative that the user manually synchronize on the returned map when iterating over any of its collection views:
```java
Map<K, V> m = Collections.synchronizedMap(new HashMap<>());
Set<K> s = m.keySet(); // Needn't be in synchronized block
	...
synchronized(m) { // Synchronizing on m, not s!
	for (K key : s)
		key.f();
}
```
		Failure to follow this advice may result in non-deterministic behavior.
	- Not thread-safe — Instances of this class are mutable. To use them concurrently, clients must surround each method invocation (or invocation sequence) with external synchronization of the clients’ choosing. Examples include the general-purpose collection implementations, such as ArrayList and HashMap.
	- Thread-hostile — This class is unsafe for concurrent use even if every method invocation is surrounded by external synchronization. Thread hostility usually results from modifying static data without synchronization. No one writes a thread-hostile class on purpose; such classes typically result from the failure to consider concurrency. When a class or method is found to be thread-hostile, it is typically fixed or deprecated. The generateSerialNumber method in Item 78 would be thread-hostile in the absence of internal synchronization, as discussed on page 322.

- A client can mount a denial-of-service (DNS) attack by holding the publicly accessible lock for a prolonged period. To prevent this denial-of-service attack, you can use a private lock object instead of using synchronized methods (which imply a publicly accessible lock):

```java
// Private lock object idiom - thwarts denial-of-service attack
private final Object lock = new Object();
	public void foo() {
		synchronized(lock) {
			...
	}
}
```

Because the private lock object is inaccessible outside the class, it is impossible for clients to interfere with the object’s synchronization. In effect, we are applying the advice of Item 15 by encapsulating the lock object in the object it synchronizes.

Note that the lock field is declared final. This prevents you from inadvertently changing its contents, which could result in catastrophic unsynchronized access (Item 78). We are applying the advice of Item 17, by minimizing the mutability of the lock field. __Lock fields should always be declared final__. This is true whether you use an ordinary monitor lock (as shown above) or a lock from the `java.util.concurrent.locks` package.

The private lock object idiom can be used only on unconditionally thread-safe classes. Conditionally thread-safe classes can’t use this idiom because they must document which lock their clients are to acquire when performing certain method invocation sequences.

Every class should clearly document its thread safety properties with a carefully worded prose description or a thread safety annotation. The `synchronized` modifier plays no part in this documentation. Conditionally thread-safe classes must document which method invocation sequences require external synchronization, and which lock to acquire when executing these sequences. If you write an unconditionally thread-safe class, consider using a private lock object in place of synchronized methods. This protects you against synchronization interference by clients and subclasses and gives you the flexibility to adopt a more sophisticated approach to concurrency control in a later release.

## Item 83: Use lazy initialization judiciously

Lazy initialization is the act of delaying the initialization of a field until its value is needed. If the value is never needed, the field is never initialized. This technique is applicable to both static and instance fields. While lazy initialization is primarily an optimization, it can also be used to break harmful circularities in class and instance initialization.

- As is the case for most optimizations, the best advice for lazy initialization is “don’t do it unless you need to” (Item 67). Lazy initialization is a double-edged sword. It decreases the cost of initializing a class or creating an instance, at the expense of increasing the cost of accessing the lazily initialized field. Depending on what fraction of these fields eventually require initialization, how expensive it is to initialize them, and how often each one is accessed once initialized, lazy initialization can (like many “optimizations”) actually harm performance

- If a field is accessed only on a fraction of the instances of a class and it is costly to initialize the field, then lazy initialization may be worthwhile. The only way to know for sure is to measure the performance of the class with and without lazy initialization.

- __If you use lazy initialization to break an initialization circularity, use a synchronized accessor__ because it is the simplest, clearest alternative:
```java
// Lazy initialization of instance field - synchronized accessor
private FieldType field;
private synchronized FieldType getField() {
	if (field == null)
		field = computeFieldValue();
	return field;
}
```

- __If you need to use lazy initialization for performance on a static field, use the lazy initialization holder class idiom__. This idiom exploits the guarantee that a class will not be initialized until it is used. Here’s how it looks:
```java
// Lazy initialization holder class idiom for static fields
private static class FieldHolder {
	static final FieldType field = computeFieldValue();
}

private static FieldType getField() { return FieldHolder.field; }
```
When `getField` is invoked for the first time, it reads `FieldHolder.field` for the first time, causing the initialization of the `FieldHolder` class. The beauty of this idiom is that the `getField` method is not synchronized and performs only a field access, so lazy initialization adds practically nothing to the cost of access

- __If you need to use lazy initialization for performance on an instance field, use the double-check idiom__. This idiom avoids the cost of locking when accessing the field after initialization (Item 79). The idea behind the idiom is to check the value of the field twice (hence the name double-check): once without locking and then, if the field appears to be uninitialized, a second time with locking. Only if the second check indicates that the field is uninitialized does the call initialize the field. Because there is no locking once the field is initialized, it is critical that the field be declared `volatile` (Item 78). Here is the idiom:
```java
// Double-check idiom for lazy initialization of instance fields
private volatile FieldType field;
private FieldType getField() {
	FieldType result = field;
	if (result == null) { // First check (no locking)
		synchronized(this) {
			if (field == null) // Second check (with locking)
				field = result = computeFieldValue();
		}
	}
	return result;
}
```
This code may appear a bit convoluted. In particular, the need for the local variable (`result`) may be unclear. What this variable does is to ensure that field is read only once in the common case where it’s already initialized. While not strictly necessary, this may improve performance and is more elegant by the standards applied to low-level concurrent programming. On my machine, the method above is about 1.4 times as fast as the obvious version without a local variable

You should initialize most fields normally, not lazily. If you must initialize a field lazily in order to achieve your performance goals, or to break a harmful initialization circularity, then use the appropriate lazy initialization technique. For instance fields, it is the double-check idiom; for static fields, the lazy initialization holder class idiom. For instance fields that can tolerate repeated initialization, you may also consider the single-check idiom.

## Item 84: Don’t depend on the thread scheduler

- The best way to write a robust, responsive, portable program is to ensure that the average number of runnable threads is not significantly greater than the number of processors. This leaves the thread scheduler with little choice: it simply runs the runnable threads till they’re no longer runnable. The program’s behavior doesn’t vary too much, even under radically different thread-scheduling policies. Note that the number of runnable threads isn’t the same as the total number of threads, which can be much higher. Threads that are waiting are not runnable.

- The main technique for keeping the number of runnable threads low is to have each thread do some useful work, and then wait for more. Threads should not run if they aren’t doing useful work. In terms of the Executor Framework (Item 80), this means sizing thread pools appropriately [Goetz06, 8.2] and keeping tasks short, but not too short, or dispatching overhead will harm performance.

- Threads should not busy-wait, repeatedly checking a shared object waiting for its state to change. Besides making the program vulnerable to the vagaries of the thread scheduler, busy-waiting greatly increases the load on the processor, reducing the amount of useful work that others can accomplish.

- When faced with a program that barely works because some threads aren’t getting enough CPU time relative to others, resist the temptation to “fix” the program by putting in calls to Thread.yield. You may succeed in getting the program to work after a fashion, but it will not be portable. The same yield invocations that improve performance on one JVM implementation might make it worse on a second and have no effect on a third. Thread.yield has no testable semantics. A better course of action is to restructure the application to reduce the number of concurrently runnable threads.

- Thread priorities are among the least portable features of Java. It is not unreasonable to tune the responsiveness of an application by tweaking a few thread priorities, but it is rarely necessary and is not portable. It is unreasonable to attempt to solve a serious liveness problem by adjusting thread priorities. The problem is likely to return until you find and fix the underlying cause.

Do not depend on the thread scheduler for the correctness of your program. The resulting program will be neither robust nor portable. As a corollary, do not rely on `Thread.yield` or thread priorities. These facilities are merely hints to the scheduler. Thread priorities may be used sparingly to improve the quality of service of an already working program, but they should never be used to “fix” a program that barely works.
# Chapter 10: Exceptions

When used to best advantage, exceptions can improve a program’s readability, reliability, and maintainability. When used improperly, they can have the opposite effect. This chapter provides guidelines for using exceptions effectively.

## Item 69: Use exceptions only for exceptional conditions

Exceptions are designed for use in exceptional conditions. Don’t use them for ordinary control flow/"performance gains", and don’t write APIs that force others to do so.

- Provide state-testing methods to avoid having to rely on exceptions or return an empty optional (Item 55) or null

## Item 70: Use checked exceptions for recoverable conditions and runtime exceptions for programming errors

- Java provides three kinds of throwables: checked exceptions, runtime exceptions, and errors.

- __Use checked exceptions for conditions from which the caller can reasonably be expected to recover__. By throwing a checked exception, you force the caller to handle the exception in a catch clause or to propagate it outward. Each checked exception that a method is declared to throw is therefore a potent indication to the API user that the associated condition is a possible outcome of invoking the method. By confronting the user with a checked exception, the API designer presents a mandate to recover from the condition. The user can disregard the mandate by catching the exception and ignoring it, but this is usually a bad idea (Item 77).

- 2 kinds of unchecked throwables: runtime exceptions and errors. They are identical in their behavior: both are throwables that needn’t, and generally shouldn’t, be caught. If a program throws an unchecked exception or an error, it is generally the case that recovery is impossible and continued execution would do more harm than good. If a program does not catch such a throwable, it will cause the current thread to halt with an appropriate error message.
    - __Use runtime exceptions to indicate programming errors__. The great majority of runtime exceptions indicate precondition violations. A precondition violation is simply a failure by the client of an API to adhere to the contract established by the API specification. e.g. the contract for array access specifies that the array index must be between zero and the array length minus one, inclusive. `ArrayIndexOutOfBoundsException` indicates that this precondition was violated.
    - There is a strong convention that errors are reserved for use by the JVM to indicate resource deficiencies, invariant failures, or other conditions that make it impossible to continue execution. Given the almost universal acceptance of this convention, it’s best not to implement any new `Error` subclasses. Therefore, __all of the unchecked throwables you implement should subclass `RuntimeException` (directly or indirectly). Not only shouldn’t you define `Error` subclasses, but with the exception of `AssertionError`, you shouldn’t throw them either__.

- __Never define a throwable that is not a subclass of `Exception`, `RuntimeException`, or `Error`__. It is possible to define a throwable that is not a subclass of any of them. They behave as ordinary checked exceptions (which are subclasses of `Exception` but not `RuntimeException`), but they have no benefits over ordinary checked exceptions and would serve merely to confuse the user of your API.

- Exceptions are full-fledged objects on which arbitrary methods can be defined. The primary use of such methods is to provide code that catches the exception with additional information concerning the condition that caused the exception to be thrown.
    - __Don't parse the string representation of exception for information about it__ as this is likely to lead to nonportable and fragile code. 
    - Because checked exceptions generally indicate recoverable conditions, it’s especially important for them to provide methods that furnish information to help the caller recover from the exceptional condition. e.g. suppose a checked exception is thrown when an attempt to make a purchase with a gift card fails due to insufficient funds. The exception should provide an accessor method to query the amount of the shortfall. This will enable the caller to relay the amount to the shopper.

Throw checked exceptions for recoverable conditions and unchecked exceptions for programming errors. When in doubt, throw unchecked exceptions. Don’t define any throwables that are neither checked exceptions nor runtime exceptions. Provide methods on your checked exceptions to aid in recovery.

## Item 71: Avoid unnecessary use of checked exceptions

Many Java programmers dislike checked exceptions, but used properly, they can improve APIs and programs. Unlike return codes and unchecked exceptions, they force programmers to deal with problems, enhancing reliability. That said, overuse of checked exceptions in APIs can make them far less pleasant to use. If a method throws checked exceptions, the code that invokes it must handle them in one or more `catch` blocks, or declare that it throws them and let them propagate outward. Either way, it places a burden on the user of the API. The burden increased in Java 8, as __methods throwing checked exceptions can’t be used directly in streams__ (Items 45–48).

This burden may be justified if the exceptional condition cannot be prevented by proper use of the API and the programmer using the API can take some useful action once confronted with the exception. Unless both of these conditions are met, an unchecked exception is appropriate. As a litmus test, ask yourself how the programmer will handle the exception. Is this the best that can be done?

```java
} catch(TheCheckedException e) {
    throw new AssertionError(); // Can't happen!
}
```

How about this?

```java
} catch(TheCheckedException e) {
    e.printStackTrace();        // Oh well, we lose.
    System.exit(1);
}
```

- The easiest way to eliminate a checked exception is to return an optional of the desired result type (Item 55). Instead of throwing a checked exception, the method simply returns an empty optional. The disadvantage of this technique is that the method can’t return any additional information detailing its inability to perform the desired computation. Exceptions, by contrast, have descriptive types, and can export methods to provide additional information (Item 70).

- You can turn a checked exception into an unchecked exception by breaking the method that throws the exception into two methods, the first of which returns a `boolean` that indicates whether the exception would be thrown. This API refactoring transforms the calling sequence from this:

```java
// Invocation with checked exception
try {
    obj.action(args);
} catch(TheCheckedException e) {
    // Handle exceptional condition
    ...
}
```

to this:

```java
// Invocation with state-testing method and unchecked exception
if (obj.actionPermitted(args)) {
    obj.action(args);
} else {
    // Handle exceptional condition
    ...
}
```

This refactoring is not always appropriate, but where it is appropriate, it can make an API more pleasant to use. While the latter calling sequence is no prettier than the former, the resulting API is more flexible. In cases where the programmer knows the call will succeed or is content to let the thread terminate if the call fails, the refactoring also allows this simple calling sequence:

```java
obj.action(args);
```

If you suspect that the simple calling sequence will be the norm, then this API refactoring may be appropriate.

When used sparingly, checked exceptions can increase the reliability of programs; when overused, they make APIs painful to use. If callers won’t be able to recover from failures, throw unchecked exceptions. If recovery may be possible and you want to force callers to handle exceptional conditions, first consider returning an optional. Only if this would provide insufficient information in the case of failure should you throw a checked exception.

## Item 72: Favor the use of standard exceptions

An attribute that distinguishes expert programmers from less experienced ones is that experts strive for and usually achieve a high degree of code reuse. Exceptions are no exception to the rule that code reuse is a good thing. The Java libraries provide a set of exceptions that covers most of the exception-throwing needs of most APIs.

This list summarizes the most commonly reused exceptions:

- `IllegalArgumentException`: the exception to throw when the caller passes in an argument whose value is inappropriate.
- `IllegalStateException`: the exception to throw if the invocation is illegal because of the state of the receiving object. For example, this would be the exception to throw if the caller attempted to use some object before it has been properly initialized.
- `NullPointerException`: Parameter value is null where prohibited.
- `IndexOutOfBoundsException`: Index parameter value is out of range.
- `ConcurrentModificationException`: Concurrent modification of an object has been detected where it is prohibited.
- `UnsupportedOperationException`: Object does not support method. This exception is used by classes that fail to implement one or more optional operations defined by an interface they implement. e.g. an append-only `List` implementation would throw this exception if someone tried to delete an element from the list.

Arguably, all erroneous method invocations boil down to an illegal argument or illegal state, but other exceptions are standardly used for certain kinds of illegal arguments and states. If a caller passes `null` in some parameter for which `null` values are prohibited, convention dictates that `NullPointerException` be thrown rather than `IllegalArgumentException`. Similarly, if a caller passes an out-of-range value in a parameter representing an index into a sequence, `IndexOutOfBoundsException` should be thrown rather than `IllegalArgumentException`.

- __Do not reuse `Exception`, `RuntimeException`, `Throwable`, or `Error` directly__. Treat these classes as if they were abstract. You can't reliably test for these exceptions because they are superclasses of other exceptions that a method may throw.

- Subclass a standard exception if you want to add more detail (Item 75), but remember that exceptions are serializable (Chapter 12). That alone is reason not to write your own exception class without good reason.

## Item 73: Throw exceptions appropriate to the abstraction

It is disconcerting when a method throws an exception that has no apparent connection to the task that it performs. This often happens when a method propagates an exception thrown by a lower-level abstraction. Not only is it disconcerting, but it pollutes the API of the higher layer with implementation details. If the implementation of the higher layer changes in a later release, the exceptions it throws will change too, potentially breaking existing client programs. 

To avoid this problem, __higher layers should catch lower-level exceptions and, in their place, throw exceptions that can be explained in terms of the higher-level abstraction__. This idiom is known as exception translation:

```java
// Exception Translation
try {
    ... // Use lower-level abstraction to do our bidding
} catch (LowerLevelException e) {
    throw new HigherLevelException(...);
}
```

- Exception chaining: passing the lower level exception to the higher level exception, which provides an accessor method (`Throwable`'s `getCause` method) to retrieve the lower-level exception. This is helpful to someone debugging the problem that caused the higher-level abstraction.

```java
// Exception Chaining
try {
    ... // Use lower-level abstraction to do our bidding
} catch (LowerLevelException cause) {
    throw new HigherLevelException(cause);
}
```

The higher-level exception’s constructor passes the cause to a chaining-aware superclass constructor, so it is ultimately passed to one of `Throwable`’s chaining-aware constructors, such as `Throwable(Throwable)`:

```java
// Exception with chaining-aware constructor
class HigherLevelException extends Exception {
    HigherLevelException(Throwable cause) {
        super(cause);
    }
}
```

Most standard exceptions have chaining-aware constructors. For exceptions that don’t, you can set the cause using `Throwable`’s `initCause` method. Not only does exception chaining let you access the cause programmatically (with `getCause`), but it integrates the cause’s stack trace into that of the higher-level exception.

- While exception translation is superior to mindless propagation of exceptions from lower layers, it should not be overused. Where possible, the best way to deal with exceptions from lower layers is to avoid them, by ensuring that lower-level methods succeed. Sometimes you can do this by checking the validity of the higher-level method’s parameters before passing them on to lower layers.

If it is impossible to prevent exceptions from lower layers, the next best thing is to have the higher layer silently work around these exceptions, insulating the caller of the higher-level method from lower-level problems, e.g. by logging the exception using some appropriate logging facility such as `java.util.logging`. This allows programmers to investigate the problem, while insulating client code and the users from it.

If it isn’t feasible to prevent or to handle exceptions from lower layers, use exception translation, unless the lower-level method happens to guarantee that all of its exceptions are appropriate to the higher level. Chaining provides the best of both worlds: it allows you to throw an appropriate higher-level exception, while capturing the underlying cause for failure analysis (Item 63).

## Item 74: Document all exceptions thrown by each method

- Always declare checked exceptions individually, and document precisely the conditions under which each one is thrown using the Javadoc `@throws` tag. Don’t take the shortcut of declaring that a method throws some superclass of multiple exception classes that it can throw. As an extreme example, don’t declare that a public method throws Exception or, worse, throws Throwable. In addition to denying any guidance to the method’s user concerning the exceptions it is capable of throwing, such a declaration greatly hinders the use of the method because it effectively obscures any other exception that may be thrown in the same context. One exception to this advice is the main method, which can safely be declared to throw Exception because it is called only by VM.
    - While the language does not require programmers to declare the unchecked exceptions that a method is capable of throwing, it is wise to document them as carefully as the checked exceptions. Unchecked exceptions generally represent programming errors (Item 70), and familiarizing programmers with all of the errors they can make helps them avoid making these errors. A well-documented list of the unchecked exceptions that a method can throw effectively describes the preconditions for its successful execution. It is essential that every public method’s documentation describe its preconditions (Item 56), and documenting its unchecked exceptions is the best way to satisfy this requirement.
    - It is particularly important that methods in interfaces document the unchecked exceptions they may throw. This documentation forms a part of the interface’s general contract and enables common behavior among multiple implementations of the interface.

- Use the Javadoc `@throws` tag to document each exception that a method can throw, but do not use the throws keyword on unchecked exceptions. The documentation generated by the Javadoc `@throws` tag without a corresponding throws clause in the method declaration provides a strong visual cue to the programmer that an exception is unchecked.

- __If an exception is thrown by many methods in a class for the same reason, you can document the exception in the class’s documentation comment__ rather than documenting it individually for each method. A common example is `NullPointerException`. It is fine for a class’s documentation comment to say, “All methods in this class throw a `NullPointerException` if a null object reference is passed in any parameter,” or words to that effect.

Document every exception that can be thrown by each method that you write. This is true for unchecked as well as checked exceptions, and for abstract as well as concrete methods. Provide individual `throws` clauses for each checked exception and do not provide `throws` clauses for unchecked exceptions. If you fail to document the exceptions that your methods can throw, it will be difficult or impossible for others to make effective use of your classes and interfaces.

## Item 75: Include failure-capture information in detail messages

It is critically important that the exception’s toString method return as much information as possible concerning the cause of the failure. In other words, the detail message of an exception should capture the failure for subsequent analysis.

- To capture a failure, the detail message of an exception should contain the values of all parameters and fields that contributed to the exception. For example, the detail message of an `IndexOutOfBoundsException` should contain the lower bound, the upper bound, and the index value that failed to lie between the bounds. This information tells a lot about the failure.
    - One caveat concerns security-sensitive information. Because stack traces may be seen by many people in the process of diagnosing and fixing software issues, do not include passwords, encryption keys, and the like in detail messages.

- Lengthy prose descriptions of the failure are superfluous; the information can be gleaned by reading the documentation and source code.

- The detail message is primarily for the benefit of programmers or site reliability engineers (as opposed to end users), when analyzing a failure. Therefore, information content is far more important than readability.

- One way to ensure that exceptions contain adequate failure-capture
information in their detail messages is to require this information in their constructors instead of a string detail message. The detail message can then be generated automatically to include the information. For example, instead of a String constructor, IndexOutOfBoundsException should have had a constructor that looks like this:

```java
/**
* Constructs an IndexOutOfBoundsException.
*
* @param lowerBound the lowest legal index value
* @param upperBound the highest legal index value plus one
* @param index the actual index value
*/
public IndexOutOfBoundsException(int lowerBound, int upperBound,
    int index) {
    // Generate a detail message that captures the failure
    super(String.format(
        "Lower bound: %d, Upper bound: %d, Index: %d",
        lowerBound, upperBound, index));

    // Save failure information for programmatic access
    this.lowerBound = lowerBound;
    this.upperBound = upperBound;
    this.index = index;
}
```

- As suggested in Item 70, it may be appropriate for an exception to provide accessor methods for its failure-capture information (`lowerBound`, `upperBound`, and `index` in the above example). It is more important to provide such accessor methods on checked exceptions than unchecked, because the failure-capture information could be useful in recovering from the failure.

## Item 76: Strive for failure atomicity

After an object throws an exception, it is generally desirable that the object still be in a well-defined, usable state, even if the failure occurred in the midst of performing an operation. This is especially true for checked exceptions, from which the caller is expected to recover. __Generally speaking, a failed method invocation should leave the object in the state that it was in prior to the invocation__. A method with this property is said to be failure atomic.

- The simplest way to guarantee failure atomicity is to design immutable objects (Item 17). If an object is immutable, failure atomicity is free. If an operation fails, it may prevent a new object from getting created, but it will never leave an existing object in an inconsistent state, because the state of each object is consistent when it is created and can’t be modified thereafter.

- For methods that operate on mutable objects, the most common way to
achieve failure atomicity is to check parameters for validity before performing the operation (Item 49). This causes most exceptions to get thrown before object modification commences. For example, consider the `Stack.pop` method in Item 7:

```java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;
}
```

If the initial size check were eliminated, the method would still throw an exception when it attempted to pop an element from an empty stack. It would, however, leave the size field in an inconsistent (negative) state, causing any future method invocations on the object to fail. Additionally, the `ArrayIndexOutOfBoundsException` thrown by the pop method would be inappropriate to the abstraction (Item 73).

- Basically, order the computation so that any part that may fail takes place before any part that modifies the object.

- A third approach to achieving failure atomicity is to perform the operation on a temporary copy of the object and to replace the contents of the object with the temporary copy once the operation is complete.

- A last and far less common approach to achieving failure atomicity is to write recovery code that intercepts a failure that occurs in the midst of an operation, and causes the object to roll back its state to the point before the operation began. This approach is used mainly for durable (disk-based) data structures.

- Failure atomicity is not always achieveable

- Errors are unrecoverable, so you need not even attempt to preserve failure atomicity when throwing `AssertionError`.

## Item 77: Don’t ignore exceptions

It is easy to ignore exceptions by surrounding a method invocation with a `try` statement with an empty `catch` block:

```java
// Empty catch block ignores exception - Highly suspect!
try {
    ...
} catch (SomeException e) {
}
```

__An empty catch block defeats the purpose of exceptions__, which is to force you to handle exceptional conditions. If you choose to ignore an exception, the catch block should contain a comment explaining why it is appropriate to do so, and the exception variable should be named `ignored`:

```java
Future<Integer> f = exec.submit(planarMap::chromaticNumber);
int numColors = 4; // Default; guaranteed sufficient for any map
try {
    numColors = f.get(1L, TimeUnit.SECONDS);
} catch (TimeoutException | ExecutionException ignored) {
    // Use default: minimal coloring is desirable, not required
}
```

The advice in this item applies equally to checked and unchecked exceptions. Whether an exception represents a predictable exceptional condition or a programming error, ignoring it with an empty `catch` block will result in a program that continues silently in the face of error. The program might then fail at an arbitrary time in the future, at a point in the code that bears no apparent relation to the source of the problem. Properly handling an exception can avert failure entirely. Merely letting an exception propagate outward can at least cause the program to fail swiftly, preserving information to aid in debugging the failure.
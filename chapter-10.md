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

Document every exception that can be thrown by each method that you write. This is true for unchecked as well as checked exceptions, and for abstract as well as concrete methods. Provide individual `throws` clauses for each checked exception and do not provide `throws` clauses for unchecked exceptions. If you fail to document the exceptions that your methods can throw, it will be difficult or impossible for others to make effective use of your classes and interfaces.

## Item 75: Include failure-capture information in detail messages

It is critically important that the exception’s toString method return as much information as possible concerning the cause of the failure. In other words, the detail message of an exception should capture the failure for subsequent analysis.

## Item 76: Strive for failure atomicity

After an object throws an exception, it is generally desirable that the object still be in a well-defined, usable state, even if the failure occurred in the midst of performing an operation. This is especially true for checked exceptions, from which the caller is expected to recover. Generally speaking, a failed method invocation should leave the object in the state that it was in prior to the invocation. A method with this property is said to be failure atomic.

## Item 77: Don’t ignore exceptions

It is easy to ignore exceptions by surrounding a method invocation with a `try` statement with an empty `catch` block:

```java
// Empty catch block ignores exception - Highly suspect!
try {
    ...
} catch (SomeException e) {
}
```

An empty catch block defeats the purpose of exceptions, which is to force you to handle exceptional conditions. At the very least, the catch block should contain a comment explaining why it is appropriate to ignore the exception.

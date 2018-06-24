# Chapter 8. Methods

This chapter discusses several aspects of method design: how to treat parameters and return values, how to design method signatures, and how to document methods. Much of the material in this chapter applies to constructors as well as to methods. Like Chapter 4, this chapter focuses on usability, robustness, and flexibility.

## Item 49: Check parameters for validity

Each time you write a method or constructor, you should think about what restrictions exist on its parameters. You should __document these restrictions and enforce them with explicit checks at the beginning of the method body__. It is important to get into the habit of doing this. The modest work that it entails will be paid back with interest the first time a validity check fails.

- This is a special case of the general principle that you should attempt to detect errors as soon as possible after they occur. Failing to do so makes it less likely that an error will be detected and makes it harder to determine the source of an error once it has been detected.
    - It is particularly important to check the validity of parameters that are not used by a method, but stored for later use (e.g. in constructors).

- For public and protected methods, use the Javadoc `@throws` tag to document the exception that will be thrown if a restriction on parameter values is violated

Example:
```java
/**
* Returns a BigInteger whose value is (this mod m). This method
* differs from the remainder method in that it always returns a
* non-negative BigInteger.
*
* @param m the modulus, which must be positive
* @return this mod m
* @throws ArithmeticException if m is less than or equal to 0
*/
public BigInteger mod(BigInteger m) {
    if (m.signum() <= 0)
        throw new ArithmeticException("Modulus <= 0: " + m);
    ... // Do the computation
}
```

- The `Objects.requireNonNull` method, added in Java 7, is flexible and convenient, so there’s no reason to perform null checks manually anymore. You can specify your own exception detail message if you wish. The method returns its input, so you can perform a null check at the same time as you use a value:

```java
// Inline use of Java's null-checking facility
this.strategy = Objects.requireNonNull(strategy, "strategy");
```

You can also ignore the return value and use `Objects.requireNonNull` as a freestanding null check where that suits your needs.

- For an unexported method, you, as the package author, control the circumstances under which the method is called, so you can and should ensure that only valid parameter values are ever passed in. Therefore, nonpublic methods can check their parameters using assertions, as shown below:

```java
// Private helper function for a recursive sort
private static void sort(long a[], int offset, int length) {
    assert a != null;
    assert offset >= 0 && offset <= a.length;
    assert length >= 0 && length <= a.length - offset;
    ... // Do the computation
}
```

In essence, these assertions are claims that the asserted condition will be true, regardless of how the enclosing package is used by its clients. Unlike normal validity checks, assertions throw `AssertionError` if they fail. And unlike normal validity checks, they have no effect and essentially no cost unless you enable them, which you do by passing the `-ea` (or `-enableassertions`) flag to the java command. For more information on assertions, see the tutorial [Asserts].

- Do not infer from this item that arbitrary restrictions on parameters are a good thing. On the contrary, __you should design methods to be as general as it is practical to make them. The fewer restrictions that you place on parameters, the better__, assuming the method can do something reasonable with all of the parameter values that it accepts. Often, however, some restrictions are intrinsic to the abstraction being implemented.

## Item 50: Make defensive copies when needed

One thing that makes Java a pleasure to use is that it is a safe language. This means that in the absence of native methods it is immune to buffer overruns, array overruns, wild pointers, and other memory corruption errors that plague unsafe languages such as C and C++. In a safe language, it is possible to write classes and to know with certainty that their invariants will hold, no matter what happens in any other part of the system. This is not possible in languages that treat all of memory as one giant array.

- __You must program defensively, with the assumption that clients of your class will do their best to destroy its invariants__. This is increasingly true as people try harder to break the security of systems, but more commonly, your class will have to cope with unexpected behavior resulting from the honest mistakes of well-intentioned programmers. Either way, it is worth taking the time to write classes that are robust in the face of ill-behaved clients.

- If you have mutable parameters to constructors, clients can modify an instance indirectly by modifying the parameter after construction. 
    - __It is essential to make a defensive copy of each mutable parameter to the constructor and to use the copies as components of the instance in place of the originals__:

```java
// Repaired constructor - makes defensive copies of parameters
public Period(Date start, Date end) {
    this.start = new Date(start.getTime());
    this.end = new Date(end.getTime());
    if (this.start.compareTo(this.end) > 0)
        throw new IllegalArgumentException(this.start + " after " + this.end);
}
```
    Note that defensive copies are made before checking the validity of the parameters (Item 49), and the validity check is performed on the copies rather than on the originals. While this may seem unnatural, it is necessary. It protects the class against changes to the parameters from another thread during the window of vulnerability between the time the parameters are checked and the time they are copied. In the computer security community, this is known as a time-of-check/time-of-use or TOCTOU attack. 

- Do not use the `clone` method to make a defensive copy of a parameter whose type is subclassable by untrusted parties.

- __Modify the accessors to return defensive copies of mutable internal fields__:

```java
// Repaired accessors - make defensive copies of internal fields
public Date start() {
    return new Date(start.getTime());
}

public Date end() {
    return new Date(end.getTime());
}
```

With the new constructor and the new accessors in place, `Period` is truly immutable. No matter how malicious or incompetent a programmer, there is simply no way to violate the invariant that the start of a period does not follow its end (without resorting to extralinguistic means such as native methods and reflection). This is true because there is no way for any class other than Period itself to gain access to either of the mutable fields in a Period instance. These fields are truly encapsulated within the object.

- Whether or not your class is immutable, you should think twice before returning a reference to an internal component that is mutable. Chances are, you should return a defensive copy. Remember that nonzero-length arrays are always mutable. Therefore, you should always make a defensive copy of an internal array before returning it to a client. Alternatively, you could return an immutable view of the array. Both of these techniques are shown in Item 15.

- The real lesson in all of this is that you should, where possible, __use immutable objects as components of your objects so that you that don’t have to worry about defensive copying (Item 17)__.

- Classes containing methods or constructors whose invocation indicates a transfer of control cannot defend themselves against malicious clients. Such classes are acceptable only when there is mutual trust between a class and its client or when damage to the class’s invariants would harm no one but the client.

If a class has mutable components that it gets from or returns to its clients, the class must defensively copy these components. If the cost of the copy would be prohibitive and the class trusts its clients not to modify the compo- nents inappropriately, then the defensive copy may be replaced by documentation outlining the client’s responsibility not to modify the affected components.

## Item 51: Design method signatures carefully

- Choose method names carefully. Names should always obey the standard naming conventions ([Item 68](chapter-9.md#item-68-adhere-to-generally-accepted-naming-conventions)).
- Don’t go overboard in providing convenience methods. Consider providing a “shorthand” only if it will be used often. When in doubt, leave it out.
- Avoid long parameter lists. Aim for four parameters or fewer.
    - There are three techniques for shortening overly long parameter lists. One is to __break the method up into multiple methods, each of which requires only a subset of the parameters__. If done carelessly, this can lead to too many methods, but it can also help reduce the method count by increasing orthogonality (each method does something completely distinct).
    - A second technique for shortening long parameter lists is to __create helper classes to hold groups of parameters__ (e.g. a `Card` class encoding suit and number). 
    - A third technique that combines aspects of the first two is to __adapt the Builder pattern (Item 2) from object construction to method invocation__. If you have a method with many parameters, especially if some of them are optional, it can be beneficial to define an object that represents all of the parameters and to allow the client to make multiple “setter” calls on this object, each of which sets a single parameter or a small, related group. Once the desired parameters have been set, the client invokes the object’s “execute” method, which does any final validity checks on the parameters and performs the actual computation.
- For parameter types, favor interfaces over classes ([Item 64](chapter-9.md#item-64-refer-to-objects-by-their-interfaces)). e.g. `Map` over `HashMap`. 
- Prefer two-element enum types to boolean parameters. Enums make your code easier to read and to write. Also, they make it easy to add more options later. For example, you might have a `Thermometer` type with a static factory that takes this enum:

```java
public enum TemperatureScale { FAHRENHEIT, CELSIUS }
```

Not only does `Thermometer.newInstance(TemperatureScale.CELSIUS)` make a lot more sense than `Thermometer.newInstance(true)`, but you can add `KELVIN` to `TemperatureScale` in a future release without having to add a new static factory to `Thermometer`.

## Item 52: Use overloading judiciously

- Overriding: the most specific method is used on an object.
- Overloading: the selection of which method to use is made at compile time based on the compile-time types of the parameters (no dynamic type resolution is run yet).
- Avoid confusing uses of overloading. Try to never export two overloadings with te same number of paramters. __You can always give methods different names instead of overloading them__.
- Overloading unavoidable with constructors. Exporting multiple overloadings with the same number of parameters is unlikely to confuse programmers if it is always clear which overloading will apply to any given set of actual parameters. This is the case when at least one corresponding formal parameter in each pair of overloadings has a “radically different” type in the two overloadings. Two types are radically different if it is clearly impossible to cast any non-null expression to both types. Under these circumstances, which overloading applies to a given set of actual parameters is fully determined by the runtime types of the parameters and cannot be affected by their compile-time types, so a major source of confusion goes away.


Just because you can overload methods doesn’t mean you should. You should generally refrain from overloading methods with multiple signatures that have the same number of parameters. In some cases, especially where constructors are involved, it may be impossible to follow this advice. In that case, you should at least avoid situations where the same set of parameters can be passed to different overloadings by the addition of casts. If such a situation cannot be avoided, for example, because you are retrofitting an existing class to implement a new interface, you should ensure that all overloadings behave identically when passed the same parameters. If you fail to do this, programmers will be hard pressed to make effective use of the overloaded method or constructor, and they won’t understand why it doesn’t work.

## Item 53: Use varargs judiciously

Varargs methods, formally known as variable arity methods [JLS, 8.4.1], accept zero or more arguments of a specified type. The varargs facility works by first creating an array whose size is the number of arguments passed at the call site, then putting the argument values into the array, and finally passing the array to the method. For example, here is a varargs method that takes a sequence of int arguments and returns their sum. As you would expect, the value of `sum(1, 2, 3)` is 6, and the value of `sum()` is 0:

```java
// Simple use of varargs
static int sum(int... args) {
    int sum = 0;
    for (int arg : args)
    sum += arg;
    return sum;
}
```

- If you need a minimum of x parameters (i.e. not 0+ parameters) then declare the method to take x normal parameters of the specified type and one varargs parameter of this type:

```java
// The right way to use varargs to pass one or more arguments
static int min(int firstArg, int... remainingArgs) {
    int min = firstArg;
    for (int arg : remainingArgs)
        if (arg < min)
            min = arg;
    return min;
}
```

- Exercise care when using varargs in performance-critical situations. Every invocation of a varargs method causes an array allocation and initialization. Suppose you’ve determined that 95 percent of the calls to a method have three or fewer parameters. Then declare five overloadings of the method, one each with zero through three ordinary parameters, and a single varargs method for use when the number of arguments exceeds three:

```java
public void foo() { }
public void foo(int a1) { }
public void foo(int a1, int a2) { }
public void foo(int a1, int a2, int a3) { }
public void foo(int a1, int a2, int a3, int... rest) { }
```

Now you know that you’ll pay the cost of the array creation only in the 5 percent of all invocations where the number of parameters exceeds three.

Varargs methods are a convenient way to define methods that require a variable number of arguments, but they should not be overused. They can produce confusing results if used inappropriately.

## Item 54: Return empty arrays or collections, not nulls

__Never return null in place of an empty array or collection__. It makes your API more difficult to use and more prone to error, and it has no performance advantages.

If you have evidence showing that allocating empty collections/arrays is harming performance, you can avoid the allocations by returning the same immutable empty collection/array repeatedly, as immutable objects may be shared freely (Item 17). Here is the code to do it, using the Collections.emptyList method. If you were returning a set, you’d use Collections.emptySet; if you were returning a map, you’d use Collections.emptyMap. But remember, this is an optimization, and it’s seldom called for. If you think you need it, measure performance before and after, to ensure that it’s actually helping:

```java
// Optimization - avoids allocating empty collections
public List<Cheese> getCheeses() {
    return cheesesInStock.isEmpty() ? Collections.emptyList()
        : new ArrayList<>(cheesesInStock);
}

// Optimization - avoids allocating empty arrays
private static final Cheese[] EMPTY_CHEESE_ARRAY = new Cheese[0];
public Cheese[] getCheeses() {
    return cheesesInStock.toArray(EMPTY_CHEESE_ARRAY);
}
```

## Item 55: Return optionals judiciously

Prior to Java 8, there were two approaches you could take when writing a method that was unable to return a value under certain circumstances. Either you could throw an exception, or you could return `null` (assuming the return type was an object reference type). Neither of these approaches is perfect. Exceptions should be reserved for exceptional conditions (Item 69), and throwing an exception is expensive because the entire stack trace is captured when an exception is created. Returning `null` doesn’t have these shortcomings, but it has its own. If a method returns null, clients must contain special-case code to deal with the possibility of a null return, unless the programmer can prove that a null return is impossible. If a client neglects to check for a null return and stores a null return value away in some data structure, a `NullPointerException` may result at some arbitrary time in the future, at some place in the code that has nothing to do with the problem.

In Java 8, there is a third approach to writing methods that may not be able to return a value. The `Optional<T>` class represents an immutable container that can hold either a single non-null `T` reference or nothing at all. An optional that contains nothing is said to be empty. A value is said to be present in an optional that is not empty. An optional is essentially an immutable collection that can hold at most one element.

A method that conceptually returns a T but may be unable to do so under certain circumstances can instead be declared to return an `Optional<T>`. This allows the method to return an empty result to indicate that it couldn’t return a valid result. An `Optional`-returning method is more flexible and easier to use than one that throws an exception, and it is less error-prone than one that returns `null`.

```java
// Returns maximum value in collection as an Optional<E>
public static <E extends Comparable<E>>
        Optional<E> max(Collection<E> c) {
    if (c.isEmpty())
        return Optional.empty();

    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
            result = Objects.requireNonNull(e);
    
    return Optional.of(result);
}
```

- `Optional.empty()` returns an empty optional, and `Optional.of(value)` returns an optional containing the given non-null value. It is a programming error to pass `null` to `Optional.of(value)`. If you do this, the method responds by throwing a `NullPointerException`. The `Optional.ofNullable(value)` method accepts a possibly null value and returns an empty optional if `null` is passed in. 

- __Never return a null value from an `Optional`-returning method: it defeats the entire purpose of the facility__.

- If a method returns an optional, the client gets to choose what action to take if the method can’t return a value. You can specify a default value:

```java
// Using an optional to provide a chosen default value
String lastWordInLexicon = max(words).orElse("No words...");
```

or you can throw any exception that is appropriate. Note that we pass in an exception factory rather than an actual exception. This avoids the expense of creating the exception unless it will actually be thrown:

```java
// Using an optional to throw a chosen exception
Toy myToy = max(toys).orElseThrow(TemperTantrumException::new);
```

If you can prove that an optional is nonempty, you can get the value from the optional without specifying an action to take if the optional is empty, but if you’re wrong, your code will throw a NoSuchElementException:

```java
// Using optional when you know there’s a return value
Element lastNobleGas = max(Elements.NOBLE_GASES).get();
``` 

- `Optional` provides the `isPresent()` method, which may be viewed as a safety valve. It returns `true` if the optional contains a value, `false` if it’s empty.

- __Container types, including collections, maps, streams, arrays, and optionals should not be wrapped in optionals__. Rather than returning an empty `Optional<List<T>>`, you should simply return an empty `List<T>` (Item 54). Returning the empty container will eliminate the need for client code to process an optional.

- __You should declare a method to return Optional<T> if it might not
be able to return a result and clients will have to perform special processing if no result is returned__.

- An `Optional` is an object that has to be allocated and initialized, and reading the value out of the optional requires an extra indirection. This makes optionals inappropriate for use in some performance-critical situations. Whether a particular method falls into this category can only be determined by careful measurement (Item 67).

- __It is almost never appropriate to use an optional as a key, value, or element in a collection or array__.

If you find yourself writing a method that can’t always return a
value and you believe it is important that users of the method consider this possibility every time they call it, then you should probably return an optional. You should, however, be aware that there are real performance consequences associated with returning optionals; for performance-critical methods, it may be better to return a null or throw an exception. Finally, __you should rarely use an optional in any other capacity than as a return value__.

## Item 56: Write doc comments for all exposed API elements

Documentation comments are the best, most effective way to document your API. Their use should be considered mandatory for all exported API elements. Adopt a consistent style that adheres to standard conventions. Remember that arbitrary HTML is permissible within documentation comments and that HTML metacharacters must be escaped.

- __Precede every exported class, interface, constructor, method, and field declaration with a doc comment__. If a class is serializable, you should also document its serialized form (Item 87). In the absence of a doc comment, the best that Javadoc can do is to reproduce the declaration as the sole documentation for the affected API element.

- __Whether or not a class or static method is threadsafe, you should document its thread-safety level__

- __The doc comment for a method should describe succinctly the contract between the method and its client__. With the exception of methods in classes designed for inheritance (Item 19), the contract should say what the method does rather than how it does its job. The doc comment should enumerate all of the method’s preconditions, which are the things that have to be true in order for a client to invoke it, and its postconditions, which are the things that will be true after the invocation has completed successfully. Typically, preconditions are described implicitly by the `@throws` tags for unchecked exceptions; each unchecked exception corresponds to a precondition violation. Also, preconditions can be specified along with the affected parameters in their `@param` tags.

- Methods should document any side effects. A side effect is an observable change in the state of the system that is not obviously required in order to achieve the postcondition. For example, if a method starts a background thread, the documentation should make note of it.

- By convention, the text following an `@param` tag or `@return` tag should be a __noun phrase__ describing the value represented by the parameter or return value. Rarely, arithmetic expressions are used in place of noun phrases; see `BigInteger` for examples. The text following an `@throws` tag should consist of the word “if,” followed by a clause describing the conditions under which the exception is thrown. By convention, the phrase or clause following an `@param`, `@return`, or `throws` tag is not terminated by a period. All of these conventions are illustrated by the following doc comment:

```java
/**
* Returns the element at the specified position in this list.
*
* <p>This method is <i>not</i> guaranteed to run in constant
* time. In some implementations it may run in time proportional
* to the element position.
*
* @param index index of element to return; must be
* non-negative and less than the size of this list
* @return the element at the specified position in this list
* @throws IndexOutOfBoundsException if the index is out of range
* ({@code index < 0 || index >= this.size()})
*/
E get(int index);
```

- The Javadoc utility translates doc comments into HTML, and arbitrary HTML elements in doc comments end up in the resulting HTML document. Occasionally, programmers go so far as to embed HTML tables in their doc comments, although this is rare.

- To include a multiline code example in a doc comment, use a Javadoc `{@code}` tag wrapped inside an HTML `<pre>` tag. In other words, precede the code example with the characters `<pre>{@code` and follow it with `}</pre>`. This preserves line breaks in the code, and eliminates the need to escape HTML metacharacters, but not the at sign (`@`), which must be escaped if the code sample uses annotations.

- As mentioned in Item 15, when you design a class for inheritance, you must document its self-use patterns, so programmers know the semantics of overriding its methods. These self-use patterns should be documented using the `@implSpec` tag, added in Java 8. As of Java 9, the Javadoc utility still ignores the `@implSpec` tag unless you pass the command line switch `-tag "implSpec:a:Implementation Requirements:"`.

- Escape HTML metacharacters, such as the less-than sign (<), the greater-than sign (>), and the ampersand (&). The best way to get these characters into documentation is to surround them with the {@literal} tag, which suppress processing of HTML markup and nested Javadoc tags.

    e.g. `A geometric series converges if {@literal |r| < 1}`

- The first “sentence” of each doc comment (as defined below) becomes the summary description of the element to which the comment pertains. __No two members or constructors in a class or interface should have the same summary description__.

- The summary description ends (i.e. gets cut off) at the first period that is followed by a space, tab, or line terminator (or at the first block tag) [Javadoc-ref]. The best solution is to surround the offending period and any associated text with an `{@literal}` tag, so the period is no longer followed by a space in the source code:

```java
/**
* A college degree, such as B.S., {@literal M.S.} or Ph.D.
*/
public class Degree { ... }
```

- Can index items in your Javadocs to make them searchable with the built-in client-side index for Javadocs HTML pages
    e.g. `This method complies with the {@index IEEE 754} standard.`

- For your custom annotation types: be sure to document any members as well as the type itself. Document members with noun phrases, as if they were fields. For the summary description of the type, use a verb phrase that says what it means when a program element has an annotation of this type:

```java
/**
* Indicates that the annotated method is a test method that
* must throw the designated exception to pass.
*/
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest {
    /**
    * The exception that the annotated test method must throw
    * in order to pass. (The test is permitted to throw any
    * subtype of the type described by this class object.)
    */
    Class<? extends Throwable> value();
}
```

- Package-level doc comments should be placed in a file named `packageinfo.java`. In addition to these comments, `package-info.java` must contain a package declaration and may contain annotations on this declaration. Similarly, if you elect to use the module system (Item 15), module-level comments should be placed in the `module-info.java` file.
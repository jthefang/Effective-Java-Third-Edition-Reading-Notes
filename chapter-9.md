# Chapter 9: General Programming

This chapter is devoted to the nuts and bolts of the language. It discusses local variables, control structures, libraries, data types, and two extralinguistic facilities: *reflection* and *native methods*. Finally, it discusses optimization and naming conventions.

## Item 57: Minimize the scope of local variables

- Declare a local variable where it is first used. If a variable is declared before it is used, it’s just clutter—one more thing to distract the reader who is trying to figure out what the program does. By the time the variable is used, the reader might not remember the variable’s type or initial value.
	- Variables declared in a loop (even in the for loop declaration itself) are local only to scope of the loop
- Nealy every local variable declaration should contain an initializer (i.e. it should be initialized to some value), except `try-catch` statements.
- Prefer for loops to while loops.
- Keep methods small and focused.

## Item 58: Prefer for-each loops to traditional for loops

```java
// The preferred idiom for iterating over collections and arrays: for-each loop
for (Element e : elements) { //for e in elements
	... // Do something with e
}
```

The for-each loop provides compelling advantages over the traditional `for` loop in clarity, flexibility, and bug prevention, with no performance penalty. You should use it wherever you can. Unfortunately, there are three common situations where you can’t use a for-each loop:

1. **Destructive filtering** — If you need to traverse a collection and remove selected elements, then you need to use an explicit iterator so that you can call its `remove` method. You can often avoid explicit traversal by using `Collection`’s `removeIf` method, added in Java 8.
2. **Transforming** — If you need to traverse a list or array and replace some or all of the values of its elements, then you need the list iterator or array index in order to set the value of an element.
3. **Parallel iteration** — If you need to traverse multiple collections in parallel, then you need explicit control over the iterator or index variable, so that all iterators or index variables can be advanced in lockstep.

- The for-each loop lets you iterate over any object that implements the `Iterable` interface, which consists of a single method:

```java
public interface Iterable<E> {
	// Returns an iterator over the elements in this iterable
	Iterator<E> iterator();
}
```

It is a bit tricky to implement Iterable if you have to write your own Iterator implementation from scratch, but if you are writing a type that represents a group of elements, you should strongly consider having it implement `Iterable`, even if you choose not to have it implement `Collection`. This will allow your users to iterate over your type using the for-each loop, and they will be forever grateful.

- Here is another loop idiom that minimizes the scope of local variables and caches the result of the limit computation:

```java
for (int i = 0, n = expensiveComputation(); i < n; i++) {
... // Do something with i;
}
```

The important thing to notice about this idiom is that it has two loop variables, `i` and `n`, both of which have exactly the right scope. The second variable, `n`, is used to store the limit of the first, thus avoiding the cost of a redundant computation in every iteration. As a rule, you should use this idiom if the loop test involves a method invocation that is guaranteed to return the same result on each iteration.

## Item 59: Know and use the libraries

Don’t reinvent the wheel. By using a standard library, you take advantage of the knowledge of the experts who wrote it and the experience of those who used it before you.

- The performance of standard libraries tends to improve over time, with no effort on your part.

- They tend to gain functionality over time

- Your code becomes "mainstream", i.e. it's more accessible, therefore maintainable and reusable by other developers.

- Numerous features are added to the libraries in every major release, and it pays to keep abreast of these additions.

- Every programmer should be familiar with the basics of `java.lang`, `java.util`, and `java.io`, and their subpackages. Knowledge of other libraries can be acquired on an as-needed basis.
	- The collections framework and the streams library (Items 45–48) should be part of every programmer’s basic toolkit, as should parts of the concurrency utilities in `java.util.concurrent`. This package contains both high-level utilities to simplify the task of multithreaded programming and low-level primitives to allow experts to write their own higherlevel concurrent abstractions. The high-level parts of `java.util.concurrent` are discussed in Items 80 and 81.
	- If you can’t find what you need in Java platform libraries, your next choice should be to look in high-quality third-party libraries, such as Google’s excellent, open source Guava library [Guava]. If you can’t find the functionality that you need in any appropriate library, you may have no choice but to implement it yourself.

- As of Java 7, you should no longer use Random. For most uses, the random number generator of choice is now `ThreadLocalRandom`. *It produces higher quality random numbers, and it’s very fast.* On my machine, it is 3.6 times faster than Random. For fork join pools and parallel streams, use SplittableRandom.

## Item 60: Avoid float and double if exact answers are required

- __The float and double types are particularly ill-suited for monetary calculations__ because it is impossible to represent 0.1 (or any other negative power of ten) as a float or double exactly.
	- Get cases where value is .0000000000000001 off
	- __Use BigDecimal, int, or long for monetary calculations__. Use `BigDecimal`’s `String` constructor rather than its `double` constructor to avoid introducing inaccurate values into the computation.
	- Using `BigDecimal` is a lot slower and less convenient than a primitive
	- You could use `int` or `long`, depending on the amounts involved, and keep track of the decimal point yourself


Don’t use float or double for any calculations that require an exact answer. Use `BigDecimal` if you want the system to keep track of the decimal point and you don’t mind the inconvenience and cost of not using a primitive type. Using `BigDecimal` has the added advantage that it gives you full control over rounding, letting you select from eight rounding modes whenever an operation that entails rounding is performed. This comes in handy if you’re performing business calculations with legally mandated rounding behavior. If performance is of the essence, you don’t mind keeping track of the decimal point yourself, and the quantities aren’t too big, use `int` or `long`. If the quantities don’t exceed nine decimal digits, you can use `int`; if they don’t exceed eighteen digits, you can use `long`. If the quantities might exceed eighteen digits, you must use `BigDecimal`.

## Item 61: Prefer primitive types to boxed primitives

There are three major differences between primitives and boxed primitives.
	1. Primitives have only their values, whereas boxed primitives have identities distinct from their values. In other words, two boxed primitive instances can have the same value and different identities. 
	2. Primitive types have only fully functional values, whereas each boxed primitive type has one nonfunctional value, which is null, in addition to all the functional values of the corresponding primitive type. 
	3. Primitives are more time- and space-efficient than boxed primitives. All three of these differences can get you into real trouble if you aren’t careful.

- Legitimate uses of boxed primitives:
	- As elements, keys, and values in collections. You can’t put primitives in collections, so you’re forced to use boxed primitives.
	- You must use boxed primitives as type parameters in parameterized types and methods (Chapter 5), because the language does not permit you to use primitives
	- You must use boxed primitives when making reflective method invocations

Use primitives in preference to boxed primitives whenever you have the choice. Primitive types are simpler and faster. If you must use boxed primitives, be careful! Autoboxing reduces the verbosity, but not the danger, of using boxed primitives. When your program compares two boxed primitives with the == operator, it does an identity comparison, which is almost certainly not what you want. When your program does mixed-type computations involving boxed and unboxed primitives, it does unboxing, and __when your program does unboxing, it can throw a NullPointerException__. Finally, when your program boxes primitive values, it can result in costly and unnecessary object creations.

## Item 62: Avoid strings where other types are more appropriate

- Strings are poor substitutes for other value types. When a piece of data comes into a program from a file, from the network, or from keyboard input, it is often in string form. There is a natural tendency to leave it that way, but this tendency is justified only if the data really is textual in nature. If it’s numeric, it should be translated into the appropriate numeric type, such as int, float, or BigInteger. If it’s the answer to a yes-or-no question, it should be translated into an appropriate enum type or a boolean. More generally, if there’s an appropriate value type, whether primitive or object reference, you should use it; if there isn’t, you should write one.

- Strings are poor substitutes for enum types. As discussed in Item 34, enums make far better enumerated type constants than strings

Avoid the natural tendency to represent objects as strings when better data types exist or can be written. Used inappropriately, strings are more cumbersome, less flexible, slower, and more error-prone than other types. Types for which strings are commonly misused include primitive types, enums, and aggregate types (e.g. representing a list as a comma separated string).

## Item 63: Beware the performance of string concatenation

__Using the string concatenation operator repeatedly to concatenate n strings requires time quadratic in `n`__. This is an unfortunate consequence of the fact that strings are immutable (Item 17). When two strings are concatenated, the contents of both are copied.
	- To achieve acceptable performance, use a `StringBuilder` in place of a `String` to store the statement under construction:

```java
public String statement() {
	StringBuilder b = new StringBuilder(numItems() * LINE_WIDTH);
	for (int i = 0; i < numItems(); i++)
		b.append(lineForItem(i));
	return b.toString();
}
```

Don’t use the string concatenation operator to combine more than a few strings unless performance is irrelevant. Use `StringBuilder`’s append method instead. Alternatively, use a character array, or process the strings one at a time instead of combining them.

## Item 64: Refer to objects by their interfaces

[Item 51](chapter-8.md#item-51-design-method-signatures-carefully) contains the advice that you should use interfaces rather than classes as parameter types. More generally, you should favor the use of interfaces rather than classes to refer to objects. __If appropriate interface types exist, then parameters, return values, variables, and fields should all be declared using interface types__. The only time you really need to refer to an object’s class is when you’re creating it with a constructor. 

- If there is no appropriate interface, just use the least specific class in the class hierarchy that provides the required functionality.

- __If you get into the habit of using interfaces as types, your program will be much more flexible__.

Example:
```java
// Good - uses interface as type
Set<Son> sonSet = new LinkedHashSet<>();

// Bad - uses class as type!
LinkedHashSet<Son> sonSet = new LinkedHashSet<>();
```

Then you can replace `LinkedHashSet` with `HashSet` anytime by making a single change at construction.

There is one caveat: if the original implementation offered some special functionality not required by the general contract of the interface and the code depended on that functionality, then it is critical that the new implementation (i.e. the substitute in a code replacement) provide the same functionality.

## Item 65: Prefer interfaces to reflection

Reflection is a powerful facility that is required for certain sophisticated system programming tasks, but it has many disadvantages. If you are writing a program that has to work with classes unknown at compile time, you should, if at all possible, use reflection only to instantiate objects, and access the objects using some interface or superclass that is known at compile time.

## Item 66: Use native methods judiciously

Think twice before using native methods. Rarely, if ever, use them for improved performance. If you must use native methods to access low-level resources or legacy libraries, use as little native code as possible and test it thoroughly. A single bug in the native code can corrupt your entire application.

## Item 67: Optimize judiciously

Do not strive to write fast programs—strive to write good ones; speed will follow. Do think about performance issues while you’re designing systems and especially while you’re designing APIs, wire-level protocols, and persistent data formats. When you’ve finished building the system, measure its performance. If it’s fast enough, you’re done. If not, locate the source of the problems with the aid of a profiler, and go to work optimizing the relevant parts of the system. The first step is to examine your choice of algorithms: no amount of low-level optimization can make up for a poor choice of algorithm. Repeat this process as necessary, measuring the performance after every change, until you’re satisfied.

## Item 68: Adhere to generally accepted naming conventions

Internalize the standard naming conventions and learn to use them as second nature.
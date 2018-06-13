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

The core reflection facility, `java.lang.reflect`, offers programmatic access to arbitrary classes. Given a `Class` object, you can obtain `Constructor`, `Method`, and `Field` instances representing the constructors, methods, and fields of the class represented by the `Class` instance. These objects provide programmatic access to the class’s member names, field types, method signatures, and so on.

Moreover, `Constructor, Method`, and `Field` instances let you manipulate their underlying counterparts reflectively: you can construct instances, invoke methods, and access fields of the underlying class by invoking methods on the `Constructor, Method`, and `Field` instances. For example, `Method.invoke` lets you invoke any method on any object of any class (subject to the usual security constraints). Reflection allows one class to use another, even if the latter class did not exist when the former was compiled. This power, however, comes at a price:
	- __You lose all the benefits of compile-time type checking__, including exception checking. If a program attempts to invoke a nonexistent or inaccessible method reflectively, it will fail at runtime unless you’ve taken special precautions.
	- __The code required to perform reflective access is clumsy and verbose__. It is tedious to write and difficult to read.
	- __Performance suffers__. Reflective method invocation is much slower than normal method invocation. 

- Examples of applications that require reflection: code analysis tools and ependency injection frameworks.

- __If you have any doubts as to whether your application requires reflection it probably doesn't.__

- __You can obtain many of the benefits of reflection while incurring few of its costs by using it only in a very limited form__. For many programs that must use a class that is unavailable at compile time, there exists at compile time an appropriate interface or superclass by which to refer to the class (Item 64). If this is the case, you can __create instances reflectively and access them normally via their interface or superclass__.

Example: For example, here is a program that creates a `Set<String>` instance whose class is specified by the first command line argument. The program inserts the remaining command line arguments into the set and prints it. Regardless of the first argument, the program prints the remaining arguments with duplicates eliminated. The order in which these arguments are printed, however, depends on the class specified in the first argument. If you specify `java.util.HashSet`, they’re printed in apparently random order; if you specify `java.util.TreeSet`, they’re printed in alphabetical order because the elements in a TreeSet are sorted:

```java
// Reflective instantiation with interface access
public static void main(String[] args) {
	// Translate the class name into a Class object
	Class<? extends Set<String>> cl = null;
	try {
	cl = (Class<? extends Set<String>>) // Unchecked cast!
	Class.forName(args[0]);
	} catch (ClassNotFoundException e) {
	fatalError("Class not found.");
	}
	// Get the constructor
	Constructor<? extends Set<String>> cons = null;
	try {
	cons = cl.getDeclaredConstructor();
	} catch (NoSuchMethodException e) {
	fatalError("No parameterless constructor");
	}
	// Instantiate the set
	Set<String> s = null;
	try {
	s = cons.newInstance();
	} catch (IllegalAccessException e) {
	fatalError("Constructor not accessible");
	} catch (InstantiationException e) {
	fatalError("Class not instantiable.");
	} catch (InvocationTargetException e) {
	fatalError("Constructor threw " + e.getCause());
	} catch (ClassCastException e) {
	fatalError("Class doesn't implement Set");
	}
	// Exercise the set
	s.addAll(Arrays.asList(args).subList(1, args.length));
	System.out.println(s);
}

private static void fatalError(String msg) {
	System.err.println(msg);
	System.exit(1);
}
```

Reflection is a powerful facility that is required for certain sophisticated system programming tasks, but it has many disadvantages. If you are writing a program that has to work with classes unknown at compile time, you should, if at all possible, use reflection only to instantiate objects, and access the objects using some interface or superclass that is known at compile time.

## Item 66: Use native methods judiciously

The Java Native Interface (JNI) allows Java programs to call native methods, which are methods written in native programming languages such as C or C++. Historically, native methods have had three main uses. They provide access to platform-specific facilities such as registries. They provide access to existing libraries of native code, including legacy libraries that provide access to legacy data. Finally, native methods are used to write performance-critical parts of applications in native languages for improved performance.

It is legitimate to use native methods to access platform-specific facilities, but it is seldom necessary: as the Java platform matured, it provided access to many features previously found only in host platforms. For example, the process API, added in Java 9, provides access to OS processes. It is also legitimate to use native methods to use native libraries when no equivalent libraries are available in Java.

- It is rarely advisable to use native methods for improved performance. In early releases (prior to Java 3), it was often necessary, but JVMs have gotten much faster since then. For most tasks, it is now possible to obtain comparable performance in Java.

- Use of native languages come with disadvantages:
	- they aren't easily portable
	- are not safe (i.e. susceptible to memory corruption errors)
	- harder to debug
	- require tedious glue code (JNI)

Think twice before using native methods. Rarely, if ever, use them for improved performance. If you must use native methods to access low-level resources or legacy libraries, use as little native code as possible and test it thoroughly. A single bug in the native code can corrupt your entire application.

## Item 67: Optimize judiciously

More computing sins are committed in the name of efficiency (without necessarily achieving it) than for any other single reason—including blind stupidity.
—William A. Wulf [Wulf72]

We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil.
—Donald E. Knuth [Knuth74]

We follow two rules in the matter of optimization:
	Rule 1. Don’t do it.
	Rule 2 (for experts only). __Don’t do it yet—that is, not until you have a perfectly clear and unoptimized solution__.
—M. A. Jackson [Jackson75]

Basically it is easy to do more harm than good, especially if you optimize prematurely. In the process, you may produce software that is neither fast nor correct and cannot easily be fixed.

- Strive to write good programs rather than fast ones. If a good program is not fast enough, its architecture will allow it to be optimized. Good programs embody the principle of information hiding: where possible, they localize design decisions within individual components, so individual decisions can be changed without affecting the remainder of the system (Item 15).

- Strive to avoid design decisions that limit performance. The components of a design that are most difficult to change after the fact are those specifying interactions between components and with the outside world. Chief among these design components are APIs, wire-level protocols, and persistent data formats. Not only are these design components difficult or impossible to change after the fact, but all of them can place significant limitations on the performance that a system can ever achieve.

- Measure performance before and after each attempted optimization. You may be surprised by what you find. Often, attempted optimizations have no measurable effect on performance; sometimes, they make it worse. The main reason is that it’s difficult to guess where your program is spending its time. The part of the program that you think is slow may not be at fault, in which case you’d be wasting your time trying to optimize it. Common wisdom says that programs spend 90 percent of their time in 10 percent of their code.
	- __Profiling tools can help you decide where to focus your optimization efforts. These tools give you runtime information, such as roughly how much time each method is consuming and how many times it is invoked__. In addition to focusing your tuning efforts, this can alert you to the need for algorithmic changes. If a quadratic (or worse) algorithm lurks inside your program, no amount of tuning will fix the problem. You must replace the algorithm with one that is more efficient. The more code in the system, the more important it is to use a profiler. It’s like looking for a needle in a haystack: the bigger the haystack, the more useful it is to have a metal detector. __Another tool that deserves special mention is jmh, which is not a profiler but a microbenchmarking framework that provides unparalleled visibility into the detailed performance of Java code__.

Do not strive to write fast programs — strive to write good ones; speed will follow. But do think about performance while you’re designing systems, especially while you’re designing APIs, wire-level protocols, and persistent data formats. When you’ve finished building the system, measure its performance. If it’s fast enough, you’re done. If not, locate the source of the problem with the aid of a profiler and go to work optimizing the relevant parts of the system. The first step is to examine your choice of algorithms: no amount of low-level optimization can make up for a poor choice of algorithm. Repeat this process as necessary, measuring the performance after every change, until you’re satisfied.

## Item 68: Adhere to generally accepted naming conventions

- Package and module names should be hierarchical with the components separated by periods. Components should consist of lowercase alphabetic characters and, rarely, digits. The name of any package that will be used outside your organization should begin with your organization’s Internet domain name with the components reversed, for example, `edu.cmu`, `com.google`, `org.eff`. The standard libraries and optional packages, whose names begin with `java` and `javax`, are exceptions to this rule. __Users must not create packages or modules whose names begin with java or javax__.
	- The remainder of a package name should consist of one or more components describing the package. Components should be short, generally eight or fewer characters. Meaningful abbreviations are encouraged, for example, `util` rather than `utilities`. Acronyms are acceptable, for example, `awt`. Components should generally consist of a single word or abbreviation.

- Class and interface names, including enum and annotation type names, should consist of one or more words, with the first letter of each word capitalized, for example, `List` or `FutureTask`. Abbreviations are to be avoided, except for acronyms and certain common abbreviations like `max` and `min`. 
	- Capitalize only the first letter in abbreviations (e.g. `HttpUrl` vs `HTTPURL`)

- Constant field names should consists of uppercase words sepearted by underscores, the only recommended use of underscores.

- Input parameters shoudl be named very carefully because their names are an integral part of their method's documentation.

- Type parameter names usually consist of a single letter. Most commonly it is one of these five: `T` for an arbitrary type, `E` for the element type of a collection, `K` and `V` for the key and value types of a map, and `X` for an exception. The return type of a function is usually `R`. A sequence of arbitrary types can be `T`, `U`, `V` or `T1`, `T2`, `T3`.

Reference for typographical conventions:
| Identifier Type    | Examples |
|--------------------|----------|
| Package or module  | `org.junit.jupiter.api, com.google.common.collect` |
| Class or Interface | `Stream, FutureTask, LinkedHashMap, HttpClient`|
| Method or Field    | `remove, groupingBy, getCrc` |
| Constant Field     | `MIN_VALUE, NEGATIVE_INFINITY` |
| Local Variable     | `i, denom, houseNum` |
| Type Parameter     | `T, E, K, V, X, R, U, V, T1, T2` |

Instance methods that convert the type of an object, returning an independent object of a different type, are often called `to`*Type*, for example, `toString` or `toArray`. Methods that return a view (Item 6) whose type differs from that of the receiving object are often called `as`*Type*, for example, `asList`. Methods that return a primitive with the same value as the object on which they’re invoked are often called *type*`Value`, for example, `intValue`. Common names for static factories include `from`, `of`, `valueOf`, `instance`, `getInstance`, `newInstance`, `getType`, and `newType` (Item 1, page 9).
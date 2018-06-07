# Chapter 7. Lambdas and Streams

In Java 8, functional interfaces, lambdas, and method references were added to make it easier to create function objects. 
The streams API was added in tandem with these language changes to provide library support for processing sequences of data 
elements. In this chapter, we discuss how to make best use of these facilities.

## Item 42: Prefer lambdas to anonymous classes

Historically, interfaces (or, rarely, abstract classes) with a single abstract method were used as function types. Their instances, known as function objects, represent functions or actions. Since JDK 1.1 was released in 1997, the primary means of creating a function object was the *anonymous class* (Item 24). Here’s a code snippet to sort a list of strings in order of length, using an anonymous class to create the sort’s comparison function (which imposes the sort order):

```java
// Anonymous class instance as a function object - obsolete!
Collections.sort(words, new Comparator<String>() {
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
});
```

Anonymous classes were adequate for the classic objected-oriented design patterns requiring function objects, notably the Strategy pattern. The `Comparator` interface represents an abstract strategy for sorting; the anonymous class above is a concrete strategy for sorting strings. The verbosity of anonymous classes, however, made functional programming in Java an unappealing prospect.

In Java 8, the language formalized the notion that interfaces with a single abstract method are special and deserve special treatment. These interfaces are now known as functional interfaces, and the language allows you to create instances of these interfaces using *lambda expressions*, or lambdas for short. Lambdas are similar in function to anonymous classes, but far more concise.

Here’s how the code snippet above looks with the anonymous class replaced by a lambda. The boilerplate is gone, and the behavior is clearly evident:

```java
// Lambda expression as function object (replaces anonymous class)
Collections.sort(words,
    (s1, s2) -> Integer.compare(s1.length(), s2.length()));
```

Note that *the types of the lambda* (`Comparator<String>`), of its parameters (`s1` and `s2`, both `String`), and of its return value (`int`) are not present in the code. The compiler deduces these types from context, using a process known as type inference. In some cases, the compiler won’t be able to determine the types, and you’ll have to specify them. The rules for type inference are complex: they take up an entire chapter in the JLS [JLS, 18]. Few programmers understand these rules in detail, but that’s OK. __Omit the types of all lambda parameters unless their presence makes your program clearer__. If the compiler generates an error telling you it can’t infer the type of a lambda parameter, then specify it. Sometimes you may have to cast the return value or the entire lambda expression, but this is rare.

Incidentally, the comparator in the snippet can be made even more succinct if a comparator construction method is used in place of a lambda (Items 14. 43):

```java
Collections.sort(words, comparingInt(String::length));
```

In fact, the snippet can be made still shorter by taking advantage of the `sort` method that was added to the `List` interface in Java 8:

```java
words.sort(comparingInt(String::length));
```

- Another example from Item 34:

```java
// Enum with function object fields & constant-specific behavior
public enum Operation {
    PLUS ("+", (x, y) -> x + y),
    MINUS ("-", (x, y) -> x - y),
    TIMES ("*", (x, y) -> x * y),
    DIVIDE("/", (x, y) -> x / y);

    private final String symbol;
    private final DoubleBinaryOperator op;

    Operation(String symbol, DoubleBinaryOperator op) {
        this.symbol = symbol;
        this.op = op;
    }

    @Override 
    public String toString() { return symbol; }
    
    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }
}
```

Note that we’re using the DoubleBinaryOperator interface for the lambdas that represent the enum constant’s behavior. This is one of the many predefined functional interfaces in java.util.function (Item 44). It represents a function that takes two double arguments and returns a double result.

- Unlike methods and classes, __lambdas lack names and documentation; if a computation isn’t self-explanatory, or exceeds a few lines, don’t put it in a lambda__. One line is ideal for a lambda, and three lines is a reasonable maximum. If you violate this rule, it can cause serious harm to the readability of your programs. If a lambda is long or difficult to read, either find a way to simplify it or refactor your program to eliminate it.

- The arguments passed to enum constructors are evaluated in a static context. Thus, lambdas in enum constructors can’t access instance members of the enum. Constant-specific class bodies are still the way to go if an enum type has constant-specific behavior that is difficult to understand, that can’t be implemented in a few lines, or that requires access to instance fields or methods.

- Lambdas are limited to functional interfaces. If you want to create an instance of an abstract class, you can do it with an anonymous class, but not a lambda. Similarly, you can use anonymous classes to create instances of interfaces with multiple abstract methods.

- In a lambda, the `this` keyword refers to the enclosing instance, which is typically what you want. In an anonymous class, the `this` keyword refers to the anonymous class instance. If you need access to the function object from within its body, then you must use an anonymous class.

- __You should rarely, if ever, serialize a lambda__ (or an anonymous class instance). If you have a function object that you want to make serializable, such as a `Comparator`, use an instance of a private static nested class (Item 24).

As of Java 8, lambdas are by far the best way to represent small function objects. __Don’t use anonymous classes for function objects unless you have to create instances of types that aren’t functional interfaces__. Also, remember that lambdas make it so easy to represent small function objects that it opens the door to functional programming techniques that were not previously practical in Java.

## Item 43: Prefer method references to lambdas
The primary advantage of lambdas over anonymous classes is that they are more succinct. Java provides a way to generate function objects even more succinct than lambdas: method references. Here is a code snippet from a program that maintains a map from arbitrary keys to `Integer` values. If the value is interpreted as a count of the number of instances of the key, then the program is a multiset implementation. The function of the code snippet is to associate the number 1 with the key if it is not in the map and to increment the associated value if the key is already present:

```java
map.merge(key, 1, (count, incr) -> count + incr);
```

Note that this code uses the `merge` method, which was added to the `Map` interface in Java 8. If no mapping is present for the given key, the method simply inserts the given value; if a mapping is already present, `merge` applies the given function to the current value and the given value and overwrites the current value with the result. This code represents a typical use case for the `merge` method.

The code reads nicely, but there’s still some boilerplate. The parameters `count` and `incr` don’t add much value, and they take up a fair amount of space. Really, all the lambda tells you is that the function returns the sum of its two arguments. As of Java 8, `Integer` (and all the other boxed numerical primitive types) provides a static method `sum` that does exactly the same thing. We can simply pass a reference to this method and get the same result with less visual clutter:

```java
map.merge(key, 1, Integer::sum);
```

The more parameters a method has, the more boilerplate you can eliminate with a method reference. In some lambdas, however, the parameter names you choose provide useful documentation, making the lambda more readable and maintainable than a method reference, even if the lambda is longer.

- There’s nothing you can do with a method reference that you can’t also do with a lambda (with one obscure exception—see JLS, 9.9-2 if you’re curious).

- Method references usually result in shorter, clearer code. They also give you an out if a lambda gets too long or complex: You can extract the code from the lambda into a new method and replace the lambda with a reference to that method. You can give the method a good name and document it to your heart’s content.

- You should nearly always replace a lambda with a method reference except when the lambda will be more succinct. This happens most often when the method is in the same class as the lambda:

```java
service.execute(GoshThisClassNameIsHumongous::action);

//The lambda equivalent looks like this:
service.execute(() -> action());
```

- the `Function` interface provides a generic static factory method to return the identity function, `Function.identity()`. It’s typically shorter and cleaner not to use this method but to code the equivalent lambda inline: `x -> x`.

- Many method references refer to static methods, but there are four kinds that
do not.
    1. Bound references: the receiving object (of the method) is specified in the method reference. The function object takes the same arguments as the referenced method
    2. Unbound references: the receiving object is specified when the function object is applied, via an additional parameter before the method's declared parameters (like Python's `self` param)
    3. Constructor references serve as factory objects and there are 2 kinds, for classes and arrays

| __Method Ref Type__ | __Example__              | __Lambda Equivalent__           |
| ------------------- | ------------------------ | -----------------------------   |
| Static              | `Integer::parseInt`      | `str -> Integer.parseInt(str)`  |
| Bound               | `Instant.now()::isAfter` | `Instant then = Instant.now();` <br/> `t -> then.isAfter(t)` |
| Unbound             | `String::toLowerCase`    | `str -> str.toLowerCase()`      |
| Class Constructor   | `TreeMap<K,V>::new`      | `() -> new TreeMap<K,V>`        |
| Array Constructor   | `int[]::new`             | `len -> new int[len]`           |

Method references often provide a more succinct alternative to lambdas. __Where method references are shorter and clearer, use them; where they aren’t, stick with lambdas__.

## Item 44: Favor the use of standard functional interfaces

Now that Java has lambdas, best practices for writing APIs have changed considerably. For example, the Template Method pattern, wherein a subclass overrides a primitive method to specialize the behavior of its superclass, is far less attractive. The modern alternative is to provide a static factory or constructor that accepts a function object to achieve the same effect. More generally, you’ll be writing more constructors and methods that take function objects as parameters. Choosing the right functional parameter type demands care.

Now that Java has lambdas, it is imperative that you design your APIs with lambdas in mind. Accept functional interface types on input and return them on output. It is generally best to use the standard interfaces provided in java.util.function.Function, but keep your eyes open for the relatively rare cases where you would be better off writing your own functional interface.

## Item 45: Use streams judiciously

The streams API was added in Java 8 to ease the task of performing bulk operations, sequentially or in parallel. This API provides two key abstractions: the stream, which represents a finite or infinite sequence of data elements, and the stream pipeline, which represents a multistage computation on these elements. The elements in a stream can come from anywhere. Common sources include collections, arrays, files, regular expression pattern matchers, pseudorandom number generators, and other streams. The data elements in a stream can be object references or primitive values. Three primitive types are supported: int, long, and double.

A stream pipeline consists of a source stream followed by zero or more intermediate operations and one terminal operation. Each intermediate operation transforms the stream in some way, such as mapping each element to a function of that element or filtering out all elements that do not satisfy some condition. Intermediate operations all transform one stream into another, whose element type may be the same as the input stream or different from it. The terminal operation performs a final computation on the stream resulting from the last intermediate operation, such as storing its elements into a collection, returning a certain element, or printing all of its elements.

Stream pipelines are evaluated lazily: evaluation doesn’t start until the terminal operation is invoked, and data elements that aren’t required in order to complete the terminal operation are never computed. This lazy evaluation is what makes it possible to work with infinite streams. Note that a stream pipeline without a terminal operation is a silent no-op, so don’t forget to include one.

The streams API is fluent: it is designed to allow all of the calls that comprise a pipeline to be chained into a single expression. In fact, multiple pipelines can be chained together into a single expression.

By default, stream pipelines run sequentially. Making a pipeline execute in parallel is as simple as invoking the parallel method on any stream in the pipeline, but it is seldom appropriate to do so (Item 48).

The streams API is sufficiently versatile that practically any computation can be performed using streams, but just because you can doesn’t mean you should. When used appropriately, streams can make programs shorter and clearer; when used inappropriately, they can make programs difficult to read and maintain. There are no hard and fast rules for when to use streams, but there are heuristics.

Some tasks are best accomplished with streams, and others with iteration. Many tasks are best accomplished by combining the two approaches. There are no hard and fast rules for choosing which approach to use for a task, but there are some useful heuristics. In many cases, it will be clear which approach to use; in some cases, it won’t. __If you’re not sure whether a task is better served by streams or iteration, try both and see which works better__.

## Item 46: Prefer side-effect-free functions in streams

If you’re new to streams, it can be difficult to get the hang of them. Merely expressing your computation as a stream pipeline can be hard. When you succeed, your program will run, but you may realize little if any benefit. Streams isn’t just an API, it’s a paradigm based on functional programming. In order to obtain the expressiveness, speed, and in some cases parallelizability that streams have to offer, you have to adopt the paradigm as well as the API.

The most important part of the streams paradigm is to structure your computation as a sequence of transformations where the result of each stage is as close as possible to a pure function of the result of the previous stage. A pure function is one whose result depends only on its input: it does not depend on any mutable state, nor does it update any state. In order to achieve this, any function objects that you pass into stream operations, both intermediate and terminal, should be free of side-effects.

The essence of programming stream pipelines is side-effect-free function objects. This applies to all of the many function objects passed to streams and related objects. The terminal operation forEach should only be used to report the result of a computation performed by a stream, not to perform the computation. In order to use streams properly, you have to know about collectors. The most important collector factories are `toList`, `toSet, toMap, groupingBy`, and `joining`.

## Item 47: Prefer Collection to Stream as a return type

Many methods return sequences of elements. Prior to Java 8, the obvious return types for such methods were the collection interfaces Collection, Set, and List; Iterable; and the array types. Usually, it was easy to decide which of these types to return. The norm was a collection interface. If the method existed solely to enable for-each loops or the returned sequence couldn’t be made to implement some Collection method (typically, contains(Object)), the Iterable interface was used. If the returned elements were primitive values or there were stringent performance requirements, arrays were used. In Java 8, streams were added to the platform, substantially complicating the task of choosing the appropriate return type for a sequence-returning method.

When writing a method that returns a sequence of elements, remember that some of your users may want to process them as a stream while others may want to iterate over them. Try to accommodate both groups. If it’s feasible to return a collection, do so. If you already have the elements in a collection or the number of elements in the sequence is small enough to justify creating a new one, return a standard collection such as ArrayList. Otherwise, consider implementing a custom collection as we did for the power set. If it isn’t feasible to return a collection, return a stream or iterable, whichever seems more natural. If, in a future Java release, the Stream interface declaration is modified to extend Iterable, then you should feel free to return streams because they will allow for both stream processing and iteration.

## Item 48: Use caution when making streams parallel

Do not even attempt to parallelize a stream pipeline unless you have good reason to believe that it will preserve the correctness of the computation and increase its speed. The cost of inappropriately parallelizing a stream can be a program failure or performance disaster. If you believe that parallelism may be justified, ensure that your code remains correct when run in parallel, and do careful performance measurements under realistic conditions. If your code remains correct and these experiments bear out your suspicion of increased performance, then and only then parallelize the stream in production code.
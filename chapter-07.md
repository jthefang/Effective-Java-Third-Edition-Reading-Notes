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

Now that Java has lambdas, best practices for writing APIs have changed considerably. For example, the Template Method pattern, wherein a subclass overrides a primitive method to specialize the behavior of its superclass, is far less attractive. The modern alternative is to provide a static factory or constructor that accepts a function object to achieve the same effect. More generally, __you’ll be writing more constructors and methods that take function objects as parameters__. Choosing the right functional parameter type demands care.

- The `java.util.function` package provides a large collection of standard functional interfaces for your use. If one of the standard functional interfaces does the job, you should generally use it in preference to a purpose-built functional interface. This will make your API easier to learn, by reducing its conceptual surface area, and will provide significant interoperability benefits, as many of the standard functional interfaces provide useful default methods.

There are forty-three interfaces in java.util.Function. You can’t be
expected to remember them all, but if you remember six basic interfaces, you can derive the rest when you need them. The basic interfaces operate on object reference types. 

1. The `Operator` interfaces represent functions whose result and argument types are the same. 
2. The `Predicate` interface represents a function that takes an argument and returns a boolean. 
3. The `Function` interface represents a function whose argument and return types differ. 
4. The `Supplier` interface represents a function that takes no arguments and returns (or “supplies”) a value. 
5. Finally, `Consumer` represents a function that takes an argument and returns nothing, essentially consuming its argument. 

The six basic functional interfaces are summarized below:

| Interface         | Function Signature  | Example             |
|-------------------|---------------------|---------------------|
| UnaryOperator<T>  | T apply(T t)        | String::toLowerCase |
| BinaryOperator<T> | T apply(T t1, T t2) | BigInteger::add     |
| Predicate<T>      | boolean test(T t)   | Collection::isEmpty |
| Function<T,R>     | R apply(T t)        | Arrays::asList      |
| Supplier<T>       | T get()             | Instant::now        |
| Consumer<T>       | void accept(T t)    | System.out::println |

There are also three variants of each of the six basic interfaces to operate on the primitive types `int`, `long`, and `double`. Their names are derived from the basic interfaces by prefixing them with a primitive type. So, for example, a predicate that takes an `int` is an `IntPredicate`, and a binary operator that takes two `long` values and returns a `long` is a `LongBinaryOperator`. None of these variant types is parameterized except for the `Function` variants, which are parameterized by return type. For example, `LongFunction<int[]>` takes a long and returns an `int[]`.

- Most of the standard functional interfaces exist only to provide support for primitive types. Don’t be tempted to use basic functional interfaces with boxed primitives instead of primitive functional interfaces. While it works, it violates the advice of Item 61, “prefer primitive types to boxed primitives.” The performance consequences of using boxed primitives for bulk operations can be deadly.

- You should seriously consider writing a purpose-built functional interface in preference to using a standard one if you need a functional interface that shares one or more of the following characteristics with `Comparator`:
    - It will be commonly used and could benefit from a descriptive name.
    - It has a strong contract associated with it.
    - It would benefit from custom default methods.

If you elect to write your own functional interface, remember that it’s an interface and hence should be designed with great care (Item 21).

- __Always annotate your functional interfaces with the @FunctionalInterface annotation__

Now that Java has lambdas, it is imperative that you design your APIs with lambdas in mind. Accept functional interface types on input and return them on output. It is generally best to use the standard interfaces provided in java.util.function.Function, but keep your eyes open for the relatively rare cases where you would be better off writing your own functional interface.

## Item 45: Use streams judiciously

The streams API was added in Java 8 to ease the task of performing bulk operations, sequentially or in parallel. This API provides two key abstractions: 1) the *stream*, which represents a finite or infinite sequence of data elements, and 2) the *stream pipeline*, which represents a multistage computation on these elements. The elements in a stream can come from anywhere. Common sources include collections, arrays, files, regular expression pattern matchers, pseudorandom number generators, and other streams. The data elements in a stream can be object references or primitive values. Three primitive types are supported: `int`, `long`, and `double`.

A stream pipeline consists of a source stream followed by zero or more intermediate operations and one terminal operation. Each intermediate operation transforms the stream in some way, such as mapping each element to a function of that element or filtering out all elements that do not satisfy some condition. Intermediate operations all transform one stream into another, whose element type may be the same as the input stream or different from it. The terminal operation performs a final computation on the stream resulting from the last intermediate operation, such as storing its elements into a collection, returning a certain element, or printing all of its elements.

- Stream pipelines are evaluated lazily: evaluation doesn’t start until the terminal operation is invoked, and data elements that aren’t required in order to complete the terminal operation are never computed. This lazy evaluation is what makes it possible to work with infinite streams. Note that a stream pipeline without a terminal operation is a silent no-op, so don’t forget to include one.

- The streams API is fluent: it is designed to allow all of the calls that comprise a pipeline to be chained into a single expression. In fact, multiple pipelines can be chained together into a single expression.

- By default, stream pipelines run sequentially. Making a pipeline execute in parallel is as simple as invoking the `parallel` method on any stream in the pipeline, but it is seldom appropriate to do so (Item 48).

- The streams API is sufficiently versatile that practically any computation can be performed using streams, but just because you can doesn’t mean you should. When used appropriately, streams can make programs shorter and clearer; when used inappropriately, they can make programs difficult to read and maintain. There are no hard and fast rules for when to use streams, but there are heuristics.
    - __Overusing streams makes programs hard to read and maintain__

Example:
```java
// Prints all large anagram groups in a dictionary iteratively
public class Anagrams {
    public static void main(String[] args) throws IOException {
        File dictionary = new File(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);

        Map<String, Set<String>> groups = new HashMap<>();
        try (Scanner s = new Scanner(dictionary)) {
            while (s.hasNext()) {
                String word = s.next();
                groups.computeIfAbsent(alphabetize(word),
                    (unused) -> new TreeSet<>()).add(word);
            }
        }

        for (Set<String> group : groups.values())
            if (group.size() >= minGroupSize)
                System.out.println(group.size() + ": " + group);
    }
        
    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}

// Overuse of streams - don't do this!
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        
        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(
                groupingBy(word -> word.chars().sorted()
                    .collect(StringBuilder::new,
                    (sb, c) -> sb.append((char) c),
                    StringBuilder::append).toString()))
                .values().stream()
                .filter(group -> group.size() >= minGroupSize)
                .map(group -> group.size() + ": " + group)
                .forEach(System.out::println);
        }
    }
}

// Tasteful use of streams enhances clarity and conciseness
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(groupingBy(word -> alphabetize(word)))
                .values().stream()
                .filter(group -> group.size() >= minGroupSize)
                .forEach(g -> System.out.println(g.size() + ": " + g));
        }
    }
    // alphabetize method is the same as in original version
}
```

Even if you have little previous exposure to streams, this program is not hard to understand. It opens the dictionary file in a try-with-resources block, obtaining a stream consisting of all the lines in the file. The `stream` variable is named `words` to suggest that each element in the stream is a word. The pipeline on this stream has no intermediate operations; its terminal operation collects all the words into a map that groups the words by their alphabetized form (Item 46). This is exactly the same map that was constructed in both previous versions of the program. Then a new `Stream<List<String>>` is opened on the `values()` view of the map. The elements in this stream are, of course, the anagram groups. The stream is filtered so that all of the groups whose size is less than `minGroupSize` are ignored, and finally, the remaining groups are printed by the terminal operation `forEach`.

- __In the absence of explicit types, careful naming of lambda parameters is essential to the readability of stream pipelines__

- Using helper methods is even more important for readability in stream pipelines than in iterative code because pipelines lack explicit type information and named temporary variables (e.g. `alphabetize` above).

- Refrain from using streams to process char values (it returns them in `int` form)

- Using streams instead of iterative loops:
    - From a code block, you can read or modify any local variable in scope; from a lambda, you can only read final or effectively final variables, and you can’t modify any local variables.
    - From a code block, you can `return` from the enclosing method, `break` or `continue` an enclosing loop, or throw any checked exception that this method is declared to throw; from a lambda you can do none of these things.
    - However, streams are useful to
        - Uniformly transform sequences of elements
        - Filter sequences of elements
        - Combine sequences of elements using a single operation (for example to add them, concatenate them, or compute their minimum)
        - Accumulate sequences of elements into a collection, perhaps grouping them by some common attribute
        - Search a sequence of elements for an element satisfying some criterion

Some tasks are best accomplished with streams, and others with iteration. Many tasks are best accomplished by combining the two approaches. There are no hard and fast rules for choosing which approach to use for a task, but there are some useful heuristics. In many cases, it will be clear which approach to use; in some cases, it won’t. __If you’re not sure whether a task is better served by streams or iteration, try both and see which works better__.

## Item 46: Prefer side-effect-free functions in streams

Streams isn’t just an API, it’s a paradigm based on functional programming. In order to obtain the expressiveness, speed, and in some cases parallelizability that streams have to offer, you have to adopt the paradigm as well as the API.

The most important part of the streams paradigm is to structure your computation as a sequence of transformations where the result of each stage is as close as possible to a pure function of the result of the previous stage. A pure function is one whose result depends only on its input: it does not depend on any mutable state, nor does it update any state. In order to achieve this, any function objects that you pass into stream operations, both intermediate and terminal, should be free of side-effects.

Example:

```java
// Uses the streams API but not the paradigm--Don't do this!
Map<String, Long> freq = new HashMap<>();
    try (Stream<String> words = new Scanner(file).tokens()) {
        words.forEach(word -> {
        freq.merge(word.toLowerCase(), 1L, Long::sum);
    });
}

// Proper use of streams to initialize a frequency table
Map<String, Long> freq;
try (Stream<String> words = new Scanner(file).tokens()) {
    freq = words
        .collect(groupingBy(String::toLowerCase, counting()));
}
```

- the `forEach` operation is among the least powerful of the terminal operations and the least stream-friendly. It’s explicitly iterative, and hence not amenable to parallelization. The `forEach` operation should be used only to report the result of a stream computation, not to perform the computation. Occasionally, it makes sense to use `forEach` for some other purpose, such as adding the results of a stream computation to a preexisting collection.

- The improved code uses a collector, which is a new concept that you have to learn in order to use streams. Think of a collector as an opaque object that encapsulates a *reduction strategy*. In this context, reduction means combining the elements of a stream into a single object. The object produced by a collector is typically a collection (which accounts for the name collector).

- The collectors for gathering the elements of a stream into a true `Collection` are straightforward. There are three such collectors: `toList()`, `toSet()`, and `toCollection(collectionFactory)`. They return, respectively, a set, a list, and a programmer-specified collection type. Armed with this knowledge, we can write a stream pipeline to extract a top-ten list from our frequency table.

```java
// Pipeline to get a top-ten list of words from a frequency table
List<String> topTen = freq.keySet().stream()
    .sorted(comparing(freq::get).reversed())
    .limit(10)
    .collect(toList());
```

Note that we haven’t qualified the toList method with its class, Collectors. It is customary and wise to __statically import all members of `Collectors` because it makes stream pipelines more readable__.

The only tricky part of this code is the comparator that we pass to sorted, `comparing(freq::get).reversed()`. The comparing method is a comparator construction method (Item 14) that takes a key extraction function. The function takes a word, and the “extraction” is actually a table lookup: the bound method reference `freq::get` looks up the word in the frequency table and returns the number of times the word appears in the file. Finally, we call reversed on the comparator, so we’re sorting the words from most frequent to least frequent. Then it’s a simple matter to limit the stream to ten words and collect them into a list.

- Many other functions in `Collectors` that let you collect streams into maps, where each stream element is associated with a key and a vlue, and multiple stream elements can be associated with the same key

The essence of programming stream pipelines is side-effect-free function objects. This applies to all of the many function objects passed to streams and related objects. The terminal operation forEach should only be used to report the result of a computation performed by a stream, not to perform the computation. In order to use streams properly, you have to know about collectors. The most important collector factories are `toList`, `toSet, toMap, groupingBy`, and `joining`.

## Item 47: Prefer Collection to Stream as a return type

Many methods return sequences of elements. Prior to Java 8, the obvious return types for such methods were the collection interfaces `Collection`, `Set`, and `List`; `Iterable`; and the array types. Usually, it was easy to decide which of these types to return. The norm was a collection interface. If the method existed solely to enable for-each loops or the returned sequence couldn’t be made to implement some `Collection` method (typically, `contains(Object)`), the `Iterable` interface was used. If the returned elements were primitive values or there were stringent performance requirements, arrays were used. In Java 8, streams were added to the platform, substantially complicating the task of choosing the appropriate return type for a sequence-returning method.

- It is difficult to for-each loop over a stream. You can use an adapter method. The JDK does not provide such a method, but it’s easy to write one, using the same technique used in-line in the snippets above. Note that no cast is necessary in the adapter method because Java’s type inference works properly in this context:

```java
// Adapter from Stream<E> to Iterable<E>
public static <E> Iterable<E> iterableOf(Stream<E> stream) {
    return stream::iterator;
}
```

With this adapter, you can iterate over any stream with a for-each statement:

```java
for (ProcessHandle p : iterableOf(ProcessHandle.allProcesses())) {
    // Process the process
}
```

- Note that the stream versions of the `Anagrams` program in Item 34 use the `Files.lines` method to read the dictionary, while the iterative version uses a scanner. The `Files.lines` method is superior to a scanner, which silently swallows any exceptions encountered while reading the file. Ideally, we would have used `Files.lines` in the iterative version too. This is the sort of compromise that programmers will make if an API provides only stream access to a sequence and they want to iterate over the sequence with a for-each statement.

- Conversely, a programmer who wants to process a sequence using a stream pipeline will be justifiably upset by an API that provides only an Iterable. Again the JDK does not provide an adapter, but it’s easy enough to write one:

```java
// Adapter from Iterable<E> to Stream<E>
public static <E> Stream<E> streamOf(Iterable<E> iterable) {
    return StreamSupport.stream(iterable.spliterator(), false);
}
```

- The `Collection` interface is a subtype of `Iterable` and has a stream method, so it provides for both iteration and stream access. Therefore, __`Collection` or an appropriate subtype is generally the best return type for a public, sequence-returning method__. Arrays also provide for easy iteration and stream access with the `Arrays.asList` and `Stream.of` methods. If the sequence you’re returning is small enough to fit easily in memory, you’re probably best off returning one of the standard collection implementations, such as `ArrayList` or `HashSet`. But do not store a large sequence in memory just to return it as a collection.

When writing a method that returns a sequence of elements, remember that some of your users may want to process them as a stream while others may want to iterate over them. Try to accommodate both groups. If it’s feasible to return a collection, do so. If you already have the elements in a collection or the number of elements in the sequence is small enough to justify creating a new one, return a standard collection such as ArrayList. Otherwise, consider implementing a custom collection as we did for the power set. If it isn’t feasible to return a collection, return a stream or iterable, whichever seems more natural. If, in a future Java release, the Stream interface declaration is modified to extend Iterable, then you should feel free to return streams because they will allow for both stream processing and iteration.

## Item 48: Use caution when making streams parallel

Among mainstream languages, Java has always been at the forefront of providing facilities to ease the task of concurrent programming. When Java was released in 1996, it had built-in support for threads, with synchronization and wait/notify. Java 5 introduced the java.util.concurrent library, with concurrent collections and the executor framework. Java 7 introduced the fork-join package, a high-performance framework for parallel decomposition. Java 8 introduced streams, which can be parallelized with a single call to the parallel method. Writing concurrent programs in Java keeps getting easier, but writing concurrent programs that are correct and fast is as difficult as it ever was. Safety and liveness violations are a fact of life in concurrent programming, and parallel stream pipelines are no exception.

- Parallelizing a pipeline is unlikely to increase its performance if the source is from `Stream`.iterate, or the intermediate operation `limit` is used

- Performance gains from parallelism are best on streams over `ArrayList`, `HashMap`, `HashSet`, and `ConcurrentHashMap` instances; arrays; `int` ranges; and `long` ranges. What these data structures have in common is that they can all be accurately and cheaply split into subranges of any desired sizes, which makes it easy to divide work among parallel threads. The abstraction used by the streams library to perform this task is the *spliterator*, which is returned by the `spliterator` method on `Stream` and `Iterable`.
    - Another important factor that all of these data structures have in common is that they provide good-to-excellent locality of reference when processed sequentially: sequential element references are stored together in memory. The objects referred to by those references may not be close to one another in memory, which reduces locality-of-reference. Locality-of-reference turns out to be critically important for parallelizing bulk operations: without it, threads spend much of their time idle, waiting for data to be transferred from memory into the processor’s cache. The data structures with the best locality of reference are primitive arrays because the data itself is stored contiguously in memory.
    - The nature of a stream pipeline’s terminal operation also affects the effectiveness of parallel execution. If a significant amount of work is done in the terminal operation compared to the overall work of the pipeline and that operation is inherently sequential, then parallelizing the pipeline will have limited effectiveness. The best terminal operations for parallelism are reductions, where all of the elements emerging from the pipeline are combined using one of Stream’s `reduce` methods, or prepackaged reductions such as `min`, `max`, `count`, and `sum`. The shortcircuiting operations `anyMatch`, `allMatch`, and `noneMatch` are also amenable to parallelism. The operations performed by Stream’s `collect` method, which are known as mutable reductions, are not good candidates for parallelism because the overhead of combining collections is costly.

- It’s important to remember that parallelizing a stream is strictly a performance optimization. As is the case for any optimization, you must test the performance before and after the change to ensure that it is worth doing (Item 67). Ideally, you should perform the test in a realistic system setting. Normally, all parallel stream pipelines in a program run in a common fork-join pool. A single misbehaving pipeline can harm the performance of others in unrelated parts of the system

- Under the right circumstances, it is possible to achieve near-linear speedup in the number of processor cores simply by adding a parallel call to a stream pipeline. Certain domains, such as machine learning and data processing, are particularly amenable to these speedups.

Do not even attempt to parallelize a stream pipeline unless you have good reason to believe that it will preserve the correctness of the computation and increase its speed. The cost of inappropriately parallelizing a stream can be a program failure or performance disaster. If you believe that parallelism may be justified, ensure that your code remains correct when run in parallel, and do careful performance measurements under realistic conditions. If your code remains correct and these experiments bear out your suspicion of increased performance, then and only then parallelize the stream in production code.
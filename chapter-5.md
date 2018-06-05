# Chapter 5. Generics

Since Java 5, generics have been a part of the language. Before generics, you had to cast every object you read from a collection. If someone accidentally inserted an object of the wrong type, casts could fail at runtime. With generics, you tell the compiler what types of objects are permitted in each collection. The compiler inserts casts for you automatically and tells you at compile time if you try to insert an object of the wrong type. This results in programs that are both safer and clearer, but these benefits, which are not limited to collections, come at a price. This chapter tells you how to maximize the benefits and minimize the complications.

## Item 26: Don’t use raw types in new code

First, a few terms. A class or interface whose declaration has one or more type parameters is a generic class or interface. For example, the
`List` interface has a single type parameter, E, representing its element type. The full name of the interface is `List<E>` (read "list of E"), but people often call it List for short. Generic classes and interfaces are collectively known as generic types, whereas the undecorated type without any type parameters is a raw type (e.g. `List` is a raw type, `List<E>` is a generic type).

- While you shouldn’t use raw types such as `List`, it is fine to use types that are parameterized to allow insertion of arbitrary objects, such as `List<Object>`.

- You might be tempted to use a raw type for a collection whose element type is unknown and doesn’t matter. For example, suppose you want to write a method that takes two sets and returns the number of elements they have in common. Here’s how you might write such a method if you were new to generics:

```java
// Use of raw type for unknown element type - don't do this!
static int numElementsInCommon(Set s1, Set s2) {
  int result = 0;
  for (Object o1 : s1)
    if (s2.contains(o1))
      result++;
  return result;
}
```

This method works but it uses raw types, which are dangerous. The safe alternative is to use unbounded wildcard types. If you want to use a generic type but you don’t know or care what the actual type parameter is, you can use a question mark instead. For example, the unbounded wildcard type for the generic type Set<E> is Set<?> (read “set of some type”). It is the most general parameterized Set type, capable of holding any set. Here is how the numElementsInCommon declaration looks with unbounded wildcard types:

```java
// Uses unbounded wildcard type - typesafe and flexible
static int numElementsInCommon(Set<?> s1, Set<?> s2) { ... }
```

- There are a few minor exceptions to the rule that you should not use raw
types. 
    1. You must use raw types in class literals. In other words, `List.class`, `String[].class`, and `int.class` are all legal, but `List<String>.class` and `List<?>.class` are not.
    2. The instanceof operator. Because generic type information is erased at runtime, it is illegal to use the instanceof operator on parameterized types other than unbounded wildcard types. The use of unbounded wildcard types in place of raw types does not affect the behavior of the instanceof operator in any way. In this case, the angle brackets and question marks are just noise. This is the preferred way to use the instanceof operator with generic types:

    ```java
    // Legitimate use of raw type - instanceof operator
    if (o instanceof Set) { // Raw type
        Set<?> s = (Set<?>) o; // Wildcard type
        ...
    }
    ```

Using raw types can lead to exceptions at runtime, so don’t use them in new code. They are provided only for compatibility and interoperability with legacy code that predates the introduction of generics. As a quick review, `Set<Object>` is a parameterized type representing a set that can contain objects of any type, `Set<?>` is a wildcard type representing a set that can contain only objects of some unknown type, and `Set` is a raw type, which opts out of the generic type system. The first two are safe and the last is not.

For quick reference, the terms introduced in this item (and a few introduced later in this chapter) are summarized in 
the following table:
!===================================================================================!
Term                                Example                         Item

Parameterized type                  List<String>                    Item 26

Actual type parameter               String                          Item 26

Generic type                        List<E>                         Items 26, 29

Formal type parameter               E                               Item 26

Unbounded wildcard type             List<?>                         Item 26

Raw type                            List                            Item 26

Bounded type parameter              <E extends Number>              Item 29

Recursive type bound                <T extends Comparable<T>>       Item 30

Bounded wildcard type               List<? extends Number>          Item 31

Generic method                      static <E> List<E> asList(E[] a)Item 30

Type token                          String.class                    Item 33
!===================================================================================!

## Item 27: Eliminate unchecked warnings

Unchecked warnings are important. Don’t ignore them. Every unchecked warning represents the potential for a `ClassCastException` at runtime. Do your best to eliminate these warnings. If you can’t eliminate an unchecked warning and you can prove that the code that provoked it is typesafe, suppress the warning with an `@SuppressWarnings("unchecked")` annotation __in the narrowest possible scope__. Record the rationale for your decision to suppress the warning in a comment. If you ignore unchecked warnings that you know to be safe (instead of suppressing them), you won’t notice when a new warning crops up that represents a real problem. The new warning will get lost amidst all the false alarms that you didn’t silence.

## Item 28: Prefer lists to arrays

Arrays and generics have very different type rules. Arrays are covariant and reified; generics are invariant and erased. As a consequence, arrays provide runtime type safety but not compile-time type safety and vice versa for generics. Generally speaking, arrays and generics don’t mix well. If you find yourself mixing them and getting compile-time errors or warnings, your first impulse should be to replace the arrays with lists.

## Item 29: Favor generic types

Generic types are safer and easier to use than types that require casts in client code. When you design new types, make sure that they can be used without such casts. This will often mean making the types generic. Generify your existing types as time permits. This will make life easier for new users of these types without breaking existing clients.

- You can't make arrays generic, but you can cast and if you're sure it's safe, suppress the warnings.
```java
// Initial attempt to generify Stack - won't compile!
public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    /* Won't compile
    public Stack() {
        elements = new E[DEFAULT_INITIAL_CAPACITY];
    }*/

    // The elements array will contain only E instances from push(E).
    // This is sufficient to ensure type safety, but the runtime
    // type of the array won't be E[]; it will always be Object[]!
    @SuppressWarnings("unchecked")
    public Stack() {
        elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(E e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public E pop() {
        if (size == 0)
            throw new EmptyStackException();
        E result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }
    ... // no changes in isEmpty or ensureCapacity
}
```

- There are some generic types that restrict the permissible values of their type parameters. For example, consider `java.util.concurrent.DelayQueue`, whose declaration looks like this:

```java
class DelayQueue<E extends Delayed> implements BlockingQueue<E>
```

The type parameter list (`<E extends Delayed>`) requires that the actual type parameter E be a subtype of `java.util.concurrent.Delayed`. This allows the `DelayQueue` implementation and its clients to take advantage of `Delayed` methods on the elements of a `DelayQueue`, without the need for explicit casting or the risk of a `ClassCastException`. The type parameter `E` is known as a bounded type parameter. Note that the subtype relation is defined so that every type is a subtype of itself [JLS, 4.10], so it is legal to create a `DelayQueue<Delayed>`.

## Item 30: Favor generic methods

Generic methods, like generic types, are safer and easier to use than methods that require their clients to cast input parameters and return values. Like types, you should make sure that your new methods can be used without casts, which will often mean making them generic. And like types, you should generify your existing methods to make life easier for new users without breaking existing clients.

- The type parameter list, which declares the type parameters, goes between a method’s modifiers and its return type. In this example, the type parameter list is <E>, and the return type is Set<E>. The naming conventions for type parameters are the same for generic methods and generic types (Items 29, 68):

```java
// Generic method
public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
    Set<E> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
}
```

- Many methods take a collection of elements implementing `Comparable` to sort it, search within it, calculate its minimum or maximum, and the like. To do these things, it is required that every element in the collection be comparable to every other element in it, in other words, that the elements of the list be mutually comparable. Here is how to express that constraint:

```java
// Using a recursive type bound to express mutual comparability
// Returns max value in a collection
public static <E extends Comparable<E>> E max(Collection<E> c) {
    if (c.isEmpty())
        throw new IllegalArgumentException("Empty collection");

    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
            result = Objects.requireNonNull(e);

    return result;
}
```

The type bound `<E extends Comparable<E>>` may be read as “any type `E` that can be compared to itself,” which corresponds more or less precisely to the notion of mutual comparability.

## Item 31: Use bounded wildcards to increase API flexibility

As noted in Item 28, parameterized types are invariant. In other words, for any two distinct types `Type1` and `Type2`, `List<Type1>` is neither a subtype nor a supertype of `List<Type2>`. Although it is counterintuitive that `List<String>` is not a subtype of `List<Object>`, it really does make sense. You can put any object into a `List<Object>`, but you can put only strings into a `List<String>`. Since a `List<String>` can’t do everything a `List<Object>` can, it isn’t a subtype (by the Liskov substitution principal, Item 10).

- Suppose we want to add a method to a custom `Stack` class that takes a sequence of elements and pushes them all onto the stack. Here’s a first attempt:

```java
// pushAll method without wildcard type - deficient!
public void pushAll(Iterable<E> src) {
    for (E e : src)
        push(e);
}
```

This method compiles cleanly, but it isn’t entirely satisfactory. If the element type of the Iterable src exactly matches that of the stack, it works fine. But suppose you have a `Stack<Number>` and you invoke `push(intVal)`, where `intVal` is of type `Integer`. This works because `Integer` is a subtype of `Number`. So logically, it seems that this should work, too:

```java
Stack<Number> numberStack = new Stack<>();
Iterable<Integer> integers = ... ;
numberStack.pushAll(integers);
```

If you try it, however, you’ll get this error message because parameterized types are invariant:

```java
StackTest.java:7: error: incompatible types: Iterable<Integer> cannot be converted to Iterable<Number> numberStack.pushAll(integers);
```

The language provides a special kind of parameterized type call a bounded wildcard type to deal with situations like this. The type of the input parameter to `pushAll` should not be "`Iterable` of `E`" but "`Iterable` of some subtype of `E`," and there is a wildcard type that means precisely that: `Iterable<? extends E>`. (The use of the keyword extends is slightly misleading: recall from Item 29 that subtype is defined so that every type is a subtype of itself, even though it does not extend itself.) Let’s modify `pushAll` to use this type:

```java
// Wildcard type for a parameter that serves as an E producer
public void pushAll(Iterable<? extends E> src) {
for (E e : src)
push(e);
}
```

With this change, not only does `Stack` compile cleanly, but so does the client code that wouldn’t compile with the original `pushAll` declaration. Because `Stack` and its client compile cleanly, you know that everything is typesafe.

- Now suppose you want to write a `popAll` method to go with `pushAll`. The `popAll` method pops each element off the stack and adds the elements to the given collection. Here’s how a first attempt at writing the `popAll` method might look:

```java
// popAll method without wildcard type - deficient!
public void popAll(Collection<E> dst) {
    while (!isEmpty())
        dst.add(pop());
}
```

Again, this compiles cleanly and works fine if the element type of the destination collection exactly matches that of the stack. But again, it isn’t entirely satisfactory. Suppose you have a `Stack<Number>` and variable of type `Object`. If you pop an element from the stack and store it in the variable, it compiles and runs without error. So shouldn’t you be able to do this, too?

```java
Stack<Number> numberStack = new Stack<Number>();
Collection<Object> objects = ... ;
numberStack.popAll(objects);
```

If you try to compile this client code against the version of `popAll` shown earlier, you’ll get an error very similar to the one that we got with our first version of `pushAll`: `Collection<Object>` is not a subtype of `Collection<Number>`. Once again, wildcard types provide a way out. The type of the input parameter to `popAll` should not be "collection of `E`" but "collection of some supertype of `E`" (where supertype is defined such that `E` is a supertype of itself JLS, 4.10]). Again, there is a wildcard type that means precisely that: `Collection<? super E>`. Let’s modify popAll to use it:

```java
// Wildcard type for parameter that serves as an E consumer
public void popAll(Collection<? super E> dst) {
    while (!isEmpty())
    dst.add(pop());
}
```

With this change, both Stack and the client code compile cleanly.

The lesson is clear. __For maximum flexibility, use wildcard types on input parameters that represent producers or consumers__. If an input parameter is both a producer and a consumer, then wildcard types will do you no good: you need an exact type match, which is what you get without any wildcards.

- __Do not use bounded wildcard types as return types__. Rather than providing additional flexibility for your users, it would force them to use wildcard types in client code. __If the user of a class has to think about wildcard types, there is probably something wrong with its API__.

- __Comparables are always consumers, so you should generally use Comparable<? super T> in preference to Comparable<T>__. The same is true of comparators; therefore, you should generally use `Comparator<? super T>` in preference to `Comparator<T>`. More generally, the wildcard is required to support types that do not implement `Comparable` (or `Comparator`) directly but extend a type that does.

- __If a type parameter appears only once in a method declaration, replace it with a wildcard__. You can't put any value except `null` into a `List<?>`. Instead write a private helper method to capture the wildcard type. The helper method must be a generic method in order to capture the type. Here’s how it looks:

```java
public static void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);
}

// Private helper method for wildcard capture
private static <E> void swapHelper(List<E> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));
}
```

Using wildcard types in your APIs, while tricky, makes the APIs far more flexible. If you write a library that will be widely used, the proper use of wildcard types should be considered mandatory. Remember the basic rule: producer-extends (e.g. `pushAll` collection parameter was a producer of `E` instances for the stack), consumer-super (PECS) (e.g. `popAll`'s collection parameter was a consumer of `E` instances on the stack). And remember that all comparables and comparators are consumers.

## Item 32: Combine generics and varargs judiciously 

The purpose of varargs is to allow clients to pass a variable number of arguments to a method, but it is a leaky abstraction: when you invoke a varargs method, an array is created to hold the varargs parameters; that array, which should be an implementation detail, is visible. As a consequence, you get confusing compiler warnings when varargs parameters have generic or parameterized types. Heap pollution occurs when a variable of a parameterized type refers to an object that is not of that type. It can cause the compiler's automatically generated casts to fail, violating the fundamental guarantee of the generic type system. Thus, __it is unsafe to store a value in generic varargs array parameter__.

- Methods with varargs parameters of generic or parameterized types can be very useful in practice, so the language designers opted to live with this inconsistency. In fact, the Java libraries export several such methods, including `Arrays.asList(T... a), Collections.addAll(Collection<? super T> c, T... elements)`, and `EnumSet.of(E first, E... rest)`. However, these library methods are typesafe.

- In Java 7, the `SafeVarargs` annotation was added to the platform, to allow the author of a method with a generic varargs parameter to suppress client warnings automatically. In essence, the `SafeVarargs ` annotation constitutes a promise by the author of a method that it is typesafe. In exchange for this promise, the compiler agrees not to warn the users of the method that calls may be unsafe.

It is critical that you do not annotate a method with `@SafeVarargs` unless it actually is safe. So what does it take to ensure this? Recall that a generic array is created when the method is invoked, to hold the varargs parameters. *If the method 1) doesn’t store anything into the array (which would overwrite the parameters) and 2) doesn’t allow a reference to the array to escape (which would enable untrusted code to access the array), then it’s safe. In other words, if the varargs parameter array is used only to transmit a variable number of arguments from the caller to the method—which is, after all, the purpose of varargs—then the method is safe*.

- __It is unsafe to give another method access to a generic varargs parameter array__, with two exceptions: it is safe to pass the array to another varargs method that is correctly annotated with `@SafeVarargs`, and it is safe to pass the array to a non-varargs method that merely computes some function of the contents of the array.

- An alternative to using the `SafeVarargs` annotation is to take the advice of Item 28 and replace the varargs parameter (which is an array in disguise) with a `List` parameter. __Lists were built for use with generics whereas arrays were not__. Here’s how this approach looks when applied to our `flatten` method. Note that only the parameter declaration has changed:

```java
// List as a typesafe alternative to a generic varargs parameter
static <T> List<T> flatten(List<List<? extends T>> lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists)
        result.addAll(list);
    return result;
}
```

This method can then be used in conjunction with the static factory method `List.of` to allow for a variable number of arguments. Note that this approach relies on the fact that the `List.of` declaration is annotated with `@SafeVarargs`:

```java
audience = flatten(List.of(friends, romans, countrymen));
```

The advantage of this approach is that the compiler can prove that the method is typesafe. You don’t have to vouch for its safety with a `SafeVarargs` annotation, and you don’t have worry that you might have erred in determining that it was safe. The main disadvantage is that the client code is a bit more verbose and may be a bit slower.

In summary, varargs and generics do not interact well because the varargs facility is a leaky abstraction built atop arrays, and arrays have different type rules from generics. Though generic varargs parameters are not typesafe, they are legal. If you choose to write a method with a generic (or parameterized) varargs parameter, first ensure that the method is typesafe, and then annotate it with @SafeVarargs so it is not unpleasant to use.

## Item 33: Consider type safe heterogeneous containers

The normal use of generics, exemplified by the collections APIs, restricts you to a fixed number of type parameters per container. You can get around this restriction by placing the type parameter on the key rather than the container. You can use `Class` objects as keys for such typesafe heterogeneous containers. A `Class` object used in this fashion is called a type token. You can also use a custom key type. For example, you could have a `DatabaseRow` type representing a database row (the container), and a generic type `Column<T>` as its key.

- Example: The API for the `Favorites` class is simple. It looks just like a simple map, except that the key is parameterized instead of the map. The client presents a `Class` object when setting and getting favorites. Here is the API:

```java
// Typesafe heterogeneous container pattern - API
public class Favorites {
    public <T> void putFavorite(Class<T> type, T instance);
    public <T> T getFavorite(Class<T> type);
}
```

Here is a sample program that exercises the `Favorites` class, storing, retrieving, and printing a favorite `String, Integer`, and `Class` instance:

```java
// Typesafe heterogeneous container pattern - client
public static void main(String[] args) {
    Favorites f = new Favorites();
    f.putFavorite(String.class, "Java");
    f.putFavorite(Integer.class, 0xcafebabe);
    f.putFavorite(Class.class, Favorites.class);
    String favoriteString = f.getFavorite(String.class);
    int favoriteInteger = f.getFavorite(Integer.class);
    Class<?> favoriteClass = f.getFavorite(Class.class);
    System.out.printf("%s %x %s%n", favoriteString,
        favoriteInteger, favoriteClass.getName());
}
```

As you would expect, this program prints `Java cafebabe Favorites`. Note, incidentally, that Java's printf method differs from C's in that you should use `%n` where you’d use `\n` in C. The `%n` generates the applicable platform-specific line separator, which is `\n` on many but not all platforms.

A Favorites instance is typesafe: it will never return an Integer when you ask it for a String. It is also heterogeneous: unlike an ordinary map, all the keys are of different types. Therefore, we call Favorites a typesafe heterogeneous container.

The implementation of `Favorites` in its entirety:

```java
// Typesafe heterogeneous container pattern - implementation
public class Favorites {
    private Map<Class<?>, Object> favorites = new HashMap<>();
    
    //Achieving runtime type safety with a dynamic cast
    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), type.cast(instance));
    }

    public <T> T getFavorite(Class<T> type) {
        return type.cast(favorites.get(type));
    }
}
```
    - Each Favorites instance is backed by a `private Map<Class<?>, Object>` called `favorites`. You might think that you couldn’t put anything into this `Map` because of the unbounded wildcard type, but the truth is quite the opposite. The thing to notice is that the wildcard type is nested: it’s not the type of the map that’s a wildcard type but the type of its key. This means that every key can have a different parameterized type: one can be `Class<String>`, the next `Class<Integer>`, and so on. That’s where the heterogeneity comes from.
    - The value type of the favorites `Map` is simply `Object`. In other words, the `Map` does not guarantee the type relationship between keys and values, which is that every value is of the type represented by its key. In  fact, Java’s type system is not powerful enough to express this. But we know that it’s true, and we take advantage of it when the time comes to retrieve a favorite.
    - The `putFavorite` implementation is trivial: it simply puts into favorites a mapping from the given `Class` object to the given favorite instance. As noted, this discards the “type linkage” between the key and the value; it loses the knowledge that the value is an instance of the key. But that’s OK, because the `getFavorites` method can and does reestablish this linkage.
    - The implementation of `getFavorite` is trickier. First, it gets from the favorites map the value corresponding to the given `Class` object. This is the correct object reference to return, but it has the wrong compiletime type: it is `Object` (the value type of the favorites map) and we need to return a `T`. So, the `getFavorite` implementation dynamically casts the object reference to the type represented by the `Class` object, using `Class`’s cast method.

    The cast method is the dynamic analogue of Java’s cast operator. It simply checks that its argument is an instance of the type represented by the `Class` object. If so, it returns the argument; otherwise it throws a `ClassCastException`. We know that the cast invocation in `getFavorite` won’t throw `ClassCastException`, assuming the client code compiled cleanly. That is to say, we know that the values in the favorites map always match the types of their keys.
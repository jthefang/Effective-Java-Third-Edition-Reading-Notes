# Chapter 4: Classes and Interfaces

Classes and interfaces lie at the heart of the Java programming language. They are its basic units of abstraction. The language provides many powerful elements that you can use to design classes and interfaces. This chapter contains guidelines to help you make the best use of these elements so that your classes and interfaces are usable, robust, and flexible.

## Item 15: Minimize the accessibility of classes and members

*Information hiding* or *encapsulation* is one of the fundamental tenets of software design. 

It is important for several reasons, stemming from the fact that it decouples the components of a system, allowing them to be independently developed, tested, optimized, used, understood and modified:
- development can be parallelized 
- eases maintenance
- eases debugging and performance tuning 
- allows for software reuse
- decrease the risk inherent in building large systems since individual components may prove successful even if the system does not

You should always reduce accessibility as much as possible, i.e. __make each class or member as inaccessible as possible__. 

If a package-private top-level class or interface is used by only one class, consider making the top-level class a private static nested class of the sole class that uses it (Item 24). This reduces its accessibility from all the classes in its package to the one class that uses it.

The four access levels:
- __private__ - The member is accessible only from the top-level class where it is declared.
- __package-private__ - The member is accessible from any class in the package where it is declared. Technically known as default access, this is the access level you get if no access modifier is specified (except for interface members, which are public by default).
- __protected__ - The member is accessible from subclasses of the class where it is declared (subject to a few restrictions [JLS, 6.6.2]) and from any class in the package where it is declared.
- __public__ - The member is accessible from anywhere.

After carefully designing a minimal public API, you should prevent any stray classes, interfaces, or members from becoming a part of the API. With the exception of public static final fields (i.e. constants, by convention all uppercase and underscores to separate words), public classes should have no public fields. Ensure that objects referenced by public static final fields are immutable ([Item 15](chapter-4.md#item-17-minimize-mutability)).

Note that a nonzero-length array is always mutable, so __it is wrong for a class to have a public static final array field, or an accessor that returns such a field__. If a class has such a field or accessor, clients will be able to modify the contents of the array. This is a frequent source of security holes:

```java
// Potential security hole!
public static final Thing[] VALUES = { ... };
```

Beware of the fact that some IDEs generate accessors that return references to private array fields, resulting in exactly this problem. There are two ways to fix the problem. You can make the public array private and add a public immutable list:

```java
private static final Thing[] PRIVATE_VALUES = { ... };
public static final List<Thing> VALUES =
    Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
```

Alternatively, you can make the array private and add a public method that
returns a copy of a private array:

```java
private static final Thing[] PRIVATE_VALUES = { ... };
public static final Thing[] values() {
    return PRIVATE_VALUES.clone();
}
```

## Item 16: In public classes, use accessor methods, not public fields

Public classes should never expose mutable fields. It is less harmful, though still questionable, for public classes to expose immutable fields. It is, however, sometimes desirable for package-private or private nested classes to expose fields, whether mutable or immutable, because this approach generates less visual clutter than the accessor-method approach, both in the class definition and in the client code that uses it.

## Item 17: Minimize mutability

An immutable class is simply a class whose instances cannot be modified (e.g. `String`, boxed primitive classes, `BigInteger` and `BigDecimal`). All of the information contained in each instance is provided when it is created and is fixed for the lifetime of the object. There are many good reasons for this: Immutable classes are easier to design, implement, and use than mutable classes. They are less prone to error and are more secure.

To make a class immutable:
1. Don’t provide any methods that modify the object’s state (known as mutators).
2. Ensure that the class can’t be extended (i.e. make the class `final`).
    1. There is another, more flexible alternative. Instead of making an immutable class final, you can make all of its constructors private or package-private and add public static factories in place of the public constructors (Item 1). To make this concrete, here’s how `Complex` would look if you took this approach:
    ```java
    // Immutable class with static factories instead of constructors
    public class Complex {
        private final double re;
        private final double im;
        private Complex(double re, double im) {
            this.re = re;
            this.im = im;
        }
        public static Complex valueOf(double re, double im) {
            return new Complex(re, im);
        }
        ... // Remainder unchanged
    }
    ```

    It is the most flexible because it allows the use of multiple package-private implementation classes. To its clients that reside outside its package, the immutable class is effectively final because it is impossible to extend a class that comes from another package and that lacks a public or protected constructor. This approach also makes it possible to tune the performance of the class in subsequent releases by improving the object-caching capabilities of the static factories.
3. Make all fields final.
4. Make all fields private.
5. Ensure exclusive access to any mutable components. If your class has any fields that refer to mutable objects, ensure that clients of the class cannot obtain references to these objects. Never initialize such a field to a client-provided object reference or return the field from an accessor. Make defensive copies (Item 50) in constructors, accessors, and readObject methods (Item 88).

- __Immutable objects are inherently thread-safe; they require no synchronization__. They cannot be corrupted by multiple threads accessing them concurrently. This is far and away the easiest approach to achieve thread safety. Since no thread can ever observe any effect of another thread on an immutable object, immutable objects can be shared freely. Immutable classes should therefore encourage clients to reuse existing instances wherever possible. One easy way to do this is to provide public static final constants for commonly used values. For example, the Complex class might provide these constants: 

```java
public static final Complex ZERO = new Complex(0, 0); 
public static final Complex ONE = new Complex(1, 0); 
public static final Complex I = new Complex(0, 1);
```

This approach can be taken one step further. An immutable class can provide static factories (Item 1) that cache frequently requested instances to avoid creating new instances when existing ones would do

- If you need to perform multistep operations on an immutable object that generates a new object at every step (e.g. string construction), your best bet is to provide a public mutable companion class (e.g. `StringBuilder`).

- __If a class cannot be made immutable, limit its mutability as much as possible__. Reducing the number of states in which an object can exist makes it easier to reason about the object and reduces the likelihood of errors. Therefore, make every field final unless there is a compelling reason to make it nonfinal. Combining the advice of this item with that of Item 15, your natural inclination should be to __declare every field private final unless there’s a good reason to do otherwise__.


## Item 18: Favor composition over inheritance

Inheritance is powerful, but it is problematic because it violates encapsulation, i.e. a subclass depends on the implementation details of its superclass for its proper function. Unless the super class was explicitly designed to be extended, the subclass may break with subsequent updates to the API. 

Instead of extending an existing class, give your new class a private field that references an instance of the existing class. This design is called *composition* because the existing class becomes a component of the new one. Each instance method in the new class invokes the corresponding method on the contained instance of the existing class and returns the results. This is known as *forwarding*, and the methods in the new class are known as forwarding methods. The resulting class will be rock solid, with no dependencies on the implementation details of the existing class. Even adding new methods to the existing class will have no impact on the new class. The new class is known as a *wrapper* class because each instance of it contains ("wraps") an instance of the existing class. This is also known as the *Decorator* pattern because the new class “decorates” the existing by adding functionality. Sometimes the combination of composition and forwarding is loosely referred to as delegation. Technically it’s not delegation unless the wrapper object passes itself to the wrapped object.

It is appropriate only when a genuine subtype ("is a") relationship exists between the subclass and the superclass. Even then, inheritance may lead to fragility if the subclass is in a different package from the superclass and the superclass is not designed for inheritance. To avoid this fragility, use composition and forwarding instead of inheritance, especially if an appropriate interface to implement a wrapper class exists. Not only are wrapper classes more robust than subclasses, they are also more powerful.

## Item 19: Design and document for inheritance or else prohibit it

- The class must document precisely the effects of overriding any method, i.e. __the class must document its self-use of overridable methods__. Try to eliminate the class's self-use of overridable methods entirely, if you plan for it to be subclassed (i.e. move the code to private helper methods). This way, overriding a method will never affect the behavior of any other method.
    -A method that invokes overridable methods contains a description of these invocations at the end of its documentation comment. The description is in a special section of the specification, labeled "Implementation Requirements", which is generated by the Javadoc tag @implSpec. To document a class so that it can be safely subclassed, you must describe implementation details that should otherwise be left unspecified. This tag should be enabled by default, but as of Java 9, the Javadoc utility still ignores it unless you pass the command line switch `-tag "apiNote:a:API Note:"`.
- Constructors must not invoke overridable methods, directly or indirectly. All subclasses run the super classes' constructor before their own, so any state changes in the overridden methods will cause `null` pointer exceptions since the subclass wasn't even initialized yet.
- If you implement `Cloneable` or `Serializable` in a class designed for inheritance, be aware that because the `clone` and `readObject` methods behave a lot like constructors, a similar restriction applies: __neither `clone` nor `readObject` may invoke an overridable method, directly or
indirectly__.
    - Finally, if you decide to implement `Serializable` in a class designed for inheritance and the class has a `readResolve` or `writeReplace` method, you must make the `readResolve` or `writeReplace` method protected rather than private. If these methods are private, they will be silently ignored by subclasses.
- Prohibit subclassing in classes that are not designed and documented to be safely subclassed by:
    1. Declaring the class final
    2. Make all constructors private or package-private (therefore unsubclassable) and add public static factories in place of the constructors

## Item 20: Prefer interfaces to abstract classes

The Java programming language provides two mechanisms for defining a type that permits multiple implementations: interfaces and abstract classes. Both mechanisms allow you to provide implementations for some instance methods (e.g. default methods for interfaces, regular methods for abstract classes). A major difference is that to implement the type defined by an abstract class, a class must be a subclass of the abstract class. Because Java permits only single inheritance, this restriction on abstract classes severely constrains their use as type definitions. Any class that defines all of the required methods and obeys the general contract is permitted to implement an interface, regardless of where the class resides in the class hierarchy. 

- Existing classes can easily be retrofiteed to implement a new interface. Not so much for extending an abstract class, since it implicates a class hierarchy that may not be appropriate for existing subclasses.

- Interfaces are ideal for defining mixins. Loosely speaking, a mixin is a type that a class can implement in addition to its "primary type," to declare that it provides some optional behavior. For example, `Comparable` is a mixin interface that allows a class to declare that its instances are ordered with respect to other mutually comparable objects. Such an interface is called a mixin because it allows the optional functionality to be “mixed in” to the type’s primary functionality. Abstract classes can’t be used to define mixins for the same reason that they can’t be retrofitted onto existing classes: a class cannot have more than one parent, and there is no reasonable place in the class hierarchy to insert a mixin.

- Interfaces allow for the construction of nonhierarchical type frameworks

- Interfaces are not permitted to contain instance fields or nonpublic static members (with the exception of private static methods). Finally, you can’t add default methods to an interface that you don’t control

- You can combine the advantages of interfaces and abstract classes by providing an abstract *skeletal implementation* class to go with an interface.The interface defines the type, perhaps providing some default methods, while the skeletal implementation class implements the remaining non-primitive interface methods atop the primitive interface methods. Extending a skeletal implementation takes most of the work out of implementing an interface. This is the Template Method pattern.
    - By convention, skeletal implementation classes are called Abstract*Interface*, where *Interface* is the name of the interface they implement. For example, the Collections Framework provides a skeletal implementation to go along with each main collection interface: `AbstractCollection`, `AbstractSet`, `AbstractList`, and `AbstractMap`.
    - Furthermore, the skeletal implementation can still aid the implementor’s task. The class implementing the interface can forward invocations of interface methods to a contained instance of a private inner class that extends the skeletal implementation. This technique, known as simulated multiple inheritance, is closely related to the wrapper class idiom discussed in Item 18. It provides many of the benefits of multiple inheritance, while avoiding the pitfalls.
    - __Good documentation is absolutely essential in a skeletal implementation__

Example of skeleton class implementation:

```java
// Skeletal implementation class
public abstract class AbstractMapEntry<K,V>
    implements Map.Entry<K,V> {
    // Entries in a modifiable map must override this method
    @Override public V setValue(V value) {
        throw new UnsupportedOperationException();
    }
    // Implements the general contract of Map.Entry.equals
    @Override public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry) o;
        return Objects.equals(e.getKey(), getKey())
            && Objects.equals(e.getValue(), getValue());
    }
    // Implements the general contract of Map.Entry.hashCode
    @Override public int hashCode() {
        return Objects.hashCode(getKey())
            ^ Objects.hashCode(getValue());
    }
    @Override public String toString() {
        return getKey() + "=" + getValue();
    }
}
```

Note that this skeletal implementation could not be implemented in the `Map.Entry` interface or as a subinterface because default methods are not permitted to override `Object` methods such as `equals`, `hashCode`, and `toString`.

To summarize, an interface is generally the best way to define a type that permits multiple implementations. If you export a nontrivial interface, you should strongly consider providing a skeletal implementation to go with it. To the extent possible, you should provide the skeletal implementation via default methods on the interface so that all implementors of the interface can make use of it. That said, restrictions on interfaces typically mandate that a skeletal implementation take the form of an abstract class.

## Item 21: Design interfaces for posterity
You can inject new functionality into interfaces via default methods, but existing implementations of an interface may compile without error or warning but fail at runtime if developers don't check for compatibility.

## Item 22: Use interfaces only to define types

Interfaces should be used only to define types. They should not be used to export constants.

## Item 23: Prefer class hierarchies to tagged classes

Tagged classes are seldom appropriate. If you’re tempted to write a class with an explicit tag field, think about whether the tag could be eliminated and the class replaced by a hierarchy. When you encounter an existing class with a tag field, consider refactoring it into a hierarchy.

## Item 24: Favor static member classes over nonstatic

There are four different kinds of nested classes, and each has its place. If a nested class needs to be visible outside of a single method or is too long to fit comfortably inside a method, use a member class. If each instance of the member class needs a reference to its enclosing instance, make it nonstatic; otherwise, make it static. Assuming the class belongs inside a method, if you need to create instances from only one location and there is a preexisting type that characterizes the class, make it an anonymous class; otherwise, make it a local class.

## Item 25: Limit source files to a single top-level class
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

When a class implements an interface, the interface serves as a type that can be used to refer to instances of the class. That a class implements an interface should therefore say something about what a client can do with instances of the class. It is inappropriate to define an interface for any other purpose.

Interfaces should be used only to define types. They should not be used to export constants.

If you want to export constants, there are several reasonable choices:

1. If the constants are strongly tied to an existing class or interface, you should add them to the class or interface. For example, all of the boxed numerical primitive classes, such as `Integer` and `Double`, export `MIN_VALUE` and `MAX_VALUE` constants. 
2. If the constants are best viewed as members of an enumerated type, you should export them with an enum type (Item 34). 
3. Otherwise, you should export the constants with a noninstantiable utility class (Item 4). Here is a utility class version of the `PhysicalConstants` example shown earlier:

```java
// Constant utility class
package com.effectivejava.science;
public class PhysicalConstants {
    private PhysicalConstants() { } // Prevents instantiation
    public static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    public static final double BOLTZMANN_CONST = 1.380_648_52e-23;
    public static final double ELECTRON_MASS = 9.109_383_56e-31;
}

// Use of static import to avoid qualifying constants
import static com.effectivejava.science.PhysicalConstants.*;
    public class Test {
        double atoms(double mols) {
        return AVOGADROS_NUMBER * mols;
    }
    ...
    // Many more uses of PhysicalConstants justify static import
}
```

## Item 23: Prefer class hierarchies to tagged classes

Tagged classes are seldom appropriate. If you’re tempted to write a class with an explicit tag field, think about whether the tag could be eliminated and the class replaced by a hierarchy. When you encounter an existing class with a tag field, consider refactoring it into a hierarchy.

To transform a tagged class into a class hierarchy, first define an abstract class containing an abstract method for each method in the tagged class whose behavior depends on the tag value. In the `Figure` class, there is only one such method, which is `area`. This abstract class is the root of the class hierarchy. If there are any methods whose behavior does not depend on the value of the tag, put them in this class. Similarly, if there are any data fields used by all the flavors, put them in this class.

Next, define a concrete subclass of the root class for each flavor of the original tagged class. In our example, there are two: circle and rectangle. Include in each subclass the data fields particular to its flavor. In our example, `radius` is particular to circle, and `length` and `width` are particular to rectangle. Also include in each subclass the appropriate implementation of each abstract method in the root class.

## Item 24: Favor static member classes over nonstatic

A nested class is a class defined within another class. A nested class should exist only to serve its enclosing class. If a nested class would be useful in some other context, then it should be a top-level class. There are four different kinds of nested classes: *static member classes, nonstatic member classes, anonymous classes*, and *local classes*. All but the first kind are known as inner classes. 

1. Static member class
    - An ordinary class that happens to be declared inside another class and has access to all of the enclosing class’s members, even those declared private
    - Is a static member of its enclosing class and obeys the same accessibility rules as other static members
    - If it is declared private, it is accessible only within the enclosing class, and so forth
    - One common use of a static member class is as a public helper class, useful only in conjunction with its outer class.
2. Nonstatic member class
    - Each instance of a nonstatic member class is implicitly associated with an enclosing instance of its containing class
        -The association between a nonstatic member class instance and its enclosing instance is established when the member class instance is created and cannot be modified thereafter. Normally, the association is established automatically by invoking a nonstatic member class constructor from within an instance method of the enclosing class. It is possible, though rare, to establish the association manually using the expression `enclosingInstance.new MemberClass(args)`. As you would expect, the association takes up space in the nonstatic member class instance and adds time to its construction.
    - Within instance methods of a nonstatic member class, you can invoke methods on the enclosing instance or obtain a reference to the enclosing instance using the *qualified this* construct 
    - If an instance of a nested class can exist in isolation from an instance of its enclosing class, then the nested class must be a static member class: it is impossible to create an instance of a nonstatic member class without an enclosing instance. __If you declare a member class that does not require access to an enclosing instance, always put the static modifier in its declaration__, making it a static rather than a nonstatic member class. If you omit this modifier, each instance will have a hidden extraneous reference to its enclosing instance (potential memory error)
    - One common use of a nonstatic member class is to define an `Adapter` that allows an instance of the outer class to be viewed as an instance of some unrelated class. For example, implementations of the `Map` interface typically use nonstatic member classes to implement their collection views, which are returned by `Map`’s `keySet`, `entrySet`, and `values` methods. Similarly, implementations of the collection interfaces, such as `Set` and `List`, typically use nonstatic member classes to implement their iterators:

    ```java
    // Typical use of a nonstatic member class
    public class MySet<E> extends AbstractSet<E> {
        ... // Bulk of the class omitted
        
        @Override public Iterator<E> iterator() {
            return new MyIterator();
        }

        private class MyIterator implements Iterator<E> {
            ...
        }
    }
    ```

3. Anonymous class
    - An anonymous class has no name. It is not a member of its enclosing class. Rather than being declared along with other members, it is simultaneously declared and instantiated at the point of use. 
    - Anonymous classes are permitted at any point in the code where an expression is legal. 
    - Anonymous classes have enclosing instances if and only if they occur in a nonstatic context. But even if they occur in a static context, they cannot have any static members other than constant variables, which are final primitive or string fields initialized to constant expressions
    - Limitations on the applicability of anonymous classes: 
        - You can’t instantiate them except at the point they’re declared 
        - You can’t perform `instanceof` tests or do anything else that requires you to name the class. 
        - You can’t declare an anonymous class to implement multiple interfaces or to extend a class and implement an interface at the same time. 
        - Clients of an anonymous class can’t invoke any members except those it inherits from its supertype. 
        - Because anonymous classes occur in the midst of expressions, they must be kept short — about ten lines or fewer — or readability will suffer.
    - Before lambdas were added to Java (Chapter 6), anonymous classes were the preferred means of creating small function objects and process objects on the fly, but lambdas are now preferred (Item 42). - Another common use of anonymous classes is in the implementation of static factory methods (see `intArrayAsList` in Item 20).\

4. Local classes
    - least frequently used of the four kinds of nested classes 
    - A local class can be declared practically anywhere a local variable can be declared and obeys the same scoping rules
    - Like member classes, they have names and can be used repeatedly. 
    - Like anonymous classes, they have enclosing instances only if they are defined in a nonstatic context, and they cannot contain static members
    - Like anonymous classes, they should be kept short so as not to harm readability

Recap: if a nested class needs to be visible outside of a single method or is too long to fit comfortably inside a method, use a member class. If each instance of the member class needs a reference to its enclosing instance, make it nonstatic; otherwise, make it static. Assuming the class belongs inside a method, if you need to create instances from only one location and there is a preexisting type that characterizes the class, make it an anonymous class; otherwise, make it a local class.

## Item 25: Limit source files to a single top-level class
While the Java compiler lets you define multiple top-level classes in a single source file, there are no benefits associated with doing so, and there are significant risks. The risks stem from the fact that defining multiple top-level classes in a source file makes it possible to provide multiple definitions for a class. Which definition gets used is affected by the order in which the source files are passed to the compiler.

While the Java compiler lets you define multiple top-level classes in a single source file, there are no benefits associated with doing so, and there are significant risks. The risks stem from the fact that defining multiple top-level classes in a source file makes it possible to provide multiple definitions for a class. Which definition gets used is affected by the order in which the source files are passed to the compiler.

__Never put multiple top-level classes or interfaces in a single source file__. Following this rule guarantees that you can’t have multiple definitions for a single class at compile time. This in turn guarantees that the class files generated by compilation, and the behavior of the resulting program, are independent of the order in which the source files are passed to the compiler.
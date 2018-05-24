# Chapter 2. Creating and Destroying Objects
This chapter concerns creating and destroying objects: when and how to create them, when and how to avoid creating them, how to ensure they are destroyed in a timely manner, and how to manage any cleanup actions that must precede their destruction.

## Item 1: Consider static factory methods instead of constructors

1. One advantage of static factory methods is that, unlike constructors, they have names.

```java
BigInteger(int, int, Random)
BigInteger.probablePrime     // Better expressed
```

2. A second advantage of static factory methods is that, unlike constructors, they are not required to create a new object each time they’re invoked.

```java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```

3. A third advantage of static factory methods is that, unlike constructors, they can return an object of any subtype of their return type.

For example, the class `java.util.EnumSet`, introduced in release 1.5, has no public constructors, only static factories. They return one of two implementations, depending on the size of the underlying enum type: if it has sixty-four or fewer elements, as most enum types do, the static factories return a `RegularEnumSet` instance, which is backed by a single `long`; if the enum type has sixty-five or more elements, the factories return a `JumboEnumSet` instance, backed by a `long` array.

4. A fourth advantage of static factory methods is that they reduce the verbosity of creating parameterized type instances.

```
Map<String, List<String>> m = new HashMap<String, List<String>>();
Map<String, List<String>> m = HashMap.newInstance();
```

5. A fifth advantage of static factories is that the class of the returned object need not exist when the class containing the method is written.



1. The main disadvantage of providing only static factory methods is that classes without public or protected constructors cannot be subclassed.

2. A second disadvantage of static factory methods is that they are not readily distinguishable from other static methods.

Here are some common names for static factory methods:

- `valueOf` - Returns an instance that has, loosely speaking, the same value as its parameters. Such static factories are effectively type-conversion methods.
- `of` - A concise alternative to `valueOf`, popularized by `EnumSet` (Item 32).
- `getInstance` - Returns an instance that is described by the parameters but cannot be said to have the same value. In the case of a singleton, `getInstance` takes no parameters and returns the sole instance.
- `newInstance` - Like `getInstance`, except that newInstance guarantees that each instance returned is distinct from all others.
- `get`*Type*  - Like `getInstance`, but used when the factory method is in a different class. *Type* indicates the type of object returned by the factory method.
- `new`*Type* - Like `newInstance`, but used when the factory method is in a different class. *Type* indicates the type of object returned by the factory method.

In summary, static factory methods and public constructors both have their uses, and it pays to understand their relative merits. Often static factories are preferable, so avoid the reflex to provide public constructors without first considering static factories.

## Item 2: Consider a builder when faced with many constructor parameters

- Telescoping constructor pattern - does not scale well
- JavaBeans pattern - allows inconsistency, mandates mutability
- Builder pattern - is good when constructors or static factories would have more than a handful of parameters

```java
// Builder Pattern
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        // Required parameters
        private final int servingSize;
        private final int servings;

        // Optional parameters - initialized to default values
        private int calories      = 0;
        private int fat           = 0;
        private int carbohydrate  = 0;
        private int sodium        = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings    = servings;
        }

        public Builder calories(int val) {
            calories = val;
            return this;
        }

        public Builder fat(int val) {
            fat = val;
            return this;
        }

        public Builder carbohydrate(int val) {
            carbohydrate = val;
            return this;
        }

        public Builder sodium(int val) {
            sodium = val;
            return this;
        }

        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }

    private NutritionFacts(Builder builder) {
        servingSize  = builder.servingSize;
        servings     = builder.servings;
        calories     = builder.calories;
        fat          = builder.fat;
        sodium       = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```

Note that `NutritionFacts` is immutable, and that all parameter default values are in a single location. The builder’s setter methods return the builder itself so that invocations can be chained. Here’s how the client code looks:

```java
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8).
    calories(100).sodium(35).carbohydrate(27).build();
```
This client code is easy to write and, more importantly, easy to read. 
The Builder pattern simulates named optional parameters as found in Python and Scala.

The Builder pattern is well suited to class hierarchies. 
Use a parallel hierarchy of builders, each nested in the corresponding class. 
Abstract classes have abstract builders; concrete classes have concrete builders. 
For example, consider an abstract class at the root of a hierarchy representing various kinds of pizza:
```java
// Builder pattern for class hierarchies
public abstract class Pizza {

   public enum Topping { HAM, MUSHROOM, ONION, PEPPER, SAUSAGE }

   final Set<Topping> toppings;

   abstract static class Builder<T extends Builder<T>> {

      EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);

      public T addTopping(Topping topping) {

         toppings.add(Objects.requireNonNull(topping));

         return self();

      }

      abstract Pizza build();

      // Subclasses must override this method to return "this"

      protected abstract T self();

   }

   Pizza(Builder<?> builder) {

      toppings = builder.toppings.clone(); // See Item  50

   }
}
```
Note that Pizza.Builder is a generic type with a recursive type parameter (Item 30). 
This, along with the abstract self method, allows method chaining to work properly in subclasses, without the need for casts. 
This workaround for the fact that Java lacks a self type is known as the simulated self-type idiom.

Here are two concrete subclasses of Pizza, one of which represents a standard New-York-style pizza, the other a calzone. The former has a required size parameter, while the latter lets you specify whether sauce should be inside or out:
```java
public class NyPizza extends Pizza {
    public enum Size { SMALL, MEDIUM, LARGE }
    private final Size size;

    public static class Builder extends Pizza.Builder<Builder> {

        private final Size size;

        public Builder(Size size) {

            this.size = Objects.requireNonNull(size);

        }

        @Override public NyPizza build() {
            return new NyPizza(this);
        }

        @Override protected Builder self() { return this; }
    }
    
    private NyPizza(Builder builder) {
        super(builder);
        size = builder.size;
    }
}

public class Calzone extends Pizza {
    private final boolean sauceInside;
    
    public static class Builder extends Pizza.Builder<Builder> {

        private boolean sauceInside = false; // Default

        public Builder sauceInside() {
            sauceInside = true;
            return this;

        }

        @Override public Calzone build() {
            return new Calzone(this);
        }
        
        @Override protected Builder self() { return this; }
    }

    private Calzone(Builder builder) {
        super(builder);
        sauceInside = builder.sauceInside;
    }
}
```
Note that the build method in each subclass’s builder is declared to return the correct subclass: the build method of NyPizza.Builder returns NyPizza, while the one in Calzone.Builder returns Calzone. This technique, wherein a subclass method is declared to return a subtype of the return type declared in the super-class, is known as covariant return typing. It allows clients to use these builders without the need for casting.

The client code for these “hierarchical builders” is essentially identical to the code for the simple NutritionFacts builder. The example client code shown next assumes static imports on enum constants for brevity:
```java
NyPizza pizza = new NyPizza.Builder(SMALL)
        .addTopping(SAUSAGE).addTopping(ONION).build();

Calzone calzone = new Calzone.Builder()
        .addTopping(HAM).sauceInside().build();
```
A minor advantage of builders over constructors is that builders can have multiple varargs parameters because each parameter is specified in its own method. Alternatively, builders can aggregate the parameters passed into multiple calls to a method into a single field, as demonstrated in the addTopping method earlier.

The Builder pattern is quite flexible. A single builder can be used repeatedly to build multiple objects. The parameters of the builder can be tweaked between invocations of the build method to vary the objects that are created. A builder can fill in some fields automatically upon object creation, such as a serial number that increases each time an object is created.

Disadvantages: 
1. In order to create an object, you must first create its builder. It could be a problem in performance-critical situations.
2. Builder pattern is more verbose than the telescoping constructor pattern, so it should be used only if there are enough parameters to make it worthwhile, say four or more. 
    1. However, you may want to add more parameters in the future. If you start out with constructors or static factories and switch to a builder when the class evolves to the point where the number of parameters gets out of hand, the obsolete constructors or static factories will stick out like a sore thumb. Therefore, it’s often better to start with a builder in the first place.

In summary, the Builder pattern is a good choice when designing classes whose constructors or static factories would have more than a handful of parameters, especially if many of the parameters are optional or of identical type. Client code is much easier to read and write with builders than with telescoping constructors, and builders are much safer than JavaBeans.

## Item 3: Enforce the singleton property with a private constructor or an enum type

A class that is instantiated exactly once, typically representing a stateless object (e.g. a function) or a system component that is intrinsically unique. 

~~Two common ways to implement, both keeping the constructor private and exporting a public static member to provide access to the sole instance.

1. The member is a final field:

```java
// Singleton with public final field
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public void leaveTheBuilding() { ... }
}
```

The private constructor is called only once. Lack of public or protected (only accessible to subclasses) guarantees exactly one instance will ever exist. 

However, a privileged client can invoke the private constructor reflectively (Item 65) with the aid of the AccessibleObject.setAccessible method. To prevent this, you can modify the constructor to make it throw an exception if it’s asked to create a second instance.

2. The public member is a static factory method:

```java
// Singleton with static factory
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public static Elvis getInstance() { return INSTANCE; }
    public void leaveTheBuilding() { ... }
}
```

All calls to Elvis.getInstance return the same object reference, and no other Elvis instance will ever be created.

The first approach with the public field makes clear that the class is a singleton and is simpler. The advantage to the second approach with the static factory is that it gives you the flexibility to change your mind about whether the class is a singleton without changing its
API. The factory method it could be modified to return, say, a separate instance for each thread that invokes it. A second advantage
is that you can write a generic singleton factory if your application requires it (Item 30). A final advantage of using a static factory is that a method reference can be used as a supplier, for example Elvis::instance is a Supplier<Elvis>. Unless one of these advantages is relevant, the public field approach is preferable.

To make a singleton class that uses either of these approaches serializable (Chapter 12), declare all instance fields
transient and provide a readResolve method (Item 89). Otherwise, each time a serialized instance is deserialized, a new instance will be created, leading, in the case of our example, to spurious Elvis sightings.

```java
// readResolve method to preserve singleton property
private Object readResolve() {
    // Return the one true Elvis and let the garbage collector
    // take care of the Elvis impersonator.
    return INSTANCE;
}
```~~

3. A single-element enum type is the best way to implement a singleton.

```java
// Enum singleton - the preferred approach
public enum Elvis {
   INSTANCE;
   public void leaveTheBuilding() { ... }
}
```

Similar to the public field approach, but more concise and provides ironclad guarantee against multiple instantiation even against sophisticated serialization or reflection attacks. Note that you can’t
use this approach if your singleton must extend a superclass other than Enum (though you can declare an enum to implement interfaces).

## Item 4: Enforce noninstantiability with a private constructor
Occasionally you’ll want to write a class that is just a grouping of static methods and static fields. They can be used to group related methods on primitive values or arrays, in the manner of java.lang.Math or java.util.Arrays. They can also be used to group static methods, including factories (Item 1), for objects that implement some interface,
in the manner of java.util.Collections. (As of Java 8, you can also put such methods in the interface, assuming it’s yours to modify.) Lastly, such classes can be used to group methods on a final class, since you can’t put them in a subclass.

In the absence of explicit constructors, however, the compiler provides a public, parameterless default constructor. To a user, this constructor is indistinguishable from any other. It is not uncommon to see unintentionally instantiable classes in published APIs.

Attempting to enforce noninstantiability by making a class abstract does not work. The class can be subclassed and the subclass instantiated. 
Furthermore, it misleads the user into thinking the class was designed for inheritance (Item 19).  There is, however, a simple idiom to ensure noninstantiability. A default constructor is generated only if a class contains no explicit constructors, so a class can be made noninstantiable by including a private constructor:

```java
// Noninstantiable utility class
public class UtilityClass {
    // Suppress default constructor for noninstantiability
    private UtilityClass() {
        throw new AssertionError();
    }
    ...  // Remainder omitted
}
```

The AssertionError isn’t strictly required, but it provides insurance against the case that the constructor is accidentally invoked from within the class.

As a side effect, this idiom also prevents the class from being subclassed. All constructors must invoke a superclass constructor, explicitly or implicitly, and a subclass would have no accessible superclass constructor to invoke.

## Item 5: Prefer dependency injection to hardwiring resources
Static utility classes and singletons are inappropriate for classes whose behavior is parameterized by an underlying resource, e.g.:
```java
// Inappropriate use of static utility - inflexible & untestable!
public class SpellChecker {
    private static final Lexicon dictionary = ...;
    private SpellChecker() {} // Noninstantiable
    public static boolean isValid(String word) { ... }
    public static List<String> suggestions(String typo) { ... }
}

// Inappropriate use of singleton - inflexible & untestable!
public class SpellChecker {
    private final Lexicon dictionary = ...;
    private SpellChecker(...) {}
    public static INSTANCE = new SpellChecker(...);
    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```
Neither of these approaches is satisfactory, because they assume that there is only one dictionary worth using.

We need to support multiple instances of the class (in our example, SpellChecker), each of which uses the resource desired by the client (in our example, the dictionary). Thus, we should __pass the resource into the constructor when creating a new instance__. This is dependency injection: the dictionary is a dependency of the spell checker and is injected into the spell checker when it is created. 
```java
// Dependency injection provides flexibility and testability
public class SpellChecker {
    private final Lexicon dictionary;

    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }

    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```
Dependency injection preserves immutability (Item 17), so multiple clients can share dependent objects (assuming the clients desire the same underlying resources). Dependency injection is equally applicable to constructors, static factories (Item 1), and builders (Item 2).

We can also pass a resource factory to the constructor. __A factory is an object that can be called repeatedly to create instances of a type__. The Supplier<T> interface, introduced in Java 8, is perfect for representing factories. Methods that take a Supplier<T> on input should typically constrain the factory’s type parameter using a bounded wildcard type (Item 31) to allow the client to pass in a factory that creates any subtype of a specified type. For example, here is a method that makes a mosaic using a client-provided factory to produce each tile:
```java
Mosaic create(Supplier<? extends Tile> tileFactory) { ... }
```
Although dependency injection greatly improves flexibility and testability, it can clutter up large projects, which typically contain thousands of dependencies.  This clutter can be eliminated by using a dependency injection framework, such as Dagger [Dagger], Guice [Guice], or Spring [Spring]. APIs designed for manual dependency injection are trivially adapted for use by these frameworks.

In summary, do not use a singleton or static utility class to implement a class that depends on one or more underlying resources whose behavior affects that of the class, and do not have the class create these resources directly. Instead, pass the resources, or factories to create them, into the constructor (or static factory or builder). This practice, known as dependency injection, will greatly enhance the flexibility, reusability, and testability of a class.

## Item 6: Avoid creating unnecessary objects

## Item 7: Eliminate obsolete object references 

## Item 8: Avoid finalizers and cleaners 

## Item 9: Prefer try-with-resources to try-finally

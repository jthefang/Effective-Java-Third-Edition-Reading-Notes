# Chapter 6. Enums and Annotations

Java supports two special-purpose families of reference types: a kind of class called an enum type, and a kind of interface called an annotation type. This chapter discusses best practices for using these type families.

## Item 34: Use enums instead of int constants

An enumerated type is a type whose legal values consist of a fixed set of constants, such as the seasons of the year, the planets in the solar system, or the suits in a deck of playing cards. It's essentially a set of constants representing some enumeration. Before enum types were added to the language, a common pattern for representing enumerated types was to declare a group of named int constants, one for each member of the type:

```java
// The int enum pattern - severely deficient!
public static final int APPLE_FUJI = 0;
public static final int APPLE_PIPPIN = 1;
public static final int APPLE_GRANNY_SMITH = 2;

public static final int ORANGE_NAVEL = 0;
public static final int ORANGE_TEMPLE = 1;
public static final int ORANGE_BLOOD = 2;
```

This technique, known as the int enum pattern, has many shortcomings. 

- Luckily, Java provides an alternative that avoids all the shortcomings of the `int` and `string` enum patterns and provides many added benefits. It is the enum type. Here’s how it looks in its simplest form:

```java
public enum Apple { FUJI, PIPPIN, GRANNY_SMITH }
public enum Orange { NAVEL, TEMPLE, BLOOD }
```

On the surface, these enum types may appear similar to those of other languages, such as C, C++, and C#, but appearances are deceiving. Java’s enum types are full-fledged classes, far more powerful than their counterparts in these other languages, where enums are essentially int values.

- The basic idea behind Java’s enum types is simple: *they are classes that export one instance for each enumeration constant via a public static final field*. Enum types are effectively final, by virtue of having no accessible constructors. Because clients can neither create instances of an enum type nor extend it, there can be no instances but the declared enum constants. In other words, enum types are instance-controlled (page 6). They are a generalization of singletons (Item 3), which are essentially single-element enums.

- Compile-time type safety: if you declare a parameter to be of type `Apple`, you are guaranteed that any non-null object reference passed to the parameter is one of the three valid `Apple` values (else a compile-time error).

- Enum types with identically named constants coexist peacefully because each type has its own namespace. You can add or reorder constants in an enum type without recompiling its clients because the fields that export the constants provide a layer of insulation between an enum type and its clients: constant values are not compiled into the clients as they are in the int enum patterns.

- Translate enums into printable strings by calling their `toString` method, which returns the declared name of each enum value. You can change this string representation by overriding the `toString` method for the enum.

- Enum types let you add arbitrary methods and fields and implement arbitrary interfaces. They provide high-quality implementations of all the `Object` methods (Chapter 3), they implement Comparable (Item 14) and Serializable (Chapter 12), and their serialized form is designed to withstand most changes to the enum type. i.e. Enum types can be full-featured abstractions.

Example: consider the eight planets of our solar system. Each planet has a mass and a radius, and from these two attributes you can compute its surface gravity. This in turn lets you compute the weight of an object on the planet’s surface, given the mass of the object. Here’s how this enum looks. The numbers in parentheses after each enum constant are parameters that are passed to its constructor. In this case, they are the planet’s mass and radius:

```java
// Enum type with data and behavior
public enum Planet {
    MERCURY(3.302e+23, 2.439e6),
    VENUS (4.869e+24, 6.052e6),
    EARTH (5.975e+24, 6.378e6),
    MARS (6.419e+23, 3.393e6),
    JUPITER(1.899e+27, 7.149e7),
    SATURN (5.685e+26, 6.027e7),
    URANUS (8.683e+25, 2.556e7),
    NEPTUNE(1.024e+26, 2.477e7);
 
    private final double mass; // In kilograms
    private final double radius; // In meters
    private final double surfaceGravity; // In m / s^2

    // Universal gravitational constant in m^3 / kg s^2
    private static final double G = 6.67300E-11;
    
    // Constructor
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
        surfaceGravity = G * mass / (radius * radius);
    }
    
    public double mass() { return mass; }
    public double radius() { return radius; }
    public double surfaceGravity() { return surfaceGravity; }
    
    public double surfaceWeight(double mass) {
        return mass * surfaceGravity; // F = ma
    }
}
```

While the Planet enum is simple, it is surprisingly powerful. Here is a short program that takes the earth weight of an object (in any unit) and prints a nice table of the object’s weight on all eight planets (in the same unit):

```java
public class WeightTable {
    public static void main(String[] args) {
        double earthWeight = Double.parseDouble(args[0]);
        double mass = earthWeight / Planet.EARTH.surfaceGravity();
        for (Planet p : Planet.values())
            System.out.printf("Weight on %s is %f%n",
                p, p.surfaceWeight(mass));
    }
}
```

- __To associate data with enum constants, declare instance fields and write a constructor that takes the data and stores it in the fields__. Enums are by their nature immutable, so all fields should be final (Item 17). Fields can be public, but it is better to make them private and provide public accessors (Item 16).

- Note that Planet, like all enums, has __a static `values` method that returns an array of its values in the order they were declared__.

- If an enum is generally useful, it should be a top-level class; if its use is tied to a specific top-level class, it should be a member class of that top-level class (Item 24).

- To program fundamentally different behavior for each constant/value of an enum type, you can:
    1. Switch on the value of the enum:
    ```java
    // Enum type that switches on its own value - questionable
    public enum Operation {
        PLUS, MINUS, TIMES, DIVIDE;
        // Do the arithmetic operation represented by this constant
        public double apply(double x, double y) {
            switch(this) {
                case PLUS: return x + y;
                case MINUS: return x - y;
                case TIMES: return x * y;
                case DIVIDE: return x / y;
            }
            throw new AssertionError("Unknown op: " + this);
        }
    }
    ```

    Won't compile without the `throw` statement since the end of the method is technically reachable, though it will never be reached. Fragile, since everytime you add a new constant but forget to add a switch case, this code still compiles but will fail at runtime.

    2. *Constant-specific method implementations*: declare an abstract apply method in the enum type, and override it with a concrete method for each constant in a constant-specific class body. Such methods are known as constant-specific method implementations:

    ```java
    // Enum type with constant-specific class bodies and data
    public enum Operation {
        PLUS("+") {
        public double apply(double x, double y) { return x + y; }
        },
        MINUS("-") {
        public double apply(double x, double y) { return x - y; }
        },
        TIMES("*") {
        public double apply(double x, double y) { return x * y; }
        },
        DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
        };

        private final String symbol;
        
        Operation(String symbol) { this.symbol = symbol; }
        
        @Override public String toString() { return symbol; }
        
        public abstract double apply(double x, double y);
    }
    ```

-Enum types have an automatically generated `valueOf(String)` method that translates a constant’s name into the constant itself. 

- If you override the `toString` method in an enum type, consider writing a `fromString` method to translate the custom string representation back to the corresponding enum. The following code (with the type name changed appropriately) will do the trick for any enum, so long as each constant has a unique string representation:

```java
// Implementing a fromString method on an enum type
private static final Map<String, Operation> stringToEnum =
    Stream.of(values()).collect(toMap(Object::toString, e -> e));

// Returns Operation for string, if any
public static Optional<Operation> fromString(String symbol) {
    return Optional.ofNullable(stringToEnum.get(symbol));
}
```

Note that the `fromString` method returns an `Optional<String>`. This allows the method to indicate that the string that was passed in does not represent a valid operation, and it forces the client to confront this possibility (Item 55).

- Switches on enums are good for augmenting enum types with constant-specific behavior. For example, suppose the Operation enum is *not under your control* and you wish it had an instance method to return the inverse of each operation. You could simulate the effect with the following static method:

```java
// Switch on an enum to simulate a missing method
public static Operation inverse(Operation op) {
    switch(op) {
        case PLUS: return Operation.MINUS;
        case MINUS: return Operation.PLUS;
        case TIMES: return Operation.DIVIDE;
        case DIVIDE: return Operation.TIMES;
        default: throw new AssertionError("Unknown op: " + op);
    }
}
```

- __Use enums any time you need a set of constants whose members are known at compile time__.

The advantages of enum types over int constants are compelling. Enums are far more readable, safer, and more powerful. Many enums require no explicit constructors or members, but many others benefit from associating data with each constant and providing methods whose behavior is affected by this data. Far fewer enums benefit from associating multiple behaviors with a single method. In this relatively rare case, prefer constant-specific methods to enums that switch on their own values. Consider the strategy enum pattern if multiple enum constants share common behaviors (i.e. create a private nested enum type that enumerates the strategies/behaviors you wish of your outer enum constants).

## Item 35: Use instance fields instead of ordinals

All enums have an `ordinal` method, which returns the numerical position of each enum constant in its type.

__Never derive a value associated with an enum from its ordinal; store it in an instance field instead__:

```java
public enum Ensemble {
    SOLO(1), DUET(2), TRIO(3), QUARTET(4), QUINTET(5),
    SEXTET(6), SEPTET(7), OCTET(8), DOUBLE_QUARTET(8),
    NONET(9), DECTET(10), TRIPLE_QUARTET(12);
 
    private final int numberOfMusicians;
    Ensemble(int size) { this.numberOfMusicians = size; }
    public int numberOfMusicians() { return numberOfMusicians; }
}
```

Most programmers will have no use for this method. It is designed for use by general-purpose enum-based data structures such as EnumSet and EnumMap. Unless you are writing such a data structure, you are best off avoiding the ordinal method entirely.

## Item 36: Use EnumSet instead of bit fields

If the elements of an enumerated type are used primarily in sets, it is traditional to use the int enum pattern (Item 34), assigning a different power of 2 to each constant:

```java
// Bit field enumeration constants - OBSOLETE!
public class Text {
    public static final int STYLE_BOLD = 1 << 0; // 1
    public static final int STYLE_ITALIC = 1 << 1; // 2
    public static final int STYLE_UNDERLINE = 1 << 2; // 4
    public static final int STYLE_STRIKETHROUGH = 1 << 3; // 8
    // Parameter is bitwise OR of zero or more STYLE_ constants
    public void applyStyles(int styles) { ... }
}
```

This representation lets you use the bitwise OR operation to combine several constants into a set, known as a bit field:

```java
text.applyStyles(STYLE_BOLD | STYLE_ITALIC);
```

The bit field representation also lets you perform set operations such as union and intersection efficiently using bitwise arithmetic. __But bit fields have all the disadvantages of int enum constants and more__.

Here is how the previous example looks when modified to use enums and enum sets instead of bit fields. It is shorter, clearer, and safer:

```java
// EnumSet - a modern replacement for bit fields
public class Text {
    public enum Style { BOLD, ITALIC, UNDERLINE, STRIKETHROUGH }
    
    // Any Set could be passed in, but EnumSet is clearly best
    public void applyStyles(Set<Style> styles) { ... }
}
```

Here is client code that passes an EnumSet instance to the applyStyles method. The EnumSet class provides a rich set of static factories for easy set creation, one of which is illustrated in this code: 

```java
text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
```

Just because an enumerated type will be used in sets, there is no reason to represent it with bit fields. The `EnumSet` class combines the conciseness and performance of bit fields with all the many advantages of enum types described in [Item 34](chapter-6.md#item-34-use-enums-instead-of-int-constants). The one real disadvantage of `EnumSet` is that it is not, as of Java 9, possible to create an immutable `EnumSet`, but this will likely be remedied in an upcoming release. In the meantime, you can wrap an `EnumSet` with `Collections.unmodifiableSet`, but conciseness and performance will suffer.

## Item 37: Use EnumMap instead of ordinal indexing

Occasionally you may see code that uses the ordinal method (Item 35) to index into an array or list. For example, consider this simplistic class meant to represent a plant:

```java
class Plant {
    enum LifeCycle { ANNUAL, PERENNIAL, BIENNIAL }
    final String name;
    final LifeCycle lifeCycle;
    
    Plant(String name, LifeCycle lifeCycle) {
        this.name = name;
        this.lifeCycle = lifeCycle;
    }

    @Override public String toString() {
        return name;
    }
}
```

Now suppose you have an array of plants representing a garden, and you want to list these plants organized by life cycle (annual, perennial, or biennial). To do this, you construct three sets, one for each life cycle, and iterate through the garden, placing each plant in the appropriate set. Some programmers would do this by putting the sets into an array indexed by the life cycle’s ordinal:

```java
// Using ordinal() to index into an array - DON'T DO THIS!
Set<Plant>[] plantsByLifeCycle =
    (Set<Plant>[]) new Set[Plant.LifeCycle.values().length];
for (int i = 0; i < plantsByLifeCycle.length; i++)
    plantsByLifeCycle[i] = new HashSet<>();

for (Plant p : garden)
    plantsByLifeCycle[p.lifeCycle.ordinal()].add(p);

// Print the results
for (int i = 0; i < plantsByLifeCycle.length; i++) {
    System.out.printf("%s: %s%n",
        Plant.LifeCycle.values()[i], plantsByLifeCycle[i]);
}
```

This technique works, but it is fraught with problems. There is a very fast `Map` implementation designed for use with enum
keys, known as `java.util.EnumMap`. Here is how the program looks when it is rewritten to use `EnumMap`:

```java
// Using an EnumMap to associate data with an enum
Map<Plant.LifeCycle, Set<Plant>> plantsByLifeCycle =
    new EnumMap<>(Plant.LifeCycle.class);
for (Plant.LifeCycle lc : Plant.LifeCycle.values())
    plantsByLifeCycle.put(lc, new HashSet<>());
for (Plant p : garden)
    plantsByLifeCycle.get(p.lifeCycle).add(p);
System.out.println(plantsByLifeCycle);
```

This program is shorter, clearer, safer, and comparable in speed to the original version.

It is rarely appropriate to use ordinals to index arrays: use `EnumMap` instead.

## Item 38: Emulate extensible enums with interfaces

While you cannot write an extensible enum type (i.e. you cannot have one enumerated type extend another), you can emulate it by writing an interface to go with a basic enum type that implements the interface. This allows clients to write their own enums (or other types) that implement the interface. Instances of these types can then be used wherever the basic enum type can be used, assuming APIs are written in terms of the interface.

## Item 39: Prefer annotations to naming patterns

e.g. JUnit test annotations

- You can also define your own annotations for use with custom tools

- Annotations don’t change the semantics of the annotated code but
enable it for special treatment by tools (e.g. JUnit)

If you write a tool that requires programmers to add information to source files, define an appropriate set of annotation types. There is simply no reason to use naming patterns when you can use annotations instead. That said, with the exception of toolsmiths, most programmers will have no need to define annotation types. All programmers should, however, use the predefined annotation types that Java provides (Items 40 and Item 27). Also, consider using the annotations provided by your IDE or static analysis tools. Such annotations can improve the quality of the diagnostic information provided by these tools. Note, however, that these annotations have yet to be standardized, so you may have some work to do if you switch tools or if a standard emerges.

## Item 40: Consistently use the Override annotation

The compiler can protect you from a great many errors if you use the `Override` annotation on every method declaration that you believe to override a supertype declaration.

## Item 41: Use marker interfaces to define types

A marker interface is an interface that contains no method declarations but merely designates (or “marks”) a class that implements the interface as having some property. For example, consider the Serializable interface (Chapter 12). By implementing this interface, a class indicates that its instances can be written to an ObjectOutputStream (or “serialized”).

You may hear it said that marker annotations (Item 39) make marker interfaces obsolete. This assertion is incorrect. Marker interfaces have two advantages over marker annotations.

1. Marker interfaces define a type that is implemented by instances of the marked class; marker annotations do not. The existence of a marker interface type allows you to catch errors at compile time that you couldn’t catch until runtime if you used a marker annotation.

2. Another advantage of marker interfaces over marker annotations is that they can be targeted more precisely. If an annotation type is declared with target `ElementType.TYPE`, it can be applied to any class or interface. Suppose you have a marker that is applicable only to implementations of a particular interface. If you define it as a marker interface, you can have it extend the sole interface to which it is applicable, guaranteeing that all marked types are also subtypes of the sole interface to which it is applicable.

e.g. the `Set` interface is just such a restricted marker interface. It is applicable only to `Collection` subtypes, but it adds no methods beyond those defined by `Collection`. It is not generally considered to be a marker interface because it refines the contracts of several `Collection` methods, including `add`, `equals`, and `hashCode`. But it is easy to imagine a marker interface that is applicable only to subtypes of some particular interface and does not refine the contracts of any of the interface’s methods. Such a marker interface might describe some invariant of the entire object or indicate that instances are eligible for processing by a method of some other class (in the way that the `Serializable` interface indicates that instances are eligible for processing by `ObjectOutputStream`).

- The chief advantage of marker annotations over marker interfaces is that they are part of the larger annotation facility. Therefore, marker annotations allow for consistency in annotation-based frameworks. Also, they can be applied to types other than classes and interfaces (unlike marker interfaces).

Marker interfaces and marker annotations both have their uses. If you want to define a type that does not have any new methods associated with it, a marker interface is the way to go. If you want to mark program elements other than classes and interfaces, to allow for the possibility of adding more information to the marker in the future, or to fit the marker into a framework that already makes heavy use of annotation types, then a marker annotation is the correct choice. If you find yourself writing a marker annotation type whose target is `ElementType.TYPE`, take the time to figure out whether it really should be an annotation type, or whether a marker interface would be more appropriate.

This item is a complement of Item 22. If you want to define a type, use an interface.
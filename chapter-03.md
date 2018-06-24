# Chapter 3. Methods Common to All Objects
ALTHOUGH Object is a concrete class, it is designed primarily for extension. All of its nonfinal methods (equals, hashCode, toString, clone, and finalize) have explicit general contracts because they are designed to be overridden. It is the responsibility of any class overriding these methods to obey their general contracts; failure to do so will prevent other classes that depend on the contracts (such as HashMap and HashSet) from functioning properly in conjunction with the class.

This chapter tells you when and how to override the nonfinal Object methods. The finalize method is omitted from this chapter because it was discussed in Item 8. While not an Object method, Comparable.compareTo is discussed in this chapter because it has a similar character.

## Item 10: Obey the general contract when overriding equals
Don't override equals (i.e. each instance of the class should be equal only to itself, must be same exact object reference) if:
- Each instance of the class is inherently unique (e.g. Threads, which represent active entities rather than values)
- There is no need for the class to provide a "logical equality" test
- A superclass has overriden equals and its behavior is appropriate for this class (e.g. Set implementations inherit equals from AbstractSet)
- The class is private or package-private, and you're certain that its equals method will never be invoked. You can ensure this with:
```java
@Override public boolean equals(Object o) {
    throw new AssertionError(); // Method is never called
}
```

The equals contract for non-null objects:
- Reflexive: x.equals(x) must be true
- Symmetric: If x.equals(y) => y.equals(x)
- Transitive: If x.equals(y) && y.equals(z) => x.equals(z)
- Consistent: x.equals(y) must consistently return the same value, as long as no information used in equals comparisons is modified
- For any non-null reference value x, x.equals(null) must return false

Many classes (e.g. collections classes) depend on the objects passed to them to obey the equals contract in order to work properly. __Once you’ve violated the equals contract, you simply don’t know how other objects will behave when confronted with your object__.

__There is no way to extend an instantiable class and add a value component while preserving the equals contract, unless you’re willing to forgo the benefits of object-oriented abstraction__. Most ways of doing this will either violate transitivity or symmetry (e.g. Timestamp and Date classes violate this and should not be used together) or not allow an instance of the subclass to be comparable to an instance of the superclass. You can forgo inheritance:

```java
// Adds a value component without violating the equals contract
public class ColorPoint {
    private final Point point;
    private final Color color;
    public ColorPoint(int x, int y, Color color) {
        point = new Point(x, y);
        this.color = Objects.requireNonNull(color);
    }
    /**
    * Returns the point-view of this color point.
    */
    public Point asPoint() {
        return point;
    }
    @Override public boolean equals(Object o) {
        if (!(o instanceof ColorPoint))
            return false;
        ColorPoint cp = (ColorPoint) o;
        return cp.point.equals(point) && cp.color.equals(color);
    }
    ... // Remainder omitted
}
```

Here’s a recipe for a high-quality equals method:

- Use the `==` operator to check if the argument is a reference to this object. If so, return `true`.
- Use the `instanceof` operator to check if the argument has the correct type. If not, return `false`.
    - Use an interface if the class implements an interface that refines the equals contract to permit comparisons across classes that implement the interface. Collection interfaces such as Set, List, Map, and Map.Entry have this property.
- Cast the argument to the correct type. Because this cast was preceded by an `instanceof` test, it is guaranteed to succeed.
- For each "significant" field in the class, check if that field of the argument matches the corresponding field of this object. If all these tests succeed, return `true`; otherwise, return `false`.
    - For primitive fields (not `float` or `double`) use the `==` operator; for object reference fields call `equals` method
    - For `float` fields, use the static `Float.compare(float, float)` method; for `double` fields, use `Double.compare(double, double)`. This is made necessary by the existence of `Float.NaN`, `-0.0f` and the analogous `double` values
    - For arrays use one of the `Arrays.equals` methods

Example:
```java
// Class with a typical equals method
public final class PhoneNumber {
    private final short areaCode, prefix, lineNum;

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = rangeCheck(areaCode, 999, "area code");
        this.prefix = rangeCheck(prefix, 999, "prefix");
        this.lineNum = rangeCheck(lineNum, 9999, "line num");
    }

    private static short rangeCheck(int val, int max, String arg) {
        if (val < 0 || val > max)
            throw new IllegalArgumentException(arg + ": " + val);
        return (short) val;
    }

    @Override 
    public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof PhoneNumber))
            return false;
        PhoneNumber pn = (PhoneNumber)o;
        return pn.lineNum == lineNum && pn.prefix == prefix
            && pn.areaCode == areaCode;
    }
    ... // Remainder omitted
}
```

- When you are finished writing your `equals` method, ask yourself three questions: Is it symmetric? Is it transitive? Is it consistent?
- Always override `hashCode` when you override `equals` ([Item 9](chapter-3.md#item-9-always-override-hashcode-when-you-override-equals)).
- Don’t try to be too clever.
- Don’t substitute another type for `Object` in the `equals` declaration (then you won't be overriding `equals` but will instead be overloading it).
- Use Google's open source AutoValue framework to automatically generate `equals` and `hashCode` methods. It tracks changes in the class automatically. Most IDEs have facilities to generate `equals` and `hashCode` methods that are more verbose and that do not track changes and therefore require testing.

## Item 11: Always override hashCode when you override equals
You must override `hashCode` in every class that overrides `equals`. Failure to do so will result in a violation of the general contract for `Object.hashCode`, which will prevent your class from functioning properly in conjunction with all hash-based collections, including `HashMap`, `HashSet`, and `Hashtable`.

Here's the `hashCode` contract: 
- When the hashCode method is invoked on an object repeatedly during an execution of an application, it must consistently return the same value, provided no information used in equals comparisons is modified. This value need not remain consistent from one execution of an application to another.
- If two objects are equal according to the equals(Object) method, then calling hashCode on the two objects must produce the same integer result.
- If two objects are unequal according to the equals(Object) method, it is not required that calling hashCode on each of the objects must produce distinct results. However, the programmer should be aware that producing distinct results for unequal objects may improve the performance of hash tables. 

A good hash function tends to produce unequal hash codes for unequal instances. Ideally, a hash function should distribute any reasonable collection of unequal instances uniformly across all int values. Achieving this ideal can be difficult. Luckily it’s not too hard to achieve a fair approximation. Here is a simple recipe:

1. Declare an int variable named result, and initialize it to the hash code c for the first significant field in your object, as computed in step 2.i. (Recall from Item 10 that a significant field is a field that affects equals comparisons.) Exclude any field that are not used in `equals` comparisons.
2. For every remaining significant field f in your object, do the following: 
    1. Compute an int hash code `c` for the field:
        1. If the field is of a primitive type, compute `Type.hashCode(f)`, where `Type` is the boxed primitive class corresponding to f’s type.
        2. If the field is an object reference and this class’s `equals` method compares the field by recursively invoking `equals`, recursively invoke `hashCode` on the field. If a more complex comparison is required, compute a “canonical representation” for this field and invoke `hashCode` on the canonical representation. If the value of the field is null, use 0 (or some other constant, but 0 is traditional).
        3. If the field is an array, treat it as if each significant element were a separate field. That is, compute a hash code for each significant element by applying these rules recursively, and combine the values per step 2.ii. If the array has no significant elements, use a constant, preferably not `0`. If all elements are significant, use `Arrays.hashCode`.
    2. Combine the hash code `c` computed in step 2.i into result as follows: `result = 31 * result + c;`. 
        1. This makes it so that `result` depends on the order of the fields, yielding a much better hash function if the class has multiple similar fields. If the multiplication is omitted, all anagrams would have identical hash codes.
        2. The value 31 was chosen because it is an odd prime. If it were even and the multiplication overflowed, information would be lost, because multiplication by 2 is equivalent to shifting. The advantage of using a prime is less clear, but it is traditional. A nice property of 31 is that the multiplication can be replaced by a shift and a subtraction for better performance on some architectures: 31 * i == (i << 5) - i. Modern VMs do this sort of optimization automatically.
3. Return `result`.

Example:
```java
// Typical hashCode method
@Override 
public int hashCode() {
    int result = Short.hashCode(areaCode);
    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);
    return result;
}
```

If you have a bona fide need for hash functions less likely to produce collisions, see Guava’s `com.google.common.hash.Hashing`.

Performance enhancement:
```java
// hashCode method with lazily initialized cached hash code
private int hashCode; // Automatically initialized to 0
@Override public int hashCode() {
    int result = hashCode;
    if (result == 0) {
        result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        hashCode = result;
    }
    return result;
}
```

## Item 12: Always override toString
Default `Object toString` method: class name followed by an @ sign and the unsigned hexadecimal representation of the hash code, e.g. PhoneNumber@163b91.

__Providing a good toString implementation makes your class
much more pleasant to use and makes systems using the class easier to debug__. The toString method is automatically invoked when an object is passed to println, printf, the string concatenation operator, or assert, or is printed by a debugger. Even if you never call toString on an object, others may. For example, a component that has a reference to your object may include the string representation of the object in a logged error message. If you fail to override toString, the message may be useless.

__When practical, the toString method should return all of the interesting information contained in the object__. Else you could get test failure reports like: `Assertion failure: expected {abc, 123}, but was {abc, 123}`. Well, shit.

__Provide programmatic access to the information contained in the value returned by toString__, i.e. via getters (and possibly setters).

To recap, override Object’s toString implementation in every instantiable class you write, unless a superclass has already done so. It makes classes much more pleasant to use and aids in debugging. The toString method should return a concise, useful description of the object, in an aesthetically pleasing format.


## Item 13: Override clone judiciously
The `Cloneable` interface was intended as a mixin interface (Item 20) for classes to advertise that they permit cloning. Unfortunately, it fails to serve this purpose. It lacks a `clone` method, and Object’s `clone` method is protected (accessible only to class, package, and subclasses).

- Cannot invoke `clone` without resorting to reflection (Item 65) 
    - Even a reflective invocation may fail, because there is no guarantee that the object has an accessible `clone` method
- Despite this flaw and many others, the facility is in reasonably wide use, so it pays to understand it. 
- This item tells you how to implement a well-behaved `clone` method, discusses when it is appropriate to do so, and presents alternatives.

If a class implements `Cloneable`, Object’s `clone` method returns a field-by-field copy of the object; otherwise it throws `CloneNotSupportedException`. This is a highly atypical use of interfaces and not one to be emulated. Normally, implementing an interface says something about what a class can do for its clients. In this case, it modifies the behavior of a protected method on a superclass.

__In practice, a class implementing `Cloneable` is expected to provide a properly functioning public `clone` method__.

__Immutable classes should never provide a clone method because it would
merely encourage wasteful copying__.

__In effect, the clone method functions as a constructor; you must ensure that it does no harm to the original object and that it properly establishes invariants on the clone__.

All classes that implement `Cloneable` should override `clone` with a public method whose return type is the class itself. This method should first call `super.clone` and then fix any fields that need to be fixed. Typically, this means copying any mutable objects that comprise the internal "deep structure" of the object being cloned, and replacing the clone's references to these objects with references to the copies (this is why, like serialization, __the Cloneable architecture is incompatible with normal use of final fields referring to mutable objects__, except in cases where the mutable objects may be safely shared between an object and its clone). 

While these internal copies can generally be made by calling `clone` recursively, this is not always the best approach. If the class contains only primitive fields or references to immutable objects, then it is probably the case that no fields need to be fixed. There are exceptions to this rule. For example, a field representing a serial number or other unique ID or a field representing the object’s creation time will need to be fixed/changed, even if it is primitive or immutable.

```java
// Clone method for class with no references to mutable state
@Override public PhoneNumber clone() {
    try {
        //Cast is guaranteed to succeed
        return (PhoneNumber) super.clone();
    } catch (CloneNotSupportedException e) {
        //a checked exception (checked for at compile time to be handled)
        throw new AssertionError(); // Can't happen
    }
}
```

```java
// Clone method for class with references to mutable state
@Override 
public Stack clone() {
    try {
        Stack result = (Stack) super.clone();
        result.elements = elements.clone(); //proper way to clone an array
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

__Public clone methods should omit the throws clause__, as methods that don’t throw checked exceptions are easier to use (Item 71).

Leave clone implementation up to subclasses. If you want to prevent subclasses from implementing a clone method:

```java
// clone method for extendable class not supporting Cloneable
@Override
protected final Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
}
```

__A better approach to object copying is to provide a copy constructor or copy factory__.

```java
// Copy constructor
public Yum(Yum yum) { ... };

// Copy factory
public static Yum newInstance(Yum yum) { ... };
```

Given all the problems associated with Cloneable, new interfaces should not extend it, and new extendable classes should not implement it. While it’s less harmful for final classes to implement Cloneable, this should be viewed as a performance optimization, reserved for the rare cases where it is justified (Item 67). As a rule, copy functionality is best provided by constructors or factories. A notable exception to this rule is arrays, which are best copied with the clone method.

## Item 14: Consider implementing Comparable
By implementing `Comparable`, a class indicates that its instances have a natural ordering. This makes sorting possible and searching efficient (think binary search).

If you are writing a value class with an obvious natural ordering, such as alphabetical order, numerical order, or chronological order, you should implement the Comparable interface:

```java
public interface Comparable<T> {
    int compareTo(T t);
}
```

In the following description, the notation `sgn(expression)` designates the mathematical signum function, which is defined to return -1, 0, or 1, according to whether the value of expression is negative, zero, or positive.

- The implementor must ensure `sgn(x.compareTo(y)) == -sgn(y.compareTo(x))` for all `x` and `y`. (This implies that `x.compareTo(y)` must throw an exception if and only if `y.compareTo(x)` throws an exception.)
- The implementor must also ensure that the relation is transitive: `(x.compareTo(y) > 0 && y.compareTo(z) > 0)` implies `x.compareTo(z) > 0`.
- Finally, the implementor must ensure that `x.compareTo(y) == 0` implies that `sgn(x.compareTo(z)) == sgn(y.compareTo(z))`, for all `z`.
- It is strongly recommended, but not strictly required, that `(x.compareTo(y) == 0) == (x.equals(y))`. Generally speaking, any class that implements the `Comparable` interface and violates this condition should clearly indicate this fact. The recommended language is "Note: This class has a natural ordering that is inconsistent with equals."
    - A class whose `compareTo` method imposes an order that is inconsistent with `equals` will still work, but sorted collections containing elements of the class may not obey the general contract of the appropriate collection interfaces (`Collection`, `Set`, or `Map`). This is because the general contracts for these interfaces are defined in terms of the `equals` method, but sorted collections use the equality test imposed by `compareTo` in place of `equals`. It is not a catastrophe if this happens, but it’s something to be aware of.
    - e.g. the `BigDecimal` class, whose `compareTo` method is inconsistent with `equals`. If add to a `HashSet` instance `new BigDecimal("1.0")` and `new BigDecimal("1.00")`, the set will contain two elements because the two `BigDecimal` instances added to the set are unequal when compared using the `equals` method. If you use `TreeSet` instead HashSet, the set will contain only one element because the two `BigDecimal` instances are equal when compared using the `compareTo` method. (See the `BigDecimal` documentation for details.)

- Same caveat as `equals` because of the properties of reflexivity, symmetry and trsntitivity: there is no way to extend an instantiable class with a new value component while preserving the `compareTo` contract, unless you are willing to forgo the benefits of object-oriented abstraction (Item 10). (i.e. How do you add the value component to the `compareTo` without accessing elements you're not allowed to?) The same workaround applies, too. If you want to add a value component to a class that implements `Comparable`, don’t extend it; write an unrelated class containing an instance of the first class. Then provide a "view" method that returns the contained instance. This frees you to implement whatever `compareTo` method you like on the containing class, while allowing its client to view an instance of the containing class as an instance of the contained class when needed.

- In a `compareTo` method, fields are compared for order rather than equality. To compare object reference fields, invoke the `compareTo` method recursively. If a field does not implement Comparable or you need a nonstandard ordering, use a `Comparator` instead. You can write your own comparator or use an existing one, as in this `compareTo` method for `CaseInsensitiveString` in Item 10:

```java
// Single-field Comparable with object reference field
public final class CaseInsensitiveString
    implements Comparable<CaseInsensitiveString> {
    public int compareTo(CaseInsensitiveString cis) {
        return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
    }
    ... // Remainder omitted
}
```

Note that `CaseInsensitiveString` implements `Comparable<CaseInsensitiveString>`. This means that a `CaseInsensitiveString` reference can be compared only to another `CaseInsensitiveString` reference. This is the normal pattern to follow when declaring a class to implement `Comparable`.

- In Java 7, static `compare` methods were added to all of Java’s boxed primitive classes. __Use of the relational operators < and > in `compareTo` methods is verbose and error-prone and no longer recommended__.

If a class has multiple significant fields, the order in which you compare them is critical. Start with the most significant field and work your way down. If a comparison results in anything other than zero (which represents equality), you’re done; just return the result. If the most significant field is equal, compare the next-most-significant field, and so on, until you find an unequal field or compare the least significant field. Here is a `compareTo` method for the `PhoneNumber` class in Item 11 demonstrating this technique:

```java
// Multiple-field Comparable with primitive fields
public int compareTo(PhoneNumber pn) {
    int result = Short.compare(areaCode, pn.areaCode);
    if (result == 0) {
        result = Short.compare(prefix, pn.prefix);
    if (result == 0)
        result = Short.compare(lineNum, pn.lineNum);
    }
    return result;
}
```

- In Java 8, the `Comparator` interface was outfitted with a set of comparator construction methods, which enable fluent construction of comparators. These comparators can then be used to implement a `compareTo` method, as required by the `Comparable` interface. Many programmers prefer the conciseness of this approach, though it does come at a modest performance cost: sorting arrays of `PhoneNumber` instances is about 10% slower on my machine. When using this approach, consider using Java’s static import facility so you can refer to static comparator construction methods by their simple names for clarity and brevity. Here’s how the `compareTo` method for `PhoneNumber` looks using this approach:

```java
// Comparable with comparator construction methods
private static final Comparator<PhoneNumber> COMPARATOR =
comparingInt((PhoneNumber pn) -> pn.areaCode)
.thenComparingInt(pn -> pn.prefix)
.thenComparingInt(pn -> pn.lineNum);
public int compareTo(PhoneNumber pn) {
return COMPARATOR.compare(this, pn);
}
```

This implementation builds a comparator at class initialization time, using two comparator construction methods. The first is comparingInt. It is a static method that takes a key extractor function that maps an object reference to a key of type `int` and returns a comparator that orders instances according to that key. In the previous example, `comparingInt` takes a lambda () that extracts the area code from a `PhoneNumber` and returns a `Comparator<PhoneNumber>` that orders phone numbers according to their area codes. Note that the lambda explicitly specifies the type of its input parameter (`PhoneNumber pn`). It turns out that in this situation, Java’s type inference isn’t powerful enough to figure the type out for itself, so we’re forced to help it in order to make the program compile.

If two phone numbers have the same area code, we need to further refine the comparison, and that’s exactly what the second comparator construction method, `thenComparingInt`, does. It is an instance method on `Comparator` that takes an `int` key extractor function, and returns a comparator that first applies the original comparator and then uses the extracted key to break ties. You can stack up as many calls to `thenComparingInt` as you like, resulting in a lexicographic ordering. In the example above, we stack up two calls to `thenComparingInt`, resulting in an ordering whose secondary key is the prefix and whose tertiary key is the line number.

There are also comparator construction methods for object reference types. The static method, named `comparing`, has two overloadings. One takes a key extractor and uses the keys' natural order. The second takes both a key extractor and a comparator to be used on the extracted keys. There are three overloadings of the instance method, which is named `thenComparing`. One overloading takes only a comparator and uses it to provide a secondary order. A second overloading takes only a key extractor and uses the key’s natural order as a secondary order. The final overloading takes both a key extractor and a comparator to be used on the extracted keys.

- Don't be clever:

```java
// BROKEN difference-based comparator - violates transitivity!
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return o1.hashCode() - o2.hashCode();
    }
};
```

Do not use this technique. It is fraught with danger from integer overflow and IEEE 754 floating point arithmetic artifacts [JLS 15.20.1, 15.21.1]. Furthermore, the resulting methods are unlikely to be significantly faster than those written using the techniques described in this item. Use either a static compare method:

```java
// Comparator based on static compare method
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return Integer.compare(o1.hashCode(), o2.hashCode());
    }
};
```

or a comparator construction method:

```java
// Comparator based on Comparator construction method
static Comparator<Object> hashCodeOrder =
    Comparator.comparingInt(o -> o.hashCode());
```
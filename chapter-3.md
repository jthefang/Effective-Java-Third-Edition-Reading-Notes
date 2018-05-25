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
Whether or not you specify the format, provide programmatic access to all of the information contained in the value returned by `toString`. For example, the `PhoneNumber` class should contain accessors for the area code, prefix, and line number. If you fail to do this, you force programmers who need this information to parse the string. Besides reducing performance and making unnecessary work for programmers, this process is error-prone and results in fragile systems that break if you change the format. By failing to provide accessors, you turn the string format into a de facto API, even if you’ve specified that it’s subject to change.

## Item 13: Override clone judiciously
All classes that implement `Cloneable` should override `clone` with a public method whose return type is the class itself. This method should first call `super.clone` and then fix any fields that need to be fixed. Typically, this means copying any mutable objects that comprise the internal “deep structure” of the object being cloned, and replacing the clone’s references to these objects with references to the copies. While these internal copies can generally be made by calling `clone` recursively, this is not always the best approach. If the class contains only primitive fields or references to immutable objects, then it is probably the case that no fields need to be fixed. There are exceptions to this rule. For example, a field representing a serial number or other unique ID or a field representing the object’s creation time will need to be fixed, even if it is primitive or immutable.

## Item 14: Consider implementing Comparable
In the following description, the notation `sgn(expression)` designates the mathematical signum function, which is defined to return -1, 0, or 1, according to whether the value of expression is negative, zero, or positive.

- The implementor must ensure `sgn(x.compareTo(y)) == -sgn(y.compareTo(x))` for all `x` and `y`. (This implies that `x.compareTo(y)` must throw an exception if and only if `y.compareTo(x)` throws an exception.)
- The implementor must also ensure that the relation is transitive: `(x.compareTo(y) > 0 && y.compareTo(z) > 0)` implies `x.compareTo(z) > 0`.
- Finally, the implementor must ensure that `x.compareTo(y) == 0` implies that `sgn(x.compareTo(z)) == sgn(y.compareTo(z))`, for all `z`.
- It is strongly recommended, but not strictly required, that `(x.compareTo(y) == 0) == (x.equals(y))`. Generally speaking, any class that implements the `Comparable` interface and violates this condition should clearly indicate this fact. The recommended language is “Note: This class has a natural ordering that is inconsistent with equals.”
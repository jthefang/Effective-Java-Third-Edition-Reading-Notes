# Chapter 12: Serialization

This chapter concerns object serialization, which is Java’s framework for encoding objects as byte streams (serializing) and reconstructing objects from their encodings (deserializing). Once an object has been serialized, its encoding can be sent from one VM to another or stored on disk for later deserialization. This chapter focuses on the dangers of serialization and how to minimize them.

## Item 85: Prefer alternatives to Java serialization

__Serialization is dangerous and should be avoided__. If you are designing a system from scratch, use a cross-platform structured-data representation such as JSON or protobuf instead. Do not deserialize untrusted data. If you must do so, use object deserialization filtering, but be aware that it is not guaranteed to thwart all attacks. Avoid writing serializable classes. If you must do so, exercise great caution.

- The security issues described in previous editions of this book turned out to be every bit as serious as some had feared. The vulnerabilities discussed in the early 2000s were transformed into serious exploits over the next decade, famously including a ransomware attack on the San Francisco Metropolitan Transit Agency Municipal Railway (SFMTA Muni) that shut down the entire fare collection system for two days in November 2016 [Gallagher16].

A fundamental problem with serialization is that its attack surface is too big to protect, and constantly growing: Object graphs are deserialized by invoking the `readObject` method on an `ObjectInputStream`. This method is essentially a magic constructor that can be made to instantiate objects of almost any type on the class path, so long as the type implements the Serializable interface. In the process of deserializing a byte stream, this method can execute code from any of these types, so the code for all of these types is part of the attack surface.

The attack surface includes classes in the Java platform libraries, in third-party libraries such as Apache Commons Collections, and in the application itself. Even if you adhere to all of the relevant best practices and succeed in writing serializable classes that are invulnerable to attack, your application may still be vulnerable.

- Deserialization of untrusted streams can result in remote code execution (RCE), denial-of-service (DoS), and a range of other exploits. Applications can be vulnerable to these attacks even if they did nothing wrong.

- Attackers and security researchers study the serializable types in the Java libraries and in commonly used third-party libraries, looking for methods invoked during deserialization that perform potentially dangerous activities. Such methods are known as gadgets. Multiple gadgets can be used in concert, to form a gadget chain. From time to time, a gadget chain is discovered that is sufficiently powerful to allow an attacker to execute arbitrary native code on the underlying hardware, given only the opportunity to submit a carefully crafted byte stream for deserialization. This is exactly what happened in the SFMTA Muni attack. This attack was not isolated. There have been others, and there will be more.

- Without using any gadgets, you can easily mount a denial-of-service attack by causing the deserialization of a short stream that requires a long time to deserialize. Such streams are known as deserialization bombs.

- __The best way to avoid serialization exploits is never to deserialize anything__. In the words of the computer named Joshua in the 1983 movie WarGames, “the only winning move is not to play.” __There is no reason to use Java serialization in any new system you write__. There are other mechanisms for translating between objects and byte sequences that avoid many of the dangers of Java serialization, while offering numerous advantages, such as cross-platform support, high performance, a large ecosystem of tools, and a broad community of expertise. In this book, we refer to these mechanisms as *cross-platform structured-data representations*. While others sometimes refer to them as serialization systems, this book avoids that usage to prevent confusion with Java serialization.
	- At the very least __never deserialize untrusted data__. 

- The leading cross-platform structured data representations are JSON [JSON] and Protocol Buffers, also known as protobuf [Protobuf]. JSON was designed by Douglas Crockford for browser-server communication, and protocol buffers were designed by Google for storing and interchanging structured data among its servers. Even though these representations are sometimes called language-neutral, JSON was originally developed for JavaScript and protobuf for C++; both representations retain vestiges of their origins.

| JSON 						| protobuf 				|
|---------------------------|-----------------------|
|- text based, human readable
- extremely efficient for a text-based representation
- data representation exclusively 
| - binary, substantially more efficient
- offers schemas (types) to document and enforce appropriate usage
- pbtxt is an alt. text representation for human readability use |

- If you must using Java serialization, e.g. for legacy systems:
	- use object deserialization filtering (`java.io.ObjectInputFilter`) which lets you specify a filter that is applied to data streams before they're deserialized. It operates at the class granularity, letting you accept or reject certain classes. Accepting classes by default and rejecting a list of potentially dangerous ones is known as blacklisting; rejecting classes by default and accepting a list of those that are presumed safe is known as whitelisting. Prefer whitelisting to blacklisting, as blacklisting only protects you against known threats. A tool called Serial Whitelist Application Trainer (SWAT) can be used to automatically prepare a whitelist for your application [Schneider16]. The filtering facility will also protect you against excessive memory usage, and excessively deep object graphs, but it will not protect you against serialization bombs like the one shown above.

## Item 86: Implement `Serializable` with great caution

The ease of implementing `Serializable` is specious. Unless a class is to be used only in a protected environment where versions will never have to interoperate and servers will never be exposed to untrusted data, implementing `Serializable` is a serious commitment that should be made with great care. Extra caution is warranted if a class is designed for inheritance. For such classes, an intermediate design point between implementing `Serializable` and prohibiting it in subclasses is to provide an accessible parameterless constructor. This design point permits, but does not require, subclasses to implement `Serializable`.

- __A major cost of implementing Serializable is that it decreases the flexibility to change a class’s implementation once it has been released__. Its byte stream encoding (or serialized form) becomes part of its exported API. If you don't make the effor to design a custom serialized form but merely accept the default, the serialized form will forever be tied to the class's original internal representation: the classes's private and package-private instance fields become part of its exported API, and the practice of minimizing access to fields (Item 15) loses its effectiveness as a tool for information hiding.
	- If you accept the default serialized form and later change a class’s internal representation, an incompatible change in the serialized form will result. Clients attempting to serialize an instance using an old version of the class and deserialize it using the new one (or vice versa) will experience program failures. It is possible to change the internal representation while maintaining the original serialized form (using `ObjectOutputStream.putFields` and `ObjectInputStream.readFields`), but it can be difficult and leaves visible warts in the source code. If you opt to make a class serializable, you should carefully design a high-quality serialized form that you’re willing to live with for the long haul (Items 87, 90). Doing so will add to the initial cost of development, but it’s worth the effort. Even a well-designed serialized form places constraints on the evolution of a class; an ill-designed serialized form can be crippling.
	- e.g. the constraints on evolution imposed by serializability concerns stream unique identifiers, more commonly known as serial version UIDs. Every serializable class has a unique identification number associated with it. If you do not specify this number by declaring a static final long field named serialVersionUID, the system automatically generates it at runtime by applying a cryptographic hash function (SHA-1) to the structure of the class. This value is affected by the names of the class, the interfaces it implements, and most of its members, including synthetic members generated by the compiler. If you change any of these things, for example, by adding a convenience method, the generated serial version UID changes. If you fail to declare a serial version UID, compatibility will be broken, resulting in an `InvalidClassException` at runtime.

- __A second cost of implementing `Serializable` is that it increases the likelihood of bugs and security holes (Item 85)__. Normally, objects are created with constructors; serialization is an extralinguistic mechanism for creating objects. Whether you accept the default behavior or override it, deserialization is a “hidden constructor” with all of the same issues as other constructors. Because there is no explicit constructor associated with deserialization, it is easy to forget that you must ensure that it guarantees all of the invariants established by the constructors and that it does not allow an attacker to gain access to the internals of the object under construction. Relying on the default deserialization mechanism can easily leave objects open to invariant corruption and illegal access (Item 88).

- __A third cost of implementing `Serializable` is that it increases the testing burden associated with releasing a new version of a class__. When a serializable class is revised, it is important to check that it is possible to serialize an instance in the new release and deserialize it in old releases, and vice versa. The amount of testing required is thus proportional to the product of the number of serializable classes and the number of releases, which can be large. You must ensure both that the serialization-deserialization process succeeds and that it results in a faithful replica of the original object. The need for testing is reduced if a custom serialized form is carefully designed when the class is first written (Items 87, 90).

- Implementing `Serializable` is not a decision to be undertaken lightly. It is essential if a class is to participate in a framework that relies on Java serialization for object transmission or persistence. Also, it greatly eases the use of a class as a component in another class that must implement Serializable. There are, however, many costs associated with implementing Serializable. Historically, value classes such as `BigInteger` and `Instant` implemented `Serializable`, and collection classes did too. Classes representing active entities, such as thread pools, should rarely implement `Serializable`.

- __Classes designed for inheritance (Item 19) should rarely implement
Serializable, and interfaces should rarely extend it__.

- __Inner classes (Item 24) should not implement `Serializable`__. They use compiler-generated synthetic fields to store references to enclosing instances and to store values of local variables from enclosing scopes. How these fields correspond to the class definition is unspecified, as are the names of anonymous and local classes. Therefore, the default serialized form of an inner class is ill-defined. A static member class can, however, implement `Serializable`.

## Item 87: Consider using a custom serialized form

- __Don't accept the default serialized form without first consdiering whether it's appropriate__. Serialization implementations will generally be inflexible to future updates, especially the default form. Generally speaking, you should accept the default serialized form only if it is largely identical to the encoding that you would choose if you were designing a custom serialized form.
	- The default serialized form of an object is a reasonably efficient encoding of the *physical representation* of the object graph rooted at the object. In other words, it describes the data contained in the object and in every object that is reachable from this object. It also describes the topology by which all of these objects are interlinked. The ideal serialized form of an object contains *only* the logical data represented by the object. It is independent of the physical representation.
	- __The default serialized form is likely to be appropriate if an object’s physical representation is identical to its logical content__ (i.e. *how* the data is represented is identical to *what* it represents) For example, the default serialized form would be reasonable for the following class, which simplistically represents a person’s name:

```java
// Good candidate for default serialized form
public class Name implements Serializable {
	/**
	* Last name. Must be non-null.
	* @serial
	*/
	private final String lastName;
	
	/**
	* First name. Must be non-null.

	* @serial
	*/
	private final String firstName;
	
	/**
	* Middle name, or null if there is none.
	* @serial
	*/
	private final String middleName;

	... // Remainder omitted
}
```

Logically speaking, a name consists of three strings that represent a last name, a first name, and a middle name. The instance fields in `Name` precisely mirror this logical content.

- Even if you decide that the default serialized form is appropriate, you often must provide a `readObject` method to ensure invariants and security. In the case of `Name`, the `readObject` method must ensure that the fields `lastName` and `firstName` are non-null. This issue is discussed at length in Items 88 and 90.

Near the opposite end of the spectrum from Name, consider the following class, which represents a list of strings (ignoring for the moment that you would probably be better off using one of the standard List implementations):
```java
// Awful candidate for default serialized form
public final class StringList implements Serializable {
	private int size = 0;
	private Entry head = null;

	private static class Entry implements Serializable {
		String data;
		Entry next;
		Entry previous;
	}

	... // Remainder omitted
}
```

Logically speaking, this class represents a sequence of strings. Physically, it represents the sequence as a doubly linked list. If you accept the default serialized form, the serialized form will painstakingly mirror every entry in the linked list and all the links between the entries, in both directions.

If using the default serializable form, you must document all fields even though they are private. That is because these private fields define a public API, which is the serialized form of the class, and this public API must be documented. The presence of the @serial tag tells Javadoc to place this documentation on a special page that documents serialized forms.

- __Using the default serialized form when an object’s physical representation differs substantially from its logical data content has four disadvantages__:
	- __It permanently ties the exported API to the current internal representation__
	- __It can consume excessive space__
	- __It can consume excessive time__. The serialization logic has no knowledge of the topology of the object graph, so it must go through an expensive graph traversal. In the example above, it would be sufficient simply to follow the `next` references.
	- __It can cause stack overflows__. The default serialization procedure performs a recursive traversal of the object graph, which can cause stack overflows even for moderately sized object graphs. 

- A reasonable serialized form for `StringList` is simply the number of strings in the list, followed by the strings themselves. This constitutes the logical data represented by a `StringList`, stripped of the details of its physical representation. Here is a revised version of `StringList` with `writeObject` and `readObject` methods that implement this serialized form. As a reminder, the `transient` modifier indicates that an instance field is to be omitted from a class’s default serialized form:
```java
// StringList with a reasonable custom serialized form
public final class StringList implements Serializable {
	private transient int size = 0;
	private transient Entry head = null;

	// No longer Serializable!
	private static class Entry {
		String data;
		Entry next;
		Entry previous;
	}

	// Appends the specified string to the list
	public final void add(String s) { ... }
	
	/**
	* Serialize this {@code StringList} instance.
	*
	* @serialData The size of the list (the number of strings
	* it contains) is emitted ({@code int}), followed by all of
	* its elements (each a {@code String}), in the proper
	* sequence.
	*/
	private void writeObject(ObjectOutputStream s)
			throws IOException {
		s.defaultWriteObject();
		s.writeInt(size);

		// Write out all elements in the proper order.
		for (Entry e = head; e != null; e = e.next)
			s.writeObject(e.data);
	}

	private void readObject(ObjectInputStream s)
			throws IOException, ClassNotFoundException {
		s.defaultReadObject();
		int numElements = s.readInt();

		// Read in all elements and insert them in list
		for (int i = 0; i < numElements; i++)
			add((String) s.readObject());
	}

	... // Remainder omitted
}
```
The first thing `writeObject` does is to invoke `defaultWriteObject`, and the first thing `readObject` does is to invoke `defaultReadObject`, even though all of `StringList`’s fields are transient. The presence of these calls makes it possible to add nontransient instance fields in a later release while preserving backward and forward compatibility. If an instance is serialized in a later version and deserialized in an earlier version, the added fields will be ignored. Had the earlier version’s `readObject` method failed to invoke `defaultReadObject`, the deserialization would fail with a `StreamCorruptedException`.

Note that there is a documentation comment on the `writeObject` method, even though it is private. This is analogous to the documentation comment on the private fields in the Name class. This private method defines a public API, which is the serialized form, and that public API should be documented. Like the `@serial` tag for fields, the `@serialData` tag for methods tells the Javadoc utility to place this documentation on the serialized forms page.

- Every instance field that isn’t labeled transient will be serialized when the `defaultWriteObject` method is invoked. __Therefore, every instance field that can be declared transient should be. Before deciding to make a field nontransient, convince yourself that its value is part of the logical state of the object__. If you use a custom serialized form, most or all of the instance fields should be labeled transient, as in the `StringList` example above.

- If you are using the default serialized form and you have labeled one or more fields transient, remember that these fields will be initialized to their default values when an instance is deserialized: null for object reference fields, zero for numeric primitive fields, and false for boolean fields [JLS, 4.12.5]. If these values are unacceptable for any transient fields, you must provide a `readObject` method that invokes the `defaultReadObject` method and then restores transient fields to acceptable values (Item 88). Alternatively, these fields can be lazily initialized the first time they are used (Item 83).

- __You must impose any synchronization on object serialization that you would impose on any other method that reads the entire state of the object__. So, for example, if you have a thread-safe object (Item 82) that achieves its thread safety by synchronizing every method and you elect to use the default serialized form, use the following `writeObject` method:
```java
// writeObject for synchronized class with default serialized form
private synchronized void writeObject(ObjectOutputStream s)
	throws IOException {
	s.defaultWriteObject();
}
```

If you put synchronization in the writeObject method, you must ensure that it adheres to the same lock-ordering constraints as other activities, or you risk a resource-ordering deadlock [Goetz06, 10.1.5].

- __Regardless of what serialized form you choose, declare an explicit serial version UID in every serializable class you write__. This eliminates the serial version UID as a potential source of incompatibility (Item 86). There is also a small performance benefit. If no serial version UID is provided, an expensive computation is performed to generate one at runtime.
```java
private static final long serialVersionUID = randomLongValue;
```

The value can be anything. You can generate the value by running the `serialver` utility on the class, but it’s also fine to pick a number out of thin air. It is not required that serial version UIDs be unique. If you modify an existing class that lacks a serial version UID, and you want the new version to accept existing serialized instances, you must use the value that was automatically generated for the old version. You can get this number by running the `serialver` utility on the old version of the class—the one for which serialized instances exist.
	- __Do not change the serial version UID unless you want to break compatibility with all existing serialized instances of a class__.

If you have decided that a class should be serializable ([Item 86](chapter-12.md#item-86-implement-serializable-with-great-caution)), think hard about what the serialized form should be. Use the default serialized form only if it is a reasonable description of the logical state of the object; otherwise design a custom serialized form that aptly describes the object. You should allocate as much time to designing the serialized form of a class as you allocate to designing its exported methods ([Item 51](chapter-08.md#item-51-design-method-signatures-carefully)). Just as you cannot eliminate exported methods from future versions, you cannot eliminate fields from the serialized form; they must be preserved forever to ensure serialization compatibility. Choosing the wrong serialized form can have a permanent, negative impact on the complexity and performance of a class.

## Item 88: Write readObject methods defensively

The `readObject` method is effectively another public constructor, and it demands all of the same care as any other constructor. Just as a constructor must check its arguments for validity (Item 49) and make defensive copies of parameters where appropriate (Item 50), so must a `readObject` method. If a `readObject` method fails to do either of these things, it is a relatively simple matter for an attacker to violate the class’s invariants.
	- Loosely speaking, `readObject` is a constructor that takes a byte stream as its sole parameter. In normal use, the byte stream is generated by serializing a normally constructed instance. The problem arises when `readObject` is presented with a byte stream that is artificially constructed to generate an object that violates the invariants of its class. Such a byte stream can be used to create an impossible object, which could not have been created using a normal constructor.
	- Provide a `readObject` method for `Period` that calls `defaultReadObject` and then checks the validity of the deserialized object. If the validity check fails, the `readObject` method throws `InvalidObjectException`, preventing the deserialization from completing:
```java
// readObject method with validity checking - insufficient!
private void readObject(ObjectInputStream s)
		throws IOException, ClassNotFoundException {
	s.defaultReadObject();

	// Check that our invariants are satisfied
	if (start.compareTo(end) > 0)
		throw new InvalidObjectException(start +" after "+ end);
}
```

__When an object is deserialized, it is critical to defensively copy any field containing an object reference that a client must not possess__. Therefore, every serializable immutable class containing private mutable components must defensively copy these components in its `readObject` method. The following `readObject` method suffices to ensure `Period`’s invariants and to maintain its immutability:
```java
// readObject method with defensive copying and validity checking
private void readObject(ObjectInputStream s)
		throws IOException, ClassNotFoundException {
	s.defaultReadObject();

	// Defensively copy our mutable components
	start = new Date(start.getTime());
	end = new Date(end.getTime());
	
	// Check that our invariants are satisfied
	if (start.compareTo(end) > 0)
		throw new InvalidObjectException(start +" after "+ end);
}
```

Note that the defensive copy is performed prior to the validity check and that we did not use `Date`’s `clone` method to perform the defensive copy. Both of these details are required to protect `Period` against attack (Item 50). Note also that defensive copying is not possible for final fields. To use the `readObject` method, we must make the `start` and `end` fields nonfinal. This is unfortunate, but it is the lesser of two evils.

- Like a constructor, a `readObject` method must not invoke an overridable method, either directly or indirectly (Item 19). If this rule is violated and the method in question is overridden, the overriding method will run before the subclass’s state has been deserialized. A program failure is likely to result.

Anytime you write a readObject method, adopt the mind-set that you are writing a public constructor that must produce a valid instance regardless of what byte stream it is given. Do not assume that the byte stream represents an actual serialized instance. While the examples in this item concern a class that uses the default serialized form, all of the issues that were raised apply equally to classes with custom serialized forms.

Here, in summary form, are the guidelines for writing a `readObject` method:
- For classes with object reference fields that must remain private, defensively copy each object in such a field. Mutable components of immutable classes fall into this category.
- Check any invariants and throw an `InvalidObjectException` if a check fails. The checks should follow any defensive copying.
- If an entire object graph must be validated after it is deserialized, use the `ObjectInputValidation` interface (not discussed in this book).
- Do not invoke any overridable methods in the class, directly or indirectly.

## Item 89: For instance control, prefer enum types to readResolve

- Any `readObject` method, whether explicit or default, returns a newly created instance, which will not be the same instance that was created at class initialization time. This is why a singleton class cannot just `implements Serializable`.

- The `readResolve` feature allows you to substitute another instance for the one created by `readObject` [Serialization, 3.7]. If the class of an object being deserialized defines a `readResolve` method with the proper declaration, this method is invoked on the newly created object after it is deserialized. The object reference returned by this method is then returned in place of the newly created object. In most uses of this feature, no reference to the newly created object is retained, so it immediately becomes eligible for garbage collection.

- If the `Elvis` class (from Item 3) is made to implement `Serializable`, the following `readResolve` method suffices to guarantee the singleton property:

```java
// readResolve for instance control - you can do better!
private Object readResolve() {
	// Return the one true Elvis and let the garbage collector
	// take care of the Elvis impersonator.
	return INSTANCE;
}
```

This method ignores the deserialized object, returning the distinguished `Elvis` instance that was created when the class was initialized. Therefore, the serialized form of an `Elvis` instance need not contain any real data; all instance fields should be declared transient. In fact, if you depend on `readResolve` for instance control, all instance fields with object reference types must be declared transient. Otherwise, it is possible for a determined attacker to secure a reference to the deserialized object before its `readResolve` method is run, using a technique that is somewhat similar to the `MutablePeriod` attack in Item 88.

- The accessibility of `readResolve` is significant. If you place a `readResolve` method on a final class, it should be private. If you place a `readResolve` method on a nonfinal class, you must carefully consider its accessibility. If it is private, it will not apply to any subclasses. If it is package-private, it will apply only to subclasses in the same package. If it is protected or public, it will apply to all subclasses that do not override it. If a `readResolve` method is protected or public and a subclass does not override it, deserializing a subclass instance will produce a superclass instance, which is likely to cause a `ClassCastException`.

Use enum types to enforce instance control invariants wherever possible. If this is not possible and you need a class to be both serializable and instance-controlled, you must provide a `readResolve` method and ensure that all of the class’s instance fields are either primitive or transient.

## Item 90: Consider serialization proxies instead of serialized instances

The serialization proxy pattern is reasonably straightforward. 
1. Design a private static nested class that concisely represents the logical state of an instance of the enclosing class. This nested class is known as the serialization proxy of the enclosing class. It should have a single constructor, whose parameter type is the enclosing class. This constructor merely copies the data from its argument: it need not do any consistency checking or defensive copying. By design, the default serialized form of the serialization proxy is the perfect serialized form of the enclosing class. Both the enclosing class and its serialization proxy must be declared to implement `Serializable`.

```java
// Serialization proxy for Period class
private static class SerializationProxy implements Serializable {
	private final Date start;
	private final Date end;
	
	SerializationProxy(Period p) {
		this.start = p.start;
		this.end = p.end;
	}

	private static final long serialVersionUID =
		234098243823485285L; // Any number will do (Item 87)
}
```

2. Next, add the following `writeReplace` method to the enclosing class. This method can be copied verbatim into any class with a serialization proxy:

```java
// writeReplace method for the serialization proxy pattern
private Object writeReplace() {
	return new SerializationProxy(this);
}
```

The presence of this method on the enclosing class causes the serialization system to emit a `SerializationProxy` instance instead of an instance of the enclosing class. In other words, the `writeReplace` method translates an instance of the enclosing class to its serialization proxy prior to serialization.

3. With this `writeReplace` method in place, the serialization system will never generate a serialized instance of the enclosing class, but an attacker might fabricate one in an attempt to violate the class’s invariants. To guarantee that such an attack would fail, merely add this `readObject` method to the enclosing class:

```java
// readObject method for the serialization proxy pattern
private void readObject(ObjectInputStream stream)
		throws InvalidObjectException {
	throw new InvalidObjectException("Proxy required");
}
```

4. Finally, provide a `readResolve` method on the `SerializationProxy` class that returns a logically equivalent instance of the enclosing class. The presence of this method causes the serialization system to translate the serialization proxy back into an instance of the enclosing class upon deserialization. Here is the readResolve method for Period.SerializationProxy above:

```java
// readResolve method for Period.SerializationProxy
private Object readResolve() {
	return new Period(start, end); // Uses public constructor
}
```

- The serialization proxy pattern has two limitations. It is not compatible with classes that are extendable by their users (Item 19). Also, it is not compatible with some classes whose object graphs contain circularities: if you attempt to invoke a method on such an object from within its serialization proxy’s `readResolve` method, you’ll get a `ClassCastException` because you don’t have the object yet, only its serialization proxy.

-  The added power and safety of the serialization proxy pattern are not free. On my machine, it is 14 percent more expensive to serialize and deserialize `Period` instances with serialization proxies than it is with defensive copying.

Consider the serialization proxy pattern whenever you find yourself having to write a `readObject` or `writeObject` method on a class that is not extendable by its clients. This pattern is perhaps the easiest way to robustly serialize objects with nontrivial invariants.
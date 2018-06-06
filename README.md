# Effective-Java-Third-Edition-Reading-Notes
My notes on Effective Java, Third Edition

## [Chapter 2: Creating and Destroying Objects](chapter-2.md)

- [Item 1: Consider static factory methods instead of constructors](chapter-2.md#item-1-consider-static-factory-methods-instead-of-constructors)
- [Item 2: Consider a builder when faced with many constructor parameters](chapter-2.md#item-2-consider-a-builder-when-faced-with-many-constructor-parameters)
- [Item 3: Enforce the singleton property with a private constructor or an enum type](chapter-2.md#item-3-enforce-the-singleton-property-with-a-private-constructor-or-an-enum-type)
- [Item 4: Enforce noninstantiability with a private constructor](chapter-2.md#item-4-enforce-noninstantiability-with-a-private-constructor)
- [Item 5: Prefer dependency injection to hardwiring resources](chapter-2.md#item-5-prefer-dependency-injection-to-hardwiring-resources)
- [Item 6: Avoid creating unnecessary objects](chapter-2.md#item-6-avoid-creating-unnecessary-objects)
- [Item 7: Eliminate obsolete object references ](chapter-2.md#item-7-eliminate-obsolete-object-references )
- [Item 8: Avoid finalizers and cleaners](chapter-2.md#item-8-avoid-finalizers-and-cleaners)
- [Item 9: Prefer try-with-resources to try-finally](chapter-2.md#item-9-prefer-try-with-resources-to-try-finally)

## [Chapter 3: Methods Common to All Objects](chapter-3.md)

- [Item 10: Obey the general contract when overrriding equals](chapter-3.md#item-10-obey-the-general-contract-when-overriding-equals)
- [Item 11: Always override hashCode when you override equals](chapter-3.md#item-11-always-override-hashcode-when-you-override-equals)
- [Item 12: Always override toString](chapter-3.md#item-12-always-override-tostring)
- [Item 13: Override clone judiciously](chapter-3.md#item-13-override-clone-judiciously)
- [Item 14: Consider implementing Comparable](chapter-3.md#item-14-consider-implementing-comparable)

## [Chapter 4: Classes and Interfaces](chapter-4.md)

- [Item 15: Minimize the accessibility of classes and members](chapter-4.md#item-15-minimize-the-accessibility-of-classes-and-members)
- [Item 16: In public classes, use accessor methods, not public fields](chapter-4.md#item-16-in-public-classes-use-accessor-methods-not-public-fields)
- [Item 17: Minimize mutability](chapter-4.md#item-17-minimize-mutability)
- [Item 18: Favor composition over inheritance](chapter-4.md#item-18-favor-composition-over-inheritance)
- [Item 19: Design and document for inheritance or else prohibit it](chapter-4.md#item-19-design-and-document-for-inheritance-or-else-prohibit-it)
- [Item 20: Prefer interfaces to abstract classes](chapter-4.md#item-20-prefer-interfaces-to-abstract-classes)
- [Item 21: Design interfaces for posterity](chapter-4.md#item-21-design-interfaces-for-posterity)
- [Item 22: Use interfaces only to define types](chapter-4.md#item-22-use-interfaces-only-to-define-types)
- [Item 23: Prefer class hierarchies to tagged classes](chapter-4.md#item-23-prefer-class-hierarchies-to-tagged-classes)
- [Item 24: Favor static member classes over nonstatic](chapter-4.md#item-24-favor-static-member-classes-over-nonstatic)
- [Item 25: Limit source files to a single top-level class](chapter-4.md#item-25-limit-source-files-to-a-single-top-level-class)

## [Chapter 5: Generics](chapter-5.md)

- [Item 26: Don’t use raw types in new code](chapter-5.md#item-26-dont-use-raw-types-in-new-code)
- [Item 27: Eliminate unchecked warnings](chapter-5.md#item-27-eliminate-unchecked-warnings)
- [Item 28: Prefer lists to arrays](chapter-5.md#item-28-prefer-lists-to-arrays)
- [Item 29: Favor generic types](chapter-5.md#item-29-favor-generic-types)
- [Item 30: Favor generic methods](chapter-5.md#item-30-favor-generic-methods)
- [Item 31: Use bounded wildcards to increase API flexibility](chapter-5.md#item-31-use-bounded-wildcards-to-increase-api-flexibility)
- [Item 32: Combine generics and varargs judiciously](chapter-5.md#item-32-combine-generics-and-varargs-judiciously)
- [Item 33: Consider type safe heterogeneous containers](chapter-5.md#item-33-consider-type-safe-heterogeneous-containers)

## [Chapter 6: Enums and Annotations](chapter-6.md)

- [Item 34: Use enums instead of int constants](chapter-6.md#item-34-use-enums-instead-of-int-constants)
- [Item 35: Use instance fields instead of ordinals](chapter-6.md#item-35-use-instance-fields-instead-of-ordinals)
- [Item 36: Use EnumSet instead of bit fields](chapter-6.md#item-36-use-enumset-instead-of-bit-fields)
- [Item 37: Use EnumMap instead of ordinal indexing](chapter-6.md#item-37-use-enummap-instead-of-ordinal-indexing)
- [Item 38: Emulate extensible enums with interfaces](chapter-6.md#item-38-emulate-extensible-enums-with-interfaces)
- [Item 39: Prefer annotations to naming patterns](chapter-6.md#item-39-prefer-annotations-to-naming-patterns)
- [Item 40: Consistently use the Override annotation](chapter-6.md#item-40-consistently-use-the-override-annotation)
- [Item 41: Use marker interfaces to define types](chapter-6.md#item-41-use-marker-interfaces-to-define-types)
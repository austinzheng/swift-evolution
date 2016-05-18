# Enhanced Existential Support

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Authors: [Matthew Johnson](mailto:matthew@anandabits.com), TBD, Austin Zheng
* Status: **[Awaiting review](#rationale)**
* Review manager: TBD

## Introduction

*NOTE TO DRAFT READERS*: Much of this proposal assumes that a smaller proposal, which simply renames `protocol<>` to `Any<>`, is accepted for Swift 3. If it isn't, this proposal takes on the task of proposing that change as well.

Swift's support for existential types is currently quite limited: one or more protocols can be composed using the `Any<>` syntax. We propose, in the spirit of [*Completing Generics*](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md), to add support to Swift for describing more complex existential types, including those involving protocols with associated types.

## Motivation

There are a variety of useful types which cannot be expressed currently within Swift's type system. Here are two examples:

1. **Types involving protocols with associated types, and constraints upon those associated types**.

	It's possible to express the notion of "an `Array` containing `Int`s": 

	```
	let a : [Int]
	```

	It's also possible to express the notion of "a `Set` containing `Int`s":

	```
	let a : Set<Int>
	```

	But it's not possible to express the notion of "any `Collection` whose elements are `Int`s":

	```
	// Not valid, can't use `Collection` as a protocol type
	let a : Collection
	// How would we even express the 'elements are Ints' requirement?
	```

2. **Types involving classes with additional requirements**.

	Objective-C programmers are familiar with the following construct, which is ubiquitous within Cocoa and Cocoa Touch programming:

	```
	// Objective-C
	// A UIViewController or subclass thereof, which conforms to both protocols
	@property (nonatomic, strong) UIViewController<ASZProtocol1, ASZProtocol2> *myViewController;
	```

	However, this sort of type is impossible to express in Swift without resorting to generic programming.

## Proposed solution

An improved version of the `Any<...>` construct will serve as the primary method for declaring an existential type.

Within the angle brackets `<` and `>` are zero or more *requirements*. Requirements are separated by commas.

`Any<>` will be typealiased or special-cased to `Any`, the existential capable of containing any other type. For consistency, `Any<...>` with one class requirement or protocol requirement clause is exactly equal to just that bare class or protocol. `P` can be used in lieu of `Any<P>`, where `P` is a protocol with or without associated type or self requirements.

The following requirements are valid. Detailed descriptions follow.

* **Any-class requirement**. This must be the first requirement, if present. This requirement consists of the keyword `class`, and requires the existential to be of any class type.

* **Class requirement**. This must be the first requirement, if present. This requirement consists of the name of a class type, and requires the existential to be of a specific class or its subclasses.

* **Protocol requirement**. This requirement consists of the name of a protocol, and requires the existential to conform to the protocol.

* **Protocol requirement with `where`**. This requirement consists of the name of a protocol with associated types, followed by the keyword `where`, followed by a list of constraints. The existential must conform to the protocol, and the associated types of the protocol must comply with the given `where` constraints.

* **Bound protocol requirement**. This requirement consists of the name of a protocol with associated types, followed by the keyword `as`, followed by an identifier.

* **Bound protocol requirement with `where`**. This reqirement consists, in order, of the name of a protocol with associated types, `as`, an identifier, `where`, and a list of constraints.

* **Nested `Any<...>`**. This requirement consists of another `Any<...>` construct.

A summary of the `Any<...>` grammar can be found in the *Detailed Design* section.

**Simple `Any<...>`**

A 'simpler' version of the `Any<...>` construct is also defined, hereby referred to as **simple `Any<...>`**. This is identical to the full `Any<...>`, but can only contain any-class requirements, class requirements, protocol requirements, and nested simple `Any<...>` requirements. Use of simple `Any<...>` is detailed in later sections.


### Any-class requirement

Places a constraint on the existential to be any class type. This requirement must be the first one, if present. There can be only one any-class requirement, and it is mutually exclusive with class requirements.

Example:

```
// Can be any class type that conforms to SomeProtocol.
let a : Any<class, SomeProtocol>
```

### Class requirement

Places a constraint on the existential to be an instance of the named class, or any subclass thereof. This requirement must be the first one, if present. There can be only one class name constraint, and it is mutually exclusive with the any-class requirement.

Example:

```
// Can be any class type that is a UIView or a subclass of UIView,
// that also conforms to SomeProtocol.
let a : Any<UIView, SomeProtocol>
```

### Protocol requirement

Places a constraint on the existential to be a type which conforms to the named protocol. If the protocol has associated types, the associated types remain unconstrained.

Example:

```
// Can be any type that conforms to both Equatable and Streamable.
let a : Any<Equatable, Streamable>
```

### Protocol requirement with `where`

Places a constraint on the existential to be a type which conforms to the named protocol. The protocol must have one or more associated types. The `where` clause contains one or more constraints. Constraints must involve the associated types of the named protocol. `Self` cannot be constrained.

The acceptable constraints following the `where` clause are identical to those following the `where` clause on a constrained extension or generic function/type definition:

* Type equality constraint: `X == ConcreteType`
* Type conformance constraint: `X : SomeProtocol`
* Type inheritance constraint: `X : SomeClass`
* Composite constraint: `X : SomeSimpleAny`

Example:

```
// Can be any Collection whose elements are Ints.
let a : Any<Collection where Collection.Element == Int>

// Can be any Collection whose elements are Streamable; the Collection itself 
// must also be Streamable.
let b : Any<Streamable, Collection where Collection.Element == Streamable> 
```

Associated types used within the `where` clause constraints must belong to the protocols in the current or previous requirements. For example, this is wrong:

```
// NOT ALLOWED
// SetAlgebraType wasn't used in a protocol requirement anywhere in the
// existential, so we can't use its ElementType.
let a : Any<Collection where Collection.Element == SetAlgebraType.Element>
```

### Bound protocol requirement

Places a constraint on the existential to be a type which conforms to the named protocol. The protocol is bound to the identifier following the `as` keyword, and can be used in any subsequent requirements.

Example:

```
// Can be any type that is both a collection and an algebraic set, where the
// collection's elements and the algebraic set's elements are modeled as the
// same type.
let a : Any<Collection as T, SetAlgebraType where SetAlgebraType.Element == T.Element>
```

### Bound protocol requirement with `where`

This is a straightforward combination of the previous two requirements. An example follows:

```
let a : Any<Collection as T where Collection.Element : Streamable, SetAlgebraType where SetAlgebraType.Element == T.Element>
```

### Nested `Any<...>`

Places a constraint on the existential to meet all constraints imposed by the nested `Any<...>` construct.

Example:
```
// Can be any type that conforms to ProtocolA, ProtocolB, and ProtocolC.
let a : Any<ProtocolA, Any<ProtocolB, ProtocolC>>
```

A nested `Any<...>` may declare a class or any-class constraint, even if its parent `Any<...>` contains a class or any-class constraint, or a sibling `Any<...>` constraint declares a class or any-class constraint. However, one of the classes must be a subclass of every other class declared within such constraints, or all the classes must be the same. This class, if it exists, is chosen as the `Any<...>` construct's constraint.

The motivation for this feature, besides the principle of least surprise, is to allow generic typealiases to be composed using the `Any<...>` construct. See *Future Directions*, below.

Examples:

```
// Can be any type that is a UITableView conforming to ProtocolA.
// UITableView is the most specific class, and it is a subclass of the other
// two classes.
let a : Any<UIScrollView, Any<UITableView, Any<UIView, ProtocolA>>>

// NOT ALLOWED: no class is a subclass of all the other classes. This cannot be
// satisfied by any type.
let b : Any<NSObject, Any<UIView, Any<NSData, ProtocolA>>>
```

Likewise, a nested `Any<...>` may declare a protocol constraint involving a protocol that has already been constrained by the parent or a sibling `Any<...>` clause. However, if a protocol is constrained in such a way that an associated type has two contradictory same-type constraints or subclass constraints, this is an error:

```
// NOT ALLOWED
// This is impossible to fufill. A collection's elements cannot be both strings
// and integers at the same time.
let a : Any<Collection where Collection.Element == String, Any<Collection where Collection.Element == Int>>
``` 

This requirement cannot be the only requirement in an `Any<...>` construct:

```
// NOT ALLOWED
let a : Any<Any<ProtocolA, ProtocolB>>
```

Any `Any<...>` containing nested `Any<...>`s can be conceptually 'flattened' and written as an equivalent `Any<...>` containing no nested `Any<...>` requirements. The two representations are exactly interchangeable.

## Detailed design

### Usage

An `Any<...>` existential type can be trivially used in any of the following situations:

* Local variable type
* Property type
* Function parameter
* Function return value

Existentials cannot be used in the following ways:

* As part of the superclass/protocol conformance portion of a type declaration.

#### Dynamic Casting

Existential use as the predicate of a type checking expression (`as` case, `as?`, `as!`), is limited to only simple `Any<...>`.

Expanded use of existentials ('full `Any<...>`') in runtime type checking falls under the 'opening existentials' follow-up feature.

#### Generics

Existentials can be used in generic declarations in one of several ways:

* As a parameter value or return value for a generic function:

	```
	func myFunc<T>(x: T, y: T) -> Any<Collection where Collection.Element == T> { ... }
	```

* As the type of a property in a generic type:

	```
	class MyClass<T> {
		var collection : Any<Collection where Collection.Element == T> = []
	}
	```

* As the object of a constraint which counts as one or both of 'conforms to' and 'subclass of' in `where` clauses:

	```
	func myFunc<T where T : Any<class, FooProtocol, BarProtocol>>(x: T) { ... }
	```

	In this case I strongly suggest existential support should be limited, like in the dynamic casting case above, to simple `Any<...>`. This does *not* represent a loss of expressive capability. It does head off a potential issue where coding styles diverge, C++-style, as some shops retain the existing generic syntax and others define generics in terms of elaborate, deeply nested full `Any<...>` clauses.

	One counterargument: complex conditionals can be broken up using `Any<...>` and generic typealiases, possibly making such code read more easily.

	Even if we do think that full `Any<...>` support is desirable, I still think it is a good idea to limit the scope of this proposal to using simple `Any<...>` here, and allow a follow-up proposal to introduce pervasive full `Any<...>` support (see *Future Directions* below).

### Flattening

Any `Any<...>` existential type can be conceptually flattened into the following information:

* **Zero or one classes, or just any class.** This is done by taking all any-class and class requirements defined in the `Any<...>` construct or any nested `Any<...>`s and finding the single class which is a subclass of all the other classes, or just that class if all class constraints are the same.

* **A list of zero or more protocols.** This is done by creating a list of all protocols from protocol constraints defined in the `Any<...>` construct or any nested `Any<...>`s, and discarding any protocols which serve as parents to other protocols in this list.

* **A list of all associated types from the above protocols, and constraints on those associated types.** If a requirement involving a parent protocol has a `where` clause, and a requirement involving a child protocol has a `where` clause, the child protocol is retained and the constraints from both `where` clauses are merged.

This provides a mental model for reasoning about how the various requirements in an existential should work together.

### Associated types

It should be possible to invoke methods (etc) on a protocol with associated types which has been declared as part of an `Any<...>` existential. As stated above, 'flattening' the existential should produce a list of all associated types involved with the existential, as well as any constraints on those associated types.

Each associated type may be defined as one of the following:

* No constraints were placed on the associated type, so that type is equal to whatever requirements the associated type is defined with, or `Any` if none exist.

* Inheritance or conformance constraints were placed on the associated type, so that associated type is equal to an existential combining these constraints.

* The associated type was bound to a concrete type with a type equality constraint.

Here is a sample protocol, to be used with later examples:

```
protocol FooProtocol {
	associatedtype AssocA : Streamable
	associatedtype AssocB

	func first(x: Int)
	func second(x: AssocB) -> AssocA
	func third() -> AssocB
	func fourth() -> (AssocB, Int)
}
```

There are a couple of aspects to this feature:

1. The simplest case are protocol methods which do not use the associated types at all. These should always be accessible.

	Example:

	```
	let a : Any<FooProtocol>
	// Can call first(), because it uses no associated types.
	a.first(10)
	```

2. If a protocol method uses associated types, and all the associated types have been bound to concrete types using equality constraints, that protocol method should be accessible.

	Example:

	```
	let a : Any<FooProtocol where FooProtocol.AssocA == MyStreamer, FooProtocol.AssocB == Int>
	// Can call second(), since all the associated types used in that method are known.
	let returnValue : MyStreamer = a.second(5)
	```

3. If a protocol method uses associated types, and all associated types are bound except for the return type (which is an associated type), and the return type is 'bare' or a 'bare optional' (e.g. not part of a different composite type, like a tuple), then that protocol method should be accessible, and the return value should be bound to either `Any` or whatever constraints the associated type was defined with.

	Examples:

	```
	let a : Any<FooProtocol where FooProtocol.AssocB == Int>
	// Can call second(), since the only unbound associated type in that method is 
	// AssocA, which is used as a return type.
	let r1 : Streamable = a.second(10)

	let b : Any<FooProtocol>
	// Can call third(), for the same reason.
	// This would have worked even if the function had this signature:
	// "func third() -> AssocB?"
	// This is because Optional is covariant via compiler magic.
	let r2 : Any = b.third()

	// NOT ALLOWED, for now
	// Swift has no general notion of user-defined variance
	let r3 : (Any, Int) =  b.fourth()
	```

Anything more sophisticated almost certainly requires covariance and contravariance annotations. (For example, simply assuming that input parameters can be treated as covariant can lead to incorrect code.) Refining requirements on which associated type protocol methods are exposed to an existential type can go into a follow-up proposal, because such enhancements would be additive in nature.

### Repeated Protocols

In a single `Any<...>` construct, a given protocol can only show up in one non-nested-`Any<...>` requirement. For example, the following is illegal:

```
// Not accepted, because Collection shows up in two requirements.
let a : Any<Collection, Collection where Collection.Element == Int>
```

However, a protocol is allowed to show up in a requirement, even if a protocol it inherits from is present in another requirement:

```
// Okay, but redundant, because all Hashables are Equatable.
let a : Any<Equatable, Hashable>
```

In such a case, it may be desirable for the compiler to provide a warning.

### Grammar

This grammar is preliminary, and is provided only as a guide to understanding the structure of the new constructs described above. I regret any errors.

*any-existential* → `Any<` *requirement-list*<sub>opt</sub> `>`

*simple-existential* → `Any<` *simple-requirement-list*<sub>opt</sub> `>`

*requirement-list* → *requirement* | *requirement*, *requirement-list*

*simple-requirement-list* → *simple-requirement* | *simple-requirement*, *simple-requirement-list*

*requirement* → *any-class-requirement* | *class-requirement* | *protocol-requirement* | *protocol-where-requirement* | *protocol-bind-requirement* | *protocol-bind-where-requirement* | *any-existential*

*simple-requirement* → *any-class-requirement* | *class-requirement* | *protocol-requirement* | *simple-existential*

*any-class-requirement* → `class`

*class-requirement* → `type-name` (name of a class)

*protocol-requirement* → `type-name` (name of a protocol)

*requirement-clause* (see *The Swift Programming Language*)

*as-clause* → `as` *type-identifier*

*protocol-where-requirement* → *protocol-requirement* *requirement-clause*

*protocol-bind-requirement* → *protocol-requirement* *as-clause*

*protocol-bind-where-requirement* → *protocol-requirement* *as-clause* *requirement-clause*

## Future Directions

In addition to adding capabilities to Swift directly, this proposal lays the groundwork for other proposals which fit into the Completing Generics constellation.

### Shortcut dot-notation

This proposal requires the constraints in a `where` clause to spell out the name of the associated type's primary type:

```
let a : Any<Collection where Collection.Element == Int>
```

It should be possible, at some point, to allow the name of the primary type to be omitted when writing out such constraints:

```
let a : Any<Collection where .Element == Int>
```

The exact semantics of this feature can be narrowed down and discussed thoroughly in a separate proposal. (Note that such shortcut notation can become confusing if there are multiple types who have similarily-named associated types in play.) This is an additive feature, sugar for convenience, so its omission from this proposal should be harmless.

### Generic typealiases

It may be possible in the future to use generic typealiases to allow for the composition of existential types. This is a powerful feature which aids writing modular code and cleanly defined interfaces. The design of this feature accounts for this possibility:

```
// Any custom UIViewController that serves as a delegate for a table view.
typealias CustomTableViewController = Any<UIViewController, UITableViewDataSource, UITableViewDelegate>

// Pull in the previous typealias and add more refined constraints.
typealias PiedPiperResultsViewController = Any<CustomTableViewController, PiedPiperListViewController, PiedPiperDecompressorDelegate>
```

### Opening existentials

As discussed previously, it is challenging to properly expose protocol methods with associated types when those types aren't bound. A way to get around this is to allow existentials to be manually opened.

```
// Taken from 'Completing Generics'
if let storedInE1 = e1 openas T {
  // is e2 also a T? 
  if let storedInE2 = e2 as? T {
    // okay: storedInT1 and storedInE2 are both of type T, which we know is Equatable
    if storedInE1 == storedInE2 { ... }
  }
}
```

Here is an example, as it pertains to working with associated types.

```
// (see FooProtocol, from earlier)

// No information on FooProtocol's associated types at compile time
let a : Any<FooProtocol>

// ...

if let openedA = a openas _ where FooProtocol.AssocA == MyStreamer, FooProtocol.AssocB == Bool {
	// Now we know what types AssocA and AssocB are, and can use second()
	let x : MyStreamer = openedA.second(true)
}
```

### Deprecating 'simple `Any<...>`'

If the community feels strongly, it may be worth deprecating 'simple `Any<...>`' in favor of use of full `Any<...>` everywhere. This is an additive change; existing code won't be broken but expressions that aren't accepted with this proposal will become valid.

One use case is breaking complex sub-constraints into more legible typealiases. For example:

```
// A collection containing equatable collections; these collections in turn
// contain Ints.
let a : Any<Collection where Collection.Element : Equatable, Collection.Element : Collection, Collection.Element.Element == Int>

// becomes...
typealias IntCollection = Any<Collection where Collection.Element == Int>

let b : Any<Collection where Collection.Element : Any<IntCollection, Equatatable>>
```

Note that use of full `Any<...>` in a constraint clause doesn't really share the same semantics as existentials; it is more of a self-contained nested constraint within a larger constraint.


### Additional kind constraints

Currently, Swift supports only the generic `class` constraint, for specifying "any class". Additional constraints, such as `struct`, `enum`, and/or `value`, might be proposed in the future. If so, retrofitting them to work with this proposal should be conceptually simple: one can imagine an 'any-struct' counterpart to the existing 'any-class' requirement.

## Impact on existing code

No impact. Greenfield feature.

## Alternatives considered

Don't add in enhanced existential support. Force Swift users to continue using generics if they want to work with protocols containing associated types.

-------------------------------------------------------------------------------

# Rationale

On [Date], the core team decided to **(TBD)** this proposal.
When the core team makes a decision regarding this proposal,
their rationale for the decision will be written here.

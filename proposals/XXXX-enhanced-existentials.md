# Enhanced Existential Support

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Authors: [Matthew Johnson](mailto:matthew@anandabits.com), Thorsten Seitz, David Smith, Adrian Zubarev, TBD, Austin Zheng
* Status: **[Awaiting review](#rationale)**
* Review manager: TBD

## Introduction

*NOTE TO DRAFT READERS*: Much of this proposal assumes that a smaller proposal, which simply renames `protocol<>` to `Any<>`, is accepted for Swift 3. If it isn't, this proposal takes on the task of proposing that change as well. It also assumes the existence of typealiases in protocols with associated types, including `Collection.Iterator.Element` being aliased to just `Collection.Element`.

Swift's support for existential types is currently quite limited: one or more protocols can be composed using the `Any<>` syntax. We propose, in the spirit of [*Completing Generics*](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md), to add support to Swift for describing more complex existential types, including those involving protocols with associated types.

## Motivation

There are a variety of useful types which cannot be expressed currently within Swift's type system. Here are two examples:

1. **Types involving protocols with associated types, and constraints upon those associated types**.

	It's possible to express the notion of "an `Array` containing `Int`s": 

	```swift
	let a : [Int]
	```

	It's also possible to express the notion of "a `Set` containing `Int`s":

	```swift
	let a : Set<Int>
	```

	But it's not possible to express the notion of "any `Collection` whose elements are `Int`s":

	```swift
	// Not valid, can't use `Collection` as a protocol type
	let a : Collection
	// How would we even express the 'elements are Ints' requirement?
	```

2. **Types involving classes with additional requirements**.

	Objective-C programmers are familiar with the following construct, which is ubiquitous within Cocoa and Cocoa Touch programming:

	```objective-c
	// Objective-C
	// A UIViewController or subclass thereof, which conforms to both protocols
	@property (nonatomic, strong) UIViewController<ASZProtocol1, ASZProtocol2> *myViewController;
	```

	However, this sort of type is impossible to express in Swift without resorting to generic programming.

## Proposed solution

An improved version of the `Any<...>` construct will serve as the primary method for declaring an existential type.

Within the angle brackets `<` and `>` are zero or more *requirements*. Requirements are separated by commas. After the requirements is an optional `where` clause, in which constraints are placed upon any associated types introduced by previous requirements.

`Any<>` will be typealiased or special-cased to `Any`, the existential capable of containing any other type. For consistency, `Any<...>` with one class requirement or protocol requirement clause is exactly equal to just that bare class or protocol. `P` can be used in lieu of `Any<P>`, where `P` is a protocol with or without associated type or self requirements.

The following requirements are valid. Detailed descriptions follow.

* **Any-class requirement**. This must be the first requirement, if present. This requirement consists of the keyword `class`, and requires the existential to be of any class type.

* **Class requirement**. This must be the first requirement, if present. This requirement consists of the name of a class type, and requires the existential to be of a specific class or its subclasses.

* **Protocol requirement**. This requirement consists of the name of a protocol, and requires the existential to conform to the protocol.

* **Nested `Any<...>`**. This requirement consists of another `Any<...>` construct.

A summary of the `Any<...>` grammar can be found in the *Detailed Design* section.

### Any-class requirement

Places a constraint on the existential to be any class type. This requirement must be the first one, if present. There can be only one any-class requirement, and it is mutually exclusive with class requirements.

Example:

```swift
// Can be any class type that conforms to SomeProtocol.
let a : Any<class, SomeProtocol>
```

### Class requirement

Places a constraint on the existential to be an instance of the named class, or any subclass thereof. This requirement must be the first one, if present. There can be only one class name constraint, and it is mutually exclusive with the any-class requirement.

Example:

```swift
// Can be any class type that is a UIView or a subclass of UIView,
// that also conforms to SomeProtocol.
let a : Any<UIView, SomeProtocol>
```

### Protocol requirement

Places a constraint on the existential to be a type which conforms to the named protocol. If the protocol has associated types, the associated types remain unconstrained.

Example:

```swift
// Can be any type that conforms to both Equatable and Streamable.
let a : Any<Equatable, Streamable>
```

### Nested `Any<...>`

Places a constraint on the existential to meet all constraints imposed by the nested `Any<...>` construct.

Example:
```swift
// Can be any type that conforms to ProtocolA, ProtocolB, and ProtocolC.
let a : Any<ProtocolA, Any<ProtocolB, ProtocolC>>
```

A nested `Any<...>` may declare a class or any-class constraint, even if its parent `Any<...>` contains a class or any-class constraint, or a sibling `Any<...>` constraint declares a class or any-class constraint. However, one of the classes must be a subclass of every other class declared within such constraints, or all the classes must be the same. This class, if it exists, is chosen as the `Any<...>` construct's constraint.

Examples:

```swift
// Can be any type that is a UITableView conforming to ProtocolA.
// UITableView is the most specific class, and it is a subclass of the other
// two classes.
let a : Any<UIScrollView, Any<UITableView, Any<UIView, ProtocolA>>>

// Allowed, but pointless.
// Identical to Any<ProtocolA, ProtocolB>
let b : Any<Any<ProtocolA, ProtocolB>>

// NOT ALLOWED: no class is a subclass of all the other classes. This cannot be
// satisfied by any type.
let c : Any<NSObject, Any<UIView, Any<NSData, ProtocolA>>>
```

Likewise, a nested `Any<...>` may declare a protocol constraint involving a protocol that has already been constrained by the parent or a sibling `Any<...>` clause. However, if a protocol is constrained in such a way that an associated type has two contradictory same-type constraints or subclass constraints, this is an error:

```swift
// NOT ALLOWED
// This is impossible to fufill. A collection's elements cannot be both strings
// and integers at the same time.
let a : Any<Collection where Collection.Element == String, Any<Collection where Collection.Element == Int>>
```

Using typealiases, it is possible for `Any<...>` existentials to be composed. This allows API requirements to be expressed in a more natural, modular way:

```swift
// Any custom UIViewController that serves as a delegate for a table view.
typealias CustomTableViewController = Any<UIViewController, UITableViewDataSource, UITableViewDelegate>

// Pull in the previous typealias and add more refined constraints.
typealias PiedPiperResultsViewController = Any<CustomTableViewController, PiedPiperListViewController, PiedPiperDecompressorDelegate>
```

Any `Any<...>` containing nested `Any<...>`s can be conceptually 'flattened' and written as an equivalent `Any<...>` containing no nested `Any<...>` requirements. The two representations are exactly interchangeable.

### `where` clause

An `Any<...>` existential can have zero or one `where` clauses, following the list of requirements.

The `where` clause contains one or more constraints. Constraints must involve the associated types of the previously required protocols. `Self` cannot be constrained.

The acceptable constraints following the `where` clause are identical to those following the `where` clause on a constrained extension or generic function/type definition. In each case `X` must be the name of an associated type defined on one of the `Any<...>` existential's component protocols:

* Type equality constraint: `X == ConcreteType`
* Type conformance constraint: `X : SomeProtocol`
* Type inheritance constraint: `X : SomeClass`
* Composite constraint: `X : Any<...>`

Example:

```swift
// Can be any Collection whose elements are Ints.
let a : Any<Collection where Collection.Element == Int>

// Can be any Collection whose elements are Streamable; the Collection itself 
// must also be Streamable.
let b : Any<Streamable, Collection where Collection.Element == Streamable> 
```

Associated types used within the `where` clause must belong to the protocols in the current or previous requirements. For example, this is wrong:

```swift
// NOT ALLOWED
// SetAlgebraType wasn't used in a protocol requirement anywhere in the
// existential, so we can't use its ElementType.
let a : Any<Collection where Collection.Element == SetAlgebraType.Element>
```

**Shortcut 'dot' notation**: If there is only one protocol with associated types specified in the requirements, and there are no nested `Any<...>` requirements with `where` clauses of their own, that protocol's name can be omitted from the `where` clause constraints:

```swift
// Okay
// Would otherwise be Any<... where Collection.Element == Int>
let a : Any<class, Collection, Any<Streamable, CustomStringConvertible> where .Element == Int>

// NOT ALLOWED
// Both Collection and OptionSetType have associated types.
let b : Any<Collection, OptionSetType where .Element == Int>
```

## Detailed design

### Usage

An `Any<...>` existential type can be trivially used in any of the following situations:

* Local variable type
* Property type
* Function parameter
* Function return value
* Dynamic casting (the object of `as?`, an `as` case, `as!`, etc)

Existentials cannot be used in the following ways:

* As part of the superclass/protocol conformance portion of a type declaration.

#### Generics

Existentials can be used in generic declarations in one of several ways:

* As a parameter value or return value for a generic function:

	```swift
	func myFunc<T>(x: T, y: T) -> Any<Collection where Collection.Element == T> { ... }
	```

* As the type of a property in a generic type:

	```swift
	class MyClass<T> {
		var collection : Any<Collection where Collection.Element == T> = []
	}
	```

* As the object of a constraint in a generic declaration's `where` clause:

	```swift
	func myFunc<T where T : Any<class, FooProtocol, BarProtocol>>(x: T) { ... }
	```

	One use case is breaking complex sub-constraints into more legible typealiases. For example:

	```swift
	// A collection containing collections that have customized descriptions;
	// these collections in turn contain Ints.
	let a : Any<Collection where .Element : CustomStringConvertible, .Element : Collection, .Element.Element == Int>

	// becomes...
	typealias IntCollection = Any<Collection where .Element == Int>

	let b : Any<Collection where .Element : Any<IntCollection, CustomStringConvertible>>
	```

### Flattening

Any `Any<...>` existential type can be conceptually flattened into the following information:

* **Zero or one classes, or just any class.** This is done by taking all any-class and class requirements defined in the `Any<...>` construct or any nested `Any<...>`s and finding the single class which is a subclass of all the other classes, or just that class if all class constraints are the same.

* **A list of zero or more protocols.** This is done by creating a list of all protocols from protocol constraints defined in the `Any<...>` construct or any nested `Any<...>`s, and discarding any protocols which serve as parents to other protocols in this list.

* **A list of all associated types from the above protocols, and constraints on those associated types.** Constraints from all `where` clauses (whether in the `Any<...>` or any nested `Any<...>`s) are coalesced. Constraints on a parent protocol's associated types are 'given' to the child protocol when the redundant parent protocol is discarded.

This provides a mental model for reasoning about how the various requirements in an existential should work together, and how existentials can be ordered from 'narrower' to 'wider'. Any two existentials which flatten to the same three sets of information above should be considered equivalent, no matter how they were represented in code (i.e. ordering of requirements, how requirements are nested).

Here is a (highly contrived) example:

```swift
let a : Any<UIView, 
  Any<class, Equatable>,
  Any<Hashable, Sequence where Sequence.Element : FooProtocol>,
  Any<UITableView, BarProtocol, BazProtocol>,
  Collection
  where Collection.Element : BazProtocol
  >
```

becomes:

* Class requirement: `UITableView`
* Protocols: `Hashable`, `FooProtocol`, `BarProtocol`, `BazProtocol`, `Collection`
* Associated types: `Collection.Element` (conforms to `FooProtocol`, conforms to `BazProtocol`)

Note that `class` and `UIView` are discarded as redundant class requirements, `Equatable` and `Sequence` are discarded as redundant protocol requirements, and `Sequence`'s associated type constraints are given to `Collection`.


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

*NOTE: Core team feedback would be appreciated for the entire proposal, but this section is in particular need of a type system engineer's expertise.*

1. The simplest case are protocol methods which do not use the associated types at all. These should always be accessible.

	Example:

	```swift
	let a : Any<FooProtocol>
	// Can call first(), because it uses no associated types.
	a.first(10)
	```

2. If a protocol method uses associated types, and all the associated types have been bound to concrete types using equality constraints, that protocol method should be accessible.

	Example:

	```swift
	let a : Any<FooProtocol where FooProtocol.AssocA == MyStreamer, FooProtocol.AssocB == Int>
	// Can call second(), since all the associated types used in that method are known.
	let returnValue : MyStreamer = a.second(5)
	```

3. If a protocol method uses associated types, and all associated types are bound except for the return type (which is an associated type), and the return type is 'bare' or a 'bare optional' (e.g. not part of a different composite type, like a tuple), then that protocol method should be accessible, and the return value should be bound to either `Any` or whatever constraints the associated type was defined with.

	Examples:

	```swift
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

In addition to guarantees provided by the existential at the point of definition, the dynamic `as?` cast should also be able to narrow an existential at a use site:

```swift
// No information on FooProtocol's associated types at compile time
let a : Any<FooProtocol>

// ...

if let narrowedA = a as? Any<FooProtocol where .AssocA == MyStreamer, .AssocB == Bool> {
	// Now we know what types AssocA and AssocB are, and can use second()
	let x : MyStreamer = narrowedA.second(true)
}
```

Note that `as?` casts which are guaranteed to fail and can be deduced as so at compile-time are prohibited:

```swift
let a : Any<FooProtocol where .AssocB == Int>

// NOT ALLOWED; the compiler can see that the requirements are mutually
// exclusive at runtime and will error.
if let narrowedA = a as? Any<FooProtocol where .AssocB == String> {
	// ...
}
```

Narrowing does not preserve relationships that were in the original type but aren't expressed in the casted-to type, as per expected type-casting semantics.

```swift
let a : Any<FooProtocol>

// This is fine
a.someFooProtocolMethod()

if let narrowedA = a as? Any<BazProtocol> {
	// Can no longer call a FooProtocol method on narrowedA:
	narrowedA.someFooProtocolMethod() // DOES NOT WORK
	narrowedA.someBazProtocolMethod() // fine
}

if let narrowedA = as as? Any<FooProtocol, BazProtocol> {
	// Can now do both.
	narrowedA.someFooProtocolMethod() // fine
	narrowedA.someBazProtocolMethod() // fine
}
```

Anything more sophisticated almost certainly requires covariance and contravariance annotations. (For example, simply assuming that input parameters can be treated as covariant can lead to incorrect code.) Refining requirements on which associated type protocol methods are exposed to an existential type can go into a follow-up proposal, because such enhancements would be additive in nature.

### Repeated Protocols

In a single `Any<...>` construct, a given protocol can only show up in one non-nested-`Any<...>` requirement. For example, the following is illegal:

```swift
// Not accepted, because Collection shows up in two requirements.
let a : Any<Collection, Collection where Collection.Element == Int>
```

However, a protocol is allowed to show up in a requirement, even if a protocol it inherits from is present in another requirement:

```swift
// Okay, but redundant, because all Hashables are Equatable.
let a : Any<Equatable, Hashable>
```

In such a case, it may be desirable for the compiler to provide a warning.

### Grammar

This grammar is preliminary, and is provided only as a guide to understanding the structure of the new constructs described above. I regret any errors.

*any-existential* → `Any<` *requirement-list*<sub>opt</sub> `>`

*requirement-list* → *base-requirement-list* *where-clause*<sub>opt</sub>

*base-requirement-list* → *requirement* | *requirement* `,` *base-requirement-list*

*requirement* → *any-class-requirement* | *class-requirement* | *protocol-requirement* | *any-existential*

*any-class-requirement* → `class`

*class-requirement* → `type-name` (name of a class)

*protocol-requirement* → `type-name` (name of a protocol)

*where-clause* → *requirement-clause* (see *The Swift Programming Language*)

## Future Directions

In addition to adding capabilities to Swift directly, this proposal lays the groundwork for other proposals which fit into the Completing Generics constellation.

### Opening existentials

The *Associated Types* section above discusses runtime narrowing of existential types using `as?` and similar keywords. However, `as?` casting only operates on a single existential type in isolation; it does not allow a user to take two different existential types and compare their `Self` values.

An example illustrates this problem. Two different existential types are both `Equatable`. Since they are both `Equatable`, it's *possible* that their values might be comparable using `==`. However, this is only true if the underlying concrete types of the values are identical: 

```swift
struct Foo : Equatable, Protocol1, Protocol2 { ... }

let a : Any<Equatable, Protocol1> = ... // assigned a Foo
let b : Any<Equatable, Protocol2> = ... // assigned a Foo as well

// Is a's concrete type equal to b's concrete type?
// If so, a == b is valid
// If not, a == b is meaningless
```

There is no way to decide at compile time whether `a` and `b` contain values of the same concrete type in all cases; this checking must be deferred to runtime. What is required is a way to 'open' an existential and bind its underlying concrete type to a type variable. That type variable can then be subsequently used in conjunction with dynamic casts to determine whether a different existential also shares that same underlying type. If so, the compiler can allow subsequent code to assume both values are the same type.

One possible way of accomplishing this is by defining an `openas` operator, analogous to `as?`. The following example is taken from *Completing Generics*. (Note that, even though the `openas` operator is invoked in an `if let` statement, it should unconditionally succeed.)

```swift
let e1: Equatable = ...
let e2: Equatable = ...

if let storedInE1 = e1 openas T {
  // is e2 also a T? 
  if let storedInE2 = e2 as? T {
    // okay: storedInT1 and storedInE2 are both of type T, which we know is Equatable
    if storedInE1 == storedInE2 { ... }
  }
}
```

### Additional 'kind' (value/reference/etc) constraints

Currently, Swift supports only the generic `class` constraint, for specifying "any class". Additional constraints, such as `struct`, `enum`, and/or `value`, might be proposed in the future. If so, retrofitting them to work with this proposal should be conceptually simple: for example, one can imagine an 'any-struct' counterpart to the existing 'any-class' requirement.

## Impact on existing code

No impact. Greenfield feature.

## Alternatives considered

Alternatives here are discussed in terms of deprecated features which were originally part of the proposal.

### Separate `where` clauses

An earlier version of the proposal allowed individual requirements to each have their own `where` clauses. While this might have improved legibility in certain cases by placing constraints on associated types closer to the protocols they involved, it also made the feature more complex and difficult to understand.

### Binding protocols to identifiers

An earlier version of the proposal allowed requirements to bind a protocol name to an identifier. This was intended as a way to allow users to specify local short names for use in `where` clauses. However, the usefulness of this feature is debatable (is it really clearer to alias `Collection` to `T`?), and it provides no increased expressive capability.

### Simple version of `Any<...>`

A simple version of `Any<...>` lacking `where` clauses was proposed for use in generic type declarations and dynamic type casing expressions. Feedback indicated that this was overly limiting.

-------------------------------------------------------------------------------

# Rationale

On [Date], the core team decided to **(TBD)** this proposal.
When the core team makes a decision regarding this proposal,
their rationale for the decision will be written here.

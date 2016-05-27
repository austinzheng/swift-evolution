# Enhanced Existential Support

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Authors: Douglas Gregor, Austin Zheng, et al.
* Status: **[Awaiting review](#rationale)**
* Review manager: TBD

## Introduction

Swift's support for existential types is currently quite limited: one or more protocols can be composed using the `Any<>` syntax. We propose, in the spirit of [*Completing Generics*](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md), to add support to Swift for describing more complex existential types, including those involving protocols with associated types.

## Acknowledgements

Many thanks to [Matthew Johnson](https://github.com/anandabits), Thorsten Seitz, [David Smith](https://twitter.com/Catfish_Man), [Joe Groff](https://twitter.com/jckarter), [Adrian Zubarev](https://github.com/DevAndArtist), and TBD, whose feedback and contributions were instrumental in composing this proposal.

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

Within the angle brackets `<` and `>` are one or more *requirements*. Requirements are separated by commas. After the requirements is an optional `where` clause, in which constraints are placed upon any associated types introduced by previous requirements.

`Any` will be defined as equivalent to the (illegal) form `Any<>`, the existential capable of containing any other type.

For consistency, `Any<...>` with exactly one protocol requirement is exactly equal to just that bare protocol. `P` can be used in lieu of `Any<P>`, where `P` is a protocol with or without associated type or self requirements.

`Any<...>` with an any-class requirement or a class requirement, but no protocol requirements, is prohibited and will result in a compiler warning to use `AnyObject` or that class directly, respectively. For example, `Any<Any<NSView>>`, `Any<NSObject, Any<NSView>>`, and `Any<NSView>` are all prohibited. (*Rationale*: `Any<...>` denotes an existential type, which cannot be used in the same ways as a concrete type, but an `Any<NSView>` would be a synonym for `NSView`, which isn't an existential. Hence we'd have a type that looked like an existential, but actually wasn't, which would only confuse people without any real benefit.)

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

An existential that declares a requirement which is a `class`-only protocol can also declare the any-class requirement, although doing so is redundant:

```swift
protocol A : class { }

// Acceptable, but exactly the same as Any<A>, or just A
let a : Any<class, A>
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

Places a constraint on the existential to be a type which conforms to the named protocol. Both protocols that have associated types or self requirements, and those that don't, are acceptable.

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

A nested `Any<...>` may declare a class or any-class constraint, even if its parent `Any<...>` contains a class or any-class constraint, or a sibling `Any<...>` constraint declares a class or any-class constraint. However, one of the classes must be a subclass of every other class declared within such constraints, or all the classes must be the same. This class, if it exists, is chosen as the `Any<...>` construct's constraint. (See *Existential Type Equivalence*, below.)

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

However, recursive (self-referential) nested typealiases are not allowed.

An `Any<...>`'s `where` clause may reference protocols defined within nested `Any<...>` requirements, but the reverse is not true: an `Any<...>`'s `where` clause cannot reference protocols in the parent `Any<...>`.

Any `Any<...>` containing nested `Any<...>`s can be conceptually 'flattened' and written as an equivalent `Any<...>` containing no nested `Any<...>` requirements. The two representations are exactly interchangeable.

### `where` clause

An `Any<...>` existential can optionally have a single `where` clause. If this `where` clause exists, it comes after all the requirements.

(Note that an existential can have a `where` clause, but it might also have nested `Any<...>` requirements that have their own `where` clauses. This is allowed.)

The `where` clause contains one or more constraints. Constraints must involve the associated types of the previously required protocols. `Self` cannot be constrained.

The acceptable constraints following the `where` clause are identical to those following the `where` clause on a constrained extension or generic function/type definition. In each case `X` must be the name of an associated type defined on one of the `Any<...>` existential's component protocols:

* Type equality constraint: `X == ConcreteType`
* Type equality constraint: `X == AnotherAssociatedType`
* Type conformance constraint: `X : SomeProtocol`
* Type inheritance constraint: `X : SomeClass`
* Composite constraint: `X : Any<...>`

Example:

```swift
// Can be any Collection whose elements are Ints.
let a : Any<Collection where Collection.Element == Int>

// Can be any Collection whose elements are Streamable; the Collection itself 
// must also be Streamable.
let b : Any<Streamable, Collection where Collection.Element : Streamable> 
```

Associated types used within the `where` clause must belong to one of the protocols previously declared in a requirement. For example, this is wrong:

```swift
// NOT ALLOWED
// RawRepresentable wasn't used in a protocol requirement anywhere in the
// existential, so we can't use its RawValue. We can only use the associated
// types of 'Collection', since that's the only protocol the existential is
// required to conform to.
let a : Any<Collection where Collection.Element == RawRepresentable.RawValue>
```

**Shortcut 'dot' notation**: If there is only one protocol with associated types specified in the requirements, and there are no nested `Any<...>` requirements with `where` clauses of their own, that protocol's name can be omitted from the `where` clause constraints:

```swift
// Okay
// Would otherwise be Any< ~ where Collection.Element == Int>
let a : Any<class, Collection, Any<Streamable, CustomStringConvertible> where .Element == Int>

// NOT ALLOWED
// Both Collection and OptionSetType have associated types.
let b : Any<Collection, OptionSetType where .Element == Int>
```

### Associated typealias rewriting

Existentials using `Any<...>` should be able to accept constraints using associated type typealiases. For example, given:

```swift
protocol IteratorProtocol {
	associatedtype Element
}

protocol Sequence {
	associatedtype Iterator : IteratorProtocol
	typealias Element = Iterator.Element
}
```

The user should thus be allowed to write:

```swift
let a : Any<Sequence where .Element == Int>
```

rather than:

```swift
let a : Any<Sequence where Sequence.Iterator : Any<IteratorProtocol where IteratorProtocol.Element == Int>>
```

We will use the above example to illustrate the algorithm by which associated type typealiases are recursively rewritten. To expand a typealias that refers to other associated types in the protocol, take the right-hand side of the typealias expression and do the following:

1. The outermost associated type is 'peeled off' (example; `Iterator`).
2. A `where` clause constraint is created that constrains that associated type to an `Any` existential containing the protocol requirement containing the associated type described by the new outermost item. (Example: `Element` belongs to `IteratorProtocol`, the first item in `Iterator.Element` after `Iterator` is removed.)
3. If the second-outermost item is the last item, then add a `where` clause to the existential containing the type and object of the constraint. (Example: `where IteratorProtocol.Element == Int`)
4. Otherwise, go to step 2 and repeat the process.

## Theory

### Existential type equivalence

The first question we need to answer is how to reason about the equivalence of two existentials.

Any `Any<...>` existential type can be conceptually normalized into the following information:

* **Zero or one classes, or just 'any class'.** This is done by taking all any-class and class requirements defined in the `Any<...>` construct or any nested `Any<...>`s and finding the single class which is a subclass of all the other classes, or just that class if all class constraints are the same. Any protocols listed at requirements that define a `class` requirement add `class` to the list of class requirements. Any associated types with a class requirement at the site of definition add that requirement to the list.

* **A list of zero or more protocols.** This is done by creating a list of all protocols from protocol constraints defined in the `Any<...>` construct or any nested `Any<...>`s, and discarding any protocols which serve as parents to other protocols in this list. Any associated types with protocol requirements at the site of definition add those protocol requirements to the list.

* **A list of all associated types from the above protocols, and constraints on those associated types.** Constraints from all `where` clauses (whether in the `Any<...>` or any nested `Any<...>`s) are coalesced. Any associated types with constraints on protocol requirements at the site of definition add those constraints to the list. Constraints on a parent protocol's associated types are 'given' to the child protocol when the redundant parent protocol is discarded. Any duplicate constraints (e.g. an associated type is required to conform to both a parent and child protocol) are discarded.

Any two existentials which normalize to the same three sets of information above should be considered equivalent, no matter how they were represented in code (i.e. ordering of requirements, how requirements are nested).

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

### Ordering

Following are rules for determining whether one existential is a subtype of another. Please note that all references to requirements refer to the normalized form of the existential, following the preceding set of rules. A type is vacuously the subtype of itself.

A type `A` is a *subtype* of another type `B` iff `B` can be used anywhere `A` can without changing the semantics of the program. If, according to the rules outlined below, two non-equivalent types `A` and `B` seem like they would be subtypes of each other (e.g. some rules make `A` a subtype of `B`, and others vice versa), they are not related at all.

Note that for the sake of the descriptions below, any concrete class is considered a subclass of `class`. Therefore, `P<MyClass, Protocol1>` is a subtype of `P<class, Protocol1>`.

* A concrete type that satisfies an existential's requirements is a subtype of that existential. `String` can be used anywhere an `Any<Streamable>` can. 

* Existential type `A` is a subtype of existential type `B` if and only if `A` imposes requirements that are at least equivalent to those of `B`. This includes class requirements, protocol requirements, and constraints on associated types.

	```swift
	protocol Foo { }
	protocol Bar { }
	protocol Baz { }

	// An existential
	typealias Parent = Any<Foo, Bar>

	// A subtype of Parent
	// Imposes all requirements on Parent
	// Any class that satisfies 'Child' also satisfies 'Parent'
	typealias Child = Any<Foo, Bar, Baz>

	// Not a valid subtype of Parent
	// 'Foo' requirement on Parent is missing
	typealias NotChild = Any<Bar, Baz>
	```

* All else being equal, an existential `A` which imposes any sort of class constraint is a subtype of an existential `B` that does not impose any class constraint.

* All else being equal, an existential `A` which imposes a class constraint `Child`, where `Child : Parent`, is a subtype of an existential `B` that imposes a class constraint `Parent`.

* All else being equal, an existential `A` which imposes an additional protocol constraint `P`, is a subtype of an existential `B` that does not impose that protocol constraint.

* All else being equal, an existential `A` which imposes an additional protocol constraint `Child`, where `Child : Parent`, is a subtype of an existential `B` that imposes only a protocol constraint on `Parent`.

* All else being equal, an existential `A` which imposes a protocol conformance constraint on any associated type, is a subtype of an existential `B` that does not impose that protocol conformance constraint.

* All else being equal, an existential `A` which imposes a protocol conformance constraint `Child`, where `Child : Parent`, on any associated type, is a subtype of an existential `B` that only imposes a protocol conformance constraint `Parent` on that associated type.

* All else being equal, an existential `A` which imposes a subclassing constraint on an associated type, is a subtype of an existential `B` that does not impose that subclassing constraint. 

* All else being equal, an existential `A` which imposes a subclassing constraint `Child`, where `Child : Parent`, on an associated type, is a subtype of an existential `B` that only imposes a subclassing constraint `Parent` on that associated type.

* All else being equal, an existential `A` which imposes a concrete type requirement constraint (e.g. `AssocType == Int`) on any associated type, is a subtype of an existential `B` that imposes zero or one conformance or subclassing constraints instead on that associated type.

* All else being equal, an existential `A` which imposes a concrete type requirement constraint `Child`, where `Child` is a subtype of `Parent`, on any associated type, is a subtype of an existential `B` that imposes a type requirement constraint `Parent` on that associated type.

* All else being equal, an existential `A` which imposes a type equality constraint (e.g. `AssocType1 == AssocType2`) on any associated type, is a subtype of an existential `B` that does not impose that type equality constraint. Note that if such a type equality constraint is imposed on `A`, all constraints on `AssocType1` are implicitly imposed on `AssocType2`, and all constraints on `AssocType2` are implicitly imposed on `AssocType1` for the purposes of determining whether other existentials are subtypes of `A`.

### Associated types and member exposure

The type system will endeavor to allow users to invoke protocol methods on objects of existential type, while still preserving type safety. A comprehensive overview as to how this should work follows.

A variable of existential type may have zero or more associated types attached to it, accreted by conforming to protocols that define associated types. The associated types belonging to a value of existential type will be referred to as **anonymous associated types**, since their exact underlying type is not necessarily known at compile time or relevant at runtime.

These associated types can be *opened* and accessed through a syntax consisting of the variable name, followed by the dot accessor operator, followed by the name of the associated type. An example follows:

```swift
let a : Any<Collection>

// A variable whose type is the Index associated type of the underlying
// concrete type of 'a'.
let theIndex : a.Index = ...

// A variable whose type is the Element associated type of the underlying
// concrete type of 'a'.
let theElement : a.Element = ...
```

Any such anonymous associated type, or any variable of such a type, is only valid within the lexical scope within which the parent existential was defined. Anoymous associated types cannot escape their scopes; values of these types must be cast to 'real' types before they can leave:

```swift
// This is expressly prohibited. "a.Element" is meaningless outside the context
// of the function body.
func doSomething(a: Any<Collection>) -> a.Element {
	return a.first!
}
```

Anonymous associated types can be used to specialize generic functions and instantiate generic types:

```swift
protocol MyProtocol { 
	func foo()
}

func doSomething<T : MyProtocol>(arg: T) {
	arg.foo()
}

func doSomethingElse(a: Any<Collection where .Element : MyProtocol>) {
	let firstElement : a.Element = a.first!

	// This is okay
	doSomething(firstElement)
}
```

This, for example, allows the creation of generic objects parameterized on an anonymous associated type, like the array in the following example:

```swift
protocol MyProtocol {
	associatedtype T
	func foo(_ x: Int) -> T
	func bar(_ y: [T]) -> String
}

func doSomething(a: Any<MyProtocol>) -> String {
	// Create an array of "a.T" to hold return values from foo()
	var foos : [a.T] = []
	for i in 0..<10 {
		foos.append(a.foo(i))
	}
	// Pass that "[a.T]" into bar() to get a String
	let finalResult = a.bar(foos)
	return finalResult
}
```

Their metatypes are also available, although subject to the same restrictions as the types themselves.

Naming conflicts (for example, if a `Collection`-conforming type has an `Index` property), will be resolved in favor of the anonymous associated type. In practice, given the Swift naming conventions, this problem should rarely come up, if ever. It is probably not worth spending additional time addressing.

#### Protocol APIs and associated types

If an existential conforms to a protocol with associated types, the APIs of that protocol are exposed in terms of the anonymous associated types belonging to that existential. This applies to both the types of arguments and the types of return values, or the equivalent concepts for initializers, protocols, and subscripts.

Here is an example of a function that does useful work exclusively through the use of anonymous associated types:

```swift
// Given a mutable collection, swap its first and last items.
// Not a generic function. 
func swapFirstAndLast(inout collection: Any<BidirectionalMutableCollection>) {
	// firstIndex and lastIndex both have type "collection.Index"
	guard let firstIndex = collection.startIndex,
		lastIndex = collection.endIndex?.predecessor(collection) where lastIndex != firstIndex else {
			print("Nothing to do")
			return
	}

	// oldFirstItem has type "collection.Element"
	let oldFirstItem = collection[firstIndex]

	collection[firstIndex] = collection[lastIndex]
	collection[lastIndex] = oldFirstItem
}
```

#### Real types to anonymous associated types

It is often important to convert between a real type (e.g. nominal and structural types) and an anonymous associated type. In both directions, type translation will be accomplished through the unconditional casting operator `as` at compile time, and the `as?` and `as!` conditional casting operators at runtime.

A real type can only be unconditionally translated into an anonymous associated type if the existential constrained that associated type to be a concrete type with a same-type constraint. This is because protocol members' inputs are contravariant. An example follows:

```swift
var a : Any<BidirectionalMutableCollection where .Element == String> = ...

let input = "West Meoley"

// Not actually necessary, since the compiler knows "a.Element" is String.
// A fully constrained anonymous associated type is synonymous with the concrete
// type it's forced to take on, and the two are interchangeable.
// However, 'as' casting is still available if desired.
let anonymousInput = input as a.Element

a[a.startIndex] = anonymousInput

// as mentioned, this also works:
a[a.startIndex] = input
```

However, it should always be possible to conditionally cast from a real type to an anonymous associated type, no matter how loosely the associated type is constrained:

```swift
// If the collection allows it, set the first element in the collection to a given string.
func setFirstElementIn(inout collection: Any<Collection> toString string: String) {
	if let element = string as? collection.Element {
		// At this point, 'element' is of type "collection.Element"
		collection[collection.startIndex] = element
	}
}
```

Note that `as?` can be used to narrow existentials at runtime, allowing the user to fully constrain the associated types and making working with the APIs more convenient. See *Dynamic casting using as?*, below.

#### Anonymous associated types to real types

An anonymous associated type can also be translated into a real type. The most specific type that unconditional `as` casting works with for a given anonymous associated type is the most specific out of the following, from top to bottom:

* `Any`

* If the associated type is defined with requirements, an existential formed from those requirements

* If the associated type is constrained in the existential definition, an existential formed from those constraints and the requirements from above

* If the associated type is constrained with a concrete type requirement, that concrete type

Some examples follow. In this first example, `AssocType` is completely unconstrained, both in the protocol and the existential definition, so the return value can only be unconditionally cast to `Any`.

```swift
protocol Foo {
	associatedtype AssocType
	func someFunc() -> AssocType
}

let a : Any<Foo> = ...

let result : a.AssocType = a.someFunc()

// Okay
let r1 = result as Any

// Also okay
// At runtime, check if 'result' is a String, and if so downcast it and put it
// in 'r2', otherwise 'r2' is nil.
let r2 = result as? String
```

In this example, `AssocType` is constrained to `Protocol1` and `Streamable` when defined, but unconstrained in the existential definition:

```swift
extension String : Protocol1 { }

protocol Foo {
	associatedtype AssocType : Protocol1, Streamable
	func someFunc() -> AssocType
}

let a : Any<Foo> = ...

let result : a.AssocType = a.someFunc()

// Okay
let r1 = result as Any<Protocol1, Streamable>

// Okay, because String conforms to both Protocol1 and Streamable
// (Note that this is a conditional cast and could fail at runtime, returning 
// nil instead of a String)
let r2 = result as? String

// Not okay; Int is known not to conform at runtime (unless it is retroactively
// extended)
let r3 = result as? Int
```

In this example, `AssocType` is still constrained, but the existential definition also constrains it to `Protocol3`:

```swift
protocol Foo {
	associatedtype AssocType : Protocol1, Protocol2
	func someFunc() -> AssocType
}

let a : Any<Foo where .AssocType : Protocol3> = ...

let result : a.AssocType = a.someFunc()

// Okay
// Result constrained to be Any<Protocol1, Protocol2, Protocol3>
// (Think of it as an 'Any<Any<Protocol1, Protocol2>, Protocol3>')
let r1 = result as Any<Protocol1, Protocol2, Protocol3>
```

In this example, `AssocType` is constrained to be the concrete type `Small`, so that is what is returned:

```swift
protocol Foo {
	associatedtype AssocType : Protocol1, Protocol2
	func someFunc() -> AssocType
}

class Small : Protocol1, Protocol2 { ... }

class Big : Foo {
	func someFunc() -> Small { ... }
}

let a : Any<Foo where .AssocType == Small> = Big()

let result : a.AssocType = a.someFunc()

// Okay
let r1 : Small = result as Small

// Also okay, since the type is fully constrained and thus synonymous.
let r2 : Small = result
```

Unconditional casting to covariant generic output types are allowed, even without concrete type constraints.

```swift
protocol Foo {
	associatedtype AssocType
	// Optional<T> is covariant on T
	func someFunc() -> AssocType?
}

let a : Any<Foo where .AssocType : Protocol1> = ...

let result : a.AssocType? = a.someFunc()

// Okay
// Turn the ".a.AssocType?" into an "Protocol1?"
let r1 = result as Protocol1?
```

(In practice, this will be limited to `Optional`. Only optionals and the three value-semantic collections in Swift are covariant, and variance cannot be defined on any other type. However, the value-semantic collections cannot be cast from `SwiftCollection<SomeConcreteType>` to `SwiftCollection<SomeExistentialType>` unless `SomeConcreteType` is a reference type, so they cannot be used with this feature either.)

## Usage

An `Any<...>` existential type can be trivially used in any of the following situations:

* Local variable type
* Property type
* Subscript input or output type
* Parameter to non-generic function or initializer
* Function return value
* Dynamic casting (the object of `as?`, an `as` case, `as!`, etc)

Existentials cannot be used in the following ways:

* As part of the superclass/protocol conformance portion of a type declaration.

### Generics

Existentials can be used in generic declarations in the following ways:

* As a parameter value or return value for a generic function or subscript, or a parameter value for a generic initializer. In this case generic type variables are only allowed to be used in the existential type's `where` clause constraints:

	```swift
	func myFunc<T>(x: T, y: T) -> Any<Collection where .Element == T> { ... }
	```

* As the type of a property in a generic type. Again, generic type variables are only allowed to be used in the existential type's `where` clause constraints:

	```swift
	class MyClass<T> {
		var collection : Any<Collection where Collection.Element == T> = []
	}
	```

* As the object of a constraint in a generic declaration's `where` clause. However, this is not actually an existential, even though its syntax is identical. Instead, a 'pseudo-existential' used in this way is more like a bundle of constraints upon a type variable. I therefore propose that this usage require the keyword `AllOf` instead of `Any`. (This does not affect the use of existential typealiases in `where` clauses, as seen below.)

	```swift
	func myFunc<T where T : AllOf<class, FooProtocol, BarProtocol>>(x: T) { ... }

	// Is exactly synonymous with:
	// func myFunc<T where T : AnyObject, T : FooProtocol, T : BarProtocol>(x: T) { ... }
	```

	One use case is breaking complex sub-constraints into more legible typealiases. For example:

	```swift
	// A collection containing collections that have customized descriptions;
	// these collections in turn contain Ints.
	let a : Any<Collection where .Element : CustomStringConvertible, .Element : Collection, .Element.Element == Int>

	// becomes...
	typealias IntCollection = Any<Collection where .Element == Int>

	let b : Any<Collection where .Element : AllOf<IntCollection, CustomStringConvertible>>
	```

Existentials cannot be used with generics in the following ways:

* In generic declarations, with the requirements composed out of generic type variables:

	```swift
	// NOT ALLOWED
	func foo<A, B>(x: A, y: B) -> Any<A, B> { ... }
	```

* Passed as arguments of generic type `T` to generic functions, unless `T` is completely unconstrained and the argument's type is `T` or a generic type covariant on `T`. This is consistent with Swift's current behavior: generics cannot be specialized on existential types, only concrete types.


### Dynamic casting using `as?`

In addition to guarantees provided by the existential at the point of definition, the dynamic `as?` cast should also be able to cast an existential at a use site.

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

Casting does not preserve relationships that were in the original type but aren't expressed in the casted-to type, as per expected type-casting semantics. (This also implies that an existential `A` can be dynamically cast to another existential `B` that isn't a subtype of `A`, and is completely okay.)

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

Here is an example of how `as?` casting can be used to do 'useful' work at runtime by casting an existential type to a sufficiently constrained subtype:

```swift
// Set the first element of a string or integer collection to some value.
func setFirstElement(collection: Any<Collection>, theInt: Int, theString: String) -> Bool {
	guard !collection.isEmpty else { return false }

	if let intCollection = collection as? Any<Collection where .Element == Int> {
		intCollection[0] = theInt
	} else if let stringCollection = collection as? Any<Collection where .Element == String> {
		stringCollection[0] = theString
	} else {
		return false
	}
	return true
}
```

## Miscellaneous

### Metatype

The metatype of an existential allows for access to any static methods defined across the protocols and classes comprising the existential, and any initializers defined across the protocols comprising the initializer, subject to the accessibility requirements described above in *Associated Types and Member Exposure*. It is defined as `Any<...>.Protocol`, where `...` is replaced by the requirements and `where` clause defined by an existential type.

Note that existentials containing class requirements are still considered to have metatypes of `Protocol` type. An existential containing class requirements and protocol requirements is not equivalent to just the class described in the class requirement.

### Repeated Protocols

In a single `Any<...>` construct, a given protocol can only show up in one non-nested-`Any<...>` requirement. For example, the following is illegal:

```swift
// Not accepted, because Collection shows up in two requirements.
let a : Any<Collection, Collection where Collection.Element == Int>
```

However, a protocol is allowed to show up in a requirement, even if a protocol it inherits from is present in another requirement:

```swift
// Okay, but redundant, because all Hashables are Equatable.
// This is more important for nesting existentials using typealiases
let a : Any<Equatable, Hashable>
```

This matches Swift's current behavior, where declaring duplicate conformances is an error, but conforming a type to both a parent and child protocol is not.

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

One possible way of accomplishing this is by allowing for a runtime test that can compare the underlying types and 'open' them into an anonymous type variable. Here is an example of how this might be done:

```swift
let e1: Equatable = ...
let e2: Equatable = ...

if let storedInE1 = e1 openas T, storedInE2 = e2 as? T {
  let areEqual = e1 == e2
}
```

### Additional 'kind' (value/reference/etc) constraints

Currently, Swift supports only the generic `class` constraint, for specifying "any class". Additional constraints, such as `struct`, `enum`, and/or `value`, might be proposed in the future. If so, retrofitting them to work with this proposal should be conceptually simple: for example, one can imagine an 'any-struct' counterpart to the existing 'any-class' requirement.

### Bottom type support

Some discussion participants expressed interest in a bottom type for Swift (a type that is considered a subtype for all other types, much like how `Any` is a top type). If such a type is ever implemented, a follow-up proposal can relax existential semantics in a way that any existential defined with contradicting constraints (such as `Any<Int, String>`) is considered synonymous to that bottom type. We could then allow existential requirements to be built out of generic type parameters, with any such 'contradicting' constraints resulting in an existential of the bottom type.

## Impact on existing code

No impact. Greenfield feature.

## Alternatives considered

Alternatives here refer to alternate ways the proposal could be implemented.

### Require associated type definition constraints to be repeated

An earlier version of the proposal proposed the following:

"If an associated type is constrained at the site of definition (an example: `Collection`'s `Index` must be a `ForwardIndexType`), those constraints must be repeated in the existential definition. However, they do not need to be repeated if the associated type is constrained such that it must be a subtype of the definition site constraints, or a concrete type."

However, this would make defining simple existentials extremely cumbersome: `Any<Collection>` would become something like `Any<Collection where .SubSequence : CollectionType, .Generator : GeneratorType, .Index : ForwardIndexType>`.

### Separate `where` clauses

An earlier version of the proposal allowed individual requirements to each have their own `where` clauses. While this might have improved legibility in certain cases by placing constraints on associated types closer to the protocols they involved, it also made the feature more complicated and difficult to understand.

### Binding protocols to identifiers

An earlier version of the proposal allowed requirements to bind a protocol name to a local type identifier. This was intended as a way to allow users to specify short names for use in `where` clauses. However, the usefulness of this feature is debatable (is it really clearer to alias `Collection` to `T`?), and it provides no increased expressive capability.

### Simple version of `Any<...>`

A simple version of `Any<...>` lacking `where` clauses was proposed for use in generic type declarations and dynamic type casing expressions. Feedback indicated that this was overly limiting.

-------------------------------------------------------------------------------

# Rationale

On [Date], the core team decided to **(TBD)** this proposal.
When the core team makes a decision regarding this proposal,
their rationale for the decision will be written here.

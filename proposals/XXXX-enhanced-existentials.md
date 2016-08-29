# Enhanced Existential Support

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-name.md)
* Authors: [Douglas Gregor](https://github.com/DougGregor), [Joe Groff](https://twitter.com/jckarter), Austin Zheng
* Status: **[Awaiting review](#rationale)**
* Review manager: TBD

## Introduction

Swift's support for existential types is currently quite limited: two or more protocols can be composed using the `P1 & P2 & P3` syntax. We propose, in the spirit of [*Completing Generics*](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md), to add support to Swift for describing more complex existential types, including those involving protocols with associated types.

## Acknowledgements

Many thanks to [Matthew Johnson](https://github.com/anandabits), Thorsten Seitz, [David Smith](https://twitter.com/Catfish_Man), [Adrian Zubarev](https://github.com/DevAndArtist), Dave Abrahams, and TBD, whose feedback and contributions were instrumental in composing this proposal.

## Motivation

There are a variety of useful types which cannot be expressed currently within Swift's type system. Here are two examples:

1. **Types involving protocols with associated types, and constraints upon those associated types**.

	It's possible to express the notion of "an `Array` containing `Int`s", or the notion of "a `Set` containing `Int`s":

	```swift
	let myArray : [Int]
	let mySet : Set<Int>
	```

	But it's not possible to express the notion of "any `Collection` whose elements are `Int`s":

	```swift
	// Not valid, can't use `Collection` as a protocol type
	let myCollection : Collection
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

Almost all of these problems can be solved by **generalizing the use of existential types**. Currently, an existential type can be composed of one or more protocols, none of which have `Self` or associated type requirements. We propose to extend existential types so that they can include protocols with `Self` or associated type requirements, and so that they can define class bounds as well as protocol requirements. A couple of applications of such a feature include:

1. **Abstracting over `Collection`s**.

	Currently, the concept of "any `Collection` whose elements are `Int`s" is represented by the standard library `AnyCollection<Int>` data structure. This concept can be expressed far more elegantly using a generalized existential type:

	```swift
	// "Any type conforming to Collection, whose Elements are Ints"
	var a = Collection where .Element == Int
	a = [1, 2, 3, 4]
	a = Set([4, 5, 6, 7])
	```

	Generic typealiases provide an opportunity to make such existential types even more ergonomic:

	```swift
	typealias AnyCollection<T> = Collection where .Element == T
	// Similar typealiases for the other two Collections, and for Sequence...

	let a : AnyCollection<Int> = [1, 2, 3, 4]
	```

2. **Replacing `AnyObject`**.

	The `AnyObject` existential, which is currently implemented with heavy compiler support, could be replaced with a straightforward typealias:

	```swift
	// "Any type which is a class"
	typealias AnyObject = class
	```

3. **Dynamic yet typesafe behavior**.

	The following example illustrates an example of dynamic yet type-safe behavior involving protocols with associated types, made possible through the extended existential capabilities detailed in this proposal. It would be very difficult to implement equivalent code in current versions of Swift without resorting to the use of type-erasing helper objects:

	```swift
	protocol PetFood { }
	class DogBiscuit : PetFood { }
	class CannedTuna : PetFood { }

	protocol Pet {
		associatedtype Food : PetFood
		var isHungry : Bool { get }
		func feed(_ food: Food)
	}

	class Dog : Pet {
		func feed(_ food: DogBiscuit)
		// ...
	}

	class Cat : Pet {
		func feed(_ food: CannedTuna)
		// ...
	}

	class PetDayCare {
		var pet : Pet

		func feedPet(foodItems: [PetFood]) {
			for food in foodItems {
				guard pet.isHungry else { return }
				if let edibleFood = food as? pet.Food {
					// only feed pets food that they'll eat
					pet.feed(edibleFood)
				}
			}
			if pet.isHungry { print("Pet is still hungry...") }
		}
	}
	```

## Proposed solution

An improved version of the `P1 & P2` construct will serve as the primary method for declaring an existential type.

An existential type is comprised of one or more *requirements*. Requirements are separated by `&`. After the requirements is an optional `where` clause, in which constraints are placed upon any associated types introduced by previous requirements.

The following requirements are valid. Detailed descriptions follow.

* **Any-class requirement**. This must be the first requirement, if present. This requirement consists of the keyword `class`, and requires the existential to be of any class type.

* **Class requirement**. This must be the first requirement, if present. This requirement consists of the name of a class type, and requires the existential to be of a specific class or its subclasses. It cannot be the only requirement.

* **Protocol requirement**. This requirement consists of the name of a protocol, and requires the existential to conform to the protocol.

* **Nested existential**. This requirement consists of another existential construct, delineated using `()`.

### Any-class requirement

Places a constraint on the existential to be any class type. This requirement must be the first one, if present. There can be only one any-class requirement, and it is mutually exclusive with class requirements.

Example:

```swift
// Can be any class type that conforms to SomeProtocol.
let a : class
```

An existential that declares a requirement which is a `class`-only protocol can also declare the any-class requirement, although doing so is redundant:

```swift
protocol A : class { }

// Acceptable, but exactly the same as just A
let a : class & A
```

### Class requirement

Places a constraint on the existential to be an instance of the named class, or any subclass thereof. This requirement must be the first one, if present. There can be only one class name constraint, and it is mutually exclusive with the any-class requirement.

Example:

```swift
// Can be any class type that is a UIView or a subclass of UIView,
// that also conforms to SomeProtocol.
let a : UIView & SomeProtocol
```

### Protocol requirement

Places a constraint on the existential to be a type which conforms to the named protocol. Both protocols that have associated types or self requirements, and those that don't, are acceptable.

Example:

```swift
// Can be any type that conforms to both Equatable and Streamable.
let a : Equatable & Streamable
```

### Nested typealias existential

Places a requirement on the existential to meet all requirements imposed by the existential named by the typealias. (A typealias whose type is not an existential does not qualify as a nested typealias existential requirement.)

Example:
```swift
typealias Foo = ProtocolB & ProtocolC

// Can be any type that conforms to ProtocolA, ProtocolB, and ProtocolC.
let a : ProtocolA & Foo
```

The existential named by the typealias may declare a class or any-class constraint, even if the containing existential also declares such a constraint. However, one of the classes must be a subclass of every other class declared within such constraints, or all the classes must be the same. This class, if it exists, is chosen as the existential's constraint. (See *Existential Type Equivalence*, below.)

Examples:

```swift
typealias SpecialView = UIView & ProtocolA
typealias SpecialTableView = UITableView & SpecialView

// Can be any type that is a UITableView conforming to ProtocolA.
// UITableView is the most specific class, and it is a subclass of the other
// two classes.
let a : UIScrollView & SpecialTableView

typealias SpecialData = NSData & ProtocolA

// NOT ALLOWED: no class is a subclass of all the other classes. This cannot be
// satisfied by any type.
let b : UIView & SpecialData
```

Likewise, a nested typealias existential may declare a protocol constraint involving a protocol that has already been constrained by the parent or a sibling nested typealias existential clause. However, if a protocol is constrained in such a way that an associated type has two contradictory same-type constraints or subclass constraints, this is an error:

```swift
typealias CollectionOfInts = Collection where .Element == Int

// NOT ALLOWED
// This is impossible to fufill. A collection's elements cannot be both strings
// and integers at the same time.
let a : CollectionOfInts where .Element == String
```

The composition of existentials via typealiases allows API requirements to be expressed in a more natural, modular way:

```swift
// Any custom UIViewController that serves as a delegate for a table view.
typealias CustomTableViewController = UIViewController & UITableViewDataSource & UITableViewDelegate

// Pull in the previous typealias and add more refined constraints.
typealias PiedPiperResultsViewController = CustomTableViewController & PiedPiperListViewController & PiedPiperDecompressorDelegate
```

However, recursive (self-referential) nested typealiases are not allowed.

A nested typealias existential's `where` clause may reference protocols defined within further-nested existential typealias requirements, but the reverse is not true: a nested typealias existential's `where` clause cannot reference protocols in the parent existential definition.

Any existential type containing nested typealias existential requirements can be conceptually 'flattened' and written as an equivalent existential containing no nested typealias existential requirements. The two representations are exactly interchangeable.

### `where` clause (typealiases, variables, and properties)

An existential type spelled out as the type of a local variable, property, or typealias can optionally have a single `where` clause. If this `where` clause exists, it is placed following all the requirements.

(Note that an existential can have a `where` clause, but it might also have nested existential requirements that have their own `where` clauses. This is allowed.)

The `where` clause contains one or more constraints. Constraints must involve the associated types of the previously required protocols. `Self` cannot be constrained.

The acceptable constraints following the `where` clause are identical to those following the `where` clause on a constrained extension or generic function/type definition. In each case `X` must be the name of an associated type defined on one of the existential's component protocols:

* Type equality constraint: `X == ConcreteType`
* Type equality constraint: `X == AnotherAssociatedType`
* Type conformance constraint: `X : SomeProtocol`
* Type inheritance constraint: `X : SomeClass`
* Composite constraint: `X : P1 & P2`

Associated types are referred to with a leading dot, as in the following example. Note that any given type may only have one associated type with a given name (for example, given protocols `Foo` and `Bar`, both with the associated type `Baz`, a type cannot conform to both `Foo` and `Bar` but satisfy `Foo.Baz` and `Bar.Baz` with different types).

```swift
// Can be any Collection whose elements are Ints.
let a : Collection where .Element == Int

// Can be any Collection whose elements are Streamable; the Collection itself 
// must also be Streamable.
let b : Streamable & Collection where .Element : Streamable
```

Associated types used within the `where` clause must belong to one of the protocols previously declared in a requirement. For example, this is wrong:

```swift
// NOT ALLOWED
// Collection doesn't have an associated type named RawValue, so we can't use
// it to define constraints.
let a : Collection where .Element == .RawValue
```

### `where` clause (arguments and return values)

An existential defined as the type of a function (or function-like member) argument or return value cannot be defined with a `where` clause attached directly to the existential declaration.

Instead, a `where` clause can be defined following the function-like member's return value (or argument list, if no equivalent to the return value exists), just as if the member were generic. All existential constraints can be placed in this `where` clause. These constrains can exist alongside generic type parameter constraints if such exist.

An existential's associated types are fully qualified, with the name of the existential argument (or `return`, for the return value) followed by a dot and then the name of the associated type.

```swift
// Note the single 'where' clause, which places constraints on all three
// existentials.
func doSomething(x: Collection, y: Collection) -> Collection
  where x.Element == Int, y.Element == Int, return.Element == Double
{
	// ...
}
```

In such a case, it is not allowed to define constraints that create a relation between different existential values:

```swift
// NOT ALLOWED; each constraint can reference at most a single existential
// value (argument or return value)
func doSomething(x: Collection, y: Collection)
	where x.Element == y.Element
{
	// ...
}
```

### Analogy to generics

Given the requirements and constraints described above, it should be apparent that an existential type signature encapsulates much of the same information as a generic type signature describing a single generic type parameter:

```swift
let a : SomeClass & SomeProtocol where .Assoc1 == SomeType, .Assoc2 : AnotherProtocol

func foo<T>(z: T) where T : SomeClass, T : SomeProtocol, T.Assoc1 == SomeType, T.Assoc2 : AnotherProtocol {
	// ...
}
```

In the above example, the value of existential type `a` and the function parameter of generic type `z` can be inhabited by the same group of concrete types.

## Existential API

### Associated types and member exposure

The type system will endeavor to allow users to invoke protocol methods on objects of existential type, while still preserving type safety. A comprehensive overview as to how this should work follows.

A constant value of existential type may have zero or more associated types attached to it, accreted by conforming to protocols that define associated types. These associated types can be *opened* and accessed through a syntax consisting of the variable name, followed by the dot accessor operator, followed by the name of the associated type.

An example follows:

```swift
let a : Collection

// A variable whose type is the Index associated type of the underlying
// concrete type of 'a'.
let theIndex : a.Index = ...

// A variable whose type is the Element associated type of the underlying
// concrete type of 'a'.
let theElement : a.Element = ...
```

The associated types of an existential are only accessible upon constant values (`let` properties or local bindings; non-`inout` function arguments):

```swift
var a : Collection
// NOT ALLOWED; a.Index can change since 'a' is a var
let index : a.Index
```

Any of an existential's associated types, or any variable of such a type, is only valid within the lexical scope within which the parent existential was defined. An existential's associated types cannot escape the scope within which the existential value is valid; values of these types must be cast to 'real' types before they can leave:

```swift
// This is expressly prohibited. "a.Element" is meaningless outside the context
// of the function body.
func doSomething(a: Collection) -> a.Element {
	return a.first!
}
```

An existential's associated types can be used to specialize generic functions and populate generic types:

```swift
protocol MyProtocol { 
	func foo()
}

func doSomething<T : MyProtocol>(arg: T) {
	arg.foo()
}

func doSomethingElse(a: Collection) where a.Element : MyProtocol {
	let firstElement : a.Element = a.first!

	// This is okay
	doSomething(firstElement)
}
```

This, for example, allows the creation of generic objects parameterized on an existential's associated type, like the array in the following example:

```swift
protocol MyProtocol {
	associatedtype T
	func foo(_ x: Int) -> T
	func bar(_ y: [T]) -> String
}

func doSomething(a: MyProtocol) -> String {
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

The metatypes of an existential's associated types are also available, although subject to the same restrictions as the types themselves.

A further such dependent type exists on all existentials: `.Self`. See *Existential opening using `as?`*, below, for how this type can be used.

Naming conflicts (for example, if a `Collection`-conforming type has an `Index` property), will be resolved in favor of the associated type. In practice, given the Swift naming conventions, this problem should rarely come up, if ever. It is probably not worth spending additional time addressing.

### Protocol APIs and associated types

If an existential conforms to a protocol with associated types, the APIs of that protocol are exposed in terms of the associated types belonging to that existential. This applies to both the types of arguments and the types of return values, or the equivalent concepts for initializers, protocols, and subscripts.

Here is an example of a function that does useful work exclusively through the use of an existential's associated types:

```swift
// Given a mutable collection, swap its first and last items.
// Not a generic function. 
func swapFirstAndLast(inout collection: BidirectionalMutableCollection) {
	let c = collection
	// firstIndex and lastIndex both have type "c.Index"
	guard let firstIndex = c.startIndex,
		lastIndex = c.endIndex?.predecessor(c) where lastIndex != firstIndex else {
			print("Nothing to do")
			return
	}

	// oldFirstItem has type "c.Element"
	let oldFirstItem = c[firstIndex]

	c[firstIndex] = c[lastIndex]
	c[lastIndex] = oldFirstItem
	collection = c
}
```

### Real types to existential's associated types

It is often important to convert between a real type (e.g. nominal and structural types) and an existential's associated type. In both directions, type translation will be accomplished through the unconditional casting operator `as` at compile time, and the `as?` and `as!` conditional casting operators at runtime.

A real type can only be `as`-casted into an associated type if the existential constrained that associated type to be a concrete type with a same-type constraint. An example follows:

```swift
let a : BidirectionalMutableCollection where .Element == String = ...
let i : String = "West Meoley"
let i2 = i as a.Element

let elementsAreSame = a[a.startIndex] == i2

// This also works:
let elementsAreSame = a[a.startIndex] == i
```

However, it should always be possible to `as?`-cast from a real type to an associated type, no matter how loosely the associated type is constrained:

```swift
let a : BidirectionalMutableCollection = ...
let i : String = "West Meoley"

let elementsAreSame : Bool
if let openedI = i as? a.Element {
	elementsAreSame = a[a.startIndex] == openedI
} else {
	elementsAreSame = false
}
```

Note that `as?` can be used to narrow existentials at runtime, allowing the user to fully constrain the associated types and making working with the APIs more convenient. See *Dynamic casting using as?*, below.

### Existential's associated types to real types

An existential's associated type can also be translated into a real type. The most specific type that unconditional `as`-casting works with for a given associated type is the most specific out of the following, from top to bottom:

* `Any`
* If the associated type is defined with requirements, an existential formed from those requirements
* If the associated type is constrained in the existential definition, an existential formed from those constraints and the requirements from above
* If the associated type is constrained with a concrete type requirement, that concrete type

In this first example, `AssocType` is completely unconstrained, both in the protocol and the existential definition, so the return value can only be unconditionally cast to `Any`.

```swift
protocol Foo {
	associatedtype AssocType
	func someFunc() -> AssocType
}

let a : Foo = ...

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

let a : Foo = ...

let result : a.AssocType = a.someFunc()

// Okay
let r1 = result as Protocol1 & Streamable

// Okay, because String conforms to both Protocol1 and Streamable
// (Note that this is a conditional cast and could fail at runtime, returning 
// nil instead of a String)
let r2 = result as? String

// Not okay; Int is known not to conform at runtime (unless it is retroactively
// extended)
let r3 = result as? Int
```

Unconditional casting to covariant generic output types (i.e. `Optional<T>`) is allowed, even without concrete type constraints.

```swift
protocol Foo {
	associatedtype AssocType
	// Optional<T> is covariant on T
	func someFunc() -> AssocType?
}

let a : Foo where .AssocType : Protocol1 = ...

let result : a.AssocType? = a.someFunc()

// Okay
// Turn the ".a.AssocType?" into an "Protocol1?"
let r1 = result as Protocol1?
```

## Usage

An existential type can be trivially used in any of the following situations:

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
	func myFunc<T>(x: T, y: T) -> Collection where return.Element == T { ... }
	```

* As the type of a property in a generic type. Again, generic type variables are only allowed to be used in the existential type's `where` clause constraints:

	```swift
	class MyClass<T> {
		var collection : Collection where .Element == T = []
	}
	```

* As the object of a constraint in a generic declaration's `where` clause. However, this is not actually an existential, even though its syntax is identical. Instead, a 'pseudo-existential' used in this way is more like a bundle of constraints upon a type variable.

	```swift
	func myFunc<T>(x: T) where T : class & FooProtocol & BarProtocol { ... }

	// Is exactly synonymous with:
	// func myFunc<T>(x: T) where T : AnyObject, T : FooProtocol, T : BarProtocol { ... }
	```

	One use case is breaking complex sub-constraints into more legible typealiases. For example:

	```swift
	// A collection containing collections that have customized descriptions;
	// these collections in turn contain Ints.
	let a : Collection where .Element : CustomStringConvertible, .Element : Collection, .Element.Element == Int

	// becomes...
	typealias IntCollection = Collection where .Element == Int

	let b : Collection where .Element : IntCollection & CustomStringConvertible
	```

Existentials cannot be used with generics in the following ways:

* In generic declarations, with the requirements composed out of generic type variables:

	```swift
	// NOT ALLOWED
	func foo<A, B>(x: A, y: B) -> A & B { ... }
	```

* Passed as arguments of generic type `T` to generic functions, unless `T` is completely unconstrained and the argument's type is `T` or a generic type covariant on `T`.

### Dynamic casting using `as?`

In addition to guarantees provided by the existential at the point of definition, the dynamic `as?` cast should also be able to cast an existential at a use site.

Note that when casting a value to an existential type with a `where` clause with more than one constraint, the existential type must be spelled out in parentheses in order to disambiguate between commas used to separate `where` constraints, and commas used to separate clauses in the `if let` or `if case` statement.

```swift
// No information on FooProtocol's associated types at compile time
let a : FooProtocol

// ...

if let narrowedA = a as? (FooProtocol where .AssocA == MyStreamer, .AssocB == Bool) {
	// Now we know what types AssocA and AssocB are, and can use second()
	let x : MyStreamer = narrowedA.second(true)
}
```

Note that `as?` casts which are guaranteed to fail and can be deduced as such at compile-time are prohibited:

```swift
let a : FooProtocol where .AssocB == Int

// NOT ALLOWED; the compiler can see that the requirements are mutually
// exclusive at runtime and will error.
if let narrowedA = a as? (FooProtocol where .AssocB == String) {
	// ...
}
```

Casting does not preserve relationships that were in the original type but aren't expressed in the casted-to type, as per expected type-casting semantics. (This also implies that an existential `A` can be dynamically cast to another existential `B` that isn't a subtype of `A`, and is completely okay.)

```swift
let a : FooProtocol

// This is fine
a.someFooProtocolMethod()

if let narrowedA = a as? BazProtocol {
	// Can no longer call a FooProtocol method on narrowedA:
	narrowedA.someFooProtocolMethod() // DOES NOT WORK
	narrowedA.someBazProtocolMethod() // fine
}

if let narrowedA = as as? FooProtocol & BazProtocol {
	// Can now do both.
	narrowedA.someFooProtocolMethod() // fine
	narrowedA.someBazProtocolMethod() // fine
}
```

Here is an example of how `as?` casting can be used to do 'useful' work at runtime by casting an existential type to a sufficiently constrained subtype:

```swift
// Set the first element of a string or integer collection to some value.
func setFirstElement(collection: Collection, theInt: Int, theString: String) where Collection : class -> Bool {
	guard !collection.isEmpty else { return false }

	if let intCollection = collection as? Collection where .Element == Int {
		intCollection[0] = theInt
	} else if let stringCollection = collection as? Collection where .Element == String> {
		stringCollection[0] = theString
	} else {
		return false
	}
	return true
}
```

### Existential opening using `as?`

In addition to associated types, every existential has access to the `.Self` type, which reflects the underlying concrete type of the existential. This `.Self` type can be used to bridge existential types to generic types:

```swift
let a : Collection where .Element == Int = // ...
let b : Collection where .Element == Int = // ...

func someGenericFunc<C : Collection>(x: C, y: C) where C.Element == Int {
	// ...
}

// Not allowed, for type soundness reasons!
// someGenericFunc(x: a, y: b)

if let openedA = a as? a.Self, let openedB = b as? a.Self {
	// openedA is type a.Self; openedB is type a.Self
	// We now know that openedA and openedB are the same concrete type, which
	// conforms to Collection with Elements that are Ints
	// This is okay
	someGenericFunc(x: openedA, y: openedB)
}
```

If two values are known to be the same concrete type through opening, their associated types are also considered equal:

```swift
let a : Collection = // ...
let b : Collection = // ...

func someFunc<C : Collection>(x: C, y: C.Index) {
	// ...
}

// Again, prohibited
// someFunc(x: b, y: a.startIndex) 

if let openedA = a as? a.Self, let openedB = b as? a.Self {
	// This is okay.
	// Since a.Self == b.Self, a.Index == b.Index
	let start = openedA.startIndex
	someFunc(x: openedB, y: start)
}
```

### Metatype

The metatype of an existential allows for access to any static methods defined across the protocols and classes comprising the existential, and any initializers defined across the protocols comprising the initializer, subject to the accessibility requirements described above in *Associated Types and Member Exposure*. It is defined as `(...).Protocol`, where `...` is replaced by the requirements and `where` clause defined by an existential type. For example, `(P1 & P2 where .Foo == Int).Protocol`.

Note that existentials containing class requirements are still considered to have metatypes of `Protocol` type. An existential containing class requirements and protocol requirements is not equivalent to just the class described in the class requirement.

## Detailed Design

### Existential type equivalence

Two existentials are *equivalent* if they:

* Define the same class requirement, or no class requirement
* Define the same list of protocols to conform to
* Constrain their associated types in exactly the same way

For the class requirement test, if multiple class requirements are defined as part of an existential construction, only the most specific class is considered. Every specific class constraint is considered more specific than the any-class constraint.

For the protocol requirements test, any protocols that are supertypes to other protocols are ignored.

For the constraints test, duplicate constraints are discarded (e.g. an associated type is required to conform to both a parent and child protocol); any constraints specific to a parent protocol's associated types are given to the most specific child when that protocol is discarded.

The ordering or presence of nested existential requirements is irrelevant.

### Ordering

A type `A` is a *subtype* of another type `B` iff `B` can be used anywhere `A` can without changing the semantics of the program. If, according to the rules outlined below, two non-equivalent types `A` and `B` seem like they would be subtypes of each other (e.g. some rules make `A` a subtype of `B`, and others vice versa), they are not related at all, unless they are equivalent.

Note that for the sake of the descriptions below, any concrete class is considered a subclass of `class`. Therefore, `MyClass & Protocol1` is a subtype of `class & Protocol1`. Also, `Child : Parent` for all type relationships below.

* A concrete type that satisfies an existential's requirements is a subtype of that existential. `String` can be used anywhere an `Streamable` can. 

* Existential type `A` is a subtype of existential type `B` if and only if `A` imposes requirements that are at least equivalent to those of `B`. This includes class requirements, protocol requirements, and constraints on associated types.

* Ceteris paribus, class requirements are ordered from least specific to most: no class requirement, any class, `Parent`, `Child`.

* Ceteris paribus, protocol requirements are ordered from least specific to most: no protocol requirement, `Parent`, `Child`.

* Ceteris paribus, protocol conformance constraints on an associated type are ordered from least specific to most: no protocol constraint, `Parent`, `Child`, (existence of concrete type equality constraint). If the concrete type does not conform to the most specific protocol no subtyping relationship exists.

* Ceteris paribus, subclassing constraints on an associated type are ordered from least specific to most: no subclass constraint, `Parent`, `Child`, (existence of concrete type equality constraint). If the concrete type is not the most specific class or a subclass thereof no subtyping relationship exists.

* Ceteris paribus, concrete type equality constraints on an associated type are ordered from least specific to most: no constraint, `Parent`, `Child`.

* Ceteris paribus, an existential `A` which imposes a type equality constraint (e.g. `Assoc1 == Assoc2`) on any associated type, is a subtype of an existential `B` that does not impose that type equality constraint. Note that if such a type equality constraint is imposed on `A`, all constraints on `Assoc1` are implicitly imposed on `Assoc2`, and all constraints on `Assoc2` are implicitly imposed on `Assoc1` for the purposes of determining whether other existentials are subtypes of `A`.

## Impact on existing code

No impact. Greenfield feature.

-------------------------------------------------------------------------------

# Rationale

On [Date], the core team decided to **(TBD)** this proposal.
When the core team makes a decision regarding this proposal,
their rationale for the decision will be written here.

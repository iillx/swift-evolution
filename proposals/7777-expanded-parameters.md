# Introduce Expanded Parameters

## Introduction

Swift is a language that allows us to write expressive API interfaces. With features like constrained Generics, method overloading, trailing closures, and default arguments, you can reduce code duplication while still achieving quite flexible APIs. This proposal aims to increment this part of the language with `@expanded`, a new attribute for function parameters.

## Motivation

Let's start with an example. SwiftUI has great APIs for styling shapes with gradients. For instance, you can fill a Rectangle with a linear gradient by passing an array of colors, startPoint, and endPoint to the `linearGradient()` static function:

```swift
Rectangle()
  .fill(
    .linearGradient(
      colors: [.yellow, .teal],
      startPoint: .init(x: 0.5, y: 0),
      endPoint: .init(x: 0.5, y: 0.6)
    )
)
```

Because it's possible to create a `Gradient` with an array of `Gradient.Stop`, this API also lets you pass an array of them instead. Or you can pass a `Gradient` value directly. For each of these options, there's an overload of the `linearGradient` method. 

```swift
// A linear gradient.
static func linearGradient(Gradient, startPoint: UnitPoint, endPoint: UnitPoint) -> LinearGradient

// A linear gradient defined by a collection of colors.
static func linearGradient(colors: [Color], startPoint: UnitPoint, endPoint: UnitPoint) -> LinearGradient

// A linear gradient defined by a collection of color stops.
static func linearGradient(stops: [Gradient.Stop], startPoint: UnitPoint, endPoint: UnitPoint) -> LinearGradient
```

This API has one "original" function that takes a `Gradient` directly, and two "convenience" overloads for each way a `Gradient` can be created. 

*I don't know how these are actually implemented inside SwiftUI, but for this proposal let's assume these methods will create a `Gradient` somewhere down the line and apply it to a shape, with only the initialization method for the Gradient value differing between them.* 

So what if we were to add a third initializer to `Gradient`? 

```swift
extension Gradient {
  init(materials: [Material]) { ... } 
}
```

In that case, we might want to add the "equivalent" `linearGradient` overload method, to keep our API consistent. 

```swift
// A linear gradient defined by a collection of materials.
static func linearGradient(materials: [Material], startPoint: UnitPoint, endPoint: UnitPoint) -> LinearGradient
```

Given this, the potential of an overload "explosion" is already a problem. But it gets worse considering that this pattern spreads out fairly quickly — now there's an entire family of gradients that could be updated. Radial, angular, and elliptical, all with their respective helper methods, would be good candidates for adding an overload with a materials parameter to support the new initializer.

```swift
// A radial gradient.
static func radialGradient(Gradient, center: UnitPoint, startRadius: CGFloat, endRadius: CGFloat) -> RadialGradient

// A radial gradient defined by a collection of colors.
static func radialGradient(colors: [Color], center: UnitPoint, startRadius: CGFloat, endRadius: CGFloat) -> RadialGradient

// A radial gradient defined by a collection of color stops.
static func radialGradient(stops: [Gradient.Stop], center: UnitPoint, startRadius: CGFloat, endRadius: CGFloat) -> RadialGradient

// A radial gradient defined by a collection of materials.
static func radialGradient(materials: [Material], center: UnitPoint, startRadius: CGFloat, endRadius: CGFloat) -> RadialGradient
```

Writing flexible APIs shouldn't always come at the cost of code duplication, and that's the problem `@expanded` aims to solve.

## Proposed solution

We propose a new type attribute that allows function parameters to be fulfilled with the arguments of an initializer instead. So we can write a new version of `linearGradient` that takes an `@expanded` Gradient.

```swift
static func linearGradient(_ gradient: @expanded Gradient, startPoint: UnitPoint, endPoint: UnitPoint) -> LinearGradient { // do gradient stuff ... } 
```

At the call site, the gradient parameter can be fulfilled with the arguments for `Gradient` initializers, so all of these are legal:

```swift
// call it with an array of Materials
linearGradient(materials: [.thinMaterial, .thickMaterial], startPoint: 0.1, endPoint: 0.5)

// or an array of Colors
linearGradient(colors: [.yellow, .teal], startPoint: 0.1, endPoint: 0.5)

// or even these Gradient.Stop
linearGradient(stops: [.init(color: .blue, location: 12)], startPoint: 0.1, endPoint: 0.5)
```

It eliminates the need to add several overloads to support the different ways a parameter's value can be initialized. It's also fine to call it the old "non-expanded" way:

```swift
let myGradient = Gradient(colors: [.yellow, .teal])

// non-expanded call.
linearGradient(myGradient, startPoint: 0.1, endPoint: 0.5)
```

### Detailed design

`@expanded` (bike-shedding is welcome here) is a Type Attribute and can be used in functions and subscripts parameters. This allows API authors to be in control of which parameters would support this functionality at the call site. 

When given arguments with labels that don't match the original one, the compiler will use overload resolution to select an initializer. 

```swift
class MyClass {
  init(a: String) { print("initializer A chosen") }
  init(b: Bool) { print("initializer B chosen" }
}

func testExpanded(myClass: @expanded MyClass) {  }
testExpanded(a: "hello!") // prints initializer A chosen

extension MyClass {
  init(myClass: Bool) { print("initializer C chosen") }
}

testExpanded(myClass: true) // error, the compiler expects a MyClass instance.
```

This attribute works by introducing new rules to argument-parameter matching in the compiler, and it places a few limitations on which parameters can be *next* to an `@expanded`.

### Multiple expanded parameters

API Authors can choose one, or many, parameters to allow expanding **in the same function. In the case of many expanded parameters, there's a limitation to be aware of: two expanded parameters aren't allowed to be next to each other.

```swift
// not okay
func testExpanded(myClass: @expanded MyClass, myClass2: @expanded MyClass) { }

// okay
func testExpanded(myClass: @expanded MyClass, c: Int, myClass2: @expanded MyClass) { }

```

### Default arguments

`@expanded` parameters can't be *immediately* followed by a parameter with a default argument. Unless this parameter is the *last* one. An error will be thrown, suggesting you move the parameter with the default argument to the end of the function. 

```swift
// not allowed
func testExpanded(myClass: @expanded MyClass, b: Int = 10, c: Bool) { }

// okay
func testExpanded(c: Bool, myClass: @expanded MyClass, b: Int = 10) { }
func testExpanded(myClass: @expanded MyClass, c: Bool, b: Int = 10) { }
```

The `@expanded` parameter itself can have a default value.

```swift
extension MyClass {
  static func test() -> MyClass { // } 
}

func testExpanded(a: @expanded MyClass = .test()) { }
```

### Nominal types

Since `@expanded` expands an argument into its initializer call, it can only be used with types that can have initializers. The compiler will enforce it by checking if the type to which this attribute is attached is a Nominal Type (aka a type with a name, not a structural type). 

```swift
func testExpanded(a: @expanded () -> Bool) {} // not allowed
func testExpanded(a: @expanded (Bool, Bool)) {} // not allowed
```

Access control works as usual for expanded parameters. If a certain initializer isn't visible from the call site of the expanded method, it can't be used. 

```swift
public class Duck { 
  let named: String
  private init(named: String) { // .. }
}

func pet(a: @expanded Duck) { }
pet(named: "Maria") // error

```

### Protocols

This feature can't be used with Protocols, even if the protocol has an initializer requirement. The compiler needs to know which concrete type to instantiate. 

Generics aren't part of the scope at this moment.

## Impact on existing code

This feature introduces the opportunity for a lot of APIs to be refactored. Adding `@expanded` to an ABI-public function parameter isn't a source-breaking change. But since it won't emit overloads to the ABI, adding `@expanded` doesn't allow the deletion of obsolete overloads either.

## Alternatives considered

## Future directions

Add support for enums to use `@expanded` parameters.



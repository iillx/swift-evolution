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

*I don't know how these are actually implemented inside SwiftUI, but for this proposal let's assume these methods will create a `Gradient` somewhere down the line and apply it to a shape, with only the initialization method for the `Gradient` value differing between them.* 

So what if we were to add a third initializer to `Gradient`? 

```swift
extension Gradient {
  init(materials: [Material]) { ... } 
}
```

In that case, we might want to add the equivalent `linearGradient` overload method to keep our API consistent. 

```swift
// A linear gradient defined by a collection of materials.
static func linearGradient(materials: [Material], startPoint: UnitPoint, endPoint: UnitPoint) -> LinearGradient
```

Given this, the potential of an overload "explosion" is already a problem. But it gets worse considering this pattern can spread out fairly quickly — now there's an entire family of gradients that could be updated. Radial, angular, and elliptical, all with their respective helper methods, would be good candidates for adding an overload with a `materials` parameter to support the new initializer.

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

Expanded parameters can also benefit non-public-API code. Functions with long and repeated parameter lists are often refactored to replace those parameters with a single struct containing all of them, as described by this objc.io [post](https://www.objc.io/blog/2018/05/01/extracting-parameters/). The call-site of these methods uses the memberwise initializers structs get. Simplifying the snippets from that post, we have: 

```swift
struct Context {
    let state: SomeState
    let pushViewController: (UIViewController) -> ()
}

func hotspotForm(_ context: Context) {
  // ...
}

// at call site
hotspotForm(
  .init(
    state: SomeState(),
    pushViewController: { //... } 
  )
)
```

This approach relies on the generated `Context` initializer and could benefit from `@expanded`, making the `.init` unnecessary. 

```swift
func hotspotForm(_ context: @expanded Context) { }
hotspotForm(state: SomeState(), pushViewController: { // ... })
```

Another way to get around the `.init` is to manually define a `hotspotForm` overload that takes the parameters directly and forwards them to the initializer. However, maintaining this boilerplate in sync with the struct as it adds/removes properties is error-prone and undermines the benefit of the memberwise init. If instead we rely on `@expanded`, we get both benefits: omitting the `.init` part of the call and no boilerplate to maintain. 

## Detailed design

`@expanded` (bike-shedding is welcome here) is a Type Attribute and can be used in functions, closures, and subscripts parameters. It allows API authors to be in control of which parameters would support this functionality at the call site. 

This feature can help create APIs with fewer overloads and thus make code easier to navigate, comprehend and evolve. Therefore, a function with an `@expanded` parameter can't have overloads. To prevent this abstraction from causing unpredictable behavior at the call site, only a single expanded parameter is allowed, and it must be the first one on the parameter list. 

```swift
// not allowed
func testExpanded(myClass: @expanded MyClass, myClass2: @expanded MyClass) { }

// not allowed
func testExpanded(c: Int, myClass: @expanded MyClass) { }

// ok 
func testExpanded(myClass: @expanded MyClass, c: Int) { }
```

When given arguments with labels that don't match the original one, the compiler will use overload resolution to select an initializer. 

```swift
class Pastry {
  init(a: String) { print("initializer A chosen") }
  init(b: Bool) { print("initializer B chosen" }
}

func bake(pastry: @expanded Pastry) {  }
bake(a: "cake") // prints initializer A chosen

extension Pastry {
  init(pastry: Bool) { print("initializer C chosen") }
}

bake(pastry: true) // error, the compiler expects a Pastry instance.
```

This attribute works by introducing new rules to argument-parameter matching in the compiler, and it places a few limitations on which parameters can be *next* to an `@expanded`.

### Argument-parameter matching

This is the process of pairing *parameters* of the function declaration with their respective *arguments*. It iterates on the declaration taking one parameter at a time and looking for an argument with the same label. At this point, only labels are relevant for matching — types are not part of the example.

To illustrate that, let's look at how argument-parameter matching works for the function `test`.

```swift
class Example {
  init(name: String, count: Int) { }
  init(count: Int) { }
}

func test(first: @expanded Example, second: Int) { }
test(name: "A", count: 2, second: 10)
```

The first iteration is the one that matters for this example. It will take the 'first' parameter and look for an argument with the same label, just like it would for a normal parameter. If it finds one, then this parameter won't be expanded. Now, if it *doesn't* find it, we'll do some extra fun things since this is an `@expanded` parameter. 

The compiler will first look at the label of the following parameter if there is one. In this case, the label is 'second'. After that, we'll start taking the subsequent arguments that *don't match* the label of the next parameter and use them to build the "expanded initializer" call. We stop when we find an argument with a label that matches 'second'.

This way, 'name' and 'count' are the arguments that will end up being used for the expanded initializer. Using these arguments the compiler will select the appropriate `Example` init.  

After properly matching 'first', we match the 'second' parameter to its argument, and we're done.

This is a simplification to convey *why* the parameter next to `@expanded` is so important. The limitations mentioned in the following sections are a consequence of that.

### Default arguments

`@expanded` parameters can't be *immediately* followed by a parameter with a default argument. Unless this parameter is the *last* one. An error will be thrown, suggesting you move the parameter with the default argument to the end of the function. 

```swift
// not allowed
func testExpanded(myClass: @expanded MyClass, b: Int = 10, c: Bool) { }

// okay
func testExpanded(myClass: @expanded MyClass, c: Bool, b: Int = 10) { }
```

The `@expanded` parameter itself can have a default value.

```swift
extension MyClass {
  static func test() -> MyClass { } 
}

func testExpanded(a: @expanded MyClass = .test()) { }
```

### Closures
Closures can make use of `@expanded` parameters as well, with the same rules applying to them. 
```swift
struct Example {
  static let plot: (@expanded CGPoint) -> Void = { // ... }
}

Example.plot(x: 42, y: 42)
```

### Initializers
Only initializers available at the function _declaration_ can be used with `@expanded`, excluding inits defined outside of the module.

• **Initializers with trailing closures**  
When a given expanded type has initializers with closures, the trailing closure syntax can't be used. The expanded expression is limited by the bounds of the "original" parenthesis. 
```swift
class FancyClass {
  init(a: Int, _ b: () -> Void) { }
}

func test(one: @expanded FancyClass) {}
test(a: 7, { print("🦆") })  // okay

func testAgain(one: @expanded FancyClass, two: Bool) {} 
testAgain(a: 7, { print("🦆") }, two: true) // okay
```

### Nominal types

Since `@expanded` expands an argument into its initializer call, it can only be used with types that can have initializers. The compiler will enforce it by checking if the type to which this attribute is attached is a Nominal Type (aka a type with a name, not a structural type). 

```swift
func testExpanded(a: @expanded () -> Bool) {} // error
func testExpanded(a: @expanded (Bool, Bool)) {} // error
```

### Subclassing

The compiler expects arguments to build an initializer call of the parameter type. Therefore, subclassing doesn't work with this feature. 

```swift
class SuperClassy {
  init(one: Bool, two: Int) {}
}

class SimpleClass: SuperClassy {
  init(a: Int, b: Int) {}
}

func test(classy: @expanded SuperClassy) {}
test(a: 10, b: 10) // error, "a" and "b" are arguments of the "SimpleClass" initializer. 
```

### Protocols

This feature can't be used with Protocols – even if the protocol has an initializer requirement. The compiler needs to know which concrete type to instantiate. 

```swift
protocol MyProtocol {
  init(a: Int)
}

struct SimpleStruct: MyProtocol {
  init(a: Int) {}
}

func test(x: @expanded MyProtocol) {}
test(a: 10) // error
```

The example above would be the equivalent of trying to construct a protocol type `test(x: MyProtocol(a: 10))`.

### Inout

`inout` and `@expanded` cannot be used together since the expanded call constructs a new value, and `inout` relies on modifying an existing one.

## Impact on existing code

This feature introduces the opportunity for a lot of APIs to be refactored. Adding `@expanded` to an ABI-public function parameter isn't a source-breaking change. But since it won't emit overloads to the ABI, adding `@expanded` doesn't allow the deletion of obsolete overloads either.

## Alternatives considered

## Future directions

Add support for enums to use `@expanded` parameters.


### Optionals and failable initializers
When dealing with optional expanded parameters, it's unclear whether the appropriate behavior is to build an initializer call to the `Wrapped` type or the `Optional` type itself. Therefore when an optional type is given to an expanded parameter, the compiler will try to build the desugared type.

```swift
class SimpleClass {
  init?(a: Int) { }
  init(a: Int, b: Int) { }
}

func test3(x: @expanded SimpleClass?) {}
test3(a: 10) // error
test3(a: 10, b: 20) // error
```

In the example above the compiler will try to build `Optional<SimpleClass>`. In the future this behavior can be explored to allow unwrapping the optional.

### Expanded trailing closures
In the future, this feature could be _expanded_ (🥱) to allow trailing closure syntax to be used when the expanded parameter is the last one.

```swift
class FancyClass {
  init(a: Int, _ b: () -> Void) { }
}

func test(one: @expanded FancyClass) {} // trailing closure syntax allowed
test(a: 7) { 
  print("🦆") 
 }

func testAgain(one: @expanded FancyClass, two: Bool) {} // not allowed
testAgain(a: 7, { print("🦆") }, two: true) 
```

---

layout: post

title: "Swift  5.7：可以作为变量类型的 PATs"

---

## 协议的概念

Swift 中名为协议（protocol）的概念在诸多支持面向对象编程的语言中都找到其对应概念，例如 Java 和 C# 中的 `interface`，C++ 中的抽象基类等。总的来说，协议描述了一组变量/方法的集合，来提供完成某些特定任务的能力。class/struct/enum 可以通过声明自身满足某个协议，并实现指定的变量/方法，来获得对应的能力。举个简单的例子：

```swift
protocol Greetable {
  func greet() -> String
}
```

`Greetable` 通过描述 `greet` 方法提供了打招呼的能力，任何 class/struct/enum 通过实现协议都可以获得这种能力。

```swift
struct AmericanGreeter: Greetable {
  func greet() -> String {
    "Hi~"
  }
}

struct ChineseGreeter: Greetable {
  func greet() -> String {
    "你好"
  }
}
```

## 作为类型的协议

既然协议可以描述接口，提供能力，那么在只需要使用对应能力的场景下，完全可以使用协议作为类型，而隐去其背后的实现。在这种场景下，协议可以作为变量的类型、函数的参数、函数的返回值等使用。

```swift
let greeter: Greetable = ChineseGreeter()
greeter.greet()

func randomGreeter() -> Greetable {
  if .random() {
      return ChineseGreeter()
  } else {
      return AmericanGreeter()
  }
}

func getGreeting(greeter: Greetable) -> String {
  return greeter.greet()
}

extension Array: Greetable where Element: Greetable {
  func greet() -> String {
      return map { $0.greet() }.joined(separator: "\n")
  }
}
```

## 带关联类型的协议

回到正题，Swift 中给协议提供了一个名为关联类型（`associatetype`）的能力，协议可以指定一个或多个抽象的关联类型在协议方法中使用，最终由实现协议的 class/struct/enum 等提供具体的类型。

```swift
protocol Generator {
  associatedtype Element

  func generate() -> Element
}

struct StringGenerator: Generator {
  typealias Element = String
  func generate() -> Element {
    return ["foo", "bar"].randomElement()!
  }
}

struct IntGenerator: Generator {
  func generate() -> Int {
    Int.random(in: 0...100)
  }
}
```

在实现协议时，可以通过 `typealias` 显式指定关联类型的具体类型，也可以让编译器通过协议方法对应位置的类型推断出关联类型。

虽然同样是协议，但当尝试使用带关联类型的协议作为类型使用时：

```swift
let generator: Generator = StringGenerator() // Error: Protocol 'Generator' can only be used as a generic constraint because it has Self or associated type requirements

func randomGenerator() -> Generator { // Error: Protocol 'Generator' can only be used as a generic constraint because it has Self or associated type requirements
  if .random() {
    return IntGenerator()
  } else {
    return StringGenerator()
  }
}
```

会收到编译器的报错：

> Protocol 'XXX' can only be used as a <u>generic constraint</u> because it has Self or associated type requirements

带关联类型的协议（Protocol with Associated Types，以下统称 PATs）虽然也是协议，但和无关联类型的协议相比在使用上有许多不同点，其中之一就是：<u>PATs 不能直接作为类型使用</u>。原因很简单，在上例 `let generator: Generator = StringGenerator()` 中，编译器无法确定需要给名为 `generator` 的变量分配多少内存空间。

那么问题来了：既然编译器无法确定 PATs 类型的变量所需的内存空间，那怎么能确定类型为普通协议的变量所需的内存空间呢？答案是一样无法确定🐶。具体的机制在 WWDC 的 session 以及 Swift 官方文档中多次提及过，简而言之，在遇到类型为协议的变量时，编译器会固定分配 5 words 大小的空间，并分为三份：

- 3 words：存储小于 3 words 的值类型的变量，如果装不下/引用类型，存储堆上变量的引用

- 1 word：存储指向类型 metadata 的指针（对于要求只能由 class 实现的协议，这里会被忽略，通过前 3 words 存储的变量的引用能够获取 metadata）

- 1 word：存储指向 protocol witness table 的指针

  

基于上述理由，PATs 在使用时只能作为泛型函数的类型约束使用，例如：

```swift
func getElement<T: Generator>(generator: T) -> T.Element {
  return generator.generate()
}

func isEqual<T: Equatable>(lhs: T, rhs: T) -> Bool {
  return lhs == rhs
}
```

能这么使用的原因是，作为 `getElement` 和 `isEqual` 方法的调用方，在提供参数时也提供了参数的具体类型，只要这个类型满足 PATs，调用便能够成立。而返回的类型，以及 PATs 的 associatetype 同样也是由传入参数的类型决定的，得益于此，PATs 中没有了未确定的类型。

## Swift 5.7 的新特性

在新发布的 Swift 5.7 版本中，根据 [Unlock existentials for all protocols](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md) 提案的描述，不在只是普通的协议可以作为类型的约束，PATs 也获得了这种能力。上面的代码在 Xcode 14 中的报错变为了：

> Use of protocol 'Generator' as a type must be written 'any Generator'
>
> Replace 'Generator' with 'any Generator'

当我们按照报错的指引将代码修改为下面的形式时，错误便消失了。

```swift
let generator: any Generator = StringGenerator()

func randomGenerator() -> any Generator { 
  if .random() {
    return IntGenerator()
  } else {
    return StringGenerator()
  }
}
```

错误虽然消失了，但问题并没有解决。当我们调用类型为 `any Generator` 变量的 `generate()` 方法时，生成的元素是何种类型呢？答案是，编译器也无法确定。

当 PATs 的 associatetype 没有任何类型约束的时候，只有用最大的容器才能装下未知类型的元素，此时生成元素的类型为 `any`：

```swift
// Declaration
// let intOrString: Any
let intOrString = randomGenerator().generate()
```

如果 PATs 对 associatetype 的类型有协议的约束时，生成元素的类型自然必须满足这些协议，例如将 `Generator` 的声明修改为如下形式，生成元素的类型也对应产生了变化：

```swift
protocol Generator {
  associatedtype Element: Equatable, Hashable, CustomStringConvertible

  func generate() -> Element
}

// Declaration
// let intOrString: CustomStringConvertible & Hashable
let intOrString = randomGenerator().generate()
```



## some Protocol 对比 any Protocol

从形式上看

```swift
protocol P {
  associatetype E
}

func foo() -> some P {
  ...
}

func bar() -> any P {
  ...
}
```

返回类型为 `some P` 的函数与返回类型为 `any P` 的函数有一些相似之处。其中前者是 [Opaque Result Types](https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md) 提案引入 Swift 5.1 版本的，后者是 Swift 5.7 版本新引入的。

从解决的问题看，二者都从一个角度解决了需要返回 PATs 类型的限制。

最大的不同其实是解决问题的思路。`some P` 提供的是编译器对类型的检查，对于一个返回值类型为 `some P` 的函数来说，可以返回任意满足 `P` 协议的对象，但所有返回的对象必须为同一类型。

```swift
func randomGenerator() -> some Generator { // Error: Function declares an opaque return type 'some Generator', but the return statements in its body do not have matching underlying types
  if .random() {
    return IntGenerator()
  } else {
    return StringGenerator()
  }
}
```

上述代码返回了两种不同的满足 `Generator` 协议的对象，编译器报错要求返回对象必须为同一类型。



> 题外话，为什么在 SwiftUI 中返回两种不同的 `View` 并无问题呢？
>
> ```swift
> struct FooView: View {
>   var body: some View {
>     if .random() {
>       Text("foo")
>     } else {
>       Color.red
>     }
>   }
> }
> ```
>
> 原因在于，`body` 的 block 并非普通的 block，而是被 `@ViewBuilder` 修饰的。
>
> ```swift
> public protocol View {
>     associatedtype Body : View
> 
>     @ViewBuilder @MainActor var body: Self.Body { get }
> }
> ```
>
> 最终这个 block 返回的也并非 `Text` 或 `Color` 类型，而是一个类型为 `SwiftUI._ConditionalContent<SwiftUI.Text, SwiftUI.Color>` 的容器，自然满足 `some View` 要求的返回同一类型。



而 `any P` 提供的是 PATs 作为容器类型的能力。之前变量或函数返回值的类型都不能是 PATs，现在 PATs 可以作为容器类型，而最终将不确定的类型传递到了从容器中取出的元素。最后从容器中取出元素的调用方判断取出元素的类型。这就是为什么说二者解决问题的思路不同。

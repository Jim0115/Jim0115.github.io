# @ViewBuilder，声明式 Swift 的关键

Swift 的目标为一门实用的工业语言，从其诞生之日起，便在对各种编程范式的支持上越走越远。面向对象编程（Object Oriented Programming）、函数式编程（Functional Programming）、泛型编程（Generic Programming）等思想都在 Swift 中得到了体现与运用。但同时作为 iOS 开发的官方语言， Swift 基于自身命令式编程的本质，并非编写 UI 的最佳方式。虽然苹果在 Xcode 中集成了 Interface Builder，并希望以此解决界面开发的问题，但由于 IB 在多人协作等方面的问题，始终未能在大规模项目中得到应用。因此在 WWDC 2019 中，苹果推出了 [SwiftUI](https://developer.apple.com/xcode/swiftui/) 作为一种新的用户界面实现方式，同时也向 Swift 中引入了新的编程范式 —— 声明式编程（Declarative Programming）。

## 命令式 VS 声明式

命令式编程和声明式编程有着本质上的区别，引用知乎的一个回答：

> - **命令式编程**：命令“机器”*如何*去做事情*(how)*，这样不管你想要的*是什么(what)*，它都会按照你的命令实现。
> - **声明式编程**：告诉“机器”你想要的*是什么(what)*，让机器想出*如何*去做(how)。

换句话说，作为机器语言的高度抽象，命令式编程的每一行代码都可以视为一条命令，也都需要有其执行结果。

```swift
override func viewDidLoad() {
    super.viewDidLoad()

    UILabel() // Warning: Result of 'UILabel' initializer is unused
}
```

命令式编程的确可以通过一条条语句创建控件、改变属性、添加层级并最终实现用户界面。但相较于声明式编程，始终缺乏直观性，在控件的可复用性、甚至代码量上也难以占到优势。

```swift
struct HelloWorldView: View {
    var body: some View {
        VStack {
            Text("Hello")
            Text("world")
            Text("!")
        }
    }
}
```

虽然从语法上看，SwiftUI 确实引入了一些声明式语言的特征，但 Swift 的本质仍是一门命令式编程语言，那么，这一切是如何实现的呢？

## Property Wrapper

为了提供后续理解的思路，暂时不脱离命令式编程的范畴，在 Swift 5.1 中引入的 [Property Wrapper](https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md) 便是与 ViewBuilder 相同思想的产物。Property Wrapper 提供了对属性（property）的封装，用一个常见需求便能很容易理解。

Swift 中的计算属性（Computed Properties）对应了其他语言中的重写 getter/setter。为了对设置做持久化，在 iOS 开发中用 `UserDefaults` 承载某个属性是一种常见情况。

```swift
class Settings {
    private static let AutoplayVideoUserDefaultsKey = "AutoplayViewUserDefaultsKey"
    
    var autoplayVideo: Bool {
        get {
            UserDefaults.standard.bool(forKey: Settings.AutoplayVideoUserDefaultsKey)
        }
        set {
            UserDefaults.standard.set(newValue, forKey: Settings.AutoplayVideoUserDefaultsKey)
        }
    }
}
```

@propertyWrapper 实现了对上述需求的抽象，同时封装了对 `UserDefaults` 的读写操作：

```swift
// https://stackoverflow.com/questions/59298225/
@propertyWrapper
struct UserDefaultWrapper<T> {
    let key: String
    let value: T

    init(key: String, value: T) {
        self.key = key
        self.value = value
    }

    var wrappedValue: T {
        get {
            return UserDefaults.standard.value(forKey: self.key) as? T ?? self.value
        }
        set {
            UserDefaults.standard.set(newValue, forKey: self.key)
        }
    }
}
```

基于 @propertyWrapper，上述代码可改写为：

```swift
class Settings {
    private static let AutoplayViewUserDefaultsKey = "AutoplayViewUserDefaultsKey"

    @UserDefaultWrapper(key: AutoplayViewUserDefaultsKey, value: false) var autoplayVideo: Bool
}
```



## ViewBuilder

```swift
public struct VStack<Content> : View where Content : View {
    @inlinable public init(alignment: HorizontalAlignment = .center, spacing: CGFloat? = nil, @ViewBuilder content: () -> Content)

    public typealias Body = Never
}
```

从 `VStack` 的构造方法看，其最后一个参数只是一个返回值为泛型类型 `Content` 的普通 block，唯一的不同点在于其之前的 `@ViewBuilder` 修饰。

继续深入查看 `ViewBuilder` 的实现：

```swift
@_functionBuilder public struct ViewBuilder {
    public static func buildBlock() -> EmptyView

    public static func buildBlock<Content>(_ content: Content) -> Content where Content : View
}
```

再参考上述 [Property Wrapper](#Property Wrapper) 章节的内容，可以看到 `@propertyWrapper` 与 `@_functionBuilder` 有着相同的设计思想，不过二者一个修饰属性，另一个修饰函数。

[Function builders]() 的提案可以参考此处，从介绍中可以看出其给 Swift 添加了 DSL 的支持。

## TupleView：背后的承载

继续 `ViewBuilder` 实现部分，这里以两个子 view 的构造为例：

```swift
extension ViewBuilder {
    public static func buildBlock<C0, C1>(_ c0: C0, _ c1: C1) -> TupleView<(C0, C1)> where C0 : View, C1 : View
}
```

在 SwiftUI 的文档 [View Layout and Presentation](https://developer.apple.com/documentation/swiftui/view_layout_and_presentation) 有着 Infrequently Used Views 一章。`AnyView` 作为一种常见的 `type-earse` 思想的产物不再赘述，`TupleView` 便是 SwiftUI 声明式语法背后的承载者。虽然不同于 Swift 语言本身，SwiftUI 暂未开源，但从接口上也可管中窥豹。

大胆假设：**`ViewBuilder` 的实现中将 block 中代码每一行所创建的 View 收集到一个 tuple 中，并构造一个 `TupleView` 用于承载所收集到的所有 View。**因此：

1. **SwiftUI 中的 View 只有一个直接持有的 subview。**
2. **对于有多个 subviews 的 View，通过 `TupleView` 中的 tuple 与其 subviews 建立联系。**
3. **对于只有一个 subview 的 View，则直接建立联系。**

求证：

通过 `Mirror` 可以看到属性背后的类型

```swift
struct HelloWorldView: View {
    var body: some View {
        VStack {
            Text("Hello")
            Text("world")
            Text("!")
        }
    }
}

struct HelloView: View {
    var body: some View {
        VStack {
            Text("Hello")
        }
    }
}

print(Mirror(reflecting: HelloView().body)) // Mirror for VStack<Text>
print(Mirror(reflecting: HelloWorldView().body)) // Mirror for VStack<TupleView<(Text, Text, Text)>>
```

## 最后的一个小问题

为什么任何 `View` 最多只能添加 **10** 个 subviews？

```swift
struct NumbersView: View {
    var body: some View {
        VStack {
            Text("0")
            Text("1")
            Text("2")
            Text("3")
            Text("4")
            Text("5")
            Text("6")
            Text("7")
            Text("8")
            Text("9")
            Text("10") // Error: Extra argument in call
        }
    }
}
```

答案很简单，从 `ViewBuilder` 的实现中可以看出：

```swift
@available(iOS 13.0, OSX 10.15, tvOS 13.0, watchOS 6.0, *)
extension ViewBuilder {
    public static func buildBlock<C0, C1>(_ c0: C0, _ c1: C1) -> TupleView<(C0, C1)> where C0 : View, C1 : View
    ...
    public static func buildBlock<C0, C1, C2, C3, C4, C5, C6, C7, C8, C9>(_ c0: C0, _ c1: C1, _ c2: C2, _ c3: C3, _ c4: C4, _ c5: C5, _ c6: C6, _ c7: C7, _ c8: C8, _ c9: C9) -> TupleView<(C0, C1, C2, C3, C4, C5, C6, C7, C8, C9)> where C0 : View, C1 : View, C2 : View, C3 : View, C4 : View, C5 : View, C6 : View, C7 : View, C8 : View, C9 : View
}
```

`ViewBuilder` 提供的静态方法 `buildBlock` 的重载中最多提供了接受 **10** 个参数的方法。由于参数的每个 View 都有其泛型类型，因此无法对整体的 tuple 再添加泛型。解决的方法同样很简单，通过抽出新的自定义 View 类型或通过 `Group` 将一维的列表拆分成二维，使每一层的 subviews 数量小于 10 即可。




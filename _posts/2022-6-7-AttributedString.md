---

layout: post

title: "AttributedString: 当然不是去掉了 NS 的 NSAttributedString"

---

`NSAttributedString` 是 iOS 开发过程中经常使用到的类型，然而对于 Swift 开发者来说，使用 `NSAttributedString` 是一件相当不方便的事情，原因在于 `NSAttributedString` 仍然带有浓烈的 Objective-C 风格。

在数年前的 Swift 3.0 版本中，Foundation 中的常用的数据类型，包括 `NSString` 、`NSDate`、 `NSData` 等都经历了从引用类型到值类型的转换，虽然可能背后实际的承载类型没变，但 API 的变化对于 Swift 开发者来说非常友好，再也不用考虑类似

```swift
let maybeMutable: NSMutableString = ...
var maybeImmutable: NSString = ...
```

之类的可变性问题（虽然这样的设计被 wangyin 点名批评）。详细的变化可以参考 [Mutability and Foundation Value Types](https://github.com/apple/swift-evolution/blob/master/proposals/0069-swift-mutability-for-foundation.md) 和 [Drop NS Prefix in Swift Foundation](https://github.com/apple/swift-evolution/blob/f55963fb0abd77aae643882ca3fc8939b1f969f2/proposals/0086-drop-foundation-ns.md)。

然而同在 Foundation 框架内，`NSAttributedString` 却没有在这次改动中被一同修改，具体的原因在上述引用文章的末尾可以找到：

> `NSAttributedString`: This is an obvious candidate for a value type. However, we want to take more time to get this one right, since it is the fundamental class for the entire text system. We will address it in a future proposal.

直到 iOS 15 版本，`AttributedString` 作为 `NSAttributedString` 的继任终于被推出。那么 `AttributedString` 仅仅是去掉了 “NS” 的 `NSAttributedString` 吗？

## 1 NSAttributedString 对于 Swift 开发者的不便

在使用 Swift 开发 iOS 时，`NSAttributedString`  的 API 带来了诸多不便：

### 1.1 NSAttributedString 与 NSMutableAttributedString

相比于其他数据类型，`NSAttributedString` 仍然与原有的 Objective-C 风格一致，带有一个可变类型 `NSMutableAttributedString`，并没有类似 `Set` 与 `NSSet` 的选择，因此会出现

```swift
let attributedString = NSMutableAttributedString()
```

这种不 Swift 的风格。

### 1.2 Attributes 缺少类型支持

`NSAttributedString` 所需的 Attributes 的类型在 Objective-C 中的类型为 `NSDictionary<NSAttributedStringKey, id> *`，桥接到 Swift 中就变成了 `[NSAttributedString.Key : Any]`。这个 Attributes 字典中的中的许多 key 都有着不同的 value 类型，在 Swift 中就要写成

```swift
let attributes: [NSAttributedString.Key : Any] =
    [.underlineStyle : NSUnderlineStyle.single.rawValue,
     .font : UIFont.systemFont(ofSize: 20),
     .foregroundColor : UIColor.red]
```

失去了类型系统的支持，非常容易出现类型错配，增加了不少开发成本。

### 1.3 Range 与 NSRange 的不兼容

给一个 `NSMutableAttributedString` 的特定范围添加 Attributes 在 Swift 中也不是一件容易的事，原因在获取NSRange 的困难，例如：

```swift
let attributedString = NSMutableAttributedString(string: "I'm learning Swift.")
let underlinedRange = (attributedString.string as NSString).range(of: "Swift")
print(underlinedRange) // {13, 5}

if underlinedRange.location != NSNotFound {
    let attributes: [NSAttributedString.Key : Any] = [.underlineStyle : NSUnderlineStyle.single.rawValue]
    attributedString.addAttributes(attributes, range: underlinedRange)
}
```

由于 Swift 中 `NSString` 已经被默认桥接成了 `String` 类型，调用其 `range(of: String)` 方法返回的类型为 `Range<String.Index>?`，而 `NSMutableAttributedString` 的 `addAttributes` 方法所需的 range 类型为 `NSRange`，因此必须显式转换为 `NSString` 类型后再调用方法。另外， `NSString` 的 `range(of: String)` 方法虽然返回 `NSRange`，但对于不熟悉 Objective-C 的开发者来说会很容易忘掉将 `range.location` 与 `NSNotFound` 比较的步骤。

更重要的是，选择使用 `NSString` 而不是 `String` 会失去 Swift.String 对于 Unicode 的良好支持，在类似下述极端的情况中可能会得到错误的 range 而无法实现预期效果：

```swift
let attributedString = NSMutableAttributedString(string: "👨‍⚕️ = 👨 + ⚕️")
let underlinedRange = (attributedString.string as NSString).range(of: "⚕️")
print(underlinedRange) // {3, 2}

if underlinedRange.location != NSNotFound {
    let attributes: [NSAttributedString.Key : Any] =
    [.underlineStyle : NSUnderlineStyle.single.rawValue]
    attributedString.addAttributes(attributes, range: underlinedRange)
}
```

（某些浏览器可能展示不出医生emoji，emoji 参见 [Man Health Worker](https://emojipedia.org/man-health-worker/)）

## 2 AttributedString 如何解决上述问题

### 2.1 AttributedString 是 struct

`AttributedString` 使用 Swift struct 实现，因此 `AttributedString` 与 `Data`、`String` 等类型一样有着值类型的特点：可变性由 let/var 决定；不会变意外修改；支持 copy-on-write 避免无谓复制开销等等。得益于此，`AttributedString` 不需要 Objective-C 风格的可变类型。同样因此，`AttributedString` 无法对 Objective-C 提供接口。

### 2.2 使用 AttributeContainer 代替 Attributes Dictionary

既然 Dictionary 无法针对不同的 key 指定不同的 value 类型，`AttributedString` 使用了新的 `AttributeContainer` 类型用于承载属性。`AttributeContainer` 同样是 Swift struct，因此 `AttributeContainer` 得到了许多 Swift 原生类型的能力使其更加易用。

对于最普遍的场景，`AttributeContainer` 可以作为一个带类型的 Dictionary 使用，例如：

```swift
var container = AttributeContainer()
container.underlineStyle = .single
container.font = .systemFont(ofSize: 20)
```

需要注意的一点是，`AttributeContainer` 是支持跨平台使用的，因此在设置 container 的属性时要明确设置的平台，例如：

```swift
var container = AttributeContainer()
container.font = Font.title
container.font = UIFont.systemFont(ofSize: 10)
print(container)
```

输出的结果是

```
{
	SwiftUI.Font = Font(provider: SwiftUI.(unknown context at $1ba609b68).FontBox<SwiftUI.Font.(unknown context at $1ba637910).TextStyleProvider>)
	NSFont = <UICTFont: 0x14b70afb0> font-family: ".SFUI-Regular"; font-weight: normal; font-style: normal; font-size: 10.00pt
}
```

原因在于，`font` 并非是定义在 `AttributeContainer` 的一个属性，而是 `AttributeContainer` 基于 `dynamicMemberLookup` 动态生成的属性，因此下面两行代码是等价的：

```swift
container.font = .systemFont(ofSize: 10)
container[dynamicMember: \.font] = .systemFont(ofSize: 10)
```

对于一些多平台共有的属性，可以通过显式指定 scope 的方式设置，在无歧义的情况下也可以通过显式指定类型设置：

```swift
container.uiKit.foregroundColor = .red // UIColor.red
container.swiftUI.foregroundColor = .red // Color.red

// or
container.foregroundColor = UIColor.red // UIKit
```

`AttributeContainer` 支持的不同 scope 如下：

- AttributeScopes.AccessibilityAttributes
- AttributeScopes.AppKitAttributes
- AttributeScopes.FoundationAttributes
- AttributeScopes.FoundationAttributes.NumberFormatAttributes
- AttributeScopes.SwiftUIAttributes
- AttributeScopes.UIKitAttributes

此外，通过 `dynamicMemberLookup` 与支持 `callAsFunction` 功能的 `Builder` 的组合，`AttributeContainer` 也支持链式语法：

```swift
let container = AttributeContainer
    .foregroundColor(UIColor.red)
    .font(UIFont.systemFont(ofSize: 20))

let newContainer = container.backgroundColor(UIColor.black)
```

### 2.3 AttributedString 直接使用 Range

由于不再需要考虑对 Objective-C 的兼容，`AttributedString` 不再需要 `NSRange` 做桥接，因此对于设置特定范围内属性的语法也更有 Swift 风格：

```swift
var attributedString = AttributedString("I'm learning Swift.")

var container = AttributeContainer()
container.uiKit.underlineStyle = .single

if let underlinedRange = attributedString.range(of: "Swift") {
    attributedString[underlinedRange].setAttributes(container)
}
```

同样遵从 Swift 的 API 设计规范，`AttributedString` 的修改方法都有在原对象上修改和返回修改后新对象的两种：

```swift
mutating func setAttributes(_ attributes: AttributeContainer)
func settingAttributes(_ attributes: AttributeContainer) -> AttributedString

mutating func mergeAttributes(_ attributes: AttributeContainer, mergePolicy: AttributedString.AttributeMergePolicy = .keepNew)
func mergingAttributes(_ attributes: AttributeContainer, mergePolicy: AttributeMergePolicy = .keepNew) -> AttributedString

mutating func replaceAttributes(_ attributes: AttributeContainer, with others: AttributeContainer)
func replacingAttributes(_ attributes: AttributeContainer, with others: AttributeContainer) -> AttributedString
```

## 3 如何使用 AttributedString

对于 iOS、macOS 等原本使用 `NSAttributedString` 的平台来说，从 iOS 15 版本开始，`NSAttributedString` 有了一组新的便捷构造方法，能够将 `AttributedString` 转换为 `NSAttributedString`：

```swift
convenience init(_ attrStr: AttributedString)
convenience init<S>(_ attrStr: AttributedString, including scope: S.Type) throws where S : AttributeScope
convenience init<S>(_ attrStr: AttributedString, including scope: KeyPath<AttributeScopes, S.Type>) throws where S : AttributeScope
```

在 SwiftUI 中，`Text` 控件同样添加了新的构造方法：

```swift
init(_ attributedContent: AttributedString)
```

需要注意的是，`AttributedString` 以及作用在 `Text` 控件上的 modifier 都能修改文字的属性，并且 <u>`AttributedString` 的优先级高于 modifier</u>。

## 4 AttributedString 的新功能不止于此

`AttributedString` 不仅仅提供了新的 API，还提供了一系列新功能，包括但不限于：

- 对 Codable 协议的完整支持
- 支持 MarkDown 语法
- 通过 `AttributedString.Runs` 拆分 `AttributedString`

有兴趣可以继续深入探索。



## 参考资料

[dynamicMemberLookup](https://docs.swift.org/swift-book/ReferenceManual/Attributes.html)

[callAsFunction](https://docs.swift.org/swift-book/ReferenceManual/Declarations.html)

[What's new in Foundation](https://developer.apple.com/videos/play/wwdc2021/10109/)

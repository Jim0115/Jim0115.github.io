---
layout: post
title: "走近科学：Combine 的 Map 为何会有两种不同的行为？"
---
# 走近科学：Combine 的 Map 为何会有两种不同的行为？

## 一个奇怪的现象

在一次日常研究 Combine 代码的过程中，无意中发现了一个不大符合认知的行为，示例代码如下：

```swift
let combineRandom3Publisher = ["It", "doesn't", "matter"].publisher
    .map { _ in Int.random(in: 0...100) }

combineRandom3Publisher.sink { print($0) }
combineRandom3Publisher.sink { print($0) }
```

按照正常的理解来说，`map` 作为一个普通且~~自信~~常用的 Operator，只负责用持有的 `transform` 处理上游传来的 value，然后传递到下游，不应该对传递的过程产生影响。然而。。。

```swift
combineRandom3Publisher.sink { print($0) } // Output: 13 79 26
combineRandom3Publisher.sink { print($0) } // Output: 13 79 26
```

这说明了 `map` 操作的行为是 `eager` 的，具体来说：**`map` 在没有收到下游订阅者的前提下，从上游索取了全部的 `value`，提前进行了 `transform` 并储存**。这是一种非常不合理的行为，因为上游产生一个新 `value` 的过程对下游来说是不可知的，上游可能只是从某个数组中取了某个值，也可能是通过复杂的不可重复的操作从网络获取了某些信息，类似的 `eager` 行为模式会造成不可预期的后果。

我感觉自己似乎发现了 `Combine` 的一个重大缺陷，于是当我把 `map` 接入到真正的网络请求上时：

```swift
let url = URL(string: "http://httpbin.org/uuid")!
let randomUUIDPublisher = URLSession.shared
    .dataTaskPublisher(for: url)
    .map { String(data: $0.data, encoding: .utf8)! }

let sub1 = randomUUIDPublisher.sink { completion in
    print(completion)
} receiveValue: { value in
    print(value) // "uuid": "704ded90-117a-45ea-a081-29d1c9cdd185"
}

let sub2 = randomUUIDPublisher.sink { completion in
    print(completion)
} receiveValue: { value in
    print(value) // "uuid": "993045c7-4815-45e7-ba5b-a4af4c2e2715"
}
```

对 `map` 两次不同的订阅产生了两个不同的结果，这说明 `map` 的行为又变回了预期的 `lazy`，`URLSession.shared.dataTaskPublisher` 确实收到了两个订阅，发了两次网络请求。同样的 `map` 操作符居然会有不同的行为，一时间我有点不知道该怎么解释这种现象。

## 知识点补充：`Sequence` 与 `LazySequence`

在猜测这种现象产生的原因之前，先忘掉 `Combine`，回顾一下 Swift 中对应的概念： `Sequence` 和 `LazySequence`。

在 Swift 标准库中，`map`、`filter` 或 `reduce` 等针对 `Sequence` 的操作符默认行为都是 `eager` 的，这符合大多数人对这些操作的理解。

```swift
let eagerSequence = [1, 2, 3, 4]
    .filter { $0.isMultiple(of: 2) } // [2, 4]
    .map { $0 * 2 } // [4, 8]

eagerSequence.forEach { print($0) } // Output: 4, 8
eagerSequence.forEach { print($0) } // Output: 4, 8

/*
filter: 1
filter: 2
filter: 3
filter: 4
map: 2
map: 4
4
8
4
8
*/
```

`eager` 作为默认行为的优势不言而喻，以 `map` 操作为例：<u>每个 `map` 操作对应的 `transform` 对于 Sequence 中的每个元素只会执行一次，后续对 `map` 结果的多次获取不会再进行 `transform` 操作</u>。这是一种用空间换时间的常见做法。

这种做法当然不是没有缺点，假设我对一个很大的数组进行 `map` 操作，且只需要结果的第一个元素：

```swift
let eagerSequence = Array(1...1000)
    .map { value -> Int in
        print("map:", value)
        return value * 2
    }

print(eagerSequence.first!) // Output: 2

/*
map: 1
map: 2
...
map: 999
map: 1000
2
*/
```

这里对于除了第一个元素之外的其他 `transform` 操作的开销都是多余的。因此，Swift 也提供了 `lazy` 操作符用于在出现类似需求是将行为变成 `lazy` 的：

```swift
let lazySequence = Array(1...1000).lazy
    .map { value -> Int in
        print("map:", value)
        return value * 2
    }

print(lazySequence.first!) // Output: 2

/*
map: 1
2
*/
```

简单来说，`lazy` 之后的 `map` 操作延迟了 `transform` 的执行时间到真正的取值开始。从具体实现上看，`lazy` 操作将 `Sequence<T>` 类型包装成了 `LazySequence<Sequence<T>>` 类型，在 `LazySequence` 上的 `map` 操作进一步将类型包装成 `LazyMapSequence<LazySequence<Sequence<T>>, U>`。类型一旦嵌套起来看起来可能会有点复杂，但只需要记住这些类型都是遵守原始的 `Sequence` 协议的，只是在行为上略有差别而已。

使用 `lazy` 行为当然也不是没有缺点的，同样是以 `map` 操作为例：<u>每个 `map` 操作对应的 `transform` 在真正的取值操作前不会执行，但后续对 `map` 结果的每次获取都会再进行一次 `transform` 操作</u>。可以理解为这是在用时间换空间。

从另一个角度也可以证明这两种行为的不一致：

```swift
// Sequence.map
func map<T>(_ transform: (Self.Element) throws -> T) rethrows -> [T]

// LazySequence.map
func map<U>(_ transform: @escaping (Base.Element) -> U) -> LazyMapSequence<Base, U>
```

`LazySequence.map` 的 `transform` 操作是 `escaping` 的，这表示它可以被持有，在函数之外发挥作用。而 `Sequence.map` 只会在函数内执行。

### 阶段总结

这两种行为没有优劣之分，一切都取决于后续的行为。简单的判断标准是：后续操作会用到序列中的大部分元素，例如 `max()`、`joined()`，用 `eager`。后续操作只会用到序列中的部分元素，例如 `first`、`prefix()`，用 `lazy`。

## 回到正题

既然已经了解了 `eager` 与 `lazy` 的不同行为模式，那么 `Combine` 中的 `map` 纠结为什么会有两种不同的行为模式呢？

### 猜想1： Publisher 无视 back pressure[^1] 强行发送元素

首先怀疑的就是 `["It", "doesn't", "matter"].publisher` 作为 `Publisher` 会不会强行向下游发送数据呢，创建一个不向上游获取元素的 `Subscriber` 确认一下：

```swift
class IDontNeedAnythingSubsciber<Input, Failure: Error>: Subscriber {
    func receive(subscription: Subscription) {
        // do nothing
    }

    func receive(_ input: Input) -> Subscribers.Demand {
        assertionFailure("should't receive input")
        return .none
    }

    func receive(completion: Subscribers.Completion<Failure>) {
        assertionFailure("should't receive completion")
    }
}

let subsciber = IDontNeedAnythingSubsciber<String, Never>()
["It", "doesn't", "matter"].publisher
    .subscribe(subsciber)
```

在 `IDontNeedAnythingSubsciber` 的 `func receive(subscription: Subscription)` 实现中，在收到订阅时不通过订阅向上游获取元素，用来测试猜想。答案是否定的，`IDontNeedAnythingSubsciber` 不会收到任何元素，排除了 `Publisher` 强行发送的可能性。

### 猜想2：在 Reactive 的 定义中，Map 的行为本来就是 eager 的

使用 `RxSwift` 实现一个同样的逻辑试验一下：

```swift
let rxRandom3Observable = Observable.from(["It", "doesn't", "matter"])
    .map { _ in Int.random(in: 0...100) }

rxRandom3Observable.subscribe(onNext: { print($0) }) // Output: 17 36 92
rxRandom3Observable.subscribe(onNext: { print($0) }) // Output: 22 5 53
```

答案也是否定的，`Map` 在 `RxSwift` 中的行为确实是 `lazy` 的。

### 猜想3：Map 有能力根据上游来源的不同，决定自己的行为方式

想要验证这种猜想也很简单，尝试在 `map` 操作之前抹掉上游的类型：

```swift
let combineRandom3Publisher = ["It", "doesn't", "matter"].publisher
    .eraseToAnyPublisher()
    .map { _ in Int.random(in: 0...100) }

combineRandom3Publisher.sink { print($0) } // Output: 11 46 69
combineRandom3Publisher.sink { print($0) } // Output: 19 90 93
```

这次的结果果然发生了变化，`map` 的行为变成了 `lazy` 的。

## 那么，一切是怎么实现的呢？

对比一下两次的代码：

```swift
// eager
let combineRandom3Publisher = ["It", "doesn't", "matter"].publisher
    .map { _ in Int.random(in: 0...100) }
    
// lazy
let combineRandom3Publisher = ["It", "doesn't", "matter"].publisher
    .eraseToAnyPublisher()
    .map { _ in Int.random(in: 0...100) }
```

看来一切的根源在于，两次调用的并不是同一个 `map` 方法！

```swift
// eager
func map<T>(_ transform: (Elements.Element) -> T) -> Publishers.Sequence<[T], Failure>

// lazy
func map<T>(_ transform: @escaping (String) -> T) -> Publishers.Map<AnyPublisher<Publishers.Sequence<[String], Never>.Output, Never>, T>
```

在 `Publishers.Sequence` 上调用的 `map` 方法依然返回了一个 `Publishers.Sequence` 类型的 `Publisher`，这个 `map` 的行为是 `eager` 的，`transform` 参数也没有 `@escaping` 修饰。

在 `AnyPublisher` 上调用的 `map` 方法返回的是一个 `Publishers.Map` 类型的 `Publisher`，这次的行为是 `lazy` 的，`transform` 参数被 `@escaping` 修饰，表示其可能会被持有留到后续使用。

### 还有其他的类型有类似的特性吗？

有。以下类型重写了 `Publish` 协议中 `map` 方法的默认实现，并改变了行为方式：

```swift
// Just
func map<T>(_ transform: (Output) -> T) -> Just<T>

// Publishers.Sequence
func map<T>(_ transform: (Elements.Element) -> T) -> Publishers.Sequence<[T], Failure>
```

以下类型重写了方法，但是只是为了减少类型的嵌套，没有修改 `map` 行为方式：

```swift
// Publishers.CompactMap
func map<T>(_ transform: @escaping (Output) -> T) -> Publishers.CompactMap<Upstream, T>

// Publishers.Map
func map<T>(_ transform: @escaping (Output) -> T) -> Publishers.Map<Upstream, T>

// Publishers.TryMap
func map<T>(_ transform: @escaping (Output) -> T) -> Publishers.TryMap<Upstream, T>
```

### 还有其他操作符被重写吗？

也有，但主要集中在 `Just` 和 `Publishers.Sequence` 两个类型中。二者重写了大部分对 `value` 进行操作的方法，修改了类似 `transform` 参数的行为，同时将返回值修改为了具体的类型。所以在使用这两种类型的时候，优先考虑操作符的行为是否和默认行为有差异。如果有需要默认行为的地方，可以考虑使用 `eraseToAnyPublisher` 操作符抹掉具体的类型。

## 参考资料

一篇介绍集合 lazy 行为的文章：https://www.avanderlee.com/swift/lazy-collections-arrays/

[^1]:Apple 关于 back pressure 的官方文档：https://developer.apple.com/documentation/combine/processing-published-elements-with-subscribers

Combine 中的各种 Publisher：https://developer.apple.com/documentation/combine/publishers


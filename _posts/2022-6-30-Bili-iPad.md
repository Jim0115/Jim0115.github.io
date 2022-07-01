---

layout: post

title: "B站 iPad 端的迷之滑动交互是如何实现的？"

---



作为一个 Bilibili 的重度用户，同时也是 iPad 的重度用户，我的日均使用时间已经到了 3.5 个小时。相比于可能被官方更加看重的使用推荐序的首页，反而是传统的时间序的关注 tab 更适合我个人的口味。然而在 iPad 上，关注 tab 的 scrollView 有一个奇怪的交互。

![IMG_C816F6BEF338-1](/asserts/bili-ipad/img1.jpg)

图中红色标记的部分是被关注的 up 主们，点击可以在蓝色部分显示对应 up 主的动态详情。理所当然，红蓝两个部分都是支持纵向滑动的 scrollView。然而在第一次看到这样的界面时，我深深鄙视了b站 UI 设计的水平，完全没有利用更宽的屏幕显示出更多的有效内容，而是在两边留了无意义的空间。然而奇怪的地方在于，看似是两边留白的绿色，在使用手指拖动时却能带动蓝色区域滑动。本文并非讨论这种交互的优劣或是设计出发点，只是从技术角度讨论该如何实现这种交互。

## 猜想一：蓝色区域与绿色区域同属一个 scrollView

最先想到，也是最容易实现的方式就是使用一点障眼法：让 scrollView 的内容只显示在屏幕中部，同时调整 scroll Indicator 的位置，使其位于合适的位置。一种简单的实现方式如下：

```swift
override func viewDidLoad() {
  super.viewDidLoad()

  let scrollView = UIScrollView()
  view.addSubview(scrollView)
  // scrollView 占据全部空间
  scrollView.snp.makeConstraints { make in
      make.edges.equalToSuperview()
  }

  // scrollView 只能纵向滑动
  scrollView.contentLayoutGuide.snp.makeConstraints { make in
      make.width.equalTo(scrollView.frameLayoutGuide.snp.width)
  }

  let stackView = UIStackView()
  stackView.axis = .vertical
  stackView.spacing = 5
  scrollView.addSubview(stackView)
  stackView.snp.makeConstraints { make in
      make.top.bottom.centerX.equalTo(scrollView.contentLayoutGuide)
  }

  let cellWidth: CGFloat = 800
  for i in 0...100 {
      var config = UIListContentConfiguration.cell()
      config.text = i.description

      let cell = UIListContentView(configuration: config)
      cell.backgroundColor = .lightGray
      cell.snp.makeConstraints { make in
          make.height.equalTo(44)
          make.width.equalTo(cellWidth)
      }

      stackView.addArrangedSubview(cell)
  }

  // 调整 verticalScrollIndicatorInsets 使 scrollIndicator 位于合适位置
  scrollView.verticalScrollIndicatorInsets.right = (view.bounds.width - cellWidth) / 2
}
```

实现出来的效果：

![custom_impl_1](/asserts/bili-ipad/img2.png)

这种方式确实可以实现对应的效果，但可以确定B站客户端没有使用这种方式，从一个细节就能看出。熟悉 iOS 系统交互的都了解，默认情况下点击 scrollView 对应纵向位置顶部的状态栏可以让 scrollView 回到顶部。然而点击图中顶部状态栏绿框部分不能回到顶部，点击蓝框部分却可以，这表明 scrollView 的大小确实只有蓝色区域，并非整个屏幕大小。（点击状态栏滑动到顶部的交互可以很容易地通过 `UIScrollView.scrollToTop` 属性开关，想做到部分区域开关却并不容易。）

![IMG_C816F6BEF338-2](/asserts/bili-ipad/img3.jpg)

## 猜想二：修改 touch 事件的传递

既然确认了 scrollView 的大小，就能看出实际上的触摸是在 scrollView 之外的，自然而然怀疑的对象就变成了 touch 事件的传递过程。那么能否通过修改 touch 事件的传递实现相应的效果呢？

最容易想到的思路是通过修改 scrollView 的 superview，重写其 `hitTest:withEvent:` 方法，当触摸发生在 superview 自身而不是其他 subview 上时，返回指定的 scrollView。

```swift
class ExpandableView: UIView {
    var expandedView: UIView?

    override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        let originalView = super.hitTest(point, with: event)
        if let expandedView = expandedView, originalView == self {
            return expandedView
        }
        return originalView
    }
}
```

测试证明，可以实现对应效果。

### 知其所以然

这种方式确实可以实现效果，那么为什么可以呢？

按照苹果官方文档的描述，`hitTest:withEvent:` 方法的系统实现应该如下：

```swift
func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
    guard isUserInteractionEnabled && !isHidden && alpha > 0.01 else {
        return nil
    }

    guard self.point(inside: point, with: event) else {
        return nil
    }

    for subview in subviews.reversed() {
        let convertedPoint = subview.convert(point, from: self)
        if let hitTestView = subview.hitTest(convertedPoint, with: event) {
            return hitTestView
        }
    }

    return self
}
```

按照此实现推断，如果我们按照上述方式重写 scrollView 的 superview 的 `hitTest:withEvent:` 方法，原本应该是 superview 自身处理的 touch 事件全部被我们指定的 `expandedView` 处理了，自然在 superview 上滑动得到的结果就是 scrollView 滑动。

### 既然 touch 事件被传递给了 tableView，为什么点击 superview 不会触发 cell 的点击呢？

答案其实也很简单，虽然我们重写了 superview 的 `hitTest:withEvent:` 方法，但 tableView 还是维持系统默认行为。从 tableView 的视角来看，点击的位置其实是发生在自身之外，因此其 `pointInside:withEvent:` 还是按照默认行为返回 false，自然其上的任何事件都不会触发。重写方法添加 log 也可印证这种猜想。

```swift
// 点击 tableView 左侧
point(inside:with:) point: (-50.0, 100.0) return: false
point(inside:with:) point: (-50.0, 100.0) return: false
point(inside:with:) point: (-50.0, 100.0) return: false

// 点击 tableView 右侧
point(inside:with:) point: (911.5, 126.0) return: false
point(inside:with:) point: (911.5, 126.0) return: false
point(inside:with:) point: (911.5, 126.0) return: false

// 点击 tableView
point(inside:with:) point: (89.5, 62.5) return: true
point(inside:with:) point: (89.5, 62.5) return: true
point(inside:with:) point: (89.5, 62.5) return: true
point(inside:with:) point: (89.5, 62.5) return: true
point(inside:with:) point: (89.5, 62.5) return: true
point(inside:with:) point: (89.5, 62.5) return: true
```

## 拓展一下

虽然在大部分情况下我们都可以依靠系统的默认行为实现交互，但在一些特殊场景时还是需要做一些对应修改。

### 增加/缩小某些 view 的有效范围

既然  `hitTest:withEvent:` 方法内部是使用 `pointInside:withEvent:` 方法判断，那么重写 `pointInside:withEvent:` 方法自然可以变化 view 的有效范围，甚至到 view 的边界之外。

```swift
class FooView: UIView {
    var exInsets = UIEdgeInsets.zero

    override func point(inside point: CGPoint, with event: UIEvent?) -> Bool {
        let expandedBounds = bounds.inset(by: exInsets)
        return expandedBounds.contains(point)
    }
}
```

需要注意三点：

1. `exInsets` 的正值代表边界内缩，也即触摸事件的有效范围减小，反之则是增大
2. 需要注意与 sibling views 之间的关系，从 `hitTest:withEvent:` 的默认实现可以看出，排在 `subviews` 数组后面（即位置更高）的 view 优先响应，如果有效范围扩大到与 sibling view 相交，要考虑响应顺序。
3. 即使扩大一个 view 的范围也不能扩大到 superview 的边界之外，因为 superview 自身就不会响应边界之外的 touch 事件，除非也对应地扩大 superview 的范围。

### 实现一个 view 不响应触摸事件，却不影响其 subviews

有时候可能会遇到的情况是，给一个指定的 view 添加蒙层，而其上的 button 不添加。因此蒙层 view 自身不响应事件，其 subviews 却需要响应。通过重写蒙层 view 的 `hitTest:withEvent:` 方法也能很容易地实现这种效果。

```swift
override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
    guard let hitTestView = super.hitTest(point, with: event), hitTestView != self else {
        return nil
    }
    return hitTestView
}
```




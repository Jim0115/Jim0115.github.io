---
layout: post
title: "UIView 的 frame, bounds, center 与 transform"
---

### tl;dr
如果没有修改过 view 的 `transform`，不需要使用 `bounds` 和 `center`。如果修改过 `transform`，不要使用 `frame`。

## frame
~~~swift
var frame: CGRect { get set }
~~~
`UIView` 的 `frame` 属性代表一个 view 在其 superview 坐标系中的位置和大小。修改 `frame` 会导致其 `bounds` 和 `center` 的值对应变化。
#### 注意
**如果一个 view 的 `transform` 属性不为 `.identity`，对其 `frame` 的读写结果都是未定义的。**

~~~swift
let transformView = UIView(frame: .init(x: 50, y: 50, width: 100, height: 100))

transformView.frame // (50, 50, 100, 100)
transformView.bounds // (0, 0, 100, 100)
transformView.center // (100, 100)

transformView.transform = transformView.transform.rotated(by: .pi / 4)

transformView.frame // (29.289, 29.289, 141.421, 141.421)
transformView.bounds // (0, 0, 100, 100)
transformView.center // (100, 100)

~~~

## center
~~~swift
var center: CGPoint { get set }
~~~

与 `frame` 类似，`center` 属性代表一个 view 的中心点在其 superview 坐标系中的位置，在 `transform` 为 `.identity`的情况下，等价于 `(frame.midX, frame.midY)`。修改 `center` 的值会使 `frame` 做对应变化，准确的说，` center += delta <==> frame.origin += delta`。  
在大多数情况下，对 `center` 的修改都可以通过修改 `frame` 达到同样的效果，除非需要指定 view 的中心点，或是修改 `transform` 不等于 `.identity` 的 view。

## bounds
~~~swift
var bounds: CGRect { get set }
~~~
`bounds` 描述一个 view 在其自身坐标系中的位置和大小。  
默认情况下，`bounds` 与 `frame` 满足下列关系：

~~~swift
bounds.origin == .zero
bounds.size == frame.size
~~~
在 `transform != .identity` 时，读写 `frame`是未定义行为，这种情况下可以通过修改 `bounds.size` 来修改 view 的大小。
#### bounds.origin 一定是 (0, 0) 吗
并不是。只是因为 `bounds` 的默认值为 `((0, 0), frame.size)`，而且通常情况下没有修改 `bounds.origin` 的必要。`bounds` 作为对 view 自身坐标系的描述，修改 `bounds.origin` 会影响其 subview 的位置。 
 
具体来说，假设 view A 有 subview B，且 `A.frame = (0, 0, 50, 50)`，`B.frame = (0, 0, 10, 10)`，则默认情况下 view B 位于 view A 的左上角，换句话说，`B.frame.origin == .zero` 表示 view B 的原点位于 view A 的原点处。修改 `A.bounds.origin = (-10, -10)` 会使得 view A 的坐标系原点发生偏移，因此 view B 的原点看似出现在了 (10, 10) 的位置上。  
 
同理，由于绘制的区域是 `view.bounds`，修改 `view.bounds.origin` 会使绘制区域发生类似的偏移。

## transform
~~~swift
var transform: CGAffineTransform { get set }
~~~
`transform` 用于对 view 进行仿射变换，包括偏移、缩放和旋转，通常在动画中使用。  
由于 Auto Layout 基于 `frame`，修改 `transform` 属性不会对其产生影响。 
#### 仿射变换 (affine transformation) 的定义
> 详见 [Affine transformation](https://en.wikipedia.org/wiki/Affine_transformation)

#### CGAffineTransform 实际上是一个 3x3 矩阵
通常情况下，得益于系统提供的良好的 API 封装，在使用 `transform` 时并不需要了解太多仿射变换及其背后的原理。只需要根据对应的操作选择合适的构造方法：

~~~swift
init(rotationAngle angle: CGFloat) // 旋转
init(scaleX sx: CGFloat, y sy: CGFloat) // 缩放
init(translationX: CGFloat, y: CGFloat) // 移动
~~~
在其背后，`CGAffineTransform` 实际上是一个如图所示的 3x3 矩阵
![matrix](/asserts/frame-bounds-center-transform/Affine.png)

对于view中的任意一点 `(x, y)`，最终的显示都会经过 `transform` 的转换 ![transform](/asserts/frame-bounds-center-transform/Transform.png)  
得到![transform](/asserts/frame-bounds-center-transform/Result.png) 

需要注意的一点是，对 `transform` 的操作并不满足交换律，即操作顺序会对最后的结果产生影响。本质上来说，多次操作等价于多个仿射矩阵做乘法，而矩阵乘法通常不满足交换律。

#### 变换的中心
对 view 的 transform 变换的中心点取决于 `view.layer.anchorPoint`，其默认值为 (0.5, 0.5)，即变换的中心点为 `view.center`。因此，上图中所指 (x, y) 均为相对于 `view.center` 的坐标。

#### 不再可用的 frame
上文中提到，在 view 的 `tranform` 属性不为 `.identity` 时，对 `frame` 的读写是未定义的。实际上，此时的 `frame` 属性为经过 `transform` 变换后的值。仍以上述 view A (0, 0, 50, 50), subview B (0, 0, 10, 10) 举例。

默认情况下 `B.frame = (0, 0, 10, 10)`。当设置 `B.transform = CGAffineTransform(scaleX: 2, y: 2)` 时，`B.frame` 变成了 (-5, -5, 20, 20)，这正是以 `B.center` 为变换中心进行变换后得到的结果。旋转之后所得的 `frame` 属性为刚好能够容纳旋转后图形大小的矩形，由于计算复杂，这里就不再赘述。虽然变换之后的 `frame` 属性的值有法可循，但还是应该按照文档中的建议不应对其进行操作。

### 参考资料
[View Programming Guide for iOS](https://developer.apple.com/library/archive/documentation/WindowsViews/Conceptual/ViewPG_iPhoneOS/WindowsandViews/WindowsandViews.html#//apple_ref/doc/uid/TP40009503-CH2-SW1)  
[UIView - UIKit | Apple Developer Documentation](https://developer.apple.com/documentation/uikit/uiview)

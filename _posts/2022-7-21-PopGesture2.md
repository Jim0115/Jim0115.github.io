---

layout: post

title: "（续）如何实现非边缘侧滑返回手势？"

---

书接上文，既然非边缘侧滑返回手势在不违背苹果交互原则的前提下，可以解决大屏幕手机上滑动返回操作困难的问题，那么应该如何实现呢？

## 思路一：传递滑动事件

系统在 `UINavigationController` 中提供了 `interactivePopGestureRecognizer` 属性用于获取边缘滑动返回的手势，那么，很容易就能想到，如果同时添加一个非边缘的滑动手势，将这个手势的事件传递给原有的边缘滑动返回手势，即可实现所需效果。既然如此，具体的代码要怎么写呢？

好消息是，这种思路确实可行；坏消息是，实现这种思路需要使用到私有 API。我们先抛开审核问题，看看应该如何实现。

从所有手势的父类 `UIGestureRecognizer` 的构造方法 `init(target: Any?, action: Selector?)`  可以看出，构造一个 `UIGestureRecognizer` 需要提供一个目标（target）以及其上的一个方法（action）用于当手势识别时调用，那么这个目标与方法的二元组应该会储存在 `UIGestureRecognizer ` 内部。从 [UIGestureRecognizerTarget.h](https://github.com/nst/iOS-Runtime-Headers/blob/master/PrivateFrameworks/UIKitCore.framework/UIGestureRecognizerTarget.h) 可以看出这个二元组被封装在一个名为 `UIGestureRecognizerTarget` 的对象内，从 [UIGestureRecognizer.h](https://github.com/nst/iOS-Runtime-Headers/blob/master/PrivateFrameworks/UIKitCore.framework/UIGestureRecognizer.h) 可以看出 `UIGestureRecognizer` 内部有一个名为 `_targets`，类型为 `NSMutableArray` 的私有属性储存一个 `UIGestureRecognizer` 所有的目标与方法二元组。既然如此，实现就很简单了，添加一个新的非边缘的滑动手势，将其 `_targets` 设置为与原 `interactivePopGestureRecognizer` 相同即可。

```swift
class CustomNavigationViewController: UINavigationController {
    private var secondaryPopGestureRecognizer = UIPanGestureRecognizer()

    override func viewDidLoad() {
        super.viewDidLoad()
        guard let interactivePopGestureRecognizer = interactivePopGestureRecognizer,
              let targets = interactivePopGestureRecognizer.value(forKey: "targets") else {
            return
        }
      
        view.addGestureRecognizer(secondaryPopGestureRecognizer)

        secondaryPopGestureRecognizer.setValue(targets, forKey: "targets")
        secondaryPopGestureRecognizer.delegate = self
    }
}
```

同时，在实测时发现，如果在 `UINavigationController` 中只有一个子 ViewController 的情况下触发 `_target`，会导致 `UINavigationController` 出现异常，无法继续 push 新 ViewController，因此需要控制自定义 `secondaryPopGestureRecognizer` 何种情况下应当触发。另外，对于这个 API 的使用者来说，禁用 `interactivePopGestureRecognizer` 可能代表着对应场景下不需要侧滑返回手势，因此新增的非边缘手势最好也同步禁用。综上，需要补充新增手势的触发判断条件：

```swift
extension CustomNavigationViewController: UIGestureRecognizerDelegate {
    func gestureRecognizerShouldBegin(_ gestureRecognizer: UIGestureRecognizer) -> Bool {
        guard let interactivePopGestureRecognizer = interactivePopGestureRecognizer else {
            return false
        }
        return interactivePopGestureRecognizer.isEnabled && viewControllers.count > 1
    }
}
```



虽然这种实现方式能够最大限度的利用系统的原有机制，却不可避免地需要使用私有 API，有不小的审核风险。那么，有没有其他的实现方式呢？



## 思路二：自定义 Pop 转场

另一种思路是，将自定义 Pan 手势与 pop 转场动画进行关联，各种参数设置合理的话，可以给使用者一种界面跟随手指同步的感觉。这种思路的实现方式相对来说就更加复杂，主要分以下几部分。

首先是通过 `UINavigationController.delegate` 向 navigationController 提供自定义的专场动画实现：

```swift
extension CustomNavigationViewController: UINavigationControllerDelegate {
    func navigationController(_ navigationController: UINavigationController, animationControllerFor operation: UINavigationController.Operation, from fromVC: UIViewController, to toVC: UIViewController) -> UIViewControllerAnimatedTransitioning? {
        transitioning.operation = operation
        return transitioning
    }

    func navigationController(_ navigationController: UINavigationController, interactionControllerFor animationController: UIViewControllerAnimatedTransitioning) -> UIViewControllerInteractiveTransitioning? {
        if let interactiveTransitioning = animationController as? CustomTransitioning,
           interactiveTransitioning.operation == .pop {
            return interactiveTransitioning
        }
        return nil
    }
}
```

这里的实现方式有很多种，可以只针对 pop 场景提供自定义转场，为了保持一致，也可以为 push/pop 提供相反的动画。需要注意的是，`interactionAnimationController` 的优先级要更高，如果提供了满足 `UIViewControllerInteractiveTransitioning` 协议的对象，则原本的 `UIViewControllerAnimatedTransitioning` 协议中的方法就不会被调用。



`CustomTransitioning` 同样有多种实现方式，这里选择的是基于系统提供的 `UIPercentDrivenInteractiveTransition` 作为基类。

```swift
extension CustomTransitioning {
    override func startInteractiveTransition(_ transitionContext: UIViewControllerContextTransitioning) {
        context = transitionContext

        let fromVC = transitionContext.viewController(forKey: .from)!
        let fromView = transitionContext.view(forKey: .from)!
        let toView = transitionContext.view(forKey: .to)!

        let containerView = transitionContext.containerView

        let fromViewInitialFrame = transitionContext.initialFrame(for: fromVC)

        containerView.addSubview(toView)
        toView.frame = fromViewInitialFrame.offsetBy(dx: -fromViewInitialFrame.width, dy: 0)

        animator.addAnimations {
            fromView.frame.origin.x = fromView.frame.width
            toView.frame.origin = .zero
        }

        animator.addCompletion { _ in
            transitionContext.finishInteractiveTransition()
            transitionContext.completeTransition(true)
        }

        animator.startAnimation()
    }

    override func update(_ percentComplete: CGFloat) {
        animator.stopAnimation(true)

        let toVC = context.viewController(forKey: .to)!
        let fromView = context.view(forKey: .from)!
        let toView = context.view(forKey: .to)!

        let containerView = context.containerView
        containerView.addSubview(toView)

        let toViewEndFrame = context.finalFrame(for: toVC)

        let width = fromView.frame.width

        toView.frame = toViewEndFrame
        toView.frame.origin.x = width * (percentComplete - 1)
        fromView.frame.origin.x = width * percentComplete
    }

    override func cancel() {
        let fromVC = context.viewController(forKey: .from)!
        let fromView = context.view(forKey: .from)!
        let toView = context.view(forKey: .to)!

        let containerView = context.containerView

        let fromViewInitialFrame = context.initialFrame(for: fromVC)

        let width = fromView.frame.width

        UIView.animate(withDuration: 0.25) {
            fromView.frame = fromViewInitialFrame
            toView.frame = fromViewInitialFrame.offsetBy(dx: -width, dy: 0)
        } completion: { _ in
            self.context.cancelInteractiveTransition()
            self.context.completeTransition(false)
        }

    }

    override func finish() {
        let fromView = context.view(forKey: .from)!
        let toView = context.view(forKey: .to)!

        UIView.animate(withDuration: 0.25, delay: 0, options: .curveEaseOut) {
            fromView.frame.origin.x = fromView.frame.width
            toView.frame.origin = .zero
        } completion: { _ in
            context.finishInteractiveTransition()
            self.context.completeTransition(true)
        }
    }
}
```

这里给出的实现只是一个最简单的水平滑动，为了实现更加细腻的转场，通常在动画过程中会添加许多类似透明度、缩放、蒙层等效果，可以具体情况具体分析。

最后是将自定义的滑动手势与转场动画进行关联：

```swift
@objc func handleSecondaryPopGesture(gesture: UIPanGestureRecognizer) {
        let translationX = gesture.translation(in: nil).x
        let progress = min(max(0, translationX / view.bounds.width), 1)

        switch gesture.state {
        case .began:
            popViewController(animated: true)
        case .changed:
            transitioning.update(progress)
        case .ended:
            if progress > 0.5 {
                transitioning.finish()
            } else {
                transitioning.cancel()
            }
        case .cancelled:
            transitioning.cancel()
        default:
            break
        }
    }
```

最终呈现的效果：

<video autoplay=true src="/asserts/PopGesture/Demo.mp4" controls loop=true width="50%"></video>

### 思路二最需要注意的地方：判断条件

对于动画的审美可能因人而异，有人觉得最简单的横向滑动就足够，有人追求绚丽的转场效果。然而个人认为在这种实现方式中最重要的点在于返回成功与否的判断条件。

可以看到在上面的实例代码中，将滑动结束时究竟是 cancel 还是 finish 的判断条件简单粗暴地设置为 progress > 0.5，也即滑动的距离大于屏幕宽度的一半。然而这就意味着用户如果从屏幕的右半边开始滑动，无论如何也无法完成 pop，即使从屏幕左侧开始，也一样需要滑动很长的距离。如果这种实现放到 iPad 上，效果更是灾难性的。

一种可行的优化思路是，设置一个较小的固定值，滑动距离超过这个值即完成 pop。或是将比例缩小，例如滑动超过屏幕宽度 1/3 即认为完成 pop。这些都不失为更好的方案，只是需要注意越小的滑动距离意味着更大的误操作概率，越大的滑动距离则增加了操作的困难性，如何取得最佳的用户体验还是需要严谨的实验确定。

另一种优化思路是，综合考虑滑动距离以及手指离开屏幕时的速度。从 `UIPanGestureRecognizer ` 的 `func velocity(in view: UIView?) -> CGPoint` 方法可以得到滑动任意状态时的速度信息，例如出现滑动距离很小却有很大的向右速度是认为完成 pop，或是滑动距离很大同时有很大的向左速度是认为取消 pop 等等。这种处理方式也有其合理性。

## 最后需要关注的点

无论使用思路一还是思路二的方式实现需求，都不可避免的给应用增加了全局的滑动手势，需要格外关注手势冲突的可能性，有可能是具体 view 上根据业务需要添加的滑动手势，以及包括 TableViewCell 的 leadingSwipeActions、ScrollView 的 `panGestureRecognizer`、PageViewController 的 `gestureRecognizers` 等的系统默认添加的手势。

不要因为全局滑动手势的存在导致上述控件的功能缺失，也不要走到另一个极端，只要有手势冲突可能性的页面一律禁用全局返回手势，具体判断每一种情况，做出用户体验最佳的选择。



最后的最后，无论是选择自定义的非边缘滑动手势，系统提供的边缘滑动手势，还是干脆全局不支持滑动返回，都要保证 App 内体验的一致。某些页面可以滑动返回，有些页面不可以，这种页面间割裂的体验实际效果甚至不如全局不支持滑动返回。

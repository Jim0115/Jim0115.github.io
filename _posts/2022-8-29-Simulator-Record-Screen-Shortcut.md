---

layout: post

title: "iOS 开发小技巧：修改 iOS 模拟器 Simulator 的录屏快捷键"

---

从 [Xcode 12.5](https://developer.apple.com/documentation/xcode-release-notes/xcode-12_5-release-notes) 版本开始，Xcode 自带的 Simulator 在原有截屏的基础上，新增了录屏功能。这个功能确实给开发阶段的沟通带来了方便，之前类似的需求需要使用 macOS 自带的录屏来实现。然而美中不足的是，Simulator 默认的录屏启动快捷键是 `⌘ + R`。众所周知，Xcode 的运行快捷键也是 `⌘ + R`，很容易在没意识到的情况下启动了 Simulator 的录屏。虽然 Simulator 没有像 Xcode 一样提供修改快捷键的设置，然而经过一番摸索，我还是找到了在系统层面进行修改的方法。

![Settings-1](/asserts/simulator-shortcuts/Settings-1.png)

（以下设置基于 macOS 12.5.1 和 Xcode 13.4，macOS 13 的系统设置有不小的界面改动，但操作流程不变）

首先打开 macOS 的系统偏好设置，找到键盘设置：

![Settings-1](/asserts/simulator-shortcuts/Settings-1.png)

在键盘设置里找到 App 快捷键选项：

![Settings-2](/asserts/simulator-shortcuts/Settings-2.png)

点击右侧下方的 “+” 按钮，弹出 App 选择界面：

![Settings-4](/asserts/simulator-shortcuts/Settings-4.png)

由于 Simulator.app 不在默认的 /Applications 路径下，因此需要点击下拉菜单中的最后一项”其他...“，手动选择 Simulator.app 的位置。这里可以在文件选择窗口的右上角搜索“simulator”，也可以通过文件路径 `/Applications/Xcode.app/Contents/Developer/Applications/Simulator.app` 直接找到 simulator.app：

![Settings-5](/asserts/simulator-shortcuts/Settings-5.png)

之后，在菜单标题输入完整的命令名称，由于 Simulator.app 并没有做国际化，这里直接输入英文命令 `Record Screen`即可。最后，在最下面的“键盘快捷键”输入框中设置新的快捷键，这里以 `⌥ + ⌘ + R` 为例：

![Settings-6](/asserts/simulator-shortcuts/Settings-6.png)

点击保存，可以看到修改已经完成，回到 Simulator.app 也能看到修改已经生效。

![Settings-7](/asserts/simulator-shortcuts/Settings-7.png)

![ShortCut-2](/asserts/simulator-shortcuts/ShortCut-2.png)



按照同样的逻辑，我们也可以给不知出于何种原因消失的动画慢动作选项设置原来的快捷键 `⌘ + T`：

![Shortcut-3](/asserts/simulator-shortcuts/Shortcut-3.png)

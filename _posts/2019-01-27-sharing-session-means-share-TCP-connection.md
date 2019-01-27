---
title: URL Loading System 篇1：复用 Session 就是复用 TCP 连接？
layout: post
---

在官方弃用  `URLConnection` 转为推荐使用 `URLSession` 之后，网络库抽象层级变得更高，也不再有**连接**这一概念存在了。但是从根本上来说，HTTP 仍需要建立 TCP 连接用于收发数据。那么，从高度抽象的 `URLSession` 到更加通用的 TCP 之间有无直接的联系呢？在官方强烈推荐复用 `session` 的前提下，复用 `session` 究竟复用了什么呢？让我们先从 TCP 的状态开始谈起。

### TCP 连接的生命周期

![TCP Status](https://upload.wikimedia.org/wikipedia/commons/thumb/f/f6/Tcp_state_diagram_fixed_new.svg/1920px-Tcp_state_diagram_fixed_new.svg.png)

由图可知，TCP 连接的生命周期大体上分为三个阶段：连接创建、数据传送和连接终止。显然，处于连接终止状态的 TCP 连接是不可能被复用的。那么，在其他两种状态的 TCP 连接会是什么情况呢？

### 连接建立阶段的 TCP 复用

当一个 `session` 正在为其 `task` 建立一条 TCP 连接时，用同一个 `session` 创建一个新的 `task` 会怎么样呢？会等待 TCP 连接创建完成，两个 `task` 使用同一 TCP 连接？还是为第二个 `task` 创建一条新连接？通过代码来测试一下。

在正式测试之前，先来看一下建立一条 TCP 连接需要多长时间:

```shell
nc httpbin.org 80
```

![TCP-Handshake-wireshark](/asserts/URLLoadingSystem1/TCP-Handshake-wireshark.png)

从 Wireshark 的结果可以看到，从发送第一条 `SYN` 到收到服务器端的 `SYN, ACK` 的时间大约为 **300ms**。也就是说，两个 `sessionTask` 的启动时间相差 300ms 左右基本上就能满足所需的要求。

```swift
let session = URLSession(configuration: .default)

let baseURL = URL(string: "http://httpbin.org")!
let getURL = URL(string: "/get", relativeTo: baseURL)!
let headerURL = URL(string: "/headers", relativeTo: baseURL)!

let handler: (Data?, URLResponse?, Error?) -> Void = { (data, response, error) in
    guard error == nil else { fatalError(error!.localizedDescription )}
    
    if let data = data, let body = String(data: data, encoding: .utf8) {
        print(body)
    } else {
        print(data)
    }
}

let task1 = session.dataTask(with: getURL, completionHandler: handler)
let task2 = session.dataTask(with: headerURL, completionHandler: handler)

task1.resume()
DispatchQueue.main.asyncAfter(deadline: .now() + .milliseconds(200)) {
    task2.resume()
}
```

![Reuse-connection-in-handshake1](/asserts/URLLoadingSystem1/Reuse-connection-in-handshake1.png)

![Reuse-connection-in-handshake2](/asserts/URLLoadingSystem1/Reuse-connection-in-handshake2.png)

从 Wireshark 的结果可以看出，同一 `session` 的两个 `task` 使用了两条 TCP 连接，这说明在 TCP 的连接建立阶段，**新 `task` 不会等待现有连接建立成功后复用连接，而是选择建立一条新连接。**

#### 使用系统 API 获取相关信息

当然，除了使用 Wireshark 直接根据 TCP 分组判断之外，系统也提供了满足对应需求的 API。在 iOS 10 之后，`URLSessionTaskDelegate` 中提供了 `func urlSession(URLSession, task: URLSessionTask, didFinishCollecting: URLSessionTaskMetrics)` 方法用于获取 task 和连接相关的信息。设置 session 的 delegate 后再次执行上述代码。

![Session-taskdelete-result](/asserts/URLLoadingSystem1/Session-taskdelete-result.png)

与 Wireshark 的结果一致， `task2` 开始于  `task1` 的握手阶段，并建立了一条新的 TCP 连接。

### 数据传送阶段的 TCP 复用

既然知道了三次握手所需的大致时间，那么在连接建立之后且被释放之前创建一个新的 `task` 会怎么样呢？我们延长两个 `task ` 的间隔时间：

```swift
task1.resume()
DispatchQueue.main.asyncAfter(deadline: .now() + .seconds(2)) {
    task2.resume()
}
```

![Reuse-connection-in-transfer](/asserts/URLLoadingSystem1/Reuse-connection-in-transfer.png)

可以看到， `task2` 复用了 `task1` 建立的 TCP 连接。

### 问题的关键

![WWDC-Transaction-Metrics](/asserts/URLLoadingSystem1/WWDC-Transaction-Metrics.png)

从图上可以看出，一个 `task` 的周期中明确的分为几个阶段：请求开始（查找缓存），域名解析，连接建立（对 https 请求包含 TLS 握手），发送请求，等待响应，接收响应。

![Session-taskdelete-timestamps](/asserts/URLLoadingSystem1/Session-taskdelete-timestamps.png)

这些阶段花费的大致时间

- 请求开始：10ms，未找到缓存数据，直接进入下一阶段
- 域名解析：1ms，Wireshark 中没有对应 DNS 请求，推测是直接使用了本地缓存的 DNS 查询结果
- 连接建立：300ms，由于未使用 https，该阶段不包括 TLS 握手，具体数值取决于网络状况和具体的请求类型
- 发送请求：0ms，该阶段反应的是应用层数据进入传输层的开始和结束时间
- 等待响应：300ms，从客户端的角度是从请求发送结束到收到响应的时间
- 接受响应：0ms，传输层将数据传递给应用层的开始和结束时间

其中最为关键的一个时间点就是响应结束的时间，在不考虑超时释放连接的情况下，`task2` 的开始时间与 `task1` 响应结束时间的关系决定了能否复用已建立的连接。

当 `task2` 开始于 `task1` 结束之后：

![Session-taskdelete-reuse-success](/asserts/URLLoadingSystem1/Session-taskdelete-reuse-success.png)

`task2` 开始于 `task1`结束之前：

![Session-taskdelete-reuse-failed](/asserts/URLLoadingSystem1/Session-taskdelete-reuse-failed.png)

### 一些特殊情况

#### 超时释放

一个 TCP 连接不可能被无限制的保留，当超过某个既定的超时时间后，`URLSession` 会主动释放该连接，下次使用时需要重新握手建立连接。

#### 手动控制

`URLSession` 也提供了一些接口用于控制 TCP 连接。`URLSessionConfiguration` 的 `httpMaximumConnectionsPerHost` 属性提供了控制每个 `session` 对每个 `host` 的最大连接数。该值在 iOS 中默认为 4。如果将这个值设为 1，即要求每个 `session` 对每个 `host` 最多只能保留 1 条连接。但实际上**强烈不建议**使用这种方法控制连接数量。通常控制连接数量的目的在于通过减少握手花费的时间减少整体的网络时延，但使用该方法会使得所有请求使用同一条 TCP 连接，虽然避免了多次握手花费的时间，但这样原本能够并行进行的请求被强制变成了串行，即上一个请求的响应结束前其他请求不能开始，只能排队等候，这样会显著增加整体的网络时延。

#### 跨 Session 复用

不同的 session 之间能不能复用 TCP 连接？不能。每个 `session` 只能控制其自身创建的 `task` 对应的连接，这也是官方建议复用 `session` 的目的之一。

### 参考资料

[URL Loading System](https://developer.apple.com/documentation/foundation/url_loading_system)

[NSURLSession: New Features and Best Practices](https://developer.apple.com/videos/play/wwdc2016/711/)
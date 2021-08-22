# 多线程导读
## 官方文档

[Introduction](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html)

中文版本：

[Threading Programming Guide(1)](http://yulingtianxia.com/blog/2017/08/28/Threading-Programming-Guide-1/)

[Threading Programming Guide(2)](http://yulingtianxia.com/blog/2017/09/17/Threading-Programming-Guide-2/)

[Threading Programming Guide(3)](http://yulingtianxia.com/blog/2017/10/08/Threading-Programming-Guide-3/)

## 官方的并行编程指南

[Concurrency and Application Design](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ConcurrencyandApplicationDesign/ConcurrencyandApplicationDesign.html)

### 远离线程

### 为什么要使用并行编程

1. 充分利用计算机的多核；
2. 更好的用户体验。

### 为什么避免使用线程（`NSThread`）

1. 使用 `NSThread` 你需要自己管理线程池；
2. 需要根据系统的状态调整线程的数量；
3. 需要保持线程的高效运行，避免互相干扰。

### Dispatch Queues

Dispatch Queues 是一种基于 C 的机制，可执行自定义任务，支持串行和并行，任务按照 FIFO 顺序执行。
优点：

- 提供直观简单的接口；
- 提供自动和全面的线程池管理；
- 提供了优化后的速度；
- 更高效率的内存占用（线程栈帧不需要常驻内存）；
- 不需要与内核交互；
- 分发到异步队列的任务不会造成队列死锁；
- 优雅的扩展；
- 串行队列提供了一个比锁或者其它原始的同步方案更高效的替代品。

### Dispatch Sources

Dispatch Sources 是一种基于 C 的机制，用于异步处理特定类型的系统。 Dispatch Source 会包含有关特定类型系统事件的信息，在事件发生时将特定的 block 提交给 Dispatch Queue 。你可以使用 Dispatch Sources 来监听以下几种类型的系统事件：

- Timer 定时器；
- Signal 监听UNIX信号；
- Descriptor-related events 监听文件和 Socket 相关操作；
- Process-related events 监听进程相关状态；
- Mach port events 监听 Mach 相关事件；
- Custom events that you trigger 监听自定义事件；

Dispatch sources 是 GCD 的一部分。

### Operation Queues

Operation Queue 是 Cocoa 提供的，和并行的 Dispatch Queue 是相同概念的东西，具体类型为 `NSOperationQueue` 类。跟 Dispatch Queue 保持 FIFO 的执行顺序不同， Operation Queue 支持自定义的执行顺序。你可以在定义任务时设置相关的依赖，以此来创造一个复杂的执行顺序图。

Operation Queue 中任务对应的类型为 `NSOperation` 类。`NSOperation` 封装了你需要执行的任务和所有相关的数据。 `NSOperation` 是一个抽象的基类，你可以自定义一些子类来执行自己的任务。系统也提供了一些特定的子类。

`NSOperation` 提供了 KVO 通知，可用于监听任务的进度。Operation Queue 通常会并发执行任务，你可以通过设置依赖来保证它们按照所计划的顺序执行。

### Asynchronous Design Techniques

### Define Your Application’s Expected Behavior

当你想要给自己的应用添加并发代码时，你应该先定义好应用的预期行为，以此来验证接入并发编程后的行为是否正确以及测试性能收益。

定义相关任务和数据结构。理清各个任务间的依赖关系。

### Factor Out Executable Units of Work

找出任务的最小执行单元，将其封装进 `block` 或者 `NSOperation` ，然后派发到适当的队列中。不用担心任务分得太细而影响性能。队列会帮你处理好这一切，当然了，最好还是通过性能测试来调整任务大小。

### Identify the Queues You Need

确定好队列的属性。
当使用 GCD 时，如果需要特定的执行顺序，使用串行队列，如果不需要特定的执行队列，可使用并发队列或者多个队列。
当使用 `NSOperation` 时，可通过设置各个 `NSOperation` 之间的依赖来调整它们之间的执行顺序。

### Tips for Improving Efficiency

- 如果你的应用已经受内存限制，那么现在直接使用计算值可能比从主内存加载缓存的值要快。 计算值直接使用处理器核心的寄存器和缓存，这比主内存快得多。 当然了，还是需要性能测试来确定是否有利于性能优化；
- 找出串行任务，尽力把它们变得更加并发。 如果由于某个任务依赖某些共享资源而必须串行执行该任务，请考虑更改架构以删除该共享资源。 可以考虑为每个需要共享资源的用户拷贝一份对应的资源，或者完全清除共享资源；
- 避免使用锁。Dispatch queue 和 Operation Queue 在大多数情况下都不需要使用锁。避免使用锁来保护共享资源，使用串行队列或者设置 NSOperation 间的依赖来确保以正确的顺序执行任务；
- 尽量使用系统框架。实现功能时优先考虑现有的系统 API 是否可以满足需求。

## 相关分类

[NSOperation](./NSOperation.md) 

[Grand Central Dispatch](https://www.notion.so/Grand-Central-Dispatch-a0f552b226b44f3fb8f6cdbc7a8d73fe) 

[pthread 和 NSThread](https://www.notion.so/pthread-NSThread-c2839897019440aba14acdae51166ce5) 

[Lock](https://www.notion.so/Lock-794065e788bb4741a870c4434323de5b) 

## 相关文章

### ObjC 专题

[ObjC 中国 - 并发编程：API 及挑战](https://objccn.io/issue-2-1/)

[ObjC 中国 - 底层并发 API](https://objccn.io/issue-2-3/)

[ObjC 中国 - 线程安全类的设计](https://objccn.io/issue-2-4/)

[ObjC 中国 - 测试并发程序](https://objccn.io/issue-2-5/)

![](media/16296020555796.jpg)

### Sindrilin 关于多线程的文章

[](http://sindrilin.com/2017/09/09/thread_safe.html)

如何保证线程安全，从原子性到线程锁（互斥，自旋，信号量），也说到了 `barrier` 操作。

[](http://sindrilin.com/2017/09/27/producers_consumers.html)

### Swift 专题

[Swift 中的并发编程(第一部分：现状）](https://swift.gg/2017/09/04/all-about-concurrency-in-swift-1-the-present/)

这篇文章介绍了 Swift 未支持协程 ( `async/await` )前的并发编程模式，很好地总结了 Swift 中目前可用的外部并发框架，包括锁类型， GCD 和操作队列。

Swift 3 中已经去掉了 `dispatch_once` ， `dispatch_once` 在 Objective-C 中常用于构建线程安全的单例。在 Swift 中可以通过全局常量来初始化单例，Swift 确保使用原子化的方式来进行初始化：

```swift
final class Singleton {

    public static let sharedInstance: Singleton = Singleton()

    private init() { }

    ...
}
```

也可以通过串行队列加 token 实现类似 `dispatch_once` 功能：

```swift
import Foundation

public extension DispatchQueue {
    
    private static var onceTokens = [Int]()
    private static var internalQueue = DispatchQueue(label: "dispatchqueue.once")
    
    public class func once(token: Int, closure: (Void)->Void) {
        internalQueue.sync {
            if onceTokens.contains(token) {
                return
            }else{
                onceTokens.append(token)
            }
            closure()
        }
    }
}

let t = 1
DispatchQueue.once(token: t) {
    print("only once!")
}
DispatchQueue.once(token: t) {
    print("Two times!?")
}
DispatchQueue.once(token: t) {
    print("Three times!!?")
}
```

Swift 3 新增了一个函数，可用于判断任务是否在预期的队列中执行， `DispatchPredicate` 提供了三个枚举值：

- `onQueue` ：验证任务是否在指定队列中执行；
- `notOnQueue` ：与 `onQueue` 情况相反；
- `onQueueAsBarrier` ：验证当前任务是否作为一个队列的屏障。

```swift
dispatchPrecondition(condition: .notOnQueue(mainQueue))
dispatchPrecondition(condition: .onQueue(queue))
```

### 重点

[iOS探索 多线程面试题分析](https://juejin.im/post/6844904138623418376)
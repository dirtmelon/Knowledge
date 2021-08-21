# KVO
## 基础

官方文档：

[Introduction to Key-Value Observing Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html#//apple_ref/doc/uid/10000177-BCICJDHA)

开启 KVO 需要严格遵循以下 3 个步骤：

1. 使用 `addObserver:forKeyPath:options:context:` 方法注册监听者；
2. 在监听类中实现 `observeValueForKeyPath:ofObject:change:context:` 方法来接收通知；
3. 当不需要接收时，需要调用 `removeObserver:forKeyPath:` 。在监听者 `dealloc` 方法中需要调用这个方法来移除监听。

其它：

`automaticallyNotifiesObserversForKey:` 默认返回 `YES` ，当重写并对某个 `Key` 返回 `NO` 时，那么修改属性时就需要手动调用 `(void)willChangeValueForKey:(NSString *)key` 与 `-(void)didChangeValueForKey:(NSString *)key` 发送通知，我们也可以通过这样在 `Setter` 方法判断对象是否真的发生改变，只有真的发生改变时才发送通知。

## KVO 详解

[iOS 中的 KVO](https://kingcos.me/posts/2019/kvo_in_ios/)

这篇文章非常详细，从 `KVO` 的使用到原理都进行了说明。

## KVC 和 KVO

[ObjC 中国 - KVC 和 KVO](https://objccn.io/issue-7-3/)

一个需要注意的地方是，KVO 行为是同步的，并且发生与所观察的值发生变化的同样的线程上。没有队列或者 RunLoop 的处理。

[objcio/issue-7-lab-color-space-explorer](https://github.com/objcio/issue-7-lab-color-space-explorer/blob/master/Lab%20Color%20Space%20Explorer/KeyValueObserver.m)

## Friday Q&A About KVO

[mikeash.com: Friday Q&A 2009-01-23](https://www.mikeash.com/pyblog/friday-qa-2009-01-23.html)

Mikeash 关于 KVO 原理的文章：

- 动态生成一个 `KVO` 的子类，实现了 `dealloc` ， `_isKVOA` ， `class` 方法；
- 只会生成一个 `KVO` 子类，对所有监听的属性的设置方法都进行了替换，如果针对不同的属性监听生成不同类，就需要动态生成大量的不同的类，所以苹果选择了只生成一个类；
- 替换了对应的方法的 `IMP` ，改用内部的 `NSSet...ValueAndNotify` ；

## Key-Value Observing Done Right

[mikeash.com: Key-Value Observing Done Right](https://www.mikeash.com/pyblog/key-value-observing-done-right.html)

Mikeash 先是吹捧了一下 KVO 机制，非常强大和好用，但是 API 设计非常糟糕：

1. `-addObserver:forKeyPath:options:context:` 不支持 `selector` 参数，对比 `NSNotificationCenter` 的设计，可谓高下立判， KVO 必须要在  `-observeValueForKeyPath:ofObject:change:context:` 中处理消息或者传递给父类；
2. 因为不支持 `selector` 参数，所以如果在相同的 `observer` 监听相同的 `KeyPath` 时，需要通过 `context` 参数来进行区分；
3. `-removeObserver:forKeyPath:` 不支持 `context` 参数， KVO 是在 iOS2.0 时增加的，后面在 iOS5.0 新增了 `-removeObserver:forKeyPath:context:` ，支持 `context` 参数。

## KVO Considered Harmful

[kvo-considered-harmful](https://khanlou.com/2013/12/kvo-considered-harmful/)

KVO 缺点：

1. 所有回调都在同一个方法中进行，稍不留意这个方法就会快速膨胀；
2. 使用字符串硬编码，如果被监听的对象修改了属性名，编译期无法察觉；
3. 要求处理父类的 KVO 流程；
4. 移除 observer 时有可能会崩溃；
5. 充斥着大量有可能会失败的操作，作者认为一个好的 API 设计应该起到使用者成功地调用他们，即使没有解释为什么要这样去调用；
6. 流程过于隐藏，没办法追踪数据改变的流程，与 delegate 模式相比， KVO 在 debug 时比较麻烦，且需要在运行时通过 `isKindOfClass:` 动态判断类型；
7. 有可能造成死循环，如果不小心在回调中修改了监听的属性，那么就会造成死循环，如果说两个属性在不同的 KVO 流程中互相修改，也会造成死循环，且难于 debug ；
8. KVO 在某些场景下会失效，比如说 `__weak` 属性，在 `__weak` 对象被释放时， KVO 是不会去清理对应的监听，导致可能会出现野指针崩溃；
9. KVO 是一种老旧的模式，在 Apple 平台上，我们可以通过其它方式比如说 Delegate ，Block 和明确的发布/订阅 （ `NSNotificationCenter` ）方式来解决问题，而不是使用 KVO 这种隐晦的方式。

什么时候可以使用 KVO ：

1. Apple 官方要求，比如说 `AVPlayer` ，要求通过监听 `status` 属性来获取播放器的状态；
2. 设计相关的 API 给其他开发者使用。

## 刨根问题 KVO 原理

[刨根问底KVO原理](https://juejin.im/post/5c22023df265da6124157a25)

通过源码相关的伪代码来探究 `KVO` 的实现方式，如果需要深入了解 `KVO` 的原理，可以阅读下这篇文章。 `KVO` 的原理看起来虽然比较简单，但是实现时还是有不少坑，比如说多线程，系统的具体实现也体现了这一点，通过 `pthread_mutex_lock` 来保证线程安全。

## KVOController 解析

[draveness/analyze](https://github.com/draveness/analyze/blob/master/contents/KVOController/KVOController.md)

[facebookarchive/KVOController](https://github.com/facebookarchive/KVOController)

为了解决 `KVO` 非常难用的问题，Facebook 开源了 `KVOController` ，优点如下：

1. 不需要手动移除 `observer` ，这里利用了关联属性在对象释放时也会被释放的原理，在关联属性的 `dealloc` 方法中移除 `observer` ；
2. 支持使用 `block` ，减少复杂度，添加监听和处理通知的代码可以放在同一处。

## 基于 KVO hook 子类的方法

[一种基于KVO的页面加载，渲染耗时监控方法](http://satanwoo.github.io/2017/11/27/KVO-Swizzle/)

在做 `ViewController` 的耗时检测时，我们需要记录各个 `UIViewController` 子类对应方法的耗时，如果只是针对 `UIViewController` 的方法进行 `hook` ，那么只能记录到 `UIViewController` 的方法耗时，无法获取子类的方法耗时。

在进行 `KVO` 时 `runtime` 实际上会帮你创建一个 `KVO` 相关的子类，由此可以在初始化时进行一次 `KVO` 来生成一个新的子类，然后对这个子类方法进行耗时检测。

至于为什么使用 `KVO` 的方式，下面这篇文章有进行解释，而且也给出了具体实现代码：

[巧妙利用KVO实现精准的VC耗时检测](https://punmy.cn/2018/06/18/15278496835424.html)

[panmingyang2009/VCProfiler](https://github.com/panmingyang2009/VCProfiler)

## KVO 在不同的二进制中多个符号并存的 Crash 问题

[KVO在不同的二进制中多个符号并存的Crash问题](http://satanwoo.github.io/2017/09/11/KVO-CRASH/)

当两个产物都有相同的类名时，比如主二进制和动态库中，这两个类都会被 realize ，都能够被正常调用。

> 其原因在于苹果使用的是 `two level namespace` 的技术。在这种形式下，符号所在的“库”的名称也会作为符号的一部分。链接的时候， `staic linker` 会标记住在这个符号是来自于哪个库的。这样不仅大大减少了dyld搜索符号所需要的时间，也更好对后续库的更新进行了兼容。

但是由于全局类表的存在，在动态创建 `KVO` 的子类时，只能产生一个。所以就导致 `allocate` 失败，从而引发 `register` 过程的 Crash 问题。
# NSOperation
## 官方文档

[Operation Queues](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationObjects/OperationObjects.html#//apple_ref/doc/uid/TP40008091-CH101-SW1)

[Operation](https://developer.apple.com/documentation/foundation/operation)

[OperationQueue](https://developer.apple.com/documentation/foundation/operationqueue)

[BlockOperation](https://developer.apple.com/documentation/foundation/blockoperation)

官方文档写的非常详细， `Operation` 和 `OperationQueue` 都写到，强烈推荐。

- `Operation` 支持异步和同步，可以通过 `isAsynchronous` 来判断是否异步，如果添加到 `OperationQueue` 中， `OperationQueue` 会忽略掉这个属性，且默认为异步，可以通过设置 `OperationQueue` 的 `maxConcurrentOperationCount` 为 1 来强制串行。
- 自定义 `Operation` 时记得设置相关状态和 `KVO` 配置，如果 A 操作依赖 B 操作，即使 B 操作取消了， A 操作也会执行，A 操作是否 Ready 是通过 B 操作的 `isFinished` 来判断的，所以可能需要加入额外的判断，判断 B 操作是否成功执行。
- `Operation` 的 `start()` 和 `main()` 的区别。
- 优先级配置： `Operation.QueuePriority` ， `Operation` 不是严格按照优先级高低来执行，如果高优先级的 `Operation` 还没准备好， `OperationQueue` 就会选择去执行低优先级的 `Opeartion` 。如果对顺序有严格要求的话还是要通过依赖来进行配置。
- `Operation` 执行完毕，它就会调用它的 `completionBlock` 。

## iOS多线程：『NSOperation、NSOperationQueue』详尽总结

[iOS多线程：『NSOperation、NSOperationQueue』详尽总结](https://juejin.im/post/5a9e57af6fb9a028df222555)

`NSOperation` 和 `NSOperationQueue` 的优点：
1. 可添加完成时调用的任务，便于在操作完成后执行；
2. 方便管理各操作之间的依赖关系；
3. 设定操作执行的优先级；
4. 支持取消；
5. 支持通过 KVO 观察操作的状态： `isExecuteing` 、 `isFinished` 、`isCancelled` 。

`NSOperation` 是一个抽象类，不支持直接使用，需要使用系统提供的子类或者自定义子类：
1. 使用子类 `NSInvocationOperation` ；
2. 使用子类 `NSBlockOperation` ；
3. 使用自定义子类。

`NSOperationQueue` 支持以下两种队列：
1. 主队列， `NSOperationQueue *queue = [NSOperationQueue mainQueue]` ；
2. 自定义队列，`NSOperationQueue *queue = [[NSOperationQueue alloc] init]` 。

进阶：
1. 通过 `maxConcurrentOperationCount` 可以在串行和并发间切换；
2. 通过 `-addDependency:` 和 `- removeDependency:` 可以进行依赖关系管理；
3. 通过 `queuePriority` 进行优先级设置，默认为 `NSOperationQueuePriorityNormal` ，优先级不能取代依赖关系，执行任务的顺序以依赖关系为优先。

## NSOperation

[NSOperation](https://nshipster.com/nsoperation/)

NSHipster 出品的简短介绍。
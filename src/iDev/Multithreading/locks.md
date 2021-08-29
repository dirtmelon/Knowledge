# 锁
## 自旋锁

当线程等待自旋锁时不会进入睡眠，自旋锁由于在获取锁时，线程会一直处于忙等状态，有可能会造成任务的优先级反转。

### OSSpinLock

> 新版 iOS 中，系统维护了 5 个不同的线程优先级/QoS: background，utility，default，user-initiated，user-interactive。高优先级线程始终会在低优先级线程前执行，一个线程不会受到比它更低优先级线程的干扰。这种线程调度算法会产生潜在的优先级反转问题，从而破坏了 spin lock。

> 具体来说，如果一个低优先级的线程获得锁并访问共享资源，这时一个高优先级的线程也尝试获得这个锁，它会处于 spin lock 的忙等状态从而占用大量 CPU。此时低优先级线程无法与高优先级线程争夺 CPU 时间，从而导致任务迟迟完不成、无法释放 lock。这并不只是理论上的问题，libobjc 已经遇到了很多次这个问题了，于是苹果的工程师停用了 OSSpinLock。

[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

## 互斥锁

当等待互斥锁时，线程会进入睡眠，锁释放时就会唤醒线程。互斥锁又分为递归锁和非递归锁：

- 递归锁：可重入，同一个线程在锁释放前可再次获取锁，可以递归调用；
- 非递归锁：不可重入，必须等锁释放后才能再次获取。

### pthread_mutex

`pthread_mutex` 是互斥锁，对性能要求比较高的场景可以使用， API 比较简单：

```objectivec
// 导入头文件
#import <pthread.h>
// 全局声明互斥锁
pthread_mutex_t _lock;
// 初始化互斥锁
pthread_mutex_init(&_lock, NULL);
// 加锁
pthread_mutex_lock(&_lock);
// 这里做需要线程安全操作

// 解锁
pthread_mutex_unlock(&_lock);
// 释放锁
pthread_mutex_destroy(&_lock);

```

### @synchronized

[mikeash.com: Friday Q&A 2015-02-20: Let's Build @synchronized](https://www.mikeash.com/pyblog/friday-qa-2015-02-20-lets-build-synchronized.html)

[关于 @synchronized，这儿比你想知道的还要多](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)

性能比较低，因为有容错处理，和使用全局表。

1. 不能使用`非OC对象`作为加锁条件——`id2data`中接收参数为id类型
2. 多次锁同一个对象会有什么后果吗——会从高速缓存中拿到data，所以只会锁一次对象
3. 都说@synchronized性能低——是因为在底层`增删改查`消耗了大量性能
4. 加锁对象不能为nil，否则加锁无效，不能保证线程安全

### NSLock

`NSLock` 对 `pthread_mutex` 进行了一层封装，提供了 `Objective-C` 层级的 API ：

```objectivec
NSLock *lock = [[NSLock alloc] init]
[lock lock];
[lock unlock];
```

从 Swift 开源版的 Foundation 中可以看到 NSLock 是基于 `pthread_mutex` 的封装：

[NSLock.swift](https://github.com/apple/swift-corelibs-foundation/blob/main/Sources/Foundation/NSLock.swift)

删除其它平台的代码后大概实现如下：

```swift
open class NSLock: NSObject, NSLocking {
		public override init() {
        pthread_mutex_init(mutex, nil)
        pthread_cond_init(timeoutCond, nil)
        pthread_mutex_init(timeoutMutex, nil)
    }

		open func lock() {
        pthread_mutex_lock(mutex)
    }

    open func unlock() {
        pthread_mutex_unlock(mutex)
        // Wakeup any threads waiting in lock(before:)
        pthread_mutex_lock(timeoutMutex)
        pthread_cond_broadcast(timeoutCond)
        pthread_mutex_unlock(timeoutMutex)
    }
}
extension NSLock {
    // 同步执行 closure 操作
    internal func synchronized<T>(_ closure: () -> T) -> T {
        self.lock()
        defer { self.unlock() }
        return closure()
    }
}

```

而 `NSLock` 非递归锁，在递归调用时会堵塞。如果需要递归调用可以通过 `NSRecursiveLock` 加锁， `NSRecursiveLock` 实现与 `NSLock` 类似，只是在初始化时设置 `mutex` 为 `RECURSIVE` 类型：

```swift
open class NSRecursiveLock: NSObject, NSLocking {
    internal var mutex = _RecursiveMutexPointer.allocate(capacity: 1)
    private var timeoutCond = _ConditionVariablePointer.allocate(capacity: 1)
    private var timeoutMutex = _MutexPointer.allocate(capacity: 1)

    public override init() {
        super.init()
        var attrib = pthread_mutexattr_t()
        withUnsafeMutablePointer(to: &attrib) { attrs in
            pthread_mutexattr_init(attrs)
						// 设置为 RECURSIVE
            pthread_mutexattr_settype(attrs, Int32(PTHREAD_MUTEX_RECURSIVE))
            pthread_mutex_init(mutex, attrs)
        }
        pthread_cond_init(timeoutCond, nil)
        pthread_mutex_init(timeoutMutex, nil)
    }

    open func lock() {
        pthread_mutex_lock(mutex)
    }

    open func unlock() {
        pthread_mutex_unlock(mutex)
        // Wakeup any threads waiting in lock(before:)
        pthread_mutex_lock(timeoutMutex)
        pthread_cond_broadcast(timeoutCond)
        pthread_mutex_unlock(timeoutMutex)
    }
}

```

而递归锁同时对同一个对象使用锁时也会产生死锁：

### NSCondition

[NSCondition](https://developer.apple.com/documentation/foundation/nscondition)

`NSCondition` 对象在给定线程中充当锁和检查点 （ checkpoint ）。锁在检测条件和执行由条件触发的任务时保护你的代码。检查点则要求线程在执行其任务之前条件为 `true` 。当条件为 `false` 时，线程会阻塞，直到另一个线程向条件对象发出信号。伪代码：

```
lock the condition
while (!(boolean_predicate)) {
    wait on condition
}
do protected work
(optionally, signal or broadcast the condition again or change a predicate value)
unlock the condition

```

与信号量类似， Swift 源码中也有 `NSCondition` 的实现：

```swift
open class NSCondition: NSObject, NSLocking {
    internal var mutex = _MutexPointer.allocate(capacity: 1)
    internal var cond = _ConditionVariablePointer.allocate(capacity: 1)

    public override init() {
        pthread_mutex_init(mutex, nil)
        pthread_cond_init(cond, nil)
    }

    deinit {
        pthread_mutex_destroy(mutex)
        pthread_cond_destroy(cond)
        mutex.deinitialize(count: 1)
        cond.deinitialize(count: 1)
        mutex.deallocate()
        cond.deallocate()
    }

    open func lock() {
        pthread_mutex_lock(mutex)
    }

    open func unlock() {
        pthread_mutex_unlock(mutex)
    }

    open func wait() {
        pthread_cond_wait(cond, mutex)
    }

    open func wait(until limit: Date) -> Bool {
        guard var timeout = timeSpecFrom(date: limit) else {
            return false
        }
        return pthread_cond_timedwait(cond, mutex, &timeout) == 0
    }

    open func signal() {
        pthread_cond_signal(cond)
    }

    open func broadcast() {
        pthread_cond_broadcast(cond)
    }

    open var name: String?
}

```

用法如下：

```swift
let cond = NSCondition()
var available = false
var SharedString = ""

class WriterThread : Thread {

    override func main(){
        for _ in 0..<5 {
            cond.lock()
            SharedString = "😅"
            available = true
            cond.signal() // 通知并且唤醒等待的线程
            cond.unlock()
        }
    }
}

class PrinterThread : Thread {

    override func main(){
        for _ in 0..<5 { // 循环 5 次
            cond.lock()
            while(!available){   // 通过伪信号进行保护
                cond.wait()
            }
            print(SharedString)
            SharedString = ""
            available = false
            cond.unlock()
        }
    }
}

let writet = WriterThread()
let printt = PrinterThread()
printt.start()
writet.start()

```

### NSConditionLock

[NSConditionLock](https://developer.apple.com/documentation/foundation/nsconditionlock)

`NSConditionLock` 与 `NSCondition` 不同，自带支持复杂的条件锁，比如说：消费者-提供者场景。 `lock(whenCondition:)` 在条件成立时可以获取到锁，或者等待另外一个线程调用 `unlock(withCondition:)` 释放锁和设置对应的值。

```swift
let NO_DATA = 1
let GOT_DATA = 2
let clock = NSConditionLock(condition: NO_DATA)
var SharedInt = 0

class ProducerThread : Thread {

    override func main(){
        for i in 0..<5 {
            clock.lock(whenCondition: NO_DATA) //当条件为 NO_DATA 获取该锁
              // 如果不想等待消费者，直接调用 clock.lock() 即可
            SharedInt = i
            clock.unlock(withCondition: GOT_DATA) //解锁并设置条件为 GOT_DATA
        }
    }
}

class ConsumerThread : Thread {

    override func main(){
        for i in 0..<5 {
            clock.lock(whenCondition: GOT_DATA) // 当条件为 GOT_DATA 获取该锁
            print(i)
            clock.unlock(withCondition: NO_DATA) //解锁并设置条件为 NO_DATA
        }
    }
}

let pt = ProducerThread()
let ct = ConsumerThread()
ct.start()
pt.start()

```

### dispatch_semaphore

[dispatch_semaphore](https://developer.apple.com/documentation/dispatch/dispatchsemaphore)

`distpatch_semaphore` 在 Swift 上已替换为 `DispatchSemaphore` ，相应的 API 也有所改变。

信号量，可以作为同步锁使用，也可以控制 GCD 的最大并发数：

```
let semaphore = DispatchSemaphore(value: 0)
semaphore.signal()
semaphore.wait()

```

### os_unfair_lock

`os_unfair_lock` 是苹果在 iOS 10/macOS 10.12 上提供的，用于替换 `OSSpinLock` 。

[os_unfair_lock_lock](https://developer.apple.com/documentation/os/1646466-os_unfair_lock_lock)

## 各个锁在 Swift 上的性能测试

[Updated for Xcode 8, Swift 3; added os_unfair_lock](https://gist.github.com/steipete/36350a8a60693d440954b95ea6cbbafc)

这里有个锁的性能测试，如果只需要支持到 iOS10 ，那么使用 `os_unfair_lock_s` 是最好的选择， `OSSpinLock` 性能最好，但是有优先级反转的问题：

如果需要支持 iOS10 以下那么信号量或者 `pthread_mutex` 在性能上表现最好。如果需要支持 `Linux` 平台可以选择 `pthread_mutex` 。

其余的 `NSLock` ， `DispatchQueue` 和 `@Syncronized` 性能都较差，因为苹果在里面做了不少容错处理和进行一些全局记录（ `@Syncronized` ）

## Locks, Thread Safety, and Swift

[mikeash.com: Friday Q&A 2017-10-27: Locks, Thread Safety, and Swift: 2017 Edition](https://www.mikeash.com/pyblog/friday-qa-2017-10-27-locks-thread-safety-and-swift-2017-edition.html)

锁是一个确保在同一时机内只能有一个线程访问特定区间内代码的机制，确保了多线程访问可变数据时的一致性。锁有以下三种类型：

1. 线程等待锁时会进入休眠状态，获取到锁后再唤醒；
2. 线程等待锁时会一直忙等，直到获取锁，在等待时间较短时会更有效率，但是会浪费 CPU 时间；
3. 读写锁，支持多个读线程同时进入同一区间，但是只能支持一个写线程进行写数据；
4. 递归锁，支持同一个线程多次获取同一锁。

相关 API ：

1. `pthread_mutex_t` 互斥锁，支持配置为递归锁；
2. `pthread_rwlock_t` 互斥的读写锁；
3. `DispatchQueue` 可以派发阻塞的 `block` ，结合并发队列和 `barrier blocks` 实现读写锁，也支持异步派发；
4. `OperationQueue` 支持的功能和 `DispatchQueue` 类似；
5. `NSLock` 是 Objective-C 层级的锁， `NSRecursiveLock` 为支持递归的锁；
6. `os_unfair_lock` 是更底层的锁；
7. `@synchronized` 为 Objective-C 提供的语言特性。

`pthread` 在初始化时需要多注意下，需要通过 `pthread` 提供的方法 `pthread_mutex_init` 或者 `pthread_rwlock_init` 进行初始化：

```c
var mutex = pthread_mutex_t()
pthread_mutex_init(&mutex, nil)
```

### 值类型

`phtread_mutex_t` ， `pthread_rwlock_t` 和 `os_unfair_lock` 都是值类型。如果使用 `=` 进行赋值，那么就会进行值拷贝，但是这些锁类型并不支持拷贝。如果对 `pthread` 类型进行拷贝，拷贝得到的值是不可用的，会在使用时触发崩溃。 `pthread` 函数会假设 `pthread` 类型在初始化时是同一个内存地址，所以拷贝到别的内存地址并不是个好主意。 `os_unfair_lock` 不会崩溃，但是会得到另外一个锁。

### 如何选择对应的 Lock API

`DispatchQueue` 是比较好的选择，有着友好的，更加 Swift 的 API ，同时也支持各种各样的特性。但是 `DispatchQueue` 也不是完美的，在内存中，队列会作为一个对象来使用，所以会有一些性能上的开销。同时也不支持条件变量或者递归。

`os_unfair_lock` 在对性能有较高追求和不需要一些高级特性时是个不错的选择。它的实现相当于一个 32-bit 的整数，所以性能开销极小。但是正如它名字所说的那样，这是一个不公平的锁，它不会确保每个线程都有机会来获取锁，所以有可能某个线程会迅速地释放和获取锁，而其它线程则一直在等待。

`pthread_mutex` 则位于两者中间，它比 `os_unfair_lock` 考虑得更多，占用内存为 64 bytes ，在苹果的实现上，它是一个公平的锁。

`pthread_rwlock` 提供了一个读写锁，使用了 200 bytes ，但是没有提供更多的特性。

`NSLock` 基于 `pthread_mutex` 进行封装，如果说你不想手动管理 `phread_mutex` 的初始化和销毁，那么可以使用 `NSLock` 。

`OperationQueue` 提供了依赖管理。

优先考虑使用 `DispatchQueue` ，对性能有追求时可以使用 `os_unfair_lock` ，只有少数情况才需要考虑其它锁。

Swift 没有用于线程同步的语言工具，但是锁的 API 弥补了这一点。 GCD 依然是 Apple 皇冠上的明珠之一。虽然说我们没有 `@synchronized` 或者原子属性，但是我们有更好的选择。

## iOS Locks

[blog/ios-lock.md at master · bestswifter/blog](https://github.com/bestswifter/blog/blob/master/articles/ios-lock.md)

自旋锁伪代码：

```cpp
bool lock = false; // 一开始没有锁上，任何线程都可以申请锁
do {
    while(lock); // 如果 lock 为 true 就一直死循环，相当于申请锁
    lock = true; // 挂上锁，这样别的线程就无法获得锁
        Critical section  // 临界区
    lock = false; // 相当于释放锁，这样别的线程可以进入临界区
        Reminder section // 不需要锁保护的代码        
}
```

原子操作需要由硬件支持，在执行时会把总线锁住，使得其它 CPU 不能执行相同的操作：

```cpp
bool test_and_set (bool *target) {
    bool rv = *target; 
    *target = TRUE; 
    return rv;
}
```

也介绍了其它锁的相关原理。

## iOS 中的那些锁

[iOS探索 细数iOS中的那些锁](https://juejin.cn/post/6844904167010467854)

### `atomic` 的原理：

```cpp
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    if (offset == 0) {
        object_setClass(self, newValue);
        return;
    }

    id oldValue;
    id *slot = (id*) ((char*)self + offset);

    if (copy) {
        newValue = [newValue copyWithZone:nil];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:nil];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }

    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;        
        slotlock.unlock();
    }

    objc_release(oldValue);
}
```

如果使用了 `atomic` 进行声明，那么就会改用 `spinlock_t` 来进行加锁，而 `spinlock_t` 则是使用 `os_unfair_lock` 实现：

```cpp
using spinlock_t = mutex_tt<LOCKDEBUG>;

class mutex_tt : nocopy_t {
    os_unfair_lock mLock;
    ...
}
```

且由于 `atomic` 的性能问题，在 iOS 上基本上都是使用 `nonatomic` 来进行声明。
# Objective-C 类属性
[Objective-C Class Properties](https://useyourloaf.com/blog/objective-c-class-properties/)

虽然说 Swift 出来后 Objective-C 改进都比较少了，基本上都是为了更好地和 Swift 进行交互，但是对于需要编写 Objective-C 的程序员来说依旧是喜闻乐见的。

Xcode 8 release notes ：

> Objective-C now supports class properties, which interoperate with Swift type properties. They are declared as: @property (class) NSString *someStringProperty;. They are never synthesized. (23891898)

通过 `class` 声明属性：

```objectivec
@interface User : NSObject
@property (class, nonatomic, assign, readonly) NSInteger userCount;
@property (class, nonatomic, copy) NSUUID *identifier;
+ (void)resetIdentifier;
@end
```

上述代码声明了两个类属性，可读的 `userCount` 和可写的 `identifier` 。由于 Objective-C 不像 Swift 那样支持 `class` 层级的变量存储，所以需要声明对应的静态变量：

```objectivec
@implementation User
static NSUUID *_identifier = nil;
static NSInteger _userCount = 0;
```

对应 Objective-C 的 `class` 属性，Xcode 不会自动生成 `setter` 和 `getter` 方法，所以需要手动编译对应的方法。由于 `userCount` 只是可读的，所以只需要 `getter` 方法：

```objectivec
+ (NSInteger)userCount {
  return _userCount;
}
```

而对于 `identifier` 则需要补充 `setter` 和 `getter` 方法，同时需要支持 `copy` 语义：

```objectivec
+ (NSUUID *)identifier {
  if (_identifier == nil) {
    _identifier = [[NSUUID alloc] init];
  }
  return _identifier;
}

+ (void)setIdentifier:(NSUUID *)newIdentifier {
  if (newIdentifier != _identifier) {
    _identifier = [newIdentifier copy];
  }
}
```

也可以通过初始化方法来更新 `userCount` 属性：

```objectivec
- (instancetype)init
{
  self = [super init];
  if (self) {
    _userCount += 1;
  }
  return self;
}
```

`resetIdentifier` 可用于创建一个新的 `identifier` ：

```objectivec
+ (void)resetIdentifier {
  _identifier = [[NSUUID alloc] init];
}

@end
```

可以直接通过类名进行访问：

```objectivec
User.userCount;
User.identifier;
for (int i = 0; i < 3; i++) {
    self.user = [[User alloc] init];
    NSLog(@"User count: %ld",(long)User.userCount);
    NSLog(@"Identifier = %@",User.identifier);
}

[User resetIdentifier];    
NSLog(@"Identifier = %@",User.identifier);
```

生成对应的 Swift 代码如下：

```swift
public class User : NSObject { 
  public class var userCount: Int { get }
  public class var identifier: UUID!   
  public class func resetIdentifier()
}
```
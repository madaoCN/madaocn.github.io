---
layout: post
title: 长文预警）面向切面 Aspects 源码阅读
date: 2018-02-10
Author: madaoCN
categories: 
tags: [源码阅读]
comments: true
---

> ##### __前言__
__AOP(Aspect-oriented programming)__ 也称之为 __“面向切面编程”__, 是一种通过[预编译](https://baike.baidu.com/item/%E9%A2%84%E7%BC%96%E8%AF%91)方式和运行期动态代理实现程序功能的统一维护的一种技术。简单来说可以做到 __业务隔离__ ，__解耦__ 等等效果。__AOP__ 技术在__JAVA__ 的 __Spring__ 框架中已经提供了非常全面成熟的解决方案。然而 __iOS__ 等移动端在这方面的运用并不是很多，但是不妨碍它涌现出非常出色的三方库，比如，我们接下来要说的三方库 [Aspects](https://github.com/steipete/Aspects) .


那么我们什么时候使用 __AOP__ 比较适合呢
* 当我们执行某种方法时候，对方法进行安全检查 （例如，NSArray的数组越界问题）
* 对某些操作进行日志记录
* 对购物车进行交互时候，根据用户的操作，触发建议或者提示
* 激进一些，也可以用来去基类继承，例如笔者的架构实例：[NonBaseClass-MVVM-ReactiveObjc  (干货带代码)](https://www.jianshu.com/p/921dd65e79cb)

大家会说，传统的 __OOP(Object Oriented Programming)__ 即面向对象编程，也完全能够实现这些功能。
是的，没错，但是一个好的 __OOP__ 架构应该是单一职责的，添加额外的切面需求意味着破坏了单一职责。例如，一个 __Module__ 仅仅负责订单业务，但是你其添加了安全检查，日志记录，建议提示等等功能，这个 __Module__  会变得难以理解和维护，而且整个应用都将充斥着 日志，安全检查 等等逻辑，想想都头皮发麻。

![头皮发麻](http://upload-images.jianshu.io/upload_images/1749699-1e0706a9af076c2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/120)

__AOP__ 正是解决这一系列问题的良药。

![切面](http://upload-images.jianshu.io/upload_images/1749699-e981fb51a4b5d126.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)


> ##### __Aspects__基础用法

__Aspects__使用起来也是非常简单，只需要使用两个简单的接口，同时支持 __类Hook__ 和 __实例Hook__，提供了更细致的操作
```swift
/// Adds a block of code before/instead/after the current `selector` for a specific class.
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;

/// Adds a block of code before/instead/after the current `selector` for a specific instance.
- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;
```

例如，我们需要统计用户进入某个 __ViewController__ 的次数
```swift
[UIViewController aspect_hookSelector:@selector(viewWillAppear:) withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo, BOOL animated) {
    NSLog(@"View Controller %@ will appear animated: %tu", aspectInfo.instance, animated);
} error:NULL];
```
从此，我们不必在基类中添加丑陋的代码了

> ##### __Aspects架构剖析__

![架构](http://upload-images.jianshu.io/upload_images/1749699-6c1b1377d54d3e93.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/480)


##### __swizzledClassesDict__ 全局字典
访问接口
```swift
static NSMutableDictionary *aspect_getSwizzledClassesDict() {
    static NSMutableDictionary *swizzledClassesDict;
    static dispatch_once_t pred;
    dispatch_once(&pred, ^{
        swizzledClassesDict = [NSMutableDictionary new];
    });
    return swizzledClassesDict;
}
```
存储对象类型 __<Class : AspectTracker *>__   
此全局字典记录了对Hook的类及其父类的追踪信息对象 __AspectTracker__


##### __swizzledClasses__ 全局集合
访问接口
```swift
static void _aspect_modifySwizzledClasses(void (^block)(NSMutableSet *swizzledClasses)) {
    static NSMutableSet *swizzledClasses;
    static dispatch_once_t pred;
    dispatch_once(&pred, ^{
        swizzledClasses = [NSMutableSet new];
    });
    @synchronized(swizzledClasses) {
        block(swizzledClasses);
    }
}
```
__swizzledClasses__ 是一个全局集合，存储对象类型 __<Class>__
被Hook的对象类名都会被存储在此容器中

##### __AspectBlockRef__

__Aspects__ 中的 __AspectBlockRef__ 也就是我们使用接口中的参数 __usingBlock__ 中的Block，在上述的例子中如下形式
```swift
^(id<AspectInfo> aspectInfo, BOOL animated) {
    NSLog(@"View Controller %@ will appear animated: %tu", aspectInfo.instance, animated);
}
```
__AspectBlockRef__ 的源代码如下
```swift
typedef NS_OPTIONS(int, AspectBlockFlags) {
  AspectBlockFlagsHasCopyDisposeHelpers = (1 << 25),
  AspectBlockFlagsHasSignature          = (1 << 30)
};
/////////
typedef struct _AspectBlock {
  __unused Class isa;
  AspectBlockFlags flags;
  __unused int reserved;
  void (__unused *invoke)(struct _AspectBlock *block, ...);
  struct {
    unsigned long int reserved;
    unsigned long int size;
    // requires AspectBlockFlagsHasCopyDisposeHelpers
    void (*copy)(void *dst, const void *src);
    void (*dispose)(const void *);
    // requires AspectBlockFlagsHasSignature
    const char *signature;
    const char *layout;
  } *descriptor;
  // imported variables
} *AspectBlockRef;
``` 
看上去有点复杂，但是，我们一直使用的Block就是这样的结构体，在这里  __AspectBlockRef__ 其实就是 ___GloablBlock__类型，__AspectBlockRef__ 只是名字改了一下而已，本质上就是 Block
很幸运，苹果开源了Block的相关代码 ：[Block实现源码传送门](https://opensource.apple.com/source/libclosure/libclosure-67/)  我们得以一窥究竟
```swift
/////////////////////// Block_private.h

// Values for Block_layout->flags to describe block objects
enum {
    BLOCK_DEALLOCATING =      (0x0001),  // runtime
    BLOCK_REFCOUNT_MASK =     (0xfffe),  // runtime
    BLOCK_NEEDS_FREE =        (1 << 24), // runtime
    BLOCK_HAS_COPY_DISPOSE =  (1 << 25), // compiler
    BLOCK_HAS_CTOR =          (1 << 26), // compiler: helpers have C++ code
    BLOCK_IS_GC =             (1 << 27), // runtime
    BLOCK_IS_GLOBAL =         (1 << 28), // compiler
    BLOCK_USE_STRET =         (1 << 29), // compiler: undefined if !BLOCK_HAS_SIGNATURE
    BLOCK_HAS_SIGNATURE  =    (1 << 30), // compiler
    BLOCK_HAS_EXTENDED_LAYOUT=(1 << 31)  // compiler
};

#define BLOCK_DESCRIPTOR_1 1
struct Block_descriptor_1 {
    uintptr_t reserved;
    uintptr_t size;
};

#define BLOCK_DESCRIPTOR_2 1
struct Block_descriptor_2 {
    // requires BLOCK_HAS_COPY_DISPOSE
    void (*copy)(void *dst, const void *src);
    void (*dispose)(const void *);
};

#define BLOCK_DESCRIPTOR_3 1
struct Block_descriptor_3 {
    // requires BLOCK_HAS_SIGNATURE
    const char *signature;
    const char *layout;     // contents depend on BLOCK_HAS_EXTENDED_LAYOUT
};

struct Block_layout {
    void *isa;
    volatile int32_t flags; // contains ref count
    int32_t reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 *descriptor;
    // imported variables
};
```
其中 __flags__ 代表着Block的操作数
```swift
//  AspectBlockRef
AspectBlockFlags flags;   
//  Block_private.h
volatile int32_t flags; // contains ref count
```
__Aspects__ 中只有两种flag
```swift
AspectBlockFlagsHasCopyDisposeHelpers = (1 << 25),
AspectBlockFlagsHasSignature          = (1 << 30)
```
对应 Block 定义枚举中的
```swift
BLOCK_HAS_COPY_DISPOSE =  (1 << 25), // compiler
BLOCK_HAS_SIGNATURE  =    (1 << 30), // compiler
```
* 当Block被Copy时 `flag & AspectBlockFlagsHasCopyDisposeHelpers ` 为真时，Block布局中将会添加 `Block_descriptor_2 ` 
* 当Block带有方法签名时 `flag & AspectBlockFlagsHasSignature ` 为真时，Block布局中存在`Block_descriptor_3 ` 

__invoke__ 函数指针也只是第一个参数由泛型变成了___AspectBlock__而已
```swift
//  AspectBlockRef
void (__unused *invoke)(struct _AspectBlock *block, ...);
//  Block_private.h
void (*invoke)(void *, ...);
```
扯远了，打住。

##### __AspectToken__

![AspectToken](http://upload-images.jianshu.io/upload_images/1749699-4e8f33bdfa1037de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/480)

__Aspects__ 内定义的协议，用来撤销Hook
```swift
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;
```
Hook方法会返回一个遵循 __AspectToken__ 协议的方法，要取消对应的Hook，只需要调用代理对象的协议方法 `remove`就可以了

##### __AspectInfo__

![AspectInfo](http://upload-images.jianshu.io/upload_images/1749699-d78de2482a9fade0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/480)

__AspectInfo__ 对象遵循了 __AspectInfo__协议（同名协议），代表着切面信息，也是上文 __AspectBlockRef__ 中的首个参数

* instance ：当前被Hook的实例
* originalInvocation ：当前被Hook 原始的 __NSInvocation__ 对象
* arguments：所有方法的参数，一个计算属性，被调用的时候才会有值

这里特别提一下，参数是从 __NSInvocation__ 中拿到的
```swift
NSMutableArray *argumentsArray = [NSMutableArray array];
for (NSUInteger idx = 2; idx < self.methodSignature.numberOfArguments; idx++) {
  [argumentsArray addObject:[self aspect_argumentAtIndex:idx] ?: NSNull.null];
}
```
这里参数下标是从2开始的，因为下标 0，1，已经分别对应了 __消息接受对象__和 __selector__, 对于参数装箱细节，可以细看 __Aspects__ 的内部 __NSInvocation__ 分类

##### __AspectIdentifier__

![AspectIdentifier](http://upload-images.jianshu.io/upload_images/1749699-f667ab375a89d88b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

实际上 __AspectIdentifier__ 就是 __aspect_hookSelector__ 函数的返回值
```swift
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;
```
__AspectIdentifier__ 提供了remove 方法的实现，然而我并没有在源码中见到 __AspectIdentifier__ 有声明遵循 __<AspectToken>__协议

以下方法对需要hook的类进行信息封装操作，方法内部对usingBlock参数中的Block进行适配检查
```swift
+ (instancetype)identifierWithSelector:(SEL)selector object:(id)object options:(AspectOptions)options block:(id)block error:(NSError **)error 
{
    // 获取Block的方法签名
    NSMethodSignature *blockSignature =   aspect_blockMethodSignature(block, error); // TODO: check   signature compatibility, etc.
    // 兼容性检测
    if (!aspect_isCompatibleBlockSignature(blockSignature, object, selector, error)) {
        return nil;
    }
    // hook 信息封装
    AspectIdentifier *identifier = nil;
    if (blockSignature) {
        identifier = [AspectIdentifier new];
        identifier.selector = selector;
        identifier.block = block;
        identifier.blockSignature = blockSignature;
        identifier.options = options;
        identifier.object = object; // weak
    }
    return identifier;
}
```

##### __AspectsContainer__

![AspectsContainer](http://upload-images.jianshu.io/upload_images/1749699-11de74c7fe47441a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

__AspectsContainer__ 顾名思义，也就是切面的容器类，内部根据不同的option选项将 __AspectIdentifier__ 放入不同的容器内
*  NSArray *beforeAspects  对应 AspectPositionBefore 选项
*  NSArray *insteadAspects 对应 AspectPositionInstead 选项
*  NSArray *afterAspects 对应 AspectPositionAfter 选项

要注意的是 __Asepcts__通过 __aspect_getContainerForObject__ 方法从关联的对象获取
```swift
static AspectsContainer *aspect_getContainerForObject(NSObject *self, SEL selector) {
     // 获取方法别名（原方法添加 aspects_ 前缀）
    SEL aliasSelector = aspect_aliasForSelector(selector);
    AspectsContainer *aspectContainer = objc_getAssociatedObject(self, aliasSelector);
    if (!aspectContainer) {
        aspectContainer = [AspectsContainer new];
       // 以方法别名来关联 AspectsContainer 容器对象
        objc_setAssociatedObject(self, aliasSelector, aspectContainer, OBJC_ASSOCIATION_RETAIN);
    }
    return aspectContainer;
}
```

##### __AspectTracker__

![AspectTracker](http://upload-images.jianshu.io/upload_images/1749699-f5ce89f6ffaaadf6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

__AspectTracker__ 代表着对切面的追踪，存储在全局字典 __swizzledClassesDict__ 中，从子类向上追踪记录信息
* __selectorNames__ 记录当前被追踪的类需要hook的方法名
* __selectorNamesToSubclassTrackers__ 通过`addSubclassTracker: hookingSelectorName `记录子类的 __AspectTracker__ 对象 
  
```swift
// Add the selector as being modified.
AspectTracker *subclassTracker = nil;
do {
    tracker = swizzledClassesDict[currentClass];
    if (!tracker) {
        tracker = [[AspectTracker alloc] initWithTrackedClass:currentClass];
        swizzledClassesDict[(id<NSCopying>)currentClass] = tracker;
    }
    if (subclassTracker) {
        [tracker addSubclassTracker:subclassTracker hookingSelectorName:selectorName];
    } else {
        [tracker.selectorNames addObject:selectorName];
    }

    // All superclasses get marked as having a subclass that is modified.
    subclassTracker = tracker;
}while ((currentClass = class_getSuperclass(currentClass)));
```
> ##### __Aspects核心解析__ 

![核心](http://upload-images.jianshu.io/upload_images/1749699-78b9d92ab88d6398.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

##### __方法入口__

![方法入口](http://upload-images.jianshu.io/upload_images/1749699-748843211ef4038c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

```swift
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error {
    return aspect_add((id)self, selector, options, block, error);
}

/// @return A token which allows to later deregister the aspect.
- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error {
    return aspect_add(self, selector, options, block, error);
}
```
__Aspects__ 既支持对类的 Hook，也支持对实例的Hook,  其核心在于 __aspect_add__  方法

```swift
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {

    __block AspectIdentifier *identifier = nil;
    aspect_performLocked(^{
        // 判断是否允许Hook类
        if (aspect_isSelectorAllowedAndTrack(self, selector, options, error)) {
            // 获取关联容器
            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
            // hook信息封装成AspectIdentifier对象
            identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
            if (identifier) {
                [aspectContainer addAspect:identifier withOptions:options];

                // hook 核心操作
                aspect_prepareClassAndHookSelector(self, selector, error);
            }
        }
    });
    return identifier;
}
```
基本上 __aspect_add__ 核心操作有三步
* __aspect_isSelectorAllowedAndTrack__方法 判断是否允许 Hook
* hook信息封装成AspectIdentifier对象
* __aspect_prepareClassAndHookSelector__ 方法执行 Hook 操作（核心操作）

接下来一步步进行解析
##### __aspect_isSelectorAllowedAndTrack__ 判断是否允许Hook

1、黑名单过滤
```swift
static NSSet *disallowedSelectorList;
static dispatch_once_t pred;
dispatch_once(&pred, ^{
    disallowedSelectorList = [NSSet setWithObjects:@"retain", @"release", @"autorelease", @"forwardInvocation:", nil];
});

NSString *selectorName = NSStringFromSelector(selector);
if ([disallowedSelectorList containsObject:selectorName]) {
    ......
    return NO;
}
```
特殊方法不允许hook

2、__dealloc__ 方法hook只允许使用 __AspectPositionBefore__ 选项
```swift
AspectOptions position = options&AspectPositionFilter;
if ([selectorName isEqualToString:@"dealloc"] && position != AspectPositionBefore) {
    ......
    return NO;
}
```
3、过滤无法响应的方法
```swift
if (![self respondsToSelector:selector] && ![self.class instancesRespondToSelector:selector]) {
    return NO;
}
```
4、实例和类过滤

在此之前插播下元类的概念

![元类的概念](http://upload-images.jianshu.io/upload_images/1749699-51d566fc971ea2ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/480)
 
元类的定义：__元类是类对象的类__ 

还有需要注意的一点，`class_isMetaClass(object_getClass(self))` 使用的是 `object_getClass ` 方法 而不是类似 `[obj class]`的方法
两者有何区别呢？

|        |     object_getClass    |  [obj class]    |
| :-------------: |:-------------:| :-----:|
| 实例对象      | isa指针指向 | isa指针指向 |
| 类对象      | isa指针指向     |  对象本身 |
| 元类对象      | isa指针指向     |  对象本身 |

好了，接下来我们看下过滤的代码
```swift
if (class_isMetaClass(object_getClass(self))) {
  ......
}else{
  return YES;
}
```
在这里，笔者认为 `class_isMetaClass(object_getClass(self))` 作用是判断 hook 的对象是 __实例__ 还是 __类__

```swift
NSObject *obj = [[NSObject alloc] init];
[obj aspect_hookSelector:@selector(copy) withOptions:AspectPositionAfter usingBlock:^(id <AspectInfo> info){
} error:nil];

Class getClass = object_getClass(self);
BOOL isMeta = class_isMetaClass(getClass); // 打印 NO
```
如果是实例对象的话 `getClass ` 也就是isa指针，指向了对应的类，然后 `class_isMetaClass `判断自然是为NO

```swift
[NSObject aspect_hookSelector:@selector(copy) withOptions:AspectPositionAfter usingBlock:^(id <AspectInfo> info){

} error:nil];

Class getClass = object_getClass(self);
BOOL isMeta = class_isMetaClass(getClass); // 打印 YES
```
如果是类对象的话 `getClass ` 也就是isa指针，指向了对应的元类，然后 `class_isMetaClass `判断自然是为YES

如果`class_isMetaClass(object_getClass(self))`返回YES，也就是说我们 Hook 的对象是类对象的话
```swift
Class klass = [self class];
NSMutableDictionary *swizzledClassesDict = aspect_getSwizzledClassesDict();
Class currentClass = [self class];

//  已经hook过的方法不再进行重复hook
AspectTracker *tracker = swizzledClassesDict[currentClass];
if ([tracker subclassHasHookedSelectorName:selectorName]) {
    return NO;
}

// 向上查找父类
do {
    tracker = swizzledClassesDict[currentClass];
    if ([tracker.selectorNames containsObject:selectorName]) {
        if (klass == currentClass) {
            // 如果是已经遍历到了父类顶层
            return YES;
        }
        //  已经hook过的方法不再进行重复hook
        return NO;
    }
} while ((currentClass = class_getSuperclass(currentClass)));

// 向上查找父类，生成 AspectTracker 信息
currentClass = klass; 
AspectTracker *subclassTracker = nil;
do {
    tracker = swizzledClassesDict[currentClass];
    if (!tracker) {
        tracker = [[AspectTracker alloc] initWithTrackedClass:currentClass];
        swizzledClassesDict[(id<NSCopying>)currentClass] = tracker;
    }
    if (subclassTracker) {
        [tracker addSubclassTracker:subclassTracker hookingSelectorName:selectorName];
    } else {
        [tracker.selectorNames addObject:selectorName];
    }

    // All superclasses get marked as having a subclass that is modified.
    subclassTracker = tracker;
}while ((currentClass = class_getSuperclass(currentClass)));
```
所以，hook 实例方法不需要向上遍历父类方法，这也符合直觉和逻辑

__Aspects__ hook 之前做了非常健全的前置检查，非常值得学习！

![](http://upload-images.jianshu.io/upload_images/1749699-52b589af3a06655a.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
OK，下一个tips

__Hook 信息封装成AspectIdentifier对象__

```swift
+ (instancetype)identifierWithSelector:(SEL)selector object:(id)object options:(AspectOptions)options block:(id)block error:(NSError **)error {

    NSMethodSignature *blockSignature = aspect_blockMethodSignature(block, error); // TODO: check signature compatibility, etc.
    if (!aspect_isCompatibleBlockSignature(blockSignature, object, selector, error)) {
        return nil;
    }

    AspectIdentifier *identifier = nil;
    if (blockSignature) {
        identifier = [AspectIdentifier new];
        identifier.selector = selector;
        identifier.block = block;
        identifier.blockSignature = blockSignature;
        identifier.options = options;
        identifier.object = object; // weak
    }
    return identifier;
}
```
方法首先通过`aspect_blockMethodSignature `提取 Block 的方法签名
```swift
static NSMethodSignature *aspect_blockMethodSignature(id block, NSError **error) {
    AspectBlockRef layout = (__bridge void *)block;
    // 通过判断block的flags 来确定Block是否存在方法签名
    if (!(layout->flags & AspectBlockFlagsHasSignature)) {
        return nil;
    }
    // 获取 descriptor 
    void *desc = layout->descriptor;
    /* 计算内存地址偏移量来确定 signature 的位置 */
    // 加上 reserved 和 size的偏移量 （类型皆为unsigned long int 所以 乘以 2）
    desc += 2 * sizeof(unsigned long int);
    // 如果有 copy 和 dispose 的 flag 需要 加上对应偏移量
    if (layout->flags & AspectBlockFlagsHasCopyDisposeHelpers) {
        desc += 2 * sizeof(void *);
    }
    if (!desc) {
        return nil;
    }
    // 获取到了真正的方法签名
    const char *signature = (*(const char **)desc);
    return [NSMethodSignature signatureWithObjCTypes:signature];
}
```
涨姿势了，通过如上代码我们知道了如何获得Block的方法签名，
我们可以先理解消化本文解释 __AspectBlockRef__ 的内存布局，然后再来理解这段代码

至于代码中的二级指针`(*(const char **)desc)`
```
// descriptor 是一个结构体指针，保存的是结构体的起始地址
struct {
    const char *signature;
} *descriptor;
// 结构体指针 赋值给 泛型指针desc
void *desc = layout->descriptor;
//  当我们获取到真正的方法签名，此时 desc 已经指向了 signature 所在的地址
//  signature 的类型是 const char *，那么 signature 的地址自然就是const char **咯
//  最后我们要取 desc 指针中的值，也就是 *obj 咯
const char ** obj = (const char **)desc;
const char *signature = *obj;
```
恩，没毛病，下一个Tips, 也是我们的重头戏

__aspect_prepareClassAndHookSelector 方法执行 Hook 操作__

我们来看下`aspect_prepareClassAndHookSelector`的源码
```swift
static void aspect_prepareClassAndHookSelector(NSObject *self, SEL selector, NSError **error) {

    Class klass = aspect_hookClass(self, error);
    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);
    if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
        // Make a method alias for the existing method implementation, it not already copied.
        const char *typeEncoding = method_getTypeEncoding(targetMethod);
        SEL aliasSelector = aspect_aliasForSelector(selector);
        if (![klass instancesRespondToSelector:aliasSelector]) {
            __unused BOOL addedAlias = class_addMethod(klass, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
        }

        // We use forwardInvocation to hook in.
        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
    }
}
```

这里源码中比较重要和核心的在于 `aspect_hookClass ` 方法
```swift
static Class aspect_hookClass(NSObject *self, NSError **error) {
    NSCParameterAssert(self);
        // class
  Class statedClass = self.class;
       // isa指针
  Class baseClass = object_getClass(self);
  NSString *className = NSStringFromClass(baseClass);

    // 已经子类化过了，也就是已经添加过子类后缀 “_Aspects_”
  if ([className hasSuffix:AspectsSubclassSuffix]) {
    return baseClass;
  }
    // 如果 Hook 的是 类对象，那么混写 类对象的ForwardInvocation方法 （类对象的指针指向本身）
    else if (class_isMetaClass(baseClass)) {
        return aspect_swizzleClassInPlace((Class)self);
    }
    // 如果 Hook 的是 一个 已经被kvo子类化的实例对象，我们需要混写它的 metaClass 的ForwardInvocation方法
    else if (statedClass != baseClass) {
        return aspect_swizzleClassInPlace(baseClass);
    }

    // 混写实例对象

   //  添加 默认后缀 _Aspects_ 
  const char *subclassName = [className stringByAppendingString:AspectsSubclassSuffix].UTF8String;
   //  获取 添加默认后缀后的类
  Class subclass = objc_getClass(subclassName);

  if (subclass == nil) {
    // 如果还没有动态创建过子类
    subclass = objc_allocateClassPair(baseClass, subclassName, 0);
    if (subclass == nil) {
            return nil;
        }
        // 混写 forwardInvocation:方法
    aspect_swizzleForwardInvocation(subclass);
        // 替换 class 的 IMP 指针
        // subClass.class = statedClass
    aspect_hookedGetClass(subclass, statedClass);
        // subClass.isa.class = statedClass 
    aspect_hookedGetClass(object_getClass(subclass), statedClass);
        // 注册新类
    objc_registerClassPair(subclass);
  }
    // 复写 isa 指针，指向新的类
  object_setClass(self, subclass);
  return subclass;
}
```
值得注意的是

```swift
// 如果 Hook 的是 类对象，那么混写 类对象 （类对象的指针指向本身）
Class baseClass = object_getClass(self);
else if (class_isMetaClass(baseClass)) {
    return aspect_swizzleClassInPlace((Class)self);
}
// 如果 Hook 的是 一个 已经被kvo子类化的实例对象，我们需要混写它的 metaClass
else if (statedClass != baseClass) {
    return aspect_swizzleClassInPlace(baseClass);
}
```
`class_isMetaClass(baseClass)`前面已经说过了，是用来判断是否 hook 的是类对象，那么 `statedClass != baseClass`  是什么含义呢

首先我们得要先知道 KVO 实现的原理: __当观察某对象 A 时，KVO 机制动态创建一个对象A当前类的子类，并为这个新的子类重写了被观察属性 keyPath 的 setter 方法__

从前文我们知道，实例对象  `object_getClass ` 和 `[obj class]` 的指向是一致的，当hook被KVO子类化的实例时候，实例对象的isa指针的指向 和 class 的指向才会不一致 


我们还学会了动态创建一个类的流程
* objc_allocateClassPair
* class_addMethod
* class_addIvar
*  objc_registerClassPair

> 小结：对类对象的hook是通过混写类对象的 __forwardInvocation__ 方法来实现，对实例对象的 hook 是通过子类化，然后混写子类的 __forwardInvocation__ 来实现的

emmmmm， 接下来我们聊一聊混写__forwardInvocation__ 的一些细节

> ##### aspect_swizzleForwardInvocation 混写  forwardInvocation

![方法转发流程](http://upload-images.jianshu.io/upload_images/1749699-e6581be2bcc35030.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)
方法转发流程，这里就不赘述，__Aspects__ 核心思想就是通过混写 __forwardInvocation__ 来实现切面执行自定义代码

```swift
static void aspect_swizzleForwardInvocation(Class klass) {
    NSCParameterAssert(klass);
    // If there is no method, replace will act like class_addMethod.
    IMP originalImplementation = class_replaceMethod(klass, @selector(forwardInvocation:), (IMP)__ASPECTS_ARE_BEING_CALLED__, "v@:@");
    if (originalImplementation) {
        class_addMethod(klass, NSSelectorFromString(AspectsForwardInvocationSelectorName), originalImplementation, "v@:@");
    }
}
```
上述代码将类的 __forwardInvocation__ 方法 IMP 替换成 __ __ASPECTS_ARE_BEING_CALLED__ __ ，原来的 IMP 与 __AspectsForwardInvocationSelectorName__ 相关联了

接下来我们看看  __ __ASPECTS_ARE_BEING_CALLED__ __ 方法

```swift
static void __ASPECTS_ARE_BEING_CALLED__(__unsafe_unretained NSObject *self, SEL selector, NSInvocation *invocation) {

    SEL originalSelector = invocation.selector;
  SEL aliasSelector = aspect_aliasForSelector(invocation.selector);
    invocation.selector = aliasSelector;
    // 获取实例的容器
    AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
    // 获取类的容器
    AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
    AspectInfo *info = [[AspectInfo alloc] initWithInstance:self invocation:invocation];
    NSArray *aspectsToRemove = nil;

    // Before hooks.
    aspect_invoke(classContainer.beforeAspects, info);
    aspect_invoke(objectContainer.beforeAspects, info);

    // Instead hooks.
    BOOL respondsToAlias = YES;
    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
        // 执行 insteadAspects 中的操作
        aspect_invoke(classContainer.insteadAspects, info);
        aspect_invoke(objectContainer.insteadAspects, info);
    }else {
         // 正常执行方法 附加判断是否能够响应方法
        Class klass = object_getClass(invocation.target);
        do {
            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
                [invocation invoke];
                break;
            }
        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
    }

    // After hooks.
    aspect_invoke(classContainer.afterAspects, info);
    aspect_invoke(objectContainer.afterAspects, info);

   
    if (!respondsToAlias) {
        invocation.selector = originalSelector;
        SEL originalForwardInvocationSEL = NSSelectorFromString(AspectsForwardInvocationSelectorName);

        // 如何hook对象不响应 aliasSelector 那么执行原有的 forwardInvocation
        if ([self respondsToSelector:originalForwardInvocationSEL]) {
            ((void( *)(id, SEL, NSInvocation *))objc_msgSend)(self, originalForwardInvocationSEL, invocation);
        }else {
            // 如果类没有实现 forwardInvocation 方法的话，将会抛出 doesNotRecognizeSelector 错误
            [self doesNotRecognizeSelector:invocation.selector];
        }
    }

    // 移除 aspectsToRemove 中的 AspectIdentifier，并执行  AspectIdentifier 的 remove 方法
    [aspectsToRemove makeObjectsPerformSelector:@selector(remove)];
}
```

上述代码执行了对应的 beforeAspects，insteadAspects，afterAspects
如果有`insteadAspects `  操作则执行`insteadAspects ` 操作，否则执行 `aliasSelector `， 若不响应 `aliasSelector `, 那么将执行 hook 之前的 `forwardInvocation `的方法，没有实现 `forwardInvocation `那么将抛出 `doesNotRecognizeSelector ` 异常

我们再来看看 `aspect_invoke ` 执行切面的细节

> #####  aspect_invoke

```swift
#define aspect_invoke(aspects, info) \
for (AspectIdentifier *aspect in aspects) {\
    [aspect invokeWithInfo:info];\
    if (aspect.options & AspectOptionAutomaticRemoval) { \
        aspectsToRemove = [aspectsToRemove?:@[] arrayByAddingObject:aspect]; \
    } \
}
```
宏定义执行了__AspectIdentifier__  的 `invokeWithInfo:`方法

```swift
- (BOOL)invokeWithInfo:(id<AspectInfo>)info {
    NSInvocation *blockInvocation = [NSInvocation invocationWithMethodSignature:self.blockSignature];
    NSInvocation *originalInvocation = info.originalInvocation;
    NSUInteger numberOfArguments = self.blockSignature.numberOfArguments;

    // 检查参数是否适配，已经是第二次检查了
    if (numberOfArguments > originalInvocation.methodSignature.numberOfArguments) {
        return NO;
    }

    // 设置参数
    if (numberOfArguments > 1) {
        [blockInvocation setArgument:&info atIndex:1];
    }
    
    // 遍历 originalInvocation 参数，将参数值复制到  blockInvocation
  void *argBuf = NULL;
    for (NSUInteger idx = 2; idx < numberOfArguments; idx++) {
        const char *type = [originalInvocation.methodSignature getArgumentTypeAtIndex:idx];
    NSUInteger argSize;
    NSGetSizeAndAlignment(type, &argSize, NULL);
        
    if (!(argBuf = reallocf(argBuf, argSize))) {
      return NO;
    }
        
    [originalInvocation getArgument:argBuf atIndex:idx];
    [blockInvocation setArgument:argBuf atIndex:idx];
    }
    // 执行 blockInvocation
    [blockInvocation invokeWithTarget:self.block];
    
    if (argBuf != NULL) {
        free(argBuf);
    }
    return YES;
}
```

这里比较有意思的是 `[blockInvocation setArgument:&info atIndex:1];` 按照正常来说 参数下标0 和 1 分别是 target, selector

![](http://upload-images.jianshu.io/upload_images/1749699-9ca4bc445b6928b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/240)


打印 `blockInvocation ` 的方法签名，发现参数下标为 1 类型 并不是__selector__ ，而是 __id <AspectInfo>__ 
有知道的大佬希望能够指出提点下

完结，撒花


> 还有什么？

* __aspect_remove__ 对Hook的清理操作，这里就不赘述了

* 有对实践感兴趣的话，可以看笔者粗浅的架构实例：[NonBaseClass-MVVM-ReactiveObjc (干货带代码)](https://www.jianshu.com/p/921dd65e79cb) 

* 最后祝愿大家新年快乐，过个好年，且行且珍惜，么么哒😘
![](http://upload-images.jianshu.io/upload_images/1749699-ae89c523379ff672.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/320)

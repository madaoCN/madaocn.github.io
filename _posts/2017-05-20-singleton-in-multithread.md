---
layout: post
title: IOS单例模式下多线程和继承写法总结
date: 2017-05-20
Author: madaoCN
categories: 
tags: [开发总结]
comments: true
---

* 前言
单例模式基本上是最简单的设计模式，它使类的一个对象成为整个系统的唯一实例。

* 基础款

```
#import "Singleton.h"

@implementation Singleton

+ (Singleton *)sharedInstance
{
    static Singleton *sharedSingleton = nil;
    if (sharedSingleton) {
        sharedSingleton = [[Singleton alloc] init];
    }
    return sharedSingleton;
}

@end

```
基本上是最简单的单例创建形式，首先检查唯一实例是否已经创建，若没有则创建新的实例并且返回，考虑到多线程等因素，这不够好。

* 多线程款

```
+ (Singleton *)sharedInstance
{
    static Singleton *sharedSingleton = nil;
    static dispatch_once_t token;
    dispatch_once(&token, ^{
        sharedSingleton = [[Singleton alloc] init];
    });

    return sharedSingleton;
}
```

`dispatch_once ` 该方法的作用就是执行且在整个程序的声明周期中，仅执行一次某一个block对象，满足线程要求。

* 更严谨款
然而，我们直接调用 `[[sharedInstance alloc] init]`方法显然并不能保证放回对象的同一个实例。我们需要隔绝各种导致不返回同一实例的写法。苹果提供了复杂的标准写法 [传送门](https://developer.apple.com/legacy/library/documentation/Cocoa/Conceptual/CocoaFundamentals/CocoaObjects/CocoaObjects.html#//apple_ref/doc/uid/TP40002974-CH4-SW32)

```objc

+ (Singleton *)sharedInstance
{
    static Singleton *sharedSingleton = nil;
    static dispatch_once_t token;
    dispatch_once(&token, ^{
        sharedSingleton = [[super allocWithZone:NULL] init];
    });
    
    return sharedSingleton;
}


+ (id)allocWithZone:(NSZone *)zone
{
    return [self sharedInstance];
}

- (id)copyWithZone:(NSZone *)zone
{
    return self;
}

// mrc
- (id)retain
{
    return self;
}
// mrc
- (NSUInteger)retainCount
{
    return NSUIntegerMax;  //denotes an object that cannot be released
}
// mrc
- (void)release
{
    //do nothing
}
// mrc
- (id)autorelease
{
    return self;
}

```
为什么`        sharedSingleton = [[super allocWithZone:NULL] init];`用的是super，而不是self？ 因为我们已经重载了`allocWithZone `的对象分配方法，所以我们需要借助父类来实现内存分配。

* 子类化单例款
然而，由于我们重写了单例 的`allocWithZone `方法 ，并且把内存分配交给了`super`，那么我们不做修改的子类化单例，返回的实例始终将会是`Singleton `而不是`Singleton `的子类对象。
那么我们要怎样子类化单例呢？我们知道为对象创建分配内存的是`alloc`方法，我们得看看`alloc`方法是怎样实现的


**alloc**:
```
+ (id)alloc {
    return _objc_rootAlloc(self);
}

// Base class implementation of +alloc. cls is not nil.
// Calls [cls allocWithZone:nil].
id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}

// Call [cls alloc] or [cls allocWithZone:nil], with appropriate 
// shortcutting optimizations.
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    if (slowpath(checkNil && !cls)) return nil;

#if __OBJC2__
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        // No alloc/allocWithZone implementation. Go straight to the allocator.
        // fixme store hasCustomAWZ in the non-meta class and 
        // add it to canAllocFast's summary
        if (fastpath(cls->canAllocFast())) {
            // No ctors, raw isa, etc. Go straight to the metal.
            bool dtor = cls->hasCxxDtor();
            id obj = (id)calloc(1, cls->bits.fastInstanceSize());
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            obj->initInstanceIsa(cls, dtor);
            return obj;
        }
        else {
            // Has ctor or raw isa or something. Use the slower path.
            id obj = class_createInstance(cls, 0);
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            return obj;
        }
    }
#endif

    // No shortcuts available.
    if (allocWithZone) return [cls allocWithZone:nil];
    return [cls alloc];
}
```
我们注意到官方源码`_objc_rootAlloc `之上的注释
```
// Base class implementation of +alloc. cls is not nil.
// Calls [cls allocWithZone:nil].
```

  我们首先注意到`callAlloc `中的注释`No alloc/allocWithZone implementation. Go straight to the allocator.`，如果没有类没有实现alloc或者allocWithZone方法才会执行`#if __OBJC2__`和`#endif`之间的逻辑，否则执行`if (allocWithZone) return [cls allocWithZone:nil];`该语句
其次 `_objc_rootAlloc `方法的注释` cls is not nil.  Calls [cls allocWithZone:nil]`，如果cls不为空，则调用allocWithZone方法
显然，大部分情况下`alloc`直接调用的就是`allocWithZone `方法

** allocWithZone**:
```
+ (id)allocWithZone:(struct _NSZone *)zone {
    return _objc_rootAllocWithZone(self, (malloc_zone_t *)zone);
}

id
_objc_rootAllocWithZone(Class cls, malloc_zone_t *zone)
{
    id obj;

#if __OBJC2__
    // allocWithZone under __OBJC2__ ignores the zone parameter
    (void)zone;
    obj = class_createInstance(cls, 0);
#else
    if (!zone) {
        obj = class_createInstance(cls, 0);
    }
    else {
        obj = class_createInstanceFromZone(cls, 0, zone);
    }
#endif

    if (slowpath(!obj)) obj = callBadAllocHandler(cls);
    return obj;
}

```
allocWithZone 就比较简单了，在objc2下直接调用的是`class_createInstance `方法


** class_createInstance **
这个方法处于`<objc/runtime.h>`

```
/* Instantiating Classes */

/** 
 * Creates an instance of a class, allocating memory for the class in the 
 * default malloc memory zone.
 * 
 * @param cls The class that you wish to allocate an instance of.
 * @param extraBytes An integer indicating the number of extra bytes to allocate. 
 *  The additional bytes can be used to store additional instance variables beyond 
 *  those defined in the class definition.
 * 
 * @return An instance of the class \e cls.
 */
OBJC_EXPORT id class_createInstance(Class cls, size_t extraBytes)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0)
    OBJC_ARC_UNAVAILABLE;
```
然而`OBJC_ARC_UNAVAILABLE ` arc下不可用，不过没关系`-fno-objc-arc`来支持混编。
所以，最后的最后，支持子类化的单例就写好了

```
+ (Singleton *)sharedInstance
{
    static Singleton *sharedSingleton = nil;
    static dispatch_once_t token;
    dispatch_once(&token, ^{
//        sharedSingleton = [[super allocWithZone:NULL] init];
        sharedSingleton = [class_createInstance([self class], 0) init ];
    });
    
    return sharedSingleton;
}

```

That is all， thank you
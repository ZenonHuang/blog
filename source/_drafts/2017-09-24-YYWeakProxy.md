---
layout: post
date: 2017-09-24 10:25:00
title: YYKit(一)--YYWeakProxy 
category: 技术
keywords: iOS
description: 最近工作使用 CAAnimation ,它的代理为 strong ，使用 YYWeakProxy 解决。故而阅读其代码学习。
---


[YYWeakProxy v1.0.9](https://github.com/ibireme/YYKit/blob/4e1bd1cfcdb3331244b219cbd37cc9b1ccb62b7a/YYKit/Utility/YYWeakProxy.h)
==============

## 作用

关于 YYWeakProxy 的作用，在它的头文件就可以清楚的看到:

```
/**
 A proxy used to hold a weak object.
 It can be used to avoid retain cycles, such as the target in NSTimer or CADisplayLink.
 
 sample code:
 
     @implementation MyView {
        NSTimer *_timer;
     }
     
     - (void)initTimer {
        YYWeakProxy *proxy = [YYWeakProxy proxyWithTarget:self];
        _timer = [NSTimer timerWithTimeInterval:0.1 target:proxy selector:@selector(tick:) userInfo:nil repeats:YES];
     }
     
     - (void)tick:(NSTimer *)timer {...}
     @end
 */
```
YYWeakProxy 是用来持有一个 weak 对象的代理，避免循环引用。

这里引用链是：

>self -> timer -> proxy -> target (self)

由于 target 为弱应用，当 self 引用计数为 0 时, target 将为 nil, 于是打破了引用链。

## NSProxy

YYWeakProxy 继承于 NSProxy. NSProxy 是什么？

NSProxy 是除了NSObject之外的另一个基类。同时它也是一个抽象类，你可以通过继承它，并重写消息转发的方法，以实现消息转发到另一个实例的目的。

文档是这么说的:
>`NSProxy` implements the basic methods required of a root class, including those defined in the [`NSObjectProtocol`](https://developer.apple.com/documentation/objectivec/nsobjectprotocol) protocol. However, as an abstract class it doesn’t provide an initialization method, and it raises an exception upon receiving any message it doesn’t respond to. A concrete subclass must therefore provide an initialization or creation method and override the [`forwardInvocation(_:)`](https://developer.apple.com/documentation/foundation/nsproxy/1416417-forwardinvocation) and [`methodSignatureForSelector:`](https://developer.apple.com/documentation/foundation/nsproxy/1589828-methodsignatureforselector) methods to handle messages that it doesn’t implement itself

`NSProxy`可以说除了重载消息转发机制外没有别的用法，这也是它被设计的初衷，自己什么都不干，转给代理对象就好。往这个proxy发消息是注定会走消息转发的。

## NSProxy 和 NSObject 差异

两者同样都可以作为消息转发创建的代理类，但是存在一定的差异。

### 自动转发

通过继承自 `NSObject` 的代理类是不会自动转发 `respondsToSelector:和isKindOfClass:` 这两个方法的, 而继承自 `NSProxy` 的代理类却是可以的.

### NSObject 的所有 Category 方法不能完成转发

`valueForKey:` 是定义在 NSKeyValueCoding 这个 NSObject 的 `Category` 中的方法.

## NSProxy 多继承

## 实现

YYWeakProxy 作为 NSProxy 的子类， **`必须`**实现 forwardInvocation: methodSignatureForSelector: 方法进行对象转发，这是在苹果官方文档中说明的。

以开头的场景为例子,当发送消息时,proxy 的方法列表里找不到 tick: ,就会开始走消息转发。

### 消息转发机制

#### forwardingTargetForSelector

```
- (id)forwardingTargetForSelector:(SEL)selector {
    return _target;
}
```

**`forwardingTargetForSelector`** 会返回你需要转发消息的对象，假如返回的是 nil，那么就走到 `forwardInvocation:` 做转发处理。

这里返回的是 _target 对象，那么实际就是调用 _target 对应的 selector 。


#### methodSignatureForSelector
```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector {
    return [NSObject instanceMethodSignatureForSelector:@selector(init)];
}
```
//return [self.target methodSignatureForSelector:selector];

**`methodSignatureForSelector`** 用来生成方法签名，这个签名就是给forwardInvocation中的参数NSInvocation调用的。

这里调用的是 NSObject 的 init 方法的签名。


#### forwardInvocation

```
- (void)forwardInvocation:(NSInvocation *)invocation {
    void *null = NULL;
    [invocation setReturnValue:&null];
}
```
//[invocation invokeWithTarget:self.target];
**`forwardInvocation:`** 做消息转发的处理。 

##### setReturnValue

```
- (void)setReturnValue:(void *)retLoc;
```

[苹果文档](https://developer.apple.com/documentation/foundation/nsinvocation/1437848-setreturnvalue?language=objc) 对于 setReturnValue 的解释是： 

> Sets the receiver’s return value.

设置消息接受者的返回值。

##### getReturnValue







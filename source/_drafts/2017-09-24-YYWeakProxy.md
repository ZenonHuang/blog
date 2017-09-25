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
YYWeakProxy 是用来弱引用对象的代理，避免强引用循环。

## NSProxy

YYWeakProxy 继承于 NSProxy. NSProxy 是什么？

## 实现

一旦 target 被释放,会走消息转发的方法

```
- (void)forwardInvocation:(NSInvocation *)invocation {
    void *null = NULL;
    [invocation setReturnValue:&null];
}
```

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector {
    return [NSObject instanceMethodSignatureForSelector:@selector(init)];
}
```

### setReturnValue

```
- (void)setReturnValue:(void *)retLoc;
```

[苹果文档](https://developer.apple.com/documentation/foundation/nsinvocation/1437848-setreturnvalue?language=objc) 对于 setReturnValue 的解释是： 

> Sets the receiver’s return value.

### getReturnValue




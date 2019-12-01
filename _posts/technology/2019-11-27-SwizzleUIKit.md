---
layout: post
date: 2019-11-27 11:33:46
title:  Method Swizzle 打点 UIKit 的问题
category: 技术
description: 利用 Method Swizzle 统一进行日志打点，基本上介绍 AOP 使用的文章都会提到，经过实际使用，某些场景是有问题的。特此记录。 
---

# 问题背景

在这个数据为王的时代,市面有用户的 APP,都会进行日志打点，我们也不例外。

如果一个个页面去打点，实在费时费力，我们不免想通过 AOP 方式去 Hook 我们想要的方法，就能做到一次打点，统一管理的目的了。

比如对于一个页面的进出，只需要对 `viewWillAppear` 和 `viewWillDisappear` 做记录。

有鉴于此，iOS 上对于 UIKit ,我们项目有了一套基于 ` Method Swizzle ` 实现的的事件打点方案。

然而我们发现了一个问题:
> 加入某一个第三方库，进行使用出现了崩溃，经过排查，确定到和父子类初始化顺序有关。

## Method Swizzle 方案

先简单的说一下我们打点追踪的 `Method Swizzle ` 方案。

###  随着 App 启动开始做方法交换

我们在 APP 启动时，在 `TrackerCenter` 中开始做方法交换:

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.
    [[HZTrackerCenter sharedInstance] beginTracker];
    return YES;
}

```


`beginTracker` 中各个 UI 类的方法做交换:


```objc
- (void)beginTracker
{
    ...
    [UIControl HZ_swizzle];
    [UICollectionView HZ_swizzle];
    ...
}

```


###  Method Swizzle 的统一方法

方法交换的关键代码，统一使用到的方法 `HZ_swizzleMethod:newSel:` ，也是市面常见的代码,如下:

```
+ (BOOL)HZ_swizzleMethod:(SEL)originalSel newSel:(SEL)newSel {
    Method originMethod = class_getInstanceMethod(self, originalSel);
    Method newMethod = class_getInstanceMethod(self, newSel);
    
    if (originMethod && newMethod) {
        if (class_addMethod(self, originalSel, method_getImplementation(newMethod), method_getTypeEncoding(newMethod))) {
            
            IMP orginIMP = method_getImplementation(originMethod);
            class_replaceMethod(self, newSel, orginIMP, method_getTypeEncoding(originMethod));
        } else {
            method_exchangeImplementations(originMethod, newMethod);
        }
        return YES;
    }
    return NO;
}
```

思路是:
1.根据 SEL 取得两个 Method,判断两个 Method 是否存在，是否可以进行交换
2.使用 `class_addMethod()` ,如果类中没有实现 originalSel 对应的方法，那就先添加 Method . 如果本类中包含一个同名的实现，则函数会返回NO,这里就会直接使用  `method_exchangeImplementations` 对两个方法进行交换。
3.当 `class_addMethod()` 成功，则表示 originalSel 的 IMP,已经为 newMethod 的实现。下一步则使用 `class_replaceMethod` 对 newSel 进行 IMP 的替换。

> note: 而为什么不直接使用 ` method_exchangeImplementations`, 而是先添加再交换，是为了保证只在子类中交换方法，不影响父类。
> 如果本类中没有 originalSel 的实现，class_getInstanceMethod() 返回的是某父类 Method 对象，直接交换的后果，会把父类的 IMP 跟这个类的 Swizzle IMP 交换。影响到整个父类和其子类。

不了解 SEL,Method,IMP ，建议可以看一看这篇 [Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/ "Objective-C Runtime")

## 不同的类，方法交换的策略

#### 普通 UI 类，直接操作

对于普通的 UI 类，如 `UIControl` , 我们直接就进行交换了,如下:

```objc


@implementation UIControl (Tracker)

- (void)HZ_sendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event {
 
    if (target && action && ![NSStringFromSelector(action) hasPrefix:@"_"]) {
        //进行事件记录
    }
    
    [self HZ_sendAction:action to:target forEvent:event];
}

+ (void)HZ_swizzle {
    [UIControl HZ_swizzleMethod:@selector(sendAction:to:forEvent:)
                         newSel:@selector(ET_sendAction:to:forEvent:)];
}

@end


```

#### 特定 UI 类，对代理对象操作


不同于其它 UI 类，对于 `UITableView` 和 `UICollectionView` ，想要统计它们的点击事件，就不可以对其类直接进行交换。因为它的对应的事件实现，是在 `delegate` 对象中。

于是在 `setDelegate:` 的时机，对 `delegate ` 对象 进行操作。

例如 `UICollectionView`


```objc

@implementation UICollectionView (Tracker)

- (void)HZ_setDelegate:(id<UICollectionViewDelegate>)delegate {
    if ([delegate isKindOfClass:[NSObject class]]) {
        SEL sel = @selector(collectionView:didSelectItemAtIndexPath:);
        
        //newSel 名： HZ_collectionView:didSelectItemAtIndexPath:
        SEL newSel = [NSObject HZ_newSelFormOriginalSel:sel];
        
        Method originMethod = class_getInstanceMethod(delegate.class, sel);
          
        if (originMethod && ![delegate.class HZ_methodHasSwizzed:sel]) {

        
            IMP newIMP =  (IMP)HZ_collectionViewDidSelectRowAtIndexPath;
            class_addMethod(delegate.class, newSel,newIMP, method_getTypeEncoding(originMethod));
            
            [delegate.class HZ_swizzleMethod:sel newSel:newSel];
            [delegate.class HZ_setMethodHasSwizzed:sel];
        }
    }
    [self HZ_setDelegate:delegate];
}

+ (void)HZ_swizzle {
    [UICollectionView HZ_swizzleMethod:@selector(setDelegate:)
                                newSel:@selector(HZ_setDelegate:)];
}

@end
```

这里的思路是:

1. 根据 Sel 生成一个 newSel。
2. 判断 Sel 的 Method 存在，并且 sel 还没有被被 swizzle 过。
3. 对 delegate 对象的类，增加 newSel 实现。
4. 对 sel 和 newSel 进行交换
5. 对已经 swizzle 到 Sel 标记

# 问题场景

上述对于特定的 UI 类， `UITableView` 和  `UICollectionView` 的代理对象做 swizzle 。乍看是是并没有问题的，并且稳定运行了很久。

直到有一天，我们引入一个第三方库。

这个第三方是一个 UIView ,不过这个 UIView 是一个 UICollectionView 的 delegate 对象.

这本身也并无问题。

而出于业务需要，我们继承了它，实现了一个子类。子类中也没有进行 UICollectioView 的代理方法覆写。  

这个时候，就出现了循环调用的问题。


## 出现问题的流程

排查发现，只要父类先于子类进行交换的操作，之后点击子类，就会发生循环调用，导致崩溃。

我们来复原整个问题的流程:

1.按照上文方案，父类进行 swizzle ，结果为:

| SEL | OriginSel | NewSel |
| --- | --- | --- |
| IMP | NewImp | OriginImp  |


 - OriginSel 实现对应 NewImp
 - NewSel 实现对应 OriginImp
 
2.对子类进行 swizzle,先插入了 NewSel ，实现对应为 NewImp :

| SEL | NewSel | 
| --- | --- | 
| IMP | NewImp | 

3.交换方法中，进行了 `class_addMethod`,OriginImp 对应实现为 NewImp：

| SEL | OriginSel | NewSel |
| --- | --- | --- |
| IMP | NewImp | NewImp  |

4.交换方法中，对 NewSel 进行 `class_replaceMethod`:

```objc

IMP orginIMP = method_getImplementation(originMethod);
class_replaceMethod(self, newSel, orginIMP,method_getTypeEncoding(originMethod));

```

在进行上面 `步骤 4` 的时候，问题就显现了，因为子类中未实现 `originalSel`。而 `originMethod` 通过 `class_getInstanceMethod(self, originalSel)` 得来，获取到的实际上是父类 `originalSel` 的实现，看到上表，父类已经被交换，获取的实现为  `NewImp`.

所以这时候 replace 操作，并没有达到真正的目的。

最后子类的结果仍然为 ：

| SEL | OriginSel | NewSel |
| --- | --- | --- |
| IMP | NewImp | NewImp  |


## 问题表现

到了这一步，开始还原发生循环的过程.

其中 `NewImp` 的实现为一个用来打日志的静态方法: 

```objc

void HZ_collectionViewDidSelectRowAtIndexPath(id self, SEL _cmd, UICollectionView *collectionView, NSIndexPath *indexPath) {
    
    //do your track thing
    
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
    
    SEL sel = [NSObject HZ_newSelFormOriginalSel:@selector(collectionView:didSelectItemAtIndexPath:)];
    [self performSelector:sel
               withObject:collectionView 
               withObject:indexPath];
#pragma clang diagnostic pop
}
```

上面的方法，最后会再调用 NewSel.

正常的调用链应该是: 
>OriginSel->NewImp->NewSel->OriginImp.

在 NewImp 中打点完成后，调用系统真正的 OriginImp,就结束了。


而我们现在 Swizzle 后的子类，调用链是这样的:
> OriginSel->NewImp->NewSel->NewImp->NewSel->NewImp..

这样就出现了 `NewImp->NewSel->NewImp->NewSel..`的俄罗斯套娃,所以引发了崩溃。

# 问题解决

经过上述分析，发生问题就在于，子类进入 Swizzle 后，子类本身没有实现，OriginalMethod 使用 `method_getImplementation` 方法,是会拿父类的实现。父类已经交换，结果拿到的是 NewImp。

如果要解决问题，由于无法去控制使用者调用父子类的顺序，我们要在进行 Swizzle 前进行判断，避免这样的情况发生。

## 初步解决方案 

第一时间，我想到的是通过 OriginalSel 的实现进行判断。

因为无论父子类， OriginalSel 的实现都是拿的父类中的，第二次去拿，会发生危险。

通过标志位的真假来决定，是否进行 Swizzle：

- 当父类 OriginalSel 进行过修改，子类再进来，就不再进行 Swizzle  操作。

- 当子类 OriginalSel 进行修改，父类进来，也不再进行 Swizzle  操作。

但测试证明，如果再加上一个 `孙子类`，这时候又将发生问题。仍然和之前的类似，感兴趣的可以试验一下。

## 最终解决方案

最后的方案，就是直接切入最核心的一点，判断 OriginalIMP 和 NewIMP。

问题发生，就是在于 OriginalIMP 实际上变成了 NewIMP 。

那么只要在 Swizzle 前，取出来 OriginalIMP 和 NewIMP 直接比对:

- 如果相同，证明有父类实现已经进行过交换。则不再做 Swizzle。
- 不是的话，则可以进行安全的 Sizzle 操作。


核心判断如下：

```
IMP originIMP = method_getImplementation(originMethod);
IMP newIMP =  (IMP)HZ_collectionViewDidSelectRowAtIndexPath;

if (originMethod && !(originIMP==newIMP))
{      
    class_addMethod(delegate.class, newSel,newIMP, method_getTypeEncoding(originMethod));
    [delegate.class HZ_swizzleMethod:sel newSel:newSel];
}
```


经过这样处理之后，我们的日志打点就可以正常运行，再也不用担心 父子重复 Swizzle 导致循环调用了。


> 相关的示例代码，包括了问题和解决方案，都已经上传到 [GitHub](https://github.com/ZenonHuang/MyDemos/tree/master/CollectionViewHook) .


# MLeaksFinder 

MLeaksFinder 是一个在实际工程中，非常实用的工具。

出去面试也会常常问到，之前前对它的了解，仅处于扫了一眼，知道是通过延迟调用来做检查而已。

这次打算从几个关键问题入手，学习它的实现。

关于 MLeaksFinder:

>[MLeaksFinder](https://github.com/Zepo/MLeaksFinder) 是 iOS 平台的自动内存泄漏检测工具，引进 MLeaksFinder 后，就可以在日常的开发，调试业务逻辑的过程中自动地发现并警告内存泄漏。开发者无需打开 instrument 等工具，也无需为了找内存泄漏而去跑额外的流程。并且，由于开发者是在修改代码之后一跑业务逻辑就能发现内存泄漏的，这使得开发者能很快地意识到是哪里的代码写得问题。这种及时的内存泄漏的发现在很大的程度上降低了修复内存泄漏的成本。



## 调用时机

MLeaksFinder 调用时机，是在一个控制器 pop 或者 dismiss 的时候，在 `UIViewController+MemoryLeak` 的文件里，可以清楚的看到:

```
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        [self swizzleSEL:@selector(viewDidDisappear:) withSEL:@selector(swizzled_viewDidDisappear:)];
        [self swizzleSEL:@selector(viewWillAppear:) withSEL:@selector(swizzled_viewWillAppear:)];
        [self swizzleSEL:@selector(dismissViewControllerAnimated:completion:) withSEL:@selector(swizzled_dismissViewControllerAnimated:completion:)];
    });
}
```

- swizzled_viewWillAppear ：在 viewWillAppear 里 set 一个 BOOL 类型的 kHasBeenPoppedKey 的关联对象。初始值为 @(NO).
- swizzled_viewDidDisappear: 在 viewDidDisappear 里 get 到 kHasBeenPoppedKey 关联对象，如果值为 YES ，则调用 `willDealloc`.
- swizzled_dismissViewControllerAnimated:completion:

## 检测机制

它关键的检测机制，就是通过下面的代码，进行了一个检查:

```objc
- (BOOL)willDealloc {
   ...
  
    __weak id weakSelf = self;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        __strong id strongSelf = weakSelf;
        [strongSelf assertNotDealloc];
    });
    
    return YES;
}

```

通过 dispatch_after 延迟 2 秒，调用一个 `assertNotDealloc` 方法：

```objc
- (void)assertNotDealloc {
    if ([MLeakedObjectProxy isAnyObjectLeakedAtPtrs:[self parentPtrs]]) {
        return;
    }
    [MLeakedObjectProxy addLeakedObject:self];
    
    NSString *className = NSStringFromClass([self class]);
    NSLog(@"Possibly Memory Leak.\nIn case that %@ should not be dealloced, override -willDealloc in %@ by returning NO.\nView-ViewController stack: %@", className, className, [self viewStack]);
}
```

第一先调用 `isAnyObjectLeakedAtPrts` ，传入 [self parentPtrs], 如果返回为 YES ,则直接 retrun.

这里的 parentPtrs 存储类型为 `uintptr_t` 的指针，指向对象。顾名思义，存储的是父类对象

```objc
+ (BOOL)isAnyObjectLeakedAtPtrs:(NSSet *)ptrs {
    NSAssert([NSThread isMainThread], @"Must be in main thread.");
    
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        leakedObjectPtrs = [[NSMutableSet alloc] init];
    });
    
    if (!ptrs.count) {
        return NO;
    }
    if ([leakedObjectPtrs intersectsSet:ptrs]) {
        return YES;
    } else {
        return NO;
    }
}
```

为 YES 的情况，就是 leakedObjectPtrs 和传入的 ptrs 有交集的情况。

可以发现 leakedObjectPtrs 是一个静态的全局对象:
```
static NSMutableSet *leakedObjectPtrs;
```

第二调用 ` [MLeakedObjectProxy addLeakedObject:self]`,在这里面做弹框逻辑 ：

```objc
+ (void)addLeakedObject:(id)object {
    NSAssert([NSThread isMainThread], @"Must be in main thread.");
    
    MLeakedObjectProxy *proxy = [[MLeakedObjectProxy alloc] init];
    proxy.object = object;
    proxy.objectPtr = @((uintptr_t)object);
    proxy.viewStack = [object viewStack];
    static const void *const kLeakedObjectProxyKey = &kLeakedObjectProxyKey;
    objc_setAssociatedObject(object, kLeakedObjectProxyKey, proxy, OBJC_ASSOCIATION_RETAIN);
    
    [leakedObjectPtrs addObject:proxy.objectPtr];
    
#if _INTERNAL_MLF_RC_ENABLED
    [MLeaksMessenger alertWithTitle:@"Memory Leak"
                            message:[NSString stringWithFormat:@"%@", proxy.viewStack]
                           delegate:proxy
              additionalButtonTitle:@"Retain Cycle"];
#else
    [MLeaksMessenger alertWithTitle:@"Memory Leak"
                            message:[NSString stringWithFormat:@"%@", proxy.viewStack]];
#endif
}
```

在这里获取视图调用栈的信息，以及对象的地址。并存入  leakedObjectPtrs .

## 遍历相关对象

解决类似 VC 释放，VC 对象上的属性没释放问题

## 堆栈信息

## 扩展其它类型的对象

# 其它情况的内存泄露处理

## 使用 instrument


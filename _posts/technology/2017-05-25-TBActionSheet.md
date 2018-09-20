---
layout: post
date: 2017-05-23 17:25:00
title: TBActionSheet 源码阅读
category: 技术
tags: 
- 源码阅读
keywords: iOS
description: 为了自制一个弹出框，对2个自己平常比较常用的弹框库的源码进行学习
---

[TBActionSheet](https://github.com/yulingtianxia/TBActionSheet)  是杨萧玉做的一个库，已经在腾讯的厂子里用了。

TBActionSheet 对横竖屏状态支持的很好，同时还有一个很不错的点，就是可以在显示后动态更新 UI。

几乎就是我心里对弹框库的期待了。

然而因为支持的功能太多，里面的 api 太多了，而且不能自定义弹出和隐藏动画（给自己一个造轮子的理由..）。

以下是 TBActionSheet 的结构：

![TBActionSheet](http://7ni3rk.com1.z0.glb.clouddn.com/TBActionSheet/overview.jpg)

这次阅读分析，主要针对 `TBActionSheetController` 和 `TBActionSheet` 两个类.

# TBActionSheetController

首先添加 TBActionSheet ，然后对 横竖屏 ／ 状态栏 ／ actionSheet 做的一些设置：

```
//返回YES可以自动旋转 
- (BOOL)shouldAutorotate
{
    return YES;
}
//返回支持的方向
- (UIInterfaceOrientationMask)supportedInterfaceOrientations
{
    return self.actionSheet.supportedInterfaceOrientations;
}

//设置 StatusBarStyle
- (UIStatusBarStyle)preferredStatusBarStyle
{
    UIWindow *window = self.actionSheet.previousKeyWindow;
    if (!window) {
        window = [[UIApplication sharedApplication].windows firstObject];
    }
    return [[window tb_viewControllerForStatusBarStyle] preferredStatusBarStyle];
}

//是否隐藏 statusBar
- (BOOL)prefersStatusBarHidden
{
    UIWindow *window = self.actionSheet.previousKeyWindow;
    if (!window) {
        window = [[UIApplication sharedApplication].windows firstObject];
    }
    return [[window tb_viewControllerForStatusBarHidden] prefersStatusBarHidden];
}

//旋转完成之后的回调方法
- (void)didRotateFromInterfaceOrientation:(UIInterfaceOrientation)fromInterfaceOrientation
{
    if (self.actionSheet.blurEffectEnabled && !kiOS8Later) {
        [self.actionSheet setupStyle];
    }
}

```

这里的巧妙之处，在于使用了控制器处理了旋转问题。

# TBActionSheet

在 TBActionSheet 里面，使用到了 +(void)initialize 的类方法 ,并且使用 appearance 来做一些初始化设置。

```
+ (void)initialize
{
    if (self != [TBActionSheet class]) {
        return;
    }
    TBActionSheet *appearance = [self appearance];
    appearance.buttonHeight = 56;
     ······
    appearance.supportedInterfaceOrientations = UIInterfaceOrientationMaskAll;
}

```

## initialize

initialize 在程序里的每个类都会调用一次，与 init 方法有所不同。

例如： 

```
 Duck *duck1=[Duck new];
 Duck *duck2=[Duck new];
 Duck *duck3=[Duck new];
```

只有在第一次实例化 duck1 的时候，调用一次 initialize 方法。之后的 duck2, duck3 都不会再调用。 

运行时间的行为之一就是initialize.

如果一个子类没有实现 initialize 类方法，那么超类会调用这个方法两次，一次为自己，而一次为子类。


## UIAppearance

我们肯定使用过如下类似的代码：

```
[[UINavigationBar appearance] setTintColor:myColor]
```

以此全局设置某个控件的属性的代码

不过在自定义的时候， 对 appearance 的使用还


UIAppearance 使得视图和控件的外观在整个 App 范围内统一定义。

UIAppearance是一个协议：

```
+ (instancetype)appearance;
+ (instancetype)appearanceWhenContainedInInstancesOfClasses:(NSArray<Class <UIAppearanceContainer>> *)containerTypes NS_AVAILABLE_IOS(9_0);
```

如 UIView 实现了 UIAppearance 这两种协议，既可以获取外观代理，也可以作为外观容器。

而 UIViewController 则是仅实现了 UIAppearanceContainer 协议，很简单，它本身是控制器而不是 view，作为容器，为 UIView 等服务。

要对让相应的属性支持 appearance 设置的话，要在后面使用 `UI_APPEARANCE_SELECTOR` :

```
@property (nonatomic, strong) UIColor *headerColor UI_APPEARANCE_SELECTOR;
```

注意：appearance 生效是在被添加到视图时，所以，在此之后设置 appearance，则不会起作用。而在手动设置属性之后被添加到视图树上，手动设置的会被覆盖。

## 可变参数

一个可变参数函数是指一个函数拥有不定的参数，即为一个函数可接收多个参数。

最经典的就是 NSLog（ C 为 printf ） ，它可以指定格式的输出，格式化输出的内容。

看到初始化方法里，有这么一段：

```
- (instancetype)initWithTitle:(NSString *)title 
                     delegate:(id<TBActionSheetDelegate>)delegate 
            cancelButtonTitle:(NSString *)cancelButtonTitle 
       destructiveButtonTitle:(NSString *)destructiveButtonTitle 
            otherButtonTitles:(NSString *)otherButtonTitles, ...
{
    va_list argList;
    // 从 otherButtonTitles 开始遍历参数，不包括 otherButtonTitles 本身.
    va_start(argList, otherButtonTitles);
    self = [self initWithTitle:title
                       message:nil
                      delegate:delegate
             cancelButtonTitle:cancelButtonTitle
        destructiveButtonTitle:destructiveButtonTitle
         firstOtherButtonTitle:otherButtonTitles
                     titleList:argList];
    va_end(argList);
    return self;
}
```
首先看到 `otherButtonTitles:(NSString *)otherButtonTitles, ...` ,这里就传递了一个可变参数。

### va_list 

定义一个指向个数可变的参数列表指针.

说明一个变量ap(argument pointer -- 可变参数指针），此变量将依次引用可变参数列表中用省略号“...”代替的每一个参数。即指向将要操作的变参。 

`va_list` 的定义：

```
#ifndef _VA_LIST_T
#define _VA_LIST_T
typedef __darwin_va_list va_list;
#endif /* _VA_LIST_T */
```

`va_list` 实际上是 `__darwin_va_list` 的别名。

`__darwin_va_list` 的定义如下：

```
#if (__GNUC__ > 2)
typedef __builtin_va_list	__darwin_va_list;	/* va_list */
#else
typedef void *			__darwin_va_list;	/* va_list */
#endif
```

这里看到，针对 `__GUNUC__` 的判断有不同的定义。

 `__builtin_va_list` 等带有 `__builtin` 都是编译器所能识别的函数，无法看到具体处理了。

### va_start(args,firstArg)

初始化变量刚定义的 va_list 变量，这个宏的第二个参数是第一个可变参数的前一个参数，是一个固定的参数.

### va_end(args)

清空参数列表，并置参数指针 args 无效

### va_arg(ap, type)

该表达式具有变长参数列表中下一个参数的值和类型。每次调用 va_arg 都会修改，用 va_list 声明的对象从而使该对象指向参数列表中的下一个参数。

接收 argList 的函数处理代码如下：

```
····
NSString* eachArg;
while ((eachArg = va_arg(argList, NSString*))) {
      // 从 args 中遍历出参数，NSString* 指明类型
      [self addButtonWithTitle:eachArg style:TBActionButtonStyleDefault];
}
····      
```

在处理 argList 的时候，使用 `va_arg(ap, type)` 返回可变的参数。


# 总结

TBActionSheet 使用了 frame 做布局，里面有大量计算的代码。就没有再仔细去看计算的细节了。作者考虑比较周全，除了没有自定义动画，可以说是非常好了。

initialize / UIApperance / 可变参数 的使用，都是我以前没有接触到的。


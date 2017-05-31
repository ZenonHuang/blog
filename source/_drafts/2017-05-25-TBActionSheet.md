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

这次阅读，主要针对 TBActionSheet.

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

# TBActionSheet

在 TBActionSheet 里面，看到了非常多的代码。

其中使用到了 +(void)initialize 的类方法 ,并且使用 appearance 来做一些初始化设置。

```
+ (void)initialize
{
    if (self != [TBActionSheet class]) {
        return;
    }
    TBActionSheet *appearance = [self appearance];
    appearance.buttonHeight = 56;
    appearance.offsetY = - bigFragment;
    appearance.tintColor = [UIColor blackColor];
    appearance.destructiveButtonColor = [UIColor redColor];
    appearance.cancelButtonColor = [UIColor blackColor];
    appearance.sheetWidth = MIN(kScreenWidth, kScreenHeight) - 20;
    appearance.backgroundTransparentEnabled = YES;
    appearance.backgroundTouchClosureEnabled = YES;
    appearance.blurEffectEnabled = YES;
    appearance.rectCornerRadius = 10;
    appearance.ambientColor = [UIColor colorWithWhite:1 alpha:0.65];
    appearance.separatorColor = [UIColor clearColor];
    appearance.animationDuration = 0.2;
    appearance.animationDampingRatio = 1;
    appearance.animationVelocity = 1;
    appearance.supportedInterfaceOrientations = UIInterfaceOrientationMaskAll;
}

```

### initialize

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

### UIAppearance



## TBActionBackground 

## TBActionContainer


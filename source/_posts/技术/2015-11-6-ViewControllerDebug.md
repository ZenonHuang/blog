---
layout: post
title: iOS开发--自定义Debug文件
date: 2015-11-06
category: 技术
tags: 实习工作
keywords: iOS
description: iOS日常工具
---

今天介绍一段代码，用于在执行程序的时候，显示控制器的名称。
在我们debug的时候比较有用，尤其是项目文件比较多的时候。可以减少我们的功夫，直接的看出问题出在哪一个controller里。

直接在对应的工程下，新建一个xxxDebug.m文件。

文件代码如下：

    #import <UIKit/UIKit.h>
    #import <objc/runtime.h>

    /**
    *  这个类只做调试用
    */
    @implementation UIViewController (DFDebug)

    - (void)zz_viewDidAppear:(BOOL)animated
    {
    [self zz_viewDidAppear:animated];
    NSLog(@"Debug by zzgo ==> %@", NSStringFromClass([self class]));
    }

    + (void)load
    {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
    Class class = [self class];

    SEL originalSelector = @selector(viewDidAppear:);
    SEL swizzledSelector = @selector(zz_viewDidAppear:);

    Method originalMethod = class_getInstanceMethod(class, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

    method_exchangeImplementations(originalMethod, swizzledMethod);
    });
    }

    @end



 



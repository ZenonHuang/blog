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

然而因为支持的功能太多，里面的 api 太多了，而且貌似不能自定义弹出和隐藏动画（给自己一个造轮子的理由..）。

在我初学的时候，也经常阅读作者的博客学习。

以下是 TBActionSheet 的大概结构：

![TBActionSheet](http://7ni3rk.com1.z0.glb.clouddn.com/TBActionSheet/overview.jpg)

这次阅读，主要针对 TBActionSheet.

# TBActionSheetController

# TBActionSheet


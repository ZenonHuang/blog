---
layout: post
date: 2017-12-17 12:25:00
title: 关于 iOS 中的库
category: 技术
keywords: iOS
description: 辨析动态库/静态库/Framework的概念
---

# 库是什么？

库一般是封装好的代码，便于复用。例如 GitHub 上可以看到的开源库，开箱即用。

下面我们主要说的，是编译成二进制的库。

## 什么场景使用二进制的库?

- 保护源代码。日常开发中的各种商业 SDK 就属于此类。只暴露头文件给使用者，而隐藏具体的实现。
- 加快编译速度。因为是编译完成的二进制，所以编译的时候只需要 Link 一下。

# 动态库与静态库

根据 Link 的方式，分成了2种库，静态库和动态库。

## 什么是静态库？

静态库，即静态链接库。不同平台的文件格式如下:


| 系统 | 静态库文件格式 | 
| --- | --- | 
| Windows | .lib |
| Linux | .a  |
| MacOS／iOS | .a  |

静态库的特点，在于编译的时候会直接复制到目标程序里。

意味着代码在目标程序中，不会再改变了。如图:

![3D044AD0-C6D0-40E4-ABD7-AB07945E6E26](https://user-gold-cdn.xitu.io/2017/12/15/16058b32b378fdd2?w=516&h=262&f=png&s=18920)


## 什么是动态库？

动态库，即动态链接库。
不同平台的文件格式如下:

| 系统 | 动态库文件格式 | 
| --- | --- | 
| Windows | .dll |
| Linux | .so  |
| MacOS/iOS | .dylib  |

动态库的特点，在于编译的时候，并不会被拷贝到目标程序中。

目标程序中，只会存储指向动态库的引用。程序运行时，动态库才真正被加载。

而同一个动态库，可以被多个程序使用。如图:

![91EDA5EA-2038-4B89-9ECB-BAA8DD4D94FE](https://user-gold-cdn.xitu.io/2017/12/15/16058b32ae0891fd?w=508&h=289&f=png&s=16877)


## MacOS/iOS 里的 Framework 

Framework 实际上是一个 bundle 或者说特殊形式的文件夹，将库的 **二进制文件/头文件/资源文**件 打包到一起。

Framework 可以选择创建为动态或者静态。

### 区别

由于 **.a /.dylib** 等只能打包库的二进制代码，可以用以下公式区别：

> 动态库/静态库 + .h + 资源文件 = Framework.


### 如何制作 Framework？

制作 Framework 已经有很多教程，可以直接谷歌百度查阅，这里就不纠结具体过程了。可以参考这篇文章->[静态库和动态库的制作(OC、Swift)](http://www.jianshu.com/p/f14553494d88)

### 关于动态的 Framework

真正动态的 Framework ，只有苹果自家的 UIKit.Framework，Foundation.Framework 等。

由于 iOS 的沙盒机制，自己创建的 Framework 和系统 Framework 不同，App 中使用的 Framework 运行在沙盒里，而不是系统中。每个 App 都只能用自己对应签名的动态库，做不到多个 App 使用一个动态库。

如果不同的 App 使用了同样的 Framework，还是会有多份的 Framework 被分别签名，打包和加载。

所以，iOS 上的动态库只能是私有的，我们无法将动态库放置在除了自身沙盒以外的地方。


# CocoaPods 管理库使用的是哪种形式？

CocoaPods 是 iOS 项目的依赖管理工具，类似于 Android 的 gradle。不过gradle 还能负责构建, CocoaPods 只管理依赖。

CocoaPods 目前分别支持了 `Framework 动态库`和 `.a 静态库` 2种方式。

> 默认使用的是 .a 静态库的方式，主工程依赖 libPods.a，通过脚本，把资源等文件复制到目标目录。

> 在 Podfile 使用了 `use_frameworks!` 进行声明,则使用 Framework 动态库的形式.主工程对 Pods 的依赖为 Pods.framework。

 目前 Swift 项目不支持静态库，如果为 Swift 项目，就必须采用 `user_frameworks!` 声明。


# 参考

>[dylib浅析](https://makezl.github.io/2016/06/27/dylib/)

>[iOS 静态库，动态库与 Framework](https://segmentfault.com/a/1190000004920754)

>[iOS 静态库和动态库的基本介绍和使用](http://ios.jobbole.com/89871/) 

>[CocoaPods 原理总结](http://www.cloudchou.com/ios/post-990.html)

>[如何打造一个让人愉快的框架](https://onevcat.com/2016/01/create-framework/)





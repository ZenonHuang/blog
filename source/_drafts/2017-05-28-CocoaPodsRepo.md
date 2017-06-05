---
layout: post
date: 2017-06-01 10:25:00
title: Potatso源码学习
category: 技术
keywords: iOS
description: 公司做科学上网业务，学习 Potatso 开源项目
---

# 前言

[Potatso](https://github.com/Potatso/Potatso) 是一个非常好的 iOS 科学上网 app 。此前曾在 GitHub 开源，后来因为很多人不经同意便直接复制项目上架，作者关闭了该项目。

当时是 Swift2 的时代，现在已经要走到 Swift4 了。 但网上相关的资料又比较缺乏，出于快速学习的需要，还是重新找到一份自己当时fork Potatso 的 [源码](https://github.com/ZenonHuang/Potatso) 进行学习。 

# Podfile

## 多依赖管理

首先打开它的 Podfile :

![podfile](http://7xiym9.com1.z0.glb.clouddn.com/potatso_podfile.png)

看到它使用了 def ，利用 Ruby 的函数，定义出了两个 target 可以共享的依赖，以及只能在 App 中使用的依赖，然后通过函数调用去给 target 设置依赖，这样既解决了重复问题，又能精确控制每个 target 的依赖。

## 环境区分

还注意到了以下的设置：

```
pod 'Reveal-iOS-SDK', '~> 1.6.2', :configurations => ['Debug']
```

利用  `:configurations => ['Debug']` ,来指定 pod 在 Debug 环境下生效。

# Cartfile

cartfile 只有一行代码 :

```
github "mirek/YAML.framework" "master"
```

表示使用 GitHub 上的 YAML.framework 项目 ，我这里不明白的是，为什么不直接一起使用 CocoaPods 进行管理？

# AppDelegate

打开它的 AppDelegate.swift 文件，一片空白：

![appdelegat](http://7xiym9.com1.z0.glb.clouddn.com/potatso_appdelegate.png)

但是仍然捕捉到一行代码 ：

```
import ICMMainFramework
```

## ICMMainFramework

`ICMMainFramework` 是什么框架？查找发现 Podfile 中引用了 ICMMainFramework ，是一个本地的Pod：

```
def library
    ...
    pod 'ICSMainFramework', :path => "./Library/ICSMainFramework/"
    ...
end
```

这里我们可以看到它的路径是 `./Library/ICSMainFramework/`,顺藤摸瓜找到它，看看里面都有些什么。

### open 关键字

### ？？比较

### 




# 参考

[使用 CocoaPods 管理多个 Target 的依赖](https://skyline75489.github.io/post/2015-11-26_cocoapods_multiple_target.html)

[Debug 与 Release 分别引入不同的 Pod](https://zorro.im/debug-yu-release-fen-bie-yin-ru-bu-tong-de-pod/)




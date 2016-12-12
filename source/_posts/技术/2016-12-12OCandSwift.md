---
layout: post
date: 2016-12-12 21:50:46
title: Objective-C与Swift混编
category: 技术
tags: 
- 资料
- swift
- 混编
keywords: swift
description: Objective-C项目中使用Swift
---

# 前言
看swift的语法也有一小段时间了。但是一直干看语法，显然是不可能熟练使用的。趁着项目有空，开始尝试开辟新分支，对Objective-C项目进行swift混编。

# Objective-C项目创建Swift文件

选择创建一个Swift类，Xcode会提示：
>Would you like to configure an Objective-C bridging header?

选择“Create Bridging Header”，Xcode就会自动建好一个中间的桥接文件，一般是"项目名-Bridging-Header.h"的形式。

桥接文件中可以导入OC类的头文件，这样你就可以在Swift文件里面去使用OC类了。

完成好这些，你可以在OC项目里使用Swift文件了。

## 遇到的坑

### CocoaPods集成

#### 要使用　use_frameworks!　在Podfile里进行声明

#### 集成了YYKit
YYKit里有使用到很多类似下面的代码:

````
#if __has_include(<YYCache/YYCache.h>)
FOUNDATION_EXPORT double YYCacheVersionNumber;
FOUNDATION_EXPORT const unsigned char YYCacheVersionString[];
#import <YYCache/YYMemoryCache.h>
#import <YYCache/YYDiskCache.h>
#import <YYCache/YYKVStorage.h>
#elif __has_include(<YYWebImage/YYCache.h>)
#import <YYWebImage/YYMemoryCache.h>
#import <YYWebImage/YYDiskCache.h>
#import <YYWebImage/YYKVStorage.h>
#else
#import "YYMemoryCache.h"
#import "YYDiskCache.h"
#import "YYKVStorage.h"
#endif
````

其中`#if...#endif`是属于C++的条件编译预处理命令，`__has_include`是用于检查是否引入了某个类的宏。

而在Swift里，这是无法识别使用的。一直报错。

于是都注释掉，只留直接引入类的形式:

```
 #import "yourClass.h"
```

# 在Objective-C项目里，使用Swift

Xcode会自动为Project生成头文件，以便Swift在Objective-C中调用。

在OC文件中如果想要调用Swift类的方法,只需要导入一个头文件。该头文件在工程中不可见，默认格式为xxx-Swift.h，其中xxx为项目名。

只需要#import "项目名-Swift.h"，即可在Objective-C文件里调用Swift。


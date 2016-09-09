---
layout: post
date: 2016-08-21 16:50:46
title: iOS代码Objective-C编程规范
category: 技术
tags: 资料
keywords: iOS
description: 根据自己经验总结的编程规范
---

# iOS代码Objective-C编程规范

## 前言

在经过一段时间学习和工作之后，自己写了不少代码，也看了不少代码。深知千人千面，每个人写的代码都有自己的风格。

而坏的代码各有各的"特色"，好的代码总是清晰明了（当然有一些代码由于高度的抽象，导致不那么"清楚"，需要我们深入去理解它们）。

在一个团队中，如果没有一个文档规范，特别是当新人进入时，每次都进行口头的说教，将带来不必要的时间成本。

因此有了这一份iOS代码的Objective-C编程规范。

## 项目目录结构

关于目录结构和架构等组织方式，现在总结出一个道理：
>架构是逐渐演进的，适合自己的业务的架构才是最好的架构，没有所谓最好的架构

之前自己忙于工作，没记录下来思考过程有些遗憾，这一篇[《iOS移动端架构的那些事》](http://www.jianshu.com/p/15e5b83ab70e)说的非常好。文中最后的middleman以及urlRoute，我已经有一些涉猎和实践，但是在这里不做讨论。

在一般的开发当中，我们使用按`模块`划分的方法，已经足够用了。
目前使用的目录结构如下：

```

|---AppDelegate
  |---AppDelegate+category1
  |---AppDelegate+category2
  ...
|---Classes
  |---Module
    |---Controller
    |---Model
    |---View
  ...
|---Expand
  |---Category
    |---UIKit
    |---Foundation
  |---Const
    |---HZConst
    |---APIConst
  |---Macros
  |---NetWork
  |---Utility
  ...
|---Resource
  |---Image
  |---Plist
  |---File
  |---Video
  |---Sound
  ...
|---Vendor

```

乍一看很多，容我慢慢讲解。

#### AppDelegate
App的Delegate类实现，它必须有。但是有时候会在里面做很多操作，比如：监听前后台切换，或者第三方登录，统计分析，通知注册回调等等的操作。因而将代码进行拆分，对AppDelegate按功能分类。
#### Classes
这里是我们写业务的地方。

首先横向拆分业务、功能模块。

然后模块里面纵向拆分，一般采用MVC/MVVM/MVP等方式都可以。

>注意，这个地方曾经有同事倾向于按界面去分模块。这样有两个不好的地方:第一，比如电商APP，在首页/收藏/商品买卖等等各个地方，都需要一个商品详情模块，那这个模块界面层级，该怎么放？按业务拆分时，就不应该再用界面去划分模块。第二，代码组织的层级很容易过深，如果界面复和功能复杂起来，那展开一个文件可能需要打开很多层文件夹。

#### Expand
顾名思义，这里放的都是我们的扩展文件,可以存放Category(分类文件)，Const(常用常量，api常量)，Macros(宏)，Network(网络库的封装扩展)，Utility(工具箱)等等。

#### Resource
在Resource里面，存放我们的各种资源文件,plist,image,video等等。

#### Vendor
这里面统一存放的是我们引入的第三方库和框架。

虽然可以用CocoaPods和Carthage很方便地管理第三方库，但偶尔也会需要直接引入第三方的文件。

## 文件内代码组织

考虑到知易行难这样亘古不变的道理，我已经整理好代码模板文件到[我的Github](https://github.com/zzgo/CodeTemplate)上,直接clone下来后，放在如下路径:

>/Applications/Xcode.app/Contents/Developer/Library/Xcode/Templates/File Templates/Source/

### 文件头注释

所有的文件在文件头注释上，应当具有:

* 文件创建的时间
* 文件创建者
* 文件创建者的联系方式

如下所示：

```
//
//  Created by zzgo on 16/8/21.
//  Contact by zzgoCC@gmail.com
//  Copyright © 2016年 quseit. All rights reserved.
//
```

意义在于让后来者知道代码创建的时间，以及碰到问题时，可以及时联系上原作者。

### @interface
在接口中，先写属性（delegate在最后），再写方法(类方法在前)。
如:

    @interface Object : NSObject
    @property (nonatomic, readwrite, strong) NSString *name;
    @property (nonatomic, readwrite, weak) id<ObjectDelegate> delegate;
    + (void)testLog;
    - (void)configure;
    @end

#### 接口设计原则
避免 “厨房水槽（kitchen-sink）” 式的 API。如果一个函数压根没必要公开，就不要这么做。提一下接口的设计原则:

- 只暴露外部需要看到的。尽可能少的`暴露`类的方法，属性，和成员变量等。
- 合理分组
- 避免头文件污染
- 避免接口过度设计
- 隐藏继承关系中的私有接口
- 避免单例的滥用

有兴趣的可以看一看这个文章<[objc@interface的设计哲学与设计技巧](http://blog.sunnyxx.com/2014/04/13/objc_dig_interface/)>

另外，对于一个外部引用对象，且外部不会发生set操作的对象，使用readonly属性:

```
 //.h文件中
@interface MyClass : NSObject
@property (nonatomic, readonly, strong) NSObject *object;
@end
//.m文件中
@interface MyClass ()
@property (nonatomic, readwrite, strong) NSObject *object;
@end

@implementation MyClass
//Do Something cool
@end
```

### .m文件的布局顺序

布局顺序如下：

```
    //文件头注释
    //import
    //文件内部使用的宏
    //常量定义
    //@interface()
    //@implementation
```

#### @implementation中的组织

```
#pragma mark - Intial Methods

#pragma mark - Override/Life Cycle

#pragma mark - Notification

#pragma mark - KVO

#pragma mark - target-action/IBActions

#pragma mark - delegate dataSource protocol

#pragma mark - public

#pragma mark - private

#pragma mark - getter / setter
```

#### 控制流语句的括号使用
在使用控制流语句时，必须使用括号.
推荐:

```
if (someObject) {
  [someObject doSomething];
} 
```

不推荐:

```
if (someObject) [someObject doSomething];
```

括号让我们的控制流语句的执行体看起来更明了。

### 其他具体风格
关于更细节的代码风格，统一使用Xcode的插件 [clang-format](https://github.com/travisjeffery/ClangFormat-Xcode) ，来格式化代码风格。

>clang-format是基于clang的一个命令行工具。这个工具能够自动化格式C/C++/Obj-C代码，支持多种代码风格：Google, Chromium, LLVM, Mozilla, WebKit，也支持自定义风格（通过编写.clang-format文件）。

在工程目录或者workspace目录下创建一个".clang-format"文件，就可以使用自定义的格式。

在这里我们采用统一的`.clang-format`文件.同样可以前往[我的Github](https://github.com/zzgo/CodeTemplate)上下载。


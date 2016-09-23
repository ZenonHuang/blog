---
layout: post
date: 2016-09-10 23:50:46
title: 利用TravisCI自动部署Hexo（待完成）
category: 技术
tags: 持续集成
keywords: iOS
description: 持续集成踩坑
---

# 我是前言

今天这篇文章比较大，无论是Hexo或者是Travis CI都可以单独拿一个文章说。

不过这类文章已经够多了，跟着做下来也很快。

于是尝试来一点有意思的玩法。下面将先介绍持续集成，再介绍一下hexo。

重点在于，学会利用自动构建来解放我们的时间，而且又能保证程序的正确，何乐而不为？

# 持续集成介绍

第一次接触这么高逼格的名词，必须好好研究一番。到底什么是持续集成呢？

`持续集成`又叫`Continuous integration`，简称`CI`。

它的核心在于，代码集成到主干之前，必须通过自动化测试。只要有一个测试用例失败，就不能集成。

这让我们可以快速迭代，同时还能保持高质量。


# TravisCI简介

今天要介绍的使用[Travis CI](https://travis-ci.org/)来进行持续集成（如果不想看我这段废话，可以直接跳到下一段）。

当然，目前还有其它好用的CI工具，例如Jenkins等。

`Travis CI ` 是一个开源的持续集成项目，高度集成 `GitHub` ,为托管于 GitHub 的开源项目提供`免费`的持续集成测试服务。

使用TravisCI的开源项目，一般都有下面这样的标志：

![C723F377-BD91-4594-B886-8608B7E2AA8D](http://7xiym9.com1.z0.glb.clouddn.com/C723F377-BD91-4594-B886-8608B7E2AA8D.png)

一个开源库是否能正常编译通过，看到这一个标志就知道靠谱！至少编译环节没有什么问题的。

Github上已经有大部分的项目已经采用了它。

还记得我第一次提PR，到饿了么的仓库上一交代码，"刷刷刷"的就开始`自动构建`了，你还可以看到对应的构建记录。瞬间觉得高大上。

我自己也开了一个项目[LeanCloudChatDemo](https://github.com/zzgo/LeanCloudChatKit)玩玩

## Travis CI简单使用

使用Travis CI非常的简单，你只需要一个github账号。


### 关联账号

首先进入Travis CI 的[官网](https://travis-ci.org/)，使用你的github账号来登陆。

![travis-index](http://7xiym9.com1.z0.glb.clouddn.com/890FC21E-6DB6-4667-A4A6-EFDEC9D8DAB4.png)

如图，点击右上角的绿色按钮“Sign with GitHub”.

>PS:如果是私有项目，可以进这个[Travis CI私有项目](http://travis-ci.com/)

### 关联仓库

登录之后，网站会同步你的仓库信息，可能需要稍等一会。

如下图，看到你的仓库信息:

![C280E33E-426E-4F30-A0D6-3EF63AD42531-w562](http://7xiym9.com1.z0.glb.clouddn.com/C280E33E-426E-4F30-A0D6-3EF63AD42531.png)

接下来，点击按钮，`勾选`你想要使用Travis CI来进行构建的仓库。

![82F8D9EC-6258-41FB-85CB-E5F45AF0B5CD](http://7xiym9.com1.z0.glb.clouddn.com/82F8D9EC-6258-41FB-85CB-E5F45AF0B5CD.png)


### 添加.travis.yml文件

每个使用Travis CI的项目，都要配置对应的`.travis.yml `文件.

进入项目的根目录下，使用touch命令，新建文件：

```
touch .travis.yml
```

然后使用vim或者其他文本编辑器，编辑文件内容。

这里有官方提供的，对应各种语言的示范:

>[Travis CI 配置示例](https://docs.travis-ci.com/user/languages/)

怕还有的同学不懂，我这里提供Android和iOS的简单配置示例.

#### 安卓.travis.yml配置

```
language: android   # 声明构建语言环境

notifications:      # 每次构建的时候是否通知，如果不想收到通知邮箱（个人感觉邮件贼烦），那就设置false吧
  email: false

sudo: false         # 开启基于容器的Travis CI任务，让编译效率更高。

android:            # 配置信息
  components:
    - tools
    - build-tools-23.0.2              
    - android-23                     
    - extra-android-m2repository     # Android Support Repository
    - extra-android-support          # Support Library

before_install:     
 - chmod +x gradlew  # 改变gradlew的访问权限

script:              # 执行:下面的命令
  - ./gradlew assembleRelease  

before_deploy:       # 部署之前
  # 使用 mv 命令进行修改apk文件的名字
  - mv app/build/outputs/apk/app-release.apk app/build/outputs/apk/buff.apk  

deploy:              # 部署
  provider: releases # 部署到GitHub Release，除此之外，Travis CI还支持发布到fir.im、AWS、Google App Engine等
  api_key:           # 填写GitHub的token （Settings -> Personal access tokens -> Generate new token）
    secure: 7f4dc45a19f742dce39cbe4d1e5852xxxxxxxxx 
  file: app/build/outputs/apk/buff.apk   # 部署文件路径
  skip_cleanup: true     # 设置为true以跳过清理,不然apk文件就会被清理
  on:     # 发布时机           
    tags: true       # tags设置为true表示只有在有tag的情况下才部署
```

#### iOS的.travis.yml配置

```
osx_image: xcode7     #指定osx下的xcode版本
language: objective-c  #指定语言
xcode_workspace: ChatDemo/ChatDemo.xcworkspace #项目文件相对于.travis.yml文件的路径（因为这里使用的是cocoaPods，所以是xxx.xcworkspace）
xcode_schemes: ChatDemo  #项目的schemes

profile: ChatDemo/Podfile #podfile路径
```

>PS:以上提供的两个版本的文件配置不可能情况都适用。真正要使用，还是去看官方的教程说明。调整对应的参数，来达到效果。

### 推送.travis.yml文件
完成.travis.yml文件的配置后，直接推送上去就可以了。

之后仓库的`commit`和`pull request`等操作都会被构建（这里构建的时机和条件可以自己设置）。

前面说了那么久的Travis CI，还没真正见识到它的威力。我相信大家初始一个工程，配置之后，基本都是能pass的。

只是这样看它pass还不够爽，下面介绍用Travis CI来使用Hexo自动构建博客。

不但可以免去你的每次`渲染`，`推送`的重复工作，更可以让你`随时随地`写博客，而不必在意是否具有Hexo的`环境`。

# Hexo介绍

Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

最棒的是，它具有多种多样的主题供我们选择，配置。非常简单，易用。

不过对Hexo安装部署，这里不做教程，推荐大家前往官网进行学习，文档已经很详细了:

>[Hexo官方文档](https://hexo.io/zh-cn/docs/#安装)

## submodule形式管理hexo主题

这里我假设大家已经完成了hexo的安装和使用，可以自己切换主题了。

由于

## 配置文件版本踩坑

test



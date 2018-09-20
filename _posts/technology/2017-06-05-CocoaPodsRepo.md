---
layout: post
date: 2017-06-01 10:25:00
title: 私有 CocoaPods 的仓库
category: 技术
keywords: iOS
description: 公司内部做组件化，采用 CocoaPods 作为管理依赖的工具
---

由于公司业务需要，开始打造内部的框架和封装组件，利用 CocoaPods 作为依赖管理工具

# 创建私有的 Spec Repo

在打造私有的组件之前，要首先建立一个私有的 Spec Repo.

> Spec Repo 是所有的 Pods 的一个索引，就是一个容器，所有公开的 Pods 都在这个里面.实际只是一个Git仓库r.它的作用就像字典的目录一样，提供 Pods 的地址，版本等等

你可以选择 GitLab 或者 GitHub, Coding 等任何提供托管的地方，去建立一个自己的 Spec Repo。

## 添加私有 Spec Repo 到本地

在建立好远程的 Repo 之后，把它添加到本地：

```
pod repo add RepoName SOURCE_URL
```

例如:

我要添加一个名为 zenon ，repo地址为 https://github.com/MediatorDemoRepo/PrivatePods

则命令为：

```
pod repo add zenon https://github.com/MediatorDemoRepo/PrivatePods
```

有了远程仓库之后，我们就可以提交组件到自己的 Spec Repo 里了。

## 移除 repo 

对于不想用的 Spec Repo,可以在本地进行移除：

```
pod repo remove [name]
```

例如，移除名为 zenon 的 Spec Repo:

```
pod repo remove zenon
```


## 提交组件到 Spec Repo

有了远程仓库之后，我们就可以提交组件到 Spec Repo 里 ：

```
pod repo push REPO_NAME SPEC_NAME.podspec
```

在此之前，要到使用 

```
pod lib lint 
```

验证 Spec_Name.podspec 文件后才可以推送到 Spec Repo里。

> 推送成功后，远程的 Spec Repo 目录应该如下:

```
├── Specs
    └── [SPEC_NAME]
        └── [VERSION]
            └── [SPEC_NAME].podspec
```



## 查看本地 Spec Repo 列表

```
pod repo list
```

# 组件工程

## 创建组件工程

CocoaPods 为我们提供了快捷的创建方式，可以直接从 GitHub 上拉一个完整的模版工程下来：

```
pod lib create YourProjectName
```

模版工程创建后，结构如下:
 
```
YourProjectName
├── Example                                  #demo APP
│   ├── YourProjectName
│   ├── YourProjectName.xcodeproj
│   ├── YourProjectName.xcworkspace
│   ├── Podfile                              #demo APP 的依赖描述文件
│   ├── Podfile.lock
│   ├── Pods                                #demo APP 的依赖文件
│   └── Tests
├── LICENSE                          #开源协议 默认MIT
├── Pod                              #组件目录
│   ├── Assets                           #组件资源文件
│   └── Classes                          #组件类文件
├── YourProjectName.podspec          #组件的podspec文件
└── README.md                        #markdown格式的README

```

在模版工程生成之后，我们所要做的，就是向其中的组件目录 Pod 下添加库文件和资源分别至 Pod/Classes 和 Pod/Assets 目录下。

在放入完成后，再进入到 Example 文件下，执行:

```
pod update
```

>每次更改组件目录 Pod 的文件时，都应该执行一次 ` pod update ` 来更新

## 推送组件工程

在运行工程，确定无误后，把它推送到远程仓库。

和 Spec Repo 一样，去GitHub或其他的Git服务提供商那里创建一个私有的仓库.

然后进入到模版本工程的根目录下，进行提交：

```
//添加记录
git add .

//进行提交，并说明
git commit -s -m "Initial Commit of Library"

//添加远程仓库地址
git remote add origin git@coding.net:username/YourProjectName.git    

//提交到远端仓库   
git push origin master     
```

## 添加版本号

podspec文件中获取 Git 版本控制的项目还需要 tag 号来对应版本，所以我们要打tag:

```
git tag -m "push tag" 0.1.0 #版本号根据实际情况填写，要跟podspec文件里的一致
git push --tags     #推送tag到远端
```


## 填写 podspec 文件

```
Pod::Spec.new do |s|
  s.name             = 'YourProjectName'
  s.version          = '0.1.0'         #版本号，和之前打的tag要一致
  s.summary          = '组件的简介.'

  s.description      = <<-DESC
      详细描述
                       DESC

  s.homepage         = 'YourProjectUrl'  #主页地址
  # s.screenshots     = 'www.example.com/screenshots_1', 'www.example.com/screenshots_2'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'nickname' => 'your email' }
  s.source           = { :git => 'your git repo', :tag => s.version.to_s }
  # s.social_media_url = 'https://twitter.com/<TWITTER_USERNAME>'

  s.ios.deployment_target = '8.0'  #支持的平台及版本

  s.source_files = 'YourProjectName/Classes/**/*' #代码的源文件地址
  
  # s.resource_bundles = {
  #   'SZPopupView' => ['SZPopupView/Assets/*.png']
  # }  #资源文件地址
  
  #s.public_header_files = 'Pod/Classes/*.h'  #公开头文件地址
  
  #s.frameworks = 'UIKit', 'MapKit' #所需的framework，多个用逗号隔开
  
  # s.dependency 'AFNetworking', '~> 2.3'  #依赖关系，该项目所依赖的其他库，如果有多个需要填写多个s.dependency
end

```

正确填写 podspec 文件十分重要，应该仔细核对 仓库地址，依赖，目录关系等。

## 测试验证

确认填写无语后，在 YourProjectName 的根目录下面，执行验证命令:

```
pod lib lint
```

如果有信息输出，则根据报错修改。

当看到:

```
YourProjectName passed validation.
```

就说明验证通过了


# 向私有 Spec Repo 提交

验证通过后，就可以向远程的 Sepc Repo 提交了。

```
//前面是本地Repo名字 后面是 podspec 名字
pod repo push Repo_Name YourProjectName.podspec
```

# 引用 Pod

## 使用私有 Spec Repo

对于私有的 Spec Repo， 在使用时，在对应的 Podfile 文件中，添加 source 地址：

```
source 'URL_TO_REPOSITORY'
```

例如:

```
source 'git@github.com:MediatorDemoRepo/PrivatePods.git'
```


# 删除和更新

## 删除 Pod  Repo

```
pod repo remove RepoName
```

## 更新 Pod  Repo

```
pod repo update RepoName
```



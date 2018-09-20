---
layout: post
title:  iOS工具--CocoaPods
date: 2015-11-10
category: 技术
tags: 实习工作
keywords: cocoapods
description: iOS第三方包管理工具
---
# 前言
学习ios的时间很短，从第一次开始，就接触了对于我来说，很庞大的项目。感谢严哥的信任。

它的界面美观，功能齐全，用户众多。
“我以后也要做一个这样的app”，我默默在心里许下愿望。

我在里面看到了很多的库，应该有几十个。库的使用，很大程度上减少了我们的工作量。

对于我这样的，连正式入行都还不算的学生来说，学会找到合适的库，利用库，是一项很不错的技能。

很有幸，经过一个大项目，我学会了自己从网上（更多是直接在github）寻找库。

# 使用CocoaPods的原因
目前我负责开发公司实验型的一个app，从基础界面，到主要功能，都要我自己去慢慢做的话，无疑要用到库。

而且越到后期，我估计使用的库也会越多。
CocoaPods的使用，可以大大节省我们的时间。
我们不用再去重复这样的过程：

1.把第三方库的源代码文件，复制到项目中。或设置成git的submodule

2.手工地，将这些开源库的依赖系统的framework，一个个地添加到项目依赖中。

3.对某些开源库，设置-licucore或者-fno-objc-arc登编译参数

4.管理这些依赖包的更新

# 安装CocoaPods
今天我自己做安装的时候，遇到了几点问题：

1.ruby的软件源问题

2.gem更新

为了避免这些问题，我建议大家直接先更新ruby的源，再升级一次gem.做完这些工作以后，我相信安装的问题应该不大了。

## ruby源替换
一，移除原来官方的ruby源：

	gem sources --remove https://rubygems.org/
 	 
 	 
二，替换成淘宝源：

	gem sources -a https://ruby.taobao.org/
	
这里要提醒大家的是，在网上有些教程或者书籍中，使用的是http方式，现在已经改成了https。

三，检查一下是否替换成功:

	gem sources -l
 		
 	
打完之后出现下面这些的话，那么就证明成功了：

	zz:~ zzgo$ gem sources -l
	*** CURRENT SOURCES ***

	https://ruby.taobao.org/
	
## gem升级
gem升级非常简单，在更换源之后，访问的限制也不存在，相信速度也非常快。只要输入如下代码：

	sudo gem update --system

最后出现“RubyGems system software updated”，即为成功。
## 使用ruby的gem命令下载安装
相信你已经迫不及待了，它的安装方式异常简单:

	sudo gem install cocoapods
  	
出现“xx gems installed”，就算成功了。之后：

	 pod setup
  	
只要按照顺序，输入两行代码就能ok。不过在pod setup之后，还需要耐心等待一会。出现“Setup completed”就可以结束。

### 报错
在我换了新的mac后，安装时，出现如下错误提醒：

```
ERROR:  Error installing cocoapods:
	activesupport requires Ruby version >= 2.2.2.
```
意思是ruby的版本不够。
使用homebrew升级：
```
brew install ruby
```
升级完毕后，重新执行cocoaPods安装命令。

# 在项目里使用CocoaPods

## Podfile
Podfile是使用CocoaPods的关键文件。

一，新建一个ios项目

二，在*.xcodeproj文件的同级目录，新建一个名为Podfile的文件

三，编辑Podfile文件，将依赖的库名字依次列在文件中：

	 platform:ios
	 pod 'ASIHTTPRequest', '~> 1.8.2'
 	 
我们可以在使用之前，通过pod search命令，检查一下CocoaPods管理的库中，是否有要使用的库:
	
	zz:CocoaPodsDemo zzgo$ pod search ASIHTTP


	-> ASIHTTPRequest (1.8.2)
	Easy to use CFNetwork wrapper for HTTP requests, Objective-C, Mac OS X and iPhone.
	pod 'ASIHTTPRequest', '~> 1.8.2'
	- Homepage: http://allseeing-i.com/ASIHTTPRequest
	- Source:   https://github.com/pokeb/asi-http-request.git
	- Versions: 1.8.2, 1.8.1 [master repo]
	- Subspecs:
    	  	- ASIHTTPRequest/Core (1.8.2)
     	 	- ASIHTTPRequest/ASIWebPageRequest (1.8.2)
     	  	- ASIHTTPRequest/CloudFiles (1.8.2)
     	  	- ASIHTTPRequest/S3 (1.8.2)


我在上面使用了 'pod search ASIHTTP'搜索。就显示了库的信息和版本。

之后就再我们的ios项目下，执行:

	  	pod install

出现类似：

	Analyzing dependencies
	Downloading dependencies
	Installing ASIHTTPRequest (1.8.2)
	Generating Pods project
	Integrating client project

	[!] From now on use `CocoaPodsDemo.xcworkspace`.


那证明我们的第三方库已经下载完成，并且设置好了编译参数和依赖。注意：我们今后将使用提示的'*.xcworkspace'文件来打开工程。而不是原来的' *.xcodeproj'文件。

打开项目，可以看到这样的结构：

 ![](http://7xiym9.com1.z0.glb.clouddn.com/prj.png)
 
 
现在就可以import一下，来使用我们的库了。

>本书参考：《ios开发进阶》电子工业出版社


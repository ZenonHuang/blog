---
layout: post
title: Xcode插件
date: 2015-11-15
category: 技术
tags: 实习工作
keywords: ios
description: ios日常工具
---
在日常开发中，使用合适的插件，可以提高开发的效率。让我们更集中精力在处理代码的逻辑上。

#Alcatraz

Alcatraz是用来管理Xcode插件，模板和颜色配置的工具。

我们很容易就可以安装它。

首先关闭Xcode.

在命令行里使用如下代码：

	curl -fsSL https://raw.github.com/supermarin/Alcatraz/master/Scripts/install.sh | sh
	
如果不想使用了，可以用下面的命令删除：

	rm -rf ~/Library/Application\ Support/Developer/Shared/Xcode/Plug-ins/Alcatraz.xcplugin
	
最后出现"xxxsuccessfu! "的字样，就是安装成功了。

重启Xcode.

在Xcode的顶部菜单栏，找到Window选项，可以看到“PackageManager”的选项。

单击它，就可以启动插件列表界面，搜索自己想要的插件下载。

#常用插件介绍

##KSImageNamed

开发时，经常不得不频繁查看资源文件夹以查找合适的图片的名称。使用KSImageNamed插件后，会自动弹出图片名称的列表以供选择，而且还有缩略图，十分便捷。

当你输入[UIImage imageNamed:]时，会自动弹出上下文菜单，供你选择你需要输入的图片资源名称。选择图片资源时，还能在左侧预览。

##XVim
这里要重点介绍一下XVim.

它的功能是可以在Xcode的编辑窗口中开启vim模式。

vim的最大好处，是可以全键盘操作，可以方便的移动光标，以及复制，粘贴代码。

想一想，当我们可以去掉鼠标，全心放在代码输入上的时候该多舒服。而不是再用鼠标滑动一下，按一下，再放到键盘上去。

每天我们这的重复操作耗去了很多时间。

当我们学会用vim进行全键盘操作时，该是多么酷。

同时也可以培养我们在最原始的情况下，进行程序编写的能力，让它成为自己短小锋利的编程武器。

##FuzzyAutocompletePlugin

它允许使用模糊的方式来进行代码的自动补全。

相信很多人和我一样，在输入tableView的方法时很讨厌输入那么多才把方法打出来。

使用它之后，我们只要依次输入方法的任意字母（不过一定要按单词出现顺序），即可匹配到对应的方法。

例如要重载viewDidAppear:,只要输入“veapp”这样的单词就可以了

##ColorSense

它和KSImageNamed一样，也是一个输入辅助工具。不过是关于UIColor颜色的，在编写UIColor时，可以实时预览响应的颜色。

##ClangFormat

它是一个自动调整代码风格的工具。

Xcode本身的代码缩进自动调整功能比较弱，特别是对于JSON格式。

它可以更好的排版，并且内置各种风格，也支持自定义风格。


##XcodeBoost

XcodeBoost包含多个辅助修改代码的小功能。

通过配置，我们可以使用光标,或者不精确的选择就可以剪切或者拷贝代码行，可以在粘贴代码的时候不触发代码格式化，还可以通过在.m文件中拷贝方法，粘贴进.h文件的时候就可以得到自动格式成的方法声明，还有好些功能都可以实现。

参考：

>《iOS进阶》唐巧

>[慕课网](http://www.imooc.com/article/2115)






---
layout: post
date: 2017-06-16 14:25:00
title: Swift 里的 throws
category: 技术
keywords: iOS
description: Swift 阅读源码时多次看到 throws 的使用，和之前的 OC 很不相同
---

首先得明晰两个概念，一个是异常（exception）,一个是错误（error）。

### 异常

异常一般来说，是由程序员控制的代码，导致程序无法运行。类似这样所导致的程序无法运行的问题，应该在开发阶段就被全部解决，而不应当出现在实际的产品中。

### 错误

相对来说，由 NSError 代表的错误更多地是指那些“合理的”，在用户使用 app 中可能遇到的情况。


---
layout: post
title: python urllib2使用（用于解决web跨域访问）
date: 2015-09-24
category: 技术
tags: 实习工作
keywords:
description: 在实习时用python做内部使用的小工具，碰到的问题
---



## urllib2模块的介绍

### 简介：

urllib2是python的一个获取url（Uniform Resource Locators，统一资源定址器）的模块。它用urlopen函数的形式提供了一个非常简洁的接口。这使得用各种各样的协议获取url成为可能。它同时也提供了一个稍微复杂的接口，来处理常见的状况-如基本的认证，cookies，代理，等等。

这些都是由叫做opener和handler的对象来处理的。

### urlib和urllib2

两者都是接受URL请求的相关模块，但是提供了不同的功能.



>区别：
>
>urllib2可以接受一个Request类的实例来设置URL请求的headers，urllib仅可以接受URL。
>
>这意味着，你不可以伪装你的User Agent字符串等。
>
>urllib提供**urlencode**方法用来GET查询字符串的产生，而urllib2没有。这是为何urllib常和urllib2一起使用的原因。



## urllib2的使用

### 基本的用法

在我们的使用中，只需要import,就可以把urllib2给引入。

就像下面这么简单：




	import urllib2

	response = urllib2.urlopen('http://python.org/')

	html = response.read()



通过**urlopen**的方法，向对应的url访问，也就是**http://python.org/**。

这上面的response对象，类似file，

我们通过**read()**函数来读取请求后，得到的信息。





>在真正的应用中，我们往往还需要携带数据，来作出请求，比如提交表单信息等。接下来，分别来看看如何用***post***和***get***方法来发送我们的请求



### post和get请求方式的介绍和区别



HTTP定义了与服务器交互的不同方法，最基本的方法有4种，分别是GET，POST，PUT，DELETE。

这里只介绍post,get：

GET一般用于<mark>获取/查询</mark>资源信息，根据HTTP规范，GET用于信息获取，而且应该是安全的和幂等的。

POST一般用于<mark>更新</mark>资源信息。根据HTTP规范，POST表示可能修改变服务器上的资源的请求。



而最明显看出来的区别，就是在我们传输数据的提交形式

>GET请求的数据会附在URL之后,用<mark>"?"</mark>号来分隔。例如：http://xxxx.com?name=hyddd&password=E4%BD%A0%E5%A5%BD
>
>POST提交的数据，则是放置在HTTP包的包体中。一般都要用到form表单进行提交



POST的安全性要比GET的安全性高：

>注意：这里所说的安全性, 和上面GET提到的“安全”不是同个概念。
>
>上面“安全”的含义仅仅是不作数据修改，而这里安全的含义是真正的Security的含义，比如：通过GET提交数据，用户名和密码将明文出现在URL上。因为：
>(1)登录页面有可能被浏览器缓存
>(2)其他人查看浏览器的历史纪录，那么别人就可以拿到你的账号和密码了
>除此之外，使用GET提交数据还可能会造成Cross-site request forgery攻击。



### Urllib2发送POST请求



	import urllib

	import urllib2



	url = 'http://www.someserver.com/register.cgi'

	values = {'name' : 'Michael Foord','language' : 'Python' }



	data = urllib.urlencode(values)

	req = urllib2.Request(url, data)



	response = urllib2.urlopen(req)

	the_page = response.read()







你肯定注意到了，我们在这里不仅使用了urllib2模块,还用到了之前所说的**urllib**模块。

原因在于，这些数据需要被以标准的方式encode，而encode需要在urllib中完成。

在values(要提交的数据)encode完成后，data（encode后的数据）才作为一个数据参数传送给Request对象。



完成这一系列的操作，我们才使用**urlopen**函数来获得了我们请求后的数据。

再使用read()函数读取。



### Urllib2发送GET请求


	import urllib2

	import urllib

	values['name'] = 'Somebody Here'
	values['language'] = 'Python'


	data = urllib.urlencode(values)
	 url = 'http://www.example.com/example.cgi'


	full_url = url + '?' + data


	response = urllib2.open(full_url)
	the_page = response.read()


在GET方式里，我们不需要像POST方式使用到Request函数。

在values经过encdoe后，直接把full_url按照GET请求的格式，拼凑好，就可以放在**open**函数里去使用了。

非常方便。





参考链接：


[urlib2模块使用介绍](http://mxdxm.iteye.com/blog/512728/)

[post,get介绍](http://www.cnblogs.com/hyddd/archive/2009/03/31/1426026.html)


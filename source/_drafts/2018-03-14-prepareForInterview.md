## TCP/IP 四层

TCP/IP网络协议栈四层分为：

- 应用层（Application）:FTP SMTP HTTP ... 
- 传输层（Transport）:TCP UDP 
- 网络层（Network）:IP ICMP ARP 
- 链路层（Link）:以太网 令牌环 FDDI ...

包含了一系列构成互联网基础的网络协议。
这些协议最早发源于美国国防部的 DARPA 互联网项目。
TCP/IP 字面上代表了两个协议:TCP 传输控制协议和 IP 互联网协议。 

## 五层基于 TCP/IP 加了一个物理层

[网络分层](http://www.ha97.com/3215.html)

## TCP 三次握手

C->S: syn,
S->C: syn ack,
C->S: ack

少了一次就建立连接，那么超时请求过来，就又将开启连接，浪费资源。这时候对于 S 回来的信息，C已经关闭，没有请求新的连接，会无视S这一次的ACK、Seq和ACK number。而对A来讲，连接已经建立了，其为此次连接分配的资源会一直等待下去

## TCP 四次挥手

C->S : FIN,准备关闭
S->C : ACK.知道.

C 进入 FIN_WAIT 状态

S->C : FIN.可以关闭了.
C->S : ACK,好的

C 进入 TIME_WAIT 状态一小段时间，如果Server端没有收到ACK则可以重传。时间过后，进行关闭。

## Shadowsocks 协议

# 其他

## 多线程

- Pthreads
>POSIX线程（POSIX threads），简称Pthreads，是线程的POSIX标准。该标准定义了创建和操纵线程的一整套API。在类Unix操作系统（Unix、Linux、Mac OS X等）中，都使用Pthreads作为操作系统的线程。

- NSThread
- GCD
  >串行队列，并行队列
  >同步提交，异步提交
  为什么说 GCD 死锁是队列导致的而不是线程，死锁不是操作系统的概念么？
  GCD队列种类：dispatch_get_main_queue（主队列）串行队列，dispatch_get_global_queue(全局队列) 并发队列 子线程执行，dispatch_queue_create（用户队列）串并都可以，子线程执行.
  
  主要的死锁就是当前串行队列里面同步执行当前串行队列。解决的方法就是将同步的串行队列放到另外一个线程执行。


- NSOperation & NSOperationQueue
> NSOperation 是苹果公司对 GCD 的封装，完全面向对象，所以使用起来更好理解。 大家可以看到 NSOperation 和 NSOperationQueue 分别对应 GCD 的 任务 和 队列 。操作步骤也很好理解：
>将要执行的任务封装到一个 NSOperation 对象中。
>将此任务添加到一个 NSOperationQueue 对象中。


## weak 的实现

weak 关键字的作用弱引用，所引用对象的计数器不会加一，并在引用对象被释放的时候自动被设置为 nil.

用一张记录对象的弱引用表，当释放时，根据弱引用表。 key 是对象的内存地址， value 是该对象的所有弱引用的指针数组,把它们置 nil.

## kvo 的实现

Apple 使用了 isa 混写（isa-swizzling）来实现 KVO 。当观察对象A时，KVO机制动态创建一个新的名为：NSKVONotifying_A 的新类，该类继承自对象A的本类，且 KVO 为 NSKVONotifying_A 重写观察属性的 setter 方法，setter 方法会负责在调用原 setter 方法之前和之后，通知所有观察对象属性值的更改情况。
（备注： isa 混写（isa-swizzling）isa：is a kind of ； swizzling：混合，搅合；

观察者观察的是属性，只有遵循 KVO 变更属性值的方式才会执行 KVO 的回调方法，例如是否执行了 setter 方法、或者是否使用了 KVC 赋值。
如果赋值没有通过 setter 方法或者 KVC，而是直接修改属性对应的成员变量，例如：仅调用 _name = @"newName"，这时是不会触发 KVO 机制，更加不会调用回调方法的。
所以使用 KVO 机制的前提是遵循 KVO 的属性设置方式来变更属性值。

## Runloop

NSRunLoop 和 CFRunLoopRef.

CFRunLoopRef 是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。
NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。

线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。

主线程的 RunLoop 里有两个预置的 Mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。这两个 Mode 都已经被标记为”Common”属性。DefaultMode 是 App 平时所处的状态，TrackingRunLoopMode 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个TableView时，RunLoop 会将 mode 切换为 TrackingRunLoopMode，这时 Timer 就不会被回调，并且也不会影响到滑动操作。

## AutoReleasePool

在没有手加Autorelease Pool的情况下，Autorelease对象是在当前的runloop迭代结束时释放的，而它能够释放的原因是系统在每个runloop迭代中都加入了自动释放池Push和Pop


官方文档建议，for循环中遍历产生大量autorelease变量时，就需要手加局部AutoreleasePool咯。

## 单例

```
+ (__class *)sharedInstance 
{ 
static dispatch_once_t once; 
static __class * __singleton__; 
dispatch_once( &once, ^{ __singleton__ = [[__class alloc] init]; } ); 
return __singleton__; 
}
```

## 关闭隐式动画

可以通过动画事务(CATransaction)关闭默认的隐式动画效果

```
 [CATransaction begin];
 [CATransaction setDisableActions:YES];
 self.myview.layer.position = CGPointMake(10, 10);
 [CATransaction commit];
```
# Block

iOS开发中经常会使用block结合gcd来完成多线程编程，block也属于对象，主要有三种类型.



## NSConcreteGlobalBlock 类型

全局静态Block,不访问任何外部变量，不会涉及到任何拷贝

存储在程序的数据区域(text段).

NSGlobalBlock：在block内部没有引用任何外部变量

```
void (^globalBlock) () = ^ () {
      NSLog(@"global block");
};

NSLog(@"%@", globalBlock);
//输出：<__NSGlobalBlock__: 0x1096e20c0>
```

对 NSGlobalBlock 的 retain、copy、release 操作都无效。

它既不在栈中，也不在堆中，我理解为它可能在内存的全局区。

## NSConcreteStackBlock 类型

存储在栈上.

有访问到外部变量的Block,该保存在栈中，当函数返回时被销毁

当栈中的block执行一次之后就被清除出栈了，所以无法多次使用。

## NSConcreteMallocBlock 类型

存储在堆上.

block 通常不会在源码中直接出现，因为默认它是当一个 block 被 copy 的时候，才会将这个 block 复制到堆中.

该类型的Block是由NSConcreteStackBlock 复制到堆中形成的。


### 三种 Block 关系

如果 Block 在记述全局变量的地方被设置或者 Block 没有捕获外部变量，那就生成一个 _NSConcreteGlobalBlock 实例。

其它情况都会生成一个 _NSConcreteStackBlock 实例，也就是说，它是在栈上的，所以一旦它所属的变量超出了变量作用域，该 Block 就被废弃了。

而当发生以下任一情况时：

- 手动调用 Block 的实例方法copy
- Block 作为函数返回值返回
- 将 Block 赋值给附有__strong修饰符的成员变量
- 在方法名中含有usingBlock的 Cocoa 框架方法或 GCD 的 API 中传递 Block

如果此时 Block 在栈上，那就复制一份到堆上，并将复制得到的 Block 实例的isa指针设为 _NSConcreteMallocBlock。

此时 Block 已经在堆上，那就把该 Block 的引用计数加1

## Block 循环引用

当block获取到外部变量时，由于编译器无法预测获取到的变量何时会被突然释放，为了保证程序能够正确运行，让block持有获取到的变量。

这就可能造成循环引用的问题。

主要思路，就是打破引用链，可以通过三种方法：
第一个，调用完 Block 后就将它置空
第二个，对于引用的对象，作为 weak 处理
第三个，对于引用的对象使用后置空

weak-strong 

strong 延长捕获对象的生命周期，一旦 Block 执行完，对象被释放，而 Block 也会被释放.

## 图像处理

OpenGLES的世界坐标系是[-1, 1]，故而点(0, 0)是在屏幕的正中间。

纹理坐标系的取值范围是[0, 1]，原点是在左下角。故而点(0, 0)在左下角，点(1, 1)在右上角

### 离屏渲染

Off-Screen Rendering 意为离屏渲染，指的是 GPU 在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作。

相比于当前屏幕渲染，离屏渲染的代价是很高的，主要体现在两个方面：

* 创建新缓冲区 要想进行离屏渲染，首先要创建一个新的缓冲区。

* 上下文切换 离屏渲染的整个过程，需要多次切换上下文环境：先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上有需要将上下文环境从离屏切换到当前屏幕。而上下文环境的切换是要付出很大代价的。







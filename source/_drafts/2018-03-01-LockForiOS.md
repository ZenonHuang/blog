又到了春天挪坑的季节，想起多次被问及到锁的概念，决定好好总结一番。

翻看目前关于 iOS 开发里的锁的文章，大部分都起源于 ibireme 的 [《不再安全的 OSSpinLock》](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)。

这里不会说锁的使用，主要在于想依次解决关于锁的疑问:

1. 锁是什么?
2. 为什么要有锁？
3. 锁的分类问题
4. 为什么 OSSpinLock 不安全？
5. 解决自旋锁不安全问题的几种方式
6. 为什么换用其它的锁，可以解决 OSSpinLock 的问题？
7. 自旋锁和互斥锁的关系是平行对立的吗？
8. 信号量和互斥量的关系
9. 递归锁使用的场景
10. 信号量和条件变量的区别

# 锁是什么

锁 -- 是保证线程安全常见的同步工具。锁是一种非强制的机制，每一个线程在访问数据或者资源前，要先获取(Acquire) 锁，并在访问结束之后释放(Release)锁。如果锁已经被占用，其它试图获取锁的线程会等待，直到锁重新可用。

## 为什么要有锁？

前面说到了，锁是用来保护线程安全的工具。

可以试想一下，多线程编程时，没有锁的情况 -- 也就是线程不安全。

当多个线程同时对一块内存发生读和写的操作，可能出现意料之外的结果:

程序执行的顺序会被打乱,可能造成提前释放一个变量,计算结果错误等情况。

所以我们需要将线程不安全的代码 “锁” 起来。保证一段代码或者多段代码操作的原子性，保证多个线程对同一个数据的访问**同步 (Synchronization)**。

### 属性设置 atomic  

上面提到了原子性，我马上想到了属性关键字里， atomic 的作用。

设置 atomic 之后，默认生成的 getter 和 setter 方法执行是原子的。 

但是它只保证了自身的读/写操作，却不能说是线程安全。

如下情况: 

	//thread A
	for (int i = 0; i < 100000; i ++) {
    if (i % 2 == 0) {
        self.arr = @[@"1", @"2", @"3"];
    }
    else {
        self.arr = @[@"1"];
    }
    	NSLog(@"Thread A: %@\n", self.arr);
	}

    
	//thread B
	if (self.arr.count >= 2) {
    	NSString* str = [self.arr objectAtIndex:1];
	}
	

就算在 **thread B** 中针对 arr 数组进行了大小判断，但是仍然可能在 `objectAtIndex:` 操作时被改变数组长度，导致出错。这种情况声明为 atomic 也没有用。

而解决方式，就是进行加锁。需要注意的是，读/写的操作都需要加锁，不仅仅是对一段代码加锁。

# 锁的分类

锁的分类方式，可以根据锁的状态，锁的特性等进行不同的分类，很多锁之间其实并不是并列的关系，而是一种锁下的不同实现。可以参考 [几个关于锁的名词联系是什么](https://www.zhihu.com/question/39850927) 中 `Tim Chen` 的回答。

很多谈论锁的文章，都会提到互斥锁，自旋锁。很少有提到它们的关系，其实自旋锁，也是互斥锁的一种实现，`spin lock`和 `mutex` 都是为了解决某项资源的互斥使用,在任何时刻只能有一个保持者。

区别在于 `spin lock`和 `mutex` 调度机制上有所不同。

# OSSpinLock 

OSSpinLock 是一种自旋锁。 它的特点是在线程等待时会一直轮询，处于忙等状态。自旋锁由此得名。

自旋锁看起来是比较耗费 cpu 的，然而在互斥临界区计算量较小的场景下，它的效率远高于其它的锁。

因为它是一直处于 running 状态，减少了线程切换上下文的消耗。

## 为什么 OSSpinLock 不再安全？

关于 OSSpinLock 不再安全，原因就在于优先级反转问题。

### 优先级反转(Priority Inversion)

什么情况叫做优先级反转？

wikipedia 上是这么定义的：

>优先级倒置，又称优先级反转、优先级逆转、优先级翻转，是一种不希望发生的任务调度状态。在该种状态下，一个高优先级任务间接被一个低优先级任务所抢先(preemtped)，使得两个任务的相对优先级被倒置。
>这往往出现在一个高优先级任务等待访问一个被低优先级任务正在使用的临界资源，从而阻塞了高优先级任务；同时，该低优先级任务被一个次高优先级的任务所抢先，从而无法及时地释放该临界资源。这种情况下，该次高优先级任务获得执行权。

再消化一下

>有：高优先级任务A / 次高优先级任务B / 低优先级任务C / 资源Z 。
>A 等待 C 使用 Z，而 B 并不需要 Z，抢先获得时间片执行。C 由于没有时间片，无法执行。
>这种情况造成 A 在 B 之后执行,使得优先级被倒置了。
>而如果 A 等待资源时不是阻塞等待，而是忙循环，则可能永远无法获得资源。此时 C 无法与 A 争夺 CPU 时间，从而 C 无法执行，进而无法释放资源。造成的后果，就是 A 无法获得 Z 而继续推进。

而 OSSpinLock 忙等的机制，就可能造成高优先级一直 running ，占用 cpu 时间片。而低优先级任务无法抢占时间片，变成迟迟完不成，不释放锁的情况。

### 优先级反转的解决方案

关于优先级反转一般有以下三种解决方案

#### 优先级继承

优先级继承，故名思义，是将占有锁的线程优先级，继承等待该锁的线程高优先级，如果存在多个线程等待，就取其中之一最高的优先级继承。

#### 优先级天花板

优先级天花板，则是直接设置优先级上限，给临界区一个最高优先级，进入临界区的进程都将获得这个高优先级。

如果其他试图进入临界区的进程的优先级，都低于这个最高优先级，那么优先级反转就不会发生。

#### 禁止中断

禁止中断的特点，在于任务只存在两种优先级：可被抢占的 / 禁止中断的 。

前者为一般任务运行时的优先级，后者为进入临界区的优先级。

通过禁止中断来保护临界区，没有其它第三种的优先级，也就不可能发生反转了。

## 为什么使用其它的锁，可以解决优先级反转？

我们看到很多使用 OSSpinLock 的项目，都该用了其它方式替代，比如 `pthread_mutex` 和 ` dispatch_semaphore` 。

那为什么其它的方式，就不会有优先级反转的问题呢？按照上面的想法，其它锁也可能出现优先级反转。

原因在于，其它锁出现优先级反转后，高优先级的任务不会忙等。因为处于等待状态的高优先级任务，没有占用时间片，所以低优先级任务一般都能进行下去，从而释放掉锁。

## 线程调度

为了帮助理解，要提一下有关`线程调度`的概念。

无论多核心还是单核，我们的线程运行总是 "并发" 的。

当 cpu 数量大于等于线程数量，这个时候是真正并发，可以多个线程同时执行计算。

当 cpu 数量小于线程数量，总有一个 cpu 会运行多个线程，这时候"并发"就是一种模拟出来的状态。操作系统通过不断的切换线程，每个线程执行一小段时间，让多个线程看起来就像在同时运行。这种行为就称为 "线程调度（Thread Schedule）"。

### 线程状态

在线程调度中，线程至少拥有三种状态 : 运行(Running),就绪(Ready),等待(Waiting)。

处于 Running 的线程拥有的执行时间，称为 时间片 (Time Slice)，时间片 用完时，进入 Ready 状态。如果在 Running 状态，时间片没有用完，就开始等待某一个事件（通常是 IO 或 同步 ），则进入 Waiting 状态。 

如果有线程从 Running 状态离开，调度系统就会选择一个 Ready 的线程进入 Running 状态。而 Waiting 的线程等待的事件完成后，就会进入 Ready 状态。

# dispatch_semaphore

dispatch_semaphore 是 GCD 中同步的一种方式，与他相关的只有三个函数，一个是创建信号量，一个是等待信号，一个是发送信号。

## 信号量机制

信号量中，二元信号量，是一种最简单的锁。只有两种状态，占用和非占用。

## 信号量和互斥量的区别

信号量是允许并发访问的，也就是说，允许多个线程同时执行多个任务。信号量可以由一个线程获取，然后由不同的线程释放。

互斥量只允许一个线程同时执行一个任务。也就是同一个程获取，同一个线程释放。

之前我对，互斥量只由一个线程获取和释放，理解的比较狭义，以为系统强制要求的意思，用 `NSLock ` 实验发现可以在不同线程获取和释放，就很疑惑。

实际上，的确能在不同线程获取和释放同一个互斥锁，但是本来使用互斥锁目的，就是在同一个线程中上锁。

这里的关键，就是理解信号量，可以允许 N 个信号量允许 N 个线程并发地执行任务。

在这里附上一个实验的 [Demo]() 

## @synchonized

@synchonized 是一个递归锁。

### 递归锁

递归锁也称为可重入锁。互斥锁可以分为非递归锁/递归锁两种，主要区别在于:同一个线程可以重复获取递归锁，不会死锁; 同一个线程重复获取非递归锁，则会产生死锁。

因为是递归锁，我们可以写类似这样的代码:

   	- (void)testLock{
       if(_count>0){ 
		  @synchronized (obj) {
		     _count = _count - 1;
    		 [self testLock];
	 	  }
	 	}
	 }
	
而如果换成 NSLock ，它就会因为递归发生死锁了。

### 实际使用问题

如果 obj 为 nil,或者 obj 地址不同，锁会失效。

所以我们要防止如下的情况:
		
		@synchronized (obj) {
			obj = newObj;
		}	

这里的 obj 被更改后，等到其它线程访问时，就和没加锁一样直接进去了。

另外一种情况，就是 `@synchonized(self)`. 不少代码都是直接将self传入@synchronized当中，而 `self` 很容易作为一个外部对象，被调用和修改。所以它和上面是一样的情况，需要避免使用。

正确的做法是什么？obj 应当传入一个类内部维护的NSObject对象，而且这个对象是对外不可见的,不被随便修改的。

## pthread_mutex

pthread 定义了一组跨平台的线程相关的 API，其中可以使用 pthread_mutex 作为互斥锁。

`pthread_mutex ` 不是使用忙等，而是同信号量一样，会阻塞线程并进行等待，调用时进行线程上下文切换。

`pthread_mutex` 本身拥有设置协议的功能，通过设置它的协议，来解决优先级反转:
>pthread_mutexattr_setprotocol(pthread_mutexattr_t *attr, int protocol) 

其中协议类型包括以下几种：
- PTHREAD_PRIO_NONE：线程的优先级和调度不会受到互斥锁拥有权的影响。 - PTHREAD_PRIO_INHERIT：当高优先级的等待低优先级的线程锁定互斥量时，低优先级的线程以高优先级线程的优先级运行。这种方式将以继承的形式传递。当线程解锁互斥量时，线程的优先级自动被降到它原来的优先级。该协议就是支持优先级继承类型的互斥锁，它不是默认选项，需要在程序中进行设置。 - PTHREAD_PRIO_PROTECT：当线程拥有一个或多个使用 PTHREAD_PRIO_PROTECT 初始化的互斥锁时，此协议值会影响其他线程（如 thrd2）的优先级和调度。thrd2 以其较高的优先级或者以 thrd2 拥有的所有互斥锁的最高优先级上限运行。基于被 thrd2 拥有的任一互斥锁阻塞的较高优先级线程对于 thrd2的调度没有任何影响。

所以，设置协议类型为 `PTHREAD_PRIO_INHERIT` ，运用优先级继承的方式，解决优先级反转的问题。

而我们在 iOS 中使用的 NSLock,NSRecursiveLock 都是基于 pthread_mutex 做实现的。


## NSLock

NSLock 属于 pthread_mutex 的一层封装, 设置了属性为 PTHREAD_MUTEX_ERRORCHECK。

它会损失一定性能换来错误提示。并简化直接使用 pthread_mutex 的定义。

## NSCondition

NSCondition 是通过**条件变量(condition variable)** pthread_cond_t 来实现的。

NSCondition 其实是封装了一个互斥锁和条件变量， 它把前者的 lock 方法和后者的 wait/signal 统一在 NSCondition 对象中，暴露给使用者。


[conditionLock wait];
[conditionLock signal];


### 条件变量

条件变量，它有点像信号量，提供了线程阻塞与信号机制，因此可以用来阻塞某个线程，并等待某个数据就绪，随后唤醒线程，比如常见的生产者-消费者模式。

线程间的同步还有这样一种情况：
线程A需要等某个条件成立,才能继续往下执行.现在这个条件不成立，线程A就阻塞等待.
而线程B在执行过程中使这个条件成立了，就唤醒线程A继续执行。

在pthread库中通过条件变量（Condition Variable）来阻塞等待一个条件，或者唤醒等待这个条件的线程。
Condition Variable用pthread_cond_t类型的变量表示。

一个Condition Variable总是和一个Mutex搭配使用的。一个线程可以调用pthread_cond_wait在一个Condition Variable上阻塞等待。

采用条件变量控制线程同步，最为经典的例子要数生产者和消费者问题。

#### 生产者-消费者问题

生产者消费者问题,是一个著名的线程同步问题，该问题描述如下：

有一个生产者在生产产品，这些产品将提供给若干个消费者去消费，为了使生产者和消费者能并发执行，在两者之间设置一个具有多个缓冲区的缓冲池，生产者将它生产的产品放入一个缓冲区中，消费者可以从缓冲区中取走产品进行消费，显然生产者和消费者之间必须保持同步，即不允许消费者到一个空的缓冲区中取产品，也不允许生产者向一个已经放入产品的缓冲区中再次投放产品。

使用 NSCondition 解决生产消费者问题。[Demo]()

#### 条件变量和信号量的区别

每个信号量有一个与之关联的值，挂出时+1，等待时-1，那么任何线程都可以挂出一个信号，即使没有线程在等待该信号量的值。

不过对于条件变量来说，如果pthread_cond_signal之后，没有任何线程阻塞在 pthread_cond_wait 上，那么此条件变量上的信号丢失。


## NSConditionLock

NSConditionLock，即条件锁，可以设置自定义的条件来获得锁。

NSConditionLock 借助 NSCondition 来实现，它的本质就是一个生产者-消费者模型。“条件被满足”可以理解为生产者提供了新的内容。

相比于 NSLock 多了个 condition 参数，我们可以理解为一个条件标示。

[conditionLock lockWhenCondition:3];
[conditionLock unlockWithCondition:3];

值得注意的是，利用 NSConditionLock 还可以实现任务之间的依赖.

## NSRecursiveLock

NSRecursiveLock 与 NSLock 的区别在于内部封装的 pthread_mutex_t 对象的类型不同，前者的类型为 PTHREAD_MUTEX_RECURSIVE。

## NSDistributedLock

这里顺带提一下 `NSDistributedLock`, 是 macOS 下的一种锁. 

[苹果文档](https://developer.apple.com/documentation/foundation/nsdistributedlock?language=objc) 对于NSDistributedLock 的描述是:
>A lock that multiple applications on multiple hosts can use to restrict access to some shared resource, such as a file

意思是说，它是一个用在多个主机间的多应用的锁，可以限制访问一些共享资源，例如文件。

按字面意思翻译，`NSDistributedLock` 应该就叫做 分布式锁。但是看概念和资料，在 [解决NSDistributedLock进程互斥锁的死锁问题(一)](http://www.tanhao.me/pieces/1731.html/) 里面看到，NSDistributedLock 更类似于文件锁的概念。 有兴趣的可以看一看 [Linux 2.6 中的文件锁](https://www.ibm.com/developerworks/cn/linux/l-cn-filelock/)


## 保证线程安全的方式

### 使用单线程访问

首先，尽量避免多线程的设计。因为多线程访问会出现很多不可控制的情况。有些情况即使上锁，也无法保证百分之百的安全，例如自旋锁的问题。

### 不对资源做修改

而如果还是得用多线程，那么避免对资源做修改。

如果都是访问共享资源，而不去修改共享资源，也可以保证线程安全。

比如 NSArry 这样不可变类是线程安全的。然而它们的可变版本，比如 NSMutableArray 是线程不安全的。事实上，如果是在一个队列中串行地进行访问的话，在不同线程中使用它们也是没有问题的。

# 总结

如果实在要使用多线程，也不必过分追求效率，而是更多的考虑安全问题，使用对应的锁。

对于平时编写应用层多线程安全代码，我还是建议大家多使用 @synchronized，NSLock，或者dispatch_semaphore_t，多线程安全比多线程性能更重要。

# 参考

[线程同步：互斥锁，条件变量，信号量](http://www.cnblogs.com/549294286/p/3687678.html)

[Objective-C中不同方式实现锁(一)](http://www.tanhao.me/pieces/616.html/)

[关于 @synchronized，这儿比你想知道的还要多](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)

[linux c学习笔记----互斥锁属性](http://lobert.iteye.com/blog/1762844)

[Java中的锁分类](https://www.cnblogs.com/qifengshi/p/6831055.html)

[正确使用多线程同步锁@synchronized()](http://mrpeak.cn/blog/synchronized/)

[iOS多线程到底不安全在哪里？](http://mrpeak.cn/blog/ios-thread-safety/)

[iOS 中几种常用的锁总结](https://www.jianshu.com/p/1e59f0970bf5)




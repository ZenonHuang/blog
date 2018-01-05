# UIScrollView 自动算高

# UITableViewCell高度 自动算高

# Cell 高度缓存

高度的询问都会命中缓存，减少cpu的计算任务。

# 异步计算

将布局的计算放在后台线程，异步计算，同时为了减少过多线程切换的开销，在MainRunLoop即将休眠时把计算好的布局应用到视图上。

## ASDK

## YYAsyncLayer

## Runloop 任务分发

## GCD

# UIView 改用 Layer 绘制

不用响应的事件的图层，使用 Layer 展示.而 UIView 和 CALayer 不是线程安全的。并且只能在主线程创建、访问和销毁。

# FlexBoxLayout 源码阅读

之前对于 FlexBox 作了一个学习，接受组长的任务，继续挖掘它的实际使用场景。比如常见的滑动视图，tableView 布局等。

由于看到了 [FlexBoxLayout](https://github.com/LPD-iOS/FlexBoxLayout) ，对于这些操作多已经有了解决。决定对其项源码进行学习。

# Component

首先来看它的组件部分

## FBLayoutDiv

FBLayoutDiv 是一个虚拟视图 Div 。看它的头文件带有一个属性 `frame`，和两个实例方法设置主轴，主轴对齐方式，侧轴对齐方式等。

FBLayoutDiv 还遵守 `FBLayoutProtocol` 的协议。

### 初始化 

FBLayoutDiv 的初始化方法， 做了三个事情:

```
    //实例化一个 _fb_layout  对象
    _fb_layout = [FBLayout new];
    
    //将 _fb_layout 的 context 赋值成 self
    _fb_layout.context = self;
    
    //实例化 _fb_children 数组
    _fb_children = [NSMutableArray array];
```

这里涉及到了几个问题先保留:

FBLayout 对象是什么？

FBLayout 的 context 属性是作什么的?

### 设置 Layout 

在初始化后， `FBLayoutDiv` 就会执行 `fb_makeLayout:`,设置它布局的方向。


## UIScrollView+FBLayout

`UIScrollView` 这里的处理比较简单，也很巧妙。

放置了一个 `FBLayoutDiv` 类型的 `fb_contentDiv`，以及 `fb_clearLayout` 的方法。

重点在于交换了 `layoutSubviews` 方法，用 `fb_layoutSubviews` 代替。在里面根据 `fb_contentDiv` 的大小，赋值给 UIScrollView 的 `contentSize`.

`fb_clearLayout` 里面就是直接把 `fb_contentDiv` 置 nil .

里面对关联对象的写法，也值得学习。

## UITableView+FBLayout

### 算高

### 缓存高度

#### UITableView+FDIndexPathHeightCache

参考 `UITableView-FDTemplateLayoutCell`

## UIView+FBLayout

## UIView+CellStyle

# Transaction

## FBAsyLayoutTransaction

# Layout

## FBLayout

先说 context ，进到 setContext 方法,看到下面的代码:

```
- (void)setContext:(id)context {
  _context = context;
  YGNodeSetContext(_fbNode, (__bridge void *)(context));
}
```

首先给 id 对象 context 赋值，然后使用了一个  YGNodeSetContext 的操作 -- 放进去一个 YGNodeRef 结构体，再给出一个 context 的指针。

### YGNodeRef 

YGNodeRef is the main object you will be interfacing with when using Yoga in C. YGNodeRef is a pointer to an internal YGNode struct.

### Context

Context is important when integrating Yoga into another layout system. Context allows you to associate another object with a `YGNodeRef`. This context can then be retrieved from a `YGNodeRef` when for example its measure function is called. This is what enables Yoga to rely on the Android and iOS system implementations of text measurement in React Native.

```source-c
void YGNodeSetContext(YGNodeRef node, void *context);
void *YGNodeGetContext(YGNodeRef node);
```


## FBViewLayoutCache



# 参考

[优化UITableViewCell高度计算的那些事](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/)

[iOS 保持界面流畅的技巧](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)


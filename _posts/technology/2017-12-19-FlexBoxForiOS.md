---
layout: post
date: 2017-12-25 16:25:00
title: iOS 上的 FlexBox 布局 
category: 技术
keywords: iOS
description: 学习 FlexBox 布局的思想，并对 iOS 上的 FlexBox 布局进行实践。 
---


# 为什么要了解 FlexBox?

最近时不时的听到关于 FlexBox 的声音，除了在 **Weex** 以及 **React Native** 两个著名的跨平台项目里有用到 FlexBox 外，**AsyncDisplayKit** 也同样引入了 FlexBox 。 

先说说 iOS 本身提供给我们 2 种布局方式:

- Frame,直接设置横纵坐标，并指定宽高。
- Auto Layout，通过设置相对位置的约束进行布局。

Frame 没什么太多可说的了，直接制定坐标和大小，设置绝对值。

`Auto Layout` 本身用意是好的，试图让我们从 Frame 中解放出来，摆脱关于坐标和大小的刻板思考方式。转而利用 UI 之间的相对位置关系，设置对应约束进行布局。

但是 `Auto Layout` 好心并未做成好事，它的语法又臭又长! 至今学习 iOS 两年，我使用到原生 `Auto Layout` 语法的时候屈指可数。只能靠 [Masonry](https://github.com/SnapKit/Masonry) 这样的第三方库来使用它。

## Auto Layout 的原理

说完了 `Auto Layout` 的使用，再来看看它工作原理。 

实际上，我们设置 `Auto Layout` 的约束，就构成一系列的条件,成为一个方程。然后解出 Frame 的坐标和大小。

例如，我们设置一个名为 A 的 UI :

    A.center = super.center
    A.width  = 40
    A.height = 40

则:
A.frame = (super.center.x,super.center.y,40,40)

再设置一个 B:

    B.width  =  A.width
    B.height =  A.height
    B.top    =  A.bottom + 50
    B.left   =  A.left

则: 
B.frame =  ( A.x , A.y + A.height + 50 , A.width , A.height )

如图：

![ZenonHuang_FlexBox_1](https://user-gold-cdn.xitu.io/2017/12/25/1608d2744632d1f8?w=341&h=290&f=png&s=5192)


### Cassowary

`Auto Layout` 内部有专门用来处理约束关系的算法，我一直以为是苹果自家研发的，查阅资料才发现是来自一个叫 `Cassowary` 的算法。

>Cassowary是个解析工具包，能够有效解析线性等式系统和线性不等式系统，用户的界面中总是会出现不等关系和相等关系，Cassowary开发了一种规则系统可以通过约束来描述视图间关系。约束就是规则，能够表示出一个视图相对于另一个视图的位置。
> - 戴铭 [<深入剖析Auto Layout，分析iOS各版本新增特性>](https://ming1016.github.io/2015/11/03/deeply-analyse-autolayout/)

有兴趣的可以进一步了解该算法的实现。

## Frame / Auto Layout  / FlexBox 的性能对比

在对 `Auto Layout` 进行一番了解之后,我们很容易得出 `Auto Layout` 因为多余的计算，性能差于 Frame 的结论。

但究竟差多少呢？FlexBox 的表现又如何呢?

这里根据 [从 Auto Layout 的布局算法谈性能](https://draveness.me/layout-performance) 里的测试代码进行修改，对 Frame / Auto Layout / FlexBox 进行布局，分段测算 10 ～ 350 个 UIView 的布局时间。取 100 次布局时间的平均值作为结果,耗时单位为秒。

结果如下图:

![ZenonHuang_FlexBox_2](https://user-gold-cdn.xitu.io/2017/12/25/1608d27446d5177d?w=1886&h=1418&f=png&s=225064)


虽然测试结果难免有偏差，但是根据折线图可以明显发现，FlexBox 的布局性能是比较接近 Frame 的。 

`60 FPS` 作为一个 iOS 流畅度的黄金标准，要求布局在 0.0166667 s 内完成，`Auto Layout` 在超过 50 个视图的时候，可能保持流畅就会开始有问题了。

本次测试使用的机器配置如下：
![ZenonHuang_FlexBox_3](https://user-gold-cdn.xitu.io/2017/12/25/1608d27446a5cc5a?w=291&h=150&f=png&s=44836)

采用 Xcode9.2 ,iPad Pro (12.9-inch)(2nd generation) 模拟器。

测试布局的项目代码上传在 [GitHub](https://github.com/ZenonHuang/MyDemos/tree/master/LayoutTest)


# FlexBox 是什么？

`FlexBox` 是一种 UI 布局方式，并得到了所有浏览器的支持。`FlexBox` 首先是基于 [*盒装状型*](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Box_Model/Introduction_to_the_CSS_box_model) 的，Flexible 意味着弹性，使其能适应不同屏幕，补充盒状模型的灵活性。

`FlexBox` 把每个视图，都看作一个矩形盒子，拥有内外边距，沿着主轴方向排列，并且，同级的视图之间没有依赖。

和 `Auto Layout` 类似，`FlexBox` 采用了描述性的语言去进行布局，而不像 Frame 直接用绝对值坐标进行布局。

>弹性布局的主要思想是让 Flex Container 有能力来改变 Flex Item 的宽度和高度，以填满可用空间（主要是为了容纳所有类型的显示设备和屏幕尺寸）的能力。

最重要的是, `FlexBox` 布局与方向无关，常规的布局设计缺乏灵活性，无法支持大型和复杂的应用程序（特别是涉及到方向转变，缩放、拉伸和收缩等）。

## FlexBox 组成

采用 `FlexBox` 布局的元素，称为 `Flex Container`。

`Flex Container` 的所有子元素，称为 `Flex Item`。

![ZenonHuang_FlexBox_4](https://user-gold-cdn.xitu.io/2017/12/25/1608d2744660ab9c?w=602&h=300&f=png&s=17165)


下面会讲一下 FlexBox 里面的一些概念，方便之后进行 FlexBox 的使用。

### Flex Container

前面提到了，`FlexBox` 的一个特点，就是视图之间，是没有依赖的。

`Flex Item` 的排布，就依赖于 `Flex Container` 的属性设置，而不用相互之间进行设置。

所以先说一下 `Flex Containner` 的属性设置。




#### Flex Direction

FlexBox 有一个 `主轴(main axis)` 和 `侧轴(cross axis)`的概念。侧轴垂直于主轴。

它们可以是水平，也可以是垂直。

主轴默认为 `Row` , 侧轴默认为 `Column`：

![0DF515D1-1EEF-4C38-9782-F875C1433AE0](https://user-gold-cdn.xitu.io/2017/12/25/1608d27447ccff3d?w=769&h=276&f=png&s=60925)

`Flex Direction` 决定了 `Flex Containner ` 内的主轴排布方向。

主轴默认为 Row (从左到右):

![87691D2C-34C3-4805-B960-4D8217717D98](https://user-gold-cdn.xitu.io/2017/12/25/1608d274448a4734?w=338&h=50&f=jpeg&s=7002)

同时，也可以设置 RowRevers(从右至左):
![1F430BC5-A0BE-474B-9791-23F2B308AEE9](https://user-gold-cdn.xitu.io/2017/12/25/1608d274f5df32a4?w=329&h=50&f=png&s=7728)


Column(从上到下):
![AA2DF5F6-1164-4440-ACF2-9897D0D82730](https://user-gold-cdn.xitu.io/2017/12/25/1608d274f3d36520?w=119&h=200&f=png&s=7603)


ColumnRevers(从下到上):
![C33D6321-66C3-4A78-B429-E82B3F83CB6E](https://user-gold-cdn.xitu.io/2017/12/25/1608d274f88da247?w=98&h=200&f=png&s=7477)

#### Flex Wrap

Flex Wrap 决定在轴线上排列不下时，视图的换行方式。

Flex Wrap 默认设置为 NoWrap，不会换行，一直沿着主轴排列到屏幕之外:

![9C0FD351-E504-4A6B-A442-E3DE1E084FAC](https://user-gold-cdn.xitu.io/2017/12/25/1608d274fd9310c9?w=419&h=50&f=png&s=9444)

设置为 Wrap ,则空间不足时，自动换行:

![2C14AAC1-6DDB-4450-B393-5497E0743AFC](https://user-gold-cdn.xitu.io/2017/12/25/1608d27504a1e778?w=202&h=50&f=png&s=6615)


设置 WrapReverse，则换行方向与 Wrap 相反:

![35B10621-C2BD-4699-B5B5-4383B35F510E](https://user-gold-cdn.xitu.io/2017/12/25/1608d275053823fd?w=203&h=50&f=png&s=6569)

这是一个非常有用的属性。比如典型的`九宫格布局`，iOS 如果不是用 `UICollectionView` 做，那么就需要保存 `9` 个实例，然后做判断，计算 frame ，可维护性实在不高。使用`UICollectionView` 可以很好的解决布局，但很多场景并不能复用，做起来也不是特别简单。 

FlexBox 布局的话，用 `Flex Wrap` 属性设置 `Wrap` 就可以直接搞定。

移动平台上相似的方案，比如 Android 的 Linear Layout 和 iOS 的 UIStackView ，但却远没有 FlexBox 强大。

#### Display

Display 选择是否计算它，默认为 Flex. 如果设置为 None 自动忽略该视图的计算。

在根据逻辑显示 UI 时，比较有用。

比如我们现有的业务，需要显示的腾讯身份标示。按照一般做法，多个 icon 互相连成一排，根据身份去设置不同的距离，同时隐藏其他 icon ,比较的麻烦。iOS 最好的办法是使用 UIStackView ，这又有版本兼容等问题。而使用 FlexBox 布局,当不是某个身份时，只要设置 Display 为 None,就不会被纳入 UI 计算当中。

#### Justify Content

`Justify Content` 用于定义 `Flex Item` 在主轴上的对齐方式:FlexStart(主轴起点对齐)，FlexEnd(主轴终点对齐)，Center(居中对齐)。

还有SpaceBetween（两端对齐）:

![7F4F84F0-6B50-462A-BDA6-D11D087FFCE0](https://user-gold-cdn.xitu.io/2017/12/25/1608d2751e764ed5?w=414&h=50&f=png&s=7247)

设置两端对齐，让 `Flex Item` 之间的间隔相等。

SpaceAround(外边距相等排列)：

![3B7E08DD-6F78-4A8D-9D56-07565E5F9E24](https://user-gold-cdn.xitu.io/2017/12/25/1608d27526597add?w=412&h=50&f=png&s=7178)

让每个 `Flex Item` 四周的外边距相等

#### Align Items

`Align Items` 定义 `Flex Item` 在侧轴上的对齐方式。

除了可以和主轴对齐方式  `Justify Content` 一样设置FlexStart ,FlexEnd,Center,SpaceBetween,SpaceAround 之外，还有 Baseline(基线对齐)：

![25B54897-E6D4-4742-837E-13E5E4D827DA](https://user-gold-cdn.xitu.io/2017/12/25/1608d27528e5b368?w=346&h=100&f=png&s=6160)

如图所示，它是基于 `Flex Item` 的第一行文字的基线对齐。

`Baseline` 如果 `Flex Item` 的行内轴与侧轴为同一条，则该值与 `FlexStart` 等效。 其它情况下，该值将参与基线对齐。


Stretch：
![F40B6D31-225F-4A99-9209-15886475CC1F](https://user-gold-cdn.xitu.io/2017/12/25/1608d27529552a7b?w=341&h=100&f=png&s=6099)

`Stretch` 让 `Flex Item` 拉伸填充整个`Flex Container`。`Stretch`会使`Flex Item`的外边距在遵照对应属性限制下,尽可能接近所在行或列的尺寸。


如果 `Flex Item` 未设置数值,或设为 **`auto`**，将占满整个`Flex Container`的高度

#### Align Content

`Align Content` 也是侧轴在 `Flex Item` 里的对齐方式，只不过是以一整个行，作为最小单位。

注意，如果`Flex Item`只有一根轴线（只有一行的`Flex Itme`），该属性不起作用。

调整为 `FlexWrap` 为 `Wrap`,效果才显示出来： 

![6C8FB222-16DC-4B6E-A2D9-251C4CA69F8E](https://user-gold-cdn.xitu.io/2017/12/25/1608d2754269c25f?w=271&h=300&f=png&s=24290)


### Flex Item

在上面说完了 `Flex Container` 的属性,终于说到了 `Flex Item`.  `Flex Container` 里的属性，都是作用于自己包含的 `Flex Item`，`Flex Item` 的属性，都是作用于自己本身，. 

#### AlignSelf

`AlignSelf` 可以让单个 `Flex Item` 与其它 `Flex Item` 有不一样的对齐方式，覆盖 `Align Items`属性。

默认值为`auto`，表示继承`Flex Container`的`Align Items`属性。如果它本身没有`Flex Container`，则等同于`Stretch`。 

#### FlexGrow

`FlexGrow` 可以设置分配`剩余空间`的`比例`。即如何扩大。

`FlexGrow` 默认值为 `0`，如果没有去定义 `FlexGrow`，该布局是不会拥有分配剩余空间权利的。

例如：

整体宽度 100 , sub1 宽为 10 ，sub2 宽为 20 ，则`剩余空间`为 70。

设置 `FlexGrow` 就是分配这 70 宽度的比例。

再说比例值的问题:

如果所有 `Flex Item` 的 `FlexGrow` 属性都为 `1` ，如果有剩余空间的话，则等分剩余空间。

如果一个 `Flex Item` 的 `FlexGrow` 属性为 `2`，其余 `Flex Item` 都为 `1` ，则前者占据的剩余空间将比其他 `Flex Item` 多 `1` 倍。

#### FlexShrink

与 `FlexGrow` 处理空间剩余相反，`FlexShrink` 用来处理空间不足的情况。即怎么缩小。

`FlexShrink` 默认为1，即如果空间不足，该项目将缩小

如果所有 `Flex Item` 的 `FlexShrink` 属性都为 `1`，当空间不足时，都将等比例缩小。

如果一个 `Flex Item` 的 `FlexShrink` 属性为 `0` ，其余 `Flex Item` 都为`1`，则空间不足时，`FlexShrink` 为 `0` 的前者不缩小。

#### FlexBasis

`FlexBasis` 定义了在分配多余的空间之前， `Flex Item` 占据的 `main size（主轴空间）`。浏览器根据这个属性，计算主轴是否有多余空间。

`FlexBasis` 的默认值为 `auto`，即 `Flex Item` 的本来大小。

想了解更多 FlexBox 属性，可以参考 [A Complete Guide to Flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)

# FlexBox 的实现 -- Yoga 

最开头已经介绍过，FlexBox 布局已经应用于几个知名的开源项目，它们用到的就是来自于 Facebook 的 Yoga.

[Yoga](https://github.com/facebook/yoga) 是由 C 实现的 Flexbox 布局引擎，性能和稳定性已经在各大项目中得到了很好的验证,但不足的是 Yoga 只实现了 W3C 标准的一个子集。

下面将针对 Yoga  iOS 上的实现 [`YogaKit`](https://github.com/facebook/yoga/tree/master/YogaKit) 做一些讲解。

基于上面对 `FlexBox` 布局的基本了解，作一些简单的布局。

## YGLayout 

整个 YogaKit 的关键，就在于  `YGLayout` 对象当中。通过  `YGLayout` 来设置布局属性。

在 `UIView+Yoga.h` 的文件里:

```
/**
 The YGLayout that is attached to this view. It is lazily created.
 */
@property (nonatomic, readonly, strong) YGLayout *yoga;

/**
 In ObjC land, every time you access `view.yoga.*` you are adding another `objc_msgSend`
 to your code. If you plan on making multiple changes to YGLayout, it's more performant
 to use this method, which uses a single objc_msgSend call.
 */
- (void)configureLayoutWithBlock:(YGLayoutConfigurationBlock)block
    NS_SWIFT_NAME(configureLayout(block:));
```

可以看到一个名为 `yoga` 的 `YGLayout` 只读对象，和 `configureLayoutWithBlock:(YGLayoutConfigurationBlock)block` 方法，并且还使用了  `NS_SWIFT_NAME()` 来定义在 Swift 里的方法名。

这样我们就可以直接使用 UIView 的实例对象，来直接设置它对应的布局了。

### isEnabled

`YGLayout.h` 里是这么定义 `isEnabled` 的。

```
/**
 The property that decides during layout/sizing whether or not styling properties should be applied.
 Defaults to NO.
 */
@property (nonatomic, readwrite, assign, setter=setEnabled:) BOOL isEnabled;
```

`isEnabled` 默认为 `NO`，需要我们在布局期间设置为 `YES`，来开启 Yoga 样式.
 
### applyLayoutPreservingOrigin:

对于这个方法，头文件里是这么解释的:

```
/**
 Perform a layout calculation and update the frames of the views in the hierarchy with the results.
 If the origin is not preserved, the root view's layout results will applied from {0,0}.
 */
- (void)applyLayoutPreservingOrigin:(BOOL)preserveOrigin
    NS_SWIFT_NAME(applyLayout(preservingOrigin:));
```

简单来说，就是用于执行 layout 计算的。所以，一旦在布局代码完成之后，就要在`根视图`的属性 yoga 对象上调用这个方法，应用布局到`根视图`和`子视图`。


## 布局演示

下面通过实例来介绍如何使用 `Yoga` 进行 `FlexBox` 布局。

### 居中显示

```
[self configureLayoutWithBlock:^(YGLayout * layout) {
                layout.isEnabled = YES;
                layout.justifyContent =  YGJustifyCenter;
                layout.alignItems     =  YGAlignCenter;
            }];
        
[self.redView configureLayoutWithBlock:^(YGLayout * layout) {
                layout.isEnabled = YES;
                layout.width=layout.height= 100;
            }];
            
[self addSubview:self.redView];

[self.yoga applyLayoutPreservingOrigin:YES];
    
```

效果如下:

![75F9B6E0-1C63-4A91-97EE-3F3A6087BE3B](https://user-gold-cdn.xitu.io/2017/12/25/1608d27539c27747?w=168&h=300&f=png&s=4181)

我们真正的布局代码，只用设置 `Flex Container` 的 `justifyContent` 和 `alignItems` 就可以了.


### 嵌套布局

让一个 `view` 略小于其 `superView`,边距为10:

```
    [self.yellowView configureLayoutWithBlock:^(YGLayout *layout) {
                layout.isEnabled = YES;
                layout.margin = 10;
                layout.flexGrow = 1;
            }];
    [self.redView addSubview:self.yellowView];
```

效果如下：

![E2BE382B-85DA-47E9-93D3-FD94DD7A1ABA](https://user-gold-cdn.xitu.io/2017/12/25/1608d2754cd48389?w=169&h=300&f=png&s=4414)

布局代码只用设置, View 的 `margin` 和 `flexGrow `.

### 等间距排列

纵向等间距的排列一组 view:

```
            [self configureLayoutWithBlock:^(YGLayout *layout) {
                layout.isEnabled = YES;
                
                layout.justifyContent =  YGJustifySpaceBetween;
                layout.alignItems     =  YGAlignCenter;
            }];
            
            for ( int i = 1 ; i <= 10 ; ++i )
            {
                UIView *item = [UIView new];
                item.backgroundColor = [UIColor colorWithHue:( arc4random() % 256 / 256.0 )
                                                  saturation:( arc4random() % 128 / 256.0 ) + 0.5
                                                  brightness:( arc4random() % 128 / 256.0 ) + 0.5
                                                       alpha:1];
                [item  configureLayoutWithBlock:^(YGLayout *layout) {
                    layout.isEnabled = YES;
                    
                    layout.height     = 10*i;
                    layout.width      = 10*i;
                }];
                
                [self addSubview:item];
            }
```

效果如下:

![8E963A9D-F9AF-47A6-966C-5AEAA836E598](https://user-gold-cdn.xitu.io/2017/12/25/1608d2754fec530a?w=169&h=300&f=png&s=5987)

只要设置 `Flex Container` 的 `layout.justifyContent =  YGJustifySpaceBetween`，就可以很轻松的做到。

### 等间距，自动设宽

让两个高度为 `100` 的 `view` 垂直居中,等宽,等间隔排列,间隔为10.自动计算其宽度：

```
            [self configureLayoutWithBlock:^(YGLayout *layout) {
                layout.isEnabled = YES;
                layout.flexDirection  =  YGFlexDirectionRow;
                layout.alignItems     =  YGAlignCenter;
                
                layout.paddingHorizontal = 5;
            }];
            
            
            
            YGLayoutConfigurationBlock layoutBlock =^(YGLayout *layout) {
                layout.isEnabled = YES;
                
                layout.height= 100;
                layout.marginHorizontal = 5;
                layout.flexGrow = 1;
            };
            
            
            [self.redView configureLayoutWithBlock:layoutBlock];
            [self.yellowView configureLayoutWithBlock:layoutBlock];
            
            [self addSubview:self.redView];
            [self addSubview:self.yellowView];

```

效果如下 :

![7221D26D-E78E-4AF3-B129-6D983C842](https://user-gold-cdn.xitu.io/2017/12/25/1608d2756ab3039f?w=169&h=300&f=png&s=4475)

我们只要设置 `Flex Container` 的 paddingHorizontal ，以及 `Flex Item`的marginHorizontal，flexGrow 就可以了。并且可以复用  `Flex Item` 的 layout 布局样式。



### UIScrollView 排列自动计算 contentSize

在 `UIScrollView` 顺序排列一些 `view`,并自动计算 `contentSize`：

```
            [self configureLayoutWithBlock:^(YGLayout *layout) {
                layout.isEnabled = YES;
                layout.justifyContent =  YGJustifyCenter;
                layout.alignItems     =  YGAlignStretch;
            }];
            
            UIScrollView *scrollView = [[UIScrollView alloc] init] ;
            scrollView.backgroundColor = [UIColor grayColor];
            [scrollView configureLayoutWithBlock:^(YGLayout *layout) {
                layout.isEnabled = YES;

                layout.flexDirection = YGFlexDirectionColumn;
                layout.height =500;
            }];
            [self addSubview:scrollView];

            UIView *contentView = [UIView new];
            [contentView configureLayoutWithBlock:^(YGLayout * _Nonnull layout) {
                layout.isEnabled = YES;
            }];
            
            
            for ( int i = 1 ; i <= 20 ; ++i )
            {
                UIView *item = [UIView new];
                item.backgroundColor = [UIColor colorWithHue:( arc4random() % 256 / 256.0 )
                                                  saturation:( arc4random() % 128 / 256.0 ) + 0.5
                                                  brightness:( arc4random() % 128 / 256.0 ) + 0.5
                                                       alpha:1];
                [item  configureLayoutWithBlock:^(YGLayout *layout) {
                    layout.isEnabled = YES;

                    layout.height     = 20*i;
                    layout.width      = 100;
                    layout.marginLeft = 10;
                }];

                [contentView addSubview:item];
            }
            
            [scrollView addSubview:contentView];
            [scrollView.yoga applyLayoutPreservingOrigin:YES];
            scrollView.contentSize = contentView.bounds.size;
```

效果如下:

![679CD74E-2C82-4B4F-871A-46D42832C8CB](https://user-gold-cdn.xitu.io/2017/12/25/1608d2756ff7af1e?w=171&h=300&f=jpeg&s=4742)


布置 `UIScrollView` 主要是使用了一个中间 `contentView`,起到了计算 `scrollview` 的 `contentSize` 的作用。这里要注意的是，要在`scrollview`调用完 `applyLayoutPreservingOrigin:` 后进行设置,否则得不到结果。

UIScrollView 的用法，目前在网上也没找到比较官方的示例，完全是笔者自己摸索的，欢迎知道的大佬指教。

>上面所用的示例代码，已经上传至 [GitHub](https://github.com/ZenonHuang/MyDemos/tree/master/YogaSample)

# 总结

FlexBox 的确是一个非常适用于移动端的布局方式，语意清晰，性能稳定，现在移动端 UI 视图越来越复杂，尤其是在所有浏览器都已经支持了 FlexBox 之后，作为移动开发者有必要了解新的解决方式。

大家在熟练使用 YogaKit 的方式之后，也可以尝试自己封装一套布局代码，加快开发效率。

# 参考:

[Flex 布局教程：语法篇](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)

[FlexBox 布局模型](http://yangjh.oschina.io/front-end/css/flex.html)

[YogaKit](https://facebook.github.io/yoga/docs/api/yogakit/)

[Yoga Tutorial: Using a Cross-Platform Layout Engine](https://www.raywenderlich.com/161413/yoga-tutorial-using-cross-platform-layout-engine?utm_source=raywenderlich.com+Weekly&amp;utm_campaign=e7e557ef6a-raywenderlich_com_Weekly_Issue_125&amp;utm_medium=email&amp;utm_term=0_83b6edc87f-e7e557ef6a-415701885)

[从 Auto Layout 的布局算法谈性能](https://draveness.me/layout-performance)






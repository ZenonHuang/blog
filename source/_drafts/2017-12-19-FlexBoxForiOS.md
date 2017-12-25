# 为什么要了解 FlexBox?

最近时不时的听到关于 FlexBox 的声音，除了在 `Weex` 以及 `React Native` 两个著名的跨平台项目里有用到 FlexBox 外，` AsyncDisplayKit` 也同样引入了 FlexBox 。 

先说说 iOS 本身提供给我们 2 种布局方式:

- Frame,直接设置横纵坐标，并指定宽高。
- Auto Layout，相对布局，设置相对位置的约束进行布局。

Frame 没什么太多可说的了，相对于原点坐标，设置绝对值。

`Auto Layout` 本身用意是好的，试图让我们从 Frame 中解放出来，摆脱关于坐标，大小的刻板思考方式。转而利用 UI 之间的相对位置关系，设置对应约束进行布局。

但是 `Auto Layout` 好心并未做成好事，它的语法又臭又长! 至今学习 iOS 两年，我使用到原生 `Auto Layout` 语法的时候屈指可数。只能靠 [Masonry](https://github.com/SnapKit/Masonry) 这样的第三方库来使用它。

## Auto Layout 的原理

说完了 `Auto Layout` 的使用，再来看看它工作原理。 

实际上，我们设置 `Auto Layout` 的约束，就构成一系列的条件,成为一个方程。然后解出 Frame 的坐标和大小。

例如，我们设置一个名为 A 的 UI :

``` 
 A.center = super.center
 A.width  = 40
 A.height = 40
``` 

A.frame = (super.center.x,super.center.y,40,40)

再设置一个 B:

```
B.width  =  A.width
B.height =  A.height
B.top    =  A.bottom + 50
B.left   =  A.left
``` 

B.frame =  ( A.x , A.y + A.height + 50 , A.width , A.height )

如图：

![ZenonHuang_FlexBox_1](http://7xiym9.com1.z0.glb.clouddn.com/B9E4AB21-289F-46F5-A475-EFD6828E2757-1.png)


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

![ZenonHuang_FlexBox_2](http://7xiym9.com1.z0.glb.clouddn.com/1C2DBFC6-B0B2-4FE5-BB81-039B196CCA15.png)


虽然测试结果难免有偏差，但是根据折线图可以明显发现，FlexBox 的布局性能是比较接近 Frame 的。 

60 FPS 作为一个 iOS 流畅度的黄金标准，要求布局在 0.0166667 s 内完成，Auto Layout 在超过 50 个视图的时候，可能保持流畅就会开始有问题了。

本次测试使用的机器配置如下：
![ZenonHuang_FlexBox_3](http://7xiym9.com1.z0.glb.clouddn.com/36DD7D25-B13A-4A2C-8EC7-A58E29089E59.png)

采用 Xcode9.2 ,iPad Pro (12.9-inch)(2nd generation) 模拟器。

测试布局的项目代码上传在 [GitHub](https://github.com/ZenonHuang/MyDemos/tree/master/LayoutTest)


# FlexBox 是什么？

`FlexBox` 是一种 UI 布局方式，并得到了所有浏览器的支持。`FlexBox` 首先是基于 [*盒装状型*](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Box_Model/Introduction_to_the_CSS_box_model) 的，Flexible 意味着弹性，使其能适应不同屏幕，补充盒状模型的灵活性。

`FlexBox` 把每个视图，都看作一个矩形盒子，拥有内外边距，沿着主轴方向排列，并且，同级的视图之间没有依赖。

和 `Auto Layout` 类似，`FlexBox` 采用了描述性的语言去进行布局，而不像 Frame 直接用绝对值坐标进行布局。

>弹性布局的主要思想是让 Flex Container 有能力来改变 Flex Item 的宽度和高度，以填满可用空间（主要是为了容纳所有类型的显示设备和屏幕尺寸）的能力。

最重要的是, `FlexBox` 布局与方向无关，常规的布局设计缺乏灵活性，无法支持大型和复杂的应用程序（特别是涉及到方向转变，缩放、拉伸和收缩等）。

## FlexBox 组成

采用 Flex 布局的元素，称为 `Flex Container`。

`Flex Container` 的所有子元素，称为 `Flex Item`。

![ZenonHuang_FlexBox_4](http://7xiym9.com1.z0.glb.clouddn.com/D0FA8E7B-D39D-46FD-9532-FCF1FC212722.png)



下面会讲一下 FlexBox 里面的一些概念，方便之后进行 FlexBox 的使用。

### Flex Container

前面提到了，FlexBox 的一个特点，就是视图之间，是没有依赖的。

`Flex Item` 的排布，就依赖于 `Flex Container` 的属性设置，而不用相互之间进行设置。

下面就说一下 Flex Containner 的属性设置。

#### Flex Direction

FlexBox 有一个 `主轴(main axis)`，和`侧轴(cross axis)`的概念。

视图排布依据主轴的方向，侧轴，则是垂直于主轴的方向。

`Flex Direction` 决定了 `Flex Containner ` 内的主轴排布方向。方向设置有 `行(Row)`，`列(Column)`的概念,代表水平和垂直方向。

主轴默认为 Row (从左到右):

![87691D2C-34C3-4805-B960-4D8217717D98](http://7xiym9.com1.z0.glb.clouddn.com/87691D2C-34C3-4805-B960-4D8217717D98.png?imageView2/2/h/50)

同时，也可以设置 RowRevers(从右至左):
![1F430BC5-A0BE-474B-9791-23F2B308AEE9](http://7xiym9.com1.z0.glb.clouddn.com/1F430BC5-A0BE-474B-9791-23F2B308AEE9.png?imageView2/2/h/50)


Column(从上到下):
![AA2DF5F6-1164-4440-ACF2-9897D0D82730](http://7xiym9.com1.z0.glb.clouddn.com/AA2DF5F6-1164-4440-ACF2-9897D0D82730.png?imageView2/2/h/200)


ColumnRevers(从下到上):
![C33D6321-66C3-4A78-B429-E82B3F83CB6E](http://7xiym9.com1.z0.glb.clouddn.com/C33D6321-66C3-4A78-B429-E82B3F83CB6E.png?imageView2/2/h/200)

#### Flex Wrap

Flex Wrap 决定在轴线上排列不下时，视图的换行方式。

Flex Wrap 默认设置为 NoWrap，不会换行，一直沿着主轴排列到屏幕之外:

![9C0FD351-E504-4A6B-A442-E3DE1E084FAC](http://7xiym9.com1.z0.glb.clouddn.com/9C0FD351-E504-4A6B-A442-E3DE1E084FAC.png?imageView2/2/h/50)

设置为 Wrap ,则空间不足时，自动换行:

![2C14AAC1-6DDB-4450-B393-5497E0743AFC](http://7xiym9.com1.z0.glb.clouddn.com/2C14AAC1-6DDB-4450-B393-5497E0743AFC.png?imageView2/2/h/50)


设置 WrapReverse，则换行方向与 Wrap 相反:

![35B10621-C2BD-4699-B5B5-4383B35F510E](http://7xiym9.com1.z0.glb.clouddn.com/35B10621-C2BD-4699-B5B5-4383B35F510E.png?imageView2/2/h/50)

这是一个非常有用的属性。比如典型的`九宫格布局`，iOS 如果不是用 `UICollectionView` 做，那么就需要保存 `9` 个实例，然后做判断，计算 frame ，可维护性实在不高。使用`UICollectionView` 可以很好的解决布局，但很多场景并不能复用，做起来也不是特别简单。 

FlexBox 布局的话，用 `Flex Wrap` 属性设置 `Wrap` 就可以直接搞定。

移动平台上相似的方案，比如 Android 的 Linear Layout 和 iOS 的 UIStackView ，但却远没有 FlexBox 强大。

#### Display

Display 选择是否计算它，默认为 Flex. 如果设置为 None 自动忽略该视图的计算。

在根据逻辑显示 UI 时，比较有用。

比如我们现有的业务，需要显示的腾讯身份标示。按照一般做法，多个 icon 互相连成一排，根据身份去设置不同的距离，同时隐藏其他 icon ,比较的麻烦。iOS 最好的办法是使用 UIStackView ，这又有版本兼容等问题。而使用 FlexBox 布局,当不是某个身份时，只要设置 Display 为 None,就不会被纳入 UI 计算当中。

#### Flex Flow

flex-flow属性是flex-direction属性和flex-wrap属性的简写形式

#### Justify Content

`justify-content` 用于定义 `Flex Item` 在主轴上的对齐方式:FlexStart(主轴起点对齐)，FlexEnd(主轴终点对齐)，Center(居中对齐)。

还有SpaceBetween（两端对齐）:

![7F4F84F0-6B50-462A-BDA6-D11D087FFCE0](http://7xiym9.com1.z0.glb.clouddn.com/7F4F84F0-6B50-462A-BDA6-D11D087FFCE0.png?imageView2/2/h/50)

设置两端对齐，让 `Flex Item` 之间的间隔相等。

SpaceAround(间隔相等排列)：

![3B7E08DD-6F78-4A8D-9D56-07565E5F9E24](http://7xiym9.com1.z0.glb.clouddn.com/3B7E08DD-6F78-4A8D-9D56-07565E5F9E24.png?imageView2/2/h/50)

让 `Flex Item` 两侧的间隔相等

#### Align Items

`align-items` 属性定义 `Flex Item` 在侧轴上的对齐方式。和主轴对齐方式  `Justify Content` 类似，除了 FlexStart ,FlexEnd,Center,SpaceBetween,SpaceAround 可以设置之外，还有 Baseline(基线对齐)：

![25B54897-E6D4-4742-837E-13E5E4D827DA](http://7xiym9.com1.z0.glb.clouddn.com/25B54897-E6D4-4742-837E-13E5E4D827DA.png?imageView2/2/h/100)

Baseline 如果伸缩项目的行内轴与侧轴为同一条，则该值与[flex-start]等效。 其它情况下，该值将参与基线对齐。


Stretch：

Stretch伸缩项目拉伸填充整个伸缩容器。此值会使项目的外边距盒的尺寸在遵照「min/max-width/height」属性的限制下尽可能接近所在行的尺寸。

Auto:
![F40B6D31-225F-4A99-9209-15886475CC1F](http://7xiym9.com1.z0.glb.clouddn.com/F40B6D31-225F-4A99-9209-15886475CC1F.png?imageView2/2/h/100)


如果 `Flex Item` 未设置数值或设为auto，将占满整个容器的高度

#### Align Content

align-content定义了多根轴线的对齐方式。如果项目只有一根轴线，该属性不起作用。属性可以用来调准伸缩行在伸缩容器里的对齐方式，这与调准伸缩项目在主轴上对齐方式的[justify-content]属性类似。只不过这里元素是以一行为单位。请注意本属性在只有一行的伸缩容器上没有效果。当使用flex-wrap:wrap时候多行效果就出来了。

![6C8FB222-16DC-4B6E-A2D9-251C4CA69F8E](http://7xiym9.com1.z0.glb.clouddn.com/6C8FB222-16DC-4B6E-A2D9-251C4CA69F8E.png)


### Flex Item

#### Align-self

`align-self`属性允许单个项目有与其他项目不一样的对齐方式，可覆盖`align-items`属性。默认值为`auto`，表示继承父元素的`align-items`属性，如果没有父元素，则等同于`stretch`。 

#### FlexGrow

用于分配剩余空间的比例。

例如：
整体宽度 100 , sub1 宽为 10 ，sub2 宽为 20 ，则剩余空间为 70。设置 FlexGrow 就是分配这 70 宽度的比例。

默认值为0，如果没有去定义该属性，该布局是不会拥有分配剩余空间权利的。

默认为`0`，即如果存在剩余空间，也不放大。

本例中b,c两项都显式的定义了flex-grow，可以看到总共将剩余空间分成了4份，其中b占1份，c占3分，即1:3.

如果所有项目的`flex-grow`属性都为`1`，则它们将等分剩余空间（如果有的话）。如果一个项目的`flex-grow`属性为`2`，其他项目都为`1`，则前者占据的剩余空间将比其他项多一倍。

#### Flex Shrink

#### flexBasis



想了解更多 FlexBox 属性，可以参考 [A Complete Guide to Flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)

## FlexBox 的实现 -- Yoga 



###  YGLayout 对象

Flex 方向，对齐内容，对齐项目，填充和边距。
 
一旦完成，你在根视图的 YGLayout 上调用applyLayout(preservingOrigin:)。这会计算并应用布局到根视图和子视图。
 
### applyLayoutPreservingOrigin:




# 基于 Yoga 实现一个 FlexBox 布局系统

# 参考:

[Flex 布局教程：语法篇](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)

[YogaKit](https://facebook.github.io/yoga/docs/api/yogakit/)

[Yoga Tutorial: Using a Cross-Platform Layout Engine](https://www.raywenderlich.com/161413/yoga-tutorial-using-cross-platform-layout-engine?utm_source=raywenderlich.com+Weekly&amp;utm_campaign=e7e557ef6a-raywenderlich_com_Weekly_Issue_125&amp;utm_medium=email&amp;utm_term=0_83b6edc87f-e7e557ef6a-415701885)



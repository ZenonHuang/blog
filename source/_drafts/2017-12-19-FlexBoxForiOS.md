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

![B9E4AB21-289F-46F5-A475-EFD6828E2757](http://7xiym9.com1.z0.glb.clouddn.com/B9E4AB21-289F-46F5-A475-EFD6828E2757-1.png)


### Cassowary

`Auto Layout` 内部有专门用来处理约束关系的算法，我一直以为是苹果自家研发的，查阅资料才发现是来自一个叫 `Cassowary` 的算法。

>Cassowary是个解析工具包，能够有效解析线性等式系统和线性不等式系统，用户的界面中总是会出现不等关系和相等关系，Cassowary开发了一种规则系统可以通过约束来描述视图间关系。约束就是规则，能够表示出一个视图相对于另一个视图的位置。
> - 戴铭 [<深入剖析Auto Layout，分析iOS各版本新增特性>](https://ming1016.github.io/2015/11/03/deeply-analyse-autolayout/)

有兴趣的可以进一步了解该算法的实现。

## Frame / Auto Layout  / FlexBox 的性能对比

在对 `Auto Layout` 进行一番之后,我们很容易得出 `Auto Layout` 因为多余的计算，性能差于 Frame 的结论。

但究竟差多少呢？到什么地步？FlexBox 的表现又如何呢?

Don't talk, show you my code.

|   | Frame | Auto Layout | FlexBox |
| --- | --- | --- | --- |
| 非嵌套 |  |  |  |
| 嵌套 |  |  |  |


# FlexBox 是什么？

FlexBox 是一种 UI 布局方式，并得到了所有浏览器的支持。和 `Auto Layout`类似，它也采用了描述性的语言去进行布局，而不使用绝对值进行布局。

FlexBox 首先是基于 盒装模型 的，Flexible 意味着弹性，使其能适应不同屏幕，补充盒装模型的灵活性。


>弹性布局的主要思想是让容器有能力来改变项目的宽度和高度，以填满可用空间（主要是为了容纳所有类型的显示设备和屏幕尺寸）的能力。

>最重要的是弹性盒子布局与方向无关，相对于常规的布局（块是垂直和内联水平为基础），很显然，这些工作以及网页设计缺乏灵活性，无法支持大型和复杂的应用程序（特别当它涉及到改变方向，缩放、拉伸和收缩等）。

## FlexBox 能解决的问题

## FlexBox 组成

采用 Flex 布局的元素，称为 Flex Container。Flex Container 的所有子元素，称为 Flex Item。

## FlexBox 的实现 -- Yoga 

### ASDK

###  ComponentKit

# 基于 Yoga 实现一个 FlexBox 布局系统



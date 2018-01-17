# FDTemplateLayoutCell

[FDTemplateLayoutCell](https://github.com/forkingdog/UITableView-FDTemplateLayoutCell) 是一个用于  UITableViewCell 高度自动计算的库。

UITableViewCell 的高度计算比较麻烦，系统的自动算高也存在效率问题，并且不兼容低版本。关于这些介绍可以看看 [优化UITableViewCell高度计算的那些事](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/) 。

## FDTemplateLayoutCell 是怎么解决的？

### TemplateLayoutCell

由于在 `tableView: heightForRowAtIndexPath:` 里取高度的时候，Cell 还没有进行设置。

所以 `FDTemplateLayoutCell`使用了一个 TemplateLayoutCell ，达到预先获取高度的目的，并不显示在屏幕上。

同时 TemplateLayoutCell 和每个 UITableViewCell ReuseID 做一一对应。 

重要的一点是， TemplateLayoutCell 通过 UITableView 的 dequeueCellForReuseIdentifier: 方法 lazy 创建并保存，所以要求这个 ReuseID 必须已经被注册到了 UITableView 中。

### 根据 autolayout 约束自动计算

调用系统在 iOS6 就提供的 `systemLayoutSizeFittingSize:` ,得到高度。

### 根据 index path 的一套高度缓存机制

已经得到高度之后，可以把这一组高度给存起来。

计算一次之后，后面的高度询问都会命中缓存，减少了非常可观的多余计算。

问题，高度变化怎么更新？更新后，如何刷新 UI.

### 自动的缓存失效机制

由于高度缓存了，就存在一个刷新的问题。

FDTemplateLayoutCell 对系统的几个刷新方法都进行了交换，来达到更新的目的。

无须担心你数据源的变化引起的缓存失效，当调用如-reloadData，-deleteRowsAtIndexPaths:withRowAnimation:等任何一个触发 UITableView 刷新机制的方法时，已有的高度缓存将以最小的代价执行失效。如删除一个 indexPath 为 [0:5] 的 cell 时，[0:0] ~ [0:4] 的高度缓存不受影响，而 [0:5] 后面所有的缓存值都向前移动一个位置。自动缓存失效机制对 UITableView 的 9 个公有 API 都进行了分别的处理，以保证没有一次多余的高度计算。

### 预缓存机制

在 UITableView 没有滑动的空闲时刻执行，计算和缓存那些还没有显示到屏幕中的 cell，整个缓存过程完全没有感知，这使得完整列表的高度计算既没有发生在加载时，又没有发生在滑动时，同时保证了加载速度和滑动流畅性。



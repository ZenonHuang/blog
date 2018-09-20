---
layout: post
date: 2017-05-23 15:25:00
title: MMPopupView 源码阅读
category: 技术
tags: 
- 源码阅读
keywords: iOS
description: 为了自制一个弹出框，对2个自己平常比较常用的弹框库的源码进行学习
---

## MMPopupView

[MMPopupView](https://github.com/adad184/MMPopupView) 是我自己比较喜欢的一个库，风格简洁好看，调用和自定义都很方便。缺点是对横竖屏的支持不是很好。

在我初学的时候，也经常阅读作者的博客学习。

以下是 MMPopupView 的整体结构：

![MMPopupView结构](http://7xibqv.com1.z0.glb.clouddn.com/2015-09-07-opensource-mmpopupview8.jpg)

这次阅读，就到 MMPopupView 的阶段，因为之后的如 MMAlertView , MMSheetView 等都是基于 MMPopupView 自定义出来的。

### MMPopupWindow

首先从作为所有 UI 的容器MMPopupWindow说起。

#### 单例

在 MMPopupWindow 的类里，可以看到单例方法 ：


```
+ (MMPopupWindow *)sharedWindow
{
    static MMPopupWindow *window;
    static dispatch_once_t onceToken;
    
    dispatch_once(&onceToken, ^{
        window = [[MMPopupWindow alloc] initWithFrame:[[UIScreen mainScreen] bounds]];
        window.rootViewController = [UIViewController new];
    });
    
    return window;
}
```


首先初始化一个全屏大小的 window,然后为 window 设定 rootViewController 。

#### 初始化

初始化的方法为:

```
- (instancetype)initWithFrame:(CGRect)frame
{
    self = [super initWithFrame:frame];
    
    if ( self )
    {
        self.windowLevel = UIWindowLevelStatusBar + 1;
        
        UITapGestureRecognizer *gesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(actionTap:)];
        gesture.cancelsTouchesInView = NO;
        gesture.delegate = self;
        [self addGestureRecognizer:gesture];
    }
    return self;
}
```



##### windowLevel

注意到有一个 windowLevel 的设置：

```
self.windowLevel = UIWindowLevelStatusBar + 1;
```

windowLevel 为 UIWindowLevel 类型. 

而 UIWindowLevel 实际的值为 CGFloat 类型 .

UIWindow 在显示时，会根据 UIWindowLevel 做排序，Level 值高的将排在前面。

系统有三个 UIWindowLevel 层级:

```
UIKIT_EXTERN const UIWindowLevel UIWindowLevelNormal;
UIKIT_EXTERN const UIWindowLevel UIWindowLevelAlert;
UIKIT_EXTERN const UIWindowLevel UIWindowLevelStatusBar __TVOS_PROHIBITED;
```

UIWindow 的默认级别是 UIWindowLevelNormal .

对应的关系为： 

```
UIWindowLevelNormal < UIWindowLevelStatusBar < UIWindowLevelAlert .
```

而 UIWindowLevel 的值可以是任意的，代码里的 `UIWindowLevelStatusBar + 1` 表示级别在 StatusBar 和 Alert 之间：

```
UIWindowLevelStatusBar < windowLevel < UIWindowLevelAlert 
```


##### 添加手势

这里添加了一个 tap 的手势：

```
  UITapGestureRecognizer *gesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(actionTap:)];
        gesture.cancelsTouchesInView = NO;
        gesture.delegate = self;
        [self addGestureRecognizer:gesture];

```

可以看到，在指定手势响应方法之外，对 `cancelsTouchesInView` 设置为 NO.

`cancelsTouchesInView` 默认为 YES，当识别到手势的时候，会终止发送所有触摸事件。
`cancelsTouchesInView` 为NO,则不发送终止触摸的消息，让 Tap 手势和其它响应方法，同时响应触摸事件。

#### Tap 手势代理

设置是否响应点击操作,返回NO代表不做操作,返回YES则会做出响应.

```
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldReceiveTouch:(UITouch *)touch
{
    //self.attachView.mm_dimBackgroundView 为  self.attachView 的半透明黑色背景
    return ( touch.view == self.attachView.mm_dimBackgroundView );
}
```

从上面的代码可以看到，从判断 touch.view 和 self.attachView.mm_dimBackgroundView是否一样，来决定做不做响应。所以只有点击背景时，才会有响应。


#### attachView

从下面代码看到，attachView实际就是根控制器的视图。

```
- (UIView *)attachView
{
    return self.rootViewController.view;
}
```

#### 手势响应事件

下面是响应点击的代码

```
- (void)actionTap:(UITapGestureRecognizer*)gesture
{
    if ( self.touchWildToHide && !self.mm_dimBackgroundAnimating )
    {
        for ( UIView *v in [self attachView].mm_dimBackgroundView.subviews )
        {
            if ( [v isKindOfClass:[MMPopupView class]] )
            {
                MMPopupView *popupView = (MMPopupView*)v;
                [popupView hide];
            }
        }
    }
}
```

当 touchWildToHide 为 YES ,以及 mm_dimBackgroundAnimating 为NO,则会把 [self attachView].mm_dimBackgroundView 里的所有 MMPopupView 实例隐藏。

#### cacheWindow

```
- (void)cacheWindow
{
    [self makeKeyAndVisible];
    [[[UIApplication sharedApplication].delegate window] makeKeyAndVisible];
    
    [self attachView].mm_dimBackgroundView.hidden = YES;
    self.hidden = YES;
}
```

首先 MMPopupWindow 发送 makeKeyAndVisible 消息，让它成为应用的主窗口并显示。

再获取 AppDelegate 的 window 让它成为主窗口并显示。

再将背景和MMPopupWindow都给隐藏。

### MMPopupView

#### 初始化设置

##### setup

首先初始化调用了 `setup` 方法

```
- (void)setup
{
    self.type = MMPopupTypeAlert;
    self.animationDuration = 0.3f;
    self.attachedView = [MMPopupWindow sharedWindow].attachView;
    
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(notifyHideAll:) name:MMPopupViewHideAllNotification object:nil];
}
```

指定 `type` 为 MMPopupTypeAlert. `animationDuration` 为 0.3 秒.

设置 `MMPopupView` 的 attachedView 为 `MMPopupWindow` 的  attachedView .

设置了一个名为 `MMPopupViewHideAllNotification `  的通知，和处理通知消息的方法 `notifyHideAll:`


看一下 `setType:` 方法：

```
- (void)setType:(MMPopupType)type
{
    _type = type;
    
    switch (type)
    {
        case MMPopupTypeAlert:
        {
            self.showAnimation = [self alertShowAnimation];
            self.hideAnimation = [self alertHideAnimation];
            break;
        }
        case MMPopupTypeSheet:
        {
            self.showAnimation = [self sheetShowAnimation];
            self.hideAnimation = [self sheetHideAnimation];
            break;
        }
        case MMPopupTypeCustom:
        {
            self.showAnimation = [self customShowAnimation];
            self.hideAnimation = [self customHideAnimation];
            break;
        }
            
        default:
            break;
    }
}
```

以上代码可以看到，在判断 Type 后，都是针对两个 MMPopupView 的属性 showAnimation 和 hideAnimation 做设置。

它们都是 MMPopupBlock 类型的 block：

```
@property (nonatomic, copy) MMPopupBlock   showAnimation;       // custom show animation block.
@property (nonatomic, copy) MMPopupBlock   hideAnimation;       // custom hide animation block.
```

通过以上两个属性来设置 弹出 和 隐藏 动画。

#### 显示

##### showWithBlock

```
- (void)showWithBlock:(MMPopupCompletionBlock)block
{
    if ( block )
    {
        self.showCompletionBlock = block;
    }
    
    if ( !self.attachedView )
    {
        self.attachedView = [MMPopupWindow sharedWindow].attachView;
    }
    [self.attachedView mm_showDimBackground];
    
    MMPopupBlock showAnimation = self.showAnimation;
    
    NSAssert(showAnimation, @"show animation must be there");
    
    showAnimation(self);
    
    if ( self.withKeyboard )
    {
        [self showKeyboard];
    }
}
```

判断 MMPopupCompletionBlock ，进行设置。

判断 attachView,进行设置。

显示 attachedView 背景。

用 NSAssert 检测，调用 showAnimation 的动画 block。

然后检测是否有键盘一起弹出。


#### 隐藏

##### hideWithBlock

```
- (void)hideWithBlock:(MMPopupCompletionBlock)block
{
    if ( block )
    {
        self.hideCompletionBlock = block;
    }
    
    if ( !self.attachedView )
    {
        self.attachedView = [MMPopupWindow sharedWindow].attachView;
    }
    [self.attachedView mm_hideDimBackground];
    
    if ( self.withKeyboard )
    {
        [self hideKeyboard];
    }
    
    MMPopupBlock hideAnimation = self.hideAnimation;
    
    NSAssert(hideAnimation, @"hide animation must be there");
    
    hideAnimation(self);
}
```

判断 MMPopupCompletionBlock ，进行设置。

判断 attachView,进行设置。

隐藏 attachedView 背景。

然后检测是否有键盘，一起隐藏。

用 NSAssert 检测，调用 hideAnimation 的动画 block。




### MMPopupCategory

#### Associated Object

MMPopupCategory 里的很多方法都利用到了 Associated Object ,也就是 关联对象 来添加实例变量和获取实例变量。

这里是关联对象 getter/setter 方法的原型：

```
id objc_getAssociatedObject(id object, const void *key);

void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
```

object 是要添加属性的对象， key 是变量名称，value 是你要给定的值。

而 policy 为 objc_AssociationPolicy 类型，是一个枚举:

```
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```

不同的 `objc_AssociationPolicy` 对应了不通的属性修饰符：

| objc_AssociationPolicy | modifier |
| :-- | :-: |
| OBJC_ASSOCIATION_ASSIGN | assign |
| OBJC_ASSOCIATION_RETAIN_NONATOMIC | nonatomic, strong |
| OBJC_ASSOCIATION_COPY_NONATOMIC | nonatomic, copy |
| OBJC_ASSOCIATION_RETAIN | atomic, strong |
| OBJC_ASSOCIATION_COPY | atomic, copy |

而一般实现的属性都是用 `nonatomic` 和 `strong` 修饰符, 可以选择`OBJC_ASSOCIATION_RETAIN_NONATOMIC`。

具体的需要根据场景去确定。


#### mm_dimBackgroundView

这里的 mm_dimBackgroundView 位于 MMPopupCategory 里，根据代码可以看出来，是作为半透明黑色背景的。

```
//mm_dimBackgroundView
- (UIView *)mm_dimBackgroundView
{
    UIView *dimView = objc_getAssociatedObject(self, mm_dimBackgroundViewKey);
    
    if ( !dimView )
    {
        dimView = [UIView new];
        [self addSubview:dimView];
        [dimView mas_makeConstraints:^(MASConstraintMaker *make) {
            make.edges.equalTo(self);
        }];
        dimView.alpha = 0.0f;
        dimView.backgroundColor = MMHexColor(0x0000007F);
        dimView.layer.zPosition = FLT_MAX;
        
        self.mm_dimAnimationDuration = 0.3f;
        
        objc_setAssociatedObject(self, mm_dimBackgroundViewKey, dimView, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    
    return dimView;
}

```

## 总结

这次阅读的源码结构很清晰，不算难懂。

除了解到整个 MMPopupView 的设计思路之外，从中也学习到一些额外的知识和技巧。

打算学习它的方式，给出常用的 AlertView , SheetView ，方便使用者直接使用。
并提供内置动画，也支持自定义动画。



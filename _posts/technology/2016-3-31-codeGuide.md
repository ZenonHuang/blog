---
layout: post
date: 2016-03-31 23:50:46
title: OC代码小Tips
category: 技术
tags: 资料
keywords: iOS
description: iOS资料整理
excerpt: 关于属性修饰，方法使用，变量命名等日常开发，要时时注意的细节总结。保持良好的代码习惯！
---


首先要说明的是，命名的最大规则是可读的，清晰的。

其次，才考虑如何简洁的命名。


## 代码组织


在函数分组和protocol/delegate实现中使用#pragma mark -来分类方法，要遵循以下一般结构：

	#pragma mark - Lifecycle（View 的生命周期）  
	- (instancetype)init {}  
	- (void)dealloc {}  
	- (void)viewDidLoad {}  
	- (void)viewWillAppear:(BOOL)animated {}  
	- (void)didReceiveMemoryWarning {}  
	#pragma mark - Custom Accessors  （自定义访问器）
	- (void)setCustomProperty:(id)value {}  
	- (id)customProperty {}  
	#pragma mark - IBActions  
	- (IBAction)submitData:(id)sender {}  
	#pragma mark - Public  
	- (void)publicMethod {}  
	#pragma mark - Private  
	- (void)privateMethod {}  
	#pragma mark - Protocol conformance  
	#pragma mark - UITextFieldDelegate  
	#pragma mark - UITableViewDataSource  
	#pragma mark - UITableViewDelegate  
	#pragma mark - NSCopying  
	- (id)copyWithZone:(NSZone *)zone {}  
	#pragma mark - NSObject  
	- (NSString *)description {}  


## 注释


当需要注释时，注释应该用来解释这段特殊代码为什么要这样做。

任何被使用的注释都必须保持最新或被删除。

一般都避免使用块注释，因为代码尽可能做到自解释，只有当断断续续或几行代码时才需要注释。

例外：这不应用在生成文档的注释

## 字面量

NSString、NSDictionary、NSArray和NSNumber的字面值，应该在创建这些类的不可变实例时被使用。

请特别注意---nil值不能传入NSArray和NSDictionary字面值，因为这样会导致crash。

推荐
```objc
	NSArray *names = @[@"Brian", @"Matt", @"Chris", @"Alex", @"Steve", @"Paul"];  
	NSDictionary *productManagers = @{@"iPhone": @"Kate", @"iPad": @"Kamal", @"Mobile Web": @"Bill"};  
	NSNumber *shouldUseLiterals = @YES;  
	NSNumber *buildingStreetNumber = @10018;  
```
不推荐

	NSArray *names = [NSArray arrayWithObjects:@"Brian", @"Matt", @"Chris", @"Alex", @"Steve", @"Paul", nil];  
	NSDictionary *productManagers = [NSDictionary dictionaryWithObjectsAndKeys: @"Kate", @"iPhone", @"Kamal", @"iPad", @"Bill", @"Mobile Web", nil];  
	NSNumber *shouldUseLiterals = [NSNumber numberWithBool:YES];  
	NSNumber *buildingStreetNumber = [NSNumber numberWithInteger:10018]; 
	 
## 变量

### 实例变量

实例变量，应该在实现文件.m中声明。

如果一定要直接在.h文件声明，加上@priavte，另外，使用@private、@public，前面需要一个缩进空格。

一般以@property形式，在.h文件中声明。

>推荐多用属性，并且避免直接访问实例变量。

PS:尽可能保证 .h文件的简洁性，可以不公开的API就不要公开了，写在实现文件中即可。
### 属性

用@property声明属性后，默认的实例变量名称是

	_variableName。

在.m实现文件中，可以用@synthesize,改变实例变量名称

	 @synthessize variableName=goodVariableName
 

如果你使用 @synthesize 关键字时，不去指明实例变量的名字的话，像下面这样：

	@synthesize firstName;
	
那么实例变量将会被命成与属性一样的名字。 在这个例子中，实例变量将会被命名为 firstName ,并且不会加上下划线前缀。

#### 属性特性

所有属性特性应该显式地列出来，有助于阅读代码。

顺序与在Interface Builder连接UI元素时自动生成代码一致。

属性特性的顺序，应该是
>1.原子性:atomic,nonatomic
>
>2.读写:readonly,readwrite
>
>3.内存管理：weak,strong,copy。

- 属性可以存储一个代码块。为了让它存活到定义的块的结束，必须使用 copy （block 最早在栈里面创建，使用 copy让 block 拷贝到堆里面去）

- 你必须使用 nonatomic，除非特别需要的情况。在iOS中，atomic带来的锁特别影响性能。

- 对于非可变类（比如NSString,NSArray），通常使用copy修饰。

- 对于一个外部引用对象，且外部不会发生set操作的对象，使用readonly属性。比如在创建界面元素的时候:

		@interface BWView : UIView
		@property (nonatomic, strong, readonly) UIView* backgroundView;
		@end
		@implementation BWView
		@end

readonly 表明其不能修改只能读取，在这种情况下编译器将会合成一个名为backgroundView getter方法，而不是一个setBackgroundView方法。

- 为了完成一个共有的 getter 和一个私有的 setter，你应该声明公开的属性为 readonly ,并且在类扩展中重新定义通用的属性为 readwrite 的。

		//.h文件中
		@interface MyClass : NSObject
		@property (nonatomic, readonly, strong) NSObject *object;
		@end
		//.m文件中
		@interface MyClass ()
		@property (nonatomic, readwrite, strong) NSObject *object;
		@end

		@implementation MyClass
		//Do Something cool
		@end
		
		
### 可变对象

任何可以用一个可变的对象设置的(比如 NSString,NSArray,NSURLRequest)属性,它的内存管理类型必须是 copy 的。

这是为了防止在不明确的情况下，修改被封装好的对象的值。

>例如：array定义为 copy 的 NSArray 实例，当执行 array= mutableArray，copy 属性会让 array 的 setter 方法为 array = [mutableArray copy], [mutableArray copy] 返回的是不可变的 NSArray 实例，就保证了正确性。
>
>用其他属性修饰符修饰，容易在直接赋值的时候，array 指向的是 NSMuatbleArray 的实例，在之后可以随意改变它的值，就容易出错。


同时，应该避免在公开的接口中，暴露可变的对象。

因为这允许你的类的使用者，改变类自己的内部表示，并且破坏类的封装。你可以提供只读的属性，来返回你对象的不可变的副本。如：


	/* .h */
	@property (nonatomic, readonly) NSArray *elements

	/* .m */
	- (NSArray *)elements {
  	return [self.mutableElements copy];
	}
	

#### 私有属性

私有属性应该定义在类的实现文件的类的扩展 (匿名的 category) 中。不允许在有名字的 category(如 ZOCPrivate）中定义私有属性，除非你扩展其他类。

推荐

	@interface RWTDetailViewController ()  
	@property (strong, nonatomic) GADBannerView *googleAdView;  
	@property (strong, nonatomic) ADBannerView *iAdView;  
	@property (strong, nonatomic) UIWebView *adXWebView;  
	@end
	

## 常量

#### 常量使用

常量是容易重复被使用，和无需通过查找和代替就能快速修改值的。

常量应该使用static来声明，而不是使用#define，除非显式地使用宏
所有的单词首字母大写，和加上与类名有关的前缀

推荐

	static NSString * const RWTAboutViewControllerCompanyName = @"RayWenderlich.com";  
	static CGFloat const RWTImageThumbnailHeight = 50.0;  
	
不推荐

	#define CompanyName @"RayWenderlich.com"  
	#define thumbnailHeight 2  

### 常量小写k开头

常量以小写字母k开头，后续首字母大写

	static NSString* const kBWBarTitle = @"动态";


### 常量应该使用驼峰式命名规则。

推荐

	static NSTimeInterval const RWTTutorialViewControllerNavigationFadeAnimationDuration = 0.3;  
	
不推荐

	static NSTimeInterval const fadetime = 1.7;  
	

。
## 方法

### 方法参数名

方法参数名前，一般使用的前缀

>the,an,new,a

	- (void)setTitle:(NSString *)aTitle;
	- (void)setName:(NSString *)newName;
	- (id)keyForOption:(CDCOption *)anOption
	- (NSArray *)emailsForMailbox:(CDCMailbox *)theMailbox;
	- (CDCEmail *)emailForRecipients:(NSArray *)    theRecipients;

### "and"这个词的用法应该保留

”and“不应该用于多个参数来说明，就像initWithWidth:height以下这个例子.

推荐
 
	- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height;  

不推荐

	- (instancetype)initWith:(int)width and:(int)height;  // Never do this.  
	

### 方法长度
函数长度不要超过50行，小函数要比大函数可阅读性和可复用性强。

零元函数最好，一元函数也不错，二元函数担心了，三元函数有风险，高于三元需重构。函数的参数越多，引起其变化的因素就越多。越不利于以后的修改。

### 避免出现火车链式的情况

推荐

	BWAppSetting* shareSetting = [BWAppSetting GetInstance];
	BWLockDictionary* defaultSettings = [shareSetting appSetting];
	_needLogoutAccount = [[defaultSettings valueForKeyPath:NeedLogoutAccounts] retain];

不推荐

	_needLogoutAccount = [[[[BWAppSetting GetInstance] appSetting] valueForKey:NeedLogoutAccounts] retain];
	
	
### 取值方法

- 取值方法前，不要加前缀“get”。
- 如果BOOL属性的名字是一个形容词，属性就能忽略"is"前缀，但要指定get访问器的惯用名称。例如：

		@property (assign, getter=isEditable) BOOL editable;


### 方法调用

属性和幂等方法，推荐使用点标记语法访问。

其他情况，使用方括号标记语法。

推荐

	view.backgroundColor = [UIColor orangeColor];
	[UIApplication sharedApplication].delegate;
	
不推荐
	
	[view setBackgroundColor:[UIColor orangeColor]];
	UIApplication.sharedApplication.delegate;

## 委托代理模式

### 区分delegate和datasource

委托模式(delegate pattern)：事件发生的时候，委托者需要通知代理者。

数据源模式(datasource pattern): 委托者需要从数据源对象拉取数据。

重点:数据源模式，会要求数据源的对象来实现对应方法，达到委托者本身获取数据的目标。



	@class ZOCSignUpViewController;
    //Delegate
	@protocol ZOCSignUpViewControllerDelegate <NSObject>
	- (void)signUpViewControllerDidPressSignUpButton:(ZOCSignUpViewController *)controller;
	@end
	//DataSource
	@protocol ZOCSignUpViewControllerDataSource <NSObject>
	- (ZOCUserCredentials *)credentialsForSignUpViewController:(ZOCSignUpViewController *)controller;
	@end


	@interface ZOCSignUpViewController : UIViewController

	@property (nonatomic, weak) id<ZOCSignUpViewControllerDelegate> delegate;
	@property (nonatomic, weak) id<ZOCSignUpViewControllerDataSource> dataSource;

	@end

PS：

- Protocol单独用一个文件来创建。尽量不要与相关类混在一个文件中。

- 写delegate的时候类型应该为weak弱引用，以避免循环引用。

当delegate对象不存在后，我们写的delegate也就没有存在意义了自然是需要销毁的

	@property(nonatomic,weak) id<viewControllerDelegate> delegat;


## Block

Block 是 Objective-C 版本的 lambda 或者 closure（闭包）。

当使用代码块和异步分发的时候，要注意避免引用循环。在block中使用到self变量的时候，一定要先weak再strong.

推荐

	__weak typeof(self) weakSelf = self;
	[self doABlockOperation:^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (strongSelf) {
        ...
    }
	}];

不推荐

	[self executeBlock:^(NSData *data, NSError *error) {
    [self doSomethingWithData:data];
	}];
## 控制结构

### 分支结构

当需要满足一定条件时才执行某项操作时，最左边缘应该是愉快路径代码。不要将愉快路径代码内嵌到if语句中。多个return是正常合理的。

推荐

	- (void) someMethod {
	if (![someOther boolValue]) {
      return;
  	}
  	//Do something important
	}

不推荐

	- (void) someMethod {
 	 if ([someOther boolValue]) {
   	   //Do something important
 	 }
	}


### 循环结构

遍历可变容器(容器:如数组，字典等)，应该复制该容器，使用copy。

	NSArray *Arrays=[self.multableArray copy];
	for(id obj in Arrays){
	//do something
	}
	
尽量不要使用异常，尤其是不要将异常做为业务逻辑的一部分，在异常中尝试进行灾难恢复。

## 类
### 保持公共API简单

 其实也就说，保持.h文件的整洁和干净。

 避免 “厨房水槽（kitchen-sink）” 式的 API。如果一个函数压根没必要公开，就不要这么做。用私有类别，保证公共头文件整洁。"
### 初始化

保持init函数简洁，不要让init函数成为千行的大函数，当超过50行的时候，适当考虑分拆一下。

良好风格实例：

	- (void) commonInit
	{
    _rightAppendImageView = [UIImageView new];
    [self.contentView addSubview:_rightAppendImageView];
	}
	- (instancetype) initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier
	{
    self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
    if (!self) {
        return self;
    }
    [self commonInit];
    return self;
	}
	
## 其他

### 布尔值

Objective-C使用YES和NO。

因为true和false应该只在CoreFoundation，C或C++代码使用。

既然nil解析成NO，所以没有必要在条件语句比较。

不要拿某样东西直接与YES比较，因为当YES被定义为1和一个BOOL会被设置为8位。

推荐

	if (someObject) {}  
	if (![anotherObject boolValue]) {}  
	
不推荐

	if (someObject == nil) {}  
	if ([anotherObject boolValue] == NO) {}  
	if (isAwesome == YES) {} // Never do this.  
	if (isAwesome == true) {} // Never do this.  
###尽量不要在界面布局的写任何死数字

推荐

	CGFloat cellHeight = CGRectGetHeight(self.frame);
	CGFloat cellWidth = CGRectGetWidth(self.frame);
	CGRect numFrame = CGRectZero;
	numFrame.size = CGSizeMake(cellWidth,cellHeight);

不推荐
	
	CGFloat delta = SYSTEM_VERSION >= 7.0 ? 0.0f : -14.0f;
	newFrame = CGRectMake(245 + delta,
                              (self.frame.size.height - tipNewSize.height)/2,
                              tipNewSize.width,
                              tipNewSize.height);
    dotFrame = CGRectMake(258.0 + delta,  (self.frame.size.height - tipDotSize.height)/2,
                              tipDotSize.width,
                              tipDotSize.height);
    iconFrame = CGRectMake(245 + delta,
                               (self.frame.size.height - tipIconSize.height)/2,
                               tipIconSize.width,
                               tipIconSize.height);
    numFrame = CGRectMake(245+delta, (self.frame.size.height - tipNumSize.height)/2, tipNumSize.width, tipNumSize.height);

## 设计模式

PS：使用设计模式的最基本原则----
>除非你明确知道自己要做件什么事情，而且知道使用特定设计模式带来的影响.
>
>否则，不要刻意的使用设计模式。

### 单例模式
如果可能，请尽量避免使用单例而是依赖注入。 然而，如果一定要用，请使用一个线程安全的模式来创建共享的实例。对于 GCD，用 dispatch_once() 函数就可以咯。

	+ (instancetype)sharedInstance {  
 	 	static id sharedInstance = nil;  
  	 	static dispatch_once_t onceToken=0;  
 	 	dispatch_once(&onceToken, ^{  
    	sharedInstance = [[self alloc] init];  
 	 	});  
 	 	return sharedInstance;  
	}  


使用 dispatch_once()，来控制代码同步，取代了原来的约定俗成的用法。

	+ (instancetype)sharedInstance
	{
    static id sharedInstance;
    @synchronized(self) {
        if (sharedInstance == nil) {
            sharedInstance = [[MyClass alloc] init];
        }
    }
    return sharedInstance;
	}


dispatch_once() 的优点是，它更快，而且语法上更干净，因为dispatch_once()的意思就是 “把一些东西执行一次”，就像我们做的一样。

同时，这可以防止[possible and sometimes prolific crashes](http://cocoasamurai.blogspot.com/2011/04/singletons-your-doing-them-wrong.html)。

### 观察者模式

如果只是单纯的传递数据，不要使用观察者模式，容易导致逻辑链断裂。


>参考
>
>[禅与 Objective-C 编程艺术 （Zen and the Art of the Objective-C Craftsmanship 中文翻译）](https://github.com/oa414/objc-zen-book-cn/#%E9%BB%84%E9%87%91%E5%A4%A7%E9%81%93)
>
>[programming with objective-c](http://wiki.jikexueyuan.com/project/programming-with-objective-c/encapsulating-data.html)
>
>[Object-C代码规范](http://www.cocoachina.com/ios/20140520/8484.html)
>
>[Objective-C 编码建议](http://www.cocoachina.com/ios/20151118/14242.html)
>
>[Objective-C编码规范：26个方面解决iOS开发问题](http://www.csdn.net/article/2015-06-01/2824818-objective-c-style-guide/1)
>
>《iOS6编程实战》 人民邮电出版社

